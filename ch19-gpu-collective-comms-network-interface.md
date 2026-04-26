# Chapter 19 — GPU Collective Communications: Network Interface

**Part VI: Storage & Adjacent Infrastructure** | ~10 pages

> **Scope note:** This chapter covers NCCL, RCCL, and oneCCL only from the perspective of what they demand from and expose to the network fabric. The library internals, algorithm selection, and training framework integration are covered in the companion investigation on GPU collective communications.

---

## Introduction

Every modern large-scale AI training job is fundamentally a distributed program: hundreds or thousands of GPU processes, each holding a shard of the model and a different batch of training data, must periodically synchronize their gradient computations before the optimizer can take a step. This synchronization — the AllReduce collective operation — is the most network-intensive workload in all of computing. A single AllReduce step for a 175-billion-parameter model in bfloat16 precision moves 350 GB of data across the training fabric. The frequency is one per optimizer step, and the entire GPU cluster stalls until the AllReduce completes. Every microsecond of additional latency, every dropped packet that triggers retransmission, every ECMP hash collision that leaves half the fabric idle while the other half is saturated — all translate directly into degraded GPU utilization and longer time-to-train.

This chapter covers **NCCL** (NVIDIA Collective Communications Library), **RCCL** (AMD's drop-in equivalent for **ROCm**), and Intel **oneCCL** — but only from the perspective of what these libraries demand from and expose to the network fabric. The internal algorithms (ring-AllReduce, double-binary-tree), the **PyTorch** and **Megatron-LM** integration, and the optimizer step sequence are addressed in the companion investigation; here the focus is on the network engineer's questions: which NICs does **NCCL** select, how does it fall back between transports, what environment variables control fabric behavior, and how can the fabric operator diagnose a slow collective from the network side.

A key architectural insight is that **NCCL**'s transport is not hardwired. The plugin architecture allows **NCCL** to use **libibverbs** (for **InfiniBand** and **RoCEv2**), the **UCX** library (for any **UCX**-supported transport including shmem and CXL), or a TCP socket fallback — selected dynamically at initialization based on what hardware is available. This makes **NCCL**'s fabric requirements both flexible and transparent: an operator can observe exactly which transport was selected from the initialization log and tune accordingly.

**SHARP** (Scalable Hierarchical Aggregation and Reduction Protocol) represents the most aggressive fabric optimization available for **InfiniBand** clusters — offloading the AllReduce computation itself into the switch ASICs, reducing total fabric traffic by up to 330× for large collectives. Understanding when **SHARP** helps and what it requires from the switch infrastructure is essential for **InfiniBand** fabric design at scale.

This chapter connects directly to Chapter 2 (**RDMA**/**RoCEv2**, on which the primary **NCCL** transport is built), Chapter 4 (**UCX** and **libfabric**, which back the **UCX** and **oneCCL** transports), Chapter 24 (**InfiniBand** fabric management, which covers **SHARP** switch configuration), and Chapter 20 (distributed training runtimes, which quantifies the AllReduce traffic patterns that **NCCL** generates at different levels of parallelism).

---

## Installation

The **NCCL** library (**libnccl**) and the **nccl-tests** benchmark suite are the primary tools for this chapter: **nccl-tests** drives AllReduce, AllGather, and ReduceScatter collectives at configurable message sizes and reports bus bandwidth and algorithm bandwidth, making the fabric the measurable variable rather than application logic. The **UCX** library is installed alongside **NCCL** because it provides the transport plugin that allows **NCCL** to use **UCX**-managed **RDMA**, shared-memory, and CXL transports as alternatives to the default **libibverbs** path, and comparing **UCX**-transport bandwidth against native **RDMA** isolates the overhead of each abstraction layer. The **CUDA** toolkit (or **ROCm** for AMD GPUs) supplies the device runtime that both **NCCL** and **UCX** require to move data between GPU memory and the NIC without staging through host memory. On **InfiniBand** clusters, the **SHARP** libraries are installed when switch firmware supports in-network aggregation, as **SHARP** offloads AllReduce computation into the switch ASICs and reduces fabric traffic by an order of magnitude for large collectives.

This chapter's lab uses **Docker** to simulate two **NCCL** ranks over the socket backend, so no physical GPUs are required. Install the following on Ubuntu 24.04.

### System packages

```bash
# Docker (if not already installed)
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER   # log out and back in after this

# RDMA fabric diagnostics (for reading and reference on real systems)
sudo apt install -y rdma-core ibverbs-utils infiniband-diags

# Packet capture for the optional tcpdump step
sudo apt install -y tcpdump
```

### GPU system path (Option A — skip if CPU-only)

```bash
# CUDA toolkit and build dependencies for nccl-tests
sudo apt install -y cuda-toolkit-12-4 build-essential

# Build nccl-tests against the local CUDA installation
git clone https://github.com/NVIDIA/nccl-tests
cd nccl-tests && make CUDA_HOME=/usr/local/cuda MPI=0
# Produces ./build/all_reduce_perf and friends
```

### CPU-only / Docker simulation path (Option B — used in the lab)

```bash
# Pull the NGC PyTorch image — includes NCCL with socket backend compiled in
# NGC (NVIDIA GPU Cloud) is NVIDIA's container registry at nvcr.io providing pre-built,
# performance-optimized containers for deep learning frameworks (PyTorch, TensorFlow, etc.)
docker pull nvcr.io/nvidia/pytorch:24.01-py3

# Create a dedicated Docker network for the two simulated ranks
docker network create nccl-lab
```

### Python environment (uv)

```bash
uv venv .venv && source .venv/bin/activate

# CPU-only PyTorch (no CUDA required)
uv pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu

# paramiko — used by optional remote-execution helper scripts
uv pip install paramiko
```

---

## 19.1 What the Fabric Must Deliver

NCCL's performance model is simple: every nanosecond that a GPU waits for AllReduce to complete is a nanosecond of wasted compute. The AllReduce completion time is bounded by the slowest link in the collective:

```
AllReduce time ≈ max(
    2 × (N-1)/N × message_size / bandwidth,  # ring-AllReduce transfer
    latency × 2 × log2(N)                    # tree-AllReduce latency
)
```

For ring-AllReduce at N=1000 ranks, 100GB gradient tensor, 400Gbps NIC:
- Transfer time: 2 × 999/1000 × 100GB / 400Gbps ≈ 4 seconds

This is the *minimum* time given perfect fabric utilization. Any packet drop, ECMP collision, or congestion-triggered retransmission adds to this directly.

**The fabric must provide:**
1. Bandwidth: at least 90% of NIC line rate for large messages
2. Latency: p99 < 10 µs for control messages that synchronize collective phases
3. Loss-free operation: a single dropped packet causes RDMA retransmission, stalling the entire collective
4. ECMP fairness: no hash collisions that leave some links idle while others are saturated. ECMP (Equal-Cost Multi-Path) is the routing mechanism by which multiple parallel paths of equal cost are used to distribute traffic across a leaf-spine fabric; hash collisions occur when many flows hash to the same path, causing one link to be overloaded while adjacent links remain idle.

---

## 19.2 NCCL Plugin Architecture

NCCL is not transport-monolithic. It selects a network plugin at initialization based on what's available:

```
NCCL runtime
    │
 ┌──┴─────────────────────────────────────────────┐
 │  Net plugins (selected at init via dlopen)     │
 │  ncclNet_t interface:                          │
 │    net.init(), net.devices(), net.connect()    │
 │    net.send(), net.recv(), net.test()          │
 ├────────────────────────────────────────────────┤
 │  nccl-net-ib    ← libibverbs (IB / RoCE)      │
 │  nccl-net-socket ← TCP fallback               │
 │  nccl-net-ucx   ← UCX plugin (all transports) │
 │  nccl-net-sharp ← SHARP in-network aggregation│
 └────────────────────────────────────────────────┘
```

`libibverbs` is the user-space InfiniBand verbs library — the standard RDMA programming API that both InfiniBand and RoCEv2 Ethernet expose to applications. It provides queue pair (QP) management, memory registration, and one-sided RDMA operations. The UCX plugin allows NCCL to use any UCX-supported transport (IB, RoCE, shmem, CXL) without NCCL itself needing transport-specific code. `HCA` (Host Channel Adapter) is the RDMA NIC used in InfiniBand and RoCEv2 deployments; NVIDIA Mellanox ConnectX series adapters are the dominant HCA in AI clusters, exposed to the OS as `mlx5_N` devices.

### Selecting the Transport

```bash
# Force IB/RoCE transport
NCCL_NET=IB

# Force UCX plugin
NCCL_NET=UCX UCX_TLS=rc,shmem

# Force specific NIC devices
NCCL_IB_HCA=mlx5_0:1,mlx5_1:1   # use port 1 of mlx5_0 and mlx5_1

# Enable SHARP in-network aggregation
NCCL_NET=SHARP SHARP_COLL_ENABLE_SAT=1

# TCP fallback (for debugging)
NCCL_SOCKET_IFNAME=eth0
```

---

## 19.3 SHARP — In-Network Aggregation

SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) offloads AllReduce computation to InfiniBand switch ASICs. Instead of all ranks sending gradients to a root and back, the switches themselves perform the partial sums:

```
Without SHARP:
  1000 ranks × 100GB = 100 TB traverses the fabric (ingress + egress)

With SHARP:
  Leaf switch aggregates 8 ranks: 8 → 1 message per hop
  Total fabric traffic: ~100GB × log₈(1000) ≈ 300 GB
  Reduction: ~330×
```

```bash
# Enable SHARP in NCCL
NCCL_NET=SHARP
SHARP_COLL_ENABLE_SAT=1
SHARP_COLL_LOG_LEVEL=3    # verbose for debugging

# Check SHARP allocation
sharp_cmd --command alloc_resources  # verify SHARP tree allocated correctly
```

SHARP requires InfiniBand switches with SHARP support (NVIDIA Quantum series). It does not work over RoCEv2 Ethernet.

---

## 19.4 Topology-Aware Rank Assignment

NCCL builds a topology map at initialization, using system information to understand:
- Which GPUs share NVLink (intra-node)
- Which nodes share a ToR switch (intra-rack)
- The full fabric topology

```bash
# Provide explicit topology file for complex fabrics
NCCL_TOPO_FILE=/etc/nccl/cluster_topo.xml

# Example topology file (NCCL XML format):
# <system>
#   <cpu id="0" affinity="0-23" arch="x86_64" vendor="GenuineIntel">
#     <pci busid="0000:3b:00.0" class="0x030200" vendor="0x10de" device="0x2330">
#       <!-- H100 GPU 0 -->
#     </pci>
#     <pci busid="0000:3c:00.0" class="0x020000" vendor="0x15b3" device="0x101d">
#       <!-- Mellanox NIC on this socket -->
#     </pci>
#   </cpu>
# </system>

# NCCL uses topology to minimize cross-rail traffic:
# Ranks on same node → NVLink (intra-node AllReduce)
# Ranks on same rail → same fabric path (intra-rack AllReduce)
# Cross-rack → fabric uplinks
```

---

## 19.5 RCCL and oneCCL — Fabric Interface Differences

**RCCL (AMD ROCm Collective Communications Library):**
- API-compatible with NCCL; drop-in replacement for AMD MI-series GPUs
- Uses ROCm communication fabric; HIP (Heterogeneous-computing Interface for Portability) is AMD's GPU programming API analogous to CUDA; HIP peer memory provides GPU-direct equivalent functionality — direct DMA transfers between AMD GPU memory and the RDMA NIC without staging through CPU DRAM
- Plugin architecture mirrors NCCL; UCX and IB plugins work with the same environment variables

**oneCCL (Intel oneAPI):**
- Supports CPU and Intel GPU (Xe/Gaudi) collectives
- Uses MPI or OFI (Open Fabric Interface — the LibFabric abstraction layer that provides a transport-agnostic API over InfiniBand verbs, RoCE, shmem, and other providers) as the transport backend
- Less flexible plugin model than NCCL; configuration via CCL_ATL_TRANSPORT variable

```bash
# oneCCL — select LibFabric transport
CCL_ATL_TRANSPORT=ofi
FI_PROVIDER=verbs      # use IB verbs via LibFabric
FI_VERBS_IFACE=ib0

# oneCCL — select MPI transport
CCL_ATL_TRANSPORT=mpi
I_MPI_FABRICS=shm:ofa  # shared memory + OpenFabrics
```

---

## 19.6 NCCL Debugging and Fabric Diagnosis

```bash
# Enable NCCL verbose logging
NCCL_DEBUG=INFO    # summary per-rank
NCCL_DEBUG=TRACE   # per-operation detail (very verbose)

# Useful log entries:
# NCCL INFO Ring 0 : 0 -> 1 -> 2 -> 3 -> 0  ← ring topology
# NCCL INFO Using internal Network Socket       ← fell back to TCP!
# NCCL INFO NET/IB : Using [0]mlx5_0:1/RoCE    ← using RDMA

# Test collectives without training framework
git clone https://github.com/NVIDIA/nccl-tests
cd nccl-tests && make
mpirun -np 8 ./build/all_reduce_perf -b 8 -e 4G -f 2 -g 1
# Shows bandwidth at each message size from 8B to 4GB
```

When `nccl-tests` shows low bandwidth at large message sizes, the fabric is the bottleneck. When it shows high bandwidth but training is slow, the culprit is compute-communication overlap (check GPU utilization during AllReduce).

---

## Lab Walkthrough 19 — Simulating Two NCCL Ranks Over the Socket Backend

This walkthrough uses two Docker containers on a shared Docker network to simulate two NCCL ranks communicating over TCP sockets. No NVIDIA GPU or RDMA NIC is required. The goal is to observe NCCL's initialization output, ring topology selection, and the effect of algorithm and environment variable changes.

### Step 1: Create the Docker network

```bash
docker network create nccl-lab
```

Expected output:
```
<sha256-hash>
```

Verify the network exists:

```bash
docker network inspect nccl-lab --format '{{.Name}} driver={{.Driver}} subnet={{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

Expected output (subnet may vary):
```
nccl-lab driver=bridge subnet=172.18.0.0/16
```

### Step 2: Start rank 0 (the root container)

Open a terminal and run:

```bash
docker run --rm -it \
  --name nccl-rank0 \
  --network nccl-lab \
  --hostname rank0 \
  -e NCCL_DEBUG=INFO \
  -e NCCL_SOCKET_IFNAME=eth0 \
  -e NCCL_ALGO=Ring \
  nvcr.io/nvidia/pytorch:24.01-py3 \
  bash
```

Inside the container, verify the network interface:

```bash
ip addr show eth0
```

Expected output (IP will differ):
```
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 172.18.0.2/16 ...
```

Note the IP address — you will need it for rank 1 in the next step.

### Step 3: Start rank 1 (the worker container)

Open a second terminal and run:

```bash
docker run --rm -it \
  --name nccl-rank1 \
  --network nccl-lab \
  --hostname rank1 \
  -e NCCL_DEBUG=INFO \
  -e NCCL_SOCKET_IFNAME=eth0 \
  -e NCCL_ALGO=Ring \
  nvcr.io/nvidia/pytorch:24.01-py3 \
  bash
```

### Step 4: Run a minimal NCCL AllReduce test using Python

On rank 0, start a Python-based two-rank rendezvous. Paste the following script into a file named `nccl_test.py` in both containers (or create it once and copy):

```python
# nccl_test.py
import os
import torch
import torch.distributed as dist
import time

def main():
    rank = int(os.environ["RANK"])
    world_size = int(os.environ["WORLD_SIZE"])
    master_addr = os.environ["MASTER_ADDR"]
    master_port = os.environ.get("MASTER_PORT", "29400")

    dist.init_process_group(
        backend="gloo",          # gloo works on CPU without NCCL; swap to "nccl" on GPU systems
        init_method=f"tcp://{master_addr}:{master_port}",
        rank=rank,
        world_size=world_size,
    )

    tensor = torch.ones(1024 * 1024) * (rank + 1)   # rank 0 sends 1.0, rank 1 sends 2.0
    print(f"[rank {rank}] before AllReduce: {tensor[0].item():.1f}")

    t0 = time.perf_counter()
    dist.all_reduce(tensor, op=dist.ReduceOp.SUM)
    elapsed = time.perf_counter() - t0

    print(f"[rank {rank}] after AllReduce:  {tensor[0].item():.1f}  (expected 3.0)")
    print(f"[rank {rank}] AllReduce latency: {elapsed*1000:.2f} ms")

    dist.destroy_process_group()

if __name__ == "__main__":
    main()
```

> **Note:** The NGC PyTorch image includes NCCL compiled in. On CPU-only containers without NVIDIA drivers, use `backend="gloo"`. To observe NCCL environment variable effects on a real GPU system, switch to `backend="nccl"` and the NCCL_DEBUG lines in the output below will appear from the NCCL runtime.

On **rank 0** (replace `172.18.0.2` with the IP from Step 2):

```bash
RANK=0 WORLD_SIZE=2 MASTER_ADDR=172.18.0.2 MASTER_PORT=29400 python nccl_test.py
```

On **rank 1** (use the same MASTER_ADDR pointing to rank 0):

```bash
RANK=1 WORLD_SIZE=2 MASTER_ADDR=172.18.0.2 MASTER_PORT=29400 python nccl_test.py
```

Expected output on each rank (gloo backend):

```
[rank 0] before AllReduce: 1.0
[rank 0] after AllReduce:  3.0  (expected 3.0)
[rank 0] AllReduce latency: 12.47 ms

[rank 1] before AllReduce: 2.0
[rank 1] after AllReduce:  3.0  (expected 3.0)
[rank 1] AllReduce latency: 12.51 ms
```

Verification: Both ranks must print `3.0`. The latency will be higher than NCCL+RDMA (which achieves < 1 ms on InfiniBand) because gloo uses TCP over Docker's virtual bridge.

### Step 5: Observe NCCL_DEBUG=INFO output on a GPU system

If you have access to a GPU system, repeat Step 4 with `backend="nccl"` and `NCCL_DEBUG=INFO`. The NCCL runtime will emit lines like:

```
NCCL INFO Bootstrap : Using eth0:172.18.0.2<0>
NCCL INFO NET/Socket : Using [0]eth0:172.18.0.2<0>
NCCL INFO Ring 00 : 0 -> 1 -> 0
NCCL INFO Ring 00 : Coll 0 nchannels 1 comm 0x...
NCCL INFO Connected all rings
```

Key lines to interpret:
- `NET/Socket : Using [0]eth0` — NCCL fell back to the socket (TCP) transport because no IB NIC was found. On a real cluster this line would read `NET/IB : Using [0]mlx5_0:1/RoCE`.
- `Ring 00 : 0 -> 1 -> 0` — NCCL built a two-rank ring: rank 0 sends to rank 1, rank 1 sends back to rank 0.
- `Connected all rings` — collective initialization is complete; the AllReduce can proceed.

### Step 6: Compare NCCL_ALGO=Ring vs NCCL_ALGO=Tree

On GPU systems with NCCL backend, the algorithm choice affects latency and bandwidth:

```bash
# Ring algorithm — optimal for large messages (bandwidth-bound)
NCCL_ALGO=Ring RANK=0 WORLD_SIZE=2 MASTER_ADDR=172.18.0.2 \
  python nccl_test.py

# Tree algorithm — optimal for small messages (latency-bound)
NCCL_ALGO=Tree RANK=0 WORLD_SIZE=2 MASTER_ADDR=172.18.0.2 \
  python nccl_test.py
```

Expected behavior (from NCCL literature, N=2 ranks):
- Ring: lower latency overhead for large tensors; each rank sends/receives one full chunk per step.
- Tree: fewer hops in the reduction tree reduces latency at small message sizes; at N=2 the tree degenerates to a single root, so the difference is negligible.
- At N=8 or more ranks the distinction becomes measurable: Ring wins above ~1 MB; Tree wins below ~64 KB.

The gloo backend does not expose `NCCL_ALGO`; this variable is a NCCL-specific tuning knob and is ignored by gloo.

### Step 7: Capture NCCL socket traffic with tcpdump

On the host (not inside Docker), start a packet capture on the Docker bridge interface:

```bash
# Find the Docker bridge interface name
ip link show | grep -A1 "nccl-lab"
# Typically shows something like: br-<hash>

# Capture on that interface, filtering for the rendezvous port
sudo tcpdump -i br-$(docker network inspect nccl-lab --format '{{.Id}}' | cut -c1-12) \
  -w nccl_lab.pcap port 29400
```

While the capture runs, re-run the AllReduce test in both containers (Steps 4). Then stop tcpdump with Ctrl-C.

Inspect the capture:

```bash
sudo tcpdump -r nccl_lab.pcap -nn -q | head -30
```

Expected output (timestamps and IPs will differ):

```
12:01:00.001234 IP 172.18.0.2.29400 > 172.18.0.3.54321: tcp 0
12:01:00.001301 IP 172.18.0.3.54321 > 172.18.0.2.29400: tcp 0
12:01:00.001450 IP 172.18.0.2.29400 > 172.18.0.3.54321: tcp 148
12:01:00.002100 IP 172.18.0.3.54321 > 172.18.0.2.29400: tcp 148
...
```

What to look for:
- The first few packets on port 29400 are the rendezvous handshake — ranks exchange addresses and collective metadata.
- After the handshake, data-plane packets appear on higher ephemeral ports (NCCL opens its own data sockets after rendezvous).
- The data-plane flow for a 4 MB AllReduce between 2 ranks over TCP should complete in roughly 10–50 ms over a Docker bridge, dominated by loopback RTT and process scheduling rather than NIC bandwidth.

### Step 8: What NCCL_IB_HCA and NCCL_NET=IB would do on a real system

On a machine with Mellanox/NVIDIA RDMA NICs:

```bash
# Tell NCCL which HCA ports to use (port 1 of mlx5_0 and mlx5_1)
export NCCL_IB_HCA=mlx5_0:1,mlx5_1:1

# Force IB/RoCE transport (disables socket fallback)
export NCCL_NET=IB

# Verify IB devices are visible
ibv_devices
# Expected output:
# device          node GUID
# ------          ----------------
# mlx5_0          e41d2d03006c3d60

ibstat mlx5_0
# Shows port state, link layer (InfiniBand or Ethernet/RoCE), active speed
```

When `NCCL_NET=IB` is set and an IB NIC is present, the `NCCL INFO` line changes from:

```
NCCL INFO NET/Socket : Using [0]eth0:...
```

to:

```
NCCL INFO NET/IB : Using [0]mlx5_0:1/RoCE   GID 0
```

This confirms NCCL is using RDMA. The AllReduce latency for a 100 MB tensor drops from ~50 ms (TCP/gloo) to ~2 ms (RoCEv2 at 100 Gbps), and to < 1 ms on HDR InfiniBand at 200 Gbps.

### Step 9: Clean up

```bash
# Stop the containers (they were run with --rm so they delete automatically)
docker stop nccl-rank0 nccl-rank1

# Remove the Docker network
docker network rm nccl-lab
```

---

## Summary

- NCCL demands loss-free, low-tail-latency, high-bandwidth RDMA from the fabric; any degradation at the fabric level appears as AllReduce stall time in the training job.
- NCCL's plugin architecture (UCX, IB, SHARP) allows the same high-level library to use whatever transport the fabric provides.
- SHARP in-network aggregation eliminates gradient traffic from the fabric by performing partial AllReduce in switch ASICs — requiring InfiniBand Quantum switches.
- Topology-aware rank assignment (via `NCCL_TOPO_FILE`) ensures the most bandwidth-intensive communication happens over the fastest available paths.

---

## References

- NCCL documentation: docs.nvidia.com/deeplearning/nccl
- NCCL GitHub: github.com/NVIDIA/nccl
- NCCL tests: github.com/NVIDIA/nccl-tests
- SHARP documentation: docs.nvidia.com/networking/display/sharpv300
- RCCL: github.com/ROCm/rccl
- oneCCL: github.com/oneapi-src/oneCCL


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).