# Appendix C — RFC & Specification Index

All RFCs, IEEE standards, and open specifications cited in *The Open AI Network Stack*, listed in numerical order within each category. Short annotations describe the scope and relevance to AI cluster networking. Chapter references indicate where each standard is treated in depth.

---

## C.1 IETF RFCs

**RFC 4271 — A Border Gateway Protocol 4 (BGP-4)**
Rekhter, Li, Hares. January 2006.
The base BGP specification. Defines path-vector routing, UPDATE message format, and the BGP FSM. In AI cluster fabrics, BGP replaces traditional IGPs as the underlay routing protocol for its scalability and policy flexibility. See Chapters 8, 11, 17.
[https://datatracker.ietf.org/doc/html/rfc4271](https://datatracker.ietf.org/doc/html/rfc4271)

---

**RFC 4291 — IP Version 6 Addressing Architecture**
Hinden, Deering. February 2006.
Defines the IPv6 address space structure: link-local (`fe80::/10`), global unicast (`2000::/3`), multicast (`ff00::/8`), and loopback (`::1`). Required reading for IPv6 dual-stack fabric design. See Chapter 30.
[https://datatracker.ietf.org/doc/html/rfc4291](https://datatracker.ietf.org/doc/html/rfc4291)

---

**RFC 4862 — IPv6 Stateless Address Autoconfiguration**
Thomson, Narten, Jinmei. September 2007.
Defines SLAAC: how hosts derive link-local and global unicast addresses from router advertisements. Relevant to dual-stack AI cluster nodes where IPv6 management addressing is auto-configured. See Chapter 30.
[https://datatracker.ietf.org/doc/html/rfc4862](https://datatracker.ietf.org/doc/html/rfc4862)

---

**RFC 5880 — Bidirectional Forwarding Detection (BFD)**
Katz, Ward. June 2010.
Defines BFD: a lightweight hello protocol for sub-second detection of forwarding path failures. In AI cluster BGP fabrics, BFD with aggressive timers (e.g., 100 ms × 3 = 300 ms detection) provides fast link failure notification to BGP, enabling rapid route convergence and training job recovery. See Chapters 17, 28.
[https://datatracker.ietf.org/doc/html/rfc5880](https://datatracker.ietf.org/doc/html/rfc5880)

---

**RFC 6241 — Network Configuration Protocol (NETCONF)**
Enns, Bjorklund, Schoenwaelder, Bierman. June 2011.
The IETF model-driven configuration protocol. NETCONF uses XML over SSH, with a capability negotiation handshake and a datastore model (candidate/running/startup). The `<edit-config>`, `<get>`, and `<lock>` operations form the basis for transactional network configuration. See Chapter 14.
[https://datatracker.ietf.org/doc/html/rfc6241](https://datatracker.ietf.org/doc/html/rfc6241)

---

**RFC 7311 — The Accumulated IGP Metric Attribute for BGP**
Mohapatra, Fernando, Rosen, Uttaro. August 2014.
Defines the BGP Link Bandwidth extended community used by FRR and GoBGP to encode link capacity. UCMP (Unequal-Cost Multipath) implementations read this community to set weighted ECMP ratios. Referenced in Chapter 27 for UCMP traffic engineering.
[https://datatracker.ietf.org/doc/html/rfc7311](https://datatracker.ietf.org/doc/html/rfc7311)

---

**RFC 7348 — Virtual eXtensible Local Area Network (VXLAN)**
Mahalingam et al. August 2014.
Defines VXLAN encapsulation: a 24-bit VNI embedded in a UDP outer header (dst port 4789) allows layer-2 frames to be tunneled over a layer-3 underlay. VXLAN is the encapsulation substrate for BGP-EVPN overlays in AI cluster fabrics. See Chapters 11, 21.
[https://datatracker.ietf.org/doc/html/rfc7348](https://datatracker.ietf.org/doc/html/rfc7348)

---

**RFC 7432 — BGP MPLS-Based Ethernet VPN (EVPN)**
Sajassi et al. February 2015.
The foundational EVPN RFC. Defines route types 1–4 (Ethernet Auto-Discovery, MAC/IP Advertisement, Inclusive Multicast, Ethernet Segment), the NLRI format, and MAC mobility semantics. BGP-EVPN with VXLAN (RFC 8365) is the dominant DC overlay control plane for AI clusters. See Chapters 11, 21.
[https://datatracker.ietf.org/doc/html/rfc7432](https://datatracker.ietf.org/doc/html/rfc7432)

---

**RFC 7950 — The YANG 1.1 Data Modeling Language**
Bjorklund. August 2016.
YANG is the data modeling language underpinning NETCONF, RESTCONF, and gNMI. RFC 7950 defines YANG 1.1: modules, submodules, containers, lists, leafs, `must`/`when` XPath constraints, and derivation mechanisms. All OpenConfig and vendor YANG models are written in YANG 1.1. See Chapters 14, 15.
[https://datatracker.ietf.org/doc/html/rfc7950](https://datatracker.ietf.org/doc/html/rfc7950)

---

**RFC 8040 — RESTCONF Protocol**
Bierman, Bjorklund, Watsen. January 2017.
Defines RESTCONF: an HTTP/JSON binding for YANG-modeled configuration and operational data. RESTCONF uses REST methods (GET, PUT, PATCH, DELETE) against a resource tree derived from the YANG model. Simpler than NETCONF for ad-hoc queries and web integration. See Chapter 14.
[https://datatracker.ietf.org/doc/html/rfc8040](https://datatracker.ietf.org/doc/html/rfc8040)

---

**RFC 8173 — Precision Time Protocol Version 2 (PTPv2) MIB**
Shankarkumar, Montini, Frost, Dowd. June 2017.
Defines the SNMP Management Information Base for PTPv2 (IEEE 1588-2008). Allows monitoring of PTP clock datasets, port state machines, and offset measurements via SNMP — relevant where gNMI-based PTP telemetry is unavailable. See Chapter 3.
[https://datatracker.ietf.org/doc/html/rfc8173](https://datatracker.ietf.org/doc/html/rfc8173)

---

**RFC 8365 — A Network Virtualization Overlay Solution Using Ethernet VPN (EVPN)**
Sajassi et al. March 2018.
Extends RFC 7432 for VXLAN and NVGRE encapsulations. Defines Type-5 IP Prefix routes (for routing between VXLANs without MAC learning), symmetric IRB (Integrated Routing and Bridging), and asymmetric IRB. The RFC that makes BGP-EVPN/VXLAN a complete overlay solution for multi-tenant AI clusters. See Chapters 11, 21, 30.
[https://datatracker.ietf.org/doc/html/rfc8365](https://datatracker.ietf.org/doc/html/rfc8365)

---

**RFC 8986 — Segment Routing over IPv6 (SRv6) Network Programming**
Filsfils et al. February 2021.
Defines SRv6 behaviors: End (node segment), End.X (adjacency), End.DT4 (decapsulate and forward to IPv4 VRF), and more. SRv6 enables traffic engineering and VPN services using the IPv6 extension header as a programmable forwarding instruction set. Referenced in Chapter 27 for SRv6-based traffic engineering in AI fabrics.
[https://datatracker.ietf.org/doc/html/rfc8986](https://datatracker.ietf.org/doc/html/rfc8986)

---

**RFC 9254 — YANG-Push: Subscribing to YANG Datastore Push Updates**
Veillette et al. October 2022.
Defines YANG-Push: a subscription mechanism for streaming configuration and operational state changes from NETCONF servers. Complements gNMI for NETCONF-capable devices. See Chapter 15.
[https://datatracker.ietf.org/doc/html/rfc9254](https://datatracker.ietf.org/doc/html/rfc9254)

---

## C.2 IEEE Standards

**IEEE 802.1p — Traffic Class Expediting and Dynamic Multicast Filtering**
IEEE Standards Association. 1998 (now integrated into 802.1Q).
Defines the 3-bit Priority Code Point (PCP) field in the 802.1Q VLAN tag, providing 8 traffic classes (0–7) for QoS prioritization. In AI cluster fabrics, PCP values differentiate RoCE RDMA traffic (class 3 or 4), storage traffic, and best-effort management traffic. See Chapters 2, 8.
[https://standards.ieee.org/ieee/802.1Q](https://standards.ieee.org/ieee/802.1Q)

---

**IEEE 802.1Q — Bridges and Bridged Networks**
IEEE Standards Association. 2022 revision.
The foundational standard for VLAN tagging, spanning tree, and bridge operation in Ethernet networks. Modern DC fabrics disable STP and use it primarily for VLAN encapsulation. Contains 802.1p PCP and 802.1Qbb PFC as integrated amendments. See Chapters 2, 8, 11.
[https://standards.ieee.org/ieee/802.1Q](https://standards.ieee.org/ieee/802.1Q)

---

**IEEE 802.1Qbb — Priority-based Flow Control (PFC)**
IEEE Standards Association. 2011 (integrated into 802.1Q).
Defines PFC: per-priority PAUSE frames that allow a receiver to halt transmission of a specific traffic class without affecting others. PFC is a hard requirement for lossless RoCEv2 transport: without it, packet drops trigger RDMA retransmissions that collapse collective communication bandwidth. Key concern: PFC deadlocks under specific fabric topologies. See Chapters 2, 24, 27, 28.
[https://standards.ieee.org/ieee/802.1Qbb](https://standards.ieee.org/ieee/802.1Qbb)

---

**IEEE 1588-2019 — Precision Clock Synchronization Protocol for Networked Measurement and Control Systems (PTPv2.1)**
IEEE Standards Association. 2019.
The governing standard for PTP. Defines the grandmaster clock, boundary clock, transparent clock, the Best Master Clock Algorithm (BMCA), and the Announce/Sync/Follow_Up/Delay_Req/Delay_Resp message sequence. Supersedes IEEE 1588-2008. Sub-microsecond synchronization enabled by hardware timestamping is a hard requirement for ECN-based RoCE congestion control. See Chapter 3.
[https://standards.ieee.org/ieee/1588](https://standards.ieee.org/ieee/1588)

---

## C.3 Industry Specifications & Open Standards

**InfiniBand Architecture Specification (IBTA)**
InfiniBand Trade Association. Volume 1 (General Architecture), current release 1.7.
Defines the complete InfiniBand protocol stack: physical layer, link layer (LRH, BTH, AETH headers), transport layer (RC, UC, UD, XRC QP types), and subnet management (SMA, SA, subnet managers). Required for understanding chapter 24 (InfiniBand fabric management) and the RDMA verbs model in chapter 2.
[https://www.infinibandta.org/ibta-specification/](https://www.infinibandta.org/ibta-specification/)

---

**NVMe Base Specification**
NVM Express Workgroup. Revision 2.1, current.
Defines the NVMe command set, submission/completion queue model, and namespace management for PCIe-attached NVM storage. The foundation for both local NVMe and NVMe-oF. See Chapter 6.
[https://nvmexpress.org/specifications/](https://nvmexpress.org/specifications/)

---

**NVMe over Fabrics (NVMe-oF) Specification**
NVM Express Workgroup. Revision 1.1, current.
Extends NVMe over RDMA (RoCE, iWARP), Fibre Channel, and TCP fabrics. Defines the fabric-specific command set, capsule format, and connection establishment. SPDK implements NVMe-oF targets; see Chapter 6.
[https://nvmexpress.org/specifications/](https://nvmexpress.org/specifications/)

---

**P4₁₆ Language Specification**
P4 Language Consortium. Version 1.2.4.
The specification for the P4 domain-specific language: header types, parsers, match-action tables, actions, externs, package declarations, and the v1model/PSA/PNA portable architectures. See Chapter 9.
[https://p4lang.github.io/p4-spec/docs/P4-16-working-draft.html](https://p4lang.github.io/p4-spec/docs/P4-16-working-draft.html)

---

**P4Runtime API Specification**
P4 Language Consortium. Version 1.3.
The gRPC-based control-plane API for managing P4-programmed forwarding devices: table entry management (`Write`, `Read`), packet-in/out, counter/meter access, and device metadata. See Chapter 9.
[https://p4lang.github.io/p4runtime/spec/main/P4Runtime-Spec.html](https://p4lang.github.io/p4runtime/spec/main/P4Runtime-Spec.html)

---

**gNMI Specification**
OpenConfig Working Group. Version 0.10.0.
Defines the gRPC Network Management Interface: `Get`, `Set`, `Subscribe` RPCs; path encoding; subscription modes (ONCE, POLL, STREAM with TARGET_DEFINED, ON_CHANGE, SAMPLE subtypes). The modern replacement for SNMP in DC fabric monitoring. See Chapters 8, 14, 15.
[https://openconfig.net/docs/gnmi/gnmi-specification/](https://openconfig.net/docs/gnmi/gnmi-specification/)

---

**OpenConfig YANG Models**
OpenConfig Working Group. Ongoing.
Vendor-neutral YANG models for: interfaces, BGP, OSPF, MPLS, platform, LLDP, telemetry, and QoS. SR Linux and SONiC expose OpenConfig paths via gNMI; FRR supports a subset. See Chapters 8, 14, 15.
[https://openconfig.net/projects/models/](https://openconfig.net/projects/models/)

---

**PMIx Standard**
PMIx Administrative Steering Committee. Version 5.0, 2023.
Defines the Process Management Interface for Exascale: job launch, process wire-up, key-value namespace, fence operations, and fault notification. Used by OpenMPI and NCCL launchers to bootstrap distributed jobs across nodes. See Chapter 4.
[https://pmix.org/uploads/2023/05/pmix-standard-v5.0.pdf](https://pmix.org/uploads/2023/05/pmix-standard-v5.0.pdf)

---

**DCQCN — Data Center Quantized Congestion Notification**
Zhu et al. ACM SIGCOMM 2015.
Describes the DCQCN algorithm: the congestion control scheme that makes RoCEv2 practical in lossy Ethernet fabrics. Combines ECN marking at switches, CNP generation at receivers, and a rate-control algorithm at senders. Implementation is split between NIC firmware (rate limiter) and switch ASIC (ECN marking thresholds). See Chapter 2.
[https://conferences.sigcomm.org/sigcomm/2015/pdf/papers/p523.pdf](https://conferences.sigcomm.org/sigcomm/2015/pdf/papers/p523.pdf)

---

**SAI — Switch Abstraction Interface**
Open Compute Project. Version 1.14+.
Defines a vendor-neutral C API for programming switch ASICs: ports, VLANs, routes, ACLs, QoS, and buffers. SONiC's `orchagent` translates intent from Redis ConfigDB into SAI calls, making SONiC portable across Broadcom, Mellanox, Marvell, and Barefoot ASIC vendors. See Chapter 8.
[https://github.com/opencomputeproject/SAI](https://github.com/opencomputeproject/SAI)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).