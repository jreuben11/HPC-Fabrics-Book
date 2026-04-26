# Chapter 8 — Open Network Operating Systems

**Part III: Programmable Fabric** | ~25 pages

---

## Installation

This chapter uses Containerlab as the lab orchestrator (a tool that deploys multi-vendor network topologies as Docker containers, wiring them together with virtual links defined in a YAML file), Docker as the container runtime, SR Linux and SONiC-VS as the NOS images, FRR as the host-router control plane, and gnmic (a gNMI CLI client for querying and subscribing to network telemetry) for gNMI streaming telemetry. All tools run on Ubuntu 24.04.

### Docker

```bash
sudo apt install -y docker.io
sudo usermod -aG docker $USER
# Log out and back in, or run: newgrp docker

docker --version
# Expected: Docker version 24.x or 25.x
```

### Containerlab

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"

containerlab version
# Expected:
#                     _       _       _
#   ___ ___  _ __  | |_ __ _(_)_ __ | | __ _| |__
#  / __/ _ \| '_ \ | __/ _` | | '_ \| |/ _` | '_ \
# | (_| (_) | | | || || (_| | | | | | | (_| | |_) |
#  \___\___/|_| |_| \__\__,_|_|_| |_|_|\__,_|_.__/
#
#     version: 0.55.x
```

### gnmic

```bash
bash -c "$(curl -sL https://get.gnmic.openconfig.net)"

gnmic version
# Expected: gnmic version v0.38.x
```

### FRR (for host router containers)

```bash
sudo apt install -y frr frr-pythontools

# Enable the daemons you need
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo sed -i 's/^bfdd=no/bfdd=yes/' /etc/frr/daemons

sudo systemctl restart frr
sudo systemctl status frr | grep Active
# Expected: Active: active (running)
```

### Python automation environment (uv)

`ncclient` is a Python library implementing the NETCONF client protocol (RFC 6241) for configuration management. `netmiko` is a multi-vendor SSH library that simplifies parameterized CLI interactions with network devices from many vendors.

```bash
# Install uv if not already present
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# Create a project venv for automation scripts
uv venv .venv
source .venv/bin/activate

# Install NETCONF/SSH automation libraries
uv pip install ncclient paramiko netmiko

python -c "import ncclient, paramiko, netmiko; print('All imports OK')"
# Expected: All imports OK
```

---

## Introduction

The network operating system (NOS) running on a switch determines how that switch is configured, automated, observed, and extended. For most of networking history, that software was proprietary, opaque, and tightly coupled to a single vendor's hardware. The rise of open NOSes — driven by hyperscalers who needed to operate networks at a scale and velocity that proprietary platforms could not support — fundamentally changed this equation, and AI cluster operators are the direct beneficiaries.

This chapter surveys the open NOS landscape as it applies to AI cluster fabrics. SONiC (Software for Open Networking in the Cloud) is the production-grade choice for hyperscaler-class switching: a collection of containerized daemons communicating through a Redis database, running on any SAI-compliant ASIC from Broadcom, Mellanox, or Marvell. SR Linux is Nokia's programmability-first NOS: every configuration object is YANG-modeled, every state path is accessible via gNMI, and custom applications can be injected via the NDK without forking the NOS itself. FRRouting (FRR) provides the routing control plane — BGP, OSPF, IS-IS, EVPN — for both, and is the standard tool for configuring eBGP underlay fabrics in AI clusters.

The reader will learn: how SONiC's Redis-centric architecture decouples configuration intent from ASIC programming; how SR Linux's YANG-native model enables controller-free automation; how to configure FRR for a spine-leaf eBGP fabric with ECMP and sub-second BFD failure detection; and how Containerlab composes these NOSes into a full lab topology on a single laptop. The chapter also surveys VyOS, DENT, OpenWrt, FreeRTOS, and Zephyr — the open NOS and RTOS landscape at the edges of the AI cluster fabric.

The lab walkthrough builds a two-spine, two-leaf BGP fabric using SR Linux and SONiC-VS containers, verifies ECMP convergence, simulates a link failure with BFD, and streams gNMI telemetry from SR Linux to gnmic — all within Containerlab running on Ubuntu 24.04.

This chapter connects backward to Chapter 5 (DPDK) and Chapter 7 (eBPF), which addressed the data-plane performance problems inside individual hosts, and forward to Chapter 14 (NETCONF/YANG/RESTCONF) and Chapter 15 (gNMI/OpenConfig), which cover the management protocols that open NOSes like SR Linux expose natively.

---

## 8.1 The End of the Monolithic Network Appliance

For decades, a network switch was a black box: proprietary ASIC, proprietary OS, proprietary CLI. Configuration required vendor-specific commands, automation required screen-scraping, and replacing the switch meant re-learning everything.

Disaggregation broke this model. The emergence of merchant silicon (Broadcom Tomahawk, Trident, Intel Tofino) — commodity ASICs delivering the same forwarding performance as proprietary chips — enabled a decoupling of hardware from software. The Switch Abstraction Interface (SAI) is an open API specification, maintained by the Open Compute Project, that defines a vendor-neutral C interface between a NOS and the underlying ASIC SDK, so the same NOS binary can target ASICs from multiple vendors without modification.

Three open NOSes now dominate different parts of the AI infrastructure landscape: **SONiC** for hyperscaler DC switching, **SR Linux** for programmability-first environments, and **FRR** as the universal routing engine embedded in both.

---

## 8.2 SONiC — Software for Open Networking in the Cloud

### 8.2.1 Architecture Overview

SONiC is a collection of containerized networking daemons running on a standard Debian Linux host, with a Redis database at the center. Redis is an open-source in-memory data structure store used here as SONiC's inter-process communication bus: each daemon reads and writes to named tables in Redis, and changes propagate through a publish-subscribe mechanism without direct daemon-to-daemon coupling:

```
┌─────────────────────────────────────────────────────────┐
│  Management plane: CLI, REST API, gNMI, NETCONF         │
├─────────────────────────────────────────────────────────┤
│  CONFIG_DB ← configuration intent                       │
│  APP_DB    ← application state (FIB, neighbors, etc.)   │  Redis
│  STATE_DB  ← operational state                          │
│  ASIC_DB   ← ASIC-level representation                  │
├──────────┬──────────┬──────────┬──────────┬────────────┤
│ bgpd     │ teamd    │ lldpd    │ snmpd    │ dhcprelayd │  Containers
│ (FRR)    │ (LAG)    │          │          │            │
├──────────┴──────────┴──────────┴──────────┴────────────┤
│  orchagent  — translates APP_DB → ASIC_DB               │
├─────────────────────────────────────────────────────────┤
│  syncd      — translates ASIC_DB → SAI calls            │
├─────────────────────────────────────────────────────────┤
│  SAI (Switch Abstraction Interface)                     │
├─────────────────────────────────────────────────────────┤
│  ASIC SDK (Broadcom, Mellanox/NVIDIA, Marvell, ...)     │
└─────────────────────────────────────────────────────────┘
```

Every configuration change follows the same path: write to CONFIG_DB → orchagent (the orchestration agent daemon that translates high-level application state from APP_DB into low-level ASIC_DB entries) reads APP_DB → translates to ASIC_DB entries → syncd (the synchronization daemon that bridges ASIC_DB changes to vendor SAI API calls) calls SAI → ASIC SDK programs the forwarding hardware.

### 8.2.2 Redis ConfigDB

SONiC's configuration lives in Redis hashes. Direct manipulation is possible for automation:

```bash
# Show all interface configurations
redis-cli -n 4 hgetall "PORT|Ethernet0"
# 1) "admin_status"
# 2) "up"
# 3) "speed"
# 4) "400000"
# 5) "mtu"
# 6) "9100"

# Configure an IP address
redis-cli -n 4 hmset "INTERFACE|Ethernet0|192.168.1.1/24" \
    "scope" "global" "family" "IPv4"

# Trigger orchagent to pick up the change
redis-cli -n 4 publish CONFIG_DB_UPDATED ""
```

In practice, the `config` CLI writes to CONFIG_DB:

```bash
# SONiC CLI equivalents
config interface ip add Ethernet0 192.168.1.1/24
config bgp startup all
config save     # persist CONFIG_DB to /etc/sonic/config_db.json
```

### 8.2.3 SONiC Container Management

Each SONiC daemon runs in its own Docker container:

```bash
# List running SONiC containers
docker ps --format "table {{.Names}}\t{{.Status}}"

# Inspect BGP container (FRR)
docker exec bgp vtysh -c "show bgp summary"

# View orchagent logs
docker logs orchagent --tail 50

# Enter syncd container to debug SAI
docker exec syncd bash
```

### 8.2.4 SONiC-VS — Virtual SONiC for Development

SONiC-VS (Virtual Switch) runs SONiC in a VM or container without real ASIC hardware, using a software-simulated SAI. Used for:
- CI pipelines testing configuration changes
- Developer workstations
- Containerlab topologies

```bash
# Pull SONiC-VS image
docker pull docker.io/sonicdev/sonic-vs:latest

# Run in Containerlab topology:
# topology.yml:
# nodes:
#   spine1:
#     kind: sonic-vs
#     image: sonicdev/sonic-vs:latest
```

---

## 8.3 SR Linux — Nokia's Programmable NOS

### 8.3.1 Design Principles

SR Linux was designed from the ground up with three principles: everything is YANG-modeled (YANG is a data modeling language, defined in RFC 7950, that describes the structure, types, and constraints of network configuration and state data in a machine-readable schema), everything is gNMI-accessible (gNMI, gRPC Network Management Interface, is a gRPC-based protocol for streaming telemetry and configuration management using YANG paths as addresses), and everything is extensible via the NDK (Network Developer Kit). There is no concept of a CLI-only feature — if it exists in SR Linux, it has a YANG path.

### 8.3.2 YANG-Native Configuration

```bash
# SR Linux CLI uses a YANG-aware, path-structured interface
# Enter candidate configuration
enter candidate

# Configure interface
set /interface ethernet-1/1 admin-state enable
set /interface ethernet-1/1 subinterface 0 ipv4 address 192.168.1.1/24

# BGP configuration — YANG paths are self-documenting
set /network-instance default protocols bgp router-id 10.0.0.1
set /network-instance default protocols bgp autonomous-system 65001
set /network-instance default protocols bgp group SPINE peer-as 65000
set /network-instance default protocols bgp neighbor 192.168.1.2 peer-group SPINE

# Commit
commit now
```

### 8.3.3 gNMI-First Access

Because all state is YANG-modeled, gNMI works without any special configuration:

```bash
# Subscribe to interface counters via gnmic
gnmic -a srlinux:57400 --skip-verify subscribe \
    --path /interface[name=ethernet-1/1]/statistics \
    --mode stream --stream-mode sample --sample-interval 5s

# Get BGP neighbor state
gnmic -a srlinux:57400 get \
    --path /network-instance[name=default]/protocols/bgp/neighbor
```

### 8.3.4 NDK — Network Developer Kit

NDK allows writing custom applications in Go or Python that run alongside SR Linux and have access to all system state:

```go
// Go NDK application skeleton
import (
    ndk "github.com/nokia/srlinux-ndk-go/ndk"
)

func main() {
    conn, _ := grpc.Dial("localhost:50053", grpc.WithInsecure())
    client := ndk.NewSdkMgrServiceClient(conn)

    // Register application
    client.AgentRegister(ctx, &ndk.AgentRegistrationRequest{
        AgentName: "my-app",
    })

    // Subscribe to route table changes
    routeStream, _ := client.GetRouteTableChangedNotifStream(ctx,
        &ndk.RouteTableGetRequest{})

    for {
        notif, _ := routeStream.Recv()
        // React to route changes
    }
}
```

---

## 8.4 FRRouting (FRR)

FRR is the routing daemon suite that powers BGP, OSPF, IS-IS, EVPN, and more in both SONiC and SR Linux. It is not a full NOS — it provides only the control plane. The forwarding plane is handled by the kernel FIB (on Linux routers) or the ASIC (via SAI in SONiC).

### 8.4.1 Daemon Architecture

```
vtysh  ←→  zebra (FIB manager)
             ↕ Zserv protocol
       bgpd  ospfd  isisd  ldpd  pathd  ...
```

`zebra` is the central daemon. Protocol daemons (`bgpd`, `ospfd`, etc.) compute routes and send them to `zebra`, which installs them in the kernel FIB (or ASIC_DB in SONiC).

### 8.4.2 BGP for AI Cluster Underlay

```bash
# vtysh configuration — BGP for spine-leaf fabric
configure terminal

router bgp 65001
 bgp router-id 10.0.0.1
 bgp bestpath as-path multipath-relax    # allow ECMP across different ASNs
 maximum-paths 64                         # up to 64 ECMP paths

 neighbor SPINES peer-group
 neighbor SPINES remote-as external       # eBGP
 neighbor SPINES bfd                     # BFD for fast failure detection
 neighbor 192.168.1.0 peer-group SPINES
 neighbor 192.168.2.0 peer-group SPINES

 address-family ipv4 unicast
  network 10.1.0.0/24
  neighbor SPINES activate
  neighbor SPINES route-map IMPORT in
  neighbor SPINES route-map EXPORT out
 exit-address-family
!
end
write memory
```

### 8.4.3 EVPN in FRR

EVPN (Ethernet VPN, RFC 7432) is a BGP address family that distributes MAC and IP reachability information for overlay networks, most commonly used as the control plane for VXLAN tunnels. It replaces the flood-and-learn MAC discovery of traditional bridging with a scalable BGP-based mechanism.

FRR supports BGP-EVPN natively for VXLAN overlay control:

```bash
router bgp 65001
 address-family l2vpn evpn
  neighbor SPINE activate
  advertise-all-vni           # advertise all local VNIs
 exit-address-family
!

# VNI-to-VLAN mapping (in conjunction with Linux bridge + VXLAN interface)
vni 10100
 rd 10.0.0.1:100
 route-target import 65000:100
 route-target export 65001:100
!
```

### 8.4.4 BFD — Fast Failure Detection

BFD (Bidirectional Forwarding Detection) detects link failures in milliseconds, far faster than BGP hold timers:

```bash
# Enable BFD on BGP neighbor
router bgp 65001
 neighbor 192.168.1.0 bfd 100 100 3    # min-tx, min-rx, multiplier (300ms detection)
!

# Monitor BFD sessions
show bfd peers counters
```

---

## 8.5 Choosing Between SONiC and SR Linux

| Criterion | SONiC | SR Linux |
|---|---|---|
| Ecosystem | Hyperscaler-proven; broadest hardware support | Nokia-ecosystem; containerized |
| ASIC support | Broadcom, Mellanox, Marvell, Barefoot, ... | Nokia silicon (ASICs) + container (vs) |
| Programmability | Python scripts + Redis access | NDK (Go/Python gRPC) + YANG/gNMI native |
| Management API | gNMI, RESTCONF, NETCONF (via modules) | gNMI-native, JSON-RPC |
| Community | OCP, large hyperscaler community | Nokia + open community |
| Best for | Production DC switching at scale | Programmability-first, Containerlab reference |

For a laptop learning environment: use SR Linux (free container) + SONiC-VS (container) + FRR (router containers) in Containerlab.

---

## 8.6 Additional FOSS NOSes and RTOSes

The three NOSes above handle the data-center switching and routing workloads that dominate AI cluster fabrics. Several other open-source systems are relevant at the edges of that fabric — for CPE, campus, out-of-band management, and embedded device roles — or provide context for how the Linux-based NOS model evolved. This section surveys them briefly.

### 8.6.1 VyOS

VyOS is a full-featured router NOS built on Debian Linux. It combines FRR for routing, StrongSwan for IPsec VPN, and a unified CLI and configuration system modeled on Vyatta (the original open-source router project). Its strength is the edge and WAN role: BGP peering with upstream providers, policy-based routing, site-to-site VPN, and NAT — capabilities that are not the focus of SONiC or SR Linux.

In AI cluster contexts, VyOS appears at the cluster boundary: the border routers that connect the GPU fabric to the corporate WAN, to cloud on-ramps, or to out-of-band management networks. It also appears in Containerlab as a fully functional virtual router when a more complete edge device is needed than FRR alone provides.

```bash
# Install VyOS (live ISO or Docker image for labs)
docker pull vyos/vyos:current

# Basic BGP peering (VyOS CLI syntax)
configure
set protocols bgp system-as 65100
set protocols bgp neighbor 10.0.0.1 remote-as 65000
set protocols bgp neighbor 10.0.0.1 address-family ipv4-unicast
commit
save
```

**Docs:** [vyos.io](https://vyos.io) · [docs.vyos.io](https://docs.vyos.io)

### 8.6.2 DENT

DENT (Disaggregated Ethernet NOS) is a Linux Foundation project targeting enterprise edge and campus switching — the access and distribution layers that connect office networks, campus buildings, and light-industrial environments. It uses the `switchdev` kernel subsystem (rather than SAI) as its hardware abstraction layer. `switchdev` is a Linux kernel framework introduced in 4.0 that allows a physical switch ASIC to be represented as a Linux network device, offloading bridge, FIB, and ACL forwarding entries to the hardware through standard netlink APIs rather than a vendor-specific user-space SDK.

DENT is not used in AI cluster core fabrics, but it is relevant for the out-of-band management networks that every AI cluster requires: the 1GbE or 10GbE switches that carry IPMI/BMC traffic, console server access, and cluster management plane communication. Running DENT on commodity hardware for these management racks aligns with the same open-NOS philosophy as the data plane.

**Docs:** [dent.dev](https://dent.dev)

### 8.6.3 OpenWrt

OpenWrt is a Linux-based NOS for embedded networking hardware — home routers, industrial gateways, and CPE devices. It uses a custom package manager (`opkg`) and a lightweight configuration framework (`UCI`). OpenWrt is not used inside AI cluster data centers, but understanding it is useful for two reasons:

1. **Historical context:** OpenWrt demonstrated that open, Linux-based software on commodity router hardware was practical, predating both SONiC and SR Linux by a decade.
2. **Lab sensors:** Out-of-band sensors, environmental monitors, and serial console servers in AI cluster deployments sometimes run OpenWrt-derived firmware. Understanding its package model and network configuration (`/etc/config/network`) helps when integrating or debugging these devices.

**Docs:** [openwrt.org](https://openwrt.org)

### 8.6.4 FreeRTOS

FreeRTOS is a lightweight RTOS (Real-Time Operating System) for microcontrollers. It provides a preemptive task scheduler, inter-task communication primitives (queues, semaphores, event groups), and a TCP/IP stack (`FreeRTOS+TCP`). It has no concept of a network operating system in the SONiC/SR Linux sense — it is the OS running on the microcontroller inside a network device's management card, fan controller, or power supply unit.

In AI cluster hardware, FreeRTOS or an equivalent RTOS frequently runs on the BMC (Baseboard Management Controller) or embedded management microcontrollers inside switches and servers. Understanding this layer matters for hardware bringup, firmware debugging, and out-of-band management tooling.

**Docs:** [freertos.org](https://www.freertos.org/Documentation/00-Overview)

### 8.6.5 Zephyr

Zephyr is a Linux Foundation RTOS for constrained and embedded devices. Unlike FreeRTOS (which started as a simple scheduler), Zephyr was designed as a full embedded OS from the start, with a device tree hardware description model, a Kconfig build system, and a broad hardware abstraction layer. It supports over 500 boards and a growing set of network protocols.

**Why Zephyr matters for AI cluster infrastructure:**

Zephyr's network subsystem (`net_core`) supports Ethernet, IPv4/IPv6, TCP, UDP, DNS, MQTT, CoAP, and LwM2M. This makes it a candidate for:

- **Out-of-band management devices** — smart PDUs, environmental sensors, and serial aggregators in the cluster rack that need a small IP stack but not a full Linux OS.
- **Custom telemetry endpoints** — rack-level temperature, humidity, and power sensors that publish metrics via MQTT or CoAP to the cluster observability pipeline.
- **NIC firmware components** — some NIC ASICs include an embedded ARM Cortex-M core that runs firmware; Zephyr's RTOS model maps naturally to this environment.

```c
/* Zephyr: minimal UDP socket example (net_core API) */
#include <zephyr/net/socket.h>

int sock = zsock_socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);

struct sockaddr_in dst = {
    .sin_family = AF_INET,
    .sin_port   = htons(9273),           /* Prometheus push gateway */
};
zsock_inet_pton(AF_INET, "10.0.0.1", &dst.sin_addr);

const char *metric = "rack_temp_celsius 42.1\n";
zsock_sendto(sock, metric, strlen(metric), 0,
             (struct sockaddr *)&dst, sizeof(dst));
zsock_close(sock);
```

Zephyr's Kconfig system allows enabling only the network protocols actually needed, keeping the firmware image small enough for devices with 256 KB of flash:

```kconfig
# prj.conf — Zephyr Kconfig for a telemetry sensor node
CONFIG_NETWORKING=y
CONFIG_NET_IPV4=y
CONFIG_NET_UDP=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_SOCKETS_POSIX_NAMES=y
CONFIG_MQTT_LIB=y

CONFIG_NET_IPV6=n          # disable unused protocols to save flash
CONFIG_NET_TCP=n
```

**Building for a Nordic nRF52840 (common IoT/sensor board):**

```bash
# Install Zephyr SDK and west tool
pip install west
west init ~/zephyrproject
cd ~/zephyrproject && west update
west zephyr-export

# Build a UDP telemetry sample
cd ~/zephyrproject/zephyr
west build -b nrf52840dk_nrf52840 samples/net/sockets/udp
west flash    # flashes to connected board
```

**Docs:** [docs.zephyrproject.org](https://docs.zephyrproject.org/latest/introduction/index.html) · [GitHub](https://github.com/zephyrproject-rtos/zephyr)

---

## Lab Walkthrough 8 — Spine-Leaf Fabric with SONiC-VS and SR Linux

This walkthrough builds a full spine-leaf BGP fabric inside Containerlab using only free, containerised NOSes. You will write the topology definition from scratch, deploy it, configure eBGP step by step on each NOS, verify ECMP, simulate a link failure with BFD, and stream telemetry via gnmic — all on a single Ubuntu 24.04 laptop or server.

**Topology:**
- 2 SR Linux spine nodes (`spine1`, `spine2`), each in AS 65000
- 2 SONiC-VS leaf nodes (`leaf1`, `leaf2`), in AS 65001 and AS 65002
- 1 FRR "host" container per leaf (`host1`, `host2`) to originate prefixes

**Prerequisites:**

```bash
# Confirm Containerlab and Docker are available
containerlab version | grep version
docker info | grep "Server Version"

# Confirm the required images are available (or will be pulled)
docker images | grep -E "srlinux|sonic-vs"

# Confirm gnmic is available
gnmic version
```

---

### Step 1 — Pull the NOS container images

SR Linux is free to download. SONiC-VS requires pulling the community image.

```bash
# SR Linux — Nokia's free container NOS
docker pull ghcr.io/nokia/srlinux:24.3.2
# Expected: pull output ending with:
# Status: Downloaded newer image for ghcr.io/nokia/srlinux:24.3.2

# SONiC-VS — virtual switch image from the sonic-net community
docker pull docker.io/sonicdev/sonic-vs:latest
# Expected: pull output ending with:
# Status: Downloaded newer image for sonicdev/sonic-vs:latest

# FRR — lightweight router container
docker pull frrouting/frr:v9.1.0
# Expected: pull output ending with:
# Status: Downloaded newer image for frrouting/frr:v9.1.0

# Verify all three images are present
docker images | grep -E "srlinux|sonic-vs|frr"
# Expected (3 lines):
# ghcr.io/nokia/srlinux      24.3.2   ...
# sonicdev/sonic-vs           latest   ...
# frrouting/frr               v9.1.0   ...
```

---

### Step 2 — Write the Containerlab topology YAML

Create a working directory and write the full topology file:

```bash
mkdir -p ~/clab-spineleaf
cd ~/clab-spineleaf
```

Save the following as `topology.yml`:

```yaml
# topology.yml — 2-spine / 2-leaf / 2-host BGP fabric
name: spineleaf

mgmt:
  network: clab-mgmt
  ipv4-subnet: 172.20.20.0/24

topology:
  nodes:
    # ── Spines (SR Linux, AS 65000) ──────────────────────────────────────
    spine1:
      kind: srl
      image: ghcr.io/nokia/srlinux:24.3.2
      mgmt-ipv4: 172.20.20.11

    spine2:
      kind: srl
      image: ghcr.io/nokia/srlinux:24.3.2
      mgmt-ipv4: 172.20.20.12

    # ── Leaves (SONiC-VS) ────────────────────────────────────────────────
    leaf1:
      kind: sonic-vs
      image: sonicdev/sonic-vs:latest
      mgmt-ipv4: 172.20.20.21

    leaf2:
      kind: sonic-vs
      image: sonicdev/sonic-vs:latest
      mgmt-ipv4: 172.20.20.22

    # ── Hosts (FRR — originate prefixes) ─────────────────────────────────
    host1:
      kind: linux
      image: frrouting/frr:v9.1.0
      mgmt-ipv4: 172.20.20.31
      env:
        DAEMONS: "zebra bgpd bfdd"

    host2:
      kind: linux
      image: frrouting/frr:v9.1.0
      mgmt-ipv4: 172.20.20.32
      env:
        DAEMONS: "zebra bgpd bfdd"

  links:
    # spine1 uplinks
    - endpoints: ["spine1:ethernet-1/1", "leaf1:Ethernet0"]
    - endpoints: ["spine1:ethernet-1/2", "leaf2:Ethernet0"]

    # spine2 uplinks
    - endpoints: ["spine2:ethernet-1/1", "leaf1:Ethernet4"]
    - endpoints: ["spine2:ethernet-1/2", "leaf2:Ethernet4"]

    # host-to-leaf downlinks
    - endpoints: ["host1:eth1", "leaf1:Ethernet8"]
    - endpoints: ["host2:eth1", "leaf2:Ethernet8"]
```

Validate the topology file before deploying:

```bash
containerlab inspect --topo topology.yml 2>/dev/null || \
containerlab graph --topo topology.yml --srv 0 &
# If graph command works, a browser-viewable SVG is generated.
# For a quick sanity check, just try deploying — clab validates YAML on deploy.
```

---

### Step 3 — Deploy the topology

```bash
cd ~/clab-spineleaf
sudo containerlab deploy --topo topology.yml
# Expected output (takes 60-120 seconds while images initialise):
# INFO[0000] Containerlab v0.55.x
# INFO[0000] Parsing & checking topology file: topology.yml
# INFO[0000] Creating lab directory: /root/clab-spineleaf
# INFO[0001] Creating docker network: Name=clab-mgmt, IPv4Subnet=172.20.20.0/24
# INFO[0002] Creating node: spine1
# INFO[0003] Creating node: spine2
# INFO[0004] Creating node: leaf1
# INFO[0005] Creating node: leaf2
# INFO[0006] Creating node: host1
# INFO[0007] Creating node: host2
# INFO[0020] Creating link: spine1:ethernet-1/1 <--> leaf1:Ethernet0
# INFO[0021] Creating link: spine1:ethernet-1/2 <--> leaf2:Ethernet0
# INFO[0022] Creating link: spine2:ethernet-1/1 <--> leaf1:Ethernet4
# INFO[0023] Creating link: spine2:ethernet-1/2 <--> leaf2:Ethernet4
# INFO[0024] Creating link: host1:eth1 <--> leaf1:Ethernet8
# INFO[0025] Creating link: host2:eth1 <--> leaf2:Ethernet8
# INFO[0090] Adding containerlab host entries to /etc/hosts file
# +----+-------------------------+----------+----------------------------+-------+---------+
# | #  | Name                    | Kind     | Image                      | State | IPv4    |
# +----+-------------------------+----------+----------------------------+-------+---------+
# |  1 | clab-spineleaf-host1    | linux    | frrouting/frr:v9.1.0      | running | 172.20.20.31 |
# |  2 | clab-spineleaf-host2    | linux    | frrouting/frr:v9.1.0      | running | 172.20.20.32 |
# |  3 | clab-spineleaf-leaf1    | sonic-vs | sonicdev/sonic-vs:latest  | running | 172.20.20.21 |
# |  4 | clab-spineleaf-leaf2    | sonic-vs | sonicdev/sonic-vs:latest  | running | 172.20.20.22 |
# |  5 | clab-spineleaf-spine1   | srlinux  | ghcr.io/nokia/srlinux:... | running | 172.20.20.11 |
# |  6 | clab-spineleaf-spine2   | srlinux  | ghcr.io/nokia/srlinux:... | running | 172.20.20.12 |
# +----+-------------------------+----------+----------------------------+-------+---------+

# Verify all six containers are running
docker ps --format "table {{.Names}}\t{{.Status}}" | grep clab
# Expected: all 6 containers show "Up X seconds" or "Up X minutes"

# Confirm the management network exists
docker network ls | grep clab-mgmt
# Expected: clab-mgmt  bridge  local
```

---

### Step 4 — Configure eBGP on SR Linux spine1

SSH into spine1 and configure it step by step. The SR Linux CLI uses a YANG-structured candidate/commit model.

```bash
# SSH into spine1 using the management address
ssh admin@172.20.20.11
# Default credentials: admin / NokiaSrl1!
# Expected prompt: Welcome to SR Linux!
#                  A:spine1#
```

Inside the SR Linux CLI:

```
# Switch to candidate configuration mode
A:spine1# enter candidate

# ── Interface configuration ──────────────────────────────────────────────
# Link to leaf1 (ethernet-1/1)
A:spine1[candidate]# set /interface ethernet-1/1 description "to leaf1"
A:spine1[candidate]# set /interface ethernet-1/1 admin-state enable
A:spine1[candidate]# set /interface ethernet-1/1 subinterface 0 admin-state enable
A:spine1[candidate]# set /interface ethernet-1/1 subinterface 0 ipv4 admin-state enable
A:spine1[candidate]# set /interface ethernet-1/1 subinterface 0 ipv4 address 10.0.1.0/31

# Link to leaf2 (ethernet-1/2)
A:spine1[candidate]# set /interface ethernet-1/2 description "to leaf2"
A:spine1[candidate]# set /interface ethernet-1/2 admin-state enable
A:spine1[candidate]# set /interface ethernet-1/2 subinterface 0 admin-state enable
A:spine1[candidate]# set /interface ethernet-1/2 subinterface 0 ipv4 admin-state enable
A:spine1[candidate]# set /interface ethernet-1/2 subinterface 0 ipv4 address 10.0.2.0/31

# Loopback for router-id
A:spine1[candidate]# set /interface lo0 admin-state enable
A:spine1[candidate]# set /interface lo0 subinterface 0 ipv4 address 10.0.0.1/32

# ── Bind interfaces to the default network-instance ──────────────────────
A:spine1[candidate]# set /network-instance default interface ethernet-1/1.0
A:spine1[candidate]# set /network-instance default interface ethernet-1/2.0
A:spine1[candidate]# set /network-instance default interface lo0.0

# ── BGP configuration ─────────────────────────────────────────────────────
A:spine1[candidate]# set /network-instance default protocols bgp router-id 10.0.0.1
A:spine1[candidate]# set /network-instance default protocols bgp autonomous-system 65000

# Create a peer-group for leaves (they are all eBGP peers)
A:spine1[candidate]# set /network-instance default protocols bgp group LEAVES \
    peer-as-local false \
    export-policy [ accept-all ] \
    import-policy [ accept-all ]

# Configure multipath (ECMP) — accept up to 4 equal-cost paths
A:spine1[candidate]# set /network-instance default protocols bgp ipv4-unicast \
    multipath max-paths-level-1 4 \
    multipath allow-multiple-as true

# Configure neighbor to leaf1
A:spine1[candidate]# set /network-instance default protocols bgp neighbor 10.0.1.1 \
    peer-group LEAVES \
    peer-as 65001 \
    description "leaf1"

# Configure neighbor to leaf2
A:spine1[candidate]# set /network-instance default protocols bgp neighbor 10.0.2.1 \
    peer-group LEAVES \
    peer-as 65002 \
    description "leaf2"

# ── Commit ────────────────────────────────────────────────────────────────
A:spine1[candidate]# commit now
# Expected:
# All changes have been committed. Waiting for next commit.
# A:spine1#
```

Verify the configuration took effect:

```
A:spine1# show interface brief
# Expected:
# +------------------+--------+-----+----------+---------------------------+
# | Interface        | Admin  | Oper| MTU      | IPv4 Address              |
# +------------------+--------+-----+----------+---------------------------+
# | ethernet-1/1.0   | enable | up  | 1500     | 10.0.1.0/31               |
# | ethernet-1/2.0   | enable | up  | 1500     | 10.0.2.0/31               |
# | lo0.0            | enable | up  | 65535    | 10.0.0.1/32               |
# +------------------+--------+-----+----------+---------------------------+

A:spine1# show network-instance default protocols bgp summary
# Expected (before leaves are configured — neighbors will show "Active"):
# BGP is enabled and up in network-instance "default"
# Global AS number  : 65000
# BGP identifier    : 10.0.0.1
# ...
# Neighbor            AS       State       ...
# 10.0.1.1            65001    Active      ...   <- waiting for leaf1
# 10.0.2.1            65002    Active      ...   <- waiting for leaf2

exit
```

Repeat the same process for `spine2` (172.20.20.12), using router-id `10.0.0.2`, loopback `10.0.0.2/32`, and link addresses `10.0.3.0/31` (to leaf1) and `10.0.4.0/31` (to leaf2).

---

### Step 5 — Configure BGP on SONiC-VS leaf1

```bash
# SSH into leaf1
ssh admin@172.20.20.21
# Default credentials: admin / YourPaSsWoRd (or blank — check image docs)
# Expected prompt: admin@leaf1:~$
```

SONiC-VS provides a `config` CLI that writes to CONFIG_DB. For BGP configuration, use `vtysh` (which accesses the FRR bgpd container running inside SONiC):

```bash
# First, configure the uplink interfaces via SONiC config CLI
sudo config interface ip add Ethernet0 10.0.1.1/31
sudo config interface ip add Ethernet4 10.0.3.1/31

# Configure the loopback
sudo config interface ip add Loopback0 10.1.0.1/32

# Verify interface config was written to CONFIG_DB
redis-cli -n 4 keys "INTERFACE|*"
# Expected:
# 1) "INTERFACE|Ethernet0|10.0.1.1/31"
# 2) "INTERFACE|Ethernet4|10.0.3.1/31"
# 3) "INTERFACE|Loopback0|10.1.0.1/32"

# Verify IP addresses are up
ip addr show Ethernet0
# Expected: inet 10.0.1.1/31 ...

ip addr show Ethernet4
# Expected: inet 10.0.3.1/31 ...
```

Now configure BGP inside vtysh (this configures the FRR bgpd container):

```bash
sudo vtysh
```

Inside vtysh:

```
leaf1# configure terminal

leaf1(config)# router bgp 65001
leaf1(config-router)#  bgp router-id 10.1.0.1
leaf1(config-router)#  bgp bestpath as-path multipath-relax
leaf1(config-router)#  maximum-paths 4

leaf1(config-router)#  neighbor SPINES peer-group
leaf1(config-router)#  neighbor SPINES remote-as external
leaf1(config-router)#  neighbor SPINES bfd

leaf1(config-router)#  neighbor 10.0.1.0 peer-group SPINES
leaf1(config-router)#  neighbor 10.0.1.0 description "spine1"
leaf1(config-router)#  neighbor 10.0.3.0 peer-group SPINES
leaf1(config-router)#  neighbor 10.0.3.0 description "spine2"

leaf1(config-router)#  address-family ipv4 unicast
leaf1(config-router-af)#   network 10.1.0.1/32
leaf1(config-router-af)#   neighbor SPINES activate
leaf1(config-router-af)#  exit-address-family
leaf1(config-router)# !
leaf1(config-router)# end

leaf1# write memory
# Expected: Note: Configuration saved to /etc/frr/frr.conf

leaf1# exit
```

Verify BGP sessions come up from leaf1's perspective:

```bash
sudo vtysh -c "show bgp summary"
# Expected (after both spines are configured):
# IPv4 Unicast Summary:
# BGP router identifier 10.1.0.1, local AS number 65001 vrf-id 0
# BGP table version 3
# RIB entries 3, using 576 bytes of memory
# Peers 2, using 46 KiB of memory
# Peer groups 1, using 64 bytes of memory
#
# Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
# 10.0.1.0        4      65000        12        12        3    0    0 00:02:30            2        2
# 10.0.3.0        4      65000        11        11        3    0    0 00:02:28            2        2
#
# Total number of neighbors 2
```

Both neighbors should show a numeric `State/PfxRcd` value (not `Active` or `Connect`). If sessions do not come up within 2 minutes, check interface reachability with `ping 10.0.1.0` from leaf1.

Repeat the configuration on `leaf2` (172.20.20.22), using AS 65002, loopback `10.1.0.2/32`, and link addresses `10.0.2.1/31` (to spine1) and `10.0.4.1/31` (to spine2).

---

### Step 6 — Verify BGP convergence and ECMP on the spines

SSH back into spine1 and verify full convergence:

```bash
ssh admin@172.20.20.11
```

Inside SR Linux:

```
A:spine1# show network-instance default protocols bgp summary
# Expected (fully converged):
# Neighbor            AS       State       Rx/Tx-Pfx  Up/Down
# 10.0.1.1            65001    Established 2/2        00:04:12
# 10.0.2.1            65002    Established 2/2        00:04:09
#
# 2 peers, 2 established

A:spine1# show network-instance default route-table ipv4-unicast
# Expected — spine1 should see both leaf loopbacks:
# Prefix          Next-hop        Protocol  Metric  Pref
# 10.0.1.0/31    local                     0       0
# 10.0.2.0/31    local                     0       0
# 10.0.3.0/31    local                     0       0    (spine2's link to leaf1)
# 10.1.0.1/32    10.0.1.1        BGP       0       170   <- leaf1 loopback via spine1
# 10.1.0.2/32    10.0.2.1        BGP       0       170   <- leaf2 loopback via spine1

# Check ECMP: spine2's loopback reachable via both leaves?
# (In a 2-spine/2-leaf topology each spine only has 1 path to each leaf loopback.
#  ECMP is visible on the leaves which have 2 spines to choose from.)

exit
```

Verify ECMP on leaf1:

```bash
ssh admin@172.20.20.21
sudo vtysh -c "show ip bgp 10.1.0.2/32"
# Expected — leaf1 should see leaf2's loopback via BOTH spines:
# BGP routing table entry for 10.1.0.2/32
# Paths: (2 available, best #1, table default)
#   Advertised to non peer-group peers:
#   10.0.1.0 10.0.3.0
#   65000 65002
#     10.0.1.0 from 10.0.1.0 (10.0.0.1)   <- via spine1
#       Origin IGP, valid, external, multipath, best (Neighbor IP)
#   65000 65002
#     10.0.3.0 from 10.0.3.0 (10.0.0.2)   <- via spine2
#       Origin IGP, valid, external, multipath

sudo vtysh -c "show ip route 10.1.0.2/32"
# Expected — two ECMP next-hops:
# B>* 10.1.0.2/32 [20/0] via 10.0.1.0, Ethernet0, weight 1, 00:03:45
#                         via 10.0.3.0, Ethernet4, weight 1, 00:03:45

# Count the paths
sudo vtysh -c "show ip route 10.1.0.2/32" | grep -c "via"
# Expected: 2
```

---

### Step 7 — Simulate a BFD link failure and time BGP withdrawal

BFD is configured between leaf1 and both spines. BFD detects link failures in ~300ms (3 × 100ms interval), which is far faster than BGP hold-timer expiry (default 90s).

First, confirm BFD sessions are active:

```bash
# On leaf1
sudo vtysh -c "show bfd peers"
# Expected:
# BFD Peers:
#     peer 10.0.1.0 vrf default
#       ID: 1
#       Remote ID: 2
#       Active mode
#       Status: up              <- IMPORTANT: must show "up" before testing
#       Uptime: 5 minutes
#       Diagnostics: ok
#       Remote diagnostics: ok
#       Peer Type: dynamic, auto-created
#       Local timers:
#         Detect-multiplier: 3
#         Receive interval: 100ms
#         Transmission interval: 100ms
#         Echo receive interval: 50ms
#       Remote timers:
#         ...
#     peer 10.0.3.0 vrf default
#       Status: up
```

Now simulate the failure. Open two terminals. In the first, watch for BGP route withdrawal. In the second, bring the link down.

**Terminal 1 — watch BGP table with timestamps:**

```bash
ssh admin@172.20.20.21
sudo vtysh
```

Inside vtysh, enable BGP debug logging:

```
leaf1# debug bgp updates
leaf1# terminal monitor
```

You should see BGP UPDATE messages as routes are received and withdrawn.

**Terminal 2 — bring down the link to spine1:**

```bash
# Find the veth interface on the host that corresponds to spine1:ethernet-1/1 <-> leaf1:Ethernet0
# Containerlab creates veth pairs named clab-xxx
ip link show | grep -E "clab.*spine1.*leaf1|clab.*leaf1.*spine1"
# Example output:
# 12: eth-spine1-1@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
# (inside the leaf1 container it's Ethernet0; on the host it's the peer veth)

# To simulate the failure, shut down the veth on the HOST side
# Get the host-side veth name for the spine1↔leaf1 link
VETH=$(sudo ip link show | grep -B1 "clab-spineleaf-leaf1" | grep -oP '\w+(?=@)' | head -1)
echo "Bringing down: $VETH"

# Record the time before bringing the link down
TIME_BEFORE=$(date +%s%3N)
sudo ip link set "$VETH" down
echo "Link down at: $(date +%T.%3N)"

# Watch for the BFD/BGP withdrawal in the bgp log
# In a real environment, watch /var/log/frr/bgpd.log or the debug terminal
```

**Terminal 1 — observe the withdrawal:**

You should see output similar to:

```
2026/04/22 10:35:42.341 BGP: [EC 100663302] 10.0.1.0(leaf1) rcv UPDATE wlen 0 attrlen 0 alen 0
2026/04/22 10:35:42.341 BGP: [EC 100663302] leaf1: withdraw 10.1.0.2/32
```

Measure the elapsed time:

```bash
TIME_AFTER=$(date +%s%3N)
echo "Detection time: $((TIME_AFTER - TIME_BEFORE)) ms"
# Expected: 200-400 ms (BFD 3x100ms detection + BGP processing)
# Compare to BGP hold-timer default: up to 90,000 ms
```

Verify the route table after failure:

```bash
sudo vtysh -c "show ip route 10.1.0.2/32"
# Expected — only ONE path remains (via spine2):
# B>* 10.1.0.2/32 [20/0] via 10.0.3.0, Ethernet4, weight 1, 00:00:08
# (the via 10.0.1.0 path through spine1 is GONE)

sudo vtysh -c "show ip route 10.1.0.2/32" | grep -c "via"
# Expected: 1   (was 2 before — one ECMP path was withdrawn)
```

Restore the link and verify ECMP returns:

```bash
sudo ip link set "$VETH" up
sleep 2
sudo vtysh -c "show ip route 10.1.0.2/32" | grep -c "via"
# Expected: 2   (both paths restored)
```

---

### Step 8 — Subscribe to SR Linux interface counters via gnmic

gnmic connects to SR Linux's built-in gNMI server (port 57400) and streams interface statistics in real time. No additional configuration on SR Linux is required — all state is YANG-modeled.

```bash
# Basic get: retrieve current counter snapshot for ethernet-1/1
gnmic -a 172.20.20.11:57400 \
    --username admin \
    --password NokiaSrl1! \
    --skip-verify \
    get \
    --path '/interface[name=ethernet-1/1]/statistics'
# Expected JSON output:
# [
#   {
#     "source": "172.20.20.11:57400",
#     "timestamp": 1745316000123456789,
#     "time": "2026-04-22T10:40:00.123Z",
#     "updates": [
#       {
#         "Path": "interface[name=ethernet-1/1]/statistics",
#         "values": {
#           "interface/statistics": {
#             "in-octets": "12345678",
#             "in-packets": "89012",
#             "in-unicast-packets": "88900",
#             "out-octets": "9876543",
#             "out-packets": "67890",
#             ...
#           }
#         }
#       }
#     ]
#   }
# ]

# Stream mode: sample every 5 seconds
gnmic -a 172.20.20.11:57400 \
    --username admin \
    --password NokiaSrl1! \
    --skip-verify \
    subscribe \
    --path '/interface[name=ethernet-1/1]/statistics/in-packets' \
    --path '/interface[name=ethernet-1/1]/statistics/out-packets' \
    --mode stream \
    --stream-mode sample \
    --sample-interval 5s
# Expected: streaming JSON notifications every 5 seconds, e.g.:
# {
#   "source": "172.20.20.11:57400",
#   "subscription-name": "default-1",
#   "timestamp": 1745316005000000000,
#   "time": "2026-04-22T10:40:05Z",
#   "updates": [
#     {"Path": "interface[name=ethernet-1/1]/statistics/in-packets",  "values": {"in-packets": "89237"}},
#     {"Path": "interface[name=ethernet-1/1]/statistics/out-packets", "values": {"out-packets": "68001"}}
#   ]
# }
# (Ctrl-C to stop)

# Subscribe to BGP neighbor state — useful for monitoring convergence events
gnmic -a 172.20.20.11:57400 \
    --username admin \
    --password NokiaSrl1! \
    --skip-verify \
    subscribe \
    --path '/network-instance[name=default]/protocols/bgp/neighbor[peer-address=10.0.1.1]/session-state' \
    --mode stream \
    --stream-mode on-change
# Expected: an event notification whenever the BGP session state changes, e.g.:
# {
#   "updates": [
#     {"Path": ".../session-state", "values": {"session-state": "established"}}
#   ]
# }
# Bring the link down in another terminal to see a "active" event appear here.
```

---

### Step 9 — Tear down the lab

```bash
cd ~/clab-spineleaf
sudo containerlab destroy --topo topology.yml
# Expected:
# INFO[0000] Destroying lab: spineleaf
# INFO[0001] Removed container: clab-spineleaf-host1
# INFO[0002] Removed container: clab-spineleaf-host2
# INFO[0003] Removed container: clab-spineleaf-leaf1
# INFO[0004] Removed container: clab-spineleaf-leaf2
# INFO[0005] Removed container: clab-spineleaf-spine1
# INFO[0006] Removed container: clab-spineleaf-spine2
# INFO[0007] Removed docker network: clab-mgmt
# INFO[0008] Removed /etc/hosts entries

# Confirm no clab containers remain
docker ps | grep clab
# Expected: (empty)
```

---

## Summary

- SONiC is the production hyperscaler NOS: containerized daemons, Redis ConfigDB, SAI ASIC abstraction, and the broadest hardware support in the open ecosystem.
- SR Linux is the programmability-first NOS: 100% YANG coverage, gNMI-native, and NDK for custom applications; the de facto Containerlab reference NOS.
- FRR provides the routing control plane (BGP, OSPF, IS-IS, EVPN) for both; its eBGP configuration is the standard underlay for AI cluster fabrics.
- BFD on top of BGP achieves sub-second failure detection, critical for minimizing AllReduce stalls caused by link failures.

---

## References

- SONiC architecture wiki: github.com/sonic-net/SONiC/wiki
- SR Linux documentation: documentation.nokia.com/srlinux
- FRR documentation: docs.frrouting.org
- SAI specification: github.com/opencomputeproject/SAI


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).