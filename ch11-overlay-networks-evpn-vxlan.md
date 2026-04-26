# Chapter 11 — Overlay Networks: BGP-EVPN, VXLAN, OVS & OVN

**Part IV: Overlay & Kubernetes Networking** | ~20 pages

---

## Introduction

AI compute clusters run two fundamentally different classes of network traffic simultaneously. The first is the GPU-to-GPU collective communication fabric — low-latency, high-bandwidth RDMA flows that must cross the physical network without encapsulation overhead. The second is the management and multi-tenant fabric — container orchestration, storage access, monitoring pipelines, and tenant isolation — where the ability to define isolated Layer 2 domains across a shared routed underlay is essential. This chapter addresses the second class.

VXLAN (Virtual Extensible LAN) and BGP-EVPN (Ethernet VPN) are the industry-standard answer for building scalable overlay networks on top of a pure IP underlay. VXLAN extends Ethernet frames across the IP fabric using UDP encapsulation with 24-bit segment identifiers, providing 16 million logical isolation domains in place of the 4094 VLAN limit. BGP-EVPN eliminates the flooding required to learn remote MAC and IP addresses by distributing that reachability information as BGP routes, enabling data centers with thousands of endpoints to operate without BUM (Broadcast, Unknown-unicast, Multicast) flooding at scale.

Open vSwitch (OVS) and Open Virtual Network (OVN) bring this overlay model into the host: OVS is a production-quality software switch with kernel and DPDK fast paths that implements VXLAN tunneling and an OpenFlow-programmable forwarding pipeline, while OVN adds a logical abstraction layer — logical switches, routers, and ACLs — that OVN compiles down to per-host OVS flow rules. Together they form the datapath that Kubernetes CNIs such as OVN-Kubernetes rely on to implement pod networking.

The reader will leave this chapter able to build a BGP-EVPN overlay fabric from first principles, inspect and program the OVS OpenFlow pipeline, and configure OVN logical networks. The lab walkthrough constructs a four-leaf, two-spine EVPN fabric in Containerlab with SONiC-VS leaves and SR Linux spines, and traces a live VXLAN-encapsulated packet from source to destination.

This chapter sits within Part IV alongside Chapters 12 and 13. Chapter 12 builds on the overlay model by showing how Cilium uses eBPF to replace the OVS/iptables datapath entirely. Chapter 13 extends the Kubernetes pod networking model to support multiple NICs per pod — the prerequisite for attaching a pod to both the overlay management network and the RDMA data-plane network simultaneously.

---

---

## Installation

The chapter requires iproute2 and bridge-utils to create and inspect VXLAN interfaces, bridge FDB entries, and VTEP state directly from the Linux kernel. Open vSwitch (ovs-vsctl, ovs-ofctl) provides the software switch datapath and the OpenFlow pipeline that OVN compiles logical network definitions into. FRR (Free Range Routing), specifically the bgpd and zebra daemons, runs the BGP-EVPN control plane that distributes MAC/IP reachability across VTEPs, eliminating the flooding that would otherwise be required. Containerlab is used to wire together the four-leaf, two-spine lab topology with SR Linux spines and SONiC-VS leaves, giving a realistic fabric to exercise the full EVPN-VXLAN stack.

### System packages (Ubuntu 24.04)

```bash
sudo apt update
sudo apt install -y iproute2 bridge-utils tcpdump vlan net-tools
```

### Containerlab

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
containerlab version
```

### FRR (for host containers and bare-metal testing)

FRR (Free Range Routing) is an open-source IP routing suite that implements BGP, OSPF, IS-IS, and other protocols. It is used here to run BGP-EVPN on Linux hosts and leaf switches.

```bash
sudo apt install -y frr frr-pythontools
# Enable EVPN-related daemons
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo sed -i 's/^zebra=no/zebra=yes/' /etc/frr/daemons
sudo systemctl restart frr
```

### CMake (required if building OVS or iproute2 from source)

```bash
sudo apt install -y cmake build-essential pkg-config
cmake --version
```

### Python environment (device automation)

`netmiko` is a Python library for SSH-based network device automation; `ncclient` is a Python client for the NETCONF protocol (covered in depth in Chapter 14).

```bash
uv venv .venv && source .venv/bin/activate
uv pip install netmiko ncclient
python -c "import netmiko, ncclient; print('OK')"
```

---

## 11.1 Why Overlays in AI Clusters?

AI clusters have at least two distinct networking concerns running simultaneously:

1. **Training fabric:** Low-latency, high-bandwidth RDMA traffic between GPU NICs. No encapsulation overhead acceptable. Addressed in Parts I–III.
2. **Management and multi-tenant fabric:** Container orchestration, storage access, monitoring, and multi-tenant isolation. Here, IP overlays (VXLAN + EVPN) provide scalable L2 extension and L3 routing across a routed underlay.

VXLAN and EVPN are the standard answer for the second concern. They allow the cluster operator to present each tenant with an isolated L2 domain and L3 routing environment while the physical fabric runs pure IP/BGP.

---

## 11.2 VXLAN — Virtual Extensible LAN

### 11.2.1 Encapsulation Format

VXLAN (RFC 7348) wraps an inner Ethernet frame inside an outer UDP/IP packet:

```
Outer Ethernet header (14 bytes)
Outer IPv4/IPv6 header (20/40 bytes)
Outer UDP header     (8 bytes)   dst port = 4789
VXLAN header         (8 bytes)   [I flag | 24-bit VNI]
Inner Ethernet frame (original packet)
```

The **VNI (VXLAN Network Identifier)** is a 24-bit segment ID (16M possible segments), replacing the 12-bit VLAN tag.

### 11.2.2 VTEP Model

VXLAN tunnel endpoints (VTEPs) originate and terminate tunnels:

```
Host A (VTEP 10.0.0.1)                Host B (VTEP 10.0.0.2)
┌──────────────────┐                   ┌──────────────────┐
│ Container (VNI 100)│                 │ Container (VNI 100)│
│  IP: 192.168.1.1  │                 │  IP: 192.168.1.2  │
└──────┬────────────┘                 └────────┬──────────┘
       │ inner frame                           │
   [Linux VXLAN interface]               [Linux VXLAN interface]
       │ encap in UDP/4789                     │
   [Physical NIC] ─── IP fabric ─── [Physical NIC]
```

### 11.2.3 Linux VXLAN Interface

```bash
# Create VXLAN interface for VNI 100
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    local 10.0.0.1 \
    dev eth0

ip link set vxlan100 up

# Bridge with a local container's veth
ip link add veth0 type veth peer name veth0_peer
ip link add br0 type bridge
ip link set br0 up
ip link set vxlan100 master br0
ip link set veth0 master br0
ip link set veth0_peer netns container_ns

# BUM (Broadcast/Unknown/Multicast) traffic handling options:
# 1. Multicast group (traditional)
ip link add vxlan100 type vxlan id 100 group 239.1.1.1 dev eth0

# 2. Ingress replication (head-end replication, used with EVPN)
# — no multicast needed; known VTEPs are listed explicitly
bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 10.0.0.2
bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 10.0.0.3
```

---

## 11.3 BGP-EVPN — Control Plane for VXLAN

Without a control plane, VTEPs learn remote MAC/IP mappings via flooding (BUM traffic). This doesn't scale. EVPN (RFC 7432, RFC 8365) distributes MAC/IP reachability via BGP, eliminating flooding at scale.

### 11.3.1 EVPN Route Types

| Type | Name | Purpose |
|---|---|---|
| Type 1 | Ethernet Auto-Discovery | Multi-homing (fast convergence) |
| Type 2 | MAC/IP Advertisement | MAC learning via BGP instead of flooding |
| Type 3 | Inclusive Multicast Ethernet Tag | BUM traffic handling; VTEP discovery |
| Type 4 | Ethernet Segment | Designated forwarder election |
| Type 5 | IP Prefix | L3 routing across VNIs (symmetric IRB) |

### 11.3.2 Type 2 Route — MAC/IP Advertisement

When a host sends its first packet, the local VTEP learns its MAC/IP and advertises a Type 2 NLRI:

```
BGP EVPN NLRI Type 2:
  Route Distinguisher: 10.0.0.1:100
  ESI: 00:00:00:00:00:00:00:00:00:00
  Ethernet Tag: 0
  MAC length: 48
  MAC: aa:bb:cc:dd:ee:ff
  IP length: 32
  IP: 192.168.1.1
  MPLS Label1: VNI 100 (encoded as MPLS label)
  Extended communities:
    RT: 65001:100   (import/export control)
    Encap: VXLAN
    Router MAC: 00:11:22:33:44:55
```

### 11.3.3 Symmetric IRB — Inter-VNI Routing

Asymmetric IRB routes between VNIs at the ingress VTEP. **Symmetric IRB** (the modern approach) uses a dedicated L3 VNI: both ingress and egress VTEPs participate in routing, enabling ECMP and proper TTL behavior.

```bash
# FRR configuration for symmetric IRB
router bgp 65001
 address-family l2vpn evpn
  advertise-all-vni
  advertise ipv4 unicast    # redistribute connected routes as Type 5
  vni 10100
   rd 10.0.0.1:100
   route-target import 65001:100
   route-target export 65001:100
  exit-vni
  vni 99999                    # L3 VNI for inter-subnet routing
   rd 10.0.0.1:99999
   route-target import 65001:99999
   route-target export 65001:99999
  exit-vni
 exit-address-family
!
```

---

## 11.4 Open vSwitch (OVS)

OVS is a production-quality software switch implementing IEEE 802.1Q VLANs, LACP, BFD, VXLAN, and OpenFlow. It is the datapath for OVN and is used by Kubernetes CNIs (Calico OVS mode, OVN-Kubernetes).

### 11.4.1 OVS Architecture

```
ovs-vsctl / ovs-ofctl (management)
           ↕
     ovsdb-server (OVSDB — config + state database)
           ↕
      ovs-vswitchd (daemon with in-memory flow table)
           ↕
    kernel datapath (fast path via openvswitch.ko)
    or DPDK datapath (with --with-dpdk)
```

### 11.4.2 Basic OVS Configuration

```bash
# Create bridge and add ports
ovs-vsctl add-br br0
ovs-vsctl add-port br0 eth1
ovs-vsctl add-port br0 vxlan0 -- set Interface vxlan0 \
    type=vxlan options:remote_ip=10.0.0.2 options:key=100

# Inspect bridges and ports
ovs-vsctl show

# Show flow table
ovs-ofctl dump-flows br0

# Add an OpenFlow rule: drop ARP from a specific MAC
ovs-ofctl add-flow br0 \
    "priority=100,arp,dl_src=aa:bb:cc:dd:ee:ff,action=drop"

# Statistics per flow
ovs-ofctl dump-flows br0 --no-stats | head -20
```

### 11.4.3 VXLAN with EVPN via FRR + OVS

```bash
# OVS: create VXLAN interface (FRR will populate FDB via BGP-learned MACs)
ovs-vsctl add-port br-vxlan vxlan100 -- set Interface vxlan100 \
    type=vxlan options:remote_ip="flow" options:key="flow" \
    options:dst_port=4789

# FRR EVPN populates bridge FDB with learned remote MACs:
# bridge fdb show dev vxlan100
# aa:bb:cc:dd:ee:ff dst 10.0.0.2 self permanent  ← installed by FRR
```

### 11.4.4 OpenFlow Programmability in OVS

OVS's control plane is OpenFlow. Every packet forwarding decision — VXLAN encapsulation, EVPN-learned MAC forwarding, security group ACLs — is expressed as entries in an OpenFlow multi-table pipeline that `ovs-vswitchd` executes in the kernel datapath.

**Pipeline overview**

OVS uses a multi-table pipeline by default. Tables are numbered 0–254; packets start at table 0 and are redirected with `goto_table` actions:

```
Table 0  — ingress classification (port, VLAN tag, tunnel key)
Table 10 — VXLAN/tunnel decap and source MAC learning
Table 20 — L2 lookup (MAC → output port)
Table 30 — egress ACL / security groups
Table 90 — normal forwarding / output
```

Inspect the pipeline with `ovs-ofctl`:

```bash
# Show all flow entries across all tables
ovs-ofctl -O OpenFlow13 dump-flows br-vxlan

# Example: tunnel decap rule (table 10)
# cookie=0x0, table=10, priority=100,tun_id=0x64,dl_dst=aa:bb:cc:dd:ee:ff
#   actions=set_field:3->metadata,output:LOCAL

# Show the pipeline tables and their capacities
ovs-ofctl -O OpenFlow13 dump-tables br-vxlan
```

**Installing a flow with `ovs-ofctl`**

```bash
# Table 20: forward frames destined for aa:bb:cc:dd:ee:ff out port vxlan100
ovs-ofctl -O OpenFlow13 add-flow br-vxlan \
  "table=20,priority=100,dl_dst=aa:bb:cc:dd:ee:ff \
   actions=set_field:100->tun_id,output:vxlan100"

# Table 20: unknown unicast — flood to all VXLAN ports
ovs-ofctl -O OpenFlow13 add-flow br-vxlan \
  "table=20,priority=1 \
   actions=output:vxlan100,output:vxlan200"

# Table 30: drop traffic from untrusted source MAC
ovs-ofctl -O OpenFlow13 add-flow br-vxlan \
  "table=30,priority=200,dl_src=de:ad:be:ef:00:01 \
   actions=drop"
```

**Group tables for multicast/ECMP**

OpenFlow 1.3 group tables allow a single action to fan out to multiple ports or choose among them:

```bash
# Create a SELECT group for ECMP across two VTEP paths
ovs-ofctl -O OpenFlow13 add-group br-vxlan \
  "group_id=1,type=select,\
   bucket=weight:50,actions=set_field:10.0.0.2->tun_dst,output:vxlan100,\
   bucket=weight:50,actions=set_field:10.0.0.3->tun_dst,output:vxlan200"

# Reference the group from a flow
ovs-ofctl -O OpenFlow13 add-flow br-vxlan \
  "table=20,priority=100,dl_dst=ff:ff:ff:ff:ff:ff \
   actions=group:1"

# Inspect groups
ovs-ofctl -O OpenFlow13 dump-groups br-vxlan
```

**How OVN generates OpenFlow**

When OVN is deployed (see §11.5), `ovn-controller` compiles logical network policies (logical switches, routers, ACLs, NAT rules) into OVS OpenFlow rules automatically. Administrators define intent at the logical layer; `ovn-controller` translates it to the physical flow table. Direct `ovs-ofctl` inspection remains the primary debugging interface:

```bash
# Trace how a packet from port a→b is processed
ovs-appctl ofproto/trace br-vxlan \
  in_port=1,dl_src=aa:bb:cc:00:00:01,dl_dst=aa:bb:cc:00:00:02,\
  dl_type=0x0800,nw_src=192.168.1.1,nw_dst=192.168.1.2

# Output shows each table, matched rule, and resulting actions:
# Flow: in_port=1,...
# Rule: table=0 priority=100,in_port=1 actions=goto_table:10
# Rule: table=10 priority=100 actions=goto_table:20
# Rule: table=20 priority=100,dl_dst=aa:bb:cc:00:00:02 actions=output:vxlan100
# Final flow: ...
```

This trace-driven debugging workflow — define logical intent in OVN or FRR EVPN, then verify the concrete OpenFlow rules with `ovs-ofctl dump-flows` and `ofproto/trace` — is the standard operational model for OVS-based overlay fabrics.

---

## 11.5 OVN — Open Virtual Network

OVN is a logical network abstraction layer built on top of OVS. While OVS operates at the flow level (match bits → action), OVN operates at the intent level (logical switches, routers, ACLs) and compiles that intent into OVS OpenFlow rules.

### 11.5.1 OVN Architecture

```
CMS (Kubernetes, OpenStack)
        │ northbound API
   ovn-northd
        │ translates logical → physical
   OVN Southbound DB
     (flow DB, chassis registry)
        │
   ovn-controller (per host)
        │ programs
   ovs-vswitchd (per host)
```

### 11.5.2 Logical Network Configuration

```bash
# Create a logical switch
ovn-nbctl ls-add ls-ai-cluster

# Add logical switch ports
ovn-nbctl lsp-add ls-ai-cluster lsp-gpu0
ovn-nbctl lsp-set-addresses lsp-gpu0 "aa:bb:cc:dd:ee:01 192.168.100.1"

ovn-nbctl lsp-add ls-ai-cluster lsp-gpu1
ovn-nbctl lsp-set-addresses lsp-gpu1 "aa:bb:cc:dd:ee:02 192.168.100.2"

# Create a logical router
ovn-nbctl lr-add lr-fabric
ovn-nbctl lrp-add lr-fabric lrp-to-ls 00:11:22:33:44:55 192.168.100.254/24

# Connect logical switch to router
ovn-nbctl lsp-add ls-ai-cluster lsp-to-lr
ovn-nbctl lsp-set-type lsp-to-lr router
ovn-nbctl lsp-set-addresses lsp-to-lr router
ovn-nbctl lsp-set-options lsp-to-lr router-port=lrp-to-ls

# ACL: allow only known hosts
ovn-nbctl acl-add ls-ai-cluster to-lport 100 \
    "ip4.src == 192.168.100.0/24" allow-related
ovn-nbctl acl-add ls-ai-cluster to-lport 50 "ip4" drop
```

OVN's `ovn-controller` on each host translates these logical definitions into OVS flows, handling tunneling (Geneve or VXLAN) transparently.

---

## Lab Walkthrough 11 — EVPN Fabric with SONiC-VS and FRR

This walkthrough builds a full BGP-EVPN overlay fabric using Containerlab with SR Linux spines, SONiC-VS leaves, and Linux host containers running FRR. SR Linux is Nokia's open, model-driven network operating system available as a free container image for lab use. SONiC-VS (Software for Open Networking in the Cloud — Virtual Switch) is the virtual/containerized form of Microsoft's open-source SONiC NOS, widely deployed on merchant-silicon switches in hyperscale data centers. Every command is shown with expected output.

### Step 1 — Write the Containerlab topology file

Create `evpn-lab.clab.yaml`:

```yaml
name: evpn-lab

topology:
  nodes:
    # Spine nodes (SR Linux)
    spine1:
      kind: srl
      image: ghcr.io/nokia/srlinux:23.10.1
      startup-config: configs/spine1.cfg
    spine2:
      kind: srl
      image: ghcr.io/nokia/srlinux:23.10.1
      startup-config: configs/spine2.cfg

    # Leaf nodes (SONiC-VS)
    leaf1:
      kind: sonic-vs
      image: docker.io/antrea/sonic-vs:20230531
      startup-config: configs/leaf1.json
    leaf2:
      kind: sonic-vs
      image: docker.io/antrea/sonic-vs:20230531
      startup-config: configs/leaf2.json
    leaf3:
      kind: sonic-vs
      image: docker.io/antrea/sonic-vs:20230531
      startup-config: configs/leaf3.json
    leaf4:
      kind: sonic-vs
      image: docker.io/antrea/sonic-vs:20230531
      startup-config: configs/leaf4.json

    # Host containers (Linux with FRR)
    host-a:
      kind: linux
      image: frrouting/frr:v9.0.1
      exec:
        - ip addr add 192.168.100.10/24 dev eth1
        - ip route add default via 192.168.100.1
    host-b:
      kind: linux
      image: frrouting/frr:v9.0.1
      exec:
        - ip addr add 192.168.100.20/24 dev eth1
        - ip route add default via 192.168.100.1

  links:
    # Spine1 uplinks to all leaves
    - endpoints: [spine1:e1-1, leaf1:e1-49]
    - endpoints: [spine1:e1-2, leaf2:e1-49]
    - endpoints: [spine1:e1-3, leaf3:e1-49]
    - endpoints: [spine1:e1-4, leaf4:e1-49]
    # Spine2 uplinks to all leaves
    - endpoints: [spine2:e1-1, leaf1:e1-50]
    - endpoints: [spine2:e1-2, leaf2:e1-50]
    - endpoints: [spine2:e1-3, leaf3:e1-50]
    - endpoints: [spine2:e1-4, leaf4:e1-50]
    # Host connections
    - endpoints: [host-a:eth1, leaf1:e1-1]
    - endpoints: [host-b:eth1, leaf3:e1-1]
```

### Step 2 — Deploy the lab

```bash
sudo containerlab deploy -t evpn-lab.clab.yaml
```

Expected output (abbreviated):

```
INFO[0000] Containerlab v0.54.0 started
INFO[0000] Parsing & checking topology file: evpn-lab.clab.yaml
INFO[0001] Creating lab directory: /home/user/clab-evpn-lab
INFO[0002] Creating docker network: Name="clab", IPv4Subnet="172.20.20.0/24"
INFO[0005] Creating container: spine1
INFO[0007] Creating container: spine2
INFO[0009] Creating container: leaf1
INFO[0011] Creating container: leaf2
INFO[0013] Creating container: leaf3
INFO[0015] Creating container: leaf4
INFO[0017] Creating container: host-a
INFO[0018] Creating container: host-b
INFO[0020] Creating virtual wire: spine1:e1-1 <--> leaf1:e1-49
INFO[0020] Creating virtual wire: spine1:e1-2 <--> leaf2:e1-49
INFO[0020] Creating virtual wire: spine1:e1-3 <--> leaf3:e1-49
INFO[0020] Creating virtual wire: spine1:e1-4 <--> leaf4:e1-49
INFO[0020] Creating virtual wire: spine2:e1-1 <--> leaf1:e1-50
INFO[0020] Creating virtual wire: spine2:e1-2 <--> leaf2:e1-50
INFO[0020] Creating virtual wire: spine2:e1-3 <--> leaf3:e1-50
INFO[0020] Creating virtual wire: spine2:e1-4 <--> leaf4:e1-50
INFO[0021] Creating virtual wire: host-a:eth1 <--> leaf1:e1-1
INFO[0021] Creating virtual wire: host-b:eth1 <--> leaf3:e1-1
INFO[0025] Adding containerlab host entries to /etc/hosts file
+---+---------------------+--------------+----------------------------+-------+---------+
| # |        Name         | Container ID |           Image            | Kind  |  State  |
+---+---------------------+--------------+----------------------------+-------+---------+
| 1 | clab-evpn-lab-host-a| a1b2c3d4e5f6 | frrouting/frr:v9.0.1       | linux | running |
| 2 | clab-evpn-lab-host-b| b2c3d4e5f6a7 | frrouting/frr:v9.0.1       | linux | running |
| 3 | clab-evpn-lab-leaf1 | c3d4e5f6a7b8 | antrea/sonic-vs:20230531   | sonic | running |
| 4 | clab-evpn-lab-leaf2 | d4e5f6a7b8c9 | antrea/sonic-vs:20230531   | sonic | running |
| 5 | clab-evpn-lab-leaf3 | e5f6a7b8c9d0 | antrea/sonic-vs:20230531   | sonic | running |
| 6 | clab-evpn-lab-leaf4 | f6a7b8c9d0e1 | antrea/sonic-vs:20230531   | sonic | running |
| 7 | clab-evpn-lab-spine1| a7b8c9d0e1f2 | ghcr.io/nokia/srlinux:23.10.1 | srl | running |
| 8 | clab-evpn-lab-spine2| b8c9d0e1f2a3 | ghcr.io/nokia/srlinux:23.10.1 | srl | running |
+---+---------------------+--------------+----------------------------+-------+---------+
```

Verify all containers are running:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep evpn-lab
```

### Step 3 — Configure the BGP underlay (eBGP on each leaf)

Shell into leaf1's FRR daemon. `vtysh` is FRR's unified configuration shell — it presents a Cisco-like CLI that wraps all FRR daemons (zebra, bgpd, etc.) in a single interactive interface.

```bash
docker exec -it clab-evpn-lab-leaf1 vtysh
```

Inside vtysh, configure eBGP underlay toward both spines (leaf1 is AS 65101):

```
leaf1# configure terminal
leaf1(config)# router bgp 65101
leaf1(config-router)#  bgp router-id 10.1.1.1
leaf1(config-router)#  no bgp ebgp-requires-policy
leaf1(config-router)#  neighbor 10.0.0.1 remote-as 65000
leaf1(config-router)#  neighbor 10.0.0.1 description spine1
leaf1(config-router)#  neighbor 10.0.0.2 remote-as 65000
leaf1(config-router)#  neighbor 10.0.0.2 description spine2
leaf1(config-router)#  address-family ipv4 unicast
leaf1(config-router-af)#   network 10.1.1.1/32
leaf1(config-router-af)#  exit-address-family
leaf1(config-router)# exit
leaf1(config)# end
leaf1# write memory
```

Repeat for leaf2 (AS 65102, router-id 10.1.1.2), leaf3 (AS 65103, router-id 10.1.1.3), and leaf4 (AS 65104, router-id 10.1.1.4) with their respective spine-facing IPs.

Verify BGP underlay is established:

```bash
docker exec -it clab-evpn-lab-leaf1 vtysh -c "show bgp summary"
```

Expected output:

```
IPv4 Unicast Summary (VRF default):
BGP router identifier 10.1.1.1, local AS number 65101 vrf-id 0
BGP table version 8
RIB entries 7, using 1344 bytes of memory
Peers 2, using 1450 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.0.0.1        4      65000       142       140        8    0    0 00:11:23            4        4 spine1
10.0.0.2        4      65000       139       141        8    0    0 00:11:20            4        4 spine2

Total number of neighbors 2
```

Both peers show `State/PfxRcd` as a number (not `Active` or `Idle`), confirming BGP sessions are established.

### Step 4 — Configure VXLAN interfaces on each leaf

These commands run directly in the leaf container's Linux shell (not vtysh). Leaf1 example (VTEP IP 10.1.1.1):

```bash
docker exec -it clab-evpn-lab-leaf1 bash
```

Inside leaf1's shell:

```bash
# Create VXLAN interface for VNI 100
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    local 10.1.1.1 \
    nolearning \
    dev eth0

# Bring up the VXLAN interface
ip link set vxlan100 up

# Create a bridge and add the VXLAN interface to it
ip link add br100 type bridge
ip link set br100 up
ip link set vxlan100 master br100

# Assign the bridge an anycast gateway IP (for IRB)
ip addr add 192.168.100.1/24 dev br100

# Verify interface is created
ip link show vxlan100
```

Expected output of `ip link show vxlan100`:

```
7: vxlan100: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br100 state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 8a:3f:11:22:33:44 brd ff:ff:ff:ff:ff:ff
```

Repeat on leaf2 (local 10.1.1.2), leaf3 (local 10.1.1.3), and leaf4 (local 10.1.1.4) with the same VNI 100.

### Step 5 — Configure FRR EVPN on each leaf

Re-enter vtysh on leaf1:

```bash
docker exec -it clab-evpn-lab-leaf1 vtysh
```

Add the EVPN address-family to the existing BGP configuration:

```
leaf1# configure terminal
leaf1(config)# router bgp 65101
leaf1(config-router)#  address-family l2vpn evpn
leaf1(config-router-af)#   neighbor 10.0.0.1 activate
leaf1(config-router-af)#   neighbor 10.0.0.2 activate
leaf1(config-router-af)#   advertise-all-vni
leaf1(config-router-af)#   vni 100
leaf1(config-router-af-vni)#    rd 10.1.1.1:100
leaf1(config-router-af-vni)#    route-target import 65001:100
leaf1(config-router-af-vni)#    route-target export 65001:100
leaf1(config-router-af-vni)#   exit-vni
leaf1(config-router-af)#  exit-address-family
leaf1(config-router)# exit
leaf1(config)# end
leaf1# write memory
```

Repeat on leaf2, leaf3, leaf4 — changing `rd` to their own router-id (e.g., `10.1.1.3:100` on leaf3) but keeping the same route-target values so all leaves import each other's routes.

### Step 6 — Verify BGP EVPN routes

On leaf1, check BGP EVPN table:

```bash
docker exec -it clab-evpn-lab-leaf1 vtysh -c "show bgp l2vpn evpn"
```

Expected output (all four leaves visible):

```
BGP table version is 12, local router ID is 10.1.1.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-1 prefix: [1]:[EthTag]:[ESI]:[IPlen]:[VTEP-IP]:[Frag-id]
EVPN type-2 prefix: [2]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-4 prefix: [4]:[ESI]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 10.1.1.1:100
*> [3]:[0]:[32]:[10.1.1.1]
                    10.1.1.1                           32768 i
                    ET:8 RT:65001:100
Route Distinguisher: 10.1.1.2:100
*> [3]:[0]:[32]:[10.1.1.2]
                    10.1.1.2                               0 65000 65102 i
                    ET:8 RT:65001:100
Route Distinguisher: 10.1.1.3:100
*> [3]:[0]:[32]:[10.1.1.3]
                    10.1.1.3                               0 65000 65103 i
                    ET:8 RT:65001:100
Route Distinguisher: 10.1.1.4:100
*> [3]:[0]:[32]:[10.1.1.4]
                    10.1.1.4                               0 65000 65104 i
                    ET:8 RT:65001:100
```

The `[3]` entries are Type 3 routes (VTEP membership/IMET — Inclusive Multicast Ethernet Tag, the EVPN route type that advertises a VTEP's presence in a VNI and is used to build the BUM flood list). After host-a sends its first packet, Type 2 routes appear:

```bash
docker exec -it clab-evpn-lab-leaf1 vtysh -c "show bgp l2vpn evpn route type macip"
```

Expected output after host-a has sent traffic:

```
BGP table version is 15, local router ID is 10.1.1.1
   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 10.1.1.1:100
*> [2]:[0]:[48]:[aa:bb:cc:11:22:01]:[32]:[192.168.100.10]
                    10.1.1.1                           32768 i
                    ET:8 RT:65001:100 Rmac:8a:3f:11:22:33:44
Route Distinguisher: 10.1.1.3:100
*> [2]:[0]:[48]:[aa:bb:cc:11:22:03]:[32]:[192.168.100.20]
                    10.1.1.3                               0 65000 65103 i
                    ET:8 RT:65001:100 Rmac:8a:3f:11:22:33:55
```

The `[2]` entries are Type 2 routes carrying MAC+IP pairs. Leaf1 has learned host-b's MAC (192.168.100.20 at leaf3) via BGP without any flooding.

### Step 7 — Ping from host-a to host-b

Open a shell on host-a:

```bash
docker exec -it clab-evpn-lab-host-a bash
ping -c 4 192.168.100.20
```

Expected output:

```
PING 192.168.100.20 (192.168.100.20) 56(84) bytes of data.
64 bytes from 192.168.100.20: icmp_seq=1 ttl=64 time=2.34 ms
64 bytes from 192.168.100.20: icmp_seq=2 ttl=64 time=1.87 ms
64 bytes from 192.168.100.20: icmp_seq=3 ttl=64 time=1.92 ms
64 bytes from 192.168.100.20: icmp_seq=4 ttl=64 time=1.95 ms

--- 192.168.100.20 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 1.870/2.020/2.340/0.190 ms
```

If you see 100% packet loss on the first attempt, wait 5 seconds for ARP to resolve via EVPN and retry.

### Step 8 — Capture and dissect the VXLAN-encapsulated packet

While the ping is running, on the host machine capture traffic on the link between leaf1 and spine1. First find the interface name:

```bash
# On the host, list veth interfaces created by Containerlab
ip link show | grep -E "clab|e1-49"
# The interface on the host side of leaf1:e1-49 will appear as something like:
# 42: leaf1-e1-49@if2: <BROADCAST,MULTICAST,UP,LOWER_UP>
```

Capture VXLAN packets (UDP port 4789) on that interface:

```bash
sudo tcpdump -i leaf1-e1-49 -n -v udp port 4789
```

Expected output for one ICMP echo inside VXLAN:

```
14:23:15.112034 IP (tos 0x0, ttl 63, id 12345, offset 0, flags [DF], proto UDP (17), length 134)
    10.1.1.1 > 10.1.1.3: UDP, length 106
        VXLAN, flags [I] (0x08), vni 100
        IP (tos 0x0, ttl 64, id 54321, offset 0, flags [DF], proto ICMP (1), length 84)
            192.168.100.10 > 192.168.100.20: ICMP echo request, id 7, seq 2, length 64
```

Layer breakdown:

```
Outer Ethernet :  leaf1 MAC → spine1 MAC            (14 bytes)
Outer IP       :  src=10.1.1.1 (leaf1 VTEP)          (20 bytes)
                  dst=10.1.1.3 (leaf3 VTEP)
Outer UDP      :  src=<ephemeral>  dst=4789           ( 8 bytes)
VXLAN header   :  flags=0x08 (I bit set), VNI=100     ( 8 bytes)
Inner Ethernet :  host-a MAC → host-b MAC
Inner IP       :  src=192.168.100.10 dst=192.168.100.20
Inner ICMP     :  echo request seq=2
```

To capture as a pcap for offline analysis:

```bash
sudo tcpdump -i leaf1-e1-49 -n -w /tmp/vxlan-capture.pcap udp port 4789
# Then open with: wireshark /tmp/vxlan-capture.pcap
```

In Wireshark, the VXLAN dissector will automatically decode the inner Ethernet frame. Filter with: `vxlan.vni == 100`

### Step 9 — Verify FDB population from EVPN

On leaf1, check that FRR has populated the bridge FDB with the remote host-b MAC learned via BGP:

```bash
docker exec -it clab-evpn-lab-leaf1 bridge fdb show dev vxlan100
```

Expected output:

```
00:00:00:00:00:00 dev vxlan100 dst 10.1.1.2 self permanent
00:00:00:00:00:00 dev vxlan100 dst 10.1.1.3 self permanent
00:00:00:00:00:00 dev vxlan100 dst 10.1.1.4 self permanent
aa:bb:cc:11:22:03 dev vxlan100 dst 10.1.1.3 self permanent
```

Interpretation:
- The three `00:00:00:00:00:00` entries are BUM (broadcast/unknown multicast) flood list entries — FRR installs one per remote VTEP discovered via Type 3 IMET routes.
- The `aa:bb:cc:11:22:03` entry is a unicast MAC entry for host-b's MAC address, learned via Type 2 EVPN route from leaf3 (VTEP 10.1.1.3). Traffic to host-b will be unicast-tunneled to 10.1.1.3 without flooding.

To verify the full connectivity matrix, repeat the FDB check on leaf3 and confirm host-a's MAC appears pointing to VTEP 10.1.1.1.

### Step 10 — Destroy the lab

```bash
sudo containerlab destroy -t evpn-lab.clab.yaml --cleanup
```

---

## Summary

- VXLAN provides 24-bit segment IDs over a UDP encapsulation, enabling 16M isolated L2 domains over a shared IP underlay.
- BGP-EVPN replaces MAC flooding with BGP-distributed MAC/IP reachability; Type 2 routes carry MAC+IP, Type 3 routes carry VTEP membership, Type 5 routes carry IP prefixes for inter-subnet routing.
- OVS is the datapath engine: OpenFlow-programmable, VXLAN-capable, with kernel and DPDK fast paths.
- OVN is the intent-layer above OVS: logical switches, routers, and ACLs compiled down to per-host OVS flows by `ovn-controller`.

---

## References

- RFC 7348: VXLAN — datatracker.ietf.org/doc/html/rfc7348
- RFC 7432: BGP MPLS-Based Ethernet VPN — datatracker.ietf.org/doc/html/rfc7432
- RFC 8365: Network Virtualization over L3 — datatracker.ietf.org/doc/html/rfc8365
- OVS documentation: docs.openvswitch.org
- OVN documentation: docs.ovn.org


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).