# Chapter 5 — DPDK: Kernel-Bypass Packet Processing

**Part II: Kernel-Bypass & Programmable I/O** | ~25 pages

---

## Introduction

The **Linux** kernel network stack is a marvel of software engineering — correct, secure, and general-purpose. It is also fundamentally unsuited to the demands of 400 Gbps line-rate packet processing. Each packet received through the kernel path requires a hardware interrupt, a `sk_buff` allocation, protocol demultiplexing, socket buffer copies, and a system call return. At 400 Gbps with 64-byte minimum frames, there are roughly 595 million packets per second, leaving less than 2 nanoseconds of CPU budget per packet — a single cache miss exceeds that budget. **DPDK** (Data Plane Development Kit) exists to solve this problem by eliminating the kernel from the data path entirely.

**DPDK** is an open-source framework, primarily maintained under the **Linux Foundation**, that provides user-space poll-mode drivers (**PMD**s) for a broad range of **NIC**s. Rather than waiting for interrupt-driven notifications, a **DPDK** application dedicates one or more CPU cores to busy-polling **NIC** receive rings directly from user space. Packets arrive into **DPDK**'s `rte_mbuf` structures from **DMA** memory that the **NIC** and the CPU share via hugepages, with zero kernel involvement, zero copies, and zero system calls on the data path.

This chapter builds the **DPDK** programming model from the environment abstraction layer (**EAL**) through hugepage configuration, **NUMA**-aware memory allocation, multi-queue **RSS**, hardware flow steering, and checksum offloads. The architecture is layered: **EAL** handles platform initialization and device binding; **PMD**s implement the **NIC**-specific **DMA** ring protocol; `rte_mempool` and `rte_mbuf` provide the packet buffer management layer; and application libraries (`rte_hash`, `rte_lpm`, `rte_acl`) implement forwarding-plane data structures.

**DPDK**'s primary role in AI cluster networking is in the **DPU**/**SmartNIC** programmable offload pipeline (Chapter 10) and in **OVS-DPDK**, the kernel-bypass virtual switch used to forward tenant traffic on multi-tenant AI cloud infrastructure. Understanding **DPDK**'s poll-model, memory architecture, and flow-steering primitives is prerequisite for Chapter 10 and provides the conceptual foundation for contrasting with **eBPF**/**XDP** (Chapter 7), which achieves kernel-bypass-level performance for some workloads while retaining kernel integration.

The lab walkthrough exercises **DPDK** end-to-end using the `net_tap` virtual device — no spare **NIC** is required — and measures achievable packet rates and CPU efficiency with and without checksum offload.

---

## Installation

This section installs the **dpdk**, **dpdk-dev**, and **dpdk-tools** packages, which provide the **DPDK** runtime libraries, development headers, and the `dpdk-devbind` utility used to bind **NIC**s to the kernel-bypass driver. Hugepage configuration is performed at this stage because **DPDK**'s zero-copy **DMA** memory model requires large physically contiguous pages that are mapped directly into user space and shared with the **NIC**. Either the **vfio-pci** or **uio_pci_generic** kernel module must be loaded and the target **NIC** bound to it, transferring device ownership from the **Linux** kernel to **DPDK**'s poll-mode driver so that the kernel's network stack is bypassed entirely on the data path.

### Ubuntu 24.04 — apt packages

```bash
sudo apt install -y dpdk dpdk-dev libdpdk-dev python3-pyelftools \
    meson ninja-build libnuma-dev pkg-config \
    linux-modules-extra-$(uname -r)
```

Enable the vfio-pci kernel module (needed to hand a NIC to DPDK):

```bash
sudo modprobe vfio-pci
# Make it persistent across reboots
echo "vfio-pci" | sudo tee /etc/modules-load.d/vfio-pci.conf
```

Verify DPDK is visible via pkg-config:

```bash
pkg-config --modversion libdpdk
# Expected output: 23.11.x  (or whichever version apt installed)
```

### Building DPDK from source (preferred for latest PMDs)

The apt package may lag behind upstream. Building from source gives access to all PMDs and the latest fixes:

```bash
# Fetch source
git clone https://dpdk.org/git/dpdk-stable --branch v23.11 --depth 1
cd dpdk-stable

# Configure with meson; install to /usr/local
meson setup build --prefix=/usr/local -Dexamples=l2fwd,l3fwd
cd build
ninja
sudo ninja install
sudo ldconfig
```

Confirm the install:

```bash
dpdk-devbind.py --version
# Expected: dpdk-devbind.py version 23.11.x
```

### CMake build system for C examples

When integrating DPDK into a CMake-based project, locate the library via pkg-config:

```cmake
cmake_minimum_required(VERSION 3.20)
project(dpdk_example C)

find_package(PkgConfig REQUIRED)
pkg_check_modules(DPDK REQUIRED libdpdk)

add_executable(l2fwd l2fwd.c)
target_compile_options(l2fwd PRIVATE ${DPDK_CFLAGS_OTHER})
target_include_directories(l2fwd PRIVATE ${DPDK_INCLUDE_DIRS})
target_link_libraries(l2fwd ${DPDK_LIBRARIES})
```

Build it:

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)
```

---

## 5.1 The Problem with the Kernel Network Stack

The Linux kernel network stack was designed for correctness and generality, not for sustained line-rate I/O. At 400 Gbps, a host receives 595 million packets per second (at 84-byte minimum frames). The kernel path for each packet involves:

1. Hardware interrupt → interrupt handler → NAPI poll (New API, the Linux kernel's interrupt-mitigation framework that coalesces packet delivery into polling bursts)
2. `sk_buff` (socket buffer) allocation from slab — the kernel's per-packet metadata structure, roughly 200 bytes of overhead per packet
3. Protocol processing (Ethernet → IP → TCP/UDP)
4. Socket buffer copy into user space
5. System call return

Each of these steps has a cost measured in nanoseconds. Aggregated across 595 Mpps, the CPU budget per packet is 1.7 ns — less than a single cache miss. The kernel stack consumes 3–8 CPU cores just to sustain this rate, cores that would otherwise serve GPU workloads.

**DPDK's answer:** poll the NIC from user space, batch packet processing, and never involve the kernel on the data path.

---

## 5.2 DPDK Architecture

```
┌─────────────────────────────────────────────────────┐
│ Application (L2 forwarder, vSwitch, packet generator)│
├─────────────────────────────────────────────────────┤
│ DPDK libraries: rte_ethdev, rte_mbuf, rte_mempool,  │
│                 rte_ring, rte_hash, rte_lpm ...      │
├─────────────────────────────────────────────────────┤
│ Poll-Mode Drivers (PMDs):                            │
│   mlx5_pmd   i40e    ixgbe    virtio    vhost        │
├─────────────────────────────────────────────────────┤
│ Environment Abstraction Layer (EAL)                  │
│   hugepages  CPU affinity  PCI device binding        │
├──────────────────────────┬──────────────────────────┤
│        UIO / VFIO        │       NIC hardware        │
└──────────────────────────┴──────────────────────────┘
```

### 5.2.1 Environment Abstraction Layer (EAL)

The EAL handles all platform-specific initialization:

```c
int main(int argc, char *argv[]) {
    // Initialize EAL — must be first call
    int ret = rte_eal_init(argc, argv);
    if (ret < 0)
        rte_exit(EXIT_FAILURE, "EAL init failed\n");
    argc -= ret;
    argv += ret;
    // ... application init follows
}
```

EAL command-line options:
```bash
./l2fwd \
  -l 0-3           \  # use logical cores 0,1,2,3
  -n 4             \  # 4 memory channels
  --proc-type auto \
  --huge-dir /dev/hugepages \
  --vdev net_pcap0,rx_pcap=in.pcap   # virtual device for testing
  -- \
  -p 0x3           # application args: port mask
```

### 5.2.2 Hugepages

The kernel TLB can only map 4KB pages efficiently. At high packet rates, TLB thrash from 4KB mempool entries wastes cycles on page-table walks. DPDK requires hugepages (2MB or 1GB):

```bash
# Reserve 2MB hugepages
echo 2048 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
mount -t hugetlbfs nodev /dev/hugepages

# Reserve 1GB hugepages (preferred for large mempools)
echo 4 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages
```

### 5.2.3 CPU Affinity and NUMA

DPDK pins each lcore (logical core) to a specific CPU core via `pthread_setaffinity_np`. For NUMA (Non-Uniform Memory Access) systems — servers with multiple CPU sockets, each with its own local DRAM — memory is allocated from the socket local to the running lcore, avoiding expensive cross-socket memory transactions:

```c
// Allocate mempool on NUMA socket of port 0
unsigned socket_id = rte_eth_dev_socket_id(0);
struct rte_mempool *mp = rte_pktmbuf_pool_create(
    "MBUF_POOL",
    NUM_MBUFS,          // total mbufs
    MBUF_CACHE_SIZE,    // per-lcore cache
    0,                  // private data size
    RTE_MBUF_DEFAULT_BUF_SIZE,
    socket_id           // NUMA socket
);
```

---

## 5.3 Poll-Mode Drivers (PMDs)

PMDs replace kernel interrupt-driven drivers with a busy-poll loop:

```c
// Transmit/receive loop — no interrupts, no system calls
while (1) {
    // Burst receive from port 0, queue 0
    uint16_t nb_rx = rte_eth_rx_burst(0, 0, pkts, BURST_SIZE);

    // Process packets
    for (int i = 0; i < nb_rx; i++) {
        process_packet(pkts[i]);
    }

    // Burst transmit
    uint16_t nb_tx = rte_eth_tx_burst(0, 0, pkts, nb_rx);

    // Free unsent mbufs
    for (int i = nb_tx; i < nb_rx; i++)
        rte_pktmbuf_free(pkts[i]);
}
```

`rte_eth_rx_burst` reads directly from the NIC's DMA ring in user space — no kernel context switch, no interrupt latency.

### Device Binding

DPDK requires unbinding a NIC from its kernel driver and binding it to `vfio-pci` (preferred — Virtual Function I/O, a kernel framework for safe user-space DMA using IOMMU isolation) or `uio_pci_generic` (a simpler but less secure UIO driver without IOMMU protection):

```bash
# Find PCI address
lspci | grep Ethernet

# Bind to vfio-pci
dpdk-devbind.py --bind=vfio-pci 0000:03:00.0

# Verify
dpdk-devbind.py --status
# Network devices using DPDK-compatible driver
# 0000:03:00.0 'ConnectX-6 Dx' drv=vfio-pci unused=mlx5_core
```

---

## 5.4 mbuf and Mempool Design

The `rte_mbuf` is DPDK's packet buffer, analogous to the kernel's `sk_buff` but designed for cache efficiency:

```c
struct rte_mbuf {
    void      *buf_addr;      // data buffer pointer
    uint16_t   data_off;      // offset to packet data within buffer
    uint16_t   nb_segs;       // number of chained segments
    uint16_t   pkt_len;       // total packet length
    uint16_t   data_len;      // data in this segment
    uint32_t   ol_flags;      // offload flags (checksum, VLAN, etc.)
    // ... hardware offload fields
};
```

Mbufs come from `rte_mempool`, a lockless, per-lcore cached pool:

```c
// Allocate a burst of mbufs
struct rte_mbuf *mbufs[BURST_SIZE];
rte_pktmbuf_alloc_bulk(pool, mbufs, BURST_SIZE);

// Access packet data
char *data = rte_pktmbuf_mtod(mbufs[0], char *);
uint16_t len = rte_pktmbuf_pkt_len(mbufs[0]);
```

Mempool sizing: too small → exhaustion under burst; too large → cache pressure. Rule of thumb: at least 2× the maximum number of mbufs in flight (NIC Rx rings + Tx rings + application queues).

---

## 5.5 Multi-Queue, RSS, and Flow Director

Modern NICs expose multiple hardware Rx/Tx queues. RSS (Receive Side Scaling) distributes incoming packets across queues by hashing packet headers, allowing multiple CPU cores to process traffic in parallel without contention. DPDK exposes these directly:

```c
struct rte_eth_conf port_conf = {
    .rxmode = {
        .mq_mode = RTE_ETH_MQ_RX_RSS,  // Receive Side Scaling
    },
    .rx_adv_conf.rss_conf = {
        .rss_hf = RTE_ETH_RSS_IP | RTE_ETH_RSS_TCP,
        .rss_key = NULL,  // use default key
    },
};

// Configure 4 Rx queues, 4 Tx queues
rte_eth_dev_configure(port, 4, 4, &port_conf);

// Setup each queue on the appropriate NUMA socket
for (int q = 0; q < 4; q++) {
    rte_eth_rx_queue_setup(port, q, RX_RING_SIZE,
        rte_eth_dev_socket_id(port), NULL, pool);
    rte_eth_tx_queue_setup(port, q, TX_RING_SIZE,
        rte_eth_dev_socket_id(port), NULL);
}
```

**Flow Director (FDIR):** allows explicit packet steering by 5-tuple to a specific queue. Used to pin specific flows to specific lcores, avoiding cross-core cache invalidation:

```c
struct rte_flow_attr attr = { .ingress = 1 };
struct rte_flow_item pattern[] = { /* 5-tuple match */ };
struct rte_flow_action actions[] = {
    { .type = RTE_FLOW_ACTION_TYPE_QUEUE,
      .conf = &(struct rte_flow_action_queue){ .index = 2 } },
    { .type = RTE_FLOW_ACTION_TYPE_END },
};
struct rte_flow *flow = rte_flow_create(port, &attr, pattern, actions, &error);
```

---

## 5.6 OVS-DPDK

Open vSwitch with a DPDK data plane replaces the kernel-based OVS fast path with a DPDK poll-mode loop, enabling software vSwitch throughput of 100+ Gbps per core.

```bash
# Initialize OVS with DPDK
ovs-vsctl set Open_vSwitch . other_config:dpdk-init=true
ovs-vsctl set Open_vSwitch . other_config:pmd-cpu-mask=0xC  # cores 2,3

# Create DPDK bridge
ovs-vsctl add-br br0 -- set Bridge br0 datapath_type=netdev

# Add DPDK port
ovs-vsctl add-port br0 dpdk0 -- set Interface dpdk0 type=dpdk \
    options:dpdk-devargs=0000:03:00.0

# Add vhost-user port for VM/container
ovs-vsctl add-port br0 vhost0 -- set Interface vhost0 type=dpdkvhostuserclient \
    options:vhost-server-path=/var/run/vhost0
```

OVS-DPDK is the standard vSwitch in DPU/SmartNIC deployments (Chapter 10) where it runs on the DPU ARM cores rather than the host CPU.

---

## 5.7 Hardware Offloads via DPDK

Modern NICs offload many operations that would otherwise consume CPU cycles:

```c
// Send with checksum offload
mbuf->ol_flags |= RTE_MBUF_F_TX_IP_CKSUM | RTE_MBUF_F_TX_TCP_CKSUM;
mbuf->l2_len = sizeof(struct rte_ether_hdr);
mbuf->l3_len = sizeof(struct rte_ipv4_hdr);

// Receive: check hardware-verified checksum
if (mbuf->ol_flags & RTE_MBUF_F_RX_IP_CKSUM_BAD)
    rte_pktmbuf_free(mbuf);

// TSO (TCP Segmentation Offload) — allows the application to pass a large
// buffer and have the NIC split it into MTU-sized segments, saving CPU cycles
mbuf->ol_flags |= RTE_MBUF_F_TX_TCP_SEG;
mbuf->tso_segsz = 1460;
```

---

## Lab Walkthrough 5 — DPDK l2fwd Benchmarking

This walkthrough takes you from a bare Ubuntu 24.04 machine through a full software-loopback benchmark using the `net_tap` virtual device, so no spare NIC is required. Each step includes the exact command, expected output, and a verification check.

---

### Step 1 — Reserve hugepages and verify

Check the baseline hugepage allocation before making changes:

```bash
grep -i huge /proc/meminfo
```

Expected output (before):

```
AnonHugePages:         0 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlbsize:     1048576 kB
```

Reserve 1024 × 2 MB hugepages (2 GB total):

```bash
echo 1024 | sudo tee /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

Mount the hugetlbfs filesystem (DPDK requires this mount point):

```bash
sudo mkdir -p /dev/hugepages
sudo mount -t hugetlbfs nodev /dev/hugepages
```

Verify allocation succeeded:

```bash
grep -i huge /proc/meminfo
```

Expected output (after):

```
HugePages_Total:    1024
HugePages_Free:     1024
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

If `HugePages_Total` is still 0, the kernel may lack contiguous memory. Try allocating earlier at boot by adding `hugepages=1024` to `GRUB_CMDLINE_LINUX` in `/etc/default/grub` and running `sudo update-grub`.

---

### Step 2 — Bind a NIC to vfio-pci (or use net_tap if no spare NIC)

**Option A — real NIC (requires a spare NIC not used for SSH/management):**

```bash
# List all Ethernet PCI devices and their current driver
dpdk-devbind.py --status-dev net
```

Sample output:

```
Network devices using kernel driver
============================================
0000:03:00.0 'ConnectX-6 Dx 101d' if=ens3f0 drv=mlx5_core unused=vfio-pci
0000:03:00.1 'ConnectX-6 Dx 101d' if=ens3f1 drv=mlx5_core unused=vfio-pci
```

Enable the vfio-pci module and bind port 1 (keep port 0 for management):

```bash
sudo modprobe vfio-pci
sudo dpdk-devbind.py --bind=vfio-pci 0000:03:00.1
dpdk-devbind.py --status-dev net
```

Expected output after binding:

```
Network devices using DPDK-compatible driver
============================================
0000:03:00.1 'ConnectX-6 Dx 101d' drv=vfio-pci unused=mlx5_core
```

**Option B — net_tap virtual device (no spare NIC required, used in the rest of this walkthrough):**

The `net_tap` PMD creates a kernel TAP interface that DPDK polls from user space. No NIC binding is needed. This is the safest way to explore DPDK on a single machine.

```bash
# No binding step needed — net_tap is a software vdev
# Confirm the tap PMD is available in your DPDK build:
dpdk-testpmd --vdev=net_tap0 -- --help 2>&1 | head -5
```

Expected first line:

```
EAL: Detected 4 lcore(s)
```

---

### Step 3 — Build l2fwd from DPDK examples

If you installed from apt, the examples are pre-built in `/usr/lib/x86_64-linux-gnu/dpdk/examples/`. If you built from source, build the example explicitly:

```bash
# From source build (run from dpdk-stable/build directory):
ninja examples/dpdk-l2fwd

# Verify the binary exists:
ls -lh build/examples/dpdk-l2fwd
```

Expected:

```
-rwxr-xr-x 1 user user 1.2M Apr 22 10:00 build/examples/dpdk-l2fwd
```

If using the apt package, the binary is at `/usr/bin/dpdk-l2fwd` or similar. Check:

```bash
dpkg -L dpdk | grep l2fwd
```

---

### Step 4 — Run l2fwd with net_tap for software testing

Launch l2fwd using the TAP virtual device on logical cores 0–1. The `--` separates EAL arguments from application arguments. `-p 0x1` enables port 0 only; `-q 1` uses 1 queue per lcore:

```bash
sudo ./build/examples/dpdk-l2fwd \
    -l 0-1 \
    -n 2 \
    --vdev "net_tap0,iface=dpdk_tap0" \
    -- \
    -p 0x1 \
    -q 1
```

Expected startup output:

```
EAL: Detected 16 lcore(s)
EAL: Detected 1 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'VA'
EAL: 1024 hugepages of size 2097152 reserved
PMD: net_tap: Initializing pmd_tap for dpdk_tap0 as dpdk_tap0
Port 0: MAC address 02:00:00:00:00:00

Lcore 0: RX port 0 Queue 0
Lcore 1: TX port 0 Queue 0

Port statistics ====================================
Statistics for port 0 --------
Packets sent:                        0
Packets received:                    0
Packets dropped:                     0
```

In a second terminal, generate traffic through the TAP interface:

```bash
# Send 10000 ICMP packets through the tap interface
sudo ping -c 10000 -f -I dpdk_tap0 192.0.2.1
```

Watch the l2fwd statistics update every second:

```
Port statistics ====================================
Statistics for port 0 --------
Packets sent:                    10000
Packets received:                10000
Packets dropped:                     0

Aggregate statistics =======
Total packets sent:              10000
Total packets received:          10000
Total packets dropped:               0
```

---

### Step 5 — Interpret output: Mpps and Gbps at different frame sizes

l2fwd prints per-second packet and byte rates when traffic is flowing continuously. To drive sustained traffic, use a packet generator. With `dpdk-pktgen` (a DPDK-based traffic generator that can drive multiple ports at line rate with configurable frame sizes and patterns; build from source: https://github.com/pktgen/Pktgen-DPDK — no distro package exists) on the same machine via two TAP devices:

```bash
# Create a veth pair bridged to two TAP devices for loopback testing
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth0 up
sudo ip link set veth1 up
```

Then run l2fwd with two ports:

```bash
sudo ./build/examples/dpdk-l2fwd \
    -l 0-3 -n 4 \
    --vdev "net_tap0,iface=dpdk_tap0" \
    --vdev "net_tap1,iface=dpdk_tap1" \
    -- -p 0x3 -q 2
```

Example throughput results at various frame sizes (software TAP, single socket):

| Frame size | Mpps (1 lcore) | Gbps (1 lcore) | CPU % |
|-----------|---------------|---------------|-------|
| 64 B      | ~2.1 Mpps     | ~1.1 Gbps     | 100%  |
| 256 B     | ~1.8 Mpps     | ~3.7 Gbps     | 100%  |
| 1500 B    | ~0.5 Mpps     | ~6.0 Gbps     | 100%  |

Note: TAP device throughput is limited by kernel bridge overhead. On a real NIC with vfio-pci, a Mellanox ConnectX-6 achieves 100+ Mpps at 64B on a single core.

The l2fwd statistics line format:

```
Port 0: RX-packets: 21000000  RX-missed: 0  RX-bytes: 1344000000
Port 0: TX-packets: 21000000  TX-errors: 0  TX-bytes: 1344000000
Throughput (since last show)
Rx-pps:      2100000          Rx-bps:   1075200000
Tx-pps:      2100000          Tx-bps:   1075200000
```

- `Rx-pps` — millions of packets per second received
- `Rx-bps` — aggregate bit rate (divide by 1e9 for Gbps)
- `RX-missed` — packets dropped because the Rx ring was full; non-zero means you need more lcores or a larger ring

---

### Step 6 — Enable checksum offload and measure CPU reduction

Without offload, l2fwd recomputes the IP checksum in software for every forwarded packet. With NIC offload, the hardware handles it. To enable TX checksum offload, recompile l2fwd after adding the offload flag to the mbuf:

In `examples/l2fwd/main.c`, add inside the forwarding function:

```c
mbuf->ol_flags |= RTE_MBUF_F_TX_IP_CKSUM | RTE_MBUF_F_TX_UDP_CKSUM;
mbuf->l2_len = sizeof(struct rte_ether_hdr);
mbuf->l3_len = sizeof(struct rte_ipv4_hdr);
```

And in the port configuration struct set:

```c
.txmode = { .offloads = RTE_ETH_TX_OFFLOAD_IPV4_CKSUM |
                        RTE_ETH_TX_OFFLOAD_UDP_CKSUM },
```

Rebuild and rerun, then monitor CPU usage in a second terminal:

```bash
# Watch CPU consumption of the l2fwd process
top -p $(pgrep dpdk-l2fwd) -d 1
```

Example comparison (real NIC, 10GbE, 64B frames, 1 lcore):

| Mode                     | Mpps | CPU % |
|--------------------------|------|-------|
| Software checksum        | 14.8 | 100%  |
| Hardware checksum offload| 14.8 | 72%   |

The throughput stays the same (line-rate) but the lcore spends ~28% fewer cycles on checksum computation, freeing headroom for other work (ACL lookups — Access Control List packet classification using `rte_acl` — metering, encapsulation).

---

### Step 7 — Verification checklist

After completing the walkthrough, confirm:

```bash
# 1. Hugepages are still allocated
grep HugePages_Free /proc/meminfo
# HugePages_Free should be less than HugePages_Total (DPDK consumed some)

# 2. vfio-pci module is loaded (if you used a real NIC)
lsmod | grep vfio
# Expected: vfio_pci   ...   0

# 3. Rebind the NIC back to its kernel driver when done
sudo dpdk-devbind.py --bind=mlx5_core 0000:03:00.1
dpdk-devbind.py --status-dev net
# NIC should appear back under "Network devices using kernel driver"

# 4. Release hugepages
echo 0 | sudo tee /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
grep HugePages_Total /proc/meminfo
# HugePages_Total:       0
```

---

## Summary

- DPDK bypasses the kernel entirely by polling NIC rings from user space, enabling near-line-rate packet processing on commodity hardware.
- Hugepages and NUMA-aware memory allocation are mandatory for performance; CPU pinning eliminates scheduling jitter.
- The PMD + mbuf + mempool trio is the fundamental building block; everything else (multi-queue, Flow Director, offloads) builds on top.
- OVS-DPDK is the standard kernel-bypass vSwitch used in DPU deployments.

---

## References

- DPDK documentation: doc.dpdk.org
- DPDK Performance Report: core.dpdk.org/perf-report/
- Rizzo, L., *netmap: a novel framework for fast packet I/O*, USENIX ATC 2012
- pfring, DPDK, and netmap comparative analysis: ntop.org


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).