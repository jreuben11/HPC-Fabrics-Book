# Appendix E — Hardware Selection Guide

This appendix provides a practical reference for infrastructure engineers selecting network hardware for AI compute clusters. Decisions at the NIC, switch ASIC, and cable layer have multi-year consequences for performance, cost, and operational complexity. The guidance here targets clusters ranging from a single rack of H100s to multi-thousand-GPU fabrics.

---

## E.1 NIC / HCA Selection

Network interface cards and Host Channel Adapters are the first hop in any AI cluster fabric. The choice determines which RDMA transport is available, what offload capabilities can be leveraged, and how tightly the fabric couples to GPU-side software (NCCL, RCCL, SHARP). The table below compares the leading options as of 2025–2026.

### E.1.1 Comparison Table

| Vendor / Model | Form Factor | Port Speed | RDMA Type | Key Feature | AI Use Case |
|---|---|---|---|---|---|
| NVIDIA ConnectX-7 (CX7) | PCIe Gen5 x16 HHHL or OCP 3.0 | 200GbE / 400GbE / NDR IB (400 Gb/s) | RoCEv2, InfiniBand | VPI (Ethernet + IB in one card), ASAP2 hardware offload, GPUDirect RDMA, SHARP client | Best-in-class HCA for H100/H200 clusters; drop-in for both IB and RoCE fabrics |
| NVIDIA BlueField-3 DPU | PCIe Gen5 x16 full-height | 200GbE / 400GbE / NDR IB | RoCEv2, InfiniBand | All CX7 features + 16× Arm A78 cores (SNAP, DOCA), hardware root-of-trust, isolated data path | Storage offload (NVMe-oF), network isolation in multi-tenant clusters, DPU-accelerated encryption |
| Intel E810 (Columbiaville) | PCIe Gen4 x16 | 10/25/100GbE | None (no native RDMA) | CVL ASIC with P4-influenced DDP pipeline, DPDK PMD, 1 PPS PTP hardware timestamping, SR-IOV 256 VFs | Programmable pipeline research, eBPF/XDP offload testing, non-RDMA east-west at 100GbE |
| Broadcom Thor2 / BCM57608 | PCIe Gen5 x16 | 100/200/400GbE | RoCEv2 | Cost-competitive, high VF density (512 VFs), RDMA over standard Ethernet, RoCEv2 DCQCN hardware | Hyperscaler cost-optimized deployments; large-scale CPU inference or secondary storage fabric |
| Marvell FastLinQ QL45000 | PCIe Gen4 x16 | 100/200GbE | RoCEv2 | Dual-port, iSCSI/NVMe-oF hardware offload, strong NAS integration | Storage-heavy clusters (Weka, VAST); training clusters where shared storage fabric is the NIC |

### E.1.2 Selection Criteria

**Bandwidth envelope.** For NDR InfiniBand (400 Gb/s per port), only CX7 and BF3 are available. For 400GbE RoCEv2, CX7, BF3, and Thor2 are choices. Intel E810 tops at 100GbE and lacks RDMA.

**RDMA transport.** If the fabric is InfiniBand (chapter 2), the choice is CX7 or BF3. If the fabric is Ethernet RoCEv2 (the more common hyperscaler choice), CX7, BF3, Thor2, and FastLinQ are all valid.

**GPUDirect RDMA.** All NVIDIA NICs support GPUDirect RDMA natively. Thor2 requires `nvidia-peermem` kernel module loaded. E810 does not support GPUDirect RDMA.

**Programmability.** If you need a P4-programmable NIC-side pipeline or custom packet processing, the E810 CVL ASIC and CX7/BF3 DOCA are the two paths; they are architecturally distinct (E810 uses a P4-influenced DDP pipeline programmed via Intel P4 Studio, BF3 is Arm cores + DOCA C SDK).

### E.1.3 Decision Tree

```
Need InfiniBand (NDR/HDR)?
  YES → NVIDIA CX7 (or BF3 if offload also needed)
  NO  → Ethernet RoCEv2 path:
          Need DPU offload / tenant isolation?
            YES → NVIDIA BlueField-3
            NO  → Cost-sensitive hyperscaler?
                    YES → Broadcom Thor2 / BCM57608
                    NO  → Storage-heavy workload?
                            YES → Marvell FastLinQ
                            NO  → Programmable pipeline / eBPF XDP?
                                    YES → Intel E810
                                    NO  → NVIDIA CX7 (safest default for H100 clusters)
```

### E.1.4 Driver and Software Ecosystem Notes

- **NVIDIA CX7 / BF3**: Requires MOFED (Mellanox OpenFabrics Enterprise Distribution) or the upstream `mlx5` kernel driver. MOFED is recommended for production: it bundles perftools (`ibv_rc_pingpong`, `ib_write_bw`), diagnostics (`ibdiagnet`, `ibstat`), and SHARP client libraries. DOCA SDK is required for BF3 application development.
- **Intel E810**: Upstream `ice` driver is production-ready since Linux 5.14. DPDK `ice` PMD available from DPDK 20.11. XDP/AF_XDP support is excellent; full P4 pipeline via Intel P4 Studio (closed-source).
- **Broadcom Thor2**: `bnxt_en` kernel driver. DPDK `bnxt` PMD. RDMA via `bnxt_re` kernel module.
- **Marvell FastLinQ**: `qede` kernel driver; RDMA via `qedr`.

### E.1.5 Emerging NIC Technologies

**AMD Pensando Elba DPU**

AMD's acquisition of Pensando brought the Elba DPU into the AI networking landscape. The Elba ASIC provides 100GbE dual-port connectivity with a P4-programmable pipeline implemented on a dedicated packet processing subsystem. The AMD ROCm software stack has not yet achieved the same level of GPUDirect RDMA integration as NVIDIA MOFED, but the hardware path is present via the PCIe peer-to-peer BAR mapping mechanism. Elba is deployed in several hyperscaler secure-host environments where the DPU enforces micro-segmentation independently of the host OS.

**Fungible / Microsoft DPU**

The Fungible F1 DPU (now owned by Microsoft) introduced a torus-topology interconnect between multiple ASIC dies, targeting NVMe-oF storage acceleration at 100+ GB/s. While not a general-purpose AI training NIC, the F1 DPU demonstrates the direction of converged compute+network+storage accelerators that are likely to influence the next generation of AI cluster endpoint hardware.

**Google Titanium**

Google's internal Titanium network cards (used in TPU Pod clusters and Google Cloud A3 VMs) are custom ASICs co-designed with the host GPU interconnect. Titanium supports Google's Jupiter fabric protocol and provides hardware acceleration for distributed training collectives. This is referenced as a hyperscaler custom silicon example; it is not available as a commercial product but illustrates the performance gap that commodity NICs must close.

### E.1.6 Key NIC Specifications Explained

**VPI (Virtual Protocol Interconnect)**: NVIDIA's term for a single physical NIC that can operate as either an InfiniBand HCA or a 100/200/400GbE NIC. The port mode is set in firmware. This allows a single SKU to serve both IB and Ethernet fabrics, simplifying spares management for organizations operating both fabric types.

**ASAP2 (Accelerated Switching And Packet Processing)**: NVIDIA's NIC-side technology for offloading OVS (Open vSwitch) flow tables to hardware. In AI cluster contexts, ASAP2 is used to accelerate VM/container networking overlay (VXLAN encap/decap) at line rate without software datapath involvement, freeing host CPU cycles for ML workloads.

**SR-IOV Virtual Functions**: All production AI NICs support SR-IOV. For Kubernetes AI clusters, the SR-IOV Network Operator provisions VFs and assigns them to pods via device plugin. NIC VF count matters: CX7 supports 254 VFs per physical function; Thor2 supports 512 VFs. For large multi-tenant nodes running many containers, VF count can be a limiting factor.

**RSS (Receive Side Scaling) Queues**: The number of hardware receive queues determines how many CPU cores can process inbound traffic in parallel. For RDMA workloads, RSS is less critical (RDMA bypasses the kernel receive path) but matters for management traffic (BGP, gNMI, SSH) sharing the same NIC.

---

## E.2 Switch ASIC Comparison

Switch silicon determines fabric bandwidth, buffer profile, routing feature set, and in-network compute capabilities (SHARP). The choice of ASIC also constrains which NOS options are available and what telemetry granularity is achievable.

### E.2.1 Comparison Table

| ASIC | Primary Vendor NOS | Total BW | Representative Port Config | Buffer | P4-Programmable | ECMP | Adaptive Routing | RDMA/AI Features |
|---|---|---|---|---|---|---|---|---|
| Broadcom Tomahawk 5 (BCM78900) | SONiC, Arista EOS, Cisco NX-OS | 51.2 Tbps | 128× 400GbE or 64× 800GbE | Shallow (~64 MB on-chip) | No | 64K groups, 16K paths | No native AR | ECN marking, PFC support, standard RoCE fabric |
| Broadcom Trident 4 (BCM56990) | SONiC, Cumulus, Arista EOS | 12.8 Tbps | 128× 100GbE or 32× 400GbE | Deep (up to 200 MB ext) | No (limited FlexPort) | 16K groups | No native AR | Strong ACL/QoS, PFC, good ToR for RoCE leaf |
| Intel Tofino 2 | ONF Stratum, custom | 12.8 Tbps | 32× 400GbE | Shallow (on-chip only) | Fully P4-programmable | Configurable | Implementable via P4 | INT natively; RDMA awareness implemented in P4 |
| NVIDIA Spectrum-4 (SN5000 series) | NVIDIA Cumulus, SONiC, NVIDIA NOS | 51.2 Tbps | 64× 800GbE or 128× 400GbE | Deep (up to 64 MB shared + headroom) | No (fixed pipeline but rich telemetry API) | 128K groups, 128K paths | Native hardware AR | SHARP v3 in-network aggregation, SRD transport, RoCEv2 full hardware support, DSCP-aware PFC |
| Marvell Prestera (AC5X / Aldrin2) | Marvell Kinetic / SONiC | 3.2–6.4 Tbps | 32–64× 100GbE | Moderate | No | Standard | No | PFC support, cost-effective, limited AI-specific features |

### E.2.2 ASIC Selection by Fabric Role

**Spine layer (high-radix, high-bandwidth)**

The spine must sustain full bisection bandwidth and survive link failures without traffic disruption. Two leading choices:

- **Broadcom Tomahawk 5**: Industry default. Enormous installed base means NOS support is universal (SONiC, Arista EOS, Cumulus, Cisco NX-OS). Shallow buffers are acceptable at spine if leaf switches absorb burst. ECMP to 64K groups covers any realistic fat-tree.
- **NVIDIA Spectrum-4**: Preferred when the compute fabric is InfiniBand or RoCEv2 and you want SHARP in-network reduction. Adaptive Routing at hardware line rate is uniquely valuable for AI workloads where traffic matrices change every collective operation. Deeper buffer improves flow completion time under incast.

**Leaf / ToR layer (server-facing, absorbs burst)**

- **Broadcom Trident 4**: The most deployed leaf ASIC. Deep external packet buffer handles GPU-side incast. FlexPort supports mixed 100G/400G server-facing and 400G uplink. SONiC on Trident4 is production-proven at hyperscale.
- **NVIDIA Spectrum-4 in leaf mode**: For pure IB fabrics, the Quantum-3 HDR/NDR switches (Quantum ASIC, not Spectrum) are the leaf; Spectrum-4 is the Ethernet equivalent and supports SHARP even at leaf.

**Programmable research / INT fabric**

- **Intel Tofino 2**: The only shipping fully P4-programmable merchant silicon at 12.8 Tbps (as of 2025). If you are building INT-based telemetry infrastructure, custom load balancing, or in-network ML (as described in Chapter 9), Tofino 2 is the only option without going to custom ASICs. ONF Stratum NOS or custom ONOS P4Runtime integration required.

**IB/RoCE with SHARP aggregation**

- **NVIDIA Spectrum-4 / Quantum-3**: SHARP (Scalable Hierarchical Aggregation and Reduction Protocol) requires NVIDIA switch silicon. If AllReduce performance at scale is the primary objective and cost is secondary, a full NVIDIA IB stack (CX7 NICs + Quantum-3 HDR/NDR switches) or RoCE stack (CX7 + Spectrum-4) gives the most optimized collective performance through in-network compute.

### E.2.3 ASIC Telemetry Capabilities

Telemetry is increasingly a first-class requirement for AI cluster networking, where understanding collective communication behavior requires sub-second per-flow visibility.

| ASIC | Streaming Telemetry | INT Support | gNMI | Per-Flow Counters | Queue Depth Visibility |
|---|---|---|---|---|---|
| Tomahawk 5 | Yes (via NOS) | No | Via NOS (SONiC/Arista) | Port-level only | Limited (thresholds) |
| Trident 4 | Yes (via NOS) | No | Via NOS | Port and CoS level | Better than TH5 |
| Tofino 2 | Yes (P4 INT program) | Native P4 INT | Via Stratum | Per-flow (via INT) | Full per-queue |
| Spectrum-4 | Yes (NVIDIA NOS/SONiC) | Via NVIDIA SDK | Yes (gNMI native) | Port and queue-level | Full WRED/ECN stats |
| Prestera | Limited | No | Partial | Port-level | Basic |

For AI cluster observability, Spectrum-4's native gNMI with per-queue ECN marking statistics and Tofino 2's INT-based per-flow telemetry represent the two highest-fidelity options. Tomahawk5 is workable with SONiC gNMI but requires careful configuration to expose queue-depth statistics for DCQCN analysis.

### E.2.4 Buffer Sizing Philosophy

Shallow-buffer ASICs (Tomahawk 5, Tofino 2) rely on lossless fabrics (PFC) or ECN-based congestion control to prevent drop. They are appropriate when:
- Traffic is dominated by large, long flows (RDMA Write)
- Congestion control (DCQCN) is correctly tuned
- PFC is enabled on the RDMA priority class

Deep-buffer ASICs (Trident 4 with external DRAM, Spectrum-4) absorb microbursts without PFC-induced pause propagation. They are appropriate when:
- Incast is severe and temporary (AllReduce scatter phase)
- PFC is unavailable or undesirable (storage + compute converged fabric)
- Mixed lossless/lossy traffic on the same fabric

### E.2.5 NOS Ecosystem per ASIC

| ASIC | SONiC | Arista EOS | Cumulus | Cisco NX-OS | Custom/Stratum |
|---|---|---|---|---|---|
| Tomahawk 5 | Yes (tier-1 support) | Yes | Yes | Yes | No |
| Trident 4 | Yes | Yes | Yes | Yes | No |
| Tofino 2 | Partial (via SAI shim) | No | No | No | ONF Stratum (P4Runtime) |
| Spectrum-4 | Yes (NVIDIA fork) | No | Yes (NVIDIA) | No | NVIDIA NOS (proprietary) |
| Prestera | Yes (community) | No | No | No | Marvell Kinetic |

### E.2.6 Fabric Topology Implications per ASIC

**Fat-tree (CLOS) with Broadcom Tomahawk5**

The classic 3-tier fat-tree using Tomahawk5 at spine and Trident4 at leaf is the most deployed AI cluster fabric topology. A 3-tier fat-tree with k=64 (64-port switches) provides 64/2 × 64/2 × 64/2 = ~130,000 server ports at full oversubscription 1:1, or ~32,000 GPU ports with no oversubscription on downlinks. ECMP at 64K groups ensures even traffic distribution across 32+ spine switches. The absence of adaptive routing means traffic distribution is determined at flow establishment time; ECMP hash collisions on highly correlated AllReduce traffic patterns are the primary performance risk.

**Dragonfly+ with NVIDIA Spectrum-4 / Quantum-3**

Dragonfly and dragonfly+ topologies are increasingly used for clusters exceeding 4,096 GPUs, where the cable count of a fat-tree becomes prohibitive. In a dragonfly+, groups of switches are connected in a flat topology with each group connected to every other group via a small number of global links. NVIDIA Quantum-3 is the primary platform for dragonfly IB deployments; Spectrum-4 for Ethernet dragonfly variants. The adaptive routing capability of Spectrum-4 is especially important in dragonfly topologies where the global links are few and must be load-balanced precisely. Without adaptive routing, collective communication patterns cause severe global link hotspots.

**Rail-Optimized Topology**

The rail-optimized design (as used in NVIDIA DGX SuperPOD) connects GPU 0 of every node to fabric rail 0, GPU 1 to rail 1, and so on. This creates 8 independent non-blocking fabrics (one per GPU). Each rail uses a two-tier CLOS structure. The benefit is that each AllReduce ring stays within a single rail, eliminating inter-rail interference. The downside is that the 8 rails cannot be used collectively for a single large flow; load balancing across rails requires application-level awareness.

---

## E.3 Cable and Optics

Cabling is frequently the last decision made and the first source of operational problems. For AI clusters, the key variables are reach (distance between switches and servers), port density (breakout ratios), and cost per active port.

### E.3.1 Cable Technology Overview

**DAC — Direct Attach Copper**

Twinaxial copper cable with QSFP-DD or QSFP56 connectors on each end. No optical conversion; passive DAC up to approximately 3 m, active DAC up to 7 m. Cheapest per-port cost. Power dissipation is near zero. The preferred choice for top-of-rack server-to-leaf connections where the server is in the same rack as the leaf switch.

- Standard: 400GbE QSFP-DD passive DAC ≤3 m
- Use: server NIC to leaf ToR port when distance ≤3 m

**AOC — Active Optical Cable**

Integrated optical transceiver in the QSFP housing with a fiber cable permanently attached. Multimode OM3/OM4 fiber. Range 3 m to 100 m. Slightly more expensive than DAC but flexible for inter-rack runs within a pod. No separate transceiver management.

- Standard: 400GbE QSFP-DD AOC 5–100 m multimode
- Use: server to leaf when server is not in same rack (cross-aisle, adjacent rack)

**QSFP-DD SR8 — Short Reach 400GbE**

Pluggable 400GbE transceiver using 8 lanes × 50G-NRZ or 8 lanes × 50G-PAM4 on OM3/OM4 multimode fiber. Two fibers (MPO-16 or 2× MPO-8 breakout). Reach 100 m on OM4. Separates the transceiver from the cable, enabling cable reuse.

- Standard: IEEE 802.3cm 400GBASE-SR8
- Use: leaf-to-spine runs within a data center row, MPO trunk cabling plant

**QSFP-DD DR4 — 500 m Singlemode 400GbE**

4 lanes × 100G-PAM4 on OS2 singlemode fiber. Reach 500 m. Required for inter-building or long data center runs. Higher cost than SR8. Two fibers per link (duplex LC or MPO-4).

- Standard: IEEE 802.3bs 400GBASE-DR4
- Use: spine-to-super-spine, inter-pod runs, campus MAN

**InfiniBand HDR Cables (200 Gb/s per port)**

HDR (High Data Rate) InfiniBand runs at 200 Gb/s per QSFP56 port (4 lanes × 50 Gb/s). Copper HDR cables reach up to 0.5 m (passive) or 2 m (active). AOC reaches up to 100 m. HDR connects H100 servers (via CX7 in IB mode) to Quantum-2 switches.

- Passive copper: ≤0.5 m (within rack)
- Active copper: ≤2 m
- AOC: ≤100 m
- Use: NDR IB fabric (see below) has replaced HDR for new H100 builds, but HDR is the in-service standard on A100-generation clusters

**InfiniBand NDR Cables (400 Gb/s per port)**

NDR (Next Data Rate) InfiniBand runs at 400 Gb/s per OSFP port (4 lanes × 100 Gb/s PAM4). Passive copper ≤0.5 m, active copper ≤2 m, AOC ≤100 m. Quantum-3 switches and CX7 NICs. This is the current top-of-line for H100/H200 DGX clusters.

**Breakout Cables**

A single 400GbE QSFP-DD port can be broken out to four 100GbE QSFP28 ports using a fan-out cable (1× QSFP-DD → 4× QSFP28 DAC) or an MPO-to-LC optical breakout. This is used when the leaf switch has 400G uplinks but servers have 100G NICs, or to reduce cost on non-GPU servers sharing the fabric.

- 1× 400GbE → 4× 100GbE: most common GPU cluster breakout
- 1× 800GbE → 2× 400GbE: emerging for next-gen spine ports
- 1× 400GbE → 8× 50GbE: storage or CPU-only server tier

**400G ZR / ZR+ Coherent DWDM**

For inter-data-center AI fabric links (connecting two geographically separated pods of GPUs for synchronous training), 400G ZR (OpenROADM standard, 80 km on singlemode dark fiber) and 400G ZR+ (up to 1,200 km with EDFA amplifiers) transceivers eliminate the need for dedicated DWDM line systems. ZR/ZR+ modules plug into standard QSFP-DD ports. Latency is higher than local fabric (3–5 ms per 600 km), which constrains synchronous collective frequency. Used in geo-distributed training configurations where the AllReduce is batched infrequently (gradient accumulation steps).

**800GbE — Next Generation**

800GbE (IEEE 802.3df, using 8 lanes × 100G-PAM4) is entering deployment in 2025 on the latest ASIC generations (Tomahawk5 has 800G port capability). Cable options: 800GBASE-SR8 (100 m, OM4), 800GBASE-DR8 (500 m, SMF). These are not yet widely deployed in production AI clusters but will be the standard for the next NIC generation (ConnectX-8 and equivalents).

### E.3.2 Cable Selection Quick Reference

| Technology | Max Reach | Fiber Type | Port Standard | Relative Cost | Best For |
|---|---|---|---|---|---|
| Passive DAC | 3 m | Copper | QSFP-DD 400GbE | Lowest | Same-rack ToR |
| Active DAC | 7 m | Copper | QSFP-DD 400GbE | Low | Adjacent rack |
| AOC | 100 m | OM4 multimode | QSFP-DD 400GbE | Medium | Cross-aisle, inter-rack |
| SR8 | 100 m | OM4 multimode | QSFP-DD 400GbE | Medium-High | Structured cabling |
| DR4 | 500 m | OS2 singlemode | QSFP-DD 400GbE | High | Inter-pod, long runs |
| HDR IB Copper | 2 m | Copper | QSFP56 HDR | Medium | A100-gen IB ToR |
| NDR IB Copper | 2 m | Copper | OSFP NDR | Medium | H100-gen IB ToR |
| NDR IB AOC | 100 m | OM4 multimode | OSFP NDR | High | IB inter-rack |
| Breakout DAC | 3 m | Copper | QSFP-DD to 4×QSFP28 | Low | Mixed-speed ToR |

### E.3.3 Fiber Plant Planning

**MPO trunk cabling** is the standard approach for structured AI cluster cabling. Pre-terminated MPO-24 or MPO-16 trunk cables run between patch panels at the top of each rack and cross-connect panels at the spine layer. This eliminates field termination and enables rapid reconfiguration. Key considerations:

- MPO-16 is aligned with 400GBASE-SR8 (2 fibers per lane × 8 lanes = 16 fibers per port)
- MPO-24 is compatible via breakout modules (3 × 400G SR8 per MPO-24 trunk)
- Polarity planning (Method A/B/C per TIA-568) must be resolved before installation; polarity errors cause link failure with no obvious indication

**DOM (Digital Optical Monitoring)**: All production pluggable transceivers support DOM, exposing temperature, voltage, Tx power, Rx power, and bias current via I2C. For AI clusters, DOM monitoring through the switch NOS or host `ethtool -m <interface>` provides early warning of degrading optics before they cause link errors.

```bash
# Check optic health on a QSFP port:
ethtool -m eth0
# Or via SONiC:
show interfaces transceiver info Ethernet32
show interfaces transceiver eeprom Ethernet32 --dom
```

Expected healthy DOM reading:
```
Module temperature: 42.00 C
Tx power: -1.5 dBm   (nominal for 400G SR8: -6 to +3 dBm)
Rx power: -1.8 dBm
```

Bad: Rx power below -10 dBm indicates fiber loss or dirty connector.

---

## E.4 Cost/Performance Reference Table

The table below provides order-of-magnitude networking cost estimates. Prices fluctuate significantly by vendor relationship, volume, and whether components are purchased vs. leased. These figures reflect 2024–2025 list prices and should be treated as relative benchmarks, not procurement targets.

| Cluster Size | GPU Count | Recommended NIC | Recommended Leaf ASIC | Recommended Spine ASIC | Est. $/GPU (Networking Only) | Notes |
|---|---|---|---|---|---|---|
| Small pod | 64 GPUs (8 nodes × 8 GPU) | NVIDIA CX7 NDR or 400GbE | Broadcom Trident4 or Spectrum-4 leaf | Not required (single-tier) | $800–$1,200 | Single fat-tree tier; leaf is also spine. NDR IB simplest. |
| Medium cluster | 512 GPUs (64 nodes × 8 GPU) | NVIDIA CX7 NDR | Spectrum-4 or Trident4 leaf | Tomahawk5 or Spectrum-4 spine | $1,500–$2,500 | Two-tier fat-tree; SHARP viable at this scale with Spectrum-4 |
| Large cluster | 4,096 GPUs (512 nodes × 8 GPU) | NVIDIA CX7 NDR (2× per GPU node) | Spectrum-4 or Quantum-3 leaf | Spectrum-4 / Quantum-3 spine | $2,500–$4,500 | Three-tier CLOS or dragonfly; dual-rail common; SHARP mandatory for AllReduce at scale |
| Hyperscale | 32,768+ GPUs | NVIDIA BF3 DPU (+ CX7 for compute rail) | Spectrum-4 leaf | Tomahawk5 spine (cost) or Spectrum-4 (SHARP) | $3,000–$6,000+ | Multi-plane fat-tree or dragonfly+; separate storage and compute rail; DPU for tenant isolation |

**Notes on cost drivers:**

- NIC cost is approximately $3,000–$6,000 per CX7 (NDR IB), $4,000–$8,000 per BF3 DPU (2025 street prices).
- Switch cost: Spectrum-4 (128× 400GbE) approximately $40,000–$80,000 per switch. Tomahawk5 whitebox approximately $25,000–$50,000.
- Optics and cabling often represent 20–35% of total networking BoM. NDR AOC is expensive ($800–$1,500 per cable for 10 m runs).
- Dual-rail designs (two independent fabrics per server) double NIC and fabric cost but halve effective bisection bandwidth required per rail and eliminate single fabric as a failure domain.

### E.4.1 Total Cost of Ownership Considerations

The $/GPU figure in the table above captures only capital expenditure on networking hardware. A complete TCO model should include:

**Power and cooling**: A 51.2 Tbps spine switch (Spectrum-4 or Tomahawk5) consumes 600–900W. At 100 spine switches in a 32K GPU cluster, switch power alone is 60–90 kW, exclusive of server NICs. CX7 NICs consume 30W each; at 2 NICs per GPU server with 8 GPUs, NIC power per GPU is 7.5W.

**Spares inventory**: Industry practice for AI cluster networking is 5–10% cold spare inventory for switches and 3–5% for NICs. NDR AOC cables fail at higher rates than DAC; budget 3% annual AOC failure rate.

**Operational labor**: SONiC-based whitebox deployments require more in-house NOS expertise than Arista/Cisco. Budget 1.5–2× the operational FTE cost for whitebox vs. vendor-supported NOS, especially in the first 18 months.

**Software licensing**: Arista EOS includes CloudVision subscription ($20–$40 per port/year). Cisco NX-OS includes Nexus Dashboard license. SONiC is open-source but commercial support (from DriveNets, Edgecore, etc.) is available. MOFED is free to download but NVIDIA enterprise support contracts are significant for large deployments.

### E.4.2 Procurement Checklist

Before issuing a purchase order for AI cluster networking hardware, verify the following:

- [ ] Confirm ASIC generation matches NOS version (SONiC `vs` versioning is ASIC-specific)
- [ ] Verify MOFED version compatibility with kernel and CUDA version
- [ ] Confirm NIC firmware version meets minimum for GPUDirect RDMA with the installed GPU driver
- [ ] Verify optic reach and fiber type against actual cable plant (OM3 vs OM4 reach differs by 30–50%)
- [ ] Confirm breakout cable fan-out matches switch port numbering convention (some ASICs use non-sequential fan-out)
- [ ] Verify switch port-to-port ASIC grouping for oversubscription (not all ports on a leaf switch connect to the same internal crossbar slice)
- [ ] Confirm vendor support SLA (next-business-day HW replacement for spines; 4-hour for leaves in large clusters)

---

## E.5 Vendor Ecosystem Summary

### NVIDIA (End-to-End AI Networking)

NVIDIA offers the most vertically integrated AI networking stack. ConnectX-7 NICs, BlueField-3 DPUs, Quantum-3 NDR InfiniBand switches, and Spectrum-4 Ethernet switches form a coherent ecosystem. SHARP in-network reduction is a differentiator available only on NVIDIA switch silicon and reduces AllReduce latency dramatically at scale by offloading the reduction computation to the switches. MOFED unifies the driver stack. NCCL is tuned for CX7+Quantum and CX7+Spectrum fabrics. The tradeoff is vendor lock-in and premium pricing. For H100/H200 clusters where training throughput is the primary KPI, the NVIDIA end-to-end stack typically delivers the highest out-of-box performance.

### Broadcom (Volume and Cost Efficiency)

Broadcom dominates merchant silicon shipments. Tomahawk5 is the default spine ASIC across Arista, Cisco, Dell, and whitebox SONiC vendors. Trident4 is the default leaf ASIC for most hyperscaler ToR deployments. Thor2 NICs offer competitive RoCEv2 at lower cost than CX7. The Broadcom ecosystem is NOS-agnostic: Tomahawk5 runs under SONiC, Arista EOS, Cumulus, and Cisco NX-OS equally. The tradeoff is absence of in-network SHARP compute and less aggressive adaptive routing compared to Spectrum-4. For very large clusters where $/port matters and SHARP is not required (e.g., inference at scale), Broadcom silicon + SONiC is the cost-optimal path.

### Intel (Programmability and Open Pipeline)

Intel's Tofino 2 is the only merchantable fully P4-programmable switch ASIC at data-center scale (as of 2025). It is the platform of choice for research environments, operators building custom telemetry (INT), and those implementing non-standard load balancing. The E810 NIC's CVL ASIC is similarly P4-influenced on the host side. Intel's limitation is ecosystem maturity: Tofino 2 requires more engineering effort per deployment than Broadcom or NVIDIA, NOS options are limited (Stratum/P4Runtime), and RDMA is not natively supported. Intel announced discontinuation of the Tofino product line in 2024; existing deployments will be supported but new designs should plan for migration to Broadcom or NVIDIA alternatives.

### Arista + Cisco (NOS Ecosystem and Enterprise Operability)

Arista EOS and Cisco NX-OS are the two dominant enterprise NOS platforms. Both run on Broadcom Tomahawk and Trident ASICs (and Cisco on custom ASICs for Nexus 9000 series). Their value is operational: robust CLI tooling, mature BGP/EVPN implementations, strong TAC support, and integration with enterprise management platforms. For AI clusters where the networking team comes from an enterprise background, Arista or Cisco reduces operational risk. Arista's CloudVision provides network-wide analytics and configuration management. Cisco's Nexus Dashboard integrates with ACI for policy-based networking. Neither vendor exposes SHARP; for SHARP, the fabric must be NVIDIA.

### SONiC Open-Source Ecosystem

SONiC (Software for Open Networking in the Cloud) has matured from a Microsoft internal project into a broad open-source ecosystem with deployments at Microsoft Azure, Alibaba Cloud, Goldman Sachs, and a growing number of AI labs. SONiC's architecture — a Redis-based state database decoupling the NOS applications from the ASIC SDK via SAI — makes it uniquely portable across Broadcom, NVIDIA, Marvell, and (to a degree) Intel ASICs.

For AI networking, SONiC offers:
- Native YANG-modeled configuration management via RESTCONF and gNMI
- First-class RDMA-related QoS primitives (PFC watchdog, DSCP-to-queue mapping, ECN thresholds) configurable via CLI or `config db`
- Integration with NVIDIA SHARP through the `sonic-mgmt` telemetry pipeline
- Active community development of AI-specific features (buffer tuning for AllReduce traffic, in-flight NCCL telemetry hooks)

The principal risk with SONiC is operational: debugging subtle NOS bugs requires understanding the Redis state machine, `syncd` behavior, and ASIC SDK, which is a very different skill set from traditional Arista/Cisco CLI operations. Organizations adopting SONiC for AI clusters should budget for dedicated NOS engineering capability.

---

## E.6 Quick Reference: NIC to GPU Pairing Matrix

The following matrix summarizes which NIC is the primary choice for each GPU generation and why:

| GPU Generation | Recommended NIC | Rationale |
|---|---|---|
| NVIDIA H200 SXM5 | ConnectX-7 (NDR IB or 400GbE) | PCIe Gen5 x16 host interface matches CX7; NVLink 4 intra-node; NDR IB inter-node |
| NVIDIA H100 SXM5 / PCIe | ConnectX-7 (NDR IB or 400GbE) | Same as H200; MOFED 23.x+ required for full GPUDirect RDMA performance |
| NVIDIA A100 SXM4 / PCIe | ConnectX-6 Dx (HDR IB or 200GbE) or CX7 | HDR IB (200 Gb/s) was native; CX7 is forward-looking upgrade |
| AMD MI300X | ConnectX-7 (RoCEv2) or Broadcom Thor2 | ROCm 6.x GPUDirect RDMA works with mlx5; Thor2 more cost-effective for RoCE-only |
| AMD MI250X | ConnectX-6 Dx or CX7 (RoCEv2) | ROCm 5.x+ support; must load `amdgpu_iommu` peer memory module |
| Intel Gaudi 3 | Intel E2000 (integrated 200GbE, 8 ports) | Gaudi 3 includes 8× 200GbE RDMA ports on-die via Intel's RDMA engine; no separate NIC needed |
| Google TPU v5 | Google Titanium (internal) | Not applicable to external hardware selection; listed for completeness |

**Note on AMD and Intel GPU RDMA**: Both AMD ROCm and Intel OneAPI provide GPUDirect RDMA-equivalent functionality (`amdgpu_iommu`, `intel_iommu` peer memory), but software ecosystem maturity (NCCL plugin quality, MOFED equivalent, diagnostic tooling) lags NVIDIA by 12–18 months as of 2025. Factor additional integration engineering cost when selecting AMD or Intel GPU + third-party NIC combinations.

---

*For diagnostic procedures related to hardware deployed per this guide, see Appendix G. For glossary definitions of terms used in this appendix (MOFED, SHARP, DCQCN, SR-IOV, etc.), see Appendix F.*


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).