# Chapter 33 — AI Inference Serving Fabric

> **Scope note:** This chapter focuses on the network architecture of AI inference serving — the traffic patterns, transfer protocols, hardware topology decisions, and load-balancing primitives that govern how a trained model responds to live user requests. Training fabric demands are covered in Chapter 20; RDMA and RoCEv2 fundamentals are in Chapter 2; SR-IOV and RDMA in Kubernetes pods are in Chapter 13; NCCL collectives are in Chapter 19; NVIDIA Dynamo scheduling and Grove placement are in Chapter 29; Gateway API extension points are in Chapter 32.

---

## Introduction

Deploying a large language model into production exposes a set of network engineering problems that are structurally different from the problems encountered during training. A training job runs for days or weeks, operates on a fixed set of GPUs, and produces a single workload profile: sustained, synchronous all-reduce collectives that require maximum bisection bandwidth and strict loss-free delivery. An inference serving cluster, by contrast, must handle thousands of independent, unpredictable requests per second, each with its own latency budget, context length, and caching potential. The network sees bursty fan-out, tight latency constraints, and a fundamentally new primitive — the **KV cache** — that must be transferred, cached, and routed as a first-class network object.

This chapter builds the inference serving fabric from first principles. We begin with the taxonomy of inference traffic and why it differs from training traffic, work through the compute and bandwidth profiles of the prefill and decode phases, derive the architecture of **disaggregated prefill/decode** serving from those profiles, explain the KV cache RDMA transfer mechanisms that bind the two phases together, survey the parallelism communication patterns within an inference cluster, examine the network requirements of the dominant serving frameworks (**vLLM**, **TensorRT-LLM**, **SGLang**, **NVIDIA Triton Inference Server**), and conclude with load balancing strategies and fabric sizing guidance. A lab simulates a two-process disaggregated inference pipeline using **soft-RoCE** to make the KV transfer path concrete and measurable.

---

## Installation

The lab in this chapter requires only a standard Linux workstation. No GPU is needed.

```bash
# Ubuntu 24.04
sudo apt-get update
sudo apt-get install -y rdma-core libibverbs-dev ibverbs-utils \
    perftest python3-pyverbs

# Load soft-RoCE over loopback interface
sudo modprobe rdma_rxe
sudo rdma link add rxe0 type rxe netdev lo

# Verify
ibv_devices          # should show rxe0
rdma link show       # should show ACTIVE state
```

---

## 33.1 Inference vs. Training Traffic Taxonomy

Training and inference are both GPU-intensive, both network-intensive, and both sensitive to tail latency — but the *nature* of their network demands differs in almost every detail.

**Table 33.1 — Training vs. Inference Traffic Comparison**

| Property | Training (DDP / TP) | Inference (Prefill + Decode) |
|---|---|---|
| **Pattern type** | Synchronous collective (all-reduce, all-gather, reduce-scatter) | Request-response; async fan-out |
| **Traffic shape** | Sustained, bulk, all-to-all | Bursty, point-to-point / small fan-out |
| **Primary metric** | Throughput (GB/s aggregate, GPU utilisation) | Latency (TTFT, inter-token latency) |
| **Message size** | 350 GB per all-reduce step (175B model, BF16) | KV cache blocks: 320 KB – 40 GB per request |
| **Frequency** | One all-reduce per optimizer step (~1 Hz for large batch) | 1,000+ requests/second per cluster |
| **Flow symmetry** | Symmetric (every rank sends and receives equal data) | Asymmetric (prefill node sends; decode node receives) |
| **Loss tolerance** | Zero — single packet loss stalls entire collective | Very low — but TCP fallback possible for KV cache |
| **Congestion control** | DCQCN (ECN-based, zero-drop) mandatory | DCQCN preferred; TCP UCX fallback acceptable |
| **Intra-node fabric** | NVLink for tensor parallelism | NVLink for intra-rack KV migration; PCIe for decode |
| **Inter-node fabric** | InfiniBand or RoCEv2 at rail bandwidth | RoCEv2 (or IB) for cross-node KV transfer |
| **Scheduling** | Gang scheduling (all-or-nothing) | Continuous batching; per-request |
| **State** | Stateless between steps (gradients only) | Stateful per-token (KV cache persists in DRAM/HBM) |

The single most important distinction for fabric design is **pattern type**. Training lives or dies on bulk-bandwidth synchronous collectives. Inference lives or dies on the latency budget for individual requests, measured in **time-to-first-token (TTFT)** for the prefill phase and **inter-token latency (ITL)** for the decode phase.

---

## 33.2 Prefill and Decode Phases: Compute and Bandwidth Profiles

Every LLM inference request proceeds in two sequential phases with sharply different resource profiles.

### 33.2.1 Prefill Phase

The prefill phase processes the entire input **prompt** — all input tokens — in a single forward pass. It is **compute-bound**: the attention mechanism operates on a large matrix (sequence length × hidden dimension), and the GPU arithmetic units are saturated. For a 1,024-token prompt on a 70B-parameter model, the prefill pass performs approximately 1.4 × 10¹⁴ floating-point operations (roughly 2 FLOPs per parameter per input token: 2 × 70 × 10⁹ × 1,024 ≈ 1.43 × 10¹⁴). A single H100 SXM sustains ~989 TFLOPs/s BF16, placing single-GPU prefill time for a 1K-token prompt at roughly 145 ms. In practice, tensor parallelism across 8 GPUs (TP=8, connected by NVLink) reduces this to ~18–25 ms per request; for smaller models (7B) or shorter prompts a single GPU achieves 1–10 ms. Prefill time scales linearly with prompt length, so a 32K-token prompt on a TP=8 Llama-3 70B takes roughly 18 ms × 32 ≈ 576 ms on H100 NVLink hardware — a substantial TTFT contribution at long context.

The **key/value cache (KV cache)** — the intermediate key and value tensors for all attention heads across all layers — is produced as a side effect of prefill and stored for reuse during decode. It must be retained for the duration of the request. The KV cache is the primary data object transferred between prefill and decode nodes in a disaggregated architecture.

### 33.2.2 Decode Phase

The decode phase generates one output token per forward pass, autoregressively. Each step attends over all previously generated tokens using the accumulated KV cache. Decode is **memory-bandwidth-bound**: the full model weight set (140 GB for 70B in BF16) must be streamed through the memory subsystem on every step, but only a tiny fraction of the weight bytes participate in useful arithmetic per token. An H100 provides 3.35 TB/s HBM bandwidth; streaming 140 GB of weights per step limits a single H100 to approximately 3,350 GB/s ÷ 140 GB ≈ **24 decode steps per second** without optimisation. With TP=8 on NVLink, each GPU holds 17.5 GB of weights and achieves ~190 steps/s per rank, with the all-reduce synchronisation adding ~5–15% overhead; the cluster emits roughly 160–180 generated tokens/second total for the TP group.

The asymmetry is stark:

```
Prefill:  high FLOP/byte ratio  →  arithmetic-limited  →  needs fast CUDA cores / Tensor Cores
Decode:   low FLOP/byte ratio   →  memory-bandwidth-limited  →  needs wide HBM bandwidth
```

Running both phases on the same GPU means the hardware is never matched to its workload. Prefill requests preempt decode steps, injecting latency spikes into ITL. Decode steps keep the GPU idle between attention reads, wasting compute that prefill could use. This mis-match motivates **disaggregation**.

---

## 33.3 Disaggregated Prefill/Decode Architecture

Disaggregated prefill/decode separates the two phases onto different physical hardware pools, connected by a KV cache transfer network. The architecture resolves the FLOP/byte mismatch and makes TTFT and ITL independently tunable.

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Request Router                                │
│    (Dynamo Smart Router / llm-d / Gateway API Endpoint Picker)       │
└───────────────────┬──────────────────────────────┬───────────────────┘
                    │ (1) Route request             │ (4) Route decode
                    ▼                               ▼
  ┌─────────────────────────────┐   ┌───────────────────────────────────┐
  │     Prefill Node Pool        │   │        Decode Node Pool           │
  │                              │   │                                   │
  │  GPU: compute-optimised      │   │  GPU: memory-BW-optimised         │
  │  (H100 SXM, high TFLOPs)     │   │  (H100 SXM or NVL-variant)       │
  │                              │   │                                   │
  │  Forward pass on full prompt │   │  One token per forward pass       │
  │  → produces KV cache blocks  │   │  KV cache resident in HBM         │
  └─────────────┬───────────────┘   └────────────┬──────────────────────┘
                │                                 │
                │  (2) KV cache transfer           │ (5) Stream tokens
                │  NIXL / UCX / RoCEv2 RDMA       │     back to router
                │  (NVLink if same rack)           │
                └────────────────►────────────────┘
                   (3) KV blocks land in decode HBM
```

**Key claim**: Disaggregated prefill/decode does **not** improve throughput — the same FLOPs and memory bandwidth are consumed regardless. The benefit is latency predictability: TTFT is determined by the prefill pool alone, and ITL is determined by the decode pool alone, eliminating the head-of-line blocking that occurs when a long prefill preempts a running decode sequence on shared hardware (see vLLM's documentation: "Disaggregated prefill DOES NOT improve throughput").

### 33.3.1 When to Disaggregate

Disaggregation is justified when:

1. **TTFT latency SLO is tight** (< 500 ms) and prompt lengths vary widely, causing prefill time variance that bleeds into decode jitter.
2. **Tail ITL** matters (p99 ITL > 2× median indicates prefill-decode contention on shared hardware).
3. The workload mix is skewed: high-volume short-prompt requests (chatbot) co-exist with long-prompt summarisation or RAG pipelines.

For batch inference workloads with uniform prompt length and no latency SLO, co-located prefill/decode with chunked prefill is simpler and avoids the KV transfer overhead.

---

## 33.4 KV Cache Size Estimation

Understanding KV cache transfer requires knowing how large the cache is. The formula is:

```
KV_bytes = 2          # K + V tensors
         × num_layers
         × num_kv_heads
         × head_dim
         × seq_len
         × bytes_per_element
```

For **Llama 3 70B** (80 layers, 8 KV heads via GQA, head_dim 128, BF16 = 2 bytes):

```python
num_layers      = 80
num_kv_heads    = 8    # Grouped Query Attention — 8 KV heads for 64 query heads
head_dim        = 128
bytes_per_elem  = 2    # BF16

# per-token KV cache size
bytes_per_token = 2 * num_layers * num_kv_heads * head_dim * bytes_per_elem
# = 2 × 80 × 8 × 128 × 2 = 327,680 bytes ≈ 320 KB per token

print(f"KV cache per token:  {bytes_per_token / 1024:.1f} KB")
# → 320.0 KB

for ctx_len, label in [(1024, "1K"), (8192, "8K"), (32768, "32K"), (131072, "128K")]:
    mb = bytes_per_token * ctx_len / 1024**2
    print(f"  At {label:5s} tokens: {mb:7.1f} MB  ({mb/1024:.2f} GB)")
```

Output:
```
KV cache per token:  320.0 KB
  At    1K tokens:   320.0 MB  (0.31 GB)
  At    8K tokens:  2560.0 MB  (2.50 GB)
  At   32K tokens: 10240.0 MB  (10.00 GB)
  At  128K tokens: 40960.0 MB  (40.00 GB)
```

For a typical 4K-context chatbot request, the KV cache transferred from prefill to decode is approximately **1.25 GB**. At 400 Gb/s (50 GB/s) RoCEv2, this takes approximately **27 ms** — which must fit within the TTFT budget before the first decode token can be produced. At 8K context it is approximately 2.5 GB / 50 GB/s ≈ **54 ms**. These numbers set the floor for TTFT even before compute time.

---

## 33.5 KV Cache RDMA Transfer

### 33.5.1 GPUDirect and DMA-BUF

The most efficient KV cache transfer path bypasses the CPU entirely. **GPUDirect RDMA** (see Chapter 2) enables a network adapter to perform DMA directly into or out of GPU **HBM**, without staging through host DRAM. The data path is:

```
Prefill GPU HBM  →  PCIe bus  →  NIC DMA engine  →  Wire (RoCEv2)
                                                           │
                                                           ▼
                                                 Decode NIC DMA  →  PCIe  →  Decode GPU HBM
```

The RDMA operation is an `RDMA_WRITE` from the prefill node to a pre-registered memory region in the decode GPU's address space. The NIC's DMA engine issues the PCIe read directly to GPU BAR2 (with P2P enabled) and never touches host DRAM. Round-trip setup latency for small messages is approximately 8–17 µs measured on ConnectX-7 hardware; for large KV cache transfers (1–40 GB) the transfer time is bandwidth-dominated: `T ≈ bytes / NIC_bandwidth + setup_overhead`.

**DMA-BUF** is the Linux kernel's unified mechanism for sharing DMA-accessible buffers across devices. NVIDIA's kernel driver exposes GPU memory allocations as DMA-BUF file descriptors, allowing the RDMA subsystem to import them without a separate memory copy. Modern MLNX_OFED drivers support DMA-BUF import via `ibv_reg_dmabuf_mr()`, which registers the GPU HBM region as an MR in a Protection Domain without any host-side copy.

### 33.5.2 NVLink vs. RoCEv2 for KV Migration

The choice of interconnect depends on node placement:

| Scenario | Interconnect | Bandwidth | Notes |
|---|---|---|---|
| Prefill and decode on same **GB200 NVL72** rack | NVLink (Grace-Blackwell NVLink) | 900 GB/s aggregate | Zero PCIe overhead; lowest latency path |
| Prefill and decode on same server (multi-GPU) | NVLink switch | 600–900 GB/s | Intra-node P2P via `cudaMemcpyPeerAsync` |
| Prefill and decode on adjacent nodes, same rack | RoCEv2 over 400 GbE | ~50 GB/s per NIC | GPUDirect RDMA; single switch hop |
| Prefill and decode on different rack clusters | RoCEv2 over 400 GbE | ~50 GB/s per NIC | Two switch hops; latency ~10 µs added |

**NVIDIA Grove** (see Chapter 29 and §33.8 below) uses topology labels to ensure that prefill/decode pairs are co-located on the highest-bandwidth interconnect available — NVLink domains on GB200 NVL72 racks, or at minimum the same RDMA rail for cross-node RoCEv2 transfers.

### 33.5.3 NIXL — NVIDIA Inference Transfer Library

**NIXL** (NVIDIA Inference Transfer Library) is a point-to-point data-movement API purpose-built for inference. It is **not** a wire protocol; it is an abstraction layer that presents a consistent non-blocking, non-contiguous transfer API over several pluggable backends:

| NIXL Backend | Wire Protocol | Use Case |
|---|---|---|
| **UCX** | InfiniBand verbs / RoCEv2 | GPU-to-GPU KV transfer over RDMA |
| **GDS** (GPUDirect Storage) | NVMe-oF / local NVMe | KV cache offload to local SSD |
| **S3** | HTTPS | KV cache offload to object storage |

NIXL is the transfer library used by **NVIDIA Dynamo** for KV migration. When Dynamo assigns a request to a prefill worker, the prefill worker registers its KV cache memory region with NIXL and calls `nixl_send()` targeting the decode worker's pre-registered receive buffer. The backend selection is configured per-deployment; for low-latency serving the UCX backend over RoCEv2 is standard.

### 33.5.4 Mooncake Transfer Engine

**Mooncake** is an independently developed KV cache transfer engine originating from Moonshot AI, now integrated into both vLLM v1 (as `MooncakeConnector`) and SGLang's **Encode-Prefill-Decode (EPD) disaggregation** mode. Mooncake implements zero-copy RDMA transfer using its own RDMA engine (separate from NIXL), optimized for disaggregated serving workloads. It maintains a distributed KV cache directory that allows decode workers to retrieve specific KV blocks by content hash, enabling cross-request KV reuse (prefix caching across disaggregated nodes).

---

## 33.6 vLLM Disaggregated Prefill/Decode

**vLLM** (see Chapter 20 for training context) is the dominant open-source LLM serving framework. Its disaggregated prefill implementation uses a **KV Transfer Config** that designates each instance as either a KV producer (prefill) or KV consumer (decode), mediated by a **connector** that handles the actual data movement.

### 33.6.1 Connector Architecture

vLLM defines a connector abstraction with the following concrete implementations:

| Connector | Transport | Notes |
|---|---|---|
| `NixlConnector` | NIXL (UCX/GDS/S3 backends) | Recommended for production RDMA |
| `P2pNcclConnector` | NCCL P2P (NVLink or net) | Simpler setup; used in examples |
| `MooncakeConnector` | Mooncake RDMA engine | Zero-copy; cross-request prefix cache |
| `LMCacheConnectorV1` | NIXL via LMCache | Adds semantic KV cache management |
| `OffloadingConnector` | CPU DRAM → network | For CPU-staging architectures |
| `MultiConnector` | Configurable chain | Fallback across multiple backends |

The connector is configured per-instance via `--kv-transfer-config`:

```bash
# Prefill instance — KV producer on GPU 0, port 8100
CUDA_VISIBLE_DEVICES=0 vllm serve meta-llama/Llama-3-70b-Instruct \
    --host 0.0.0.0 \
    --port 8100 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.8 \
    --kv-transfer-config '{
        "kv_connector": "NixlConnector",
        "kv_role": "kv_producer",
        "kv_rank": 0,
        "kv_parallel_size": 2,
        "kv_connector_extra_config": {
            "nixl_backend": "UCX"
        }
    }'

# Decode instance — KV consumer on GPU 1, port 8200
CUDA_VISIBLE_DEVICES=1 vllm serve meta-llama/Llama-3-70b-Instruct \
    --host 0.0.0.0 \
    --port 8200 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.8 \
    --kv-transfer-config '{
        "kv_connector": "NixlConnector",
        "kv_role": "kv_consumer",
        "kv_rank": 1,
        "kv_parallel_size": 2,
        "kv_connector_extra_config": {
            "nixl_backend": "UCX"
        }
    }'
```

A proxy server running on port 8000 routes each incoming request: it forwards to the prefill instance with `max_tokens=1` to trigger prefill-only execution, then forwards the request (with KV cache already transferred) to the decode instance for generation. This proxy can be a simple Python HTTP server or **NGINX** with custom upstream logic.

### 33.6.2 Distributed Executor and Tensor Parallelism

vLLM's **distributed executor** coordinates tensor-parallel inference across multiple GPUs. Within a single multi-GPU node (e.g., 8× H100 with NVLink), tensor parallelism (`--tensor-parallel-size 8`) shards the model weight matrices across GPUs. Each forward pass requires two **all-reduce** operations per transformer layer (one after the attention projection, one after the MLP projection), synchronizing the partial sums across all TP ranks.

Within a node, vLLM uses NCCL over NVLink for these all-reduces. Cross-node tensor parallelism requires NCCL over RoCEv2 or InfiniBand, and incurs significantly higher latency — the research literature (see arXiv:2507.14392) reports TPOT (time per output token) degrading from 0.86 ms intra-node to 11.56 ms inter-node for the same TP degree, making cross-node tensor parallelism impractical for latency-sensitive serving.

**Context Parallelism (CP)**, also called sequence parallelism, addresses very long contexts by splitting the sequence dimension across GPUs. Each GPU attends over its own slice of the sequence, then exchanges partial KV sums via all-gather and reduce-scatter. This pattern appears in very long-context serving (> 32K tokens) where a single GPU's HBM is insufficient to hold the full KV cache.

---

## 33.7 SGLang: RadixAttention and Hierarchical Caching

**SGLang** (Structured Generation Language) is a high-performance serving framework focused on multi-turn conversations and structured output generation, where prefix reuse is especially valuable.

### 33.7.1 RadixAttention

SGLang's core caching primitive is **RadixAttention**, which maintains a **radix tree** of KV cache blocks keyed by token sequence. When a new request arrives whose prompt shares a prefix with a previous request, the system retrieves the already-computed KV blocks from the tree rather than recomputing them. This is in-instance, in-process prefix sharing — the radix tree lives in the serving process's GPU HBM and is shared across concurrent requests within the same replica.

Benefits of RadixAttention within a single replica:
- Eliminates duplicate prefill compute for shared system prompts
- Reduces average TTFT for multi-turn conversations (subsequent turns reuse the conversation history KV cache)
- Enables larger effective batch sizes by reducing per-request KV footprint

**Important scoping note**: RadixAttention is an *intra-instance* optimisation. It does not provide cross-instance KV sharing by itself.

### 33.7.2 HiCache — Hierarchical KV Storage

**SGLang HiCache** (announced September 2025) extends RadixAttention with a **HiRadixTree** that manages KV cache across a hierarchy of storage tiers: GPU HBM → CPU DRAM → local NVMe SSD, and potentially remote storage. A cache controller automatically migrates KV blocks between tiers based on access recency, treating the hierarchy as a unified address space with a single logical radix tree interface.

The HiCache controller communicates with storage backends asynchronously to avoid blocking decode steps. For the GPU → CPU DRAM transfer path, DMA-BUF pinned transfers are used. For the CPU DRAM → NVMe path, direct-I/O writes avoid double buffering.

### 33.7.3 EPD Disaggregation with Mooncake

SGLang supports **Encode-Prefill-Decode (EPD) disaggregation** using the **Mooncake** transfer backend, which uses its own RDMA engine for zero-copy KV migration between independently deployed SGLang instances. In this mode, an "encode" node tokenizes the input, a "prefill" node executes the forward pass and ships KV blocks, and a "decode" node runs the autoregressive generation. Cross-instance prefix sharing through Mooncake's content-addressed KV directory enables prefix cache hits across the disaggregated pool.

---

## 33.8 NVIDIA Dynamo: Network Architecture

*(This section extends §29.9.3, which covers Dynamo's scheduling model. Here the focus is on what crosses the wire.)*

**NVIDIA Dynamo** is an open-source distributed inference serving framework that implements disaggregated prefill/decode at cluster scale. Its network architecture has three distinct communication planes.

### 33.8.1 Control Plane: Smart Router

The **Dynamo Smart Router** is the request routing component. It maintains a global view of KV cache block locality across the prefill and decode worker pools, using a distributed **radix tree** indexed by request prefix hash. When a new request arrives, the router:

1. Computes the **prefix overlap score** between the request's prompt and the KV blocks currently cached in each decode worker's HBM.
2. Routes to the decode worker with the highest overlap score (to maximize KV cache hit rate and avoid retransfer).
3. If no decode worker has a matching prefix, routes to the least-loaded prefill worker and schedules a KV transfer to the selected decode worker.

This routing strategy delivers the measured improvements cited by the llm-d project: prefix-aware routing reduced P99 TTFT from 6,800 ms to 1,000 ms on the same hardware — an 85% reduction — by eliminating redundant prefill compute and KV retransfer.

### 33.8.2 Data Plane: NIXL KV Transfer

The data plane carries KV cache blocks between prefill and decode workers using NIXL. The backend matrix:

- **Intra-rack on GB200 NVL72**: NIXL uses the NVLink interconnect (via NVLink-aware UCX transport), achieving near-memory-bandwidth transfer without leaving the rack's NVLink domain.
- **Cross-node on the same RoCEv2 rail**: NIXL's UCX backend uses GPUDirect RDMA with `RDMA_WRITE` operations, achieving ~50 GB/s per 400 GbE NIC.
- **KV offload to SSD**: NIXL's GDS backend uses GPUDirect Storage for direct GPU-to-NVMe transfers, useful for deep KV cache tiering.
- **KV offload to object storage**: NIXL's S3 backend for warm-tier KV archival.

NIXL transfers are **non-blocking and non-contiguous**: the API accepts a scatter/gather list of KV block addresses and initiates transfers without holding the GPU idle. This allows the prefill worker to begin the next request's prefill immediately after initiating the KV send.

### 33.8.3 Grove: Topology-Aware Worker Placement

**Grove** is NVIDIA's Kubernetes scheduling extension within the Dynamo ecosystem, providing **network topology-aware gang scheduling** for inference workloads (see Chapter 29 for the Kubernetes scheduling layer). From a network perspective, Grove optimises placement along the following hierarchy:

```
Cloud Region
  └── Availability Zone
        └── Data Center
              └── Network Block (spine domain)
                    └── Rack (ToR switch)
                          └── Host
                                └── NUMA Node
                                      └── GPU
```

For disaggregated inference, Grove's primary constraint is **co-location of prefill/decode pairs within the highest-bandwidth interconnect boundary available**:

- On **GB200 NVL72** hardware, Grove schedules prefill and decode pods within the same NVLink domain (72 GPUs, single rack), enabling NVLink KV transfer.
- On **H100 DGX** multi-node clusters, Grove schedules prefill/decode pairs on nodes sharing the same top-of-rack (ToR) switch to minimise switch hops in the KV transfer path.
- Decode replicas are spread across racks for fault tolerance, while each decode pod is paired with a prefill pod on the same rack.

Grove uses **PodCliqueScalingGroups** to manage this: a PodCliqueScalingGroup defines a set of PodCliques (e.g., one prefill + one decode) that scale and are scheduled together as a gang, but where each component scales independently. The scheduler respects GPU topology labels (NVLink domain membership, PCIe root complex affinity, NUMA node) when placing each PodClique.

### 33.8.4 NIC and Fabric Requirements for Dynamo

| Requirement | Specification |
|---|---|
| **Inter-node KV transfer** | 400 GbE RoCEv2 (ConnectX-7 or BlueField-3 DPU) per node, GPUDirect RDMA enabled |
| **Intra-rack KV transfer** | NVLink (NVL72) or 400 GbE within same ToR domain |
| **Control plane** | 10/25 GbE management NIC for router → worker gRPC |
| **Lossless fabric** | PFC + DCQCN mandatory for RoCEv2 KV transfer path |
| **Congestion control** | DCQCN (same as training fabric, see Ch2); ECN marking at 10% buffer threshold |

---

## 33.9 TensorRT-LLM Executor

**NVIDIA TensorRT-LLM** is a compilation-based inference engine that generates fused CUDA kernels per model and shape, exposed via an **Executor API** used by frameworks including **Triton Inference Server**. Its distributed communication uses:

- **Tensor parallelism**: NCCL all-reduce operations over NVLink (intra-node) or RoCEv2 (cross-node). Same pattern as vLLM but with kernel-fused all-reduce for smaller latency per step.
- **Pipeline parallelism**: MPI-based coordination between pipeline stages (using `NCCL_P2P_*` or `nccl_send/nccl_recv`) for inter-stage activation passing.
- **Inflight batching**: TensorRT-LLM supports continuous batching within a single executor instance, keeping the GPU maximally utilised by immediately scheduling new requests into vacated KV cache slots.

The combination of compilation-based kernel fusion and continuous batching makes TensorRT-LLM particularly efficient for single-node or small-cluster serving where KV migration between nodes is not needed. For larger disaggregated deployments, TensorRT-LLM instances can be fronted by Triton Inference Server with custom ensemble models that coordinate prefill/decode split.

---

## 33.10 Triton Inference Server: gRPC Endpoint Sizing

**NVIDIA Triton Inference Server** is a general-purpose model serving runtime that exposes models over **gRPC** (HTTP/2) and HTTP/REST endpoints, with a C API for embedding. In LLM deployments, Triton typically fronts a TensorRT-LLM backend and handles:

- **Dynamic batching**: combining multiple in-flight inference requests into a single larger batch to improve GPU throughput. The `max_queue_delay_microseconds` parameter controls how long Triton waits for additional requests before dispatching — trading throughput for latency.
- **Sequence batching**: tracking multi-turn conversation state across requests, routing all requests from the same session to the same backend instance (for KV cache reuse).
- **Model ensemble**: coordinating multiple model backends (e.g., a retrieval model → embedding model → generation model) in a single request pipeline.

### 33.10.1 NIC Utilization from Dynamic Batching

Dynamic batching has a non-obvious effect on NIC utilization. When Triton aggregates requests into large batches:

- **Uplink (client → Triton)**: many small request messages; NIC is dominated by packet rate, not bandwidth. At 1,000 requests/second with average 2 KB payloads, the uplink load is 2 MB/s — well within any NIC capacity.
- **Downlink (Triton → client)**: response streaming for long generations; with 1,000 tokens × 4 bytes × 1,000 req/s = 4 GB/s sustained if streaming, but typically bursty.
- **Backend (Triton → TensorRT-LLM)**: internal UNIX socket or shared memory for co-located backends; no NIC traffic.
- **KV cache coordination**: if Triton orchestrates a disaggregated pipeline, KV transfer traffic appears on the data-plane NIC (400 GbE RoCEv2).

The gRPC protocol adds per-stream framing and flow control overhead. For high request-rate deployments (> 5,000 req/s), **HTTP/2 multiplexed gRPC streams** keep connection overhead manageable, but at > 10,000 req/s it is worth considering **batched unary calls** or switching to the **C API** for collocated backends. Each gRPC connection uses roughly one OS thread in the Triton server thread pool, so connection count scaling must be planned alongside NIC queue depth.

```yaml
# Triton model configuration for LLM with dynamic batching
name: "llm_model"
backend: "tensorrtllm"
max_batch_size: 64

dynamic_batching {
  preferred_batch_size: [1, 8, 16, 32]
  max_queue_delay_microseconds: 1000   # 1 ms wait; tune per SLO
}

sequence_batching {
  max_sequence_idle_microseconds: 5000000  # 5 s session timeout
}
```

---

## 33.11 Speculative Decoding Communication

**Speculative decoding** improves decode throughput by using a small, fast **draft model** to speculatively generate several tokens, which are then verified in a single forward pass by the larger **verifier model**. When accepted, the verified tokens are emitted; when rejected, generation falls back to the verifier's output.

### 33.11.1 Communication Patterns

In production deployments today, speculative decoding typically runs co-located: the draft model and verifier model both execute on the same GPU node (or even the same GPU), connected via NVLink or shared GPU memory. In this configuration, no fabric communication occurs — the draft tokens are passed as CUDA tensors.

Cross-node speculative decoding (draft model on one node, verifier on another) is an active research area. The network sensitivity is high:

- Draft tokens are **small messages** (a handful of token ID integers per speculative step — typically 1 to 70 tokens × 4 bytes = 4 to 280 bytes).
- The **round-trip** from draft node → verifier node → draft node must complete faster than one equivalent verifier forward pass to achieve any speedup.
- At a verifier step time of ~40 ms (70B model, single GPU), and a 10 µs network RTT for RoCEv2, the RTT itself is negligible. The bottleneck is the **synchronisation overhead**: draft and verifier must coordinate on accept/reject for every speculative block.

Academic systems (e.g., Decentralized Speculative Decoding, arXiv:2511.11733) reduce synchronization from *k* rounds (one per speculative token) to **one round** by verifying all *k* tokens jointly. This is the correct design for any cross-node SD deployment and reduces network traffic to one small message per speculative block rather than one per token.

For current production serving, speculative decoding is treated as a single-node optimisation. The network fabric is not a constraint unless the draft/verifier split is explicitly deployed across nodes.

---

## 33.12 Load Balancing Inference Endpoints

Inference load balancing differs from stateless web load balancing because GPU-backed model servers maintain **in-memory KV cache state** that makes subsequent requests cheaper if routed to the same server. A naive round-robin scheduler ignores this state, forcing constant KV recomputation. The right routing strategy depends on the type of inference workload.

### 33.12.1 Decision Tree: Routing Strategy Selection

```
Is the request part of a multi-turn conversation or long session?
├─ YES → Session-affinity routing
│         Hash(session_id) → consistent decode worker
│         All turns for the same session hit the same KV cache
│
└─ NO → Is the prompt likely to share a prefix with recent requests?
         (e.g., system prompt, RAG context, few-shot examples)
         ├─ YES → Prefix-aware routing
         │         Hash(prompt_prefix) → decode worker with matching KV cache
         │         Dynamo Smart Router / llm-d KV cache indexer
         │
         └─ NO → Is the decode pool heterogeneous (different GPU types)?
                  ├─ YES → Resource-matching routing
                  │         Route by request complexity (context length, model size)
                  │         to appropriately-sized GPU instances
                  │
                  └─ NO → Least-loaded routing
                            Power-of-two-choices: sample two workers at random,
                            route to the one with shorter queue length
```

### 33.12.2 Consistent Hashing for KV Cache Locality

Consistent hashing applied to prompt prefix tokens provides a stable mapping from prompt content to a decode worker, so that requests with the same prefix (e.g., the same system prompt) are always routed to the same worker and find the KV cache warm.

The **SkyWalker** system (arXiv:2505.24095) demonstrates two algorithms for cross-region inference:
1. **User/session ID hashing**: simple consistent hash of the user identifier — effective for multi-turn sessions.
2. **Prefix trie routing**: maintains a distributed prefix trie of cached prompt prefixes; new requests are matched against the trie and routed to the worker with the longest matching prefix.

The llm-d project reports that prefix-cache-aware scheduling delivers **57× faster response times** and **double throughput** versus standard load balancing on identical hardware, by eliminating redundant KV recomputation.

### 33.12.3 Power-of-Two-Choices for Decode Nodes

For requests that cannot benefit from prefix caching (short unique prompts, cold requests), the **power-of-two-choices** algorithm provides good load distribution with low overhead:

1. Sample two decode workers at random.
2. Query each for their current queue depth (number of active decode sequences).
3. Route to the worker with the shorter queue.

This requires each decode worker to expose a lightweight metric endpoint (a single integer: current active sequences). Polling this metric is far cheaper than a full health check and can be done by the proxy/router component at query time.

### 33.12.4 Gateway API Inference Extension

The **Gateway API Inference Extension** (introduced in Kubernetes 2025, see Chapter 32 for Gateway API fundamentals) adds inference-aware routing primitives on top of the standard Gateway API. The extension introduces two new Custom Resources:

**InferencePool**: A pool of pods running model servers on shared GPU compute, similar to a Kubernetes Service but inference-aware. It defines the compute pool, the deployed model(s), and the scaling/load-balancing policy.

**InferenceModel**: A user-facing model endpoint mapped to an InferencePool. It maps public names (e.g., `meta/llama-3-70b`) to actual model server pods, and supports LoRA adapter variants and traffic-splitting policies.

The routing pipeline extends Gateway API's standard flow:

```
Client
  │
  ▼
Gateway (e.g., Envoy with Cilium)
  │  HTTPRoute matches /v1/completions
  ▼
Endpoint Selection Extension (ESE)
  │  Evaluates real-time pod metrics:
  │  - Queue length per pod
  │  - KV cache occupancy per pod
  │  - Loaded LoRA adapters per pod
  ▼
Selected inference pod (with warm KV cache and matching adapter)
```

The **Endpoint Picker / ESE** component implements the routing policy (prefix-aware, least-queue, adapter-matching) and exposes it as a pluggable extension to the Gateway controller. For gRPC-based serving (TensorRT-LLM, Triton), **GRPCRoute** provides method-level matching (e.g., route `/triton.GRPCInferenceService/ModelInfer` to a specific InferencePool), while the ESE handles the within-pool selection (see Chapter 32 §32.5.5 for GRPCRoute configuration).

```yaml
# Gateway API Inference Extension — InferencePool for Llama 3 70B
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferencePool
metadata:
  name: llama3-70b-pool
  namespace: inference
spec:
  targetPortNumber: 8080
  selector:
    matchLabels:
      app: vllm-decode-worker
  extensionRef:
    name: kv-cache-endpoint-picker   # custom ESE with prefix-aware routing

---
# GRPCRoute for Triton Inference Server gRPC endpoint
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: triton-grpc-route
  namespace: inference
spec:
  parentRefs:
    - name: ai-gateway
  rules:
    - matches:
        - method:
            service: triton.GRPCInferenceService
            method: ModelInfer
      backendRefs:
        - group: inference.networking.x-k8s.io
          kind: InferencePool
          name: llama3-70b-pool
```

---

## 33.13 Fabric Design Implications

### 33.13.1 Spine-Leaf Sizing for Inference

An inference fabric differs from a training fabric in its traffic directionality and burstiness:

- **Training**: all-to-all, sustained, bisection-bandwidth-bound. All spine links must sustain full bandwidth simultaneously.
- **Inference**: fan-out from router, point-to-point KV transfers, bursty. Only a subset of spine links carry KV traffic at any time.

For a 256-GPU inference cluster with 100 requests/second at 4K average context length:

```
KV transfer bandwidth per request:
  = 1.25 GB × 100 req/s = 125 GB/s aggregate KV traffic

Per prefill-to-decode NIC link required bandwidth:
  Assuming 8 prefill nodes → 16 GB/s per prefill NIC sustained
  (well within 400 GbE = 50 GB/s capacity)

Spine oversubscription acceptable: 2:1 (inference is bursty, not sustained)
Leaf-to-spine uplink: 1 × 400 GbE per leaf switch is typically sufficient
```

Contrast with training: a 256-GPU training cluster running 70B model with tensor and pipeline parallelism sustains 350 GB/s aggregate all-reduce traffic continuously, requiring 1:1 oversubscription (full bisection bandwidth) and all spine links active simultaneously.

### 33.13.2 TTFT Latency Budget

A practical TTFT budget decomposition for a 4K-context request on a TP=8 Llama-3 70B deployment:

```
TTFT = t_routing + t_prefill_compute + t_kv_transfer + t_first_decode + t_response

t_routing:         <  1 ms     (router prefix lookup, ESE pod selection)
t_prefill_compute: 70–100 ms   (70B model, 4K tokens, TP=8 H100 NVLink;
                                ~18 ms × 4 ≈ 72 ms scaling from 1K estimate)
t_kv_transfer:     27–54 ms    (1.25 GB at 50 GB/s RoCEv2 for 4K context;
                                eliminated by NVLink co-location)
t_first_decode:    5–15 ms     (one token, memory-bandwidth-bound, TP=8)
t_response:        <  1 ms     (TCP/gRPC response serialisation)

TTFT budget:     ~100–170 ms  without queuing delay (RoCEv2 KV transfer)
                  ~75–115 ms  with NVLink intra-rack KV co-location (no transfer)
```

Queuing delay dominates in overloaded systems — a 100 ms queue wait dwarfs all compute and transfer times. This is why **disaggregated prefill** helps tail TTFT: it prevents a long prefill request from blocking decode steps that have already been prefilled and are waiting for their first decode token.

### 33.13.3 Storage Network for Model Weight Loading

Model weights must be loaded from storage into GPU HBM at startup and during model swaps (e.g., LoRA adapter hot-swap). For a 70B model in BF16:

```
Weight size = 70 × 10⁹ parameters × 2 bytes = 140 GB

Load time at 35 GB/s (NVMe SSD tier) = 4 seconds
Load time at 200 GB/s (NVMe-oF / WEKA distributed storage) = 0.7 seconds
```

For cluster-scale inference with many node failures and restarts, the storage network throughput directly determines how quickly spare nodes can be brought into service. A storage fabric capable of 200 GB/s per server using NVMe-oF (see Chapter 6) allows even cold node bring-up to complete in under one second per model copy — essential for autoscaling inference pools under bursty load.

---

## 33.14 Lab: Disaggregated Inference Simulation with Soft-RoCE

This lab simulates the prefill→decode KV cache transfer path using two Python processes communicating over **soft-RoCE** (`rdma_rxe`). No GPU is required. The goal is to measure how KV transfer time varies with payload size (representing different context lengths) and relate it to the TTFT budget calculated in §33.13.2.

**Prerequisites**: soft-RoCE loaded and `rxe0` device active (see Installation section).

### Lab Step 1 — Install Python RDMA bindings

```bash
pip3 install pyverbs  # Python bindings for libibverbs
```

### Lab Step 2 — Verify Connectivity with ibv_rc_pingpong

Use the standard `perftest` binary to confirm that the soft-RoCE QP can be set up and messages exchanged. This replicates the handshake that a prefill-to-decode KV transfer performs at connection setup time.

```bash
# Terminal 1 — receiver side (simulates decode node)
ibv_rc_pingpong -d rxe0 -g 0

# Terminal 2 — sender side (simulates prefill node)
ibv_rc_pingpong -d rxe0 -g 0 127.0.0.1
```

Expected output on both terminals: a ping-pong exchange report with RTT in the 100–500 µs range for soft-RoCE over loopback (memory bandwidth limited, not wire-speed). On real ConnectX-7 hardware over 400 GbE, RTT for small messages is 8–17 µs.

### Lab Step 3 — Baseline Latency at Small Message Sizes

```bash
# Terminal 1 — server
ib_write_lat -d rxe0 --report_gbits

# Terminal 2 — client; test 4-byte message (one token ID)
ib_write_lat -d rxe0 -s 4 -n 1000 127.0.0.1
```

### Lab Step 4 — Measure Bandwidth at KV-Representative Sizes

Use `ib_write_bw` (from `perftest`) to measure RDMA write bandwidth at payload sizes corresponding to KV caches for different context lengths:

```bash
# Terminal 1 — server side
ib_write_bw -d rxe0 --report_gbits

# Terminal 2 — client side; sweep KV cache sizes
for KV_MB in 320 640 1280 2560 5120 10240; do
    echo "=== KV size: ${KV_MB} MB (approx. context length $((KV_MB / 320))K tokens) ==="
    ib_write_bw -d rxe0 --report_gbits \
        -s $((KV_MB * 1024 * 1024)) \
        -n 10 \
        localhost
    echo ""
done
```

Expected output (soft-RoCE over loopback — bandwidth limited by system memory, not real RDMA NIC):

```
=== KV size: 320 MB (approx. context length 1K tokens) ===
  ...  BW average [Gb/sec]:  ~80–120  (memory-to-memory copy over loopback)
=== KV size: 2560 MB (approx. context length 8K tokens) ===
  ...  BW average [Gb/sec]:  ~80–120
```

On real 400 GbE ConnectX-7 hardware, expect ~380–400 Gb/s (47–50 GB/s).

### Lab Step 5 — Build TTFT Model from Measurements

```python
#!/usr/bin/env python3
# ttft_model.py — compute TTFT from measured KV transfer bandwidth

# Llama-3 70B KV cache parameters
NUM_LAYERS      = 80
NUM_KV_HEADS    = 8
HEAD_DIM        = 128
BYTES_PER_ELEM  = 2  # BF16

bytes_per_token = 2 * NUM_LAYERS * NUM_KV_HEADS * HEAD_DIM * BYTES_PER_ELEM
# = 327,680 bytes ≈ 320 KB per token

# Measured or assumed network bandwidth (bytes/sec)
ROCEV2_BW_GBs   = 50.0   # 400 GbE RoCEv2 real hardware
SOFTROCE_BW_GBs = 10.0   # soft-RoCE over loopback (memory bandwidth limited)

# Prefill compute time estimate (ms)
# Model: Llama-3 70B, TP=8 on H100 NVLink
# Approx: 2 FLOPs per parameter per token; 989 TFLOPs/s per H100; 8 GPUs parallel
TFLOPS_H100_BF16 = 989e12   # per GPU, H100 SXM5
TP_DEGREE        = 8         # tensor parallelism
TP_OVERHEAD      = 1.12      # ~12% overhead for all-reduce sync over NVLink

def prefill_ms(ctx_len):
    """Rough prefill time for 70B, TP=8, H100 NVLink."""
    flops_total = 2 * ctx_len * 70e9       # 2 FLOPs per param per token
    flops_per_gpu = flops_total / TP_DEGREE
    ideal_ms = flops_per_gpu / TFLOPS_H100_BF16 * 1000
    return ideal_ms * TP_OVERHEAD

print(f"{'Context':>10} | {'KV (MB)':>9} | {'KV xfer @RoCEv2':>16} | {'Prefill TP=8':>13} | {'TTFT estimate':>14}")
print("-" * 80)
for ctx_len in [1024, 4096, 8192, 32768, 131072]:
    kv_bytes = bytes_per_token * ctx_len
    kv_mb    = kv_bytes / 1024**2
    xfer_ms  = kv_bytes / (ROCEV2_BW_GBs * 1e9) * 1000
    pre_ms   = prefill_ms(ctx_len)
    ttft_ms  = pre_ms + xfer_ms + 10  # +10 ms overhead (routing + first decode)
    label = f"{ctx_len//1024}K" if ctx_len >= 1024 else str(ctx_len)
    print(f"{label:>10} | {kv_mb:>9.1f} | {xfer_ms:>14.1f} ms | {pre_ms:>11.1f} ms | {ttft_ms:>12.1f} ms")
```

Expected output (for Llama-3 70B TP=8 H100 NVLink, 400 GbE RoCEv2):
```
   Context |   KV (MB) | KV xfer @RoCEv2 | Prefill TP=8 |  TTFT estimate
--------------------------------------------------------------------------------
        1K |     320.0 |            6.7 ms |        20.3 ms |         37.0 ms
        4K |    1280.0 |           26.8 ms |        81.2 ms |        118.0 ms
        8K |    2560.0 |           53.7 ms |       162.3 ms |        226.0 ms
       32K |   10240.0 |          214.7 ms |       649.3 ms |        874.0 ms
      128K |   40960.0 |          859.0 ms |      2597.1 ms |       3466.1 ms
```

**Interpretation**: At short-to-medium contexts (1K–4K), prefill compute (20–80 ms) dominates over KV transfer (7–27 ms). At 8K and above, both become significant and both grow linearly with context. NVLink intra-rack co-location (Grove) eliminates the KV transfer column entirely — reducing TTFT from 118 ms to ~91 ms at 4K, a meaningful improvement for latency-sensitive workloads. At 128K context, prefill compute alone exceeds 2.5 seconds on a single TP=8 group; multi-node TP or speculative prefill optimisations are required to meet any reasonable TTFT SLO.

---

## 33.15 Summary

Inference serving imposes a fundamentally different set of network requirements compared to distributed training. The key engineering choices are:

1. **Disaggregate prefill from decode** when TTFT SLO tightness and prompt length variance create ITL jitter on shared hardware — but accept the KV transfer cost (§33.3).
2. **Size the KV cache transfer budget** using the per-token formula (320 KB/token for Llama-3 70B in BF16) to determine whether NVLink intra-rack co-location or RoCEv2 cross-node transfer is required (§33.4).
3. **Use NIXL** as the unified transfer API across backends (UCX/RoCEv2 for cross-node, NVLink UCX for intra-rack, GDS for SSD offload); Mooncake provides an alternative RDMA path with content-addressed cross-request prefix reuse (§33.5).
4. **Route requests with KV cache awareness** — prefix-consistent hashing or trie routing — rather than round-robin, to achieve 57× TTFT improvements on the same hardware through cache hit amplification (§33.12).
5. **Deploy Grove** for topology-aware gang scheduling that ensures prefill/decode pairs land on the highest-bandwidth interconnect boundary available (§33.8.3).
6. **Use the Gateway API Inference Extension** (InferencePool + Endpoint Picker) rather than plain GRPCRoute for production inference routing — it adds GPU-metric-aware, KV-cache-aware pod selection that plain load balancers cannot provide (§33.12.4).
7. **Size the inference spine at 2:1 oversubscription** (not 1:1 as in training) because inference KV traffic is bursty, not sustained (§33.13.1).

---

## References

- **vLLM Disaggregated Prefilling documentation**: https://docs.vllm.ai/en/latest/features/disagg_prefill/
- **vLLM Disaggregated Prefill example (P2pNcclConnector / NixlConnector)**: https://docs.vllm.ai/en/latest/examples/online_serving/disaggregated_prefill/
- **NVIDIA Dynamo — Introducing Dynamo**: https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/
- **NVIDIA Dynamo — KV Cache Bottlenecks**: https://developer.nvidia.com/blog/how-to-reduce-kv-cache-bottlenecks-with-nvidia-dynamo/
- **NVIDIA NIXL**: https://developer.nvidia.com/dynamo (NIXL section)
- **NVIDIA Dynamo 1.0 production release**: https://developer.nvidia.com/blog/nvidia-dynamo-1-production-ready/
- **NVIDIA Grove — Kubernetes Inference Scheduling**: https://developer.nvidia.com/blog/streamline-complex-ai-inference-on-kubernetes-with-nvidia-grove/
- **Grove GitHub Repository**: https://github.com/ai-dynamo/grove
- **Grove Deployment Guide**: https://docs.nvidia.com/dynamo/latest/kubernetes/grove.html
- **llm-d architecture**: https://llm-d.ai/docs/architecture
- **llm-d KV cache aware routing**: https://developers.redhat.com/articles/2025/10/07/master-kv-cache-aware-routing-llm-d-efficient-ai-inference
- **llm-d prefill/decode disaggregation guide**: https://llm-d.ai/docs/guide/Installation/pd-disaggregation
- **Mooncake Transfer Engine**: https://kvcache-ai.github.io/Mooncake/
- **SGLang RadixAttention paper**: https://arxiv.org/abs/2312.07104
- **SGLang HiCache**: https://www.lmsys.org/blog/2025-09-10-sglang-hicache/
- **SGLang GitHub**: https://github.com/sgl-project/sglang
- **TensorRT-LLM documentation**: https://nvidia.github.io/TensorRT-LLM/
- **Characterizing Communication Patterns in Distributed LLM Inference (arXiv:2507.14392)**: https://arxiv.org/abs/2507.14392
- **Gateway API Inference Extension — Kubernetes Blog**: https://kubernetes.io/blog/2025/06/05/introducing-gateway-api-inference-extension/
- **GRPCRoute — Gateway API**: https://gateway-api.sigs.k8s.io/api-types/grpcroute/
- **SkyWalker Locality-Aware LLM Load Balancer (arXiv:2505.24095)**: https://arxiv.org/abs/2505.24095
- **Decentralized Speculative Decoding (arXiv:2511.11733)**: https://arxiv.org/abs/2511.11733
- **Prefill is Compute-Bound, Decode is Memory-Bound (Towards Data Science)**: https://towardsdatascience.com/prefill-is-compute-bound-decode-is-memory-bound-why-your-gpu-shouldnt-do-both/
- **NVIDIA Triton Inference Server — Dynamic Batching**: https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/user_guide/optimization.html
- **KV Cache Size Calculator (LMCache)**: https://lmcache.ai/kv_cache_calculator.html
- **GPUDirect RDMA overview (NVIDIA Developer)**: https://developer.nvidia.com/gpudirect
- **rdma-core / rxe documentation**: https://github.com/linux-rdma/rdma-core/blob/master/Documentation/rxe.md

---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
