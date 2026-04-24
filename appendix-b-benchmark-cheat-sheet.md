# Appendix B — Benchmark Cheat Sheet

A collection of ready-to-run commands for measuring and validating performance at every layer of the AI cluster network stack. Commands are organized by tool. For each, the most useful variant is shown first, followed by key flags and their meanings. Output interpretation notes follow where the metric is non-obvious.

Unless otherwise noted, commands assume Linux with MOFED or upstream RDMA drivers installed, and require at least two nodes or two terminals (server/client roles are labeled).

---

## B.1 RDMA Bandwidth & Latency — `perftest`

`perftest` is the standard RDMA microbenchmark suite. Install via `apt install perftest` (distro) or via MOFED (`mlnx-ofed`).

### Write Bandwidth (large message, bidirectional)

```bash
# Server (node A):
ib_write_bw -d mlx5_0 --report_gbits -s 65536 -n 5000 -b

# Client (node B):
ib_write_bw -d mlx5_0 --report_gbits -s 65536 -n 5000 -b <server-ip>
```

| Flag | Meaning |
|------|---------|
| `-d mlx5_0` | RDMA device name (`ibstat` to list) |
| `--report_gbits` | Report in Gb/s (not MB/s) |
| `-s 65536` | Message size in bytes (64 KB optimal for bandwidth) |
| `-n 5000` | Number of iterations |
| `-b` | Bidirectional mode |

Expected: ConnectX-7 NDR IB (400 Gb/s) → ~380–395 Gb/s; CX7 400GbE RoCEv2 → ~360–390 Gb/s.

### Write Latency (small message)

```bash
# Server:
ib_write_lat -d mlx5_0 --report_gbits -s 8 -n 10000

# Client:
ib_write_lat -d mlx5_0 --report_gbits -s 8 -n 10000 <server-ip>
```

Expected (back-to-back, no fabric): ~1.0–1.5 µs RTT for CX7 NDR; ~1.5–2.5 µs for RoCEv2 over a single switch hop.

### Read Bandwidth and Send Bandwidth

```bash
# RDMA Read (pull model — client reads from server memory)
ib_read_bw  -d mlx5_0 --report_gbits -s 65536 -n 5000 <server-ip>

# Send/Receive (uses QP send semantics)
ib_send_bw  -d mlx5_0 --report_gbits -s 65536 -n 5000 <server-ip>
ib_send_lat -d mlx5_0 -s 8 -n 10000 <server-ip>
```

### Validate ECN / DCQCN During Load

Run `ib_write_bw` at near-line rate and watch for CNP (Congestion Notification Packet) generation — a sign of switch ECN marking:

```bash
# On sender NIC:
ethtool -S <rdma-netdev> | grep -i cnp

# On receiver NIC (should see non-zero if congested):
ethtool -S <rdma-netdev> | grep -E 'rx_cnp|tx_cnp'
```

---

## B.2 NCCL Collective Throughput — `nccl-tests`

`nccl-tests` measures end-to-end collective performance as seen by the training framework. Build from source against your installed NCCL version.

```bash
# Build (requires CUDA toolkit and NCCL installed):
git clone https://github.com/NVIDIA/nccl-tests && cd nccl-tests
make MPI=1 MPI_HOME=/usr/local/mpi CUDA_HOME=/usr/local/cuda NCCL_HOME=/usr/local/nccl
```

### AllReduce Bandwidth Sweep

```bash
# Single-node, 8 GPUs:
./build/all_reduce_perf -b 8 -e 4G -f 2 -g 8

# Multi-node (2 nodes × 8 GPUs each), via MPI:
mpirun -np 16 -H node1:8,node2:8 \
  -x NCCL_IB_HCA=mlx5_0,mlx5_1 \
  -x NCCL_NET_GDR_LEVEL=5 \
  -x NCCL_DEBUG=INFO \
  ./build/all_reduce_perf -b 8 -e 4G -f 2 -g 8
```

| Flag | Meaning |
|------|---------|
| `-b 8` | Start message size (8 bytes) |
| `-e 4G` | End message size (4 GB) |
| `-f 2` | Message size multiplier per step (×2) |
| `-g 8` | GPUs per MPI process |

Key output column: `algbw` (algorithm bandwidth in GB/s) and `busbw` (bus bandwidth, normalized to the AllReduce pattern — compare this to switch bisection bandwidth).

### AllGather, ReduceScatter, AllToAll

```bash
./build/all_gather_perf   -b 8 -e 2G -f 2 -g 8
./build/reduce_scatter_perf -b 8 -e 2G -f 2 -g 8
./build/alltoall_perf     -b 8 -e 512M -f 2 -g 8
```

### NCCL Environment Tuning Variables

```bash
export NCCL_IB_HCA="mlx5_0,mlx5_1"     # which HCAs to use
export NCCL_IB_GID_INDEX=3              # RoCEv2 GID (IPv4-mapped)
export NCCL_NET_GDR_LEVEL=5            # GPUDirect RDMA: 5 = NVLink
export NCCL_SOCKET_IFNAME=eth0         # fallback TCP interface
export NCCL_DEBUG=INFO                  # verbose: TRACE for per-op logs
export NCCL_ALGO=Ring                   # Ring, Tree, or CollNet
export NCCL_TOPO_FILE=/etc/nccl-topo.xml   # custom topology hint
```

---

## B.3 TCP Throughput & Latency — `iperf3`

`iperf3` remains the quickest sanity check for TCP-layer bandwidth before troubleshooting RDMA.

### Throughput (single stream)

```bash
# Server:
iperf3 -s -p 5201

# Client (10-second test, JSON output):
iperf3 -c <server-ip> -p 5201 -t 10 -J | jq '.end.sum_received.bits_per_second / 1e9'
```

### Parallel Streams (saturate high-speed links)

```bash
# 8 parallel streams to saturate a 100GbE link:
iperf3 -c <server-ip> -P 8 -t 30 -i 5
```

### UDP Latency with `qperf`

```bash
# Server:
qperf

# Client (TCP latency, then bandwidth):
qperf <server-ip> tcp_lat tcp_bw
```

### Netcat RTT (quick baseline, no install needed)

```bash
# Server:
nc -l 12345

# Client (measures time for 1 byte round trip via time):
time echo "" | nc <server-ip> 12345
```

---

## B.4 NVMe-oF Storage — `fio`

`fio` (Flexible I/O Tester) is the standard tool for storage benchmarking. When used against an NVMe-oF target, it exercises the full SPDK or kernel NVMe-oF path.

### Sequential Read Bandwidth (NVMe-oF target)

```bash
# Connect the NVMe-oF target first (kernel NVMe-oF TCP):
nvme connect -t tcp -n nqn.2024-01.io.spdk:cnode1 -a <target-ip> -s 4420

# Confirm device visible:
nvme list

# Sequential read, 128K block, queue depth 64:
fio --name=seq-read \
    --filename=/dev/nvme0n1 \
    --rw=read \
    --bs=128k \
    --numjobs=4 \
    --iodepth=64 \
    --runtime=30 \
    --group_reporting \
    --ioengine=libaio \
    --direct=1
```

### Random 4K IOPS (latency-sensitive checkpoint write)

```bash
fio --name=rand-write \
    --filename=/dev/nvme0n1 \
    --rw=randwrite \
    --bs=4k \
    --numjobs=8 \
    --iodepth=32 \
    --runtime=30 \
    --group_reporting \
    --ioengine=libaio \
    --direct=1
```

### Checkpoint Simulation (large sequential write)

LLM checkpoints are large sequential writes, typically 1–64 GB per rank, issued simultaneously from all GPU workers:

```bash
fio --name=checkpoint-sim \
    --filename=/mnt/checkpoint/ckpt-$RANK \
    --rw=write \
    --bs=1m \
    --numjobs=1 \
    --iodepth=8 \
    --size=8G \
    --group_reporting \
    --ioengine=libaio \
    --direct=1
```

Run this simultaneously on all nodes to simulate incast on the storage fabric. Watch for bandwidth collapse compared to single-node baseline.

### SPDK Benchmark (user-space NVMe-oF)

```bash
# Using SPDK's built-in perf tool against an NVMe-oF/RDMA target:
/opt/spdk/build/bin/nvme_perf \
  -r "trtype:rdma adrfam:IPv4 traddr:<target-ip> trsvcid:4420 subnqn:nqn.2024-01.io.spdk:cnode1" \
  -q 128 -o 131072 -w read -t 30
```

---

## B.5 gNMI Streaming Telemetry — `gnmic`

`gnmic` is the reference gNMI CLI. Install: `go install github.com/openconfig/gnmic@latest` or use the Docker image `ghcr.io/openconfig/gnmic`.

### Subscribe to Interface Counters (STREAM mode)

```bash
gnmic -a <device-ip>:57400 \
      --username admin --password admin \
      --insecure \
      subscribe \
      --path "/interface[name=ethernet-1/1]/statistics/in-octets" \
      --path "/interface[name=ethernet-1/1]/statistics/out-octets" \
      --mode stream \
      --stream-mode sample \
      --sample-interval 10s
```

### Get BGP Session State (one-shot)

```bash
gnmic -a <device-ip>:57400 --insecure get \
  --path "/network-instance[name=default]/protocols/bgp/neighbors/neighbor[peer-address=10.0.1.1]/session-state"
```

### Subscribe to All Interface Errors (SR Linux)

```bash
gnmic -a <srlinux-ip>:57400 --insecure subscribe \
  --path "/interface[name=*]/statistics/in-error-packets" \
  --path "/interface[name=*]/statistics/out-error-packets" \
  --mode stream --stream-mode on-change
```

### Collect into Prometheus via Telegraf

```toml
# telegraf.conf snippet:
[[inputs.gnmi]]
  addresses = ["<device-ip>:57400"]
  username = "admin"
  password = "admin"
  redial = "10s"

  [[inputs.gnmi.subscription]]
    name = "interface_counters"
    path = "/interface/statistics"
    subscription_mode = "sample"
    sample_interval = "10s"

[[outputs.prometheus_client]]
  listen = ":9273"
```

---

## B.6 Prometheus PromQL — GPU Cluster Dashboards

These queries assume the standard exporter stack: `node_exporter` on each host, `dcgm-exporter` for GPU metrics, and a gNMI/SNMP exporter for switch metrics. Metric names follow the exporters' default naming conventions.

### Network Throughput (bits/sec, per interface)

```promql
# Rx throughput on all eth1 interfaces across nodes:
rate(node_network_receive_bytes_total{device="eth1"}[1m]) * 8

# Top-5 highest Tx interfaces across all switch ports:
topk(5, rate(ifOutOctets[1m]) * 8)
```

### RDMA Error Rate

```promql
# CNP (Congestion Notification Packets) rate — DCQCN congestion indicator:
rate(rdma_hw_cnp_sent_total[1m])

# Packet retransmissions per NIC port:
rate(rdma_hw_np_cnp_sent[1m])

# RoCE RNR NAK (Receiver Not Ready) — queue pressure:
rate(rdma_hw_rnr_nak_retry_err[1m])
```

### GPU Utilization & Memory (DCGM Exporter)

```promql
# Mean GPU utilization across all GPUs in a job:
avg by (kubernetes_pod_name) (DCGM_FI_DEV_GPU_UTIL)

# GPU memory used (GB):
DCGM_FI_DEV_FB_USED / 1024

# NVLink bandwidth (RX, GB/s) — inter-GPU within node:
rate(DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL[1m]) / 1e9
```

### AllReduce Timing Alert

```promql
# Alert if NCCL AllReduce 99th-percentile exceeds 500ms
# (requires custom NCCL timing metrics emitted via OpenTelemetry):
histogram_quantile(0.99,
  rate(nccl_allreduce_duration_seconds_bucket[5m])
) > 0.5
```

### Switch Buffer Utilization (SONiC SAI counters via SNMP exporter)

```promql
# Per-port queue depth (frames in buffer):
snmp_queue_buffer_usage_bytes{port=~"Ethernet.*", queue="3"}

# Buffer watermark (high-water mark in last scrape interval):
snmp_ingress_buffer_watermark_bytes
```

### Disk I/O — Checkpoint Throughput

```promql
# Write throughput to checkpoint mount across all nodes (MB/s):
rate(node_disk_written_bytes_total{device="nvme0n1"}[1m]) / 1e6

# I/O wait — a spike here indicates storage becoming a training bottleneck:
rate(node_cpu_seconds_total{mode="iowait"}[1m])
```

---

## B.7 Quick Reference — Baseline Expectations

| Test | Tool | Expected (good) |
|------|------|-----------------|
| RDMA write BW (CX7 NDR IB) | `ib_write_bw` | ≥ 380 Gb/s |
| RDMA write BW (CX7 400GbE RoCEv2) | `ib_write_bw` | ≥ 360 Gb/s |
| RDMA write latency (1 switch hop) | `ib_write_lat` | ≤ 2.5 µs |
| NCCL AllReduce busbw (8× NDR) | `all_reduce_perf` | ≥ 300 GB/s per node |
| TCP throughput (100GbE) | `iperf3 -P 8` | ≥ 90 Gb/s |
| NVMe-oF seq read BW (1 target) | `fio` | ≥ 5 GB/s |
| NVMe-oF random 4K IOPS | `fio` | ≥ 300k IOPS |
| gNMI subscribe latency | `gnmic` | ≤ 100 ms first notification |


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).