# Chapter 34 — Congestion Control & the Ultra Ethernet Consortium

**Part VI: Storage & Adjacent Infrastructure** | ~15 pages

---

## Introduction

AI training clusters at the 1,000-GPU scale push Ethernet fabrics into a regime that traditional TCP congestion control was never designed to handle. A single 400 Gb/s **NCCL** all-reduce injects hundreds of gigabits of traffic in tight, synchronised bursts; if even one packet is dropped, the RDMA send queue stalls until a timeout expires, dragging the entire collective down with it. The classic solution — **Priority Flow Control (PFC)** — freezes an Ethernet lane to prevent drops, but that pause propagates backward through the fabric like a shockwave, creating head-of-line blocking and, in unlucky topologies, permanent deadlock.

**Data Center Quantized Congestion Notification (DCQCN)** has been the production answer to this problem since 2015: a feedback loop that uses **Explicit Congestion Notification (ECN)** markings from switch ASICs to throttle senders before queues overflow, eliminating most of the need for PFC pause frames. DCQCN is deeply integrated into NVIDIA ConnectX NICs and the **RoCEv2** RDMA transport, and configuring it correctly is one of the highest-leverage tuning tasks an AI cluster operator performs.

This chapter covers DCQCN in depth — its three-actor architecture, its rate control state machine, and its concrete configuration knobs — and then surveys the four algorithms that have emerged as challengers or complements: **Swift** (Google), **HPCC** (Alibaba), **TIMELY** (Microsoft/Google), and the transport designed by the **Ultra Ethernet Consortium (UEC)**. The chapter closes with the UEC's full-stack architecture, whose 1.0 specification was published in June 2025 and whose first hardware began reaching the market in late 2025.

Cross-references: RoCEv2 transport basics are covered in *(see Chapter 2)*; PFC lossless design and deadlock detection are covered in *(see Chapter 24)*; adaptive routing, packet spraying, and ECMP collision are covered in *(see Chapter 27)*; PFC watchdog and fault tolerance are covered in *(see Chapter 28)*.

---

## 34.1 The Bandwidth-Delay Product Problem in AI Fabrics

Congestion control in classical TCP operates at millisecond timescales and tolerates occasional packet loss as a signal. RDMA over Ethernet operates in a completely different regime. A single **InfiniBand-equivalent** message may be 256 MB or larger; the receiving QP (Queue Pair) has no TCP-style retransmit; and a single dropped or severely out-of-order packet forces a costly QP-level recovery that can take tens of milliseconds.

The **bandwidth-delay product (BDP)** quantifies the problem: at 400 Gb/s with a 1 µs switch-to-switch propagation delay, the pipe holds 50 KB of data per hop. A fat-tree with four hops holds 200 KB in flight per flow. At 1,000 concurrent flows this is 200 MB of live data in the network fabric at any instant. Even a 1% burst above line rate fills queues in microseconds.

The answer is to signal congestion *before* queues overflow, rate-limit senders precisely in response, and — ideally — eliminate the need for PFC altogether. The three-actor model used by DCQCN and its successors divides this responsibility:

- **Congestion Point (CP)**: The switch egress port ASIC detects queue growth and sets the ECN **Congestion Experienced (CE)** bit in the packet's IP header *(RFC 3168)*.
- **Notification Point (NP)**: The receiving NIC detects CE-marked packets and synthesises a **Congestion Notification Packet (CNP)** — a special RoCEv2 control message — and sends it back to the sender.
- **Reaction Point (RP)**: The sending NIC receives the CNP and reduces its transmission rate via a hardware rate limiter.

This loop, when properly tuned, keeps queue occupancy well below the PFC trigger threshold on almost all flows, dramatically reducing the frequency of pause frames.

---

## 34.2 DCQCN In Depth

**DCQCN** (Data Center Quantized Congestion Notification) was introduced by Zhu *et al.* at Microsoft and Mellanox at SIGCOMM 2015. It draws inspiration from **QCN** (Quantized Congestion Notification, IEEE 802.1Qau, 2010) but operates at Layer 3 over RoCEv2/UDP/IP rather than at Layer 2, and remains a vendor-implemented algorithm rather than an IEEE-standardised protocol. DCQCN combines switch-based **RED** (Random Early Detection) ECN marking with a NIC-side rate control algorithm.

### 34.2.1 ECN Marking at the Switch

The switch egress port is configured with three parameters that govern when it starts marking CE bits in the IP header:

| Parameter | Meaning | Typical default |
|-----------|---------|----------------|
| **K_min** | Queue depth at which marking begins (bytes) | 40 KB |
| **K_max** | Queue depth at which marking probability reaches P_max | 1000 KB |
| **P_max** | Maximum marking probability | 1.0 (100%) |

Between K_min and K_max the marking probability rises linearly. Below K_min no packet is ever marked; at or above K_max every packet is marked. This is the RED/WRED function operating at **WRED start-fill-level** = K_min and **WRED end-fill-level** = K_max.

On Linux hosts used for testing (with `tc qdisc red`), the equivalent parameters are `min`, `max`, and `probability`, with the `ecn` flag causing the kernel to set the CE bit rather than drop the packet:

```bash
# Install a RED qdisc on the loopback interface with ECN marking
# Simulates switch-side DCQCN ECN behaviour for lab testing
sudo tc qdisc add dev eth0 parent 1:1 handle 10: red \
    limit 4000000 \
    min 300000 \
    max 1000000 \
    avpkt 1500 \
    burst 100 \
    probability 1.0 \
    ecn

# Inspect instantaneous queue depth and mark counters
sudo tc -s qdisc show dev eth0
# Output includes: marked N packets (CE bit set), dropped 0 (ECN absorbs loss)
```

The `min` value (300 KB here) corresponds to K_min; `max` (1000 KB) to K_max; `probability 1.0` to P_max. The `avpkt` and `burst` parameters govern the EWMA averaging that smooths instantaneous queue fluctuations.

### 34.2.2 The DCQCN Rate-Control State Machine

Once the switch marks CE bits, the NP and RP engage in a closed-loop feedback. The RP's state machine has three phases:

```
  ┌─────────────────────────────────────────────────────────┐
  │             DCQCN Reaction Point State Machine          │
  │                                                         │
  │  Current rate: R_C                                      │
  │  Target rate:  R_T                                      │
  │  Alpha (α):    0–1, tracks congestion severity          │
  │                                                         │
  │  ┌─────────┐   CNP received    ┌─────────────────────┐ │
  │  │  FAST   │──────────────────▶│  RATE DECREASE      │ │
  │  │RECOVERY │                   │  R_C = R_C*(1-α/2)  │ │
  │  │ (R_C↑) │◀──────────────────│  R_T = R_C (saved)  │ │
  │  └────┬────┘  Timer: T_RD      └─────────────────────┘ │
  │       │ byte/time threshold                             │
  │       ▼                                                 │
  │  ┌─────────┐                                            │
  │  │ADDITIVE │  No CNP for T_RI byte/time intervals       │
  │  │INCREASE │  R_C += R_AI  (small increment, e.g. 5 Mbps)│
  │  │  (AI)   │                                            │
  │  └────┬────┘                                            │
  │       │ HAI threshold reached                           │
  │       ▼                                                 │
  │  ┌─────────┐                                            │
  │  │  HYPER  │  R_C += R_HAI (large increment, e.g. 50 Mbps)│
  │  │ADDITIVE │  Quickly reclaims unused bandwidth         │
  │  │INCREASE │                                            │
  │  └─────────┘                                            │
  └─────────────────────────────────────────────────────────┘
```

**Alpha (α) decay** is the central innovation. α tracks the sender's estimate of the network's congestion level, updated by an EWMA:

```
α ← (1 − g) · α + g · F
```

where `g` is the update weight (typically 1/16) and `F` is 1 if a CNP was received in the last measurement interval, 0 otherwise. When no CNPs arrive, α decays toward 0, allowing aggressive rate recovery. When CNPs arrive continuously, α approaches 1, triggering a 50% rate cut. This avoids both the oscillation of fixed-step rate control and the sluggishness of pure additive increase.

The RP's rate limiters impose the rate `R_C` as a token-bucket shaper in the NIC ASIC — no software is in the data path once configured.

**Key DCQCN timing parameters** (representative defaults compiled from NVIDIA documentation; exact field semantics and valid ranges vary by ConnectX silicon generation — always consult the NVIDIA Enterprise Support article for the specific adapter):

| Parameter | Role | Typical value |
|-----------|------|--------------|
| `initial_alpha_value` | Starting α at flow start | 1023 (≈1.0) |
| `rpg_time_reset` | Timer T_RD: delay before rate recovery begins | 300 µs |
| `rpg_byte_reset` | Byte threshold for T_RD (whichever is smaller) | 32768 bytes |
| `rpg_ai_rate` | Additive increase step R_AI | 5 Mbps |
| `rpg_hai_rate` | Hyper-additive increase step R_HAI | 50 Mbps |
| `rpg_min_dec_fac` | Minimum rate reduction factor (%) | 50 |
| `rpg_gd` | α EWMA weight denominator (g = 1/2^rpg_gd) | 11 (i.e. g ≈ 1/2048) |

### 34.2.3 Configuring DCQCN with mlxconfig and mlxreg

NVIDIA ConnectX NICs (CX-4 through CX-7) expose DCQCN parameters through two tools: `mlxconfig` for persistent NIC firmware settings and `mlxreg` for live per-port register access.

```bash
# Check current RoCE CC mode (ROCE_CC_PRIO_MASK_P = 255 means all priorities use DCQCN)
sudo mlxconfig -d /dev/mst/mt4125_pciconf0 q | grep -E "ROCE_CC|ECN"
# Example output:
#   ROCE_CC_PRIO_MASK_P=0xff  (ECN/DCQCN enabled for all 8 traffic classes)
#   ROCE_CC_LEGACY_DCQCN=1

# Disable legacy DCQCN to use NIC-side programmable CC (needed for newer CC modes)
sudo mlxconfig -d /dev/mst/mt4125_pciconf0 -y s ROCE_CC_LEGACY_DCQCN=0

# Enable user-programmable congestion control
sudo mlxconfig -d /dev/mst/mt4125_pciconf0 -y s \
    USER_PROGRAMMABLE_CC=1 \
    RDMA_SELECTIVE_REPEAT_EN=1

# Re-query to confirm
sudo mlxconfig -d /dev/mst/mt4125_pciconf0 q | grep -E "USER_PROG|RDMA_SEL"
```

Per-port DCQCN register tuning uses `mlxreg`. Register IDs and field offsets are silicon-specific (CX-4/5/6/7 differ); always consult the NVIDIA Enterprise Support article "HowTo Configure DCQCN (RoCE CC) values for ConnectX-4, Linux" for the current register map. The general pattern is:

```bash
# Example: set initial_alpha_value=1023 and rpg_ai_rate for port 1
# Register 0x506E is the RoCE CC profile register on CX-5 (verify for your silicon)
sudo mlxreg -d /dev/mst/mt4125_pciconf0 \
    --reg_id 0x506e \
    --reg_len 0x40 \
    --set "0x0.0:8=2,0x4.0:4=15"
# Syntax: byte_offset.bit_offset:width=value
# Consult NVIDIA docs for exact field map per adapter generation
```

Switch-side ECN configuration on Mellanox/NVIDIA Spectrum ASICs is done via `mlnx-nos` CLI:

```text
(config) # interface ethernet 1/1
(config-if) # traffic-class 3 ecn minimum-absolute 40 maximum-absolute 1000 probability 100
# Sets K_min=40KB, K_max=1000KB, P_max=100% on TC3 of the egress port
```

Verify that ECN is being applied to RoCEv2 packets (UDP/4791):

```bash
# NP-side: count CNPs sent back toward the source
cat /sys/class/infiniband/mlx5_0/ports/1/hw_counters/np_ecn_marked_roce_packets

# CP-side (on the switch): verify marks in hardware telemetry
# (Spectrum CLI)
show interface ethernet 1/1 counters ecn
```

---

## 34.3 PFC and Its Pathologies

**Priority Flow Control (PFC)** is the IEEE 802.1Qbb mechanism that allows a receiver to issue a PAUSE frame for a specific traffic class (priority), freezing the sender for up to 65,535 bit-times. PFC is the foundational lossless guarantee that RoCEv2 requires — without it, any dropped packet triggers a QP-level recovery that can stall the flow for milliseconds *(see Chapter 24)*.

The pathologies of PFC are well understood and catalogued:

**Head-of-line blocking (HoL blocking)**: When PFC pauses a link for priority 3 (the RoCEv2 traffic class by convention in most NVIDIA reference deployments), all traffic at priority 3 is blocked on that link, including traffic from flows that are not themselves congested. A slow receiver for one RDMA flow can therefore stall completely unrelated flows sharing the same uplink.

**Congestion spreading**: The paused sender's ingress buffer fills, triggering a PFC pause toward *its* upstream neighbour. This propagates backward through the fabric: a single slow receiver can eventually pause ports multiple hops away. In worst-case topologies this reaches ports that carry traffic from GPUs not participating in the congested flow at all.

**PFC deadlock**: If a traffic pattern creates a cyclic dependency — flow A pauses link X, flow B pauses link Y, and the two flows share enough links to form a cycle — the pause frames cannot resolve and the fabric deadlocks permanently. Deadlock requires a mix of PFC and specific traffic matrices; it is rare in carefully designed fat-trees but has been observed in production *(see Chapter 24)*.

**PFC pause storms**: A malfunctioning NIC or switch port can emit a continuous stream of PFC PAUSE frames for a priority, permanently freezing traffic on that priority across many links. This is sometimes called a "PFC storm."

The **PFC watchdog** is the last line of defence. It is implemented in both NIC firmware and switch ASICs and monitors how long a priority queue has been continuously paused. If the pause duration exceeds a configurable threshold (typically 100–200 ms), the watchdog classifies the situation as a deadlock or storm and takes one of two actions: it either drops packets from the affected queue (breaking the deadlock at the cost of traffic loss) or disables PFC on the affected port entirely. Some implementations support auto-recovery, where PFC is re-enabled after a quiet period; others require operator intervention *(see Chapter 28)*.

The key insight that motivates all of the algorithms discussed in the rest of this chapter is: **DCQCN keeps queues below the PFC threshold under most traffic patterns, but cannot prevent PFC entirely, and PFC's pathologies are fundamental to the mechanism, not configuration bugs.** Eliminating PFC requires either tolerating loss (with retransmit) or replacing the lossless assumption with a different mechanism.

---

## 34.4 Beyond DCQCN: Alternative Congestion Control Algorithms

### 34.4.1 TIMELY — RTT-Gradient-Based Control (Microsoft/Google, SIGCOMM 2015)

**TIMELY** (Radhika Mittal *et al.*, SIGCOMM 2015) was the first delay-based congestion control protocol designed specifically for datacenters. Rather than relying on ECN marks from the switch, it measures round-trip time at microsecond precision using hardware timestamps in the NIC, and uses the *gradient* of RTT change — d(RTT)/dt — as the congestion signal.

The TIMELY rate control law is:

- If d(RTT)/dt ≤ 0 (RTT decreasing): R ← R + δ (additive increase)
- If d(RTT)/dt > 0 (RTT increasing): R ← R · (1 − β · normalised_gradient) (multiplicative decrease)

TIMELY does not require ECN support in the network, making it attractive for deployments where switch ECN configuration is difficult. However, RTT measurements are noisier than ECN marks — RTT at the NIC captures both network queuing and end-host processing delays — and several papers showed TIMELY is harder to tune to stability than DCQCN.

TIMELY was an important proof-of-concept that switch-free congestion control is feasible, and its ideas directly influenced Swift.

### 34.4.2 Swift — Decomposed RTT-Based Control (Google, SIGCOMM 2020)

**Swift** (Gautam Kumar *et al.*, Google Brain, SIGCOMM 2020) is Google's production congestion control algorithm, deployed across their entire datacenter fleet. Swift simplifies and corrects TIMELY's instabilities by targeting an *absolute* delay value rather than the derivative, and by decomposing RTT into two independent components:

- **Fabric delay** (network queuing): measured as the NIC-to-NIC latency minus the baseline propagation delay. This drives **fcwnd** (fabric congestion window).
- **Endpoint delay** (host processing, NIC RX queue): measured from hardware timestamps at NIC ingress. This drives **ecwnd** (endpoint congestion window).

Both `fcwnd` and `ecwnd` follow the same AIMD law with separate delay targets:

```
if delay < target:
    cwnd += 1  (additive increase per RTT)
else:
    cwnd = cwnd * (1 - (delay - target) / delay)  (multiplicative decrease)

effective_cwnd = min(fcwnd, ecwnd)
```

Separating fabric and endpoint congestion allows Swift to respond correctly when the bottleneck is a congested switch (reduce fcwnd), a busy receiver NIC (reduce ecwnd), or both.

Swift does not require ECN support — it needs only hardware timestamps in the NIC and a baseline RTT measurement per destination. In large-scale Google testbed experiments, Swift achieved tail latency below 50 µs for short RPCs while sustaining near-100 Gbps throughput per server, with near-zero packet drops.

Swift's deployment at Google required no PFC, making it one of the first production-scale demonstrations that lossless-without-PFC is achievable — provided the congestion control algorithm reacts fast enough.

### 34.4.3 HPCC — INT-Feedback Precise Rate Control (Alibaba, SIGCOMM 2019)

**HPCC** (High Precision Congestion Control, Yuliang Li *et al.*, Alibaba Group, MIT, Cambridge, Harvard, SIGCOMM 2019) takes a fundamentally different approach: instead of estimating congestion from RTT or ECN marks, it reads precise per-link load information directly from the network via **In-Network Telemetry (INT)**.

INT is a P4-programmable switch feature that inserts per-hop metadata into packet headers as they traverse the network *(see Chapter 9)*. When an HPCC packet traverses a switch, the switch appends its current:
- **txRate**: current egress link utilisation (bytes/second)
- **qLen**: current egress queue length (bytes)
- **ts**: timestamp at the switch

The receiving NIC reads this INT data from the packet and includes it in the ACK, allowing the sender to compute the inflight bytes for each link on the path and adjust its window accordingly.

HPCC's rate control algorithm limits inflight bytes to `U * BDP` where U is the target utilisation (default η = 95%). The rate update is:

```
# For each link i on the path (INT data from ACK):
W_i = (BDP_i * U + qLen_i) / (1 + qLen_i / BDP_i)
# New window = minimum across all links
W = min(W_i for all i)
```

If the computed window is larger than the current window (no congestion), HPCC uses Additive Increase for `maxStage` consecutive measurements before jumping to the computed target. This avoids oscillation during free-bandwidth reclaim.

HPCC's three key parameters are:
- **η (eta)**: target link utilisation, default 0.95. Lower η reduces queue length at the cost of bandwidth.
- **maxStage**: number of AI steps before jumping to computed target, default 5. Higher values improve stability.
- **W_AI**: additive increase step size, controls convergence to fairness.

HPCC's advantage over DCQCN and TIMELY is precision: because it observes every link's actual load, it needs no heuristic tuning of K_min/K_max, no α decay time-constants, and no RTT smoothing. In simulation and testbed experiments, HPCC reduced flow completion times by up to 95% compared to DCQCN under large-scale incast.

The limitation is deployment complexity: all switches on the path must support and be configured for INT telemetry insertion, which requires P4-capable ASICs.

### 34.4.4 Comparison Table

| Algorithm | Feedback signal | ECN required | Switch INT required | Reorder tolerant | PFC required | Deployment status |
|-----------|----------------|:------------:|:-------------------:|:----------------:|:------------:|-------------------|
| **DCQCN** | ECN CE bit | Yes | No | No | Typically yes | Production (NVIDIA ConnectX, RoCEv2) |
| **TIMELY** | RTT gradient (NIC timestamps) | No | No | No | Typically yes | Research; influenced Swift |
| **Swift** | RTT decomposed (fabric + endpoint) | No | No | No | No (Google) | Production (Google datacenter fleet) |
| **HPCC** | INT per-link load (txRate, qLen) | No | Yes (P4 INT) | No | No | Production (Alibaba); open-source NS3 sim |
| **UEC (NSCC+RCCC)** | RTT + ECN (NSCC); receiver credits (RCCC) | NSCC: Yes | No | **Yes** (RUD) | **No** (LLR replaces PFC) | Spec v1.0 (June 2025); early HW 2025–2026 |

---

## 34.5 RoCEv2's Reorder Intolerance

A critical constraint that all of the algorithms above must work around — and that the UEC was specifically designed to eliminate — is **RoCEv2's strict requirement for in-order packet delivery**.

RoCEv2 uses UDP/IP for routing. The RDMA transport layer above assigns sequence numbers to packets within a QP and expects them to arrive in order. When a packet arrives out of order, the receiving QP treats it as a loss: it does not buffer the out-of-order packet, it generates a **NACK**, and it requests retransmission from the sequence number where order was violated. This means a single reordering event causes one or more packets to be retransmitted even though they were not actually lost.

The standard approach to preventing reordering is **5-tuple ECMP**: packets of the same flow (same source IP, source port, destination IP, destination port, protocol) always hash to the same ECMP path through the fabric. All packets of a given RDMA QP follow the same physical path, guaranteeing in-order delivery.

The problem is that 5-tuple ECMP provides very coarse load balancing. Two large RDMA flows that hash to the same ECMP path will share a single physical link even if adjacent paths are entirely idle. This is the **ECMP collision** problem that motivates adaptive routing and packet spraying *(see Chapter 27)*. The collision penalty can be severe: two 400 Gb/s flows on a shared link each get 200 Gb/s when each could have 400 Gb/s on separate paths.

Packet spraying — sending consecutive packets of the same flow across different paths — solves the collision problem but violates RoCEv2's ordering requirement. The UEC's architecture is the first standard transport designed to support spraying natively.

---

## 34.6 The Ultra Ethernet Consortium (UEC)

### 34.6.1 Formation and Goals

The **Ultra Ethernet Consortium (UEC)** was announced in July 2023, launched under the Linux Foundation's Joint Development Foundation. The founding steering members were: **AMD**, **Arista**, **Broadcom**, **Cisco**, **Eviden** (Atos), **HPE**, **Intel**, **Meta**, and **Microsoft**. The technical work had begun earlier: AMD, Broadcom, HPE, Intel, and Microsoft started collaborating informally in early 2022 on the requirements that would become Ultra Ethernet. By late 2024 the consortium had grown to over 100 member companies with more than 1,000 individual participants.

The consortium's mission statement is: to deliver an Ethernet-based, open, interoperable, high-performance, full-communications stack architecture to meet the growing network demands of AI and HPC at scale.

The four design goals that distinguish Ultra Ethernet from RoCEv2 are:

1. **Packet spraying with native reorder tolerance**: every packet of a message may take a different path through the fabric; the transport layer reassembles them correctly.
2. **Lossless semantics without PFC pause frames**: achieve drop-free delivery through a combination of credit-based flow control, congestion control, and link-level retry — not by freezing ports.
3. **New congestion control algorithms**: two complementary CC algorithms (NSCC and RCCC) that use both RTT and ECN signals, and handle both fabric congestion and receiver incast.
4. **Backward compatibility with Ethernet**: Ultra Ethernet runs on standard Ethernet physical and link layers; existing switching infrastructure does not need to be replaced.

### 34.6.2 The Ultra Ethernet Specification

The **Ultra Ethernet (UE) Specification v1.0** was released on June 11, 2025, as a 562-page document covering the complete stack from the physical layer through the application interface. Version 1.0.2 followed in January 2026 with clarifications and errata. The specification is publicly available at:

`https://ultraethernet.org/wp-content/uploads/sites/20/2025/06/UE-Specification-6.11.25.pdf`

The **Ultra Ethernet Transport (UET)** is the transport layer defined by this specification. It replaces the InfiniBand/RoCEv2 transport semantics with a new protocol designed for modern hardware and workloads.

### 34.6.3 UET Architecture and Stack Layers

The UET stack is vertically integrated from Ethernet frames to RDMA verbs:

```
  ┌─────────────────────────────────────────────────────────────┐
  │              Application / MPI / Collective Communications  │
  ├─────────────────────────────────────────────────────────────┤
  │         Semantic Sub-layer (SES)                            │
  │  RDMA verbs │ Collective ops │ MPI semantics               │
  ├─────────────────────────────────────────────────────────────┤
  │         Packet Delivery Sub-layer (PDS)                     │
  │  Reliable Unordered Delivery (RUD) — default                │
  │  Reliable Ordered Delivery (ROD)                            │
  │  Unreliable Unordered Delivery (UUD)                        │
  │  Congestion Control (NSCC + RCCC)                           │
  │  Selective ACKs (64-bit bitmaps)                            │
  ├─────────────────────────────────────────────────────────────┤
  │         Link Layer                                          │
  │  Link Level Retry (LLR)    │  Credit-Based Flow Control     │
  ├─────────────────────────────────────────────────────────────┤
  │         Ethernet MAC / Physical Layer (unchanged)           │
  └─────────────────────────────────────────────────────────────┘
```

### 34.6.4 Reliable Unordered Delivery (RUD) and Packet Spraying

**Reliable Unordered Delivery (RUD)** is the default bulk-transfer mode in UET and the key to packet spraying support. Under RUD:

- Each packet is assigned a monotonically increasing **Packet Sequence Number (PSN)**.
- The receiver maintains a **64-bit bitmap** tracking which PSNs in the current window have arrived, regardless of order.
- Selective ACKs are sent using these bitmaps, allowing the sender to retransmit only the specific packets that are missing.
- The Semantic Sub-layer above receives packets for reassembly in message order; the transport layer does not require in-order delivery from the network.

Because RUD tolerates out-of-order arrival, the UE NIC can implement **per-packet path selection** (packet spraying): the **Entropy Value (EV)** field in the UE packet header is varied for each successive packet of the same message, causing ECMP hash functions in the fabric to map successive packets to different equal-cost paths. In expectation, traffic is distributed evenly across all available paths, eliminating ECMP collision without requiring any changes to the switching hardware.

### 34.6.5 NSCC: Network Signal-Based Congestion Control

**NSCC** (Network Signal-based Congestion Control) is the primary sender-side CC algorithm. It combines ECN CE bits and RTT measurements into a single congestion estimate, addressing the respective weaknesses of each signal alone:

- **ECN alone** reacts quickly but is a 1-bit signal; it cannot quantify how congested the network is.
- **RTT alone** provides a gradient but lags behind actual queue buildup and conflates network and endpoint delays.

NSCC defines four operating regimes based on the combination of signals:

| ECN marked | RTT signal | NSCC action |
|:----------:|:----------:|-------------|
| Yes | Low | Mild decrease (building congestion, not yet severe) |
| Yes | High | Aggressive window decrease |
| No | Low | Quick additive increase |
| No | High | Gentle additive increase |

The NSCC congestion window tracks inflight unacknowledged bytes. When NSCC detects congestion it reduces the window; when the path is clear it applies AIMD increase. The combination of ECN and RTT signals provides faster reaction than RTT-only algorithms and more granular control than ECN-only algorithms.

NSCC also incorporates a **Quick Adapt** algorithm that uses packet loss signals (not just CE marks) to quickly identify bottleneck links and converge the rate to the available bandwidth.

### 34.6.6 RCCC: Receiver Credit-Based Congestion Control

**RCCC** (Receiver Credit-Based Congestion Control) is the complementary receiver-driven CC algorithm, specifically designed to handle **incast**: the many-to-one traffic pattern where hundreds of senders simultaneously target a single receiver or a small rack of receivers.

NSCC cannot handle incast well because it observes network signals — it cannot easily distinguish "I am hitting my fair share of a congested link" from "there are 500 senders all pointed at my receiver and its NIC buffer is full." RCCC solves this by inverting the control plane:

- The **receiver** knows exactly how many incoming flows it is managing and how much buffer it has available.
- The receiver issues **credits** to each sender at a rate that respects receiver-side resources.
- The **sender** does not increase its window based on network signals; it sends only when it has receiver credits.

NSCC and RCCC are designed to operate jointly: NSCC responds to fabric-level congestion while RCCC limits receiver-side incast. A sender may reduce its window due to either NSCC or RCCC independently.

### 34.6.7 Link Level Retry (LLR) — Replacing PFC

**Link Level Retry (LLR)** is the Ultra Ethernet mechanism that replaces PFC as the guarantee against packet loss. LLR operates at Layer 2, on a single physical link, and provides reliable delivery of frames across that link without involving the transport layer:

- The transmitter stores all LLR-eligible frames in a **replay buffer** (a small FIFO in the NIC or switch ASIC).
- Each frame is assigned a **sequence number encoded in the Ethernet preamble** (without changing the frame format visible to the MAC).
- If the receiver detects a missing sequence number, it sends a **NACK**.
- The transmitter retransmits from the missed sequence number (go-back-N) or from individual frames (selective if the hardware supports it).
- A **timeout** covers tail loss (no NACK received because the receiver never saw the packet).

LLR eliminates the need for PFC because it handles bit errors and link-level transient drops locally, without issuing a fabric-wide pause. The transport layer (PDS) handles end-to-end reliability; LLR handles link-level reliability; the two layers compose cleanly.

Between switches (in the network fabric), PFC is explicitly disabled in Ultra Ethernet deployments. The specification notes that PFC between switches "can block valid flows" and that LLR + NSCC/RCCC together provide equivalent lossless semantics.

### 34.6.8 Credit-Based Flow Control (CBFC)

At the link layer, Ultra Ethernet also supports **Credit-Based Flow Control (CBFC)** as an alternative to PFC. CBFC provides per-virtual-channel flow control using credits rather than pause frames:

- Credits represent buffer space reserved at the receiver end of each link.
- A sender may transmit a frame on a virtual channel only if it has a credit for that channel.
- Credits are returned as ACKs from the receiver, not as explicit pause frames.

CBFC requires less buffer headroom than PFC (because it prevents overflow rather than reacting to it), enables local scheduling decisions per virtual channel, and does not propagate congestion signals backward through the fabric.

### 34.6.9 UEC vs. InfiniBand and RoCEv2

| Property | InfiniBand | RoCEv2 + DCQCN | Ultra Ethernet (UE 1.0) |
|----------|-----------|----------------|-------------------------|
| Physical layer | IB-specific | Standard Ethernet | Standard Ethernet |
| In-order requirement | Yes | Yes | No (RUD default) |
| Packet spraying | No | No | Yes (per-packet EV) |
| Congestion control | IB CC (credit-based) | DCQCN (ECN) | NSCC + RCCC |
| Loss recovery | Link-level (IB) | PFC + DCQCN | LLR + PDS retransmit |
| PFC required | No (IB has own CC) | Yes (typically) | No |
| Open standard | Yes (IBTA) | IEEE 802.1 + IETF | Yes (Linux Foundation / JDF) |
| Ecosystem | Mature (25+ years) | Mature (10+ years) | Emerging (hardware 2025+) |

### 34.6.10 Deployment Outlook

As of early 2026, Ultra Ethernet is transitioning from specification to hardware. Arista, Broadcom, and Intel confirmed lab testing of UE 1.0-compliant hardware shortly after the spec release. First-generation NICs and switches are expected to reach production deployments in 2026. The UEC holds ISO status and operates with over 100 member organisations spanning cloud providers, system vendors, and silicon companies.

The critical open question for the industry is whether **RDMA verbs compatibility** — the primary API through which MPI, NCCL, and storage fabrics access RDMA hardware — can be maintained above the UET Semantic Sub-layer. The UEC has stated backward compatibility as a design goal; achieving it without sacrificing UET's reorder-tolerance and spraying benefits requires per-verb mapping work that is still in progress at the library and driver level.

---

## 34.7 Lossless vs. Lossy Fabric: The PFC Elimination Path

The fundamental choice in designing an AI cluster fabric is: **maintain lossless semantics by preventing drops, or accept occasional drops and retransmit?**

RoCEv2 with DCQCN pursues the former: ECN keeps queues low, PFC catches the overflow, and together they guarantee that no RDMA packet is ever dropped by the fabric. The cost is complexity: ECN must be carefully tuned per-queue, PFC creates deadlock risk, and the watchdog is a necessary but disruptive fallback.

Ultra Ethernet pursues a hybrid: LLR eliminates link-level loss (equivalent to PFC's role), NSCC/RCCC keep queues low (equivalent to DCQCN's role), and RUD provides end-to-end selective retransmit for any residual loss that escapes LLR. The result is **lossless semantics without pause frames**: the transport layer guarantees delivery, but the mechanism is retransmit rather than pause.

InfiniBand occupies a third position: its credit-based flow control at the link level is architecturally similar to UEC's CBFC, and its transport layer has always supported selective retransmit. IB's advantage is 25 years of ecosystem maturity; its disadvantage is that it requires IB-specific hardware and does not run on commodity Ethernet switches.

The practical implication for cluster design is:

- **2023–2025 deployments**: RoCEv2 + DCQCN with carefully tuned ECN thresholds is the dominant choice for Ethernet-based AI clusters. PFC is still required in most deployments.
- **2026+ new builds**: UEC-compliant hardware offers a PFC-free path. Early adopters will be running mixed fabrics (legacy RoCEv2 nodes + UE NICs) during the transition.
- **Very large scale (>10,000 GPUs)**: InfiniBand retains a significant installed base advantage due to mature HPC software stacks and switch-side credit-based flow control.

---

## 34.8 Lab: DCQCN Parameter Sweep with Soft-RoCE, ECN Measurement, and Queue Visualisation

This lab uses Linux's software RDMA stack (`rdma_rxe`, also called **Soft-RoCE**) and the `tc qdisc red` module to demonstrate the DCQCN signalling path — ECN marking at the "switch" (loopback interface), CNP generation at the receiver, and queue depth vs. throughput measurement. 

**Important caveat**: `rdma_rxe` implements the RoCEv2 protocol in software on the Linux network stack. It does not implement the DCQCN hardware rate limiter in the NIC ASIC. The rate-decrease feedback loop (rpg_* registers) is only observable on real ConnectX hardware. This lab demonstrates the CP → NP signalling path (ECN marking → CNP generation) and allows measurement of queue depth vs. throughput using Linux kernel counters.

### Step 1: Install prerequisites

```bash
sudo apt install -y rdma-core libibverbs-utils ibv-utils perftest iproute2
# Load the soft-RoCE kernel module
sudo modprobe rdma_rxe
lsmod | grep rdma_rxe
# Expected: rdma_rxe   [...]
```

### Step 2: Create soft-RoCE devices over loopback

```bash
# Create rxe0 over the loopback interface
sudo rdma link add rxe0 type rxe netdev lo
# Verify
rdma link show
# Expected: link rxe0/1 state ACTIVE physical_state POLLING netdev lo

# Check IB device is visible
ibv_devinfo
# Expected: hca_id: rxe0 ...
```

### Step 3: Install ECN-marking RED qdisc (simulating switch-side CP)

```bash
# Add a root HTB qdisc with a leaf class for shaping
sudo tc qdisc add dev lo root handle 1: htb default 10
sudo tc class add dev lo parent 1: classid 1:10 htb rate 10gbit burst 1m

# Attach a RED qdisc with ECN to simulate switch ECN marking
# K_min=300KB, K_max=1000KB, P_max=1.0
sudo tc qdisc add dev lo parent 1:10 handle 10: red \
    limit 4000000 \
    min 300000 \
    max 1000000 \
    avpkt 1500 \
    burst 100 \
    probability 1.0 \
    ecn

# Confirm the qdisc is installed
sudo tc qdisc show dev lo
```

### Step 4: Run RDMA bandwidth test and observe ECN counters

```bash
# Terminal 1 — start the ib_write_bw server
ib_write_bw -d rxe0 --run_infinitely

# Terminal 2 — run the client for 30 seconds
ib_write_bw -d rxe0 127.0.0.1 \
    --duration 30 \
    --report_gbits \
    --size 65536

# Terminal 3 — monitor queue depth and ECN mark counters every second
watch -n 1 'sudo tc -s qdisc show dev lo | grep -A5 "red 10:"'
# Look for: "marked N" (packets where CE bit was set instead of drop)
```

### Step 5: Observe CNP generation (NP role)

```bash
# Capture CNPs (UDP/4791 packets with BTH opcode 0x81 = CNP)
sudo tcpdump -i lo udp port 4791 -vv -c 50 2>/dev/null | grep -A2 "CNP\|length 52"
# CNPs are small (52-byte) special RoCEv2 control packets sent by the NP back to the RP
```

### Step 6: Parameter sweep — vary K_min and measure throughput

```bash
#!/usr/bin/env python3
"""
Sweep K_min (RED min) and measure ib_write_bw throughput.
Demonstrates how ECN threshold affects queue depth and bandwidth.
"""
import subprocess, re, time

kmin_values = [100_000, 200_000, 300_000, 500_000, 1_000_000]  # bytes

for kmin in kmin_values:
    # Remove old qdisc
    subprocess.run("sudo tc qdisc del dev lo root 2>/dev/null", shell=True)
    time.sleep(0.5)
    # Install new RED with this K_min
    subprocess.run(f"""
        sudo tc qdisc add dev lo root handle 1: htb default 10 &&
        sudo tc class add dev lo parent 1: classid 1:10 htb rate 10gbit burst 1m &&
        sudo tc qdisc add dev lo parent 1:10 handle 10: red \
            limit 4000000 min {kmin} max 1000000 avpkt 1500 burst 100 probability 1.0 ecn
    """, shell=True)

    # Run a short bandwidth test
    result = subprocess.run(
        "ib_write_bw -d rxe0 127.0.0.1 --duration 5 --report_gbits --size 65536",
        shell=True, capture_output=True, text=True, timeout=30
    )
    # Parse throughput from last data line
    lines = [l for l in result.stdout.splitlines() if re.search(r'\d+\.\d+', l)]
    bw = lines[-1].split()[-1] if lines else "N/A"

    # Read ECN mark count
    stats = subprocess.run("sudo tc -s qdisc show dev lo", shell=True,
                           capture_output=True, text=True)
    marked = re.search(r'marked\s+(\d+)', stats.stdout)
    mark_count = marked.group(1) if marked else "0"

    print(f"K_min={kmin//1000}KB: throughput={bw} Gbps, ECN_marks={mark_count}")
```

Expected output pattern (approximate — rdma_rxe throughput is software-limited):

```text
K_min=100KB:  throughput=3.2 Gbps, ECN_marks=12847
K_min=200KB:  throughput=3.8 Gbps, ECN_marks=4231
K_min=300KB:  throughput=4.1 Gbps, ECN_marks=1087
K_min=500KB:  throughput=4.3 Gbps, ECN_marks=198
K_min=1000KB: throughput=4.4 Gbps, ECN_marks=11
```

Lower K_min causes earlier and more aggressive ECN marking, reducing throughput (simulating the RP reaction) and increasing CNP traffic. Higher K_min allows larger queues before marking, increasing throughput but also increasing latency. On real ConnectX hardware, the RP's rate limiter would clamp the send rate, making the throughput vs. K_min curve much steeper.

### Step 7: Cleanup

```bash
sudo tc qdisc del dev lo root 2>/dev/null || true
sudo rdma link delete rxe0
sudo rmmod rdma_rxe
```

---

## 34.9 Summary

Congestion control in AI cluster fabrics is not a single algorithm but a system of interacting components: the switch ASIC marking ECN bits, the NIC generating CNPs, the sender's rate limiter adjusting transmission speed, and the PFC watchdog as the fallback. **DCQCN** has been the production standard since 2015 and remains the dominant choice for RoCEv2 deployments, but its dependence on PFC and its sensitivity to K_min/K_max tuning create operational challenges at scale.

**Swift**, **HPCC**, and **TIMELY** demonstrate that delay-based and INT-based alternatives can match or exceed DCQCN performance while eliminating the need for PFC in some deployments. Each trades ECN simplicity for either RTT measurement infrastructure or INT-capable switching hardware.

The **Ultra Ethernet Consortium** has published a complete stack specification — UE 1.0, released June 2025 — that addresses the root cause of RoCEv2's limitations: in-order delivery dependency. By defining **Reliable Unordered Delivery** as the default transport mode, UET enables full packet spraying, eliminating ECMP collision. **LLR** replaces PFC as the lossless guarantee. **NSCC+RCCC** provide dual congestion signals for both fabric and receiver congestion. First hardware is reaching the market in 2025–2026; whether the ecosystem (MPI libraries, NCCL, storage) transitions as quickly as the hardware remains the critical open question.

---

## References

1. Zhu, Y., *et al.* "Congestion Control for Large-Scale RDMA Deployments." ACM SIGCOMM 2015. [https://www.microsoft.com/en-us/research/publication/congestion-control-for-large-scale-rdma-deployments/](https://www.microsoft.com/en-us/research/publication/congestion-control-for-large-scale-rdma-deployments/)

2. Mittal, R., *et al.* "TIMELY: RTT-based Congestion Control for the Datacenter." ACM SIGCOMM 2015. [https://dl.acm.org/doi/10.1145/2785956.2787510](https://dl.acm.org/doi/10.1145/2785956.2787510)

3. Kumar, G., *et al.* "Swift: Delay is Simple and Effective for Congestion Control in the Datacenter." ACM SIGCOMM 2020. [https://research.google/pubs/swift-delay-is-simple-and-effective-for-congestion-control-in-the-datacenter/](https://research.google/pubs/swift-delay-is-simple-and-effective-for-congestion-control-in-the-datacenter/)

4. Li, Y., *et al.* "HPCC: High Precision Congestion Control." ACM SIGCOMM 2019. [https://dl.acm.org/doi/10.1145/3341302.3342085](https://dl.acm.org/doi/10.1145/3341302.3342085)

5. Ultra Ethernet Consortium. "Ultra Ethernet Specification v1.0." June 11, 2025. [https://ultraethernet.org/wp-content/uploads/sites/20/2025/06/UE-Specification-6.11.25.pdf](https://ultraethernet.org/wp-content/uploads/sites/20/2025/06/UE-Specification-6.11.25.pdf)

6. Ultra Ethernet Consortium. Official Website. [https://ultraethernet.org/](https://ultraethernet.org/)

7. Linux Foundation. "Announcing the Ultra Ethernet Consortium (UEC)." July 2023. [https://www.linuxfoundation.org/press/announcing-ultra-ethernet-consortium-uec](https://www.linuxfoundation.org/press/announcing-ultra-ethernet-consortium-uec)

8. Herbert, T. "A (mostly) Unbiased Review of the Ultra Ethernet Specification v1.0." Medium, 2025. [https://medium.com/@tom_84912/a-mostly-unbiased-review-of-the-ultra-ethernet-specification-10d816227839](https://medium.com/@tom_84912/a-mostly-unbiased-review-of-the-ultra-ethernet-specification-10d816227839)

9. arxiv.org. "Ultra Ethernet's Design Principles and Architectural Innovations." 2025. [https://arxiv.org/html/2508.08906v1](https://arxiv.org/html/2508.08906v1)

10. NVIDIA Enterprise Support. "HowTo Configure DCQCN (RoCE CC) values for ConnectX-4, Linux." [https://enterprise-support.nvidia.com/s/article/howto-configure-dcqcn--roce-cc--values-for-connectx-4--linux-x](https://enterprise-support.nvidia.com/s/article/howto-configure-dcqcn--roce-cc--values-for-connectx-4--linux-x)

11. Juniper Networks. "Data Center Quantized Congestion Notification (DCQCN)." Junos OS Documentation. [https://www.juniper.net/documentation/us/en/software/junos/traffic-mgmt-qfx/topics/topic-map/cos-qfx-series-DCQCN.html](https://www.juniper.net/documentation/us/en/software/junos/traffic-mgmt-qfx/topics/topic-map/cos-qfx-series-DCQCN.html)

12. Linux kernel documentation. `tc-red(8)` man page. [https://man7.org/linux/man-pages/man8/tc-red.8.html](https://man7.org/linux/man-pages/man8/tc-red.8.html)

13. linux-rdma project. `rdma_rxe` Soft-RoCE documentation. [https://github.com/linux-rdma/rdma-core/blob/master/Documentation/rxe.md](https://github.com/linux-rdma/rdma-core/blob/master/Documentation/rxe.md)

14. Arista Networks. "Demystifying Ultra Ethernet." [https://blogs.arista.com/blog/demystifying-ultra-ethernet](https://blogs.arista.com/blog/demystifying-ultra-ethernet)

15. Zhu, Y., *et al.* "ECN or Delay: Lessons Learnt from Analysis of DCQCN and TIMELY." CoNEXT 2016. [https://people.csail.mit.edu/ghobadi/papers/ecn_vs_delay_conext_2016.pdf](https://people.csail.mit.edu/ghobadi/papers/ecn_vs_delay_conext_2016.pdf)

---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
