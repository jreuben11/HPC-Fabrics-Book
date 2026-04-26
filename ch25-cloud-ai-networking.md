# Chapter 25 — Cloud AI Networking

**Part I: Foundations** | ~20 pages

---

## Introduction

Chapter 25 examines what changes when an AI training cluster moves from on-premises hardware to cloud infrastructure — specifically what each of the three major cloud providers (**AWS**, **Azure**, **GCP**) exposes as their **RDMA** or near-**RDMA** networking layer, how those layers map to the verbs/**libfabric**/**NCCL** software stacks covered in earlier chapters, and how to configure and tune them for maximum **AllReduce** throughput. The Lab Walkthrough runs a **libfabric** provider smoke test, captures **fabtests** bandwidth numbers, and walks through the **NCCL** topology file workflow on a non-GPU lab host.

The practical context is hard to ignore: as of 2025, a majority of large AI training runs happen in cloud rather than on dedicated on-premises clusters. The three providers take fundamentally different approaches. **AWS EFA** uses a custom **RDMA**-capable NIC with a **libfabric** abstraction that sidesteps standard IB verbs entirely. Azure NDv5 and HBv3 expose real **Mellanox** **InfiniBand** HCAs through **SR-IOV**, making the software stack nearly identical to on-premises IB — with the important caveat that the Subnet Manager and fabric routing are managed by Microsoft and invisible to tenants. GCP A3 uses **GPUDirect-TCPX**, a hybrid approach that provides NIC hardware DMA assist over standard TCP, trading some latency for the ability to span availability zones.

Chapter 24 established the full on-premises IB management picture: **OpenSM**, LFTs, diagnostic tools, and **PFC** configuration. Chapter 25 builds on that foundation by showing which parts of the IB management stack disappear in cloud (SM, MAD access, firmware control) and which parts remain the operator's responsibility (**NCCL** environment variables, topology files, placement group selection, host-level **RDMA** driver tuning). Chapter 26 covers security for the same cloud fabrics.

A key theme is that cloud **RDMA** abstraction layers are deep and provider-specific, but they converge at the **NCCL** interface: all three providers ultimately provide a `NCCL_NET` plugin that plugs into **NCCL**'s network transport interface. Understanding that convergence point — the `ncclNet_v8_t` plugin ABI — makes it possible to reason about performance and portability across providers without understanding every layer of each provider's proprietary stack.

Section 25.1 compares the on-prem and cloud networking models. Sections 25.2–25.4 cover **EFA**, Azure **RDMA**, and **GPUDirect-TCPX** in depth. Section 25.5 addresses cloud-specific **NCCL** tuning. Section 25.6 covers VPC design for multi-node AI clusters. Section 25.7 provides a cloud-versus-on-prem decision matrix.

---

## Installation

The `libfabric` and `fabtests` packages provide the core **OFI** communication library and its benchmark suite (`fi_pingpong`, `fi_msg_bw`) used to compare provider latency and bandwidth across `shm`, `tcp`, and hardware-backed providers; they are the common substrate that all three cloud **RDMA** transports (**EFA**, Azure verbs, **TCPX**) plug into. On **AWS EFA**-capable instances, `efa-utils` installs the **EFA** userspace driver and device enumeration tools needed to activate the `efa` **libfabric** provider. The **aws-ofi-nccl** plugin is built from source and acts as the bridge between **NCCL**'s `ncclNet_v8_t` transport interface and the **libfabric** `efa` provider, enabling GPU collective operations to use **EFA**'s SRD transport rather than standard TCP sockets. **MPI** (provided by AWS **HPC-X** or **OpenMPI**) is required for multi-node collective performance benchmarks and for launching distributed **NCCL** test programs across cluster nodes.

### System Packages (Ubuntu 24.04)

```bash
sudo apt update
sudo apt install -y \
    libfabric1 \
    libfabric-dev \
    fabtests \
    rdma-core \
    ibverbs-utils \
    numactl \
    hwloc \
    pciutils \
    build-essential \
    cmake \
    pkg-config
```

On AWS EC2 instances with EFA NICs, install the EFA userspace utilities:

```bash
# EFA-specific package (AWS-managed repository; run on EFA-capable instances only)
# On non-EFA lab hosts the efa-utils package is unavailable — use shm/tcp providers instead
sudo apt install -y efa-utils 2>/dev/null || echo "efa-utils not available on this host (expected outside AWS)"

# Verify libfabric installation
dpkg -l | grep -E "libfabric|fabtests"
# Expected:
# ii  fabtests     ...  Test suite for libfabric
# ii  libfabric-dev  ...  Development files for libfabric
# ii  libfabric1   ...  libfabric communication library

# Check available providers
fi_info --list
# Expected (on a standard Linux host):
# tcp
# udp
# shm
# rxm
# verbs   (if rdma_rxe or real HCA is present)
```

### Python uv environment

```bash
# Install uv if not already present
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# Create project environment
mkdir -p ~/cloud-ai-net && cd ~/cloud-ai-net
uv venv .venv && source .venv/bin/activate
uv pip install boto3 fabric paramiko

# Verify
python -c "
import boto3, fabric, paramiko
print('boto3:', boto3.__version__)
print('fabric:', fabric.__version__)
print('paramiko:', paramiko.__version__)
"
# Expected:
# boto3: 1.34.x
# fabric: 3.x.x
# paramiko: 3.x.x
```

### aws-ofi-nccl build from source

`aws-ofi-nccl` is the libfabric transport plugin that NCCL loads to use EFA (or any libfabric provider) instead of its built-in TCP/IB transports. Build steps (requires NCCL and CUDA headers; shown for documentation purposes):

```bash
# Clone the repository
git clone https://github.com/aws/aws-ofi-nccl.git
cd aws-ofi-nccl

# Configure with EFA provider and CUDA paths
# On a real p4d/p5 instance with NCCL and CUDA installed:
./autogen.sh
./configure \
    --with-mpi=/opt/amazon/openmpi \
    --with-libfabric=/opt/amazon/efa \
    --with-cuda=/usr/local/cuda \
    --with-nccl=/usr/local/nccl \
    --prefix=/usr/local/aws-ofi-nccl \
    --enable-platform-aws

make -j$(nproc)
sudo make install

# The plugin is installed as a shared library:
ls /usr/local/aws-ofi-nccl/lib/
# Expected:
# libnccl-net.so   libnccl-net.so.0   libnccl-net.so.0.0.0

# To activate, add the plugin directory to LD_LIBRARY_PATH and set:
# export NCCL_NET=AWS-OFI-NCCL
```

---

## 25.1 On-Prem vs Cloud AI Networking: What Changes

Moving an AI training cluster from on-premises to cloud fundamentally changes the networking stack in three dimensions: hardware access, fabric management, and software configuration.

### 25.1.1 Physical Access and Fabric Control

On-premises, you own the switches, cables, and NICs. You configure OpenSM, tune PFC thresholds, update switch firmware, and respond to link flap alerts at 3 AM. The entire control plane is yours.

In cloud, the fabric is **managed**. AWS, Azure, and GCP operate the physical network and provide a bounded set of abstractions:

| Layer | On-Prem | Cloud |
|---|---|---|
| NIC driver | mlx5_core, rxe, etc. | Cloud-specific (EFA driver, Azure RDMA driver) |
| Fabric topology | You design and cable it | Hidden; controlled by placement groups |
| Congestion control | You tune DCQCN/PFC | Managed by cloud provider |
| SM / routing | OpenSM or UFM | Managed (invisible to tenant) |
| Firmware updates | You schedule downtime | Cloud provider, no notice required |
| Observability | Full MAD access, switch CLI | API-mediated counters only |

### 25.1.2 Software Stack Differences

Each cloud provider ships a customized software stack that replaces or extends the standard RDMA userspace. `libfabric` (also called OFI, OpenFabrics Interfaces) is a vendor-neutral network communication library that abstracts different RDMA and high-speed network technologies behind a common API through pluggable "providers" (such as `efa` for AWS EFA, `verbs` for IB/RoCE, `tcp` for standard TCP); it was designed to avoid the IB-centrism of the original `libibverbs` API. `NCCL` (NVIDIA Collective Communications Library) is NVIDIA's GPU-optimized library for collective operations — AllReduce, AllGather, ReduceScatter, Broadcast — across GPUs; it detects available network transports via a pluggable `NCCL_NET` interface and handles topology-aware algorithm selection automatically.

- **AWS:** `libfabric` with the `efa` provider replacing `libibverbs` for inter-instance RDMA; `aws-ofi-nccl` bridges NCCL to libfabric.
- **Azure:** Standard `libibverbs` + Mellanox OFED (HBv3/NDv5 expose real IB HCAs); `hpc-x` is a pre-built MPI/NCCL stack optimized for Azure's IB fabric.
- **GCP:** `GPUDirect-TCPX` uses a custom NIC hardware-assist mechanism over TCP rather than a full RDMA protocol; the `tcpxo` plugin provides NCCL integration.

### 25.1.3 Instance Placement and Proximity

All three cloud providers use **placement groups** (AWS), **proximity placement groups** (Azure), or **compact placement** (GCP) to co-locate instances within a single rack or physical cluster boundary. Without explicit placement configuration, VM instances may land in different racks or AZs, yielding 10-100x higher latency for collective operations.

---

## 25.2 AWS EFA: Elastic Fabric Adapter

EFA is AWS's custom RDMA-capable NIC for HPC/AI workloads. It is exposed via the `libfabric` API using the `efa` provider, intentionally bypassing the standard `libibverbs` path used by InfiniBand/RoCE NICs.

### 25.2.1 Architecture

```
Application / NCCL
       │
  aws-ofi-nccl plugin    (NCCL_NET=AWS-OFI-NCCL)
       │
  libfabric (efa provider)
       │
  EFA kernel driver (efa.ko)
       │
  EFA NIC hardware (SRD — Scalable Reliable Datagram)
       │
  AWS internal fabric (not exposed as Ethernet or IB)
```

EFA's transport is **SRD (Scalable Reliable Datagram)** — a transport developed by AWS that provides reliable delivery without requiring per-connection state (unlike RC in IB). This allows a single EFA port to efficiently communicate with all peers in a large cluster without the N² QP memory problem.

### 25.2.2 EFA-Capable Instance Types

| Instance | GPU | EFA bandwidth | Notes |
|---|---|---|---|
| p4d.24xlarge | 8× A100 (SXM4) | 4× 100 Gbps EFA | Prior-gen AI flagship |
| p5.48xlarge | 8× H100 (SXM5) | 3200 Gbps EFA (32× 100G) | Current AI flagship |
| trn1.32xlarge | 16× Trainium | 2× 100 Gbps EFA | AWS custom AI silicon |
| trn2.48xlarge | 16× Trainium2 | 1600 Gbps EFA (16× 100G) | Trainium2 flagship |
| hpc7g.16xlarge | (no GPU) | 200 Gbps EFA | HPC/MPI workloads |

### 25.2.3 Cluster Placement Group

A **cluster placement group** places all instances within a single Availability Zone, on the same high-bisection-bandwidth segment of the AWS network. This is mandatory for EFA-accelerated collective operations:

```bash
# Create a cluster placement group (AWS CLI)
aws ec2 create-placement-group \
    --group-name ai-training-cluster \
    --strategy cluster \
    --query 'PlacementGroup.GroupId'
# Expected:
# "pg-0a1b2c3d4e5f6a7b8"

# Launch instances into the group
aws ec2 run-instances \
    --instance-type p5.48xlarge \
    --placement '{"GroupName":"ai-training-cluster"}' \
    --network-interfaces '[{"DeviceIndex":0,"InterfaceType":"efa","DeleteOnTermination":true}]' \
    --count 8 \
    --image-id ami-0abcdef1234567890
```

### 25.2.4 EFA Provider Query

```bash
# On an EFA-capable instance:
fi_info -p efa
# Expected:
# provider: efa
#     fabric: EFA-fe80::...
#     domain: rdmap0s31-rdm
#     version: 116.0
#     type: FI_EP_RDM
#     protocol: FI_PROTO_EFA
#     tx_attr:
#         caps: FI_MSG|FI_TAGGED|FI_RMA|FI_ATOMIC|FI_READ|FI_WRITE|FI_SEND
#         inject_size: 32
#         tclass: FI_TC_UNSPEC
#     rx_attr:
#         caps: FI_MSG|FI_TAGGED|FI_RMA|FI_ATOMIC|FI_REMOTE_READ|FI_REMOTE_WRITE|FI_RECV
#     domain_attr:
#         name: rdmap0s31-rdm
#         threading: FI_THREAD_DOMAIN
#         data_progress: FI_PROGRESS_AUTO
#         resource_mgmt: FI_RM_ENABLED
#         mr_mode: [ FI_MR_LOCAL FI_MR_VIRT_ADDR FI_MR_ALLOCATED FI_MR_PROV_KEY ]
#         max_ep_tx_ctx: 1
#         max_ep_rx_ctx: 1
```

### 25.2.5 NCCL + EFA Integration

```bash
# Build aws-ofi-nccl (see Installation section above)
# Then set environment variables:

export NCCL_NET=AWS-OFI-NCCL
export LD_LIBRARY_PATH=/usr/local/aws-ofi-nccl/lib:$LD_LIBRARY_PATH
export NCCL_DEBUG=INFO
export FI_EFA_USE_DEVICE_RDMA=1      # enable device RDMA (GPUDirect)
export FI_EFA_FORK_SAFE=1            # required for multi-process safety

# Launch multi-node NCCL AllReduce test:
mpirun -np 16 --hostfile hostfile \
    -x NCCL_NET -x LD_LIBRARY_PATH -x FI_EFA_USE_DEVICE_RDMA \
    python -c "
import torch
import torch.distributed as dist
dist.init_process_group('nccl')
t = torch.ones(1024*1024, device='cuda')
dist.all_reduce(t)
print(f'rank {dist.get_rank()}: AllReduce OK, result sum = {t.sum().item():.0f}')
"
```

---

## 25.3 Azure RDMA: HBv3 and NDv5

Azure exposes real InfiniBand NICs to VMs via SR-IOV passthrough. HBv3 (CPU-only HPC) uses HDR 200 Gbps IB; NDv5 (A100 GPU) and the newer NCv5 (H100) use NDR 400 Gbps IB.

### 25.3.1 Azure RDMA Driver Stack

```
Application / MPI / NCCL
       │
  hpc-x (MPI: OpenMPI or MVAPICH2; NCCL: hpcx-nccl)
       │
  libibverbs + mlx5 provider
       │
  mlx5_core kernel driver (SR-IOV VF)
       │
  Azure fabric (InfiniBand HDR/NDR, managed SM)
```

The Azure SM is operated by Microsoft and is not accessible to tenants. LID assignment and fabric routing happen transparently. Tenants see a standard IB port via `ibstat`:

```bash
# On an HBv3 or NDv5 instance:
ibstat mlx5_0
# Expected (NDv5, NDR 400G):
# CA 'mlx5_0'
#     CA type: MT4129
#     Number of ports: 1
#     Firmware version: 28.36.1010
#     Port 1:
#         State: Active
#         Physical state: LinkUp
#         Rate: 400                # Gbps, NDR
#         Base lid: 2847
#         LMC: 0
#         SM lid: 1
#         Port GUID: 0x000d3a504c3f0001
#         Link layer: InfiniBand
```

### 25.3.2 hpc-x MPI Stack

Azure provides `hpc-x` as a pre-built tarball containing OpenMPI + MVAPICH2 + UCX + NCCL, tuned for their IB fabric:

```bash
# Download hpc-x (Azure HPC images pre-install this)
wget https://azhpcstor.blob.core.windows.net/azhpc-images-store/hpcx-v2.16-gcc-mlnx_ofed-ubuntu22.04-cuda12-x86_64.tbz
tar xf hpcx-v2.16-*.tbz -C /opt/
source /opt/hpcx-v2.16-*/hpcx-init.sh
hpcx_load

# Run bandwidth test across two NDv5 instances
mpirun -np 2 --map-by node -x UCX_TLS=rc \
    ib_write_bw -d mlx5_0 --run_infinitely
```

### 25.3.3 Proximity Placement Groups

```bash
# Create a proximity placement group (Azure CLI)
az ppg create \
    --resource-group ai-rg \
    --name ai-ppg \
    --location eastus \
    --type Standard

# Place a VM Scale Set into the PPG
az vmss create \
    --resource-group ai-rg \
    --name ai-vmss \
    --vm-sku Standard_ND96asr_v4 \
    --proximity-placement-group ai-ppg \
    --instance-count 8
```

---

## 25.4 GCP GPUDirect-TCPX

GCP's A3 instances (H100 GPU) use a fundamentally different approach from IB or EFA. Rather than an RDMA-over-dedicated-fabric, GPUDirect-TCPX provides GPU-to-GPU data transfer over standard TCP with NIC hardware assist that enables GPU memory DMA directly to/from the NIC without CPU staging.

### 25.4.1 Architecture

```
GPU HBM  ←→  NIC (hardware DMA via GPUDirect-TCPX)  ←→  TCP over 200 Gbps NIC
```

Unlike GPUDirect RDMA (which bypasses the OS transport), TCPX uses a modified kernel TCP stack with:
- A custom kernel module (`tcpxo.ko`) that registers GPU memory regions with the NIC's DMA engine.
- A libfabric provider (`tcpxo`) that NCCL loads via `NCCL_NET=tcpxo`.
- Standard TCP framing (no custom transport protocol), allowing traffic to cross AZ boundaries.

### 25.4.2 A3 Instance and tcpxo Configuration

```bash
# On an A3 instance (GCP):
# Verify GPUDirect-TCPX kernel module
lsmod | grep tcpxo
# Expected:
# tcpxo                 196608  0

# Check available libfabric providers including tcpxo
fi_info --list
# Expected (A3 instance):
# ...
# tcpxo
# tcp
# shm

# Configure NCCL to use TCPX
export NCCL_NET=tcpxo
export NCCL_SOCKET_IFNAME=^lo,docker    # exclude loopback and docker interfaces
export NCCL_DEBUG=WARN

# Baseline NCCL bandwidth on A3 (8× H100):
# Single-node NVLink AllReduce: ~3.2 TB/s (NVSwitch-limited)
# Inter-node TCPX AllReduce (2 nodes): ~150-180 GB/s effective
# (vs ~100 GB/s with plain TCP)
```

### 25.4.3 TCPX vs Standard TCP NCCL

| Metric | Standard TCP (NCCL_NET=Socket) | GPUDirect-TCPX |
|---|---|---|
| CPU involvement in data path | Full (GPU→CPU copy + send) | NIC DMA direct from GPU HBM |
| Effective AllReduce bandwidth (2×A3) | ~80 GB/s | ~160 GB/s |
| Latency per message | 50-200 µs | 30-80 µs |
| Cross-AZ support | Yes | Yes (uses standard TCP) |
| Setup complexity | None | tcpxo kernel module + libfabric provider |

---

## 25.5 Cloud-Specific NCCL Tuning

### 25.5.1 Critical Environment Variables

```bash
# Select the network transport backend
export NCCL_NET=AWS-OFI-NCCL     # EFA on AWS
# export NCCL_NET=tcpxo           # TCPX on GCP A3
# export NCCL_NET=IB              # InfiniBand (Azure NDv5, on-prem)

# Select the IB/RDMA device to use (mlx5_0, mlx5_1, etc.)
export NCCL_IB_HCA=mlx5_0,mlx5_1,mlx5_2,mlx5_3    # Azure NDv5 (4 IB ports)

# Select network interface for NCCL OOB (out-of-band) communication
export NCCL_SOCKET_IFNAME=eth0    # AWS: usually eth0 or ens5
# On Azure: use the RDMA interface name, e.g., ib0

# Disable cross-NIC bonding (forces single NIC per rank)
export NCCL_IB_GID_INDEX=3       # GID index for RoCEv2 (use 0 for IB)

# NCCL tree AllReduce threshold (bytes; use ring above this size)
export NCCL_TREE_THRESHOLD=0      # disable tree, use ring always (better for IB)
# export NCCL_TREE_THRESHOLD=4294967296  # AWS p5: enable tree for large messages

# Logging
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=NET,GRAPH,TUNING
```

### 25.5.2 Topology File Generation

NCCL's automatic topology detection works well on-prem but can misidentify cloud VM topology (virtual NUMA, hidden PCIe hierarchy). A manually-generated topology XML improves collective algorithm selection:

```bash
# Generate topology with hwloc (hwloc is the Hardware Locality library; it
# enumerates the physical topology of the machine — NUMA nodes, CPU cores,
# PCIe buses, and attached devices — and exposes it as an XML file that NCCL
# uses to determine which GPUs share a PCIe switch or NVLink domain)
lstopo --output-format xml --whole-system > /tmp/nccl_topo.xml

# Tell NCCL to use this file
export NCCL_TOPO_FILE=/tmp/nccl_topo.xml
```

### 25.5.3 Bandwidth Baselines by Instance Type

Use these baselines for commissioning and regression testing:

| Instance | Collective | Expected BW | Notes |
|---|---|---|---|
| p4d.24xlarge (A100) | AllReduce 1GB ring | ~80 Gbps | 4× 100G EFA |
| p5.48xlarge (H100) | AllReduce 1GB ring | ~350 Gbps | 32× 100G EFA |
| NDv5 (A100, 8-node) | AllReduce 1GB ring | ~150 Gbps | 8× NDR IB per node |
| A3 (H100, 2-node) | AllReduce 1GB ring | ~160 Gbps | GPUDirect-TCPX |
| trn1.32xlarge | AllReduce 1GB EFA | ~50 Gbps | Trainium NeuronCore fabric |

---

## 25.6 Multi-Node Cluster Networking in Cloud

### 25.6.1 VPC Design for AI Clusters

AI training clusters require careful VPC design to avoid network bottlenecks:

```
┌─────────────────────────────────────────────────────────┐
│  Availability Zone (single AZ — mandatory for EFA/IB)   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Cluster Placement Group                        │    │
│  │                                                 │    │
│  │  [p5 node 0] ──EFA── [p5 node 1] ──EFA── ...   │    │
│  │      │                    │                     │    │
│  │  eth0 (mgmt)          eth0 (mgmt)               │    │
│  └─────────────────────────────────────────────────┘    │
│                    │                                     │
│            VPC private subnet                           │
│            10.0.0.0/16                                  │
│                    │                                     │
│         NAT Gateway / VPC Endpoints                     │
│         (S3, ECR, CloudWatch)                           │
└─────────────────────────────────────────────────────────┘
```

Key design rules:
- **Single AZ per training job.** EFA cluster placement groups span one AZ only. Inter-AZ EFA is not supported.
- **Separate management and data subnets.** EFA interfaces carry training traffic; eth0 carries SSH, monitoring, and checkpoint traffic. Never route training traffic through the management interface.
- **VPC endpoints for S3.** Checkpoint writes to S3 should use VPC endpoints to avoid traversing the internet gateway and incurring egress costs.
- **Security groups.** Allow all traffic between nodes within the placement group; restrict management ports to jump hosts.

### 25.6.2 Inter-AZ Latency Impact on AllReduce

For AllReduce collective operations, every node must communicate with every other node. The latency adds directly to the synchronization barrier time:

```
Within-AZ EFA (p5, same placement group):
  Point-to-point latency: ~10-15 µs
  AllReduce (8 nodes, 1 GB tensor): ~0.9 seconds

Cross-AZ (standard Ethernet, no EFA):
  Point-to-point latency: ~0.5-2 ms
  AllReduce (8 nodes, 1 GB tensor): ~25+ seconds (33× slower)
```

The practical rule: **never span a training job across AZs.** Use multi-AZ only for inference serving, checkpointing, and asynchronous data pipelines where synchronization barriers are not on the critical path.

### 25.6.3 Elastic Fabric Adapter Limits

```bash
# Check EFA device limits on a running p5 instance
# (requires efa-utils)
efadv_device_list
# Expected (p5.48xlarge):
# EFA Devices:
#  rdmap0s31    max_rdma_size: 2097152    device_caps: RDMA_WRITE SEND RECV
#  rdmap16s1    ...
#  ... (32 EFA interfaces on p5)

# Check current QP count and limits
cat /sys/class/infiniband/rdmap0s31/ports/1/counters/rdma_rc_current_connections
# Shows current active RC connections (important for large clusters)
```

---

## 25.7 Cloud vs On-Prem Decision Matrix

| Factor | Cloud | On-Prem |
|---|---|---|
| **Capital cost** | $0 upfront; per-hour billing | $500K–$50M+ for large clusters |
| **Operating cost** | Included in per-hour rate | Power, cooling, staff (~20% of CapEx/year) |
| **Time to first GPU** | Minutes (instance launch) | 3–18 months (procurement, cabling, burn-in) |
| **Peak bandwidth (per GPU)** | p5: 400 Gbps / GPU (EFA) | ConnectX-7 400GbE: 400 Gbps / GPU |
| **Fabric latency** | 10–15 µs (EFA) | 1–2 µs (IB HDR), 2–5 µs (RoCEv2) |
| **Burst capacity** | Instant (if capacity available) | Fixed; requires over-provisioning |
| **Fabric observability** | Limited (no SM access, no MADs) | Full (OpenSM, ibnetdiscover, perfquery) |
| **Custom firmware / tuning** | Not possible | Full control |
| **Multi-tenant security** | Provider guarantees | You control isolation |
| **Long-running jobs (>6 months)** | Expensive (reserved instances help) | Cost-effective |
| **Variable workloads** | Ideal (pay per use) | Inefficient (idle hardware) |

**Recommendation for 2024–2025:**
- **Startups and exploratory R&D:** Cloud. Time-to-experiment matters more than cost-per-FLOP.
- **Production training at scale (>1000 GPUs sustained):** On-prem or hybrid. Cloud reserved instances at $10M+/year exceed on-prem TCO.
- **Inference at scale:** Cloud-native (managed services, auto-scaling) strongly preferred.
- **Regulated industries (healthcare, finance):** On-prem or cloud with specific compliance certifications.

---

## Lab Walkthrough 25 — libfabric Provider Comparison: shm vs tcp vs simulated efa

This walkthrough explores libfabric providers available on any Ubuntu 24.04 host (no cloud account required), measures latency and bandwidth for the `shm` and `tcp` providers, writes a NCCL environment advisor script, and demonstrates the aws-ofi-nccl plugin discovery mechanism.

**Prerequisites:** All packages installed per the Installation section above. Python uv environment created.

### Step 1: Install libfabric and survey available providers

```bash
# Confirm fabtests tools are installed
which fi_info fi_pingpong fi_msg_bw
# Expected:
# /usr/bin/fi_info
# /usr/bin/fi_pingpong
# /usr/bin/fi_msg_bw

# List all available providers and their capabilities
fi_info --list
# Expected (standard Ubuntu 24.04 host):
# tcp
# udp
# shm
# rxm
# ofi_rxm
# verbs    (if rdma_rxe or real HCA is loaded)

# Full provider enumeration with capability flags
fi_info 2>/dev/null | grep -E "^provider:|^    fabric:|type:|protocol:"
# Expected excerpt:
# provider: shm
#     fabric: shm
#     type: FI_EP_RDM
#     protocol: FI_PROTO_SHM
# provider: tcp
#     fabric: 127.0.0.0/8
#     type: FI_EP_MSG
#     protocol: FI_PROTO_SOCK_TCP
# provider: udp
#     fabric: 127.0.0.0/8
#     type: FI_EP_DGRAM
#     protocol: FI_PROTO_UDP
```

### Step 2: Inspect the shm provider in depth

```bash
fi_info -p shm
# Expected output (key fields annotated):
#
# provider: shm
#     fabric: shm
#     domain: shm
#     version: 2.0               # shm provider version
#     type: FI_EP_RDM            # Reliable Datagram — connectionless reliable
#     protocol: FI_PROTO_SHM
#     tx_attr:
#         caps: FI_MSG|FI_TAGGED|FI_RMA|FI_ATOMIC|FI_READ|FI_WRITE|FI_SEND
#         inject_size: 4096      # max inline data size (no DMA for small msgs)
#         size: 1024             # tx context depth
#     rx_attr:
#         caps: FI_MSG|FI_TAGGED|FI_RMA|FI_ATOMIC|FI_RECV
#         size: 1024
#     domain_attr:
#         name: shm
#         threading: FI_THREAD_DOMAIN
#         data_progress: FI_PROGRESS_MANUAL
#         mr_mode: [ FI_MR_VIRT_ADDR FI_MR_ALLOCATED FI_MR_PROV_KEY ]
#         av_type: FI_AV_MAP
#     ep_attr:
#         max_msg_size: 4294967296    # 4 GB max message
#         mem_tag_format: 0xffff...

# inject_size: 4096 means messages ≤4 KB are transferred via a zero-copy
# shared-memory ring buffer without any DMA or system call.
# Larger messages use a separate mmap-based mechanism.
```

### Step 3: Inspect the tcp provider

```bash
fi_info -p tcp
# Expected (key differences from shm):
#
# provider: tcp
#     fabric: 127.0.0.0/8
#     domain: lo                 # loopback interface
#     type: FI_EP_MSG            # MSG = connection-oriented (unlike shm RDM)
#     protocol: FI_PROTO_SOCK_TCP
#     tx_attr:
#         caps: FI_MSG|FI_RMA|FI_READ|FI_WRITE|FI_SEND
#         inject_size: 65528     # TCP can inline up to ~64 KB in kernel send buf
#     domain_attr:
#         threading: FI_THREAD_SAFE
#         data_progress: FI_PROGRESS_AUTO    # kernel handles progress automatically
#         mr_mode: [ FI_MR_LOCAL ]           # no IOMMU pinning needed for TCP
#     ep_attr:
#         max_msg_size: 18446744073709551615 # effectively unlimited

# Key comparison vs shm:
# - tcp is FI_EP_MSG (connection-oriented) vs shm FI_EP_RDM (datagram)
# - tcp threading: FI_THREAD_SAFE (safe from multiple threads)
# - tcp data_progress: FI_PROGRESS_AUTO (kernel drives progress)
# - shm data_progress: FI_PROGRESS_MANUAL (app must call fi_cq_read)
```

### Step 4: fi_pingpong latency with shm provider

```bash
# Start server in background (uses UNIX socket at /tmp/shm-server)
fi_pingpong -p shm &
SHM_SERVER_PID=$!
sleep 0.5

# Run client (no IP needed — shm uses process communication)
fi_pingpong -p shm localhost
# Expected output:
#
# fi_pingpong: libfabric version: 1.20.x
# Options:
#   Provider:     shm
#   Transfer Size:  256
#   Iterations:     1000
#
# name:    shm
# bytes    #itr    t_min(us)    t_max(us)    t_avg(us)
# 256      1000    1.23         8.47         1.51
#
# Results:
#   Avg latency: 1.5 µs    (shared memory — extremely fast, no kernel crossing)
#   Min latency: 1.2 µs
#   Max latency: 8.5 µs    (occasional OS scheduler jitter)

wait $SHM_SERVER_PID 2>/dev/null

# Sweep message sizes
for size in 64 256 1024 4096 65536; do
    fi_pingpong -p shm -S $size &
    sleep 0.3
    fi_pingpong -p shm -S $size localhost 2>/dev/null | grep "^$size" || true
    wait 2>/dev/null
done
# Expected pattern: latency grows slowly from ~1 µs at 64B to ~5 µs at 64KB
```

### Step 5: fi_msg_bw bandwidth with shm provider

```bash
# Bandwidth test: server
fi_msg_bw -p shm &
BW_SERVER_PID=$!
sleep 0.5

# Client
fi_msg_bw -p shm localhost
# Expected output:
#
# fi_msg_bw: libfabric version: 1.20.x
# Options:
#   Provider:     shm
#   Transfer Size:  1048576    (1 MB default)
#   Iterations:     1000
#
# name:    shm
# bytes        #itr    BW peak(Gb/s)    BW avg(Gb/s)
# 1048576      1000    28.4             27.9
#
# shm bandwidth is limited by memory copy speed (DRAM bandwidth),
# not any network hardware. Expect 20-40 Gb/s on modern servers.

wait $BW_SERVER_PID 2>/dev/null
```

### Step 6: fi_pingpong with tcp provider — compare with shm

```bash
# TCP server on loopback
fi_pingpong -p tcp -s 127.0.0.1 &
TCP_SERVER_PID=$!
sleep 0.5

# TCP client
fi_pingpong -p tcp 127.0.0.1
# Expected output:
#
# fi_pingpong: libfabric version: 1.20.x
# Options:
#   Provider:     tcp
#   Transfer Size:  256
#   Iterations:     1000
#
# name:    tcp
# bytes    #itr    t_min(us)    t_max(us)    t_avg(us)
# 256      1000    15.82        124.37       18.44
#
# TCP loopback is ~12x slower than shm (18 µs vs 1.5 µs) due to kernel
# TCP stack traversal even on loopback. On a real network with a real EFA
# or IB NIC, the gap narrows: EFA ~8-15 µs, IB ~1.5-2 µs.

wait $TCP_SERVER_PID 2>/dev/null

# Summary comparison:
echo "=== Provider Latency Comparison (256 byte message) ==="
echo "  shm:  ~1.5 µs  (shared memory, same host)"
echo "  tcp:  ~18 µs   (kernel TCP loopback)"
echo "  efa:  ~8-15 µs (EFA SRD, cross-host, p5.48xlarge)"
echo "  IB:   ~1.5 µs  (InfiniBand RC, back-to-back HDR)"
```

### Step 7: Write nccl_env_advisor.py

This script parses `fi_info` output and recommends the optimal NCCL environment variables based on what providers and interfaces are available:

```bash
cat > ~/cloud-ai-net/nccl_env_advisor.py << 'PYEOF'
#!/usr/bin/env python3
"""
nccl_env_advisor.py — Inspect available libfabric providers and network
interfaces, then emit recommended NCCL_NET and NCCL_SOCKET_IFNAME values.
"""

import subprocess
import re
import json
from dataclasses import dataclass, field


@dataclass
class Provider:
    name: str
    fabric: str = ""
    domain: str = ""
    ep_type: str = ""
    inject_size: int = 0
    caps: list[str] = field(default_factory=list)


def parse_fi_info() -> list[Provider]:
    """Run fi_info and parse provider blocks."""
    try:
        out = subprocess.check_output(["fi_info"], text=True, stderr=subprocess.DEVNULL)
    except (FileNotFoundError, subprocess.CalledProcessError):
        return []

    providers: list[Provider] = []
    current: Provider | None = None

    for line in out.splitlines():
        line = line.rstrip()
        if line.startswith("provider:"):
            if current:
                providers.append(current)
            current = Provider(name=line.split(":", 1)[1].strip())
        elif current is None:
            continue
        elif "fabric:" in line:
            current.fabric = line.split(":", 1)[1].strip()
        elif "domain:" in line and not current.domain:
            current.domain = line.split(":", 1)[1].strip()
        elif "type:" in line:
            current.ep_type = line.split(":", 1)[1].strip()
        elif "inject_size:" in line:
            try:
                current.inject_size = int(line.split(":", 1)[1].strip())
            except ValueError:
                pass
        elif "caps:" in line:
            caps_str = line.split(":", 1)[1].strip()
            current.caps = [c.strip() for c in caps_str.split("|")]

    if current:
        providers.append(current)

    # Deduplicate by name (keep first occurrence per provider name)
    seen: set[str] = set()
    unique: list[Provider] = []
    for p in providers:
        if p.name not in seen:
            seen.add(p.name)
            unique.append(p)
    return unique


def get_network_interfaces() -> dict[str, dict]:
    """Return dict of interface name → {speed_mbps, state}."""
    interfaces: dict[str, dict] = {}
    try:
        # Use ip -j link to get interface names and states
        out = subprocess.check_output(
            ["ip", "-j", "link"], text=True, stderr=subprocess.DEVNULL
        )
        links = json.loads(out)
        for link in links:
            name = link.get("ifname", "")
            state = link.get("operstate", "UNKNOWN")
            interfaces[name] = {"state": state, "speed_mbps": 0}

        # Try to get speed via ethtool (may require sudo for some interfaces)
        for iface in list(interfaces.keys()):
            try:
                speed_out = subprocess.check_output(
                    ["ethtool", iface], text=True, stderr=subprocess.DEVNULL
                )
                m = re.search(r"Speed:\s+(\d+)", speed_out)
                if m:
                    interfaces[iface]["speed_mbps"] = int(m.group(1))
            except (subprocess.CalledProcessError, FileNotFoundError):
                pass
    except (subprocess.CalledProcessError, FileNotFoundError, json.JSONDecodeError):
        pass
    return interfaces


def recommend_nccl(providers: list[Provider], interfaces: dict[str, dict]) -> dict[str, str]:
    """Return recommended NCCL environment variables."""
    provider_names = {p.name for p in providers}
    env: dict[str, str] = {}

    # Determine NCCL_NET
    if "efa" in provider_names:
        env["NCCL_NET"] = "AWS-OFI-NCCL"
        env["FI_PROVIDER"] = "efa"
        env["FI_EFA_USE_DEVICE_RDMA"] = "1"
        env["FI_EFA_FORK_SAFE"] = "1"
        comment = "# AWS EFA detected — use aws-ofi-nccl plugin"
    elif "tcpxo" in provider_names:
        env["NCCL_NET"] = "tcpxo"
        comment = "# GCP GPUDirect-TCPX detected"
    elif "verbs" in provider_names:
        env["NCCL_NET"] = "IB"
        env["NCCL_IB_GID_INDEX"] = "3"
        comment = "# InfiniBand/RoCE verbs detected"
    else:
        env["NCCL_NET"] = "Socket"
        comment = "# No RDMA provider found — falling back to TCP socket transport"

    # Determine NCCL_SOCKET_IFNAME
    # Prefer the fastest non-loopback UP interface
    up_ifaces = {
        name: info for name, info in interfaces.items()
        if info["state"] == "UP" and name not in ("lo",) and not name.startswith("docker")
    }
    if up_ifaces:
        best = max(up_ifaces, key=lambda n: up_ifaces[n]["speed_mbps"])
        excluded = ["lo"] + [n for n in interfaces if n.startswith("docker")]
        exclude_str = "^" + ",".join(excluded)
        env["NCCL_SOCKET_IFNAME"] = exclude_str
        env["_RECOMMENDED_IFACE"] = best  # informational
    else:
        env["NCCL_SOCKET_IFNAME"] = "^lo,docker0"

    env["_COMMENT"] = comment
    return env


def main() -> None:
    print("=== NCCL Environment Advisor ===\n")

    providers = parse_fi_info()
    print(f"Detected libfabric providers: {[p.name for p in providers]}")

    interfaces = get_network_interfaces()
    up_ifaces = {n: i for n, i in interfaces.items() if i["state"] == "UP"}
    print(f"UP network interfaces: {list(up_ifaces.keys())}\n")

    recommendations = recommend_nccl(providers, interfaces)

    print("Recommended NCCL environment variables:")
    print("-" * 50)
    comment = recommendations.pop("_COMMENT", "")
    recommendations.pop("_RECOMMENDED_IFACE", None)
    if comment:
        print(comment)
    for key, value in recommendations.items():
        print(f"export {key}={value}")

    print("\n# Add to your training launch script or ~/.bashrc")
    print("# Re-run this script after adding/removing network interfaces or RDMA drivers")


if __name__ == "__main__":
    main()
PYEOF

# Run the advisor
cd ~/cloud-ai-net
python nccl_env_advisor.py
# Expected output on a standard Ubuntu 24.04 host with rxe loaded:
#
# === NCCL Environment Advisor ===
#
# Detected libfabric providers: ['tcp', 'udp', 'shm', 'rxm', 'verbs']
# UP network interfaces: ['lo', 'eth0', 'veth0']
#
# Recommended NCCL environment variables:
# --------------------------------------------------
# # InfiniBand/RoCE verbs detected
# export NCCL_NET=IB
# export NCCL_IB_GID_INDEX=3
# export NCCL_SOCKET_IFNAME=^lo,docker0
#
# # Add to your training launch script or ~/.bashrc
# # Re-run this script after adding/removing network interfaces or RDMA drivers
```

### Step 8: aws-ofi-nccl build and plugin discovery

Even without an AWS account, understanding the plugin discovery mechanism helps debug NCCL transport issues on any platform:

```bash
# Show the aws-ofi-nccl build configuration (from Installation section)
# The key configure flags and what they control:

cat << 'EOF'
# aws-ofi-nccl configure flags explained:
#
# --with-libfabric=/opt/amazon/efa
#   Path to libfabric headers and libraries (fi.h, libfabric.so)
#   The plugin calls fi_getinfo(), fi_domain(), fi_endpoint() etc.
#
# --with-cuda=/usr/local/cuda
#   CUDA headers needed for GPU memory registration
#   (cudaGetDeviceProperties, cuPointerGetAttribute for GPUDirect)
#
# --with-nccl=/usr/local/nccl
#   NCCL headers: nccl_net.h defines the transport plugin interface:
#     struct ncclNet_v8_t {
#         const char* name;       // "AWS-OFI-NCCL"
#         ncclResult_t (*init)(ncclDebugLogger_t logFunction);
#         ncclResult_t (*devices)(int* ndev);
#         ncclResult_t (*getProperties)(int dev, ncclNetProperties_v8_t* props);
#         ncclResult_t (*listen)(...);
#         ncclResult_t (*connect)(...);
#         ncclResult_t (*accept)(...);
#         ncclResult_t (*regMr)(...);    // register memory (fi_mr_reg)
#         ncclResult_t (*deregMr)(...);
#         ncclResult_t (*isend)(...);    // fi_tsend / fi_write
#         ncclResult_t (*irecv)(...);    // fi_trecv
#         ncclResult_t (*test)(...);     // fi_cq_read (poll completions)
#         ncclResult_t (*closeSend)(...);
#         ncclResult_t (*closeRecv)(...);
#         ncclResult_t (*closeConn)(...);
#     };
#
# --enable-platform-aws
#   Enables AWS-specific optimizations:
#   - SRD transport hints (FI_OPT_EFA_RNR_RETRY)
#   - EFA device ordering aligned with GPU topology
#   - Optimized MR key management for EFA

EOF

# Plugin discovery mechanism: NCCL searches LD_LIBRARY_PATH for libnccl-net.so
# At startup, NCCL calls dlopen("libnccl-net.so") and then looks for the symbol:
#   ncclNet_v8_t* ncclNetPlugin_v8;
# If NCCL_NET=AWS-OFI-NCCL, NCCL also tries dlopen("libnccl-net.so") explicitly.

# Simulate plugin discovery without AWS hardware:
# Create a stub plugin directory to demonstrate the dlopen path
mkdir -p /tmp/nccl-plugin-demo
cat << 'CEOF' > /tmp/nccl-plugin-demo/check_plugin.c
#include <stdio.h>
#include <dlfcn.h>

int main(int argc, char *argv[]) {
    const char *plugin_path = argc > 1 ? argv[1] : "libnccl-net.so";
    void *handle = dlopen(plugin_path, RTLD_NOW);
    if (!handle) {
        printf("Plugin NOT found: %s\n", dlerror());
        return 1;
    }
    // Try to find the plugin symbol (v8 is current as of NCCL 2.19+)
    void *sym = dlsym(handle, "ncclNetPlugin_v8");
    if (sym) {
        printf("Plugin loaded OK: found ncclNetPlugin_v8 symbol\n");
    } else {
        printf("Plugin loaded but ncclNetPlugin_v8 symbol not found\n");
        printf("  (older plugin? try ncclNetPlugin_v6 or ncclNetPlugin_v7)\n");
    }
    dlclose(handle);
    return 0;
}
CEOF

gcc -o /tmp/nccl-plugin-demo/check_plugin /tmp/nccl-plugin-demo/check_plugin.c -ldl
echo "Plugin checker compiled."

# Check if aws-ofi-nccl plugin is installed (will only be present after build)
if [ -f /usr/local/aws-ofi-nccl/lib/libnccl-net.so ]; then
    LD_LIBRARY_PATH=/usr/local/aws-ofi-nccl/lib \
        /tmp/nccl-plugin-demo/check_plugin libnccl-net.so
else
    echo "aws-ofi-nccl not installed on this host (expected outside AWS)"
    echo "In production, the activation flow is:"
    echo "  1. Build and install aws-ofi-nccl → /usr/local/aws-ofi-nccl/lib/libnccl-net.so"
    echo "  2. export LD_LIBRARY_PATH=/usr/local/aws-ofi-nccl/lib:\$LD_LIBRARY_PATH"
    echo "  3. export NCCL_NET=AWS-OFI-NCCL"
    echo "  4. NCCL startup: dlopen(\"libnccl-net.so\") → ncclNetPlugin_v8 → plugin init()"
fi
```

### Step 9: Topology file generation script

NCCL uses an XML topology file to understand the hardware hierarchy (NUMA nodes, PCIe topology, GPU-NIC affinity). The following script generates a `NCCL_TOPO_FILE`-compatible XML by enumerating local hardware:

```bash
cat > ~/cloud-ai-net/gen_nccl_topo.py << 'PYEOF'
#!/usr/bin/env python3
"""
gen_nccl_topo.py — Generate a NCCL_TOPO_FILE-compatible XML topology
by enumerating NUMA nodes, GPUs (via nvidia-smi or stub), and RDMA NICs.
"""

import subprocess
import re
import os
from pathlib import Path
from xml.etree.ElementTree import Element, SubElement, tostring
from xml.dom.minidom import parseString


def get_numa_nodes() -> list[int]:
    """Return list of NUMA node IDs."""
    numa_path = Path("/sys/devices/system/node")
    if not numa_path.exists():
        return [0]
    nodes = [
        int(p.name[4:])
        for p in numa_path.iterdir()
        if p.name.startswith("node") and p.name[4:].isdigit()
    ]
    return sorted(nodes)


def get_gpus() -> list[dict]:
    """Return list of GPU dicts with index, pci_bus, and numa_node."""
    gpus = []
    try:
        out = subprocess.check_output(
            ["nvidia-smi", "--query-gpu=index,pci.bus_id,numa_affinity",
             "--format=csv,noheader"],
            text=True, stderr=subprocess.DEVNULL,
        )
        for line in out.strip().splitlines():
            parts = [p.strip() for p in line.split(",")]
            if len(parts) >= 2:
                idx = int(parts[0])
                pci = parts[1]
                numa = int(parts[2]) if len(parts) > 2 and parts[2].isdigit() else 0
                gpus.append({"index": idx, "pci_bus": pci, "numa_node": numa})
    except (FileNotFoundError, subprocess.CalledProcessError):
        # Stub: pretend 2 GPUs on NUMA 0 for lab purposes
        gpus = [
            {"index": 0, "pci_bus": "0000:01:00.0", "numa_node": 0},
            {"index": 1, "pci_bus": "0000:02:00.0", "numa_node": 0},
        ]
        print("  (nvidia-smi not found — using stub GPU data)")
    return gpus


def get_rdma_nics() -> list[dict]:
    """Return list of RDMA NIC dicts with name, netdev, and numa_node."""
    nics = []
    ib_path = Path("/sys/class/infiniband")
    if not ib_path.exists():
        return nics
    for dev_path in sorted(ib_path.iterdir()):
        name = dev_path.name
        # Find the netdev associated with this RDMA device
        netdev = "unknown"
        netdev_path = dev_path / "device" / "net"
        if netdev_path.exists():
            netdevs = list(netdev_path.iterdir())
            if netdevs:
                netdev = netdevs[0].name
        # Find NUMA node
        numa_node = 0
        numa_file = dev_path / "device" / "numa_node"
        if numa_file.exists():
            try:
                numa_node = max(0, int(numa_file.read_text().strip()))
            except ValueError:
                pass
        nics.append({"name": name, "netdev": netdev, "numa_node": numa_node})
    return nics


def build_topology_xml(
    numa_nodes: list[int],
    gpus: list[dict],
    nics: list[dict],
) -> str:
    """Build NCCL topology XML."""
    root = Element("system")
    root.set("version", "1")

    for numa_id in numa_nodes:
        cpu = SubElement(root, "cpu")
        cpu.set("numaid", str(numa_id))
        cpu.set("affinity", f"{numa_id}")
        cpu.set("arch", "x86_64")

        pci_node = SubElement(cpu, "pci")
        pci_node.set("busid", f"0000:{numa_id:02x}:00.0")

        # Add GPUs on this NUMA node
        for gpu in gpus:
            if gpu["numa_node"] == numa_id:
                gpu_elem = SubElement(pci_node, "gpu")
                gpu_elem.set("id", str(gpu["index"]))
                gpu_elem.set("sm", "90")          # stub: H100 SM version
                gpu_elem.set("rank", str(gpu["index"]))
                gpu_elem.set("gdr", "1")           # GPUDirect RDMA capable
                pci_gpu = SubElement(gpu_elem, "pci")
                pci_gpu.set("busid", gpu["pci_bus"])

        # Add RDMA NICs on this NUMA node
        for nic in nics:
            if nic["numa_node"] == numa_id:
                nic_elem = SubElement(pci_node, "net")
                nic_elem.set("name", nic["netdev"])
                nic_elem.set("dev", nic["name"])
                nic_elem.set("speed", "400000")    # Mbps, stub value
                nic_elem.set("port", "1")
                nic_elem.set("gdr", "1")

    xml_str = tostring(root, encoding="unicode")
    return parseString(xml_str).toprettyxml(indent="  ")


def main() -> None:
    output_path = Path("/tmp/nccl_topo.xml")

    print("Enumerating hardware for NCCL topology file...")

    numa_nodes = get_numa_nodes()
    print(f"  NUMA nodes: {numa_nodes}")

    gpus = get_gpus()
    print(f"  GPUs: {[g['index'] for g in gpus]}")

    nics = get_rdma_nics()
    print(f"  RDMA NICs: {[n['name'] for n in nics]}")

    xml = build_topology_xml(numa_nodes, gpus, nics)

    output_path.write_text(xml)
    print(f"\nTopology file written to: {output_path}")
    print(f"  {len(xml.splitlines())} lines")
    print(f"\nActivate with:")
    print(f"  export NCCL_TOPO_FILE={output_path}")
    print(f"\nTopology XML preview:")
    print("-" * 60)
    for line in xml.splitlines()[:30]:
        print(line)
    if len(xml.splitlines()) > 30:
        print(f"  ... ({len(xml.splitlines()) - 30} more lines)")


if __name__ == "__main__":
    main()
PYEOF

# Run the topology generator
cd ~/cloud-ai-net
python gen_nccl_topo.py
# Expected output:
#
# Enumerating hardware for NCCL topology file...
#   NUMA nodes: [0]
#   (nvidia-smi not found — using stub GPU data)
#   GPUs: [0, 1]
#   RDMA NICs: ['rxe0', 'rxe1']   (if rxe devices exist)
#
# Topology file written to: /tmp/nccl_topo.xml
#   42 lines
#
# Activate with:
#   export NCCL_TOPO_FILE=/tmp/nccl_topo.xml
#
# Topology XML preview:
# ------------------------------------------------------------
# <?xml version="1.0" ?>
# <system version="1">
#   <cpu numaid="0" affinity="0" arch="x86_64">
#     <pci busid="0000:00:00.0">
#       <gpu id="0" sm="90" rank="0" gdr="1">
#         <pci busid="0000:01:00.0"/>
#       </gpu>
#       <gpu id="1" sm="90" rank="1" gdr="1">
#         <pci busid="0000:02:00.0"/>
#       </gpu>
#       <net name="veth0" dev="rxe0" speed="400000" port="1" gdr="1"/>
#       <net name="veth1" dev="rxe1" speed="400000" port="1" gdr="1"/>
#     </pci>
#   </cpu>
# </system>
```

### Step 10: Cleanup

```bash
# Kill any background fi_pingpong or fi_msg_bw processes
pkill -f fi_pingpong 2>/dev/null || true
pkill -f fi_msg_bw 2>/dev/null || true

# Remove rxe devices if created during this session
sudo rdma link delete rxe0 2>/dev/null || true
sudo rdma link delete rxe1 2>/dev/null || true

# Remove veth pair if created
sudo ip link del veth0 2>/dev/null || true

# Clean up build artifacts
rm -rf /tmp/nccl-plugin-demo

# Remove generated topology file (optional — export NCCL_TOPO_FILE if keeping it)
rm -f /tmp/nccl_topo.xml

# Unload rxe module if loaded only for this lab
sudo modprobe -r rdma_rxe 2>/dev/null || true

# Verify cleanup
fi_info --list
# Expected: tcp, udp, shm (no verbs if rxe was removed)

echo "Lab 25 cleanup complete"
```

---

## Summary

- Cloud AI networking replaces physical fabric management with provider-managed abstractions; the key implication is that engineers control congestion and placement through instance type selection and placement group configuration rather than switch-level tuning.
- AWS EFA uses the libfabric `efa` provider with SRD transport; `aws-ofi-nccl` bridges NCCL to EFA via a plugin that implements the `ncclNet_v8_t` interface; p5.48xlarge provides 3200 Gbps aggregate EFA bandwidth.
- Azure exposes real InfiniBand NICs (HDR/NDR) via SR-IOV passthrough on HBv3/NDv5; `hpc-x` provides a pre-tuned MPI+NCCL stack; the Azure SM is fully managed and invisible to tenants.
- GCP GPUDirect-TCPX achieves GPU-to-GPU data movement over TCP with NIC hardware assist, delivering ~2x the bandwidth of CPU-mediated TCP while retaining standard IP routability.
- Critical NCCL variables are `NCCL_NET` (selects transport plugin), `NCCL_IB_HCA` (selects RDMA devices), `NCCL_SOCKET_IFNAME` (selects management interface), and `NCCL_TOPO_FILE` (overrides automatic topology detection).
- Training jobs must be confined to a single AZ and placement group; cross-AZ AllReduce is 30–100x slower and economically indefensible for synchronous training.
- The libfabric `shm` provider delivers ~1.5 µs latency and ~28 Gb/s bandwidth for intra-host communication; `tcp` loopback is ~18 µs; the gap vs. EFA (~10 µs) and IB (~1.5 µs) reflects real hardware capability, not software configuration.
- The cloud vs. on-prem decision hinges on workload duration and scale: cloud is optimal for burst and exploratory workloads; on-prem has lower per-FLOP cost for sustained large-scale training above roughly 1000 GPUs.

---

## References

- [AWS EFA (Elastic Fabric Adapter) User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [aws-ofi-nccl (AWS OFI NCCL Plugin)](https://github.com/aws/aws-ofi-nccl)
- [efa-utils](https://github.com/aws/efa-utils)
- [libfabric / OFI (OpenFabrics Interfaces)](https://github.com/ofiwg/libfabric)
- [libfabric Programmer's Manual](https://ofiwg.github.io/libfabric/)
- [fabtests (fi_pingpong, fi_msg_bw, fi_info)](https://github.com/ofiwg/libfabric/tree/main/fabtests)
- [NCCL (NVIDIA Collective Communications Library)](https://docs.nvidia.com/deeplearning/nccl/)
- [Azure NDv5 series (HPC VMs with InfiniBand)](https://learn.microsoft.com/en-us/azure/virtual-machines/ndv5-series)
- [Azure RDMA / InfiniBand enablement](https://learn.microsoft.com/en-us/azure/virtual-machines/extensions/enable-infiniband)
- [Google Cloud TPU / Pathways](https://cloud.google.com/tpu/docs)
- [Google Cloud GPUDirect-TCPX overview](https://cloud.google.com/compute/docs/gpus/gpudirect)
- [hpc-x (Mellanox HPC toolkit: OpenMPI + UCX)](https://docs.nvidia.com/networking/display/hpcxv2)
- [UCX (Unified Communication X)](https://openucx.org)
- [OpenMPI](https://www.open-mpi.org)
- [MVAPICH2](https://mvapich.cse.ohio-state.edu)
- [hwloc / lstopo](https://www.open-mpi.org/projects/hwloc/)
- [rdma-core (libibverbs)](https://github.com/linux-rdma/rdma-core)
- [RoCEv2](https://www.infinibandta.org/roce/)
- [boto3 (AWS SDK for Python)](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [paramiko](https://www.paramiko.org)
- [fabric (Python SSH library)](https://www.fabfile.org)
- Klenk et al., *An In-Depth Analysis of the EFA SRD Transport*, IEEE IPDPS 2020 — [ieeexplore.ieee.org/document/9139797](https://ieeexplore.ieee.org/document/9139797)
- Hoefler et al., *Scalable Communication Protocols for Dynamic Sparse Data Exchange*, PPoPP 2010 — [dl.acm.org/doi/10.1145/1837853.1693465](https://dl.acm.org/doi/10.1145/1837853.1693465)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).