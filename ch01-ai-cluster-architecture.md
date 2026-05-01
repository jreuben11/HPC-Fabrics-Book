# Chapter 1 — Architecture of the AI Cluster Network

**Part I: Foundations** | ~15 pages

---

## Introduction

AI compute clusters are, at their core, distributed memory systems in which network bandwidth and latency are as critical to performance as **GPU** compute throughput. This chapter establishes the architectural vocabulary and first-principles reasoning that every subsequent chapter builds upon. Understanding why the network is a first-class citizen of the AI training stack — not merely a plumbing detail — is the prerequisite for every design decision that follows.

We begin with the physical topology choices that practitioners face when building **GPU** clusters, from the classical **fat-tree**/**Clos** fabric to the **rail-optimized** variants now standard in production hyperscale deployments. These choices determine bisection bandwidth, **ECMP** hash collision risk, and the physical routing of collective traffic. Getting topology wrong at design time is expensive to correct; getting it right means the network becomes invisible during training, which is exactly what it should be.

The chapter then maps the three dominant parallelism strategies — data parallelism, pipeline parallelism, and tensor parallelism — to their distinct traffic signatures. Each parallelism type imposes different bandwidth, latency, and flow-count requirements on the fabric. A data-parallel **AllReduce** is an all-to-all barrier; pipeline parallelism is point-to-point and latency-dominated; tensor parallelism demands ultra-low latency within a small group, typically served by **NVLink** within a node. Designing a single fabric to serve all three simultaneously is the central engineering challenge of the AI cluster network.

We close with a quantitative bandwidth and latency reference table spanning **NVLink** through cross-fabric paths, and a layer-by-layer map of the full stack covered by this book. The lab walkthrough provisions a minimal two-spine, two-leaf **fat-tree** stub using **Containerlab** and **SR Linux**, then demonstrates **ECMP** path distribution using **iperf3** — giving concrete, hands-on grounding for the topological concepts in the chapter.

Adjacent chapters build directly on this foundation: Chapter 2 dives into the **RDMA** transport layer that carries collective traffic, Chapter 3 addresses clock synchronization across cluster nodes, and Chapter 4 covers the **UCX**/**LibFabric** abstraction libraries that sit between collective runtimes and the physical transport.

---

## Installation

This section installs **Docker** and **Containerlab**, which together provide the container runtime and topology orchestration engine needed to instantiate the **fat-tree** lab fabric. The **SR Linux** container image is pulled as the switch OS for each spine and leaf node, because it supports the **ECMP**-capable **BGP** and IP fabric configuration exercised in the walkthrough. The **iperf3** package provides the traffic generation tool used to drive multi-flow load across the topology and observe how the fabric distributes flows across **ECMP** paths.

### System Packages (Ubuntu 24.04)

```bash
sudo apt update
sudo apt install -y docker.io iperf3 iproute2 curl

# Start and enable Docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

### Containerlab

Containerlab is not in the Ubuntu apt repository; install via the official installer script:

```bash
# Install Containerlab via the official script
bash -c "$(curl -sL https://get.containerlab.dev)"

# Verify installation
containerlab version
# Expected output: Containerlab version X.Y.Z

# Verify SR Linux container image is accessible
docker pull ghcr.io/nokia/srlinux:latest
docker images ghcr.io/nokia/srlinux
```

### iperf3

- https://github.com/esnet/iperf 

```bash
# Already installed via apt above; verify
iperf3 --version
# Expected: iperf 3.x (cJSON 1.x.x)

# For container-based testing, pull a lightweight iperf3 image
docker pull networkstatic/iperf3
docker run --rm networkstatic/iperf3 --version
```

### Verify Prerequisites

```bash
# All tools available
which containerlab docker iperf3 ip
containerlab version
docker info | grep -E "Server Version|Cgroup"
```

---

## 1.1 Why the Network Is the Bottleneck

A modern GPU training cluster is, at its core, a distributed memory system. Each GPU has fast local **HBM** (High Bandwidth Memory, the on-package DRAM stack on modern accelerators), but no single GPU holds the full model. Training requires frequent synchronization — parameter gradients must be aggregated across every rank after each backward pass. At scale, a 1000-GPU job running **AllReduce** (a collective operation that sums tensors across all participating GPUs and distributes the result back to each) over a 70B-parameter model moves hundreds of gigabytes of gradient data per step. If the network cannot sustain that bandwidth at low latency, GPUs stall waiting for data. Compute utilization collapses.

The implication: the network is not infrastructure that passively moves bits. It is a first-class component of the training system, and its topology, bandwidth profile, congestion behavior, and latency tail all directly determine model throughput.

---

## 1.2 Physical Topology Choices

### 1.2.1 Fat-Tree (Clos) Networks

The dominant topology in hyperscale AI clusters is the **fat-tree**, a specific realization of a **folded** Clos network. A k-ary fat-tree has three layers of switches — edge (**ToR** {Top of Rack}), aggregation, and core — and supports k³/4 hosts with full bisection bandwidth.

```
                 [Core switches]
                /       |       \
        [Agg]  [Agg]  [Agg]  [Agg]
         |  \   |  \   |  \   |  \
        [ToR][ToR][ToR][ToR][ToR][ToR]
         |         |         |
       [hosts]   [hosts]   [hosts]
```

**Clos network**: a multi-stage, non-blocking network topology originaly designed by Charles Clos in 1952 to optimize telephony switches. It organizes switches into three stages (ingress, middle, egress) to provide any-to-any connectivity using smaller switches, reducing total crosspoints. Modern data centers use this architecture, often as a **two-tier "spine-leaf" (folded) design**, to deliver high-performance, low-latency communication with **equal-cost multipath (ECMP)** routing

**Equal-Cost Multi-Path (ECMP)**: a routing strategy that allows a router to use multiple best paths, having identical metrics, for forwarding packets to a single destination. It load-balances traffic, increasing bandwidth, redundancy, and fault tolerance. Typically, it uses **per-flow hashing** to ensure in-order delivery, compatible with protocols like OSPF and BGP.

**Per-flow hashing**: a network traffic distribution technique that maps specific data flows (defined by source/destination IP, port, and protocol) to the same physical link or path (**LAG/ECMP**). This ensures packet ordering within a flow while balancing, providing efficient utilization across multiple paths. It is used for load balancing and in DPDK applications for fast lookup.

**Full bisection bandwidth** means any permutation of host-to-host communication can run at line rate simultaneously. For AllReduce, this is the ideal — no traffic pattern is privileged. The cost is switch count: a k=48 fat-tree needs 27 spine switches, 27 aggregation switches, and scales to ~1728 servers.

**Key parameter:** k (**radix** of each switch). Higher radix reduces the number of tiers and hop count but requires higher port-count ASICs (e.g., Tomahawk4 at 512×100G ports). Modern clusters use 400G or 800G links to reduce port count.

### 1.2.2 Rail-Optimized Fabric

For GPU clusters, a variant called the **rail-optimized** (or GPU-rail) fabric has become standard. The insight: a multi-GPU server (e.g., 8× H100) has 8 NICs, one per GPU. Rather than connecting all 8 NICs to the same ToR, each NIC connects to a *different* ToR in a set of 8 "rails."

```bash
GPU0 ── Rail0-ToR0 ──┐
GPU1 ── Rail1-ToR0 ──┤
GPU2 ── Rail2-ToR0 ──┤   #All Rail0 ToRs connect to Rail0 Spine
GPU3 ── Rail3-ToR0 ──┘
...
```

In a ring-AllReduce, each GPU only communicates with GPUs on the same rail (same GPU index across different servers). Rail-optimized fabric ensures that each GPU's traffic stays on a single set of uplinks, avoiding hash collisions on shared ECMP paths that would cause head-of-line blocking between flows of the same collective.

**Trade-off:** Rail-optimized fabric sacrifices generality (non-AllReduce traffic patterns suffer) for AllReduce optimality. It is a deliberate match between the collective algorithm and the physical topology.

### 1.2.3 Dragonfly and Hypercube

At extreme scale (>100,000 GPUs), pure fat-trees become impractical due to cable density and switch count. **Dragonfly** topologies reduce global link count by accepting that some traffic must traverse multiple hops through "groups" of all-to-all connected switches. **Hypercube** topologies (used in some TPU pod networks) offer predictable bisection at the cost of complex routing. Both are beyond typical on-prem deployments but appear in NIC/ASIC vendor roadmaps.

**Dragonfly topology**: a hierarchical, high-radix interconnection network designed to minimize network diameter and cost in large-scale HPC and AI systems by grouping switches in a fully connected mesh. It uses local links for intra-group connections and expensive long-distance global cables to connect all groups, typically achieving low latency in only three hops

**Hypercube Topology**: a network structure used in parallel computing, consisting of \(N=2^n\) nodes arranged in \(n\) dimensions. Each processor connects directly to \(n\) neighbors, with node addresses differing by exactly one bit. It features a low diameter (\(\log _{2}N\)) and high fault tolerance, making it efficient for parallel algorithms and message-passing
---

## 1.3 Traffic Pattern Taxonomy

Understanding what traffic flows over the fabric at what volume and frequency is the design input for every subsequent chapter.

### 1.3.1 AllReduce (Data Parallelism)

In **Data Parallel** training, each GPU holds a full copy of the model and a **shard of the batch**. After the **backward pass**, **gradients** must be summed across all GPUs. This is AllReduce.

- **Volume:** 2 × model_size × (N-1)/N bytes per step (**ring-AllReduce** algorithm)
- **Pattern:** All-to-all; every rank both sends and receives; traffic is balanced
- **Frequency:** Once per training step (typically every 100–500 ms for large models)
- **Latency sensitivity:** Moderate — the GPU waits for AllReduce to complete before the next forward pass begins

For a 70B-parameter model in bf16 (2 bytes/param): ~140 GB of gradient data per step. At 400 Gbps NIC bandwidth and ring-AllReduce efficiency: ~2.8 seconds of network time per step (before any compute overlap).

### 1.3.2 Pipeline Parallelism

In **Pipeline Parallel** training, different layers of the model reside on different GPUs. Microbatches flow forward and backward through the pipeline in stages.

- **Volume:** Activation tensor size × pipeline depth
- **Pattern:** Point-to-point between adjacent pipeline stages; highly directional
- **Frequency:** High (every microbatch boundary)
- **Latency sensitivity:** High — pipeline bubbles directly reduce GPU utilization

Pipeline traffic is typically within a node (NVLink, NVIDIA's proprietary high-bandwidth GPU-to-GPU interconnect with up to 900 GB/s aggregate bandwidth on H100 systems) or between adjacent nodes. It demands low *latency* more than raw bandwidth.

### 1.3.3 Tensor Parallelism

In **Tensor Parallel** training, individual **weight matrices are sharded** across GPUs. Every forward pass requires AllReduce within the tensor-parallel group.

- **Volume:** Small (shard of activation) but frequent
- **Pattern:** All-to-all within a small group (typically 4–8 GPUs, often within a node)
- **Latency sensitivity:** Very high — tensor-parallel AllReduce is on the critical path of every transformer layer

Tensor parallelism traffic is usually handled over NVLink (intra-node) specifically because cross-node latency would make it uneconomical.

### 1.3.4 Checkpoint I/O and Dataset Streaming

- **Checkpoint:** Periodic dump of all model parameters + optimizer state to distributed storage. 70B model at fp32+Adam: ~800 GB. Must complete quickly to minimize GPU idle time.
- **Dataset streaming:** Training data loaded from object/parallel storage throughout training. Must sustain the training step rate without stalling the data pipeline.

Both are storage-fabric concerns (Chapter 18) but their bandwidth demands must be provisioned separately from the collective traffic — checkpoints can saturate storage uplinks and starve gradient synchronization if not isolated.

---

## 1.4 ECMP Hashing and Flow Collisions

Fat-tree fabrics use **Equal-Cost Multi-Path (ECMP)** routing to spread traffic across parallel paths. ECMP hashes each flow (typically on 5-tuple: src IP, dst IP, proto, src port, dst port) to one of N uplinks.

The problem for AI traffic: **NCCL** (NVIDIA Collective Communications Library, the dominant GPU collective runtime) collective operations use a small number of long-lived flows between the same pairs of endpoints. With only 8 GPU flows from a server, ECMP hashing easily produces collisions — multiple flows hash to the same uplink, leaving others idle.

**Mitigations:**
- **ECMP with flowlet switching**: re-hash after a burst gap, spreading **elephant flows** across paths dynamically
- **Per-packet load balancing (DLRS, Dynamic Load-balancing with Resequencing)**: hash each *packet* rather than each *flow*; requires reorder tolerance at the receiver (**RoCEv2 is not reorder-tolerant** — this only applies to TCP flows)
- **Rail-optimized topology** (Section 1.2.2): eliminate the collision problem by ensuring each GPU's flows land on a dedicated set of uplinks
- **NCCL topology files**: allow NCCL to express which GPU pairs communicate, enabling deterministic NIC selection

---

## 1.5 Bandwidth and Latency Requirements

| Tier | Typical link | Round-trip latency |
|---|---|---|
| NVLink (intra-GPU, H100) | 900 GB/s | ~1 µs |
| NVLink (intra-node) | 450 GB/s bidirectional | ~2 µs |
| InfiniBand HDR (NIC-to-ToR) | 200 Gbps | ~1–2 µs |
| RoCEv2 over Ethernet (NIC-to-ToR) | 400 Gbps | ~2–5 µs |
| ToR-to-Spine (fat-tree tier 1) | 400 Gbps | +1–2 µs |
| Cross-fabric (any node to any node) | limited by bisection | ~5–15 µs |

**Tail latency matters more than mean latency.** AllReduce is a barrier: every rank waits for the slowest. A p99 latency of 50 µs means 1% of steps stall the entire cluster for that duration.

---

## 1.6 The Full Stack, Layer by Layer

The remainder of this book addresses each layer of the following stack:

```
┌───────────────────────────────────────────────────┐
│  Distributed Training Runtime (PyTorch/DeepSpeed) │  Ch 20 (brief)
├───────────────────────────────────────────────────┤
│  GPU Collective Library (NCCL / RCCL)             │  Ch 19 (brief)
├───────────────────────────────────────────────────┤
│  Interconnect Abstraction (UCX / LibFabric)       │  Ch 4
├───────────────────────────────────────────────────┤
│  RDMA Transport (rdma-core / RoCEv2 / IB verbs)   │  Ch 2
├───────────────────────────────────────────────────┤
│  Kernel-Bypass I/O (DPDK / SPDK / eBPF-XDP)       │  Ch 5, 6, 7
├───────────────────────────────────────────────────┤
│  Kubernetes / Container Networking (Cilium/SR-IOV)│  Ch 12, 13
├───────────────────────────────────────────────────┤
│  Programmable Fabric (SONiC / SR Linux / P4)      │  Ch 8, 9, 10
├───────────────────────────────────────────────────┤
│  Overlay & Routing (BGP-EVPN / VXLAN / FRR)       │  Ch 11
├───────────────────────────────────────────────────┤
│  Management & Telemetry (YANG / gNMI / OTel)      │  Ch 14–17
├───────────────────────────────────────────────────┤
│  Storage Fabric (Ceph / Lustre / DAOS)            │  Ch 18
├───────────────────────────────────────────────────┤
│  Time Synchronization (PTP / linuxptp)            │  Ch 3
└───────────────────────────────────────────────────┘
```

---

## Lab Walkthrough 1 — Containerlab 2-Spine/2-Leaf ECMP Topology

This walkthrough stands up a minimal fat-tree stub (2 spine switches, 2 leaf switches, 4 iperf3 host containers) using Containerlab (a container-based network topology emulator that provisions virtual switches and links inside Docker) with SR Linux nodes (Nokia's open, containerized network operating system), then demonstrates ECMP path distribution. iperf3 is a widely used network throughput testing tool that measures achievable bandwidth between a client and server.

**Prerequisites:** Docker running, Containerlab installed, SR Linux image pulled (all covered in the Installation section above).

### Step 1: Verify the environment

```bash
# Confirm Docker daemon is up and you are in the docker group
docker ps
# Expected: empty table header (no error means Docker is accessible)

# Confirm Containerlab is installed
containerlab version
# Expected output (version numbers will vary):
#   ____ ___  _   _ _____ _    ___ _   _ _____ ____
#  / ___/ _ \| \ | |_   _/ \  |_ _| \ | | ____|  _ \
# | |  | | | |  \| | | |/ _ \  | ||  \| |  _| | |_) |
# | |__| |_| | |\  | | / ___ \ | || |\  | |___| _ <
#  \____\___/|_| \_| |_/_/   \_\___|_| \_|_____|_| \_\
#
#   version: 0.x.y

# Confirm SR Linux image is present
docker images ghcr.io/nokia/srlinux --format "{{.Repository}}:{{.Tag}}\t{{.Size}}"
# Expected: ghcr.io/nokia/srlinux:latest    ~1.2GB
```

### Step 2: Create the Containerlab topology file

Create a working directory and write the topology definition:

```bash
mkdir -p ~/clab-ecmp-lab && cd ~/clab-ecmp-lab
```

Write the following to `~/clab-ecmp-lab/ecmp-topo.yml`:

```yaml
name: ecmp-lab

topology:
  kinds:
    srl:
      image: ghcr.io/nokia/srlinux:latest
    linux:
      image: networkstatic/iperf3

  nodes:
    # Spine switches
    spine1:
      kind: srl
      type: ixrd2l
    spine2:
      kind: srl
      type: ixrd2l

    # Leaf switches
    leaf1:
      kind: srl
      type: ixrd2l
    leaf2:
      kind: srl
      type: ixrd2l

    # Host containers
    host1:
      kind: linux
    host2:
      kind: linux
    host3:
      kind: linux
    host4:
      kind: linux

  links:
    # Leaf1 uplinks to both spines
    - endpoints: ["leaf1:e1-1", "spine1:e1-1"]
    - endpoints: ["leaf1:e1-2", "spine2:e1-1"]

    # Leaf2 uplinks to both spines
    - endpoints: ["leaf2:e1-1", "spine1:e1-2"]
    - endpoints: ["leaf2:e1-2", "spine2:e1-2"]

    # Host downlinks to leaves
    - endpoints: ["host1:eth1", "leaf1:e1-3"]
    - endpoints: ["host2:eth1", "leaf1:e1-4"]
    - endpoints: ["host3:eth1", "leaf2:e1-3"]
    - endpoints: ["host4:eth1", "leaf2:e1-4"]
```

### Step 3: Deploy the topology

```bash
cd ~/clab-ecmp-lab
sudo containerlab deploy --topo ecmp-topo.yml
```

Expected output (truncated):

```bash
INFO[0000] Containerlab v0.x.y started
INFO[0000] Parsing & checking topology file: ecmp-topo.yml
INFO[0001] Creating lab directory: /root/clab-ecmp-lab/clab-ecmp-lab
INFO[0002] Creating docker network: Name="clab", IPv4Subnet="172.20.20.0/24"
INFO[0003] Creating container: "spine1"
INFO[0003] Creating container: "spine2"
INFO[0004] Creating container: "leaf1"
INFO[0004] Creating container: "leaf2"
INFO[0005] Creating container: "host1"
...
INFO[0015] Adding containerlab host entries to /etc/hosts file
+---+------------------+--------------+-------------------------------+---------------+---------+
| # |       Name       | Container ID |             Image             |     Kind      |  State  |
+---+------------------+--------------+-------------------------------+---------------+---------+
| 1 | clab-ecmp-lab-host1  | abc123   | networkstatic/iperf3          | linux         | running |
| 2 | clab-ecmp-lab-host2  | def456   | networkstatic/iperf3          | linux         | running |
| 3 | clab-ecmp-lab-host3  | ghi789   | networkstatic/iperf3          | linux         | running |
| 4 | clab-ecmp-lab-host4  | jkl012   | networkstatic/iperf3          | linux         | running |
| 5 | clab-ecmp-lab-leaf1  | mno345   | ghcr.io/nokia/srlinux:latest  | srl           | running |
| 6 | clab-ecmp-lab-leaf2  | pqr678   | ghcr.io/nokia/srlinux:latest  | srl           | running |
| 7 | clab-ecmp-lab-spine1 | stu901   | ghcr.io/nokia/srlinux:latest  | srl           | running |
| 8 | clab-ecmp-lab-spine2 | vwx234   | ghcr.io/nokia/srlinux:latest  | srl           | running |
+---+------------------+--------------+-------------------------------+---------------+---------+
```

### Step 4: Verify all containers are running

```bash
docker ps --filter "label=clab-node-name" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Expected: all 8 containers show `Up N seconds` with no `Exited` entries.

### Step 5: Inspect SR Linux interface state on leaf1

```bash
# SSH into leaf1 (Containerlab sets up SSH with default credentials admin/NokiaSrl1!)
ssh admin@clab-ecmp-lab-leaf1

# Inside SR Linux CLI https://learn.srlinux.dev/get-started/cli/ , show interface summary
show interface brief
```

Expected output:

```
+-----------+----------+-----+------+------------------+
| Interface | Oper     | Adm | MTU  | Description      |
+-----------+----------+-----+------+------------------+
| ethernet-1/1 | up    | up  | 9232 | (to spine1)      |
| ethernet-1/2 | up    | up  | 9232 | (to spine2)      |
| ethernet-1/3 | up    | up  | 9232 | (to host1)       |
| ethernet-1/4 | up    | up  | 9232 | (to host2)       |
+-----------+----------+-----+------+------------------+
```

Exit the SR Linux CLI:

```
quit
```

### Step 6: Configure IP addresses and static routes on SR Linux nodes

For each leaf/spine, configure subinterfaces with IP addresses. On leaf1:

```bash
ssh admin@clab-ecmp-lab-leaf1
```

Inside SR Linux:

```bash
enter candidate
    /interface ethernet-1/1 subinterface 0 ipv4 address 10.0.1.1/30
    /interface ethernet-1/2 subinterface 0 ipv4 address 10.0.2.1/30
    /interface ethernet-1/3 subinterface 0 ipv4 address 192.168.1.1/24
    /interface ethernet-1/4 subinterface 0 ipv4 address 192.168.2.1/24
    /network-instance default interface ethernet-1/1.0
    /network-instance default interface ethernet-1/2.0
    /network-instance default interface ethernet-1/3.0
    /network-instance default interface ethernet-1/4.0
commit now
```

Repeat analogously for leaf2, spine1, and spine2 with non-overlapping address ranges (10.0.3.x, 10.0.4.x, etc.).

### Step 7: Verify ECMP routes on leaf1

After configuring routing (static or BGP), inspect the route table:

```bash
ssh admin@clab-ecmp-lab-leaf1
show network-instance default route-table ipv4-unicast prefix 192.168.3.0/24
```

Expected output showing two equal-cost next-hops (ECMP across spine1 and spine2):

```bash
------------------------------------------------------------------------
Prefix            : 192.168.3.0/24
Route type        : static
Active            : true
Metric            : 0
Preference        : 5
Next-hops:
  10.0.1.2  via ethernet-1/1.0  [ECMP]
  10.0.2.2  via ethernet-1/2.0  [ECMP]
------------------------------------------------------------------------
```

Also verify via the Linux `ip` command on the host side:

```bash
docker exec clab-ecmp-lab-host1 ip route show
# Expected:
# default via 192.168.1.1 dev eth1
# 192.168.1.0/24 dev eth1 proto kernel scope link src 192.168.1.10
```

### Step 8: Assign IP addresses to host containers

```bash
# host1
docker exec clab-ecmp-lab-host1 ip addr add 192.168.1.10/24 dev eth1
docker exec clab-ecmp-lab-host1 ip route add default via 192.168.1.1

# host3 (on leaf2's subnet)
docker exec clab-ecmp-lab-host3 ip addr add 192.168.3.10/24 dev eth1
docker exec clab-ecmp-lab-host3 ip route add default via 192.168.3.1
```

Verify connectivity across the fabric (host1 to host3, crossing two spine paths):

```bash
docker exec clab-ecmp-lab-host1 ping -c 4 192.168.3.10
```

Expected:

```bash
PING 192.168.3.10 (192.168.3.10) 56(84) bytes of data.
64 bytes from 192.168.3.10: icmp_seq=1 ttl=62 time=0.8 ms
64 bytes from 192.168.3.10: icmp_seq=2 ttl=62 time=0.6 ms
64 bytes from 192.168.3.10: icmp_seq=3 ttl=62 time=0.7 ms
64 bytes from 192.168.3.10: icmp_seq=4 ttl=62 time=0.6 ms
--- 192.168.3.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

TTL=62 confirms two routing hops (leaf1 → spine → leaf2, decremented twice).

### Step 9: Run iperf3 bandwidth test across the fabric

Start iperf3 server on host3:

```bash
docker exec -d clab-ecmp-lab-host3 iperf3 -s -p 5201
```

Run iperf3 client from host1 to host3 with 4 parallel streams (to stress ECMP):

```bash
docker exec clab-ecmp-lab-host1 iperf3 -c 192.168.3.10 -p 5201 -P 4 -t 10 -i 2
```

Expected output:

```bash
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-2.00   sec   112 MBytes   470 Mbits/sec
[  5]   2.00-4.00   sec   114 MBytes   478 Mbits/sec
...
[SUM]   0.00-10.00  sec   446 MBytes   374 Mbits/sec  (sum of 4 streams)
```

Note: actual throughput is limited by the virtual Ethernet links inside Docker; the important observation is that traffic flows without error and that multiple streams share the two spine paths.

### Step 10: Observe ECMP path distribution

Check interface counters on spine1 to confirm traffic is flowing over it:

```bash
ssh admin@clab-ecmp-lab-spine1
show interface ethernet-1/1 statistics
```

Expected: rising `out-octets` and `in-octets` counters matching the iperf3 traffic volume. Compare `ethernet-1/1` (toward leaf1) vs `ethernet-1/2` (toward leaf2) — both should show traffic, confirming ECMP is distributing across both spines.

```
Interface : ethernet-1/1
  in-octets  : 23847293
  out-octets : 31029481
  in-unicast-packets  : 182340
  out-unicast-packets : 237821
```

### Step 11: Observe flow hashing with different source ports

Run two iperf3 clients simultaneously from host1 and host2 (both on leaf1) to host3 (on leaf2), using different source ports to produce different ECMP hashes:

```bash
# Client A — from host1
docker exec -d clab-ecmp-lab-host1 iperf3 -c 192.168.3.10 -p 5201 -t 20

# Client B — from host2 (different src IP → different hash)
docker exec clab-ecmp-lab-host2 ip addr add 192.168.2.10/24 dev eth1
docker exec clab-ecmp-lab-host2 ip route add default via 192.168.2.1
docker exec -d clab-ecmp-lab-host2 iperf3 -c 192.168.3.10 -p 5201 -t 20
```

While traffic runs, watch per-interface counters on both spines and confirm different flows are hashed to different uplinks:

```bash
# Poll counters every 2 seconds on spine1 and spine2
ssh admin@clab-ecmp-lab-spine1 'show interface ethernet-1/1 statistics | grep out-octets'
ssh admin@clab-ecmp-lab-spine2 'show interface ethernet-1/1 statistics | grep out-octets'
```

If ECMP hashing is working, you will see traffic on both spine interfaces. If both flows hash identically (collision), only one spine's counters will rise — this is the ECMP collision scenario described in Section 1.4.

### Step 12: Tear down the lab

When finished, destroy all containers and clean up:

```bash
cd ~/clab-ecmp-lab
sudo containerlab destroy --topo ecmp-topo.yml --cleanup
```

Expected:

```bash
INFO[0000] Destroying lab: ecmp-lab
INFO[0001] Removed container: clab-ecmp-lab-host1
INFO[0001] Removed container: clab-ecmp-lab-host2
...
INFO[0005] Removing containerlab host entries from /etc/hosts file
INFO[0005] Removing lab directory: /root/clab-ecmp-lab/clab-ecmp-lab
```

Verify all lab containers are gone:

```bash
docker ps --filter "label=clab-node-name"
# Expected: empty table (no containers listed)
```

---

## Summary

- Fat-tree / Clos topologies provide full bisection bandwidth; rail-optimized variants align physical paths with ring-AllReduce communication patterns.
- The three AI training parallelism strategies (data, pipeline, tensor) produce fundamentally different traffic profiles; designing for one without understanding the others produces fabric surprises.
- ECMP hashing collisions are a real source of AllReduce performance collapse; mitigations range from flowlet switching to topology-aware NIC assignment.
- Every subsequent chapter addresses one layer of the stack summarized in Section 1.6.

---

## References

- NVIDIA Networking: *Understanding the GPU Fabric for Large Language Model Training* — [developer.nvidia.com](https://developer.nvidia.com/networking)
- Gangidi et al., *RoCEv2 Congestion Management in Production AI Clusters*, NSDI 2024 — [usenix.org/conference/nsdi24](https://www.usenix.org/conference/nsdi24/technical-sessions)
- Al-Fares et al., *A Scalable, Commodity Data Center Network Architecture* (fat-tree/Clos paper), SIGCOMM 2008 — [dl.acm.org/doi/10.1145/1402946.1402967](https://dl.acm.org/doi/10.1145/1402946.1402967)
- Rashidi et al., *ASTRA-SIM: Enabling SW/HW Co-Design Exploration for Distributed DL Training Platforms*, ISPASS 2020 — [arxiv.org/abs/1910.04940](https://arxiv.org/abs/1910.04940)
- [Containerlab](https://containerlab.dev) — container-based network topology emulator
- [SR Linux](https://learn.srlinux.dev) — Nokia containerized network operating system
- [iperf3](https://iperf.fr) — network throughput testing tool
- [iproute2](https://wiki.linuxfoundation.org/networking/iproute2) — Linux IP routing and network configuration utilities
- [FRRouting (FRR)](https://frrouting.org) — open-source IP routing suite (BGP, OSPF, etc.)
- [NCCL](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/) — NVIDIA Collective Communications Library
- [UCX](https://openucx.org) — Unified Communication X transport abstraction
- [DPDK](https://doc.dpdk.org) — Data Plane Development Kit
- [linuxptp](https://linuxptp.sourceforge.net) — Linux PTP (IEEE 1588) implementation
- [NVLink](https://www.nvidia.com/en-us/data-center/nvlink/) — NVIDIA high-bandwidth GPU-to-GPU interconnect
- [NVIDIA BlueField DPU](https://www.nvidia.com/en-us/networking/products/data-processing-unit/) — NVIDIA data processing unit / SmartNIC
- [SONiC](https://sonic-net.github.io/SONiC/) — open-source network operating system
- [PyTorch](https://pytorch.org/docs/) — deep learning framework
- [DeepSpeed](https://www.deepspeed.ai/docs/) — deep learning optimization library
- [OpenTelemetry](https://opentelemetry.io/docs/) — observability framework for distributed traces and metrics
- [Ceph](https://docs.ceph.com/) — distributed storage system
- [Lustre](https://wiki.lustre.org/Main_Page) — parallel distributed file system
- [DAOS](https://docs.daos.io/) — Distributed Asynchronous Object Storage
- [Cilium](https://docs.cilium.io/) — eBPF-based Kubernetes networking and security
- [eBPF / XDP](https://ebpf.io/docs/) — kernel-bypass programmable packet processing
- [SPDK](https://spdk.io/doc/) — Storage Performance Development Kit
- [P4](https://p4.org/specs/) — programming language for data plane programmability
- [gNMI](https://github.com/openconfig/reference/tree/master/rpc/gnmi) — gRPC Network Management Interface


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).