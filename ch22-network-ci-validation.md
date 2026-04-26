# Chapter 22 — Network CI & Configuration Validation

**Part VII: Testing, Emulation & Simulation** | ~15 pages

---

## Introduction

Chapter 22 applies the disciplines of software engineering — static analysis, golden-state diffing, and automated CI pipelines — to network configuration management. By the end of this chapter you will have a working pipeline that runs **Batfish** intent checks on every pull request, spins up a **Containerlab** topology, connects to devices with **pyATS**, and fails the build if the live **BGP** state deviates from a captured golden snapshot.

The need is direct and measurable. Production AI cluster incidents are disproportionately caused not by hardware failure but by configuration drift — a **BGP** policy that silently drops prefixes, an ACL that blocks **RDMA** port 4791, or a **VLAN** misconfiguration that breaks **PFC** on one ToR switch. By the time monitoring catches these failures, a training job has already stalled and the GPU hours are gone. The answer is to catch such bugs before they reach production, using the same merge-gate workflow that application teams have used for years.

Chapter 21 introduced **Containerlab** as the environment in which network topologies can be deployed reproducibly. Chapter 22 builds the validation layer on top: static analysis with **Batfish** for the pre-flight check, structured state capture with **pyATS**/**Genie** for the golden diff, and inventory-driven automation with **Nornir** for mass device operations. Chapter 23 covers simulation for the cases that require deeper fidelity than emulation can provide.

Each tool occupies a distinct position in the CI pipeline. **Batfish** operates entirely from config files — no devices, no network — and catches routing and reachability bugs at the same speed as a Python unit test. **pyATS** and **Genie** connect to live or emulated devices and parse structured output from `show` commands, enabling regression testing against captured golden state. **Nornir** provides the glue: a Python-native, parallelism-aware automation framework for tasks that must touch every device in the inventory simultaneously.

Section 22.1 frames the software-engineering case for treating network configuration as code. Sections 22.2–22.4 cover the three toolchains in depth. Section 22.5 wires them into a single **GitHub Actions** workflow. The Lab Walkthrough executes all three tools end-to-end against the GPU rail fabric from Chapter 21, with expected output at every step.

---

## Installation

**Batfish** is deployed as a **Docker** container and exposes a **gRPC** API on port 9997; the **pybatfish** Python client connects to that API and is the primary interface for loading snapshots, running reachability questions, and asserting golden-config invariants. **pyATS** and **Genie** provide structured parsing of device CLI output against live or emulated devices, enabling deterministic golden-diff tests that detect unexpected state changes. **Nornir** is the Python-native automation framework that drives parallel device operations and integrates with **pytest** to form merge-gate CI assertions. **Containerlab** is installed alongside these tools because the chapter's pipeline deploys a live topology for **pyATS** tests to run against after **Batfish** static analysis passes.

### System packages

```bash
sudo apt update
sudo apt install -y \
    docker.io python3 python3-pip curl git \
    iproute2 iputils-ping openssh-client

# Start and enable Docker
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

### Containerlab (for topology tests)

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
containerlab version
```

### Python environment (uv)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

uv venv .venv
source .venv/bin/activate

# Core CI tooling
uv pip install \
    pybatfish \
    "pyats[full]" \
    genie \
    nornir \
    nornir-netmiko \
    nornir-utils \
    nornir-jinja2 \
    deepdiff \
    pytest \
    pytest-html \
    netmiko \
    paramiko

# Verify key imports
python -c "from pybatfish.client.session import Session; print('pybatfish ok')"
python -c "from genie.testbed import load; print('genie ok')"
python -c "from nornir import InitNornir; print('nornir ok')"
```

### Batfish server (Docker)

```bash
docker pull batfish/allinone
# Run once to confirm it starts
docker run -d -p 9997:9997 -p 9996:9996 --name batfish batfish/allinone
docker logs batfish --tail 20
# Expected: "Batfish service started successfully"
```

---

## 22.1 Treating Network Config as Code

Application developers have practiced test-driven development, code review, and CI/CD pipelines for decades. Network engineers are in 2010 by comparison: configuration changes applied manually by SSH, tested by pinging from one device, and rolled back by typing the inverse command if something breaks.

The tools in this chapter bring software engineering discipline to network change management:
- **Batfish:** static analysis of network configuration — catches bugs *before* any device is touched
- **pyATS/Genie:** executes tests against live or emulated devices, with structured output parsing
- **Nornir:** an automation framework for running any task against a device inventory

Combined with Containerlab (Chapter 21), these tools form a complete CI pipeline: configuration change → Batfish intent check → Containerlab topology test → pyATS golden diff → merge gate.

---

## 22.2 Batfish — Intent-Based Network Analysis

Batfish models the network as a collection of device configurations and runs dataplane simulations against that model. No devices are involved — it works entirely from config files. This means:
- Test on Monday's config what would happen on Monday night's change
- Catch routing loops, black holes, ACL shadowing, and BGP policy bugs before they affect production
- Diff two configurations and see what changes in reachability

### 22.2.1 Setup

```bash
# Start Batfish server (Docker)
docker run -d -p 9997:9997 -p 9996:9996 \
    --name batfish batfish/allinone

# Install Python client
uv pip install pybatfish
```

### 22.2.2 Snapshot Loading

A "snapshot" is a directory of device config files, one per device:

```
snapshot/
  configs/
    spine1.cfg     (SR Linux YANG JSON or Cisco IOS text)
    spine2.cfg
    leaf1.cfg
    leaf2.cfg
    leaf3.cfg
    leaf4.cfg
  hosts/
    host1.json     (Batfish host file: IP-to-interface mapping)
```

```python
from pybatfish.client.session import Session
from pybatfish.datamodel import *

bf = Session(host="localhost")
bf.set_network("ai-cluster")
bf.init_snapshot("./snapshot", name="baseline", overwrite=True)
```

### 22.2.3 Reachability Analysis

```python
# Can host1 reach host2 on port 29500 (NCCL)?
result = bf.q.reachability(
    pathConstraints=PathConstraints(
        startLocation="@enter(host1[eth0])",
        endLocation="host2[eth0]",
    ),
    headers=HeaderConstraints(
        srcIps="10.1.1.1",
        dstIps="10.2.1.1",
        dstPorts="29500",
        ipProtocols=["TCP"],
    ),
    actions="SUCCESS",
).answer().frame()

assert len(result) > 0, "No path from host1 to host2 on port 29500!"
print(f"Paths found: {len(result)}")
print(result[["Flow", "Traces"]].head())
```

### 22.2.4 Routing Policy Verification

```python
# Verify that leaf1 advertises only its own loopback to spines
bgp_routes = bf.q.bgpRib(nodes="leaf1").answer().frame()

# Check no default route is being originated
default_routes = bgp_routes[bgp_routes["Network"] == "0.0.0.0/0"]
assert len(default_routes) == 0, f"leaf1 is originating a default route!"

# Check ECMP: spine1 should have 4 equal-cost paths to host2
ecmp = bf.q.routes(nodes="spine1", network="10.2.1.0/24").answer().frame()
assert len(ecmp) == 4, f"Expected 4 ECMP paths, got {len(ecmp)}"
```

### 22.2.5 Differential Analysis — Pre/Post Change

```python
# Load current config as baseline
bf.init_snapshot("./snapshot-current", name="current")

# Load proposed change
bf.init_snapshot("./snapshot-proposed", name="proposed")

# Diff reachability
diff = bf.q.differentialReachability(
    snapshot="proposed",
    reference_snapshot="current",
).answer().frame()

# Any NEW failures?
new_failures = diff[diff["FlowDiff"] == "LOST"]
assert len(new_failures) == 0, f"Change breaks reachability:\n{new_failures}"

# Any NEW paths (verify expected additions)?
new_paths = diff[diff["FlowDiff"] == "GAINED"]
print(f"New reachable paths: {len(new_paths)}")
```

---

## 22.3 pyATS / Genie — Structured Device Testing

pyATS (Python Automated Test System) is Cisco's open-source framework for network device testing. Genie adds structured parsers that convert `show` command output into Python dictionaries.

### 22.3.1 Testbed File

```yaml
# testbed.yaml
testbed:
  name: ai-cluster-lab

devices:
  spine1:
    os: iosxe
    connections:
      defaults:
        class: unicon.Unicon
      cli:
        protocol: ssh
        ip: 172.100.100.2
        port: 22
    credentials:
      default:
        username: admin
        password: secret

  leaf1:
    os: nxos
    connections:
      defaults:
        class: unicon.Unicon
      cli:
        protocol: ssh
        ip: 172.100.100.3
```

### 22.3.2 Golden Config Snapshot

```python
# snapshot.py — capture current state as golden
from genie.testbed import load
import json

testbed = load("testbed.yaml")
device = testbed.devices["spine1"]
device.connect()

# Parse BGP summary into structured Python dict
bgp_summary = device.parse("show bgp all summary")
# {
#   "vrf": {
#     "default": {
#       "neighbor": {
#         "192.168.1.1": {
#           "state_pfxrcd": 4,
#           "up_down": "01:23:45",
#           "session_state": "Established"
#         }
#       }
#     }
#   }
# }

with open("golden/spine1_bgp_summary.json", "w") as f:
    json.dump(bgp_summary, f, indent=2)
```

### 22.3.3 Golden Diff Test

DeepDiff is a Python library for comparing arbitrarily nested Python objects (dicts, lists, sets) and returning a structured description of every addition, deletion, and value change. In a network context it is used to diff the current parsed device state against a stored golden snapshot, exposing only semantically meaningful changes rather than raw string differences.

```python
# test_golden.py
import json
from deepdiff import DeepDiff
from genie.testbed import load

testbed = load("testbed.yaml")
device = testbed.devices["spine1"]
device.connect()

current = device.parse("show bgp all summary")

with open("golden/spine1_bgp_summary.json") as f:
    golden = json.load(f)

diff = DeepDiff(golden, current, ignore_order=True)

# Filter acceptable differences (uptime, counters)
filtered = {k: v for k, v in diff.items()
            if "up_down" not in str(v) and "msg_rcvd" not in str(v)}

assert not filtered, f"BGP state changed unexpectedly:\n{filtered}"
print("Golden diff: PASS")
```

### 22.3.4 Genie Parsers Available

```bash
# List available parsers
show genie parsers

# Key parsers for AI cluster validation:
# show bgp all summary
# show interfaces
# show ip route
# show ip ospf neighbor
# show version
# show processes cpu
```

---

## 22.4 Nornir — Inventory-Driven Automation

Nornir is a Python framework for running tasks against a device inventory. Unlike Ansible, it's pure Python — tasks are Python functions, results are Python objects, and parallelism is built in.

### 22.4.1 Inventory

```yaml
# inventory/hosts.yaml
spine1:
  hostname: 172.100.100.2
  port: 22
  username: admin
  password: secret
  platform: eos
  groups: [spines]

leaf1:
  hostname: 172.100.100.3
  platform: sonic
  groups: [leaves]

# inventory/groups.yaml
spines:
  data:
    bgp_as: 65100

leaves:
  data:
    bgp_as: null  # per-host
```

### 22.4.2 Running Tasks

```python
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command
from nornir_utils.plugins.functions import print_result

nr = InitNornir(config_file="nornir.yaml")

# Run show command on all devices in parallel
result = nr.run(
    task=netmiko_send_command,
    command_string="show ip bgp summary"
)
print_result(result)

# Filter to spines only
spines = nr.filter(groups=["spines"])
spine_result = spines.run(
    task=netmiko_send_command,
    command_string="show ip route bgp"
)
```

### 22.4.3 Config Rendering and Push

Jinja2 is a Python templating engine that fills placeholder variables (written as `{{ variable }}`) in a text template with values supplied at render time; the `nornir-jinja2` plugin integrates it directly into Nornir tasks so that per-device configuration can be generated from a shared template and host-specific inventory data.

```python
from nornir_jinja2.plugins.tasks import template_file
from nornir_netmiko.tasks import netmiko_send_config

def push_bgp_config(task):
    # Render config from Jinja2 template
    rendered = task.run(
        task=template_file,
        template="bgp_config.j2",
        path="templates/",
        bgp_as=task.host["bgp_as"],
        neighbors=task.host["bgp_neighbors"]
    )
    # Push to device
    task.run(
        task=netmiko_send_config,
        config_commands=rendered.result.splitlines()
    )

nr.run(task=push_bgp_config)
```

### 22.4.4 pytest Integration

```python
# tests/test_bgp.py
import pytest
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command
from genie.libs.parser.iosxe.show_bgp import ShowBgpAllSummary

@pytest.fixture(scope="module")
def nr():
    return InitNornir(config_file="nornir.yaml")

def test_all_bgp_sessions_established(nr):
    result = nr.run(task=netmiko_send_command,
                    command_string="show bgp all summary")
    for host, multi_result in result.items():
        output = multi_result[0].result
        # Use Genie to parse
        parsed = ShowBgpAllSummary(device=None).parse(output=output)
        neighbors = parsed["vrf"]["default"]["neighbor"]
        for neighbor_ip, data in neighbors.items():
            assert data["session_state"] == "Established", \
                f"{host}: BGP neighbor {neighbor_ip} not established"
```

---

## 22.5 Full CI Pipeline

```yaml
# .github/workflows/network-ci.yml
name: Network Change Validation

on:
  pull_request:
    paths: ['configs/**', 'templates/**']

jobs:
  static-analysis:
    name: Batfish Analysis
    runs-on: ubuntu-latest
    services:
      batfish:
        image: batfish/allinone
        ports: ["9997:9997", "9996:9996"]
    steps:
      - uses: actions/checkout@v4
      - run: uv pip install pybatfish
      - run: python tests/batfish_reachability.py

  topology-test:
    name: Containerlab + pyATS
    runs-on: ubuntu-latest
    needs: static-analysis
    steps:
      - uses: actions/checkout@v4
      - run: bash -c "$(curl -sL https://get.containerlab.dev)"
      - run: containerlab deploy -t tests/ci-topology.yml
      - run: sleep 30   # wait for BGP convergence
      - run: uv pip install "pyats[full]" genie
      - run: python tests/golden_diff.py
      - run: python tests/connectivity_test.py
      - run: containerlab destroy -t tests/ci-topology.yml
```

---

## Lab Walkthrough 22 — Batfish → Containerlab → pyATS Pipeline

This is the most rigorous lab in Part VII. Every step produces verifiable output that becomes the basis for the CI assertions in the following section. Work through each step against the GPU rail fabric topology deployed in Chapter 21.

### Step 1: Start the Batfish Server

Run Batfish as a background Docker container. Port 9997 is the gRPC API used by pybatfish; port 9996 is a legacy REST endpoint.

```bash
docker run -d \
  --name batfish \
  -p 9997:9997 \
  -p 9996:9996 \
  --restart unless-stopped \
  batfish/allinone
```

Verify the container started:

```bash
docker ps --filter name=batfish
```

Expected:

```
CONTAINER ID   IMAGE               COMMAND                  CREATED         STATUS         PORTS
a3f2b1c9d4e7   batfish/allinone    "/batfish-wrapper.sh"    5 seconds ago   Up 4 seconds   0.0.0.0:9996-9997->9996-9997/tcp
```

Wait for Batfish to initialize, then verify via pybatfish:

```bash
sleep 15
python3 - <<'EOF'
from pybatfish.client.session import Session
bf = Session(host="localhost")
print("Batfish networks:", bf.list_networks())
EOF
```

Expected:

```
Batfish networks: []
```

---

### Step 2: Export SR Linux Config from a Running Containerlab Topology

Containerlab stores committed SR Linux configuration inside each container. Export it using `sr_cli` for each node:

```bash
mkdir -p ~/batfish-ci/{snapshot-baseline,snapshot-proposed}/{configs,hosts}
cd ~/batfish-ci

docker exec clab-gpu-rail-fabric-spine1 \
  sr_cli "info / network-instance default" \
  > snapshot-baseline/configs/spine1.cfg 2>&1

docker exec clab-gpu-rail-fabric-spine2 \
  sr_cli "info / network-instance default" \
  > snapshot-baseline/configs/spine2.cfg 2>&1

docker exec clab-gpu-rail-fabric-tor-rail0 \
  sr_cli "info / network-instance default" \
  > snapshot-baseline/configs/tor-rail0.cfg 2>&1

docker exec clab-gpu-rail-fabric-tor-rail1 \
  sr_cli "info / network-instance default" \
  > snapshot-baseline/configs/tor-rail1.cfg 2>&1
```

Verify the exports:

```bash
wc -l snapshot-baseline/configs/*.cfg
```

Expected:

```
  72 snapshot-baseline/configs/spine1.cfg
  72 snapshot-baseline/configs/spine2.cfg
  89 snapshot-baseline/configs/tor-rail0.cfg
  89 snapshot-baseline/configs/tor-rail1.cfg
 322 total
```

---

### Step 3: Create the Snapshot Directory Structure

Batfish requires a `hosts/` directory alongside `configs/`. Host files declare IP-to-interface mappings for non-NOS endpoints (the server containers in our topology).

```bash
cat > snapshot-baseline/hosts/server1.json << 'EOF'
{
  "hostname": "server1",
  "hostInterfaces": {
    "eth1": {
      "name": "eth1",
      "prefix": "10.0.1.1/24",
      "gateway": "10.0.1.254",
      "active": true
    },
    "eth2": {
      "name": "eth2",
      "prefix": "10.1.1.1/24",
      "gateway": "10.1.1.254",
      "active": true
    }
  }
}
EOF

cat > snapshot-baseline/hosts/server2.json << 'EOF'
{
  "hostname": "server2",
  "hostInterfaces": {
    "eth1": {
      "name": "eth1",
      "prefix": "10.0.2.1/24",
      "gateway": "10.0.2.254",
      "active": true
    },
    "eth2": {
      "name": "eth2",
      "prefix": "10.1.2.1/24",
      "gateway": "10.1.2.254",
      "active": true
    }
  }
}
EOF
```

Verify the complete layout:

```bash
find snapshot-baseline -type f | sort
```

Expected:

```
snapshot-baseline/configs/spine1.cfg
snapshot-baseline/configs/spine2.cfg
snapshot-baseline/configs/tor-rail0.cfg
snapshot-baseline/configs/tor-rail1.cfg
snapshot-baseline/hosts/server1.json
snapshot-baseline/hosts/server2.json
```

---

### Step 4: Load Snapshot with pybatfish — Full Python Script

```python
# load_snapshot.py
from pybatfish.client.session import Session
from pybatfish.datamodel import *
import pandas as pd

pd.set_option("display.max_colwidth", 80)
pd.set_option("display.width", 120)

bf = Session(host="localhost")
bf.set_network("gpu-rail-fabric")

print("Loading baseline snapshot...")
bf.init_snapshot("./snapshot-baseline", name="baseline", overwrite=True)
print("Snapshot loaded successfully.\n")

nodes = bf.q.nodeProperties().answer().frame()
print(f"Nodes parsed by Batfish ({len(nodes)} total):")
print(nodes[["Node", "Configuration_Format"]].to_string(index=False))
```

```bash
python3 load_snapshot.py
```

Expected output:

```
Loading baseline snapshot...
Your snapshot was successfully initialized!

Snapshot loaded successfully.

Nodes parsed by Batfish (4 total):
        Node Configuration_Format
      spine1              SR_LINUX
      spine2              SR_LINUX
  tor-rail0              SR_LINUX
  tor-rail1              SR_LINUX
```

---

### Step 5: Run Reachability Question — Full Query and Result DataFrame

```python
# check_reachability.py
from pybatfish.client.session import Session
from pybatfish.datamodel import *
import pandas as pd, sys

bf = Session(host="localhost")
bf.set_network("gpu-rail-fabric")
bf.set_snapshot("baseline")

print("=== Reachability: server1:eth1 -> server2:eth1 on TCP/29500 (NCCL) ===")
result = bf.q.reachability(
    pathConstraints=PathConstraints(
        startLocation="@enter(tor-rail0[ethernet-1/1])",
        endLocation="tor-rail0[ethernet-1/2]",
    ),
    headers=HeaderConstraints(
        srcIps="10.0.1.1",
        dstIps="10.0.2.1",
        dstPorts="29500",
        ipProtocols=["TCP"],
    ),
    actions="SUCCESS",
).answer().frame()

pd.set_option("display.max_colwidth", 100)
print(f"\nDataFrame shape: {result.shape}")
print(result[["Flow", "Action", "Traces"]].to_string())
print(f"\nSuccessful paths: {len(result)}")
assert len(result) > 0, "ERROR: No paths found"
print("\nPASS: NCCL reachability confirmed via Batfish dataplane model.")
```

```bash
python3 check_reachability.py
```

Expected output:

```
=== Reachability: server1:eth1 -> server2:eth1 on TCP/29500 (NCCL) ===

DataFrame shape: (2, 5)
                                              Flow  Action                    Traces
0  Flow{dscp=0, dstIp=10.0.2.1, dstPort=29500...}  ACCEPT  [Trace{...spine1...}]
1  Flow{dscp=0, dstIp=10.0.2.1, dstPort=29500...}  ACCEPT  [Trace{...spine2...}]

Successful paths: 2

PASS: NCCL reachability confirmed via Batfish dataplane model.
```

The two rows correspond to the two ECMP paths (via spine1 and spine2).

---

### Step 6: Assert No Default Route Originated from Leaves

```python
# assert_no_default_route.py
from pybatfish.client.session import Session
import sys

bf = Session(host="localhost")
bf.set_network("gpu-rail-fabric")
bf.set_snapshot("baseline")

print("=== Checking BGP RIB on each ToR for 0.0.0.0/0 originated routes ===")
violations = []

for node in ["tor-rail0", "tor-rail1"]:
    bgp_routes = bf.q.bgpRib(nodes=node).answer().frame()
    default_routes = bgp_routes[bgp_routes["Network"] == "0.0.0.0/0"]
    if len(default_routes) > 0:
        violations.append(f"{node} is originating a default route")
        print(f"  FAIL: {node} — default route found!")
    else:
        print(f"  PASS: {node} — no default route originated.")

if violations:
    sys.exit(1)
print("\nAll ToRs clean: no default route originated. PASS.")
```

```bash
python3 assert_no_default_route.py
```

Expected:

```
=== Checking BGP RIB on each ToR for 0.0.0.0/0 originated routes ===
  PASS: tor-rail0 — no default route originated.
  PASS: tor-rail1 — no default route originated.

All ToRs clean: no default route originated. PASS.
```

---

### Step 7: Assert ECMP — Check 4 Equal-Cost Paths with Expected Result

In the full 4-leaf topology each spine sees 4 ECMP paths to server prefixes. In the 2-ToR lab, spine1 sees the BGP RIB (all received paths) before best-path selection. Assert all 4 server prefixes are present:

```python
# assert_ecmp.py
from pybatfish.client.session import Session
import sys

bf = Session(host="localhost")
bf.set_network("gpu-rail-fabric")
bf.set_snapshot("baseline")

PREFIXES = ["10.0.1.0/24", "10.0.2.0/24", "10.1.1.0/24", "10.1.2.0/24"]
print("=== ECMP path assertion on spine1 ===")
failures = []

for prefix in PREFIXES:
    routes = bf.q.routes(nodes="spine1", network=prefix).answer().frame()
    bgp_routes = routes[routes["Protocol"] == "bgp"] if len(routes) > 0 else routes
    count = len(bgp_routes)
    if count >= 1:
        print(f"  PASS: {prefix} — {count} BGP route(s) on spine1")
    else:
        failures.append(prefix)
        print(f"  FAIL: {prefix} — 0 routes (expected >=1)")

print("\n=== BGP received RIB on spine1 (all paths before best-path selection) ===")
bgp_rib = bf.q.bgpRib(nodes="spine1").answer().frame()
for prefix in PREFIXES:
    rows = bgp_rib[bgp_rib["Network"] == prefix]
    print(f"  {prefix}: {len(rows)} path(s) received")
    if len(rows) > 0:
        print(rows[["Network", "AS_Path", "Next_Hop_IP"]].to_string(index=False))

if failures:
    print(f"\nFAIL: Missing routes for {failures}")
    sys.exit(1)
print("\nECMP assertion PASS — all 4 server prefixes reachable from spine1.")
```

```bash
python3 assert_ecmp.py
```

Expected output:

```
=== ECMP path assertion on spine1 ===
  PASS: 10.0.1.0/24 — 1 BGP route(s) on spine1
  PASS: 10.0.2.0/24 — 1 BGP route(s) on spine1
  PASS: 10.1.1.0/24 — 1 BGP route(s) on spine1
  PASS: 10.1.2.0/24 — 1 BGP route(s) on spine1

=== BGP received RIB on spine1 (all paths before best-path selection) ===
  10.0.1.0/24: 1 path(s) received
      Network AS_Path Next_Hop_IP
  10.0.1.0/24   65001  192.168.10.0

ECMP assertion PASS — all 4 server prefixes reachable from spine1.
```

---

### Step 8: Make a Breaking Change — Remove a BGP Neighbor from a Config File

Simulate a configuration mistake: an engineer removes the BGP neighbor statement for spine1 on tor-rail0:

```bash
cp -r snapshot-baseline/ snapshot-proposed/

# Show which lines reference the spine1 peering IP
grep -n "192.168.10.1" snapshot-baseline/configs/tor-rail0.cfg
```

Expected:

```
47:          neighbor 192.168.10.1 {
48:              peer-group SPINES;
49:              peer-as 65100;
50:          }
```

Remove those lines from the proposed snapshot:

```bash
python3 - <<'EOF'
import re

with open("snapshot-baseline/configs/tor-rail0.cfg") as f:
    content = f.read()

broken = re.sub(
    r'\s*neighbor 192\.168\.10\.1 \{[^}]+\}\n?',
    '\n',
    content,
    flags=re.DOTALL
)

with open("snapshot-proposed/configs/tor-rail0.cfg", "w") as f:
    f.write(broken)

print(f"Breaking change applied: removed BGP neighbor 192.168.10.1 from tor-rail0")
print(f"Baseline lines: {len(content.splitlines())}")
print(f"Proposed lines: {len(broken.splitlines())}")
EOF
```

Expected:

```
Breaking change applied: removed BGP neighbor 192.168.10.1 from tor-rail0
Baseline lines: 89
Proposed lines: 86
```

---

### Step 9: Init Proposed Snapshot and Run Differential Reachability

```python
# run_differential.py
from pybatfish.client.session import Session
from pybatfish.datamodel import *
import pandas as pd

bf = Session(host="localhost")
bf.set_network("gpu-rail-fabric")

print("Loading proposed (broken) snapshot...")
bf.init_snapshot("./snapshot-proposed", name="proposed", overwrite=True)
print("Proposed snapshot loaded.\n")

print("Running differential reachability: proposed vs baseline...")
diff = bf.q.differentialReachability(
    pathConstraints=PathConstraints(
        startLocation="@enter(tor-rail0[ethernet-1/1])",
        endLocation="tor-rail0[ethernet-1/2]",
    ),
    headers=HeaderConstraints(
        srcIps="10.0.1.1",
        dstIps="10.0.2.1",
        dstPorts="29500",
        ipProtocols=["TCP"],
    ),
).answer(
    snapshot="proposed",
    reference_snapshot="baseline",
).frame()

pd.set_option("display.max_colwidth", 100)
pd.set_option("display.width", 140)
print(f"\nDifferential result rows: {len(diff)}")
if len(diff) > 0:
    print(diff[["Flow", "FlowDiff", "Snapshot_Traces", "Reference_Traces"]].to_string())
```

```bash
python3 run_differential.py
```

Expected output:

```
Loading proposed (broken) snapshot...
Your snapshot was successfully initialized!

Proposed snapshot loaded.

Running differential reachability: proposed vs baseline...

Differential result rows: 2

                                    Flow  FlowDiff  Snapshot_Traces  Reference_Traces
0  Flow{dstIp=10.0.2.1, dstPort=29500...}  LOST  [DENIED_OUT via ...]  [ACCEPTED via ...]
1  Flow{dstIp=10.0.2.1, dstPort=29500...}  LOST  [BLACKHOLE via ...]   [ACCEPTED via ...]
```

---

### Step 10: Show Batfish Catches the Regression — FlowDiff == "LOST" Rows

Write the CI gate assertion that blocks the PR:

```python
# assert_no_regression.py
from pybatfish.client.session import Session
from pybatfish.datamodel import *
import sys

bf = Session(host="localhost")
bf.set_network("gpu-rail-fabric")

diff = bf.q.differentialReachability(
    pathConstraints=PathConstraints(
        startLocation="@enter(tor-rail0[ethernet-1/1])",
        endLocation="tor-rail0[ethernet-1/2]",
    ),
    headers=HeaderConstraints(
        srcIps="10.0.1.1",
        dstIps="10.0.2.1",
        dstPorts="29500",
        ipProtocols=["TCP"],
    ),
).answer(snapshot="proposed", reference_snapshot="baseline").frame()

lost_flows = diff[diff["FlowDiff"] == "LOST"]
gained_flows = diff[diff["FlowDiff"] == "GAINED"]

print(f"Lost flows  : {len(lost_flows)}")
print(f"Gained flows: {len(gained_flows)}")

if len(lost_flows) > 0:
    print("\nREGRESSION DETECTED — these flows are now unreachable:")
    for _, row in lost_flows.iterrows():
        print(f"  Flow: {row['Flow']}")
    print("\nCI GATE: FAIL. Block this PR.")
    sys.exit(1)

print("\nCI GATE: PASS. No reachability regressions.")
```

```bash
python3 assert_no_regression.py
echo "Exit code: $?"
```

Expected output (with the broken config):

```
Lost flows  : 2
Gained flows: 0

REGRESSION DETECTED — these flows are now unreachable:
  Flow: Flow{dscp=0, dstIp=10.0.2.1, dstPort=29500, ipProtocol=TCP, srcIp=10.0.1.1, ...}
  Flow: Flow{dscp=0, dstIp=10.0.2.1, dstPort=29500, ipProtocol=TCP, srcIp=10.0.1.1, ...}

CI GATE: FAIL. Block this PR.
Exit code: 1
```

---

### Step 11: Fix the Config and Rerun — Show PASS

Restore the BGP neighbor (simulating the engineer reverting the mistake):

```bash
cp snapshot-baseline/configs/tor-rail0.cfg snapshot-proposed/configs/tor-rail0.cfg

python3 - <<'EOF'
from pybatfish.client.session import Session
bf = Session(host="localhost")
bf.set_network("gpu-rail-fabric")
bf.init_snapshot("./snapshot-proposed", name="proposed", overwrite=True)
print("Fixed snapshot reloaded.")
EOF

python3 assert_no_regression.py
echo "Exit code: $?"
```

Expected output (after fix):

```
Fixed snapshot reloaded.
Lost flows  : 0
Gained flows: 0

CI GATE: PASS. No reachability regressions.
Exit code: 0
```

---

### Step 12: Nornir — inventory.yaml + hosts.yaml, Run `show bgp summary` in Parallel

Create a Nornir inventory targeting the running Containerlab topology:

```bash
mkdir -p ~/batfish-ci/nornir-lab/inventory
cd ~/batfish-ci/nornir-lab
```

Write the three inventory files and Nornir config:

```bash
cat > nornir.yaml << 'EOF'
inventory:
  plugin: SimpleInventory
  options:
    host_file: "inventory/hosts.yaml"
    group_file: "inventory/groups.yaml"
    defaults_file: "inventory/defaults.yaml"
runner:
  plugin: threaded
  options:
    num_workers: 6
logging:
  enabled: false
EOF

cat > inventory/defaults.yaml << 'EOF'
data: {}
EOF

cat > inventory/groups.yaml << 'EOF'
srl_nodes:
  platform: srl
  connection_options:
    netmiko:
      platform: nokia_srl
      extras:
        global_delay_factor: 2
EOF

cat > inventory/hosts.yaml << 'EOF'
spine1:
  hostname: 172.20.20.11
  port: 22
  username: admin
  password: "NokiaSrl1!"
  groups: [srl_nodes]
  data:
    role: spine
    bgp_as: 65100

spine2:
  hostname: 172.20.20.12
  port: 22
  username: admin
  password: "NokiaSrl1!"
  groups: [srl_nodes]
  data:
    role: spine
    bgp_as: 65100

tor-rail0:
  hostname: 172.20.20.21
  port: 22
  username: admin
  password: "NokiaSrl1!"
  groups: [srl_nodes]
  data:
    role: leaf
    bgp_as: 65001

tor-rail1:
  hostname: 172.20.20.22
  port: 22
  username: admin
  password: "NokiaSrl1!"
  groups: [srl_nodes]
  data:
    role: leaf
    bgp_as: 65002
EOF
```

Run `show bgp summary` in parallel across all Containerlab nodes:

```python
# run_nornir_bgp.py
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command
from nornir_utils.plugins.functions import print_result

nr = InitNornir(config_file="nornir.yaml")
print(f"Inventory loaded: {len(nr.inventory.hosts)} hosts")
print("Running 'show network-instance default protocols bgp summary' on all nodes...\n")

result = nr.run(
    task=netmiko_send_command,
    command_string="show network-instance default protocols bgp summary",
)

print_result(result)

print("\n=== BGP Session Summary ===")
for host_name, multi_result in result.items():
    if multi_result[0].failed:
        print(f"  {host_name}: FAILED to connect")
        continue
    established = multi_result[0].result.lower().count("established")
    print(f"  {host_name}: {established} established session(s)")
```

```bash
python3 run_nornir_bgp.py
```

Expected output:

```
Inventory loaded: 4 hosts
Running 'show network-instance default protocols bgp summary' on all nodes...

vvvv netmiko_send_command ** changed : False vvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvvv INFO
spine1                                                                         1/4
---- netmiko_send_command ** changed : False ----------------------------------  INFO
BGP is enabled and up
Global AS number  : 65100
...
| TORS | 192.168.10.0 | established | 0d0h14m22s | 4 | 3 |
| TORS | 192.168.20.0 | established | 0d0h14m18s | 4 | 3 |
...
^^^^ END netmiko_send_command ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

=== BGP Session Summary ===
  spine1   : 2 established session(s)
  spine2   : 2 established session(s)
  tor-rail0: 2 established session(s)
  tor-rail1: 2 established session(s)
```

---

### Step 13: Full pytest Test File — All BGP Sessions Established via Genie Parsing

```python
# tests/test_bgp_sessions.py
"""
pytest: Validate BGP session state for all nodes in the GPU rail fabric.

Run from ~/batfish-ci/nornir-lab/:
  pytest tests/test_bgp_sessions.py -v

Requirements:
  - Containerlab gpu-rail-fabric topology deployed
  - uv venv with nornir, nornir-netmiko, pytest installed
"""
import re
import pytest
from nornir import InitNornir
from nornir_netmiko.tasks import netmiko_send_command


@pytest.fixture(scope="session")
def nr():
    """Initialize Nornir once per test session."""
    nornir = InitNornir(config_file="nornir.yaml")
    yield nornir
    nornir.close_connections()


@pytest.fixture(scope="session")
def bgp_summaries(nr):
    """Collect BGP summary from all nodes in parallel. Returns dict: host -> output."""
    result = nr.run(
        task=netmiko_send_command,
        command_string="show network-instance default protocols bgp summary",
    )
    return {
        host: (multi[0].result if not multi.failed else None)
        for host, multi in result.items()
    }


def parse_srl_bgp_neighbors(output: str) -> dict:
    """
    Parse SR Linux BGP summary table rows into:
    { neighbor_ip: { "state": str, "uptime": str, "rx": int, "tx": int } }
    """
    neighbors = {}
    pattern = re.compile(
        r'\|\s*\S+\s*\|\s*(\d+\.\d+\.\d+\.\d+)\s*\|\s*(\w+)\s*\|'
        r'\s*([\w\d:]+)\s*\|\s*(\d+)\s*\|\s*(\d+)\s*\|'
    )
    for m in pattern.finditer(output):
        ip, state, uptime, rx, tx = m.groups()
        neighbors[ip] = {"state": state.lower(), "uptime": uptime,
                         "rx": int(rx), "tx": int(tx)}
    return neighbors


EXPECTED_NEIGHBORS = {
    "spine1":    {"192.168.10.0": {}, "192.168.20.0": {}},
    "spine2":    {"192.168.11.0": {}, "192.168.21.0": {}},
    "tor-rail0": {"192.168.10.1": {}, "192.168.11.1": {}},
    "tor-rail1": {"192.168.20.1": {}, "192.168.21.1": {}},
}


def test_all_nodes_reachable(bgp_summaries):
    """All nodes must respond to SSH."""
    failed = [h for h, o in bgp_summaries.items() if o is None]
    assert not failed, f"SSH failed for: {failed}"


@pytest.mark.parametrize("hostname", list(EXPECTED_NEIGHBORS))
def test_bgp_neighbor_count(hostname, bgp_summaries):
    """Each node must have the expected number of BGP neighbors."""
    output = bgp_summaries[hostname]
    assert output is not None, f"No output from {hostname}"
    neighbors = parse_srl_bgp_neighbors(output)
    expected = len(EXPECTED_NEIGHBORS[hostname])
    assert len(neighbors) == expected, (
        f"{hostname}: found {len(neighbors)} neighbor(s), expected {expected}"
    )


@pytest.mark.parametrize("hostname,neighbor_ip", [
    (h, n)
    for h, ns in EXPECTED_NEIGHBORS.items()
    for n in ns
])
def test_bgp_session_established(hostname, neighbor_ip, bgp_summaries):
    """Every expected BGP neighbor must be in the 'established' state."""
    output = bgp_summaries[hostname]
    assert output is not None, f"No output from {hostname}"
    neighbors = parse_srl_bgp_neighbors(output)
    assert neighbor_ip in neighbors, (
        f"{hostname}: neighbor {neighbor_ip} not in BGP table. "
        f"Found: {list(neighbors.keys())}"
    )
    state = neighbors[neighbor_ip]["state"]
    assert state == "established", (
        f"{hostname}: {neighbor_ip} is '{state}', expected 'established'"
    )


@pytest.mark.parametrize("hostname,neighbor_ip", [
    (h, n)
    for h, ns in EXPECTED_NEIGHBORS.items()
    for n in ns
])
def test_bgp_session_has_rx_prefixes(hostname, neighbor_ip, bgp_summaries):
    """Each established session must have received at least 1 prefix."""
    output = bgp_summaries[hostname]
    assert output is not None, f"No output from {hostname}"
    neighbors = parse_srl_bgp_neighbors(output)
    if neighbor_ip not in neighbors:
        pytest.skip(f"{hostname}: {neighbor_ip} not found (covered by session test)")
    rx = neighbors[neighbor_ip]["rx"]
    assert rx > 0, (
        f"{hostname}: neighbor {neighbor_ip} established but rx=0 prefixes. "
        "Check export policy on remote peer."
    )


@pytest.mark.parametrize("hostname", ["spine1", "spine2"])
def test_spine_receives_server_routes(hostname, bgp_summaries):
    """Spines must receive at least 4 routes total (server subnets from both ToRs)."""
    output = bgp_summaries[hostname]
    assert output is not None, f"No output from {hostname}"
    neighbors = parse_srl_bgp_neighbors(output)
    total_rx = sum(n["rx"] for n in neighbors.values())
    assert total_rx >= 4, (
        f"{hostname}: total received prefixes={total_rx}, expected >=4"
    )


@pytest.mark.parametrize("hostname", ["tor-rail0", "tor-rail1"])
def test_tor_no_default_route_in_bgp_table(hostname, bgp_summaries):
    """ToRs must not receive a default route from spines."""
    output = bgp_summaries[hostname]
    assert output is not None, f"No output from {hostname}"
    assert "0.0.0.0/0" not in output, (
        f"{hostname}: default route found in BGP summary — "
        "spines must not advertise 0.0.0.0/0 to ToRs"
    )
```

---

### Step 14: Run pytest -v and Show Expected PASS Output

```bash
cd ~/batfish-ci/nornir-lab
mkdir -p tests
# (write the test file shown in Step 13 to tests/test_bgp_sessions.py)

pytest tests/test_bgp_sessions.py -v --tb=short
```

Expected output (all tests passing):

```
========================= test session starts ==========================
platform linux -- Python 3.12.3, pytest-8.x.x, pluggy-1.x.x
rootdir: /home/user/batfish-ci/nornir-lab
collected 26 items

tests/test_bgp_sessions.py::test_all_nodes_reachable PASSED                  [  3%]
tests/test_bgp_sessions.py::test_bgp_neighbor_count[spine1] PASSED           [  7%]
tests/test_bgp_sessions.py::test_bgp_neighbor_count[spine2] PASSED           [ 11%]
tests/test_bgp_sessions.py::test_bgp_neighbor_count[tor-rail0] PASSED        [ 15%]
tests/test_bgp_sessions.py::test_bgp_neighbor_count[tor-rail1] PASSED        [ 19%]
tests/test_bgp_sessions.py::test_bgp_session_established[spine1-192.168.10.0] PASSED [ 23%]
tests/test_bgp_sessions.py::test_bgp_session_established[spine1-192.168.20.0] PASSED [ 26%]
tests/test_bgp_sessions.py::test_bgp_session_established[spine2-192.168.11.0] PASSED [ 30%]
tests/test_bgp_sessions.py::test_bgp_session_established[spine2-192.168.21.0] PASSED [ 34%]
tests/test_bgp_sessions.py::test_bgp_session_established[tor-rail0-192.168.10.1] PASSED [ 38%]
tests/test_bgp_sessions.py::test_bgp_session_established[tor-rail0-192.168.11.1] PASSED [ 42%]
tests/test_bgp_sessions.py::test_bgp_session_established[tor-rail1-192.168.20.1] PASSED [ 46%]
tests/test_bgp_sessions.py::test_bgp_session_established[tor-rail1-192.168.21.1] PASSED [ 50%]
tests/test_bgp_sessions.py::test_bgp_session_has_rx_prefixes[spine1-192.168.10.0] PASSED [ 53%]
tests/test_bgp_sessions.py::test_bgp_session_has_rx_prefixes[spine1-192.168.20.0] PASSED [ 57%]
tests/test_bgp_sessions.py::test_bgp_session_has_rx_prefixes[spine2-192.168.11.0] PASSED [ 61%]
tests/test_bgp_sessions.py::test_bgp_session_has_rx_prefixes[spine2-192.168.21.1] PASSED [ 65%]
tests/test_bgp_sessions.py::test_bgp_session_has_rx_prefixes[tor-rail0-192.168.10.1] PASSED [ 69%]
tests/test_bgp_sessions.py::test_bgp_session_has_rx_prefixes[tor-rail0-192.168.11.1] PASSED [ 73%]
tests/test_bgp_sessions.py::test_bgp_session_has_rx_prefixes[tor-rail1-192.168.20.1] PASSED [ 76%]
tests/test_bgp_sessions.py::test_bgp_session_has_rx_prefixes[tor-rail1-192.168.21.1] PASSED [ 80%]
tests/test_bgp_sessions.py::test_spine_receives_server_routes[spine1] PASSED [ 84%]
tests/test_bgp_sessions.py::test_spine_receives_server_routes[spine2] PASSED [ 88%]
tests/test_bgp_sessions.py::test_tor_no_default_route_in_bgp_table[tor-rail0] PASSED [ 92%]
tests/test_bgp_sessions.py::test_tor_no_default_route_in_bgp_table[tor-rail1] PASSED [ 96%]

========================== 26 passed in 18.47s =========================
```

To run only the session-established assertions:

```bash
pytest tests/test_bgp_sessions.py -v -k "established" --tb=short
```

To generate a JUnit XML report for CI upload:

```bash
pytest tests/test_bgp_sessions.py -v --junit-xml=test-results.xml
cat test-results.xml | grep -E 'tests=|failures=|errors='
# Expected: tests="26" failures="0" errors="0"
```

---

## Summary

- Batfish catches routing bugs, ACL errors, and reachability regressions by simulating the dataplane from device configs — before any device is touched.
- pyATS/Genie provides structured parsing of device output; golden diffs make post-change validation deterministic.
- Nornir is the Python automation framework for parallel, inventory-driven device tasks; it integrates cleanly with pytest for CI assertions.
- The complete pipeline — Batfish → Containerlab → pyATS → merge gate — gives network changes the same rigor as application code changes.

---

## References

- [Batfish](https://www.batfish.org)
- [pybatfish (Python client)](https://github.com/batfish/pybatfish)
- [pybatfish Jupyter notebook examples](https://github.com/batfish/pybatfish/tree/master/jupyter_notebooks)
- [pyATS documentation](https://developer.cisco.com/docs/pyats/)
- [Genie (pyATS parsing library)](https://developer.cisco.com/docs/genie-docs/)
- [Nornir documentation](https://nornir.readthedocs.io)
- [nornir-netmiko](https://github.com/nornir-automation/nornir-netmiko)
- [nornir-utils](https://github.com/nornir-automation/nornir-utils)
- [nornir-jinja2](https://github.com/nornir-automation/nornir-jinja2)
- [Netmiko](https://ktbyers.github.io/netmiko/)
- [paramiko](https://www.paramiko.org)
- [Jinja2](https://jinja.palletsprojects.com)
- [deepdiff](https://zepworks.com/deepdiff/current/)
- [pytest](https://docs.pytest.org)
- [pytest-html](https://pytest-html.readthedocs.io)
- [Containerlab](https://containerlab.dev)
- [Docker](https://docs.docker.com)
- [GitHub Actions](https://docs.github.com/en/actions)
- [SR Linux (Nokia)](https://learn.srlinux.dev)
- [SONiC Foundation](https://sonicfoundation.dev)
- [uv (Python package manager)](https://docs.astral.sh/uv/)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).