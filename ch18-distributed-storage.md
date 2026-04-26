# Chapter 18 — Distributed Storage for AI Clusters

**Part VI: Storage & Adjacent Infrastructure** | ~25 pages

---

## Introduction

Storage in an AI cluster is not a secondary concern — it is a first-class networking problem. A 1000-GPU training run generates checkpoint writes that, when issued simultaneously from all ranks, constitute an 800 GB burst hitting the storage fabric in seconds. Dataset streaming for the next training epoch begins the moment the previous one ends. Model weights must be loaded into GPU memory before the first forward pass can begin. Each of these workloads has fundamentally different access patterns, latency tolerances, and bandwidth requirements, and a storage architecture that optimizes for one will be suboptimal or actively harmful for the others.

This chapter examines the open-source and commercial storage systems deployed in production AI clusters, analyzing each through the lens of what it demands from the network fabric and how well it serves the three canonical storage workloads: dataset streaming, checkpointing, and model serving. The systems covered span the full spectrum: **Ceph** provides a unified POSIX/block/object platform on commodity hardware; **Lustre** delivers HPC-grade parallel throughput optimized for large sequential I/O across thousands of simultaneous clients; **DAOS** (Distributed Asynchronous Object Storage) eliminates the OS I/O stack entirely using **SPDK** and Storage Class Memory to achieve sub-100 microsecond checkpoint latency; **WEKA** (**WekaFS**) combines **DPDK**-accelerated networking with direct **NVMe** access to close the performance gap between **Lustre** and **DAOS** without requiring specialized hardware; and **MinIO** provides the S3-compatible object interface that every ML framework already speaks.

A thread running through all these systems is the relationship between storage and training fabric topology. Checkpoint traffic and gradient AllReduce traffic must not share bandwidth — a simultaneous checkpoint from a thousand GPUs will saturate any shared uplink and stall AllReduce operations in the most performance-critical phase of training. This chapter explains how to design storage network isolation using separate VLANs or physical switching, and how to configure QoS priority classes so that storage traffic yields to gradient traffic when contention occurs.

Readers will learn the architecture of each system, its configuration for AI workloads, the **PromQL** metrics to watch for storage-induced training stalls, and how to benchmark each tier with **fio** to establish per-system throughput baselines.

This chapter connects directly to Chapter 6 (**SPDK**/**NVMe-oF**, on which **DAOS**'s I/O path is built), Chapter 7 (**eBPF**/**XDP** for storage-traffic policing), and Chapter 20 (distributed training fabric demands, which quantifies exactly how much checkpoint bandwidth a given model size requires). Chapter 19 (GPU collective communications) explains the AllReduce patterns that storage traffic must not disrupt.

---

## Installation

This chapter benchmarks and compares five storage systems across the three canonical AI cluster workloads, so each must be installed and reachable from the same test host. **Ceph** is bootstrapped via **cephadm**, which deploys and manages the full suite of monitor, manager, OSD, and MDS daemons as containers, providing the unified POSIX/block/object platform used for general-purpose cluster storage. The **Lustre** client and server packages are installed to exercise the parallel file system that underpins most HPC-origin AI clusters and delivers the high sequential throughput that dataset streaming requires. The **DAOS** client libraries are installed to evaluate **SPDK**-based object storage that bypasses the OS I/O stack entirely for sub-100-microsecond checkpoint commit latency. **MinIO** is deployed as a single-node S3-compatible server to serve as the object-storage tier that ML frameworks address via standard **boto3** or **s3fs** interfaces. The **fio** benchmarking tool is installed across all tiers to produce consistent, comparable throughput and latency baselines under sequential, random, and mixed read/write profiles that correspond to the actual access patterns of training, checkpointing, and model loading.

### System packages (Ubuntu 24.04)

```bash
sudo apt update
sudo apt install -y cephadm fio curl wget
```

### Ceph via cephadm (bootstraps the full daemon set)

```bash
# Ubuntu 24.04 ships cephadm in apt:
sudo apt install -y cephadm

# Or fetch directly from the Ceph release page:
curl --silent --remote-name --location \
    https://download.ceph.com/rpm-2023.2/el9/noarch/cephadm
chmod +x cephadm && sudo mv cephadm /usr/local/bin/
cephadm --version   # cephadm version 18.2.x (Reef)
```

### MinIO server and client

```bash
# MinIO server binary
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio && sudo mv minio /usr/local/bin/
minio --version   # minio version RELEASE.20xx-...

# MinIO client (mc)
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/
mc --version   # mc version RELEASE.20xx-...
```

### fio (flexible I/O tester)

```bash
sudo apt install -y fio
fio --version   # fio-3.x
```

### Python environment for S3/object storage scripting

```bash
# Install uv if not present
curl -LsSf https://astral.sh/uv/install.sh | sh

uv venv .venv && source .venv/bin/activate
uv pip install boto3 s3fs minio
python -c "import boto3; print(boto3.__version__)"
```

### WEKA client (requires WEKA cluster; shown for reference)

```bash
# WEKA client (requires WEKA cluster; shown for reference)
# Download weka agent from your WEKA cluster management UI
# curl http://<weka-backend>:14000/dist/v1/install | sudo sh
# Verify:
# weka version
# Note: Lab exercises use fio against local NVMe as a stand-in
```

---

## 18.1 Storage as a Fabric Concern

Training a large model has three distinct storage demands:

| Workload | Access Pattern | Volume | Latency Req |
|---|---|---|---|
| **Dataset streaming** | Large sequential reads, many concurrent readers | Tens of TB | Throughput > latency |
| **Checkpointing** | Large sequential writes, periodic, all ranks simultaneously | 100 GB–1 TB | Must complete fast |
| **Model serving** | Read-once at startup, then compute-bound | 10–700 GB | Low startup latency |

These three demands have different optimal storage systems. A cluster that uses one storage system for all three will be suboptimal for at least two of them.

---

## 18.2 Ceph — Unified Distributed Object Storage

Ceph is the most deployed open-source distributed storage system. It provides three interfaces on top of a common object storage layer (RADOS):
- **RBD (RADOS Block Device):** block storage for VMs and containers
- **CephFS:** POSIX filesystem for shared file access
- **RGW (RADOS Gateway):** S3/Swift-compatible object storage

### 18.2.1 Architecture

```
Clients (trainers, checkpointing, Kubernetes PVCs)
        │ via librados / CephFS / RBD / S3
RADOS:
  MONs (Monitor daemons) — cluster quorum, CRUSH map, OSD map
  MGRs (Manager daemons)  — metrics, dashboard, orchestration
  OSDs (Object Storage Daemons) — one per drive; store objects, handle replication
  MDSs (Metadata Servers)  — CephFS metadata (directories, file attributes)
```

### 18.2.2 CRUSH Map — Data Placement

CRUSH (Controlled Replication Under Scalable Hashing) deterministically places objects across OSDs without a lookup table:

```bash
# Inspect CRUSH topology
ceph osd tree
# ID CLASS WEIGHT TYPE NAME          STATUS
# -1       120.00 root default
# -2        60.00   host gpu-storage-01
#  0   ssd  10.00     osd.0          up
#  1   ssd  10.00     osd.1          up
# ...

# Create a rule that spreads replicas across hosts (not just OSDs)
ceph osd crush rule create-replicated replicated-hosts default host ssd
```

### 18.2.3 Pool and Performance Tuning

```bash
# Create pool for checkpoint data (3-way replication, SSD class)
ceph osd pool create ai-checkpoints 128 128 replicated replicated-hosts
ceph osd pool set ai-checkpoints crush_rule replicated-hosts
ceph osd pool application enable ai-checkpoints rbd

# Key tuning for large sequential I/O (checkpoints):
# Increase OSD journal queue
ceph config set osd osd_op_queue wpq
ceph config set osd bluestore_cache_size_ssd 4294967296  # 4GB cache

# RBD image for checkpoint volume
rbd create ai-checkpoints/checkpoint-vol --size 2T --image-feature layering
rbd map ai-checkpoints/checkpoint-vol
mkfs.xfs /dev/rbd0
mount /dev/rbd0 /mnt/checkpoints
```

### 18.2.4 CephFS for Shared Dataset Access

CephFS allows all training ranks to access the same dataset directory simultaneously:

```bash
# Create CephFS
ceph fs new ai-datasets ai-datasets-metadata ai-datasets-data

# Mount on each training host
mount -t ceph mon1:6789,mon2:6789,mon3:6789:/ /mnt/datasets \
    -o name=admin,secret=$(ceph auth get-key client.admin)

# Tune for large-file streaming workloads
ceph config set mds mds_cache_memory_limit 8589934592  # 8GB MDS cache
ceph config set client client_readahead_max_bytes 67108864  # 64MB readahead
```

---

## 18.3 Lustre — HPC Parallel Filesystem

Lustre is the dominant filesystem in TOP500 HPC systems. Unlike Ceph, Lustre is purpose-built for parallel access to large files by thousands of simultaneous clients.

### 18.3.1 Architecture

```
Clients (compute nodes, GPU servers)
    │ via Lustre client (kernel module)
    │
MGS (Management Server) — configuration database
MDS (Metadata Server)   — namespace, directories, file attributes
    │
OSS (Object Storage Servers) — one or more per OST
  OST (Object Storage Target) — actual data storage (one per drive/RAID)
```

Data striping: a large file is split into chunks, each stored on a different OST. With 64 OSTs, a 1 TB file can be written in parallel from 64 clients at 64× the single-drive bandwidth.

### 18.3.2 Installation and Configuration

```bash
# On MDS:
mkfs.lustre --mgs /dev/sdb
mkfs.lustre --mdt --mgsnode=mds01@tcp0 --index=0 /dev/sdc
mount -t lustre /dev/sdc /mnt/mdt

# On each OSS (one per OST):
mkfs.lustre --ost --mgsnode=mds01@tcp0 --index=0 /dev/sdd
mount -t lustre /dev/sdd /mnt/ost0

# On clients:
modprobe lustre
mount -t lustre mds01@tcp0:/lustre /mnt/lustre
```

### 18.3.3 Striping for AI Workloads

```bash
# Set stripe width for dataset directory (spread across all OSTs)
lfs setstripe -c -1 -S 4M /mnt/lustre/datasets
# -c -1: stripe across all OSTs
# -S 4M: 4MB stripe size (optimal for large sequential reads)

# Verify
lfs getstripe /mnt/lustre/datasets/imagenet.tar
# obdidx  objid  group    offset    count
#     0   12345  0              0   4194304
#     1   12346  0        4194304   4194304
#     ...   (one line per OST)

# Check filesystem load balance
lfs df -h
# OST0000  [OST] 10.0T   6.5T   3.5T  65%   /mnt/lustre[MDT:0]
# OST0001  [OST] 10.0T   5.8T   4.2T  58%   /mnt/lustre[OST:0]
```

### 18.3.4 LNet — Lustre Networking

Lustre uses LNet (Lustre Networking) as its transport abstraction:

```bash
# Configure LNet over InfiniBand (o2ib) or RoCE
cat > /etc/lnet.conf << EOF
net:
  - net type: o2ib
    local NI(s):
      - interfaces:
          - ib0
EOF

lnetctl import < /etc/lnet.conf
lnetctl net show   # verify
```

LNet supports RDMA transfers, making Lustre over InfiniBand capable of full-fabric bandwidth for checkpoint writes.

---

## 18.4 DAOS — Distributed Asynchronous Object Storage

DAOS (Intel/HPE) is purpose-built for NVMe SSDs and Intel Optane (SCM — Storage Class Memory). SCM (Storage Class Memory) refers to byte-addressable, persistent memory devices such as Intel Optane DCPMM or CXL-attached DRAMs that combine DRAM-class latency with the persistence of flash. SPDK (Storage Performance Development Kit) is an Intel open-source framework that drives NVMe devices from user space using DPDK-style polling, bypassing the Linux block layer and kernel I/O scheduler entirely. DAOS eliminates the OS I/O stack by using SPDK to access NVMe directly, achieving sub-100 µs I/O latency — orders of magnitude better than Lustre or Ceph.

CART (Communication and Remote Transactions) is the DAOS network transport library built on UCX and libfabric; it provides reliable, ordered message delivery and RPC semantics over RDMA fabrics without the overhead of a general-purpose messaging layer.

### 18.4.1 Architecture

```
DAOS Client (libdaos)
    │ CART (Communication and Remote Transactions) over UCX/libfabric
DAOS Server (per storage node)
    │ SPDK (user-space NVMe)
  SCM (Optane DCPMM / CXL Memory) — metadata + small objects
  NVMe SSDs                        — bulk data
```

### 18.4.2 Pool and Container Model

```bash
# Create a DAOS pool (50TB across 10 storage nodes)
dmg pool create --size=50T --scm-ratio=5 ai-pool

# Create a POSIX container
daos container create --type=POSIX --label=checkpoints ai-pool
daos fs mount ai-pool checkpoints /mnt/daos/checkpoints

# Write a checkpoint (directly via POSIX interface)
torch.save(model.state_dict(), "/mnt/daos/checkpoints/step_5000.pt")

# Or use native DAOS array API for ultra-low latency
```

### 18.4.3 DAOS for Checkpoint Performance

DAOS writes bypass the kernel and OS file cache entirely:
- Metadata goes to SCM (Optane/CXL): ~10 µs latency
- Data goes to NVMe via SPDK: ~100 µs latency
- No journaling overhead: epoch-based consistency model

Benchmark: writing a 400 GB checkpoint with DAOS over HDR InfiniBand: ~60 seconds at ~6 GB/s. With Ceph over the same fabric: ~300 seconds at ~1.3 GB/s.

---

## 18.5 WEKA — NVMe-Native Parallel Filesystem

WEKA (WekaFS) is a commercial parallel filesystem purpose-built for NVMe flash and high-performance RDMA fabrics. It occupies the space between Lustre (mature HPC filesystem, spinning-disk heritage) and DAOS (ultra-low-latency, Optane-native), offering sub-millisecond latency on NVMe without requiring specialized SCM hardware.

### 18.5.1 Architecture

```
WEKA Clients (compute nodes, GPU servers)
    │ via WEKA client daemon (user-space DPDK / kernel shim)
    │
WEKA Backend Cluster
    ├── Frontend processes  — client protocol termination, metadata
    ├── Compute processes   — data placement, erasure coding, tiering
    ├── Drive processes     — direct NVMe access (no kernel I/O stack)
    └── Management process  — cluster configuration, monitoring
```

Key design choices that differentiate WEKA from Lustre and Ceph:

- **DPDK-based network stack:** WEKA bypasses the kernel network stack entirely, using DPDK to drive the RDMA NIC (RoCEv2 or InfiniBand) from user space. This eliminates interrupt-driven I/O and system-call overhead on the data path.
- **SSD-native I/O:** WEKA drive processes own NVMe devices directly (similar to SPDK) — no ext4, XFS, or other intermediate filesystem layer. The on-disk format is WekaFS-native and optimized for 4 KB to multi-MB I/O.
- **Distributed metadata:** Unlike Lustre (single MDS) or Ceph (limited MDS scaling), WekaFS distributes metadata across all backend nodes, avoiding the MDS bottleneck that plagues Lustre at scale.
- **Erasure coding on NVMe:** WEKA uses software erasure coding (N+2, N+4 configurations) to provide fault tolerance without 3× replication overhead — important when NVMe capacity is expensive.

### 18.5.2 Why WEKA for AI Workloads

| Property | Lustre | Ceph RBD | DAOS | WEKA |
|---|---|---|---|---|
| Typical write latency | 1–5 ms | 5–50 ms | <0.1 ms (Optane) | 0.1–0.5 ms (NVMe) |
| Sequential throughput | Very high | High | Very high | Very high |
| Random 4K IOPS | Moderate | Moderate | Extreme | High |
| Hardware requirement | Any | Any | SCM + NVMe | NVMe (no SCM needed) |
| S3 compatibility | No | Yes (RGW) | No | Yes (native S3) |
| GPUDirect Storage | Via Lustre-GDRCOPY | No | Experimental | Yes (native) |
| Operational complexity | High | High | Very High | Medium |

GPUDirect Storage (GDS) is an NVIDIA technology that establishes a direct DMA path between NVMe storage and GPU memory over the PCIe fabric, removing the CPU and system DRAM from the data path entirely. WEKA's GDS integration allows GPU memory to receive data directly from the NVMe drives over the PCIe fabric, bypassing CPU DRAM entirely. For checkpoint restore (loading a 400 GB model into GPU memory), GDS can reduce restore time by 30–50% compared to CPU-mediated reads.

### 18.5.3 Client Installation

WEKA uses an agent-based installation model. The backend cluster runs the WEKA software stack; clients install a lightweight agent that connects to the cluster and presents a POSIX mountpoint:

```bash
# Step 1: Download and install the WEKA agent from your cluster's management UI
# (requires a running WEKA cluster with the management port accessible)
curl http://<weka-backend>:14000/dist/v1/install | sudo sh

# The installer:
# - Detects kernel version and downloads matching WEKA kernel module
# - Installs the weka CLI (/usr/bin/weka)
# - Starts the weka-agent systemd service

# Step 2: Verify agent is running
sudo systemctl status weka-agent
# ● weka-agent.service - WEKA agent
#      Loaded: loaded (/etc/systemd/system/weka-agent.service; enabled)
#      Active: active (running) since ...
#     Main PID: 12345 (weka-agent)

# Step 3: Check WEKA version
weka version
# WekaIO version 4.x.y.z (client)
# Cluster version: 4.x.y.z

# Step 4: Mount a WekaFS filesystem
sudo mkdir -p /mnt/weka
sudo mount -t wekafs <weka-backend>/ai-datasets /mnt/weka

# Or with explicit options:
sudo mount -t wekafs \
    -o num_cores=4,net=eth0 \
    <weka-backend>/ai-datasets \
    /mnt/weka

# Verify mount
df -h /mnt/weka
# Filesystem              Size  Used Avail Use% Mounted on
# <weka-backend>/ai-datasets  1.2P  200T  1.0P  17% /mnt/weka
```

Note: A full WEKA backend cluster requires licensed hardware (WEKA licensing is per TiB of raw NVMe capacity). The client CLI and agent can be installed and explored against a WEKA demo environment; contact WekaIO for a demo cluster or trial license.

### 18.5.4 Key CLI Commands

```bash
# Cluster health overview
weka status
# WekaIO version 4.x.y.z
# Cluster name: ai-storage-cluster
# Cluster GUID: a1b2c3d4-e5f6-...
# Status: OK (14 backend hosts, 56 drives)
# Raw capacity: 1.2 PiB
# Used: 200 TiB (17%)
# Available: 1.0 PiB

# List filesystems
weka fs
# Filesystem   Total  Used   Free   Status  Mount
# ai-datasets  500TiB 120TiB 380TiB OK      /mnt/weka
# checkpoints  100TiB 45TiB  55TiB  OK      /mnt/ckpt

# Show cluster nodes (backends)
weka cluster nodes
# ID  Hostname         Role      Status  CPU  Memory  NICs
# 0   weka-node-01     BACKEND   UP      32   256GiB  mlx5_0 (RoCEv2)
# 1   weka-node-02     BACKEND   UP      32   256GiB  mlx5_0 (RoCEv2)
# ...

# Cluster-wide I/O statistics (1-second window)
weka stats
# Time                   Read BW    Write BW   Read IOPS  Write IOPS  Latency (rd/wr)
# 2026-04-22T10:30:00Z  24.3 GiB/s  18.7 GiB/s  512K       384K        0.21ms / 0.34ms

# Detailed stats for a specific filesystem
weka stats --fs ai-datasets --interval 5 --output-format table

# Show drive status (identify failed or degraded NVMe)
weka cluster drives
# ID  Node  Device   Model          Size   Status  I/O Errors
# 0   0     /dev/nvme0n1  Samsung PM9A3  6.4TiB UP     0
# ...
```

### 18.5.5 WEKA Tiering — NVMe Hot Tier + S3 Cold Tier

WEKA's tiering model is a core feature for AI clusters where dataset storage costs would be prohibitive on all-NVMe:

```
Write path:
  Client write → WEKA backend (NVMe hot tier, DPDK, erasure coded)
                      │
                 [tiering policy: age, capacity threshold]
                      │
               S3 object store (cold tier: AWS S3, MinIO, Ceph RGW)
```

Configuration:

```bash
# Attach an S3 object store as the cold tier
weka fs tier s3 add \
    --name cold-tier \
    --obs-name s3-backend \
    --hostname minio.example.com \
    --port 9000 \
    --bucket ai-datasets-cold \
    --access-key-id admin \
    --secret-key secret

# Apply tiering policy: move data not accessed in 7 days to S3
weka fs tier policy set ai-datasets \
    --tier-name cold-tier \
    --tiering-cue 7d \
    --capacity-threshold 80   # start tiering when NVMe tier hits 80%

# Check tiering status
weka fs tier status ai-datasets
# Filesystem: ai-datasets
# Hot tier (NVMe): 380 TiB / 500 TiB used (76%)
# Cold tier (S3):  2.1 PiB
# Tiering active:  yes
# Last tiering job: 2026-04-22T06:00:00Z (moved 12 TiB)
```

When a tiered-out object is accessed, WEKA transparently prefetches it back to the NVMe tier. From the client application's perspective, the filesystem is always fully POSIX-compliant regardless of where the data physically resides.

### 18.5.6 Checkpoint Workload Comparison: Lustre vs DAOS vs WEKA

The following comparison is based on industry benchmarks for a 400 GB checkpoint write from a 1000-GPU cluster over a 400 Gbps storage fabric:

| System | Checkpoint Write (400 GB) | Notes |
|---|---|---|
| **Lustre** (OSS + HDR IB) | ~120 seconds (~3.3 GB/s) | Limited by MDS, OST count |
| **Lustre** (OSS + 400G RoCEv2, NVMe OSTs) | ~50 seconds (~8 GB/s) | Best-case Lustre on NVMe |
| **DAOS** (SCM + NVMe, HDR IB) | ~15 seconds (~26 GB/s) | SPDK + SCM metadata; requires Optane |
| **WEKA** (NVMe, RoCEv2 400G) | ~22 seconds (~18 GB/s) | No SCM required; DPDK network stack |
| **Ceph RBD** (NVMe OSDs, 400G) | ~300 seconds (~1.3 GB/s) | Not optimized for HPC checkpoint patterns |

Key insight: WEKA closes most of the performance gap between Lustre and DAOS without requiring Intel Optane or CXL SCM hardware, making it a practical choice for new AI cluster builds where Optane is unavailable or cost-prohibitive.

---

## 18.6 MinIO — S3-Compatible Object Storage

MinIO is the practical on-premises S3 replacement. It is not optimized for HPC I/O patterns but provides the interface that every ML framework already knows. `boto3` is the Amazon Web Services SDK for Python, used here as a generic S3 client that works against any S3-compatible endpoint including MinIO. `s3fs` is a Python filesystem interface built on top of boto3 that presents an S3 bucket as a file-like object, enabling libraries such as PyTorch DataLoader to stream objects directly from object storage without downloading them to disk first.

```bash
# Single-node (for dev/test)
minio server /data --console-address :9001

# Distributed mode (4 nodes, 4 drives each)
minio server \
    http://minio{1...4}.example.com/data{1...4}

# Create bucket and upload dataset
mc alias set myminio http://minio.example.com:9000 admin secret
mc mb myminio/ai-datasets
mc cp imagenet.tar myminio/ai-datasets/

# Python — PyTorch DataLoader with S3 backend (using s3fs or boto3)
import s3fs
fs = s3fs.S3FileSystem(endpoint_url="http://minio.example.com:9000",
                       key="admin", secret="secret")
with fs.open("ai-datasets/imagenet.tar", "rb") as f:
    # stream dataset directly from MinIO
```

MinIO is best for: dataset hosting (S3 API), model artifact storage (MLflow, DVC), and checkpoint archiving (long-term, cost-efficient).

---

## 18.7 Storage Network Isolation

The cardinal rule of AI cluster storage networking: **checkpoint traffic must not share fabric bandwidth with gradient traffic.**

```
GPU training fabric (RDMA / RoCEv2):
  - AllReduce gradient traffic (all-to-all, 400 Gbps per GPU)
  - Dedicated rail-optimized topology
  - DCQCN priority class 3

Storage fabric (RDMA or iSCSI):    # iSCSI (Internet Small Computer Systems Interface) encapsulates SCSI block commands over TCP/IP, providing a lower-cost alternative to RDMA for storage fabrics where latency requirements are less stringent
  - Checkpoint writes (periodic large bursts)
  - Dataset streaming (sustained, sequential)
  - Separate VLAN or separate physical switches
  - DCQCN priority class 2 (lower than training)

Management fabric (Ethernet):
  - Kubernetes control plane
  - SSH, monitoring, DNS
  - 25GbE shared
```

Failure to isolate: a 1000-GPU checkpoint event (1000 × 800 GB/1000 = 800 GB simultaneous write) saturates shared uplinks, causing AllReduce stalls precisely during the checkpoint. The training job appears to slow down at regular intervals — a classic symptom.

---

## Lab Walkthrough 18 — Single-Node Ceph (cephadm), RBD Benchmarking with fio, OSD Failure Simulation, and MinIO S3 Operations

This walkthrough uses `cephadm bootstrap --single-host-defaults` to stand up a containerized single-node Ceph cluster entirely on one Linux host (no VMs required), runs fio benchmarks on an RBD block device, simulates an OSD failure, then runs a full MinIO S3 lab and compares I/O performance across three storage tiers. `cephadm` is Ceph's official deployment and day-2 management tool; it bootstraps the entire cluster by pulling Ceph daemon container images and orchestrating them via the host's container runtime (Docker or Podman), eliminating manual daemon installation. `fio` (Flexible I/O Tester) is the standard Linux block-device benchmarking tool; it drives configurable I/O patterns (sequential, random, mixed) against any file or block device using a variety of I/O engines including the Linux asynchronous I/O engine (`libaio`).

**Prerequisites:** Docker or Podman running, at least 16 GB RAM, 100 GB free disk, and `cephadm` installed as shown in the Installation section above.

### Step 1 — Bootstrap a single-node Ceph cluster

```bash
# cephadm bootstrap brings up MON, MGR, and a minimal OSD set
# --single-host-defaults relaxes the usual multi-host requirements
sudo cephadm bootstrap \
    --mon-ip 127.0.0.1 \
    --single-host-defaults \
    --allow-overwrite \
    --skip-monitoring-stack
```

Expected output (takes 2-4 minutes while pulling container images):

```
Verifying podman|docker is present...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Cluster fsid: a1b2c3d4-e5f6-7890-abcd-ef1234567890
Verifying IP 127.0.0.1 port 3300 ...
Verifying IP 127.0.0.1 port 6789 ...
Mon is working in container "ceph-a1b2c3d4-e5f6-7890-abcd-ef1234567890-mon-hostname"
Bootstrap complete.

You can access the Ceph CLI as following in case of multi-cluster or non-default config:

    sudo /usr/sbin/cephadm shell --fsid a1b2c3d4-e5f6-7890-abcd-ef1234567890 -- ceph -s

Or, if you only have a single cluster on your machine:

    sudo ceph -s

URL: https://127.0.0.1:8443/
User: admin
Password: <generated-password>
```

### Step 2 — Wait for cluster health OK and add an OSD

```bash
# Poll until health is OK (may take 60-120 seconds)
watch sudo ceph status

# In a fresh terminal, add a loop-device-backed OSD for the lab
# (in production each OSD maps to a physical drive)
sudo fallocate -l 20G /var/lib/ceph-osd-loop0.img
LOOP_DEV=$(sudo losetup --find --show /var/lib/ceph-osd-loop0.img)
echo "Loop device: $LOOP_DEV"   # e.g. /dev/loop0

sudo cephadm shell -- ceph orch daemon add osd "$HOSTNAME:$LOOP_DEV"
```

Wait 30-60 seconds, then check:

```bash
sudo ceph status
```

Expected output:

```
  cluster:
    id:     a1b2c3d4-e5f6-7890-abcd-ef1234567890
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum hostname (age 5m)
    mgr: hostname.abcdef(active, since 4m)
    osd: 1 osds: 1 up, 1 in

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   15 MiB used, 20 GiB / 20 GiB avail
    pgs:     1 active+clean
```

### Step 3 — Create a pool and RBD image

```bash
# Create a replicated pool for the lab (size 1 since we have 1 OSD;
# in production use size 3)
sudo ceph osd pool create lab-rbd 32

# Allow size=1 replica (single-node lab only -- never in production)
sudo ceph osd pool set lab-rbd min_size 1
sudo ceph osd pool set lab-rbd size 1

# Enable the RBD application tag
sudo ceph osd pool application enable lab-rbd rbd

# Create a 5 GB RBD image named 'bench-vol'
sudo rbd create lab-rbd/bench-vol --size 5120
sudo rbd ls lab-rbd
```

Expected output:

```
bench-vol
```

Inspect the image:

```bash
sudo rbd info lab-rbd/bench-vol
```

Expected output:

```
rbd image 'bench-vol':
	size 5 GiB in 1280 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: a1b2c3d4e5f6
	block_name_prefix: rbd_data.a1b2c3d4e5f6
	format: 2
	features: layering
	op_features:
	flags:
	create_timestamp: 2026-04-22 10:05:00.000000
	access_timestamp: 2026-04-22 10:05:00.000000
	modify_timestamp: 2026-04-22 10:05:00.000000
```

### Step 4 — Map the RBD image and create a filesystem

```bash
# Load the kernel RBD module (if not already loaded)
sudo modprobe rbd

# Map the image to a block device
sudo rbd map lab-rbd/bench-vol

# Verify the device appeared
sudo rbd showmapped
```

Expected output:

```
id  pool     namespace  image     snap  device
0   lab-rbd             bench-vol -     /dev/rbd0
```

```bash
# Format and mount
sudo mkfs.ext4 /dev/rbd0
sudo mkdir -p /mnt/rbd-bench
sudo mount /dev/rbd0 /mnt/rbd-bench
df -h /mnt/rbd-bench
```

Expected output:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/rbd0       4.9G   24M  4.6G   1% /mnt/rbd-bench
```

### Step 5 — fio benchmark on the RBD block device

Create a fio job file that covers sequential write, sequential read, and random 4K I/O:

```bash
sudo tee /tmp/rbd-bench.fio > /dev/null << 'EOF'
[global]
ioengine=libaio
direct=1
time_based=1
runtime=30
group_reporting=1
filename=/dev/rbd0

[seq-write]
rw=write
bs=1M
iodepth=16
size=2G

[seq-read]
rw=read
bs=1M
iodepth=16
size=2G

[rand-4k-read]
rw=randread
bs=4K
iodepth=64
size=2G

[rand-4k-write]
rw=randwrite
bs=4K
iodepth=64
size=2G
EOF

# Run the benchmark (unmount first for raw block access)
sudo umount /mnt/rbd-bench
sudo fio /tmp/rbd-bench.fio
```

Expected output (numbers will vary with hardware; these reflect a loop-device-backed single OSD on NVMe):

```
seq-write: (g=0): rw=write, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=16
seq-read: (g=0): rw=read, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=libaio, iodepth=16
rand-4k-read: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=64
rand-4k-write: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=64
fio-3.35
Starting 4 processes

seq-write: (groupid=0, jobs=1): err= 0: pid=12345: ...
  write: IOPS=312, BW=312MiB/s (327MB/s)(9.14GiB/30001msec)
    slat (usec): min=42, max=8943, avg=215.4, stdev=312.1
    clat (msec): min=4, max=286, avg=51.1, stdev=12.4
    lat (msec): min=4, max=286, avg=51.3, stdev=12.4
    clat percentiles (msec):
     |  1.00th=[   11],  5.00th=[   25], 10.00th=[   32], 20.00th=[   40],
     | 50.00th=[   50], 90.00th=[   65], 95.00th=[   72], 99.00th=[   92]
  cpu          : usr=0.37%, sys=0.89%, ctx=9457, majf=0, minf=17
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=99.8%, 32=0.0%, >=64=0.0%

seq-read: (groupid=0, jobs=1): err= 0: pid=12346: ...
  read: IOPS=524, BW=524MiB/s (550MB/s)(15.3GiB/30001msec)

rand-4k-read: (groupid=0, jobs=1): err= 0: pid=12347: ...
  read: IOPS=8212, BW=32.1MiB/s (33.6MB/s)(962MiB/30001msec)
    lat (usec): min=312, max=45123, avg=7791.4, stdev=3214.5

rand-4k-write: (groupid=0, jobs=1): err= 0: pid=12348: ...
  write: IOPS=2341, BW=9.14MiB/s (9.59MB/s)(274MiB/30001msec)

Run status group 0 (all jobs):
   READ: bw=524MiB/s (550MB/s), 524MiB/s-524MiB/s (550MB/s-550MB/s), io=15.3GiB, run=30001-30001msec
  WRITE: bw=312MiB/s (327MB/s), 312MiB/s-312MiB/s (327MB/s-327MB/s), io=9.14GiB, run=30001-30001msec
```

Remount after benchmark:

```bash
sudo mount /dev/rbd0 /mnt/rbd-bench
```

### Step 6 — OSD failure simulation and rebalancing observation

Open a second terminal and start watching the cluster:

```bash
# Terminal 2: live cluster events
sudo ceph -w
```

In Terminal 1, mark OSD 0 as down:

```bash
sudo ceph osd down 0
```

Expected `ceph -w` output in Terminal 2:

```
2026-04-22 10:15:01.234567 mon.hostname [WRN] Health check failed: 1 osds down (OSD_DOWN)
2026-04-22 10:15:01.234890 mon.hostname [WRN] Health check failed: Degraded data redundancy: 12/36 objects degraded (33.333%), 1 pg degraded (PG_DEGRADED)
2026-04-22 10:15:01.235001 osd.0 [INF] osd.0 reported failed by osd.0
2026-04-22 10:15:06.111222 osd.0 [INF] osd.0 is down in the osd map
2026-04-22 10:15:06.111333 mon.hostname [WRN] 1 daemons have recently crashed
```

Check cluster status:

```bash
sudo ceph status
```

Expected output:

```
  cluster:
    id:     a1b2c3d4-e5f6-7890-abcd-ef1234567890
    health: HEALTH_WARN
            1 osds down
            Degraded data redundancy: 12/36 objects degraded (33.333%), 1 pg degraded

  services:
    mon: 1 daemons, quorum hostname (age 15m)
    mgr: hostname.abcdef(active, since 14m)
    osd: 1 osds: 0 up, 1 in

  data:
    pools:   1 pools, 32 pgs
    objects: 36 objects, 48 MiB
    usage:   156 MiB used, 20 GiB / 20 GiB avail
    pgs:     12/36 objects degraded (33.333%)
             1 active+degraded
             31 active+clean
```

Mark the OSD back up (restore it):

```bash
sudo ceph osd up 0
sudo ceph osd in 0
```

Expected `ceph -w` output showing recovery:

```
2026-04-22 10:15:45.001234 mon.hostname [INF] Health check cleared: OSD_DOWN (was: 1 osds down)
2026-04-22 10:15:45.001567 mon.hostname [INF] Health check cleared: PG_DEGRADED (was: Degraded data redundancy...)
2026-04-22 10:15:45.002001 mon.hostname [INF] overall HEALTH_OK
```

Final status:

```bash
sudo ceph status
```

Expected output:

```
  cluster:
    id:     a1b2c3d4-e5f6-7890-abcd-ef1234567890
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum hostname (age 16m)
    mgr: hostname.abcdef(active, since 15m)
    osd: 1 osds: 1 up, 1 in

  data:
    pools:   1 pools, 32 pgs
    objects: 36 objects, 48 MiB
    usage:   156 MiB used, 20 GiB / 20 GiB avail
    pgs:     32 active+clean
```

### Step 7 — MinIO lab: server, bucket operations, file upload/download

Start MinIO as a background single-node server:

```bash
# Create data directory
sudo mkdir -p /var/lib/minio-data
sudo chown $USER /var/lib/minio-data

# Start MinIO in the background
export MINIO_ROOT_USER=labadmin
export MINIO_ROOT_PASSWORD=labsecret123

minio server /var/lib/minio-data \
    --address :9000 \
    --console-address :9001 \
    > /tmp/minio.log 2>&1 &

MINIO_PID=$!
echo "MinIO PID: $MINIO_PID"
sleep 3

# Verify MinIO is up
curl -s http://127.0.0.1:9000/minio/health/live && echo "MinIO: OK"
```

Expected output:

```
MinIO: OK
```

Configure the MinIO client alias:

```bash
mc alias set labminio http://127.0.0.1:9000 labadmin labsecret123
mc alias list
```

Expected output:

```
labminio
  URL       : http://127.0.0.1:9000
  AccessKey : labadmin
  SecretKey : labsecret123
  API       : S3v4
  Path      : auto
```

Create a bucket and upload a test file:

```bash
# Create the bucket
mc mb labminio/ai-datasets
```

Expected output:

```
Bucket created successfully `labminio/ai-datasets`.
```

```bash
# Create a 100 MB test file to simulate a dataset shard
dd if=/dev/urandom of=/tmp/dataset-shard-001.bin bs=1M count=100 status=progress

# Upload it
mc cp /tmp/dataset-shard-001.bin labminio/ai-datasets/
```

Expected output:

```
/tmp/dataset-shard-001.bin: 100 MiB / 100 MiB ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 1.23 GiB/s
```

```bash
# List bucket contents
mc ls labminio/ai-datasets
```

Expected output:

```
[2026-04-22 10:20:01 UTC]  100MiB STANDARD dataset-shard-001.bin
```

```bash
# Download and verify checksum
mc cp labminio/ai-datasets/dataset-shard-001.bin /tmp/dataset-shard-001-downloaded.bin
md5sum /tmp/dataset-shard-001.bin /tmp/dataset-shard-001-downloaded.bin
```

Expected output (both hashes must match):

```
a3f7c1e2d8b4f091a3f7c1e2d8b4f091  /tmp/dataset-shard-001.bin
a3f7c1e2d8b4f091a3f7c1e2d8b4f091  /tmp/dataset-shard-001-downloaded.bin
```

### Step 8 — Python boto3 upload/download script (complete, runnable)

Save and run this script in the activated Python venv:

```bash
# Ensure venv is active
source .venv/bin/activate

cat > /tmp/minio_boto3_lab.py << 'PYEOF'
#!/usr/bin/env python3
"""
Complete boto3 lab script: upload, list, download, verify, and delete.
Targets a local MinIO server.
"""

import boto3
import hashlib
import os
import sys
from botocore.client import Config

ENDPOINT   = "http://127.0.0.1:9000"
ACCESS_KEY = "labadmin"
SECRET_KEY = "labsecret123"
BUCKET     = "ai-checkpoints-boto3"
OBJECT_KEY = "checkpoints/model_step_5000.pt"
LOCAL_FILE = "/tmp/model_step_5000.pt"
DOWNLOAD_FILE = "/tmp/model_step_5000_downloaded.pt"


def md5(filepath: str) -> str:
    h = hashlib.md5()
    with open(filepath, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()


def main():
    # Create S3 client pointing at MinIO
    s3 = boto3.client(
        "s3",
        endpoint_url=ENDPOINT,
        aws_access_key_id=ACCESS_KEY,
        aws_secret_access_key=SECRET_KEY,
        config=Config(signature_version="s3v4"),
        region_name="us-east-1",
    )

    # 1. Create bucket
    print(f"[1] Creating bucket: {BUCKET}")
    try:
        s3.create_bucket(Bucket=BUCKET)
        print(f"    Bucket '{BUCKET}' created.")
    except s3.exceptions.BucketAlreadyOwnedByYou:
        print(f"    Bucket '{BUCKET}' already exists.")

    # 2. Create a fake checkpoint file (50 MB of random bytes)
    print(f"[2] Generating test file: {LOCAL_FILE} (50 MB)")
    with open(LOCAL_FILE, "wb") as f:
        f.write(os.urandom(50 * 1024 * 1024))
    original_md5 = md5(LOCAL_FILE)
    print(f"    MD5 (original):  {original_md5}")

    # 3. Upload
    print(f"[3] Uploading {LOCAL_FILE} -> s3://{BUCKET}/{OBJECT_KEY}")
    s3.upload_file(LOCAL_FILE, BUCKET, OBJECT_KEY)
    print("    Upload complete.")

    # 4. List objects in bucket
    print(f"[4] Listing objects in bucket '{BUCKET}':")
    response = s3.list_objects_v2(Bucket=BUCKET)
    for obj in response.get("Contents", []):
        size_mb = obj["Size"] / (1024 * 1024)
        print(f"    {obj['Key']}  ({size_mb:.1f} MiB)  LastModified: {obj['LastModified']}")

    # 5. Download
    print(f"[5] Downloading s3://{BUCKET}/{OBJECT_KEY} -> {DOWNLOAD_FILE}")
    s3.download_file(BUCKET, OBJECT_KEY, DOWNLOAD_FILE)
    downloaded_md5 = md5(DOWNLOAD_FILE)
    print(f"    MD5 (downloaded): {downloaded_md5}")

    # 6. Verify integrity
    if original_md5 == downloaded_md5:
        print("[6] Integrity check PASSED: MD5 hashes match.")
    else:
        print("[6] Integrity check FAILED: MD5 mismatch!", file=sys.stderr)
        sys.exit(1)

    # 7. Delete object and bucket
    print(f"[7] Deleting object and bucket.")
    s3.delete_object(Bucket=BUCKET, Key=OBJECT_KEY)
    s3.delete_bucket(Bucket=BUCKET)
    print("    Cleanup complete.")

    # 8. Cleanup local files
    os.remove(LOCAL_FILE)
    os.remove(DOWNLOAD_FILE)
    print("[8] Local temp files removed. Done.")


if __name__ == "__main__":
    main()
PYEOF

python3 /tmp/minio_boto3_lab.py
```

Expected output:

```
[1] Creating bucket: ai-checkpoints-boto3
    Bucket 'ai-checkpoints-boto3' created.
[2] Generating test file: /tmp/model_step_5000.pt (50 MB)
    MD5 (original):  d41d8cd98f00b204e9800998ecf8427e
[3] Uploading /tmp/model_step_5000.pt -> s3://ai-checkpoints-boto3/checkpoints/model_step_5000.pt
    Upload complete.
[4] Listing objects in bucket 'ai-checkpoints-boto3':
    checkpoints/model_step_5000.pt  (50.0 MiB)  LastModified: 2026-04-22 10:25:01+00:00
[5] Downloading s3://ai-checkpoints-boto3/checkpoints/model_step_5000.pt -> /tmp/model_step_5000_downloaded.pt
    MD5 (downloaded): d41d8cd98f00b204e9800998ecf8427e
[6] Integrity check PASSED: MD5 hashes match.
[7] Deleting object and bucket.
    Cleanup complete.
[8] Local temp files removed. Done.
```

### Step 9 — fio comparison: local NVMe vs RBD vs MinIO (NFS-backed)

This step runs the same sequential write workload against three targets and records the results for comparison.

**9a. Local NVMe (or loop device as baseline):**

```bash
sudo fio \
    --name=local-nvme-seq-write \
    --ioengine=libaio \
    --rw=write \
    --bs=1M \
    --iodepth=16 \
    --size=2G \
    --direct=1 \
    --runtime=30 \
    --time_based=1 \
    --filename=/tmp/fio-local.dat \
    --output-format=normal
```

Expected result (SSD):

```
local-nvme-seq-write: (g=0): rw=write, bs=(R) 1024KiB-1024KiB, ...
  write: IOPS=2048, BW=2048MiB/s (2147MB/s)(59.5GiB/30001msec)
    lat (usec): min=312, max=4123, avg=781.2, stdev=142.3
```

**9b. Ceph RBD (via kernel rbd driver):**

```bash
sudo fio \
    --name=rbd-seq-write \
    --ioengine=libaio \
    --rw=write \
    --bs=1M \
    --iodepth=16 \
    --size=2G \
    --direct=1 \
    --runtime=30 \
    --time_based=1 \
    --filename=/dev/rbd0 \
    --output-format=normal
```

Expected result (single-node loop-backed Ceph):

```
rbd-seq-write: (g=0): rw=write, bs=(R) 1024KiB-1024KiB, ...
  write: IOPS=312, BW=312MiB/s (327MB/s)(9.1GiB/30001msec)
    lat (msec): min=4, max=286, avg=51.1, stdev=12.4
```

**9c. MinIO via NFS-mounted FUSE (s3fs) — latency discussion:**

MinIO does not expose a block device; the closest comparison is using `s3fs-fuse` to mount a bucket as a POSIX filesystem and running fio against it. However, fio over a FUSE S3 mount is not a fair block-level comparison:

```bash
# Install s3fs
sudo apt install -y s3fs

echo "labadmin:labsecret123" > /tmp/.minio-creds
chmod 600 /tmp/.minio-creds

mc mb labminio/fio-test 2>/dev/null || true
sudo mkdir -p /mnt/minio-fuse

sudo s3fs fio-test /mnt/minio-fuse \
    -o passwd_file=/tmp/.minio-creds \
    -o url=http://127.0.0.1:9000 \
    -o use_path_request_style \
    -o allow_other

# Run a modest fio test (MinIO/S3 is not designed for fio-style I/O)
fio \
    --name=minio-fuse-seq-write \
    --ioengine=sync \
    --rw=write \
    --bs=1M \
    --iodepth=1 \
    --size=512M \
    --filename=/mnt/minio-fuse/fio-test-file.dat \
    --output-format=normal

sudo umount /mnt/minio-fuse
```

Expected result (object store via FUSE — significantly higher latency):

```
minio-fuse-seq-write: (g=0): rw=write, bs=(R) 1024KiB-1024KiB, ...
  write: IOPS=28, BW=28.4MiB/s (29.8MB/s)(512MiB/18014msec)
    lat (msec): min=12, max=412, avg=35.4, stdev=28.1
```

**Summary table — latency and throughput comparison:**

| Tier | Sequential Write BW | 4K Random IOPS | Write Latency (avg) | Best Use Case |
|---|---|---|---|---|
| **Local NVMe** | ~2000 MB/s | ~200K | ~0.8 ms | Temporary scratch, DALI cache |
| **Ceph RBD** (3-replica, NVMe OSD) | ~800 MB/s | ~20K | ~5 ms | Persistent volumes, checkpoint staging |
| **Ceph RBD** (lab, loop-backed) | ~312 MB/s | ~2K | ~51 ms | Lab reference only |
| **MinIO via S3 API** | ~800 MB/s (multipart) | N/A (object store) | ~35 ms (FUSE) | Dataset hosting, artifact archiving |
| **MinIO via FUSE** | ~28 MB/s | ~100 | ~35 ms | Not recommended for I/O-intensive use |

Key takeaways:
- Local NVMe is 5-10× faster than Ceph RBD for random I/O — use it for ephemeral scratch and DALI cache.
- Ceph RBD with NVMe OSDs reaches 800 MB/s sequential on a production 3-replica cluster; the lab loop-device numbers are artificially low.
- MinIO's strength is the S3 API and horizontal scalability, not block I/O performance. Direct S3 multipart uploads from PyTorch reach 800 MB/s on a 4-node MinIO cluster; FUSE should never be used for training I/O.
- Checkpoint traffic must travel on a dedicated storage VLAN to avoid starving AllReduce gradient traffic — a 1000-GPU simultaneous checkpoint is an 800 GB write burst.

### Step 10 — Cleanup

```bash
# Stop MinIO
kill $MINIO_PID 2>/dev/null

# Unmap RBD
sudo umount /mnt/rbd-bench 2>/dev/null || true
sudo rbd unmap /dev/rbd0 2>/dev/null || true

# Tear down Ceph (removes all containers and data)
sudo cephadm rm-cluster \
    --fsid $(sudo ceph fsid) \
    --force

# Remove loop device
sudo losetup -d $LOOP_DEV 2>/dev/null || true
sudo rm -f /var/lib/ceph-osd-loop0.img

# Remove MinIO data
sudo rm -rf /var/lib/minio-data

echo "Cleanup complete."
```

---

## Summary

- Ceph provides a flexible unified storage platform (block/file/object) suited for checkpointing (RBD), dataset access (CephFS), and artifact storage (RGW/S3).
- Lustre offers superior parallel throughput for large sequential files across thousands of simultaneous clients — the classic HPC filesystem, now deployed in AI clusters.
- DAOS delivers order-of-magnitude better checkpoint latency by using SCM and SPDK to eliminate OS I/O overhead, at the cost of specialized hardware.
- WEKA (WekaFS) fills the gap between Lustre and DAOS: DPDK-based NVMe-native storage with sub-millisecond latency, GPUDirect Storage support, and S3-compatible tiering — without requiring Optane or CXL SCM.
- MinIO provides the S3 API on-premises — the universal interface for dataset and artifact storage in ML workflows.
- Storage network isolation from the training RDMA fabric is non-negotiable; shared bandwidth causes periodic AllReduce stalls correlated with checkpoint intervals.

---

## References

- Ceph documentation: docs.ceph.com
- Lustre documentation: doc.lustre.org
- DAOS documentation: docs.daos.io
- MinIO documentation: min.io/docs
- WEKA documentation: docs.weka.io
- WekaIO GPUDirect Storage integration guide: docs.weka.io/usage/gpudirect-storage
- Weil, S.A. et al., *Ceph: A Scalable, High-Performance Distributed File System*, OSDI 2006


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).