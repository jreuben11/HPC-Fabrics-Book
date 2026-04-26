# Chapter 21 — Lab Environments with Containerlab

**Part VII: Testing, Emulation & Simulation** | ~20 pages

---

## Introduction

Chapter 21 introduces **Containerlab**, the infrastructure-as-code tool that turns a network topology YAML file into a fully operational virtual lab running inside **Docker** containers. By the end of this chapter you will have deployed a GPU rail fabric with two spine switches, two rail-segregated top-of-rack switches, and dual-homed server containers — all wired with **BGP** eBGP sessions and verified end-to-end with **iperf3** — and packaged that topology as a **GitHub Actions** CI job that runs on every pull request.

The motivation is practical and urgent. AI cluster networking is complex enough that a mis-configured ACL, a **BGP** policy bug, or a mis-wired ECMP path can silently degrade **AllReduce** throughput by 10× without triggering any obvious alarm. The only way to catch these problems early is to test them in an environment that mirrors production as closely as possible, as early as possible — ideally before any config change merges. **Containerlab** makes that possible on a laptop.

Chapter 20 described the fabric demands of distributed AI training: rail-optimized topology, lossless **RDMA**, sub-microsecond jitter budgets. This chapter builds the emulation environment in which those properties can be tested without production hardware. Chapter 22 extends that environment into a full CI/CD pipeline with static analysis and golden-diff validation; Chapter 23 covers simulation tools for the cases where emulation is insufficient.

**Containerlab** achieves its simplicity by treating standard **Docker** containers as network nodes and Linux veth pairs as links. Each NOS image — Nokia **SR Linux**, **SONiC-VS**, **FRR** — runs in an unmodified **Docker** container; **Containerlab** wires the containers together at the network-namespace level and injects startup configuration files. The result is a reproducible, version-controlled, diff-reviewable network topology that boots in under two minutes and is torn down with a single command.

The chapter builds from first principles: Section 21.1 establishes why infrastructure-as-code is necessary, Sections 21.2–21.5 dissect the YAML topology format and CLI, Sections 21.6–21.7 cover VM-based nodes and CI integration, and the Lab Walkthrough executes the full GPU rail fabric end-to-end with annotated expected output at every step.

---

## Installation

**Docker** provides the container runtime that **Containerlab** uses to instantiate each NOS node; without it, no topology can be deployed. **Containerlab** itself is installed via its official installer script and orchestrates **Docker** containers, virtual Ethernet pairs, and startup-config injection to build multi-NOS lab topologies from a single YAML file. The **SR Linux**, **SONiC-VS**, and **FRR** images are pulled from public registries and serve as the spine, leaf, and host NOS images throughout the lab walkthroughs and CI pipeline. The optional **vrnetlab** package is required only when a topology includes VM-based commercial NOSes such as **IOS-XE** or **vMX** that cannot run as unmodified **Docker** containers.

All commands assume Ubuntu 24.04 LTS on x86-64. Run as a non-root user with `sudo` access.

### Docker

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
# Verify
docker info | grep "Server Version"
# Expected: Server Version: 24.x.x (or later)
```

### Containerlab

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
# Verify
containerlab version
# Expected output:
#                            _       _     _
#                  _        (_)     | |   | |
#  ____ ___  ____ | |_  ____ _ ____| | __| |
# / ___) _ \|  _ \|  _)/ _  | |  __) |/ _  |
#( (__| |_| | | | | |_( ( | | | |  | ( ( | |
# \____)___/|_| |_|\___)_||_|_|_|  |_|\_|| |
#                                     (_____|
#  version: 0.57.x
#  commit: xxxxxxx
#  date: 2024-xx-xx
#  KernelVersion: 6.8.0-107-generic
#  ...
```

### gnmic (gNMI client for telemetry verification)

```bash
bash -c "$(curl -sL https://get.gnmic.openconfig.net)"
# Verify
gnmic version
# Expected: gnmic version 0.38.x
```

### iperf3

```bash
sudo apt install -y iperf3
iperf3 --version
# Expected: iperf 3.x.x (cJSON 1.x.x)
```

### Python environment (netmiko, paramiko, pyyaml)

```bash
# Install uv if not present
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

uv venv .venv && source .venv/bin/activate
uv pip install netmiko paramiko pyyaml
# Verify
python -c "import netmiko, paramiko, yaml; print('OK')"
# Expected: OK
```

---

## 21.1 Infrastructure-as-Code for Network Labs

The traditional approach to network lab work is to configure a topology by hand in a GUI (GNS3, EVE-NG), save it in a proprietary format, and share it as a file attachment. GNS3 (Graphical Network Simulator-3) and EVE-NG (Emulated Virtual Environment Next Generation) are popular GUI-driven network emulation platforms that let operators drag-and-drop virtual routers and switches into a canvas topology, but they store their topology state in opaque files rather than text that can be version-controlled or executed in CI. This is inherently unscalable: the topology is not version-controlled, changes are hard to review, and CI integration is nearly impossible.

Containerlab treats network topology as code. A YAML file describes nodes and links; `containerlab deploy` spins up the topology in seconds using Docker. The topology file can be committed to git, reviewed as a diff, and executed in a CI pipeline on every PR.

---

## 21.2 Containerlab Architecture

```
containerlab deploy -t topology.yml
        │
   Parse YAML topology
        │
   For each node:
     docker pull <image>
     docker run --network none ...   (no default networking)
     Create veth pairs for links
     Attach veth to container netns
   Configure management IP
   Execute startup-config injection
```

Each container gets a management interface (connected to a Linux bridge for OOB access) plus data-plane interfaces defined by the topology links.

---

## 21.3 Topology YAML — Anatomy

```yaml
# topology.yml
name: ai-cluster-fabric

mgmt:
  network: mgmt-net
  ipv4-subnet: 172.100.100.0/24

topology:
  defaults:
    env:
      CLAB_MGMT_VRF: mgmt   # SR Linux management VRF

  kinds:
    srl:                     # SR Linux defaults
      image: ghcr.io/nokia/srlinux:24.3.2
    sonic-vs:
      image: sonicdev/sonic-vs:20231218
    linux:
      image: ghcr.io/hellt/network-multitool:latest

  nodes:
    # Spine switches (SR Linux)
    spine1:
      kind: srl
      startup-config: configs/spine1.cfg.yml
    spine2:
      kind: srl
      startup-config: configs/spine2.cfg.yml

    # Leaf switches (SONiC-VS)
    leaf1:
      kind: sonic-vs
      startup-config: configs/leaf1.json
    leaf2:
      kind: sonic-vs
      startup-config: configs/leaf2.json
    leaf3:
      kind: sonic-vs
      startup-config: configs/leaf3.json
    leaf4:
      kind: sonic-vs
      startup-config: configs/leaf4.json

    # Host containers
    host1:
      kind: linux
      exec:
        - ip addr add 10.1.1.1/24 dev eth1
        - ip route add 10.0.0.0/8 via 10.1.1.254
    host2:
      kind: linux
      exec:
        - ip addr add 10.2.1.1/24 dev eth1
        - ip route add 10.0.0.0/8 via 10.2.1.254

  links:
    # Spine-leaf connections (full mesh)
    - endpoints: [spine1:e1-1, leaf1:e1-49]
    - endpoints: [spine1:e1-2, leaf2:e1-49]
    - endpoints: [spine1:e1-3, leaf3:e1-49]
    - endpoints: [spine1:e1-4, leaf4:e1-49]
    - endpoints: [spine2:e1-1, leaf1:e1-50]
    - endpoints: [spine2:e1-2, leaf2:e1-50]
    - endpoints: [spine2:e1-3, leaf3:e1-50]
    - endpoints: [spine2:e1-4, leaf4:e1-50]
    # Host-leaf connections
    - endpoints: [host1:eth1, leaf1:e1-1]
    - endpoints: [host2:eth1, leaf3:e1-1]
```

---

## 21.4 Node Kinds

### SR Linux

```yaml
# SR Linux startup config (YAML format)
# configs/spine1.cfg.yml
interface ethernet-1/1:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 192.168.10.1/31:

network-instance default:
  protocols:
    bgp:
      router-id: 10.0.0.1
      autonomous-system: 65100
      group LEAVES:
        export-policy: export-loopbacks
        import-policy: import-all
      neighbor 192.168.10.0:
        peer-group: LEAVES
        peer-as: 65001
```

### SONiC-VS

```json
// configs/leaf1.json (SONiC CONFIG_DB format)
{
  "DEVICE_METADATA": {
    "localhost": {
      "hostname": "leaf1",
      "bgp_asn": "65001"
    }
  },
  "INTERFACE": {
    "Ethernet48|192.168.10.0/31": {},
    "Ethernet49|192.168.10.2/31": {}
  },
  "BGP_NEIGHBOR": {
    "192.168.10.1": {
      "rrclient": "0",
      "name": "spine1",
      "local_addr": "192.168.10.0",
      "nhopself": "0",
      "holdtime": "10",
      "asn": "65100",
      "keepalive": "3"
    }
  }
}
```

### FRR Container as Host Router

FRR (Free Range Routing) is an open-source IP routing suite that implements BGP, OSPF, ISIS, and other protocols on Linux; it is the successor to Quagga and is widely used in cloud and HPC environments. In Containerlab, an FRR container deployed as `kind: linux` acts as a fully-capable software router that peers with switch nodes via standard BGP sessions.

```yaml
nodes:
  router1:
    kind: linux
    image: frrouting/frr:v9.1.0
    exec:
      - /usr/lib/frr/frrinit.sh start
    binds:
      - configs/frr1/daemons:/etc/frr/daemons
      - configs/frr1/frr.conf:/etc/frr/frr.conf
```

### Packet Generator (iperf3)

```yaml
nodes:
  pktgen:
    kind: linux
    image: networkstatic/iperf3
    exec:
      - iperf3 -s -D   # start iperf3 server in background
```

---

## 21.5 CLI Workflow

```bash
# Deploy topology
containerlab deploy -t topology.yml

# List running topologies
containerlab inspect --all

# Connect to a node
docker exec -it clab-ai-cluster-fabric-spine1 sr_cli
# or
ssh admin@172.100.100.2   # via management IP

# Destroy topology
containerlab destroy -t topology.yml

# Destroy and remove all Docker images (full cleanup)
containerlab destroy -t topology.yml --cleanup

# Generate a graph visualization
containerlab graph -t topology.yml   # starts HTTP server with interactive graph
```

---

## 21.6 VM-Based Nodes via vrnetlab

For NOS images that require real VMs (IOS-XE, JunOS vMX, vSRX), `vrnetlab` wraps the VM in a container using KVM/QEMU:

```bash
# Build a vrnetlab image for Cisco CSR1000v
cd vrnetlab/csr/
make CSR_IOSXE_IMAGE=csr1000v-universalk9.17.03.02.qcow2

# Use in topology
nodes:
  csr1:
    kind: vr-csr
    image: vrnetlab/vr-csr:17.03.02
```

vrnetlab supports: JunOS vMX, vSRX, vQFX; Cisco IOS-XE, NX-OS; Arista vEOS; Mikrotik RouterOS; and more.

---

## 21.7 CI Integration

Containerlab topologies can run in CI/CD pipelines to validate network configuration changes:

```yaml
# .github/workflows/network-ci.yml
name: Network CI

on: [push, pull_request]

jobs:
  topology-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install containerlab
        run: bash -c "$(curl -sL https://get.containerlab.dev)"

      - name: Deploy topology
        run: containerlab deploy -t tests/topology.yml

      - name: Wait for BGP convergence
        run: |
          sleep 30
          docker exec clab-test-spine1 sr_cli \
              "show network-instance default protocols bgp summary" \
              | grep -c "established" | grep -q "4"

      - name: Run connectivity tests
        run: |
          docker exec clab-test-host1 ping -c 3 10.2.1.1

      - name: Run iperf3 throughput test
        run: |
          docker exec clab-test-host2 iperf3 -s -D
          docker exec clab-test-host1 iperf3 -c 10.2.1.1 -t 10

      - name: Destroy topology
        if: always()
        run: containerlab destroy -t tests/topology.yml
```

---

## 21.8 GPU Cluster Fabric Topology in Containerlab

A practical rail-optimized GPU cluster representation:

```yaml
name: gpu-rail-fabric

topology:
  kinds:
    srl:
      image: ghcr.io/nokia/srlinux:24.3.2

  nodes:
    # Rail 0 ToR (connects GPU0 of each server)
    tor-rail0:
      kind: srl
    # Rail 1 ToR (connects GPU1 of each server)
    tor-rail1:
      kind: srl

    # Spine switches
    spine1:
      kind: srl
    spine2:
      kind: srl

    # "GPU server" containers (each has eth1 on rail0, eth2 on rail1)
    server1:
      kind: linux
      exec:
        - ip addr add 10.0.1.1/24 dev eth1   # GPU0 NIC → rail0
        - ip addr add 10.1.1.1/24 dev eth2   # GPU1 NIC → rail1

  links:
    # Rail 0 connections
    - endpoints: [server1:eth1, tor-rail0:e1-1]
    - endpoints: [server2:eth1, tor-rail0:e1-2]
    # Rail 1 connections
    - endpoints: [server1:eth2, tor-rail1:e1-1]
    - endpoints: [server2:eth2, tor-rail1:e1-2]
    # ToR uplinks to spines
    - endpoints: [tor-rail0:e1-48, spine1:e1-1]
    - endpoints: [tor-rail0:e1-49, spine2:e1-1]
    - endpoints: [tor-rail1:e1-48, spine1:e1-2]
    - endpoints: [tor-rail1:e1-49, spine2:e1-2]
```

---

## Lab Walkthrough 21 — GPU Rail Fabric: Full Step-by-Step

This is the primary lab for Part VII. Every subsequent chapter's CI pipeline builds on this topology. Work through each step in order; expected output is shown so you can verify correctness before proceeding.

### Step 1: Prerequisites Check

Confirm Docker and Containerlab are operational before pulling any images.

```bash
docker info | grep -E "Server Version|Storage Driver|Cgroup"
```

Expected output (values will vary by host):

```
 Server Version: 24.0.7
 Storage Driver: overlay2
 Cgroup Driver: systemd
 Cgroup Version: 2
```

```bash
containerlab version
```

Expected output:

```
                           _       _     _
                 _        (_)     | |   | |
 ____ ___  ____ | |_  ____ _ ____| | __| |
/ ___) _ \|  _ \|  _)/ _  | |  __) |/ _  |
( (__| |_| | | | | |_( ( | | | |  | ( ( | |
 \____)___/|_| |_|\___)_||_|_|_|  |_|\_|| |
                                    (_____|
 version: 0.57.3
 commit: b2e4fc3
 date: 2024-03-15T10:00:00Z
 KernelVersion: 6.8.0-107-generic
 GoVersion: go1.22.1
 OS: linux
 Arch: amd64
```

Also confirm iperf3 is available:

```bash
iperf3 --version
```

Expected:

```
iperf 3.16 (cJSON 1.7.15)
Linux ... 6.8.0-107-generic #107-Ubuntu SMP ...
Optional features available: CPU affinity setting, ...
```

---

### Step 2: Pull Container Images

SR Linux and the network-multitool images are pulled separately so you can see timing clearly. On a fast connection each image takes 1-3 minutes.

```bash
# Pull SR Linux (Nokia) — ~900 MB compressed
time docker pull ghcr.io/nokia/srlinux:24.3.2
```

Expected timing and output:

```
24.3.2: Pulling from nokia/srlinux
a1d0c7532777: Pull complete
...
Digest: sha256:5b...
Status: Downloaded newer image for ghcr.io/nokia/srlinux:24.3.2
ghcr.io/nokia/srlinux:24.3.2

real    2m14.381s
user    0m0.041s
sys     0m0.031s
```

```bash
# Pull host container image — ~55 MB compressed
time docker pull ghcr.io/hellt/network-multitool:latest
```

Expected:

```
latest: Pulling from hellt/network-multitool
...
Status: Downloaded newer image for ghcr.io/hellt/network-multitool:latest

real    0m18.423s
user    0m0.012s
sys     0m0.008s
```

Verify both images are present:

```bash
docker images | grep -E "srlinux|network-multitool"
```

Expected:

```
ghcr.io/nokia/srlinux          24.3.2    c3f2a1d09e88   2 weeks ago    2.31GB
ghcr.io/hellt/network-multitool latest   a7b4c3e21f01   3 months ago   148MB
```

---

### Step 3: Write the Complete GPU Rail Fabric Topology YAML

Create the working directory and write the full topology file. This YAML encodes the entire GPU rail fabric: two spine switches, two rail ToRs, and two server containers — each server simulating a dual-NIC GPU node.

```bash
mkdir -p ~/clab-gpu-rail/configs
cd ~/clab-gpu-rail
```

Write `gpu-rail-fabric.yml`:

```yaml
# gpu-rail-fabric.yml
# Models a 2-server, 2-rail GPU fabric with 2 spine switches.
# Each server has eth1 on rail0 and eth2 on rail1 (dual NIC, one per rail).
# ToRs run eBGP toward spines (AS 65100); each ToR is its own AS.
# ECMP: spine1 and spine2 each install 2 equal-cost paths to each server prefix.

name: gpu-rail-fabric

mgmt:
  network: mgmt-gpu
  ipv4-subnet: 172.20.20.0/24

topology:
  kinds:
    srl:
      image: ghcr.io/nokia/srlinux:24.3.2
    linux:
      image: ghcr.io/hellt/network-multitool:latest

  nodes:
    # ── Spine layer (AS 65100) ──────────────────────────────────────
    spine1:
      kind: srl
      mgmt-ipv4: 172.20.20.11
      startup-config: configs/spine1.yml

    spine2:
      kind: srl
      mgmt-ipv4: 172.20.20.12
      startup-config: configs/spine2.yml

    # ── ToR layer — Rail 0 (AS 65001) ──────────────────────────────
    tor-rail0:
      kind: srl
      mgmt-ipv4: 172.20.20.21
      startup-config: configs/tor-rail0.yml

    # ── ToR layer — Rail 1 (AS 65002) ──────────────────────────────
    tor-rail1:
      kind: srl
      mgmt-ipv4: 172.20.20.22
      startup-config: configs/tor-rail1.yml

    # ── Server containers (simulate dual-NIC GPU nodes) ─────────────
    server1:
      kind: linux
      mgmt-ipv4: 172.20.20.31
      exec:
        - ip addr add 10.0.1.1/24 dev eth1   # GPU0 NIC on rail0
        - ip addr add 10.1.1.1/24 dev eth2   # GPU1 NIC on rail1
        - ip route add 10.0.0.0/8 via 10.0.1.254 dev eth1

    server2:
      kind: linux
      mgmt-ipv4: 172.20.20.32
      exec:
        - ip addr add 10.0.2.1/24 dev eth1   # GPU0 NIC on rail0
        - ip addr add 10.1.2.1/24 dev eth2   # GPU1 NIC on rail1
        - ip route add 10.0.0.0/8 via 10.0.2.254 dev eth1

  links:
    # Server → ToR (rail0)
    - endpoints: [server1:eth1, tor-rail0:e1-1]
    - endpoints: [server2:eth1, tor-rail0:e1-2]

    # Server → ToR (rail1)
    - endpoints: [server1:eth2, tor-rail1:e1-1]
    - endpoints: [server2:eth2, tor-rail1:e1-2]

    # ToR rail0 uplinks to spines
    - endpoints: [tor-rail0:e1-48, spine1:e1-1]
    - endpoints: [tor-rail0:e1-49, spine2:e1-1]

    # ToR rail1 uplinks to spines
    - endpoints: [tor-rail1:e1-48, spine1:e1-2]
    - endpoints: [tor-rail1:e1-49, spine2:e1-2]
```

Now write the four SR Linux startup configs:

```bash
# configs/spine1.yml
cat > configs/spine1.yml << 'EOF'
interface ethernet-1/1:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 192.168.10.1/31: {}

interface ethernet-1/2:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 192.168.20.1/31: {}

network-instance default:
  interface:
    - name: ethernet-1/1.0
    - name: ethernet-1/2.0
  protocols:
    bgp:
      router-id: 10.0.0.1
      autonomous-system: 65100
      ebgp-default-policy:
        import-reject-all: false
        export-reject-all: false
      group TORS:
        export-policy: pass-all
        import-policy: pass-all
      neighbor 192.168.10.0:
        peer-group: TORS
        peer-as: 65001
      neighbor 192.168.20.0:
        peer-group: TORS
        peer-as: 65002
EOF

# configs/spine2.yml
cat > configs/spine2.yml << 'EOF'
interface ethernet-1/1:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 192.168.11.1/31: {}

interface ethernet-1/2:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 192.168.21.1/31: {}

network-instance default:
  interface:
    - name: ethernet-1/1.0
    - name: ethernet-1/2.0
  protocols:
    bgp:
      router-id: 10.0.0.2
      autonomous-system: 65100
      ebgp-default-policy:
        import-reject-all: false
        export-reject-all: false
      group TORS:
        export-policy: pass-all
        import-policy: pass-all
      neighbor 192.168.11.0:
        peer-group: TORS
        peer-as: 65001
      neighbor 192.168.21.0:
        peer-group: TORS
        peer-as: 65002
EOF

# configs/tor-rail0.yml
cat > configs/tor-rail0.yml << 'EOF'
interface ethernet-1/1:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 10.0.1.254/24: {}

interface ethernet-1/2:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 10.0.2.254/24: {}

interface ethernet-1/48:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 192.168.10.0/31: {}

interface ethernet-1/49:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 192.168.11.0/31: {}

network-instance default:
  interface:
    - name: ethernet-1/1.0
    - name: ethernet-1/2.0
    - name: ethernet-1/48.0
    - name: ethernet-1/49.0
  protocols:
    bgp:
      router-id: 10.0.0.21
      autonomous-system: 65001
      ebgp-default-policy:
        import-reject-all: false
        export-reject-all: false
      group SPINES:
        export-policy: pass-all
        import-policy: pass-all
      neighbor 192.168.10.1:
        peer-group: SPINES
        peer-as: 65100
      neighbor 192.168.11.1:
        peer-group: SPINES
        peer-as: 65100
EOF

# configs/tor-rail1.yml
cat > configs/tor-rail1.yml << 'EOF'
interface ethernet-1/1:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 10.1.1.254/24: {}

interface ethernet-1/2:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 10.1.2.254/24: {}

interface ethernet-1/48:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 192.168.20.0/31: {}

interface ethernet-1/49:
  admin-state: enable
  subinterface 0:
    ipv4:
      address 192.168.21.0/31: {}

network-instance default:
  interface:
    - name: ethernet-1/1.0
    - name: ethernet-1/2.0
    - name: ethernet-1/48.0
    - name: ethernet-1/49.0
  protocols:
    bgp:
      router-id: 10.0.0.22
      autonomous-system: 65002
      ebgp-default-policy:
        import-reject-all: false
        export-reject-all: false
      group SPINES:
        export-policy: pass-all
        import-policy: pass-all
      neighbor 192.168.20.1:
        peer-group: SPINES
        peer-as: 65100
      neighbor 192.168.21.1:
        peer-group: SPINES
        peer-as: 65100
EOF
```

Verify the directory layout:

```bash
find ~/clab-gpu-rail -type f | sort
```

Expected:

```
/home/user/clab-gpu-rail/configs/spine1.yml
/home/user/clab-gpu-rail/configs/spine2.yml
/home/user/clab-gpu-rail/configs/tor-rail0.yml
/home/user/clab-gpu-rail/configs/tor-rail1.yml
/home/user/clab-gpu-rail/gpu-rail-fabric.yml
```

---

### Step 4: Deploy the Topology

```bash
cd ~/clab-gpu-rail
containerlab deploy -t gpu-rail-fabric.yml
```

Expected output (timing approximately 45-90 seconds):

```
INFO[0000] Containerlab v0.57.3 started
INFO[0000] Parsing & checking topology file: gpu-rail-fabric.yml
INFO[0000] Creating lab directory: /home/user/clab-gpu-rail/clab-gpu-rail-fabric
INFO[0000] Creating docker network: Name="mgmt-gpu", IPv4Subnet="172.20.20.0/24", ...
INFO[0001] Pulling ghcr.io/nokia/srlinux:24.3.2 Docker image
INFO[0001] Image already exists
INFO[0001] Pulling ghcr.io/hellt/network-multitool:latest Docker image
INFO[0001] Image already exists
INFO[0002] Creating container: "spine1"
INFO[0002] Creating container: "spine2"
INFO[0003] Creating container: "tor-rail0"
INFO[0003] Creating container: "tor-rail1"
INFO[0004] Creating container: "server1"
INFO[0004] Creating container: "server2"
INFO[0005] Creating virtual wire: spine1:e1-1 <--> tor-rail0:e1-48
INFO[0005] Creating virtual wire: spine1:e1-2 <--> tor-rail1:e1-48
INFO[0006] Creating virtual wire: spine2:e1-1 <--> tor-rail0:e1-49
INFO[0006] Creating virtual wire: spine2:e1-2 <--> tor-rail1:e1-49
INFO[0007] Creating virtual wire: server1:eth1 <--> tor-rail0:e1-1
INFO[0007] Creating virtual wire: server2:eth1 <--> tor-rail0:e1-2
INFO[0008] Creating virtual wire: server1:eth2 <--> tor-rail1:e1-1
INFO[0008] Creating virtual wire: server2:eth2 <--> tor-rail1:e1-2
INFO[0045] Adding containerlab host entries to /etc/hosts file
INFO[0045] 🎉 New containerlab version 0.58.0 is available! ...
+----+----------------------------------+--------------+----------------------------------+-------+---------+------------------+
| #  |             Name                 | Container ID |              Image               | Kind  |  State  |   IPv4 Address   |
+----+----------------------------------+--------------+----------------------------------+-------+---------+------------------+
|  1 | clab-gpu-rail-fabric-server1     | 3a7f1b2c9d4e | ghcr.io/hellt/network-multitool  | linux | running | 172.20.20.31/24  |
|  2 | clab-gpu-rail-fabric-server2     | 8b2e4c1f7a9d | ghcr.io/hellt/network-multitool  | linux | running | 172.20.20.32/24  |
|  3 | clab-gpu-rail-fabric-spine1      | 1c4d8e2b5f0a | ghcr.io/nokia/srlinux:24.3.2     | srl   | running | 172.20.20.11/24  |
|  4 | clab-gpu-rail-fabric-spine2      | 9f3a2c7e1b4d | ghcr.io/nokia/srlinux:24.3.2     | srl   | running | 172.20.20.12/24  |
|  5 | clab-gpu-rail-fabric-tor-rail0   | 5d1b9e4c2a7f | ghcr.io/nokia/srlinux:24.3.2     | srl   | running | 172.20.20.21/24  |
|  6 | clab-gpu-rail-fabric-tor-rail1   | 2a7c3f8b1e5d | ghcr.io/nokia/srlinux:24.3.2     | srl   | running | 172.20.20.22/24  |
+----+----------------------------------+--------------+----------------------------------+-------+---------+------------------+
```

---

### Step 5: Inspect the Running Topology

```bash
containerlab inspect -t gpu-rail-fabric.yml
```

Expected output (same table as deploy, but confirms persistent state):

```
+----+----------------------------------+--------------+----------------------------------+-------+---------+------------------+
| #  |             Name                 | Container ID |              Image               | Kind  |  State  |   IPv4 Address   |
+----+----------------------------------+--------------+----------------------------------+-------+---------+------------------+
|  1 | clab-gpu-rail-fabric-server1     | 3a7f1b2c9d4e | ghcr.io/hellt/network-multitool  | linux | running | 172.20.20.31/24  |
|  2 | clab-gpu-rail-fabric-server2     | 8b2e4c1f7a9d | ghcr.io/hellt/network-multitool  | linux | running | 172.20.20.32/24  |
|  3 | clab-gpu-rail-fabric-spine1      | 1c4d8e2b5f0a | ghcr.io/nokia/srlinux:24.3.2     | srl   | running | 172.20.20.11/24  |
|  4 | clab-gpu-rail-fabric-spine2      | 9f3a2c7e1b4d | ghcr.io/nokia/srlinux:24.3.2     | srl   | running | 172.20.20.12/24  |
|  5 | clab-gpu-rail-fabric-tor-rail0   | 5d1b9e4c2a7f | ghcr.io/nokia/srlinux:24.3.2     | srl   | running | 172.20.20.21/24  |
|  6 | clab-gpu-rail-fabric-tor-rail1   | 2a7c3f8b1e5d | ghcr.io/nokia/srlinux:24.3.2     | srl   | running | 172.20.20.22/24  |
+----+----------------------------------+--------------+----------------------------------+-------+---------+------------------+
```

Also verify all containers are running:

```bash
docker ps --filter "label=clab-node-name" --format "table {{.Names}}\t{{.Status}}"
```

Expected:

```
NAMES                              STATUS
clab-gpu-rail-fabric-server1      Up 2 minutes
clab-gpu-rail-fabric-server2      Up 2 minutes
clab-gpu-rail-fabric-spine1       Up 2 minutes
clab-gpu-rail-fabric-spine2       Up 2 minutes
clab-gpu-rail-fabric-tor-rail0    Up 2 minutes
clab-gpu-rail-fabric-tor-rail1    Up 2 minutes
```

---

### Step 6: SSH into spine1 and Configure eBGP

Containerlab adds entries to `/etc/hosts`, so you can reach nodes by name:

```bash
ssh admin@clab-gpu-rail-fabric-spine1
# Default SR Linux credentials: admin / NokiaSrl1!
# Accept the host key fingerprint when prompted
```

Once inside, open the SR Linux CLI:

```
[admin@spine1 ~]$ sr_cli
Welcome to the srlinux CLI.
--{ running }--[ ]--
A:spine1#
```

Verify the startup config was applied (interfaces should already be configured):

```
A:spine1# show interface brief
+--------------------+------+-----+----------+
| Interface          | Adm  | Oper| IPv4     |
+--------------------+------+-----+----------+
| ethernet-1/1.0     | up   | up  |192.168.10.1/31|
| ethernet-1/2.0     | up   | up  |192.168.20.1/31|
| mgmt0.0            | up   | up  |172.20.20.11/24|
+--------------------+------+-----+----------+
```

If you need to add or adjust BGP configuration interactively:

```
A:spine1# enter candidate
--{ candidate shared default }--[ ]--
A:spine1# set network-instance default protocols bgp router-id 10.0.0.1
A:spine1# set network-instance default protocols bgp autonomous-system 65100
A:spine1# set network-instance default protocols bgp ebgp-default-policy import-reject-all false
A:spine1# set network-instance default protocols bgp ebgp-default-policy export-reject-all false
A:spine1# set network-instance default protocols bgp group TORS export-policy pass-all
A:spine1# set network-instance default protocols bgp group TORS import-policy pass-all
A:spine1# set network-instance default protocols bgp neighbor 192.168.10.0 peer-group TORS
A:spine1# set network-instance default protocols bgp neighbor 192.168.10.0 peer-as 65001
A:spine1# set network-instance default protocols bgp neighbor 192.168.20.0 peer-group TORS
A:spine1# set network-instance default protocols bgp neighbor 192.168.20.0 peer-as 65002
A:spine1# commit now
All changes have been committed. Leaving candidate mode.
A:spine1#
```

Exit the SSH session:

```
A:spine1# quit
[admin@spine1 ~]$ exit
```

---

### Step 7: SSH into tor-rail0 and Verify eBGP Configuration

```bash
ssh admin@clab-gpu-rail-fabric-tor-rail0
```

```
[admin@tor-rail0 ~]$ sr_cli
A:tor-rail0# show interface brief
+--------------------+------+-----+----------------+
| Interface          | Adm  | Oper| IPv4           |
+--------------------+------+-----+----------------+
| ethernet-1/1.0     | up   | up  | 10.0.1.254/24  |
| ethernet-1/2.0     | up   | up  | 10.0.2.254/24  |
| ethernet-1/48.0    | up   | up  | 192.168.10.0/31|
| ethernet-1/49.0    | up   | up  | 192.168.11.0/31|
| mgmt0.0            | up   | up  | 172.20.20.21/24|
+--------------------+------+-----+----------------+
```

Add a VXLAN-style loopback to represent the rail VTEP (optional, for VXLAN chapters):

```
A:tor-rail0# enter candidate
A:tor-rail0# set interface lo0 subinterface 0 ipv4 address 10.255.0.21/32
A:tor-rail0# set network-instance default interface lo0.0
A:tor-rail0# set network-instance default protocols bgp network-instance protocols bgp neighbor 192.168.10.1 peer-as 65100
A:tor-rail0# commit now
```

Exit:

```
A:tor-rail0# quit
[admin@tor-rail0 ~]$ exit
```

---

### Step 8: Configure Server Interfaces

The `exec` directives in the topology YAML run at container start, but verify manually:

```bash
# Verify server1 interfaces
docker exec clab-gpu-rail-fabric-server1 ip addr show
```

Expected output:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 172.20.20.31/24 brd 172.20.20.255 scope global eth0
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 ...
    inet 10.0.1.1/24 brd 10.0.1.255 scope global eth1
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 ...
    inet 10.1.1.1/24 brd 10.1.1.255 scope global eth2
```

```bash
# Verify server2 interfaces
docker exec clab-gpu-rail-fabric-server2 ip addr show
```

Expected:

```
3: eth1: ... inet 10.0.2.1/24 ...
4: eth2: ... inet 10.1.2.1/24 ...
```

Verify routing table on server1:

```bash
docker exec clab-gpu-rail-fabric-server1 ip route show
```

Expected:

```
default via 172.20.20.1 dev eth0
10.0.0.0/8 via 10.0.1.254 dev eth1
10.0.1.0/24 dev eth1 proto kernel scope link src 10.0.1.1
10.1.1.0/24 dev eth2 proto kernel scope link src 10.1.1.1
172.20.20.0/24 dev eth0 proto kernel scope link src 172.20.20.31
```

---

### Step 9: Wait for BGP Convergence and Poll

BGP in SR Linux needs approximately 30 seconds to establish sessions after all containers are healthy. Poll until sessions show "established":

```bash
# Poll every 5 seconds until all 2 BGP sessions on spine1 are established
for i in $(seq 1 12); do
  COUNT=$(docker exec clab-gpu-rail-fabric-spine1 \
    sr_cli "show network-instance default protocols bgp summary" 2>/dev/null \
    | grep -ci "established" || echo 0)
  echo "$(date +%H:%M:%S)  Established sessions on spine1: $COUNT/2"
  [ "$COUNT" -ge 2 ] && echo "BGP converged!" && break
  sleep 5
done
```

Expected output (converges at approximately T+30s):

```
14:22:01  Established sessions on spine1: 0/2
14:22:06  Established sessions on spine1: 0/2
14:22:11  Established sessions on spine1: 1/2
14:22:16  Established sessions on spine1: 2/2
BGP converged!
```

Run the full BGP summary on spine1 after convergence:

```bash
docker exec clab-gpu-rail-fabric-spine1 \
  sr_cli "show network-instance default protocols bgp summary"
```

Expected:

```
BGP is enabled and up
Global AS number  : 65100
BGP identifier    : 10.0.0.1

Instance          : "default"
+----------+-------------------+-------+--------+------+------+
| Group    | Neighbor          | State | Uptime | #Rx  | #Tx  |
+----------+-------------------+-------+--------+------+------+
| TORS     | 192.168.10.0      | established | 0d0h0m42s | 4  | 3  |
| TORS     | 192.168.20.0      | established | 0d0h0m38s | 4  | 3  |
+----------+-------------------+-------+--------+------+------+
```

---

### Step 10: Verify ECMP — 2 Paths to Each Server Prefix

Spine1 should have two equal-cost paths to server1's rail0 prefix (10.0.1.0/24) — one learned via tor-rail0 and one reflected/imported. Because this is a simple 2-ToR topology, spine1 learns the server prefixes only from tor-rail0. In the full 4-leaf topology, spine1 would see 4 paths. Here we confirm 2 paths (via the two ToRs for server-network prefixes):

```bash
docker exec clab-gpu-rail-fabric-spine1 \
  sr_cli "show network-instance default route-table ipv4-unicast prefix 10.0.1.0/24"
```

Expected output:

```
-----------------------------------------------------------------------
IPv4 unicast route table of network instance "default"
-----------------------------------------------------------------------
+------------------+------+-----+----------+-----+--------+----------+
| Prefix           | Id   | Act | Metric   | Pref| Next-  | Protocol |
|                  |      | ive |          |     | hop    |          |
+------------------+------+-----+----------+-----+--------+----------+
| 10.0.1.0/24      |  0   | *   | 0        | 170 |192.168.| bgp      |
|                  |      |     |          |     |10.0    |          |
+------------------+------+-----+----------+-----+--------+----------+

Route count: 1 active

Next-hops:
  192.168.10.0 (ethernet-1/1.0)   [tor-rail0 via spine1:e1-1]
```

Check all BGP received routes to count paths:

```bash
docker exec clab-gpu-rail-fabric-spine1 \
  sr_cli "show network-instance default protocols bgp routes ipv4 unicast summary"
```

Expected (routes from both ToRs visible):

```
+------------------+-----+--------+-------+-----------+-------+
| Prefix           | Ori | Valid  | Best  | Neighbor  | AS    |
|                  | gin |        |       |           | Path  |
+------------------+-----+--------+-------+-----------+-------+
| 10.0.1.0/24      |  e  | true   | true  |192.168.10.0| 65001|
| 10.0.2.0/24      |  e  | true   | true  |192.168.10.0| 65001|
| 10.1.1.0/24      |  e  | true   | true  |192.168.20.0| 65002|
| 10.1.2.0/24      |  e  | true   | true  |192.168.20.0| 65002|
+------------------+-----+--------+-------+-----------+-------+
```

Verify connectivity from spine1 to all server prefixes:

```bash
docker exec clab-gpu-rail-fabric-spine1 ping -c 2 10.0.1.1
docker exec clab-gpu-rail-fabric-spine1 ping -c 2 10.0.2.1
```

Expected:

```
PING 10.0.1.1 (10.0.1.1) 56(84) bytes of data.
64 bytes from 10.0.1.1: icmp_seq=1 ttl=64 time=0.412 ms
64 bytes from 10.0.1.1: icmp_seq=2 ttl=64 time=0.388 ms
--- 10.0.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss
```

---

### Step 11: Run iperf3 Between server1 and server2 (Rail 0)

Start an iperf3 server on server2's rail0 interface:

```bash
docker exec -d clab-gpu-rail-fabric-server2 iperf3 -s -B 10.0.2.1 -p 5201
```

Wait 1 second, then run the client from server1:

```bash
docker exec clab-gpu-rail-fabric-server1 \
  iperf3 -c 10.0.2.1 -p 5201 -t 10 -P 4 --bidir
```

Expected output (throughput depends on host hardware; expect 2-10 Gbps in a VM, 20-40 Gbps on bare metal):

```
Connecting to host 10.0.2.1, port 5201
[  5] local 10.0.1.1 port 58432 connected to 10.0.2.1 port 5201
[  7] local 10.0.1.1 port 58434 connected to 10.0.2.1 port 5201
[  9] local 10.0.1.1 port 58436 connected to 10.0.2.1 port 5201
[ 11] local 10.0.1.1 port 58438 connected to 10.0.2.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  1.18 GBytes  1.01 Gbps    0             sender
[  7]   0.00-10.00  sec  1.21 GBytes  1.04 Gbps    0             sender
[  9]   0.00-10.00  sec  1.19 GBytes  1.02 Gbps    0             sender
[ 11]   0.00-10.00  sec  1.17 GBytes  1.01 Gbps    0             sender
[SUM]   0.00-10.00  sec  4.75 GBytes  4.08 Gbps    0             sender
[SUM]   0.00-10.00  sec  4.73 GBytes  4.06 Gbps    0             receiver

Bidir results:
[ ID] Interval           Transfer     Bitrate
[  5]-[SUM] TX:  4.75 GBytes  4.08 Gbps
[  5]-[SUM] RX:  4.73 GBytes  4.06 Gbps
iperf Done.
```

Now simulate AllReduce traffic on rail 1 simultaneously:

```bash
docker exec -d clab-gpu-rail-fabric-server2 iperf3 -s -B 10.1.2.1 -p 5202
docker exec clab-gpu-rail-fabric-server1 \
  iperf3 -c 10.1.2.1 -p 5202 -t 10 -P 4 --bidir
```

---

### Step 12: Packet Capture on a Specific Link

Containerlab does not have a built-in capture subcommand; packet capture is done directly via `tcpdump` on the host veth interface or inside the container. Identify the host-side veth interface for the link between tor-rail0 and spine1:

```bash
# Identify the veth interface name on the host by inspecting the container's netns
CONTAINER_PID=$(docker inspect clab-gpu-rail-fabric-spine1 --format '{{.State.Pid}}')
sudo nsenter -t $CONTAINER_PID -n ip link show e1-1
# Note the veth peer index, then find it on the host:
# ip link | grep "^<peer-index>:"
```

Capture traffic on the spine1 side of the veth while running iperf3:

```bash
# In terminal 1 — start capture inside the container (save to pcap file)
docker exec clab-gpu-rail-fabric-spine1 \
  tcpdump -i e1-1 -w /tmp/spine1-e1-1.pcap &
CAPTURE_PID=$!

# In terminal 2 — generate traffic
docker exec clab-gpu-rail-fabric-server1 \
  iperf3 -c 10.0.2.1 -p 5201 -t 5 -P 4

# Stop capture (send SIGTERM to the docker exec process; tcpdump flushes and exits)
kill $CAPTURE_PID
wait $CAPTURE_PID 2>/dev/null

# Copy pcap out of the container
docker cp clab-gpu-rail-fabric-spine1:/tmp/spine1-e1-1.pcap \
  /tmp/spine1-e1-1.pcap

# Inspect the pcap
docker run --rm -v /tmp:/data nicolaka/netshoot \
  tshark -r /data/spine1-e1-1.pcap -q -z io,stat,1 2>/dev/null | head -20
```

Expected tshark output:

```
=================================
| IO Statistics                 |
|                               |
| Duration: 5.123 secs          |
| Interval: 1 secs              |
|                               |
| Col 1: Frames and bytes       |
+--------+---------+--------+
|  Time  | Frames  | Bytes  |
+--------+---------+--------+
|  0 <>1 |    8423 |9421504 |
|  1 <>2 |    8918 |9976432 |
|  2 <>3 |    9012 |9893312 |
|  3 <>4 |    8734 |9765888 |
|  4 <>5 |    7823 |8742144 |
+--------+---------+--------+
```

---

### Step 13: Tear Down the Topology

```bash
cd ~/clab-gpu-rail
containerlab destroy -t gpu-rail-fabric.yml
```

Expected output:

```
INFO[0000] Parsing & checking topology file: gpu-rail-fabric.yml
INFO[0001] Destroying lab: gpu-rail-fabric
INFO[0001] Removing container: clab-gpu-rail-fabric-server1
INFO[0001] Removing container: clab-gpu-rail-fabric-server2
INFO[0002] Removing container: clab-gpu-rail-fabric-spine1
INFO[0002] Removing container: clab-gpu-rail-fabric-spine2
INFO[0002] Removing container: clab-gpu-rail-fabric-tor-rail0
INFO[0003] Removing container: clab-gpu-rail-fabric-tor-rail1
INFO[0003] Removing docker network: mgmt-gpu
INFO[0003] Removing /etc/hosts entries for lab gpu-rail-fabric
INFO[0003] Lab gpu-rail-fabric destroyed
```

Verify cleanup:

```bash
docker ps --filter "label=clab-node-name"
# Expected: CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
# (empty — no running containers)

docker network ls | grep mgmt-gpu
# Expected: (no output)
```

---

### Step 14: Add This Topology as a GitHub Actions CI Job

Save the following as `.github/workflows/gpu-rail-ci.yml` in your repository. This runs on every push or PR that touches configs or the topology file. The runner must have Docker available — use a self-hosted runner with Docker pre-installed, or a GitHub-hosted runner with `docker` in the runner image.

```yaml
# .github/workflows/gpu-rail-ci.yml
name: GPU Rail Fabric CI

on:
  push:
    paths:
      - 'gpu-rail-fabric.yml'
      - 'configs/**'
  pull_request:
    paths:
      - 'gpu-rail-fabric.yml'
      - 'configs/**'

env:
  CLAB_VERSION: "0.57.3"

jobs:
  deploy-and-test:
    name: Deploy GPU Rail Fabric and Validate BGP
    runs-on: ubuntu-latest   # swap for self-hosted if you need KVM/QEMU

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install Containerlab ${{ env.CLAB_VERSION }}
        run: |
          bash -c "$(curl -sL https://get.containerlab.dev)" -- -v ${{ env.CLAB_VERSION }}
          containerlab version

      - name: Pull SR Linux image (cached by Docker layer cache)
        run: |
          docker pull ghcr.io/nokia/srlinux:24.3.2
          docker pull ghcr.io/hellt/network-multitool:latest

      - name: Deploy GPU rail fabric topology
        run: |
          containerlab deploy -t gpu-rail-fabric.yml --reconfigure
        timeout-minutes: 3

      - name: Wait for BGP convergence (60s maximum)
        run: |
          for i in $(seq 1 12); do
            COUNT=$(docker exec clab-gpu-rail-fabric-spine1 \
              sr_cli "show network-instance default protocols bgp summary" 2>/dev/null \
              | grep -ci "established" || echo 0)
            echo "Attempt $i/12: established=$COUNT/2"
            [ "$COUNT" -ge 2 ] && exit 0
            sleep 5
          done
          echo "ERROR: BGP did not converge in 60 seconds"
          docker exec clab-gpu-rail-fabric-spine1 \
            sr_cli "show network-instance default protocols bgp summary" || true
          exit 1

      - name: Assert reachability — server1 rail0 to server2 rail0
        run: |
          docker exec clab-gpu-rail-fabric-server1 ping -c 5 -W 2 10.0.2.1
          echo "PASS: server1 → server2 rail0 reachable"

      - name: Assert reachability — server1 rail1 to server2 rail1
        run: |
          docker exec clab-gpu-rail-fabric-server1 ping -c 5 -W 2 10.1.2.1
          echo "PASS: server1 → server2 rail1 reachable"

      - name: Run iperf3 throughput validation
        run: |
          docker exec -d clab-gpu-rail-fabric-server2 iperf3 -s -B 10.0.2.1 -p 5201
          sleep 1
          docker exec clab-gpu-rail-fabric-server1 \
            iperf3 -c 10.0.2.1 -p 5201 -t 10 -P 4 --json \
            | tee /tmp/iperf3-result.json
          # Assert minimum throughput (1 Gbps in CI environment)
          BITSPS=$(jq '.end.sum_sent.bits_per_second' /tmp/iperf3-result.json)
          python3 -c "
          bps = $BITSPS
          assert bps >= 1e9, f'Throughput {bps/1e9:.2f} Gbps below 1 Gbps minimum'
          print(f'PASS: throughput = {bps/1e9:.2f} Gbps')
          "

      - name: Verify BGP routes on spine1
        run: |
          ROUTES=$(docker exec clab-gpu-rail-fabric-spine1 \
            sr_cli "show network-instance default protocols bgp routes ipv4 unicast summary" \
            | grep -c "10\.[01]\." || echo 0)
          echo "BGP routes on spine1 to server subnets: $ROUTES"
          python3 -c "assert $ROUTES >= 4, 'Expected 4+ server routes on spine1, got $ROUTES'"
          echo "PASS: $ROUTES routes present on spine1"

      - name: Destroy topology (always runs)
        if: always()
        run: |
          containerlab destroy -t gpu-rail-fabric.yml --cleanup
          echo "Topology destroyed."
```

To test this CI job locally before pushing, use `act`:

```bash
# Install act
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Run the workflow locally
act push --job deploy-and-test \
  --secret GITHUB_TOKEN="$(gh auth token)" \
  -P ubuntu-latest=catthehacker/ubuntu:act-latest
```

---

## Summary

- Containerlab is IaC for network labs: YAML topology + Docker = reproducible, version-controlled, CI-testable network environments.
- SR Linux, SONiC-VS, and FRR containers provide a full open-source NOS ecosystem runnable on a laptop.
- vrnetlab enables VM-based commercial NOSes (IOS-XE, vMX) alongside containers in the same topology.
- CI integration enables topology tests to run on every PR — treating network configuration like application code.

---

## References

- Containerlab documentation: containerlab.dev
- SR Linux container image: github.com/nokia/srlinux-container-image
- SONiC-VS: github.com/sonic-net/sonic-buildimage
- vrnetlab: github.com/hellt/vrnetlab
- Containerlab examples: github.com/srl-labs/containerlab/tree/main/lab-examples


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).