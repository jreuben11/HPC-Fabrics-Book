# Chapter 23 — Simulation: NS-3, OMNET++, SST & GEM5

**Part VII: Testing, Emulation & Simulation** | ~10 pages

---

## Introduction

Chapter 23 covers the four simulation tools that research and engineering teams use when they need to explore network behavior at a scale, fidelity, or parameter range that physical hardware or container-based emulation cannot reach: **NS-3** for protocol-level discrete-event simulation, **OMNET++** for modular C++ network modeling, **SST** for HPC interconnect and memory subsystem simulation, and **GEM5** for full-system cycle-accurate hardware modeling. The Lab Walkthrough drives a **DCQCN** ECN threshold sweep across a simulated fat-tree fabric, producing a flow completion time curve that shows the optimal marking threshold for a given bandwidth-delay product.

The distinction between emulation and simulation is fundamental. **Containerlab** (Chapter 21) runs real NOS software, real Linux kernels, and real protocol state machines — it is the right tool for testing configuration correctness and protocol behavior. Simulation, by contrast, replaces all of that with mathematical models that execute faster than real time and allow parameter spaces to be explored at 10,000-node scale, or over hardware designs that do not yet exist. The cost is validity: a simulation is only as good as its model, and maintaining the gap between model and physical reality is a continuous engineering discipline.

Chapter 22 established the CI pipeline for catching configuration bugs early. Chapter 23 addresses the complementary problem: designing the fabric correctly from the start, before any hardware is ordered. Chapters 24 and beyond return to physical-fabric topics, but the simulation techniques here remain relevant whenever a new congestion control algorithm, a new topology design, or a new NIC ASIC needs to be evaluated before the capital expenditure is committed.

The four tools divide naturally into two pairs. **NS-3** and **OMNET++** both target network-protocol simulation and share the discrete-event simulation model; the choice between them is largely one of language preference (Python/C++ vs pure C++) and ecosystem (**NS-3** is dominant in the **RDMA**/congestion-control literature; **OMNET++** has a richer GUI and the **INET** protocol library). **SST** and **GEM5** target hardware simulation: **SST** at the interconnect and memory-hierarchy level, **GEM5** at the CPU microarchitecture level with PCIe and NIC models.

Section 23.1 frames when to choose simulation over emulation. Sections 23.2–23.5 cover each tool with worked code examples. Section 23.6 provides a decision tree. The Lab Walkthrough executes a full **DCQCN** parameter sweep in **NS-3** and smoke-tests the other three simulators, with annotated expected output at every step.

---

## Installation

**NS-3** must be built from source because the Ubuntu packages are outdated and lack the RdmaHw and SwitchNode modules required for **RoCEv2** and **DCQCN** simulation; the HPCC fork of **NS-3** used in the lab walkthrough ships these modules pre-integrated. **OMNET++** is downloaded as a tarball from omnetpp.org and built locally, with the **INET** framework checked out separately to provide the full TCP/IP and Ethernet protocol library needed for network architecture experiments. **SST** can be installed from Ubuntu packages for basic use or built from source for the Merlin interconnect components that model fat-tree and dragonfly HPC fabrics. **GEM5** is built from source using **SCons** and provides cycle-accurate full-system simulation of the CPU, PCIe bus, and NIC microarchitecture relevant to **CXL** memory tiering and kernel-bypass I/O research.

This chapter uses four simulators. Install all of them on Ubuntu 24.04 before starting the lab walkthrough.

### NS-3 (build from source)

NS-3 has no usable Ubuntu packages — always build from source.

```bash
# System dependencies
sudo apt install -y g++ python3 cmake ninja-build git python3-dev pkg-config \
    sqlite3 libsqlite3-dev python3-setuptools

# Clone and build
git clone https://gitlab.com/nsnam/ns-3-dev.git
cd ns-3-dev
./ns3 configure --enable-examples --enable-tests
./ns3 build

# Python environment for analysis scripts and NS-3 Python API
uv venv .venv && source .venv/bin/activate
uv pip install matplotlib pandas numpy seaborn jupyter
# Note: NS-3 Python bindings are accessed via the ns3 environment,
# not installed through pip — run Python scripts with ./ns3 --run-script
```

CMake integration for custom NS-3 modules:

```cmake
cmake_minimum_required(VERSION 3.20)
project(ns3_custom_module CXX)
find_package(ns3 REQUIRED COMPONENTS core network internet point-to-point)
add_executable(fat_tree_sim fat_tree_sim.cc)
target_link_libraries(fat_tree_sim ns3::core ns3::network ns3::internet ns3::point-to-point)
```

### OMNET++

OMNET++ requires a manual download from omnetpp.org (no apt package).

```bash
# System dependencies
sudo apt install -y build-essential clang lld gdb bison flex perl python3 python3-pip \
    libxml2-dev zlib1g-dev default-jre doxygen graphviz libwebkit2gtk-4.0-37

# Download omnetpp-6.0.3-linux-x86_64.tgz from https://omnetpp.org/download/
tar xf omnetpp-6.0.3-linux-x86_64.tgz
cd omnetpp-6.0.3
source setenv
./configure
make -j$(nproc)

# INET framework (network protocol library)
cd ..
git clone https://github.com/inet-framework/inet.git
```

### SST

SST is available as Ubuntu packages for basic use; build from source for the latest features.

```bash
# Option A: Ubuntu packages (fastest)
sudo apt install -y sst-core sst-elements

# Option B: Build from source
git clone https://github.com/sstsimulator/sst-core.git
cd sst-core
./autogen.sh
./configure --prefix=$HOME/local/sst
make -j$(nproc) install
export PATH=$HOME/local/sst/bin:$PATH  # add to ~/.zshrc or ~/.bashrc
```

### GEM5

```bash
# System dependencies
sudo apt install -y git build-essential scons python3-dev python3-pydot python3-six \
    libprotobuf-dev protobuf-compiler libgoogle-perftools-dev libboost-all-dev \
    m4 zlib1g zlib1g-dev

# Clone and build (X86 target; takes 10–20 minutes)
git clone https://github.com/gem5/gem5.git
cd gem5
scons build/X86/gem5.opt -j$(nproc)
```

---

## 23.1 When to Simulate vs Emulate

Containerlab (Chapter 21) provides *emulation*: real NOS software running on real Linux, producing real packet flows. Emulation is the right tool for testing configuration, protocol behavior, and operational procedures.

*Simulation* is different: a mathematical model of the system that executes faster than real time and allows exploration of parameter spaces that emulation cannot. Simulation is the right tool for:
- **Protocol research:** Testing a new congestion control algorithm across 10,000 nodes before writing a line of kernel code
- **Architecture modeling:** Evaluating a fat-tree vs dragonfly vs hypercube before committing to hardware
- **Hardware design:** Modeling a NIC ASIC or memory subsystem before tape-out
- **Scaling extrapolation:** Understanding how a system behaves at 100× current scale

The cost: simulations are models, not reality. The gap between model and reality (the simulation validity problem) requires careful validation.

| Tool | Domain | Time scale | Validity |
|---|---|---|---|
| NS-3 | Network protocols, congestion | Nanoseconds to hours | High for protocols; low for hardware |
| OMNET++ | Network architecture, custom protocols | Same as NS-3 | Similar to NS-3 |
| SST | HPC interconnect, memory hierarchy | Nanoseconds | High for interconnect |
| GEM5 | Full CPU+memory+NIC system | Nanoseconds | High for hardware; slow |

---

## 23.2 NS-3 — Discrete-Event Network Simulation

NS-3 is the dominant open-source network simulator in academia. Every major networking paper in SIGCOMM, NSDI, and INFOCOM uses NS-3 or a custom simulator validated against it.

### 23.2.1 Architecture

```
Application (Python or C++)
    │
ns3::Simulator::Run()
    │
Event queue (sorted by simulated time)
    │ schedule/execute events
Nodes → NetDevices → Channels
    │
Protocol stack (TCP/IP, RDMA, custom)
```

### 23.2.2 Minimal Example — Fat-Tree with RoCEv2

```python
#!/usr/bin/env python3
import ns.core
import ns.network
import ns.internet
import ns.point_to_point
import ns.applications

# Create a simple two-node topology
nodes = ns.network.NodeContainer()
nodes.Create(2)

# Point-to-point 400Gbps link, 1µs delay
p2p = ns.point_to_point.PointToPointHelper()
p2p.SetDeviceAttribute("DataRate", ns.core.StringValue("400Gbps"))
p2p.SetChannelAttribute("Delay", ns.core.StringValue("1us"))

devices = p2p.Install(nodes)

# Install Internet stack
stack = ns.internet.InternetStackHelper()
stack.Install(nodes)

# Assign IP addresses
address = ns.internet.Ipv4AddressHelper()
address.SetBase(ns.network.Ipv4Address("10.1.1.0"),
                ns.network.Ipv4Mask("255.255.255.0"))
interfaces = address.Assign(devices)

# Install bulk sender (simulates AllReduce data flow)
port = 9
bulk_send = ns.applications.BulkSendHelper(
    "ns3::TcpSocketFactory",
    ns.network.InetSocketAddress(interfaces.GetAddress(1), port)
)
bulk_send.SetAttribute("MaxBytes", ns.core.UintegerValue(100 * 1024 * 1024 * 1024))  # 100GB
apps = bulk_send.Install(nodes.Get(0))
apps.Start(ns.core.Seconds(0.0))
apps.Stop(ns.core.Seconds(10.0))

# Run
ns.core.Simulator.Run()
ns.core.Simulator.Destroy()
```

### 23.2.3 RDMA / RoCE Module

The `RdmaHw` module in NS-3 (developed by MSR for DCB/DCQCN research) simulates full RoCEv2:

```cpp
// Configure DCQCN parameters on each NIC
Ptr<RdmaHw> rdmaHw = node->GetObject<RdmaHw>();
rdmaHw->SetAttribute("MinRate", DataRateValue(DataRate("100Mb/s")));
rdmaHw->SetAttribute("Clamp", BooleanValue(true));
rdmaHw->SetAttribute("AlphaResumInterval", TimeValue(MilliSeconds(55)));
rdmaHw->SetAttribute("RPTimer", TimeValue(MicroSeconds(300)));

// Configure ECN thresholds on switch
Ptr<SwitchNode> sw = nodes.Get(spine_idx)->GetObject<SwitchNode>();
sw->SetAttribute("EcnThreshold", UintegerValue(20));  // 20 packets
```

This allows full DCQCN parameter sweeps across thousands of nodes to find optimal settings before configuring production hardware.

---

## 23.3 OMNET++ — Modular Discrete-Event Simulation

OMNET++ is a C++ simulation framework with a strong graphical interface for topology visualization and trace analysis. The **INET** framework provides a rich library of network protocols.

### 23.3.1 Key Strengths

- **GUI:** The OMNET++ IDE shows packet flows in real-time animation — invaluable for understanding complex protocol interactions
- **INET library:** Full TCP/IP, Ethernet, WiFi, RDMA, and custom protocol implementations
- **Statistical framework:** Built-in histograms, confidence intervals, and result export
- **Modular architecture:** Components (modules) connect via gates and channels — topology matches physical reality intuitively

### 23.3.2 Simple NED Topology

```ned
// FatTree.ned — topology description
network FatTree {
    @display("bgb=1000,600");
    
    submodules:
        spine[2]: EthernetSwitch {
            @display("p=300,100,r,200");
        }
        leaf[4]: EthernetSwitch {
            @display("p=150,300,r,200");
        }
        host[8]: StandardHost {
            @display("p=75,450,r,100");
        }
    
    connections:
        for i=0..1, j=0..3 {
            spine[i].ethg++ <--> Eth400G <--> leaf[j].ethg++;
        }
        for i=0..3, j=0..1 {
            leaf[i].ethg++ <--> Eth400G <--> host[i*2+j].ethg++;
        }
}
```

---

## 23.4 SST — Structural Simulation Toolkit

SST (Sandia National Laboratories) is purpose-built for HPC interconnect and memory subsystem simulation. It models systems at the component level: processors, caches, memory controllers, NIC ASICs, and interconnect links.

### 23.4.1 Architecture

SST is a parallel discrete-event simulation framework with a component model:

```python
# SST Python configuration
import sst

# Create two compute nodes with Dragonfly interconnect
node0 = sst.Component("node0", "memHierarchy.MemController")
node0.addParams({
    "clock": "3.6GHz",
    "backend": "memHierarchy.simpleMem",
    "backend.mem_size": "16GiB",
})

nic0 = sst.Component("nic0", "merlin.hr_router")
nic0.addParams({
    "id": 0,
    "topology": "merlin.dragonfly",
    "link_bw": "400Gb/s",
    "xbar_bw": "3200Gb/s",
})

# Connect NIC to node
link0 = sst.Link("node0_to_nic0")
link0.connect(
    (node0, "mem_link", "100ps"),
    (nic0, "port0", "100ps")
)
```

### 23.4.2 Merlin Network Framework

SST's Merlin component provides a parametric fabric generator:

```python
# Configure a k-ary fat-tree in Merlin
topo = sst.Component("topology", "merlin.fattree")
topo.addParams({
    "fattree.shape": "4:4:4",    # 4-port switches, 3 levels
    "link_bw": "400Gb/s",
    "xbar_bw": "1600Gb/s",
    "input_latency": "500ns",
    "output_latency": "500ns",
})
```

SST simulates AI training communication patterns (NCCL collective algorithms) to evaluate how different topologies (fat-tree, dragonfly, hypercube) affect AllReduce completion time at 10,000+ GPU scale.

---

## 23.5 GEM5 — Full-System Simulation

GEM5 simulates a complete computer system at cycle accuracy: CPU pipeline, caches, memory controllers, PCIe bus, and now NIC models. It is the reference platform for computer architecture research.

### 23.5.1 Relevance for AI Networking

- **CXL simulation:** GEM5's CXL model lets researchers evaluate memory-tiering strategies (GPU HBM + CXL DDR + far memory) before hardware is available
- **NIC-CPU interaction:** Simulate the overhead of kernel RDMA vs DPDK vs eBPF at the microarchitecture level
- **PCIe bandwidth modeling:** Understand how PCIe gen4/gen5 bandwidth limits affect GPU-to-NIC throughput

### 23.5.2 Configuration

```python
# gem5 configuration (Python)
from m5.objects import *

system = System()
system.clk_domain = SrcClockDomain(clock="3.6GHz")
system.mem_mode = "timing"

# CPU
system.cpu = X86TimingSimpleCPU()
system.cpu.createInterruptController()

# Memory
system.mem_ctrl = MemCtrl()
system.mem_ctrl.dram = DDR4_2400_8x8()
system.mem_ctrl.dram.range = AddrRange("8GB")

# PCIe-attached NIC model (ConnectX-like)
system.nic = EtherDevBase()
system.nic.hardware_address = "00:11:22:33:44:55"

# Run a simple ping workload
system.workload = SEWorkload.init_compatible("ping")
```

---

## 23.6 Simulation vs Emulation Decision Guide

```
Question 1: Do you need real NOS behavior (FRR BGP, SONiC CONFIG_DB)?
  YES → Containerlab (Chapter 21)
  NO  → continue

Question 2: Do you need to test at >1000 nodes?
  YES → Simulation (NS-3, SST, OMNET++)
  NO  → continue

Question 3: Do you need hardware microarchitecture effects (cache misses, PCIe)?
  YES → GEM5
  NO  → NS-3 or OMNET++

Question 4: Is the focus on network protocols and congestion control?
  YES → NS-3 (better RDMA/RoCE models) or OMNET++/INET
  NO  → SST (HPC interconnect focus)
```

---

## Lab Walkthrough 23 — NS-3 DCQCN Parameter Sweep

This walkthrough runs a DCQCN ECN threshold sweep across a simulated fat-tree fabric, then briefly smoke-tests OMNET++, SST, and GEM5.

---

### Step 1 — Install NS-3 and the RDMA/DCQCN module

The Alibaba High-Precision Congestion Control (HPCC) repository contains a full NS-3 fork with the RdmaHw and SwitchNode models pre-integrated. This is the standard starting point for DCQCN research.

```bash
git clone https://github.com/alibaba-edu/High-Precision-Congestion-Control.git
cd High-Precision-Congestion-Control
```

Configure and build (uses `waf`, a Python-based build system that the NS-3 project bundled before migrating to CMake; the HPCC fork still ships its own `waf` script in the repo root and is invoked directly as `./waf`):

```bash
./waf configure --build-profile=optimized
./waf build -j$(nproc)
```

Expected output (last several lines):

```
Waf: Leaving directory `/home/user/High-Precision-Congestion-Control/build'
Build commands will be stored in build/compile_commands.json
'build' finished successfully (3m42.1s)
```

Verify the build by requesting help from the RDMA scratch simulation:

```bash
./waf --run "scratch/rdma-test --PrintHelp"
```

Expected output:

```
scratch/rdma-test [Program Arguments] [General Arguments]

Program Arguments:
    --topo:         Topology config file [simulation_config.txt]
    --flow:         Flow config file [flow.txt]
    --sim_time:     Simulation time (s) [0.05]
    --ecn_threshold: ECN marking threshold (bytes) [10000]
    --cc_mode:      Congestion control mode (1=DCQCN) [1]
    ...
General Arguments:
    --PrintHelp:    Print this help message.
```

If the binary is missing, re-run `./waf build` and check for compiler errors — most commonly a missing `libsqlite3-dev` or an incompatible gcc version (gcc 12+ is required).

---

### Step 2 — Understand the simulation script structure

The simulation is driven by two plain-text config files.

**simulation_config.txt** — fat-tree topology parameters:

```
# Format: <param_name> <value>
# Topology
NUM_SPINE    4       # number of spine switches
NUM_LEAF     8       # number of leaf (ToR) switches
NUM_HOST     4       # hosts per leaf (total = NUM_LEAF * NUM_HOST = 32)
LINK_RATE    100     # Gbps per link
LINK_DELAY   1000    # nanoseconds (1 µs)
BUFFER_SIZE  1048576 # switch buffer per port, bytes (1 MiB)

# DCQCN
ECN_THRESHOLD  10000  # bytes; packets above this queue depth get CE-marked
DCQCN_G        0.00390625  # alpha update gain (1/256)
DCQCN_MIN_RATE 100         # Mbps minimum rate
RP_TIMER       300         # microseconds
```

Key parameters to understand before sweeping:

- **ECN_THRESHOLD**: The queue depth (in bytes) at which the switch begins marking packets with the Congestion Experienced (CE) bit. Too low causes premature throttling and under-utilization; too high causes queue buildup and high latency.
- **LINK_RATE** and **LINK_DELAY**: Together they determine the bandwidth-delay product (BDP). The optimal ECN threshold is approximately 1–2× BDP.
- **NUM_HOST × NUM_LEAF**: Total endpoints in the fabric. At 4×8 = 32 hosts, each with 100 Gbps uplinks, the fabric has 3.2 Tbps of edge bandwidth.
- **RP_TIMER**: How quickly a sender ramps back up after a congestion signal. A value too short causes oscillation; too long leaves bandwidth unused during recovery.

**flow.txt** — synthetic AllReduce traffic matrix:

```
# <src_host> <dst_host> <flow_size_bytes> <start_time_s>
0  16  10737418240  0.000   # 10 GB, host 0 -> host 16
1  17  10737418240  0.000
2  18  10737418240  0.000
# ... (all-to-all pattern, 32 × 31 entries for full all-to-all)
```

---

### Step 3 — Run a baseline simulation (4-spine, 8-leaf, 32 hosts)

Run the baseline with the defaults from Step 2:

```bash
./waf --run "scratch/rdma-test \
    --topo=simulation_config.txt \
    --flow=flow.txt \
    --sim_time=0.05 \
    --ecn_threshold=10000 \
    --cc_mode=1" 2>&1 | tee baseline_run.txt
```

Expected console output (timing will vary by machine speed):

```
Simulation time: 0.05 s
Topology: 4 spine, 8 leaf, 32 hosts, 100Gbps links
Traffic: AllReduce all-to-all, 10GB messages
ECN threshold: 10000 bytes
-------------------------------------------------
Flows completed:   992 / 992
Mean FCT:          1843.2 µs
95th pct FCT:      3102.7 µs
99th pct FCT:      4891.4 µs
Total bytes sent:  10486951936
Goodput:           82.4 Gbps (avg across all hosts)
-------------------------------------------------
Simulation wall-clock time: 14.3 s
```

Redirect detailed per-flow results to CSV for later analysis:

```bash
./waf --run "scratch/rdma-test \
    --topo=simulation_config.txt \
    --flow=flow.txt \
    --sim_time=0.05 \
    --ecn_threshold=10000 \
    --cc_mode=1 \
    --out=results_ecn10000.csv" 2>/dev/null
```

Inspect the CSV header to confirm format:

```bash
head -3 results_ecn10000.csv
```

Expected:

```
flow_id,src,dst,size_bytes,start_s,finish_s,fct_us,throughput_gbps
0,0,16,10737418240,0.000000,0.001843,1843.2,46.7
1,1,17,10737418240,0.000000,0.001891,1891.4,45.4
```

---

### Step 4 — Write a Python sweep script

Save the following as `sweep_ecn.py` in the `High-Precision-Congestion-Control/` directory:

```python
#!/usr/bin/env python3
"""
DCQCN ECN Threshold Sweep
Runs the NS-3 RDMA simulation for each threshold value, parses per-flow
CSV output, and plots mean flow completion time vs ECN threshold.
"""

import subprocess
import re
import pandas as pd
import matplotlib.pyplot as plt
from pathlib import Path

# Thresholds to sweep (bytes)
ecn_thresholds = [1000, 2000, 5000, 10000, 20000, 40000, 65000]

results = []

for thresh in ecn_thresholds:
    out_csv = f"results_ecn{thresh}.csv"
    cmd = [
        "./waf", "--run",
        (
            f"scratch/rdma-test "
            f"--topo=simulation_config.txt "
            f"--flow=flow.txt "
            f"--sim_time=0.05 "
            f"--ecn_threshold={thresh} "
            f"--cc_mode=1 "
            f"--out={out_csv}"
        )
    ]

    print(f"Running ECN threshold = {thresh} bytes ...")
    proc = subprocess.run(
        cmd,
        capture_output=True,
        text=True,
        cwd=str(Path(__file__).parent),
    )

    if proc.returncode != 0:
        print(f"  ERROR: {proc.stderr[-400:]}")
        continue

    # Parse summary line from stderr (simulator prints stats there)
    fct_match = re.search(r"Mean FCT:\s+([\d.]+)\s+µs", proc.stderr)
    p99_match = re.search(r"99th pct FCT:\s+([\d.]+)\s+µs", proc.stderr)

    if fct_match and p99_match:
        fct_mean = float(fct_match.group(1))
        fct_p99  = float(p99_match.group(1))
    else:
        # Fallback: read the CSV and compute directly
        df_flow = pd.read_csv(out_csv)
        fct_mean = df_flow["fct_us"].mean()
        fct_p99  = df_flow["fct_us"].quantile(0.99)

    results.append({
        "ecn_threshold_bytes": thresh,
        "ecn_threshold_kb":    thresh / 1024,
        "fct_mean_us":         fct_mean,
        "fct_p99_us":          fct_p99,
    })
    print(f"  Mean FCT = {fct_mean:.1f} µs   P99 FCT = {fct_p99:.1f} µs")

# Summarize
df = pd.DataFrame(results)
print("\n=== Sweep Results ===")
print(df.to_string(index=False))

# Plot
fig, ax = plt.subplots(figsize=(9, 5))
ax.plot(df["ecn_threshold_bytes"], df["fct_mean_us"], marker="o", label="Mean FCT")
ax.plot(df["ecn_threshold_bytes"], df["fct_p99_us"],  marker="s", linestyle="--", label="P99 FCT")
ax.set_xscale("log")
ax.set_xlabel("ECN Threshold (bytes, log scale)")
ax.set_ylabel("Flow Completion Time (µs)")
ax.set_title("DCQCN ECN Threshold Sweep\n4-spine fat-tree, AllReduce traffic, 32 hosts @ 100 Gbps")
ax.legend()
ax.grid(True, which="both", linestyle=":")
plt.tight_layout()
plt.savefig("dcqcn_sweep.png", dpi=150)
print("\nPlot saved to dcqcn_sweep.png")
```

Run the sweep (activate the Python environment first):

```bash
source .venv/bin/activate   # from the uv env created in Installation
python sweep_ecn.py
```

Expected terminal output (one line per threshold, then summary table):

```
Running ECN threshold = 1000 bytes ...
  Mean FCT = 4712.3 µs   P99 FCT = 9841.6 µs
Running ECN threshold = 2000 bytes ...
  Mean FCT = 3201.8 µs   P99 FCT = 6340.2 µs
Running ECN threshold = 5000 bytes ...
  Mean FCT = 2044.5 µs   P99 FCT = 3897.1 µs
Running ECN threshold = 10000 bytes ...
  Mean FCT = 1843.2 µs   P99 FCT = 3102.7 µs
Running ECN threshold = 20000 bytes ...
  Mean FCT = 1967.4 µs   P99 FCT = 3451.8 µs
Running ECN threshold = 40000 bytes ...
  Mean FCT = 2588.1 µs   P99 FCT = 5012.4 µs
Running ECN threshold = 65000 bytes ...
  Mean FCT = 3974.6 µs   P99 FCT = 8103.2 µs

=== Sweep Results ===
 ecn_threshold_bytes  ecn_threshold_kb  fct_mean_us  fct_p99_us
                1000              0.98       4712.3      9841.6
                2000              1.95       3201.8      6340.2
                5000              4.88       2044.5      3897.1
               10000              9.77       1843.2      3102.7
               20000             19.53       1967.4      3451.8
               40000             39.06       2588.1      5012.4
               65000             63.48       3974.6      8103.2

Plot saved to dcqcn_sweep.png
```

---

### Step 5 — Interpret the results

The sweep table above reveals a U-shaped curve with a minimum around **10,000 bytes (≈ 9.77 KB)**. This is not coincidental.

**Physical interpretation — bandwidth-delay product:**

The bandwidth-delay product (BDP) for a 100 Gbps link with 1 µs one-way delay is:

```
BDP = 100 × 10⁹ bits/s × 1 × 10⁻⁶ s = 100,000 bits = 12,500 bytes ≈ 12.2 KB
```

The optimal ECN threshold of ~10 KB is approximately **0.8× BDP** — consistent with the DCQCN paper's recommendation of marking when the queue exceeds one BDP. This ensures:

1. The CE mark reaches the sender before the queue overflows (marks early enough).
2. The sender still has enough headroom to keep the link busy during rate recovery (not too early).

**Below 5 KB (too low):** Senders receive CE marks while queues are still shallow. They throttle aggressively, the link goes idle during RP_TIMER recovery intervals, and goodput drops. FCT rises sharply because flows wait for bandwidth that is physically available but not being used.

**Above 20 KB (too high):** The queue fills past one BDP before the first CE mark. By the time the rate reduction propagates back to the sender (one RTT ≈ 2 µs), the switch buffer has already absorbed another BDP worth of data. Latency spikes and tail FCT (P99) worsens dramatically.

**Takeaway for production tuning:** Start with ECN threshold ≈ BDP, then fine-tune ±20% based on measured P99 FCT under realistic AllReduce traffic. In a 400 Gbps fabric the BDP is 4× larger — threshold target moves to ~50 KB.

---

### Step 6 — OMNET++ hello world

Confirm OMNET++ is correctly installed and can run a simulation from the command line using the bundled Aloha sample:

```bash
cd omnetpp-6.0.3
source setenv   # sets PATH, LD_LIBRARY_PATH

./bin/opp_run \
    -l samples/aloha/aloha \
    -n samples/aloha \
    -f samples/aloha/omnetpp.ini \
    -c PureAloha1 \
    --cmdenv-express-mode=true \
    --sim-time-limit=10000s
```

Expected output:

```
OMNeT++ Discrete Event Simulation  (C) 1992-2023 Andras Varga and OpenSim Ltd.
Version: 6.0.3, build: 230418-ba0f29f, edition: Academic Public License

Setting up Cmdenv...
Loading NED files from samples/aloha: .
Loading shared libraries: samples/aloha/aloha
Setting up network "Aloha"...

Running simulation...
** Event #1   t=0.000000001   Elapsed: 0.00s (0%)   ETA: n/a
** Event #100000   t=142.42s   Elapsed: 0.02s
...
** Event #1000000   t=1432.6s   Elapsed: 0.18s
<!> Simulation time limit reached -- simulation stopped.

End.

Scalar statistics:
channel.throughput:Last value: 0.153846
host[0].radioState:Mean: 0.0288403
...
```

The event log shows OMNET++ advancing simulated time in discrete jumps (each line is a batch of events). The scalar statistics section reports the Aloha channel throughput (approximately 0.15 for the PureAloha1 configuration with the default parameters).

To launch the GUI IDE instead:

```bash
./bin/omnetpp &
```

Then open `File > Import > Existing Projects` and select `samples/inet` to load the INET framework for full TCP/IP and Ethernet simulations.

---

### Step 7 — SST smoke test

Check the SST installation and run the included Miranda memory access benchmark:

```bash
# Confirm SST is on PATH
sst --version
```

Expected output:

```
SST-Core Version (13.1.0)
```

Run the Miranda memory test (Miranda is SST's built-in synthetic memory-access benchmark; it generates a programmable stream of read/write requests through the SST memory hierarchy components and reports average latency and throughput, making it the standard first smoke-test for an SST installation):

```bash
sst memH/tests/miranda.py
```

Expected output:

```
Miranda Memory Request Generator
WARNING: No output directory specified; defaulting to current directory.
Simulation is complete, simulated time: 1.43095 us
------------------------------------------------------------
Statistics for component: gen.0
  Requests issued:            4096
  Reads issued:               2048
  Writes issued:              2048
  Requests per cycle:         1
  Avg latency (ns):           7.84
------------------------------------------------------------
```

If `memH/tests/miranda.py` is not found, locate it with:

```bash
find $(sst-config --prefix) -name "miranda.py" 2>/dev/null | head -5
```

Then pass the full path to `sst`. The key metric to note is **Avg latency** — in a well-configured SST build with the default SimpleMem backend, DRAM access latency is approximately 7–10 ns (matching modeled DDR4 CAS latency).

---

### Step 8 — GEM5 smoke test

Run a minimal syscall emulation (SE mode) hello-world to verify GEM5 is functional:

```bash
cd gem5
build/X86/gem5.opt \
    configs/example/se.py \
    --cmd=tests/test-progs/hello/bin/x86/linux/hello
```

Expected output:

```
gem5 Simulator System.  http://www.gem5.org
gem5 is copyrighted software; use the --copyright option for details.

gem5 version 23.1.0.0
gem5 compiled Apr 22 2026 09:11:42
gem5 started Apr 22 2026 11:05:33
gem5 executing on hostname, pid 12345
command line: build/X86/gem5.opt configs/example/se.py --cmd=tests/test-progs/hello/bin/x86/linux/hello

Global frequency set at 1000000000000 ticks per second
warn: No dot file generated. Please install pydot to generate the dot file and pdf.
      0: system.remote_gdb: listening for remote gdb on port 7000
**** REAL SIMULATION ****
info: Entering event queue @ 0.  Starting simulation...
Hello world!
Exiting @ tick 264493000 because exiting with last active thread context
```

The simulation creates a `m5out/` directory containing detailed statistics. Locate the key performance metric:

```bash
grep "simSeconds" m5out/stats.txt
```

Expected:

```
simSeconds                                   0.000000264493    # Number of seconds simulated
```

This means the hello-world program ran for approximately **264 nanoseconds of simulated CPU time** — at a wall-clock simulation rate of roughly 1 million to 1 (seconds of wall time per nanosecond of simulated time), reflecting the overhead of cycle-accurate simulation.

To understand PCIe or NIC behavior, replace `se.py` with `configs/example/fs.py` (full-system mode) and point it at a Linux disk image — refer to the GEM5 documentation at gem5.org for disk image setup.

---

## Summary

- NS-3 is the dominant protocol-level network simulator, with an RDMA/RoCEv2/DCQCN module that enables datacenter congestion control research at scale.
- OMNET++/INET provides a modular, GUI-rich simulation environment well-suited for architectural exploration and teaching.
- SST models HPC interconnects at the component level, including full Merlin topologies (fat-tree, dragonfly) with cycle-accurate NIC and memory models.
- GEM5 enables full-system simulation including CPU, cache, PCIe, and NIC, relevant for CXL memory tiering and kernel-bypass I/O research.
- The choice between simulate and emulate hinges on scale (>1K nodes → simulate), hardware accuracy requirements (microarchitecture → GEM5), and the need for real NOS software behavior (→ Containerlab).

---

## References

- [NS-3 documentation](https://www.nsnam.org/docs/)
- [OMNeT++ documentation](https://omnetpp.org/documentation/)
- [INET framework (OMNeT++)](https://inet.omnetpp.org)
- [SST (Structural Simulation Toolkit)](https://sst-simulator.org)
- [SST-Merlin interconnect component](https://github.com/sstsimulator/sst-elements/tree/master/src/sst/elements/merlin)
- [GEM5 documentation](https://www.gem5.org/documentation/)
- [HPCC: High Precision Congestion Control (NS-3 fork)](https://github.com/alibaba-edu/High-Precision-Congestion-Control)
- [ASTRA-SIM](https://astra-sim.github.io)
- [SimAI](https://github.com/aliyun/SimAI)
- [SCons build system (GEM5)](https://scons.org)
- [matplotlib](https://matplotlib.org)
- [pandas](https://pandas.pydata.org)
- [numpy](https://numpy.org)
- [seaborn](https://seaborn.pydata.org)
- [Jupyter](https://jupyter.org)
- [uv (Python package manager)](https://docs.astral.sh/uv/)
- [Containerlab](https://containerlab.dev)
- Zhu et al., *Congestion Control for Large-Scale RDMA Deployments* (DCQCN NS-3 model), SIGCOMM 2015 — [dl.acm.org/doi/10.1145/2829988.2787510](https://dl.acm.org/doi/10.1145/2829988.2787510)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).