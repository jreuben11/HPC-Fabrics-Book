# Book Plan

This file has been split into two focused documents:

- [reference-map.md](reference-map.md) — technology index by domain (23 sections, external documentation links)
- [book-outline.md](book-outline.md) — chapter-by-chapter content plan with audience notes and lab descriptions

See [README.md](README.md) for the reader-facing table of contents with links to all chapter and appendix files.

---

# TODO
- plan a virtualization config chapter: vGPUs and GPU passthrough, KVM, Qemu, OpenNebula, UTM, Lima, Multipass, LibVirt, virsh, kubevirt, proxmox (go deep here), OpenShift, OpenStack, beowulf, pacemaker, MIG, MPS
- plan a K8s programming chapter: Operator SDK https://operatorframework.io/, CNI https://www.cni.dev/docs/ , CSI https://github.com/container-storage-interface/spec/blob/master/spec.md, CRI https://kubernetes.io/docs/concepts/containers/cri/, gatway API https://gateway-api.sigs.k8s.io/ 
- plan an appendix of useful bash utils with common usage examples. also SR Linux CLI https://learn.srlinux.dev/get-started/cli/
- analyze book-outline.md - are we missing anything else ?
- plan an appendix of all K8s sigs https://github.com/kubernetes-sigs 
- look at reference-map.md and suggest optimal stack (consider features, dev community size and development velocity) for each purpose and integrations between different stacks
- put a plan together for making these changes and save it in book-plan.md
---

# To Investigate:
- MACsec, IPsec, and TLS offer security at different OSI layers. MACsec provides high-speed Layer 2 point-to-point link encryption, IPsec ensures Layer 3 network-to-network or host-to-network security, and TLS secures Layer 4/5 application-to-application traffic. MACsec is ideal for data center links, IPsec for VPNs, and TLS for secure web/API communication.
- nftables vs iptables vs bpfilter vs netfilter
- Kind VS Minikube VS k3s vs microk8s vs KubeAdm https://mohamedyassine-bensaid.medium.com/minikube-vs-microk8s-vs-kubeadm-vs-kind-vs-k3s-5a8714c6835f
- RoCEv2 (RDMA over Converged Ethernet v2) is not natively reorder-tolerant and expects packets to arrive in order, unlike TCP, which is designed to handle packet reordering and loss.Here are the key details regarding RoCEv2 and reordering:In-Order Expectation: While RoCEv2 uses UDP/IP for routing, the underlying RDMA transport expects packets with the same flow identifier (UDP source port) to arrive in order.Packet Reordering Impact: Out-of-order packets can cause performance issues and may be treated as dropped packets by the receiver.Lossless Requirement: Because RoCEv2 is not tolerant of loss or significant reordering, it is typically deployed in "lossless" Ethernet environments.Hardware Dependency: To prevent reordering, RoCEv2 depends on network infrastructure, such as switches supporting **Priority Flow Control (PFC)**, and careful configuration of techniques like ECMP (Equal-Cost Multi-Pathing) to ensure packets in a single flow follow the same path.Emerging Solutions: Recent research, such as [Eunomia], seeks to introduce reordering layers on the receiver side to manage this issue.In contrast, [**NVMe over TCP**] is highlighted as more tolerant of reordering and packet loss
- NVMe over TCP (**NVMe/TCP**) is a high-performance storage networking protocol that extends Non-Volatile Memory Express (NVMe) commands over standard Ethernet networks, allowing fast, low-latency access to remote storage without specialized, costly hardware. Ratified in 2021, it is part of the **NVMe-oF** (over Fabrics) family, enabling disaggregated storage architectures in data centers and cloud-native apps
- Designing multitenant GPU infrastructure: Isolation across virtualization and Kubernetes platforms. https://www.redhat.com/en/blog/designing-multitenant-gpu-infrastructure-isolation-across-virtualization-and-kubernetes-platforms  OpenShift Virtualization—GPU enablement: Learn how to configure NVIDIA vGPU, GPU passthrough, and the GPU operator for virtual machines on OpenShift
Red Hat OpenStack Services on OpenShift—virtual GPUs: Step‑by‑step guide to exposing virtual GPUs (vGPU) to OpenStack instances
NVIDIA GPU operator with OpenShift Virtualization: Official NVIDIA guide for deploying the NVIDIA GPU operator in a KubeVirt environment.
- xGMI
- reference architectures - eg https://www.nextplatform.com/connect/2026/04/28/new-google-networks-tuned-up-for-genai-inference-and-training/5218978 

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).