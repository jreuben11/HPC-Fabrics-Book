# Chapter 3 — Precision Time Protocol

**Part I: Foundations** | ~15 pages

---

## Introduction

Precise clock synchronization is a silent prerequisite of the AI cluster network stack — one that only becomes visible when it is missing. When hundreds of **GPU** servers must coordinate collective operations, profile distributed training pipelines, and correlate telemetry events from **NIC**s, switches, and the kernel, they must all share a common notion of time accurate to tens of nanoseconds. The technology that delivers this is the **Precision Time Protocol** (**PTP**), standardized as **IEEE 1588**.

This chapter explains why **NTP**'s millisecond accuracy is insufficient for AI cluster workloads, and how **PTP** achieves sub-microsecond synchronization by exploiting hardware timestamping at the **NIC** MAC layer. The core of **PTP** is a hierarchical clock architecture — **Grandmaster**, **Boundary Clock**, **Transparent Clock**, and **Ordinary Clock** — combined with a delay-request exchange that continuously estimates and corrects for asymmetric propagation delay. The **Best Master Clock Algorithm** (**BMCA**) provides automatic leader election, ensuring clusters survive grandmaster failures without manual intervention.

We cover the practical Linux deployment of **PTP** through `linuxptp`, the canonical open-source implementation. The two central components — `ptp4l`, which disciplines the **NIC**'s hardware clock (**PHC**) to the network master, and `phc2sys`, which copies that hardware clock into the system's CLOCK_REALTIME — are explained in detail along with their key configuration parameters and the PI servo behavior that governs convergence speed and stability.

The chapter also addresses the interaction between **PTP** and `chrony` for hybrid **NTP**/**PTP** environments, the **GPS**/**PPS**-based grandmaster setup used in on-premises clusters, and the boundary-clock deployment pattern on top-of-rack switches that is standard in production AI fabrics. Understanding **PTP** is prerequisite for the congestion-control discussion in Chapter 2 (**DCQCN** **ECN** feedback relies on **NIC** hardware timestamps) and for the observability chapter (Chapter 16), where cross-host trace correlation requires synchronized clocks.

The lab walkthrough builds a three-container **PTP** domain (**Grandmaster** → **Boundary Clock** → **Ordinary Clock**) using **Containerlab**, exercises the PI servo under injected path delay, and demonstrates grandmaster failover — all the failure scenarios most likely to degrade cluster timing in production.

---

## Installation

This section installs **linuxptp**, which provides `ptp4l` (the **PTP** engine that disciplines a **NIC** hardware clock to the network grandmaster), `phc2sys` (which copies the **NIC**'s **PHC** into CLOCK_REALTIME), and `pmc` (the management client used to inspect port states and PI servo statistics). The **chrony** package is installed alongside **linuxptp** to handle **NTP**-based fallback synchronization in hybrid environments and to demonstrate the handoff between **NTP** and **PTP** time sources. The optional **gpsd** package is included for **GPS**/**PPS**-based grandmaster setups where an on-premises stratum-0 source is needed rather than a network grandmaster.

### Ubuntu 24.04 Packages

```bash
sudo apt install -y linuxptp chrony gpsd gpsd-clients ethtool
```

### Containerlab

Containerlab is used for the lab walkthrough to create a lightweight multi-node PTP topology using containers. Install from the official script:

```bash
# Install Containerlab (requires Docker to already be installed)
bash -c "$(curl -sL https://get.containerlab.dev)"

# Verify installation
containerlab version
```

Docker prerequisite (if not already installed):

```bash
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
```

Verify all tools are present:

```bash
which ptp4l phc2sys pmc ts2phc   # from linuxptp
which chronyd chronyc            # from chrony
which gpsd gpsmon                # from gpsd / gpsd-clients
which ethtool                    # from ethtool
which containerlab               # from containerlab install script
```

---

## 3.1 Why Timing Matters in AI Clusters

Three separate requirements converge on the need for sub-microsecond clock synchronization across a GPU cluster:

**1. AllReduce barrier correlation.** When profiling collective operations, engineers correlate NCCL timeline events across hundreds of hosts. Without synchronized clocks, a 10 ms skew makes it impossible to determine whether rank 42 was late to a barrier because of a network issue, a kernel scheduling hiccup, or a slow GPU kernel — the events appear to overlap when they don't.

**2. RoCEv2 ECN timestamping.** DCQCN (Chapter 2) relies on the receiver NIC generating a CNP within one RTT of observing a CE-marked packet. If NIC hardware clocks diverge significantly, ECN feedback loops become incoherent — the rate controller overshoots or undershoots. Hardware timestamping (required for sub-microsecond accuracy) uses the NIC's internal PHC (PTP Hardware Clock — a dedicated oscillator and counter inside the NIC that can be disciplined via PTP), which must be disciplined to a common reference.

**3. Distributed log and trace correlation.** OpenTelemetry traces (Chapter 16) that span GPU compute, NIC events, and switch telemetry are useless for root-cause analysis if timestamps from different hosts differ by more than a few milliseconds. PTP brings that divergence to tens of nanoseconds.

NTP achieves ±1–10 ms accuracy on LAN — fine for most services, but too coarse for all three of the above. **PTP achieves ±10–100 ns** on hardware-timestamped Ethernet.

---

## 3.2 IEEE 1588v2 (PTPv2) Architecture

PTP organizes clocks into a hierarchy:

```
[GNSS / GPS antenna]
        │
[Grandmaster Clock (GM)]  ← most accurate clock in the domain
        │
[Boundary Clock (BC)]     ← synchronizes to GM, distributes to downstream
      / | \
  [BC] [BC] [BC]
    |       |
[Ordinary Clock (OC)]     ← leaf nodes (servers, NICs)
```

### Clock Types

- **Grandmaster (GM):** The root time source, typically disciplined by a GNSS receiver (Global Navigation Satellite System — GPS, Galileo, GLONASS, or BeiDou) with a PPS (Pulse Per Second) output that provides a precise 1-Hz timing reference, or a cesium oscillator. Announces timing on all PTP ports.
- **Boundary Clock (BC):** A switch or dedicated appliance that terminates PTP upstream (syncs to GM) and re-originates it downstream. Eliminates queuing delay jitter from the path.
- **Transparent Clock (TC):** A switch that forwards PTP messages unchanged but adds a *residence time* correction field, accounting for the time the message spent queued inside the switch. Simpler than BC but less accurate.
- **Ordinary Clock (OC):** A leaf (server/NIC) with a single PTP port. Either a master (GM) or slave (time recipient).

### Best Master Clock Algorithm (BMCA)

PTP clocks announce themselves via Announce messages containing clock quality attributes (grandmaster identity, clock class, accuracy, variance). BMCA selects the best master deterministically:

1. Lowest `gmPriority1` wins
2. Ties broken by `gmClockClass` (lower = more accurate)
3. Further ties broken by `gmClockAccuracy`, then `gmOffsetScaledLogVariance`
4. Finally by `gmIdentity` (EUI-64, lowest wins)

---

## 3.3 PTP Message Exchange and Timestamping

The fundamental synchronization exchange (two-step mode):

```
Master                          Slave
  |──── Sync (t1) ──────────────►|  t2 = arrival time at slave
  |──── Follow_Up (t1) ─────────►|  carries t1 (software timestamp)
  |◄─── Delay_Req ───────────────| t3 = departure time
  |──── Delay_Resp (t4) ─────────►|

offset = ((t2 - t1) - (t4 - t3)) / 2
delay  = ((t2 - t1) + (t4 - t3)) / 2
```

**Hardware timestamping** captures t1 and t2 at the NIC MAC layer — before any software or driver processing. This is what separates PTP from NTP. Without it, queuing jitter in the NIC driver (microseconds to milliseconds) overwhelms the timing signal.

Check hardware timestamp support:
```bash
ethtool -T eth0
# Look for:
# SOF_TIMESTAMPING_TX_HARDWARE
# SOF_TIMESTAMPING_RX_HARDWARE
# HWTSTAMP_FILTER_ALL
```

---

## 3.4 linuxptp in Practice

`linuxptp` is the standard Linux PTP implementation, maintained by Richard Cochran and used in production AI cluster deployments.

### Components

| Binary | Role |
|---|---|
| `ptp4l` | PTP protocol daemon; syncs PHC to master |
| `phc2sys` | Copies PHC time to the system (CLOCK_REALTIME) or vice versa |
| `pmc` | PTP Management Client; queries daemon state |
| `ts2phc` | Timestamps GNSS PPS signal into PHC (for grandmaster setup) |
| `timemaster` | Integrates ptp4l and chrony into a unified time hierarchy |

### ptp4l Configuration

```ini
# /etc/ptp4l.conf — slave (ordinary clock) config
[global]
clockServo              pi
tx_timestamp_timeout    10
summary_interval        0
logging_level           6
time_stamping           hardware
network_transport       UDPv4

[eth0]
# No section-level overrides needed for basic operation
```

```bash
# Start as slave on eth0
ptp4l -f /etc/ptp4l.conf -i eth0 -s

# Output indicating lock:
# ptp4l[12.345]: rms 8 max 24 freq -1234 +/- 12 delay 234 +/- 3
# rms = RMS offset in nanoseconds — target <100ns for good sync
```

### phc2sys — Syncing System Clock to PHC

```bash
# Sync CLOCK_REALTIME to the PHC on eth0
phc2sys -s eth0 -c CLOCK_REALTIME -O 0 -w

# -w: wait for ptp4l to achieve lock before starting
# -O: UTC offset (0 if using TAI; -37 for UTC↔TAI in 2024)
```

### pmc — Inspecting PTP State

```bash
# Query current time properties
pmc -u -b 0 'GET TIME_STATUS_NP'

# Output:
# clockIdentity        001122.fffe.334455
# locked               true
# offset               8          ← nanoseconds
# gmIdentity           aabbcc.fffe.ddeeff

# Check GM reachability
pmc -u -b 0 'GET CURRENT_DATA_SET'
```

---

## 3.5 Grandmaster Setup with GPS/PPS

For on-premises AI clusters without access to a telecom-grade timing source:

```bash
# Hardware needed: GPS receiver with PPS output (e.g., u-blox NEO-M8T)
# Connected via serial + PPS to a dedicated timing server

# gpsd reads NMEA sentences (National Marine Electronics Association serial sentences
# encoding UTC time and position from a GPS receiver) and exposes them via a socket
gpsd /dev/ttyS0 -F /var/run/gpsd.sock

# ts2phc disciplines the NIC PHC to the PPS signal
ts2phc -f /etc/ts2phc.conf -s nmea -c eth0

# ptp4l runs as master, announcing to the network
ptp4l -f /etc/ptp4l_master.conf -i eth0
# (No -s flag; this node IS the grandmaster)
```

In a cluster, typically two GMs are deployed for redundancy. BMCA selects the primary; the secondary becomes active if the primary fails (lower `gmPriority1` on primary forces it to win normally).

---

## 3.6 chrony as Fallback and Hybrid NTP/PTP

`chrony` handles NTP synchronization and integrates with PTP for environments where hardware PTP is unavailable on all hosts.

```ini
# /etc/chrony.conf — hybrid mode
# Primary: PTP via PHC reference clock
refclock PHC /dev/ptp0 poll 0 dpoll -2 offset 0 trust

# Fallback: NTP servers
server 192.168.1.1 iburst

makestep 0.1 3      # step if offset > 100ms (startup only)
rtcsync
```

```bash
chronyc tracking
# Reference ID    : PTP0 (PHC /dev/ptp0)
# Stratum         : 1
# Ref time (UTC)  : Tue Apr 22 10:00:00 2026
# System time     : 0.000000045 seconds fast of NTP time
# RMS offset      : 0.000000123 seconds  ← 123 ns
```

---

## 3.7 Deployment Patterns for AI Clusters

### Pattern 1: BC on Top-of-Rack Switches

Modern data-center switches (SONiC, SR Linux) support PTP Boundary Clock in software or hardware:

```bash
# SR Linux — enable PTP BC on a switch port
set /system ptp admin-state enable
set /system ptp mode boundary-clock
set /interface ethernet-1/1 ptp enable true
```

Each ToR syncs to the GM via its uplink and distributes precise time to all servers on its downlinks. Servers run `ptp4l` as OC in slave mode.

### Pattern 2: TC on Switches, OC on Servers

Simpler to configure; Transparent Clock mode adds residence-time correction without terminating PTP. Less accurate than BC (residual switch queuing jitter remains) but sufficient for ±500 ns in most deployments.

### Pattern 3: PTP Over Synchronous Ethernet (SyncE)

Combines PTP (phase/time) with SyncE (Synchronous Ethernet — an ITU-T standard that recovers a frequency reference from the Ethernet PHY layer clock, distributed hop-by-hop across the physical layer independently of packet timing). SyncE distributes a frequency reference over the Ethernet physical layer independently of packets, allowing much tighter frequency tracking. Used in telco networks and some hyperscale AI deployments.

---

## Lab Walkthrough 3 — PTP Domain in Containerlab

This walkthrough builds and exercises a three-node PTP topology entirely inside containers. Because Containerlab does not expose real hardware PHCs, all nodes use software timestamping (`time_stamping software`). This is still a valid environment for learning the control plane, BMCA, servo behavior, and failure scenarios.

**Estimated time:** 45–60 minutes  
**Prerequisites:** Containerlab and linuxptp installed per the Installation section above.

---

### Step 1 — Check ethtool Hardware Timestamping on the Host NIC

Before entering the container world, verify whether your physical NIC supports hardware timestamping. This is what you would configure in a production deployment.

```bash
# Replace eth0 with your actual interface name (ip link show to find it)
ethtool -T eth0
```

Expected output on a NIC with full HW timestamp support (e.g., Intel X710, Mellanox ConnectX-5):

```
Time stamping parameters for eth0:
Capabilities:
        hardware-transmit     (SOF_TIMESTAMPING_TX_HARDWARE)
        software-transmit     (SOF_TIMESTAMPING_TX_SOFTWARE)
        hardware-receive      (SOF_TIMESTAMPING_RX_HARDWARE)
        software-receive      (SOF_TIMESTAMPING_RX_SOFTWARE)
        raw-hardware          (SOF_TIMESTAMPING_RAW_HARDWARE)
PTP Hardware Clock: 0
Hardware Transmit Timestamp Modes:
        off                   (HWTSTAMP_TX_OFF)
        on                    (HWTSTAMP_TX_ON)
Hardware Receive Filter Modes:
        none                  (HWTSTAMP_FILTER_NONE)
        all                   (HWTSTAMP_FILTER_ALL)
```

Key lines to verify:
- `SOF_TIMESTAMPING_TX_HARDWARE` and `SOF_TIMESTAMPING_RX_HARDWARE` must both appear — these confirm the NIC can timestamp at the MAC layer.
- `HWTSTAMP_FILTER_ALL` must appear under receive filter modes — this allows all PTP packets to be timestamped.
- `PTP Hardware Clock: 0` indicates `/dev/ptp0` is the associated PHC device.

If you only see `software-transmit` and `software-receive`, the NIC does not support hardware PTP. Note this; the lab will use software mode regardless, but a production deployment would require a supported NIC.

```bash
# Also enumerate all PHC devices on the system
ls -la /dev/ptp*
# Expected: /dev/ptp0  (or multiple if multiple NICs)

# Read the current PHC time directly
# phc_ctl is a linuxptp utility for directly reading and setting PHC device clocks
phc_ctl /dev/ptp0 get
# Expected: phc_ctl[...]: clock time is 1745280000.123456789, Thu Apr 22 ...
```

---

### Step 2 — Write the Containerlab Topology File

Create the topology definition. Each node runs a minimal Linux container with linuxptp pre-installed.

```bash
mkdir -p ~/ptp-lab && cd ~/ptp-lab

cat > ptp-topo.yml << 'EOF'
name: ptp-domain

topology:
  nodes:
    gm:
      kind: linux
      image: ghcr.io/srl-labs/network-multitool:latest
      exec:
        - apt-get install -y linuxptp iproute2 2>/dev/null || true

    bc:
      kind: linux
      image: ghcr.io/srl-labs/network-multitool:latest
      exec:
        - apt-get install -y linuxptp iproute2 2>/dev/null || true

    oc:
      kind: linux
      image: ghcr.io/srl-labs/network-multitool:latest
      exec:
        - apt-get install -y linuxptp iproute2 2>/dev/null || true

  links:
    - endpoints: ["gm:eth1", "bc:eth1"]
    - endpoints: ["bc:eth2", "oc:eth1"]
EOF
```

Deploy the topology:

```bash
sudo containerlab deploy -t ptp-topo.yml
```

Expected output:

```
INFO[0000] Containerlab v0.54.2 started
INFO[0000] Parsing & checking topology file: ptp-topo.yml
INFO[0001] Creating lab directory: /root/clab-ptp-domain
INFO[0001] Creating container: "gm"
INFO[0002] Creating container: "bc"
INFO[0002] Creating container: "oc"
INFO[0003] Creating link: gm:eth1 <--> bc:eth1
INFO[0003] Creating link: bc:eth2 <--> oc:eth1
INFO[0005] 3 nodes, 2 links
+---+-------------------+--------------+-----------------------------+-------+
| # |       Name        | Container ID |            Image            | State |
+---+-------------------+--------------+-----------------------------+-------+
| 1 | clab-ptp-domain-bc | a1b2c3d4e5f6 | ghcr.io/srl-labs/...       | running |
| 2 | clab-ptp-domain-gm | b2c3d4e5f6a7 | ghcr.io/srl-labs/...       | running |
| 3 | clab-ptp-domain-oc | c3d4e5f6a7b8 | ghcr.io/srl-labs/...       | running |
+---+-------------------+--------------+-----------------------------+-------+
```

Verify connectivity:

```bash
# Exec into gm and ping bc
sudo docker exec clab-ptp-domain-gm ping -c 3 172.20.20.2
# Expected: 3 packets transmitted, 3 received, 0% packet loss
```

---

### Step 3 — Start ptp4l on All Three Nodes

Open three terminal windows. Start each ptp4l in software timestamping mode (required because containers do not have hardware PHCs).

**Terminal 1 — Grandmaster (gm):**

```bash
sudo docker exec -it clab-ptp-domain-gm bash

# Inside gm container:
ptp4l -i eth1 -m --time_stamping software --priority1 128 \
      --clockClass 135 --free_running 0 2>&1 | tee /tmp/gm.log
```

Expected GM startup output:

```
ptp4l[1.000]: selected /dev/ptp0 or CLOCK_REALTIME as PTP clock
ptp4l[1.001]: port 1: INITIALIZING to LISTENING on INIT_COMPLETE
ptp4l[1.002]: port 1: LISTENING to MASTER on ANNOUNCE_RECEIPT_TIMEOUT_EXPIRES
ptp4l[2.003]: port 1: MASTER to MASTER on ANNOUNCE_RECEIPT_TIMEOUT_EXPIRES
ptp4l[2.003]: selected best master clock: aabbcc.fffe.ddeeff
ptp4l[3.005]: master offset         0 s0 freq      +0 path delay         0
```

Note `MASTER` state and `master offset 0` (GM is the root; its offset from itself is zero).

**Terminal 2 — Boundary Clock (bc):**

```bash
sudo docker exec -it clab-ptp-domain-bc bash

# Inside bc container — two interfaces, eth1 toward GM, eth2 toward OC:
ptp4l -i eth1 -i eth2 -m --time_stamping software \
      --priority1 200 2>&1 | tee /tmp/bc.log
```

Expected BC output (after syncing to GM):

```
ptp4l[1.000]: port 1: INITIALIZING to LISTENING on INIT_COMPLETE
ptp4l[1.001]: port 2: INITIALIZING to LISTENING on INIT_COMPLETE
ptp4l[3.210]: port 1: LISTENING to UNCALIBRATED on RS_SLAVE
ptp4l[5.310]: port 1: UNCALIBRATED to SLAVE on MASTER_CLOCK_SELECTED
ptp4l[5.311]: port 2: LISTENING to MASTER on ANNOUNCE_RECEIPT_TIMEOUT_EXPIRES
ptp4l[6.400]: master offset        42 s2 freq    -287 path delay       120
ptp4l[7.400]: master offset        11 s2 freq    -261 path delay       118
ptp4l[8.400]: master offset         3 s2 freq    -255 path delay       119
```

Key fields in the log line `master offset 3 s2 freq -255 path delay 119`:
- `master offset 3` — current time error vs GM, in nanoseconds. Target: converge toward 0.
- `s2` — servo state: `s0`=unlocked/free-running, `s1`=first adjustment, `s2`=locked and tracking.
- `freq -255` — frequency correction applied (parts per billion, negative = slowing clock).
- `path delay 119` — one-way propagation delay to GM, nanoseconds.

**Terminal 3 — Ordinary Clock (oc):**

```bash
sudo docker exec -it clab-ptp-domain-oc bash

# Inside oc container — single interface eth1 toward BC:
ptp4l -i eth1 -m -s --time_stamping software 2>&1 | tee /tmp/oc.log
```

Expected OC output (converging):

```
ptp4l[1.000]: port 1: INITIALIZING to LISTENING on INIT_COMPLETE
ptp4l[3.500]: port 1: LISTENING to UNCALIBRATED on RS_SLAVE
ptp4l[5.600]: port 1: UNCALIBRATED to SLAVE on MASTER_CLOCK_SELECTED
ptp4l[6.700]: master offset       -87 s2 freq    +412 path delay       235
ptp4l[7.700]: master offset       -21 s2 freq    +348 path delay       233
ptp4l[8.700]: master offset        -4 s2 freq    +331 path delay       234
ptp4l[9.700]: master offset         1 s2 freq    +333 path delay       234
ptp4l[10.700]: rms    2 max    5 freq +333 +/-   4 delay 234 +/-   0
```

The `rms` line in the summary (printed when `summary_interval` elapses) reports the **RMS offset** — root-mean-square of the offset samples. In a software-timestamped container environment, values of 2–50 ns RMS are typical. In hardware-timestamped production environments, expect 5–30 ns RMS.

---

### Step 4 — Query pmc and Interpret the Output

With ptp4l running on each node, use `pmc` (PTP Management Client) to interrogate daemon state without reading log files. Run this from a fourth shell on the host, or exec into any container.

```bash
sudo docker exec clab-ptp-domain-oc bash -c \
  "pmc -u -b 0 'GET TIME_STATUS_NP'"
```

Expected output when locked:

```
sending: GET TIME_STATUS_NP
        7884c4.fffe.5a9b01-0 seq 0 RESPONSE MANAGEMENT TIME_STATUS_NP
                master_offset              4
                ingress_time               1745280125987654321
                cumulativeScaledRateOffset +0.000000000
                scaledLastGmPhaseChange    0
                gmTimeBaseIndicator        0
                lastGmPhaseChange          0x0000'0000000000000000.0000
                gmPresent                  true
                gmIdentity                 aabbcc.fffe.ddeeff
```

Key fields to interpret:
- `master_offset 4` — current offset from GM in nanoseconds. A well-converged OC shows single-digit values.
- `gmPresent true` — the OC currently sees a reachable grandmaster. This is what changes when GM is killed.
- `gmIdentity aabbcc.fffe.ddeeff` — EUI-64 identity of the elected GM.

Also query the current data set to see the end-to-end path:

```bash
sudo docker exec clab-ptp-domain-oc bash -c \
  "pmc -u -b 0 'GET CURRENT_DATA_SET'"
```

Expected output:

```
        stepsRemoved                   2
        offsetFromMaster               4.0
        meanPathDelay                  234.0
```

- `stepsRemoved 2` — this OC is 2 hops from the GM (OC → BC → GM). This correctly reflects our topology.
- `offsetFromMaster 4.0` — offset in nanoseconds (floating point in this TLV).
- `meanPathDelay 234.0` — estimated one-way delay to master.

For the RMS offset, read directly from the ptp4l log or use:

```bash
sudo docker exec clab-ptp-domain-oc bash -c \
  "pmc -u -b 0 'GET PORT_DATA_SET'"
```

The `rms` field in the running ptp4l log is the most direct way to monitor convergence. A target of `rms < 100` (nanoseconds) indicates good synchronization.

---

### Step 5 — Inject 10 ms netem Delay and Watch Servo Behavior

`tc netem` (network emulator) is a Linux traffic-control queueing discipline that adds artificial delay, jitter, and loss to a network interface, making it possible to reproduce impaired network conditions in a lab without physical hardware. We inject 10 ms on the link between GM and BC to simulate a degraded upstream path and observe how the PI servo on BC and OC reacts.

**On the BC container** (adding delay on eth1, the GM-facing interface):

```bash
sudo docker exec clab-ptp-domain-bc bash -c \
  "tc qdisc add dev eth1 root netem delay 10ms"
```

Verify the qdisc was applied:

```bash
sudo docker exec clab-ptp-domain-bc bash -c \
  "tc qdisc show dev eth1"
```

Expected:

```
qdisc netem 8001: root refcnt 2 limit 1000 delay 10ms
```

Now watch the BC ptp4l log. Within 1–2 seconds you will see the path delay estimate jump:

```
ptp4l[45.200]: master offset         3 s2 freq    -255 path delay       119
ptp4l[46.200]: master offset     10087 s2 freq    -255 path delay     10118   ← delay jumps
ptp4l[47.200]: master offset      4231 s2 freq   -4200 path delay     10122   ← servo responding
ptp4l[48.200]: master offset       812 s2 freq   -3871 path delay     10123
ptp4l[49.200]: master offset        89 s2 freq   -3420 path delay     10124
ptp4l[50.200]: master offset        12 s2 freq   -3398 path delay     10124   ← re-converged
ptp4l[51.200]: rms   18 max   87 freq -3400 +/- 120 delay 10124 +/-   2
```

Observations:
- `path delay` increased from ~119 ns to ~10124 ns — the servo correctly measured the new 10 ms path.
- `master offset` initially spiked to 10087 ns because the servo had not yet re-estimated the new delay.
- `freq` correction swung negative aggressively (PI servo's integral term winding up) then settled.
- After 5–6 cycles the servo re-locked and `rms` returned to tens of nanoseconds.

The OC (which syncs through BC) shows a similar transient but with slightly larger excursion because it is downstream of BC.

Remove the delay to restore the baseline:

```bash
sudo docker exec clab-ptp-domain-bc bash -c \
  "tc qdisc del dev eth1 root"
```

---

### Step 6 — Kill the GM Container and Verify Slave Loses Lock

This step simulates a grandmaster failure — the most impactful failure mode in a PTP domain.

**Determine the announceReceiptTimeout.** The OC will wait `announceReceiptTimeout × announceInterval` seconds before declaring the GM lost. With defaults (interval = 1 s, timeout = 3), this is 3 seconds.

Kill the GM container:

```bash
sudo docker stop clab-ptp-domain-gm
```

Watch the OC ptp4l log in real time (from the OC terminal):

```
ptp4l[120.700]: master offset        2 s2 freq    +333 path delay       234
ptp4l[121.700]: master offset        1 s2 freq    +334 path delay       234
# ... GM stopped sending Announce messages at t=122 ...
ptp4l[125.200]: port 1: SLAVE to LISTENING on ANNOUNCE_RECEIPT_TIMEOUT_EXPIRES
ptp4l[125.201]: port 1: LISTENING to MASTER on ANNOUNCE_RECEIPT_TIMEOUT_EXPIRES
```

The OC transitions from `SLAVE` to `LISTENING` (no GM visible) then — because no better master exists — to `MASTER` itself (it self-promotes as the only clock remaining). In a real cluster with a secondary GM, BMCA would elect the secondary instead.

Query pmc immediately after the GM goes down:

```bash
sudo docker exec clab-ptp-domain-oc bash -c \
  "pmc -u -b 0 'GET TIME_STATUS_NP'"
```

Expected output showing loss of GM lock:

```
sending: GET TIME_STATUS_NP
        7884c4.fffe.5a9b01-0 seq 0 RESPONSE MANAGEMENT TIME_STATUS_NP
                master_offset              0
                ingress_time               0
                cumulativeScaledRateOffset +0.000000000
                scaledLastGmPhaseChange    0
                gmTimeBaseIndicator        0
                lastGmPhaseChange          0x0000'0000000000000000.0000
                gmPresent                  false
                gmIdentity                 000000.0000.000000
```

Critical change: `gmPresent false` and `gmIdentity 000000.0000.000000` (null identity). This is the signal a monitoring system would alert on. In production, this would trigger a page:

```
ALERT: PTP grandmaster lost on host oc-01
       gmPresent=false for >5 seconds
       Action: check GM hardware, verify BMCA elected secondary GM
```

Also check the BC, which will show similar state on its eth1 port (upstream slave port):

```bash
sudo docker exec clab-ptp-domain-bc bash -c \
  "pmc -u -b 0 'GET PORT_DATA_SET'"
```

Expected:

```
        portState              LISTENING   ← was SLAVE, now hunting for master
        ...
```

**Restart the GM and verify recovery:**

```bash
sudo docker start clab-ptp-domain-gm

# Re-run ptp4l on gm (exec back into it):
sudo docker exec clab-ptp-domain-gm bash -c \
  "ptp4l -i eth1 -m --time_stamping software --priority1 128 \
   --clockClass 135 2>&1 | tee /tmp/gm.log &"
```

Within `announceReceiptTimeout` seconds (3 s), BC and OC will re-elect the GM:

```
ptp4l[145.300]: port 1: MASTER to LISTENING on ANNOUNCE_RECEIPT_TIMEOUT_EXPIRES
ptp4l[145.301]: selected best master clock: aabbcc.fffe.ddeeff
ptp4l[145.302]: port 1: LISTENING to UNCALIBRATED on RS_SLAVE
ptp4l[147.400]: port 1: UNCALIBRATED to SLAVE on MASTER_CLOCK_SELECTED
ptp4l[148.500]: master offset       -53 s2 freq    +287 path delay       234
```

Re-run pmc to confirm `gmPresent true` is restored.

---

### Step 7 — Teardown

```bash
sudo containerlab destroy -t ~/ptp-lab/ptp-topo.yml --cleanup
```

Expected:

```
INFO[0000] Destroying lab: ptp-domain
INFO[0001] Removed container: clab-ptp-domain-gm
INFO[0001] Removed container: clab-ptp-domain-bc
INFO[0001] Removed container: clab-ptp-domain-oc
INFO[0002] Removed containerlab network: clab-ptp-domain
```

---

## Summary

- Sub-microsecond clock synchronization is a hard requirement in AI clusters for collective profiling, RoCEv2 ECN, and trace correlation.
- PTP (IEEE 1588v2) achieves ±10–100 ns accuracy on hardware-timestamped Ethernet; NTP (±1–10 ms) is insufficient.
- `linuxptp` (`ptp4l`, `phc2sys`) is the standard Linux implementation; boundary clocks on switches isolate queuing jitter from leaves.
- `chrony` provides a practical NTP fallback and integrates with PHC reference clocks for hybrid environments.
- Redundant grandmasters with GNSS/PPS are the standard on-premises design.

---

## References

- IEEE Std 1588-2019: *IEEE Standard for a Precision Clock Synchronization Protocol for Networked Measurement and Control Systems*
- linuxptp project: linuxptp.sourceforge.net / github.com/richardcochran/linuxptp
- Eidson, J.C., *Measurement, Control, and Communication Using IEEE 1588*, Springer 2006
- IETF RFC 8173: *Precision Time Protocol Version 2 (PTPv2) Management Information Base*


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).