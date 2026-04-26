# Chapter 16 — Observability Pipeline: OpenTelemetry, Prometheus & Grafana

**Part V: Management, Telemetry & Control** | ~15 pages

---

## Introduction

An AI cluster running at scale generates an enormous and continuous stream of operational signals: GPU utilization percentages, NIC byte counters, **BGP** session state changes, **RDMA** port error counts, kernel OOM events, and **NCCL** timeout messages. Without a coherent observability pipeline to collect, correlate, and visualize these signals in real time, debugging a slow AllReduce or a degraded training run becomes a multi-hour exercise in log archaeology across dozens of hosts.

This chapter builds a full observability stack grounded in three open-source pillars: **OpenTelemetry** for instrumentation and collection, **Prometheus** for metrics storage and alerting, and **Grafana** for visualization and dashboard composition. These three systems have become the de facto standard across cloud-native and HPC environments alike — and they compose cleanly with the GPU-specific and network-specific exporters that AI cluster operators depend on daily.

The central insight is that effective cluster observability requires correlating signals from different layers simultaneously. A GPU compute stall (visible in **DCGM** metrics), a spike in NIC receive drops (visible in **node_exporter** counters), and a **NCCL** timeout (visible in log tails) may all be symptoms of a single congestion event on the training fabric — but only a unified dashboard that aligns these time series on the same axis makes that causality apparent in seconds rather than hours.

Readers will learn how to deploy the **OTel** Collector as a universal telemetry aggregation point, configure the key exporters for GPU and fabric metrics, write **PromQL** queries that surface common AI cluster failure modes, build **Grafana** dashboards that correlate compute and network signals, and set up alert rules with appropriate hysteresis for production use.

This chapter sits at the intersection of Chapter 15 (**gNMI**/**OpenConfig** streaming telemetry), which provides switch-level counters via **gnmic**, and Chapter 12 (**Cilium**/**eBPF**), whose **Hubble** component feeds pod-level flow metrics into the same **Prometheus** stack. Chapter 22 (Network CI Validation) builds on this observability foundation to automate regression detection in lab and staging environments.

---

## Installation

The observability stack requires several cooperating components, each responsible for a distinct layer of signal collection. The **OpenTelemetry** Collector (**otelcol-contrib**) acts as the central aggregation point, receiving traces, metrics, and logs from every other component before forwarding them to **Prometheus** and **Loki**. **Prometheus** stores and queries time-series metrics; **Grafana** provides the dashboard layer that correlates those metrics into actionable views. The **DCGM** exporter surfaces per-GPU utilization, memory bandwidth, and **NVLink** throughput from NVIDIA's Data Center GPU Manager, **node_exporter** provides host-level CPU, memory, and NIC counters, and **Telegraf** fills gaps for sources that lack native **Prometheus** endpoints — together covering every layer of the AI cluster from the switch fabric up to the GPU compute plane.

### System packages (Ubuntu 24.04)

```bash
# node_exporter (exposes host CPU, memory, NIC counters)
sudo apt install -y prometheus-node-exporter

# Docker Compose plugin (for the full observability stack)
sudo apt install -y docker-compose-plugin

# Pull the OTel Collector contrib image (includes all receivers/exporters)
docker pull otel/opentelemetry-collector-contrib:latest

# Grafana, Prometheus, and Loki are brought up via Docker Compose (see Lab Walkthrough)
```

### Python environment (uv)

```bash
uv venv .venv && source .venv/bin/activate
uv pip install opentelemetry-sdk \
               opentelemetry-exporter-prometheus \
               opentelemetry-exporter-otlp \
               prometheus-client
```

---

## 16.1 The Three Pillars — Applied to AI Clusters

The classical observability model — metrics, logs, traces — maps to concrete problems in AI cluster operations:

| Pillar | AI Cluster Need | Tool |
|---|---|---|
| **Metrics** | GPU utilization, NIC counters, AllReduce throughput over time | Prometheus + DCGM exporter + gnmic |
| **Logs** | NCCL debug output, kernel RDMA events, OOM kills | Loki + Promtail |
| **Traces** | End-to-end latency from training step → AllReduce → network → completion | OpenTelemetry + Tempo |

Prometheus is a pull-based time-series metrics database that scrapes exporters at configurable intervals and stores samples in a local TSDB. Loki is a log aggregation system from Grafana Labs, designed to index only log metadata (labels) while compressing the raw log stream — making it inexpensive to ingest high-volume structured logs such as NCCL debug output. Promtail is the Loki log-shipping agent that tails log files on each host and forwards them to Loki. Grafana Tempo is a distributed tracing backend that stores and queries OpenTelemetry trace data, enabling end-to-end latency analysis across processes and nodes.

The practical insight: most cluster debugging requires correlating all three simultaneously — a GPU stall (metric) coincides with a NCCL timeout (log) coincides with high tail latency on a specific fabric link (trace/metric).

---

## 16.2 OpenTelemetry

OpenTelemetry (OTel) is the CNCF standard for instrumentation and telemetry collection. Its two key components:

- **SDK:** language-specific libraries for instrumenting application code (metrics, traces, logs)
- **Collector:** a vendor-agnostic agent/gateway that receives telemetry from any source, transforms it, and exports to any backend

### 16.2.1 OTel Collector Configuration

```yaml
# otelcol-config.yaml
receivers:
  otlp:                          # receive from instrumented apps
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  prometheus:                    # scrape Prometheus-format metrics
    config:
      scrape_configs:
        - job_name: node_exporter
          scrape_interval: 15s
          static_configs:
            - targets: ["localhost:9100"]
        - job_name: dcgm_exporter
          scrape_interval: 15s
          static_configs:
            - targets: ["localhost:9400"]

  filelog:                       # collect NCCL debug logs
    include: ["/var/log/nccl/*.log"]
    start_at: beginning

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

  resource:
    attributes:
      - key: cluster.name
        value: ai-prod-01
        action: upsert

exporters:
  prometheusremotewrite:
    endpoint: "http://prometheus:9090/api/v1/write"

  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"

  otlp/tempo:
    endpoint: "tempo:4317"
    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch, resource]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp, filelog]
      processors: [batch, resource]
      exporters: [loki]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo]
```

---

## 16.3 Prometheus and Exporters

Prometheus scrapes metrics from *exporters* — small HTTP servers that translate a data source's native instrumentation into the Prometheus text format. The DCGM exporter (Data Center GPU Manager exporter) is NVIDIA's official Prometheus exporter for GPU telemetry; it communicates with the DCGM daemon, which reads GPU counters directly from the driver without performance-monitoring overhead. `gnmic` is a gNMI client that can subscribe to OpenConfig telemetry streams from switches and re-expose the data as a Prometheus metrics endpoint, bridging switch-level telemetry into the same Prometheus stack as host and GPU metrics.

### Key Exporters for AI Clusters

| Exporter | Metrics | Port |
|---|---|---|
| `node_exporter` | CPU, memory, disk, NIC bytes/errors/drops | 9100 |
| `dcgm-exporter` | GPU utilization, memory, temperature, NVLink bandwidth | 9400 |
| `gnmic` (Prometheus output) | Switch interface counters, BGP state | 9273 |
| `process-exporter` | Per-process CPU/memory (NCCL processes, PyTorch) | 9256 |
| `rdma-exporter` (custom) | RDMA port counters via `rdma stat` | custom |

### Kubernetes Deployment

```yaml
# DaemonSet for node_exporter
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        args:
          - --path.sysfs=/host/sys
          - --path.procfs=/host/proc
          - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($|/)
        ports:
        - containerPort: 9100
          hostPort: 9100
        volumeMounts:
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: proc
          mountPath: /host/proc
          readOnly: true
      volumes:
      - name: sys
        hostPath: { path: /sys }
      - name: proc
        hostPath: { path: /proc }
```

### DCGM Exporter for GPU Metrics

```bash
# Deploy dcgm-exporter (requires NVIDIA drivers + DCGM)
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
    --namespace monitoring \
    --set serviceMonitor.enabled=true

# Key metrics:
# DCGM_FI_DEV_GPU_UTIL           — GPU utilization %
# DCGM_FI_DEV_MEM_COPY_UTIL      — memory bandwidth utilization %
# DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL — NVLink bandwidth GB/s
# DCGM_FI_DEV_POWER_USAGE        — power consumption W
# DCGM_FI_DEV_SM_CLOCK           — SM clock frequency MHz
# DCGM_FI_DEV_THERMAL_VIOLATION  — seconds of thermal throttling
```

### Critical PromQL Queries

```promql
# GPU utilization by job/node
avg(DCGM_FI_DEV_GPU_UTIL) by (kubernetes_node, exported_job)

# NIC receive drops rate (congestion indicator)
rate(node_network_receive_drop_total{device=~"eth.*"}[1m])

# RDMA port error rate (fabric health)
rate(infiniband_port_data_receive_packets_total[1m])
  - rate(infiniband_port_data_receive_packets_successful_total[1m])

# Switch link utilization (from gnmic)
rate(gnmi_interface_state_counters_in_octets[30s]) * 8 / 400e9

# Identify imbalanced ECMP (high variance across uplinks)
stddev(rate(gnmi_interface_state_counters_out_octets{name=~"Ethernet[0-9]+"}[1m]))
  by (source)
```

---

## 16.4 Grafana Dashboards

### Essential Dashboard Panels for AI Clusters

**GPU Training Health Dashboard:**
- Time series: GPU utilization per rank (heatmap)
- Time series: NVLink bandwidth per GPU
- Stat: collective operation throughput (from NCCL metrics)
- Alert: GPU utilization drops below 80% for >30s (stall detection)

**Fabric Health Dashboard:**
- Heatmap: interface utilization across all switches (identify hot links)
- Time series: packet drops per switch
- Stat: BGP session states (all green = established)
- Alert: any drop rate > 0 on training fabric interfaces

**Combined Correlation Panel:**
```promql
# GPU stall correlation with network drops — align on same time axis:
# Panel 1: DCGM_FI_DEV_GPU_UTIL
# Panel 2: rate(node_network_receive_drop_total[30s])
# Panel 3: rate(gnmi_interface_state_counters_in_discards[30s])
# Temporal correlation between panels reveals causality
```

---

## 16.5 Hubble Integration

Cilium Hubble (Chapter 12) exposes Kubernetes pod-level network metrics that Prometheus can scrape:

```bash
# Enable Hubble metrics
helm upgrade cilium cilium/cilium \
    --set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp}" \
    --set hubble.metrics.port=9091

# Scrape from Prometheus
- job_name: hubble
  static_configs:
    - targets: ["<node-ip>:9091"]
```

```promql
# NCCL flow volume (pod-to-pod, port 29500)
sum(rate(hubble_flows_processed_total{
    verdict="FORWARDED",
    destination_port="29500"
}[1m]))

# Policy violations (security incidents)
increase(hubble_drop_total{reason="POLICY_DENIED"}[5m])
```

---

## 16.6 Telegraf for Infrastructure Metrics

Telegraf is a plugin-driven metrics collection agent from InfluxData. It supports over 300 input plugins, making it the practical choice for infrastructure sources that lack native Prometheus exporters. SNMP (Simple Network Management Protocol) is the legacy polling protocol used by virtually all network devices to expose interface counters, error rates, and device status via a standardized MIB (Management Information Base) hierarchy. IPMI (Intelligent Platform Management Interface) is a hardware-level management interface that exposes CPU temperature, fan speed, power consumption, and other baseboard sensors independently of the operating system.

Telegraf handles metrics that Prometheus doesn't scrape well — SNMP, IPMI, NIC-specific stats:

```toml
# telegraf.conf
[[inputs.snmp]]
  agents = ["udp://192.168.1.1:161", "udp://192.168.1.2:161"]
  community = "public"
  [[inputs.snmp.table]]
    name = "ifTable"
    oid = "IF-MIB::ifTable"

[[inputs.ipmi_sensor]]
  path = ["ipmitool"]

[[inputs.gnmi]]
  addresses = ["192.168.1.1:57400", "192.168.1.2:57400"]
  username = "admin"
  password = "secret"
  [[inputs.gnmi.subscription]]
    name = "interface"
    path = "/interfaces/interface/state/counters"
    subscription_mode = "sample"
    sample_interval = "10s"

[[outputs.prometheus_client]]
  listen = ":9273"
```

---

## Lab Walkthrough 16 — Full Observability Stack

This walkthrough builds a complete observability pipeline on a single Linux host: node_exporter (host metrics) → OTel Collector → Prometheus + Loki → Grafana. Steps include exact commands, expected output, and verification checks.

### Step 1 — Write the full docker-compose.yaml

Create a working directory and write `docker-compose.yaml`:

```bash
mkdir -p ~/obs-lab && cd ~/obs-lab
```

Full `docker-compose.yaml` content:

```yaml
# ~/obs-lab/docker-compose.yaml
version: "3.9"

networks:
  obs-net:
    driver: bridge

volumes:
  prometheus-data: {}
  grafana-data: {}
  loki-data: {}

services:

  prometheus:
    image: prom/prometheus:v2.51.2
    container_name: prometheus
    restart: unless-stopped
    networks: [obs-net]
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=7d
      - --web.enable-remote-write-receiver   # accept remote_write from OTel Collector

  grafana:
    image: grafana/grafana:10.4.2
    container_name: grafana
    restart: unless-stopped
    networks: [obs-net]
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: "false"

  loki:
    image: grafana/loki:2.9.7
    container_name: loki
    restart: unless-stopped
    networks: [obs-net]
    ports:
      - "3100:3100"
    volumes:
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  otelcol:
    image: otel/opentelemetry-collector-contrib:0.99.0
    container_name: otelcol
    restart: unless-stopped
    networks: [obs-net]
    ports:
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
      - "8888:8888"    # OTel Collector self-metrics
    volumes:
      - ./otelcol-config.yaml:/etc/otelcol-contrib/config.yaml:ro
    extra_hosts:
      - "host.docker.internal:host-gateway"   # reach node_exporter on host
```

### Step 2 — Write otelcol-config.yaml

Full content of `otelcol-config.yaml` (this is the complete configuration):

```yaml
# ~/obs-lab/otelcol-config.yaml
receivers:
  otlp:                          # receive from instrumented apps via SDK
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  prometheus:                    # scrape node_exporter running on the host
    config:
      scrape_configs:
        - job_name: node_exporter
          scrape_interval: 15s
          static_configs:
            - targets: ["host.docker.internal:9100"]

        - job_name: otelcol_self
          scrape_interval: 30s
          static_configs:
            - targets: ["localhost:8888"]

  filelog:                       # tail any NCCL logs if present
    include: ["/var/log/nccl/*.log"]
    start_at: end
    operators:
      - type: regex_parser
        regex: '^\[(?P<timestamp>[^\]]+)\] (?P<body>.*)'
        timestamp:
          parse_from: attributes.timestamp
          layout: '%Y-%m-%dT%H:%M:%S'

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

  resource:
    attributes:
      - key: cluster.name
        value: obs-lab
        action: upsert
      - key: host.name
        from_attribute: host.name
        action: upsert

  memory_limiter:
    check_interval: 5s
    limit_mib: 512
    spike_limit_mib: 128

exporters:
  prometheusremotewrite:
    endpoint: "http://prometheus:9090/api/v1/write"
    tls:
      insecure: true

  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"
    default_labels_enabled:
      exporter: true
      job: true

  debug:
    verbosity: normal            # logs received data to otelcol stdout (useful for debugging)

service:
  telemetry:
    logs:
      level: info
    metrics:
      address: 0.0.0.0:8888

  pipelines:
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch, resource]
      exporters: [prometheusremotewrite]
    logs:
      receivers: [otlp, filelog]
      processors: [batch, resource]
      exporters: [loki]
```

### Step 3 — Write prometheus.yml and start all containers

Write the Prometheus scrape config:

```yaml
# ~/obs-lab/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
```

Start all containers:

```bash
docker compose up -d
```

Expected output:

```
[+] Running 5/5
 ✔ Network obs-lab_obs-net    Created                          0.1s
 ✔ Container prometheus       Started                          1.3s
 ✔ Container loki             Started                          1.4s
 ✔ Container grafana          Started                          1.5s
 ✔ Container otelcol          Started                          1.6s
```

Verify all containers are healthy:

```bash
docker compose ps
```

Expected:

```
NAME         IMAGE                                        COMMAND                  SERVICE     STATUS          PORTS
grafana      grafana/grafana:10.4.2                       "/run.sh"                grafana     Up 30 seconds   0.0.0.0:3000->3000/tcp
loki         grafana/loki:2.9.7                           "/usr/bin/loki -conf…"   loki        Up 30 seconds   0.0.0.0:3100->3100/tcp
otelcol      otel/opentelemetry-collector-contrib:0.99.0  "/otelcol-contrib --…"  otelcol     Up 30 seconds   0.0.0.0:4317->4317/tcp, ...
prometheus   prom/prometheus:v2.51.2                      "/bin/prometheus --c…"   prometheus  Up 30 seconds   0.0.0.0:9090->9090/tcp
```

Check OTel Collector logs for startup errors:

```bash
docker compose logs otelcol | tail -20
```

Expected (no errors):

```
2026-04-22T10:00:01.000Z info  service  Everything is ready. Begin running and processing data.
2026-04-22T10:00:01.000Z info  prometheusreceiver  Starting scrape manager
2026-04-22T10:00:01.000Z info  prometheusreceiver  Scrape job started: node_exporter
```

### Step 4 — Verify node_exporter is running on the host

Check node_exporter service status:

```bash
sudo systemctl status prometheus-node-exporter
```

Expected:

```
● prometheus-node-exporter.service - Prometheus exporter for machine metrics
     Loaded: loaded (/lib/systemd/system/prometheus-node-exporter.service; enabled)
     Active: active (running) since 2026-04-22 10:00:00 UTC
   Main PID: 9876 (prometheus-node)
```

Sample the metrics endpoint:

```bash
curl -s localhost:9100/metrics | head -20
```

Expected output (first 20 lines):

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 4.9351e-05
go_gc_duration_seconds{quantile="0.25"} 7.1534e-05
go_gc_duration_seconds{quantile="0.5"} 0.000108765
go_gc_duration_seconds{quantile="0.75"} 0.000203612
go_gc_duration_seconds{quantile="1"} 0.001043211
go_gc_duration_seconds_sum 0.047382
go_gc_duration_seconds_count 312
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 8
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_cpu_seconds_total{cpu="0",mode="iowait"} 23.45
node_cpu_seconds_total{cpu="0",mode="irq"} 0.00
node_cpu_seconds_total{cpu="0",mode="nice"} 0.01
node_cpu_seconds_total{cpu="0",mode="softirq"} 12.34
node_cpu_seconds_total{cpu="0",mode="steal"} 0.00
```

Key metrics visible: `node_cpu_seconds_total`, `node_network_receive_bytes_total`, `node_network_receive_drop_total`.

### Step 5 — Configure Prometheus scrape targets via prometheus.yml

Update `~/obs-lab/prometheus.yml` to add node_exporter directly (in addition to OTel remote_write):

```yaml
# ~/obs-lab/prometheus.yml  (full updated content)
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Alerting rules (referenced in Step 10)
rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node_exporter
    scrape_interval: 15s
    static_configs:
      - targets: ["host.docker.internal:9100"]
        labels:
          environment: obs-lab
          instance: local-host

  - job_name: otelcol
    scrape_interval: 30s
    static_configs:
      - targets: ["otelcol:8888"]
```

Reload Prometheus to apply:

```bash
docker compose kill -s SIGHUP prometheus
```

Verify node_exporter target is UP in Prometheus:

Open `http://localhost:9090/targets` and confirm:

```
Job: node_exporter
Endpoint: http://host.docker.internal:9100/metrics
State: UP
Last Scrape: 5.2s ago
Labels: environment="obs-lab", instance="local-host", job="node_exporter"
```

### Step 6 — Open Grafana and add the Prometheus datasource

Open `http://localhost:3000` in a browser. Log in with:
- Username: `admin`
- Password: `admin`

Grafana prompts to change the password — skip for the lab or set a new one.

Add the Prometheus datasource:

1. Click the hamburger menu (top left) → **Connections** → **Data sources**
2. Click **Add data source**
3. Select **Prometheus**
4. Set **URL** to `http://prometheus:9090`
5. Under **Scrape interval**, enter `15s`
6. Click **Save & test**

Expected result:

```
Data source connected and labels found.
```

Add the Loki datasource:

1. Click **Add data source** again
2. Select **Loki**
3. Set **URL** to `http://loki:3100`
4. Click **Save & test**

Expected: `Data source connected and labels found.`

### Step 7 — Import the node_exporter dashboard (ID 1860)

The Node Exporter Full dashboard (Grafana ID 1860) provides pre-built panels for CPU, memory, disk, and network metrics.

1. In Grafana, click the **+** icon in the left sidebar → **Import dashboard**
2. In the **Import via grafana.com** field, enter `1860`
3. Click **Load**
4. Under **Prometheus**, select the Prometheus datasource added in Step 6
5. Click **Import**

The dashboard loads with panels including:

- **CPU Busy** — percentage of CPU time not in idle
- **RAM Used** — memory utilization
- **Network Traffic** — `node_network_receive_bytes_total` and `node_network_transmit_bytes_total`
- **Network Sockstat** — TCP connections
- **Disk I/O** — read/write throughput

Verify the **Network Traffic** panel shows non-zero values by inspecting `eth0` or your primary interface.

### Step 8 — Run a drop rate PromQL query in Grafana Explore

Navigate to **Explore** (compass icon in left sidebar). Select **Prometheus** as the datasource.

Enter the following PromQL to compute the interface drop rate:

```promql
rate(node_network_receive_drop_total{device="eth0"}[1m])
```

Click **Run query**.

Interpreting the result:
- At baseline (no congestion), the value should be `0` or very close to `0 drops/second`.
- A non-zero value indicates the kernel is dropping packets at the NIC receive ring — a sign of receive buffer overflow, typically caused by a burst exceeding the NIC ring buffer size or CPU scheduling latency.
- In an AI cluster, sustained values above `0` during AllReduce operations indicate the NIC ring buffer (`ethtool -g eth0`) needs to be increased or IRQ affinity tuned.

Also run the transmit drop rate:

```promql
rate(node_network_transmit_drop_total{device="eth0"}[1m])
```

Transmit drops indicate the transmit queue is full — usually caused by the scheduler not draining the queue fast enough (tune `txqueuelen` with `ip link set eth0 txqueuelen 10000`).

### Step 9 — Simulate congestion with tc netem and watch the drop counter

Add packet loss to the `eth0` interface using the `netem` qdisc (requires `iproute2`, installed by default on Ubuntu):

```bash
sudo tc qdisc add dev eth0 root netem loss 1%
```

Verify the rule is active:

```bash
sudo tc qdisc show dev eth0
```

Expected:

```
qdisc netem 8001: root refcnt 2 limit 1000 loss 1%
```

Generate traffic to trigger drops. Run a continuous ping flood in one terminal:

```bash
ping -f -c 10000 8.8.8.8
```

In another terminal, watch the drop counter increment:

```bash
watch -n 2 'cat /proc/net/dev | grep eth0'
```

Expected (the `drop` column in the TX section increments):

```
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
  eth0: 1234567  10234    0    0    0     0          0         0  9876543   9901    0   99    0     0       0          0
```

In Grafana Explore, the PromQL query `rate(node_network_transmit_drop_total{device="eth0"}[1m])` now shows a non-zero value. Refresh every 15 seconds to see it update.

### Step 10 — Set up a Grafana Alert Rule for drop rate > 0 for 30s

First, create the alert rule file for Prometheus (optional — Grafana-managed alerts are configured in the UI):

In Grafana:

1. Navigate to **Alerting** (bell icon) → **Alert rules** → **New alert rule**

2. Set **Rule name**: `eth0 Receive Drop Rate`

3. Under **Define query and alert condition**, select **Prometheus** datasource and enter:

```promql
rate(node_network_receive_drop_total{device="eth0"}[1m])
```

4. Set the condition: **IS ABOVE** `0`

5. Set **Evaluate every**: `10s`, **For**: `30s`
   - This means the alert fires only if the condition is true for a sustained 30 seconds, avoiding false positives from single-sample spikes.

6. Under **Labels**, add:
   - `severity = warning`
   - `team = network-ops`

7. Under **Annotations**, add:
   - `summary = "eth0 receive drops detected on {{ $labels.instance }}"`
   - `description = "Drop rate: {{ $value | humanize }} drops/s. Investigate NIC ring buffer size and IRQ affinity."`

8. Click **Save rule and exit**.

Wait 30 seconds with the netem rule active. In **Alerting → Alert rules**, the rule transitions:

```
State: Normal → Pending (condition first becomes true)
       Pending → Firing (condition sustained for 30s)
```

The firing state appears in red. In a production environment, this alert would be routed to PagerDuty or Slack via a Contact Point configured under **Alerting → Contact points**.

### Step 11 — Remove the netem rule and watch the alert resolve

Remove the packet loss simulation:

```bash
sudo tc qdisc del dev eth0 root
```

Verify the qdisc is removed:

```bash
sudo tc qdisc show dev eth0
```

Expected:

```
qdisc fq_codel 0: root refcnt 2 limit 10240p flows 1024 quantum 1514 target 5ms interval 100ms memory_limit 32Mb ecn drop_batch 64
```

(The default qdisc is restored — `fq_codel` on Ubuntu 24.04. `fq_codel` is the Fair Queuing Controlled Delay scheduler, a Linux traffic-control qdisc that combines per-flow fair queuing with the CoDel AQM algorithm to actively manage queue depth and minimize bufferbloat under load.)

In Grafana Alerting, the drop rate query returns to `0`. After the next evaluation cycle (10 seconds), the alert transitions:

```
State: Firing → Normal
```

The alert is now resolved. In Grafana's alert history (**Alerting → Alert rules → View alert history**), both the Firing and Resolved events are recorded with timestamps, enabling post-incident review.

### Cleanup

```bash
docker compose down -v
sudo systemctl stop prometheus-node-exporter   # optional — re-enable with 'start'
```

---

## Summary

- OpenTelemetry Collector is the universal aggregation point: it receives metrics from Prometheus exporters, OTLP-instrumented apps, and log files; transforms; and exports to any backend.
- DCGM exporter is essential for GPU cluster observability — GPU utilization, NVLink bandwidth, and thermal violations are leading indicators of training efficiency.
- Correlating GPU stall metrics (DCGM) with network drop metrics (node_exporter, gnmic) in Grafana is the primary debugging workflow for AllReduce performance issues.
- Hubble provides pod-level flow metrics that complement switch-level counters, enabling end-to-end visibility from GPU process to fabric link.

---

## References

- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [OTLP specification](https://opentelemetry.io/docs/specs/otlp/)
- [Prometheus documentation](https://prometheus.io/docs/)
- [OpenMetrics specification](https://openmetrics.io)
- [Grafana documentation](https://grafana.com/docs/)
- [Grafana Loki](https://grafana.com/docs/loki/)
- [Grafana Tempo](https://grafana.com/docs/tempo/)
- [Jaeger distributed tracing](https://www.jaegertracing.io/docs/)
- [DCGM exporter](https://github.com/NVIDIA/dcgm-exporter)
- [Telegraf documentation](https://docs.influxdata.com/telegraf)
- [Cilium Hubble](https://docs.cilium.io/en/stable/observability/hubble/)
- [NCCL documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).