# Chapter 2 — RDMA & the RoCEv2 Stack

**Part I: Foundations** | ~25 pages

---

## Introduction

Remote Direct Memory Access (RDMA) is the single most impactful networking technology in the modern AI compute stack. By allowing a NIC to transfer data directly into or out of remote application memory — bypassing the CPU, the kernel, and the socket layer entirely — RDMA makes it possible to sustain 400 Gbps gradient exchange between GPUs at latencies measured in single-digit microseconds. Without RDMA, training large language models at scale would require orders of magnitude more CPU resources and would be bounded by software overhead rather than physics.

This chapter builds the RDMA programming model from first principles, starting with the verbs API that underpins every RDMA transport — InfiniBand, RoCEv1, and the now-dominant RoCEv2. The verbs abstraction (Protection Domains, Memory Regions, Queue Pairs, Completion Queues, and Work Requests) is the universal language of RDMA programming; understanding it is prerequisite to working with any RDMA-capable library, from `libibverbs` directly to UCX (Chapter 4) to NCCL (Chapter 19).

We examine the protocol landscape in depth: why InfiniBand dominated early HPC clusters, why RoCEv1 failed to achieve broad adoption, and why RoCEv2 — RDMA semantics over standard UDP/IP/Ethernet — won the AI cluster market. The critical trade-off is that Ethernet's lossy nature requires active congestion management: DCQCN (Data Center Quantized Congestion Notification), the ECN-based rate-control algorithm, is what prevents packet drops from triggering catastrophic RDMA retransmission cascades across an entire training job.

The chapter concludes with GPUDirect RDMA, the NVIDIA technology that removes the final software bottleneck: the copy between CPU DRAM and GPU HBM. With GPUDirect, gradient tensors flow directly from one GPU's HBM through the PCIe bus into the NIC and across the fabric into a remote GPU's HBM, with zero CPU involvement on the data path.

The lab walkthrough exercises the complete benchmarking pipeline using Soft-RoCE (`rdma_rxe`) — a kernel software implementation of RoCEv2 over standard Ethernet — on a pair of virtual interfaces, so no hardware RDMA NIC is required. The techniques (perftest bandwidth sweeps, latency histograms, multi-QP scaling) are identical to those used for production cluster sign-off. Chapter 3 (PTP) builds on the NIC hardware clock concepts introduced here; Chapter 4 (UCX/LibFabric) sits directly above the verbs layer covered in this chapter.

---

---

## Installation

This section installs rdma-core, which provides libibverbs, librdmacm, and the rdma-utils command-line tools needed to enumerate devices and manage RDMA interfaces from user space. The perftest package supplies ib_write_bw and ib_read_lat, the standard microbenchmarks used throughout this chapter for bandwidth sweeps and latency histograms. On systems with a hardware Mellanox or Broadcom RDMA NIC, the MLNX_OFED or in-kernel RDMA drivers are required to expose the device to the verbs layer; on any standard Ethernet interface, the soft-RoCE module rdma_rxe can be loaded instead to emulate a full RoCEv2 NIC in software, which is what the lab walkthrough uses.

### System Packages (Ubuntu 24.04)

```bash
sudo apt update
sudo apt install -y \
    rdma-core \
    libibverbs-dev \
    librdmacm-dev \
    infiniband-diags \
    perftest \
    ibverbs-utils \
    iproute2 \
    cmake \
    build-essential \
    pkg-config
```

Verify the RDMA userspace stack is installed:

```bash
# Check libibverbs version
ibv_devices
# If no hardware RDMA NIC is present this will print an empty list — that is expected;
# Soft-RoCE (rxe) will be loaded in the lab walkthrough below.

dpkg -l | grep -E "rdma-core|libibverbs|perftest"
# Expected lines:
# ii  libibverbs-dev  ...  Development files for the libibverbs library
# ii  perftest        ...  Infiniband verbs performance tests
# ii  rdma-core       ...  RDMA core userspace libraries and daemons
```

### Soft-RoCE kernel module

Soft-RoCE (`rdma_rxe`) ships with the upstream Linux kernel since 5.6 and is included in Ubuntu 24.04's `linux-modules-extra` package:

```bash
sudo apt install -y linux-modules-extra-$(uname -r)

# Load the module
sudo modprobe rdma_rxe

# Verify it loaded
lsmod | grep rdma_rxe
# Expected:
# rdma_rxe              196608  0
# ib_core               393216  3 rdma_rxe,...
```

### CMake build setup for the C libibverbs example

Save the following as `CMakeLists.txt` in your project directory (e.g., `~/rdma-example/`):

```cmake
cmake_minimum_required(VERSION 3.20)
project(rdma_example C)
find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBIBVERBS REQUIRED libibverbs)
add_executable(rdma_write rdma_write.c)
target_include_directories(rdma_write PRIVATE ${LIBIBVERBS_INCLUDE_DIRS})
target_link_libraries(rdma_write ${LIBIBVERBS_LIBRARIES})
```

Build the example:

```bash
mkdir -p ~/rdma-example/build
cd ~/rdma-example/build
cmake ..
make -j$(nproc)
# Expected:
# -- The C compiler identification is GNU 13.x.x
# -- Found PkgConfig: /usr/bin/pkg-config
# -- Checking for module 'libibverbs'
# --   Found libibverbs, version 39.x
# -- Configuring done
# -- Build files have been written to: /root/rdma-example/build
# [ 50%] Building C object CMakeFiles/rdma_write.dir/rdma_write.c.o
# [100%] Linking C executable rdma_write
# [100%] Built target rdma_write
```

### Python pyverbs setup (optional, for scripted benchmarks)

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# Create a virtual environment and install pyverbs
mkdir -p ~/rdma-pyverbs && cd ~/rdma-pyverbs
uv init .
uv add pyverbs

# Verify pyverbs can enumerate devices (will be empty without rxe loaded yet)
uv run python -c "
import pyverbs.device as d
devs = d.get_device_list()
print(f'Found {len(devs)} RDMA device(s):')
for dev in devs:
    print(f'  {dev.name.decode()}')
"
```

---

## 2.1 Why RDMA?

The fundamental constraint of classical networking: every packet received by a host must traverse the kernel's network stack, be copied into a socket buffer, and be read by an application via a system call. At 400 Gbps, this overhead consumes entire CPU cores that would otherwise feed GPUs. RDMA eliminates it.

**Remote Direct Memory Access** allows a NIC to read from or write to a remote host's memory without involving that host's CPU or OS. The sender describes a memory region; the NIC DMA-transfers it directly to a registered memory region on the receiver. No system calls. No copies. No kernel involvement on the data path.

For AI training, RDMA means that gradient tensors can be transferred between GPU memory regions (via GPUDirect RDMA, which bypasses the CPU entirely) at line rate, with latencies measured in single-digit microseconds.

---

## 2.2 The Verbs Programming Model

RDMA programming uses the **verbs API**, standardized originally by the InfiniBand Trade Association and implemented in userspace by `libibverbs` (part of `rdma-core`).

### 2.2.1 Key Abstractions

| Object | Description |
|---|---|
| **Protection Domain (PD)** | Namespace for access control; memory and QPs are associated with a PD |
| **Memory Region (MR)** | A pinned, registered region of virtual memory the NIC can DMA to/from; carries an `lkey` (local) and `rkey` (remote) |
| **Queue Pair (QP)** | A pair of queues — Send Queue (SQ) and Receive Queue (RQ) — defining a logical connection |
| **Completion Queue (CQ)** | Where the NIC posts completion events (successes/errors) after processing Work Requests |
| **Work Request (WR)** | A descriptor posted to a SQ or RQ describing an operation (SEND, RECV, RDMA_WRITE, RDMA_READ) |
| **Scatter-Gather Entry (SGE)** | A (addr, length, lkey) tuple; a WR can have multiple SGEs for vectorized I/O |

### 2.2.2 Connection Types

- **RC (Reliable Connected):** Point-to-point, in-order, reliable delivery. Used by most NCCL operations. Requires N² QP connections for N ranks — memory-intensive at scale.
- **UD (Unreliable Datagram):** Connectionless; one QP can communicate with any peer. Limited to MTU-sized messages. Used by collectives that tolerate retransmission at higher layers.
- **DC (Dynamically Connected, mlx5 only):** Hybrid — connection is established on first use, amortizing QP memory cost. Used by modern NCCL to avoid the N² QP problem.

### 2.2.3 Operation Types

```
RDMA_WRITE:   local → remote memory, no CPU involvement on receiver
RDMA_READ:    pull from remote memory into local buffer
SEND/RECV:    two-sided; receiver must have posted a RECV WR first
ATOMIC:       compare-and-swap, fetch-and-add on 64-bit remote words
```

---

## 2.3 libibverbs API Walkthrough

A minimal RDMA WRITE sender:

```c
// 1. Open device
struct ibv_context *ctx = ibv_open_device(dev_list[0]);

// 2. Allocate protection domain
struct ibv_pd *pd = ibv_alloc_pd(ctx);

// 3. Register memory
char buf[4096];
struct ibv_mr *mr = ibv_reg_mr(pd, buf, sizeof(buf),
    IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE);

// 4. Create completion queue
struct ibv_cq *cq = ibv_create_cq(ctx, 16, NULL, NULL, 0);

// 5. Create queue pair
struct ibv_qp_init_attr qp_attr = {
    .send_cq = cq, .recv_cq = cq,
    .qp_type = IBV_QPT_RC,
    .cap = { .max_send_wr = 8, .max_recv_wr = 8,
             .max_send_sge = 1, .max_recv_sge = 1 },
};
struct ibv_qp *qp = ibv_create_qp(pd, &qp_attr);

// 6. Transition QP to RTS (Init → RTR → RTS), exchanging GID/QPN out-of-band
// ... (ibv_modify_qp calls omitted for brevity)

// 7. Post RDMA WRITE
struct ibv_sge sge = { .addr = (uint64_t)buf, .length = 64, .lkey = mr->lkey };
struct ibv_send_wr wr = {
    .opcode = IBV_WR_RDMA_WRITE,
    .sg_list = &sge, .num_sge = 1,
    .wr.rdma = { .remote_addr = remote_addr, .rkey = remote_rkey },
    .send_flags = IBV_SEND_SIGNALED,
};
struct ibv_send_wr *bad_wr;
ibv_post_send(qp, &wr, &bad_wr);

// 8. Poll completion
struct ibv_wc wc;
while (ibv_poll_cq(cq, 1, &wc) == 0) {}
assert(wc.status == IBV_WC_SUCCESS);
```

---

## 2.4 InfiniBand vs RoCEv1 vs RoCEv2

| Property | InfiniBand | RoCEv1 | RoCEv2 |
|---|---|---|---|
| Transport | IB transport | IB transport over Ethernet L2 | IB transport over UDP/IP |
| Routing | Subnet Manager assigns LIDs | L2 only (no IP routing) | Routable over IP fabric |
| Congestion control | IB credit-based FC | — | DCQCN (ECN-based) |
| Infrastructure | Dedicated IB switches | Standard Ethernet | Standard Ethernet |
| Adoption | HPC, some AI clusters | Legacy | Dominant in new AI clusters |

RoCEv2 won because it runs over commodity Ethernet switches (avoiding IB switch cost) while remaining routable across subnets. The trade-off is that Ethernet's lossy nature requires explicit congestion control — see Section 2.5.

---

## 2.5 DCQCN: Congestion Control for RoCEv2

RoCEv2 over Ethernet is intrinsically lossy. Dropped packets break RDMA reliability and trigger expensive retransmissions. **DCQCN** (Data Center Quantized Congestion Notification) prevents loss by coordinating three components:

### Switch: ECN Marking
Switches are configured with RED-based ECN: when a queue depth exceeds a threshold, packets are probabilistically marked with the CE (Congestion Experienced) bit in the IP header. No drops yet — just a signal.

```
# SONiC ECN configuration via CONFIG_DB
{
  "WRED_PROFILE": {
    "AI_WRED": {
      "ecn": "ecn_all",
      "green_min_threshold": "1MB",
      "green_max_threshold": "4MB",
      "green_drop_probability": "0"
    }
  }
}
```

### Receiver NIC: CNP Generation
When the receiver NIC sees a CE-marked packet, it generates a **Congestion Notification Packet (CNP)** back to the sender. CNPs are sent at most once per RTT per QP.

### Sender NIC: Rate Control
On receiving a CNP, the sender NIC applies DCQCN's rate-control algorithm:
1. **Rate reduction**: multiply current rate by (1 - α/2), where α tracks congestion severity
2. **Rate recovery**: increase rate additively (AI) until the bandwidth-delay product is reached, then switch to hyperincrement (HA) for fast recovery
3. **Timer-based actions**: if no CNP arrives for a period, the NIC assumes congestion cleared and begins recovery

**Key tuning parameters** (set on the NIC via `mlnx_qos` — Mellanox's QoS configuration tool for ConnectX NICs — or DOCA, NVIDIA's data-center infrastructure SDK for BlueField DPUs):
- `cnp_dscp`: DSCP value for CNP packets (must be mapped to a high-priority QoS queue)
- `dcqcn_alpha_g`: EWMA decay for the α parameter
- `dcqcn_rp_initial_alpha`: starting α value

---

## 2.6 The rdma-core Userspace Stack

`rdma-core` is the Linux upstream repository for all RDMA userspace components:

```
rdma-core/
├── libibverbs/      # Core verbs API and provider loading
├── librdmacm/       # Connection management (RDMA CM)
├── ibacm/           # Address and route resolution daemon (resolves GIDs and LIDs)
├── rdma/            # iproute2 rdma subcommand
├── providers/
│   ├── mlx5/        # Mellanox/NVIDIA ConnectX direct verbs
│   ├── efa/         # AWS Elastic Fabric Adapter
│   ├── rxe/         # Soft-RoCE (RDMA over standard Ethernet NIC)
│   └── ...
└── pyverbs/         # Python bindings to libibverbs (Cython-based, used for scripting and testing)
```

### rdma CLI

```bash
# List RDMA devices and ports
rdma link

# Show device capabilities
rdma dev show mlx5_0

# Show QP statistics
rdma stat show

# Monitor RDMA counters
rdma stat show -j | jq '.[] | {port, rx_bytes, tx_bytes}'
```

### Soft-RoCE (rxe)

For development without a RoCE NIC:

```bash
modprobe rdma_rxe
rdma link add rxe0 type rxe netdev eth0
ibv_devinfo  # confirms rxe0 appears as an RDMA device
```

---

## 2.7 Benchmarking with perftest

`perftest` is the standard RDMA benchmarking suite. Every production cluster commissioning workflow runs these before declaring the fabric ready.

```bash
# On receiver (server):
ib_write_bw -d mlx5_0 -x 3

# On sender (client):
ib_write_bw -d mlx5_0 -x 3 192.168.1.2

# Expected output on 400GbE:
# #bytes  #iterations  BW peak[Gb/sec]  BW average[Gb/sec]  MsgRate[Mpps]
# 65536   5000         389.5            388.9               0.74

# Latency test:
ib_read_lat -d mlx5_0 192.168.1.2
# Expected: ~1.5 µs one-way on back-to-back 400GbE

# SEND/RECV bandwidth:
ib_send_bw -d mlx5_0 192.168.1.2 --report_gbits

# QP-scaling test (N parallel QPs):
ib_write_bw -d mlx5_0 --num_of_qps 8 192.168.1.2
```

Key metrics to capture for fabric sign-off:
- Peak bandwidth ≥ 90% of line rate at 64KB+ message size
- Latency ≤ 2 µs at 4B message size (one-way, back-to-back)
- No degradation under multi-QP load (N=8, N=64)

---

## 2.8 GPUDirect RDMA

GPUDirect RDMA allows the NIC to DMA directly to/from GPU HBM, bypassing the CPU and system memory entirely.

```
Without GPUDirect:  GPU HBM → PCIe → CPU DRAM → PCIe → NIC → network
With GPUDirect:     GPU HBM → PCIe → NIC → network
```

Requirements:
- `nvidia-peermem` kernel module (or `nv_peer_mem` on older systems) — a kernel module that maps GPU BAR memory into the RDMA subsystem so the NIC can DMA to it directly
- Memory region registered with `ibv_reg_mr` on a pointer returned by `cudaMalloc` (or `cuMemAlloc`)
- GPU and NIC on the same PCIe domain for optimal performance

```c
// Register GPU memory for RDMA
void *gpu_ptr;
cudaMalloc(&gpu_ptr, 1 << 20);
struct ibv_mr *gpu_mr = ibv_reg_mr(pd, gpu_ptr, 1 << 20,
    IBV_ACCESS_LOCAL_WRITE | IBV_ACCESS_REMOTE_WRITE);
// Now use gpu_mr->lkey in SGEs — the NIC will DMA directly to GPU HBM
```

---

## Lab Walkthrough 2 — RoCEv2 Benchmarking Pipeline

This walkthrough configures Soft-RoCE on two loopback-connected virtual interfaces on a single Ubuntu 24.04 host, then runs the full `perftest` benchmarking suite to measure RDMA WRITE bandwidth and READ latency. No hardware RDMA NIC is required.

**Prerequisites:** All packages installed per the Installation section above.

### Step 1: Verify the rdma_rxe kernel module is available

```bash
modinfo rdma_rxe | head -5
```

Expected output:

```
filename:       /lib/modules/6.8.0-xx-generic/kernel/drivers/infiniband/sw/rxe/rdma_rxe.ko
description:    Soft RDMA transport
author:         ...
license:        GPL v2
```

If the module is not found, install the extra modules package:

```bash
sudo apt install -y linux-modules-extra-$(uname -r)
sudo modprobe rdma_rxe
```

### Step 2: Create a veth pair to simulate two endpoints on one host

A virtual Ethernet pair (`veth0` / `veth1`) acts as a back-to-back cable between two namespaces:

```bash
# Create veth pair
sudo ip link add veth0 type veth peer name veth1

# Bring both ends up
sudo ip link set veth0 up
sudo ip link set veth1 up

# Assign IP addresses
sudo ip addr add 10.10.0.1/30 dev veth0
sudo ip addr add 10.10.0.2/30 dev veth1

# Verify
ip addr show veth0
ip addr show veth1
```

Expected output for `ip addr show veth0`:

```
5: veth0@veth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
    inet 10.10.0.1/30 scope global veth0
       valid_lft forever preferred_lft forever
```

Verify the two ends can reach each other:

```bash
ping -c 2 10.10.0.2
# Expected: 0% packet loss, < 1 ms RTT
```

### Step 3: Load the rdma_rxe module and create two rxe devices

```bash
sudo modprobe rdma_rxe

# Attach rxe device to each veth interface
sudo rdma link add rxe0 type rxe netdev veth0
sudo rdma link add rxe1 type rxe netdev veth1
```

Verify both devices are created:

```bash
rdma link show
```

Expected output:

```
link rxe0/1 state ACTIVE physical_state POLLING netdev veth0
link rxe1/1 state ACTIVE physical_state POLLING netdev veth1
```

Both lines must show `state ACTIVE`. If a link shows `INIT` or `DOWN`, check that the underlying veth interface is UP:

```bash
ip link show veth0 | grep -o "state [A-Z]*"
# Must be: state UP
```

### Step 4: Enumerate RDMA devices and verify capabilities

```bash
ibv_devinfo
```

Expected output (two devices):

```
hca_id: rxe0
        transport:                      InfiniBand (0)
        fw_ver:                         0.0.0
        node_guid:                      aabb:ccff:fedd:eeff
        sys_image_guid:                 aabb:ccff:fedd:eeff
        vendor_id:                      0x0
        vendor_part_id:                 0
        hw_ver:                         0x0
        phys_port_cnt:                  1
                port:   1
                        state:          PORT_ACTIVE (4)
                        max_mtu:        4096 (5)
                        active_mtu:     1024 (3)
                        sm_lid:         0
                        port_lid:       0
                        port_lmc:       0x00
                        link_layer:     Ethernet

hca_id: rxe1
        transport:                      InfiniBand (0)
        ...
                        state:          PORT_ACTIVE (4)
```

The critical check is `state: PORT_ACTIVE (4)` on port 1 of both `rxe0` and `rxe1`. Any other state (DOWN, INIT, ARMED) means the underlying interface is not ready.

Also verify with the `rdma` command:

```bash
rdma dev show
```

Expected:

```
0: rxe0: node-type ca fw 0.0.0 node-guid aabb:ccff:fedd:eeff sys-image-guid aabb:ccff:fedd:eeff
1: rxe1: node-type ca fw 0.0.0 node-guid ...
```

### Step 5: Run ib_write_bw — RDMA WRITE bandwidth test

The `perftest` tools use a client/server model. Because both rxe devices are on the same host, use two terminal sessions (or background the server):

**Terminal A — server (receiver):**

```bash
ib_write_bw -d rxe0 -x 3 --port 18515
```

Server will print and wait:

```
************************************
* Waiting for client to connect... *
************************************
```

**Terminal B — client (sender):**

```bash
ib_write_bw -d rxe1 -x 3 --port 18515 10.10.0.1
```

The `-x 3` flag selects GID index 3 (IPv4 RoCEv2). A GID (Global Identifier) is the 128-bit RoCEv2 address that maps to an IPv4 or IPv6 address; a LID (Local Identifier) is the 16-bit address used in InfiniBand fabric routing. The client connects to the server's IP (`10.10.0.1`).

Expected combined output after the test completes:

```
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : rxe1
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 TX depth        : 128
 CQ Moderation   : 100
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x0000 QPN 0x0011 PSN 0xaabbcc
 GID: 00:00:00:00:00:00:00:00:00:00:ff:ff:0a:0a:00:02
 remote address: LID 0x0000 QPN 0x0012 PSN 0xddeeff
 GID: 00:00:00:00:00:00:00:00:00:00:ff:ff:0a:0a:00:01
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000           9.58               9.51                 0.018
---------------------------------------------------------------------------------------
```

**Expected bandwidth ranges for Soft-RoCE over veth (no hardware):**
- 64B messages: 0.1–0.5 Gb/sec (small-message overhead dominates)
- 4KB messages: 2–5 Gb/sec
- 64KB messages: 8–12 Gb/sec (limited by software emulation and loopback bandwidth)
- 1MB messages: 10–15 Gb/sec

Hardware RoCE NICs (ConnectX-6/7) will show 380–395 Gb/sec at 64KB on 400GbE links.

### Step 6: Sweep message sizes for a bandwidth curve

Run the bandwidth test across multiple message sizes to plot the throughput curve:

```bash
for msg_size in 64 512 4096 65536 1048576; do
    # Start server in background
    ib_write_bw -d rxe0 -x 3 --port 18515 -s $msg_size > /dev/null 2>&1 &
    SERVER_PID=$!
    sleep 0.5

    # Run client and capture result
    RESULT=$(ib_write_bw -d rxe1 -x 3 --port 18515 -s $msg_size 10.10.0.1 2>/dev/null | \
             grep -E "^\s+$msg_size")
    echo "msg_size=${msg_size}B: ${RESULT}"

    wait $SERVER_PID 2>/dev/null
done
```

Expected output:

```
msg_size=64B:        64       5000           0.21               0.19             0.372
msg_size=512B:      512       5000           1.43               1.41             0.344
msg_size=4096B:    4096       5000           4.87               4.82             0.147
msg_size=65536B:  65536       5000           9.58               9.51             0.018
msg_size=1048576B: 1048576    1000          12.10              12.03             0.001
```

The bandwidth increases with message size due to amortization of per-operation overhead — this is the characteristic RDMA bandwidth curve. On hardware NICs, the plateau begins at 64KB and stays flat through 1MB+.

### Step 7: Run ib_write_lat — RDMA WRITE latency test

**Terminal A — server:**

```bash
ib_write_lat -d rxe0 -x 3 --port 18516
```

**Terminal B — client:**

```bash
ib_write_lat -d rxe1 -x 3 --port 18516 10.10.0.1
```

Expected output:

```
---------------------------------------------------------------------------------------
                    RDMA_Write Latency Test
...
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec]
      2       1000          15.23         187.34           16.45          17.12           3.82              24.18                   98.43
---------------------------------------------------------------------------------------
```

**Expected latency ranges:**
- Soft-RoCE over veth (software loopback): 15–50 µs typical, p99 20–100 µs
- Hardware RoCE (ConnectX-6, back-to-back): 1.2–2.0 µs typical, p99 < 3 µs
- Hardware IB HDR (back-to-back): 0.6–1.2 µs typical

The significantly higher latency for Soft-RoCE reflects kernel scheduling overhead — the rxe driver processes packets in softirq context rather than on dedicated NIC hardware.

### Step 8: Run ib_read_lat — RDMA READ latency test

RDMA READ pulls data from the remote side without the remote CPU posting any work. This is particularly important for parameter server patterns:

**Terminal A — server:**

```bash
ib_read_lat -d rxe0 -x 3 --port 18517
```

**Terminal B — client:**

```bash
ib_read_lat -d rxe1 -x 3 --port 18517 10.10.0.1
```

Expected output:

```
---------------------------------------------------------------------------------------
                    RDMA_Read Latency Test
...
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]
      2       1000          18.47         201.22           19.83          20.51           5.14
---------------------------------------------------------------------------------------
```

RDMA READ adds one additional network round-trip vs WRITE (the NIC must fetch data before returning), so expect ~2x the WRITE latency figure. On hardware, RDMA READ over a 1-hop fabric is typically 2.5–4 µs vs 1.5–2 µs for WRITE.

### Step 9: Run ib_write_bw with multiple QPs — QP scaling test

NCCL uses many QPs simultaneously. Verify that bandwidth scales (or at least does not collapse) under multi-QP load:

**Terminal A — server:**

```bash
ib_write_bw -d rxe0 -x 3 --port 18518 --num_of_qps 8
```

**Terminal B — client:**

```bash
ib_write_bw -d rxe1 -x 3 --port 18518 --num_of_qps 8 10.10.0.1
```

Expected: aggregate bandwidth roughly equal to single-QP result for Soft-RoCE (software bottleneck dominates). On hardware ConnectX NICs, expect near-linear scaling up to 8 QPs before the NIC's line rate is saturated.

### Step 10: Verify RDMA traffic is flowing via counters

While the bandwidth test runs, in a separate terminal, inspect the rxe port counters:

```bash
# Check hardware counters via sysfs (rxe exposes these)
cat /sys/class/infiniband/rxe0/ports/1/counters/port_xmit_data
cat /sys/class/infiniband/rxe0/ports/1/counters/port_rcv_data

# Or via the rdma CLI
rdma stat show rxe0/1
```

Expected (counters rising during the benchmark):

```
port_xmit_data: 8372940800
port_rcv_data:  8372940800
port_xmit_packets: 127808
port_rcv_packets:  127808
```

The `port_xmit_data` and `port_rcv_data` values are in units of 4 bytes (IB convention). Divide by 4 and multiply by 8 for bits. Rising values confirm RDMA traffic is actually traversing the rxe device.

Also confirm zero error counters:

```bash
cat /sys/class/infiniband/rxe0/ports/1/counters/port_rcv_errors
cat /sys/class/infiniband/rxe0/ports/1/counters/port_xmit_discards
# Expected: 0 (any non-zero value indicates a problem)
```

### Step 11: Simulate ECN-like delay with netem and observe latency impact

`netem` (Network Emulator) is a Linux traffic-control (`tc`) discipline that injects configurable delay, jitter, loss, and reordering into a network interface, enabling reproducible testing of protocol behavior under impaired conditions. Add artificial delay to simulate a congested fabric path:

```bash
# Add 1 ms delay on veth1 (simulating switch queuing)
sudo tc qdisc add dev veth1 root netem delay 1ms

# Re-run latency test
# Terminal A:
ib_write_lat -d rxe0 -x 3 --port 18519
# Terminal B:
ib_write_lat -d rxe1 -x 3 --port 18519 10.10.0.1
```

Expected: latency increases by ~2 ms RTT (1 ms each way), moving from ~17 µs to ~2017 µs typical. This demonstrates how switch queuing depth (which DCQCN prevents from growing large) directly controls RDMA latency.

Remove the netem rule when done:

```bash
sudo tc qdisc del dev veth1 root
```

### Step 12: Clean up rxe devices and veth pair

```bash
# Remove rxe devices
sudo rdma link delete rxe0
sudo rdma link delete rxe1

# Remove the veth pair
sudo ip link del veth0
# Deleting veth0 automatically removes veth1 (they are paired)

# Verify
rdma link show
# Expected: empty output
ip link show veth0
# Expected: error (device does not exist)
```

Optionally unload the kernel module if no longer needed:

```bash
sudo modprobe -r rdma_rxe
lsmod | grep rdma_rxe
# Expected: no output (module not loaded)
```

---

## Summary

- RDMA eliminates CPU and kernel involvement from the data path; at 400 Gbps this is the difference between GPU starvation and line-rate training throughput.
- The verbs model (PD → MR → QP → CQ → WR) is the universal abstraction across all RDMA transports.
- RoCEv2 is now the standard for AI clusters; its lossy-by-default Ethernet substrate demands DCQCN for loss-free operation.
- `rdma-core` provides the complete userspace stack; `perftest` is the standard commissioning benchmark suite.
- GPUDirect RDMA removes the final CPU-mediated copy between GPU memory and the NIC.

---

## References

- Kalia et al., *Design Guidelines for High Performance RDMA Systems*, USENIX ATC 2016
- Zhu et al., *Congestion Control for Large-Scale RDMA Deployments (DCQCN)*, SIGCOMM 2015
- `rdma-core` documentation: github.com/linux-rdma/rdma-core
- NVIDIA MLNX_OFED documentation: docs.nvidia.com/networking/


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).