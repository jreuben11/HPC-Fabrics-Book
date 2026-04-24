# Appendix D — Further Reading & Community Resources

Curated pointers to the communities, conferences, mailing lists, and reference materials that track the AI cluster networking ecosystem. This is not a comprehensive bibliography — it is a survival guide for staying current in a fast-moving field.

---

## D.1 Mailing Lists & Discussion Forums

**rdma-users@lists.linux-foundation.org**
The primary list for Linux RDMA stack questions: `rdma-core`, `libibverbs`, `mlx5`, `rxe` soft-RDMA. Searched archives are at [lists.linux.dev](https://lists.linux.dev/). Useful for MOFED/upstream kernel differences and `perftest` interpretation.

**netdev@vger.kernel.org**
The Linux networking kernel list. eBPF/XDP patches, DPDK kernel interactions, ethtool changes, and driver submissions all pass through here. Archive: [lore.kernel.org/netdev](https://lore.kernel.org/netdev/).

**bpf@vger.kernel.org**
The dedicated eBPF kernel list. All libbpf, BPF verifier, XDP, and Cilium-upstream changes originate here. Archive: [lore.kernel.org/bpf](https://lore.kernel.org/bpf/).

**frr-users@lists.frrouting.org / frr-dev@lists.frrouting.org**
FRRouting user and developer lists. Relevant for SONiC BGP questions, EVPN route type debugging, and BFD timer behavior.

**sonic-dev@googlegroups.com**
SONiC developer discussion. For ConfigDB schema questions, `orchagent` behavior, and SAI vendor-specific bugs.

---

## D.2 Slack Workspaces & Real-Time Communities

**Containerlab Slack** — `containerlab.dev/community`
Active community for lab topology help, new NOS kind questions, and CI integration. The fastest path to answers for Containerlab-specific issues.

**FRRouting Slack** — [frrouting.org/community](https://frrouting.org/community/)
FRR developers and operators. Useful for EVPN, OSPF, and BFD configuration questions.

**SR Linux Discord / Community** — [learn.srlinux.dev](https://learn.srlinux.dev)
Nokia SR Linux community with active labs, Python NDK agent examples, and gNMI walkthrough materials.

**eBPF Slack** — [ebpf.io/slack](https://ebpf.io/slack)
Central hub for libbpf, Cilium, Katran, and XDP questions. Cilium core developers are active here.

**CNCF Slack — #sig-network** — [slack.cncf.io](https://slack.cncf.io)
Kubernetes networking SIG: Multus, SR-IOV operator, Cilium, Whereabouts. #wg-device-management is relevant for GPU pod resource allocation.

**OpenConfig Community** — [openconfig.net/community](https://openconfig.net/community/)
Model discussions, gNMI implementation questions, and vendor deviation registrations.

---

## D.3 Conferences & Workshops

**Open Compute Project (OCP) Global Summit**
Annual. The primary venue for hyperscaler hardware announcements: new NIC ASICs (ConnectX generations, BlueField), switch silicon (Tomahawk, Spectrum, Tofino successors), and network architecture papers from Meta, Google, Microsoft, and Alibaba. Recordings are free at [opencompute.org](https://www.opencompute.org/).

**P4 Workshop** (co-located with SIGCOMM)
Annual. Covers P4 language evolution, new target architectures (SmartNIC P4, FPGAs), and production deployments of programmable data planes. Proceedings archived at [p4.org](https://p4.org/events/).

**Linux Plumbers Conference (LPC)**
Annual. The definitive technical conference for Linux networking: eBPF/XDP roadmap, DPDK kernel interactions, RDMA driver evolution, and netdev micro-conference. Recordings and slides at [lpc.events](https://lpc.events/).

**USENIX NSDI** (Networked Systems Design and Implementation)
Biannual. Academic venue with strong AI networking content: large-scale training fabric papers (Google, Meta, Microsoft), new congestion control proposals, and storage network research. Open-access proceedings at [usenix.org/nsdi](https://www.usenix.org/conference/nsdi26).

**USENIX OSDI / ATC**
Biannual. Systems research with frequent AI infrastructure papers: distributed training runtime design, storage system performance, and scheduling. Open-access at [usenix.org](https://www.usenix.org/).

**SC (Supercomputing) Conference**
Annual. The primary HPC venue: InfiniBand fabric scale results, MPI/UCX performance, Lustre/DAOS storage, and interconnect research. Many AI training scale papers appear here before appearing in ML venues.

**Hot Interconnects**
Annual. IEEE symposium focused solely on high-speed interconnects: CXL, NVLink, PCIe, and emerging memory fabrics. The venue where interconnect architects present detailed microarchitecture results.

---

## D.4 Reference Repositories & Learning Resources

**xdp-project/xdp-tutorial** — [github.com/xdp-project/xdp-tutorial](https://github.com/xdp-project/xdp-tutorial)
The canonical hands-on introduction to XDP programming with libbpf. Covers packet inspection, forwarding, and AF_XDP. Directly supplements Chapter 7.

**p4lang/tutorials** — [github.com/p4lang/tutorials](https://github.com/p4lang/tutorials)
P4 exercise repository: basic forwarding, multicast, load balancing, and INT (In-band Network Telemetry) on the `bmv2` software switch. Directly supplements Chapter 9.

**NCCL Tests** — [github.com/NVIDIA/nccl-tests](https://github.com/NVIDIA/nccl-tests)
AllReduce, AllGather, ReduceScatter, and AllToAll benchmarks. The standard tool for collective performance characterization (Appendix B).

**Containerlab Examples** — [containerlab.dev/lab-examples](https://containerlab.dev/lab-examples/)
Curated topology library maintained by the Containerlab team. SR Linux, SONiC, FRR, and Juniper cRPD examples.

**SR Linux Network Developer Kit (NDK)** — [learn.srlinux.dev/ndk](https://learn.srlinux.dev/ndk/)
Tutorials for building custom agents in Python and Go that interact with SR Linux's YANG state. Supplements Chapter 8.

**Batfish Documentation** — [batfish.readthedocs.io](https://batfish.readthedocs.io)
The Batfish network verification tool with Jupyter notebook examples for reachability, routing policy, and differential analysis. Supplements Chapter 22.

---

## D.5 Companion Investigations

This book covers the *network fabric* perspective on AI training infrastructure. Two companion investigations treat adjacent topics in comparable depth:

**GPU Collective Communications Deep Dive**
Covers NCCL internals, ring and tree AllReduce algorithm selection, SHARP in-network aggregation architecture, RCCL/oneCCL differences, and GPU topology detection. The network chapters in this book (19, 20) are intentionally brief summaries of that material.

**Distributed Training Runtimes**
Covers PyTorch Distributed (DDP, FSDP, pipeline parallelism), DeepSpeed ZeRO stages 1–3, Megatron-LM tensor and pipeline parallelism, and Ray cluster internals. Chapter 20 of this book summarizes the fabric-relevant traffic patterns; the companion investigation treats the framework internals.

Both investigations are referenced from Chapter 4 (interconnect abstractions) and from the brief Chapters 19–20.


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).