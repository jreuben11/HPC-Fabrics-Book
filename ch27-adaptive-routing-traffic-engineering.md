# Chapter 27 — Adaptive Routing & Traffic Engineering

**Part III: Programmable Fabric** | ~20 pages

---

## Introduction

AI cluster fabrics are built for one workload above all others: distributed training. In that workload, hundreds of GPU nodes exchange gradient tensors in tight synchronization loops, and the fabric must deliver bandwidth uniformly to every node or the entire training step slows to the pace of the worst link. This chapter explains why the default path-selection algorithm — **ECMP** — fails for that workload, and how to replace or augment it with adaptive techniques that restore balance.

Equal-Cost Multi-Path (**ECMP**) was designed for environments where flows are many, short, and statistically independent. AllReduce traffic is the opposite: flows are few, large, and perfectly correlated by **NCCL**'s ring topology. When a small set of long-lived flows must be distributed across a small number of equal-cost spines, hash collisions are not an edge case — they are the dominant outcome. Section 27.1 quantifies this failure mode, showing how a two-spine fabric with four GPU flows can load one spine at 3x the rate of the other.

The adaptive routing techniques in this chapter span hardware, software, and protocol layers. Flowlet switching (§27.2) exploits the burst structure of **RDMA** traffic to safely reroute flows between ACK gaps, requiring no hardware changes and yielding imbalance ratios below 1.1x. NVIDIA's Adaptive Routing (§27.3) goes further, performing per-packet path selection in Spectrum-3/4 switch silicon based on real-time egress queue depth. Packet spraying (§27.4) is the most aggressive approach, routing each packet to an independently chosen path, and it demands receiver-side out-of-order handling and careful **DCQCN** congestion tracking.

For environments that need explicit, policy-driven traffic engineering, Section 27.5 introduces **SRv6** — Segment Routing over **IPv6** — which embeds a full forwarding path in the **IPv6** extension header, enabling deterministic steering of training flows across chosen links. Section 27.6 covers **UCMP** with the **BGP** Link Bandwidth extended community (**RFC 7311**), which proportionally weights forwarding across links of unequal capacity — essential during fabric upgrades or partial failures. Section 27.7 and the lab walkthrough provide the measurement tools to quantify imbalance before and after these interventions.

This chapter builds directly on the **BGP** fabric architecture introduced in Chapter 8 (Open NOS) and the **RoCEv2** congestion control model from Chapter 2. The **SRv6** section connects to Chapter 30 (**IPv6** in datacenter fabrics), and the **FRR** configurations used throughout the lab extend the routing protocol foundation from Chapter 17 (**BGP** tooling). Chapter 28 (Fault Tolerance and Resilience) picks up where this chapter leaves off, addressing what happens when adaptive routing is not enough and a link or switch actually fails mid-training.

## 27.1 ECMP Limitations: Why Static Hashing Fails for AllReduce

Equal-Cost Multi-Path (ECMP) is the universal multi-path forwarding mechanism in data-center fabrics. When a switch has multiple equal-cost paths to a destination, it selects one using a hash of the packet's 5-tuple (source IP, destination IP, source port, destination port, IP protocol). The same 5-tuple always hashes to the same output port — this is the foundational property that keeps TCP flows in-order.

### The 5-Tuple Hash Collision Problem

ECMP hashing distributes flows, not packets. All packets in a flow follow the same path. For AllReduce operations in GPU clusters, this creates a structural problem:

```
AllReduce communication pattern (Ring-AllReduce, 2 spines, 4 GPUs):
  GPU0 → GPU2: flow (10.0.0.1:5000 → 10.0.0.3:5001)  →  spine1
  GPU1 → GPU3: flow (10.0.0.2:5000 → 10.0.0.4:5001)  →  spine1  ← collision!
  GPU2 → GPU0: flow (10.0.0.3:5001 → 10.0.0.1:5000)  →  spine2
  GPU3 → GPU1: flow (10.0.0.4:5001 → 10.0.0.2:5000)  →  spine2
```

Two flows land on the same spine. One spine carries 2x the load of the other. The overloaded spine becomes a bottleneck, and the AllReduce step stalls waiting for the slowest link. This is not a hypothetical: in observed production GPU cluster training runs, ECMP imbalance has caused 30-40% degradation in step time due to fabric congestion on one spine while the other sits idle.

### Elephant Flow Concentration

AllReduce flows are by nature "elephant flows" — long-duration, high-bandwidth streams. Unlike web traffic where flows are short and ECMP imbalance averages out quickly, AllReduce flows persist for the entire duration of a training step (potentially minutes). A single unfortunate hash collision can create sustained congestion.

The hash function itself is deterministic and depends only on the 5-tuple. For a specific AllReduce ring, the 5-tuples are fixed: the GPU IP addresses are stable, and the port numbers are assigned by NCCL's rendezvous mechanism. This means the imbalance is not random — it is reproducible on every training iteration.

### Why Static Hashing Cannot Be Fixed by Configuration Alone

Changing the ECMP hash seed can redistribute flows differently, but cannot guarantee balance for a specific set of flows. The only mathematical guarantee comes from either:
1. **Per-packet load balancing**: routes individual packets across paths (at the cost of reordering)
2. **Dynamic path selection**: reroutes flows when congestion is detected (flowlet switching, adaptive routing)
3. **Application-level workarounds**: NCCL's `NCCL_IB_QPS_PER_CONNECTION` creates multiple QPs per GPU pair, diversifying the 5-tuple space

---

## Installation

The lab environment is built on a **SONiC** virtual switch topology deployed through **Containerlab**, which requires the **Containerlab** binary and a working **Docker** installation — the **SONiC**-VS container image is pulled automatically at lab startup. **FRR** provides the **BGP** routing plane and the route-map configuration used for Link Bandwidth **UCMP**, and its `vtysh` shell is the primary interface for verifying **ECMP** path state and **SRv6** policy route injection. **Scapy** is installed as a Python library to craft raw packets with controlled 5-tuple fields, enabling reproducible hash collision analysis that is impractical with standard traffic generators. The `iproute2` package must be current enough to include **SRv6** support (`ip route add ... encap seg6` syntax), which is satisfied by the version shipped in Ubuntu 24.04.

### System packages

```bash
sudo apt install -y \
    iproute2 \
    iperf3 \
    tcpdump \
    tshark \
    netcat-openbsd \
    python3 \
    python3-pip \
    jq \
    curl \
    wget \
    frr \
    net-tools
```

Verify:

```bash
iperf3 --version | head -1
# Expected: iperf 3.16 (cJSON 1.7.15)

ip -Version
# Expected: ip utility, iproute2-6.x.x

vtysh --version 2>/dev/null || echo "FRR vtysh not in PATH (check /usr/lib/frr)"
# Expected: vtysh version 9.x (FRR 9.x on Ubuntu 24.04)

tshark --version | head -1
# Expected: TShark (Wireshark) 4.x.x
```

### Containerlab

```bash
# Install Containerlab (topology emulation engine)
bash -c "$(curl -sL https://get.containerlab.dev)"
containerlab version
# Expected:
#                            _                   _       _
#                  _        (_)                 | |     | |
#  ____ ___  ____ | |_  ____ _ ____   ____  ____| | ____| | _
# /  _ \  / _  |/ _ \/ _  | |  _ \ / _  |/ ___|  |/ _  || |
#| | | | || (_| | |_) ) (_| | | | || (_| || |   | ( (_| || |
#|_| |_|_/ \__,_|\___/ \____|_| |_| \__,_||_|   |_|\__,_||_|
#
# version: 0.56.x
```

### Python scripting environment (uv)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

uv venv .venv
source .venv/bin/activate

uv pip install pandas matplotlib seaborn scapy
```

Verify:

```bash
python -c "import pandas, matplotlib, seaborn, scapy; print('All imports OK')"
# Expected: All imports OK

python -c "from scapy.all import IP, UDP, send; print('scapy OK')"
# Expected: scapy OK
```

---

## 27.2 Flowlet Switching

Flowlet switching is a software-defined adaptive routing technique that exploits natural "gaps" in TCP/RDMA flows to reroute them without causing reordering. A **flowlet** is a burst of packets from the same flow, separated from the next burst by an inter-packet gap exceeding the **FLOWLET_TIMEOUT**.

### The Core Insight

TCP flows are not continuous bit streams — they are bursts of packets separated by ACK-induced pauses, receiver window limits, and application pacing. If a flow has been silent for longer than the maximum path-delay difference (FLOWLET_TIMEOUT), any in-flight packets from the previous burst have already arrived at the destination. It is safe to switch to a different path for the next burst without causing reordering.

```
Flow timeline:
  ─────[burst1]──────         gap > FLOWLET_TIMEOUT        ─────[burst2]──────
                    └─ switch to new path here → no reordering risk ─┘
```

### FLOWLET_TIMEOUT Selection

The timeout must exceed the maximum path-delay difference across all equal-cost paths:

```
FLOWLET_TIMEOUT > max_path_delay_difference
                = (longest_path_RTT - shortest_path_RTT) / 2
```

In a two-tier (leaf-spine) DC fabric with commodity switches, path delay differences are typically 5-50 microseconds. Conservative deployments set FLOWLET_TIMEOUT to 500 microseconds; aggressive deployments use 50 microseconds.

### SONiC Flowlet Configuration

SONiC exposes flowlet configuration via the ConfigDB:

```bash
# Enable flowlet switching with 500-microsecond timeout
# (on a SONiC switch or SONiC-VS virtual switch)
redis-cli -n 4 HSET "DEVICE_METADATA|localhost" \
    "flowlet_switching_enabled" "true"

# Configure the flowlet timeout (in microseconds)
redis-cli -n 4 HSET "SWITCH_TABLE|switch" \
    "ecmp_hash_algorithm" "CRC" \
    "flowlet_timeout" "500"

# Verify via SONiC CLI
show switch-hash global
# Expected:
# SWITCH_HASH GLOBAL
# -------------------
# ECMP HASH ALGORITHM: CRC
# FLOWLET TIMEOUT: 500 (microseconds)
# FLOWLET SWITCHING: ENABLED

# Alternative: use config_db.json (for persistent configuration)
# Add to /etc/sonic/config_db.json:
# "SWITCH_TABLE": {
#     "switch": {
#         "ecmp_hash_algorithm": "CRC",
#         "flowlet_timeout": "500"
#     }
# }
```

---

## 27.3 NVIDIA/Mellanox Adaptive Routing

Adaptive Routing (AR) is a hardware feature in NVIDIA Spectrum-3/4 and ConnectX-7 ASICs that enables **per-packet** path selection based on real-time queue depth at the switch output ports. Unlike flowlet switching, AR does not require an inter-packet gap — it can reroute individual packets mid-flow.

### Per-Packet Load Balancing Mechanics

On each packet arrival, the Spectrum ASIC:
1. Evaluates all equal-cost next hops for the destination
2. Queries the instantaneous queue depth (in cells) for each egress port
3. Selects the port with the least queue depth (or a random port, depending on the AR algorithm)
4. Forwards the packet immediately

This happens entirely in the switch pipeline at line rate, with no software involvement.

### AR Algorithm Options

| Algorithm | Description | Use Case |
|---|---|---|
| `random` | Random port selection per packet | Maximum entropy, simple implementation |
| `least-loaded` | Select the port with minimum queue depth | Best utilization under elephant flows |
| `least-loaded-with-hysteresis` | Add hysteresis to avoid oscillation | Balanced utilization with stability |

### AR Threshold

The AR threshold is a configurable queue depth (in cells) below which a port is considered "acceptable." Ports above the threshold are avoided. Setting the threshold too high defeats the purpose; too low causes all packets to land on the emptiest port, causing oscillation:

```bash
# Configure on Spectrum-3 via NVIDIA SAI CLI (on SONiC with Spectrum ASIC)
sonic-cfggen -d -v DEVICE_METADATA.localhost.switch_type
# Expected: tor (or leaf, or spine)

# Set AR parameters via SONiC config
sudo config adaptive-routing enable

# Verify in ConfigDB
redis-cli -n 4 HGET "SWITCH_TABLE|switch" "adaptive_routing_enabled"
# Expected: true
```

### RDMA/RoCE Interaction

AR with RoCEv2 requires the receiver NIC to handle out-of-order packet delivery. The ConnectX-7 NIC's Out-of-Order (OOO) completion engine:
- Buffers out-of-order packets in dedicated hardware queues
- Re-presents them to the completion path in original sequence order
- Works transparently with DCQCN congestion control (ECN marking is preserved)

The key constraint: OOO handling consumes NIC buffer space. For very deep queues or very large messages, the OOO buffer may overflow, causing packet drops. Mellanox recommends limiting per-QP inflight data to fit within the NIC's OOO buffer size (configurable via `mlxconfig`).

---

## 27.4 Packet Spraying for RDMA

Packet spraying is the most aggressive form of per-packet load balancing: every packet is independently routed across all available paths, regardless of 5-tuple or flow state.

### How It Works

The sender NIC hashes each packet to a different egress port using a per-packet counter or random seed rather than the 5-tuple. The receiver must handle significant out-of-order delivery.

### DCQCN Interaction

DCQCN (Data Center Quantized Congestion Notification) is the congestion control protocol for RoCEv2. It works by:
1. Switch marks packets with ECN when queue depth exceeds a threshold
2. Receiver sends CNP (Congestion Notification Packet) back to sender
3. Sender reduces its injection rate using a multiplicative decrease

With packet spraying, ECN marks arrive for packets that took different paths — some congested, some not. The DCQCN rate reduction reacts to the congested path's marks even though many packets are flowing on uncongested paths. This causes under-utilization of available bandwidth.

**Mitigation**: per-path CNP generation (available on ConnectX-7) allows DCQCN to track congestion state per egress path rather than per flow, preventing false rate reductions on uncongested paths.

### When to Use Packet Spraying vs Flowlet Switching

| Scenario | Recommendation |
|---|---|
| IB (InfiniBand) fabric | Packet spraying (IB handles reordering natively) |
| RoCEv2 with ConnectX-7 OOO | Packet spraying with per-path DCQCN |
| RoCEv2 with older NICs | Flowlet switching (avoid OOO) |
| Ethernet with non-RDMA TCP | Flowlet switching (TCP reordering is expensive) |
| Mixed RDMA/non-RDMA fabric | Flowlet switching (safest default) |

---

## 27.5 SRv6 for DC Traffic Engineering

Segment Routing over IPv6 (SRv6) encodes a sequence of forwarding instructions — **segments** — directly in the IPv6 extension header. Each segment is an IPv6 address that identifies both a node and an operation (function) to perform at that node.

### SRv6 SID Structure

An SRv6 SID is a 128-bit IPv6 address divided into three fields:

```
|<--- Locator (N bits) --->|<-- Function (F bits) -->|<-- Args (A bits) -->|
  identifies the node          operation to perform     additional context
  routable in the fabric

Example SID: 2001:db8:0100::/48 (locator)
             2001:db8:0100::1   (End behavior — process SRH)
             2001:db8:0100::2   (End.X behavior — forward to specific next-hop)
             2001:db8:0100::3   (End.DT4 — decapsulate + IPv4 FIB lookup)
```

### Key SRv6 Behaviors

| Behavior | Function | Use Case |
|---|---|---|
| `End` | Process the SRH and forward to next segment | Intermediate node traversal |
| `End.X` | Forward to a specific L3 next-hop and decrement SL | Force a specific egress link |
| `End.DT4` | Decap the outer SRH, lookup inner IPv4 dest in a VRF | VPN egress at PE node |
| `End.DT6` | Same as End.DT4 but for IPv6 payload | IPv6 VPN egress |

### FRR SRv6 Configuration

FRR (Free Range Routing) is an open-source IP routing suite for Linux that implements BGP, OSPF, IS-IS, SRv6, and other protocols as a collection of daemons managed by a unified configuration plane. `vtysh` is FRR's interactive CLI shell that provides a Cisco-IOS-style command interface for configuring and inspecting all FRR daemons simultaneously.

```
! FRR vtysh configuration for SRv6 on a spine router
!
! Step 1: Define the SRv6 locator (the IPv6 prefix allocated to this node)
segment-routing
 srv6
  locators
   locator MAIN
    prefix 2001:db8:1:1::/64 block-len 40 node-len 24 func-bits 16
   !
  !
 !
!

! Step 2: Enable SRv6 in BGP for VPN signaling
router bgp 65001
 !
 address-family ipv6 vpn
  import vpn
  export vpn
 !
!

! Step 3: Assign SRv6 SID to a VRF
vrf TRAINING
 sid vpn per-vrf export auto
 sid vpn export evpn-default-originate ipv4
!
```

```bash
# In vtysh — verify SRv6 locator was installed
vtysh -c "show segment-routing srv6 locator"
# Expected:
# Locator:
# Name                 ID   Prefix                   Status
# -------------------- ---- ------------------------ -------
# MAIN                    1 2001:db8:1:1::/64        Active

vtysh -c "show segment-routing srv6 locator MAIN detail"
# Expected:
# Name: MAIN
# Prefix: 2001:db8:1:1::/64
# Block-Len: 40, Node-Len: 24, Function-Len: 16, Argument-Len: 0
# Status: Active
# SID behaviors:
#   2001:db8:1:1::1     End
#   2001:db8:1:1::2     End.X    (to: 2001:db8:100::2/128)
#   2001:db8:1:1::3     End.DT4  (VRF: TRAINING)

# Verify SRv6 SID is programmed in the dataplane
ip -6 route show 2001:db8:1:1::/64
# Expected:
# 2001:db8:1:1::/64 encap seg6local action End dev lo proto kernel
#   metric 20
# 2001:db8:1:1::2/128 encap seg6local action End.X nh4 10.0.0.1 dev eth0
#   proto kernel metric 20
```

### SRH Insertion (Steering Traffic)

```bash
# Steer traffic over a specific path using iproute2 SRv6 encap
# Force traffic to 10.0.2.0/24 to traverse spine1 then spine2
ip route add 10.0.2.0/24 \
    encap seg6 mode encap \
    segs 2001:db8:spine1::1,2001:db8:spine2::1 \
    dev eth0

# Verify the route
ip route show 10.0.2.0/24
# Expected:
# 10.0.2.0/24 encap seg6 mode encap segs 1 [ 2001:db8:spine1::1 2001:db8:spine2::1 ] dev eth0 scope link

# Trace the path a packet takes with the SRH
traceroute -s 10.0.0.1 10.0.2.1
# Expected: shows the path through spine1 (2001:db8:spine1::1 hop) then spine2
```

---

## 27.6 UCMP: Unequal Cost Multi-Path

Standard ECMP assumes all next-hops have equal capacity. In real fabrics, links fail and are replaced by links of different speeds, or some spines are overloaded by collocated services. UCMP (Unequal Cost Multi-Path) distributes traffic proportionally to link capacity.

### BGP Link Bandwidth Community

FRR implements UCMP using the BGP Link Bandwidth extended community (RFC 7311). Each route advertisement carries a `bandwidth:<asn>:<bps>` community indicating the capacity of the advertising link. FRR uses this to weight its ECMP hash tables:

```
! FRR vtysh: advertise link bandwidth on spine1 (100G uplink)
! RFC 7311 encodes bandwidth as a 32-bit IEEE 754 float in bytes/sec.
! FRR route-map accepts 1-4294967295 (bps). Use proportional values:
! 100G = 4294967295 (max), 25G = 1073741823 (25% of max → 4:1 ratio preserved)
router bgp 65001
 neighbor 10.0.0.1 route-map SET-BW-100G out
!
route-map SET-BW-100G permit 10
 set extcommunity bandwidth 4294967295
!

! spine2 has a degraded 25G link (temporarily — e.g. LACP member down)
router bgp 65001
 neighbor 10.0.0.2 route-map SET-BW-25G out
!
route-map SET-BW-25G permit 10
 set extcommunity bandwidth 1073741823
!
```

### Enabling UCMP in FRR

```bash
# Enable UCMP in FRR BGP
vtysh -c "configure terminal" \
      -c "router bgp 65000" \
      -c "bgp bestpath as-path multipath-relax" \
      -c "exit" \
      -c "exit"

# More precise: enable link bandwidth UCMP in BGP
vtysh <<'EOF'
configure terminal
router bgp 65000
  bgp bestpath bandwidth default-weight-for-missing
  bgp bestpath bandwidth ignore-capability
exit
EOF

# Verify weighted ECMP in the RIB
vtysh -c "show ip route 10.2.0.0/24 json" | python3 -m json.tool
# Expected JSON excerpt:
# {
#   "10.2.0.0/24": [{
#     "nexthops": [
#       {
#         "ip": "10.0.0.1",    <- spine1 (100G)
#         "weight": 4,         <- 4x more traffic than spine2
#         "interface": "eth0"
#       },
#       {
#         "ip": "10.0.0.2",    <- spine2 (25G)
#         "weight": 1,
#         "interface": "eth1"
#       }
#     ]
#   }]
# }
```

---

## 27.7 Measuring and Diagnosing ECMP Imbalance

Before deploying adaptive routing, you need to quantify the imbalance. The key metric is the **imbalance ratio**: the ratio of the maximum per-port load to the ideal per-port load if traffic were perfectly balanced.

```
imbalance_ratio = max_port_bytes / mean_port_bytes

Perfect balance: imbalance_ratio = 1.0
Typical ECMP with elephant flows: imbalance_ratio = 1.5 - 3.0
Severe collision: imbalance_ratio > 5.0
```

### Per-Port Counter Collection

```bash
# Snapshot interface counters on a spine (works on Linux or in network namespace)
ip -s link show | grep -A 4 "^[0-9]" | grep -E "(^[0-9]|RX:|TX:)" | \
    awk '/^[0-9]/{iface=$2} /TX:/{getline; print iface, $1}' 
# Expected (on a host with multiple interfaces):
# eth0: 12345678901
# eth1: 8901234567
# eth2: 9012345678

# On a real spine with many downlinks, loop over interfaces:
for iface in $(ls /sys/class/net/ | grep -E "^(eth|swp)"); do
    TX=$(cat /sys/class/net/${iface}/statistics/tx_bytes 2>/dev/null || echo 0)
    echo "${iface} ${TX}"
done
# Expected:
# eth0 123456789012
# eth1 134567890123
# eth2 89012345678
# eth3 145678901234
```

---

## Lab Walkthrough 27 — ECMP Hash Collision Detection and Flowlet Switching

This lab quantifies ECMP imbalance, demonstrates hash collision using Scapy, visualizes the distribution of flows across spines, and shows how flowlet switching restores balance. The topology is a 2-spine / 2-leaf Clos fabric emulated with Containerlab and FRR.

Containerlab is a network topology emulation framework that instantiates routers, switches, and hosts as Docker containers connected by virtual Ethernet links defined in a YAML topology file; it supports FRR, SONiC, SR Linux, and other network OS images. Scapy is a Python packet-crafting library that allows programs to construct, send, receive, and decode arbitrary network packets at any protocol layer, making it ideal for controlled traffic experiments like the 5-tuple sweep in this lab.

**Prerequisite check:**

```bash
which containerlab iperf3 vtysh python3 wg
containerlab version | grep "version:"
# Expected: version: 0.56.x

python3 -c "from scapy.all import IP, UDP; print('scapy OK')"
# Expected: scapy OK

# Confirm Containerlab has access to FRR container image
docker pull frrouting/frr:9.1.0
# Expected: Status: Image is up to date for frrouting/frr:9.1.0
```

---

### Step 1 — Deploy 2-Spine / 2-Leaf Containerlab Topology

Save the following as `/tmp/ecmp-lab/topology.yaml`:

```yaml
name: ecmp-lab

topology:
  nodes:
    spine1:
      kind: linux
      image: frrouting/frr:9.1.0
      binds:
        - /tmp/ecmp-lab/frr/spine1:/etc/frr

    spine2:
      kind: linux
      image: frrouting/frr:9.1.0
      binds:
        - /tmp/ecmp-lab/frr/spine2:/etc/frr

    leaf1:
      kind: linux
      image: frrouting/frr:9.1.0
      binds:
        - /tmp/ecmp-lab/frr/leaf1:/etc/frr

    leaf2:
      kind: linux
      image: frrouting/frr:9.1.0
      binds:
        - /tmp/ecmp-lab/frr/leaf2:/etc/frr

    host-a:
      kind: linux
      image: nicolaka/netshoot

    host-b:
      kind: linux
      image: nicolaka/netshoot

  links:
    # Leaf1 uplinks to both spines
    - endpoints: ["leaf1:eth1", "spine1:eth1"]
    - endpoints: ["leaf1:eth2", "spine2:eth1"]
    # Leaf2 uplinks to both spines
    - endpoints: ["leaf2:eth1", "spine1:eth2"]
    - endpoints: ["leaf2:eth2", "spine2:eth2"]
    # Host connections to leaves
    - endpoints: ["host-a:eth1", "leaf1:eth3"]
    - endpoints: ["host-b:eth1", "leaf2:eth3"]
```

Create FRR configuration directories:

```bash
mkdir -p /tmp/ecmp-lab/frr/{spine1,spine2,leaf1,leaf2}

# spine1 FRR configuration
cat > /tmp/ecmp-lab/frr/spine1/frr.conf << 'EOF'
frr version 9.1
frr defaults datacenter
hostname spine1
log syslog informational
!
interface eth1
 ip address 10.1.1.1/30
 no shutdown
!
interface eth2
 ip address 10.1.2.1/30
 no shutdown
!
router bgp 65100
 bgp router-id 10.0.0.101
 bgp bestpath as-path multipath-relax
 !
 neighbor 10.1.1.2 remote-as 65001
 neighbor 10.1.1.2 description leaf1-uplink
 neighbor 10.1.2.2 remote-as 65002
 neighbor 10.1.2.2 description leaf2-uplink
 !
 address-family ipv4 unicast
  network 10.0.0.101/32
  neighbor 10.1.1.2 activate
  neighbor 10.1.2.2 activate
  maximum-paths 8
 exit-address-family
!
EOF

# spine2 FRR configuration
cat > /tmp/ecmp-lab/frr/spine2/frr.conf << 'EOF'
frr version 9.1
frr defaults datacenter
hostname spine2
log syslog informational
!
interface eth1
 ip address 10.2.1.1/30
 no shutdown
!
interface eth2
 ip address 10.2.2.1/30
 no shutdown
!
router bgp 65200
 bgp router-id 10.0.0.201
 bgp bestpath as-path multipath-relax
 !
 neighbor 10.2.1.2 remote-as 65001
 neighbor 10.2.1.2 description leaf1-uplink
 neighbor 10.2.2.2 remote-as 65002
 neighbor 10.2.2.2 description leaf2-uplink
 !
 address-family ipv4 unicast
  network 10.0.0.201/32
  neighbor 10.2.1.2 activate
  neighbor 10.2.2.2 activate
  maximum-paths 8
 exit-address-family
!
EOF

# leaf1 FRR configuration
cat > /tmp/ecmp-lab/frr/leaf1/frr.conf << 'EOF'
frr version 9.1
frr defaults datacenter
hostname leaf1
log syslog informational
!
interface eth1
 ip address 10.1.1.2/30
!
interface eth2
 ip address 10.2.1.2/30
!
interface eth3
 ip address 10.10.1.1/24
!
router bgp 65001
 bgp router-id 10.0.0.1
 bgp bestpath as-path multipath-relax
 !
 neighbor 10.1.1.1 remote-as 65100
 neighbor 10.1.1.1 description spine1
 neighbor 10.2.1.1 remote-as 65200
 neighbor 10.2.1.1 description spine2
 !
 address-family ipv4 unicast
  network 10.10.1.0/24
  neighbor 10.1.1.1 activate
  neighbor 10.2.1.1 activate
  maximum-paths 8
 exit-address-family
!
EOF

# leaf2 FRR configuration
cat > /tmp/ecmp-lab/frr/leaf2/frr.conf << 'EOF'
frr version 9.1
frr defaults datacenter
hostname leaf2
log syslog informational
!
interface eth1
 ip address 10.1.2.2/30
!
interface eth2
 ip address 10.2.2.2/30
!
interface eth3
 ip address 10.10.2.1/24
!
router bgp 65002
 bgp router-id 10.0.0.2
 bgp bestpath as-path multipath-relax
 !
 neighbor 10.1.2.1 remote-as 65100
 neighbor 10.1.2.1 description spine1
 neighbor 10.2.2.1 remote-as 65200
 neighbor 10.2.2.1 description spine2
 !
 address-family ipv4 unicast
  network 10.10.2.0/24
  neighbor 10.1.2.1 activate
  neighbor 10.2.2.1 activate
  maximum-paths 8
 exit-address-family
!
EOF

# Deploy the topology
cd /tmp/ecmp-lab
sudo containerlab deploy --topo topology.yaml
# Expected:
# INFO[0000] Containerlab v0.56.x started
# INFO[0000] Parsing & checking topology file: topology.yaml
# INFO[0001] Creating lab directory: /tmp/ecmp-lab/clab-ecmp-lab
# INFO[0002] Creating container: "spine1"
# INFO[0003] Creating container: "spine2"
# INFO[0004] Creating container: "leaf1"
# INFO[0005] Creating container: "leaf2"
# INFO[0006] Creating container: "host-a"
# INFO[0007] Creating container: "host-b"
# INFO[0015] Adding containerlab host entries to /etc/hosts
# +---+-------------------+--------------+------------------+-------+
# | # |       Name        | Container ID |      Image       | State |
# +---+-------------------+--------------+------------------+-------+
# | 1 | clab-ecmp-lab-spine1  | abc123def456 | frrouting/frr:9.1.0 | running |
# | 2 | clab-ecmp-lab-spine2  | ...          | frrouting/frr:9.1.0 | running |
# ...
```

---

### Step 2 — Configure ECMP BGP on Spines and Leaves

```bash
# Configure host-a with a default route via leaf1
docker exec clab-ecmp-lab-host-a ip addr add 10.10.1.10/24 dev eth1
docker exec clab-ecmp-lab-host-a ip route add default via 10.10.1.1

# Configure host-b with a default route via leaf2
docker exec clab-ecmp-lab-host-b ip addr add 10.10.2.10/24 dev eth1
docker exec clab-ecmp-lab-host-b ip route add default via 10.10.2.1

# Wait for BGP to converge (30 seconds on average)
sleep 30

# Verify BGP neighbors are established on leaf1
docker exec clab-ecmp-lab-leaf1 vtysh -c "show bgp summary"
# Expected:
# IPv4 Unicast Summary (VRF default):
# BGP router identifier 10.0.0.1, local AS number 65001 vrf-id 0
# BGP table version 6
# RIB entries 10, using 1920 bytes of memory
# Peers 2, using 1456 KiB of memory
#
# Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
# 10.1.1.1        4      65100        12        12        0    0    0  00:00:28            1
# 10.2.1.1        4      65200        12        12        0    0    0  00:00:28            1
#
# Total number of neighbors 2

# Verify ECMP routes on leaf1 — should see both spines as next-hops for 10.10.2.0/24
docker exec clab-ecmp-lab-leaf1 vtysh -c "show ip route 10.10.2.0/24"
# Expected:
# Routing entry for 10.10.2.0/24
#   Known via "bgp", distance 20, metric 0, best
#   Last update 00:00:30 ago
# * 10.1.1.1, via eth1, weight 1
# * 10.2.1.1, via eth2, weight 1
#   (two equal-cost next hops — ECMP active)

# Verify end-to-end reachability
docker exec clab-ecmp-lab-host-a ping -c 3 10.10.2.10
# Expected:
# PING 10.10.2.10 (10.10.2.10) 56(84) bytes of data.
# 64 bytes from 10.10.2.10: icmp_seq=1 ttl=61 time=1.23 ms
# 64 bytes from 10.10.2.10: icmp_seq=2 ttl=61 time=0.98 ms
# 64 bytes from 10.10.2.10: icmp_seq=3 ttl=61 time=1.05 ms
```

---

### Step 3 — Generate 4 Parallel iperf3 Flows, Collect Per-Interface Counters

```bash
# Start iperf3 server on host-b
docker exec -d clab-ecmp-lab-host-b iperf3 -s --logfile /tmp/iperf_server.log

# Snapshot baseline counters on spine1 interfaces (eth1, eth2)
for iface in eth1 eth2; do
    BYTES=$(docker exec clab-ecmp-lab-spine1 \
        cat /sys/class/net/${iface}/statistics/tx_bytes 2>/dev/null || echo 0)
    echo "spine1.${iface} baseline_tx_bytes: ${BYTES}"
done
# Expected:
# spine1.eth1 baseline_tx_bytes: 45231
# spine1.eth2 baseline_tx_bytes: 38912

# Same baseline for spine2
for iface in eth1 eth2; do
    BYTES=$(docker exec clab-ecmp-lab-spine2 \
        cat /sys/class/net/${iface}/statistics/tx_bytes 2>/dev/null || echo 0)
    echo "spine2.${iface} baseline_tx_bytes: ${BYTES}"
done

# Run 4 parallel iperf3 flows from host-a to host-b
# Each flow uses a different source port to vary the 5-tuple
docker exec clab-ecmp-lab-host-a \
    iperf3 -c 10.10.2.10 -t 15 -P 4 --cport 5001 2>&1 | tail -5 &

# After 15 seconds, collect final counters
sleep 16

for node in spine1 spine2; do
    for iface in eth1 eth2; do
        BYTES=$(docker exec clab-ecmp-lab-${node} \
            cat /sys/class/net/${iface}/statistics/tx_bytes 2>/dev/null || echo 0)
        echo "${node}.${iface} final_tx_bytes: ${BYTES}"
    done
done
# Expected (example showing imbalance):
# spine1.eth1 final_tx_bytes: 2345678901   <- heavy load
# spine1.eth2 final_tx_bytes: 2312345678
# spine2.eth1 final_tx_bytes: 156789012    <- light load (hash collision)
# spine2.eth2 final_tx_bytes: 198765432
```

---

### Step 4 — Write ecmp_balance.py: Imbalance Ratio Calculator

Save as `/tmp/ecmp-lab/ecmp_balance.py`:

```python
#!/usr/bin/env python3
"""
ecmp_balance.py — Collect per-interface byte counters from spine containers
and calculate the ECMP imbalance ratio.

Usage: python3 ecmp_balance.py [--interval 5] [--count 3]
"""
import subprocess
import time
import argparse
import sys

SPINES = ["clab-ecmp-lab-spine1", "clab-ecmp-lab-spine2"]
UPLINK_INTERFACES = {
    "clab-ecmp-lab-spine1": ["eth1", "eth2"],   # downlinks toward leaves
    "clab-ecmp-lab-spine2": ["eth1", "eth2"],
}


def get_tx_bytes(container: str, interface: str) -> int:
    """Read TX bytes from a container's sysfs interface counter."""
    path = f"/sys/class/net/{interface}/statistics/tx_bytes"
    result = subprocess.run(
        ["docker", "exec", container, "cat", path],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        return 0
    return int(result.stdout.strip())


def snapshot_counters() -> dict:
    """Snapshot tx_bytes for all spine interfaces."""
    snap = {}
    for spine in SPINES:
        for iface in UPLINK_INTERFACES[spine]:
            key = f"{spine}.{iface}"
            snap[key] = get_tx_bytes(spine, iface)
    return snap


def compute_imbalance(before: dict, after: dict) -> dict:
    """Calculate bytes transferred per interface in the interval."""
    delta = {k: after[k] - before[k] for k in before}
    values = list(delta.values())

    if not any(values):
        return {"delta": delta, "imbalance_ratio": 1.0, "max_port": None, "mean": 0}

    non_zero = [v for v in values if v > 0]
    if not non_zero:
        return {"delta": delta, "imbalance_ratio": 1.0, "max_port": None, "mean": 0}

    mean_val = sum(non_zero) / len(non_zero)
    max_val = max(non_zero)
    max_port = max(delta, key=delta.get)
    imbalance = max_val / mean_val if mean_val > 0 else 1.0

    return {
        "delta": delta,
        "imbalance_ratio": round(imbalance, 3),
        "max_port": max_port,
        "mean_bytes": int(mean_val),
        "max_bytes": int(max_val),
    }


def print_report(result: dict, interval: float) -> None:
    print(f"\n{'='*60}")
    print(f"ECMP Balance Report  (interval={interval}s)")
    print(f"{'='*60}")
    print(f"{'Interface':<35} {'TX bytes':>15}  {'Gbps':>8}")
    print(f"{'-'*35} {'-'*15}  {'-'*8}")
    for iface, delta in sorted(result["delta"].items()):
        gbps = (delta * 8) / (interval * 1e9)
        marker = " <-- MAX" if iface == result["max_port"] else ""
        print(f"{iface:<35} {delta:>15,}  {gbps:>8.3f}{marker}")
    print(f"\nImbalance ratio: {result['imbalance_ratio']:.3f}x")
    if result['imbalance_ratio'] > 1.5:
        print("WARNING: Significant ECMP imbalance detected (>1.5x)")
    elif result['imbalance_ratio'] > 1.2:
        print("NOTICE:  Mild imbalance (>1.2x), consider flowlet switching")
    else:
        print("OK:      Traffic is well-balanced across spines")


def main():
    parser = argparse.ArgumentParser(description="ECMP imbalance detector")
    parser.add_argument("--interval", type=float, default=5.0,
                        help="Measurement interval in seconds (default: 5)")
    parser.add_argument("--count", type=int, default=3,
                        help="Number of measurement cycles (default: 3)")
    args = parser.parse_args()

    print(f"Monitoring ECMP balance across {len(SPINES)} spines...")
    print(f"Interfaces monitored: {sum(len(v) for v in UPLINK_INTERFACES.values())}")

    for cycle in range(args.count):
        print(f"\n[Cycle {cycle+1}/{args.count}] Sampling for {args.interval}s...")
        before = snapshot_counters()
        time.sleep(args.interval)
        after = snapshot_counters()
        result = compute_imbalance(before, after)
        print_report(result, args.interval)


if __name__ == "__main__":
    main()
```

```bash
# Run during the iperf3 test (restart iperf3 first)
docker exec -d clab-ecmp-lab-host-a \
    iperf3 -c 10.10.2.10 -t 60 -P 4

python3 /tmp/ecmp-lab/ecmp_balance.py --interval 5 --count 3
# Expected output (with hash collision):
# Monitoring ECMP balance across 2 spines...
# Interfaces monitored: 4
#
# [Cycle 1/3] Sampling for 5.0s...
# ============================================================
# ECMP Balance Report  (interval=5.0s)
# ============================================================
# Interface                           TX bytes          Gbps
# ----------------------------------- ---------------  --------
# clab-ecmp-lab-spine1.eth1            3,456,789,012     5.530 <-- MAX
# clab-ecmp-lab-spine1.eth2            3,312,456,789     5.300
# clab-ecmp-lab-spine2.eth1              234,567,890     0.375
# clab-ecmp-lab-spine2.eth2              198,765,432     0.318
#
# Imbalance ratio: 3.012x
# WARNING: Significant ECMP imbalance detected (>1.5x)
```

---

### Step 5 — Scapy: Demonstrate Hash Collision with Identical 5-Tuples

```python
# /tmp/ecmp-lab/collision_demo.py
# Craft 100 packets with the SAME 5-tuple — they all land on the same spine
from scapy.all import IP, UDP, send, conf
import time

# Target: host-b at 10.10.2.10
# Fixed 5-tuple — every packet hashes identically
SRC_IP   = "10.10.1.10"
DST_IP   = "10.10.2.10"
SRC_PORT = 5001       # fixed source port
DST_PORT = 5001       # fixed destination port

print(f"Sending 100 UDP packets with fixed 5-tuple:")
print(f"  {SRC_IP}:{SRC_PORT} -> {DST_IP}:{DST_PORT}")
print("  Expected: ALL packets take the same path (one spine saturated)\n")

conf.verb = 0  # suppress output
pkts = [IP(src=SRC_IP, dst=DST_IP) / UDP(sport=SRC_PORT, dport=DST_PORT) / b"X" * 64
        for _ in range(100)]
send(pkts, iface="eth0")

print("100 packets sent. Check spine counters — only one spine should show increment.")
```

```bash
# Run the collision demo (needs root for raw socket send)
sudo python3 /tmp/ecmp-lab/collision_demo.py

# Snapshot counters immediately after — verify one spine incremented, other did not
for node in spine1 spine2; do
    TOTAL=0
    for iface in eth1 eth2; do
        B=$(docker exec clab-ecmp-lab-${node} \
            cat /sys/class/net/${iface}/statistics/rx_bytes 2>/dev/null || echo 0)
        TOTAL=$((TOTAL + B))
    done
    echo "${node} total_rx_bytes: ${TOTAL}"
done
# Expected: one spine shows significantly more RX bytes than the other
# spine1 total_rx_bytes: 456789012   <- all 100 packets landed here
# spine2 total_rx_bytes: 123456789   <- no change from collision packets
```

---

### Step 6 — Vary Source Port with Scapy to Demonstrate 5-Tuple Distribution

```python
# /tmp/ecmp-lab/distribution_demo.py
# Send packets with varying source ports and visualize ECMP distribution
import subprocess
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import numpy as np


def get_spine_rx(spine_name: str) -> int:
    total = 0
    for iface in ["eth1", "eth2"]:
        result = subprocess.run(
            ["docker", "exec", f"clab-ecmp-lab-{spine_name}",
             "cat", f"/sys/class/net/{iface}/statistics/rx_bytes"],
            capture_output=True, text=True
        )
        if result.returncode == 0:
            total += int(result.stdout.strip())
    return total


def run_scapy_sweep() -> dict:
    """Import here to allow running as root or with appropriate caps."""
    from scapy.all import IP, UDP, send, conf
    conf.verb = 0

    SRC_IP, DST_IP, DST_PORT = "10.10.1.10", "10.10.2.10", 5001
    PACKETS_PER_PORT = 20

    results = {}
    for src_port in range(5000, 5032):  # 32 different source ports
        before = {s: get_spine_rx(s) for s in ["spine1", "spine2"]}
        pkts = [IP(src=SRC_IP, dst=DST_IP) /
                UDP(sport=src_port, dport=DST_PORT) / b"X" * 64
                for _ in range(PACKETS_PER_PORT)]
        send(pkts, iface="eth0")
        after = {s: get_spine_rx(s) for s in ["spine1", "spine2"]}

        # Determine which spine received more bytes for this port
        delta = {s: after[s] - before[s] for s in before}
        winner = max(delta, key=delta.get)
        results[src_port] = winner
        print(f"  src_port={src_port}: -> {winner}")

    return results


print("Sweeping source ports 5000-5031, 20 packets each...")
results = run_scapy_sweep()

# Count how many source ports hashed to each spine
from collections import Counter
counts = Counter(results.values())
print(f"\nDistribution: spine1={counts['spine1']}, spine2={counts['spine2']}")
print(f"Balance ratio: {max(counts.values()) / min(counts.values()):.2f}x")

# Plot the distribution
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))

# Bar chart: per-port spine assignment
ports = sorted(results.keys())
colors = ['steelblue' if results[p] == 'spine1' else 'coral' for p in ports]
ax1.bar(range(len(ports)), [1] * len(ports), color=colors, edgecolor='white')
ax1.set_xticks(range(0, len(ports), 4))
ax1.set_xticklabels([str(p) for p in ports[::4]], rotation=45, fontsize=8)
ax1.set_xlabel("Source Port")
ax1.set_ylabel("Path Assignment")
ax1.set_title("ECMP Path Selection by Source Port")
spine1_patch = mpatches.Patch(color='steelblue', label='spine1')
spine2_patch = mpatches.Patch(color='coral', label='spine2')
ax1.legend(handles=[spine1_patch, spine2_patch])

# Pie chart: overall distribution
ax2.pie(
    [counts.get('spine1', 0), counts.get('spine2', 0)],
    labels=['spine1', 'spine2'],
    colors=['steelblue', 'coral'],
    autopct='%1.1f%%',
    startangle=90
)
ax2.set_title("Overall Distribution (32 source ports)")

plt.tight_layout()
plt.savefig('/tmp/ecmp-lab/ecmp_distribution.png', dpi=150)
print("\nDistribution plot saved to /tmp/ecmp-lab/ecmp_distribution.png")
```

```bash
sudo python3 /tmp/ecmp-lab/distribution_demo.py
# Expected output:
# Sweeping source ports 5000-5031, 20 packets each...
#   src_port=5000: -> spine1
#   src_port=5001: -> spine1
#   src_port=5002: -> spine2
#   src_port=5003: -> spine1
#   src_port=5004: -> spine2
#   ...
#
# Distribution: spine1=18, spine2=14
# Balance ratio: 1.29x
#
# Distribution plot saved to /tmp/ecmp-lab/ecmp_distribution.png
#
# Note: with only 32 source ports the distribution is imperfect
# (the CRC hash is deterministic — some biases are expected at small sample sizes)
```

---

### Step 7 — Enable Flowlet Switching in SONiC-VS, Re-run Experiment

Flowlet switching is a SONiC/SAI feature. On the Containerlab FRR nodes we simulate the effect by demonstrating what happens when traffic is broken into smaller bursts with inter-burst gaps exceeding the FLOWLET_TIMEOUT. On a real SONiC-VS instance the commands are:

```bash
# On a SONiC-VS node (if using sonic-vs image in containerlab):
# Enable flowlet switching with 500us timeout
docker exec clab-ecmp-lab-spine1 \
    redis-cli -n 4 HSET "SWITCH_TABLE|switch" \
        "ecmp_hash_algorithm" "CRC" \
        "flowlet_timeout" "500"

# Verify
docker exec clab-ecmp-lab-spine1 \
    redis-cli -n 4 HGET "SWITCH_TABLE|switch" "flowlet_timeout"
# Expected: 500

# With FRR nodes, simulate flowlet behavior by adding inter-burst gaps
# Re-run iperf3 with short bursts and burst intervals > FLOWLET_TIMEOUT
docker exec clab-ecmp-lab-host-a \
    bash -c 'for i in 1 2 3 4; do
        iperf3 -c 10.10.2.10 -t 3 -P 4 --cport $((5000+i)) 2>&1 | tail -2
        sleep 0.001  # 1ms gap > 500us FLOWLET_TIMEOUT
    done'
# Expected: each burst may take a different path, improving distribution

# Re-measure imbalance
python3 /tmp/ecmp-lab/ecmp_balance.py --interval 5 --count 2
# Expected (improved balance with flowlet switching active):
# ============================================================
# ECMP Balance Report  (interval=5.0s)
# ============================================================
# Interface                           TX bytes          Gbps
# ----------------------------------- ---------------  --------
# clab-ecmp-lab-spine1.eth1            1,856,234,567     2.970 <-- MAX
# clab-ecmp-lab-spine1.eth2            1,745,678,901     2.793
# clab-ecmp-lab-spine2.eth1            1,623,456,789     2.597
# clab-ecmp-lab-spine2.eth2            1,789,012,345     2.862
#
# Imbalance ratio: 1.089x
# OK:      Traffic is well-balanced across spines
```

---

### Step 8 — SRv6 Basic Configuration in FRR

```bash
# Enable the segment routing daemon in FRR on leaf1
docker exec clab-ecmp-lab-leaf1 \
    sed -i 's/^pathd=no/pathd=yes/; s/^bgpd=no/bgpd=yes/' /etc/frr/daemons

# Restart FRR to activate pathd
docker exec clab-ecmp-lab-leaf1 supervisorctl restart frr

# Configure SRv6 locator via vtysh
docker exec -it clab-ecmp-lab-leaf1 vtysh << 'VTYSH_EOF'
configure terminal
!
! Assign an SRv6 locator to this leaf node
segment-routing
 srv6
  locators
   locator MAIN
    prefix 2001:db8:1:1::/64 block-len 40 node-len 24 func-bits 16
   !
  !
 !
!
! Enable IPv6 on uplinks for SRv6 to work
interface eth1
 ipv6 address 2001:db8:100:1::2/64
!
interface eth2
 ipv6 address 2001:db8:200:1::2/64
!
end
write memory
VTYSH_EOF

# Verify SRv6 locator was installed
docker exec clab-ecmp-lab-leaf1 \
    vtysh -c "show segment-routing srv6 locator"
# Expected:
# Locator:
# Name                 ID   Prefix                   Status
# -------------------- ---- ------------------------ -------
# MAIN                    1 2001:db8:1:1::/64        Active

docker exec clab-ecmp-lab-leaf1 \
    vtysh -c "show segment-routing srv6 locator MAIN detail"
# Expected:
# Name: MAIN
# Prefix: 2001:db8:1:1::/64
# Block-Len: 40, Node-Len: 24, Function-Len: 16, Argument-Len: 0
# Status: Active
# SID behaviors:
#   2001:db8:1:1::1     End

# Verify the SRv6 SID is in the Linux routing table
docker exec clab-ecmp-lab-leaf1 \
    ip -6 route show 2001:db8:1:1::/64
# Expected:
# 2001:db8:1:1::/64 encap seg6local action End dev lo proto kernel metric 20 pref medium
```

---

### Step 9 — UCMP with BGP Link Bandwidth Community

```bash
# Configure leaf1 to advertise link-bandwidth community toward spine1
# (simulating that spine1 has a 100G uplink and spine2 has a degraded 25G link)
docker exec -it clab-ecmp-lab-leaf1 vtysh << 'VTYSH_EOF'
configure terminal
!
route-map SET-BW-HIGH permit 10
 set extcommunity bandwidth 4294967295
!
route-map SET-BW-LOW permit 10
 set extcommunity bandwidth 1073741823
!
router bgp 65001
 neighbor 10.1.1.1 route-map SET-BW-HIGH out
 neighbor 10.2.1.1 route-map SET-BW-LOW out
 !
 bgp bestpath bandwidth default-weight-for-missing
!
end
write memory
VTYSH_EOF

# Clear BGP sessions to force re-advertisement with new communities
docker exec clab-ecmp-lab-leaf1 \
    vtysh -c "clear bgp * soft out"

# Wait for BGP to reconverge
sleep 10

# On spine1, verify it received the bandwidth community from leaf1
docker exec clab-ecmp-lab-spine1 \
    vtysh -c "show bgp ipv4 unicast 10.10.1.0/24 detail"
# Expected output excerpt:
# BGP routing table entry for 10.10.1.0/24, version 8
#   Paths: (1 available, best #1)
#   ...
#   Extended Community: LB:65001:4294967295
#   ...

# Verify weighted ECMP in the FIB on a transit node
# (the weight annotation appears in 'show ip route' JSON output)
docker exec clab-ecmp-lab-spine1 \
    vtysh -c "show ip route 10.10.1.0/24 json" | \
    python3 -c "
import json, sys
data = json.load(sys.stdin)
for prefix, routes in data.items():
    for route in routes:
        for nh in route.get('nexthops', []):
            print(f\"  via {nh.get('ip','?')}: weight={nh.get('weight','N/A')}\")
"
# Expected:
#   via 10.1.1.2: weight=4
#   via 10.2.1.2: weight=1
# (4:1 ratio reflects 100G vs 25G, i.e., 4x more traffic to spine1's leaf1 path)
```

---

### Step 10 — Cleanup

```bash
# Stop all background iperf3 servers
docker exec clab-ecmp-lab-host-b pkill iperf3 2>/dev/null || true

# Destroy the Containerlab topology
cd /tmp/ecmp-lab
sudo containerlab destroy --topo topology.yaml --cleanup
# Expected:
# INFO[0000] Parsing & checking topology file: topology.yaml
# INFO[0000] Destroying lab: ecmp-lab
# INFO[0002] Removed container: clab-ecmp-lab-spine1
# INFO[0003] Removed container: clab-ecmp-lab-spine2
# INFO[0004] Removed container: clab-ecmp-lab-leaf1
# INFO[0005] Removed container: clab-ecmp-lab-leaf2
# INFO[0006] Removed container: clab-ecmp-lab-host-a
# INFO[0007] Removed container: clab-ecmp-lab-host-b
# INFO[0008] Removing containerlab host entries from /etc/hosts

# Verify no containers remain
docker ps --filter "label=containerlab=ecmp-lab"
# Expected: (empty table)

# Remove lab files
rm -rf /tmp/ecmp-lab

echo "Cleanup complete."
```

---

## Summary

- ECMP's 5-tuple hash assigns all packets from a given flow to the same path; for AllReduce traffic where flows are few, large, and long-lived, a single hash collision can create sustained 2-3x load imbalance between spines.
- Flowlet switching exploits natural inter-burst gaps in TCP/RDMA flows to reroute flowlets independently, restoring balance without per-packet reordering risk; the FLOWLET_TIMEOUT must exceed the maximum inter-path delay difference.
- NVIDIA Spectrum-3/4 adaptive routing performs per-packet path selection based on real-time egress queue depth, eliminating elephant flow concentration at the cost of requiring ConnectX-7 OOO handling at the receiver.
- Packet spraying routes each packet independently across all equal-cost paths — the most aggressive form of load balancing — but requires receiver-side reordering and careful DCQCN per-path congestion tracking to avoid spurious rate reductions.
- SRv6 encodes a sequence of forwarding instructions in the IPv6 extension header, enabling explicit traffic engineering paths (`End.X` for specific next-hop selection) with a standards-track protocol that is now natively supported in FRR 9.x and the Linux kernel.
- UCMP with BGP Link Bandwidth extended community distributes traffic proportional to link capacity, critical during incremental upgrades or partial link failures where paths have different available bandwidth.
- The imbalance ratio (`max_port_bytes / mean_port_bytes`) is the primary diagnostic metric for ECMP health; values above 1.5x indicate problematic hash skew; the `ecmp_balance.py` tool in this chapter provides a lightweight, container-native measurement framework.

---

## References

- Dixit, A. et al., "On the Impact of Packet Spraying in Data Center Networks", IEEE INFOCOM 2013 — [ieeexplore.ieee.org/document/6566899](https://ieeexplore.ieee.org/document/6566899)
- Alizadeh, M. et al., "CONGA: Distributed Congestion-Aware Load Balancing for Data Centers", ACM SIGCOMM 2014 — [dl.acm.org/doi/10.1145/2619239.2626316](https://dl.acm.org/doi/10.1145/2619239.2626316)
- Katta, N. et al., "Clove: Congestion-aware Load Balancing at the Virtual Edge", ACM CoNEXT 2017 — [dl.acm.org/doi/10.1145/3143361.3143373](https://dl.acm.org/doi/10.1145/3143361.3143373)
- [NVIDIA Spectrum Adaptive Routing](https://docs.nvidia.com/networking/display/SONiCAdaptiveRouting)
- [Flowlet switching in SONiC](https://github.com/sonic-net/sonic-swss/blob/master/doc/FlowletSwitching.md)
- [RFC 8986 — SRv6 Network Programming](https://datatracker.ietf.org/doc/html/rfc8986) — Filsfils et al., IETF 2021
- [RFC 8754 — IPv6 Segment Routing Header (SRH)](https://datatracker.ietf.org/doc/html/rfc8754)
- [RFC 7311 — BGP Link Bandwidth extended community](https://datatracker.ietf.org/doc/html/rfc7311) — Mohapatra et al., IETF 2014
- [FRR SRv6 documentation](https://docs.frrouting.org/en/latest/srmpls.html#srv6)
- [FRRouting (FRR)](https://frrouting.org)
- [SONiC (Software for Open Networking in the Cloud)](https://sonic-net.github.io/SONiC/)
- [Containerlab documentation](https://containerlab.dev/manual/topo-def-file/)
- [Scapy — Python packet crafting library](https://scapy.readthedocs.io/en/latest/)
- Zhu, Y. et al., "Congestion Control for Large-Scale RDMA Deployments" (DCQCN), ACM SIGCOMM 2015 — [dl.acm.org/doi/10.1145/2829988.2787510](https://dl.acm.org/doi/10.1145/2829988.2787510)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).