# Appendix F — Glossary

This glossary defines key terms used throughout *The Open AI Network Stack*. Terms are listed alphabetically. Where a term has both a general networking meaning and a specific AI/HPC meaning, the AI/HPC context is emphasized. Cross-references to book chapters are included where the term is treated in depth.

Abbreviations and acronyms are cross-listed: if you encounter an acronym whose meaning is not immediately clear, search both the acronym and its expansion. Terms marked *(see also: X)* have closely related glossary entries that provide complementary context.

---

**ACL (Access Control List)** — A list of rules applied to a switch port or router interface that permits or denies traffic based on packet header fields (source/destination IP, port, protocol, DSCP). In AI cluster fabrics, ACLs are used to enforce RDMA priority class isolation, block unauthorized management access, and restrict cross-tenant traffic in multi-tenant deployments. ACL scale (number of supported entries) is an important constraint on shallow-buffer ASICs. See Chapter 8.

**AF_XDP** — Address Family XDP. A Linux socket type that enables zero-copy packet delivery between the kernel XDP layer and a userspace application via a shared memory ring buffer (UMEM). AF_XDP bypasses the kernel networking stack for matched packets, achieving near-DPDK performance without leaving the kernel driver model. See Chapter 7.

**AllReduce** — A collective communication operation in which every participating process contributes an input tensor; the operation reduces (e.g., sums) all inputs and distributes the result back to every process. AllReduce is the fundamental primitive of data-parallel training: each GPU computes a gradient, AllReduce aggregates them across all GPUs, and each GPU updates with the global gradient. See Chapters 19 and 20.

**AR (Adaptive Routing)** — A forwarding strategy in which a switch selects the output port for each packet (or flowlet) dynamically based on current queue occupancy or link utilization rather than a static hash. Adaptive Routing reduces hotspots in fat-tree and dragonfly topologies during AI collective operations where traffic is highly bursty. NVIDIA Spectrum-4 and Quantum-3 implement AR in hardware. See Chapter 19.

**BAR (Base Address Register)** — A PCIe configuration space register that maps a device's memory or I/O regions into the host's physical address space. GPUDirect RDMA relies on BAR mapping to allow a NIC to directly read from or write to GPU memory without CPU involvement. BAR size limits in some platform firmware constrain the amount of GPU memory accessible via peer-to-peer PCIe. See Chapter 2.

**BFD (Bidirectional Forwarding Detection)** — A lightweight hello protocol (RFC 5880) that detects forwarding path failures in milliseconds, much faster than BGP hold timers. BFD runs as a UDP exchange between two endpoints and signals BGP (or other routing protocols) to withdraw routes on failure. In AI fabric BGP designs, aggressive BFD timers (e.g., 100 ms × 3) provide sub-second convergence at the cost of false positives on lossy paths. See Chapter 17.

**BGP (Border Gateway Protocol)** — The inter-domain routing protocol of the internet (RFC 4271), widely adopted inside data centers as an IGP replacement for its simplicity, scalability, and policy flexibility. BGP-EVPN (RFC 7432) extends BGP to carry MAC/IP overlay reachability. In AI cluster fabrics, BGP is used on the underlay for IP reachability and on the overlay for EVPN VXLAN. See Chapters 11 and 17.

**BIRD** — A modular open-source routing daemon for Linux supporting BGP, OSPF, RIP, and BFD. BIRD uses a declarative configuration language and is widely used in containerlab and virtual lab environments. The `birdc` CLI provides runtime introspection. See Chapter 17.

**BMCA (Best Master Clock Algorithm)** — The algorithm defined in IEEE 1588 (PTP) by which clocks in a network elect the Grandmaster Clock. Each clock advertises an Announce message containing clock identity, priority, accuracy class, and variance; BMCA selects the best clock deterministically. See Chapter 3.

**bpftool** — A Linux userspace utility for interacting with the BPF subsystem: loading programs, inspecting maps, dumping JIT-compiled bytecode, and querying program statistics. Used for operational debugging of eBPF-based networking (XDP, TC, Cilium). See Chapter 7.

**BPF VM** — The in-kernel virtual machine that executes BPF (Berkeley Packet Filter) or eBPF programs. The BPF VM provides a register-based ISA with 64-bit registers, a bounded stack, and verifier-enforced safety. The JIT compiler translates BPF bytecode to native code at load time. See Chapter 7.

**CNP (Congestion Notification Packet)** — In RoCEv2 DCQCN congestion control, the CNP is a special-purpose packet sent by a receiver back to the sender when the receiver detects ECN marks (CE bits set) in incoming packets. The sender uses CNPs to trigger rate reduction. CNP rate is a key diagnostic metric: a high rate of CNPs indicates persistent congestion, while zero CNPs on a loaded fabric may indicate ECN is not configured. See Chapter 2 and Appendix G.4.

**CLOS (CLOS Network / Fat-Tree)** — A multi-stage switching architecture, originally described by Charles Clos in 1953, that provides non-blocking connectivity between any input and output port using multiple stages of smaller switches. In AI clusters, a 3-stage CLOS (leaf–spine–super-spine) provides full bisection bandwidth between all GPU servers. The fat-tree topology is a specific realization of a CLOS network using commodity switches. See Chapter 1.

**CNI (Container Network Interface)** — A specification and library for configuring network interfaces in Linux containers. Kubernetes delegates pod networking to CNI plugins; popular AI cluster CNI choices include Cilium (eBPF-based), Flannel (VXLAN-based), and Multus (multi-NIC). See Chapters 12 and 13.

**CXL (Compute Express Link)** — An open interconnect standard built on the PCIe physical layer, providing cache-coherent memory access between CPUs, GPUs, accelerators, and memory expansion devices. CXL 3.0 enables shared memory fabrics across multiple hosts. Relevant to AI clusters for memory pooling and GPU memory expansion. See Chapter 1.

**DAC (Direct Attach Copper)** — A short-reach twinaxial cable with transceiver connectors (QSFP-DD, QSFP56, OSFP) permanently attached. DAC cables carry high-speed electrical signals without optical conversion, providing the lowest cost and power per port for distances up to 3 m (passive) or 7 m (active). The standard first-choice cable for server-to-ToR connections in the same rack. See Appendix E.3.

**DCQCN (Data Center Quantized Congestion Notification)** — The congestion control algorithm standardized for RoCEv2 networks, combining ECN marking at switches with rate reduction at senders (via a quantized update algorithm). DCQCN is implemented in NVIDIA ConnectX hardware and is the primary mechanism for preventing RoCEv2 queues from filling and triggering PFC pause. See Chapter 2.

**DDP (DistributedDataParallel)** — The PyTorch module (`torch.nn.parallel.DistributedDataParallel`) that wraps a model to enable synchronous data-parallel training across multiple GPUs or hosts. DDP uses process groups backed by NCCL (GPU–GPU) or Gloo (CPU) for AllReduce communication. See Chapter 20.

**DPU (Data Processing Unit)** — A category of SmartNIC that includes a full programmable processor complex (typically Arm cores) alongside the network ASIC, enabling offloading of host OS functions (storage, security, network virtualization) to the DPU. NVIDIA BlueField-3 is the leading DPU in AI infrastructure. DPUs run an isolated OS (e.g., Ubuntu) independent of the host. See Chapter 10.

**DSCP (Differentiated Services Code Point)** — A 6-bit field in the IPv4/IPv6 header used to mark traffic for QoS treatment. In RoCEv2 fabrics, DSCP values are mapped to specific 802.1p CoS priorities and switch queues, ensuring RDMA traffic lands in the lossless PFC-protected queue. The common mapping is DSCP 26 (AF31) → CoS 3 → lossless queue. See Chapter 2.

**ECN (Explicit Congestion Notification)** — An IP/TCP extension (RFC 3168) that allows routers to signal congestion to endpoints by setting bits in the IP header rather than dropping packets. In RoCEv2, ECN-capable transport (ECT) bits are set by the sender; switches mark Congestion Experienced (CE) when queue depth exceeds a threshold; the receiver sends a CNP (Congestion Notification Packet) back to the sender to trigger DCQCN rate reduction. See Chapter 2.

**DOM (Digital Optical Monitoring)** — A standard (SFF-8472, SFF-8636) by which pluggable optical transceivers expose real-time diagnostic data (temperature, supply voltage, Tx power, Rx power, laser bias current) over I2C to the host switch or server. DOM data is accessible via `ethtool -m <interface>` on Linux or via NOS-specific CLI commands. Monitoring DOM thresholds is important for proactive identification of degrading optics before they cause packet errors. See Appendix E.3.3 and Appendix G.2.

**DPDK (Data Plane Development Kit)** — A Linux Foundation project providing a set of libraries and drivers for fast packet processing in userspace. DPDK uses poll-mode drivers (PMDs) that bypass the kernel and busy-poll NIC receive queues, achieving sub-100 µs latency for software packet processing. In AI cluster contexts, DPDK is used in OVS-DPDK for virtualized network acceleration, and as the underlying I/O model for SPDK and some storage networking stacks. See Chapter 5.

**eBPF (extended Berkeley Packet Filter)** — A kernel subsystem allowing safe, sandboxed programs to run in the Linux kernel in response to events (packet arrival, syscall, tracepoint, kprobe). eBPF programs are verified for safety, JIT-compiled, and can read/write shared maps. eBPF underpins XDP, TC-based load balancing (Cilium), and observability tools (bpftrace). See Chapter 7.

**ECMP (Equal-Cost Multi-Path)** — A routing technique in which multiple next-hops with equal metric are installed for a destination prefix. Traffic is distributed across all next-hops using a hash of the packet's flow tuple (src IP, dst IP, src port, dst port, protocol). ECMP provides horizontal bandwidth scaling in fat-tree fabrics. ECMP group sizes of 64K+ are supported on modern ASICs. See Chapter 11.

**EFA (Elastic Fabric Adapter)** — AWS's proprietary network interface for high-performance computing and AI workloads on EC2. EFA provides an OS-bypass path using the SRD (Scalable Reliable Datagram) transport protocol, which delivers ordered, reliable transport with multi-path load balancing. AWS Libfabric EFA provider exposes EFA to MPI and NCCL. See Chapter 4.

**EVPN (Ethernet VPN)** — A BGP address family (RFC 7432) that distributes MAC and IP reachability information across a network to build layer-2 overlay services over an IP underlay. EVPN with VXLAN encapsulation is the dominant multi-tenant fabric technology in data centers. EVPN route types 2 (MAC/IP), 3 (multicast), and 5 (IP prefix) are most relevant to AI cluster overlays. See Chapter 11.

**ExaBGP** — A Python-based BGP implementation designed for injecting and monitoring BGP routes programmatically. ExaBGP exposes an API that allows external programs to announce/withdraw prefixes via BGP, making it useful for health-check-driven load balancing and network automation. See Chapter 17.

**FRR (FRRouting)** — A fork of Quagga providing a full suite of open-source routing daemons (BGP, OSPF, IS-IS, BFD, PIM) for Linux. FRR is widely used in SONiC, containerlab, and software routers. Each protocol runs as a separate daemon managed by `zebra`. See Chapter 17.

**Fat-Tree** — See *CLOS*.

**Flowlet** — A burst of packets from the same flow that are separated from the previous burst by an idle period longer than the maximum link delay. Flowlet-based load balancing (e.g., ECMP with flowlet switching) can reassign flowlets to different paths, improving link utilization compared to per-flow static ECMP without causing significant packet reordering. See Chapter 19 (adaptive routing context).

**Gang Scheduling** — A scheduling strategy in which all processes of a distributed job are started (and blocked) simultaneously, ensuring every rank is running at the same time. Gang scheduling eliminates straggler effects in tightly coupled MPI/NCCL jobs where one lagging rank stalls all others at collective barriers. See Chapters 4 and 20.

**gNMI (gRPC Network Management Interface)** — A gRPC-based protocol for network device management and streaming telemetry, defined by OpenConfig. gNMI supports Get, Set, and Subscribe RPCs; Subscribe enables push-based streaming of operational state at configurable intervals. See Chapter 15.

**GoBGP** — A BGP implementation written in Go, designed as an embeddable BGP library and standalone daemon. GoBGP exposes a gRPC API for programmatic route management, making it suitable for SDN controllers and route reflector use cases. See Chapter 17.

**GPUDirect RDMA** — An NVIDIA technology enabling RDMA NIC adapters to read from and write to GPU memory directly, bypassing the CPU and host system memory. GPUDirect RDMA requires `nvidia-peermem` (or equivalent peer memory kernel module) and is critical for high-bandwidth GPU-to-GPU transfers in AI training. See Chapters 2 and 19.

**HCA (Host Channel Adapter)** — The InfiniBand terminology for a network interface card; the IB equivalent of a NIC. An HCA implements the full IB transport stack in hardware, including Queue Pairs, memory registration, and RDMA verbs. NVIDIA ConnectX-7 is a VPI HCA supporting both IB and Ethernet (RoCEv2). See Appendix E.

**HPCC (High Precision Congestion Control)** — A congestion control algorithm for RDMA networks that uses in-band network telemetry (INT) to obtain precise per-hop queue depth and bandwidth utilization, enabling senders to compute exact rate limits without the approximations of ECN-based schemes. HPCC achieves higher utilization than DCQCN on bursty AI workloads. See Chapter 9.

**IBVerbs** — The RDMA Verbs API as exposed by the `libibverbs` library. IBVerbs provides a portable C API for managing RDMA resources (Protection Domains, Memory Regions, Queue Pairs, Completion Queues) across InfiniBand and RoCEv2 hardware. Programs using IBVerbs are portable between IB and RoCE fabrics. See Chapter 2.

**InfiniBand** — A high-performance interconnect standard providing RDMA, low latency (<1 µs), and high bandwidth (NDR: 400 Gb/s per port). InfiniBand uses a credit-based lossless transport and a subnet-based addressing model. The IB subnet manager (SM) handles routing and address assignment. Widely used in HPC and AI training clusters. See Chapter 2.

**INT (In-band Network Telemetry)** — A data-plane mechanism in which switches insert metadata (queue depth, link utilization, egress timestamp, switch ID) directly into packet headers as the packet traverses the network. The collector strips INT headers and reconstructs per-flow, per-hop telemetry. INT is natively supported on Intel Tofino and implementable via P4. See Chapter 9.

**Incast** — A network traffic pattern in which many senders simultaneously send to a single receiver, filling the receiver's port queue faster than it can drain. Incast is endemic to AllReduce operations: in the gather phase, all N GPUs send their gradients to a root, creating N-to-1 incast. At scale, incast is the primary cause of queue overflow and PFC storms in AI cluster fabrics. See Chapters 19 and 20.

**IPoIB (IP over InfiniBand)** — A protocol (RFC 4391) enabling standard IP networking over InfiniBand links. IPoIB allows legacy IP applications to run over IB without modification; performance is lower than native RDMA Verbs. IPoIB is used for management traffic (SSH, NFS) on IB-connected nodes. See Chapter 2.

**IRB (Integrated Routing and Bridging)** — A logical interface that bridges between a layer-2 VLAN domain and a layer-3 routed domain on the same switch. IRB interfaces (also called SVIs — Switched Virtual Interfaces) are used in EVPN VXLAN fabrics for asymmetric and symmetric IRB routing. See Chapter 11.

**Jumbo Frames** — Ethernet frames with an MTU larger than the standard 1500 bytes, typically 9000 bytes (9K jumbo) in data center networks. Jumbo frames are required for RoCEv2 performance: the overhead of IP/UDP/IB headers on a 1500-byte MTU causes fragmentation and significant throughput degradation. A 9000-byte MTU is the minimum recommended for RDMA fabrics; switches must also be configured with an MTU of 9216 or larger to accommodate headers. See Chapter 2.

**KV Store (Key-Value Store)** — In distributed systems and networking contexts, an in-memory or persistent database providing O(1) key lookup. Examples include etcd, Redis, and Consul. In distributed training, KV stores (e.g., the PyTorch `c10d` rendezvous backend) are used for rank discovery and collective group initialization. See Chapter 4.

**LibFabric (OFI)** — See **OFI**.

**libbpf** — The official C library for loading and interacting with eBPF programs from userspace. `libbpf` handles ELF parsing of compiled BPF objects, map creation, program loading, and BTF-based CO-RE (Compile Once, Run Everywhere) relocations. It is the preferred interface over the older `bpf()` syscall wrappers. See Chapter 7.

**libibverbs** — See **IBVerbs**.

**L_Key / R_Key (Local Key / Remote Key)** — Access control tokens used in RDMA operations. An L_Key authorizes a local Queue Pair to access a registered Memory Region for local DMA. An R_Key is sent to a remote peer to authorize remote RDMA Read/Write access to that Memory Region. Key exchange is performed at connection setup; a valid R_Key is sufficient for a remote host to perform RDMA into the associated memory. See Chapter 2.

**LLDP (Link Layer Discovery Protocol)** — An IEEE 802.1AB layer-2 protocol by which network devices advertise their identity, capabilities, and neighbor topology to directly connected peers. LLDP is used in AI cluster fabrics for physical topology discovery, cable verification, and auto-configuration of switch ports. See Chapter 8.

**LPM (Longest Prefix Match)** — The fundamental IP routing lookup algorithm: given a destination address, select the routing table entry whose prefix is the longest (most specific) match. TCAM-based LPM in switch ASICs performs this lookup at line rate. See Chapter 9.

**MOFED (Mellanox OpenFabrics Enterprise Distribution)** — NVIDIA's packaged kernel driver suite for ConnectX and BlueField adapters, bundling the `mlx5` kernel driver, RDMA userspace libraries (`libibverbs`, `librdmacm`), performance tools (`ib_write_bw`, `ib_read_lat`), and diagnostic utilities (`ibstat`, `ibdiagnet`, `perfquery`). MOFED is the production-standard driver stack for NVIDIA HCA deployments. See Chapters 2 and 19.

**MPI (Message Passing Interface)** — The de facto standard API for distributed memory parallel programming in HPC. MPI defines point-to-point (Send/Recv) and collective (AllReduce, Broadcast, Gather) communication operations. OpenMPI and MVAPICH2 are the dominant open-source implementations; both use UCX or LibFabric as the underlying transport layer. MPI is less common in modern ML training than NCCL/RCCL but remains essential for HPC workloads co-located with AI clusters. See Chapter 4.

**MPO (Multi-fiber Push-On connector)** — A high-density fiber connector standardized in IEC 61754-7, accommodating 8, 12, 16, or 24 fibers in a single housing. MPO connectors are used with structured cabling trunk cables in data centers. For 400GBASE-SR8, each link uses a single MPO-16 connector (8 transmit + 8 receive fibers). Polarity management is critical with MPO: incorrect polarity (Type A vs B vs C) causes crossed Tx/Rx pairs and link failure. See Appendix E.3.3.

**MR (Memory Region)** — An RDMA concept: a contiguous range of virtual memory that has been registered with the HCA, pinned in physical memory, and assigned an L_Key (local) and R_Key (remote) for access control. RDMA operations reference MRs by key; the HCA performs virtual-to-physical address translation at hardware speed. See Chapter 2.

**Multus** — A Kubernetes CNI meta-plugin that allows a Pod to have multiple network interfaces by chaining multiple CNI plugins. In AI clusters, Multus is used to attach both a primary CNI interface (Cilium, flannel) and a secondary RDMA/SR-IOV interface for NCCL traffic. See Chapter 13.

**MTU (Maximum Transmission Unit)** — The largest IP packet size (in bytes) that a network path can carry without fragmentation. MTU mismatches between hosts and switches are one of the most common root causes of RDMA performance degradation, BGP session instability (OPEN message fragmentation), and silent packet loss. See Chapters 2, 17 and Appendix G.

**NCCL (NVIDIA Collective Communications Library)** — NVIDIA's optimized library implementing AllReduce, AllGather, Broadcast, Reduce, and ReduceScatter collectives for GPU clusters. NCCL selects transport backends (NVLink, RoCEv2, InfiniBand) automatically and implements ring, tree, and direct algorithms. NCCL is the primary collective library for PyTorch DDP on NVIDIA hardware. See Chapters 19 and 20.

**NDK (Network Driver Kit / NVIDIA DOCA)** — In NVIDIA context, refers to the DOCA (Data Center Infrastructure-on-a-Chip Architecture) SDK for BlueField DPU application development. DOCA provides APIs for flow steering, crypto offload, DMA, and RegEx across BlueField hardware. See Chapter 10.

**NETCONF (Network Configuration Protocol)** — An IETF protocol (RFC 6241) for network device configuration management, using XML encoding over SSH. NETCONF operations (get-config, edit-config, commit, lock) provide transactional configuration with candidate datastore support. YANG models define the configuration schema. See Chapter 14.

**NUMA (Non-Uniform Memory Access)** — A memory architecture in multi-socket servers where each CPU socket has local DRAM accessed at lower latency, and can access remote (other socket) DRAM at higher latency. In AI networking, NUMA affinity is critical: the NIC, GPU, and process should all be on the same NUMA node to minimize cross-socket latency in GPUDirect RDMA paths. See Chapters 2 and 10.

**NVLink** — NVIDIA's proprietary high-bandwidth, low-latency GPU-to-GPU interconnect. NVLink 4.0 in H100 SXM systems provides 900 GB/s aggregate GPU-to-GPU bandwidth within a single node (via NVSwitch). NVLink traffic does not traverse the NIC/fabric; inter-node communication still requires the network. See Chapter 19.

**NVMe-oF (NVMe over Fabrics)** — A protocol extension (NVM Express specification) that transports NVMe commands over a network fabric instead of a local PCIe bus. NVMe-oF supports RDMA (RoCEv2, InfiniBand) and TCP transports. Used in AI clusters for high-performance distributed storage (checkpoint storage, dataset I/O). See Chapter 6.

**NDR (Next Data Rate)** — The InfiniBand generation following HDR, operating at 400 Gb/s per port (4 lanes × 100 Gb/s PAM4). NDR uses OSFP connectors for switch ports and CX7 NICs. NDR is the current top-of-line for H100/H200 DGX systems. See Chapter 2 and Appendix E.3.

**OFI (OpenFabrics Interfaces / LibFabric)** — A framework providing a unified, portable API for high-performance network communication across InfiniBand, RoCEv2, EFA, and shared-memory fabrics. OFI/LibFabric abstracts transport details into a consistent API used by MPI libraries (OpenMPI, MVAPICH), PMIx, and NCCL. See Chapter 4.

**OVN (Open Virtual Network)** — A virtual networking system built on OVS that provides logical switches, routers, ACLs, and load balancers as software abstractions. OVN is the SDN backend for OpenStack Neutron and some Kubernetes CNI implementations. See Chapter 11.

**OVS (Open vSwitch)** — A production-quality open-source virtual switch supporting OpenFlow, OVSDB, and kernel/DPDK data paths. OVS is widely used as the host vSwitch in cloud environments and as the underlay for OVN. See Chapter 11.

**OpenConfig** — A vendor-neutral effort to define common YANG data models for network device configuration and telemetry. OpenConfig models exist for interfaces, BGP, QoS, LLDP, and platform components. gNMI and NETCONF both use OpenConfig YANG models as the schema for network automation. See Chapters 14 and 15.

**OSFP (Octal Small Form-factor Pluggable)** — A high-density optical transceiver form factor designed for 400G and 800G applications. OSFP is physically larger than QSFP-DD and provides more power headroom (up to 12W vs. 8W for QSFP-DD). OSFP is the connector standard for NDR InfiniBand ports on Quantum-3 switches and CX7 NICs in NDR mode.

**P4 (Programming Protocol-Independent Packet Processors)** — A domain-specific language for programming packet-processing pipelines in switches, NICs, and SmartNICs. P4 programs define how headers are parsed, how tables are matched, and how actions modify packets. P4Runtime provides a standard control-plane API for P4 targets. See Chapter 9.

**PCIe (Peripheral Component Interconnect Express)** — The dominant host bus interconnect for GPUs, NICs, and NVMe SSDs. PCIe Gen5 doubles the bandwidth of Gen4 (64 GB/s for x16). PCIe topology determines GPUDirect RDMA performance (same PCIe root complex = lowest latency). See Chapters 1 and 2.

**PFC (Priority Flow Control)** — IEEE 802.1Qbb. A per-priority PAUSE mechanism that allows a receiver to pause transmission on a specific 802.1p priority class without pausing all traffic. PFC creates lossless queues for RDMA traffic while allowing lossy queues for TCP traffic on the same physical link. PFC mis-configuration (wrong priority mapping, PFC storms, PFC deadlock) is a common root cause of RoCEv2 performance problems. See Chapters 2 and Appendix G.

**PMIx (Process Management Interface – Exascale)** — An open standard API (successor to PMI/PMI2) for bootstrap and runtime services in HPC job launchers. PMIx provides key-value fence operations used by MPI and NCCL for rank discovery, initial barrier, and runtime coordination. See Chapter 4.

**PromQL (Prometheus Query Language)** — The query language for the Prometheus time-series database. PromQL supports range selectors, aggregation operators (sum, rate, topk), and mathematical functions. Used for querying RDMA, GPU, and switch telemetry metrics in AI cluster observability stacks. See Chapter 16.

**PTP (Precision Time Protocol)** — IEEE 1588. A protocol for sub-microsecond clock synchronization over packet networks, using hardware timestamping to compensate for path delay. PTP is required for accurate distributed tracing of AI workloads and for coordinated actions in time-sensitive network applications. See Chapter 3.

**QP (Queue Pair)** — The fundamental RDMA communication endpoint. A Queue Pair consists of a Send Queue and a Receive Queue; RDMA operations are posted to the Send Queue and completions are reported to a Completion Queue (CQ). QPs can operate in Reliable Connected (RC), Unreliable Connected (UC), or Unreliable Datagram (UD) modes. See Chapter 2.

**P4Runtime** — A gRPC-based API for managing the data-plane state (table entries, counter reads, digest registrations) of P4-programmed targets. P4Runtime separates the control plane from the compiled P4 data-plane program, enabling the same control-plane software to manage different P4 targets. See Chapter 9.

**PAM4 (Pulse Amplitude Modulation 4-level)** — A signaling technique that encodes 2 bits per symbol by using 4 amplitude levels (0, 1, 2, 3). PAM4 doubles the data rate over a given electrical bandwidth compared to NRZ (1 bit per symbol). 400GbE and NDR InfiniBand both use PAM4 signaling. PAM4's lower noise margin compared to NRZ means signal integrity (cable quality, connector cleanliness) is more critical at 400G than at 100G.

**PD (Protection Domain)** — An RDMA resource that groups Memory Regions and Queue Pairs; RDMA operations are only permitted between QPs and MRs within the same Protection Domain. PDs provide isolation between different applications or tenants sharing a single HCA. See Chapter 2.

**RCCL (ROCm Collective Communications Library)** — AMD's equivalent of NCCL for ROCm-based GPUs (Instinct MI series). RCCL implements the same AllReduce/AllGather/ReduceScatter API as NCCL and supports RoCEv2 and InfiniBand transports via ROCm's RDMA stack. See Chapter 19.

**RDMA (Remote Direct Memory Access)** — A network communication model in which one host can directly read from or write to the memory of a remote host without involving the remote host's CPU or OS. RDMA achieves sub-microsecond latency and near-zero CPU overhead. RDMA requires hardware support in the NIC (HCA) and a lossless or reliable transport. See Chapter 2.

**RoCEv2 (RDMA over Converged Ethernet version 2)** — An RDMA transport that encapsulates IB transport headers in UDP/IP packets, enabling RDMA over standard Ethernet infrastructure. RoCEv2 requires congestion control (DCQCN + ECN) and optionally PFC for lossless delivery. It is the dominant RDMA transport in hyperscaler AI training fabrics. See Chapter 2.

**RSS (Receive Side Scaling)** — A NIC feature that distributes incoming packets across multiple receive queues (and CPU cores) using a hash of the flow tuple. RSS improves multi-core receive throughput and is required for high-bandwidth TCP/UDP traffic in software datapaths (OVS-DPDK, userspace networking). See Chapter 5.

**RESTCONF** — An HTTP-based protocol (RFC 8040) providing a REST API to YANG-modeled configuration and operational state, using JSON or XML encoding. RESTCONF is simpler to integrate than NETCONF (no SSH, no transactions) but lacks NETCONF's candidate-datastore and rollback capabilities. Used for lightweight network automation scripts and read-only telemetry. See Chapter 14.

**Ring AllReduce** — A specific AllReduce algorithm in which GPU processes are arranged in a logical ring; each process sends data to its neighbor and receives data from its other neighbor in a pipelined fashion. Ring AllReduce achieves near-optimal bandwidth utilization (each link carries 2(N-1)/N of the data volume for N participants) and is the default algorithm in NCCL for large message sizes. See Chapter 19.

**SAI (Switch Abstraction Interface)** — An open API (developed by Microsoft, now managed by OCP) defining a vendor-neutral interface between a NOS and a switch ASIC's SDK. SAI allows SONiC to run on Broadcom, NVIDIA, Marvell, and Intel ASICs without modifying the NOS upper layers. See Chapter 8.

**SHARP (Scalable Hierarchical Aggregation and Reduction Protocol)** — NVIDIA's in-network computing technology that offloads AllReduce operations to InfiniBand or Ethernet switches (Quantum-3, Spectrum-4). Switches perform partial reductions as data flows through the fabric, reducing the volume of data transmitted and dramatically lowering AllReduce latency at scale. Requires NVIDIA switch silicon and NCCL SHARP plugin. See Chapters 19 and 20.

**SLAAC (Stateless Address Autoconfiguration)** — IPv6 mechanism (RFC 4862) by which hosts self-configure a global unicast address using a Router Advertisement prefix and their EUI-64 interface identifier. SLAAC is used in SONiC and containerlab labs for IPv6 management address assignment. See Chapter 8.

**SmartNIC** — A network interface card with additional programmable compute elements (FPGAs, NPUs, Arm cores) enabling offloading of packet processing, storage, or security workloads from the host CPU. SmartNICs range from simple offload cards to full DPUs. See Chapter 10.

**SONiC (Software for Open Networking in the Cloud)** — An open-source network operating system developed by Microsoft and contributed to the Open Compute Project. SONiC runs on multiple switch ASICs via SAI and uses a Redis-based state database (APP_DB, ASIC_DB) for decoupled NOS components. See Chapter 8.

**SPDK (Storage Performance Development Kit)** — An Intel-developed framework for high-performance NVMe storage applications. SPDK uses DPDK-style user-space polling drivers and lock-free queues to achieve multi-million IOPS with low CPU overhead. SPDK implements NVMe-oF target and initiator. See Chapter 6.

**SR-IOV (Single Root I/O Virtualization)** — A PCIe specification enabling a single physical NIC to present multiple virtual functions (VFs) to the host, each addressable as a PCI device. SR-IOV VFs are passed through directly to VMs or containers, providing near-native NIC performance. In Kubernetes AI clusters, SR-IOV Network Operator manages VF lifecycle for RDMA workloads. See Chapter 13.

**SR Linux (Nokia SR Linux)** — Nokia's disaggregated network operating system, built on Linux with a gNMI-native management stack and model-driven configuration. SR Linux is designed for use in containerlab environments (distributed as a container image) and is a popular choice for network automation labs. See Chapters 15 and 21.

**SRv6 (Segment Routing over IPv6)** — A source-routing architecture that encodes an explicit forwarding path in the IPv6 Segment Routing Header as a list of IPv6 addresses (SIDs). SRv6 enables traffic engineering without per-flow state in the core. See Chapter 11.

**SPIFFE/SVID (Secure Production Identity Framework for Everyone / SPIFFE Verifiable Identity Document)** — An open standard for workload identity in distributed systems. SPIFFE defines a URI-based workload identity (SPIFFE ID); an SVID is the X.509 certificate that carries that identity and is presented during mutual TLS. Used in AI cluster security architectures for zero-trust mutual TLS between services. See Chapter 12.

**TC (Traffic Control)** — The Linux subsystem (`tc` command, `tc-bpf`, `tc-flower`) for packet classification, queuing, and shaping in the kernel networking stack. TC eBPF programs (attached at `cls_bpf` or `tc-flower` hook points) enable programmable packet manipulation at the egress/ingress of network interfaces. See Chapter 7.

**UCX (Unified Communication X)** — An open-source communication framework providing a portable high-performance API over RDMA (InfiniBand, RoCEv2), shared memory, and TCP transports. UCX is the underlying transport framework used by OpenMPI, MVAPICH2, and is available as a NCCL transport backend. See Chapter 4.

**VXLAN (Virtual Extensible LAN)** — A network virtualization encapsulation (RFC 7348) that tunnels Ethernet frames inside UDP/IP packets using a 24-bit VNI (VXLAN Network Identifier), supporting up to 16 million logical segments. VXLAN with EVPN control plane is the standard overlay fabric for multi-tenant data centers. See Chapter 11.

**WireGuard** — A modern, cryptographically opinionated VPN protocol implemented in the Linux kernel (and as a userspace application). WireGuard uses Curve25519 key exchange, ChaCha20-Poly1305 encryption, and provides a minimal attack surface. Used in containerlab and AI cluster management networks for secure tunneling. See Chapter 21.

**XDP (eXpress Data Path)** — A Linux kernel hook that executes eBPF programs at the earliest possible point in the receive path, before memory allocation and sk_buff creation. XDP programs can DROP, PASS, TX (bounce), or REDIRECT packets at line rate. XDP is used for DDoS mitigation, load balancing (Cilium, Katran), and custom packet steering. See Chapter 7.

**YANG (Yet Another Next Generation)** — A data modeling language (RFC 6020, RFC 7950) for network configuration and state, used with NETCONF, RESTCONF, and gNMI. YANG models define the schema of device configuration trees; OpenConfig YANG models provide vendor-neutral schemas for interface, BGP, QoS, and telemetry configuration. See Chapter 14.

**ZeRO (Zero Redundancy Optimizer)** — A memory optimization strategy in DeepSpeed for training large models that partitions optimizer states, gradients, and model parameters across data-parallel ranks instead of replicating them. ZeRO Stage 1/2/3 progressively partition more state, reducing per-GPU memory at the cost of additional AllGather/ReduceScatter communication. See Chapter 20.

---

*This glossary covers terms as used in the context of AI/HPC networking. Many terms have broader definitions in their respective standards; readers are directed to the referenced RFCs, IEEE standards, and vendor documentation for normative definitions.*


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).