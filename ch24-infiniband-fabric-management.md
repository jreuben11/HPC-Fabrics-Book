# Chapter 24 — InfiniBand Fabric Management

**Part I: Foundations** | ~20 pages

---

## Introduction

Chapter 24 covers **InfiniBand** fabric management from first principles — the transport model that distinguishes IB from **RoCEv2**, the Subnet Manager that is IB's central architectural feature, the diagnostic tools that expose fabric health and routing state, and the **Priority Flow Control** mechanism that must be correctly configured for lossless **RDMA**. The Lab Walkthrough instantiates a two-node **InfiniBand** fabric using the `rdma_rxe` software HCA module, runs **OpenSM** to assign LIDs and program forwarding tables, and validates performance with the **perftest** benchmark suite.

**InfiniBand** occupies a distinct position in the AI infrastructure landscape. While the industry trend toward **RoCEv2** on commodity Ethernet is real and accelerating, IB retains dominance in on-premises HPC and AI installations where fabric management predictability, sub-microsecond latency, and native lossless operation outweigh the switch cost premium. Understanding IB fabric management is also directly useful for **RoCEv2** operations: the same LID-based addressing, the same **MAD**-based diagnostics, and the same **PFC** lossless requirements appear in both transport families, and the tooling (`ibstat`, `ibnetdiscover`, `perfquery`) is shared across both.

Chapter 23 covered simulation tools including **NS-3**'s RdmaHw module, which models the exact **DCQCN** and ECN thresholds discussed in Sections 24.5–24.6. Chapter 24 deals with the physical reality those simulations represent. Chapter 25 extends **RDMA** management to the cloud — **AWS EFA**, Azure **RDMA**, and GCP **TCPX** — where the fabric is managed by the cloud provider but the host-side configuration remains the operator's responsibility.

The Subnet Manager is what makes IB distinct. Every IB port in the fabric is in the `INIT` state at power-on; without a running SM, no data path is operational. The SM discovers the topology via directed-route **MAD**s, assigns 16-bit LIDs to every port, computes full mesh paths using a configurable routing algorithm (FTREE, LASH, DOR), and programs Linear Forwarding Tables into every switch. SM failover, sweep interval tuning, and partition key management are therefore critical operational disciplines rather than optional optimizations.

Section 24.1 contrasts the IB and **RoCEv2** transport models at the architectural level. Sections 24.2–24.3 cover **OpenSM** operation and the **MAD**-based diagnostic suite. Section 24.4 explains fat-tree routing algorithms. Section 24.5 addresses **Priority Flow Control**. Section 24.6 covers performance counter monitoring. Section 24.7 describes the Soft-IB kernel module used throughout the Lab Walkthrough.

---

## Installation

The `opensm` and `opensm-utils` packages provide the **OpenSM** Subnet Manager, which is the mandatory control-plane component that assigns LIDs, computes routing tables, and programs Linear Forwarding Tables into every switch; without a running SM, all IB ports remain in the `INIT` state and no data path is operational. The `infiniband-diags` package supplies the diagnostic suite — `ibnetdiscover`, `ibdiagnet`, `perfquery`, `ibstat`, and `ibping` — which speak **MAD**-based management protocols to enumerate fabric topology, detect link errors, and collect 64-bit performance counters. The `perftest` package provides the `ib_write_bw`, `ib_read_bw`, and `ib_write_lat` benchmarks used throughout the lab walkthrough to exercise **RDMA** verbs and validate end-to-end fabric performance. The `rdma_rxe` kernel module implements a full software **InfiniBand** HCA on top of any Ethernet interface and serves as the lab stand-in for a physical HCA in all SM and diagnostic exercises.

### System Packages (Ubuntu 24.04)

```bash
sudo apt update
sudo apt install -y \
    rdma-core \
    ibverbs-utils \
    infiniband-diags \
    opensm \
    libibmad-dev \
    libibnetdisc-dev \
    perftest \
    linux-modules-extra-$(uname -r)
```

Verify the diagnostic suite is available:

```bash
dpkg -l | grep -E "opensm|infiniband-diags|perftest|rdma-core"
# Expected lines:
# ii  infiniband-diags  ...  InfiniBand diagnostic programs
# ii  opensm            ...  InfiniBand subnet manager
# ii  perftest          ...  InfiniBand verbs performance tests
# ii  rdma-core         ...  RDMA core userspace libraries and daemons

# Verify opensm binary is present
which opensm
# Expected: /usr/sbin/opensm

# Verify diagnostic tools are present
which ibnetdiscover ibdiagnet ibstat ibping perfquery
# Expected: one path per tool in /usr/sbin or /usr/bin
```

### Soft-IB kernel module

```bash
# Ensure rdma_rxe is available (Soft-IB/Soft-RoCE driver)
sudo modprobe rdma_rxe

lsmod | grep rdma_rxe
# Expected:
# rdma_rxe              196608  0
# ib_core               393216  4 rdma_rxe,ib_uverbs,...
```

### Python uv environment

```bash
# Install uv if not already present
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# Create project environment
mkdir -p ~/ib-fabric-mgmt && cd ~/ib-fabric-mgmt
uv init .
uv add prometheus-client pandas

# Verify
uv run python -c "import prometheus_client, pandas; print('prometheus_client:', prometheus_client.__version__, '  pandas:', pandas.__version__)"
# Expected:
# prometheus_client: 0.20.x   pandas: 2.x.x
```

---

## 24.1 IB vs RoCEv2: Transport Model Differences

InfiniBand and RoCEv2 share the same verbs API (libibverbs) and largely the same kernel subsystem (ib_core), but they differ fundamentally in their transport model, fabric management requirements, and congestion handling.

### 24.1.1 Transport Encapsulation

| Property | InfiniBand | RoCEv2 |
|---|---|---|
| Layer-2 header | IB Local Route Header (LRH) | Ethernet MAC header |
| Layer-3/routing | Subnet Manager assigns LIDs; LFT-based forwarding in switches | IP routing; standard L3 forwarding |
| Encapsulation | Native IB transport (BTH, RETH, etc.) | IB transport headers over UDP/IP (UDP dst port 4791) |
| Address scheme | 16-bit LID (local), 128-bit GID (global) | GID = IPv6 or IPv4-mapped IPv6 address |
| Switch hardware | IB switches (SilverStorm, Mellanox, Cornelis) | Commodity Ethernet switches |
| Congestion control | Credit-based flow control (native, lossless by design) | DCQCN (ECN + CNP feedback loop) |

### 24.1.2 The Subnet Manager: IB's Central Difference

The most architecturally significant distinction is the **Subnet Manager (SM)**. In a pure InfiniBand fabric, every switch and HCA port is configured by the SM at startup and whenever the fabric topology changes. The SM:

1. Discovers all nodes via a depth-first sweep using directed-route MADs (Management Attribute Datagrams).
2. Assigns LIDs to all ports.
3. Computes paths across the fabric using the configured routing algorithm (FTREE, LASH, DOR, etc.).
4. Programs Linear Forwarding Tables (LFTs) into every switch.
5. Configures partition keys (P_Keys) for multi-tenant isolation.
6. Periodically sends `PortInfo` MADs to detect link failures (Sweep Interval, default 10 seconds).

Without an SM, IB ports remain in the `INIT` state and cannot form QPs. This is unlike RoCEv2, which relies entirely on IP routing infrastructure (BGP/OSPF/static routes) and requires no fabric-level orchestrator.

### 24.1.3 When to Choose Each Transport

| Scenario | Prefer IB | Prefer RoCEv2 |
|---|---|---|
| Greenfield HPC / AI cluster ≤ 1000 nodes | Yes — predictable, lossless, lower tuning burden | — |
| Multi-rack AI pod attached to existing Ethernet spine | — | Yes — reuses IP routing |
| Latency-critical MPI (sub-microsecond) | Yes — HDR/NDR hardware, no ECN loop | — |
| Mixed AI + storage traffic on same fabric | — | Yes — commodity switches support both |
| Cost sensitivity at scale | — | Yes — no IB switch premium |
| Multi-subnet routing required | Both (IB via IB router or RoCEv2 natively) | Yes |

The practical decision in 2024-2025 hyperscale AI clusters is predominantly RoCEv2 on 400GbE/800GbE because the switch cost delta has collapsed and ECN-based congestion control is mature. IB retains dominance in on-prem HPC installations where procurement is IB-centric and dedicated SM management is acceptable.

---

## 24.2 IB Subnet Manager: OpenSM

`opensm` is the open-source reference implementation of the IB Subnet Manager, maintained as part of `rdma-core`. In production, Mellanox/NVIDIA ships its own UFM (Unified Fabric Manager), but OpenSM is standard for lab, small-cluster, and open-source HPC deployments.

### 24.2.1 opensm Configuration

The primary configuration file is `/etc/opensm/opensm.conf`. Key parameters:

```ini
# /etc/opensm/opensm.conf (relevant excerpts)

# GUID of the HCA port OpenSM should bind to (use ibstat to find)
guid 0x0002c9030000f3e1

# SM priority (0 = lowest, 15 = highest). Higher priority wins election.
sm_priority 0

# Routing engine selection: ftree, minhop, lash, dor, torus-2QoS
routing_engine ftree

# Sweep interval in seconds (topology re-discovery)
sweep_interval 10

# Log verbosity: 0x00=none, 0x03=info, 0x07=verbose, 0xff=debug
log_flags 0x03

# Log file location
log_file /var/log/opensm.log

# Partition configuration file (P_Key membership)
partition_config /etc/opensm/partitions.conf

# Enable SM failover: allow a standby SM to take over if this SM fails
# Set a higher sm_priority on the backup node
```

### 24.2.2 Starting OpenSM

```bash
# Start on a specific port GUID (get GUID from ibstat)
sudo opensm -g 0x<port_guid> &

# Or as a systemd service (Ubuntu 24.04)
sudo systemctl start opensm
sudo systemctl enable opensm

# Watch SM election in the log
sudo tail -f /var/log/opensm.log | grep -E "SM|MASTER|subnet"
```

Expected log fragment during initial subnet sweep:

```
Mar 12 10:23:01 opensm[12345]: OpenSM 5.18.0
Mar 12 10:23:01 opensm[12345]: Entering DISCOVERY state
Mar 12 10:23:01 opensm[12345]: Discovered 2 HCA(s) and 0 switch(es)
Mar 12 10:23:01 opensm[12345]: Entering MASTER state
Mar 12 10:23:01 opensm[12345]: SUBNET UP: 2 ports assigned LIDs
Mar 12 10:23:01 opensm[12345]: Routing: ftree routing engine
Mar 12 10:23:01 opensm[12345]: Initialization complete
```

### 24.2.3 SM Failover and Priority

In production fabrics, a **standby SM** runs on a second node. The SM election protocol is defined in the IB spec: the SM with the higher `sm_priority` value becomes MASTER; all others remain in STANDBY and monitor the MASTER via periodic `SMInfo` MADs.

```bash
# Query current SM state and priority
smpquery -D SMInfo 0   # directed-route MAD to local port
# Expected:
# SM_Info dump:
#    GUID........................0x0002c9030000f3e1
#    SM_key......................0x0000000000000000
#    ActCount....................17
#    Priority....................0
#    SMState.....................3   # 3 = MASTER, 2 = STANDBY, 1 = NOTACTIVE
```

If the MASTER SM crashes, the STANDBY SM detects the absence of `SMInfo` heartbeats within `sminfo_polling_timeout` (default: 10 seconds × `polling_retry_number`) and transitions itself to MASTER, performing a full re-sweep.

### 24.2.4 subnet.lst

OpenSM writes a persistent copy of the subnet topology to `/var/cache/opensm/subnet.lst` after each successful sweep:

```
#
# Subnet topology dump - OpenSM
#
vendid=0x2c9 devid=0x1003 sysimgguid=0x0002c9030000f3e1 caguid=0x0002c9030000f3e1
DR path: 0
  portguid=0x0002c9030000f3e1 lid=1 dname="MT27800 Family"
  portguid=0x0002c9030000f3e2 lid=2 dname="MT27800 Family"
```

This file is the ground truth for LID-to-GUID mapping. Tooling like `ibnetdiscover` reads live MADs; `subnet.lst` provides a cached baseline for drift detection.

---

## 24.3 Fabric Discovery Tools

InfiniBand's management channel is completely separate from the data path. Management Datagrams (MADs) flow over the same physical links but use a dedicated SMP (Subnet Management Packet) protocol. The diagnostic tools all speak MADs.

### 24.3.1 ibstat and ibstatus

```bash
# ibstat: per-port detail including GID, LID, SM LID, rate, state
ibstat

# ibstatus: brief port state summary
ibstatus
```

Annotated `ibstat` output:

```
CA 'rxe0'                          # CA = Channel Adapter (HCA)
        CA type:
        Number of ports: 1
        Firmware version:
        Hardware version:
        Node GUID: 0xaabbccddeeff0001   # 64-bit node identifier
        System image GUID: 0xaabbccddeeff0001
        Port 1:
                State: Active           # Active = data path operational
                Physical state: Polling # Physical layer negotiating
                Rate: 10               # Gbps for rxe (real HCA: 200, 400)
                Base lid: 1            # LID assigned by SM
                LMC: 0                 # LID mask count (multipath)
                SM lid: 1              # LID of the current SM
                Capability mask: 0x02514868
                Port GUID: 0xaabbccddeeff0001
                Link layer: InfiniBand
```

Key state values to monitor:

| State | Meaning |
|---|---|
| `Active` | Normal operation; LID assigned |
| `Armed` | Link up but not yet configured by SM |
| `Init` | Physical link up; SM has not yet assigned LID |
| `Down` | Physical layer not established |

### 24.3.2 ibnetdiscover

`ibnetdiscover` sends directed-route MADs from the local SM port to enumerate all nodes and links in the fabric:

```bash
# Full topology dump to stdout
ibnetdiscover

# Save to file and also dump switch forwarding tables
ibnetdiscover -g --show > /tmp/topology.lst
```

Annotated output excerpt:

```
#
# Topology file: generated on Tue Mar 12 10:25:00 2024
# Max hops discovered: 1
# Initiated from node 0xaabbccddeeff0001 port 0xaabbccddeeff0001

# HCA nodes
Ca    2 "H-aabbccddeeff0001"       # Ca = Channel Adapter, 2 = port count
[1]   "H-aabbccddeeff0002"[1]      # this port connects to other HCA port 1
      # H- prefix = HCA node GUID
      # S- prefix = Switch node GUID

# Full topology:
vendid=0x0 devid=0x0
Ca        1 "H-aabbccddeeff0001"
[1](aabbccddeeff0001)               # [port](port GUID)
vendid=0x0 devid=0x0
Ca        1 "H-aabbccddeeff0002"
[1](aabbccddeeff0002)
```

In a production fat-tree fabric with Mellanox switches, the output includes switch nodes (`Sw` type) and you can reconstruct the full leaf-spine-core topology graph.

### 24.3.3 ibdiagnet

`ibdiagnet` is the comprehensive fabric diagnostic tool. It performs a full topology scan and reports errors:

```bash
# Full diagnostic scan
sudo ibdiagnet

# Scan with PC (port counter) collection
sudo ibdiagnet --pc

# Check for BER (Bit Error Rate) above threshold
sudo ibdiagnet --ber_thr 1e-12
```

Expected output on a clean 2-node rxe fabric:

```
Loading IBDIAGNET plugins ... Done
-W- IBDiagnet is not meant to run on Virtual RDMA devices
Discovering fabric ... 2 nodes discovered.

-- Link Checks --
No bad links found.

-- SM Checks --
SM: LID 0x0001 GUID 0xaabbccddeeff0001 Priority 0 State Master (0x3)

-- Error Checks --
No errors found.

-- Fabric Summary --
  Total nodes      : 2
  IB SWitches      : 0
  Channel Adapters : 2
  Routers          : 0
```

### 24.3.4 ibping and ibsysstat

```bash
# ibping: RDMA layer ping (uses MAD PerfMgt class)
# First, find the LID of the target
ibstat rxe0 | grep "Base lid"
# Expected: Base lid: 1

# Ping by LID
ibping -L 2   # ping the node with LID 2
# Expected:
# Pinging LID 2 (Guid 0xaabbccddeeff0002): time 0.132 ms
# Pinging LID 2 (Guid 0xaabbccddeeff0002): time 0.128 ms

# ibsysstat: fetch system info (CPU, memory) from a remote IB node
ibsysstat -L 2
# Expected:
# 2: Hostname: myhost
#    CPU utilization: 5%
#    Memory utilization: 42%
```

---

## 24.4 Fat-Tree Routing Algorithms

A fat-tree is the canonical topology for IB HPC clusters: k-ary fat-tree with k leaf switches (each connecting k/2 servers and k/2 uplinks) and k/2 core switches. The topology is non-blocking at full bisection bandwidth.

### 24.4.1 Linear Forwarding Tables (LFTs)

Each IB switch maintains a **Linear Forwarding Table**: a 65536-entry array indexed by LID, where each entry is the output port number for that destination. The SM computes and programs all LFTs after topology discovery.

```
LFT for switch S1 (simplified):
  LID 1  → port 3  (to HCA H1)
  LID 2  → port 4  (to HCA H2)
  LID 3  → port 1  (to core switch C1 → HCA H3)
  LID 4  → port 2  (to core switch C2 → HCA H4)
```

### 24.4.2 FTREE Routing Engine

`routing_engine ftree` in `opensm.conf` selects the fat-tree routing algorithm. FTREE:

1. Detects that the topology is a fat-tree by analyzing switch connectivity.
2. Enumerates all HCA-to-HCA paths, maximally using all uplinks (load-balanced round-robin across all equal-cost paths).
3. Ensures that the LFTs for any two communicating HCAs use disjoint paths, maximizing bisection bandwidth for all-to-all collective patterns.

```bash
# Verify routing engine selection in SM log
grep "routing engine" /var/log/opensm.log
# Expected:
# Routing engine: ftree
# Fat-tree routing: Successfully found 1 root switch level(s)
```

### 24.4.3 LASH: Layer Assignment for Source Hopping

**LASH (Layer Assignment Source Hopping)** is an alternative routing algorithm that eliminates routing loops by assigning each path to a "virtual layer" (SL, Service Level). Different SLs use different virtual output queues in switches, preventing deadlock without requiring credit-based back-pressure.

```ini
# opensm.conf for LASH
routing_engine lash

# Number of SLs to use (up to 15 in IB)
lash_start_vl 0
```

LASH is particularly effective in irregular topologies or when the fat-tree assumption breaks (e.g., cabling errors, disabled ports).

### 24.4.4 DOR: Dimension Order Routing

**DOR (Dimension Order Routing)** is used in torus and mesh topologies. Routes are computed dimension-by-dimension (X first, then Y, then Z), guaranteeing deadlock freedom by construction:

```ini
routing_engine dor
```

DOR is the preferred algorithm for 3D torus networks (common in Cray/HPE Slingshot fabrics). For standard fat-trees, FTREE is preferred.

### 24.4.5 Viewing Computed LFTs

```bash
# Dump the LFT of a switch (requires active SM and switch)
# For a switch at LID 10:
ibroute 10

# Or via ibnetdiscover with --show
ibnetdiscover --show | grep -A 20 "Switch"
```

---

## 24.5 Priority Flow Control: Lossless IB vs Lossy Ethernet

### 24.5.1 Native IB Lossless Operation

InfiniBand achieves lossless operation through **credit-based flow control** at the link level. Before transmitting, a sender checks that the receiver has posted sufficient credits (buffer space) for the frame. Credits are exchanged via `FlowCon` packets on the same physical link. Because credits are per-virtual-lane, IB switches can never drop a packet due to buffer overflow — the upstream sender is simply paused until credits are replenished.

This design means:
- No explicit congestion notification mechanism is needed (though IB also has Forward Explicit Congestion Notification, FECN/BECN, for bandwidth fairness).
- RoCEv2 on Ethernet must replicate this property using Priority Flow Control (PFC).

### 24.5.2 Priority Flow Control on Ethernet Switches

PFC (IEEE 802.1Qbb) is a per-priority-class pause mechanism for Ethernet. RDMA traffic is tagged with a specific 802.1p priority (typically PCP=3), and PFC pauses only that priority class when buffers fill.

**SONiC switch PFC configuration:**

```bash
# Set PFC to protect RDMA traffic on priority 3
config qos pfc priority 3 --enable

# Configure DSCP-to-PCP mapping (RDMA traffic uses DSCP 26)
config qos dscp-map RDMA 26 3

# Verify PFC counters on an interface
show pfc counters Ethernet4
# Expected:
# Port   Rx Pause    Tx Pause   Rx Pause Duration (usec)   Tx Pause Duration (usec)
# Eth4   0           142        0                          28400
```

**PFC configuration on Mellanox switches (via mlnxos):**

```bash
interface ethernet 1/1
  traffic-class 3 congestion-control ecn minimum-absolute 150 maximum-absolute 1500
  no shutdown
```

### 24.5.3 PFC Deadlock Detection and Prevention

PFC introduces a new failure mode: **PFC deadlock**. If pause frames create a cycle of paused links (A pauses B, B pauses C, C pauses A), all three links freeze permanently — no traffic flows and no timeout will clear it because PFC pauses are refreshed indefinitely.

Detection:

```bash
# Check for stuck PFC pauses (pause duration growing without bound)
watch -n 1 "show pfc counters | grep -v '^$'"
# A PFC deadlock manifests as monotonically increasing Pause Duration
# with zero or near-zero packet counters

# On Linux hosts — check for RX pause frames on the RDMA NIC
ethtool -S eth0 | grep pause
# Expected during deadlock:
# rx_pause_ctrl_phy: 1247839   (growing rapidly)
# tx_pause_ctrl_phy: 0         (host is not pausing switch)
```

Prevention best practices:
- Limit PFC to a single priority class (one lossless queue, rest lossy).
- Enable **PFC Watchdog** on all switches — automatically disables PFC on a port if pause duration exceeds threshold.
- Use DCQCN in addition to PFC: ECN-based rate reduction prevents buffers from filling, making PFC pauses rare.
- Avoid routing RDMA and TCP traffic over the same priority class.

---

## 24.6 Performance Counters and Monitoring

### 24.6.1 perfquery

`perfquery` reads IB performance management counters via MADs:

```bash
# Read all PM counters for port 1 at LID 1
perfquery 1 1

# Read a specific counter
perfquery -x 1 1   # extended counters (64-bit)

# Read all ports of all nodes
perfquery -a
```

Annotated output:

```
# Port counters: Lid 1 port 1
PortSelect:......................1
CounterSelect:...................0xf000
SymbolErrorCounter:..............0      # symbol-level errors (CRC failures)
LinkErrorRecoveryCounter:........0      # times link entered error recovery
LinkDownedCounter:...............0      # times link went to DOWN state
PortRcvErrors:...................0      # received packets with errors
PortRcvRemotePhysicalErrors:.....0      # remote physical errors signaled
PortRcvSwitchRelayErrors:........0      # switch couldn't route (bad LID)
PortXmitDiscards:................0      # packets dropped due to HOL blocking
PortXmitConstraintErrors:........0      # P_Key or SL violations
PortRcvConstraintErrors:.........0      # received packets violating constraints
LocalLinkIntegrityErrors:........0      # local PHY integrity issues
ExcessiveBufferOverrunErrors:....0      # buffer overrun (credit violation)
VL15Dropped:.....................0      # dropped management MADs (overload)
PortXmitData:....................183492608   # bytes transmitted (×4 for bits)
PortRcvData:.....................183492608   # bytes received
PortXmitPkts:....................2797       # packets transmitted
PortRcvPkts:.....................2797       # packets received
```

Non-zero values for `PortRcvErrors`, `PortXmitDiscards`, or `ExcessiveBufferOverrunErrors` indicate a fabric problem requiring investigation.

### 24.6.2 Sysfs RDMA Counters

The kernel exposes the same counters via sysfs for scripting:

```bash
# List available counters for rxe0 port 1
ls /sys/class/infiniband/rxe0/ports/1/counters/

# Read key counters
for c in port_rcv_errors port_xmit_discards port_xmit_data port_rcv_data; do
    val=$(cat /sys/class/infiniband/rxe0/ports/1/counters/$c)
    echo "$c = $val"
done
# Expected on idle fabric:
# port_rcv_errors = 0
# port_xmit_discards = 0
# port_xmit_data = 183492608
# port_rcv_data = 183492608
```

### 24.6.3 ibdump: Packet Capture

`ibdump` is a Mellanox/NVIDIA utility that captures raw InfiniBand packets from an HCA port and writes them to a pcap file; Wireshark has a built-in IB dissector that decodes BTH, RETH, and MAD headers directly from the capture file. It captures IB packets to a pcap file, readable by Wireshark:

```bash
# Capture on rxe0 port 1, write to file
sudo ibdump -d rxe0 -i 1 -o /tmp/ib_capture.pcap &
DUMP_PID=$!

# Run some RDMA traffic (e.g., ib_write_bw)
sleep 5

# Stop capture
kill $DUMP_PID

# Inspect with Wireshark (IB dissector is built-in)
# or with tcpdump (for RoCEv2 UDP encapsulation on real NICs)
tcpdump -r /tmp/ib_capture.pcap -c 20
```

### 24.6.4 Prometheus Integration with rdma_exporter

A production monitoring pipeline exports RDMA counters to Prometheus. The following Python script implements a minimal exporter using the sysfs counter interface:

```python
# ~/ib-fabric-mgmt/rdma_exporter.py
"""Minimal RDMA Prometheus exporter reading sysfs counters."""

import os
import time
from pathlib import Path
from prometheus_client import start_http_server, Gauge

SYSFS_IB = Path("/sys/class/infiniband")
POLL_INTERVAL = 15  # seconds

COUNTER_NAMES = [
    "port_rcv_data",
    "port_xmit_data",
    "port_rcv_pkts",
    "port_xmit_pkts",
    "port_rcv_errors",
    "port_xmit_discards",
    "port_rcv_constraint_errors",
    "excessive_buffer_overrun_errors",
    "vl15_dropped",
]

# Create gauges for each counter
gauges: dict[str, Gauge] = {}
for name in COUNTER_NAMES:
    gauges[name] = Gauge(
        f"rdma_{name}",
        f"RDMA counter: {name}",
        ["device", "port"],
    )


def collect_counters() -> None:
    """Read all sysfs counters and update Prometheus gauges."""
    if not SYSFS_IB.exists():
        return
    for dev_path in SYSFS_IB.iterdir():
        dev = dev_path.name
        ports_path = dev_path / "ports"
        if not ports_path.exists():
            continue
        for port_path in ports_path.iterdir():
            port = port_path.name
            counters_path = port_path / "counters"
            if not counters_path.exists():
                continue
            for counter_name in COUNTER_NAMES:
                counter_file = counters_path / counter_name
                if counter_file.exists():
                    try:
                        value = int(counter_file.read_text().strip())
                        gauges[counter_name].labels(device=dev, port=port).set(value)
                    except (ValueError, OSError):
                        pass


if __name__ == "__main__":
    start_http_server(9915)
    print("RDMA exporter listening on :9915/metrics")
    while True:
        collect_counters()
        time.sleep(POLL_INTERVAL)
```

Run the exporter and verify metrics:

```bash
cd ~/ib-fabric-mgmt
uv run python rdma_exporter.py &
EXPORTER_PID=$!
sleep 2

# Verify metrics endpoint
curl -s http://localhost:9915/metrics | grep rdma_port_xmit_data
# Expected:
# # HELP rdma_port_xmit_data RDMA counter: port_xmit_data
# # TYPE rdma_port_xmit_data gauge
# rdma_port_xmit_data{device="rxe0",port="1"} 1.8349260800e+08
```

---

## 24.7 Soft-IB (rdma_rxe) as a Lab Stand-In

`rdma_rxe` implements a full IB transport layer in software on top of any standard Ethernet interface. This is invaluable for CI/CD lab work but has important limitations relative to real HCAs.

### 24.7.1 Capabilities

| Feature | rdma_rxe | Real HCA (ConnectX-7) |
|---|---|---|
| IB verbs (RC, UD, UC) | Yes | Yes |
| RDMA WRITE/READ | Yes | Yes |
| Atomic operations | Yes (software) | Yes (hardware) |
| GRH/RoCEv2 encapsulation | Yes | Yes |
| OpenSM management | Yes (limited) | Full |
| GPUDirect RDMA | No | Yes |
| SR-IOV virtual functions | No | Yes |
| Hardware timestamping | No | Yes |
| Line-rate throughput | No (CPU-limited, ~10 Gbps loopback) | 400 Gbps |
| Kernel bypass (user-space DMA) | No | Yes (via IOMMU) |
| DC transport (mlx5-specific) | No | Yes |

### 24.7.2 Known Limitations for Lab Work

1. **SM interaction is incomplete.** OpenSM running against rxe devices will complete subnet discovery and assign LIDs, but switch programming (LFT push) is a no-op because rxe has no simulated switch.
2. **PFC is not emulated.** Congestion-related tests (DCQCN, PFC watchdog) must use real hardware or a network emulator (ns-3, see Chapter 23).
3. **ibdiagnet reports warnings.** `ibdiagnet` flags rxe as a "virtual device" and disables some checks. This is cosmetic — error detection still works.
4. **Throughput is software-limited.** All packet processing runs in kernel softirq context. Multi-QP tests will saturate a CPU core well before saturating a 1GbE NIC.

Despite these limitations, rxe is sufficient for:
- SM election and failover testing
- `ibnetdiscover` / `ibdiagnet` workflow practice
- `perfquery` counter collection and Prometheus integration
- Basic verbs correctness testing (bandwidth, latency functional validation)

---

## Lab Walkthrough 24 — OpenSM Subnet Manager + Fabric Diagnostics (Soft-IB rxe)

This walkthrough creates a two-node InfiniBand fabric using Soft-IB on a single Ubuntu 24.04 host, starts OpenSM, runs the full diagnostic suite, collects performance counters, injects a fault, and observes ibdiagnet's error detection.

**Prerequisites:** All packages installed per the Installation section above. The host must have the `rdma_rxe` module available.

### Step 1: Load the rxe module and create a veth pair

```bash
# Ensure the module is available
sudo modprobe rdma_rxe

lsmod | grep rdma_rxe
# Expected:
# rdma_rxe              196608  0

# Create a virtual Ethernet pair to simulate two physically-connected ports
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth0 up
sudo ip link set veth1 up
sudo ip addr add 10.20.0.1/30 dev veth0
sudo ip addr add 10.20.0.2/30 dev veth1

# Verify connectivity
ping -c 2 -W 1 10.20.0.2
# Expected:
# 2 packets transmitted, 2 received, 0% packet loss, time 1ms
# rtt min/avg/max/mdev = 0.048/0.052/0.056/0.004 ms
```

### Step 2: Attach rxe devices to both veth interfaces

```bash
# Create two Soft-IB devices, one per veth interface
sudo rdma link add rxe0 type rxe netdev veth0
sudo rdma link add rxe1 type rxe netdev veth1

# Verify both devices appear
rdma link show
# Expected:
# link rxe0/1 state ACTIVE physical_state POLLING netdev veth0
# link rxe1/1 state ACTIVE physical_state POLLING netdev veth1

# Confirm ibverbs sees both
ibv_devices
# Expected:
# device          node GUID
# ------          ---------
# rxe1            aabbccddeeff0002
# rxe0            aabbccddeeff0001
```

### Step 3: Start OpenSM on the first rxe device

```bash
# Get the port GUID for rxe0 port 1
RXE0_GUID=$(ibstat rxe0 | grep "Port GUID" | awk '{print $3}')
echo "rxe0 port GUID: $RXE0_GUID"
# Expected (GUID will vary):
# rxe0 port GUID: 0xaabbccddeeff0001

# Start OpenSM with verbose logging, binding to rxe0
sudo opensm -g "$RXE0_GUID" -v -f /tmp/opensm.log &
OPENSM_PID=$!
sleep 3

# Observe SM election in the log
grep -E "MASTER|DISCOVERY|SUBNET UP|LID" /tmp/opensm.log | head -20
# Expected:
# May 14 11:00:01 ...  Entering DISCOVERY state
# May 14 11:00:01 ...  Discovered 2 CA port(s)
# May 14 11:00:01 ...  Assigning LIDs...
# May 14 11:00:01 ...  rxe0 port 1: LID 1 assigned
# May 14 11:00:01 ...  rxe1 port 1: LID 2 assigned
# May 14 11:00:01 ...  Entering MASTER state
# May 14 11:00:01 ...  SUBNET UP
```

### Step 4: Verify port state with ibstat and ibstatus

```bash
# ibstatus — brief summary
ibstatus
# Expected:
# Infiniband device 'rxe0' port 1 status:
#         default gid:     fe80::aabb:ccff:fedd:eeff
#         base lid:        0x1
#         sm lid:          0x1
#         state:           4: ACTIVE
#         phys state:      5: LinkUp
#         rate:            10 Gb/sec (4X QDR)
#         link_layer:      InfiniBand
#
# Infiniband device 'rxe1' port 1 status:
#         default gid:     fe80::aabb:ccff:fedd:eef0
#         base lid:        0x2
#         sm lid:          0x1
#         state:           4: ACTIVE
#         phys state:      5: LinkUp
#         rate:            10 Gb/sec (4X QDR)
#         link_layer:      InfiniBand

# ibstat — detailed
ibstat rxe0
# Expected to show:
#   State: Active
#   Base lid: 1
#   SM lid: 1       <- SM is running and has configured this port
```

### Step 5: Run ibnetdiscover — full topology dump

```bash
ibnetdiscover --show 2>/dev/null
# Expected output (2-CA fabric with no switches):
#
# Topology file: generated on ...
# Max hops discovered: 1
#
# vendid=0x0 devid=0x0
# Ca    1 "H-aabbccddeeff0001"      # H- = HCA node
# [1](aabbccddeeff0001)
# vendid=0x0 devid=0x0
# Ca    1 "H-aabbccddeeff0002"
# [1](aabbccddeeff0002)
#
# Total nodes discovered: 2

# Save topology for later comparison (drift detection)
ibnetdiscover > /tmp/fabric_baseline.topo
echo "Baseline topology saved: $(wc -l < /tmp/fabric_baseline.topo) lines"
```

### Step 6: Run ibdiagnet — error scan

```bash
sudo ibdiagnet 2>&1
# Expected output on a clean fabric:
#
# Loading IBDIAGNET plugins ... Done
# -W- IBDiagnet is not meant to run on Virtual RDMA devices (informational warning only)
# Discovering fabric ... 2 nodes discovered.
#
# -- Link Checks --
# No bad links found.
#
# -- SM Checks --
# SM: LID 0x0001 GUID 0xaabbccddeeff0001 Priority 0 State Master (0x3)
#
# -- Error Checks --
# No errors found.
#
# -- Fabric Summary --
#   Total nodes      : 2
#   IB SWitches      : 0
#   Channel Adapters : 2
#   Routers          : 0
```

The key line is `No errors found.` — this is your fabric health baseline.

### Step 7: Read performance counters with perfquery

```bash
# Read all PM counters for rxe0 (LID 1, port 1)
perfquery 1 1
# Expected:
# # Port counters: Lid 1 port 1
# PortSelect:......................1
# CounterSelect:...................0xf000
# SymbolErrorCounter:..............0
# LinkErrorRecoveryCounter:........0
# LinkDownedCounter:...............0
# PortRcvErrors:...................0
# PortRcvRemotePhysicalErrors:.....0
# PortRcvSwitchRelayErrors:........0
# PortXmitDiscards:................0
# PortXmitConstraintErrors:........0
# PortRcvConstraintErrors:.........0
# LocalLinkIntegrityErrors:........0
# ExcessiveBufferOverrunErrors:....0
# VL15Dropped:.....................0
# PortXmitData:....................45824     # 4-byte units
# PortRcvData:.....................45824
# PortXmitPkts:....................47
# PortRcvPkts:.....................47

# Zero errors = healthy fabric. Non-zero PortRcvErrors or
# PortXmitDiscards = investigate immediately.

# Also read rxe1 (LID 2, port 1)
perfquery 2 1
```

### Step 8: Run ib_write_bw between the two rxe endpoints

The `perftest` suite is the standard InfiniBand verbs benchmark package, providing tools such as `ib_write_bw`, `ib_read_bw`, `ib_send_bw`, and `ib_write_lat` that exercise the full RDMA send/receive and RDMA write/read verb paths and report bandwidth and latency against real queue pairs. Since both rxe devices are on the same host, use background process control:

```bash
# Server on rxe0 (LID 1, IP 10.20.0.1)
ib_write_bw -d rxe0 -x 3 --port 18515 &
SERVER_PID=$!
sleep 1

# Client on rxe1, connecting to rxe0's IP
ib_write_bw -d rxe1 -x 3 --port 18515 10.20.0.1
# Expected output:
# ---------------------------------------------------------------------------------------
#                     RDMA_Write BW Test
#  Dual-port       : OFF          Device         : rxe1
#  Connection type : RC           Using SRQ      : OFF
#  Mtu             : 1024[B]
#  Link type       : InfiniBand
#  GID index       : 3
# ---------------------------------------------------------------------------------------
#  #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
#  65536      5000           9.84               9.71                 0.019
# ---------------------------------------------------------------------------------------

wait $SERVER_PID 2>/dev/null

# Re-read perfquery counters — xmit/rcv data should now be non-zero
perfquery 1 1 | grep -E "PortXmit|PortRcv"
# Expected (values much larger than Step 7):
# PortRcvErrors:...................0         <- still clean
# PortXmitData:....................183492608
# PortRcvData:.....................183492608
# PortXmitPkts:....................2797
# PortRcvPkts:.....................2797
```

### Step 9: Inject a fault — MTU mismatch, then re-run ibdiagnet

Simulate a misconfigured MTU on one side of the link:

```bash
# Reduce the MTU on veth1 to create a mismatch
sudo ip link set veth1 mtu 1000

# Verify MTU mismatch
ip link show veth0 | grep mtu   # still 1500
ip link show veth1 | grep mtu   # now 1000

# Re-run ibdiagnet — it should detect the MTU issue
sudo ibdiagnet 2>&1
# Expected (with MTU mismatch):
# -W- IBDiagnet is not meant to run on Virtual RDMA devices
# Discovering fabric ... 2 nodes discovered.
#
# -- Link Checks --
# -E- MTU mismatch on link between:
#     0xaabbccddeeff0001 port 1 MTU: 1024
#     0xaabbccddeeff0002 port 1 MTU: 512
#
# -- Error Checks --
# Found 1 error(s). See log for details.

# The MTU mismatch causes active_mtu to drop on the affected port
ibv_devinfo rxe1 | grep -E "max_mtu|active_mtu"
# Expected:
#   max_mtu:        4096 (5)
#   active_mtu:     512 (2)    <- degraded from 1024 to match lower side

# Restore correct MTU
sudo ip link set veth1 mtu 1500

# Re-run ibdiagnet — errors should clear
sudo ibdiagnet 2>&1 | grep -E "error|Error|MTU"
# Expected:
# No errors found.
```

### Step 10: Cleanup

```bash
# Kill OpenSM
sudo kill $OPENSM_PID 2>/dev/null
wait $OPENSM_PID 2>/dev/null

# Remove rxe devices
sudo rdma link delete rxe0
sudo rdma link delete rxe1

# Remove veth pair
sudo ip link del veth0
# veth1 is automatically removed when veth0 is deleted

# Unload module if desired
sudo modprobe -r rdma_rxe

# Verify all cleaned up
rdma link show
# Expected: (empty)
ip link show veth0 2>&1
# Expected: Device "veth0" does not exist.

echo "Cleanup complete"
```

---

## Summary

- InfiniBand differs from RoCEv2 in its use of a centralized Subnet Manager (OpenSM) to program forwarding tables and assign LIDs; RoCEv2 relies on standard IP routing and requires no SM.
- OpenSM is configured via `/etc/opensm/opensm.conf`; SM priority and failover are built into the IB protocol — the highest-priority SM wins MASTER election and standby SMs monitor it continuously.
- `ibnetdiscover` enumerates fabric topology via directed-route MADs; `ibdiagnet` performs comprehensive health checks including link state, MTU consistency, and error counter analysis.
- Fat-tree fabrics use FTREE routing in OpenSM to compute maximally load-balanced paths across all uplinks; LASH provides deadlock freedom in irregular topologies; DOR is preferred for torus networks.
- InfiniBand's credit-based flow control provides hardware-guaranteed lossless delivery; RoCEv2 on Ethernet requires PFC + DCQCN to replicate this property, with PFC deadlock as the failure mode to guard against.
- `perfquery` reads 64-bit PM counters via MADs; key health indicators are zero `PortRcvErrors`, `PortXmitDiscards`, and `ExcessiveBufferOverrunErrors`.
- `rdma_rxe` (Soft-IB) is a complete lab stand-in for verbs and SM workflow testing; it cannot replicate hardware-rate throughput, GPUDirect, SR-IOV, or hardware timestamping.

---

## References

- [OpenSM Subnet Manager](https://linux.die.net/man/8/opensm)
- [OpenSM source (linux-rdma)](https://github.com/linux-rdma/opensm)
- [rdma-core (libibverbs, ibverbs-utils)](https://github.com/linux-rdma/rdma-core)
- [ibutils2 (Mellanox diagnostics)](https://github.com/Mellanox/ibutils2)
- [OFED / MLNX_OFED](https://docs.nvidia.com/networking/display/mlnxofedv24)
- [UFM (Unified Fabric Manager)](https://docs.nvidia.com/networking/display/UFMEnterpriseUFMAppliance)
- [InfiniBand diagnostic tools (ibdiagnet, ibtracert, perfquery)](https://docs.nvidia.com/networking/display/ofedv522040)
- [perftest (RDMA performance benchmarks)](https://github.com/linux-rdma/perftest)
- [Soft-IB / rdma_rxe kernel module](https://github.com/linux-rdma/rdma-core/tree/master/kernel-headers)
- [RoCEv2 (RDMA over Converged Ethernet)](https://www.infinibandta.org/roce/)
- [IEEE 802.1Qbb — Priority-based Flow Control](https://standards.ieee.org/ieee/802.1Qbb/4selfpass/)
- [SONiC Foundation](https://sonicfoundation.dev)
- [Prometheus (prometheus_client)](https://prometheus.io/docs/introduction/overview/)
- [Wireshark](https://www.wireshark.org)
- Pfister, G.F., *An Introduction to the InfiniBand Architecture*, High Performance Mass Storage and Parallel I/O, 2001 — [doi.org/10.1016/B978-012651871-7/50022-4](https://doi.org/10.1016/B978-012651871-7/50022-4)
- de Sensi et al., *A Comprehensive Analysis of InfiniBand Congestion Control*, IEEE IPDPS 2019 — [ieeexplore.ieee.org/document/8820989](https://ieeexplore.ieee.org/document/8820989)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).