# Chapter 10 — SmartNIC & DPU Programming

**Part III: Programmable Fabric** | ~20 pages

---

## Introduction

Modern GPU servers contain an architectural tension: the host CPU must simultaneously feed data to the GPUs, manage container networking overhead, enforce security policy, and export telemetry — while also not impeding the GPUs' primary work. In practice, a busy **OVS** (**Open vSwitch**) instance or an active **IPsec** termination can consume 2–4 CPU cores per host, a significant fraction of the total CPU budget on a 2-socket server. The **DPU** (**Data Processing Unit**) resolves this tension by moving network processing off the host CPU entirely.

A **DPU** is a **SmartNIC** that integrates a high-performance NIC ASIC with a full general-purpose compute subsystem — **ARM** cores, local DRAM, and **PCIe** connectivity to the host — on a single card. From the host's perspective, a **DPU** appears as a standard **PCIe** NIC with **SR-IOV** virtual functions. From the fabric's perspective, it is a 400 Gbps network endpoint. On the **DPU** itself, a full **Linux** environment runs an independent software stack: **OVS-DPDK** for virtual switching, **RDMA** proxy for multi-tenant isolation, **IPsec** for encryption, and telemetry agents — all without consuming a single host CPU cycle.

This chapter covers the **DPU** architecture using **NVIDIA BlueField-3** as the primary reference, the **DOCA** (**Data Center Infrastructure on a Chip Architecture**) SDK for **DPU** application development, **OVS-DPDK** offload to the **DPU** **ARM** cores, **eBPF** offload to NIC silicon, and **P4**/**PNA** programs on **SmartNIC** targets such as **AMD Pensando DSC**. Each technology is placed in the context of AI cluster deployment patterns: which workloads belong on the **DPU**, which stay on the host, and how the **DPU**'s management isolation enables secure multi-tenant GPU infrastructure.

The lab walkthrough uses **OVS-DPDK** running on the host as a functional stand-in for **DPU** **OVS** — the architecture, configuration, and observability tooling are identical — and the **DOCA** development container to compile a flow steering example without requiring physical **BlueField** hardware.

This chapter sits at the intersection of Part II (Kernel-Bypass & Programmable I/O) and Part III (Programmable Fabric): the **DPU** is the natural convergence point for **DPDK** (Chapter 5), **eBPF**/**XDP** (Chapter 7), and **P4** (Chapter 9), applied at the host edge rather than in the fabric switches. It connects forward to Chapter 26 (Network Security & Zero Trust), where the **DPU**'s isolation properties enable the cryptographic separation required for secure AI infrastructure.

---

## Installation

The `openvswitch-dpdk` package installs **OVS-DPDK**, which serves as a host-side functional stand-in for the **OVS-DPDK** instance that runs on a real **DPU**'s **ARM** cores; its configuration, **OpenFlow** control, and **PMD** statistics interfaces are identical to the **DPU** deployment. The **NVIDIA** **DOCA** SDK is available either via the **NVIDIA** apt repository on physical **BlueField** hardware, or as the `nvcr.io/nvidia/doca/doca:2.6.0-devel` container image that provides the full **DOCA** headers and libraries for compiling flow-steering applications without requiring a physical card. **CMake** and `pkg-config` are needed to build the **DOCA** C example, and `grpcio` is installed to support the **Python** telemetry scripts that communicate with **DOCA**'s **gRPC** telemetry service.

### OVS-DPDK (host-side stand-in for DPU OVS)

OVS-DPDK is the DPDK-accelerated data path for Open vSwitch (OVS). Standard OVS uses the kernel's network stack and context switches for packet processing; OVS-DPDK moves the forwarding plane entirely to user space using DPDK poll-mode drivers, eliminating interrupt and system-call overhead. On a DPU, OVS-DPDK runs on the ARM cores and offloads matched flows to the ConnectX ASIC for wire-rate forwarding.

```bash
sudo apt install -y openvswitch-switch openvswitch-dpdk
# Verify
ovs-vswitchd --version
# ovs-vswitchd (Open vSwitch) 3.x.x
dpdk-testpmd --version
# DPDK testpmd version x.x
```

### DOCA SDK — on Physical BlueField Hardware

```bash
# Add NVIDIA DOCA repository (BlueField ARM OS or host x86 with BF3)
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
# Install on the BlueField ARM OS:
sudo apt install -y doca-sdk doca-tools
# Verify
doca_version
# DOCA Version: 2.6.x
```

### DOCA Development Container (no hardware required)

```bash
# Pull the official DOCA devel image from NGC
docker pull nvcr.io/nvidia/doca/doca:2.6.0-devel

# Run an interactive shell
docker run --rm -it \
    --name doca-dev \
    nvcr.io/nvidia/doca/doca:2.6.0-devel \
    bash

# Inside the container: verify DOCA is present
doca_version
# DOCA Version: 2.6.0
```

### Python (for DOCA telemetry gRPC scripts)

```bash
uv venv .venv
source .venv/bin/activate
uv pip install grpcio
python3 -c "import grpc; print('grpc OK')"
```

### CMake for DOCA C Example

Install CMake and create `CMakeLists.txt` for the `flow_steer` example:

```bash
sudo apt install -y cmake pkg-config build-essential
cmake --version
# cmake version 3.28.x
```

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.20)
project(doca_flow_example C)

find_package(PkgConfig REQUIRED)
pkg_check_modules(DOCA REQUIRED doca-flow doca-common)

add_executable(flow_steer flow_steer.c)
target_include_directories(flow_steer PRIVATE ${DOCA_INCLUDE_DIRS})
target_link_libraries(flow_steer ${DOCA_LIBRARIES})
```

Build:

```bash
mkdir -p build && cd build
cmake ..
make -j$(nproc)
# [100%] Built target flow_steer
```

---

## 10.1 The DPU as Network Control Point

A Data Processing Unit (DPU) is a SmartNIC with general-purpose compute: ARM cores, memory, and a high-performance NIC ASIC on a single PCIe card. In an AI cluster, the DPU sits between the host CPU and the network, making it the natural place to run:

- OVS-DPDK (vSwitch) without consuming host CPU cores
- RDMA proxy and GPUDirect acceleration (GPUDirect RDMA is an NVIDIA technology that allows the NIC to DMA data directly into GPU memory, bypassing the host CPU and system DRAM entirely, critical for low-latency NCCL collective operations)
- IPsec / TLS termination for secure multi-tenant fabrics
- Network telemetry collection without host involvement
- P4 programs compiled to the NIC ASIC

The key value proposition: host CPU cores freed from networking tasks are available for GPU feeding workloads. A DPU handling OVS offload can return 2–4 CPU cores per host.

### DPU vs SmartNIC vs Standard NIC

| Class | Examples | Capabilities |
|---|---|---|
| Standard NIC | Mellanox ConnectX-6 | RDMA, SR-IOV (Single Root I/O Virtualization — a PCIe standard that allows a single physical NIC to present multiple Virtual Functions to VMs or containers, each with dedicated queues and hardware isolation), hardware offloads |
| SmartNIC | Netronome Agilio, Pensando DSC | Programmable forwarding (P4/eBPF), limited compute |
| DPU | NVIDIA BlueField-3, AMD Pensando Elba, Marvell OCTEON | Full ARM SoC + high-port NIC ASIC, runs Linux |

---

## 10.2 DPU Architecture: BlueField-3

NVIDIA's BlueField-3 DPU is the most widely deployed in AI clusters:

```
Host CPU (x86)
    │
   PCIe Gen5 ×16
    │
┌───────────────────────────────────────┐
│  BlueField-3 DPU                      │
│                                       │
│  ┌────────────┐  ┌──────────────────┐ │
│  │ ARM Cortex │  │  ConnectX-7 ASIC │ │
│  │ A78 × 16  │  │  400Gbps × 2     │ │
│  │ 32GB LPDDR5│  │  RDMA, RoCEv2   │ │
│  └────────────┘  │  P4 pipeline     │ │
│       ↕           │  crypto engine   │ │
│  NVMe, PCIe bus  └──────────────────┘ │
│                                       │
└────────────────────────────────────┬──┘
                                     │
                              2× 400GbE / IB ports
                              (to fabric)
```

The DPU runs its own Linux (Ubuntu or RHEL) on the ARM cores, independent of the host OS. From the host's perspective, the DPU appears as a PCIe NIC with VFs.

---

## 10.3 NVIDIA DOCA SDK

DOCA (Data Center Infrastructure on a Chip Architecture) is NVIDIA's programming framework for BlueField DPUs. It provides:
- High-level service APIs (flow steering, telemetry, crypto, storage)
- Low-level access to the ConnectX ASIC via RDMA/DPDK
- Service framework for persistent DPU applications

### 10.3.1 DOCA Installation

```bash
# On the DPU ARM OS (after BlueField OS boot)
apt install doca-sdk doca-tools

# Verify
doca_version
```

### 10.3.2 Flow Steering with DOCA

```c
#include <doca_flow.h>

// Initialize DOCA flow
struct doca_flow_cfg flow_cfg = {
    .queues = 4,
    .mode_args = "vnf",
    .resource.nb_counters = 1024,
};
doca_flow_init(&flow_cfg, &error);

// Create pipe: match on dst IP, set egress port
struct doca_flow_match match = {
    .outer.ip4.dst_ip = 0xC0A80101,   // 192.168.1.1
};
struct doca_flow_fwd fwd = {
    .type = DOCA_FLOW_FWD_PORT,
    .port_id = 1,
};
struct doca_flow_pipe_cfg pipe_cfg = {
    .attr = { .name = "ip_fwd", .type = DOCA_FLOW_PIPE_BASIC },
    .match = &match,
    .fwd = &fwd,
};
doca_flow_pipe_create(&pipe_cfg, &fwd, NULL, &pipe, &error);

// Add entry to the pipe
doca_flow_pipe_add_entry(queue_id, pipe, &match, NULL, &fwd,
    DOCA_FLOW_NO_WAIT, NULL, &entry, &error);
```

### 10.3.3 DOCA Telemetry

```c
#include <doca_telemetry.h>

// Export NIC counters to a telemetry endpoint
struct doca_telemetry_type *counter_type;
doca_telemetry_type_add_uint64(counter_type, "rx_bytes", 0);
doca_telemetry_type_add_uint64(counter_type, "tx_bytes", 8);

// Flush telemetry data (triggers export to Prometheus/OTel)
doca_telemetry_flush(schema);
```

---

## 10.4 OVS-DPDK Offload to DPU

Running OVS on the host CPU costs 2–4 cores per host. Moving OVS-DPDK to the DPU reclaims those cores while maintaining identical vSwitch semantics.

### Architecture

```
Host (x86):
  Virtual Machine / Container
       ↕ virtio / vhost-user
  vhost-user socket (shared memory to DPU)

DPU (ARM):
  OVS-DPDK (running on ARM cores)
    ↕ DPDK PMD
  ConnectX-7 ASIC (hardware acceleration of OVS actions)
    ↕
  Network fabric (400GbE)
```

`vhost-user` is a userspace implementation of the Linux vhost protocol that uses a Unix domain socket and shared memory to transfer packet buffer descriptors between a guest VM or container and an OVS-DPDK process, eliminating kernel involvement from the fast-path packet transfer. `virtio` is the paravirtualized I/O standard that the guest VM uses to communicate with the virtual NIC presented by the host or DPU.

### Configuration

```bash
# On DPU ARM OS:
# Initialize OVS with DPDK
ovs-vsctl set Open_vSwitch . other_config:dpdk-init=true
ovs-vsctl set Open_vSwitch . other_config:pmd-cpu-mask=0xF  # cores 0-3

# Create DPDK bridge
ovs-vsctl add-br br0 -- set Bridge br0 datapath_type=netdev

# Add representor port — a software port in OVS that represents a physical
# VF (Virtual Function) from the host, used to apply OVS policies to traffic
# entering and leaving each VF.
ovs-vsctl add-port br0 pf0vf0 -- set Interface pf0vf0 \
    type=dpdk options:dpdk-devargs=0000:03:00.0,representor=[0]

# Add uplink port
ovs-vsctl add-port br0 p0 -- set Interface p0 \
    type=dpdk options:dpdk-devargs=0000:03:00.0

# Hardware offload: flows matching these rules execute on ConnectX ASIC
ovs-vsctl set Open_vSwitch . other_config:hw-offload=true
```

With hardware offload enabled, matched flows are programmed into the ConnectX ASIC and forwarded at wire rate without involving the DPU ARM cores.

---

## 10.5 eBPF Offload to SmartNIC

Some NICs (historically Netronome Agilio; experimentally on Mellanox) support eBPF program offload: the BPF bytecode is compiled to run directly on the NIC ASIC, completely bypassing the host CPU:

```bash
# Attach XDP program in offload mode (NIC-executed)
ip link set dev eth0 xdpoffload obj xdp_counter.bpf.o sec xdp

# Verify offload
bpftool prog show
# 42: xdp  name count_packets  tag abc123  offloaded
#         loaded_at 2026-04-22T10:00:00  uid 0
#         xlated 64B  jited 128B

# The program now runs on NIC silicon at line rate
# Zero host CPU involvement
```

eBPF offload is currently limited by what the NIC's eBPF JIT can express (no complex maps, limited helper calls), but is advancing rapidly.

---

## 10.6 P4 on SmartNICs

Several SmartNIC targets support P4 programs:

| Platform | P4 Architecture | Notes |
|---|---|---|
| Intel Tofino-based SmartNICs | TNA (Tofino Native Architecture — Intel's proprietary P4 architecture model for Tofino ASICs, which exposes stateful ALUs and recirculation capabilities not present in the portable V1Model) | Full Tofino pipeline on a PCIe card |
| AMD Pensando DSC | PNA (Portable NIC Architecture) | P4₁₆ with PNA extern model |
| Marvell OCTEON | Custom | Partial P4 support |

AMD's Pensando DSC uses the P4 **Portable NIC Architecture (PNA)**, standardized by the P4 Language Consortium:

```p4
#include <core.p4>
#include <pna.p4>

// PNA-specific extern: hash for RSS
Hash<bit<16>>(PNA_HashAlgorithm_t.TARGET_DEFAULT) rss_hash;

control ingress(inout Headers hdr,
                inout Metadata meta,
                in    pna_main_input_metadata_t  im,
                inout pna_main_output_metadata_t om) {

    action fwd_to_host() {
        send_to_port(PNA_PORT_HOST);
    }

    table rx_steering {
        key = { hdr.ipv4.dst_addr: lpm; }
        actions = { fwd_to_host; drop; }
    }

    apply { rx_steering.apply(); }
}
```

---

## 10.7 RDMA Proxy and GPUDirect on DPU

For hosts with multiple tenants (containers) sharing a single physical NIC, the DPU can act as an RDMA proxy: presenting each tenant with a virtual RDMA device, while multiplexing their traffic onto a single physical port with isolation and rate limiting.

```
Tenant A (container):
  virtual RDMA device → DPU proxy → physical port
Tenant B (container):
  virtual RDMA device → DPU proxy → physical port (same)
```

This enables RDMA multi-tenancy without SR-IOV VF exhaustion, implemented as a DOCA service on the DPU.

---

## 10.8 DPU in AI Cluster Context

Typical BlueField-3 deployment in a GPU server:

```
GPU server (8× H100):
  8× NIC ports (one per GPU) → 8 rails → fabric
  1× DPU → management network
    DPU handles:
      - OVS offload for container management traffic
      - IPsec for east-west encryption
      - Telemetry export (NIC counters → OpenTelemetry)
      - GPUDirect RDMA acceleration
      - Secure boot attestation
```

IPsec (Internet Protocol Security) is a suite of IETF standards that authenticate and encrypt IP packets at the network layer; running IPsec termination on the DPU provides zero-trust encrypted east-west channels between GPU servers without consuming host CPU cycles. Secure boot attestation uses the DPU's hardware root of trust (a TPM or equivalent) to cryptographically verify that each host booted unmodified firmware and a known-good kernel image, providing a verifiable chain of custody for multi-tenant AI infrastructure.

The DPU's network management port typically carries control and management traffic, completely isolated from the high-bandwidth GPU training rails.

---

## Lab Walkthrough 10 — OVS-DPDK Stand-in and DOCA Container

Since physical BlueField hardware is rarely available in development environments, this walkthrough uses OVS-DPDK on the host as a functional stand-in for DPU OVS, then uses the DOCA development container to compile and run a flow steering example.

### Step 1 — Verify OVS-DPDK Installation and Enable DPDK

Ensure hugepages are configured (DPDK requires them):

```bash
# Allocate 1 GB hugepages (1024 × 2 MB pages)
sudo sh -c 'echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages'

# Verify
cat /proc/meminfo | grep HugePages
# HugePages_Total:    1024
# HugePages_Free:     1024
# Hugepagesize:       2048 kB

# Mount hugetlbfs (if not already mounted)
sudo mkdir -p /dev/hugepages
sudo mount -t hugetlbfs nodev /dev/hugepages || true
```

Enable DPDK in OVS:

```bash
# Start OVS services
sudo systemctl start openvswitch-switch

# Tell OVS to use DPDK
sudo ovs-vsctl set Open_vSwitch . other_config:dpdk-init=true

# Set PMD thread CPU affinity (cores 0 and 1)
sudo ovs-vsctl set Open_vSwitch . other_config:pmd-cpu-mask=0x3

# Restart OVS to apply DPDK init
sudo systemctl restart openvswitch-switch
```

Verify DPDK is active:

```bash
sudo ovs-vsctl get Open_vSwitch . dpdk-initialized
# true
```

### Step 2 — Create the DPDK Bridge

```bash
# Create a DPDK-backed bridge (datapath_type=netdev = userspace DPDK)
sudo ovs-vsctl add-br br-dpdk -- \
    set Bridge br-dpdk datapath_type=netdev

# Confirm bridge creation
sudo ovs-vsctl show
# Bridge br-dpdk
#     datapath_type: netdev
#     Port br-dpdk
#         Interface br-dpdk
#             type: internal
```

### Step 3 — Add DPDK Ports Using --vdev (Virtual Devices)

In the absence of physical DPDK-capable NICs, use DPDK's `net_tap` or `net_virtio_user` virtual device driver, which creates a kernel TAP interface backed by DPDK:

```bash
# Add two virtual DPDK ports via net_tap vdev
# Port dpdk0 — backed by a TAP interface named dtap0
sudo ovs-vsctl add-port br-dpdk dpdk0 -- \
    set Interface dpdk0 \
    type=dpdk \
    options:dpdk-devargs="net_tap0,iface=dtap0"

# Port dpdk1 — backed by TAP interface dtap1
sudo ovs-vsctl add-port br-dpdk dpdk1 -- \
    set Interface dpdk1 \
    type=dpdk \
    options:dpdk-devargs="net_tap1,iface=dtap1"

# Verify ports are up
sudo ovs-vsctl show
# Bridge br-dpdk
#     datapath_type: netdev
#     Port dpdk0
#         Interface dpdk0
#             type: dpdk
#             options: {dpdk-devargs=net_tap0,iface=dtap0}
#     Port dpdk1
#         Interface dpdk1
#             type: dpdk
#             options: {dpdk-devargs=net_tap1,iface=dtap1}

# The kernel TAP interfaces dtap0 and dtap1 are now visible
ip link show dtap0
# 7: dtap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ...

ip link show dtap1
# 8: dtap1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ...
```

### Step 4 — Assign IP Addresses to the TAP Interfaces

These TAP interfaces represent the two "hosts" connected through the virtual OVS switch:

```bash
# Bring up and address the tap interfaces (simulating two hosts)
sudo ip addr add 192.168.10.1/24 dev dtap0
sudo ip link set dtap0 up

sudo ip addr add 192.168.10.2/24 dev dtap1
sudo ip link set dtap1 up

# Verify
ip addr show dtap0
# inet 192.168.10.1/24 scope global dtap0
ip addr show dtap1
# inet 192.168.10.2/24 scope global dtap1
```

### Step 5 — Add OpenFlow Rules and Test Forwarding

By default OVS installs a normal-forwarding rule. Demonstrate explicit OpenFlow control:

```bash
# Remove the default NORMAL rule
sudo ovs-ofctl del-flows br-dpdk

# Show port numbers
sudo ovs-ofctl show br-dpdk
# OFPT_FEATURES_REPLY: dpid:...
#  1(dpdk0): addr:...
#  2(dpdk1): addr:...
#  LOCAL(br-dpdk): addr:...

# Add explicit L2 forwarding rules:
# Traffic arriving on port 1 (dpdk0) → forward to port 2 (dpdk1)
sudo ovs-ofctl add-flow br-dpdk \
    "priority=100,in_port=1,actions=output:2"

# Traffic arriving on port 2 (dpdk1) → forward to port 1 (dpdk0)
sudo ovs-ofctl add-flow br-dpdk \
    "priority=100,in_port=2,actions=output:1"

# Allow ARP through (needed for IP resolution)
sudo ovs-ofctl add-flow br-dpdk \
    "priority=200,arp,actions=flood"

# Verify the flow table
sudo ovs-ofctl dump-flows br-dpdk
# cookie=0x0, duration=2.3s, table=0, n_packets=0, n_bytes=0,
#   priority=200,arp actions=FLOOD
# cookie=0x0, duration=1.8s, table=0, n_packets=0, n_bytes=0,
#   priority=100,in_port=dpdk0 actions=output:dpdk1
# cookie=0x0, duration=1.5s, table=0, n_packets=0, n_bytes=0,
#   priority=100,in_port=dpdk1 actions=output:dpdk0
```

Test forwarding with ping:

```bash
# ping from dtap0 to dtap1 (through OVS)
ping -I dtap0 -c 4 192.168.10.2
```

Expected output:

```
PING 192.168.10.2 (192.168.10.2) from 192.168.10.1 dtap0: 56(84) bytes of data.
64 bytes from 192.168.10.2: icmp_seq=1 ttl=64 time=0.4 ms
64 bytes from 192.168.10.2: icmp_seq=2 ttl=64 time=0.3 ms
64 bytes from 192.168.10.2: icmp_seq=3 ttl=64 time=0.3 ms
64 bytes from 192.168.10.2: icmp_seq=4 ttl=64 time=0.4 ms
--- 192.168.10.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

Verify packet counts in the flow table:

```bash
sudo ovs-ofctl dump-flows br-dpdk
# priority=100,in_port=dpdk0 actions=output:dpdk1
#   n_packets=4, n_bytes=392
# priority=100,in_port=dpdk1 actions=output:dpdk0
#   n_packets=4, n_bytes=392
```

### Step 6 — Add an IP-Match Rule and Observe Selective Forwarding

Add a rule that matches on a specific destination IP and sends to a controller (or drops) — demonstrating DPU-style policy enforcement:

```bash
# Drop all traffic to 192.168.10.99 (simulate a blocked tenant)
sudo ovs-ofctl add-flow br-dpdk \
    "priority=300,ip,nw_dst=192.168.10.99,actions=drop"

# Add a rule to send matching flows to the OVS "controller" port for inspection
# (In a real DPU this would route to a DOCA service on the ARM cores)
sudo ovs-ofctl add-flow br-dpdk \
    "priority=250,ip,nw_dst=192.168.10.2,tcp,tp_dst=8080,actions=controller"

# Confirm the new rules are present
sudo ovs-ofctl dump-flows br-dpdk | grep "priority=3\|priority=2[5]"
# priority=300,ip,nw_dst=192.168.10.99 actions=drop
# priority=250,ip,tcp,nw_dst=192.168.10.2,tp_dst=8080 actions=CONTROLLER:65535
```

### Step 7 — Demonstrate Hardware Offload Concepts with PMD Stats

Even without a physical offload-capable NIC, you can observe the DPDK PMD (Poll Mode Driver — a user-space driver that continuously polls NIC hardware queues rather than waiting for interrupts, trading CPU cycles for deterministic low-latency packet processing) statistics to understand how offload telemetry would appear on a real DPU:

```bash
# Show PMD thread statistics (shows which CPU cores are polling which ports)
sudo ovs-appctl dpif-netdev/pmd-stats-show
```

Expected output:

```
pmd thread numa_id 0 core_id 0:
  emc hits:                         8
  smc hits:                         0
  megaflow hits:                   12
  avg. subtable lookups per hit:  1.00
  miss with success upcall:        4
  miss with failed upcall:         0
  avg. packets per output batch:  1.00
  idle cycles:            4123456789 (99.97%)
  processing cycles:           987654 (0.03%)
  avg cycles per output batch:  24691.35
  avg processing cycles per hit: 82.31
```

Key fields:
- `emc hits` — Exact Match Cache hits (fastest path, O(1) lookup)
- `megaflow hits` — wildcard flow cache hits (DPDK datapath)
- `miss with success upcall` — packets sent to OVS userspace for rule installation (slow path)
- `idle cycles` — PMD core spinning waiting for packets

On a real DPU with hardware offload enabled, `megaflow hits` would drop toward zero after the first few packets of each flow, replaced by hardware counters in the ConnectX ASIC:

```bash
# Show per-port receive/transmit stats
sudo ovs-appctl dpif-netdev/pmd-rxq-show
# pmd thread numa_id 0 core_id 0:
#   interface: dpdk0
#     rxq 0:  cpu cycles/rxbatch=12345
#   interface: dpdk1
#     rxq 0:  cpu cycles/rxbatch=11987
```

Retrieve per-interface statistics:

```bash
sudo ovs-vsctl get Interface dpdk0 statistics
# {rx_bytes=392, rx_packets=4, tx_bytes=392, tx_packets=4}
```

### Step 8 — Run the DOCA Development Container and Verify the Toolchain

Start the DOCA devel container (no BlueField hardware required):

```bash
docker run --rm -it \
    --name doca-dev \
    --cap-add NET_ADMIN \
    nvcr.io/nvidia/doca/doca:2.6.0-devel \
    bash
```

Inside the container, verify the DOCA toolchain:

```bash
# Check DOCA version
doca_version
# DOCA Version: 2.6.0

# List installed DOCA packages
dpkg -l | grep doca
# ii  doca-sdk       2.6.0-...  amd64  DOCA Software Development Kit
# ii  doca-tools     2.6.0-...  amd64  DOCA CLI tools and utilities
# ii  doca-flow      2.6.0-...  amd64  DOCA Flow steering library

# Check DOCA headers are present
ls /opt/mellanox/doca/include/ | head -10
# doca_argp.h
# doca_buf.h
# doca_buf_inventory.h
# doca_common.h
# doca_ctx.h
# doca_dev.h
# doca_flow.h
# doca_log.h
# doca_telemetry.h
# doca_types.h

# Check pkg-config entries
pkg-config --modversion doca-flow
# 2.6.0
```

### Step 9 — Write, Compile, and Run flow_steer.c in the Container

Inside the DOCA container, create the C source file:

```bash
mkdir -p /workspace/flow_example
cat > /workspace/flow_example/flow_steer.c << 'EOF'
/*
 * flow_steer.c — Minimal DOCA Flow example
 * Initialises the DOCA framework, queries available devices,
 * and prints basic device info. Runs without physical BlueField hardware
 * using the DOCA simulation mode.
 */
#include <stdio.h>
#include <stdlib.h>
#include <doca_dev.h>
#include <doca_flow.h>
#include <doca_log.h>

DOCA_LOG_REGISTER(FLOW_STEER);

int main(void)
{
    struct doca_devinfo **devlist = NULL;
    uint32_t nb_devs = 0;
    doca_error_t result;
    char pci_addr[DOCA_DEVINFO_PCI_ADDR_SIZE];

    /* Enumerate available DOCA devices */
    result = doca_devinfo_create_list(&devlist, &nb_devs);
    if (result != DOCA_SUCCESS) {
        DOCA_LOG_ERR("Failed to list devices: %s",
                     doca_error_get_descr(result));
        return EXIT_FAILURE;
    }

    printf("Found %u DOCA device(s):\n", nb_devs);
    for (uint32_t i = 0; i < nb_devs; i++) {
        result = doca_devinfo_get_pci_addr_str(devlist[i], pci_addr);
        if (result == DOCA_SUCCESS)
            printf("  [%u] PCI address: %s\n", i, pci_addr);
        else
            printf("  [%u] PCI address: (unavailable in simulation)\n", i);
    }

    /* Initialize DOCA Flow subsystem */
    struct doca_flow_cfg flow_cfg = {
        .queues           = 1,
        .mode_args        = "vnf",
        .resource         = { .nb_counters = 64 },
    };
    struct doca_flow_error flow_err = {0};

    result = doca_flow_init(&flow_cfg, &flow_err);
    if (result != DOCA_SUCCESS) {
        printf("doca_flow_init: %s (code %d)\n",
               flow_err.message, flow_err.type);
        printf("(Expected in simulation — no physical ASIC present)\n");
    } else {
        printf("DOCA Flow initialized successfully.\n");
        doca_flow_destroy();
    }

    doca_devinfo_destroy_list(devlist);
    printf("flow_steer: done.\n");
    return EXIT_SUCCESS;
}
EOF
```

Create `CMakeLists.txt` in the container:

```bash
cat > /workspace/flow_example/CMakeLists.txt << 'EOF'
cmake_minimum_required(VERSION 3.20)
project(doca_flow_example C)

find_package(PkgConfig REQUIRED)
pkg_check_modules(DOCA REQUIRED doca-flow doca-common)

add_executable(flow_steer flow_steer.c)
target_include_directories(flow_steer PRIVATE ${DOCA_INCLUDE_DIRS})
target_link_libraries(flow_steer ${DOCA_LIBRARIES})
EOF
```

Build inside the container:

```bash
cd /workspace/flow_example
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
```

Expected cmake output:

```
-- The C compiler identification is GNU 11.x.x
-- Detecting C compiler ABI info ... done
-- Found PkgConfig: /usr/bin/pkg-config (found version 0.29.2)
-- Checking for module 'doca-flow'
--   Found doca-flow, version 2.6.0
-- Checking for module 'doca-common'
--   Found doca-common, version 2.6.0
-- Configuring done
-- Build files have been written to: /workspace/flow_example/build
```

```bash
make -j$(nproc)
```

Expected make output:

```
[ 50%] Building C object CMakeFiles/flow_steer.dir/flow_steer.c.o
[100%] Linking C executable flow_steer
[100%] Built target flow_steer
```

Run the binary:

```bash
./flow_steer
```

Expected output (simulation mode — no physical BlueField card):

```
Found 0 DOCA device(s):
doca_flow_init: No devices available (code 1)
(Expected in simulation — no physical ASIC present)
flow_steer: done.
```

On a real BlueField-3 with DOCA 2.6, the output would be:

```
Found 2 DOCA device(s):
  [0] PCI address: 0000:03:00.0
  [1] PCI address: 0000:03:00.1
DOCA Flow initialized successfully.
flow_steer: done.
```

### Step 10 — Cleanup

```bash
# Outside Docker: stop container if still running
docker rm -f doca-dev 2>/dev/null || true

# Remove OVS bridge and ports
sudo ovs-vsctl del-br br-dpdk

# Remove TAP IP addresses
sudo ip addr del 192.168.10.1/24 dev dtap0 2>/dev/null || true
sudo ip addr del 192.168.10.2/24 dev dtap1 2>/dev/null || true

# Release hugepages
sudo sh -c 'echo 0 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages'

# Confirm OVS is clean
sudo ovs-vsctl show
# (no bridges listed)
```

---

## Summary

- DPUs run a full Linux OS on ARM cores attached to a high-performance NIC ASIC, making them the natural offload target for OVS, RDMA proxy, and telemetry.
- NVIDIA DOCA provides the SDK for BlueField DPU programming: flow steering, crypto, telemetry, and storage acceleration.
- OVS-DPDK offloaded to a DPU reclaims 2–4 host CPU cores per server — a meaningful fraction of total host CPU budget.
- eBPF offload and P4/PNA on SmartNICs extend the programmable data plane concept to the NIC silicon, enabling wire-rate custom forwarding without any host CPU involvement.

---

## References

- [NVIDIA DOCA SDK documentation](https://docs.nvidia.com/doca/sdk/)
- [NVIDIA DOCA GitHub samples](https://github.com/NVIDIA/doca-samples)
- [BlueField-3 DPU — NVIDIA product page](https://www.nvidia.com/en-us/networking/products/data-processing-unit/)
- [P4 Portable NIC Architecture (PNA) specification](https://p4.org/p4-spec/docs/PNA-working-draft.html)
- [OVS hardware offload (OVS-DPDK) documentation](https://docs.openvswitch.org/en/latest/topics/dpdk/hw-offload/)
- [OVS — Open vSwitch](https://www.openvswitch.org)
- [DPDK — Data Plane Development Kit](https://www.dpdk.org)
- [Netronome Agilio SmartNIC](https://www.netronome.com/products/agilio-cx/)
- [AMD Pensando DSC SmartNIC](https://www.amd.com/en/products/accelerators/pensando.html)
- [Marvell OCTEON DPU](https://www.marvell.com/products/data-processing-units.html)
- [Intel Tofino / P4 Studio](https://www.intel.com/content/www/us/en/products/network-io/programmable-ethernet-switch.html)
- [SR-IOV — PCI-SIG specification](https://pcisig.com/specifications)
- [vhost-user protocol specification (QEMU docs)](https://qemu-project.gitlab.io/qemu/interop/vhost-user.html)
- [virtio — paravirtualized I/O standard (OASIS spec)](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html)
- [GPUDirect RDMA — NVIDIA developer documentation](https://developer.nvidia.com/gpudirect)
- [IPsec — RFC 4301 (IETF)](https://www.rfc-editor.org/rfc/rfc4301)
- [NVIDIA SHARP — in-network computing](https://developer.nvidia.com/networking/sharp)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).