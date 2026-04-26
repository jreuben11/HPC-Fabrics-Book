# Chapter 15 — gNMI & OpenConfig Streaming Telemetry

**Part V: Management, Telemetry & Control** | ~20 pages

---

## Introduction

Monitoring a thousand-node AI training cluster in real time is not an optional operational nicety — it is a prerequisite for maintaining training efficiency. A single congested switch port, a flapping BGP session, or a rising tail-drop rate on one of sixteen ECMP paths can silently stall a distributed all-reduce collective for hundreds of milliseconds, degrading GPU utilization across the entire job. Detecting and diagnosing these events requires telemetry at sub-second granularity, structured in a way that can be queried, alerted on, and visualized without vendor-specific parsing.

gNMI (gRPC Network Management Interface) is the streaming telemetry protocol designed for this purpose. It runs over gRPC — Google's high-performance RPC framework built on HTTP/2 and Protocol Buffers — and uses the OpenConfig YANG path namespace introduced in Chapter 14 as its addressing scheme. Instead of the poll-based SNMP model where a management station asks each device for counters every five minutes, gNMI uses persistent subscription streams: the collector subscribes to a set of OpenConfig paths, and the device pushes updates at the specified interval or whenever a value changes. This inversion eliminates polling overhead on the device, reduces latency from minutes to seconds or less, and delivers structured, self-describing data.

This chapter covers the gNMI protocol's four RPCs and subscription modes, the `gnmic` CLI and collector tool, the OpenConfig path namespace for interface counters, BGP state, and hardware telemetry, the complete production telemetry pipeline from switch to Grafana dashboard, and dial-out telemetry for firewall-constrained environments. PromQL expressions for AI cluster-specific monitoring — congestion detection, BGP session stability, ECMP imbalance — are derived from first principles.

The lab walkthrough builds a complete end-to-end pipeline: a Containerlab SR Linux node generates live interface counters and BGP state change events; `gnmic` subscribes with both sampled and on-change modes; Prometheus scrapes the gnmic Prometheus endpoint; and a PromQL query computes real-time throughput in Gbps as iperf3 generates traffic. An on-change BGP subscription demonstrates how state transitions as short as 500 ms are captured reliably.

This chapter closes Part V. Chapter 16 builds on the telemetry foundation established here by integrating gNMI metrics into a full-stack observability platform alongside OpenTelemetry traces, Prometheus alerting, and Grafana dashboards — adding the application-layer signals from AI training frameworks that sit above the network layer.

---

---

## Installation

gnmic is the gNMI CLI and collector that drives all protocol interaction in this chapter: it issues Capabilities and Get requests for one-shot inspection, and runs a persistent Subscribe session that exposes collected metrics as a Prometheus endpoint. Prometheus scrapes that endpoint and stores the time-series data; Grafana connects to Prometheus as a datasource and renders the dashboards and PromQL-based alerts used to detect congestion and BGP session instability. Containerlab with a Nokia SR Linux node provides the gNMI target, since SR Linux ships with a gNMI server that supports both sampled and on-change subscriptions over OpenConfig paths. The complete pipeline — SR Linux generating live counters, gnmic collecting them, Prometheus storing them, and Grafana visualizing them — is what the lab walkthrough builds end to end.

### System packages (Ubuntu 24.04)

```bash
# gnmic
bash -c "$(curl -sL https://get.gnmic.openconfig.net)"

# Prometheus
sudo apt install -y prometheus
# Or via Docker:
docker pull prom/prometheus:latest

# Grafana
sudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install -y grafana

# Containerlab + SR Linux
bash -c "$(curl -sL https://get.containerlab.dev)"
```

### Python environment (uv)

```bash
uv venv .venv && source .venv/bin/activate
uv pip install prometheus-client grpcio grpcio-tools protobuf pyyaml
```

---

## 15.1 From SNMP Polling to Streaming Telemetry

SNMP (Simple Network Management Protocol) was designed for 1990s network management: poll a device every 5 minutes, fetch a scalar counter, aggregate in an NMS (Network Management System — a centralized platform that collects, normalizes, and presents device health data). At AI cluster scale this fails in three ways:

1. **Temporal resolution:** A congestion event that causes NCCL to stall for 500ms is invisible at 5-minute polling granularity.
2. **CPU overhead on devices:** Each SNMP Get generates a context switch and encoding overhead. At high polling rates, this impacts the forwarding plane.
3. **Opaque encoding:** SNMP OIDs (Object Identifiers — numeric dotted-notation keys into the SNMP Management Information Base) require a MIB database for interpretation; vendor extensions are inconsistent.

gNMI (gRPC Network Management Interface) addresses all three: subscriptions push data at sub-second intervals, the device encodes data once and streams it, and OpenConfig path strings are self-documenting.

---

## 15.2 gNMI Protocol

gNMI runs over gRPC (Google Remote Procedure Call — a high-performance RPC framework using HTTP/2 for multiplexed transport and Protocol Buffers for compact binary serialization) on port 57400. It defines four RPCs:

| RPC | Description |
|---|---|
| `Capabilities` | Query device for supported models and encodings |
| `Get` | One-shot read of a data path |
| `Set` | Write configuration (replaces or updates) |
| `Subscribe` | Streaming subscription to one or more paths |

### 15.2.1 Path Notation

gNMI paths follow OpenConfig YANG tree structure:

```
/interfaces/interface[name=Ethernet0]/state/counters/in-octets
/network-instances/network-instance[name=default]/protocols/protocol[name=BGP][identifier=BGP]/bgp/neighbors/neighbor[neighbor-address=192.168.1.1]/state/session-state
/components/component[name=CPU0]/cpu/state/avg-usage
```

Path elements map directly to YANG list keys: `[name=X]` selects a list element.

### 15.2.2 Subscribe Modes

| Mode | Description | Use Case |
|---|---|---|
| `ONCE` | Snapshot, then close | Config audit |
| `POLL` | Stream on demand (client-triggered) | On-demand refresh |
| `STREAM/SAMPLE` | Push at fixed interval | Regular metric collection |
| `STREAM/ON_CHANGE` | Push only when value changes | Event detection (BGP state, link up/down) |

---

## 15.3 gnmic — CLI Client

`gnmic` is the standard CLI and Go library for gNMI interaction.

### Installation

```bash
bash -c "$(curl -sL https://get.gnmic.openconfig.net)"
```

### Capabilities

```bash
gnmic -a 192.168.1.1:57400 --skip-verify capabilities
# gNMI version: 0.7.0
# Supported models:
#   openconfig-interfaces, revision 2022-01-17
#   openconfig-bgp, revision 2023-07-17
#   ...
# Supported encodings: JSON_IETF, PROTO, ASCII
```

### Get

```bash
# Get a single interface's operational state
gnmic -a 192.168.1.1:57400 --skip-verify get \
    --path /interfaces/interface[name=Ethernet0]/state

# JSON output:
# {
#   "source": "192.168.1.1:57400",
#   "time": "2026-04-22T10:00:00Z",
#   "updates": [{
#     "Path": "interfaces/interface[name=Ethernet0]/state",
#     "values": {
#       "oper-status": "UP",
#       "counters": {
#         "in-octets": 1234567890,
#         "out-octets": 9876543210
#       }
#     }
#   }]
# }
```

### Subscribe (Streaming)

```bash
# Stream interface counters every 5 seconds
gnmic -a 192.168.1.1:57400 --skip-verify subscribe \
    --path /interfaces/interface[name=Ethernet0]/state/counters \
    --mode stream \
    --stream-mode sample \
    --sample-interval 5s

# Stream BGP state changes (on-change)
gnmic -a 192.168.1.1:57400 --skip-verify subscribe \
    --path "/network-instances/network-instance[name=default]/protocols/protocol[identifier=BGP][name=BGP]/bgp/neighbors/neighbor/state/session-state" \
    --mode stream \
    --stream-mode on-change

# Subscribe to multiple paths, multiple targets
gnmic --config gnmic-config.yaml subscribe
```

### gnmic Configuration File

```yaml
# gnmic-config.yaml
targets:
  leaf01:57400:
    username: admin
    password: secret
    skip-verify: true
  leaf02:57400:
    username: admin
    password: secret
    skip-verify: true

subscriptions:
  interface-counters:
    paths:
      - /interfaces/interface/state/counters
    mode: stream
    stream-mode: sample
    sample-interval: 10s
    encoding: json_ietf

  bgp-state:
    paths:
      - /network-instances/network-instance[name=default]/protocols/protocol[identifier=BGP][name=BGP]/bgp/neighbors/neighbor/state
    mode: stream
    stream-mode: on-change

outputs:
  prometheus-output:
    type: prometheus
    listen: :9273
    path: /metrics
    metric-name-prefix: gnmi_
```

---

## 15.4 OpenConfig Models in Depth

### Interface Counters

```
/interfaces/interface[name=*]/state/counters/
  in-octets          # total bytes received
  out-octets         # total bytes transmitted
  in-errors          # receive errors
  out-errors         # transmit errors
  in-discards        # drops (buffer overflow)
  out-discards       # drops (transmit queue overflow)
  in-pkts            # packets received
  out-pkts           # packets transmitted
```

### BGP Neighbor State

```
/network-instances/network-instance[name=default]/protocols/protocol[identifier=BGP][name=BGP]/bgp/neighbors/neighbor[neighbor-address=<ip>]/state/
  session-state      # IDLE/CONNECT/ACTIVE/OPENSENT/OPENCONFIRM/ESTABLISHED
  messages/received/UPDATE
  messages/sent/UPDATE
  prefixes/received
  prefixes/sent
  last-established   # Unix timestamp of last session establishment
```

### Platform / Hardware

```
/components/component[name=*]/state/
  temperature/instant     # sensor temperature (°C)
  memory/available        # available memory (bytes)
  type                    # CHASSIS/LINECARD/PORT/CPU/...

/components/component[name=CPU0]/cpu/state/
  avg-usage               # average CPU utilization %
  instant                 # current utilization %
```

---

## 15.5 Telemetry Pipeline Architecture

A production telemetry pipeline for a 1000-node AI cluster:

```
[Switches (gNMI dial-in)]         [Hosts (Prometheus node_exporter)]
# node_exporter: a Prometheus agent that exposes Linux host metrics (CPU, memory, disk, network) as a /metrics HTTP endpoint for Prometheus to scrape
         │                                    │
    gnmic collector                    Prometheus scrape
    (per datacenter row)
         │
    Protocol buffer stream
         │
    OpenTelemetry Collector
    (gNMI receiver → transform → export)
    # OpenTelemetry Collector: a vendor-neutral agent/pipeline that can receive,
    # process, and export telemetry (metrics, traces, logs) across multiple backends
         │
    ┌────┴──────────┐
    │   Prometheus  │   ← Prometheus remote_write
    │   VictoriaMetrics│ ← high-cardinality metrics
    # VictoriaMetrics: a high-performance time-series database compatible with
    # Prometheus remote_write and PromQL, optimized for very high label cardinality
    └────┬──────────┘
         │
    Grafana (dashboards)
    PagerDuty (alerting)
```

### gnmic → Prometheus

With the gnmic Prometheus output (Section 15.3), metrics are exposed at `:9273/metrics` and scraped by Prometheus:

```
# HELP gnmi_interface_state_counters_in_octets 
# TYPE gnmi_interface_state_counters_in_octets gauge
gnmi_interface_state_counters_in_octets{source="leaf01",name="Ethernet0"} 1.23456789e+09
gnmi_interface_state_counters_in_octets{source="leaf01",name="Ethernet4"} 9.87654321e+08
```

### PromQL for AI Cluster Monitoring

PromQL (Prometheus Query Language) is the functional expression language for querying time-series stored in Prometheus. It supports rate calculations, aggregations, and arithmetic across labeled metric streams — the standard tool for building dashboards and alerting rules from telemetry data.

```promql
# Interface utilization % (400G interface)
rate(gnmi_interface_state_counters_in_octets{name=~"Ethernet.*"}[1m]) * 8 / 400e9 * 100

# Dropped packets per second (sign of congestion)
rate(gnmi_interface_state_counters_in_discards[1m])

# BGP session flaps in last hour
changes(gnmi_bgp_neighbors_neighbor_state_session_state[1h])

# ECMP imbalance: max - min utilization across ports
max(rate(gnmi_interface_state_counters_out_octets[1m])) by (source)
- min(rate(gnmi_interface_state_counters_out_octets[1m])) by (source)
```

---

## 15.6 Dial-Out Telemetry

In dial-out mode, the device initiates the gRPC connection to the collector — useful when firewalls prevent the collector from reaching devices:

```yang
# SR Linux dial-out configuration
set /system gnmi-server admin-state enable
set /system telemetry destination-group collectors
  set /system telemetry destination-group collectors destination collector1
    set address 10.0.0.100
    set port 57401
    set transport grpc
    set allow-aggregation true

set /system telemetry subscription interface-counters
  set sensor-group interfaces
    set sensor-path /interface/statistics
  set destination-group collectors
  set sample-interval 10000  # milliseconds
```

---

## Lab Walkthrough 15 — gNMI Streaming Dashboard

This walkthrough builds a complete end-to-end gNMI telemetry pipeline: Containerlab SR Linux node → gnmic → Prometheus → Grafana. Each step includes the exact command, expected output, and a verification check.

### Step 1 — Write the Containerlab topology file

Create `srl-lab.yaml`:

```yaml
# srl-lab.yaml
name: gnmi-lab

topology:
  nodes:
    leaf01:
      kind: srl
      image: ghcr.io/nokia/srlinux:23.10.1
      startup-config: |
        set /system gnmi-server admin-state enable
        set /system gnmi-server rate-limit 65000
        set /network-instance default protocols bgp admin-state enable
        set /network-instance default protocols bgp router-id 10.0.0.1
        set /network-instance default protocols bgp autonomous-system 65001

    host1:
      kind: linux
      image: alpine:3.19

  links:
    - endpoints: ["leaf01:e1-1", "host1:eth1"]
```

Start the topology:

```bash
sudo containerlab deploy -t srl-lab.yaml
```

Expected output:

```
INFO[0000] Containerlab v0.54.2 started
INFO[0000] Parsing & checking topology file: srl-lab.yaml
INFO[0002] Creating lab directory: /root/clab-gnmi-lab
INFO[0004] Creating container: leaf01
INFO[0010] Creating container: host1
INFO[0012] Creating virtual wire: leaf01:e1-1 <--> host1:eth1
INFO[0013] 2 nodes, 1 links
+---+------------------+--------------+----------------------------+---------+
| # |       Name       | Container ID |           Image            |  State  |
+---+------------------+--------------+----------------------------+---------+
| 1 | clab-gnmi-lab-leaf01 | a3f1b2c4d5e6 | ghcr.io/nokia/srlinux:23.10.1 | running |
| 2 | clab-gnmi-lab-host1  | b7e8f9a0b1c2 | alpine:3.19                   | running |
+---+------------------+--------------+----------------------------+---------+
```

Verify the node is reachable:

```bash
# SR Linux management IP is printed by containerlab; typically in 172.20.20.0/24
ping -c 2 172.20.20.2
```

Expected: `2 packets transmitted, 2 received, 0% packet loss`

### Step 2 — Confirm gnmic is installed and query capabilities

```bash
gnmic version
```

Expected:

```
version : 0.36.2
 commit : abc1234
   date : 2026-01-15T00:00:00Z
 gitURL : https://github.com/openconfig/gnmic
   docs : https://gnmic.openconfig.net
```

Now query the SR Linux node's capabilities (the management address printed by Containerlab, default credentials `admin`/`NokiaSrl1!`):

```bash
gnmic -a 172.20.20.2:57400 \
      -u admin -p 'NokiaSrl1!' \
      --skip-verify \
      capabilities
```

Expected output (annotated):

```
gNMI version: 0.10.0                          # gNMI spec version the device supports

Supported models:
  - Name: urn:srl_nokia/models/interfaces:srl_nokia-interfaces
    Organization: Nokia
    Version: 2023-10-31                        # model revision date

  - Name: urn:srl_nokia/models/network-instance:srl_nokia-network-instance
    Organization: Nokia
    Version: 2023-10-31

  - Name: urn:openconfig/models:openconfig-interfaces
    Organization: OpenConfig Working Group
    Version: 2022-01-17                        # vendor also supports OC models

  - Name: urn:openconfig/models:openconfig-bgp
    Organization: OpenConfig Working Group
    Version: 2023-07-17

Supported encodings:
  - JSON_IETF                                  # preferred; human-readable JSON
  - PROTO                                      # compact binary for production
  - ASCII
```

### Step 3 — Get a single interface's state with full JSON output

```bash
gnmic -a 172.20.20.2:57400 \
      -u admin -p 'NokiaSrl1!' \
      --skip-verify \
      get \
      --path '/interfaces/interface[name=ethernet-1/1]/state' \
      --encoding json_ietf
```

Expected full JSON output:

```json
[
  {
    "source": "172.20.20.2:57400",
    "timestamp": 1745316000000000000,
    "time": "2026-04-22T10:00:00Z",
    "updates": [
      {
        "Path": "interfaces/interface[name=ethernet-1/1]/state",
        "values": {
          "interfaces/interface/state": {
            "name": "ethernet-1/1",
            "type": "iana-if-type:ethernetCsmacd",
            "mtu": 9232,
            "oper-status": "UP",
            "admin-status": "UP",
            "last-change": 1745315800000000000,
            "counters": {
              "in-octets": "482910234",
              "out-octets": "391827401",
              "in-pkts": "3217890",
              "out-pkts": "2891044",
              "in-errors": "0",
              "out-errors": "0",
              "in-discards": "0",
              "out-discards": "0",
              "in-multicast-pkts": "1204",
              "out-multicast-pkts": "980",
              "last-clear": "2026-04-22T09:00:00Z"
            }
          }
        }
      }
    ]
  }
]
```

Verification: confirm `oper-status` is `UP` and `in-octets` is a non-zero integer.

### Step 4 — Write gnmic-config.yaml with two subscriptions

Create `gnmic-config.yaml` in your working directory with the following full content:

```yaml
# gnmic-config.yaml
# Targets: SR Linux node launched by Containerlab
targets:
  172.20.20.2:57400:
    username: admin
    password: "NokiaSrl1!"
    skip-verify: true
    timeout: 10s

# Two subscriptions:
#   1. interface-counters  — sampled every 10 s (time-series metrics)
#   2. bgp-state           — on-change only (event detection)
subscriptions:
  interface-counters:
    paths:
      - /interfaces/interface/state/counters
    mode: stream
    stream-mode: sample
    sample-interval: 10s
    encoding: json_ietf
    suppress-redundant: false

  bgp-state:
    paths:
      - /network-instances/network-instance[name=default]/protocols/protocol[identifier=BGP][name=BGP]/bgp/neighbors/neighbor/state/session-state
    mode: stream
    stream-mode: on-change
    encoding: json_ietf

# Output: expose metrics as Prometheus gauges on :9273/metrics
outputs:
  prometheus-output:
    type: prometheus
    listen: :9273
    path: /metrics
    metric-name-prefix: gnmi_
    expiration: 30s          # remove stale metrics after 30 s of silence
    debug: false
```

Validate the config parses without errors:

```bash
gnmic --config gnmic-config.yaml capabilities
```

Expected: same capabilities output as Step 2, confirming the config file is valid YAML and the target is reachable.

### Step 5 — Run gnmic as a daemon with Prometheus output

Start gnmic in the background, capturing its log:

```bash
gnmic --config gnmic-config.yaml subscribe > /tmp/gnmic.log 2>&1 &
GNMIC_PID=$!
echo "gnmic PID: $GNMIC_PID"
```

Wait 5 seconds for the first sample to arrive, then check the log:

```bash
sleep 5
tail -20 /tmp/gnmic.log
```

Expected log lines:

```
2026-04-22T10:00:05Z INFO  target "172.20.20.2:57400" subscription "interface-counters" started
2026-04-22T10:00:05Z INFO  target "172.20.20.2:57400" subscription "bgp-state" started
2026-04-22T10:00:05Z INFO  starting Prometheus server on :9273
2026-04-22T10:00:10Z DEBUG received update: interfaces/interface[name=ethernet-1/1]/state/counters/in-octets = 482910234
2026-04-22T10:00:10Z DEBUG received update: interfaces/interface[name=ethernet-1/1]/state/counters/out-octets = 391827401
```

### Step 6 — Verify metrics at localhost:9273/metrics

```bash
curl -s http://localhost:9273/metrics | grep gnmi_interface
```

Expected output (key metric names):

```
# HELP gnmi_interface_state_counters_in_octets interfaces/interface/state/counters/in-octets
# TYPE gnmi_interface_state_counters_in_octets gauge
gnmi_interface_state_counters_in_octets{name="ethernet-1/1",source="172.20.20.2:57400"} 4.82910234e+08

# HELP gnmi_interface_state_counters_out_octets interfaces/interface/state/counters/out-octets
# TYPE gnmi_interface_state_counters_out_octets gauge
gnmi_interface_state_counters_out_octets{name="ethernet-1/1",source="172.20.20.2:57400"} 3.91827401e+08

# HELP gnmi_interface_state_counters_in_discards interfaces/interface/state/counters/in-discards
# TYPE gnmi_interface_state_counters_in_discards gauge
gnmi_interface_state_counters_in_discards{name="ethernet-1/1",source="172.20.20.2:57400"} 0

# HELP gnmi_interface_state_counters_out_discards interfaces/interface/state/counters/out-discards
# TYPE gnmi_interface_state_counters_out_discards gauge
gnmi_interface_state_counters_out_discards{name="ethernet-1/1",source="172.20.20.2:57400"} 0
```

Key metric names to note for PromQL:
- `gnmi_interface_state_counters_in_octets` — receive byte counter
- `gnmi_interface_state_counters_out_octets` — transmit byte counter
- `gnmi_interface_state_counters_in_discards` — receive drops (congestion indicator)
- `gnmi_interface_state_counters_out_discards` — transmit drops

### Step 7 — Add the gnmic scrape target to prometheus.yml

Edit `/etc/prometheus/prometheus.yml` and add the gnmic job under `scrape_configs`:

```yaml
# /etc/prometheus/prometheus.yml  (append to existing scrape_configs section)
scrape_configs:

  # --- existing jobs above ---

  - job_name: 'gnmic'
    scrape_interval: 15s
    scrape_timeout: 10s
    static_configs:
      - targets: ['localhost:9273']
        labels:
          environment: 'containerlab'
          site: 'lab'
```

Reload Prometheus to pick up the new target (if running as a service):

```bash
sudo systemctl reload prometheus
# or, if running via Docker:
# docker kill --signal=SIGHUP prometheus
```

### Step 8 — Start Prometheus and open the console

If Prometheus is not yet running as a service, start it:

```bash
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Expected status output:

```
● prometheus.service - Prometheus
     Loaded: loaded (/lib/systemd/system/prometheus.service; enabled)
     Active: active (running) since 2026-04-22 10:00:30 UTC
   Main PID: 12345 (prometheus)
```

Open the Prometheus web console at `http://localhost:9090`.

Navigate to **Status → Targets** and confirm the `gnmic` job shows state `UP`:

```
Endpoint                   State   Labels                        Last Scrape   Error
http://localhost:9273/metrics  UP   environment="containerlab"   2.345s ago
```

### Step 9 — Run a PromQL query for interface throughput in Gbps

In the Prometheus console Expression Browser (`http://localhost:9090`), enter:

```promql
rate(gnmi_interface_state_counters_in_octets{name="ethernet-1/1"}[1m]) * 8 / 1e9
```

This query:
- `rate(...[1m])` — computes bytes/second over the last 1 minute
- `* 8` — converts bytes to bits
- `/ 1e9` — converts bits to Gbps

Expected result (at idle): a value near `0.001` Gbps or lower.

### Step 10 — Generate traffic with iperf3 and watch the metric update

Install iperf3 in the host container and run a traffic test:

```bash
# Install iperf3 in host1
sudo docker exec clab-gnmi-lab-host1 apk add --no-cache iperf3

# Run iperf3 server inside leaf01's network namespace (SR Linux has a built-in iperf3)
# For a Linux host topology, start server on one side and client on the other.
# Here we generate traffic from host1 toward leaf01's loopback:
sudo docker exec -d clab-gnmi-lab-host1 iperf3 -s -p 5201

# Run the client for 60 seconds to generate sustained traffic
sudo docker exec clab-gnmi-lab-host1 iperf3 \
    -c 172.20.20.2 \
    -p 5201 \
    -t 60 \
    -b 1G \
    --parallel 4
```

Expected iperf3 output:

```
Connecting to host 172.20.20.2, port 5201
[  5] local 172.20.20.3 port 42100 connected to 172.20.20.2 port 5201
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-10.00  sec  1.18 GBytes   1.01 Gbits/sec
[  5]  10.00-20.00  sec  1.19 GBytes   1.02 Gbits/sec
...
- - - - - - - - - - - - - - - - - - - - - - - - -
[SUM]   0.00-60.00  sec  7.08 GBytes   1.02 Gbits/sec  sender
```

Re-run the PromQL query in the Prometheus console and observe the rate increase to approximately `1.0` Gbps. The metric should update within the next 10-second sample interval.

Verification:

```bash
curl -s 'http://localhost:9090/api/v1/query?query=rate(gnmi_interface_state_counters_in_octets%7Bname%3D%22ethernet-1%2F1%22%7D%5B1m%5D)*8%2F1e9' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['data']['result'][0]['value'][1], 'Gbps')"
```

Expected: `1.021 Gbps` (approximately).

### Step 11 — Kill the BGP session and watch the on-change notification fire

BGP sessions require a configured peer. First add a BGP neighbor on the SR Linux node:

```bash
sudo docker exec -it clab-gnmi-lab-leaf01 sr_cli
# Inside SR Linux CLI:
# enter candidate
# set network-instance default protocols bgp neighbor 192.168.1.2 peer-as 65002
# commit now
# quit
```

Tail the gnmic log in a separate terminal to watch for on-change events:

```bash
tail -f /tmp/gnmic.log | grep bgp
```

Now administratively shut the BGP session:

```bash
sudo docker exec -it clab-gnmi-lab-leaf01 sr_cli \
  "enter candidate; set network-instance default protocols bgp neighbor 192.168.1.2 admin-state disable; commit now"
```

Expected gnmic log output within 1–2 seconds (on-change fires immediately):

```
2026-04-22T10:05:12Z INFO  target "172.20.20.2:57400" subscription "bgp-state" update received
2026-04-22T10:05:12Z DEBUG path: network-instances/network-instance[name=default]/protocols/protocol[identifier=BGP][name=BGP]/bgp/neighbors/neighbor[neighbor-address=192.168.1.2]/state/session-state
2026-04-22T10:05:12Z DEBUG value: "IDLE"
```

Contrast with the `interface-counters` subscription: the next counter update arrives at the next 10-second sample interval, not immediately. On-change subscriptions are critical for event detection — a BGP state transition that lasts only 500ms would be missed by a 10-second sampled subscription.

Re-enable the BGP neighbor to restore the session:

```bash
sudo docker exec -it clab-gnmi-lab-leaf01 sr_cli \
  "enter candidate; set network-instance default protocols bgp neighbor 192.168.1.2 admin-state enable; commit now"
```

Expected on-change log: session transitions `IDLE → CONNECT → OPENSENT → OPENCONFIRM → ESTABLISHED`, each state change fires a separate notification.

### Cleanup

```bash
kill $GNMIC_PID
sudo containerlab destroy -t srl-lab.yaml
```

---

## Summary

- gNMI replaces SNMP with sub-second, push-based, self-describing telemetry over gRPC — the difference between seeing congestion after it caused a stall vs seeing it as it happens.
- OpenConfig paths are structured, vendor-neutral, and directly derivable from YANG models.
- `gnmic` provides both CLI access and a configurable collector/pipeline with outputs for Prometheus, InfluxDB, Kafka, and more.
- The standard AI cluster telemetry stack is: gnmic (collection) → Prometheus (storage) → Grafana (visualization), with on-change subscriptions to BGP and link state for event detection.

---

## References

- gNMI specification: openconfig.net/docs/gnmi/gnmi-specification
- gnmic documentation: gnmic.openconfig.net
- OpenConfig models: openconfig.net/projects/models
- IETF RFC 9254: YANG-Push — datatracker.ietf.org/doc/html/rfc9254


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).