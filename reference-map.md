# Technology Reference Map — *The Open AI Network Stack*

Index of the technologies across the AI cluster and data-center networking stack, organized by domain. Each entry links to the primary documentation and notes where the technology appears in the book.

The common thread binding this ecosystem is **disaggregation**: the deliberate dismantling of the closed, monolithic network appliance into separable, open, composable layers. Every tier of the stack reflects the same architectural pressure: eliminate proprietary lock-in, push intelligence to software, and remove latency wherever OS or protocol overhead stands between compute and data. At the silicon edge, UCX, LibFabric, and CXL decouple transport semantics from hardware; DPDK and SPDK eject the kernel from the I/O path entirely; eBPF extends this programmability deep into the Linux kernel without leaving it. In the fabric, SONiC and SR Linux replace proprietary NOSes with Linux-based systems that expose the ASIC through open APIs (SAI, YANG, gNMI), while P4 makes the forwarding pipeline itself programmable from first principles; SmartNIC/DPU SDKs push that same programmability into dedicated offload silicon. Above the hardware, BGP-EVPN/VXLAN and OVN provide scale-out overlay primitives, Cilium and Multus wire them into Kubernetes, and distributed storage fabrics (Ceph, Lustre) feed the training pipeline. Management converges on declarative, model-driven APIs (NETCONF/YANG, gNMI/OpenConfig) and a unified observability plane (OpenTelemetry, Prometheus, Grafana) so that configuration, telemetry, and intent are all expressed in the same data model. At the top, GPU collective libraries (NCCL, RCCL) and distributed training runtimes (PyTorch Distributed, DeepSpeed, Ray) are the ultimate consumers — every latency and bandwidth decision in the layers below exists to serve their AllReduce patterns. Containerlab and the simulation frameworks (NS-3, GEM5, SST) close the loop by letting the full stack be validated in software before touching production hardware.

---

## 1. Interconnect & Fabric

| Technology | Role | Docs |
|---|---|---|
| **UCX** | Unified Communication X — portable, high-performance messaging layer (RDMA, shared memory, TCP) underlying MPI/NCCL/UCC | [Docs](https://openucx.readthedocs.io/en/master/) |
| **UCC** | Unified Collective Communication — portable collective ops API (AllReduce, Barrier, etc.) built on UCX | [API](https://openucx.github.io/ucc/api/v1.4.4/html/index.html) |
| **LibFabric (OFI)** | OS Fabric Interface — vendor-neutral RDMA/messaging API (Slingshot, EFA, Omni-Path, verbs) | [Docs](https://ofiwg.github.io/libfabric/) |
| **CXL** | Compute Express Link — PCIe-based coherent memory/device interconnect; key for GPU-to-CPU memory pooling | [Resources](https://computeexpresslink.org/resource-library) |
| **PMIx** | Process Management Interface for Exascale — job launch, wire-up, and fault notification for MPI/AI workloads | [Standard v5](https://pmix.org/uploads/2023/05/pmix-standard-v5.0.pdf) |

---

## 2. High-Performance I/O (Kernel Bypass)

| Technology | Role | Docs |
|---|---|---|
| **DPDK** | Data Plane Development Kit — kernel-bypass packet I/O for line-rate forwarding in user space | [Guide](https://doc.dpdk.org/guides/prog_guide/overview.html) |
| **SPDK** | Storage Performance Development Kit — kernel-bypass NVMe/NVMe-oF and iSCSI in user space | [Docs](https://spdk.io/doc/) |
| **NVMe / NVMe-oF** | Non-Volatile Memory Express — low-latency storage protocol; NVMe-oF extends it over RDMA/TCP fabrics | [Specs](https://nvmexpress.org/specifications/) · [Linux driver](https://nvmexpress.org/drivers/linux-driver-information/) |

---

## 3. Virtual Networking & Overlay

| Technology | Role | Docs |
|---|---|---|
| **Open vSwitch (OVS)** | Programmable software switch; hypervisor and container networking backbone | [Docs](https://docs.openvswitch.org/en/latest/) |
| **OVN** | Open Virtual Network — logical overlay (routers, switches, ACLs) built on top of OVS | [Docs](https://docs.ovn.org/en/latest/) |
| **BGP-EVPN / VXLAN** | Dominant DC fabric overlay: VXLAN tunnels (RFC 7348) + BGP EVPN control plane (RFC 7432, RFC 8365) | [RFC 7348](https://datatracker.ietf.org/doc/html/rfc7348) · [RFC 7432](https://datatracker.ietf.org/doc/html/rfc7432) · [RFC 8365](https://datatracker.ietf.org/doc/html/rfc8365) · [Arista guide](https://www.arista.com/en/um-eos/eos-evpn-overview) |

---

## 4. Programmable Data Plane

| Technology | Role | Docs |
|---|---|---|
| **P4** | DSL for programming switch/NIC match-action pipelines; target-independent | [Consortium](https://p4.org/learn/) · [P4₁₆ spec](https://p4lang.github.io/p4-spec/docs/P4-16-working-draft.html) · [Tutorials](https://github.com/p4lang/tutorials) |
| **P4Runtime** | gRPC control-plane API for P4-programmed devices | [Spec](https://p4lang.github.io/p4runtime/spec/main/P4Runtime-Spec.html) |

---

## 5. Network Management & Telemetry

| Technology | Role | Docs |
|---|---|---|
| **NETCONF / YANG** | IETF config protocol (XML/SSH) + data-modeling language; datastore model (candidate/running/startup) | [RFC 6241](https://datatracker.ietf.org/doc/html/rfc6241) · [RFC 7950](https://datatracker.ietf.org/doc/html/rfc7950) |
| **RESTCONF** | HTTP/JSON variant of NETCONF for YANG-modeled data | [RFC 8040](https://datatracker.ietf.org/doc/html/rfc8040) |
| **gNMI** | gRPC streaming config + telemetry (OpenConfig); replaces SNMP in modern fabrics | [Spec](https://openconfig.net/docs/gnmi/gnmi-specification/) · [Protos](https://github.com/openconfig/gnmi) · [OpenConfig models](https://openconfig.net/projects/models/) |

> **Vendor reference (all three APIs):** PicOS (FS.com/Pica8) — [RESTCONF](https://pica8-fs.atlassian.net/wiki/spaces/PicOS444/pages/70687942/Introduction+of+RESTCONF) · [NETCONF](https://pica8-fs.atlassian.net/wiki/spaces/PicOS444/pages/70687353/Configuring+NETCONF) · [gNMI](https://pica8-fs.atlassian.net/wiki/spaces/PicOS444/pages/70687840/Configuring+gNMI-gRPC+Based+Telemetry+Technology)

---

## 6. Network Operating Systems (FOSS)

| NOS | Role | Docs |
|---|---|---|
| **SONiC** | Hyperscaler DC switch NOS (Azure, Alibaba). SAI ASIC abstraction, Redis ConfigDB. Best FOSS DC NOS. | [Site](https://sonic-net.github.io/SONiC/) · [Wiki](https://github.com/sonic-net/SONiC/wiki) · [Build](https://github.com/sonic-net/sonic-buildimage) |
| **SR Linux** (Nokia) | Modern DC NOS, 100% YANG, first-class gNMI; free container image — de facto Containerlab reference | [Learn](https://learn.srlinux.dev) · [Docs](https://documentation.nokia.com/srlinux/) · [Image](https://github.com/nokia/srlinux-container-image) |
| **FRRouting (FRR)** | The routing stack inside SONiC, VyOS, DENT, Cumulus — BGP, OSPF, IS-IS, EVPN | [Site](https://frrouting.org) · [Docs](https://docs.frrouting.org) |
| **VyOS** | Full Debian-based router NOS; edge, VPN, BGP | [Site](https://vyos.io) · [Docs](https://docs.vyos.io) |
| **DENT** | Linux Foundation NOS for enterprise edge/campus | [Site](https://dent.dev) |
| **OpenWrt** | FOSS NOS for embedded/CPE hardware; not DC-focused | [Site](https://openwrt.org) |
| **FreeRTOS** | RTOS for microcontrollers and embedded network devices | [Docs](https://www.freertos.org/Documentation/00-Overview) |
| **Zephyr** | LF-hosted RTOS for constrained/embedded devices; growing network stack (Ethernet, BT, 802.15.4, CoAP); relevant for out-of-band management and sensor nodes | [Docs](https://docs.zephyrproject.org/latest/introduction/index.html) · [GitHub](https://github.com/zephyrproject-rtos/zephyr) |

> **Containerlab starter pair:** SR Linux + FRR containers for the fabric, SONiC-VS to see the hyperscaler stack.

---

## 7. Lab & Emulation

| Tool | Role | Docs |
|---|---|---|
| **Containerlab** | IaC YAML topology → Docker-based multi-NOS lab; CI-friendly, fast | [Site](https://containerlab.dev) · [Quickstart](https://containerlab.dev/quickstart/) · [Kinds](https://containerlab.dev/manual/kinds/) |
| **GNS3** | GUI emulator (2008+); real IOS/IOS-XE/vMX/vSRX + Docker nodes | [Docs](https://docs.gns3.com) · [Appliances](https://gns3.com/marketplace/appliances) |
| **EVE-NG** | Browser-based multi-user lab emulator; Community (free, 63 nodes) / Pro (paid) | [Docs](https://www.eve-ng.net/index.php/documentation/) |
| **vrnetlab** | Containerises VM-based routers (vMX, vSRX, CSR1000v) for use inside Containerlab topologies | [GitHub](https://github.com/vrnetlab/vrnetlab) |

---

## 8. Simulation

| Tool | Role | Docs |
|---|---|---|
| **NS-3** | Discrete-event network simulator; dominant in academia | [Docs](https://www.nsnam.org/documentation/) |
| **OMNET++** | Modular discrete-event simulator; INET framework for network protocols | [Docs](https://omnetpp.org/documentation/) |
| **SST** | Structural Simulation Toolkit — HPC architecture and interconnect simulation | [Site](https://sst-simulator.org/) |
| **GEM5** | Full-system CPU + memory + interconnect simulator | [Site](https://www.gem5.org/) |
| **SystemC / TLM** | IEEE standard for hardware/SoC modeling and transaction-level simulation | [Overview](https://systemc.org/overview/systemc-tlm/) |
| **Simworld** | Network emulation framework (newer) | [Docs](https://simworld.readthedocs.io/en/latest/) |

---

## 9. GPU Collective Communications

> **Note:** Collective communications and distributed training runtimes (section 13) are covered in depth in separate investigations. Coverage here is intentionally brief — focus is on the interface between the network fabric and the collective layer, not on the libraries themselves.

The layer directly above UCX/UCC that AI training frameworks call for AllReduce, AllGather, and Broadcast across GPU ranks.

| Library | Role | Docs |
|---|---|---|
| **NCCL** | NVIDIA Collective Communications Library — optimized GPU-to-GPU collectives over NVLink, InfiniBand, RoCE, and TCP; used by PyTorch/JAX/TensorFlow | [Docs](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/index.html) · [GitHub](https://github.com/NVIDIA/nccl) |
| **RCCL** | AMD's drop-in fork of NCCL for ROCm/MI-series GPUs | [Docs](https://rocm.docs.amd.com/projects/rccl/en/latest/) · [GitHub](https://github.com/ROCm/rccl) |
| **oneCCL** | Intel oneAPI Collective Communications Library — CPU and GPU (Xe) collectives for oneAPI/DPC++ workloads | [Docs](https://oneapi-src.github.io/oneCCL/) · [GitHub](https://github.com/oneapi-src/oneCCL) |
| **OpenMPI** | Dominant MPI runtime; orchestrates collective ops across nodes using UCX/LibFabric transports | [Docs](https://www.open-mpi.org/doc/) · [GitHub](https://github.com/open-mpi/ompi) |
| **MPICH** | Reference MPI implementation; basis for many vendor MPI stacks (Cray MPICH, Intel MPI) | [Docs](https://www.mpich.org/documentation/guides/) · [GitHub](https://github.com/pmodels/mpich) |

---

## 10. eBPF & XDP

The most active subsystem in the Linux kernel: in-kernel programmability without modules, enabling kernel-bypass-class performance while staying in the kernel security model.

| Technology | Role | Docs |
|---|---|---|
| **libbpf** | Canonical C library for loading and interacting with eBPF programs from user space | [Docs](https://libbpf.readthedocs.io/en/latest/) · [GitHub](https://github.com/libbpf/libbpf) |
| **bpftool** | CLI for inspecting, loading, and pinning eBPF programs and maps | [Man page](https://manpages.ubuntu.com/manpages/noble/man8/bpftool.8.html) · [GitHub](https://github.com/libbpf/bpftool) |
| **XDP** | eXpress Data Path — hook at the NIC driver level for line-rate packet processing before the kernel stack; used in Cilium, Katran, load balancers | [Kernel docs](https://www.kernel.org/doc/html/latest/networking/af_xdp.html) · [xdp-project](https://github.com/xdp-project/xdp-tutorial) |
| **Katran** | Meta's eBPF/XDP L4 load balancer; production reference for XDP-based forwarding at scale | [GitHub](https://github.com/facebookincubator/katran) |

---

## 11. RDMA / RoCE Stack

The userspace software layer over InfiniBand and RoCEv2 hardware — the actual transport for NCCL and UCX in GPU clusters.

| Technology | Role | Docs |
|---|---|---|
| **rdma-core** | Linux RDMA userspace stack — libibverbs, librdmacm, udev rules; the foundation for all verbs-based RDMA apps | [Docs](https://github.com/linux-rdma/rdma-core/blob/master/Documentation/) · [GitHub](https://github.com/linux-rdma/rdma-core) |
| **perftest** | Standard RDMA microbenchmark suite (ib_write_bw, ib_read_lat, etc.) for validating fabric throughput and latency | [GitHub](https://github.com/linux-rdma/perftest) |
| **iproute2 RDMA** | `rdma` CLI tool for link/port/resource inspection on RDMA devices | [GitHub](https://github.com/iproute2/iproute2) |

> **RoCEv2 congestion control:** Production clusters tune DCQCN (switch ECN + NIC rate limiting) to prevent RoCE from collapsing under incast. Key knobs live in switch QoS config and NIC firmware.

---

## 12. Container & Kubernetes Networking

How AI workloads actually run: multi-NIC GPU pods wired into the fabric through CNI plugins and operator-managed device resources.

| Technology | Role | Docs |
|---|---|---|
| **Cilium** | eBPF-native CNI — L3/L4/L7 policy, kube-proxy replacement, Hubble observability, service mesh; extremely high velocity | [Docs](https://docs.cilium.io) · [GitHub](https://github.com/cilium/cilium) |
| **Multus CNI** | Meta-plugin that attaches multiple network interfaces to a pod; essential for giving GPU pods a data-plane NIC alongside the management CNI | [Docs](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md) · [GitHub](https://github.com/k8snetworkplumbingwg/multus-cni) |
| **SR-IOV Network Operator** | Manages SR-IOV VF provisioning and CNI config on Kubernetes nodes for near-native NIC performance in pods | [Docs](https://sriov-network-operator.readthedocs.io) · [GitHub](https://github.com/k8snetworkplumbingwg/sriov-network-operator) |
| **NVIDIA Network Operator** | Bundles RDMA drivers, MOFED, SR-IOV, and secondary CNI management for GPU cluster nodes | [Docs](https://docs.nvidia.com/networking/display/cokan) · [GitHub](https://github.com/Mellanox/network-operator) |
| **Whereabouts** | Cluster-wide IPAM plugin for secondary (non-default) pod networks managed by Multus | [GitHub](https://github.com/k8snetworkplumbingwg/whereabouts) |

---

## 13. Distributed AI Training Runtimes

> **Note:** Covered in depth in a separate investigation. Coverage here is intentionally brief — focus is on the traffic patterns and fabric demands these runtimes impose, not on the frameworks themselves.

The software that generates the collective communication traffic — understanding their AllReduce patterns and topology awareness directly shapes fabric and congestion-control decisions.

| Framework | Role | Docs |
|---|---|---|
| **PyTorch Distributed** | DDP, FSDP, and `torchrun` — the dominant distributed training runtime; drives NCCL collectives across ranks | [Docs](https://pytorch.org/docs/stable/distributed.html) · [GitHub](https://github.com/pytorch/pytorch) |
| **DeepSpeed** | Microsoft's ZeRO optimizer stages + pipeline/tensor parallelism; pushes collective patterns beyond vanilla DDP | [Docs](https://www.deepspeed.ai/docs/) · [GitHub](https://github.com/microsoft/DeepSpeed) |
| **Ray** | Distributed computing framework for Python; Ray Train handles large-scale LLM training, Ray Serve handles inference | [Docs](https://docs.ray.io) · [GitHub](https://github.com/ray-project/ray) |
| **Horovod** | Uber's ring-AllReduce training library; framework-agnostic (TF/PyTorch/MXNet), uses NCCL/Gloo/MPI | [Docs](https://horovod.readthedocs.io) · [GitHub](https://github.com/horovod/horovod) |
| **Megatron-LM** | NVIDIA's tensor + pipeline parallelism framework for large transformer training; reference implementation for model parallelism | [GitHub](https://github.com/NVIDIA/Megatron-LM) |

---

## 14. Observability & Telemetry Stack

Device-level gNMI telemetry covers the fabric; workload-level observability requires a separate, unified pipeline.

| Technology | Role | Docs |
|---|---|---|
| **OpenTelemetry** | Vendor-neutral SDK + collector for traces, metrics, and logs; the emerging standard for unified observability pipelines | [Docs](https://opentelemetry.io/docs/) · [GitHub](https://github.com/open-telemetry/opentelemetry-collector) |
| **Prometheus** | Pull-based metrics store with a rich ecosystem of exporters (node_exporter, DCGM for GPUs, SNMP, gNMI) | [Docs](https://prometheus.io/docs/) · [GitHub](https://github.com/prometheus/prometheus) |
| **Grafana** | Visualization and alerting layer; standard dashboard frontend for Prometheus, Loki, Tempo in AI infra | [Docs](https://grafana.com/docs/) · [GitHub](https://github.com/grafana/grafana) |
| **Hubble** | Cilium's eBPF-based network flow observability — per-pod L4/L7 flow logs and service dependency maps | [Docs](https://docs.cilium.io/en/stable/observability/hubble/) · [GitHub](https://github.com/cilium/hubble) |
| **Telegraf** | Plugin-driven metrics agent; collects from SNMP, gNMI, IPMI, DCGM and ships to InfluxDB/Prometheus | [Docs](https://docs.influxdata.com/telegraf/) · [GitHub](https://github.com/influxdata/telegraf) |

---

## 15. Distributed Storage

The storage fabric feeding training pipelines — checkpoint I/O, dataset streaming, and model serving all depend on this layer.

| Technology | Role | Docs |
|---|---|---|
| **Ceph** | Unified distributed storage (block/object/file via RADOS); widely deployed in on-prem AI clusters; high velocity | [Docs](https://docs.ceph.com) · [GitHub](https://github.com/ceph/ceph) |
| **Lustre** | Parallel filesystem dominant in HPC/supercomputing; optimized for large sequential I/O from thousands of clients | [Docs](https://doc.lustre.org) · [GitHub](https://github.com/lustre/lustre-release) |
| **DAOS** | Intel's Distributed Asynchronous Object Storage — NVMe/SCM-native, ultra-low latency, designed for exascale AI/HPC | [Docs](https://docs.daos.io) · [GitHub](https://github.com/daos-stack/daos) |
| **WEKA** | NVMe-native parallel filesystem; DPDK network stack, GPUDirect Storage support, NVMe+S3 tiering; strong in AI training clusters | [Docs](https://docs.weka.io) · [Site](https://www.weka.io) |
| **MinIO** | S3-compatible object storage; popular for on-prem ML dataset and artifact storage with high throughput | [Docs](https://min.io/docs/minio/linux/index.html) · [GitHub](https://github.com/minio/minio) |

---

## 16. SmartNIC / DPU SDKs

The DPU is becoming the network control point in AI clusters: OVS offload, RDMA proxy, security policy, and telemetry run on dedicated ARM cores attached to the NIC.

| Technology | Role | Docs |
|---|---|---|
| **NVIDIA DOCA** | SDK for programming BlueField DPUs — RDMA, OVS offload, IPsec, telemetry, service mesh acceleration | [Docs](https://docs.nvidia.com/doca/) · [GitHub](https://github.com/NVIDIA/DOCA) |
| **eBPF offload** | BPF programs compiled to NIC hardware (Netronome, some Mellanox); eliminates host CPU from the fast path entirely | [Kernel docs](https://www.kernel.org/doc/html/latest/bpf/bpf_prog_run.html) |
| **OVS-DPDK** | OVS with DPDK as the fast path; commonly offloaded to DPU for zero-host-CPU vSwitch in AI cluster nodes | [Docs](https://docs.openvswitch.org/en/latest/topics/dpdk/) |
| **P4 on SmartNICs** | P4 programs targeting SmartNIC ASICs (Tofino-based, Pensando/AMD); same language, dedicated offload silicon | [P4 arch overview](https://p4.org/p4-spec/docs/P4-16-working-draft.html) |

---

## 17. Time Synchronization

Tight clock discipline is a hard requirement for distributed training coordination, RoCE congestion control (ECN timestamping), and correlated log analysis.

| Technology | Role | Docs |
|---|---|---|
| **linuxptp** | Linux implementation of IEEE 1588 PTP — `ptp4l` (clock sync), `phc2sys` (NIC-to-system clock), `pmc` (management) | [Docs](https://linuxptp.sourceforge.net) · [GitHub](https://github.com/richardcochran/linuxptp) |
| **chrony** | NTP daemon with fast convergence and hardware timestamping; used where PTP isn't available or as a PTP fallback | [Docs](https://chrony-project.org/documentation.html) · [GitHub](https://github.com/mlichvar/chrony) |
| **gpsd + PPS** | GPS/GNSS daemon providing pulse-per-second disciplined time source; used as grandmaster reference in on-prem PTP domains | [Docs](https://gpsd.gitlab.io/gpsd/) · [GitLab](https://gitlab.com/gpsd/gpsd) |

---

## 18. Routing Daemons & BGP Tooling

Lightweight BGP daemons and route-injection tools complement FRR for specific use cases: route reflectors, programmatic prefix advertisement, and policy testing.

| Technology | Role | Docs |
|---|---|---|
| **BIRD** | Fast, compact routing daemon (BGP, OSPF, IS-IS, BFD); common as a route reflector and in Calico/Cilium BGP mode | [Docs](https://bird.network.cz/?get_doc) · [GitLab](https://gitlab.nic.cz/labs/bird) |
| **ExaBGP** | Python BGP daemon designed for programmatic route injection and health-check-driven anycast; not a full router | [Docs](https://github.com/Exa-Networks/exabgp/wiki) · [GitHub](https://github.com/Exa-Networks/exabgp) |
| **GoBGP** | Modern Go BGP implementation with gRPC API; embeddable in custom controllers and SDN applications | [Docs](https://osrg.github.io/gobgp/) · [GitHub](https://github.com/osrg/gobgp) |

---

## 19. Network CI / Validation

Version-controlling network config is only useful if it can be tested. These tools bring software engineering discipline to network change management.

| Technology | Role | Docs |
|---|---|---|
| **Batfish** | Intent-based network config analyzer — catches routing bugs, ACL shadowing, and convergence failures before deployment | [Docs](https://batfish.readthedocs.io) · [GitHub](https://github.com/batfish/batfish) |
| **pyATS / Genie** | Cisco's Python test framework for network devices; structured parsers, diff-based golden-config validation | [Docs](https://developer.cisco.com/docs/pyats/) · [GitHub](https://github.com/CiscoTestAutomation/pyats) |
| **pytest-net / nornir** | Nornir is a Python automation framework for running tasks against device inventories; pairs naturally with pytest for CI | [Nornir docs](https://nornir.readthedocs.io) · [GitHub](https://github.com/nornir-automation/nornir) |

---

## 20. InfiniBand Fabric Management

| Technology | Role | Docs |
|---|---|---|
| **OpenSM** | Open Subnet Manager — implements IBTA SM/SA spec; fat-tree routing algorithms (FTREE, LASH, DOR); SM priority and failover | [GitHub](https://github.com/linux-rdma/rdma-core/tree/master/opensm) |
| **ibnetdiscover** | Discovers and maps the full IB fabric topology; produces topology files for OpenSM routing and diagnostics | [Man page](https://linux.die.net/man/8/ibnetdiscover) |
| **ibdiagnet** | Comprehensive IB fabric diagnostic tool; checks LIDs, routing tables, errors, and partition tables | [MOFED](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/) |
| **perfquery** | Per-port IB performance counter polling; feeds Prometheus via the IB exporter | [Man page](https://linux.die.net/man/8/perfquery) |

---

## 21. Network Security & Zero-Trust

| Technology | Role | Docs |
|---|---|---|
| **WireGuard** | Modern, minimal VPN; kernel-native (Linux 5.6+), ChaCha20-Poly1305 encryption; used for inter-node and management-plane tunnels | [Site](https://www.wireguard.com) · [Paper](https://www.wireguard.com/papers/wireguard.pdf) |
| **strongSwan** | IKEv2/IPsec daemon; certificate-based auth, ESP transport mode for fabric encryption; `swanctl` CLI | [Docs](https://docs.strongswan.org) · [GitHub](https://github.com/strongswan/strongswan) |
| **SPIFFE / SPIRE** | CNCF workload identity — SVIDs (X.509 or JWT), SPIRE server/agent for automatic cert rotation and mTLS between services | [Docs](https://spiffe.io/docs/) · [GitHub](https://github.com/spiffe/spire) |
| **HashiCorp Vault** | Secrets management and PKI; issues NIC credentials, management API keys, and TLS certificates for cluster services | [Docs](https://developer.hashicorp.com/vault/docs) · [GitHub](https://github.com/hashicorp/vault) |
| **Cilium Encryption** | Transparent WireGuard or IPsec encryption for all pod-to-pod traffic; zero application changes required | [Docs](https://docs.cilium.io/en/stable/security/network/encryption-wireguard/) |

---

## 22. Kubernetes AI Scheduling

| Technology | Role | Docs |
|---|---|---|
| **Volcano** | Kubernetes batch scheduler for AI/HPC; gang scheduling, Queue/PodGroup/Job CRDs, bin-packing and preemption | [Site](https://volcano.sh) · [Docs](https://volcano.sh/en/docs/) · [GitHub](https://github.com/volcano-sh/volcano) |
| **Kueue** | Kubernetes-native job queuing and admission control; ResourceFlavor, ClusterQueue, LocalQueue | [Docs](https://kueue.sigs.k8s.io) · [GitHub](https://github.com/kubernetes-sigs/kueue) |
| **Coscheduler** | Gang scheduling as a kube-scheduler plugin; simpler than Volcano for pure gang semantics | [GitHub](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/coscheduling) |
| **SLURM** | Dominant HPC workload manager; GRES GPU allocation, topology-aware placement, PMIx bootstrap, Pyxis+Enroot containers | [Docs](https://slurm.schedmd.com/documentation.html) · [GitHub](https://github.com/SchedMD/slurm) |
| **RunAI** | Kubernetes-native GPU scheduling platform; Projects/Departments quota model, fractional GPU sharing, preemption | [Docs](https://docs.run.ai) · [Site](https://run.ai) |
| **NVIDIA Dynamo** | Disaggregated prefill-decode inference serving framework; Grove topology-aware worker placement, KV cache RDMA transfer | [GitHub](https://github.com/ai-dynamo/dynamo) · [Docs](https://docs.nvidia.com/dynamo) |

---

## 23. Cloud AI Networking

| Technology | Role | Docs |
|---|---|---|
| **AWS EFA** | Elastic Fabric Adapter — custom SRD (Scalable Reliable Datagram) transport for p4d/p5/trn1 instances; `aws-ofi-nccl` plugin | [Docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html) · [GitHub](https://github.com/aws/aws-ofi-nccl) |
| **Azure RDMA** | InfiniBand (HDR/NDR) via SR-IOV on HBv3/NDv5 VMs; `hpc-x` MPI stack and proximity placement groups | [Docs](https://learn.microsoft.com/en-us/azure/virtual-machines/extensions/enable-infiniband) |
| **GCP TCPX / TCPXO** | GPUDirect-TCPX: GPU-to-GPU RDMA over TCP with hardware assist; `tcpxo` NCCL plugin on A3/A3+ instances | [Docs](https://cloud.google.com/compute/docs/gpus/a3-vms) |


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).