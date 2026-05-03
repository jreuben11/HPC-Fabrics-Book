# Appendix J — AI Cluster Network Reference Architectures

This appendix provides end-to-end network designs for three GPU cluster scales: 64 GPUs (single rack), 512 GPUs (four-rack pod), and 4096 GPUs (32-rack multi-pod). Each design includes an ASCII topology diagram, port budget table, oversubscription calculation, NIC configuration, storage network, OOB management network, and estimated cable counts. The three designs form a coherent progression — each tier reuses and extends the patterns of the tier below it.

Cross-references point to the relevant chapters and appendices throughout the book.

---

## J.1 Design Principles and Conventions

Before examining individual architectures, the following conventions apply across all three designs.

**Topology progression.** The three cluster sizes represent a deliberate progression in complexity: the 64-GPU single-rack design uses a simple dual-port NIC per server and a single flat ToR switch — no spine, no rail separation. The 512-GPU pod introduces **rail-optimised layout** with one NIC per GPU rail. The 4096-GPU cluster extends this to a three-tier fabric. Each tier can be physically cabled and tested before the next tier is added.

**Rail-optimised layout (512-GPU and 4096-GPU designs).** Each physical GPU has its own dedicated NIC that connects to a single **rail** (a vertical stripe of leaf-switch ports running the same GPU index across all servers). This ensures that all-reduce collectives between same-index GPUs traverse only one ToR switch hop for intra-rack traffic, and at most two hops (leaf → spine → leaf) for inter-rack traffic. The rail topology is described in detail in Chapter 1.

**Oversubscription convention.** The 64-GPU design has no spine, so the oversubscription concept does not apply — the ToR backplane is always non-blocking. In rail-optimised designs (512- and 4096-GPU), intra-rail (intra-rack) traffic is always non-blocking because it never leaves the leaf switch. Cross-rail (inter-rack) traffic at the leaf-to-spine boundary runs at **2:1 oversubscription** (16 downlinks × 400GbE vs. 8 uplinks × 400GbE), which matches Meta's production RoCE design. For workloads requiring strict 1:1 bisection at all scales, a fully non-blocking fat-tree with equal uplinks and downlinks would be required (see Chapter 27). Storage and OOB management networks accept higher oversubscription ratios.

**RoCEv2 as the default transport.** Unless noted, **RoCEv2** (RDMA over Converged Ethernet v2) is the RDMA transport for compute-fabric traffic. **InfiniBand NDR** is introduced as an optional storage-fabric choice at the 4096-GPU scale (see Chapter 24).

**BGP-EVPN overlay.** Where a routed underlay is used (512- and 4096-GPU designs), **BGP-EVPN/VXLAN** is the overlay control plane for storage and management VLANs. See Chapter 11 for configuration detail.

**Switch speed notation.** All speeds are per-port full-duplex line rate. "400GbE" means 400 Gbit/s Ethernet; "400G NDR" means 400 Gbit/s InfiniBand NDR.

---

## J.2 64-GPU Cluster — Single-Rack Design

### J.2.1 Overview

This design is a **flat-fabric** single-rack cluster built around eight 8-GPU servers (for example, H100 SXM or OEM equivalent). All compute NICs connect to a single **Top-of-Rack (ToR)** switch. No spine layer is needed. Storage is converged on the same ToR fabric using **RoCEv2**. This is the simplest possible GPU cluster and is appropriate for a single team's dedicated training workload, a model evaluation cluster, or the foundation rack of a larger phased build.

**Server configuration:** 8 servers × 8 GPUs = **64 GPUs** total. Each server carries one **dual-port 200GbE ConnectX-7** NIC (providing 2 × 200GbE = 400GbE aggregate compute bandwidth per server), giving one compute connection to the ToR per server. At this single-rack scale the NVMe-oF storage array connects directly to the ToR (ports 9–10) rather than through per-server storage NICs, keeping server cabling minimal. This configuration is the standard for 1-rack "starter" clusters before rail-optimised layout is introduced at the 512-GPU scale.

### J.2.2 ASCII Topology

```
  64-GPU Single-Rack Cluster — Flat Fabric
  ══════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────┐
  │    ToR Switch: Arista 7060DX5-32 (32 × 400GbE, 1RU) │
  │                   12.8 Tbps backplane                │
  │                                                      │
  │  Compute  [p01-08]: 8 servers × 1 × 400GbE           │
  │               (via 2 × 200GbE breakout OSFP→2×QSFP)  │
  │  Storage  [p09-10]: 2 × 400GbE → NVMe-oF array       │
  │  Mgmt/uplink [p11]: 1 × 100GbE → in-band mgmt VLAN  │
  │  Headroom [p12-32]: 21 × 400GbE (expansion)          │
  └─────────────────────────────────────────────────────┘
       │  │  │  │  │  │  │  │   (8 × 400GbE DAC)
  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
  │SV1│ │SV2│ │SV3│ │SV4│ │SV5│ │SV6│ │SV7│ │SV8│
  │8G │ │8G │ │8G │ │8G │ │8G │ │8G │ │8G │ │8G │
  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘
   (each server: 1 × dual-port 200GbE NIC → 1 OSFP port on ToR;
    NVLink connects all 8 GPUs within the server at 900GB/s)

  NVMe-oF Storage Array
  ┌─────────────────────────────────────┐
  │  All-flash NVMe array               │
  │  2 × 400GbE uplinks to ToR [p09-10]│
  └─────────────────────────────────────┘

  OOB Management
  ┌─────────────────────┐
  │  1GbE OOB Switch    │  (e.g. Arista 7010TX-48 or similar)
  │  48 × 1GbE RJ45     │
  └─────────────────────┘
       │ 8 × BMC (IPMI/Redfish) + 1 × jump host
```

### J.2.3 Port Budget Table

| Port Group | Speed | Port Count | Purpose |
|---|---|---|---|
| Compute NICs (downlinks) | 400GbE (via 2 × 200G OSFP breakout) | 8 | 8 servers × 1 dual-port 200GbE NIC per server |
| NVMe-oF storage array | 400GbE | 2 | Direct-attach to NVMe array |
| In-band management uplink | 100GbE (breakout) | 1 | Management VLAN, syslog, telemetry |
| Headroom / expansion | 400GbE | 21 | Future: second rack uplinks, additional storage |
| **Total ToR ports used** | | **11 / 32** | Arista 7060DX5-32 (32 × 400GbE, 1RU, 12.8 Tbps) |

> **Switch choice.** The Arista 7060DX5-32 (32 × 400GbE, 1RU) is chosen over the 64-port 7060DX5-64S because 8 servers with one 400GbE port each use only 8 switch ports — a 64-port switch would be 87.5% idle. The 32-port switch provides 21 free ports for storage expansion and future rack growth before a spine is required. When the cluster grows beyond one rack, swap to the 7060DX5-64S as ToR and add a spine tier (see Section J.3).

### J.2.4 Oversubscription Calculation

In a single-rack flat fabric there is no spine layer; therefore the concept of uplink-to-downlink oversubscription is **not applicable**. All compute traffic between servers crosses only the ToR switch backplane at full non-blocking line rate.

```
ToR backplane capacity : 12.8 Tbps (7060DX5-32)
Active compute ports   : 8 × 400 Gbps = 3,200 Gbps = 3.2 Tbps
Utilisation            : 3.2 / 12.8 = 25%  (switch is non-blocking)
```

**Bisection bandwidth:** (8 × 400 Gbit/s) / 2 = **1.6 Tbit/s** (compute fabric only).

**NVLink intra-server bandwidth:** Within each server, all 8 GPUs communicate via **NVLink** at 900 GB/s bidirectional aggregate — far exceeding the 400GbE external link. The external Ethernet fabric is only needed for inter-server and storage traffic.

### J.2.5 Storage Network Design

Storage is **converged** on the compute ToR using **RoCEv2** and **NVMe-oF** (both NVMe/RDMA and NVMe/TCP are supported). The NVMe-oF storage array connects directly to the ToR at 400GbE (ports 9–10). **Priority Flow Control (PFC)** and **DCQCN** congestion control are enabled cluster-wide. Storage traffic is placed in a separate 802.1Q VLAN and DSCP class from compute (**DSCP AF41** for storage, **DSCP CS4** for RoCEv2 compute). See Chapter 18 for storage network isolation patterns.

### J.2.6 OOB Management Network

A dedicated 1GbE switch (48-port, e.g., Arista 7010TX-48) provides out-of-band access to all eight server **BMC** ports (Baseboard Management Controller) via **IPMI v2.0** and **Redfish**. The OOB switch connects upstream to the data centre management network via a 1GbE uplink. A jump host is co-located on the OOB network (see Section J.5.2 for the pattern). **NTP** is provided from the data centre stratum-2 server; PTP is not required at single-rack scale. DNS is served from the jump host or from the data centre.

### J.2.7 Cable Count Estimate

| Cable Type | Count | Use |
|---|---|---|
| DAC passive copper (OSFP→OSFP, ≤3m) | 8 | ToR to server dual-port 200GbE NICs (one cable per server, 400GbE) |
| DAC (OSFP, 400GbE, ≤3m) | 2 | ToR to NVMe-oF storage array |
| DAC (100GbE breakout, ≤2m) | 1 | ToR management port to in-band VLAN uplink |
| RJ45 Cat6 (1GbE) | 9 | BMC (×8) + jump host (×1) to OOB switch |
| Patch cables (reserved) | 8 | Labelled spares for future expansion ports |

---

## J.3 512-GPU Cluster — Four-Rack Pod Design

### J.3.1 Overview

This design scales to **512 GPUs** across four racks using a **two-tier rail-optimised leaf-spine** fabric. It introduces a dedicated storage fabric, an in-band management VRF, and a 10GbE OOB management network. The compute fabric runs at 2:1 oversubscription at the leaf-to-spine boundary (rail-aligned intra-rack traffic is always non-blocking); the storage fabric runs non-blocking (uplinks exceed downlinks). **BGP-EVPN/VXLAN** is the overlay for storage and management VLANs (see Chapter 11).

**Server configuration:** 16 servers per rack × 4 racks = 64 servers × 8 GPUs = **512 GPUs**. Each server carries 8 single-port 400GbE **ConnectX-7** NICs (one per GPU rail, wired to the corresponding rail-leaf switch) plus 2 additional 100GbE ports: one dedicated to the NVMe-oF storage leaf and one to the in-band management VRF. This gives 10 NIC ports per server total.

### J.3.2 ASCII Topology

```
  512-GPU Four-Rack Pod — Rail-Optimised Leaf-Spine
  ═══════════════════════════════════════════════════════════════════════

  Spine Plane (8 × Arista 7060DX5-64S, each 64 × 400GbE)
  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
  │Sp-R0 │ │Sp-R1 │ │Sp-R2 │ │Sp-R3 │ │Sp-R4 │ │Sp-R5 │ │Sp-R6 │ │Sp-R7 │
  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
     │(one spine per GPU rail; 4 active downlinks per spine — one per rack's rail-leaf)
     │
  ───┼─────────────────────────────────────────── (AOC 30m, inter-rack)
     │
  Compute Leaf Switches (8 per rack × 4 racks = 32 leaves)
  Rack-A (example):
  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
  │Lf-A0 │ │Lf-A1 │ │Lf-A2 │ │Lf-A3 │ │Lf-A4 │ │Lf-A5 │ │Lf-A6 │ │Lf-A7 │
  │(Rail0)│ │(Rail1)│ │(Rail2)│ │(Rail3)│ │(Rail4)│ │(Rail5)│ │(Rail6)│ │(Rail7)│
  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
     │  16 × 400GbE downlinks per leaf (to 16 servers in rack via DAC)
     │  8 × 400GbE uplinks per leaf (one to each spine, via AOC)
     │
  ┌──┴──────────────────────────────────────────────────────────┐
  │  16 Servers (Rack A) — each: 8 × 400GbE compute + 2 × 100G │
  │  GPU-0 NIC → Lf-A0, GPU-1 NIC → Lf-A1, ... GPU-7 → Lf-A7  │
  └─────────────────────────────────────────────────────────────┘

  Storage Fabric (1 storage leaf switch per rack = 4 storage leaves total)
  ┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐
  │ StLf-A     │     │ StLf-B     │     │ StLf-C     │     │ StLf-D     │
  │(64×400GbE) │     │(64×400GbE) │     │(64×400GbE) │     │(64×400GbE) │
  │  Rack A    │     │  Rack B    │     │  Rack C    │     │  Rack D    │
  └────────────┘     └────────────┘     └────────────┘     └────────────┘
    ↕ 16 × 100GbE to servers (1 per server)  ↕ 8 × 400GbE uplinks to storage spine

  OOB Management (10GbE switches, one per rack)
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    ┌──────────────┐
  │OOB-A     │ │OOB-B     │ │OOB-C     │ │OOB-D     │───►│ OOB Spine    │
  │10GbE     │ │10GbE     │ │10GbE     │ │10GbE     │    │ (jump host)  │
  └──────────┘ └──────────┘ └──────────┘ └──────────┘    └──────────────┘
    ↕ 1GbE to BMCs (×16 per rack)
```

### J.3.3 Port Budget Table

**Compute Fabric — per rail-leaf switch (Arista 7060DX5-64S, 64 × 400GbE):**

| Port Group | Speed | Ports | Direction |
|---|---|---|---|
| Server compute NICs (one per GPU on this rail) | 400GbE | 16 | Downlink (DAC) |
| Spine uplinks (one per spine switch) | 400GbE | 8 | Uplink (AOC) |
| Reserved | 400GbE | 40 | Headroom / future racks |
| **Total used** | | **24 / 64** | |

> **Scale note.** The 512-GPU design uses only 24 of 64 ports per leaf. This is intentional: the same switch model can grow to support 56 servers on 56 downlinks + 8 uplinks = 64 ports when the pod expands to 896 GPUs (7 racks). The 64-port model is chosen for consistent BOM and to allow rack-by-rack expansion without recabling spines.

**Spine switch port budget (one spine per rail, Arista 7060DX5-64S, 64 × 400GbE):**

| Port Group | Speed | Ports | Direction |
|---|---|---|---|
| Leaf downlinks (one per leaf in this rail, 4 racks × 8 leaves/rack / 8 rails = 4 leaves per spine) | 400GbE | 4 | Downlink (AOC) |
| Reserved for pod expansion (up to 32 additional racks) | 400GbE | 60 | Future |
| **Total used** | | **4 / 64** | |

> **Spine headroom.** At 512-GPU scale the spine switches are lightly loaded. The 7060DX5-64S was chosen here for consistency with the leaf tier; a modular chassis (Arista 7800R4) would be more appropriate for a dedicated spine in larger builds (see Section J.4).

**Storage leaf switch port budget (per rack, Arista 7060DX5-64S):**

| Port Group | Speed | Ports | Direction |
|---|---|---|---|
| Server storage NICs (100GbE via breakout) | 100GbE | 16 | Downlink (DAC breakout) |
| Storage array uplinks | 400GbE | 4 | To storage array (DAC/AOC) |
| Storage spine uplinks (inter-rack) | 400GbE | 8 | Uplink (AOC) |
| Reserved | 400GbE | 36 | Future |
| **Total used** | | **28 / 64** | |

**Cluster totals:**

| Component | Count |
|---|---|
| Compute leaf switches | 32 (8 per rack × 4 racks) |
| Compute spine switches | 8 (1 per GPU rail) |
| Storage leaf switches | 4 (1 per rack) |
| Storage spine switches | 2 (shared across all 4 storage leaves; 4 leaves × 8 uplinks ÷ 16 ports each = 2 switches) |
| OOB management switches | 4 (1 per rack, 10GbE, 48-port) |
| Servers | 64 |
| GPUs | 512 |

### J.3.4 Oversubscription Calculation

**Compute fabric:** Each rail-leaf has 16 downlinks × 400GbE = 6.4 Tbps aggregate downlink and 8 uplinks × 400GbE = 3.2 Tbps aggregate uplink. However, in rail-optimised design, intra-rail traffic (between the 16 servers on the same leaf) does not traverse the spine at all — only inter-rail traffic hits the uplinks. The **effective oversubscription ratio** at the leaf-spine boundary for cross-rail traffic is:

```
Downlink capacity  : 16 × 400 Gbps = 6,400 Gbps
Uplink capacity    : 8  × 400 Gbps = 3,200 Gbps
Uplink:Downlink ratio = 3,200 / 6,400 = 1:2 (2:1 oversubscription)
```

For pure collective operations that are rail-aligned (e.g., ring-allreduce), **0% of traffic crosses the spine** — the spine is idle during that phase. For all-to-all inter-rack collectives the 2:1 ratio applies. This matches Meta's RoCE network design where the RTSW provides "1:2 under-subscribed compared to the RTSW downlink capacity" for fragmented job placement (see Chapter 27 for adaptive routing to manage cross-rail flows).

To achieve a **strictly 1:1 non-blocking** compute fabric, use 8 uplinks and 8 downlinks per leaf (8 downlinks limits the rack to 8 servers — impractical for 64 servers per pod). The 2:1 oversubscription at the spine boundary is the standard production compromise for multi-rack all-to-all collective traffic.

**Storage fabric oversubscription:** 16 servers × 1 × 100GbE downlink = 1,600 Gbps downlink; 8 × 400GbE uplinks = 3,200 Gbps uplink to spine. Storage fabric is **non-blocking** (uplinks exceed downlinks — 2:1 uplink surplus).

**Compute fabric bisection bandwidth:**

Bisection bandwidth is constrained by the number of active spine ports at this cluster size, not the total switch capacity.

```
Active spine ports at 512-GPU scale:
  8 spine switches × 4 active downlinks per spine = 32 active spine links
  Bisection = (32 × 400 Gbps) / 2 = 6,400 Gbps = 6.4 Tbps
```

This is **6.4 Tbps** of effective bisection bandwidth at 512-GPU scale. The same 8 × Arista 7060DX5-64S spine switches have a total capacity of 8 × (64 × 400 Gbps / 2) = 102.4 Tbps — the headroom that enables non-disruptive pod expansion to 32 racks without replacing spine hardware.

### J.3.5 Storage Network Design

A dedicated **NVMe-oF** storage fabric runs on separate leaf switches (one per rack). This prevents storage I/O from interfering with RoCEv2 compute traffic and allows independent PFC/ECN tuning per fabric. Storage leaves connect via 400GbE AOC uplinks to a shared storage spine. All-flash NVMe storage arrays attach to the storage leaves at 400GbE.

The storage VRF is instantiated on the storage leaves, isolated from the compute VRF. NVMe-oF over RDMA (NVMe/RDMA) uses RoCEv2 with DSCP AF41 marking; NVMe-oF over TCP (NVMe/TCP) can share the same ports. See Chapter 18 for storage fabric configuration.

### J.3.6 In-Band Management VRF

A dedicated **in-band management VRF** (VRF `mgmt`) is configured on all compute and storage leaf switches. Each server's management-plane traffic (SSH, syslog, SNMP, gRPC telemetry) is carried within this VRF, isolated from the compute data plane. BGP sessions for eBGP underlay carry a separate address family for the management VRF routes. See Chapter 14 (NETCONF/YANG) and Chapter 15 (gNMI/OpenConfig) for telemetry from these switches.

### J.3.7 OOB Management Network

One **10GbE OOB switch** per rack (48 × 10GbE ports) provides out-of-band BMC access. Server BMC ports (Redfish / IPMI v2.0) connect at 1GbE via the switch's 1GbE access ports. An OOB spine aggregates the four rack switches. A **bastion/jump host** sits on the OOB spine and provides the only ingress point for human operators. See Section J.5 for the OOB topology pattern.

**PTP:** A single **IEEE 1588 Grandmaster** clock (GPS-disciplined, Stratum-0 source) is placed at the OOB spine level. Each compute leaf switch acts as a **Boundary Clock**, terminating the PTP domain and redistributing to servers. This two-tier PTP hierarchy (GM → BC at leaf → ordinary clock at server NIC) achieves sub-microsecond synchronisation across the pod. ConnectX-7 NICs support hardware-assisted PTP timestamping.

### J.3.8 Cable Count Estimate

| Cable Type | Reach | Count | Use |
|---|---|---|---|
| DAC passive copper (OSFP-OSFP) ≤3m | Intra-rack | 512 | Server compute NICs to rail-leaves (16 svr × 8 GPUs × 4 racks) |
| DAC (100GbE breakout) ≤3m | Intra-rack | 64 | Server storage NICs to storage leaves |
| AOC (400GbE) 10–30m | Inter-rack | 256 | Rail-leaf to spine uplinks (32 leaves × 8 uplinks) |
| AOC (400GbE) 10–30m | Inter-rack | 32 | Storage-leaf to storage spine (4 leaves × 8 uplinks) |
| RJ45 Cat6 (1GbE) | Intra-rack | 68 | BMCs (64 servers) + management ports (4 OOB switches) |
| SFP28 (10GbE) | Intra-rack | 20 | OOB switch to OOB spine + jump host |

---

## J.4 4096-GPU Cluster — 32-Rack Multi-Pod Design

### J.4.1 Overview

This design targets **4096 GPUs** in 32 racks using a **three-tier fat-tree** or **rail-optimised Clos** fabric. At this scale, a dedicated InfiniBand storage fabric becomes viable and the OOB management network requires its own BMC switches. **PTP Grandmaster** placement shifts to one GM per 8-rack pod (four GMs total), with boundary clocks at every leaf switch. Adaptive routing (ECMP with flowlet switching) is mandatory to utilise all spine paths efficiently (see Chapter 27). Fault tolerance with **BFD** sub-second failure detection is covered in Chapter 28.

**Server configuration:** 16 servers per rack × 32 racks = 512 servers × 8 GPUs = **4096 GPUs**. Each server carries 8 × 400GbE ConnectX-7 NICs (compute, one per GPU rail) + 1 × 400GbE ConnectX-7 NIC (storage, connecting to the rack storage leaf) + 1 × 100GbE NIC or breakout port (in-band management VRF) + 1 × 1GbE BMC port.

### J.4.2 ASCII Topology

```
  4096-GPU 32-Rack Cluster — Three-Tier Fat-Tree
  ══════════════════════════════════════════════════════════════════════

  Super-Spine / Tier-3 (16 × Arista 7800R4 modular, up to 576 × 800GbE)
  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  ...  ┌────────┐
  │ SS-0   │ │ SS-1   │ │ SS-2   │ │ SS-3   │       │ SS-15  │
  │ 7800R4 │ │ 7800R4 │ │ 7800R4 │ │ 7800R4 │       │ 7800R4 │
  └──┬─────┘ └──┬─────┘ └──┬─────┘ └──┬─────┘       └──┬─────┘
     │           │           │           │               │
     └────────────┴─────────────────────────────────────┘
               SR8/DR8 fibre (MPO-24 trunks, 500m reach)

  Spine / Tier-2 (32 × Arista 7060DX5-64S, 64 × 400GbE each, 8 per pod × 4 pods)
  Pod-0 (Racks 0–7):
  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
  │Sp-0  │ │Sp-1  │ │Sp-2  │ │Sp-3  │ │Sp-4  │ │Sp-5  │ │Sp-6  │ │Sp-7  │
  │Rail0 │ │Rail1 │ │Rail2 │ │Rail3 │ │Rail4 │ │Rail5 │ │Rail6 │ │Rail7 │
  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
     │  (SR8 fibre, ≤100m, to leaves)    │ (SR8/DR8 to super-spine)
  Leaf / Tier-1 (8 leaves per rack × 32 racks = 256 compute leaves)
  Rack-0:
  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
  │Lf-0  │ │Lf-1  │ │Lf-2  │ │Lf-3  │ │Lf-4  │ │Lf-5  │ │Lf-6  │ │Lf-7  │
  │R0    │ │R1    │ │R2    │ │R3    │ │R4    │ │R5    │ │R6    │ │R7    │
  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘
     │  (16 × 400GbE DAC downlinks to servers)
  Servers (16 per rack × 32 racks = 512 servers):
  ┌────────────────────────────────────────────────────────────┐
  │  SV-00 to SV-15  (Rack 0)  — 8 × 400G compute NICs        │
  │                               1 × 400G storage NIC         │
  │                               1 × 100GbE in-band mgmt port │
  │                               1 × 1GbE BMC                 │
  └────────────────────────────────────────────────────────────┘

  Storage Fabric (Option A: RoCEv2 Converged — separate leaf tier)
  ┌─────────────────────────────────────────────────────────────┐
  │ 32 × Storage Leaf (Arista 7060DX5-64S, one per rack)       │
  │ 16 × Storage Spine (Arista 7060DX5-64S, 4 per pod × 4 pods) │
  │ All-flash NVMe arrays: 400GbE uplinks to storage leaves     │
  └─────────────────────────────────────────────────────────────┘

  Storage Fabric (Option B: InfiniBand NDR — dedicated IB fabric)
  ┌─────────────────────────────────────────────────────────────┐
  │ 32 × IB Leaf (NVIDIA QM9700, 64 × NDR 400Gb/s, per rack)   │
  │  8 × IB Spine (NVIDIA QM9700, 2 per pod × 4 pods)           │
  │ NVMe-oF over InfiniBand (NVMe/RDMA)                         │
  └─────────────────────────────────────────────────────────────┘

  OOB Management (dedicated BMC switches, one per rack)
  ┌──────────┐  ┌──────────┐  ...  ┌──────────┐    ┌──────────────┐
  │BMC-SW-0  │  │BMC-SW-1  │       │BMC-SW-31 │───►│ OOB Spine    │
  │48×1GbE   │  │48×1GbE   │       │48×1GbE   │    │(10GbE agg.)  │
  └──────────┘  └──────────┘       └──────────┘    └──────┬───────┘
                                                           │
                                                    ┌──────┴───────┐
                                                    │  Jump Host   │
                                                    │  Bastion     │
                                                    └──────────────┘
  PTP: one Grandmaster per 8-rack pod (4 GMs total)
  GM → Boundary Clock at each Tier-1 leaf → Ordinary Clock at server NIC
```

### J.4.3 Port Budget Table

**Compute leaf switch (Arista 7060DX5-64S, 64 × 400GbE) — per switch:**

| Port Group | Speed | Ports | Direction |
|---|---|---|---|
| Server compute NICs (1 per GPU on this rail) | 400GbE | 16 | Downlink (DAC, ≤3m) |
| Pod spine uplinks (1 per pod-spine switch) | 400GbE | 8 | Uplink (SR8, ≤100m) |
| Reserved / expansion | 400GbE | 40 | Future |
| **Total used** | | **24 / 64** | |

**Pod spine switch (Arista 7060DX5-64S, 64 × 400GbE) — per switch (one per rail per pod):**

| Port Group | Speed | Ports | Direction |
|---|---|---|---|
| Leaf downlinks (8 racks × 1 leaf per rail per rack) | 400GbE | 8 | Downlink (SR8) |
| Super-spine uplinks | 400GbE | 16 | Uplink (SR8/DR8) |
| Reserved | 400GbE | 40 | Future pods |
| **Total used** | | **24 / 64** | |

**Super-spine switch (Arista 7800R4 modular) — per switch:**

| Port Group | Speed | Ports | Direction |
|---|---|---|---|
| Pod-spine downlinks (32 T2 spines × 16 uplinks each / 16 super-spines = 32 links per SS) | 400GbE | 32 | Downlink (DR8 fibre) |
| Reserved | 400GbE | 544+ | Expansion |
| **Total used** | | **32 / 576** | |

> **7800R4 note.** The Arista 7800R4 supports up to 576 × 800GbE (or 1152 × 400GbE) in a full chassis configuration. At 4096-GPU scale, a lightly loaded 7800R4 provides massive headroom for growth to 32,768+ GPUs without replacing spine hardware.

**Storage fabric (Option A — RoCEv2) per storage leaf (Arista 7060DX5-64S):**

| Port Group | Speed | Ports | Direction |
|---|---|---|---|
| Server storage NICs (1 per server × 16 servers) | 400GbE | 16 | Downlink (DAC) |
| Storage spine uplinks | 400GbE | 16 | Uplink (AOC/SR8) |
| NVMe array direct attach | 400GbE | 8 | To storage arrays |
| Reserved | 400GbE | 24 | |
| **Total used** | | **40 / 64** | |

**Cluster component totals:**

| Component | Count | Notes |
|---|---|---|
| Compute leaf switches (Tier-1) | 256 | 8 per rack × 32 racks |
| Pod spine switches (Tier-2) | 32 | 8 per pod × 4 pods (1 per GPU rail) |
| Super-spine switches (Tier-3) | 16 | 2 per pod or globally shared |
| Storage leaf switches (RoCEv2 option) | 32 | 1 per rack |
| Storage spine switches | 16 | 4 per pod |
| IB leaf switches (IB option) | 32 | QM9700, 1 per rack |
| IB spine switches (IB option) | 8 | QM9700, 2 per pod |
| BMC OOB switches | 32 | 1 per rack, 48 × 1GbE |
| OOB aggregation switches | 4 | 1 per 8-rack pod |
| PTP Grandmaster clocks | 4 | 1 per 8-rack pod |
| Servers | 512 | |
| GPUs | 4096 | |

### J.4.4 Oversubscription Calculation

**Tier-1 leaf to Tier-2 spine (per rail-leaf):**

```
Downlinks : 16 × 400 Gbps = 6,400 Gbps
Uplinks   :  8 × 400 Gbps = 3,200 Gbps
Oversubscription = 6,400 / 3,200 = 2:1
```

Rail-aligned collective traffic does not cross Tier-2. For inter-rack all-to-all patterns, the 2:1 ratio at Tier-1 applies.

**Tier-2 spine to Tier-3 super-spine (per pod-spine):**

```
Downlinks (from leaves) :  8 × 400 Gbps = 3,200 Gbps
Uplinks (to super-spine): 16 × 400 Gbps = 6,400 Gbps
Oversubscription = 3,200 / 6,400 = 1:2 (uplinks exceed downlinks)
```

Tier-2 to Tier-3 is **non-blocking** — the super-spine has more capacity than the pod-spine injects.

**Compute fabric bisection bandwidth (full cluster):**

```
16 super-spines × (32 active downlinks × 400 Gbps) / 2
= 16 × 6,400 Gbps
= 102.4 Tbps total bisection bandwidth
```

**InfiniBand storage fabric oversubscription (Option B, QM9700):**

Each QM9700 IB leaf has 64 NDR ports: 16 downlinks to servers (400 Gb/s each) and 16 uplinks to IB spine (400 Gb/s each), with 32 ports reserved.

```
IB leaf downlinks : 16 × 400 Gb/s = 6,400 Gb/s
IB leaf uplinks   : 16 × 400 Gb/s = 6,400 Gb/s
Oversubscription  : 1:1 (non-blocking)
```

### J.4.5 Storage Network: RoCEv2 vs InfiniBand Decision Matrix

| Criterion | RoCEv2 Converged (Option A) | InfiniBand NDR (Option B) |
|---|---|---|
| **Latency** | ~1–2 µs (hardware) | ~0.6 µs (hardware, QM9700) |
| **Throughput** | 400GbE per port | 400 Gb/s NDR per port |
| **Integration** | Unified Ethernet fabric, SONiC/EOS | Separate IB fabric, UFM management |
| **Hardware cost** | Lower (commodity Ethernet) | Higher (IB-specific switches) |
| **Operational model** | Single NOC toolchain | Two toolchains (Ethernet + IB) |
| **RDMA transport** | RoCEv2 (requires PFC/ECN tuning) | IB native (lossless by design) |
| **Storage software** | NVMe/RDMA or NVMe/TCP | NVMe/RDMA (native IB) |
| **Best for** | Greenfield Ethernet-first deployments | Mixed IB compute + storage deployments |

See Chapter 24 for InfiniBand fabric management with NVIDIA UFM.

### J.4.6 OOB Management Network

Each rack has a dedicated **BMC switch** (48 × 1GbE, e.g., Arista 7010TX-48) connecting all 16 server BMC ports plus the BMC ports of the leaf and storage switches. The BMC switches aggregate to a per-pod **OOB aggregation switch** (10GbE uplinks). A **bastion host** on the OOB aggregation network is the single ingress point for Redfish automation, IPMI v2.0 console, and emergency firmware updates.

The OOB network carries:
- **Redfish** (HTTPS/REST) for automated lifecycle management (power, firmware, inventory)
- **IPMI v2.0 over LAN** (legacy, access-controlled) for KVM-over-IP console
- **SNMPv3** traps from OOB switches to the NMS
- **NTP** distributed from a stratum-2 server on the OOB spine

### J.4.7 PTP Grandmaster Placement

One **IEEE 1588 Grandmaster** (GNSS-disciplined oscillator, e.g., Meinberg LANTIME M1000 or Microchip TimeProvider 4100) is placed at the OOB aggregation switch level for each 8-rack pod, for **four GMs** in a 32-rack cluster.

```
GNSS antenna → PTP Grandmaster (Stratum-0)
   └── OOB aggregation switch (boundary clock, per pod)
        └── Compute leaf switches (boundary clocks, Tier-1)
             └── Server NICs (ordinary clocks, hardware timestamping)
```

Boundary clocks at each Tier-1 leaf switch terminate the PTP domain and re-distribute independently, limiting the impact of PDV (packet delay variation) accumulated over the three-tier fabric. This pattern ensures sub-microsecond synchronisation required for **SHARP** (Scalable Hierarchical Aggregation and Reduction Protocol) in-network compute and for distributed training checkpoint consistency.

### J.4.8 Cable Count Estimate

| Cable Type | Reach | Count | Use |
|---|---|---|---|
| DAC passive copper (OSFP) ≤3m | Intra-rack | 4096 | Server compute NICs to Tier-1 rail-leaves |
| DAC passive copper ≤3m | Intra-rack | 512 | Server storage NICs to storage leaves (512 servers × 1 storage NIC) |
| SR8 multimode MPO (≤100m) | Intra-pod | 2048 | Tier-1 leaves to Tier-2 pod-spines (256 leaves × 8 uplinks) |
| SR8 multimode MPO (≤100m) | Intra-pod | 512 | Tier-2 pod-spines to Tier-3 super-spines (32 spines × 16 uplinks) |
| DR8 single-mode MPO (≤500m) | Inter-pod/building | 128 | Tier-3 super-spines to cross-pod super-spines or inter-building DCI |
| AOC 30m | Intra-pod storage | 512 | Storage leaf to storage spine |
| RJ45 Cat6 (1GbE) | Intra-rack | 544 | BMC ports (512 servers + 32 switch management) to BMC switches |
| SFP28 10GbE | OOB uplinks | 36 | BMC switches to OOB agg switches (32) + jump hosts (4) |

---

## J.5 Management Plane Topology

### J.5.1 OOB Network Design

The **Out-of-Band (OOB) management network** is physically separate from the data plane at all three cluster scales. This isolation ensures that network engineers can always reach server BMCs, switch management ports, and the jump host even if the compute fabric is completely down.

```
  OOB Network Hierarchy (512-GPU and 4096-GPU designs)

  Internet / Corporate WAN
       │ (VPN / dedicated circuit)
       ▼
  ┌──────────────────────────────────┐
  │  Jump Host / Bastion             │
  │  • Dual-homed: OOB + corp net    │
  │  • SSH gateway with MFA          │
  │  • Ansible / Redfish automation  │
  │  • Audit logging                 │
  └──────────────────────────────────┘
       │ (10GbE)
       ▼
  ┌──────────────────────────────────┐
  │  OOB Aggregation Switch          │
  │  • Routes Redfish / IPMI VLANs   │
  │  • NTP server (stratum-2)        │
  │  • DNS resolver for BMC names    │
  │  • PTP Grandmaster (GPS source)  │
  └──────────────────────────────────┘
       │ (10GbE uplinks per rack OOB switch)
  ┌────┴────────────────────────────┐
  │  Per-Rack OOB / BMC Switch      │
  │  48 × 1GbE access ports         │
  │  • Server BMCs (Redfish/IPMI)   │
  │  • Leaf switch management ports │
  │  • OOB VLAN: 192.168.x.0/24     │
  └─────────────────────────────────┘
```

**Static IP allocation.** Each BMC receives a static IP address in a dedicated **OOB VLAN** (RFC 1918 space, e.g., `192.168.0.0/16` subdivided per rack). DHCP is not recommended for BMC addresses in production — a static assignment linked to asset management is preferred.

**DNS.** A pair of authoritative DNS servers on the OOB aggregation network provides forward and reverse lookups for all BMC and switch management plane addresses. Name format: `<role>-<rack>-<unit>.oob.<cluster-domain>` (e.g., `svr-a-04.oob.cluster1.example.com`).

### J.5.2 Bastion / Jump Host Pattern

All human operator and automation traffic enters the cluster through a **bastion host** that is the only host with both OOB network access and external (corporate/VPN) access. The bastion enforces:

- Multi-factor authentication (SSH key + TOTP)
- Session recording and audit logging
- Redfish/IPMI access proxied through the bastion (no direct BMC exposure to corporate network)
- Ansible / Terraform automation runner co-located on bastion

### J.5.3 In-Band Management VRF

In 512-GPU and 4096-GPU designs, a **management VRF** (`VRF mgmt`) is configured on all compute leaf and spine switches. Switch management traffic (SSH to switch, gNMI telemetry, SNMP) is isolated from the compute and storage VRFs in the routing table. This means:

- A routing leak between the compute fabric and the management VRF requires explicit policy.
- gNMI streaming telemetry (see Chapter 15) can reach the telemetry collector via the management VRF without a separate physical cable.
- The management VRF is advertised via BGP with a separate route-distinguisher and communities.

---

## J.6 Cable and Fibre Planning

### J.6.1 Cable Technology Selection by Reach and Tier

| Tier | Typical Distance | Recommended Cable | Notes |
|---|---|---|---|
| Server NIC → ToR (Tier-1 leaf) | ≤2m intra-rack | **DAC** (passive copper, OSFP/QSFP-DD) | Zero optical power budget; cheapest at scale |
| ToR → storage leaf (intra-rack) | ≤3m | DAC (breakout 400G → 4×100G) | Same rack, different switch |
| Tier-1 leaf → Tier-2 spine (intra-pod, ≤30m) | 5–30m | **AOC** (Active Optical Cable, OSFP) | Eliminates optical transceiver management |
| Tier-1 leaf → Tier-2 spine (intra-pod, ≤100m) | 30–100m | **SR8** (8×50G lanes, OM4/OM5 multimode, MPO-16) | Standard for in-building pod connections |
| Tier-2 → Tier-3 super-spine (inter-pod, ≤500m) | 100–500m | **DR8** (8×50G lanes, OS2 single-mode, MPO-16) | Covers cross-building or MDA-to-HDA runs |
| DCI / inter-building | >500m | **ER8 / ZR** (single-mode, DWDM) | Not in scope for this appendix |
| OOB (BMC, switch management) | ≤100m intra-rack | RJ45 Cat6 (1GbE or 10GbE) | Copper structured cabling |

### J.6.2 MPO Trunk Routing

At the 4096-GPU scale, **MPO-24 or MPO-32 trunk cables** are pre-installed in the overhead cable trays between racks and are then broken out to MPO-16 or MPO-12 fan-outs at patch panels (the **Main Distribution Area, MDA**). This approach:

- Reduces the number of individual cables routed in cable trays (a 32-rack cluster has 4096+ inter-rack fibres; trunking to MPO-24 reduces tray occupancy by 12×).
- Enables **modular recabling** — a trunk with a failed fibre can be replaced without affecting adjacent trunks.
- Supports **structured cabling** convention: all inter-rack connections terminate at the rack's patch panel, not directly at the switch ports.

**Fibre colour coding (TIA-598-D convention):**

| Cable Type | Jacket Colour |
|---|---|
| OS2 single-mode (DR8, ER8) | Yellow |
| OM4 50µm multimode (SR8) | Aqua (or magenta for OM4) |
| OM5 wideband multimode | Lime green |
| Copper DAC | Black |
| AOC | Vendor-specific (often orange) |

### J.6.3 Cabling Discipline

- All DAC cables should be ordered in lengths rounded to 0.5m increments and labelled at both ends with the port identifiers of the switch and server.
- Spare fibre counts: plan for **10–15% spare** fibres in each trunk (e.g., an MPO-32 trunk carries 24 active fibres and 8 spares).
- For each spine port that is not yet connected during a phased deployment, install a **fibre dust cap** and a blank face-plate label with the intended destination switch and port.

---

## J.7 Cross-Reference Summary

| Topic | Location in This Book |
|---|---|
| Fat-tree, rail-optimised topology fundamentals | Chapter 1 |
| ECMP and adaptive routing for AI collectives | Chapter 27 |
| BFD and fault tolerance on compute fabric | Chapter 28 |
| BGP-EVPN/VXLAN overlay configuration | Chapter 11 |
| Storage network isolation (NVMe-oF) | Chapter 18 |
| InfiniBand fabric management (UFM) | Chapter 24 |
| gNMI/OpenConfig streaming telemetry from switches | Chapter 15 |
| NETCONF/YANG for switch configuration | Chapter 14 |
| Containerlab lab topologies (single-rack, 4-rack) | Appendix A |
| NIC and switch hardware selection tables | Appendix E |

---

## J.8 References

The following vendor reference architectures and industry papers informed this appendix.

- [NVIDIA DGX SuperPOD Reference Architecture (DGX H100)](https://docs.nvidia.com/dgx-superpod/reference-architecture-scalable-infrastructure-h100/latest/network-fabrics.html)
- [NVIDIA DGX SuperPOD Reference Architecture (DGX B200)](https://docs.nvidia.com/dgx-superpod/reference-architecture-scalable-infrastructure-b200/latest/network-fabrics.html)
- [NVIDIA DGX BasePOD Deployment Guide](https://docs.nvidia.com/dgx-basepod/deployment-guide-dgx-basepod/latest/network-overview.html)
- [Meta Engineering: RoCE Networks for Distributed AI Training at Scale](https://engineering.fb.com/2024/08/05/data-center-engineering/roce-network-distributed-ai-training-at-scale/)
- [Meta Engineering: Building Meta's GenAI Infrastructure](https://engineering.fb.com/2024/03/12/data-center-engineering/building-metas-genai-infrastructure/)
- [OCP Open Systems for AI — Reference Architectures](https://www.opencompute.org/projects/open-systems-for-ai)
- [Arista 7060X5 Series Data Sheet](https://www.arista.com/en/products/7060x5-series)
- [Arista 7800R4 Series Data Sheet](https://www.arista.com/en/products/7800r4-series)
- [NVIDIA Quantum-2 QM9700 NDR InfiniBand Switch Data Sheet](https://www.naddod.com/products/nvidia-networking/102404)
- [NVIDIA Spectrum-4 SN5600 Data Sheet](https://hardwarenation.com/product/nvidia-spectrum-4-sn5600-800g-64-port-51-2tb-s-2u-data-center-switch/)
- [Juniper Networks: Designing Data Centers for AI Clusters](https://www.juniper.net/documentation/us/en/software/nce/ai-clusters-data-center-design/ai-clusters-data-center-design.pdf)
- [CloudSwitch GPU Backend Fabric Design Guide](https://cloudswit.ch/whitepapers/gpu-backend-fabric-design-guide/)
- [Enfabrica: Scaling to 100K+ GPU AI Clusters Using Flat 2-Tier Networks](https://blog.enfabrica.net/scaling-to-100k-gpu-ai-clusters-using-flat-2-tier-network-designs-a9464d587f38)
- [IEEE 1588 Precision Time Protocol — Meinberg Global](https://blog.meinbergglobal.com/category/ieee1588/)
- [Microchip TimeProvider 4100 PTP Grandmaster](https://www.microchip.com/en-us/products/clock-and-timing/systems/ptp-grandmaster-clocks)

---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
