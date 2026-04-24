# Chapter 4 — Interconnect Abstractions: UCX, LibFabric & PMIx

**Part I: Foundations** | ~15 pages

---

## Installation

### Ubuntu 24.04 Packages

```bash
sudo apt install -y libucx-dev ucx-utils libfabric-dev libfabric-bin libpmix-dev pkg-config
```

Verify the installed tools are reachable:

```bash
which ucx_info ucx_perftest    # from ucx-utils
which fi_info                  # from libfabric-bin
pkg-config --modversion ucx    # prints UCX version, e.g. 1.14.1
pkg-config --modversion libfabric
```

### Python Environment (uv)

```bash
# Install uv (fast Python package manager from Astral)
curl -Lsf https://astral.sh/uv/install.sh | sh

# Create and activate a virtual environment
uv venv .venv && source .venv/bin/activate

# Install pyucx if available; it wraps the UCX C API via Cython
uv pip install pyucx
# Note: pyucx availability varies by platform. If the above fails with
# "No matching distribution found", the UCX Python bindings must be built
# from source (https://github.com/openucx/ucx/tree/master/bindings/python)
# or accessed via the C API directly. The examples below use both paths.
```

### CMakeLists.txt for UCX C Example

Save the following as `CMakeLists.txt` alongside your UCX C source files:

```cmake
cmake_minimum_required(VERSION 3.20)
project(ucx_example C)

find_package(PkgConfig REQUIRED)
pkg_check_modules(UCX REQUIRED ucx)

add_executable(ucx_perf ucx_perf.c)
target_include_directories(ucx_perf PRIVATE ${UCX_INCLUDE_DIRS})
target_link_libraries(ucx_perf ${UCX_LIBRARIES})
```

Build:

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

---

## 4.1 The Portability Problem

A GPU cluster is never purely homogeneous. InfiniBand HDR links connect some server pairs; RoCEv2 over Ethernet connects others; NVLink connects GPUs within a node; shared memory connects processes on the same host. A collective library like NCCL must run efficiently over all of these without reimplementing the low-level transport for each one.

UCX and LibFabric solve this by providing a **unified, transport-agnostic messaging API** that selects the fastest available transport at runtime. NCCL uses UCX; OpenMPI can use either. PMIx solves the orthogonal problem of *bootstrapping* — how do N processes on N hosts find each other at startup?

---

## 4.2 UCX — Unified Communication X

### 4.2.1 Architecture

UCX has a layered architecture:

```
Application (NCCL, OpenMPI, UCC)
        │
    UCT API   ← transport-agnostic tagged/RMA messaging
        │
 ┌──────┴──────────────────────────────┐
 │  UCT transports (components)        │
 │  rc_mlx5  ud_mlx5  dc_mlx5         │  ← InfiniBand / RoCE
 │  tcp      sockcm                   │  ← TCP fallback
 │  shmem    cma   knem               │  ← shared memory
 │  cuda_copy  cuda_ipc               │  ← GPU-to-GPU
 └─────────────────────────────────────┘
```

The **UCT layer** (transport) provides: active messages, tagged send/recv, RMA (put/get), and atomics.

The **UCP layer** (protocol) adds: connection management, multi-rail, rendez-vous for large messages, and a higher-level tagged API used by MPI.

### 4.2.2 Endpoint and Transport Selection

```python
# Python via pyucx (illustrative)
import ucx

ctx = ucx.init(config={"TLS": "rc,shmem,tcp"})  # allow these transports
endpoint = ctx.create_endpoint(remote_address)

# UCX selects the fastest transport automatically:
# - rc_mlx5 if both endpoints have Mellanox IB/RoCE
# - shmem if same host
# - tcp as universal fallback
```

Transport selection uses UCX's *scoring* mechanism: each transport advertises bandwidth, latency, and overhead; UCX picks the best for the message size (short messages favor low-latency UD; large messages favor high-bandwidth RC).

### 4.2.3 UCX Environment Variables for Tuning

```bash
# Force a specific transport
UCX_TLS=rc_mlx5,shmem

# Specify which network device to use
UCX_NET_DEVICES=mlx5_0:1

# Enable detailed logging
UCX_LOG_LEVEL=DEBUG

# Disable GPU-direct (for debugging)
UCX_CUDA_COPY_DMABUF=no

# Control rendezvous threshold (switch from eager to rendezvous at N bytes)
UCX_RNDV_THRESH=8192

# Show selected transports at init
UCX_SOCKADDR_TLS_PRIORITY=rdmacm
```

Run `ucx_info -d` to enumerate all detected transports and their capabilities on the current host.

### 4.2.4 UCC — Unified Collective Communication

UCC is the collective-operation companion to UCX, providing AllReduce, AllGather, Broadcast, Barrier, and more with pluggable backends:

```
UCC API
   │
 ┌─┴──────────────────────────────┐
 │  UCC components (CLs)          │
 │  cl_basic   ← basic CPU impl  │
 │  cl_hier    ← hierarchical    │
 │  tl_ucp     ← UCX transport   │
 │  tl_sharp   ← in-network SHARP│
 │  tl_nccl    ← NCCL backend    │
 └────────────────────────────────┘
```

SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) offloads AllReduce computation to switch ASICs — the first `tl_sharp` component competes allreduce in the network without involving host CPUs at all.

---

## 4.3 LibFabric (OFI) — OpenFabrics Interface

### 4.3.1 Design Philosophy

While UCX is tightly coupled to RDMA semantics (verbs-heritage), LibFabric takes a more abstract approach: define a minimal set of capabilities, let providers implement them, and let the application request only what it needs.

```
Application
    │
libfabric API (fi_*)
    │
 ┌──┴────────────────────────────────┐
 │  Providers                        │
 │  verbs    ← libibverbs / RoCE/IB │
 │  efa      ← AWS Elastic Fabric   │
 │  psm3     ← Intel Omni-Path      │
 │  cxi      ← HPE Slingshot        │
 │  tcp      ← TCP fallback         │
 │  shm      ← shared memory        │
 └───────────────────────────────────┘
```

### 4.3.2 Core Abstractions

| Object | Description |
|---|---|
| `fi_info` | Capability negotiation — what the provider supports |
| `fid_fabric` | Top-level fabric object |
| `fid_domain` | Memory registration and QoS domain |
| `fid_ep` | Endpoint (connected, connectionless, or scalable) |
| `fid_cq` | Completion queue |
| `fid_av` | Address vector — maps fi_addr_t to transport addresses |
| `fid_mr` | Memory region |

### 4.3.3 Minimal Send/Recv Example

```c
struct fi_info *hints = fi_allocinfo();
hints->caps = FI_MSG;
hints->ep_attr->type = FI_EP_RDM;  // reliable datagram

struct fi_info *info;
fi_getinfo(FI_VERSION(1,14), NULL, NULL, 0, hints, &info);

struct fid_fabric *fabric;
fi_fabric(info->fabric_attr, &fabric, NULL);

struct fid_domain *domain;
fi_domain(fabric, info, &domain, NULL);

struct fid_ep *ep;
fi_endpoint(domain, info, &ep, NULL);

// ... bind CQ and AV, enable endpoint ...

// Post send
fi_send(ep, buf, len, fi_mr_desc(mr), dest_addr, NULL);

// Poll completion
struct fi_cq_entry entry;
fi_cq_read(cq, &entry, 1);
```

### 4.3.4 Provider Selection

```bash
# List available providers
fi_info --list

# Get info for a specific provider
fi_info -p verbs

# Run latency/bandwidth pingpong test with specific provider
fi_pingpong -p cxi     # HPE Slingshot
fi_pingpong -p verbs   # InfiniBand / RoCE
```

---

## 4.4 PMIx — Process Management Interface for Exascale

### 4.4.1 The Bootstrap Problem

When a 1000-rank MPI job starts, every rank needs to know the network address of every other rank before any MPI communication can begin. This "wire-up" phase must complete before training starts. PMIx is the standard protocol for this.

### 4.4.2 PMIx Architecture

```
Job scheduler (SLURM / PBS)
        │  spawns
    PMIx server (per-node daemon, e.g., prte, prrte)
        │  PMIx wire protocol
Application process (MPI rank / NCCL rank)
        │  calls
    PMIx client library (libpmix)
```

### 4.4.3 Key Operations

```c
// Initialize PMIx client
PMIx_Init(&proc, NULL, 0);

// Publish local endpoint info (e.g., UCX worker address)
pmix_value_t value;
PMIX_VALUE_LOAD(&value, ucx_addr, ucx_addr_len, PMIX_BYTE_OBJECT);
PMIx_Put(PMIX_LOCAL, "UCX_ADDR", &value);
PMIx_Commit();  // flush to PMIx server

// Barrier — wait for all ranks to commit
PMIx_Fence(NULL, 0, NULL, 0);

// Fetch peer's endpoint info
pmix_proc_t peer = { .nspace = "myjob", .rank = 42 };
PMIx_Get(&peer, "UCX_ADDR", NULL, 0, &val);
// val now contains rank 42's UCX worker address
```

### 4.4.4 Fault Notification

PMIx v5 adds event notification for detecting rank failures:

```c
// Register handler for process termination events
PMIx_Register_event_handler(
    &PMIX_ERR_PROC_TERM, 1,
    NULL, 0,
    rank_failure_handler,
    NULL, NULL
);
```

This is the foundation for fault-tolerant collective operations — a long-sought capability for multi-day training jobs where hardware failures are expected.

---

## 4.5 CXL — Coherent Memory Fabric

CXL (Compute Express Link) is a PCIe-based protocol for attaching memory and accelerators to CPUs with cache-coherent access. It operates at three protocol layers:

| Protocol | Purpose |
|---|---|
| **CXL.io** | PCIe-compatible I/O transactions (config, MMIO) |
| **CXL.cache** | Device (GPU/accelerator) caches host memory coherently |
| **CXL.mem** | Host accesses device-attached memory as regular DRAM |

### Relevance for AI Clusters

- **Memory pooling (CXL.mem, Type 3):** A CXL memory expander appears to the CPU as regular DRAM. Multiple CPUs can share a CXL memory pool — enabling heterogeneous memory topologies where GPU-adjacent DRAM is shared across multiple hosts.
- **GPU-CPU coherence (CXL.cache, Type 2):** A GPU can cache host memory coherently, eliminating explicit `cudaMemcpy` for certain access patterns. NVIDIA's Grace Hopper Superchip uses a variant of this.
- **Multi-host CXL switching (CXL 3.0):** A CXL switch can expose a shared memory pool to multiple hosts — effectively a memory-level fabric above the PCIe/Ethernet boundary.

CXL 3.0 is still being deployed (2024–2026) but represents the future of memory fabric architecture for AI hardware.

---

## Lab Walkthrough 4 — UCX Transport Inspection

This walkthrough takes you from zero to a measured, transport-differentiated performance comparison using the tools installed in the Installation section. All steps work on a single Ubuntu 24.04 host (loopback and shared-memory transports are sufficient). Steps that require an IB/RoCE NIC are clearly marked optional.

**Estimated time:** 30–45 minutes  
**Prerequisites:** UCX packages and libfabric installed per the Installation section.

---

### Step 1 — Run ucx_info -d and Interpret the Transport List

`ucx_info -d` enumerates every transport (UCT component) UCX detected on this host, along with its capabilities. This is the first diagnostic step whenever UCX performance is unexpected.

```bash
ucx_info -d
```

Full expected output (on a host with no IB NIC, loopback + TCP + shared memory only):

```
#===========================================================================
# UCX version=1.14.1 revision 0
#===========================================================================
# Memory domain: posix
#   Component: posix
#     alloc_methods: mmap
#     rkey_packed_size: 24 bytes
# Memory domain: sysv
#   Component: sysv
#     alloc_methods: mmap sysv
#     rkey_packed_size: 24 bytes
# Memory domain: tcp
#   Component: tcp
#     alloc_methods: heap
# Transport: self
#   Device: memory
#   capabilities: AM RMA AMO32 AMO64 FLUSH
#   bandwidth: 0.00 MB/s
#   latency: 0 nsec
# Transport: shmem
#   Device: memory
#   capabilities: AM TAG RMA AMO32 AMO64
#   bandwidth: 10240.00 MB/s    ← in-process shared memory
#   latency: 30 nsec
# Transport: tcp
#   Device: lo port 1
#   capabilities: AM TAG
#   bandwidth: 1228.80 MB/s     ← loopback TCP (limited by software)
#   latency: 1000 nsec
# Transport: tcp
#   Device: eth0 port 1
#   capabilities: AM TAG
#   bandwidth: 1228.80 MB/s
#   latency: 1000 nsec
```

On a host with a Mellanox/NVIDIA InfiniBand or RoCE NIC you will additionally see:

```
# Transport: rc_mlx5
#   Device: mlx5_0 port 1
#   capabilities: AM TAG RMA AMO32 AMO64 FLUSH
#   bandwidth: 24576.00 MB/s   ← HDR 200G IB
#   latency: 600 nsec
# Transport: ud_mlx5
#   Device: mlx5_0 port 1
#   capabilities: AM TAG
#   bandwidth: 24576.00 MB/s
#   latency: 600 nsec
# Transport: dc_mlx5
#   Device: mlx5_0 port 1
#   capabilities: AM TAG RMA AMO32 AMO64
#   bandwidth: 24576.00 MB/s
```

Key columns to read:
- **capabilities** — `AM`=active messages, `TAG`=tagged send/recv, `RMA`=put/get, `AMO`=atomics. Not all transports support all operations; UCX picks from those that do.
- **bandwidth** — peak bandwidth in MB/s as self-reported. `shmem` is limited by memory bandwidth (~10–100 GB/s); `tcp` is limited by the NIC line rate; `rc_mlx5` reflects wire speed.
- **latency** — baseline one-way latency in nanoseconds. Shared memory is 30–100 ns; TCP loopback is ~1–10 µs; IB RC is 600 ns–2 µs.

Also run with `-v` for verbose version info and with `-c` for the full UCX configuration dump:

```bash
ucx_info -c 2>&1 | head -60   # show all tuneable parameters and their defaults
```

---

### Step 2 — Run ucx_perftest in Loopback Mode

`ucx_perftest` is the built-in micro-benchmark. In single-process loopback mode it tests the local transport stack end-to-end.

```bash
# Tagged message bandwidth — measures throughput for large messages
ucx_perftest -t tag_bw
```

Expected output:

```
+--------------+--------------+---------------------+-----------------------+
|    Test      | # iterations |    BW (MB/sec)      |  msgrate (Mpps)       |
|              |              |   min / max / avg   |   min / max / avg     |
+--------------+--------------+---------------------+-----------------------+
| tag_bw       |        10000 |  9845 / 9921 / 9883 |  0.094 / 0.095 / 0.094|
+--------------+--------------+---------------------+-----------------------+
```

Column meanings:
- **# iterations** — number of messages sent; more = more stable average.
- **BW (MB/sec)** — throughput measured as bytes / elapsed time. `min/max/avg` cover all message sizes tested.
- **msgrate (Mpps)** — millions of messages per second; most relevant for small-message latency-bound workloads.

Run additional test types:

```bash
# RMA put bandwidth — tests remote memory write throughput
ucx_perftest -t put_bw
```

Expected (loopback, shared memory):

```
+--------------+--------------+---------------------+-----------------------+
| put_bw       |        10000 | 10240 /10512 /10387 |  0.099 / 0.100 / 0.099|
+--------------+--------------+---------------------+-----------------------+
```

```bash
# RMA get latency — tests remote memory read latency (round-trip / 2)
ucx_perftest -t get_lat
```

Expected:

```
+-----------+-------------+---------------------+---------------------------+
|   Test    | # iterations|  latency (usec)     |  overhead (usec)          |
|           |             | min / max / avg     |  min / max / avg          |
+-----------+-------------+---------------------+---------------------------+
| get_lat   |       10000 |  0.16 / 0.34 / 0.19 |  0.08 / 0.17 / 0.10      |
+-----------+-------------+---------------------+---------------------------+
```

- **latency (usec)** — one-way latency estimate (round-trip / 2). For shared memory on a modern CPU, 0.15–0.25 µs is typical.
- **overhead (usec)** — software overhead (UCX stack processing time), excluding actual data transfer.

---

### Step 3 — Force UCX_TLS=tcp vs UCX_TLS=shmem and Compare Bandwidth

UCX transport selection is fully overridable via the `UCX_TLS` environment variable. This is the primary tuning knob and also the primary debugging lever when you suspect wrong transport selection.

**Baseline — let UCX auto-select (should choose shmem on same host):**

```bash
ucx_perftest -t tag_bw 2>&1 | grep -E "tag_bw|BW"
```

Expected: ~9000–10500 MB/s (shmem path, memory-bandwidth-limited).

**Force TCP transport:**

```bash
UCX_TLS=tcp ucx_perftest -t tag_bw 2>&1 | grep -E "tag_bw|BW"
```

Expected output:

```
| tag_bw       |        10000 |   978 / 1124 /  1052 |  0.009 / 0.011 / 0.010|
```

TCP loopback (via the kernel TCP/IP stack) is ~10x slower than shared memory for bulk data. This is the penalty paid when UCX falls back to TCP on a misconfigured cluster where IB/RoCE is not visible.

**Force shared memory transport (explicit):**

```bash
UCX_TLS=shmem ucx_perftest -t tag_bw 2>&1 | grep -E "tag_bw|BW"
```

Expected output:

```
| tag_bw       |        10000 |  9876 / 10201 / 9998 |  0.095 / 0.097 / 0.095|
```

**Summary table of expected results on a typical server:**

| UCX_TLS setting | Loopback BW | Latency | Notes |
|---|---|---|---|
| (auto) | ~10 GB/s | ~0.2 µs | UCX chooses shmem when co-located |
| `shmem` | ~10 GB/s | ~0.2 µs | Explicit shared-memory path |
| `tcp` | ~1 GB/s | ~5–20 µs | Kernel TCP loopback, 10x slower |
| `rc` (IB NIC required) | ~24 GB/s | ~1 µs | RDMA RC over InfiniBand or RoCE |
| `rc_mlx5` (IB NIC required) | ~24 GB/s | ~0.6 µs | Optimized mlx5 driver path |

If you have an IB or RoCE NIC:

```bash
# Find the device name first
ucx_info -d | grep "Device:" | grep mlx
# Example output: Device: mlx5_0 port 1

UCX_TLS=rc UCX_NET_DEVICES=mlx5_0:1 ucx_perftest -t tag_bw
# Expected: ~24576 MB/s (HDR200 IB) or ~12288 MB/s (HDR100 / 100GbE RoCE)
```

---

### Step 4 — Find the Transport Selection Log Line with UCX_LOG_LEVEL=DEBUG

When a performance problem appears, the first step is confirming which transport UCX actually chose. The `UCX_LOG_LEVEL=DEBUG` environment variable produces verbose output including the transport negotiation.

```bash
UCX_LOG_LEVEL=DEBUG ucx_perftest -t tag_bw 2>&1 | head -80
```

The output is large. Look for lines containing `select` or `transport`:

```bash
UCX_LOG_LEVEL=DEBUG ucx_perftest -t tag_bw 2>&1 | grep -i "select\|transport\|tl\|proto"
```

Key log lines to identify:

```
ucx  DEBUG  ucp_wireup.c:421                  ep 0x... created with transports:
ucx  DEBUG  ucp_wireup.c:422                    lane[0]: rma/shmem->memory
ucx  DEBUG  ucp_wireup.c:423                    lane[1]: tag/shmem->memory
ucx  DEBUG  ucp_wireup.c:424                    lane[2]: am/shmem->memory
```

These lines confirm that UCX selected `shmem` for RMA, tagged, and active-message lanes. If you forced `UCX_TLS=tcp` instead:

```
UCX_LOG_LEVEL=DEBUG UCX_TLS=tcp ucx_perftest -t tag_bw 2>&1 | \
  grep -i "select\|transport\|lane"
```

Expected:

```
ucx  DEBUG  ucp_wireup.c:421                  ep 0x... created with transports:
ucx  DEBUG  ucp_wireup.c:422                    lane[0]: am/tcp->lo
ucx  DEBUG  ucp_wireup.c:423                    lane[1]: tag/tcp->lo
```

Additional useful debug filters:

```bash
# See memory registration decisions
UCX_LOG_LEVEL=DEBUG ucx_perftest -t put_bw 2>&1 | grep -i "mem_reg\|mmap\|mr"

# See rendezvous vs eager protocol selection
UCX_LOG_LEVEL=DEBUG ucx_perftest -t tag_bw 2>&1 | grep -i "rndv\|eager\|thresh"
```

The rendezvous threshold line looks like:

```
ucx  DEBUG  ucp_request.c:287  msg size 65536 >= rndv_thresh 8192, using rendezvous
```

This confirms that messages above `UCX_RNDV_THRESH` (default 8192 bytes) use the rendez-vous protocol (zero-copy RDMA) instead of eager copy — important for understanding latency cliffs in profiling data.

---

### Step 5 — Interpret fi_info --list Output

LibFabric's `fi_info` tool is the equivalent of `ucx_info -d` for the OFI provider stack.

```bash
fi_info --list
```

Expected output on Ubuntu 24.04 with standard packages (no specialized NICs):

```
provider: psm3
    version: 1.16.0
    type: FI_EP_RDM
    protocol: FI_PROTO_PSMX3
provider: tcp
    version: 1.0.0
    type: FI_EP_MSG
    protocol: FI_PROTO_SOCK_TCP
provider: tcp
    version: 1.0.0
    type: FI_EP_RDM
    protocol: FI_PROTO_RXM
provider: udp
    version: 1.1.0
    type: FI_EP_RDM
    protocol: FI_PROTO_UDP
provider: shm
    version: 2.0.0
    type: FI_EP_RDM
    protocol: FI_PROTO_SHM
provider: efa (if on AWS EC2 with EFA NIC)
    version: 1.18.0
    type: FI_EP_RDM
    protocol: FI_PROTO_EFA
provider: verbs (if IB/RoCE NIC present)
    version: 1.0.0
    type: FI_EP_MSG
    protocol: FI_PROTO_RDMA_CM_IB_RC
```

How to interpret each field:

- **provider** — the OFI provider name. Maps to an underlying fabric: `verbs`=IB/RoCE libibverbs, `efa`=AWS EFA, `psm3`=Intel Omni-Path 3, `cxi`=HPE Slingshot, `tcp`=kernel TCP, `shm`=POSIX shared memory.
- **version** — provider version, independent of libfabric version.
- **type** — endpoint type.
  - `FI_EP_MSG` — connection-oriented (like TCP; requires explicit connect/accept).
  - `FI_EP_RDM` — reliable datagram (connectionless but reliable; preferred for MPI-style usage).
  - `FI_EP_DGRAM` — unreliable datagram.
- **protocol** — wire protocol the provider uses internally. `FI_PROTO_RDMA_CM_IB_RC` means RDMA Connection Manager + InfiniBand Reliable Connected QP.

Get detailed capability information for a specific provider:

```bash
fi_info -p shm
```

Expected (shared memory provider):

```
provider: shm
    fabric: shm
    domain: shm
    version: 2.0.0
    type: FI_EP_RDM
    protocol: FI_PROTO_SHM
    caps: [ FI_MSG | FI_TAGGED | FI_RMA | FI_ATOMICS | FI_DIRECTED_RECV ]
    mode: [ ]
    addr_format: FI_ADDR_STR
    src_addrlen: 0
    dest_addrlen: 0
    tx_attr:
        size: 65536
        iov_limit: 4
        inject_size: 4096
        rma_iov_limit: 4
    rx_attr:
        size: 65536
        iov_limit: 4
    domain_attr:
        name: shm
        threading: FI_THREAD_SAFE
        mr_mode: [ FI_MR_VIRT_ADDR | FI_MR_ALLOCATED | FI_MR_PROV_KEY ]
        resource_mgmt: FI_RM_ENABLED
```

Key attributes:
- **caps** — `FI_MSG` (send/recv), `FI_TAGGED` (tagged matching), `FI_RMA` (put/get), `FI_ATOMICS` (compare-and-swap, fetch-add). Applications request only what they need via `hints->caps`.
- **inject_size 4096** — messages up to 4096 bytes can be "injected" (send returns immediately, no completion needed). Above this, the application must wait for a send completion.
- **mr_mode** — memory registration semantics. `FI_MR_VIRT_ADDR` means the provider uses virtual addresses in RMA operations (simpler for the application).

To check if the `verbs` provider sees your RoCE/IB device:

```bash
fi_info -p verbs 2>&1
# If no verbs devices: "fi_getinfo: -61 (No data available)"
# If found: shows one entry per port per device
```

---

## Summary

- UCX provides a transport-agnostic messaging layer that selects IB/RoCE/shmem/TCP based on what's available; NCCL and OpenMPI both use it.
- LibFabric (OFI) takes a more abstract provider model, dominant in AWS EFA, HPE Slingshot, and Intel Omni-Path environments.
- PMIx handles the bootstrap wire-up problem — how N ranks discover each other — and is extending into fault notification for resilient training.
- CXL is the emerging coherent memory fabric that blurs the boundary between GPU HBM, CPU DRAM, and network-attached memory.

---

## References

- UCX documentation: openucx.readthedocs.io
- UCC documentation: openucx.github.io/ucc
- LibFabric documentation: ofiwg.github.io/libfabric
- PMIx Standard v5: pmix.org
- CXL Consortium specification: computeexpresslink.org


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).