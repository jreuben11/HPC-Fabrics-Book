# Chapter 17 — BGP Tooling: BIRD, ExaBGP & GoBGP

**Part V: Management, Telemetry & Control** | ~5 pages

---

## Introduction

**BGP** (Border Gateway Protocol) is the routing protocol that holds the Internet together, but inside a modern AI cluster it plays a different and equally critical role: advertising GPU node prefixes, distributing anycast VIP reachability, and enabling programmable route injection by application-layer health-checking systems. The previous chapter on Open NOS platforms (Chapter 8) covered **FRR** — the full-featured routing suite embedded in **SONiC**, **SR Linux**, and **VyOS** — which handles **BGP** as one component of a general-purpose routing stack. This chapter addresses the three specialized scenarios where **FRR** is the wrong tool, and lighter, purpose-built **BGP** implementations are the right answer.

**BGP** in a leaf-spine AI cluster serves as the fabric's control plane: every leaf advertises its directly-connected GPU server subnets, and every spine distributes reachability among all leaves. In clusters with hundreds of leaves, a route reflector eliminates the O(N²) full-mesh **BGP** peering requirement — and **BIRD**, designed as a lightweight, high-performance routing daemon, is the standard choice for this role. **BIRD**'s compact memory footprint and fast convergence make it deployable on commodity Linux servers without dedicated routing hardware.

Beyond route reflection, the need to inject and withdraw prefixes programmatically — driven by service health, not static configuration — is a recurring pattern in AI infrastructure. Load balancer VIPs, GPU node anycast addresses, and inference endpoint prefixes all need to appear in the fabric routing table only while the corresponding services are healthy. **ExaBGP** provides a clean mechanism: it runs a user-defined process, reads **BGP** announcements from that process's stdout, and handles the **BGP** state machine without requiring the application to implement it.

For SDN controllers and programmatic orchestration, **GoBGP** exposes a **gRPC** API over its entire control plane. A Python script can query the RIB, add or withdraw prefixes, and inspect peer state — all without writing Go code or touching a configuration file. This is the natural integration point for cluster management systems that need to react to GPU node failures or topology changes.

Readers will configure **BIRD** as a route reflector, implement health-checked anycast VIP injection with **ExaBGP**, and drive **GoBGP** from Python **gRPC** stubs — building toward the autonomous fabric control patterns used in production AI clusters. This chapter connects directly to Chapter 12 (**Cilium** BGP Control Plane, which uses **BIRD** internally), Chapter 15 (**gNMI** telemetry that surfaces **BGP** state), and Chapter 16 (**Prometheus** dashboards that alert on **BGP** session drops).

---

## Installation

Four **BGP** implementations are installed to cover the distinct roles they play in AI cluster routing. **BIRD2** is deployed as the route reflector daemon — its low memory footprint and fast convergence make it the standard choice for eliminating the full-mesh **BGP** peering requirement in large leaf-spine clusters. **ExaBGP** is installed as the health-check-driven announcement engine: it runs an operator-supplied process, reads **BGP** UPDATE messages from that process's stdout, and handles the **BGP** state machine so that application-layer health checks can drive prefix advertisements without implementing the protocol directly. **GoBGP** is fetched as a standalone binary and provides a **gRPC** API over its full control plane, enabling Python-based orchestration of RIB queries, prefix injection, and peer inspection from the same cluster management scripts that respond to GPU node failures. **FRR** is included for direct comparison, as it is the routing suite embedded in **SONiC**, **SR Linux**, and **VyOS** and represents the general-purpose alternative against which the specialized tools are benchmarked.

### System packages (Ubuntu 24.04)

```bash
sudo apt update
sudo apt install -y bird2 exabgp iproute2
```

### GoBGP — binary release (recommended)

```bash
GOBGP_VER=3.28.0
wget https://github.com/osrg/gobgp/releases/download/v${GOBGP_VER}/gobgp_${GOBGP_VER}_linux_amd64.tar.gz
tar xf gobgp_${GOBGP_VER}_linux_amd64.tar.gz
sudo mv gobgp gobgpd /usr/local/bin/
gobgp --version   # gobgp version 3.28.0
gobgpd --version  # gobgpd version 3.28.0
```

### Python environment (ExaBGP scripting + GoBGP gRPC client)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.cargo/env

uv venv .venv && source .venv/bin/activate
uv pip install exabgp grpcio grpcio-tools protobuf
exabgp --version   # 4.x.y
python -c "import grpc; print('grpc', grpc.__version__)"
```

### Generate Python gRPC stubs from GoBGP's proto file

GoBGP exposes its full API over gRPC. The stubs are generated once from the upstream `.proto` file:

```bash
mkdir -p gobgp_stubs && cd gobgp_stubs

# Download gobgp.proto and its dependencies
GOBGP_VER=3.28.0
curl -sLO https://raw.githubusercontent.com/osrg/gobgp/v${GOBGP_VER}/api/gobgp.proto
curl -sLO https://raw.githubusercontent.com/osrg/gobgp/v${GOBGP_VER}/api/attribute.proto
curl -sLO https://raw.githubusercontent.com/osrg/gobgp/v${GOBGP_VER}/api/capability.proto

# Generate Python stubs
python -m grpc_tools.protoc \
    -I. \
    --python_out=. \
    --grpc_python_out=. \
    gobgp.proto attribute.proto capability.proto

ls *.py
# gobgp_pb2.py  gobgp_pb2_grpc.py  attribute_pb2.py  ...

cd ..
```

---

## 17.1 Beyond FRR: Specialized BGP Use Cases

FRR (Chapter 8) handles the full routing daemon role inside SONiC, SR Linux, and VyOS. But three scenarios call for lighter-weight, specialized BGP implementations:

1. **Route reflectors:** A cluster with 200 leaf switches need not run full routing on a dedicated appliance. BIRD can act as a route reflector cluster on commodity Linux servers.
2. **Programmatic prefix injection:** A health-checking system that adds/removes anycast VIPs based on service health needs to *speak BGP* from application code. ExaBGP enables this without implementing the full BGP state machine.
3. **SDN controllers:** A custom network controller that needs programmatic BGP control uses GoBGP's gRPC API — driven from Python without writing any Go code.

---

## 17.2 BIRD

BIRD (BIRD Internet Routing Daemon) is a fast, compact routing daemon supporting BGP, OSPF, RIP, IS-IS, and BFD. BFD (Bidirectional Forwarding Detection) is a lightweight sub-second failure-detection protocol that runs alongside routing protocols to detect link or path failures far faster than BGP hold-timer expiry allows. It is widely used as:
- Route reflector in large BGP clusters (Cilium uses it for its BGP control plane)
- BGP speaker in peering fabrics
- Test/reference implementation

### Configuration Example — Route Reflector

```
# /etc/bird/bird.conf

log syslog all;
router id 10.0.0.100;

# Define a BGP template for leaf clients
template bgp LEAVES {
    local as 65000;
    rr client;           # act as route reflector for these peers
    rr cluster id 10.0.0.100;
    ipv4 {
        import all;
        export all;
        next hop self;
    };
}

# Leaf sessions use the template
protocol bgp leaf01 from LEAVES {
    neighbor 192.168.1.1 as 65001;
}
protocol bgp leaf02 from LEAVES {
    neighbor 192.168.1.2 as 65002;
}
protocol bgp leaf03 from LEAVES {
    neighbor 192.168.1.3 as 65003;
}

# Redistribute connected routes
protocol direct {
    interface "eth*";
}
protocol kernel {
    ipv4 { export all; };
}
```

```bash
# Reload config without restart
birdc configure

# Check BGP sessions
birdc show protocols all leaf01
# leaf01   BGP  ---  up  2026-04-22  Established

# Show RIB
birdc show route
```

---

## 17.3 ExaBGP

ExaBGP is a Python-based BGP implementation designed specifically for *announcing* routes from applications, not for full routing daemon use. It executes a user-defined process and reads announcements from its stdout.

### Health-Checked Anycast with ExaBGP

Anycast is a routing technique in which the same IP address prefix is advertised from multiple physical locations simultaneously; the network routes each incoming packet to the topologically nearest advertising node. In AI infrastructure, anycast VIPs (Virtual IP addresses) are used for load balancing inference endpoints and health-checked gateway addresses — the prefix exists in the routing table only while at least one healthy backend is advertising it.

```ini
# exabgp.conf
neighbor 192.168.1.1 {
    router-id 10.0.0.200;
    local-address 10.0.0.200;
    local-as 65100;
    peer-as 65000;

    api health-check {
        processes [ healthcheck ];
    }
}

process healthcheck {
    run /usr/local/bin/health_check.py;
    encoder text;
}
```

```python
# health_check.py — ExaBGP reads this process's stdout
import subprocess
import sys
import time

ANYCAST_VIP = "10.200.0.1/32"
NEXT_HOP = "10.0.0.200"

def check_service():
    result = subprocess.run(
        ["curl", "-sf", "--max-time", "2", "http://localhost:8080/health"],
        capture_output=True
    )
    return result.returncode == 0

announced = False
while True:
    healthy = check_service()
    if healthy and not announced:
        print(f"announce route {ANYCAST_VIP} next-hop {NEXT_HOP}", flush=True)
        announced = True
    elif not healthy and announced:
        print(f"withdraw route {ANYCAST_VIP} next-hop {NEXT_HOP}", flush=True)
        announced = False
    time.sleep(5)
```

When the service is healthy, ExaBGP announces the anycast VIP to the fabric; when unhealthy, it withdraws it. Traffic automatically routes to the next healthy instance.

---

## 17.4 GoBGP — Python gRPC Client

GoBGP is a BGP daemon that exposes its entire control plane over gRPC. The `gobgpd` binary runs as the daemon; any language with gRPC support can control it. Python is the natural fit for the rest of this book's tooling.

### 17.4.1 Connecting and listing peers

```python
# gobgp_client.py — requires stubs generated in Installation
import sys
sys.path.insert(0, "gobgp_stubs")

import grpc
import gobgp_pb2 as gobgp
import gobgp_pb2_grpc as gobgp_grpc

def get_stub(host="127.0.0.1", port=50051):
    channel = grpc.insecure_channel(f"{host}:{port}")
    return gobgp_grpc.GobgpApiStub(channel)

stub = get_stub()

print("=== BGP Peers ===")
for resp in stub.ListPeer(gobgp.ListPeerRequest(enable_advertised=True)):
    p = resp.peer
    state_name = gobgp.SessionState.Name(p.state.session_state)
    print(f"  {p.conf.neighbor_address:15s}  AS{p.conf.peer_asn:<6d}  {state_name}"
          f"  rx={p.state.messages.received.total}"
          f"  pfx_rx={p.afi_safis[0].state.received if p.afi_safis else 0}")
```

### 17.4.2 Announcing and withdrawing prefixes

```python
import sys
sys.path.insert(0, "gobgp_stubs")

import grpc
import gobgp_pb2 as gobgp
import gobgp_pb2_grpc as gobgp_grpc
import attribute_pb2 as attr
from google.protobuf.any_pb2 import Any

def make_path(prefix: str, prefix_len: int, next_hop: str,
              withdraw: bool = False) -> gobgp.Path:
    nlri = Any()
    nlri.Pack(gobgp.IPAddressPrefix(prefix_len=prefix_len, prefix=prefix))

    origin = Any()
    origin.Pack(attr.OriginAttribute(origin=0))   # 0 = IGP

    nh = Any()
    nh.Pack(attr.NextHopAttribute(next_hop=next_hop))

    return gobgp.Path(
        nlri=nlri,
        pattrs=[origin, nh],
        family=gobgp.Family(
            afi=gobgp.Family.AFI_IP,
            safi=gobgp.Family.SAFI_UNICAST,
        ),
        is_withdraw=withdraw,
    )

stub = gobgp_grpc.GobgpApiStub(grpc.insecure_channel("127.0.0.1:50051"))

# Announce 10.200.0.0/24
stub.AddPath(gobgp.AddPathRequest(path=make_path("10.200.0.0", 24, "10.0.0.2")))
print("Announced 10.200.0.0/24")

# Withdraw it
stub.DeletePath(gobgp.DeletePathRequest(path=make_path("10.200.0.0", 24, "10.0.0.2", withdraw=True)))
print("Withdrew  10.200.0.0/24")
```

### 17.4.3 Reading the RIB

The RIB (Routing Information Base) is the BGP daemon's in-memory table of all learned routes, including their attributes (AS path, next-hop, local preference, community strings). The RIB is distinct from the FIB (Forwarding Information Base) installed in the kernel — the RIB contains all candidate routes, while the FIB contains only the best selected route per prefix.

```python
import sys
sys.path.insert(0, "gobgp_stubs")

import grpc
import gobgp_pb2 as gobgp
import gobgp_pb2_grpc as gobgp_grpc

stub = gobgp_grpc.GobgpApiStub(grpc.insecure_channel("127.0.0.1:50051"))

family = gobgp.Family(afi=gobgp.Family.AFI_IP, safi=gobgp.Family.SAFI_UNICAST)

print(f"{'Network':<22} {'Next Hop':<16} {'AS Path':<16} Age")
for resp in stub.ListPath(gobgp.ListPathRequest(
        table_type=gobgp.TableType.GLOBAL, family=family)):
    for d in resp.destination.paths:
        nlri = gobgp.IPAddressPrefix()
        d.nlri.Unpack(nlri)
        network = f"{nlri.prefix}/{nlri.prefix_len}"
        # Extract next-hop from pattrs
        nh = "?"
        for pa in d.pattrs:
            a = gobgp.IPAddressPrefix()   # reuse for unpack attempt
            from attribute_pb2 import NextHopAttribute
            nha = NextHopAttribute()
            if pa.Is(nha.DESCRIPTOR):
                pa.Unpack(nha); nh = nha.next_hop; break
        print(f"  {network:<20}  {nh:<16}  {d.age.ToSeconds()}s ago")
```

---

## Summary

- **BIRD** is the route reflector of choice: lightweight, fast, and used as the BGP backend in Cilium, Calico, and many bare-metal Kubernetes deployments.
- **ExaBGP** enables application-driven BGP announcements — the standard pattern for health-checked anycast VIPs and software load balancer VIP advertisement.
- **GoBGP** is a standalone BGP daemon with a gRPC API. Python gRPC stubs generated from `gobgp.proto` give full programmatic control — peer management, prefix announcement, and RIB queries — without writing any Go code.

---

## Lab Walkthrough 17 — Two-Namespace BGP Peering: BIRD as Route Reflector, GoBGP as Client, ExaBGP Health-Check Injection

This walkthrough builds a self-contained BGP lab entirely inside a single Linux host using network namespaces and a veth pair. No VMs or containers are required. All commands are run as root or with `sudo`.

### Step 1 — Create two network namespaces connected by a veth pair

```bash
# Create namespaces
sudo ip netns add ns-bird
sudo ip netns add ns-gobgp

# Create veth pair
sudo ip link add veth-bird type veth peer name veth-gobgp

# Move each end into its namespace
sudo ip link set veth-bird netns ns-bird
sudo ip link set veth-gobgp netns ns-gobgp

# Assign IP addresses (one /30 subnet: .1 = BIRD, .2 = GoBGP)
sudo ip netns exec ns-bird  ip addr add 10.0.0.1/30 dev veth-bird
sudo ip netns exec ns-gobgp ip addr add 10.0.0.2/30 dev veth-gobgp

# Bring interfaces up
sudo ip netns exec ns-bird  ip link set veth-bird up
sudo ip netns exec ns-gobgp ip link set veth-gobgp up

# Verify reachability
sudo ip netns exec ns-bird ping -c 2 10.0.0.2
```

Expected output:

```
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.067 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.058 ms

--- 10.0.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss
```

### Step 2 — Write the BIRD configuration (Route Reflector, AS 65000)

```bash
sudo mkdir -p /etc/bird

sudo tee /etc/bird/bird-ns.conf > /dev/null << 'EOF'
log "/var/log/bird-ns.log" all;
router id 10.0.0.1;

protocol device {}

protocol direct {
    ipv4;
}

protocol kernel {
    ipv4 {
        export all;
    };
}

template bgp RR_CLIENT {
    local as 65000;
    rr client;
    rr cluster id 10.0.0.1;
    ipv4 {
        import all;
        export all;
        next hop self;
    };
}

protocol bgp gobgp_peer from RR_CLIENT {
    neighbor 10.0.0.2 as 65100;
    description "GoBGP client in ns-gobgp";
}
EOF
```

### Step 3 — Start BIRD inside ns-bird

```bash
# BIRD 2 uses bird2 binary; state files go to /run/bird-ns/
sudo mkdir -p /run/bird-ns /var/log

sudo ip netns exec ns-bird bird \
    -c /etc/bird/bird-ns.conf \
    -s /run/bird-ns/bird.ctl \
    -P /run/bird-ns/bird.pid \
    -u root -g root

# Give BIRD a moment to start
sleep 1

# Verify BIRD is running
sudo ip netns exec ns-bird \
    birdc -s /run/bird-ns/bird.ctl show status
```

Expected output (abbreviated):

```
BIRD 2.15 ready.
BIRD 2.15
Router ID is 10.0.0.1
Hostname is localhost
Current server time is 2026-04-22 10:00:01.123
Last reboot on 2026-04-22 10:00:00.000
Last reconfiguration on 2026-04-22 10:00:00.001
Daemon is up and running
```

### Step 4 — Write the GoBGP configuration (AS 65100, peer to BIRD)

```bash
sudo tee /tmp/gobgp.toml > /dev/null << 'EOF'
[global.config]
  as = 65100
  router-id = "10.0.0.2"
  local-address-list = ["10.0.0.2"]

[[neighbors]]
  [neighbors.config]
    neighbor-address = "10.0.0.1"
    peer-as = 65000
  [neighbors.transport.config]
    local-address = "10.0.0.2"
EOF
```

### Step 5 — Start GoBGP daemon inside ns-gobgp and prepare Python stubs

```bash
sudo ip netns exec ns-gobgp gobgpd \
    -f /tmp/gobgp.toml \
    --log-level=info \
    > /tmp/gobgpd.log 2>&1 &

GOBGPD_PID=$!
echo "gobgpd PID: $GOBGPD_PID"

# Copy the gRPC stubs into /tmp so Python scripts inside the namespace can find them
sudo mkdir -p /tmp/gobgp_stubs
cp gobgp_stubs/*.py /tmp/gobgp_stubs/

# Wait for session establishment (BGP hold time default 90s; usually up in ~5s)
sleep 6
```

### Step 6 — Verify BGP session is Established

Check from BIRD's side:

```bash
sudo ip netns exec ns-bird \
    birdc -s /run/bird-ns/bird.ctl show protocols all gobgp_peer
```

Expected output:

```
BIRD 2.15 ready.
Name       Proto      Table      State  Since         Info
gobgp_peer BGP        ---        up     2026-04-22    Established

  BGP state:          Established
    Neighbor address: 10.0.0.2
    Neighbor AS:      65100
    Hold timer:       87.4/90
    Keepalive timer:  18.2/30
  Channel ipv4
    State:          UP
    Routes:         0 imported, 0 exported, 0 preferred
```

Check from GoBGP's side using Python gRPC:

```bash
# Copy stubs into scope for the namespace exec (write script to /tmp)
cat > /tmp/gobgp_list_peers.py << 'EOF'
import sys
sys.path.insert(0, "/tmp/gobgp_stubs")
import grpc
import gobgp_pb2 as gobgp
import gobgp_pb2_grpc as gobgp_grpc

stub = gobgp_grpc.GobgpApiStub(grpc.insecure_channel("127.0.0.1:50051"))
print(f"{'Peer':<16} {'AS':<8} {'State':<14} Rx/Tx msgs")
for resp in stub.ListPeer(gobgp.ListPeerRequest()):
    p = resp.peer
    state = gobgp.SessionState.Name(p.state.session_state)
    rx = p.state.messages.received.total
    tx = p.state.messages.sent.total
    print(f"  {p.conf.neighbor_address:<14}  AS{p.conf.peer_asn:<6}  {state:<14} {rx}/{tx}")
EOF
sudo ip netns exec ns-gobgp python3 /tmp/gobgp_list_peers.py
```

Expected output:

```
Peer             AS       State          Rx/Tx msgs
  10.0.0.1        AS65000  ESTABLISHED    4/4
```

### Step 7 — Announce a prefix from GoBGP via Python gRPC and verify BIRD receives it

```bash
cat > /tmp/gobgp_announce.py << 'EOF'
import sys
sys.path.insert(0, "/tmp/gobgp_stubs")
import grpc
import gobgp_pb2 as gobgp
import gobgp_pb2_grpc as gobgp_grpc
import attribute_pb2 as attr
from google.protobuf.any_pb2 import Any

stub = gobgp_grpc.GobgpApiStub(grpc.insecure_channel("127.0.0.1:50051"))

def announce(prefix, prefix_len, next_hop):
    nlri = Any(); nlri.Pack(gobgp.IPAddressPrefix(prefix_len=prefix_len, prefix=prefix))
    origin = Any(); origin.Pack(attr.OriginAttribute(origin=0))
    nh = Any(); nh.Pack(attr.NextHopAttribute(next_hop=next_hop))
    stub.AddPath(gobgp.AddPathRequest(path=gobgp.Path(
        nlri=nlri, pattrs=[origin, nh],
        family=gobgp.Family(afi=gobgp.Family.AFI_IP, safi=gobgp.Family.SAFI_UNICAST),
    )))
    print(f"Announced {prefix}/{prefix_len} via {next_hop}")

announce("10.200.0.0", 24, "10.0.0.2")

# Print RIB
print("\nGoBGP global RIB:")
family = gobgp.Family(afi=gobgp.Family.AFI_IP, safi=gobgp.Family.SAFI_UNICAST)
for resp in stub.ListPath(gobgp.ListPathRequest(
        table_type=gobgp.TableType.GLOBAL, family=family)):
    for path in resp.destination.paths:
        nlri = gobgp.IPAddressPrefix(); path.nlri.Unpack(nlri)
        print(f"  {nlri.prefix}/{nlri.prefix_len}")
EOF
sudo ip netns exec ns-gobgp python3 /tmp/gobgp_announce.py
```

Expected output:

```
Announced 10.200.0.0/24 via 10.0.0.2

GoBGP global RIB:
  10.200.0.0/24
```

Now verify BIRD received the route from GoBGP:

```bash
sudo ip netns exec ns-bird \
    birdc -s /run/bird-ns/bird.ctl show route all 10.200.0.0/24
```

Expected output:

```
BIRD 2.15 ready.
Table master4:
10.200.0.0/24        unicast [gobgp_peer 2026-04-22 from 10.0.0.2] * (100) [AS65100i]
	via 10.0.0.2 on veth-bird
	Type: BGP univ
	BGP.origin: IGP
	BGP.as_path:
	BGP.next_hop: 10.0.0.2
	BGP.local_pref: 100
```

### Step 8 — Announce a second prefix and verify RIB growth

```bash
cat > /tmp/gobgp_announce2.py << 'EOF'
import sys
sys.path.insert(0, "/tmp/gobgp_stubs")
import grpc
import gobgp_pb2 as gobgp
import gobgp_pb2_grpc as gobgp_grpc
import attribute_pb2 as attr
from google.protobuf.any_pb2 import Any

stub = gobgp_grpc.GobgpApiStub(grpc.insecure_channel("127.0.0.1:50051"))

def announce(prefix, prefix_len, next_hop):
    nlri = Any(); nlri.Pack(gobgp.IPAddressPrefix(prefix_len=prefix_len, prefix=prefix))
    origin = Any(); origin.Pack(attr.OriginAttribute(origin=0))
    nh = Any(); nh.Pack(attr.NextHopAttribute(next_hop=next_hop))
    stub.AddPath(gobgp.AddPathRequest(path=gobgp.Path(
        nlri=nlri, pattrs=[origin, nh],
        family=gobgp.Family(afi=gobgp.Family.AFI_IP, safi=gobgp.Family.SAFI_UNICAST),
    )))
    print(f"Announced {prefix}/{prefix_len}")

announce("10.201.0.0", 24, "10.0.0.2")

# Print full RIB
print("\nGoBGP global RIB:")
family = gobgp.Family(afi=gobgp.Family.AFI_IP, safi=gobgp.Family.SAFI_UNICAST)
for resp in stub.ListPath(gobgp.ListPathRequest(
        table_type=gobgp.TableType.GLOBAL, family=family)):
    for path in resp.destination.paths:
        nlri = gobgp.IPAddressPrefix(); path.nlri.Unpack(nlri)
        print(f"  {nlri.prefix}/{nlri.prefix_len}")
EOF
sudo ip netns exec ns-gobgp python3 /tmp/gobgp_announce2.py
```

Expected output:

```
Announced 10.201.0.0/24

GoBGP global RIB:
  10.200.0.0/24
  10.201.0.0/24
```

```bash
# Confirm both routes in BIRD
sudo ip netns exec ns-bird \
    birdc -s /run/bird-ns/bird.ctl show route
```

Expected output:

```
BIRD 2.15 ready.
Table master4:
10.200.0.0/24        unicast [gobgp_peer 2026-04-22 from 10.0.0.2] * (100) [AS65100i]
	via 10.0.0.2 on veth-bird
10.201.0.0/24        unicast [gobgp_peer 2026-04-22 from 10.0.0.2] * (100) [AS65100i]
	via 10.0.0.2 on veth-bird
```

### Step 9 — ExaBGP: configure health-check anycast injection

Save the following three files. ExaBGP will run inside ns-bird and peer to BIRD, injecting the anycast VIP `10.200.0.1/32` based on service health.

**File 1 — ExaBGP configuration:**

```bash
sudo tee /tmp/exabgp.conf > /dev/null << 'EOF'
neighbor 10.0.0.1 {
    router-id 10.0.0.3;
    local-address 10.0.0.1;
    local-as 65200;
    peer-as 65000;

    api health-check {
        processes [ healthcheck ];
    }
}

process healthcheck {
    run python3 /tmp/health_check.py;
    encoder text;
}
EOF
```

**File 2 — health_check.py (simulated; does not require a real HTTP server):**

```bash
sudo tee /tmp/health_check.py > /dev/null << 'EOF'
#!/usr/bin/env python3
"""
ExaBGP health-check process.
Reads commands from stdin (for manual control in this lab):
  - Type 'healthy' + Enter  -> announces the VIP
  - Type 'unhealthy' + Enter -> withdraws the VIP
  - Type 'quit' + Enter     -> exits
"""
import sys
import time

ANYCAST_VIP = "10.200.0.1/32"
NEXT_HOP    = "10.0.0.1"

announced = False

sys.stdout.write("# ExaBGP health_check.py started\n")
sys.stdout.flush()

# Initial state: announce immediately
sys.stdout.write(f"announce route {ANYCAST_VIP} next-hop {NEXT_HOP}\n")
sys.stdout.flush()
announced = True

for line in sys.stdin:
    cmd = line.strip().lower()
    if cmd == "healthy" and not announced:
        sys.stdout.write(f"announce route {ANYCAST_VIP} next-hop {NEXT_HOP}\n")
        sys.stdout.flush()
        announced = True
    elif cmd == "unhealthy" and announced:
        sys.stdout.write(f"withdraw route {ANYCAST_VIP} next-hop {NEXT_HOP}\n")
        sys.stdout.flush()
        announced = False
    elif cmd == "quit":
        break
EOF
chmod +x /tmp/health_check.py
```

**Add the ExaBGP peer to BIRD's config** so BIRD accepts a second BGP session:

```bash
sudo tee -a /etc/bird/bird-ns.conf > /dev/null << 'EOF'

protocol bgp exabgp_peer from RR_CLIENT {
    neighbor 10.0.0.1 as 65200;
    description "ExaBGP anycast injector (loopback peer)";
}
EOF

# Reload BIRD config live
sudo ip netns exec ns-bird \
    birdc -s /run/bird-ns/bird.ctl configure
```

Expected output:

```
BIRD 2.15 ready.
Reading configuration from /etc/bird/bird-ns.conf
Reconfigured
```

### Step 10 — Start ExaBGP and observe initial announcement

```bash
# ExaBGP runs inside ns-bird, using loopback for its peer TCP connection.
# We run it interactively in a subshell; its stdout feeds BIRD.
sudo ip netns exec ns-bird \
    env exabgp /tmp/exabgp.conf --debug 2>/tmp/exabgp.log &

EXABGP_PID=$!
echo "ExaBGP PID: $EXABGP_PID"
sleep 4

# Show BIRD RIB — should now include 10.200.0.1/32 from ExaBGP
sudo ip netns exec ns-bird \
    birdc -s /run/bird-ns/bird.ctl show route
```

Expected output (both routes present: /24 from GoBGP + /32 from ExaBGP):

```
BIRD 2.15 ready.
Table master4:
10.200.0.0/24        unicast [gobgp_peer 2026-04-22 from 10.0.0.2] * (100) [AS65100i]
	via 10.0.0.2 on veth-bird
10.200.0.1/32        unicast [exabgp_peer 2026-04-22 from 10.0.0.1] * (100) [AS65200i]
	via 10.0.0.1 on veth-bird
10.201.0.0/24        unicast [gobgp_peer 2026-04-22 from 10.0.0.2] * (100) [AS65100i]
	via 10.0.0.2 on veth-bird
```

### Step 11 — Simulate unhealthy: withdraw the VIP

Send the "unhealthy" command to ExaBGP's stdin by writing to its process (or use the ExaBGP CLI pipe). In this lab we use a named pipe:

```bash
# Trigger withdrawal (send signal to health_check.py via exabgpcli or direct stdin)
# ExaBGP exposes a CLI socket — use exabgpcli in production.
# In this lab: kill and restart with withdrawal to demonstrate:
sudo ip netns exec ns-bird \
    birdc -s /run/bird-ns/bird.ctl show route table master4 10.200.0.1/32
```

Expected output before withdrawal:

```
BIRD 2.15 ready.
Table master4:
10.200.0.1/32        unicast [exabgp_peer 2026-04-22 from 10.0.0.1] * (100) [AS65200i]
	via 10.0.0.1 on veth-bird
```

Now stop ExaBGP (simulating crash / withdrawal):

```bash
sudo kill $EXABGP_PID 2>/dev/null
sleep 3

# BIRD should have withdrawn the route via BGP session teardown
sudo ip netns exec ns-bird \
    birdc -s /run/bird-ns/bird.ctl show route
```

Expected output (10.200.0.1/32 gone; /24 routes from GoBGP remain):

```
BIRD 2.15 ready.
Table master4:
10.200.0.0/24        unicast [gobgp_peer 2026-04-22 from 10.0.0.2] * (100) [AS65100i]
	via 10.0.0.2 on veth-bird
10.201.0.0/24        unicast [gobgp_peer 2026-04-22 from 10.0.0.2] * (100) [AS65100i]
	via 10.0.0.2 on veth-bird
```

The anycast VIP has been cleanly withdrawn from the routing table.

### Step 12 — Query GoBGP RIB and peer state via Python gRPC

```bash
cat > /tmp/gobgp_status.py << 'EOF'
import sys
sys.path.insert(0, "/tmp/gobgp_stubs")
import grpc
import gobgp_pb2 as gobgp
import gobgp_pb2_grpc as gobgp_grpc

stub = gobgp_grpc.GobgpApiStub(grpc.insecure_channel("127.0.0.1:50051"))

print("=== Peers ===")
for resp in stub.ListPeer(gobgp.ListPeerRequest()):
    p = resp.peer
    state = gobgp.SessionState.Name(p.state.session_state)
    print(f"  {p.conf.neighbor_address}  AS{p.conf.peer_asn}  {state}")

print("\n=== Global RIB ===")
family = gobgp.Family(afi=gobgp.Family.AFI_IP, safi=gobgp.Family.SAFI_UNICAST)
for resp in stub.ListPath(gobgp.ListPathRequest(
        table_type=gobgp.TableType.GLOBAL, family=family)):
    for path in resp.destination.paths:
        nlri = gobgp.IPAddressPrefix(); path.nlri.Unpack(nlri)
        print(f"  {nlri.prefix}/{nlri.prefix_len}")
EOF
sudo ip netns exec ns-gobgp python3 /tmp/gobgp_status.py
```

Expected output (GoBGP's own two injected prefixes; ExaBGP's /32 lives on BIRD's RIB only):

```
=== Peers ===
  10.0.0.1  AS65000  ESTABLISHED

=== Global RIB ===
  10.200.0.0/24
  10.201.0.0/24
```

### Step 13 — Cleanup

```bash
# Stop GoBGP daemon
sudo kill $GOBGPD_PID 2>/dev/null

# Stop BIRD
sudo ip netns exec ns-bird \
    birdc -s /run/bird-ns/bird.ctl down

# Remove namespaces and veth pair (veth is removed automatically with the namespace)
sudo ip netns del ns-bird
sudo ip netns del ns-gobgp

# Verify cleanup
ip netns list   # should be empty (or not contain ns-bird / ns-gobgp)
```

---

## References

- [BIRD Internet Routing Daemon](https://bird.network.cz)
- [ExaBGP](https://github.com/Exa-Networks/exabgp)
- [GoBGP](https://osrg.github.io/gobgp/)
- [FRRouting (FRR)](https://frrouting.org)
- [Cilium BGP Control Plane (uses BIRD)](https://docs.cilium.io/en/stable/network/bgp-control-plane/)
- [gRPC](https://grpc.io/docs/)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).