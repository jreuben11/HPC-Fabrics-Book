# Chapter 20 — Distributed Training Runtimes: Fabric Demands

**Part VI: Storage & Adjacent Infrastructure** | ~10 pages

> **Scope note:** PyTorch Distributed, DeepSpeed, Ray, Horovod, and Megatron-LM are covered in depth in the companion investigation on distributed AI training. This chapter focuses exclusively on the traffic patterns and fabric requirements each runtime imposes — the network engineer's view.

---

## Installation

This chapter's lab runs entirely on a single CPU-only machine using PyTorch's gloo backend. No GPUs and no cluster are required.

### System packages

```bash
sudo apt install -y tcpdump tshark python3 python3-pip
```

### Python environment (uv)

```bash
uv venv .venv && source .venv/bin/activate

# CPU-only PyTorch — gloo backend works without CUDA
uv pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
```

Verify the install:

```bash
python -c "import torch; print(torch.__version__); print(torch.distributed.is_available())"
# Expected: 2.x.x  followed by  True
```

---

## Introduction

Distributed training frameworks are the software layer that partitions a large model training job across tens, hundreds, or thousands of GPUs. From the application developer's perspective, these frameworks — PyTorch Distributed, DeepSpeed, Megatron-LM, Horovod, Ray Train — abstract away the complexity of gradient synchronization, model sharding, and inter-node communication. From the network engineer's perspective, they are traffic generators whose output the fabric must sustain at near-line rate, and whose communication patterns determine whether a given fabric topology is the right choice for a given training workload.

The central challenge is that different parallelism strategies produce fundamentally different traffic patterns. Data parallelism (DDP, FSDP) produces all-to-all AllReduce bursts at every optimizer step — a pattern that rail-optimized fat-tree topologies are designed to absorb efficiently. Pipeline parallelism produces ordered, point-to-point traffic between adjacent model stages — a pattern where cross-rack latency becomes the dominant performance variable. Tensor parallelism produces extremely latency-sensitive AllReduce operations within a small group of GPUs — a pattern so demanding that it is almost always confined to NVLink within a single node. Each pattern implies different requirements for bandwidth, latency, loss sensitivity, and topology.

Understanding these patterns is not merely academic: it determines which GPU server models to purchase, how many tiers of switching are needed, whether InfiniBand or RoCEv2 is cost-effective for a given workload, whether RDMA QP setup overhead matters, and how to configure QoS priority classes so that the right traffic gets the right service. The network engineer who understands parallelism strategies can participate meaningfully in capacity planning, can diagnose training slowdowns by recognizing their network signatures, and can size the storage and training fabrics independently to prevent cross-contamination.

This chapter covers the traffic profiles of DDP, FSDP, pipeline parallelism, tensor parallelism, DeepSpeed ZeRO stages, and Ray — providing quantitative estimates of message sizes, frequencies, and directional patterns for each. A lab walkthrough then makes these abstractions concrete: two PyTorch ranks running on a single CPU machine capture the rendezvous handshake and AllReduce data-plane traffic with tcpdump, measure how AllReduce time scales with tensor size, and establish the baseline numbers that make the case for investing in a high-performance training fabric.

This chapter builds directly on Chapter 19 (NCCL and collective communications), which covers the transport layer that executes these patterns, and connects forward to Chapter 29 (Kubernetes AI scheduling), which determines how these parallel workloads are placed on the fabric. Chapter 18 (distributed storage) addresses the checkpoint write traffic that must not compete with the AllReduce traffic analyzed here.

---

## 20.1 The Network Engineer's View of Training Runtimes

From the fabric's perspective, a distributed training runtime is a traffic generator. The questions that matter are:
- What communication pattern does it produce? (all-to-all, ring, point-to-point)
- What volume of traffic per step?
- What is the message size distribution? (determines transport efficiency)
- Is the communication overlapped with compute, or does it stall the GPU?
- What ports and protocols does it use?

---

## 20.2 Data Parallel: DDP and FSDP

### PyTorch DDP (Distributed Data Parallel)

DDP synchronizes gradients after each backward pass using AllReduce. Every rank sends and receives the full gradient tensor.

**Traffic profile:**
- **Pattern:** All-to-all (ring-AllReduce or tree-AllReduce via NCCL)
- **Message size:** Full gradient tensor = 2 × num_params × dtype_bytes
  - GPT-3 175B in bf16: 350 GB per step
- **Frequency:** Once per optimizer step
- **Overlap:** DDP buckets gradients and overlaps AllReduce with backward pass — partial overlap of compute and communication
- **Port:** NCCL selects ports dynamically; `NCCL_PORT_RANGE=60000:61000` to constrain

**Fabric implication:** Drives sustained all-to-all bandwidth; requires loss-free RDMA and low p99 latency. Rail-optimized topology (Chapter 1) is designed for this pattern.

### PyTorch FSDP (Fully Sharded Data Parallel)

FSDP shards both model parameters and optimizer state across ranks, reducing per-GPU memory. Before each layer's forward pass, FSDP performs an AllGather to reconstruct the full layer. After backward, it performs a ReduceScatter.

**Traffic profile:**
- **Pattern:** AllGather (before forward) + ReduceScatter (after backward) — functionally equivalent to AllReduce but with intermediate sharding
- **Message size:** Per-layer, not full model — smaller individual transfers but more frequent
- **Overlap:** FSDP can prefetch the next layer's parameters while the current layer computes
- **Fabric implication:** Higher message frequency, smaller average message size than DDP. Transport efficiency is important for small messages — RDMA with low QP (Queue Pair) setup overhead preferred. A QP is the fundamental RDMA communication endpoint: a pair of send and receive queues that must be connected and transitioned to the Ready-to-Receive state before data transfer begins; QP setup adds latency that is significant when individual message sizes are small relative to setup cost.

---

## 20.3 Pipeline Parallelism

Pipeline parallel splits the model into stages, each on a different set of GPUs. Activation tensors are the communication.

**Traffic profile:**
- **Pattern:** Point-to-point between adjacent pipeline stages
- **Message size:** Activation tensor size = batch_size × seq_len × hidden_dim × dtype_bytes
  - For GPT-3, hidden_dim=12288, seq_len=2048, bf16: ~400 MB per microbatch
- **Frequency:** 2× per microbatch (forward activation + backward gradient)
- **Directional:** Strictly ordered; stage N sends to stage N+1 in forward; N+1 to N in backward
- **Fabric implication:** Unidirectional, ordered point-to-point traffic. Latency dominates throughput for small microbatches. Pipeline bubble (GPU idle fraction) is inversely proportional to the communication latency between stages.

**Key variable for network engineers:** If pipeline stages span nodes (internode), the internode latency adds directly to the pipeline bubble. Rail-optimized topology with dedicated stage-to-stage paths minimizes this.

---

## 20.4 Tensor Parallelism (Megatron-LM)

Tensor parallelism shards individual weight matrices across GPUs within a tensor-parallel group (typically 4–8 GPUs).

**Traffic profile:**
- **Pattern:** AllReduce within tensor-parallel group after each matrix multiply
- **Message size:** Activation tensor / TP_degree — typically 10–100 MB
- **Frequency:** 2× per transformer layer (attention + MLP), per microbatch
- **Latency sensitivity:** Extremely high — on the critical path of every forward/backward pass
- **Fabric implication:** Tensor parallelism is almost always confined to NVLink (intra-node) because cross-node latency makes it economically unviable. The network fabric sees tensor-parallel traffic only in NVLink-bridged multi-node configurations.

---

## 20.5 DeepSpeed ZeRO Stages

DeepSpeed's ZeRO (Zero Redundancy Optimizer) partitions model state across ranks to reduce memory usage. Different stages have different communication profiles:

| ZeRO Stage | What's Partitioned | Communication |
|---|---|---|
| ZeRO-1 | Optimizer states | AllReduce gradients (= DDP) |
| ZeRO-2 | + Gradients | ReduceScatter gradients → AllGather params |
| ZeRO-3 | + Parameters | AllGather params before each forward layer |

**Stage 3 traffic profile:**
- AllGather of each layer's parameters before its forward pass
- ReduceScatter of each layer's gradients after its backward pass
- Very high message count (one AllGather + ReduceScatter per layer per step)
- Individual messages smaller but total volume ≈ DDP

**Fabric implication:** ZeRO-3's fine-grained communication makes it more sensitive to message rate (RDMA QP setup overhead, ACK latency) than to raw bandwidth. DC (Dynamically Connected) QP mode is an InfiniBand transport type that shares a single QP across many remote destinations, dramatically reducing the number of QPs required at scale compared to Reliable Connected (RC) mode — critical when ZeRO-3 is communicating with hundreds of peers simultaneously. UCX's DC transport (Chapter 4) provides the same benefit over RoCEv2. DC QP mode or UCX's DC transport (Chapter 4) is important for ZeRO-3 at scale.

---

## 20.6 Ray — Distributed Task and Object Scheduling

Ray is not primarily a training framework but a general distributed computing framework. In the AI context, Ray Train and Ray Serve are the relevant components.

**Ray's network traffic patterns:**

1. **Object store transfers:** Large tensors (model weights, batches) are stored in Ray's distributed object store (Plasma, backed by shared memory or Apache Arrow). Plasma is Ray's in-memory object store that uses shared memory to allow zero-copy reads by multiple processes on the same node; Apache Arrow is the columnar in-memory data format that Plasma uses to serialize tensors and dataframes efficiently. When an object on node A is needed by a task on node B, Ray transfers it via its own object transfer protocol (not RDMA, not NCCL).
   - Protocol: TCP/gRPC on a dynamic port
   - Fabric implication: Object transfers use the management network; for large models this can saturate a 25GbE management link

2. **GCS (Global Control Store):** Ray's cluster state database; small control-plane traffic between all workers and the GCS
   - Volume: low; few MB/s at scale

3. **Training with Ray Train + Torch Distributed:** When using Ray Train with PyTorch DDP backend, NCCL handles gradient AllReduce (as in Section 20.2); Ray itself only handles task scheduling, not data transfer.

**Fabric implication:** Separate Ray object store traffic (management network) from NCCL traffic (training fabric). Large model serving use cases where Ray moves checkpoint files should not share bandwidth with training.

---

## 20.7 Ports and Firewall Considerations

A complete port table for training fabric ACLs (Access Control Lists — firewall rules that permit or deny traffic based on source/destination IP, port, and protocol, used here to define what traffic the training fabric must allow):

| Service | Protocol | Ports | Notes |
|---|---|---|---|
| NCCL (socket fallback) | TCP | 60000–61000 | Configurable via `NCCL_PORT_RANGE` |
| PyTorch rendezvous | TCP | 29400 | `init_process_group` master port |
| NCCL IB | RDMA | IB ports | Not TCP; no firewall filtering needed |
| Ray GCS | TCP | 6379 | Configurable |
| Ray dashboard | TCP | 8265 | Management only |
| Ray object transfer | TCP | 8076 | Dynamic per node |
| gRPC (various) | TCP | 50051 | Configurable |
| Prometheus scrape | TCP | 9100, 9400 | node_exporter, dcgm-exporter |

---

## Lab Walkthrough 20 — Traffic Pattern Analysis with tcpdump and PyTorch Distributed

This walkthrough runs two PyTorch distributed ranks on a single CPU machine using the gloo backend. `gloo` is Facebook's collective communications library that implements AllReduce, AllGather, and barrier over TCP sockets — it works on CPUs without CUDA or RDMA hardware, making it ideal for development and testing. `torchrun` is PyTorch's built-in process launcher that spawns the specified number of worker processes, assigns each a RANK and WORLD_SIZE environment variable, and establishes the rendezvous over the specified master address and port. You will capture the rendezvous handshake and AllReduce traffic on the loopback interface, analyze it with tshark, and measure how AllReduce time scales with tensor size.

### Step 1: Write the two-rank AllReduce script

Create the file `ddp_allreduce.py` with the following content:

```python
# ddp_allreduce.py
"""
Two-rank AllReduce timing script using PyTorch distributed (gloo backend).
Run with:
    torchrun --nproc_per_node=2 --master_port=29400 ddp_allreduce.py
"""

import os
import time
import torch
import torch.distributed as dist

# Tensor sizes to test: 1 MB, 10 MB, 100 MB, 1 GB
# Each float32 element is 4 bytes, so:
#   1 MB  → 256 * 1024 elements
#  10 MB  → 2560 * 1024 elements
# 100 MB  → 25600 * 1024 elements
#   1 GB  → 256000 * 1024 elements
SIZES_MB = [1, 10, 100, 1024]

def main():
    rank = int(os.environ["RANK"])
    world_size = int(os.environ["WORLD_SIZE"])
    local_rank = int(os.environ.get("LOCAL_RANK", rank))

    dist.init_process_group(backend="gloo")

    if rank == 0:
        print(f"Initialized process group: world_size={world_size}, backend=gloo")

    results = []

    for size_mb in SIZES_MB:
        n_elements = (size_mb * 1024 * 1024) // 4   # float32 = 4 bytes
        tensor = torch.ones(n_elements) * (rank + 1)

        # Warm-up pass
        dist.all_reduce(tensor.clone(), op=dist.ReduceOp.SUM)

        # Timed pass — average over 3 iterations
        times = []
        for _ in range(3):
            t = tensor.clone()
            t0 = time.perf_counter()
            dist.all_reduce(t, op=dist.ReduceOp.SUM)
            times.append(time.perf_counter() - t0)

        avg_ms = sum(times) / len(times) * 1000
        results.append((size_mb, avg_ms))

        if rank == 0:
            expected = sum(r + 1 for r in range(world_size))
            assert abs(t[0].item() - expected) < 1e-3, \
                f"AllReduce correctness check failed: got {t[0].item()}, expected {expected}"
            print(f"  {size_mb:>6} MB   avg={avg_ms:7.1f} ms   "
                  f"effective_bw={size_mb/((avg_ms/1000)*1024):.2f} GB/s")

    if rank == 0:
        print("\nSize vs time summary (for fabric analysis):")
        print(f"{'Size (MB)':>12}  {'Time (ms)':>12}")
        for size_mb, ms in results:
            print(f"{size_mb:>12}  {ms:>12.1f}")
        print()
        print("Linear relationship between size and time confirms bandwidth-bound regime.")
        print("On gloo/socket, bandwidth is limited by loopback (~10 GB/s).")
        print("On NCCL+InfiniBand HDR, expect ~22 GB/s at 200 Gbps line rate.")

    dist.destroy_process_group()

if __name__ == "__main__":
    main()
```

### Step 2: Start a tcpdump capture on the loopback interface

In a separate terminal, start capturing before launching the training script. The rendezvous handshake happens on port 29400 and the gloo data plane uses additional ephemeral ports on loopback.

```bash
sudo tcpdump -i lo -w training.pcap port 29400
```

Leave this running. You will stop it after the training script exits.

### Step 3: Run the AllReduce script with torchrun

In your original terminal (with the `.venv` activated):

```bash
torchrun --nproc_per_node=2 --master_port=29400 ddp_allreduce.py
```

Expected output:

```
Initialized process group: world_size=2, backend=gloo
       1 MB   avg=    2.3 ms   effective_bw=0.43 GB/s
      10 MB   avg=   18.7 ms   effective_bw=0.53 GB/s
     100 MB   avg=  185.4 ms   effective_bw=0.54 GB/s
    1024 MB   avg= 1893.1 ms   effective_bw=0.54 GB/s

Size vs time summary (for fabric analysis):
   Size (MB)     Time (ms)
           1          2.3
          10         18.7
         100        185.4
        1024       1893.1

Linear relationship between size and time confirms bandwidth-bound regime.
On gloo/socket, bandwidth is limited by loopback (~10 GB/s).
On NCCL+InfiniBand HDR, expect ~22 GB/s at 200 Gbps line rate.
```

> **Note on actual numbers:** The exact timings depend on your machine's available memory bandwidth and scheduler. The key observation is that time scales linearly with size — confirming bandwidth-bound operation, not latency-bound. The 1 MB result may appear disproportionately slow relative to its size because it includes fixed gloo connection setup overhead.

Stop the tcpdump capture with Ctrl-C in the other terminal:

```
^C
N packets captured
N packets received by filter
0 packets dropped by kernel
```

### Step 4: Analyze the capture with tshark

`tshark` is the command-line version of Wireshark — a full network protocol analyzer that reads pcap capture files and can dissect, filter, and extract fields from any protocol it recognizes. It supports the same display filter syntax as Wireshark, making it the standard tool for scripted packet analysis.

Inspect the rendezvous handshake on port 29400:

```bash
tshark -r training.pcap -Y "tcp.port == 29400" -T fields \
  -e frame.time_relative -e ip.src -e tcp.srcport -e ip.dst -e tcp.dstport \
  -e tcp.flags.str -e tcp.len | head -30
```

Expected output (IPs will be 127.0.0.1 for loopback):

```
0.000000   127.0.0.1  29400  127.0.0.1  52341  ...S....  0
0.000051   127.0.0.1  52341  127.0.0.1  29400  ...SA...  0
0.000061   127.0.0.1  29400  127.0.0.1  52341  ....A...  0
0.000092   127.0.0.1  52341  127.0.0.1  29400  ....AP..  148
0.000103   127.0.0.1  29400  127.0.0.1  52341  ....AP..  148
...
```

What the output shows:
- The first three lines are the TCP three-way handshake (SYN, SYN-ACK, ACK) on port 29400.
- The subsequent `AP` (ACK + PUSH) packets carry the rendezvous metadata: rank addresses, collective configuration, and initial barrier synchronization.
- After the rendezvous completes on port 29400, gloo opens new TCP connections on ephemeral ports for the actual AllReduce data transfer.

To see the data-plane connections as well, capture without the port filter:

```bash
sudo tcpdump -i lo -w training_full.pcap
# re-run torchrun ... in the other terminal, then Ctrl-C here

tshark -r training_full.pcap -Y "tcp.len > 0" -T fields \
  -e frame.time_relative -e ip.src -e tcp.srcport -e ip.dst -e tcp.dstport \
  -e tcp.len | sort -k5 -n | uniq -f4 | head -20
```

This shows the distinct destination ports used by gloo for the data plane — typically two or more ephemeral ports opened after rendezvous, one per AllReduce bucket.

### Step 5: Add TORCH_DISTRIBUTED_DEBUG=DETAIL and read the log

Re-run with verbose distributed logging:

```bash
TORCH_DISTRIBUTED_DEBUG=DETAIL \
  torchrun --nproc_per_node=2 --master_port=29400 ddp_allreduce.py 2>&1 | head -60
```

Expected additional log lines (mixed with the rank output):

```
[W ProcessGroupGloo.cpp:...] Warning: TORCH_DISTRIBUTED_DEBUG was set ...
[rank0]:[I ...] Gloo ProcessGroup initialized with ...
[rank0]:[I ...] Created a socket timeout wrapper ...
[rank0]:[I ...] Rank 0 store set key ...
[rank1]:[I ...] Rank 1 store set key ...
```

Key entries to look for:
- `ProcessGroup initialized` — confirms both ranks joined the process group.
- `store set key` / `store get key` — these are the rendezvous key-value exchanges on port 29400; each rank announces itself and waits for all others before proceeding.
- If a rank fails to connect, the log shows `Timed out waiting for key` — useful for diagnosing firewall or routing issues on real clusters.

### Step 6: Plot time vs size to understand fabric implications

The numbers from Step 3 show a linear relationship. Sketch or plot them:

```
Time (ms)
 2000 |                                            *
      |
 1000 |
      |
  200 |                              *
      |
   20 |             *
    2 |  *
      +--+----------+----------------+------------+---> Size (MB)
         1          10              100          1024
```

The straight line on a linear-linear plot confirms that at all sizes tested, gloo on loopback is bandwidth-bound, not latency-bound. The effective bandwidth (slope of the line) is approximately 0.5 GB/s — limited by the gloo socket implementation and Python overhead, not the loopback interface itself (which can sustain > 10 GB/s).

**What this means for fabric design:**

| Scenario | Backend | Bandwidth | Time for 350 GB (GPT-3 step) |
|---|---|---|---|
| This lab (gloo/loopback) | gloo | ~0.5 GB/s | ~11 minutes |
| Production: 100GbE RoCE | NCCL | ~11 GB/s | ~32 seconds |
| Production: 200Gb HDR IB | NCCL | ~22 GB/s | ~16 seconds |
| Production: 400Gb NDR IB | NCCL | ~44 GB/s | ~8 seconds |

The fabric bandwidth directly determines the AllReduce wall time per training step. Doubling the NIC bandwidth halves the AllReduce time — which is why moving from 100GbE to 400GbE HDR is worth the cost for large model training.

### Step 7: Verify with TORCH_DISTRIBUTED_DEBUG on a timeout scenario

To see what timeout errors look like (useful for understanding cluster firewall issues), run with an unreachable master:

```bash
TORCH_DISTRIBUTED_DEBUG=INFO \
MASTER_ADDR=192.0.2.1 MASTER_PORT=29400 RANK=0 WORLD_SIZE=2 \
  timeout 10 python -c "
import torch.distributed as dist
dist.init_process_group(backend='gloo', init_method='env://', timeout=__import__('datetime').timedelta(seconds=5))
" 2>&1 | tail -10
```

Expected output (after ~5 seconds):

```
RuntimeError: Timed out initializing process group in store based barrier on rank 0, for key: store_based_barrier_key:1 (world_size=2, worker_count=1, timeout=0:00:05)
```

This is the exact error message that appears on production clusters when a rank cannot reach the master address — caused by incorrect `MASTER_ADDR`, a blocked port 29400, or a network misconfiguration. The timeout duration is configurable via the `timeout` parameter to `init_process_group`.

### Step 8: Brief comparison — what NCCL + RDMA would show

On a system with NVIDIA GPUs and InfiniBand:

```bash
# Switch backend to nccl and run on GPU tensors
# In ddp_allreduce.py, change:
#   dist.init_process_group(backend="gloo")
# to:
#   dist.init_process_group(backend="nccl")
# and:
#   tensor = torch.ones(n_elements)
# to:
#   tensor = torch.ones(n_elements).cuda()

NCCL_DEBUG=INFO torchrun --nproc_per_node=2 --master_port=29400 ddp_allreduce_gpu.py
```

From published NCCL benchmarks (nccl-tests, 2-node HDR InfiniBand):

| Size | gloo/CPU (this lab) | NCCL/IB HDR |
|---|---|---|
| 1 MB | ~2 ms | ~0.05 ms |
| 100 MB | ~185 ms | ~9 ms |
| 1 GB | ~1900 ms | ~90 ms |

The fabric (InfiniBand HDR at 200 Gbps) delivers ~20x better bandwidth and ~40x better latency than gloo on CPU sockets. The gloo numbers from this lab establish the baseline that makes the case for investing in a high-performance training fabric.

### Step 9: Clean up

```bash
# Remove capture files
rm -f training.pcap training_full.pcap

# Deactivate the virtual environment
deactivate
```

---

## Summary

- DDP/FSDP generate all-to-all AllReduce traffic at each optimizer step; the fabric must sustain this at near-line rate with loss-free delivery.
- Pipeline parallelism generates ordered point-to-point traffic between adjacent stages; internode latency directly inflates the pipeline bubble.
- Tensor parallelism is typically confined to NVLink intra-node; cross-node tensor parallelism only makes sense with very high-bandwidth, very low-latency interconnects.
- DeepSpeed ZeRO-3's fine-grained per-layer communication is more sensitive to RDMA connection setup overhead than to raw bandwidth.
- Ray's object store traffic should be isolated on the management network to avoid competing with NCCL AllReduce on the training fabric.

---

## References

- PyTorch Distributed documentation: pytorch.org/docs/stable/distributed.html
- DeepSpeed documentation: deepspeed.ai/docs
- Ray documentation: docs.ray.io
- Megatron-LM: github.com/NVIDIA/Megatron-LM
- Rajbhandari et al., *ZeRO: Memory Optimizations Toward Training Trillion Parameter Models*, SC 2020


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).