# The AI Cluster Network Stack
### *Building High-Performance Fabrics for GPU Clusters from First Principles*

The network is the bottleneck. Every large-scale AI training run eventually hits the same wall: GPUs sit idle, not because they lack compute, but because the **fabric** between them cannot deliver data fast enough. This book is a vertical integration guide for the open-source tools that fix that — from the **RDMA transport** model at the bottom through **kernel-bypass I/O**, **programmable fabrics**, container networking, model-driven management, distributed storage, and the testing frameworks that let you validate the whole stack before touching production hardware.

Every layer is treated as software: version-controlled, CI-tested, observable, and replaceable. The organizing principle is **disaggregation**: **UCX** and **LibFabric** decouple transport semantics from hardware; **DPDK** and **SPDK** eject the kernel from the I/O path; **eBPF** extends programmability deep into Linux without leaving it; **SONiC** and **SR Linux** replace proprietary **NOS**es with systems that expose the ASIC through open APIs; **P4** makes the forwarding pipeline itself a software artifact.

---

## Table of Contents

### Front Matter
- [Preface, Lab Setup & Conventions](front-matter.md)

### Part I — Foundations

| # | Chapter | Topics |
|---|---------|--------|
| 1 | [Architecture of the AI Cluster Network](ch01-ai-cluster-architecture.md) | Fat-tree & rail-optimized topologies, traffic patterns, ECMP |
| 2 | [RDMA & the RoCEv2 Stack](ch02-rdma-rocev2-stack.md) | Verbs API, QPs/CQs/MRs, DCQCN, perftest |
| 34 | [Congestion Control & the Ultra Ethernet Consortium](ch34-congestion-control-uec.md) | DCQCN deep dive, Swift/HPCC/TIMELY, UEC transport, lossless vs lossy |
| 3 | [Precision Time Protocol](ch03-precision-time-protocol.md) | IEEE 1588v2, linuxptp, ptp4l/phc2sys |
| 4 | [UCX, LibFabric & PMIx](ch04-ucx-libfabric-pmix.md) | Transport abstraction, provider model, PMIx wire-up, CXL |
| 24 | [InfiniBand Fabric Management](ch24-infiniband-fabric-management.md) | OpenSM, ibnetdiscover, fat-tree routing, PFC lossless |
| 25 | [Cloud AI Networking: EFA, Azure RDMA & GCP TCPX](ch25-cloud-ai-networking.md) | Cloud RDMA transports, NCCL plugins, VPC design |
| 30 | [IPv6 in the AI Cluster Fabric](ch30-ipv6-datacenter.md) | Dual-stack BGP, IPv6 EVPN, Cilium dual-stack, RoCEv2/IPv6 |

### Part II — Kernel-Bypass & Programmable I/O

| # | Chapter | Topics |
|---|---------|--------|
| 5 | [DPDK: Kernel-Bypass Packet Processing](ch05-dpdk-kernel-bypass.md) | EAL, PMDs, Rx/Tx queues, RSS, OVS-DPDK |
| 6 | [SPDK & NVMe-oF](ch06-spdk-nvme-of.md) | User-space NVMe driver, NVMe-oF over RDMA/TCP, checkpoint I/O |
| 7 | [eBPF & XDP](ch07-ebpf-xdp.md) | BPF ISA, libbpf, CO-RE, XDP hooks, AF_XDP, Katran |

### Part III — Programmable Fabric

| # | Chapter | Topics |
|---|---------|--------|
| 8 | [Open Network Operating Systems](ch08-open-nos.md) | SONiC, SR Linux, FRR, VyOS, DENT, OpenWrt, FreeRTOS, Zephyr |
| 9 | [P4 & the Programmable Data Plane](ch09-p4-programmable-data-plane.md) | OpenFlow/SDN, P4₁₆ language, bmv2/Tofino, P4Runtime, in-network AllReduce |
| 10 | [SmartNIC & DPU Programming](ch10-smartnic-dpu-programming.md) | BlueField DOCA, eBPF offload, P4 on SmartNICs, offload decisions |
| 27 | [Adaptive Routing & Traffic Engineering](ch27-adaptive-routing-traffic-engineering.md) | Flowlet switching, NVIDIA AR, packet spraying, SRv6, UCMP |

### Part IV — Overlay & Kubernetes Networking

| # | Chapter | Topics |
|---|---------|--------|
| 11 | [Overlay Networks: BGP-EVPN, VXLAN, OVS & OVN](ch11-overlay-networks-evpn-vxlan.md) | VXLAN encap, EVPN route types, OVS OpenFlow pipeline, OVN |
| 12 | [eBPF-Native Kubernetes: Cilium](ch12-cilium-ebpf-kubernetes.md) | CNI model, kube-proxy replacement, network policy, Hubble, BGP |
| 13 | [Multi-NIC GPU Pods: Multus, SR-IOV & Network Operator](ch13-multus-sriov-network-operator.md) | NetworkAttachmentDefinition, VF provisioning, RDMA in pods |
| 31 | [GPU Virtualization & Isolation: VFIO, MIG, MPS & KubeVirt](ch31-gpu-virtualization-isolation.md) | VFIO passthrough, KVM/QEMU, Proxmox, libvirt, MIG, MPS, KubeVirt, GPUDirect in VMs |
| 32 | [Extending Kubernetes for AI Infrastructure](ch32-kubernetes-extension-apis.md) | CNI spec, Device Plugin API, Operator SDK, Gateway API, CRI/CSI orientation |
| 26 | [Network Security & Zero-Trust](ch26-network-security-zero-trust.md) | WireGuard, IPsec/strongSwan, SPIFFE/SPIRE, Vault PKI, Cilium encryption |
| 29 | [Kubernetes AI Scheduling](ch29-kubernetes-ai-scheduling.md) | Volcano, Kueue, Coscheduler, SLURM, RunAI, NVIDIA Dynamo/Grove |

### Part V — Management, Telemetry & Control

| # | Chapter | Topics |
|---|---------|--------|
| 14 | [Model-Driven Management: NETCONF, YANG & RESTCONF](ch14-netconf-yang-restconf.md) | YANG 1.1, NETCONF datastores, edit-config, ncclient, pyang |
| 15 | [gNMI & OpenConfig Streaming Telemetry](ch15-gnmi-openconfig-telemetry.md) | gNMI Subscribe, OpenConfig path tree, gnmic, Telegraf pipeline |
| 16 | [Observability Pipeline: OpenTelemetry, Prometheus & Grafana](ch16-observability-otel-prometheus-grafana.md) | OTel collector, PromQL, DCGM, Hubble flow dashboards |
| 17 | [BGP Tooling: BIRD, ExaBGP & GoBGP](ch17-bgp-tooling-bird-exabgp-gobgp.md) | Route reflectors, anycast injection, embeddable BGP library |

### Part VI — Storage & Adjacent Infrastructure

| # | Chapter | Topics |
|---|---------|--------|
| 18 | [Distributed Storage for AI Clusters](ch18-distributed-storage.md) | Ceph, Lustre, DAOS, WEKA, MinIO; storage network isolation |
| 19 | [GPU Collective Communications: Network Interface](ch19-gpu-collective-comms-network-interface.md) | NCCL fabric demands, SHARP in-network aggregation, topology file *(brief)* |
| 20 | [Distributed Training Runtimes: Fabric Demands](ch20-distributed-training-fabric-demands.md) | DDP/FSDP, ZeRO stages, Ray networking, parallelism traffic patterns *(brief)* |
| 33 | [AI Inference Serving Fabric](ch33-inference-serving-fabric.md) | Disaggregated prefill/decode, KV cache RDMA, vLLM/TRT-LLM/Dynamo networking |

### Part VII — Testing, Emulation, Simulation & Resilience

| # | Chapter | Topics |
|---|---------|--------|
| 21 | [Lab Environments with Containerlab](ch21-containerlab-lab-environments.md) | Topology YAML, NOS kinds, CI integration, vrnetlab, GNS3, EVE-NG |
| 22 | [Network CI & Configuration Validation](ch22-network-ci-validation.md) | Batfish, pyATS/Genie, Nornir, full git-to-merge-gate CI pipeline |
| 23 | [Simulation: NS-3, OMNET++, SST, GEM5 & SystemC](ch23-simulation-ns3-omnetpp-sst-gem5.md) | Discrete-event, architecture, full-system, TLM hardware simulation |
| 28 | [Fault Tolerance & Network Resilience](ch28-fault-tolerance-resilience.md) | BFD, RDMA retries, PFC watchdog, torchrun recovery, checkpointing |

### Appendices

| | Appendix | Contents |
|---|----------|---------|
| A | [Lab Topology Library](appendix-a-lab-topology-library.md) | Ready-to-run Containerlab YAML topologies (spine-leaf, GPU rail, multi-NIC pod) |
| B | [Benchmark Cheat Sheet](appendix-b-benchmark-cheat-sheet.md) | perftest, nccl-tests, iperf3, fio, gnmic, PromQL one-liners |
| C | [RFC & Specification Index](appendix-c-rfc-spec-index.md) | All RFCs, IEEE standards, and open specs cited in the book |
| D | [Further Reading & Community Resources](appendix-d-further-reading.md) | Mailing lists, Slack workspaces, conferences, reference repos |
| E | [Hardware Selection Guide](appendix-e-hardware-selection.md) | NIC/switch/ASIC comparison, cable & optics, cost tables by cluster size |
| F | [Glossary](appendix-f-glossary.md) | 80+ term definitions (AF_XDP through ZeRO) |
| G | [RDMA & NCCL Troubleshooting Cookbook](appendix-g-troubleshooting-cookbook.md) | Runbook diagnostics for common failure modes |
| H | [Bash & SR Linux CLI Reference](appendix-h-bash-srlinux-cli.md) | Linux/network utility one-liners, SR Linux mode-based CLI |
| I | [Kubernetes SIGs & CNCF Project Landscape](appendix-i-k8s-sigs-cncf.md) | Curated SIG and CNCF project reference for AI infrastructure |
| J | [AI Cluster Network Reference Architectures](appendix-j-reference-architectures.md) | 64/512/4096-GPU network designs, cabling, management plane topology |

---

## Reference Documents

- [Technology Reference Map](reference-map.md) — index of all technologies by domain with external documentation links
- [Book Outline](book-outline.md) — detailed chapter-by-chapter content plan with audience notes

---

## Lab Environment

All labs run on a single laptop using [Containerlab](https://containerlab.dev). See [front-matter.md](front-matter.md) for full install instructions and the soft-RDMA (`rdma_rxe`) setup for chapters requiring RDMA hardware.

```bash
# Install Containerlab
bash -c "$(curl -sL https://get.containerlab.dev)"

# Pull the three primary NOS images
docker pull ghcr.io/nokia/srlinux:24.3.2
docker pull ghcr.io/sonic-net/sonic-vs:master
docker pull frrouting/frr:v9.1.0
```

Requires Linux with kernel ≥ 4.10; 8 GB RAM minimum, 16 GB comfortable. On macOS/Windows, use a Linux VM (UTM, Lima, Multipass, or WSL2).


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).