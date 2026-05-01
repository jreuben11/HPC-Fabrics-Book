# Book Plan

This file has been split into two focused documents:

- [reference-map.md](reference-map.md) — technology index by domain (23 sections, external documentation links)
- [book-outline.md](book-outline.md) — chapter-by-chapter content plan with audience notes and lab descriptions

See [README.md](README.md) for the reader-facing table of contents with links to all chapter and appendix files.

---

# TODO
- plan a virtualization config chapter: KVM, Qemu, OpenNebula, UTM, Lima, Multipass, LibVirt, virsh, kubevirt, proxmox
- plan an appendix of useful bash utils with common usage examples. also SR Linux CLI https://learn.srlinux.dev/get-started/cli/
- analyze book-outline.md - are we missing anything else ?
- look at reference-map.md and suggest optimal stack (consider features, dev community size and development velocity) for each purpose and integrations between different stacks
- put a plan together for making these changes and save it in book-plan.md
---

# To Investigate:
- RoCEv2 (RDMA over Converged Ethernet v2) is not natively reorder-tolerant and expects packets to arrive in order, unlike TCP, which is designed to handle packet reordering and loss.Here are the key details regarding RoCEv2 and reordering:In-Order Expectation: While RoCEv2 uses UDP/IP for routing, the underlying RDMA transport expects packets with the same flow identifier (UDP source port) to arrive in order.Packet Reordering Impact: Out-of-order packets can cause performance issues and may be treated as dropped packets by the receiver.Lossless Requirement: Because RoCEv2 is not tolerant of loss or significant reordering, it is typically deployed in "lossless" Ethernet environments.Hardware Dependency: To prevent reordering, RoCEv2 depends on network infrastructure, such as switches supporting **Priority Flow Control (PFC)**, and careful configuration of techniques like ECMP (Equal-Cost Multi-Pathing) to ensure packets in a single flow follow the same path.Emerging Solutions: Recent research, such as [Eunomia], seeks to introduce reordering layers on the receiver side to manage this issue.In contrast, [**NVMe over TCP**] is highlighted as more tolerant of reordering and packet loss
- NVMe over TCP (**NVMe/TCP**) is a high-performance storage networking protocol that extends Non-Volatile Memory Express (NVMe) commands over standard Ethernet networks, allowing fast, low-latency access to remote storage without specialized, costly hardware. Ratified in 2021, it is part of the **NVMe-oF** (over Fabrics) family, enabling disaggregated storage architectures in data centers and cloud-native apps

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).