# Chapter 6 — SPDK & NVMe-oF

**Part II: Kernel-Bypass & Programmable I/O** | ~20 pages

---

## Introduction

Storage is not a peripheral concern in AI training clusters — it is a critical path component that can idle thousands of GPUs if not engineered carefully. This chapter examines **SPDK** (**Storage Performance Development Kit**) and **NVMe-oF** (**NVMe over Fabrics**), two technologies that together bring storage I/O performance into alignment with the throughput and latency demands of large-scale model training.

The central problem is checkpoint I/O. A 70B-parameter model in fp32 with optimizer state occupies roughly 800 GB on disk. Training runs checkpoint periodically — often every few hundred steps — to guard against hardware failure. If the checkpoint write takes longer than a single training step, GPUs stall, wasting expensive accelerator time. The **Linux** kernel's **NVMe** driver, designed for general-purpose workloads, leaves significant performance on the table through interrupt overhead, block-layer latency, and memory copies between kernel and user space. **SPDK** solves this by moving the **NVMe** driver entirely into user space, adopting a polling model identical in spirit to **DPDK**'s kernel-bypass approach from Chapter 5.

**NVMe-oF** extends the **NVMe** command set across a network fabric, allowing remote SSDs to appear as local **NVMe** devices. Over **RDMA** (**RoCE** or **InfiniBand**), **NVMe-oF** achieves sub-10 µs latency — comparable to local **PCIe**-attached **NVMe**. Over **TCP**, it provides a practical checkpoint path on the management **Ethernet** when **RDMA** bandwidth is reserved for collective communication. This separation of checkpoint traffic from gradient traffic is a key architectural pattern that recurs throughout Part IV of this book.

The reader will learn: how **SPDK**'s polling reactor model eliminates I/O overhead; how to configure an **NVMe-oF** target and initiator with both **RDMA** and **TCP** transports; how **SPDK**'s blobstore and **RAID** layers compose for high-bandwidth checkpoint writing; and how to benchmark the full stack with **fio** and **io_uring**. The lab walkthrough requires no physical **NVMe** hardware — a RAM-backed `bdev_malloc` provides a functionally complete **NVMe-oF** target for development and testing.

This chapter connects directly to Chapter 5 (**DPDK**) — **SPDK**'s user-space architecture mirrors **DPDK**'s philosophy exactly, reusing **DPDK**'s **EAL** and hugepage memory allocator. It sets the stage for Chapter 18 (Distributed Storage), which addresses the cluster-wide incast and consistency problems that arise when thousands of GPUs checkpoint simultaneously.

---

## Installation

**SPDK** must be built from source because no distribution package exists; the build depends on **libaio**, **liburing**, and optionally **libibverbs** for **RDMA** transport support. The `fio` benchmark tool and `nvme-cli` are installed from apt to drive I/O workloads against the **NVMe-oF** target and connect from the initiator side. The `nvme-tcp` kernel module is loaded at runtime to enable the kernel-side **NVMe-oF** **TCP** initiator, which connects to the **SPDK** target without requiring **RDMA** hardware. **Python** management scripts use `paramiko` for issuing **SPDK** **JSON-RPC** commands to remote nodes over **SSH**.

### Ubuntu 24.04 — apt prerequisites

SPDK must be built from source (no distro package exists). Install all build and runtime dependencies first:

```bash
sudo apt install -y fio nvme-cli libaio-dev liburing-dev pkg-config \
    uuid-dev libssl-dev python3-pip \
    build-essential cmake git nasm \
    libnuma-dev libcunit1-dev libiscsi-dev \
    python3-pyelftools meson ninja-build
```

For RDMA (RoCE) transport support also install:

```bash
sudo apt install -y libibverbs-dev librdmacm-dev
```

### Building SPDK from source

```bash
git clone https://github.com/spdk/spdk --recursive
cd spdk

# Configure: --with-rdma enables RoCE/InfiniBand transport
# Omit --with-rdma if you only have standard Ethernet
./configure --with-rdma

# Build (uses all available cores)
make -j$(nproc)
```

Expected final lines from a successful build:

```
  [LINK] build/bin/nvmf_tgt
  [LINK] build/bin/spdk_tgt
  [LINK] build/examples/hello_bdev
```

Verify the target binary is present:

```bash
ls -lh build/bin/nvmf_tgt
# -rwxr-xr-x 1 user user 48M Apr 22 10:30 build/bin/nvmf_tgt
```

### Python uv setup for management scripts

SPDK exposes a JSON-RPC socket for runtime configuration. Python scripts that talk to this socket (or manage SPDK remotely via SSH) benefit from an isolated venv. `paramiko` is a Python implementation of the SSH protocol, used here to issue SPDK RPC commands on remote nodes over a secure channel:

```bash
# From within the spdk/ directory:
uv venv .venv
source .venv/bin/activate

# Install remote management dependencies
uv pip install paramiko

# Verify
python -c "import paramiko; print(paramiko.__version__)"
# Expected: 3.x.x
```

---

## 6.1 Storage as a First-Class Fabric Concern

Training a 70B-parameter model generates checkpoints at regular intervals — typically every 500–1000 steps to guard against hardware failure. A single checkpoint of weights + optimizer state in fp32 is ~800 GB. At a step time of 30 seconds, a checkpoint every 500 steps represents one checkpoint every ~4 hours — but *writing* that checkpoint must not stall the GPUs.

Without a capable storage fabric:
- Writing 800 GB to a single NVMe SSD at 7 GB/s takes ~115 seconds — GPU idle time
- Naive checkpoint to networked storage saturates the management network, starving gradient traffic
- Checkpointing across 1000 GPUs simultaneously creates a storage incast storm

SPDK and NVMe-oF solve the first two problems. Chapter 18 (Distributed Storage) addresses the third.

---

## 6.2 Why Kernel NVMe Is Insufficient

The Linux `nvme` kernel driver uses an interrupt-driven model with a queue depth (QD) typically capped at 1024. The NVMe spec supports up to 65535 I/O queues with 65536 commands each. The kernel never exposes this parallelism fully because:

- Each submitted I/O goes through the block layer (`blk_mq`), adding ~2–4 µs of overhead
- Interrupts serialize completion processing
- Memory copies between kernel and user space add latency

At NVMe speeds (7 GB/s sequential, 1M+ IOPS), even 2 µs overhead per I/O represents 500K IOPS lost — 50% of a high-end drive's capability.

---

## 6.3 SPDK Architecture

SPDK (Storage Performance Development Kit) is an open-source library from Intel that provides a collection of user-space, poll-mode drivers and tools for building high-performance storage applications. SPDK mirrors DPDK's philosophy applied to storage: move the NVMe driver entirely to user space, poll for completions rather than interrupt, and process I/O on a dedicated CPU core.

```
┌────────────────────────────────────────────────┐
│ Application / NVMe-oF target / blobstore       │
├────────────────────────────────────────────────┤
│ bdev layer (block device abstraction)          │
│   bdev_nvme  bdev_malloc  bdev_raid  bdev_lvol │
├────────────────────────────────────────────────┤
│ SPDK NVMe driver (user space)                  │
│   Submission/completion queue management       │
│   DMA buffer management                        │
├──────────────────────────┬─────────────────────┤
│        VFIO              │   NVMe SSD hardware  │
└──────────────────────────┴─────────────────────┘
```

VFIO (Virtual Function I/O) is a Linux kernel framework that allows user-space programs to directly control PCI devices — including NVMe SSDs — by safely removing them from kernel drivers and granting DMA access through the IOMMU. SPDK uses VFIO to bind NVMe devices to user space, enabling its poll-mode driver to program submission and completion queues directly without any kernel involvement.

### 6.3.1 Initialization

```c
// Initialize SPDK event framework
struct spdk_app_opts opts;
spdk_app_opts_init(&opts, sizeof(opts));
opts.name = "storage_app";
opts.json_config_file = "spdk.json";

spdk_app_start(&opts, start_fn, NULL);
```

```json
// spdk.json — bind NVMe device
{
  "subsystems": [{
    "subsystem": "bdev",
    "config": [{
      "method": "bdev_nvme_attach_controller",
      "params": {
        "name": "NVMe0",
        "trtype": "PCIe",
        "traddr": "0000:01:00.0"
      }
    }]
  }]
}
```

### 6.3.2 I/O Submission Model

```c
void io_complete(struct spdk_bdev_io *bdev_io, bool success, void *ctx) {
    spdk_bdev_free_io(bdev_io);
    // handle completion
}

// Submit asynchronous read — no blocking, no system call
spdk_bdev_read(desc, channel, buf, offset, length, io_complete, ctx);

// The event loop polls for completions:
// spdk_bdev_poll_io_completions() called by the reactor thread
```

SPDK's reactor model pins one thread per CPU core. Each reactor runs a tight poll loop that checks NVMe completion queues and dispatches callbacks — identical to DPDK's lcore model.

---

## 6.4 NVMe-oF: Extending NVMe Over the Fabric

NVMe-oF (NVMe over Fabrics) separates the NVMe command set from the physical PCIe attachment, allowing NVMe drives to be accessed over:

- **RDMA (RoCE / InfiniBand):** Highest performance, sub-10 µs latency
- **TCP:** Universal but adds ~20–50 µs; useful where RDMA isn't available
- **Fibre Channel:** Enterprise/storage-area-network environments

### 6.4.1 Target Configuration

```bash
# Configure NVMe-oF target using SPDK's nvmf_tgt
# JSON config for RoCE (RDMA) transport:
{
  "subsystems": [
    {
      "subsystem": "nvmf",
      "config": [
        {
          "method": "nvmf_create_transport",
          "params": { "trtype": "RDMA", "max_queue_depth": 128 }
        },
        {
          "method": "nvmf_create_subsystem",
          "params": {
            "nqn": "nqn.2024-01.io.spdk:cnode1",
            "allow_any_host": true,
            "serial_number": "SPDK001"
          }
        },
        {
          "method": "nvmf_subsystem_add_ns",
          "params": {
            "nqn": "nqn.2024-01.io.spdk:cnode1",
            "bdev_name": "NVMe0n1"
          }
        },
        {
          "method": "nvmf_subsystem_add_listener",
          "params": {
            "nqn": "nqn.2024-01.io.spdk:cnode1",
            "listen_address": {
              "trtype": "RDMA",
              "adrfam": "IPv4",
              "traddr": "192.168.1.10",
              "trsvcid": "4420"
            }
          }
        }
      ]
    }
  ]
}
```

### 6.4.2 Initiator (Host) Connection

```bash
# Discover targets
nvme discover -t rdma -a 192.168.1.10 -s 4420

# Connect to subsystem
nvme connect -t rdma -a 192.168.1.10 -s 4420 \
    -n nqn.2024-01.io.spdk:cnode1

# Device appears as /dev/nvmeXnY
# Benchmark with fio (flexible I/O tester):
# io_uring is a Linux kernel I/O interface (introduced in 5.1) that uses
# shared ring buffers between kernel and user space for zero-syscall async I/O.
fio --filename=/dev/nvme1n1 --rw=randread --bs=4k \
    --numjobs=4 --iodepth=64 --runtime=30 \
    --ioengine=io_uring --direct=1 --name=nvmeof_test
```

### 6.4.3 NVMe-oF TCP for Checkpoint Offload

When RDMA NICs are reserved for collective traffic (training), NVMe-oF over TCP provides a practical checkpoint path using the management Ethernet:

```bash
# Target: start TCP transport (via SPDK RPC)
scripts/rpc.py nvmf_create_transport -t TCP -u 16384

# Initiator: connect via TCP
nvme connect -t tcp -a 192.168.100.10 -s 4420 \
    -n nqn.2024-01.io.spdk:checkpoint_store

# fio with io_uring for async TCP NVMe-oF
fio --ioengine=io_uring --rw=write --bs=1M \
    --numjobs=8 --iodepth=32 --direct=1 \
    --filename=/dev/nvme2n1 --name=checkpoint
```

Expected: 3–5 GB/s write bandwidth over 25GbE management, sufficient to write an 800 GB checkpoint in ~2–4 minutes of GPU idle time.

---

## 6.5 SPDK Blobstore and BlobFS

For applications that need a file-like interface to SPDK without a full POSIX filesystem:

- **Blobstore:** manages "blobs" (variable-size objects) on an SPDK bdev; handles metadata, allocation, and crash consistency
- **BlobFS:** a minimal filesystem on top of blobstore; used by RocksDB-SPDK for write-optimized KV storage. RocksDB is Facebook's open-source log-structured merge (LSM) key-value store; the SPDK integration replaces its POSIX file I/O with direct SPDK blobstore calls to eliminate kernel overhead on write-intensive workloads.

This is relevant for checkpointing frameworks (e.g., PyTorch's async checkpoint — a feature introduced in PyTorch 2.0 that serializes model state in a background thread while training continues on the GPU) that need atomic, crash-consistent writes without the overhead of a full POSIX filesystem.

---

## 6.6 RAID and Erasure Coding

SPDK's `bdev_raid` presents a virtual RAID device across multiple underlying bdevs, all in user space:

```json
{
  "method": "bdev_raid_create",
  "params": {
    "name": "Raid0",
    "raid_level": "0",
    "strip_size_kb": 64,
    "base_bdevs": ["NVMe0n1", "NVMe1n1", "NVMe2n1", "NVMe3n1"]
  }
}
```

RAID0 across 4 NVMe drives yields ~28 GB/s sequential write — enough to write a 70B checkpoint in under 30 seconds with 8 parallel writer processes.

---

## Lab Walkthrough 6 — SPDK NVMe-oF Target and fio Benchmark

This walkthrough uses a `bdev_malloc` (RAM-backed block device) so no physical NVMe drive is required. All steps run on a single Ubuntu 24.04 host. The NVMe-oF transport used is TCP, which requires no special NICs.

---

### Step 1 — Clone and build SPDK

```bash
git clone https://github.com/spdk/spdk --recursive
cd spdk
```

Install SPDK's own dependency script (it installs CUnit, nasm, and other build deps not in apt):

```bash
sudo scripts/pkgdep.sh
```

Expected output (truncated):

```
Checking for NASM...
Installing NASM from source...
Checking for CUnit...
Installing CUnit...
```

Configure and build:

```bash
./configure --with-rdma 2>&1 | tail -5
```

Expected configure summary:

```
  RDMA      yes
  FC        no
  iSCSI     no
  vhost     yes
  crypto    no
```

```bash
make -j$(nproc)
```

A successful build ends with:

```
  [LINK] build/bin/nvmf_tgt
  [LINK] build/bin/spdk_tgt
```

Verify both required binaries exist:

```bash
ls -lh build/bin/nvmf_tgt scripts/rpc.py 2>/dev/null || ls build/bin/
```

---

### Step 2 — Run scripts/setup.sh to bind NVMe to vfio-pci

SPDK's setup script unbinds NVMe devices from the kernel `nvme` driver and binds them to `vfio-pci`, and also allocates hugepages:

```bash
sudo HUGEMEM=4096 scripts/setup.sh
```

Expected output:

```
0000:01:00.0 (class=0x010802 nvme): nvme -> vfio-pci
Reserving hugepages
2048 hugepages of size 2M reserved on NUMA node 0
```

Verify the NVMe device is now bound to vfio-pci:

```bash
sudo scripts/setup.sh status
```

Expected:

```
NVMe devices
BDF           Vendor Device NUMA    Driver
0000:01:00.0  8086   f1a8   0       vfio-pci
```

If you have no spare NVMe drive (or want to protect your data), skip this step entirely. The next step uses a `bdev_malloc` which requires no physical device.

---

### Step 3 — Test with malloc bdev (no real NVMe needed)

A `bdev_malloc` allocates a block device entirely in DRAM. It is functionally identical to a bdev_nvme from the NVMe-oF protocol perspective and is the standard way to validate SPDK NVMe-oF without risking data.

Create the target configuration file `nvmf_tcp_malloc.json`:

```json
{
  "subsystems": [
    {
      "subsystem": "bdev",
      "config": [
        {
          "method": "bdev_malloc_create",
          "params": {
            "name": "Malloc0",
            "num_blocks": 131072,
            "block_size": 512,
            "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
          }
        }
      ]
    },
    {
      "subsystem": "nvmf",
      "config": [
        {
          "method": "nvmf_create_transport",
          "params": {
            "trtype": "TCP",
            "max_queue_depth": 128,
            "max_io_size": 131072,
            "io_unit_size": 131072
          }
        },
        {
          "method": "nvmf_create_subsystem",
          "params": {
            "nqn": "nqn.2024-01.io.spdk:malloc0",
            "allow_any_host": true,
            "serial_number": "SPDKLAB01",
            "model_number": "SPDK bdev Controller"
          }
        },
        {
          "method": "nvmf_subsystem_add_ns",
          "params": {
            "nqn": "nqn.2024-01.io.spdk:malloc0",
            "bdev_name": "Malloc0",
            "nsid": 1
          }
        },
        {
          "method": "nvmf_subsystem_add_listener",
          "params": {
            "nqn": "nqn.2024-01.io.spdk:malloc0",
            "listen_address": {
              "trtype": "TCP",
              "adrfam": "IPv4",
              "traddr": "127.0.0.1",
              "trsvcid": "4420"
            }
          }
        }
      ]
    }
  ]
}
```

The malloc bdev is 64 MB (131072 blocks × 512 bytes). Adjust `num_blocks` for a larger target.

---

### Step 4 — Start nvmf_tgt with TCP transport and malloc bdev

```bash
sudo build/bin/nvmf_tgt \
    -c nvmf_tcp_malloc.json \
    -m 0x1 \
    --wait-for-rpc &
NVMF_PID=$!
```

Wait for the "Waiting for RPC" message, then apply the config via RPC:

```bash
sudo scripts/rpc.py nvmf_get_subsystems
```

Or, if using the `--wait-for-rpc` flag, start it without a config and configure via RPC calls:

```bash
# Simpler: just start with the JSON config (no --wait-for-rpc needed)
sudo build/bin/nvmf_tgt -c nvmf_tcp_malloc.json -m 0x1 &
NVMF_PID=$!
sleep 3
```

Expected startup log lines:

```
[2024-01-01 10:00:00.000] Starting SPDK v23.09 / DPDK 23.07.0 initialization...
[2024-01-01 10:00:00.100] EAL: Detected 16 lcore(s)
[2024-01-01 10:00:00.200] bdev_malloc: Malloc0 (134217728 bytes)
[2024-01-01 10:00:00.300] nvmf: TCP transport created
[2024-01-01 10:00:00.400] nvmf: Subsystem nqn.2024-01.io.spdk:malloc0 created
[2024-01-01 10:00:00.500] nvmf: Listener 127.0.0.1:4420 added
```

Confirm the target is listening:

```bash
ss -tlnp | grep 4420
# LISTEN 0  128  127.0.0.1:4420  0.0.0.0:*  users:(("nvmf_tgt",...))
```

---

### Step 5 — Connect from the initiator side with nvme-cli

Load the NVMe-oF TCP kernel module on the initiator (same machine in this lab):

```bash
sudo modprobe nvme-tcp
lsmod | grep nvme_tcp
# nvme_tcp    77824  0
# nvme_fabrics 24576  1 nvme_tcp
```

Discover the target:

```bash
sudo nvme discover -t tcp -a 127.0.0.1 -s 4420
```

Expected output:

```
Discovery Log Number of Records 1, Generation counter 2
=====Discovery Log Entry 0======
trtype:  tcp
adrfam:  ipv4
subtype: nvme subsystem
treq:    not required
portid:  0
trsvcid: 4420
subnqn:  nqn.2024-01.io.spdk:malloc0
traddr:  127.0.0.1
sectype: none
```

Connect to the subsystem:

```bash
sudo nvme connect \
    -t tcp \
    -a 127.0.0.1 \
    -s 4420 \
    -n nqn.2024-01.io.spdk:malloc0
```

Expected output:

```
[  234.123456] nvme1: new nvme ctrl device found
```

Verify the device appeared:

```bash
sudo nvme list
```

Expected:

```
Node             SN                   Model                                    Namespace Usage        Format      FW Rev
---------------- -------------------- ---------------------------------------- --------- ------------ ----------- --------
/dev/nvme1n1     SPDKLAB01            SPDK bdev Controller                     1         67.11  MB /  67.11  MB    23.09
```

Note the device path (`/dev/nvme1n1`) — you will use it in the fio commands below.

---

### Step 6 — Run fio against the connected NVMe-oF device

Run a 4K random read benchmark at queue depths 1, 4, 16, and 64. Using `io_uring` engine for lowest latency:

```bash
# QD=1
sudo fio --filename=/dev/nvme1n1 --rw=randread --bs=4k \
    --numjobs=1 --iodepth=1 --runtime=30 \
    --ioengine=io_uring --direct=1 \
    --group_reporting --name=nvmeof_qd1

# QD=4
sudo fio --filename=/dev/nvme1n1 --rw=randread --bs=4k \
    --numjobs=1 --iodepth=4 --runtime=30 \
    --ioengine=io_uring --direct=1 \
    --group_reporting --name=nvmeof_qd4

# QD=16
sudo fio --filename=/dev/nvme1n1 --rw=randread --bs=4k \
    --numjobs=1 --iodepth=16 --runtime=30 \
    --ioengine=io_uring --direct=1 \
    --group_reporting --name=nvmeof_qd16

# QD=64
sudo fio --filename=/dev/nvme1n1 --rw=randread --bs=4k \
    --numjobs=1 --iodepth=64 --runtime=30 \
    --ioengine=io_uring --direct=1 \
    --group_reporting --name=nvmeof_qd64
```

Example fio output for a single run (QD=64):

```
nvmeof_qd64: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=io_uring, iodepth=64
fio-3.36
Starting 1 process
Jobs: 1 (f=1): [r(1)][100.0%][r=1847MiB/s][r=472k IOPS][eta 00m:00s]
nvmeof_qd64: (groupid=0, jobs=1): err= 0: pid=12345: Tue Apr 22 10:05:00 2026
  read: IOPS=471k, BW=1847MiB/s (1937MB/s)(54.1GiB/30001msec)
    slat (nsec): min=1234, max=45678, avg=2100.34, stdev=512.00
    clat (usec): min=89, max=1234, avg=133.45, stdev=18.20
     lat (usec): min=91, max=1237, avg=135.55, stdev=18.30
    clat percentiles (usec):
     |  1.00th=[  110],  5.00th=[  116], 10.00th=[  119], 20.00th=[  123],
     | 50.00th=[  131], 75.00th=[  141], 90.00th=[  155], 95.00th=[  165],
     | 99.00th=[  192], 99.50th=[  206], 99.90th=[  249], 99.95th=[  277],
     | 99.99th=[  416]
  cpu          : usr=8.23%, sys=14.57%, ctx=472001, majf=0, minf=77
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=99.9%
  bw (  MiB/s): min= 1802, max= 1889, per=100.00%, avg=1847.12, stdev=19.45
  iops        : min=461366, max=483584, avg=472862.00, stdev=4980.00
```

---

### Step 7 — Compare QD=1, 4, 16, 64 IOPS — summary table

Results from a bdev_malloc target on a typical server (Xeon, DDR4-3200, loopback TCP):

| Queue Depth | IOPS      | Bandwidth   | Avg latency (µs) | p99 latency (µs) |
|-------------|-----------|-------------|------------------|------------------|
| QD=1        | ~28,000   | ~110 MB/s   | ~35              | ~58              |
| QD=4        | ~112,000  | ~437 MB/s   | ~36              | ~62              |
| QD=16       | ~310,000  | ~1,211 MB/s | ~51              | ~88              |
| QD=64       | ~471,000  | ~1,847 MB/s | ~133             | ~192             |

Key observations:
- IOPS scales near-linearly from QD=1 to QD=16 as the reactor's poll loop saturates multiple outstanding commands.
- Latency at QD=1 is bounded by the TCP loopback RTT (~30–40 µs). With RDMA, this drops to sub-10 µs.
- QD=64 shows diminishing returns — the malloc bdev's memory bandwidth (~25 GB/s DDR4) is the new bottleneck.
- For comparison: the kernel NVMe driver on the same malloc bdev via a kernel loopback target (`nvmet`) achieves roughly 60–70% of these IOPS due to `blk_mq` overhead.

---

### Step 8 — Cleanup and verification

Disconnect the initiator:

```bash
sudo nvme disconnect -n nqn.2024-01.io.spdk:malloc0
```

Expected:

```
NQN:nqn.2024-01.io.spdk:malloc0 disconnected 1 controller(s)
```

Stop the nvmf_tgt process:

```bash
sudo kill $NVMF_PID
# Or if you lost the PID:
sudo pkill nvmf_tgt
```

Release hugepages allocated by SPDK:

```bash
sudo scripts/setup.sh reset
```

Expected:

```
0000:01:00.0 (class=0x010802 nvme): vfio-pci -> nvme
Freeing hugepages
```

Verify the NVMe drive is back under the kernel driver:

```bash
sudo nvme list
# /dev/nvme0n1 should reappear with its original identity
lsblk | grep nvme
```

Final sanity check — confirm no SPDK processes are still running:

```bash
pgrep -l nvmf_tgt spdk_tgt
# Should return nothing (empty output)
```

---

## Summary

- SPDK's user-space NVMe driver eliminates kernel I/O overhead, achieving near-drive-peak IOPS and bandwidth with lower CPU utilization than the kernel driver.
- NVMe-oF extends NVMe semantics over RDMA (sub-10 µs) or TCP (practical for checkpoint traffic on management Ethernet) without the overhead of a traditional storage protocol.
- SPDK bdev RAID0 across 4+ drives provides the sequential bandwidth needed for fast LLM checkpoint writing.
- Separating checkpoint traffic (NVMe-oF TCP on management network) from gradient traffic (RDMA on training fabric) is a key architectural pattern for large clusters.

---

## References

- SPDK documentation: spdk.io/doc
- NVMe-oF specification: nvmexpress.org/specifications
- Yang et al., *SPDK: A Development Kit for High-Performance Storage*, IEEE CloudCom 2017
- io_uring documentation: kernel.dk/io_uring.pdf


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).