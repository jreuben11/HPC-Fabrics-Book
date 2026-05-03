# Chapter 7 — eBPF & XDP

**Part II: Kernel-Bypass & Programmable I/O** | ~25 pages

---

## Introduction

The **Linux** kernel processes every packet traversing a host's network stack through a series of abstractions — socket buffers, **netfilter** hooks, the routing table — that were designed for generality, not for the line-rate demands of AI cluster networking. **eBPF** (**extended Berkeley Packet Filter**) and **XDP** (**eXpress Data Path**) break this constraint by embedding a safe, **JIT**-compiled virtual machine directly inside the kernel, allowing custom programs to intercept, inspect, modify, or drop packets at the earliest possible moment: inside the NIC driver itself.

For AI cluster operators, **eBPF** provides capabilities that were previously accessible only through kernel modules or dedicated **DPDK** processes: wire-rate packet filtering, zero-copy delivery to user space via **AF_XDP**, and low-overhead tracing of any kernel or user-space function without modifying source code. These properties are directly useful for monitoring **NCCL** collective communication flows, debugging **RDMA** latency anomalies, enforcing network policy in **Kubernetes** (via **Cilium**, covered in Chapter 12), and protecting management interfaces from external traffic at driver speed.

This chapter teaches the **eBPF** programming model from first principles: the **BPF** virtual machine instruction set, the verifier's safety guarantees, the map types used for state and communication, and the **libbpf** **CO-RE** toolchain that makes programs portable across kernel versions. **XDP** attachment modes, the **AF_XDP** zero-copy socket interface, and the **TC** (**Traffic Control**) hook are each explained with working code. The chapter concludes with the **Katran** case study — Meta's production **XDP** load balancer — which shows what is achievable at hyperscale.

The lab walkthrough builds a complete **XDP** pipeline from scratch: a per-protocol packet counter, then a dynamic IP blocklist with drop-at-driver-speed capability. Every step is verified with `bpftool`, which serves as the primary debugger for **BPF** state throughout the chapter.

This chapter connects backward to Chapter 5 (**DPDK**) — both technologies pursue kernel-bypass performance, but **XDP**/**AF_XDP** achieves it while remaining within the kernel's NIC driver model. It connects forward to Chapter 12 (**Cilium**) and Chapter 13 (**Multus**/**SR-IOV**), where **eBPF** and **TC** programs form the programmable data plane of **Kubernetes**-based AI clusters.

---

## Installation

**Clang** and **LLVM** are required to compile C source files into **BPF** bytecode — the kernel's own C compiler cannot target the **BPF** instruction set architecture. The `libbpf-dev` package provides the userspace API for loading verified **BPF** objects into the kernel and interacting with **BPF** maps, while `bpftool` serves as the primary debugger for inspecting loaded programs, dumping map contents, and disassembling **JIT**-compiled output. Kernel headers matching the running kernel are necessary because **BPF** programs include kernel struct definitions directly, and `iproute2` supplies the `ip link` and `tc` commands used to attach **XDP** and **TC** programs to network interfaces.

All tools in this chapter require kernel headers, **clang**/**LLVM** for **BPF** compilation, **libbpf** for the userspace API, and **bpftool** for introspection. These packages are available in the **Ubuntu** 24.04 main archive.

### System packages

```bash
sudo apt install -y \
    clang llvm \
    libbpf-dev \
    bpftool \
    linux-headers-$(uname -r) \
    iproute2 \
    tcpdump \
    libelf-dev \
    zlib1g-dev \
    build-essential \
    cmake \
    pkg-config
```

Verify the installed versions:

```bash
clang --version          # expect clang 17+
llvm-strip --version     # same LLVM version
bpftool version          # expect v7+
uname -r                 # confirm kernel ≥ 5.15 (Ubuntu 24.04 ships 6.8)
```

### Python scripting environment (uv)

Install `uv` if you want to run the companion Python scripts that parse map output or drive test traffic. `pyroute2` is a Python netlink library for managing routes, interfaces, and TC rules programmatically; `scapy` is a packet manipulation library for crafting, sending, and parsing arbitrary network packets at the protocol level:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env    # or restart your shell

# Create and activate a project venv
uv venv .venv
source .venv/bin/activate

# Install helper libraries
uv pip install pyroute2 scapy
```

### CMake project scaffold for eBPF/XDP

Save the following as `CMakeLists.txt` in your project root. It compiles the BPF kernel object with clang and the userspace loader with gcc.

```cmake
cmake_minimum_required(VERSION 3.20)
project(xdp_example C)
find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBBPF REQUIRED libbpf)

# BPF kernel program — compiled with clang targeting the BPF ISA
# Resolve the multiarch tuple at configure time (e.g. x86_64-linux-gnu)
execute_process(
  COMMAND dpkg-architecture -qDEB_HOST_MULTIARCH
  OUTPUT_VARIABLE DEB_HOST_MULTIARCH
  OUTPUT_STRIP_TRAILING_WHITESPACE
)
add_custom_command(
  OUTPUT xdp_prog.bpf.o
  COMMAND clang -O2 -target bpf -D__TARGET_ARCH_x86
    -I/usr/include/${DEB_HOST_MULTIARCH}
    -c ${CMAKE_SOURCE_DIR}/xdp_prog.bpf.c -o xdp_prog.bpf.o
  DEPENDS xdp_prog.bpf.c
)
add_custom_target(bpf_prog ALL DEPENDS xdp_prog.bpf.o)

# Userspace loader
add_executable(xdp_loader loader.c)
target_include_directories(xdp_loader PRIVATE ${LIBBPF_INCLUDE_DIRS})
target_link_libraries(xdp_loader ${LIBBPF_LIBRARIES} elf z)
```

Build:

```bash
mkdir build && cd build
cmake ..
make -j$(nproc)
```

---

## 7.1 The eBPF Revolution

eBPF (extended Berkeley Packet Filter) is a virtual machine embedded in the Linux kernel that allows user-supplied programs to run safely in kernel context — without writing a kernel module. Originally for packet filtering, eBPF now spans tracing, security, and networking. It is arguably the most consequential addition to the Linux kernel in the last decade.

For AI cluster networking, eBPF matters at three levels:
1. **XDP:** packet processing at the NIC driver level, before `sk_buff` allocation (`sk_buff` is the kernel's primary packet descriptor structure; allocating one per received packet is a significant source of per-packet overhead) — DPDK-class throughput while staying in the kernel
2. **TC (Traffic Control):** L3/L4 policy enforcement in the kernel fast path — the engine behind Cilium's network policy (Chapter 12)
3. **Tracing:** low-overhead instrumentation of any kernel or user-space function — used for NCCL (NVIDIA Collective Communications Library, the runtime that orchestrates AllReduce and other collective operations across GPUs) performance analysis, RDMA debugging, and storage latency profiling

---

## 7.2 The BPF Virtual Machine

A BPF program is a sequence of instructions executed by the BPF VM. The ISA has:
- 11 general-purpose 64-bit registers (R0–R10)
- A 512-byte stack
- Helper function calls (into the kernel, for map access, time, etc.)
- No unbounded loops (until BPF loop helpers in kernel 5.17+)

The **verifier** statically checks every program before loading:
- All memory accesses are bounds-checked
- No null pointer dereferences
- No uninitialized reads
- No unbounded execution time (loop unrolling or bounded loops required)

This is what makes eBPF safe to run in kernel context without full kernel privileges. After verification, the kernel's BPF JIT (Just-In-Time) compiler translates BPF bytecode into native machine code for the host CPU architecture, delivering performance comparable to compiled C kernel code.

---

## 7.3 BPF Maps

Maps are the primary data structure for eBPF — used for state storage, communication between BPF programs, and user-space ↔ kernel communication:

| Map Type | Description | Use Case |
|---|---|---|
| `BPF_MAP_TYPE_HASH` | Hash table, O(1) avg lookup | Connection tracking, flow tables |
| `BPF_MAP_TYPE_ARRAY` | Fixed-size array, indexed by u32 | Per-CPU counters, configuration |
| `BPF_MAP_TYPE_LRU_HASH` | Hash with LRU eviction | NAT tables, session state |
| `BPF_MAP_TYPE_PERCPU_ARRAY` | Per-CPU array, lock-free | High-frequency counters |
| `BPF_MAP_TYPE_RINGBUF` | Lock-free ring buffer for events | Tracing, packet capture |
| `BPF_MAP_TYPE_PROG_ARRAY` | Array of program FDs for tail calls | Program chaining |
| `BPF_MAP_TYPE_XSKMAP` | XSK socket map for AF_XDP | Kernel→user packet steering |

---

## 7.4 libbpf and CO-RE

`libbpf` is the canonical library for loading BPF programs from user space. **CO-RE** (**Compile Once, Run Everywhere**) enables BPF programs to adapt to kernel struct layout differences at load time, eliminating the need to recompile for each kernel version.

Note: **BPF helper macros** https://man7.org/linux/man-pages/man7/bpf-helpers.7.html , primarily found in `bpf_helpers.h` via libbpf, enable eBPF programs to interact with the Linux kernel. They define program sections (`SEC`), facilitate map creation, handle data types, and manage debugging (`bpf_trace_printk`). Key macros include `bpf_map_lookup_elem`, `bpf_map_update_elem`, and `likely`/`unlikely`. The `__uint` and `__type` macros are part of the libbpf library and are used to define BTF-defined maps in eBPF. They allow you to declare map properties in a way that the BPF loader and kernel can understand via **BPF Type Format (BTF)**.

### Minimal XDP Program with libbpf

**Kernel side (xdp_prog.bpf.c):**
```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, 1);
    __type(key, __u32);
    __type(value, __u64);
} pkt_count SEC(".maps");

SEC("xdp")
int count_packets(struct xdp_md *ctx) {
    __u32 key = 0;
    __u64 *count = bpf_map_lookup_elem(&pkt_count, &key);
    if (count)
        __sync_fetch_and_add(count, 1);
    return XDP_PASS;  // pass packet to kernel stack
}

char LICENSE[] SEC("license") = "GPL";
```

**User side (loader.c):**
```c
#include <bpf/libbpf.h>

int main() {
    // Open, load, and verify BPF object
    struct bpf_object *obj = bpf_object__open("xdp_prog.bpf.o");
    bpf_object__load(obj);

    // Get program FD and attach to interface
    struct bpf_program *prog = bpf_object__find_program_by_name(obj, "count_packets");
    int prog_fd = bpf_program__fd(prog);

    int ifindex = if_nametoindex("eth0");
    bpf_xdp_attach(ifindex, prog_fd, XDP_FLAGS_DRV_MODE, NULL);

    // Read counter from map
    int map_fd = bpf_object__find_map_fd_by_name(obj, "pkt_count");
    __u64 total = 0, val;
    for (int cpu = 0; cpu < num_cpus; cpu++) {
        bpf_map_lookup_elem(map_fd, &(int){0}, &val);  // simplified
        total += val;
    }
    printf("Packets seen: %llu\n", total);
}
```

**Compile:**
```bash
clang -O2 -target bpf -c xdp_prog.bpf.c -o xdp_prog.bpf.o
```

---

## 7.5 XDP — eXpress Data Path

XDP is a hook point at the earliest stage of packet reception — inside the NIC driver, before `sk_buff` allocation. An XDP program can take one of four actions:

| Action | Effect |
|---|---|
| `XDP_PASS` | Continue to kernel network stack |
| `XDP_DROP` | Drop the packet immediately |
| `XDP_TX` | Transmit the packet back out the same interface |
| `XDP_REDIRECT` | Redirect to another interface, CPU, or AF_XDP socket |

### XDP Attachment Modes

```bash
# Native XDP — runs inside the driver, before sk_buff allocation (fastest)
ip link set dev eth0 xdp obj xdp_prog.bpf.o sec xdp

# Generic XDP — runs after sk_buff allocation (slower, universal fallback)
ip link set dev eth0 xdpgeneric obj xdp_prog.bpf.o sec xdp

# Offloaded XDP — program compiled and run on NIC ASIC (fastest, limited support)
ip link set dev eth0 xdpoffload obj xdp_prog.bpf.o sec xdp
```

### XDP for DDoS Mitigation in AI Clusters

AI cluster management networks are exposed to external traffic. XDP-based filtering drops attack traffic at driver speed (wire rate) before it can affect the kernel stack:

```c
SEC("xdp")
int block_syn_flood(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) return XDP_PASS;
    if (eth->h_proto != bpf_htons(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) return XDP_PASS;
    if (ip->protocol != IPPROTO_TCP) return XDP_PASS;

    struct tcphdr *tcp = (void *)ip + (ip->ihl * 4);
    if ((void *)(tcp + 1) > data_end) return XDP_PASS;

    // Drop SYN packets from blocked source ranges (checked via map)
    __u32 src_ip = ip->saddr;
    if (bpf_map_lookup_elem(&blocklist, &src_ip))
        return XDP_DROP;

    return XDP_PASS;
}
```

---

## 7.6 AF_XDP — Zero-Copy Packet Delivery to User Space

AF_XDP allows XDP to steer packets directly to a user-space socket, bypassing the kernel stack entirely — similar to DPDK but reusing the kernel NIC driver:

```
NIC DMA ring
     │ XDP_REDIRECT → XSK socket
User-space UMEM (huge-page backed shared ring)
     │
Application (reads packets directly from ring)
```

**UMEM** is a contiguous region of user-space memory, typically backed by huge pages, that is registered with the kernel and shared between the NIC's DMA engine and the application. The NIC writes received packets directly into UMEM buffers; the application reads them without any kernel-to-user copy. An **XSK (XDP Socket)** is the AF_XDP socket type that connects the XDP program running in kernel space to this user-space UMEM region.

```c
// User space: create XSK socket
struct xsk_socket_config cfg = {
    .rx_size = 2048, .tx_size = 2048,
    .libxdp_flags = 0, .xdp_flags = XDP_FLAGS_DRV_MODE,
    .bind_flags = XDP_COPY,
};
struct xsk_socket *xsk;
xsk_socket__create(&xsk, "eth0", 0, umem, &rx, &tx, &cfg);

// Receive loop
while (1) {
    unsigned int rcvd = xsk_ring_cons__peek(&rx, 64, &idx);
    for (int i = 0; i < rcvd; i++) {
        const struct xdp_desc *desc = xsk_ring_cons__rx_desc(&rx, idx + i);
        void *pkt = xsk_umem__get_data(buffer, desc->addr);
        // process pkt directly from UMEM — zero copy
    }
    xsk_ring_cons__release(&rx, rcvd);
}
```

AF_XDP is used by Cilium for its XDP-accelerated load balancer path and by DPDK as an alternative to VFIO-based PMDs on kernels where VFIO is unavailable.

---

## 7.7 TC Hook — Traffic Control eBPF

The TC (Traffic Control) hook runs after `sk_buff` allocation, providing access to the full packet including metadata not available at XDP. TC programs are attached to ingress and egress qdiscs. A **qdisc (queuing discipline)** is the kernel abstraction that controls how packets are queued and scheduled on a network interface; `clsact` is a special pseudo-qdisc that provides ingress and egress classifier attachment points for eBPF programs without introducing actual queuing:

```bash
# Attach TC eBPF program to ingress
tc qdisc add dev eth0 clsact
tc filter add dev eth0 ingress bpf obj tc_prog.bpf.o sec tc/ingress direct-action

# Attach to egress
tc filter add dev eth0 egress bpf obj tc_prog.bpf.o sec tc/egress direct-action
```

TC programs can return:
- `TC_ACT_OK` — pass to next filter
- `TC_ACT_SHOT` — drop
- `TC_ACT_REDIRECT` — redirect to another interface

Cilium uses TC egress programs for **SNAT** and Kubernetes Service load balancing.

---

## 7.8 bpftool — Inspecting BPF State

bpftool is the primary command-line utility for inspecting, managing, and debugging eBPF programs and maps within the Linux kernel. It allows users to list loaded programs, pin objects to the BPF virtual filesystem, dump BTF metadata, and generate skels. It is developed alongside the Linux kernel, acting as a key tool for BPF observability.

```bash
# List all loaded BPF programs
bpftool prog list

# Show program bytecode (disassembly)
bpftool prog dump xlated id 42

# Show JIT-compiled machine code
bpftool prog dump jited id 42

# List all maps
bpftool map list

# Dump map contents
bpftool map dump id 7

# Pin a program to the BPF filesystem
bpftool prog pin id 42 /sys/fs/bpf/my_prog

# Show XDP programs attached to interfaces
bpftool net list
```

---

## 7.9 Katran — Meta's XDP Load Balancer

Katran is Meta's production L4 load balancer, built on XDP and used at massive scale. It demonstrates what's possible with XDP:

- **Sub-microsecond packet forwarding** at line rate on commodity Mellanox NICs
- **Consistent hashing** for connection affinity, implemented entirely in BPF maps
- **IPIP encapsulation** (**IP-in-IP**: wrapping the original packet in an outer IP header addressed to the real server, a common load-balancer forwarding technique that avoids the need to modify the original destination IP) to forward packets to real servers — implemented in XDP with direct packet rewrite
- **No separate hardware load balancer** — runs on standard servers

Key design: the XDP program does a hash lookup of the destination VIP in a BPF map, selects a real server, rewrites the outer IP header, and redirects via `XDP_TX`. No kernel TCP/IP involvement.

---

## 7.10 netfilter, iptables, nftables & bpfilter: The Packet Filter Stack

Before eBPF rewrote the rules, every packet entering or leaving a Linux host passed through **netfilter** — the kernel framework that has anchored Linux firewalling since kernel 2.4. Understanding netfilter and its tooling clarifies why Cilium's eBPF-native data plane (Chapter 12) is architecturally necessary at AI cluster scale, and where the legacy stack still appears even in eBPF-first deployments.

### netfilter and its Hook Points

**netfilter** defines five hook points at which kernel subsystems can register callback functions to inspect or modify packets in flight:

| Hook | Traffic type |
|---|---|
| `NF_INET_PRE_ROUTING` | All inbound packets, before routing decision |
| `NF_INET_LOCAL_IN` | Packets destined for a local socket |
| `NF_INET_FORWARD` | Packets being forwarded between interfaces |
| `NF_INET_LOCAL_OUT` | Locally generated outbound packets |
| `NF_INET_POST_ROUTING` | All outbound packets, after routing decision |

Both **iptables** and **nftables** are implemented as netfilter hook registrations; they share the same five attachment points. **conntrack** (connection tracking) is also a netfilter subsystem, maintaining per-flow state that DNAT/SNAT and stateful firewall rules depend on.

### iptables: The Classic Tool and Its Limits

**iptables** organises rules into *tables* (`filter`, `nat`, `mangle`, `raw`) each containing built-in *chains* (PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING) and user-defined chains. A packet traverses each chain's rule list sequentially until a terminating target (`ACCEPT`, `DROP`, `DNAT`, …) is hit.

```bash
# List all rules with line numbers and packet/byte counters
sudo iptables -L -n -v --line-numbers

# Add a rule to drop all traffic from an IP
sudo iptables -I INPUT -s 192.0.2.1 -j DROP

# Save the current ruleset (iptables-nft backend on Ubuntu 24.04)
sudo iptables-save > /etc/iptables/rules.v4
```

The scalability problems are structural:

- **O(n) linear scan** — every packet walks the entire chain until a match; 10,000 DNAT rules means 10,000 comparisons in the worst case.
- **Per-packet `xt_*` module overhead** — each match extension (`xt_multiport`, `xt_conntrack`, …) is called as a function pointer per rule per packet.
- **Non-atomic ruleset updates** — `iptables-restore` replaces the full ruleset in a single `setsockopt`, causing a brief window where the kernel holds no rules.
- **conntrack table pressure** — every new Service endpoint in Kubernetes adds DNAT entries; conntrack can become a bottleneck under high connection rates.

On Ubuntu 24.04, `/usr/sbin/iptables` is `iptables-nft` — it uses the nftables kernel API under the hood, not the legacy `xt_*` path. The older `iptables-legacy` binary (using `xt_*` directly) is still available but non-default. **The critical rule:** never mix `iptables-legacy` and `nft`/`iptables-nft` on the same host — they write to separate kernel hook chains and will produce inconsistent state.

```bash
# Audit which backend is active
sudo update-alternatives --query iptables
# On Ubuntu 24.04: /usr/sbin/iptables -> /usr/sbin/iptables-nft

# View legacy xt_* rules (if any were written by older tooling)
sudo iptables-legacy-save

# View the nftables ruleset (covers iptables-nft and native nft rules)
sudo nft list ruleset
```

### nftables: The Modern Replacement

**nftables** (kernel 3.13+, CLI: `nft`) replaces iptables with a redesigned rule engine:

- **Set-based matching** — IP sets, port ranges, and connection states are stored as kernel hash tables or bitmaps, giving O(log n) or O(1) lookup instead of linear scans.
- **Compiled bytecode VM** — rules are compiled to a compact bytecode representation at load time, eliminating the per-rule function-pointer calls that `xt_*` modules require.
- **Atomic ruleset replacement** — `nft -f ruleset.nft` applies an entire ruleset as a single atomic transaction; there is no partial-state window.

```bash
# List the full active ruleset (equivalent of iptables -L for all tables)
sudo nft list ruleset

# Apply a complete ruleset file atomically
sudo nft -f /etc/nftables.conf

# Create a simple host firewall — accept established, drop new inbound
sudo nft add table inet filter
sudo nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input iif lo accept
```

kube-proxy gained an nftables backend (KEP-3866, GA in Kubernetes 1.33) that uses nftables sets for Service routing, reducing per-packet work compared to the iptables backend. For new clusters not yet migrated to Cilium, the nftables kube-proxy mode is the recommended intermediate step.

### bpfilter: An Abandoned Translation Layer

**bpfilter** was an experimental kernel subsystem (introduced in 4.18) designed to translate iptables rules into BPF bytecode in-kernel via a user-mode helper process. The premise was appealing — keep the familiar `iptables` CLI while gaining BPF's performance — but the implementation stalled: no production user ever shipped it, the user-mode helper design proved fragile, and active development ceased.

**bpfilter was removed from the kernel tree in Linux 6.8** (the same version Ubuntu 24.04 ships). The project continues as a standalone user-space library at `github.com/facebook/bpfilter`, but it is not production-ready and is not a recommended deployment path. The lesson bpfilter's failure teaches is the same lesson Cilium's success confirms: translating legacy rule models into BPF is the wrong abstraction. A purpose-built eBPF data plane that replaces the rule model entirely — rather than emulating it — is what delivers the performance gains.

### How Cilium Replaces kube-proxy

Cilium (Chapter 12) eliminates iptables DNAT chains for Kubernetes Service load balancing entirely, replacing them with BPF maps looked up in the TC hook (see §7.7). The data structures are:

- **`cilium_lb4_services_v2`** — maps a Service VIP + port to a service entry including backend count and load-balancing algorithm.
- **`cilium_lb4_backends_v3`** — maps a backend ID to the real pod IP and port.

A packet destined for a ClusterIP hits the TC ingress hook, which does a single hash map lookup in `cilium_lb4_services_v2` (O(1)), selects a backend from `cilium_lb4_backends_v3` (O(1)), and rewrites the destination in place — no conntrack entry required in **DSR** (**Direct Server Return**) mode. Compare this to kube-proxy's iptables chain: with 500 Services each with 20 endpoints, kube-proxy generates roughly 10,000 DNAT rules traversed linearly. At 10,000+ Service endpoints — the scale of an AI cluster's microservice mesh — the difference between O(1) map lookup and O(n) rule traversal is the difference between microseconds and milliseconds of added latency per packet.

### Comparison

| Criterion | iptables (legacy) | nftables / kube-proxy nft | eBPF / Cilium |
|---|---|---|---|
| Rule lookup complexity | O(n) linear | O(log n) / O(1) with sets | O(1) hash map |
| Atomic ruleset updates | No (full reload) | Yes (`nft -f`) | Yes (map update) |
| Kubernetes Service LB | Yes (DNAT chains) | Yes (1.33+ kube-proxy nft) | Yes — replaces kube-proxy |
| conntrack dependency | Required | Required | Optional (DSR mode) |
| Maintenance status | Legacy / no new features | Active (kernel + kube-proxy) | Active (Cilium 1.x) |

### Guidance

- Use **nftables** (`nft`) for any new host firewall rules; avoid `iptables-legacy` entirely on modern kernels.
- Use **Cilium**'s eBPF data plane for all Kubernetes Service routing and NetworkPolicy; it eliminates the iptables/conntrack bottleneck at the source.
- If running kube-proxy, prefer `--proxy-mode=nftables` on kernel ≥ 5.13 as an intermediate step before full Cilium adoption.
- Audit the current state with `sudo nft list ruleset` (nftables + iptables-nft rules) and `sudo iptables-legacy-save` (legacy xt_* rules) — both commands should be checked when diagnosing firewall conflicts.

---

## Lab Walkthrough 7 — XDP Packet Counter to Load Balancer

This walkthrough builds an XDP-based packet processing pipeline in five stages, each adding capability on top of the previous one. You will write real C source files, compile them, attach them to a local test interface, and verify behavior at each step using `bpftool` and standard Linux networking tools. All steps run on a single Ubuntu 24.04 host — no extra hardware required.

**Prerequisite check:**

```bash
# Confirm all required tools are installed
which clang bpftool ip tc ping
clang --version | head -1
# Expected: clang version 17.0.6 (or newer)

bpftool version
# Expected: bpftool v7.3.0 (or newer)

uname -r
# Expected: 6.8.0-xx-generic  (Ubuntu 24.04 default)

# Confirm BPF JIT is enabled (required for per-CPU maps to work correctly)
cat /proc/sys/net/core/bpf_jit_enable
# Expected: 1
# If 0, enable it:
sudo sysctl -w net.core.bpf_jit_enable=1
```

---

### Step 1 — Create a veth pair as a test interface

Rather than attaching XDP to your physical NIC, use a virtual ethernet pair. `veth0` is where you attach the XDP program; `veth1` is the peer where test traffic enters.

```bash
# Create the veth pair
sudo ip link add veth0 type veth peer name veth1

# Bring both ends up
sudo ip link set veth0 up
sudo ip link set veth1 up

# Assign an address to veth1 so you can send ping traffic
sudo ip addr add 192.168.99.1/24 dev veth0
sudo ip addr add 192.168.99.2/24 dev veth1

# Verify
ip link show type veth
# Expected output (abbreviated):
# 5: veth1@veth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
# 6: veth0@veth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...

ip addr show veth0
# Expected: inet 192.168.99.1/24 ...
```

Note the ifindex of `veth0` — you will need it in the loader:

```bash
cat /sys/class/net/veth0/ifindex
# Example output: 6
```

---

### Step 2 — Write the XDP counter program (xdp_prog.bpf.c)

This kernel-side program counts packets by L4 protocol (TCP, UDP, ICMP, other) using a per-CPU array map. Per-CPU maps avoid atomic operations — each CPU updates its own slot, and user space aggregates.

Create the file `xdp_prog.bpf.c`:

```c
// xdp_prog.bpf.c — XDP per-protocol packet counter
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/in.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

// Protocol index constants stored in the map
#define PROTO_TCP   0
#define PROTO_UDP   1
#define PROTO_ICMP  2
#define PROTO_OTHER 3
#define PROTO_MAX   4

// Per-CPU array: key = protocol index (0..3), value = packet count
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, PROTO_MAX);
    __type(key, __u32);
    __type(value, __u64);
} pkt_count SEC(".maps");

SEC("xdp")
int count_packets(struct xdp_md *ctx)
{
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    // Parse Ethernet header
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    // Only process IPv4
    if (bpf_ntohs(eth->h_proto) != ETH_P_IP)
        return XDP_PASS;

    // Parse IP header
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    // Choose the bucket based on IP protocol field
    __u32 key;
    switch (ip->protocol) {
    case IPPROTO_TCP:  key = PROTO_TCP;   break;
    case IPPROTO_UDP:  key = PROTO_UDP;   break;
    case IPPROTO_ICMP: key = PROTO_ICMP;  break;
    default:           key = PROTO_OTHER; break;
    }

    // Increment the per-CPU counter — no lock needed
    __u64 *cnt = bpf_map_lookup_elem(&pkt_count, &key);
    if (cnt)
        (*cnt)++;

    return XDP_PASS;
}

char LICENSE[] SEC("license") = "GPL";
```

---

### Step 3 — Compile the BPF object with clang

The `-target bpf` flag instructs clang to emit BPF bytecode instead of x86 machine code. The `-O2` flag is mandatory — the BPF verifier rejects some unoptimised patterns.

```bash
# Find the correct include path for your architecture
ARCH_INC=$(dpkg-architecture -qDEB_HOST_MULTIARCH)
echo "Include path: /usr/include/${ARCH_INC}"
# Example: /usr/include/x86_64-linux-gnu

# Compile
clang -O2 -target bpf \
    -I/usr/include/${ARCH_INC} \
    -c xdp_prog.bpf.c \
    -o xdp_prog.bpf.o

# Verify the output is a BPF ELF object
file xdp_prog.bpf.o
# Expected: xdp_prog.bpf.o: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), not stripped

# Inspect the sections — you should see xdp, .maps, and license
llvm-objdump -h xdp_prog.bpf.o
# Expected sections include:
#   xdp         (the program)
#   .maps       (the map definition)
#   license
```

---

### Step 4 — Write the userspace loader (loader.c)

The loader opens the BPF object, loads it into the kernel (triggering the verifier), attaches the XDP program to `veth0`, then reads back per-CPU map values every second.

Create `loader.c`:

```c
// loader.c — libbpf userspace loader for xdp_prog.bpf.o
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <net/if.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <linux/bpf.h>

#define PROTO_MAX 4
static const char *proto_names[] = {"TCP", "UDP", "ICMP", "OTHER"};

int main(int argc, char **argv)
{
    const char *iface = (argc > 1) ? argv[1] : "veth0";

    // 1. Open the BPF ELF object — does NOT load into kernel yet
    struct bpf_object *obj = bpf_object__open("xdp_prog.bpf.o");
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "ERROR: bpf_object__open failed\n");
        return 1;
    }

    // 2. Load — runs the verifier, JIT-compiles, creates maps
    if (bpf_object__load(obj)) {
        fprintf(stderr, "ERROR: bpf_object__load failed\n");
        return 1;
    }
    printf("BPF object loaded and verified successfully.\n");

    // 3. Find the XDP program by name
    struct bpf_program *prog =
        bpf_object__find_program_by_name(obj, "count_packets");
    if (!prog) {
        fprintf(stderr, "ERROR: program 'count_packets' not found\n");
        return 1;
    }
    int prog_fd = bpf_program__fd(prog);

    // 4. Resolve interface index and attach in generic XDP mode
    //    (use XDP_FLAGS_DRV_MODE on real NICs with native XDP support)
    unsigned int ifindex = if_nametoindex(iface);
    if (!ifindex) {
        fprintf(stderr, "ERROR: interface '%s' not found\n", iface);
        return 1;
    }
    if (bpf_xdp_attach(ifindex, prog_fd, XDP_FLAGS_UPDATE_IF_NOEXIST, NULL)) {
        fprintf(stderr, "ERROR: bpf_xdp_attach failed\n");
        return 1;
    }
    printf("XDP program attached to %s (ifindex=%u)\n", iface, ifindex);

    // 5. Find the map FD
    int map_fd = bpf_object__find_map_fd_by_name(obj, "pkt_count");
    if (map_fd < 0) {
        fprintf(stderr, "ERROR: map 'pkt_count' not found\n");
        return 1;
    }

    // 6. Determine the number of online CPUs
    int num_cpus = libbpf_num_possible_cpus();
    printf("Aggregating counters across %d CPUs. Ctrl-C to stop.\n\n", num_cpus);

    // 7. Poll and print counters every second
    __u64 *per_cpu_vals = calloc(num_cpus, sizeof(__u64));
    while (1) {
        printf("\033[2J\033[H");  // clear screen
        printf("%-8s %12s\n", "Protocol", "Packets");
        printf("%-8s %12s\n", "--------", "-------");
        for (__u32 key = 0; key < PROTO_MAX; key++) {
            memset(per_cpu_vals, 0, num_cpus * sizeof(__u64));
            bpf_map_lookup_elem(map_fd, &key, per_cpu_vals);
            __u64 total = 0;
            for (int cpu = 0; cpu < num_cpus; cpu++)
                total += per_cpu_vals[cpu];
            printf("%-8s %12llu\n", proto_names[key], total);
        }
        sleep(1);
    }

    free(per_cpu_vals);
    return 0;
}
```

Compile the loader:

```bash
gcc -O2 loader.c -o xdp_loader \
    $(pkg-config --cflags --libs libbpf) \
    -lelf -lz

# Verify it linked correctly
ldd xdp_loader | grep bpf
# Expected: libbpf.so.1 => /lib/x86_64-linux-gnu/libbpf.so.1
```

Alternatively, use the CMake build system from the Installation section:

```bash
mkdir -p build && cd build
cmake ..
make -j$(nproc)
cd ..
```

---

### Step 5 — Attach the program and verify with bpftool

Run the loader as root (XDP attach requires `CAP_NET_ADMIN`):

```bash
sudo ./xdp_loader veth0
# Expected output:
# BPF object loaded and verified successfully.
# XDP program attached to veth0 (ifindex=6)
# Aggregating counters across 4 CPUs. Ctrl-C to stop.
#
# Protocol      Packets
# --------      -------
# TCP                 0
# UDP                 0
# ICMP                0
# OTHER               0
```

In a second terminal, generate some test traffic:

```bash
# Send 5 ICMP echo requests
ping -c 5 192.168.99.2

# Send UDP packets with nc
echo "hello" | nc -u 192.168.99.2 9999 &

# Send a TCP SYN (will be rejected, but XDP sees it)
curl --max-time 1 http://192.168.99.2/ 2>/dev/null || true
```

Watch the counters increment in the loader window. You should see ICMP count increase by 5.

Now verify attachment independently using `bpftool`:

```bash
# List all XDP programs on all interfaces
sudo bpftool net list
# Expected output:
# xdp:
# veth0(6) driver id 42

# Show program details by the ID reported above
sudo bpftool prog show id 42
# Expected:
# 42: xdp  name count_packets  tag 3b185187f1855c4c  gpl
#         loaded_at 2026-04-22T10:15:00+0000  uid 0
#         xlated 192B  jited 128B  memlock 4096B  map_ids 5

# List all maps
sudo bpftool map list
# Expected (among others):
# 5: percpu_array  name pkt_count  flags 0x0
#         key 4B  value 8B  max_entries 4  memlock 4096B

# Dump raw map contents (per-CPU values for each key)
sudo bpftool map dump id 5
# Expected (example with some traffic):
# key: 00 00 00 00  value (CPU 00): 00 00 00 00 00 00 00 00
#                  value (CPU 01): 05 00 00 00 00 00 00 00   <- 5 ICMP on CPU 1
#                  value (CPU 02): 00 00 00 00 00 00 00 00
#                  value (CPU 03): 00 00 00 00 00 00 00 00
# key: 01 00 00 00  ...
# (key 0=TCP, 1=UDP, 2=ICMP, 3=OTHER)

# Inspect the disassembled (verifier-translated) bytecode
sudo bpftool prog dump xlated id 42
# Expected: human-readable BPF instructions, e.g.:
#    0: (61) r1 = *(u32 *)(r1 +4)
#    1: (61) r2 = *(u32 *)(r1 +0)
#   ...

# Inspect the JIT-compiled x86 machine code
sudo bpftool prog dump jited id 42
# Expected: x86_64 disassembly of the JIT output
```

---

### Step 6 — Extend to XDP_DROP with a blocklist map

Now modify the program to add a hash map that acts as a blocklist of source IPs. Any IP in the map will be dropped at driver speed.

Create `xdp_drop.bpf.c`:

```c
// xdp_drop.bpf.c — XDP packet counter + IP blocklist
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/in.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

#define PROTO_TCP   0
#define PROTO_UDP   1
#define PROTO_ICMP  2
#define PROTO_OTHER 3
#define PROTO_MAX   4

struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, PROTO_MAX);
    __type(key, __u32);
    __type(value, __u64);
} pkt_count SEC(".maps");

// Blocklist: key = IPv4 address (network byte order), value = 1 (present)
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, __u32);
    __type(value, __u8);
} blocklist SEC(".maps");

// Drop counter
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, 1);
    __type(key, __u32);
    __type(value, __u64);
} drop_count SEC(".maps");

SEC("xdp")
int filter_packets(struct xdp_md *ctx)
{
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;
    if (bpf_ntohs(eth->h_proto) != ETH_P_IP)
        return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end)
        return XDP_PASS;

    // Check blocklist — drop if source IP is in the map
    __u32 src = ip->saddr;
    if (bpf_map_lookup_elem(&blocklist, &src)) {
        __u32 zero = 0;
        __u64 *dc = bpf_map_lookup_elem(&drop_count, &zero);
        if (dc)
            (*dc)++;
        return XDP_DROP;
    }

    // Count by protocol
    __u32 key;
    switch (ip->protocol) {
    case IPPROTO_TCP:  key = PROTO_TCP;   break;
    case IPPROTO_UDP:  key = PROTO_UDP;   break;
    case IPPROTO_ICMP: key = PROTO_ICMP;  break;
    default:           key = PROTO_OTHER; break;
    }
    __u64 *cnt = bpf_map_lookup_elem(&pkt_count, &key);
    if (cnt)
        (*cnt)++;

    return XDP_PASS;
}

char LICENSE[] SEC("license") = "GPL";
```

Compile the new program:

```bash
clang -O2 -target bpf \
    -I/usr/include/$(dpkg-architecture -qDEB_HOST_MULTIARCH) \
    -c xdp_drop.bpf.c \
    -o xdp_drop.bpf.o

file xdp_drop.bpf.o
# Expected: ELF 64-bit LSB relocatable, eBPF ...
```

Detach the previous XDP program, then attach the new one:

```bash
# Detach the current XDP program from veth0
sudo ip link set veth0 xdp off

# Verify it is gone
sudo bpftool net list
# Expected: xdp: (empty)

# Attach the new program using ip link (convenient for testing)
sudo ip link set dev veth0 xdpgeneric obj xdp_drop.bpf.o sec xdp

sudo bpftool net list
# Expected:
# xdp:
# veth0(6) generic id 55

sudo bpftool prog show id 55
# Expected:
# 55: xdp  name filter_packets  tag ...  gpl
```

Now populate the blocklist map and test XDP_DROP:

```bash
# Get the map ID for the blocklist
sudo bpftool map list
# Look for: hash  name blocklist  ...  id 8

# Add 192.168.99.2 to the blocklist
# IPv4 address in hex, little-endian: 192.168.99.2 = c0.a8.63.02
# In network (big-endian) byte order as stored: 02 63 a8 c0
sudo bpftool map update id 8 \
    key hex 02 63 a8 c0 \
    value hex 01

# Verify the entry was added
sudo bpftool map dump id 8
# Expected:
# key: 02 63 a8 c0  value: 01

# Test: ping from veth1 side (src=192.168.99.2, which is now blocked)
ping -c 3 -I veth1 192.168.99.1
# Expected: 100% packet loss — XDP drops the packets before the kernel sees them

# Confirm drop counter incremented
# Get drop_count map ID (percpu_array name drop_count)
sudo bpftool map list
# Note the id for drop_count, e.g. id 9
sudo bpftool map dump id 9
# Expected: non-zero value on one of the CPUs

# Remove from blocklist and confirm ping recovers
sudo bpftool map delete id 8 key hex 02 63 a8 c0
ping -c 3 -I veth1 192.168.99.1
# Expected: 3 packets transmitted, 3 received, 0% packet loss
```

---

### Step 7 — Verify with bpftool prog show and xlated dump

After attaching `xdp_drop.bpf.o`, use bpftool to inspect what the verifier accepted and how the JIT compiler translated the program:

```bash
# Full program info
sudo bpftool prog show id 55 --pretty
# Expected JSON output showing:
# {
#   "id": 55,
#   "type": "xdp",
#   "name": "filter_packets",
#   "tag": "...",
#   "gpl_compatible": true,
#   "map_ids": [7, 8, 9],
#   "bytes_xlated": 320,
#   "jited": true,
#   "bytes_jited": 216,
#   ...
# }

# Disassemble the verifier-translated bytecode
sudo bpftool prog dump xlated id 55
# Expected (first few instructions):
# int filter_packets(struct xdp_md * ctx):
#    0: (61) r2 = *(u32 *)(r1 +4)       ; data_end = ctx->data_end
#    1: (61) r1 = *(u32 *)(r1 +0)       ; data = ctx->data
#    2: (bf) r3 = r1
#    3: (07) r3 += 14                    ; eth + 1
#    4: (2d) if r3 > r2 goto pc+NN      ; bounds check
# ...

# Show the JIT-compiled x86_64 code
sudo bpftool prog dump jited id 55 linum
# This shows machine code with line number annotations

# Verify all three maps are referenced
sudo bpftool prog show id 55 | grep map_ids
# Expected: map_ids 7,8,9
```

---

### Step 8 — Detach and verify the interface returns to normal

```bash
# Detach the XDP program
sudo ip link set veth0 xdp off

# Confirm no XDP programs remain
sudo bpftool net list
# Expected:
# xdp:
# (empty — no programs listed)

ip link show veth0
# Expected: the xdp line is absent from the interface output
# <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue ...
# NOT:
# <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp ...

# Confirm kernel processes ping normally again
ping -c 2 192.168.99.2
# Expected: 2 packets transmitted, 2 received, 0% packet loss

# Confirm no BPF programs remain loaded (unless systemd/cilium loaded others)
sudo bpftool prog list | grep xdp
# Expected: (empty)

# Clean up the veth pair
sudo ip link delete veth0
ip link show type veth
# Expected: (empty — both veth0 and veth1 are gone)
```

---

## Summary

- eBPF brings safe, JIT-compiled programmability to the Linux kernel without kernel modules; the verifier enforces safety at load time.
- XDP runs at the NIC driver level — before `sk_buff` — enabling line-rate packet processing with simple C programs.
- AF_XDP redirects XDP packets to user space with zero copy, bridging the gap between kernel eBPF and DPDK user-space packet processing.
- TC hook programs run after `sk_buff` allocation, with access to full packet context; they power Cilium's Kubernetes data plane.
- `bpftool` is the indispensable debugger for all BPF state.

---

## References

- [eBPF.io — official eBPF documentation hub](https://ebpf.io)
- [libbpf — canonical userspace BPF loading library](https://libbpf.readthedocs.io)
- [libbpf GitHub repository](https://github.com/libbpf/libbpf)
- [bpftool — BPF object introspection utility](https://github.com/libbpf/bpftool)
- [XDP tutorial (xdp-project)](https://github.com/xdp-project/xdp-tutorial)
- [AF_XDP — kernel documentation](https://www.kernel.org/doc/html/latest/networking/af_xdp.html)
- [Katran — Meta's XDP load balancer](https://github.com/facebookincubator/katran)
- [Cilium — eBPF-based Kubernetes CNI](https://cilium.io)
- [bcc — BPF Compiler Collection (iovisor)](https://github.com/iovisor/bcc)
- [clang/LLVM — compiler toolchain required for BPF compilation](https://llvm.org)
- [iproute2 — Linux networking utilities (tc, ip)](https://wiki.linuxfoundation.org/networking/iproute2)
- [pyroute2 — Python netlink library](https://pyroute2.org)
- [Scapy — Python packet manipulation library](https://scapy.net)
- [DPDK — Data Plane Development Kit](https://www.dpdk.org)
- [NCCL — NVIDIA Collective Communications Library](https://github.com/NVIDIA/nccl)
- Hoiland-Jorgensen et al., *The eXpress Data Path*, CoNEXT 2018 — [dl.acm.org/doi/10.1145/3281411.3281443](https://dl.acm.org/doi/10.1145/3281411.3281443)
- Gregg, B., *BPF Performance Tools*, Addison-Wesley 2019 — [brendangregg.com/bpf-performance-tools-book.html](https://www.brendangregg.com/bpf-performance-tools-book.html)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).