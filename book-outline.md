# Book Outline: *The Open AI Network Stack*
### *Building High-Performance Fabrics for GPU Clusters from First Principles*

**Target length:** ~600 pages  
**Audience:** Senior software/infrastructure engineers building or operating AI training clusters; network engineers moving into AI infrastructure; systems researchers.  
**Prerequisite:** Comfortable with Linux, TCP/IP fundamentals, and at least one systems language (C or Python). No prior RDMA or HPC networking assumed.  
**Lab environment:** Every chapter includes hands-on exercises runnable on a single laptop using Containerlab (SR Linux + FRR + SONiC-VS).

---

## Front Matter (~10 pp)

- Preface — why the network is the bottleneck in large-scale AI training
- How to use this book; companion investigations on collective comms and training runtimes
- Lab environment setup (Containerlab quickstart, container images, topology templates)

---

## Part I — Foundations (~110 pp)

The invariants every other chapter builds on: cluster topology, the RDMA transport model, clock discipline, and the hardware layer.

### Chapter 1 — Architecture of the AI Cluster Network (15 pp)
- Fat-tree and rail-optimized fabric topologies; why 2:1 oversubscription breaks AllReduce
- Traffic pattern taxonomy: all-reduce, pipeline-parallel, checkpoint I/O, dataset streaming
- Bandwidth, latency, and ECMP hashing — the three knobs that govern collective performance
- Stack overview: how Parts I–VII compose into a full system

### Chapter 2 — RDMA & the RoCEv2 Stack (25 pp)
- Verbs programming model: QPs, CQs, MRs, WRs; libibverbs API walkthrough
- InfiniBand vs RoCEv1 vs RoCEv2 — what changed and why RoCEv2 won in Ethernet fabrics
- DCQCN congestion control: ECN marking, NIC rate limiting, switch buffer thresholds
- `rdma-core`, `perftest` benchmarking workflow (`ib_write_bw`, `ib_read_lat`)
- *Lab: measure bandwidth and latency across a two-node RoCEv2 Containerlab topology*

### Chapter 34 — Congestion Control & the Ultra Ethernet Consortium (15 pp)
- Why DCQCN exists: the bandwidth-delay product problem, ECN marking in the switch ASIC, the NIC rate limiter feedback loop
- **DCQCN** algorithm in depth: α decay, rate increase/decrease state machine, `Kmin`/`Kmax`/`Pmax` parameters; `mlxconfig` and switch ECN threshold configuration; `mlxreg` for per-port DCQCN knobs
- PFC and its pathologies: head-of-line blocking, PFC pause propagation, deadlock formation, watchdog auto-disable (cross-ref Ch24, Ch28)
- Beyond DCQCN: **Swift** (Google — RTT-based, no ECN required), **HPCC** (Alibaba — INT-feedback precise rate control), **TIMELY** (Microsoft — latency gradient), comparison table
- RoCEv2's reorder intolerance: why 5-tuple ECMP causes out-of-order delivery and what it does to RDMA QPs (cross-ref Ch27 packet spraying)
- **Ultra Ethernet Consortium (UEC)**: formation (2023, AMD/Broadcom/Cisco/HPE/Intel/Meta/Microsoft), design goals — packet spraying with reorder tolerance, new CC algorithm, retransmit without PFC, lossless semantics without pause frames; UET (Ultra Ethernet Transport) spec status; timeline and adoption outlook
- Lossless vs. lossy fabric tradeoff: PFC elimination path, selective retransmit, UEC vs. InfiniBand lossless model
- *Lab: DCQCN parameter sweep with soft-RoCE (`rdma_rxe`); ECN threshold measurement with `tc qdisc` RED; visualise queue depth vs. throughput*

### Chapter 3 — Precision Time Protocol (15 pp)
- Why distributed training and RoCE ECN timestamping both demand sub-microsecond synchronization
- IEEE 1588v2 / PTPv2: grandmaster, boundary clock, transparent clock, BMCA
- `linuxptp` in practice: `ptp4l`, `phc2sys`, hardware timestamping, `pmc` introspection
- `chrony` as fallback and hybrid NTP/PTP topologies
- *Lab: build a PTP domain with a software grandmaster and measure clock offset convergence*

### Chapter 4 — Interconnect Abstractions: UCX, LibFabric & PMIx (15 pp)
- UCX endpoint and transport model; how NCCL selects a UCX transport at runtime
- LibFabric provider abstraction: EFA, Slingshot, verbs, tcp — same API, swappable fabric
- PMIx wire-up protocol: how MPI and NCCL ranks discover each other at job launch
- CXL: coherent memory pooling and what it means for GPU-CPU data movement

### Chapter 24 — InfiniBand Fabric Management (20 pp)
- IB vs RoCEv2: transport model differences, lossless semantics, subnet manager requirement
- OpenSM: daemon configuration, SM priority and failover, `subnet.lst`
- Fabric discovery: `ibnetdiscover`, `ibdiagnet`, `ibstat`/`ibstatus`, `ibping`
- Fat-tree routing algorithms: FTREE, LASH, DOR — how OpenSM programs LFTs
- PFC lossless design and PFC deadlock detection; `perfquery` and Prometheus exporter
- *Lab: OpenSM subnet manager + fabric diagnostics using Soft-IB (rdma_rxe)*

### Chapter 25 — Cloud AI Networking: EFA, Azure RDMA & GCP TCPX (20 pp)
- On-prem vs cloud AI networking: managed fabric, cloud-specific drivers, placement constraints
- AWS EFA: SRD transport, `efa-utils`, `fi_info -p efa`, `aws-ofi-nccl` plugin, p4d/p5/trn1 instances
- Azure RDMA: HBv3/NDv5 InfiniBand (HDR/NDR), `hpc-x` MPI stack, proximity placement groups
- GCP GPUDirect-TCPX: GPU-to-GPU over TCP with hardware assist, `tcpxo` plugin, A3 instances
- NCCL tuning: `NCCL_NET`, `NCCL_IB_HCA`, `NCCL_SOCKET_IFNAME`, `NCCL_TOPO_FILE` generation
- VPC design for AI clusters; multi-node placement constraints; cloud vs on-prem decision matrix
- *Lab: libfabric provider comparison (shm/tcp/simulated EFA) + NCCL topology file generator*

### Chapter 30 — IPv6 in the AI Cluster Fabric (15 pp)
- IPv6 addressing hierarchy: /48 site, /52 pod, /56 rack, /64 link, /128 loopback
- Dual-stack BGP with FRR: `address-family ipv6 unicast`, `show bgp ipv6 unicast summary`
- IPv6 EVPN/VXLAN: Type 2 and Type 5 routes with IPv6, symmetric IRB
- Cilium dual-stack: `--ipv6-native-routing-cidr`, dual-stack Services, IPv6 network policy
- SR Linux dual-stack configuration; RoCEv2 over IPv6; PTP over IPv6
- *Lab: dual-stack Containerlab fabric with FRR BGP; PyTorch gloo over IPv6 loopback*

---

## Part II — Kernel-Bypass & Programmable I/O (~70 pp)

Moving packet and storage I/O entirely out of the kernel path.

### Chapter 5 — DPDK: Kernel-Bypass Packet Processing (25 pp)
- Environment Abstraction Layer (EAL), huge pages, CPU pinning, NUMA topology
- Poll-mode drivers, Rx/Tx queues, `rte_mbuf` and mempool design
- Multi-queue hashing, RSS, flow director
- OVS-DPDK: vSwitch with zero kernel involvement
- *Lab: build a packet forwarder with DPDK's l2fwd and measure throughput vs kernel stack*

### Chapter 6 — SPDK & NVMe-oF (20 pp)
- SPDK's user-space NVMe driver: submission/completion queues, lock-free design
- NVMe-oF: extending NVMe over RDMA (RoCE), TCP, and FC fabrics
- SPDK blobstore and bdev abstraction layer
- Checkpoint I/O patterns in LLM training and why NVMe-oF latency matters
- *Lab: configure an SPDK NVMe-oF target and benchmark with fio*

### Chapter 7 — eBPF & XDP (25 pp)
- BPF ISA, verifier, maps, helper functions; CO-RE and `libbpf`
- Hook points: XDP (pre-stack), TC, socket, cgroup — choosing the right attachment
- AF_XDP: kernel-bypass receive into user space without DPDK
- `bpftool`: loading, pinning, introspecting programs and maps
- Katran case study: Meta's XDP L4 load balancer architecture
- eBPF offload to SmartNIC ASICs
- *Lab: write an XDP packet counter and evolve it into a stateless load balancer*

---

## Part III — Programmable Fabric (~85 pp)

Open NOSes, the programmable data plane, offload silicon, and traffic engineering.

### Chapter 8 — Open Network Operating Systems (25 pp)
- **SONiC**: SAI ASIC abstraction, Redis ConfigDB, container decomposition (`orchagent`, `syncd`, `bgpd`), SONiC-VS for development
- **SR Linux**: YANG-native architecture, NDK for custom agents, gNMI-first design, SR Linux container image
- **FRR**: daemon architecture (`bgpd`, `ospfd`, `zebra`), VRF support, EVPN route types; FRR as the routing engine inside both NOSes
- Choosing between them: SAI breadth (SONiC) vs programmability-first (SR Linux)
- Survey of additional FOSS NOSes: **VyOS** (Debian-based edge router/VPN), **DENT** (LF campus/enterprise edge), **OpenWrt** (embedded/CPE), **FreeRTOS** (microcontroller RTOS), **Zephyr** (LF RTOS with growing network stack — Ethernet, BT, 802.15.4, CoAP; relevant for out-of-band management and sensor nodes)
- *Lab: bring up a spine-leaf fabric with SONiC-VS leaves and SR Linux spines running FRR eBGP*

### Chapter 9 — P4 & the Programmable Data Plane (20 pp)
- OpenFlow SDN: match-action model, controller channel, Ryu Python controller, ONOS/OpenDaylight; limitations that motivated P4
- P4₁₆ language: headers, parsers, match-action tables, externs, controls
- Target independence: same P4 program compiled for `bmv2` (software), Tofino (ASIC), SmartNICs
- P4Runtime: gRPC control-plane API, table entry management, packet-in/out
- In-network computing: RDMA SHARP aggregation, in-network AllReduce concepts
- *Lab: implement a telemetry INT (In-band Network Telemetry) probe on bmv2 with P4Runtime controller*

### Chapter 10 — SmartNIC & DPU Programming (20 pp)
- DPU architecture: ARM cores, integrated NIC ASIC, PCIe host interface; where BlueField-3 sits in the rack
- **NVIDIA DOCA** SDK: service framework, RDMA proxy, OVS-DPDK offload, IPsec, telemetry
- eBPF offload pipeline: compiling BPF to NIC hardware, current capabilities and limitations
- P4 on SmartNICs: Tofino-based targets, Pensando/AMD DSC
- Offload decision framework: what to push to the DPU vs keep on the host
- *Lab: deploy an OVS-DPDK instance on a BlueField emulator and verify traffic bypass*

### Chapter 27 — Adaptive Routing & Traffic Engineering (20 pp)
- ECMP limitations: 5-tuple hash collisions, elephant flow concentration
- Flowlet switching: inter-packet gap threshold, SONiC ConfigDB configuration
- NVIDIA/Mellanox adaptive routing: per-packet load balancing for RoCE/IB, AR threshold and algorithm
- Packet spraying for RDMA: DCQCN interaction, out-of-order handling tradeoffs
- SRv6 for DC traffic engineering: SID structure, End/End.X/End.DT4 behaviors, FRR configuration
- UCMP: BGP link-bandwidth community, weighted ECMP; ECMP imbalance measurement and visualization
- *Lab: ECMP hash collision demo, flowlet switching in SONiC, SRv6 locator in FRR*

---

## Part IV — Overlay & Kubernetes Networking (~85 pp)

Wiring GPU pods into the fabric through container networking and overlay protocols.

### Chapter 11 — Overlay Networks: BGP-EVPN, VXLAN, OVS & OVN (20 pp)
- VXLAN encapsulation (RFC 7348): VTEP model, UDP outer header, multicast vs ingress-replication
- BGP EVPN control plane (RFC 7432, RFC 8365): route types 2/3/5, MAC/IP advertisement, symmetric IRB
- Open vSwitch: flow table model, OpenFlow multi-table pipeline, OVS-DPDK fast path
- OVN logical topology: logical switches/routers, ACLs, load balancers — mapping to OVS flows
- *Lab: build a VXLAN/EVPN overlay between SONiC-VS nodes and verify MAC learning via BGP*

### Chapter 12 — eBPF-Native Kubernetes: Cilium (25 pp)
- CNI model: what happens at pod creation; why iptables doesn't scale
- Cilium datapath: per-endpoint BPF programs, eBPF maps as the conntrack replacement
- kube-proxy replacement: Service load balancing in BPF, DSR mode
- Network policy enforcement: identity-based (not IP-based), L4 and L7 policy
- Hubble: per-flow observability from the BPF programs themselves
- Cilium BGP Control Plane: advertising pod CIDRs via BIRD-backed BGP
- *Lab: deploy Cilium on a Kind cluster, enforce L7 HTTP policy, and trace flows with Hubble UI*

### Chapter 13 — Multi-NIC GPU Pods: Multus, SR-IOV & Network Operator (20 pp)
- Multus CNI: `NetworkAttachmentDefinition`, delegating to secondary CNI plugins, MAC/IP assignment
- SR-IOV: VF provisioning, `sriov-network-operator`, VF pools, RDMA in containers
- NVIDIA Network Operator: Helm chart decomposition, MOFED, RDMA shared device plugin, NV-IPAM
- Whereabouts IPAM: cluster-wide IP allocation for secondary interfaces
- GPU-direct RDMA in Kubernetes: peer memory, GPUDirect storage
- *Lab: launch a pod with a management CNI (Cilium) + RDMA secondary NIC (SR-IOV) and run NCCL-tests across two pods*

### Chapter 31 — GPU Virtualization & Isolation: VFIO, MIG, MPS & KubeVirt (25 pp)
- GPU isolation taxonomy: bare-metal, VFIO passthrough, vGPU (NVIDIA vGPU/MxGPU), partitioning (MIG), container — when each model applies and its performance ceiling
- **VFIO-PCI**: IOMMU groups, `vfio-pci` driver binding, `iommu=pt` kernel cmdline, ACS override patch, ROM BAR issues
- **KVM/QEMU** GPU passthrough: QEMU device assignment, OVMF UEFI firmware, `qemu-system-x86_64` hostdev flags, hugepage-backed guest RAM
- **LibVirt/virsh**: XML domain `<hostdev>` PCI passthrough, `virsh nodedev-detach`, network backend choices — Linux bridge, macvtap, SR-IOV VF direct assignment
- **Proxmox VE** (in depth): PCIe passthrough setup (IOMMU, ROM dump, `pcie=1`), PCIe bifurcation for multi-GPU hosts, VM disk management (`qm` CLI, `pvesh` REST API), Proxmox networking stack (Linux bridge, Open vSwitch mode, SDN VXLAN/EVPN plugin), cluster quorum with **Corosync**, HA fencing and resource agents, Ceph integration for VM storage (RBD pool, `rbd:` storage definition)
- **NVIDIA MIG** (Multi-Instance GPU): GI/CI hierarchy, `nvidia-smi mig`, profile catalogue (1g.10gb → 7g.80gb on H100/A100), MIG in Kubernetes via NVIDIA GPU Operator (`strategy: mixed`), NCCL collective behavior across MIG slices
- **NVIDIA MPS** (Multi-Process Service): CUDA context sharing model, `nvidia-cuda-mps-control` daemon, per-client memory limits, MPS vs MIG tradeoffs — isolation vs sharing granularity, latency impact on all-reduce
- **KubeVirt**: `VirtualMachine`/`VMI` CRDs, `virtctl` CLI, network binding modes (bridge, masquerade, SR-IOV), GPU passthrough via `devices.gpus`, live migration constraints when GPUs are attached, `DataVolume` for VM disk provisioning
- Networking consequences: RDMA device visibility in VMs (VFIO direct vs virtio-net), GPUDirect with passthrough (peer memory, DMA-BUF), SR-IOV VF assignment in libvirt and KubeVirt, IOMMU and DMA-remapping impact on GPU peer-to-peer; why vGPU abstractions break GPUDirect
- Decision matrix: bare-metal vs passthrough vs MIG vs MPS vs KubeVirt — by workload type, isolation requirement, and RDMA/GPUDirect need
- *Lab: Proxmox VM with GPU passthrough (`qm set`, OVMF) + KubeVirt VMI with SR-IOV secondary NIC; verify RDMA device visibility inside the VM with `ibv_devinfo`*

### Chapter 26 — Network Security & Zero-Trust for AI Fabrics (20 pp)
- Threat model: lateral movement, NIC firmware exposure, management plane attack surface
- Cilium transparent encryption: WireGuard and IPsec modes, key rotation, `cilium encrypt status`
- Standalone WireGuard: key generation, peer config, `wg show`, overhead measurement
- IPsec with strongSwan: IKEv2, `swanctl`, certificate-based auth, ESP transport mode
- SPIFFE/SPIRE workload identity: SVID, cert-manager, mTLS between services
- HashiCorp Vault PKI: NIC credentials, management API keys, `hvac` Python client
- Cilium `CiliumNetworkPolicy`: identity-based policy, audit mode, Hubble visualization
- *Lab: WireGuard encrypted namespace tunnel + Cilium transparent encryption on Kind*

### Chapter 32 — Extending Kubernetes for AI Infrastructure: CNI, Device Plugins, Operators & Gateway API (25 pp)
- Kubernetes extension points map: CNI (network), CRI (runtime), CSI (storage), Device Plugin API (hardware), Operators (control plane), Gateway API (ingress/routing) — one-page orientation; which layers this book owns
- **CNI spec** in depth: `cmdAdd`/`cmdDel`/`cmdCheck` lifecycle, result JSON schema, IPAM delegation, chaining via `conflist`; how Multus (Ch13) and Cilium (Ch12) implement the spec; writing a minimal CNI plugin in Go using `github.com/containernetworking/cni`
- **Device Plugin API** (`k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1`): `ListAndWatch`, `Allocate`, `PreStartContainer` RPCs; how `nvidia-device-plugin` exposes `nvidia.com/gpu`; how `rdma-shared-device-plugin` exposes `rdma/hca` resources (cross-ref Ch13); writing a stub device plugin with `kubelet` health checking
- **Operator SDK / controller-runtime**: reconcile loop, `client.Client`, `scheme` registration, `controller-gen` marker annotations for CRD generation; watch predicates and rate limiting; writing a `NetworkTopology` CRD operator that maintains rack-to-node affinity annotations consumed by the scheduler (cross-ref Ch29)
- **Gateway API** (`gateway.networking.k8s.io`): resource model — `GatewayClass`, `Gateway`, `HTTPRoute`, `TCPRoute`, `GRPCRoute`; Cilium Gateway API implementation (cross-ref Ch12); `ReferenceGrant` for cross-namespace routing; relevance to AI inference serving endpoints (disaggregated prefill/decode, cross-ref Ch29 Dynamo)
- CRI and CSI orientation (brief): `RuntimeClass` for GPU workloads, `containerd` shim v2 API; CSI `NodeStageVolume`/`NodePublishVolume` — how checkpoint I/O paths attach storage (cross-ref Ch6, Ch18)
- Putting it together: a custom `RDMANetwork` operator that provisions SR-IOV VFs, creates `NetworkAttachmentDefinitions`, and updates device plugin config — end-to-end controller walkthrough
- *Lab: implement a minimal CNI plugin (Go) + stub device plugin; deploy via Operator SDK scaffolding on Kind; expose a service via Cilium Gateway API `HTTPRoute`*

### Chapter 29 — Kubernetes AI Scheduling: Volcano, Kueue & Topology Awareness (20 pp)
- Gang scheduling requirement: all-or-nothing semantics, starvation and deadlock risks
- Volcano: Queue, PodGroup, Job CRDs; `vcctl`; bin-packing vs spread; preemption
- Kueue: ResourceFlavor, ClusterQueue, LocalQueue, admission quotas
- Coscheduler: gang scheduling as a kube-scheduler plugin
- Topology-aware scheduling: NUMA, NVLink, NIC affinity; `topologySpreadConstraints`
- Network implications: AllReduce ring topology, spine traversal latency, `NCCL_TOPO_FILE`
- SLURM: HPC workload manager; GRES GPU allocation; `topology.conf`; PMIx integration (cross-ref Ch. 4); Pyxis + Enroot containers; SLURM vs Kubernetes decision matrix
- RunAI: Kubernetes-native GPU scheduling platform; Projects/Departments quota model; fractional GPU; preemption and fairness
- NVIDIA Dynamo: disaggregated prefill-decode inference serving; KV cache RDMA transfer; Grove topology-aware worker placement; network fabric requirements
- *Lab: Volcano gang scheduling demo + topology_advisor.py rack-pinning analysis*

---

## Part V — Management, Telemetry & Control (~60 pp)

Treating the network as software: declarative config, streaming telemetry, and programmatic BGP.

### Chapter 14 — Model-Driven Management: NETCONF, YANG & RESTCONF (20 pp)
- YANG 1.1 data modeling: modules, containers, lists, leafs, `must`/`when` constraints, deviations
- NETCONF datastores: candidate, running, startup; `<edit-config>`, `<get>`, transactions
- RESTCONF: resource model, YANG-JSON encoding, PATCH semantics
- `pyang` for model validation and tree visualization; `ncclient` for Python automation
- OpenConfig models vs vendor-native YANG: where they agree and where they diverge
- *Lab: configure a SONiC-VS interface and BGP session via NETCONF using ncclient*

### Chapter 15 — gNMI & OpenConfig Streaming Telemetry (20 pp)
- gNMI RPC model: `Get`, `Set`, `Subscribe` (ONCE, POLL, STREAM); path encoding
- OpenConfig model tree: interfaces, BGP, LLDP, platform; path construction
- `gnmic` CLI: subscribing, collecting, and formatting gNMI streams
- Telegraf gNMI input plugin → InfluxDB/Prometheus pipeline
- Dial-in vs dial-out; gNMI as the modern SNMP replacement
- *Lab: stream interface counters and BGP state from SR Linux via gNMI into Prometheus + Grafana*

### Chapter 16 — Observability Pipeline: OpenTelemetry, Prometheus & Grafana (15 pp)
- OpenTelemetry collector: receivers (OTLP, Prometheus scrape, gNMI), processors, exporters
- Prometheus: scrape model, metric types, PromQL for GPU cluster dashboards (DCGM + node_exporter + network exporters)
- Grafana: dashboard composition, alerting, Hubble integration for pod-level flow metrics
- Telegraf as a Swiss-army metrics agent for SNMP, IPMI, and RDMA device stats
- Correlating GPU utilization, NCCL all-reduce timing, and NIC counter anomalies

### Chapter 17 — BGP Tooling: BIRD, ExaBGP & GoBGP (5 pp)
- BIRD as a route reflector and Cilium/Calico BGP backend; filter language
- ExaBGP for health-check-driven anycast and programmatic prefix injection
- GoBGP as an embeddable BGP library for custom SDN controllers

---

## Part VI — Storage & Adjacent Infrastructure (~45 pp)

The storage fabric and the collective/training runtime interfaces, treated briefly given companion investigations.

### Chapter 18 — Distributed Storage for AI Clusters (25 pp)
- **Ceph**: RADOS object store, RBD block, CephFS; CRUSH map and placement groups; tuning for checkpoint I/O
- **Lustre**: MDT/OST architecture, striping across OSTs, Lustre Network (LNet) over RDMA, HSM for dataset tiering
- **DAOS**: SCM-native (Optane/CXL), DAOS pool/container model, libdaos API, POSIX container; designed for exascale AI
- **WEKA**: NVMe-native parallel filesystem, DPDK network stack, GPUDirect Storage support, NVMe+S3 tiering
- **MinIO**: S3-compatible, erasure coding, multi-site replication; practical for on-prem dataset and model artifact storage
- Storage network isolation: dedicated storage VLAN/fabric vs converged RoCE; avoiding storage incast poisoning training traffic

### Chapter 19 — GPU Collective Communications: Network Interface (10 pp) *(brief — see companion investigation)*
- What NCCL demands from the fabric: all-to-all bandwidth, low tail latency, no head-of-line blocking
- NCCL plugin architecture: how UCX, LibFabric, and sharp plugins attach
- SHARP in-network aggregation: offloading AllReduce to the switch ASIC
- Topology detection and `NCCL_TOPO_FILE`; ring vs tree vs all-reduce algorithm selection
- RCCL and oneCCL: where they diverge from NCCL at the fabric interface

### Chapter 20 — Distributed Training Runtimes: Fabric Demands (10 pp) *(brief — see companion investigation)*
- DDP, FSDP, and pipeline parallelism: the three collective traffic patterns and their bandwidth profiles
- DeepSpeed ZeRO stages: how partition granularity changes all-reduce volume
- Ray cluster networking: GCS, worker-to-worker object transfer, object store pinning
- What each parallelism strategy implies for fabric design: rail-optimized vs full-bisection

### Chapter 33 — AI Inference Serving Fabric (15 pp)
- Inference vs. training traffic taxonomy: request-response vs. all-reduce, latency-bound vs. throughput-bound, bursty fan-out vs. sustained collective
- Prefill and decode phases: compute and bandwidth profiles per phase; why separating them onto different hardware reduces queuing
- **Disaggregated prefill/decode** architecture: prefill nodes (compute-bound, high FLOP/byte) vs. decode nodes (memory-bandwidth-bound); KV cache transfer as the binding network operation
- KV cache RDMA transfer: P2P GPU memory copy via **GPUDirect**, DMA-BUF, NVLink vs. RoCEv2 KV migration; latency budget and impact on time-to-first-token (TTFT)
- Tensor parallelism for inference: all-reduce within a node (NVLink), all-reduce across nodes (RoCEv2); sequence parallelism and its all-gather/reduce-scatter pattern
- Speculative decoding communication: draft-model / verifier-model coordination, small-message latency sensitivity
- **NVIDIA Dynamo** network architecture: KV cache transfer protocol, **Grove** topology-aware worker placement, prefill/decode routing (cross-ref Ch29); NIC and fabric requirements
- Serving framework network requirements: **vLLM** distributed executor, **TensorRT-LLM** executor, **SGLang** radix cache and P2P sharing; **Triton Inference Server** gRPC/HTTP endpoint sizing, dynamic batching effects on NIC utilisation
- Load balancing inference endpoints: request routing strategies, consistent hashing for KV cache locality, power-of-two-choices for decode node selection, Gateway API `GRPCRoute` integration (cross-ref Ch32)
- Fabric design implications: spine-leaf sizing for inference, latency budget allocation across network hops, storage network for model weight loading
- *Lab: two-process disaggregated inference simulation — prefill process sends KV cache to decode process via soft-RoCE; measure TTFT vs. KV transfer latency*

---

## Part VII — Testing, Emulation, Simulation & Resilience (~65 pp)

Closing the loop: validating the full stack in software before touching hardware, and keeping it healthy in production.

### Chapter 21 — Lab Environments with Containerlab (20 pp)
- Topology YAML: nodes, links, `kinds`, startup config injection
- NOS kinds in depth: SR Linux, SONiC-VS, FRR, Linux bridges, packet generators
- Wiring GPU cluster topologies: spine-leaf, rail-optimized fabric, ToR redundancy
- CI integration: running Containerlab topologies in GitHub Actions / GitLab CI
- `vrnetlab` for VM-based routers (vMX, vSRX, CSR1000v) alongside containers
- Alternative emulators: GNS3 (GUI-based, large appliance library, real IOS/IOS-XE images) and EVE-NG (browser-based, multi-user, free Community tier) — when to prefer them over Containerlab's IaC approach
- *Lab: build a full 2-spine 4-leaf topology with BGP-EVPN, RDMA secondary interfaces, and Cilium overlay*

### Chapter 22 — Network CI & Configuration Validation (15 pp)
- Batfish: modeling the network as a dataplane — reachability, routing policy, ACL analysis, differential analysis between configs
- pyATS / Genie: structured parsers, testbed YAML, golden-config snapshot and diff workflow
- Nornir: inventory model, task plugins, parallel execution; pairing with Jinja2 for config rendering and pytest for assertions
- Full CI pipeline: `git push` → Batfish intent check → Containerlab topology test → pyATS golden diff → merge gate

### Chapter 23 — Simulation: NS-3, OMNET++, SST, GEM5 & SystemC (10 pp)
- When to simulate vs emulate: protocol research (NS-3/OMNET++) vs architecture modeling (SST/GEM5) vs full-system (GEM5)
- NS-3: discrete-event, RDMA/RoCE modules, DCQCN model, AI training traffic generators
- OMNET++ / INET: modular framework, graphical trace analysis, fat-tree topologies
- SST: HPC interconnect and memory hierarchy simulation, integration with DRAMSim
- GEM5: full-system simulation including NIC models, useful for CXL/memory-fabric research
- SystemC / TLM (IEEE 1666): transaction-level hardware/SoC modeling; used for NIC ASIC and interconnect IP development
- Simworld: lightweight Python-based network emulation framework; faster topology iteration than NS-3 for protocol research

### Chapter 28 — Fault Tolerance & Network Resilience (20 pp)
- Why network faults kill training jobs: RDMA timeout cascades, AllReduce barrier hang
- BFD: sub-second link failure detection, FRR `bfd profile`, timer arithmetic (300ms detection)
- RDMA timeout and retry tuning: RNR timer, retry count, `ibv_modify_qp` QP attributes
- PFC watchdog: detecting PFC storms, `mlnx_qos` auto-disable and recovery
- Training job recovery: `torchrun --max-restarts`, c10d store failover, `NCCL_TIMEOUT`
- Checkpoint strategies: async `torch.save`, atomic rename, DMTCP transparent checkpointing
- Observability for fault detection: RDMA error counter alerts, BFD state via gNMI
- *Lab: BFD 300ms link failure detection + NCCL timeout recovery with torchrun*

---

## Appendices (~50 pp)

**Appendix A — Lab Topology Library (5 pp)**  
Ready-to-run Containerlab YAML topologies: 2-spine/4-leaf BGP-EVPN, GPU cluster rail-optimized fabric, multi-NIC pod testbed.

**Appendix B — Benchmark Cheat Sheet (5 pp)**  
One-liners for: `perftest` (RDMA bandwidth/latency), `nccl-tests` (collective throughput), `iperf3`, `fio` NVMe-oF, `gnmic` subscribe, Prometheus PromQL for GPU cluster dashboards.

**Appendix C — RFC & Specification Index (3 pp)**  
All RFCs, IEEE standards, and open specs referenced in the book with short annotations.

**Appendix D — Further Reading & Community Resources (2 pp)**  
Key mailing lists, Slack workspaces, conference tracks (OCP, P4 Workshop, Linux Plumbers, USENIX NSDI), and companion investigations.

**Appendix E — Hardware Selection Guide (15 pp)**  
NIC comparison (ConnectX-7, BlueField-3, E810, Thor2), switch ASIC comparison (Tomahawk5, Spectrum-4, Tofino2, Trident4), cable and optics reference, cost/performance table by cluster size (64/512/4096 GPUs).

**Appendix F — Glossary (10 pp)**  
Alphabetical definitions of all key terms: AF_XDP, AllReduce, BFD, DCQCN, DPU, ECN, eBPF, ECMP, EFA, EVPN, FRR, gNMI, GPUDirect, HCA, IB, libbpf, MOFED, NCCL, NVMe-oF, P4, PFC, PMIx, QP, RDMA, RoCEv2, SAI, SHARP, SmartNIC, SONiC, SPDK, SR-IOV, SRv6, UCX, VXLAN, WireGuard, XDP, YANG, ZeRO, and 60+ more.

**Appendix G — RDMA & NCCL Troubleshooting Cookbook (10 pp)**  
Runbook-style diagnostics for: RDMA bandwidth below expected, link errors and retransmissions, NCCL AllReduce hang, bandwidth collapse under incast, BGP session flapping, PFC deadlock, Containerlab convergence failures, gNMI subscribe returning no data.

**Appendix H — Bash & SR Linux CLI Reference (8 pp)**  
Two-part practitioner cheat sheet:
- *Linux/Bash utilities* grouped by purpose: network inspection (`ip`, `ss`, `ethtool`, `tc`, `tcpdump`, `tshark`), hardware enumeration (`lspci`, `numactl`, `lstopo`), RDMA/IB tools (`ibstat`, `ibdev2netdev`, `rdma`, `mlxconfig`, `perfquery`), GPU management (`nvidia-smi`, `nvitop`), process & perf (`perf`, `strace`, `lsof`, `dmesg`), data wrangling (`jq`, `yq`, `awk`, `sed`), remote & session (`ssh`, `rsync`, `tmux`, `watch`, `parallel`). Each entry: one-line description + canonical usage example.
- *SR Linux CLI*: mode-based navigation (candidate/running/state), `enter candidate`, `info`, `diff`, `commit now`, `discard`, `tools`, `show` aliases, and the key operational commands used in the book's labs. Covers both the interactive CLI and `gnmic`-style one-shot execution.

**Appendix J — AI Cluster Network Reference Architectures (10 pp)**
End-to-end network designs for three cluster sizes — each with topology diagram, port budget, oversubscription ratios, and management plane layout:
- *64-GPU cluster (single rack)*: dual-port 200GbE NIC, single ToR switch, no spine, flat L2 fabric; storage converged on RoCEv2; 1GbE OOB for BMC/IPMI
- *512-GPU cluster (4 racks)*: 2-tier leaf-spine, rail-optimised fabric (one NIC per rail), dedicated 400GbE spine plane; separate storage fabric (NVMe-oF); in-band management VRF + OOB 10GbE management network; BGP-EVPN overlay
- *4096-GPU cluster (32+ racks)*: 3-tier fat-tree or dragonfly+, multiple spine planes for non-blocking bisection; dedicated InfiniBand storage fabric vs. converged RoCEv2 decision matrix; OOB management network topology; PTP grandmaster placement
- Management plane topology: OOB network design (BMC/Redfish access), in-band management VRF separation, jump host / bastion pattern
- Cable and fibre planning: DAC vs. AOC vs. SR8/DR8 reach requirements per tier, MPO trunk routing
- Cross-reference to Appendix E (hardware selection) and Appendix A (Containerlab lab topologies)

**Appendix I — Kubernetes SIGs & CNCF Project Landscape (8 pp)**  
Curated reference scoped to AI cluster infrastructure — not exhaustive:
- *Kubernetes SIGs*: ~10 directly relevant SIGs (sig-network, sig-node, sig-scheduling, sig-storage, sig-instrumentation, sig-auth, sig-multicluster, sig-cluster-lifecycle, wg-device-management, wg-serving) — one-paragraph description, key deliverables, repo/docs links
- *CNCF projects*: table of projects that appear in the book, grouped by layer — Networking (Cilium, Multus, SR-IOV Network Operator, Submariner, CNI plugins), Observability (Prometheus, Grafana, OpenTelemetry, Jaeger, Thanos), Storage (Rook/Ceph, Longhorn), Security (SPIFFE/SPIRE, Falco, OPA/Gatekeeper), Scheduling & Runtimes (Volcano, Kueue, KubeVirt, Argo Workflows, containerd), Service Mesh & Ingress (Envoy, Contour, Gateway API). Each entry: graduation status (Sandbox/Incubating/Graduated), one-line description, project URL.

---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).