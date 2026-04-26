# Chapter 9 — P4 & the Programmable Data Plane

**Part III: Programmable Fabric** | ~20 pages

---

## Introduction

Every switch deployed before the mid-2010s implemented a forwarding pipeline that was cast in silicon at fabrication time: parse Ethernet, look up the MAC table, look up the IP FIB, apply access control lists, forward. Vendors could configure the tables, but the pipeline itself was immutable. This rigidity created a fundamental mismatch with the demands of AI cluster networking, where custom telemetry schemes, novel congestion signals, and in-network computation are not optional enhancements but architectural requirements.

P4 (Programming Protocol-Independent Packet Processors) dissolves this constraint. A P4 program defines the complete forwarding behavior — which headers to parse, which match-action tables to apply in which order, how to reconstruct the packet for transmission — and is compiled to a target: a software switch for testing, an FPGA for prototyping, or a programmable ASIC (such as Intel Tofino) for production. This chapter covers the full P4 toolchain: the P4₁₆ language itself, the bmv2 behavioral model for software simulation, the P4Runtime gRPC control-plane API, and two high-impact applications — in-network AllReduce aggregation and INT (In-band Network Telemetry) — that are only possible with a programmable data plane.

The chapter begins with OpenFlow, the predecessor technology that established the match-action abstraction and the software-defined networking (SDN) paradigm. Understanding OpenFlow's capabilities and limitations motivates every design decision in P4. OpenFlow's implementation in Open vSwitch (OVS) and the Ryu controller framework is demonstrated with a working L2 learning switch, before OpenFlow's fundamental limitations — fixed pipeline stages, no stateful computation, no custom header parsing — are analyzed in detail.

The reader will learn to write a complete P4₁₆ program, compile it with `p4c`, instantiate it on `bmv2`, populate its match-action tables via a Python P4Runtime controller, and verify end-to-end packet forwarding using `scapy`. The lab does not require programmable ASIC hardware — bmv2 runs on any Linux host.

This chapter connects backward to Chapter 8 (Open NOSes), which showed how SONiC and SR Linux program fixed-function ASICs via SAI, and forward to Chapter 10 (SmartNIC/DPU), where P4 programs execute on NIC ASICs at the host edge. The INT telemetry scheme developed here feeds directly into the observability pipeline covered in Chapter 16.

---

---

## Installation

The `p4c` compiler translates P4 source programs into target-specific artifacts — JSON configuration files for bmv2 and P4Info metadata files for the control plane — and is available in the p4lang PPA for Ubuntu. The `behavioral-model` package installs `simple_switch_grpc`, a software P4 switch that runs as a Linux process and accepts P4Runtime control messages over gRPC, providing a complete simulation environment without any programmable ASIC hardware. The Python packages `grpcio`, `p4runtime`, and `protobuf` are needed to write the controller that populates match-action tables via the P4Runtime API, and `scapy` is used to craft and inject test packets to verify the switch's forwarding decisions end-to-end.

### p4c Compiler

`p4c` is the open-source reference compiler for the P4 language, maintained by the P4 Language Consortium. It translates P4 programs into target-specific artifacts: JSON configuration files for bmv2, or ASIC-specific intermediate representations for hardware targets.

```bash
# p4c compiler (Ubuntu 24.04 — available in the p4lang PPA)
sudo apt install -y p4lang-p4c

# Verify
p4c --version
# p4c 1.2.x (or similar)
```

### bmv2 Behavioral Model

`bmv2` (Behavioral Model version 2) is a software implementation of a P4-programmable switch that runs as a Linux process. It is the standard reference target for developing and testing P4 programs before deploying to hardware, and supports both the V1Model and PSA P4 architectures.

```bash
# bmv2 software P4 switch
sudo apt install -y behavioral-model

# Verify
simple_switch --version
# simple_switch: 1.15.x

# If the distro packages are unavailable or too old, build from source:
# git clone https://github.com/p4lang/behavioral-model
# cd behavioral-model
# ./install_deps.sh
# ./autogen.sh
# ./configure
# make -j$(nproc)
# sudo make install
```

### Python P4Runtime Controller (via uv)

```bash
uv venv .venv
source .venv/bin/activate
uv pip install grpcio grpcio-tools protobuf p4runtime scapy

# Verify
python3 -c "import p4.v1.p4runtime_pb2; print('P4Runtime proto OK')"
python3 -c "from scapy.all import Ether; print('Scapy OK')"
```

---

## 9.1 Beyond Fixed-Function Forwarding

Every switch ASIC shipped before ~2014 implemented a fixed forwarding pipeline: parse Ethernet → look up MAC table → look up IP FIB → apply ACL → forward. Vendors could expose configuration knobs, but the fundamental pipeline was immutable.

P4 (Programming Protocol-Independent Packet Processors) inverts this. A P4 program defines the parser, the match-action tables, and the deparser — the complete forwarding behavior — and is compiled to a specific target (software switch, FPGA, or programmable ASIC). The switch does what the program says, not what the chip vendor hardcoded.

For AI cluster networking, P4 enables:
- **In-network computing:** implement AllReduce aggregation (AllReduce is the collective communication operation in which each GPU contributes a tensor and receives the element-wise sum across all contributors — the dominant communication pattern in data-parallel distributed training) in the switch ASIC, reducing the number of bytes that must traverse the fabric
- **Custom telemetry:** INT (In-band Network Telemetry) — embed per-hop latency, queue depth, and utilization metadata directly into live traffic packets as they traverse the fabric, so the receiving host can reconstruct the complete per-hop path profile for every flow
- **Custom encapsulation:** implement RDMA-aware routing, credit-based flow control, or novel congestion signals without changing the NIC

---

## 9.2 OpenFlow: The SDN Pioneer

OpenFlow (2008, Stanford Clean Slate group) was the first practical protocol for software-defined networking. It established the match-action table abstraction that every subsequent programmable network technology — including P4 — builds on. Understanding OpenFlow clarifies what P4 improves and why.

### 9.2.1 The Match-Action Model

An OpenFlow switch maintains one or more **flow tables**. Each entry is a tuple:

```
(priority, match-fields) → (actions, counters, timeout)
```

When a packet arrives, the switch finds the highest-priority entry whose match fields agree with the packet headers, then executes the associated actions. If no entry matches, the packet is sent to the controller (Packet-In) or dropped.

```
Flow Table 0 (ingress)
┌─────────┬──────────────────────────────────┬─────────────────────────┐
│Priority │ Match                            │ Actions                 │
├─────────┼──────────────────────────────────┼─────────────────────────┤
│  200    │ ip_dst=10.0.1.0/24               │ set_output(port=2)      │
│  200    │ ip_dst=10.0.2.0/24               │ set_output(port=3)      │
│  100    │ dl_type=0x0806 (ARP)             │ FLOOD                   │
│    1    │ *                                │ CONTROLLER (Packet-In)  │
└─────────┴──────────────────────────────────┴─────────────────────────┘
```

### 9.2.2 The Controller Channel

The defining innovation of OpenFlow was externalizing the control plane into a separate process — the **controller** — connected to the switch via TCP (default port 6653). The controller installs, modifies, and removes flow entries using OpenFlow protocol messages:

```
Controller                    Switch
    │                           │
    │── HELLO ──────────────────►│  capability negotiation
    │◄─ HELLO ──────────────────│
    │── FEATURES_REQUEST ───────►│
    │◄─ FEATURES_REPLY ─────────│  switch reports port count, table count
    │                           │
    │  [packet arrives, no match]│
    │◄─ PACKET_IN ──────────────│  controller sees the packet
    │── FLOW_MOD (ADD) ─────────►│  install forwarding rule
    │── PACKET_OUT ─────────────►│  forward this specific packet
```

### 9.2.3 OpenFlow with OVS and Ryu

**Open vSwitch** (OVS) is an open-source, production-quality virtual switch that runs in the Linux kernel or as a DPDK user-space process. It implements OpenFlow 1.0 through 1.5, supports hardware offload to physical NICs, and is the forwarding engine underneath Kubernetes network plugins such as OVN-Kubernetes and OpenStack Neutron. It exposes the flow table via `ovs-ofctl`. **Ryu** is a lightweight Python controller framework that connects to OVS over the OpenFlow protocol.

Install Ryu:

```bash
pip install ryu
```

A minimal Ryu controller that implements an L2 learning switch:

```python
from ryu.base import app_manager
from ryu.controller import ofp_event
from ryu.controller.handler import CONFIG_DISPATCHER, MAIN_DISPATCHER, set_ev_cls
from ryu.ofproto import ofproto_v1_3
from ryu.lib.packet import packet, ethernet

class L2Switch(app_manager.RyuApp):
    OFP_VERSIONS = [ofproto_v1_3.OFP_VERSION]

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.mac_to_port = {}   # {dpid: {mac: port}}

    @set_ev_cls(ofp_event.EventOFPSwitchFeatures, CONFIG_DISPATCHER)
    def features_handler(self, ev):
        """Install table-miss entry: send unmatched packets to controller."""
        dp = ev.msg.datapath
        ofp, parser = dp.ofproto, dp.ofproto_parser
        match = parser.OFPMatch()
        actions = [parser.OFPActionOutput(ofp.OFPP_CONTROLLER, ofp.OFPCML_NO_BUFFER)]
        self._add_flow(dp, priority=0, match=match, actions=actions)

    @set_ev_cls(ofp_event.EventOFPPacketIn, MAIN_DISPATCHER)
    def packet_in_handler(self, ev):
        msg = ev.msg
        dp  = msg.datapath
        ofp, parser = dp.ofproto, dp.ofproto_parser
        in_port = msg.match["in_port"]

        pkt = packet.Packet(msg.data)
        eth = pkt.get_protocol(ethernet.ethernet)
        dst, src, dpid = eth.dst, eth.src, dp.id

        # Learn src MAC → in_port
        self.mac_to_port.setdefault(dpid, {})[src] = in_port

        # Look up destination
        if dst in self.mac_to_port[dpid]:
            out_port = self.mac_to_port[dpid][dst]
            # Install a forwarding rule so future packets bypass the controller
            match   = parser.OFPMatch(in_port=in_port, eth_dst=dst)
            actions = [parser.OFPActionOutput(out_port)]
            self._add_flow(dp, priority=10, match=match, actions=actions)
        else:
            out_port = ofp.OFPP_FLOOD

        actions = [parser.OFPActionOutput(out_port)]
        out = parser.OFPPacketOut(
            datapath=dp, buffer_id=msg.buffer_id,
            in_port=in_port, actions=actions, data=msg.data)
        dp.send_msg(out)

    def _add_flow(self, dp, priority, match, actions, idle_timeout=10):
        ofp, parser = dp.ofproto, dp.ofproto_parser
        inst = [parser.OFPInstructionActions(ofp.OFPIT_APPLY_ACTIONS, actions)]
        mod  = parser.OFPFlowMod(
            datapath=dp, priority=priority, match=match,
            instructions=inst, idle_timeout=idle_timeout)
        dp.send_msg(mod)
```

Run against an OVS bridge in controller mode:

```bash
# Start Ryu
ryu-manager l2_switch.py &

# Point OVS bridge to Ryu controller
ovs-vsctl set-controller br0 tcp:127.0.0.1:6633

# Verify controller connection
ovs-vsctl get-controller br0
# tcp:127.0.0.1:6633

# Watch flows being installed as packets arrive
watch -n1 "ovs-ofctl dump-flows br0 --no-stats"
```

### 9.2.4 OpenFlow's Limitations — Why P4 Was Needed

OpenFlow proved the SDN concept but hit fundamental limits that motivated P4:

| Limitation | Impact | P4 Solution |
|---|---|---|
| **Fixed match fields** | Cannot match RDMA transport headers, VXLAN inner headers, or custom telemetry headers | P4 parser defines which fields exist |
| **Fixed pipeline stages** | Match order (L2 → L3 → ACL) is hardwired in the ASIC | P4 program defines the pipeline |
| **No stateful computation** | Cannot accumulate per-flow counters, implement sketches, or perform AllReduce partial sums | P4 register arrays and stateful ALUs |
| **Controller bottleneck** | Every new 5-tuple requires a controller round-trip; unacceptable at 100M flows/sec | P4 programs the ASIC to handle flows autonomously |
| **Protocol-specific** | Extensions (OpenFlow 1.1, 1.2, 1.3, 1.4, 1.5) each required re-tooling | P4 is protocol-independent by design |

### 9.2.5 SDN Controllers: ONOS and OpenDaylight

For production networks, two controllers dominate the enterprise and carrier OpenFlow deployments:

**ONOS** (Open Network Operating System) is the production OpenFlow controller used by AT&T (CORD — Central Office Re-architected as a Datacenter — an ONF platform for virtualizing telecom central offices using commodity servers and open-source networking software), Comcast, and SK Telecom. It provides a distributed, high-availability controller cluster with a northbound intent API and southbound OpenFlow/P4Runtime/NETCONF adapters.

```bash
# Run ONOS in Docker (single-node for lab use)
docker run -it --rm -p 8181:8181 -p 6653:6653 \
  onosproject/onos:2.7.0 \
  bin/onos-service start

# Access the ONOS GUI: http://localhost:8181/onos/ui
# Default credentials: onos / rocks

# CLI
docker exec -it <container> bin/onos localhost
onos> apps -a -s   # list active applications
onos> flows        # show all installed flows
```

**OpenDaylight** (ODL) is a Linux Foundation controller with a broader scope — it supports OpenFlow, NETCONF, BGP, and PCEP from a single platform. It is more commonly used in enterprise and WAN SDN than pure data-center OpenFlow.

**Relevance to AI clusters:** In modern AI cluster fabrics, OpenFlow is primarily encountered via OVN (Open Virtual Network — a logical networking layer built on top of OVS that translates high-level Kubernetes network policies and load-balancing rules into OpenFlow table entries, used by the OVN-Kubernetes CNI plugin) and via Cilium's eBPF dataplane (which provides equivalent programmability without a controller channel). For in-network AI computing (AllReduce aggregation, INT telemetry), P4 on ASIC targets is the correct tool — OpenFlow cannot express stateful accumulation.

---

## 9.3 P4₁₆ Language

### 9.3.1 Architecture Model

A P4 program targets an **architecture**: an abstract model of a programmable pipeline. The most common is V1Model (used by `bmv2`):

```
ingress_port → [Parser] → [Ingress Pipeline] → [Traffic Manager] → [Egress Pipeline] → [Deparser] → egress_port
```

### 9.3.2 Headers and Metadata

```p4
// Header type definitions
header ethernet_t {
    bit<48> dst_addr;
    bit<48> src_addr;
    bit<16> ether_type;
}

header ipv4_t {
    bit<4>  version;
    bit<4>  ihl;
    bit<8>  diffserv;
    bit<16> total_len;
    bit<16> id;
    bit<3>  flags;
    bit<13> frag_offset;
    bit<8>  ttl;
    bit<8>  protocol;
    bit<16> hdr_checksum;
    bit<32> src_addr;
    bit<32> dst_addr;
}

// Header stack — the set of headers a packet may carry
struct headers_t {
    ethernet_t ethernet;
    ipv4_t     ipv4;
}

// Per-packet metadata
struct metadata_t {
    bit<9>  egress_port;
    bit<1>  drop;
}
```

### 9.3.3 Parser

```p4
parser MyParser(packet_in pkt,
                out headers_t hdr,
                inout metadata_t meta,
                inout standard_metadata_t std_meta) {

    state start {
        pkt.extract(hdr.ethernet);
        transition select(hdr.ethernet.ether_type) {
            0x0800:  parse_ipv4;
            default: accept;
        }
    }

    state parse_ipv4 {
        pkt.extract(hdr.ipv4);
        transition accept;
    }
}
```

### 9.3.4 Match-Action Tables

```p4
control MyIngress(inout headers_t hdr,
                  inout metadata_t meta,
                  inout standard_metadata_t std_meta) {

    // Action definitions
    action set_nhop(bit<9> port) {
        std_meta.egress_spec = port;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }

    action drop() {
        mark_to_drop(std_meta);
    }

    // Match-action table: exact match on dst IP
    table ipv4_lpm {
        key = {
            hdr.ipv4.dst_addr: lpm;   // longest prefix match
        }
        actions = {
            set_nhop;
            drop;
            NoAction;
        }
        size = 1024;
        default_action = drop;
    }

    apply {
        if (hdr.ipv4.isValid()) {
            ipv4_lpm.apply();
        }
    }
}
```

### 9.3.5 Deparser

```p4
control MyDeparser(packet_out pkt, in headers_t hdr) {
    apply {
        pkt.emit(hdr.ethernet);
        pkt.emit(hdr.ipv4);
    }
}
```

---

## 9.4 Targets

### 9.4.1 bmv2 — Behavioral Model v2

`bmv2` is a software P4 switch — a reference implementation for development and testing. It runs as a Linux process and accepts P4Runtime control messages.

```bash
# Compile P4 program for bmv2 target
p4c --target bmv2 --arch v1model -o build/ my_switch.p4

# Run bmv2 with two virtual interfaces
sudo simple_switch \
    --interface 0@veth0 \
    --interface 1@veth1 \
    build/my_switch.json \
    --nanolog ipc:///tmp/bmv2.ipc

# Populate tables via P4Runtime Python API
python3 controller.py
```

### 9.4.2 Intel Tofino — Programmable ASIC

Tofino (Barefoot, now Intel) is the first commercially successful programmable ASIC. It implements the TNA (Tofino Native Architecture) and achieves 6.4 Tbps at 100% match-action table utilization.

```p4
// TNA-specific annotations
#include <core.p4>
#include <tna.p4>

@pragma pa_atomic ingress hdr.ipv4.src_addr
// ... Tofino-specific constraints on stateful ALUs, register arrays
```

Tofino's stateful ALUs enable in-switch computation: register arrays that accumulate per-flow counters, implement sketches, or perform partial AllReduce aggregation.

---

## 9.5 P4Runtime — Control Plane API

P4Runtime is a gRPC API for managing table entries in a running P4 target:

```protobuf
// p4/v1/p4runtime.proto (simplified)
service P4Runtime {
    rpc Write(WriteRequest) returns (WriteResponse);
    rpc Read(ReadRequest) returns (stream ReadResponse);
    rpc StreamChannel(stream StreamMessageRequest)
        returns (stream StreamMessageResponse);
}

message TableEntry {
    uint32 table_id = 1;
    repeated FieldMatch match = 2;
    TableAction action = 4;
}
```

Python controller example:

```python
from p4.v1 import p4runtime_pb2
from p4.v1 import p4runtime_pb2_grpc

channel = grpc.insecure_channel('localhost:9559')
stub = p4runtime_pb2_grpc.P4RuntimeStub(channel)

# Build a table entry: match 10.0.0.0/24 → set_nhop(port=1)
entry = p4runtime_pb2.TableEntry()
entry.table_id = 33574068          # ID from P4Info
entry.match.add()                  # LPM match field
entry.match[0].field_id = 1
entry.match[0].lpm.value = b'\x0a\x00\x00\x00'   # 10.0.0.0
entry.match[0].lpm.prefix_len = 24
entry.action.action.action_id = 23766285   # set_nhop action ID
entry.action.action.params.add()
entry.action.action.params[0].param_id = 1
entry.action.action.params[0].value = b'\x00\x01'  # port 1

update = p4runtime_pb2.Update()
update.type = p4runtime_pb2.Update.INSERT
update.entity.table_entry.CopyFrom(entry)

stub.Write(p4runtime_pb2.WriteRequest(
    device_id=0,
    updates=[update]
))
```

---

## 9.6 In-Network Computing for AI

The most exciting P4 application for AI clusters is **in-network aggregation**: performing AllReduce partial sums inside the switch ASIC, reducing the traffic that must traverse the full fabric.

### SHARP (Scalable Hierarchical Aggregation and Reduction Protocol)

SHARP is NVIDIA's in-network computing technology for InfiniBand fabrics that offloads AllReduce collective operations to the switch ASIC. Instead of routing gradient tensors from every GPU through a reduction tree back to every GPU, SHARP-equipped switches perform partial sums on the data as it transits through the fabric, reducing the total bytes transferred across the network by up to a factor of N (the number of ranks). NVIDIA's SHARP implements in-network AllReduce on InfiniBand switches. The concept (implementable in P4 on Tofino):

```
Without SHARP:
  Rank0 → Rank1 → (sum) → Rank2 → (sum) → ... → final → broadcast

With in-switch aggregation:
  All ranks → switch (sums while forwarding) → result → broadcast
  Network traffic: O(N × size) → O(size) per switch hop
```

A P4 implementation sketch:

```p4
// Stateful register array for partial sums
Register<bit<32>, bit<16>>(65536) partial_sums;

RegisterAction<bit<32>, bit<16>, bit<32>>(partial_sums) accumulate = {
    void apply(inout bit<32> value, out bit<32> result) {
        value = value + hdr.allreduce.gradient;
        result = value;
    }
};

action do_aggregate() {
    bit<32> new_sum = accumulate.execute(hdr.allreduce.tensor_id);
    hdr.allreduce.gradient = new_sum;
    // If all expected contributions received: forward result
    // Otherwise: drop (contribution absorbed into register)
}
```

### INT — In-Band Network Telemetry

INT embeds per-switch telemetry (queue depth, ingress timestamp, egress timestamp, switch ID) into live data packets as they traverse the fabric:

```p4
header int_hop_t {
    bit<32> switch_id;
    bit<32> ingress_timestamp;
    bit<32> egress_timestamp;
    bit<19> dequeue_qdepth;
    bit<13> padding;
}

action add_int_hop() {
    hdr.int_stack.push_front(1);
    hdr.int_stack[0].switch_id = SWITCH_ID;
    hdr.int_stack[0].ingress_timestamp = (bit<32>)standard_metadata.ingress_global_timestamp;
    hdr.int_stack[0].dequeue_qdepth = standard_metadata.deq_qdepth;
}
```

The receiving host extracts the INT stack and exports telemetry to Prometheus/Grafana, providing per-hop latency visibility for every NCCL flow.

---

## Lab Walkthrough 9 — P4 Router on bmv2 with P4Runtime Controller

This walkthrough builds a complete P4 IPv4 router running in software using bmv2, controlled via the P4Runtime API, with two Linux network namespaces acting as hosts.

### Step 1 — Write the Full router.p4 Program

Create `router.p4`:

```p4
/* router.p4 — IPv4 LPM forwarding with packet counter */
#include <core.p4>
#include <v1model.p4>

/* ── Headers ──────────────────────────────────────────── */
header ethernet_t {
    bit<48> dst_addr;
    bit<48> src_addr;
    bit<16> ether_type;
}

header ipv4_t {
    bit<4>  version;
    bit<4>  ihl;
    bit<8>  diffserv;
    bit<16> total_len;
    bit<16> id;
    bit<3>  flags;
    bit<13> frag_offset;
    bit<8>  ttl;
    bit<8>  protocol;
    bit<16> hdr_checksum;
    bit<32> src_addr;
    bit<32> dst_addr;
}

struct headers_t {
    ethernet_t ethernet;
    ipv4_t     ipv4;
}

struct metadata_t { }

/* ── Parser ───────────────────────────────────────────── */
parser MyParser(packet_in pkt,
                out headers_t hdr,
                inout metadata_t meta,
                inout standard_metadata_t std_meta) {
    state start {
        pkt.extract(hdr.ethernet);
        transition select(hdr.ethernet.ether_type) {
            0x0800: parse_ipv4;
            default: accept;
        }
    }
    state parse_ipv4 {
        pkt.extract(hdr.ipv4);
        transition accept;
    }
}

/* ── Ingress Pipeline ─────────────────────────────────── */
control MyIngress(inout headers_t hdr,
                  inout metadata_t meta,
                  inout standard_metadata_t std_meta) {

    counter(1024, CounterType.packets_and_bytes) pkt_counter;

    action set_nhop(bit<9> port, bit<48> dst_mac) {
        std_meta.egress_spec = port;
        hdr.ethernet.dst_addr = dst_mac;
        hdr.ipv4.ttl          = hdr.ipv4.ttl - 1;
    }

    action _drop() {
        mark_to_drop(std_meta);
    }

    table ipv4_lpm {
        key     = { hdr.ipv4.dst_addr: lpm; }
        actions = { set_nhop; _drop; NoAction; }
        size    = 1024;
        default_action = _drop();
    }

    apply {
        if (hdr.ipv4.isValid()) {
            pkt_counter.count((bit<32>)std_meta.ingress_port);
            ipv4_lpm.apply();
        }
    }
}

/* ── Egress Pipeline (pass-through) ──────────────────── */
control MyEgress(inout headers_t hdr,
                 inout metadata_t meta,
                 inout standard_metadata_t std_meta) {
    apply { }
}

/* ── Checksum Verification ───────────────────────────── */
control MyVerifyChecksum(inout headers_t hdr,
                         inout metadata_t meta) {
    apply {
        verify_checksum(
            hdr.ipv4.isValid(),
            { hdr.ipv4.version, hdr.ipv4.ihl, hdr.ipv4.diffserv,
              hdr.ipv4.total_len, hdr.ipv4.id, hdr.ipv4.flags,
              hdr.ipv4.frag_offset, hdr.ipv4.ttl, hdr.ipv4.protocol,
              hdr.ipv4.src_addr, hdr.ipv4.dst_addr },
            hdr.ipv4.hdr_checksum,
            HashAlgorithm.csum16);
    }
}

/* ── Checksum Update ─────────────────────────────────── */
control MyComputeChecksum(inout headers_t hdr,
                          inout metadata_t meta) {
    apply {
        update_checksum(
            hdr.ipv4.isValid(),
            { hdr.ipv4.version, hdr.ipv4.ihl, hdr.ipv4.diffserv,
              hdr.ipv4.total_len, hdr.ipv4.id, hdr.ipv4.flags,
              hdr.ipv4.frag_offset, hdr.ipv4.ttl, hdr.ipv4.protocol,
              hdr.ipv4.src_addr, hdr.ipv4.dst_addr },
            hdr.ipv4.hdr_checksum,
            HashAlgorithm.csum16);
    }
}

/* ── Deparser ─────────────────────────────────────────── */
control MyDeparser(packet_out pkt, in headers_t hdr) {
    apply {
        pkt.emit(hdr.ethernet);
        pkt.emit(hdr.ipv4);
    }
}

/* ── Switch instantiation ─────────────────────────────── */
V1Switch(
    MyParser(),
    MyVerifyChecksum(),
    MyIngress(),
    MyEgress(),
    MyComputeChecksum(),
    MyDeparser()
) main;
```

### Step 2 — Compile with p4c

```bash
mkdir -p build
p4c --target bmv2 --arch v1model --p4runtime-files build/router.p4info.txt \
    -o build/ router.p4
```

Expected output (no errors):

```
# (silent on success)
```

Verify the output artifacts exist:

```bash
ls -lh build/
# -rw-r--r-- 1 user user  12K router.json
# -rw-r--r-- 1 user user 3.2K router.p4info.txt
```

Inspect the P4Info to find table and action IDs you will need in the controller:

```bash
cat build/router.p4info.txt
# tables {
#   preamble { id: 33574068  name: "MyIngress.ipv4_lpm" ... }
#   match_fields { id: 1  name: "hdr.ipv4.dst_addr"  match_type: LPM }
#   action_refs { id: 23766285 }   <- set_nhop
# }
# actions {
#   preamble { id: 23766285  name: "MyIngress.set_nhop" }
#   params { id: 1  name: "port" ... }
#   params { id: 2  name: "dst_mac" ... }
# }
```

Note the numeric IDs — your controller uses them to address tables and actions without string parsing.

### Step 3 — Create veth Pairs and Network Namespaces

```bash
# Create two host namespaces
sudo ip netns add host0
sudo ip netns add host1

# veth pair for host0: veth0 (bmv2 port 0) <-> veth0h (inside host0)
sudo ip link add veth0  type veth peer name veth0h
sudo ip link add veth1  type veth peer name veth1h

# Move one end of each pair into its namespace
sudo ip link set veth0h netns host0
sudo ip link set veth1h netns host1

# Bring up the bmv2-side interfaces
sudo ip link set veth0 up
sudo ip link set veth1 up

# Configure IP addresses inside each namespace
sudo ip netns exec host0 ip link set veth0h up
sudo ip netns exec host0 ip addr add 10.0.0.1/24 dev veth0h
sudo ip netns exec host0 ip route add 10.0.1.0/24 via 10.0.0.254   # default via "router"

sudo ip netns exec host1 ip link set veth1h up
sudo ip netns exec host1 ip addr add 10.0.1.1/24 dev veth1h
sudo ip netns exec host1 ip route add 10.0.0.0/24 via 10.0.1.254

# Disable checksums (bmv2 does not recompute offloaded checksums)
sudo ip netns exec host0 ethtool -K veth0h tx-checksum-ip-generic off
sudo ip netns exec host1 ethtool -K veth1h tx-checksum-ip-generic off
```

Verify the veth pairs are up:

```bash
ip link show veth0
# 5: veth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...

ip -n host0 addr show veth0h
# inet 10.0.0.1/24 scope global veth0h
```

### Step 4 — Start simple_switch

```bash
# Run bmv2 in the background, binding port 0 → veth0, port 1 → veth1
# P4Runtime gRPC server listens on port 9559
sudo simple_switch_grpc \
    --device-id 0 \
    --interface 0@veth0 \
    --interface 1@veth1 \
    --log-console \
    build/router.json \
    -- \
    --grpc-server-addr 0.0.0.0:9559 &

SWITCH_PID=$!
echo "bmv2 PID: $SWITCH_PID"
```

Expected log output (first few lines):

```
[09:00:00.001] [bmv2] [I] Thrift server was not requested
[09:00:00.002] [bmv2] [I] Starting P4 switch: simple_switch_grpc
[09:00:00.010] [bmv2] [I] Device ID set to 0
[09:00:00.011] [bmv2] [I] P4Runtime gRPC server: 0.0.0.0:9559
```

Verify the gRPC port is listening:

```bash
ss -tlnp | grep 9559
# LISTEN  0  128  0.0.0.0:9559  0.0.0.0:*  users:(("simple_switch_g",pid=...,fd=...))
```

### Step 5 — Write and Run the Python P4Runtime Controller

Save this as `controller.py`:

```python
#!/usr/bin/env python3
"""P4Runtime controller for router.p4 on bmv2."""

import grpc
import time
from p4.v1 import p4runtime_pb2, p4runtime_pb2_grpc
from p4.config.v1 import p4info_pb2
import google.protobuf.text_format as text_format

GRPC_ADDR   = "localhost:9559"
DEVICE_ID   = 0
P4INFO_FILE = "build/router.p4info.txt"
P4BIN_FILE  = "build/router.json"

# ── Helper: load P4Info ───────────────────────────────────────────────────────
def load_p4info(path):
    p4info = p4info_pb2.P4Info()
    with open(path, "r") as f:
        text_format.Merge(f.read(), p4info)
    return p4info

def get_table_id(p4info, name):
    for t in p4info.tables:
        if t.preamble.name == name:
            return t.preamble.id
    raise KeyError(f"Table not found: {name}")

def get_action_id(p4info, name):
    for a in p4info.actions:
        if a.preamble.name == name:
            return a.preamble.id
    raise KeyError(f"Action not found: {name}")

# ── Helper: push P4 program to switch ────────────────────────────────────────
def set_forwarding_pipeline(stub, p4info, device_id):
    with open(P4BIN_FILE, "rb") as f:
        device_config = f.read()
    request = p4runtime_pb2.SetForwardingPipelineConfigRequest(
        device_id=device_id,
        action=p4runtime_pb2.SetForwardingPipelineConfigRequest.VERIFY_AND_COMMIT,
    )
    request.config.p4info.CopyFrom(p4info)
    request.config.p4_device_config = device_config
    stub.SetForwardingPipelineConfig(request)
    print("[ctrl] Pipeline loaded onto switch")

# ── Helper: insert LPM forwarding entry ──────────────────────────────────────
def insert_lpm_entry(stub, p4info, device_id, prefix_ip, prefix_len, out_port, dst_mac):
    table_id  = get_table_id(p4info,  "MyIngress.ipv4_lpm")
    action_id = get_action_id(p4info, "MyIngress.set_nhop")

    # Encode IP prefix as 4-byte big-endian
    ip_bytes = bytes(int(o) for o in prefix_ip.split("."))

    entry = p4runtime_pb2.TableEntry()
    entry.table_id = table_id

    mf = entry.match.add()
    mf.field_id = 1          # hdr.ipv4.dst_addr
    mf.lpm.value      = ip_bytes
    mf.lpm.prefix_len = prefix_len

    entry.action.action.action_id = action_id

    p_port = entry.action.action.params.add()
    p_port.param_id = 1
    p_port.value    = out_port.to_bytes(2, "big")

    p_mac = entry.action.action.params.add()
    p_mac.param_id = 2
    p_mac.value    = bytes.fromhex(dst_mac.replace(":", ""))

    update = p4runtime_pb2.Update(
        type=p4runtime_pb2.Update.INSERT,
    )
    update.entity.table_entry.CopyFrom(entry)

    stub.Write(p4runtime_pb2.WriteRequest(
        device_id=device_id,
        updates=[update],
    ))
    print(f"[ctrl] Inserted: {prefix_ip}/{prefix_len} → port {out_port}  dst_mac {dst_mac}")

# ── Helper: read packet counter ───────────────────────────────────────────────
def read_counter(stub, p4info, device_id, counter_name, index):
    counter_id = None
    for c in p4info.counters:
        if c.preamble.name == counter_name:
            counter_id = c.preamble.id
            break
    if counter_id is None:
        raise KeyError(f"Counter not found: {counter_name}")

    entity = p4runtime_pb2.Entity()
    entity.counter_entry.counter_id = counter_id
    entity.counter_entry.index.index = index

    resp = stub.Read(p4runtime_pb2.ReadRequest(
        device_id=device_id,
        entities=[entity],
    ))
    for r in resp:
        for e in r.entities:
            ce = e.counter_entry
            print(f"[ctrl] Counter '{counter_name}'[{index}]:"
                  f"  packets={ce.data.packet_count}"
                  f"  bytes={ce.data.byte_count}")

# ── Main ──────────────────────────────────────────────────────────────────────
def main():
    channel = grpc.insecure_channel(GRPC_ADDR)
    stub    = p4runtime_pb2_grpc.P4RuntimeStub(channel)

    # Mastership arbitration (required before any write)
    stream = stub.StreamChannel()
    arb = p4runtime_pb2.StreamMessageRequest()
    arb.arbitration.device_id = DEVICE_ID
    arb.arbitration.election_id.high = 0
    arb.arbitration.election_id.low  = 1
    stream.send(arb)
    resp = next(stream)
    print(f"[ctrl] Arbitration status: {resp.arbitration.status.code}")

    p4info = load_p4info(P4INFO_FILE)
    set_forwarding_pipeline(stub, p4info, DEVICE_ID)

    # Route host0 (10.0.0.x) → port 0, host1 (10.0.1.x) → port 1
    # MAC addresses match the veth pairs created earlier
    insert_lpm_entry(stub, p4info, DEVICE_ID,
                     prefix_ip="10.0.0.0", prefix_len=24,
                     out_port=0, dst_mac="00:00:00:00:00:01")
    insert_lpm_entry(stub, p4info, DEVICE_ID,
                     prefix_ip="10.0.1.0", prefix_len=24,
                     out_port=1, dst_mac="00:00:00:00:00:02")

    print("[ctrl] Waiting 5 s for traffic, then reading counters …")
    time.sleep(5)
    read_counter(stub, p4info, DEVICE_ID, "MyIngress.pkt_counter", 0)
    read_counter(stub, p4info, DEVICE_ID, "MyIngress.pkt_counter", 1)

if __name__ == "__main__":
    main()
```

Run the controller:

```bash
source .venv/bin/activate
python3 controller.py
```

Expected output:

```
[ctrl] Arbitration status: 0
[ctrl] Pipeline loaded onto switch
[ctrl] Inserted: 10.0.0.0/24 → port 0  dst_mac 00:00:00:00:00:01
[ctrl] Inserted: 10.0.1.0/24 → port 1  dst_mac 00:00:00:00:00:02
[ctrl] Waiting 5 s for traffic, then reading counters …
```

### Step 6 — Verify End-to-End Forwarding with ping

Open a second terminal and run ping from host0 to host1 while the controller is waiting:

```bash
sudo ip netns exec host0 ping -c 4 10.0.1.1
```

Expected output:

```
PING 10.0.1.1 (10.0.1.1) 56(84) bytes of data.
64 bytes from 10.0.1.1: icmp_seq=1 ttl=63 time=1.2 ms
64 bytes from 10.0.1.1: icmp_seq=2 ttl=63 time=0.9 ms
64 bytes from 10.0.1.1: icmp_seq=3 ttl=63 time=1.1 ms
64 bytes from 10.0.1.1: icmp_seq=4 ttl=63 time=0.8 ms
--- 10.0.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
```

After the 5-second wait the controller prints counter readings:

```
[ctrl] Counter 'MyIngress.pkt_counter'[0]:  packets=4  bytes=336
[ctrl] Counter 'MyIngress.pkt_counter'[1]:  packets=4  bytes=336
```

Confirm TTL was decremented (TTL should be 63 for a single-hop router):

```bash
sudo ip netns exec host0 ping -c 1 -t 1 10.0.1.1  # expect TTL exceeded if only 1 hop
# Actually, TTL=64 from host → router decrements to 63 → host1 sees 63
sudo ip netns exec host0 ping -c 1 10.0.1.1 | grep ttl
# 64 bytes from 10.0.1.1: icmp_seq=1 ttl=63 time=1.0 ms
```

TTL=63 confirms the P4 router decremented it by 1.

### Step 7 — Read the Packet Counter Table via P4Runtime

Add a standalone counter-read script `read_counters.py`:

```python
#!/usr/bin/env python3
"""Read all counter indices from the running switch."""

import grpc
from p4.v1 import p4runtime_pb2, p4runtime_pb2_grpc
from p4.config.v1 import p4info_pb2
import google.protobuf.text_format as text_format

GRPC_ADDR   = "localhost:9559"
DEVICE_ID   = 0
P4INFO_FILE = "build/router.p4info.txt"

def load_p4info(path):
    p4info = p4info_pb2.P4Info()
    with open(path) as f:
        text_format.Merge(f.read(), p4info)
    return p4info

channel = grpc.insecure_channel(GRPC_ADDR)
stub    = p4runtime_pb2_grpc.P4RuntimeStub(channel)
p4info  = load_p4info(P4INFO_FILE)

counter_id = next(
    c.preamble.id for c in p4info.counters
    if c.preamble.name == "MyIngress.pkt_counter"
)

# Read ALL indices (wildcard)
entity = p4runtime_pb2.Entity()
entity.counter_entry.counter_id = counter_id

for resp in stub.Read(p4runtime_pb2.ReadRequest(device_id=DEVICE_ID, entities=[entity])):
    for e in resp.entities:
        ce = e.counter_entry
        if ce.data.packet_count > 0:
            print(f"Port {ce.index.index:>3}:  "
                  f"pkts={ce.data.packet_count:>8}  "
                  f"bytes={ce.data.byte_count:>10}")
```

```bash
python3 read_counters.py
```

Expected output after several pings:

```
Port   0:  pkts=      12  bytes=      1008
Port   1:  pkts=      12  bytes=      1008
```

### Step 8 — Craft and Send Packets with Scapy; Verify Forwarding Decisions

Scapy is a Python packet manipulation library that allows construction of arbitrary network packets at each protocol layer, injection onto a live interface, and capture and dissection of responses. It is widely used for network testing, protocol fuzzing, and verifying forwarding behavior in software switches.

Use Scapy to send crafted packets and observe that bmv2 makes the correct forwarding decision. Run from the host (not inside a namespace), injecting directly onto `veth0`:

```python
#!/usr/bin/env python3
"""Send crafted packets via Scapy and verify forwarding."""

from scapy.all import Ether, IP, ICMP, sendp, sniff, conf
import threading, time

IFACE_SEND = "veth0"   # bmv2 port 0 — traffic entering from "host0" side
IFACE_RECV = "veth1"   # bmv2 port 1 — traffic exiting toward "host1"

received = []

def sniffer():
    pkts = sniff(iface=IFACE_RECV, count=3, timeout=5,
                 filter="icmp")
    received.extend(pkts)

t = threading.Thread(target=sniffer)
t.start()

time.sleep(0.5)   # let sniffer start

# Craft ICMP echo request: src=10.0.0.1 → dst=10.0.1.1
# Dst MAC must match what we told bmv2 to rewrite to (00:00:00:00:00:02)
pkt = (Ether(dst="ff:ff:ff:ff:ff:ff", src="00:00:00:00:00:01") /
       IP(src="10.0.0.1", dst="10.0.1.1", ttl=64) /
       ICMP())

for i in range(3):
    sendp(pkt, iface=IFACE_SEND, verbose=False)
    print(f"Sent packet {i+1}")

t.join()

print(f"\nCaptured {len(received)} packets on {IFACE_RECV}:")
for p in received:
    print(f"  src={p[IP].src}  dst={p[IP].dst}  ttl={p[IP].ttl}")
    assert p[IP].ttl == 63, f"Expected TTL=63, got {p[IP].ttl}"
    assert p[IP].dst == "10.0.1.1"

print("\nAll assertions passed — bmv2 forwarded and decremented TTL correctly.")
```

```bash
sudo python3 scapy_test.py
```

Expected output:

```
Sent packet 1
Sent packet 2
Sent packet 3

Captured 3 packets on veth1:
  src=10.0.0.1  dst=10.0.1.1  ttl=63
  src=10.0.0.1  dst=10.0.1.1  ttl=63
  src=10.0.0.1  dst=10.0.1.1  ttl=63

All assertions passed — bmv2 forwarded and decremented TTL correctly.
```

Verify that a packet destined for an unrouted prefix is dropped (no capture on veth1):

```python
# Send to 192.168.99.1 — no matching LPM entry, default_action = _drop()
bad_pkt = (Ether(dst="ff:ff:ff:ff:ff:ff", src="00:00:00:00:00:01") /
           IP(src="10.0.0.1", dst="192.168.99.1", ttl=64) /
           ICMP())
pkts = sniff(iface=IFACE_RECV, count=1, timeout=2,
             started_callback=lambda: sendp(bad_pkt, iface=IFACE_SEND, verbose=False))
assert len(pkts) == 0, "Expected drop, got a packet!"
print("Drop verified: unrouted prefix correctly dropped by bmv2.")
```

### Step 9 — Cleanup

```bash
# Stop bmv2
sudo kill $SWITCH_PID

# Remove network namespaces and veth pairs
sudo ip netns del host0
sudo ip netns del host1
sudo ip link del veth0
sudo ip link del veth1
```

---

## Summary

- P4 defines the entire forwarding pipeline — parser, match-action tables, deparser — in a target-independent language; the compiler maps it to software (bmv2) or ASIC (Tofino).
- P4Runtime is the gRPC control-plane API; it cleanly separates the data plane (P4 program) from the control plane (controller).
- In-network computing (SHARP, custom P4 aggregation) is the most impactful P4 application for AI: reducing AllReduce traffic volume by performing partial sums in the switch.
- INT provides per-hop, per-packet telemetry without sampling, giving unprecedented visibility into NCCL flow latency across the fabric.

---

## References

- P4 Language Consortium: p4.org
- P4₁₆ specification: p4lang.github.io/p4-spec
- P4Runtime specification: p4lang.github.io/p4runtime
- Bosshart et al., *P4: Programming Protocol-Independent Packet Processors*, SIGCOMM CCR 2014
- Sapio et al., *In-Network Computation is a Dumb Idea Whose Time Has Come*, HotNets 2017


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).