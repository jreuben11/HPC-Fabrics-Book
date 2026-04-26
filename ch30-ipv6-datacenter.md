# Chapter 30 — IPv6 in AI Datacenter Fabrics

**Part I: Foundations** | ~15 pages

---

## Introduction

**IPv4** address exhaustion is not a distant concern for AI infrastructure engineers — it is an immediate, operational problem. A 10,000-GPU cluster with dual-port NICs consumes 20,000 host addresses before storage, management, BMCs, and **Kubernetes** pod CIDRs are accounted for. A single /16 **IPv4** block, historically considered generous for a datacenter, is simply too small. And the workaround — **NAT** — is incompatible with **RDMA**: **RoCEv2** and **InfiniBand** use memory registrations tied to actual IP addresses, and **NAT** breaks the transport by hiding those addresses from the remote peer.

**IPv6** was designed for this scale. A single /48 prefix, the standard site allocation, provides 65,536 /64 subnets. A /64 subnet supports 2^64 host addresses with **SLAAC** — Stateless Address Autoconfiguration — eliminating DHCP for link addresses. Every GPU node, every switch loopback, every pod gets a globally unique routable address, and **RDMA**, **NCCL**, and storage protocols see the actual peer address end-to-end with no translation in the path.

This chapter covers the full deployment of **IPv6** in an AI cluster fabric, working from first principles. Section 30.1 explains the motivating factors — address exhaustion, **NAT** incompatibility, and **SLAAC** simplicity. Section 30.2 establishes the addressing hierarchy (/48 site → /52 pod → /56 rack → /64 link → /128 loopback) that makes subnetting mechanical rather than a planning exercise. Sections 30.3–30.6 are hands-on: dual-stack **BGP** with **FRR** using **MP-BGP** address families, **EVPN**/**VXLAN** with **IPv6** Type 2 and Type 5 routes, **Cilium** dual-stack CNI configuration, and **SR Linux**'s per-subinterface **IPv6** model. Section 30.7 covers the operational gotchas unique to **IPv6** in AI clusters: **RoCEv2**'s 40-byte GRH overhead and the MTU 9040 adjustment, PTP multicast group changes, and the `NCCL_SOCKET_IFNAME`/`GLOO_SOCKET_IFNAME` interface selection problem that causes silent **IPv4** fallback.

The lab walkthrough builds a **Containerlab** dual-stack fabric with **FRR** nodes, verifies prefix exchange with `vtysh`, demonstrates **SLAAC** autoconfiguration with **radvd**, uses **Scapy** to craft and send **IPv4** and **IPv6** ICMP packets through the fabric, and runs a 2-rank **PyTorch** **Gloo** AllReduce over **IPv6** loopback to validate the training framework's **IPv6** compatibility.

This chapter is the capstone of the book's network layer coverage. It connects to Chapter 2 (**RoCEv2** and GRH overhead), Chapter 3 (PTP multicast addressing), Chapter 8 (**SONiC** and **FRR** as the **BGP** routing plane), Chapter 11 (**EVPN**/**VXLAN** fundamentals), Chapter 12 (**Cilium** CNI), and Chapter 17 (**BGP** tooling). Chapters 26–29 assume the addressing and routing model built here when discussing zero-trust, adaptive routing, resilience, and scheduling in a dual-stack environment.

## 30.1 Why IPv6 in AI Cluster Fabrics

### 30.1.1 Address Space Exhaustion at Scale

A modern AI compute grid is not a single server room — it is tens of thousands of GPU nodes, hundreds of top-of-rack switches, and a sprawling management, storage, and out-of-band fabric layered on top. A 10,000-GPU cluster with dual-port NICs already consumes 20,000 host addresses. Add storage backends, management interfaces, BMCs, Kubernetes pods, and service VIPs, and a single /16 block (65,536 addresses) is tight. Two or three data halls sharing a flat address space are impossible without heavy subnetting and NAT — both of which create operational pain.

IPv6 provides 2^128 addresses. A single /48 prefix — the standard site allocation from a regional registry — covers 65,536 /64 subnets. A single /64 subnet can address 2^64 hosts, which is every GPU node ever likely to be deployed. Address exhaustion is permanently off the table.

### 30.1.2 Simplified Subnetting

IPv4 subnetting requires careful planning to avoid wasting blocks and creating routing complexity. With IPv6, the standard is mechanically simple:

| Scope | Prefix Length | Address Count | Usage |
|---|---|---|---|
| Site / campus | /48 | 65,536 × /64 | Entire AI datacenter campus |
| Pod | /52 | 4,096 × /64 | Single compute pod (e.g. one spine-leaf domain) |
| Rack | /56 | 256 × /64 | Single top-of-rack switch domain |
| Link (P2P) | /64 | 2^64 hosts (practical: 2) | Point-to-point router links |
| Loopback | /128 | 1 | Router/node loopback identity |

The /64 boundary is baked into the IPv6 specification — it is the minimum prefix that allows SLAAC (Stateless Address Autoconfiguration). Every link gets its own /64 with no thought required.

### 30.1.3 No NAT

Network Address Translation (NAT) is the dominant workaround for IPv4 exhaustion. In AI clusters it creates specific problems:

- **RDMA cannot traverse NAT.** RoCEv2 and InfiniBand use memory registrations tied to the actual IP address. A NATed address breaks the transport.
- **Debugging is harder.** A packet capture on the wire shows the translated address, not the original. Correlating traces across NAT boundaries requires state tables.
- **Kubernetes complicates further.** Pod-to-pod communication in Kubernetes already involves overlay encapsulation; adding NAT on top doubles the translation overhead and breaks some CNI assumptions.

IPv6 eliminates NAT. Every endpoint has a globally unique, routable address. RDMA, NCCL, Gloo, and storage protocols all see the actual peer address end-to-end.

### 30.1.4 SLAAC — Stateless Address Autoconfiguration

SLAAC allows a host to configure its own IPv6 address from a Router Advertisement (RA) prefix without DHCP. The host derives a 64-bit Interface Identifier (IID) from its MAC address using EUI-64 — a standard method that inserts `ff:fe` in the middle of the 48-bit MAC and flips the universal/local bit to produce a 64-bit identifier — or a privacy-stable algorithm, appends it to the /64 prefix from the RA, and begins using the resulting address immediately.

In AI cluster fabrics, SLAAC is valuable for:
- **Management fabric:** BMCs, out-of-band switches, and management nodes auto-configure on day zero without a DHCP server.
- **Pod-to-pod CNI addressing:** Cilium and Calico can assign pod IPv6 addresses via SLAAC-derived IIDs.
- **Link-local addresses:** Every IPv6 interface automatically generates a `fe80::/10` link-local address from its MAC, enabling router-to-router BGP peering on point-to-point links without manual address configuration.

---

## Installation

**FRR** is the primary routing daemon for this chapter, providing **MP-BGP** with the `ipv6 unicast` and `l2vpn evpn` address families needed for dual-stack **BGP** underlay and **IPv6** **EVPN** Type 2 and Type 5 routes. The `iproute2` package provides the `ip -6` subcommands for inspecting neighbor discovery state, **IPv6** routing tables, and **SRv6** segment-list routes. **Containerlab** orchestrates the virtual dual-stack fabric, pulling **FRR** and **SR Linux** container images automatically so the topology can be brought up with a single command. **radvd** (Router Advertisement Daemon) is installed on Linux nodes to emit RA messages that drive **SLAAC** address assignment, demonstrating how GPU nodes self-configure their fabric addresses without a DHCP server. **Cilium** is deployed in dual-stack mode via **Helm** to validate that pod networking correctly assigns and advertises both **IPv4** and **IPv6** prefixes.

### System packages (Ubuntu 24.04)

```bash
sudo apt update
sudo apt install -y iproute2 iputils-ping ndisc6 radvd frr
```

### Containerlab

```bash
# Install Containerlab (requires Docker)
bash -c "$(curl -sL https://get.containerlab.dev)"
containerlab version
# clab version 0.5x.x
```

### Python environment

```bash
# Install uv if not present
curl -LsSf https://astral.sh/uv/install.sh | sh

uv venv .venv && source .venv/bin/activate
uv pip install scapy netaddr ipaddress
python -c "import scapy; print(scapy.__version__)"
```

### FRR (Free Range Routing)

```bash
# Ubuntu 24.04 apt includes FRR 9.x
sudo apt install -y frr frr-pythontools

# Enable the daemons you will use
sudo sed -i 's/bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo sed -i 's/zebra=no/zebra=yes/' /etc/frr/daemons
sudo systemctl enable --now frr
sudo vtysh -c "show version"
# FRRouting 9.x (Ubuntu 24.04).
```

---

## 30.2 IPv6 Addressing for Datacenter Fabrics

### 30.2.1 Prefix Hierarchy

A well-designed AI cluster IPv6 plan follows a strict hierarchy:

```
2001:db8::/32          (documentation prefix; replace with real /32 or /48 from your RIR)
└── 2001:db8:cafe::/48  Site block (one per datacenter campus)
    ├── 2001:db8:cafe:0000::/52   Pod 0  (first compute pod)
    │   ├── 2001:db8:cafe:0000::/56   Rack 0
    │   │   ├── 2001:db8:cafe:0000::/64  Link: Spine0 ↔ Leaf0
    │   │   ├── 2001:db8:cafe:0001::/64  Link: Spine0 ↔ Leaf1
    │   │   └── 2001:db8:cafe:00ff::/64  Rack 0 host subnet
    │   ├── 2001:db8:cafe:0100::/56   Rack 1
    │   └── ...
    ├── 2001:db8:cafe:1000::/52   Pod 1  (second compute pod)
    └── ...
```

### 30.2.2 Link-Local Addresses for Routing Peering

Every IPv6 interface automatically generates a `fe80::/64` link-local address the moment it comes up. BGP implementations (FRR, BIRD, SR Linux) support peering over link-local addresses, which means:

- Zero manual IP address configuration on P2P links between routers.
- Peering works immediately after the interface comes up.
- Link-local addresses are non-routable; they never appear in the forwarding table or BGP updates.

```bash
# On a FRR router, peer over link-local using the interface name as neighbor
# frr.conf excerpt:
neighbor eth0 interface remote-as external
address-family ipv6 unicast
  neighbor eth0 activate
  neighbor eth0 next-hop-self
exit-address-family
```

### 30.2.3 Loopback /128

Each router and compute node gets a unique /128 loopback address:

```
Spine 0:  2001:db8:cafe:ffff::1/128
Spine 1:  2001:db8:cafe:ffff::2/128
Leaf 0:   2001:db8:cafe:fffe::1/128
Leaf 1:   2001:db8:cafe:fffe::2/128
GPU Node 0: 2001:db8:cafe:0001::100/128
```

Loopback /128s are redistributed into BGP as host routes. They serve as stable identifiers for IBGP peering, management access, and telemetry target addresses.

### 30.2.4 Practical Address Assignment

```bash
# Assign a global unicast address to an interface
sudo ip -6 addr add 2001:db8:cafe:0000::1/64 dev eth0

# Assign a loopback /128
sudo ip -6 addr add 2001:db8:cafe:ffff::1/128 dev lo

# Verify
ip -6 addr show dev eth0
# inet6 2001:db8:cafe::1/64 scope global
# inet6 fe80::a00:27ff:fe00:1/64 scope link

ip -6 route show
# 2001:db8:cafe::/64 dev eth0 proto kernel metric 256
# fe80::/64 dev eth0 proto kernel metric 256
```

---

## 30.3 Dual-Stack BGP with FRR

### 30.3.1 Architecture

FRR supports multi-protocol BGP (MP-BGP) with simultaneous IPv4 and IPv6 address families. In a dual-stack AI cluster, a single BGP session can carry both IPv4 unicast NLRI and IPv6 unicast NLRI — two address families on one TCP connection. NLRI (Network Layer Reachability Information) is the BGP term for the set of prefixes being advertised; AFI/SAFI (Address Family Identifier / Subsequent AFI) is the two-number code that identifies the address family being carried, where AFI=1/SAFI=1 means IPv4 unicast and AFI=2/SAFI=1 means IPv6 unicast.

```
FRR Router A  ←——— eBGP session (IPv4 or IPv6 transport) ———→  FRR Router B
               carries: IPv4 Unicast AFI/SAFI (1/1)
                        IPv6 Unicast AFI/SAFI (2/1)
```

### 30.3.2 FRR Configuration: IPv4 + IPv6 Address Families

```
# /etc/frr/frr.conf on Router A (AS 65001)
frr version 9.0
frr defaults traditional
hostname router-a
log syslog informational

router bgp 65001
 bgp router-id 10.0.0.1
 neighbor 10.0.0.2 remote-as 65002
 neighbor 10.0.0.2 description "Router B (IPv4 transport)"
 !
 address-family ipv4 unicast
  neighbor 10.0.0.2 activate
  network 10.1.0.0/24
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor 10.0.0.2 activate
  network 2001:db8:cafe:1::/64
  network 2001:db8:cafe:ffff::1/128
 exit-address-family
!
```

Note: the `neighbor activate` under `address-family ipv6 unicast` is the key directive that enables IPv6 NLRI exchange on the (IPv4-transport) session. Alternatively, peer over IPv6:

```
router bgp 65001
 neighbor 2001:db8:cafe::2 remote-as 65002
 !
 address-family ipv6 unicast
  neighbor 2001:db8:cafe::2 activate
  network 2001:db8:cafe:1::/64
 exit-address-family
!
```

### 30.3.3 Verification Commands

```bash
# Enter vtysh
sudo vtysh

# Show BGP IPv4 summary
show bgp summary
# Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
# 10.0.0.2        4      65002       142       142        1    0    0 01:02:03            2

# Show BGP IPv6 summary
show bgp ipv6 unicast summary
# Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
# 10.0.0.2        4      65002       142       142        1    0    0 01:02:03            3

# Show IPv6 BGP table
show bgp ipv6 unicast
# BGP table version is 3, local router ID is 10.0.0.1, vrf id 0
# Default local pref 100, local AS 65001
# Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
#                i internal, r RIB-failure, S Stale, R Removed
# Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
# Origin codes:  i - IGP, e - EGP, ? - incomplete
# RPKI validation codes: V valid, I invalid, N Not found
#
#    Network          Next Hop            Metric LocPrf Weight Path
# *> 2001:db8:cafe:1::/64
#                    ::                       0         32768 i
# *> 2001:db8:cafe:ffff::1/128
#                    ::                       0         32768 i
# *  2001:db8:cafe:2::/64
#                    ::ffff:10.0.0.2           0             0 65002 i
```

---

## 30.4 IPv6 EVPN/VXLAN

EVPN (Ethernet VPN, RFC 7432) is a BGP address family that distributes MAC and IP reachability information across a fabric, enabling network-wide MAC learning without flooding. VXLAN (Virtual Extensible LAN) is the commonly paired data-plane encapsulation that tunnels Layer 2 frames in UDP, using a 24-bit VNI (VXLAN Network Identifier) to identify the tenant segment. A VTEP (VXLAN Tunnel Endpoint) is the device — switch, router, or host NIC — that performs the VXLAN encapsulation and decapsulation at the boundary of the overlay.

### 30.4.1 Type 2 Routes — MAC/IP Advertisement with IPv6

EVPN Type 2 routes carry MAC+IP bindings. In dual-stack deployments, a single endpoint generates two Type 2 routes: one with an IPv4 IP and one with an IPv6 IP bound to the same MAC.

```
EVPN Type 2 route:
  RD: 10.0.0.1:100
  ESI: 00:00:00:00:00:00:00:00:00:00
  Ethernet Tag ID: 0
  MAC Address Length: 48
  MAC Address: aa:bb:cc:dd:ee:ff
  IP Address Length: 128
  IP Address: 2001:db8:cafe:1::100   ← IPv6 host route
  MPLS Label1 (VNI): 10100
```

FRR EVPN configuration to advertise IPv6 MAC/IP bindings:

```
router bgp 65001
 address-family l2vpn evpn
  neighbor 10.0.0.2 activate
  advertise-all-vni
 exit-address-family
!

vrf TENANT-A
 vni 10100
!

router bgp 65001 vrf TENANT-A
 address-family ipv6 unicast
  advertise l2vpn evpn
 exit-address-family
!
```

### 30.4.2 Type 5 Routes — IPv6 Prefix Routes

Type 5 routes carry IP prefix information (not tied to a MAC). For IPv6, they carry /64 or more-specific prefixes:

```bash
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
# Route Distinguisher: 10.0.0.1:1
# *> [5]:[0]:[64]:[2001:db8:cafe:1::]/[0]:[0.0.0.0]
#                    10.0.0.1                           32768 i
#                    ET:8 RT:65001:10100 Rmac:aa:bb:cc:dd:ee:ff
```

### 30.4.3 Symmetric IRB with IPv6

IRB (Integrated Routing and Bridging) is the technique of combining L2 bridging and L3 routing at the same VTEP, allowing it to route between VXLAN-bridged segments without hairpinning traffic to a dedicated router. Symmetric IRB routes traffic through the VTEP's IP-VRF in both the ingress and egress directions, meaning both the source and destination VTEPs perform a VRF lookup:

```
Symmetric IRB packet flow (IPv6):
  VM-A (2001:db8:cafe:1::100) → sends to VM-B (2001:db8:cafe:2::200)
  VTEP-A: L3 lookup in TENANT-A VRF → next-hop is VTEP-B (EVPN Type 5)
  Encapsulate: outer IPv4/IPv6 header (VTEP-A → VTEP-B), inner VNI=10100
  VTEP-B: decapsulate, L3 lookup in TENANT-A VRF → deliver to VM-B
```

SONiC configuration for IPv6 EVPN:

```json
# config_db.json excerpt (SONiC)
"VXLAN_TUNNEL": {
    "vtep0": {
        "src_ip": "10.0.0.1"
    }
},
"VNET": {
    "Vnet1": {
        "vxlan_tunnel": "vtep0",
        "vni": "10100",
        "peer_list": ""
    }
},
"INTERFACE": {
    "Loopback0|2001:db8:cafe:ffff::1/128": {}
}
```

---

## 30.5 Cilium Dual-Stack

### 30.5.1 Enabling Dual-Stack in Cilium

Cilium 1.12+ supports native dual-stack Kubernetes clusters. Both IPv4 and IPv6 pod CIDRs are configured at install time:

```bash
helm install cilium cilium/cilium \
    --namespace kube-system \
    --set ipam.mode=cluster-pool \
    --set ipv4NativeRoutingCIDR=10.0.0.0/8 \
    --set ipv6NativeRoutingCIDR=2001:db8:cafe::/48 \
    --set ipam.operator.clusterPoolIPv4PodCIDRList[0]=10.244.0.0/16 \
    --set ipam.operator.clusterPoolIPv6PodCIDRList[0]=2001:db8:cafe:100::/56 \
    --set enableIPv6=true \
    --set enableIPv4=true \
    --set k8s.requireIPv4PodCIDR=true \
    --set k8s.requireIPv6PodCIDR=true
```

### 30.5.2 Dual-Stack Services

A Kubernetes Service with `ipFamilyPolicy: PreferDualStack` gets both IPv4 and IPv6 ClusterIPs:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nccl-training-svc
spec:
  selector:
    app: nccl-trainer
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6
  ports:
    - port: 29500
      targetPort: 29500
```

```bash
kubectl get svc nccl-training-svc
# NAME                 TYPE        CLUSTER-IP    CLUSTER-IP-V6           PORT(S)     AGE
# nccl-training-svc   ClusterIP   10.96.1.10    2001:db8:cafe:100::10   29500/TCP   2m
```

### 30.5.3 IPv6 Network Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-nccl-ipv6
spec:
  endpointSelector:
    matchLabels:
      app: nccl-trainer
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: nccl-trainer
      toPorts:
        - ports:
            - port: "29500"
              protocol: TCP
  egress:
    - toEndpoints:
        - matchLabels:
            app: nccl-trainer
      toPorts:
        - ports:
            - port: "29500"
              protocol: TCP
```

```bash
# Verify Cilium dual-stack status
cilium status | grep -E "IPv4|IPv6"
# IPv4:  Enabled (native routing CIDR: 10.0.0.0/8)
# IPv6:  Enabled (native routing CIDR: 2001:db8:cafe::/48)

# Show endpoint dual-stack addresses
cilium endpoint list | grep -v "not-ready"
# ENDPOINT   POLICY(ingress)   POLICY(egress)   IDENTITY   LABELS   IPv6                      IPv4
# 1234       Enabled           Enabled           4321       ...      2001:db8:cafe:100::a      10.244.0.10
```

---

## 30.6 SR Linux Dual-Stack

### 30.6.1 Interface IPv6 Configuration

```
# SR Linux CLI (interactive)
enter candidate

/ interface ethernet-1/1 {
    admin-state enable
    subinterface 0 {
        admin-state enable
        ipv4 {
            address 10.0.0.1/30 { }
        }
        ipv6 {
            address 2001:db8:cafe::/127 { }
            address fe80::1/64 {
                type link-local
            }
        }
    }
}

commit now
```

### 30.6.2 BGP IPv6 AFI/SAFI on SR Linux

```
enter candidate

/ network-instance default protocols bgp {
    autonomous-system 65001
    router-id 10.0.0.1
    group SPINES {
        export-policy [ export-ipv4 export-ipv6 ]
        import-policy [ import-all ]
        afi-safi ipv4-unicast { admin-state enable }
        afi-safi ipv6-unicast { admin-state enable }
    }
    neighbor 2001:db8:cafe::2 {
        peer-as 65002
        peer-group SPINES
    }
}

commit now
```

### 30.6.3 Verification

```bash
# Show IPv6 route table
show network-instance default route-table ipv6

# Sample output:
# -----------------------------------------------------------------------
# IPv6 unicast route table of network instance default
# -----------------------------------------------------------------------
# +-------------------------------+-------+------+-----------+-------+
# | Prefix                        | Proto | Pref | Next-hop  | Intrf |
# +-------------------------------+-------+------+-----------+-------+
# | 2001:db8:cafe:1::/64         | bgp   | 170  | 2001:..   | e1/1  |
# | 2001:db8:cafe:2::/64         | bgp   | 170  | 2001:..   | e1/2  |
# | 2001:db8:cafe:ffff::2/128    | bgp   | 170  | 2001:..   | e1/1  |
# +-------------------------------+-------+------+-----------+-------+

# Show BGP neighbors and their IPv6 state
show network-instance default protocols bgp neighbor * afi-safi ipv6-unicast

# Sample output:
# BGP neighbor summary for network-instance "default"
# Flags: S static, D dynamic, L discovered by LLDP, B BFD enabled, * disabled, H HOLDDOWN
# +-------------------+-----+--------+------------+-----------+---------+
# | Net-Inst          | Peer| Group  | Flags      | Peer-AS   | State   |
# +-------------------+-----+--------+------------+-----------+---------+
# | default           | 2001| SPINES |            | 65002     | established
# +-------------------+-----+--------+------------+-----------+---------+
```

---

## 30.7 IPv6 Gotchas in AI Clusters

### 30.7.1 RDMA over IPv6 — RoCEv2

RoCEv2 (RDMA over Converged Ethernet v2) is defined in the InfiniBand Architecture Supplement for RoCE v2, which explicitly supports IPv6. The GRH (Global Routing Header) in RoCEv2 carries an IPv6 source and destination address. Modern RDMA NICs (Mellanox ConnectX-5+, Broadcom Thor) all support RoCEv2 over IPv6.

Practical configuration:

```bash
# Assign IPv6 to RDMA NIC (RoCEv2 interface)
sudo ip -6 addr add 2001:db8:cafe:10::1/64 dev ens1f0

# Set RDMA CM to use IPv6
# The RDMA CM library resolves route using the kernel IPv6 routing table;
# ensure the route to the remote IPv6 address is via the RDMA NIC.
ip -6 route add 2001:db8:cafe:20::/64 via 2001:db8:cafe:10::254 dev ens1f0

# ib_send_bw test over IPv6
ib_send_bw -a -F --ib-dev mlx5_0 -6 &          # server
ib_send_bw -a -F --ib-dev mlx5_0 -6 \
    2001:db8:cafe:10::2                          # client
```

Known issue: some older verbs libraries have a path MTU discovery bug over IPv6 where the GRH adds 40 bytes to the packet overhead but the MTU calculation uses the Ethernet MTU without subtracting the GRH. Set interface MTU to 9040 instead of 9000 when using RoCEv2 over IPv6:

```bash
sudo ip link set ens1f0 mtu 9040
```

### 30.7.2 PTP over IPv6

IEEE 1588 PTP (Precision Time Protocol) supports IPv6 transport. The multicast groups change:

```
IPv4 PTP multicast groups:
  224.0.1.129  (PTP-primary)
  224.0.0.107  (PTP-pdelay)

IPv6 PTP multicast groups:
  ff0e::181    (PTP-primary, scope: global)
  ff02::6b     (PTP-pdelay, scope: link-local)
```

PTP4L configuration for IPv6:

```ini
# /etc/linuxptp/ptp4l.conf
[global]
network_transport       UDPv6
uds_address             /var/run/ptp4l
tx_timestamp_timeout    10
time_stamping           hardware

[ens1f0]
```

```bash
sudo ptp4l -f /etc/linuxptp/ptp4l.conf -m -i ens1f0
# ptp4l[1234.567]: master offset   -23 s2 freq  -1234 path delay   1456
```

### 30.7.3 NFS/RDMA over IPv6

Linux NFSv4.1+ supports RDMA transport (`proto=rdma`) over IPv6. The NFS mount syntax:

```bash
mount -t nfs4 \
    -o proto=rdma,port=20049 \
    [2001:db8:cafe:10::100]:/exports/datasets \
    /mnt/datasets
```

The square brackets around the IPv6 address are required by the NFS mount helper to distinguish the address from the colon-separated export path.

NFSD must listen on the IPv6 address:

```bash
# /etc/nfs.conf
[nfsd]
host=2001:db8:cafe:10::100
rdma=y
rdma-port=20049
```

### 30.7.4 NCCL and Gloo over IPv6

NCCL (NVIDIA Collective Communications Library) uses `NCCL_SOCKET_IFNAME` to select the network interface. When operating over IPv6, the interface name must be set correctly:

```bash
# For IPv6 training, set NCCL to use the IPv6-capable interface
export NCCL_SOCKET_IFNAME=ens1f0    # must match the interface with the IPv6 addr
export NCCL_DEBUG=INFO

# NCCL will bind to all addresses on the named interface, including IPv6
# Verify NCCL selected IPv6 by watching the INFO output for "NCCL Net":
# NCCL INFO Channel 00 : 0 [0] -> 1 [0] via P2P/IPC
# NCCL INFO NET/IB : Using [0]mlx5_0:1/RoCE - RoCE v2 (IPv6)
```

PyTorch Gloo (CPU-based collective backend) also supports IPv6; set `GLOO_SOCKET_IFNAME`:

```bash
export GLOO_SOCKET_IFNAME=ens1f0
```

---

## Lab Walkthrough 30 — Dual-Stack Containerlab Fabric with FRR BGP

This lab builds a two-node FRR fabric in Containerlab where all links carry both IPv4 and IPv6 addresses, establishes dual-stack BGP sessions, verifies route exchange, and exercises Scapy dual-stack packet generation, SLAAC, and neighbor discovery.

**Prerequisites:** Docker, Containerlab, and the Python venv from the Installation section. Approximately 2 GB RAM free.

### Step 1 — Create the Containerlab topology

```bash
mkdir -p /tmp/lab-ch30 && cd /tmp/lab-ch30

cat > dual-stack.clab.yaml << 'EOF'
name: ch30-dual-stack

topology:
  nodes:
    frr1:
      kind: linux
      image: frrouting/frr:9.0.1
      binds:
        - frr1/frr.conf:/etc/frr/frr.conf:ro
        - frr1/daemons:/etc/frr/daemons:ro
      exec:
        - ip link set eth1 up
        - ip link set eth2 up
        - ip addr add 10.0.12.1/30 dev eth1
        - ip -6 addr add 2001:db8:1::1/64 dev eth1
        - ip addr add 192.168.1.1/24 dev eth2
        - ip -6 addr add 2001:db8:cafe:1::1/64 dev eth2
        - ip addr add 10.255.0.1/32 dev lo
        - ip -6 addr add 2001:db8:ffff::1/128 dev lo
        - /usr/lib/frr/frrinit.sh start

    frr2:
      kind: linux
      image: frrouting/frr:9.0.1
      binds:
        - frr2/frr.conf:/etc/frr/frr.conf:ro
        - frr2/daemons:/etc/frr/daemons:ro
      exec:
        - ip link set eth1 up
        - ip link set eth2 up
        - ip addr add 10.0.12.2/30 dev eth1
        - ip -6 addr add 2001:db8:1::2/64 dev eth1
        - ip addr add 192.168.2.1/24 dev eth2
        - ip -6 addr add 2001:db8:cafe:2::1/64 dev eth2
        - ip addr add 10.255.0.2/32 dev lo
        - ip -6 addr add 2001:db8:ffff::2/128 dev lo
        - /usr/lib/frr/frrinit.sh start

    host1:
      kind: linux
      image: ubuntu:24.04
      exec:
        - ip link set eth1 up
        - ip addr add 192.168.1.100/24 dev eth1
        - ip -6 addr add 2001:db8:cafe:1::100/64 dev eth1
        - ip route add default via 192.168.1.1
        - ip -6 route add default via 2001:db8:cafe:1::1

  links:
    - endpoints: ["frr1:eth1", "frr2:eth1"]
    - endpoints: ["frr1:eth2", "host1:eth1"]
EOF
```

Create FRR daemons and configuration files:

```bash
mkdir -p frr1 frr2

# Daemons file (same for both nodes)
for node in frr1 frr2; do
cat > $node/daemons << 'DEOF'
zebra=yes
bgpd=yes
ospfd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=yes
fabricd=no
vrrpd=no

vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
bgpd_options="   -A 127.0.0.1"
DEOF
done
```

FRR1 configuration:

```bash
cat > frr1/frr.conf << 'EOF'
frr version 9.0
frr defaults traditional
hostname frr1
log syslog informational
ip forwarding
ipv6 forwarding

router bgp 65001
 bgp router-id 10.255.0.1
 bgp log-neighbor-changes
 !
 neighbor 10.0.12.2 remote-as 65002
 neighbor 10.0.12.2 description "frr2 (IPv4 transport)"
 neighbor 2001:db8:1::2 remote-as 65002
 neighbor 2001:db8:1::2 description "frr2 (IPv6 transport)"
 !
 address-family ipv4 unicast
  neighbor 10.0.12.2 activate
  network 10.255.0.1/32
  network 192.168.1.0/24
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor 10.0.12.2 activate
  neighbor 2001:db8:1::2 activate
  network 2001:db8:ffff::1/128
  network 2001:db8:cafe:1::/64
 exit-address-family
!

line vty
!
EOF
```

FRR2 configuration:

```bash
cat > frr2/frr.conf << 'EOF'
frr version 9.0
frr defaults traditional
hostname frr2
log syslog informational
ip forwarding
ipv6 forwarding

router bgp 65002
 bgp router-id 10.255.0.2
 bgp log-neighbor-changes
 !
 neighbor 10.0.12.1 remote-as 65001
 neighbor 10.0.12.1 description "frr1 (IPv4 transport)"
 neighbor 2001:db8:1::1 remote-as 65001
 neighbor 2001:db8:1::1 description "frr1 (IPv6 transport)"
 !
 address-family ipv4 unicast
  neighbor 10.0.12.1 activate
  network 10.255.0.2/32
  network 192.168.2.0/24
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor 10.0.12.1 activate
  neighbor 2001:db8:1::1 activate
  network 2001:db8:ffff::2/128
  network 2001:db8:cafe:2::/64
 exit-address-family
!

line vty
!
EOF
```

Deploy the lab:

```bash
sudo containerlab deploy -t dual-stack.clab.yaml
```

Expected output:

```
INFO[0000] Containerlab v0.5x.x started
INFO[0000] Parsing & checking topology file: dual-stack.clab.yaml
INFO[0000] Creating lab directory: /tmp/lab-ch30/clab-ch30-dual-stack
INFO[0001] Creating container: "frr1"
INFO[0002] Creating container: "frr2"
INFO[0003] Creating container: "host1"
INFO[0005] Creating link: frr1:eth1 <--> frr2:eth1
INFO[0005] Creating link: frr1:eth2 <--> host1:eth1
INFO[0008] 🎉 New containerlab version 0.5x.x is available!
+---+-----------------------------+----------+------------------------+-------+-------+---------+
| # | Name                        | Kind     | Image                  | State | IPv4  | IPv6    |
+---+-----------------------------+----------+------------------------+-------+-------+---------+
| 1 | clab-ch30-dual-stack-frr1   | linux    | frrouting/frr:9.0.1    | running | 172.20.20.2 | 2001:172:20:20::2 |
| 2 | clab-ch30-dual-stack-frr2   | linux    | frrouting/frr:9.0.1    | running | 172.20.20.3 | 2001:172:20:20::3 |
| 3 | clab-ch30-dual-stack-host1  | linux    | ubuntu:24.04           | running | 172.20.20.4 | 2001:172:20:20::4 |
+---+-----------------------------+----------+------------------------+-------+-------+---------+
```

### Step 2 — Configure FRR dual-stack BGP on both nodes

The frr.conf files are bind-mounted into the containers. Verify FRR is running inside each container and apply the configuration:

```bash
# Verify FRR is running inside frr1
sudo docker exec clab-ch30-dual-stack-frr1 vtysh -c "show version"
```

Expected output:

```
FRRouting 9.0.1 (frr1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

This is free software, covered by the GNU General Public License, and you are
welcome to change it and/or distribute copies of it under certain conditions.
There is no warranty for this program.

Configured with:
    --build=x86_64-pc-linux-gnu --prefix=/usr ...
    --with-pkg-git-version --with-pkg-extra-version=-MyOwnFRRVersion
```

Load the BGP configuration (already present in bind-mounted frr.conf; reload to apply any changes):

```bash
sudo docker exec clab-ch30-dual-stack-frr1 vtysh -c "configure terminal" \
    -c "router bgp 65001" \
    -c "bgp router-id 10.255.0.1" \
    -c "neighbor 10.0.12.2 remote-as 65002" \
    -c "address-family ipv4 unicast" \
    -c "neighbor 10.0.12.2 activate" \
    -c "network 10.255.0.1/32" \
    -c "exit-address-family" \
    -c "address-family ipv6 unicast" \
    -c "neighbor 10.0.12.2 activate" \
    -c "neighbor 2001:db8:1::2 activate" \
    -c "network 2001:db8:ffff::1/128" \
    -c "network 2001:db8:cafe:1::/64" \
    -c "exit-address-family" \
    -c "end" \
    -c "write memory"

sudo docker exec clab-ch30-dual-stack-frr2 vtysh -c "configure terminal" \
    -c "router bgp 65002" \
    -c "bgp router-id 10.255.0.2" \
    -c "neighbor 10.0.12.1 remote-as 65001" \
    -c "address-family ipv4 unicast" \
    -c "neighbor 10.0.12.1 activate" \
    -c "network 10.255.0.2/32" \
    -c "exit-address-family" \
    -c "address-family ipv6 unicast" \
    -c "neighbor 10.0.12.1 activate" \
    -c "neighbor 2001:db8:1::1 activate" \
    -c "network 2001:db8:ffff::2/128" \
    -c "network 2001:db8:cafe:2::/64" \
    -c "exit-address-family" \
    -c "end" \
    -c "write memory"
```

Wait 10-15 seconds for BGP sessions to establish, then proceed to Step 3.

### Step 3 — Verify both BGP sessions Established

```bash
# IPv4 BGP summary on frr1
sudo docker exec clab-ch30-dual-stack-frr1 vtysh -c "show bgp summary"
```

Expected output:

```
IPv4 Unicast Summary (VRF default):
BGP router identifier 10.255.0.1, local AS number 65001 vrf-id 0
BGP table version 4
RIB entries 6, using 1152 bytes of memory
Peers 1, using 20 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.0.12.2       4      65002        18        18        4    0    0 00:02:15            2        2 frr2 (IPv4 transport)

Total number of neighbors 1
```

```bash
# IPv6 BGP summary on frr1
sudo docker exec clab-ch30-dual-stack-frr1 vtysh -c "show bgp ipv6 unicast summary"
```

Expected output:

```
IPv6 Unicast Summary (VRF default):
BGP router identifier 10.255.0.1, local AS number 65001 vrf-id 0
BGP table version 4
RIB entries 6, using 1152 bytes of memory
Peers 2, using 40 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.0.12.2       4      65002        18        18        4    0    0 00:02:15            2        2 frr2 (IPv4 transport)
2001:db8:1::2   4      65002        12        12        4    0    0 00:01:58            2        2 frr2 (IPv6 transport)

Total number of neighbors 2
```

Both sessions show `State/PfxRcd` as a number (not `Active` or `Idle`), confirming the sessions are Established and prefixes are exchanged.

### Step 4 — Verify IPv6 prefixes exchanged

```bash
sudo docker exec clab-ch30-dual-stack-frr1 vtysh -c "show ipv6 route bgp"
```

Expected output:

```
Codes: K - kernel route, C - connected, S - static, R - RIPng,
       O - OSPFv3, I - IS-IS, B - BGP, N - NHRP, T - Table,
       v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

B>* 2001:db8:cafe:2::/64 [20/0] via fe80::a:1eff:fe02:0, eth1, weight 1, 00:02:15
B>* 2001:db8:ffff::2/128 [20/0] via fe80::a:1eff:fe02:0, eth1, weight 1, 00:02:15
```

The `B>*` prefix means BGP-learned, selected as best route, and installed in the FIB (forwarding table). Both of frr2's advertised prefixes (`2001:db8:cafe:2::/64` and `2001:db8:ffff::2/128`) appear with a valid next-hop.

### Step 5 — Write and run dual_stack_test.py with Scapy

```bash
cat > /tmp/lab-ch30/dual_stack_test.py << 'PYEOF'
#!/usr/bin/env python3
"""
dual_stack_test.py — Craft and send IPv4 ICMP and IPv6 ICMPv6 echo requests
using Scapy, verifying dual-stack connectivity through the FRR fabric.

Run inside the host1 container or on the Containerlab host with appropriate
interface access. Requires root (raw sockets).
"""

from scapy.all import (
    IP, IPv6, ICMP, ICMPv6EchoRequest, ICMPv6EchoReply,
    Ether, srp1, sr1, conf
)
import sys

# Target addresses via the FRR fabric
IPV4_TARGET = "192.168.2.1"   # frr2 eth2 (through frr1 → frr2)
IPV6_TARGET = "2001:db8:cafe:2::1"   # frr2 eth2 IPv6

IFACE = "eth1"   # interface on host1 facing frr1


def test_ipv4_icmp():
    print(f"\n[IPv4] Sending ICMP echo request to {IPV4_TARGET} via {IFACE}")
    pkt = IP(dst=IPV4_TARGET) / ICMP()
    reply = sr1(pkt, iface=IFACE, timeout=3, verbose=False)
    if reply and reply.haslayer(ICMP) and reply[ICMP].type == 0:
        print(f"  [PASS] IPv4 ICMP reply from {reply.src} "
              f"(RTT: {reply.time - pkt.sent_time:.3f}s is not available via sr1)")
        print(f"         Reply: {reply.summary()}")
        return True
    else:
        print(f"  [FAIL] No IPv4 ICMP reply received (timeout)")
        return False


def test_ipv6_icmpv6():
    print(f"\n[IPv6] Sending ICMPv6 echo request to {IPV6_TARGET} via {IFACE}")
    pkt = IPv6(dst=IPV6_TARGET) / ICMPv6EchoRequest(id=0x1234, seq=1, data=b"hello-ipv6")
    reply = sr1(pkt, iface=IFACE, timeout=3, verbose=False)
    if reply and reply.haslayer(ICMPv6EchoReply):
        print(f"  [PASS] ICMPv6 echo reply from {reply[IPv6].src}")
        print(f"         Data echoed: {reply[ICMPv6EchoReply].data}")
        return True
    else:
        print(f"  [FAIL] No ICMPv6 echo reply received (timeout)")
        return False


def craft_and_display():
    """Show the packet structure for both IPv4 and IPv6 without sending."""
    print("\n[CRAFT] IPv4 ICMP packet structure:")
    pkt4 = Ether() / IP(dst=IPV4_TARGET) / ICMP()
    pkt4.show2()

    print("\n[CRAFT] IPv6 ICMPv6 packet structure:")
    pkt6 = Ether() / IPv6(dst=IPV6_TARGET) / ICMPv6EchoRequest(id=0x1234, seq=1)
    pkt6.show2()


if __name__ == "__main__":
    # Show packet structures (no root needed for craft-only)
    craft_and_display()

    # Attempt live sends (requires root and correct interface)
    if "--live" in sys.argv:
        conf.iface = IFACE
        v4_ok = test_ipv4_icmp()
        v6_ok = test_ipv6_icmpv6()
        print(f"\nResults: IPv4={'PASS' if v4_ok else 'FAIL'}  "
              f"IPv6={'PASS' if v6_ok else 'FAIL'}")
        sys.exit(0 if (v4_ok and v6_ok) else 1)
    else:
        print("\nRun with --live to send packets (requires root, correct interface).")
PYEOF

source /tmp/lab-ch30/../.venv/bin/activate 2>/dev/null || true
cd /tmp/lab-ch30 && python3 dual_stack_test.py
```

Expected output (craft-only mode):

```
[CRAFT] IPv4 ICMP packet structure:
###[ Ethernet ]###
  dst       = ff:ff:ff:ff:ff:ff
  src       = 00:00:00:00:00:00
  type      = IPv4
###[ IP ]###
     version   = 4
     ihl       = None
     ...
     dst       = 192.168.2.1
###[ ICMP ]###
        type      = echo-request
        code      = 0
        chksum    = None
        id        = 0x0
        seq       = 0x0

[CRAFT] IPv6 ICMPv6 packet structure:
###[ Ethernet ]###
  ...
###[ IPv6 ]###
     version   = 6
     ...
     dst       = 2001:db8:cafe:2::1
###[ ICMPv6 Echo Request ]###
        type      = Echo Request
        code      = 0
        cksum     = None
        id        = 0x1234
        seq       = 0x1

Run with --live to send packets (requires root, correct interface).
```

Run with live sends from inside the host1 container:

```bash
sudo docker exec -it clab-ch30-dual-stack-host1 bash -c \
    "curl -LsSf https://astral.sh/uv/install.sh | sh && \
     ~/.local/bin/uv pip install --system scapy -q && \
     python3 /tmp/dual_stack_test.py --live"
```

Expected output:

```
[IPv4] Sending ICMP echo request to 192.168.2.1 via eth1
  [PASS] IPv4 ICMP reply from 192.168.2.1
         Reply: IP / ICMP 192.168.2.1 > 192.168.1.100 echo-reply 0

[IPv6] Sending ICMPv6 echo request to 2001:db8:cafe:2::1 via eth1
  [PASS] ICMPv6 echo reply from 2001:db8:cafe:2::1
         Data echoed: b'hello-ipv6'

Results: IPv4=PASS  IPv6=PASS
```

### Step 6 — Configure IPv6 SLAAC with radvd on a Linux host container

`radvd` (Router Advertisement Daemon) is the standard Linux daemon for sending IPv6 Router Advertisement messages as defined in RFC 4861; it periodically broadcasts the configured IPv6 prefix and router parameters on an interface, enabling attached hosts to autoconfigure their IPv6 addresses via SLAAC.

Install and configure `radvd` inside frr1 to send Router Advertisements on the eth2 link (facing host1):

```bash
sudo docker exec clab-ch30-dual-stack-frr1 bash -c "
apt-get update -qq && apt-get install -y -q radvd

cat > /etc/radvd.conf << 'RADVD'
interface eth2 {
    AdvSendAdvert on;
    MinRtrAdvInterval 3;
    MaxRtrAdvInterval 10;
    AdvManagedFlag off;
    AdvOtherConfigFlag off;

    prefix 2001:db8:cafe:1::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvRouterAddr on;
        AdvValidLifetime 86400;
        AdvPreferredLifetime 14400;
    };
};
RADVD

radvd -C /etc/radvd.conf -n &
echo 'radvd started'
"
```

Expected output:

```
radvd started
```

Now check that host1 has auto-configured an IPv6 address via SLAAC. The host derives the Interface Identifier from its MAC address (EUI-64):

```bash
sudo docker exec clab-ch30-dual-stack-host1 ip -6 addr show dev eth1
```

Expected output (the EUI-64-derived address may differ):

```
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 2001:db8:cafe:1:a8bb:ccff:fedd:ee01/64 scope global dynamic mngtmpaddr
       valid_lft 86363sec preferred_lft 14363sec
    inet6 fe80::a8bb:ccff:fedd:ee01/64 scope link
       valid_lft forever preferred_lft forever
```

The `2001:db8:cafe:1:a8bb:ccff:fedd:ee01/64` address is SLAAC-assigned — the host formed it by concatenating the /64 prefix from the RA with an EUI-64 IID derived from its MAC. The host did this with no DHCP server.

### Step 7 — Neighbor Discovery with ndisc6

The `ndisc6` package provides `rdisc6` (Router Discovery) and `ndisc6` (Neighbor Discovery/ARP equivalent for IPv6):

```bash
# Show Router Advertisement parameters received on host1's eth1
sudo docker exec clab-ch30-dual-stack-host1 bash -c \
    "apt-get install -y -q ndisc6 && rdisc6 eth1"
```

Expected output:

```
Soliciting ff02::2 (ff02::2) on eth1...

 Hop limit                 :           64 (      0x40)
 Stateful address conf.    :           No
 Stateful other conf.      :           No
 Mobile home agent         :           No
 Router preference         :       medium
 Neighbor discovery proxy  :           No
 Router lifetime           :         1800 (0x00000708) seconds
 Reachable time            :  unspecified (0x00000000)
 Retransmit time           :  unspecified (0x00000000)
 Source link-layer address : AA:BB:CC:DD:EE:01
 Prefix                    : 2001:db8:cafe:1::/64
  Valid time                :        86400 (0x00015180) seconds
  Pref. time                :        14400 (0x00003840) seconds
  On-link                   :          Yes
  Autonomous address conf.  :          Yes
 from fe80::1 (fe80::1)
```

Resolve the IPv6 address of frr1 via Neighbor Discovery (ND — the IPv6 equivalent of ARP):

```bash
sudo docker exec clab-ch30-dual-stack-host1 bash -c \
    "ndisc6 2001:db8:cafe:1::1 eth1"
```

Expected output:

```
Soliciting 2001:db8:cafe:1::1 (2001:db8:cafe:1::1) on eth1...

 Target link-layer address: AA:BB:CC:DD:EE:01

 from fe80::1
```

### Step 8 — Show IPv6 routing table and end-to-end ping

```bash
# Full IPv6 routing table on host1
sudo docker exec clab-ch30-dual-stack-host1 ip -6 route show
```

Expected output:

```
2001:db8:cafe:1::/64 dev eth1 proto kernel metric 256 expires 86362sec pref medium
2001:db8:cafe:1:a8bb:ccff:fedd:ee01/128 dev eth1 proto kernel metric 256 pref medium
fe80::/64 dev eth1 proto kernel metric 256 pref medium
default via 2001:db8:cafe:1::1 dev eth1 proto ra metric 1024 expires 1796sec pref medium
```

The default IPv6 route points to `2001:db8:cafe:1::1` (frr1's eth2 address), installed by the RA from radvd.

Ping frr2's loopback IPv6 address end-to-end (host1 → frr1 → frr2):

```bash
sudo docker exec clab-ch30-dual-stack-host1 ping6 -c 4 2001:db8:ffff::2
```

Expected output:

```
PING 2001:db8:ffff::2(2001:db8:ffff::2) 56 data bytes
64 bytes from 2001:db8:ffff::2: icmp_seq=1 ttl=63 time=0.312 ms
64 bytes from 2001:db8:ffff::2: icmp_seq=2 ttl=63 time=0.287 ms
64 bytes from 2001:db8:ffff::2: icmp_seq=3 ttl=63 time=0.301 ms
64 bytes from 2001:db8:ffff::2: icmp_seq=4 ttl=63 time=0.294 ms

--- 2001:db8:ffff::2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3072ms
rtt min/avg/max/mdev = 0.287/0.299/0.312/0.010 ms
```

The `ttl=63` (one hop decrement) confirms the packet traversed frr1 as a router hop before reaching frr2.

### Step 9 — IPv6-only pitfall: NCCL interface and PyTorch Gloo over IPv6 loopback

This step demonstrates a common misconfiguration in IPv6-only or dual-stack environments and how to work around it using a 2-rank PyTorch Gloo job.

First, demonstrate the failure mode. Without specifying the interface, NCCL and Gloo may bind to the wrong interface or fail with `NCCL_SOCKET_IFNAME` not set:

```bash
# Install PyTorch in the host1 container (CPU-only, for gloo test)
sudo docker exec clab-ch30-dual-stack-host1 bash -c \
    "curl -LsSf https://astral.sh/uv/install.sh | sh && \
     ~/.local/bin/uv pip install --system torch --index-url https://download.pytorch.org/whl/cpu -q"

# Create the 2-rank Gloo test script
sudo docker exec clab-ch30-dual-stack-host1 bash -c "cat > /tmp/ipv6_gloo_test.py << 'PYEOF'
#!/usr/bin/env python3
\"\"\"
2-rank PyTorch Gloo allreduce over IPv6 loopback (::1).
Demonstrates GLOO_SOCKET_IFNAME must be set to the interface
that has the IPv6 address being used.

For loopback (::1), the interface name is 'lo'.
\"\"\"
import os
import sys
import torch
import torch.distributed as dist

RANK   = int(os.environ.get('RANK', 0))
WORLD  = int(os.environ.get('WORLD_SIZE', 2))

# CRITICAL: Gloo needs to know which interface to bind.
# For IPv6 loopback (::1), set to 'lo'.
# For a fabric IPv6 address, set to the interface name (e.g. eth1).
GLOO_IFACE = os.environ.get('GLOO_SOCKET_IFNAME', 'lo')
os.environ['GLOO_SOCKET_IFNAME'] = GLOO_IFACE

print(f'[rank {RANK}] GLOO_SOCKET_IFNAME={GLOO_IFACE}')

# Init the process group — master addr is IPv6 loopback
dist.init_process_group(
    backend='gloo',
    init_method='tcp://[::1]:12355',
    rank=RANK,
    world_size=WORLD,
)

print(f'[rank {RANK}] Process group initialized. World size: {dist.get_world_size()}')

# Create a simple tensor: rank 0 has [1.0, 2.0], rank 1 has [3.0, 4.0]
tensor = torch.tensor([float(RANK + 1), float(RANK + 2)])
print(f'[rank {RANK}] Before allreduce: {tensor.tolist()}')

dist.all_reduce(tensor, op=dist.ReduceOp.SUM)

print(f'[rank {RANK}] After  allreduce: {tensor.tolist()}')
# Expected: [4.0, 6.0] (1+3=4, 2+4=6) for both ranks

dist.destroy_process_group()
print(f'[rank {RANK}] Done.')
PYEOF
"
```

Run the 2-rank Gloo job using IPv6 loopback `::1`:

```bash
# Launch rank 0 in background, rank 1 in foreground
sudo docker exec clab-ch30-dual-stack-host1 bash -c \
    "RANK=0 WORLD_SIZE=2 GLOO_SOCKET_IFNAME=lo python3 /tmp/ipv6_gloo_test.py &
     sleep 1
     RANK=1 WORLD_SIZE=2 GLOO_SOCKET_IFNAME=lo python3 /tmp/ipv6_gloo_test.py
     wait"
```

Expected output (interleaved from both ranks):

```
[rank 0] GLOO_SOCKET_IFNAME=lo
[rank 1] GLOO_SOCKET_IFNAME=lo
[rank 0] Process group initialized. World size: 2
[rank 1] Process group initialized. World size: 2
[rank 0] Before allreduce: [1.0, 2.0]
[rank 1] Before allreduce: [2.0, 3.0]
[rank 0] After  allreduce: [3.0, 5.0]
[rank 1] After  allreduce: [3.0, 5.0]
[rank 0] Done.
[rank 1] Done.
```

The allreduce result `[3.0, 5.0]` is the elementwise sum of `[1.0, 2.0]` (rank 0) and `[2.0, 3.0]` (rank 1). Both ranks receive the same reduced result, confirming the Gloo collective worked correctly over IPv6 loopback (`::1`).

Key takeaway: when your AI cluster uses IPv6 addresses, every collective communication library environment variable that selects a network interface — `NCCL_SOCKET_IFNAME`, `GLOO_SOCKET_IFNAME`, `NCCL_IB_HCA` — must be set to the interface name that holds the IPv6 address. The libraries do not automatically prefer IPv6 over IPv4, and getting this wrong results in a silent fallback to IPv4 or a hang.

### Step 10 — Containerlab destroy and cleanup

```bash
cd /tmp/lab-ch30

# Destroy all containers and links
sudo containerlab destroy -t dual-stack.clab.yaml --cleanup
```

Expected output:

```
INFO[0000] Parsing & checking topology file: dual-stack.clab.yaml
INFO[0000] Destroying lab: ch30-dual-stack
INFO[0001] Removed container: clab-ch30-dual-stack-frr1
INFO[0001] Removed container: clab-ch30-dual-stack-frr2
INFO[0001] Removed container: clab-ch30-dual-stack-host1
INFO[0001] Removing containerlab host entries from /etc/hosts file
INFO[0001] 🎉 Done destroying lab ch30-dual-stack!
```

```bash
# Remove lab working directory
sudo rm -rf /tmp/lab-ch30

# Verify no clab containers remain
sudo docker ps | grep clab
# (no output — all containers removed)

echo "Lab cleanup complete."
```

---

## Summary

- IPv6 solves address exhaustion at AI cluster scale; a single /48 site block provides 65,536 /64 subnets with no NAT and no DHCP required for link addresses.
- The standard hierarchy (/48 site → /52 pod → /56 rack → /64 link → /128 loopback) makes subnetting mechanical and documentation trivial.
- FRR dual-stack BGP carries both IPv4 and IPv6 NLRI on a single session using `address-family ipv6 unicast` / `neighbor activate`; `show bgp ipv6 unicast summary` verifies Established state.
- EVPN/VXLAN extends naturally to IPv6 via Type 2 MAC/IPv6 routes and Type 5 IPv6 prefix routes; symmetric IRB works identically with IPv6 L3 VNIs.
- Cilium dual-stack requires both `--ipv4-native-routing-cidr` and `--ipv6-native-routing-cidr` at install time; Services get both ClusterIPs automatically with `ipFamilyPolicy: PreferDualStack`.
- SR Linux configures IPv6 per-subinterface and enables ipv6-unicast AFI/SAFI in the BGP peer group; `show network-instance default route-table ipv6` verifies learned routes.
- RoCEv2 supports IPv6 transport natively; set interface MTU to 9040 (not 9000) to account for the 40-byte IPv6 GRH overhead.
- `NCCL_SOCKET_IFNAME` and `GLOO_SOCKET_IFNAME` must name the interface holding the IPv6 address; omitting them causes silent IPv4 fallback or hangs.

---

## References

- [RFC 4291 — IP Version 6 Addressing Architecture](https://datatracker.ietf.org/doc/html/rfc4291)
- [RFC 4862 — IPv6 Stateless Address Autoconfiguration (SLAAC)](https://datatracker.ietf.org/doc/html/rfc4862)
- [RFC 4861 — Neighbor Discovery for IPv6](https://datatracker.ietf.org/doc/html/rfc4861)
- [RFC 8200 — Internet Protocol Version 6 (IPv6) Specification](https://datatracker.ietf.org/doc/html/rfc8200)
- [RFC 7432 — BGP MPLS-Based Ethernet VPN (EVPN)](https://datatracker.ietf.org/doc/html/rfc7432)
- [FRRouting (FRR)](https://frrouting.org)
- [FRRouting documentation](https://docs.frrouting.org)
- [Cilium dual-stack documentation](https://docs.cilium.io/en/stable/network/kubernetes/ipam/)
- [SR Linux documentation](https://documentation.nokia.com/srlinux/)
- [Containerlab](https://containerlab.dev)
- [radvd — Router Advertisement Daemon](https://radvd.litech.org)
- [ndisc6 — IPv6 Neighbor Discovery tools](https://www.remlab.net/ndisc6/)
- [Scapy — Python packet crafting library](https://scapy.readthedocs.io/en/latest/)
- [NCCL documentation](https://docs.nvidia.com/deeplearning/nccl/)
- [InfiniBand Architecture Supplement for RoCE v2 — IBTA](https://www.infinibandta.org/ibta-specification/)
- [IEEE 1588-2019 Precision Time Protocol](https://standards.ieee.org/ieee/1588/6825/)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).