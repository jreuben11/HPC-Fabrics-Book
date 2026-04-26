# Chapter 28 — Fault Tolerance and Resilience in AI Network Fabrics

**Part VII: Testing, Emulation & Simulation** | ~20 pages

---

## Introduction

Distributed training jobs fail differently from web services. A web service can tolerate one backend going down because requests are independent — the load balancer simply stops sending traffic to the failed backend. A training job is a single synchronous computation spread across hundreds of GPUs, and every rank must complete every AllReduce before any rank can advance. When one rank's network link fails, every other rank blocks. When all ranks block, the entire cluster's GPU time is wasted until either the job dies or a watchdog fires.

This chapter builds a complete fault-tolerance stack for AI network fabrics, working layer by layer from the physical link to the training framework. The starting point is the threat model (§28.1): understanding exactly how network faults translate into job failures, why the default timeout intervals are catastrophically long for GPU clusters, and what the three failure modes are — RDMA QP retry exhaustion, AllReduce barrier hang, and checkpoint corruption — and how they interact.

From the network layer, the chapter covers BFD (§28.2), the protocol that reduces link-failure detection from 90 seconds (BGP keepalive default) to 300 milliseconds. With BFD, the routing plane knows about a link failure before NCCL's collective timeout fires, enabling clean route withdrawal and path failover rather than rank timeout and job restart. Section 28.3 completes the RDMA layer by explaining QP timeout and retry parameters — the four attributes that determine how long a Queue Pair waits before declaring a remote unreachable and surfacing an error to NCCL. Section 28.4 addresses PFC Watchdog, which prevents the lossless fabric itself from entering a deadlocked state where no packets flow at all.

The application-layer sections (§28.5–28.8) cover recovery through `torchrun` elastic training, checkpoint strategies that survive partial rank failure, circuit-breaker patterns for collective operations, and the observability infrastructure needed to correlate network events with training failures. The chapter concludes with an end-to-end lab that chains BFD, torchrun restarts, and a Prometheus RDMA exporter into a single fault injection and recovery demonstration.

This chapter connects directly to Chapter 2 (RoCEv2 and RDMA Queue Pairs), Chapter 27 (adaptive routing to reduce congestion before it causes failures), and Chapter 15 (gNMI telemetry, which provides the on-change subscription used in §28.8 to detect BFD state transitions). Chapter 19 (GPU collective communications) provides the NCCL context that makes the timeout and recovery behaviors meaningful.

## 28.1 Why Network Faults Kill Training Jobs

A long-running LLM training job is, at its core, a tightly synchronized distributed computation. Every training step requires all ranks to complete a forward pass, an AllReduce across gradients, and an optimizer step before any rank can proceed to the next step. This barrier synchronization means that a single slow or failed rank stalls the entire job.

Network faults manifest in several distinct failure modes, each with a different detection time and recovery path:

### RDMA Timeout Cascades

InfiniBand and RoCEv2 Queue Pairs have built-in retry and timeout parameters. When a link fails, in-flight RDMA operations wait for ACKs that never arrive. The QP's retry counter decrements with each retransmission attempt. Once exhausted, the QP transitions to an error state and NCCL or UCX propagates the error upward:

```
nccl WARN Call to ibv_poll_cq returned error 5 : Input/output error
nccl WARN rank 3: Transport error 5 on operation SEND
NCCL error in: /opt/conda/lib/python3.10/site-packages/torch/csrc/distributed/c10d/ProcessGroupNCCL.cpp:1785
ncclInternalError: Internal check failed.
```

Without fast link-failure detection, NCCL waits for the full QP retry timeout (which defaults to approximately 14 seconds at `timeout=14` on the QP attribute) before surfacing an error. During that wait, every other rank is blocked at the AllReduce barrier.

### AllReduce Barrier Hang

PyTorch Distributed's `dist.all_reduce()` call is a synchronous collective: it does not return until all ranks have contributed their tensors. If one rank fails silently (e.g., OOM kill, NIC firmware hang, or silent packet loss that exhausts retries without a detectable error), the surviving ranks block indefinitely unless a watchdog timeout fires first.

The `NCCL_TIMEOUT` environment variable sets the watchdog timeout in milliseconds. The default is 30 minutes (1,800,000 ms) — long enough for the compute team to notice a "stuck" job before the system acts on it. Reducing `NCCL_TIMEOUT` to a few tens of seconds cuts the blast radius.

### Checkpoint Loss

If a rank failure occurs in the middle of a checkpoint write — particularly a synchronous write where all ranks write in lockstep — the partially written checkpoint may be corrupted. Resuming from a corrupted checkpoint causes the job to fail again immediately on restart. Async checkpointing and atomic rename patterns prevent this.

---

---

## Installation

FRR is the central dependency, providing both the BFD daemon (`bfdd`) for sub-second link-failure detection and the BGP daemon (`bgpd`) that withdraws routes once BFD declares a session down. The `rdma-core` and `perftest` packages supply the `ibv_*` verbs libraries and tools such as `ib_write_bw` that are used to inject RDMA traffic and observe QP retry and timeout behavior under simulated link loss. Prometheus and Alertmanager are deployed as Docker containers to collect RDMA counters exported by the `rdma-exporter` and fire alerts when retry rates exceed thresholds. PyTorch with `torchrun` is required for the elastic training recovery lab — it is assumed to be present in the Python environment or can be installed via pip, as it is too large for an apt package.

### System packages

```bash
sudo apt install -y frr iproute2 tcpdump rdma-core libibverbs-dev perftest \
    containerlab jq bc
```

Enable FRR daemons (BFD and BGP are needed for this chapter):

```bash
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo sed -i 's/^bfdd=no/bfdd=yes/' /etc/frr/daemons
sudo systemctl enable --now frr
```

### Python environment

```bash
uv venv .venv && source .venv/bin/activate
uv pip install torch torchvision paramiko prometheus-client
# paramiko: pure-Python SSH2 library, used here to programmatically inject faults via SSH into network nodes
# prometheus-client: official Python Prometheus client library for exposing metrics via an HTTP /metrics endpoint
python -c "import torch, paramiko, prometheus_client; print('OK')"
```

### Verify RDMA tooling

`rdma-core` is the Linux RDMA userspace library stack, and `libibverbs-dev` provides the `ibverbs` API headers and development libraries for writing RDMA applications; together they supply the `rdma`, `ibv_devinfo`, and related tools. `perftest` is a collection of RDMA performance test utilities (including `ib_send_bw`, `ib_read_bw`, and `ib_write_bw`) used to benchmark and verify RDMA link throughput and latency.

```bash
rdma link show
# link mlx5_0/1 state ACTIVE physical_state POLLING netdev enp1s0f0
ibv_devinfo | head -4
# hca_id: mlx5_0
#   transport:                      InfiniBand (0)
#   fw_ver:                         20.35.1012
#   node_guid:                      e41d:2d03:0065:c1e0
```

---

## 28.2 BFD — Bidirectional Forwarding Detection

BFD (RFC 5880) is a lightweight, protocol-independent liveness detection mechanism. Two peers exchange BFD control packets at a negotiated interval; if packets stop arriving for `(min_rx * multiplier)` milliseconds, the session is declared Down and the associated routing protocol is notified immediately — without waiting for BGP keepalive timers (default hold-time 90 seconds) to expire.

### BFD Timer Arithmetic

```
Detection time = negotiated_rx_interval × detect_multiplier

Example with aggressive timers:
  min-tx = 100ms,  min-rx = 100ms,  multiplier = 3
  Detection time = 100ms × 3 = 300ms
```

The negotiated interval is `max(local_min_tx, remote_min_rx)`. Both sides advertise their minimums; the slower side wins.

### BFD in FRR

FRR's BFD daemon (`bfdd`) integrates with `bgpd` via a Unix socket. Enabling BFD on a BGP session is a one-line addition to the neighbor stanza:

```
! /etc/frr/frr.conf
router bgp 65001
  neighbor 192.168.0.2 remote-as 65002
  neighbor 192.168.0.2 bfd
  neighbor 192.168.0.2 bfd profile fast-link

bfd
  profile fast-link
    receive-interval 100
    transmit-interval 100
    detect-multiplier 3
  !
!
```

Verify session state in `vtysh`:

```bash
vtysh -c "show bfd peers"
```

```
BFD Peers:
        peer 192.168.0.2 local-address 192.168.0.1 vrf default interface eth0
                ID: 2159544756
                Remote ID: 3844201971
                Active mode
                Status: up
                Uptime: 3 minute(s), 14 second(s)
                Diagnostics: ok
                Remote diagnostics: ok
                Peer Type: dynamic
                RTT min/avg/max: 0/0/0 usec
                Local timers:
                        Detect-multiplier: 3
                        Receive interval: 100ms
                        Transmission interval: 100ms
                        Echo receive interval: 50ms
                        Echo transmission interval: disabled
                Remote timers:
                        Detect-multiplier: 3
                        Receive interval: 100ms
                        Transmission interval: 100ms
                        Echo receive interval: 50ms
```

### BFD Integration with BGP Convergence

When the BFD session goes Down, `bfdd` notifies `bgpd` synchronously. `bgpd` immediately withdraws routes learned from that peer and re-runs best-path selection. The entire sequence — BFD detection + BGP withdrawal + FIB update — completes in under 500ms with the timers above, compared to 90+ seconds with BGP keepalives alone.

---

## 28.3 RDMA Timeout and Retry Tuning

Each Queue Pair (QP) carries retry parameters set at connection time via `ibv_modify_qp`. Understanding these is essential for tuning the tradeoff between false-positive error detection and slow-failure recovery.

### QP Attributes for Reliability

```c
struct ibv_qp_attr attr = {
    .timeout         = 14,   /* Local ACK timeout: 4.096µs × 2^timeout ≈ 67ms */
    .retry_cnt       = 7,    /* Retransmit up to 7 times before error */
    .rnr_retry       = 7,    /* RNR retry count; 7 = infinite retries */
    .min_rnr_timer   = 12,   /* RNR NAK timer: ~640µs */
};
```

**timeout** is an exponent: actual timeout = `4.096µs × 2^timeout`. At `timeout=14`, one ACK timeout is about 67ms. With `retry_cnt=7`, the QP will wait up to `7 × 67ms ≈ 470ms` before declaring the remote unreachable.

**RNR (Receiver Not Ready)** occurs when a SEND arrives at a QP before the receiver has posted a receive buffer. `rnr_retry=7` means infinite retries for RNR (a value of 7 is the special "infinite" sentinel in the verbs API). Infinite RNR retry is appropriate for GPUDirect traffic where the receiver may legitimately be slower.

### Querying Error Counters

```bash
# Per-port hardware counters (Mellanox/NVIDIA NICs)
rdma stat show dev mlx5_0 port 1
```

```
MR   hw_rereg_mr_cnt    0
SRQ  hw_srq_err         0
QP   hw_duplicate_request 0
     hw_out_of_sequence    0
     hw_rnr_nak_retry_err  0
     hw_packet_seq_err     0
     hw_remote_access_err  0
     hw_remote_invalid_req 0
     hw_local_ack_timeout_err 3
     hw_resp_local_len_err 0
```

`local_ack_timeout_err` is the key counter for link-induced failures. A nonzero and growing value indicates retransmits are being exhausted — usually caused by switch congestion, PFC deadlock, or a flapping link.

### Mellanox ethtool Counters

```bash
ethtool -S enp1s0f0 | grep -E 'timeout|retry|rnr|err'
```

```
     rx_buffer_passed_thres_phy: 0
     rx_pcs_symbol_err_phy: 0
     local_ack_timeout_err_phy: 3
     resp_local_length_error_phy: 0
     req_cqe_error_phy: 0
     rx_rnr_nak_retry_err: 0
```

---

## 28.4 PFC Watchdog — Preventing Lossless Fabric Deadlocks

Priority Flow Control (PFC) is essential for lossless RoCEv2 fabrics: when a receiver's buffer is full, it sends a PAUSE frame that stops the sender for a configurable duration, preventing drops. However, PFC creates a hazard: if circular dependencies form between PFC-paused flows, the fabric deadlocks. Every switch buffers traffic waiting for a neighbor to unpause, but every neighbor is also paused. Traffic stops flowing entirely.

### PFC Storm Detection

A PFC storm is a pathological state where PAUSE frames propagate across multiple hops, causing widespread backpressure. Symptoms:
- Sudden collapse in RDMA throughput across the entire job
- `rdma stat` showing zero completions for tens of seconds
- Switch telemetry showing PFC pause frames on every port

Mellanox ConnectX NICs implement a **PFC watchdog** that automatically detects when a PFC-paused state has persisted beyond a threshold and disables PFC on the offending priority class, allowing traffic to drain (with some drops) rather than staying deadlocked.

### Configuring the PFC Watchdog

```bash
# Show current PFC watchdog configuration
mlnx_qos -i enp1s0f0 --pfc-wd

# Set watchdog polling interval (microseconds) and enable
mlnx_qos -i enp1s0f0 --pfc-wd-polling-interval 100
mlnx_qos -i enp1s0f0 --pfc-wd-action drop   # or 'forward' to disable PFC temporarily

# Per-priority-class PFC watchdog timer (100ms threshold for priority 3)
mlnx_qos -i enp1s0f0 --pfc-wd-detect-time 100 --tc 3
mlnx_qos -i enp1s0f0 --pfc-wd-restore-time 200 --tc 3
```

After automatic disable:

```bash
ethtool -S enp1s0f0 | grep pfc_wd
```

```
     pfc_wd_rx_pause_duration_msec: 0
     pfc_wd_rx_pause_transition_count: 1
     pfc_wd_rx_pause_detect_count: 1
     pfc_wd_tx_pause_duration_msec: 43
     pfc_wd_tx_pause_transition_count: 1
     pfc_wd_tx_pause_detect_count: 1
```

`detect_count > 0` indicates the watchdog fired. The `restore_time` parameter controls how long after the storm clears before PFC is re-enabled on that priority.

---

## 28.5 Training Job Recovery with torchrun

PyTorch's `torchrun` (previously `torch.distributed.launch`) supports elastic training via the `torch.distributed.elastic` module (c10d elastic). Rank failure triggers automatic re-rendezvous on a new world, reducing rank count within the configured `--min-nodes` and `--max-nodes` bounds, or restarting the full job with `--max-restarts`.

### torchrun Elastic Parameters

```bash
torchrun \
    --nnodes=1:4 \               # min:max nodes (elastic)
    --nproc-per-node=8 \         # GPUs per node
    --max-restarts=3 \           # restart the full job up to 3 times
    --rdzv-backend=c10d \        # use c10d (etcd-free) rendezvous
    --rdzv-endpoint=192.168.0.1:29400 \
    train.py
```

### c10d Store Failover

The c10d rendezvous backend stores rank membership in a `c10d::TCPStore` on the coordinator node. When a rank fails:

1. The failed rank's TCP connection to the store drops.
2. The store detects the missing heartbeat and marks the rank as departed.
3. Surviving ranks receive a `WorkerGroupFailure` exception in their NCCL call.
4. `torchrun` catches the exception, increments the restart counter, and re-launches all workers.
5. Workers call `init_process_group()` again with a fresh `store_id`, forming a new world.

### NCCL Timeout

```bash
export NCCL_TIMEOUT=30000    # 30 seconds (milliseconds)
export NCCL_ASYNC_ERROR_HANDLING=1   # surface errors asynchronously
```

`NCCL_ASYNC_ERROR_HANDLING=1` is critical: without it, a timeout in a background NCCL operation may not surface as a Python exception, leaving the rank alive but stuck. With it, the exception propagates at the next `dist` call, allowing `torchrun` to catch and restart.

### Re-rendezvous Sequence

```
t=0s    : Rank 3 NIC goes down (link failure detected by BFD in 300ms)
t=0.3s  : BGP withdraws routes to rank 3's node
t=30s   : NCCL_TIMEOUT fires on ranks 0,1,2 waiting at AllReduce barrier
t=30s   : WorkerGroupFailure raised on all surviving ranks
t=30.1s : torchrun catches exception, increments restart_count (1/3)
t=30.2s : torchrun kills all worker processes
t=30.3s : torchrun relaunches workers with --nnodes=1 (3 ranks on 3 nodes)
t=31s   : Ranks 0,1,2 complete init_process_group() with new store_id
t=31.5s : Training resumes from last checkpoint
```

The elapsed time from fault to resumption is dominated by `NCCL_TIMEOUT`. Tuning this down to 10–15 seconds (and relying on BFD + BGP for fast convergence) minimizes job disruption.

---

## 28.6 Checkpoint Strategies for Network Resilience

### Synchronous vs Asynchronous Checkpointing

Synchronous checkpointing blocks all ranks until every rank has written its shard to storage. This is simple but creates a correlated failure risk: if the storage fabric or one rank's write stalls, the entire job waits. Synchronous checkpoints also create a bursty I/O pattern that can cause storage congestion.

Async checkpointing decouples the checkpoint write from the training loop. A background thread serializes the state dict to a BytesIO buffer in memory (fast, non-blocking), then writes to storage while the next training step begins:

```python
import threading
import torch

def async_checkpoint(state_dict: dict, path: str) -> threading.Thread:
    """Serialize state dict in background; returns join-able thread."""
    # Snapshot tensors to CPU in the foreground (must happen before next step)
    cpu_state = {k: v.cpu() for k, v in state_dict.items()}

    def _write():
        torch.save(cpu_state, path + ".tmp")
        os.rename(path + ".tmp", path)   # atomic rename: either old or new, never partial

    t = threading.Thread(target=_write, daemon=True)
    t.start()
    return t

# In the training loop:
# ckpt_thread = async_checkpoint(model.state_dict(), f"ckpt_{step}.pt")
# ... run next training step ...
# ckpt_thread.join()   # wait only if next save interval arrives before write completes
```

The atomic `os.rename()` from `.tmp` to final path ensures readers never see a partially written file.

### Check-and-Restart Pattern

```python
import os, torch, torch.distributed as dist

CKPT_PATH = "checkpoint.pt"

def load_checkpoint(model, optimizer):
    if os.path.exists(CKPT_PATH):
        state = torch.load(CKPT_PATH, map_location="cpu")
        model.load_state_dict(state["model"])
        optimizer.load_state_dict(state["optimizer"])
        return state["step"]
    return 0

def save_checkpoint(model, optimizer, step):
    state = {"model": model.state_dict(),
             "optimizer": optimizer.state_dict(),
             "step": step}
    torch.save(state, CKPT_PATH + ".tmp")
    os.rename(CKPT_PATH + ".tmp", CKPT_PATH)

# On startup (after init_process_group):
start_step = load_checkpoint(model, optimizer)
for step in range(start_step, MAX_STEPS):
    loss = train_step(...)
    if step % CKPT_INTERVAL == 0 and dist.get_rank() == 0:
        save_checkpoint(model, optimizer, step)
    dist.barrier()   # all ranks sync before proceeding past checkpoint
```

### DMTCP for Transparent Checkpointing

DMTCP (Distributed MultiThreaded CheckPointing) can checkpoint arbitrary multi-process applications — including NCCL-backed training jobs — without source code modification. It intercepts `fork()`, `exec()`, and socket operations at the `LD_PRELOAD` level:

```bash
# Checkpoint a running torchrun job
dmtcp_command --checkpoint

# Restart from checkpoint
dmtcp_restart ckpt_*.dmtcp
```

DMTCP is particularly valuable for legacy training code that cannot easily be modified for application-level checkpointing.

---

## 28.7 Watchdog and Circuit-Breaker Patterns

### Health-Probe Sidecar

In Kubernetes deployments, a sidecar container can probe collective health and trigger pod restart before `NCCL_TIMEOUT` fires. The sidecar writes a heartbeat file that the main training process updates each step; if the sidecar detects a stale heartbeat, it signals the main process:

```python
# heartbeat_watchdog.py — run as sidecar
import time, os, signal, sys

HEARTBEAT_FILE = "/shared/heartbeat"
MAX_STALE_SECS  = 60   # kill if no update for 60 seconds
TRAINING_PID    = int(sys.argv[1])

while True:
    time.sleep(10)
    try:
        mtime = os.path.getmtime(HEARTBEAT_FILE)
    except FileNotFoundError:
        continue
    age = time.time() - mtime
    if age > MAX_STALE_SECS:
        print(f"[watchdog] Heartbeat stale for {age:.0f}s — sending SIGTERM to {TRAINING_PID}")
        os.kill(TRAINING_PID, signal.SIGTERM)
        sys.exit(1)
```

### Exponential Backoff for Reconnect

When a rank reconnects after a restart, it should not hammer the rendezvous store immediately — all restarting ranks doing so simultaneously creates a thundering herd. `torchrun` implements jittered exponential backoff internally, but custom rendezvous backends should do the same:

```python
import time, random

def connect_with_backoff(fn, max_retries=10, base_delay=1.0, max_delay=60.0):
    """Attempt fn() with jittered exponential backoff."""
    for attempt in range(max_retries):
        try:
            return fn()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            delay = min(max_delay, base_delay * (2 ** attempt))
            jitter = random.uniform(0, delay * 0.1)
            print(f"[retry] attempt {attempt+1} failed: {e}; retrying in {delay+jitter:.1f}s")
            time.sleep(delay + jitter)
```

### Circuit Breaker for Collective Operations

A circuit breaker wraps a collective operation and trips open after N consecutive failures, preventing a partially-recovered cluster from endlessly retrying:

```python
class CollectiveCircuitBreaker:
    CLOSED, OPEN, HALF_OPEN = "CLOSED", "OPEN", "HALF_OPEN"

    def __init__(self, failure_threshold=3, recovery_timeout=30):
        self.state = self.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self._open_time = 0

    def call(self, fn, *args, **kwargs):
        import time
        if self.state == self.OPEN:
            if time.time() - self._open_time > self.recovery_timeout:
                self.state = self.HALF_OPEN
            else:
                raise RuntimeError("Circuit OPEN — collective operations suspended")
        try:
            result = fn(*args, **kwargs)
            if self.state == self.HALF_OPEN:
                self.state = self.CLOSED
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            if self.failure_count >= self.failure_threshold:
                self.state = self.OPEN
                self._open_time = time.time()
            raise
```

---

## 28.8 Observability for Fault Detection

### RDMA Error Counter Alerts in Prometheus

A Prometheus exporter that scrapes `rdma stat` output and exports error counters enables alerting on `local_ack_timeout_err` growth rate:

```python
# rdma_exporter.py — lightweight Prometheus exporter for RDMA counters
import subprocess, time
from prometheus_client import start_http_server, Gauge

COUNTERS = [
    "local_ack_timeout_err",
    "out_of_sequence",
    "duplicate_request",
    "rnr_nak_retry_err",
    "packet_seq_err",
]

gauges = {c: Gauge(f"rdma_{c}_total", f"RDMA counter: {c}",
                   ["dev", "port"]) for c in COUNTERS}

def scrape_rdma():
    result = subprocess.run(["rdma", "stat", "show"], capture_output=True, text=True)
    dev, port = "mlx5_0", "1"   # parse dynamically in production
    for line in result.stdout.splitlines():
        for counter in COUNTERS:
            if counter in line:
                val = int(line.split()[-1])
                gauges[counter].labels(dev=dev, port=port).set(val)

if __name__ == "__main__":
    start_http_server(9101)
    print("RDMA exporter listening on :9101")
    while True:
        scrape_rdma()
        time.sleep(5)
```

Prometheus alert rule:

```yaml
# rdma_alerts.yaml
groups:
- name: rdma_fabric
  rules:
  - alert: RDMATimeoutErrRising
    expr: rate(rdma_local_ack_timeout_err_total[2m]) > 1
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "RDMA ACK timeout errors rising on {{ $labels.dev }}"
      description: "Rate {{ $value | humanize }}/s — possible link instability"
```

### BFD State Change via gNMI

OpenConfig models include BFD session state under `openconfig-bfd:bfd`. A gNMI subscription fires an update on every state transition. `gnmic` is an open-source gNMI client CLI tool (by Nokia) that supports Subscribe, Get, and Set RPCs against any gNMI-capable network device, making it the standard command-line interface for on-change telemetry subscriptions.

```bash
gnmic subscribe \
    --address 192.168.0.1:9339 \
    --path "/bfd/interfaces/interface[id=eth0]/peers/peer[local-discriminator=*]/state/session-state" \
    --mode stream \
    --stream-mode on-change
```

```
{
  "source": "192.168.0.1:9339",
  "subscription-name": "default-1715001234",
  "timestamp": 1715001234000000000,
  "time": "2026-04-22T10:13:54Z",
  "updates": [{
    "Path": "bfd/interfaces/interface[id=eth0]/peers/peer[...]/state/session-state",
    "values": {
      "bfd/interfaces/interface[id=eth0]/.../session-state": "DOWN"
    }
  }]
}
```

### Correlating NCCL Timeouts with Link Events

A practical correlation script joins three event streams: BFD state changes (from gNMI), RDMA error counter spikes (from Prometheus), and NCCL timeout log lines (from pod logs), using timestamp proximity to infer causality:

```python
# correlate_events.py
import re, datetime

BFD_RE   = re.compile(r'(\d{4}-\d{2}-\d{2}T[\d:]+).*BFD.*session.*DOWN')
NCCL_RE  = re.compile(r'(\d{4}-\d{2}-\d{2}T[\d:]+).*NCCL.*timeout')
RDMA_RE  = re.compile(r'(\d{4}-\d{2}-\d{2}T[\d:]+).*local_ack_timeout')

def parse_events(logfile, pattern):
    events = []
    with open(logfile) as f:
        for line in f:
            m = pattern.search(line)
            if m:
                ts = datetime.datetime.fromisoformat(m.group(1))
                events.append((ts, line.strip()))
    return sorted(events)

bfd_events  = parse_events("gnmi.log",   BFD_RE)
nccl_events = parse_events("rank0.log",  NCCL_RE)
rdma_events = parse_events("rdma.log",   RDMA_RE)

WINDOW = datetime.timedelta(seconds=35)   # BFD detect (0.3s) + NCCL_TIMEOUT (30s) + margin

for bfd_ts, bfd_msg in bfd_events:
    print(f"[LINK DOWN ] {bfd_ts} — {bfd_msg[:80]}")
    for rdma_ts, rdma_msg in rdma_events:
        if bfd_ts <= rdma_ts <= bfd_ts + WINDOW:
            print(f"  [RDMA ERR ] {rdma_ts} (+{(rdma_ts-bfd_ts).seconds}s) — {rdma_msg[:60]}")
    for nccl_ts, nccl_msg in nccl_events:
        if bfd_ts <= nccl_ts <= bfd_ts + WINDOW:
            print(f"  [NCCL TIMEOUT] {nccl_ts} (+{(nccl_ts-bfd_ts).seconds}s) — {nccl_msg[:60]}")
```

---

## Lab Walkthrough 28 — BFD Link Failure Detection and NCCL Timeout Recovery

This lab builds a 3-node Containerlab topology (one spine, two leaves) running FRR, demonstrates sub-second BFD-accelerated BGP convergence, then exercises PyTorch distributed fault recovery through `torchrun --max-restarts`.

### Step 1 — Create a Containerlab 3-node FRR topology and deploy

Create `bfd-lab.yaml`:

```yaml
# bfd-lab.yaml
name: bfd-nccl
topology:
  nodes:
    spine1:
      kind: linux
      image: frrouting/frr:v9.1.0
      binds:
        - spine1-frr.conf:/etc/frr/frr.conf
    leaf1:
      kind: linux
      image: frrouting/frr:v9.1.0
      binds:
        - leaf1-frr.conf:/etc/frr/frr.conf
    leaf2:
      kind: linux
      image: frrouting/frr:v9.1.0
      binds:
        - leaf2-frr.conf:/etc/frr/frr.conf
  links:
    - endpoints: ["spine1:eth1", "leaf1:eth1"]
    - endpoints: ["spine1:eth2", "leaf2:eth1"]
```

Create `spine1-frr.conf`:

```
! spine1-frr.conf
frr defaults datacenter
hostname spine1
log syslog informational

interface eth1
  ip address 10.0.1.0/31
!
interface eth2
  ip address 10.0.2.0/31
!

router bgp 65000
  bgp router-id 10.0.0.1
  neighbor 10.0.1.1 remote-as 65001
  neighbor 10.0.1.1 bfd
  neighbor 10.0.1.1 bfd profile fast-link
  neighbor 10.0.2.1 remote-as 65002
  neighbor 10.0.2.1 bfd
  neighbor 10.0.2.1 bfd profile fast-link
  address-family ipv4 unicast
    redistribute connected
  exit-address-family
!
bfd
  profile fast-link
    receive-interval 100
    transmit-interval 100
    detect-multiplier 3
  !
!
```

Create `leaf1-frr.conf` (similar, with 65001 ASN):

```
! leaf1-frr.conf
frr defaults datacenter
hostname leaf1
log syslog informational

interface eth1
  ip address 10.0.1.1/31
!

router bgp 65001
  bgp router-id 10.0.1.1
  neighbor 10.0.1.0 remote-as 65000
  neighbor 10.0.1.0 bfd
  neighbor 10.0.1.0 bfd profile fast-link
  address-family ipv4 unicast
    redistribute connected
  exit-address-family
!
bfd
  profile fast-link
    receive-interval 100
    transmit-interval 100
    detect-multiplier 3
  !
!
```

Deploy the topology:

```bash
sudo containerlab deploy --topo bfd-lab.yaml
```

Expected output:

```
INFO[0000] Containerlab v0.54.2 started
INFO[0000] Parsing & checking topology file: bfd-lab.yaml
INFO[0000] Creating lab directory: /home/user/clab-bfd-nccl
INFO[0001] Creating container: "spine1"
INFO[0001] Creating container: "leaf1"
INFO[0001] Creating container: "leaf2"
INFO[0003] Creating link: spine1:eth1 <--> leaf1:eth1
INFO[0003] Creating link: spine1:eth2 <--> leaf2:eth1
INFO[0004] Adding containerlab host entries to /etc/hosts file
INFO[0004] 🎉 New containerlab version 0.55.0 is available!
+---+----------------------+--------------+-------------------------------+-------+
| # |         Name         | Container ID |             Image             | State |
+---+----------------------+--------------+-------------------------------+-------+
| 1 | clab-bfd-nccl-leaf1  | a3f1b2c4d5e6 | frrouting/frr:v9.1.0          | running |
| 2 | clab-bfd-nccl-leaf2  | b4e2c3d6f7a8 | frrouting/frr:v9.1.0          | running |
| 3 | clab-bfd-nccl-spine1 | c5d3e4f8a9b0 | frrouting/frr:v9.1.0          | running |
+---+----------------------+--------------+-------------------------------+-------+
```

Verify BGP sessions:

```bash
docker exec clab-bfd-nccl-spine1 vtysh -c "show bgp summary"
```

```
IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.1, local AS number 65000 vrf-id 0
BGP table version 5
RIB entries 4, using 768 bytes of memory
Peers 2, using 29 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.0.1.1        4      65001        12        12        5    0    0 00:01:44            1        3 N/A
10.0.2.1        4      65002        11        11        5    0    0 00:01:43            1        3 N/A

Total number of neighbors 2
```

### Step 2 — Enable BFD on all BGP sessions with aggressive timers

The FRR config files already include BFD with the `fast-link` profile (100ms/100ms/3). Verify the profile is active:

```bash
docker exec clab-bfd-nccl-spine1 vtysh -c "show bfd profile"
```

```
BFD Profile:
  Profile name: fast-link
    Minimum Rx interval: 100ms
    Minimum Tx interval: 100ms
    Detection multiplier: 3
    Echo mode: disabled
```

Calculated detection time: `100ms × 3 = 300ms`.

### Step 3 — Show BFD peer state

```bash
docker exec clab-bfd-nccl-spine1 vtysh -c "show bfd peers"
```

```
BFD Peers:
        peer 10.0.1.1 local-address 10.0.1.0 vrf default interface eth1
                ID: 1234567890
                Remote ID: 9876543210
                Active mode
                Status: up
                Uptime: 2 minute(s), 31 second(s)
                Diagnostics: ok
                Remote diagnostics: ok
                Peer Type: dynamic
                Local timers:
                        Detect-multiplier: 3
                        Receive interval: 100ms
                        Transmission interval: 100ms
        peer 10.0.2.1 local-address 10.0.2.0 vrf default interface eth2
                ID: 2345678901
                Remote ID: 8765432109
                Active mode
                Status: up
                Uptime: 2 minute(s), 30 second(s)
                Diagnostics: ok
                Remote diagnostics: ok
                Peer Type: dynamic
                Local timers:
                        Detect-multiplier: 3
                        Receive interval: 100ms
                        Transmission interval: 100ms
```

Both sessions show `Status: up`.

### Step 4 — Simulate link failure and measure detection time

In one terminal, tail the FRR log to capture BFD and BGP events with timestamps:

```bash
docker exec clab-bfd-nccl-spine1 journalctl -f --no-hostname \
    -u frr 2>/dev/null | grep -E 'BFD|bgp|DOWN|UP'
```

In a second terminal, record the precise failure time and bring the link down:

```bash
date +%T.%N  &&  \
sudo ip link set "$(sudo docker inspect clab-bfd-nccl-spine1 \
    --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
    | xargs -I{} ip route get {} | awk '/dev/{print $3}')" down 2>/dev/null || \
docker exec clab-bfd-nccl-spine1 ip link set eth1 down
```

Expected log output in the monitoring terminal (timestamps compressed for display):

```
10:15:42.301 frr[1234]: bfdd: [BFDD-X] BFD peer 10.0.1.1 went down, reason: no-receive
10:15:42.302 frr[1234]: bgpd: %NOTIFICATION: sent to neighbor 10.0.1.1 6/9 (Cease/BFD session down)
10:15:42.303 frr[1234]: bgpd: %ADJCHANGE: neighbor 10.0.1.1 Down BFD session down
10:15:42.304 frr[1234]: bgpd: %NOTIFICATION: BGP route 10.0.1.1/24 withdrawn
```

Compare the link-down timestamp (`10:15:42.001` approximately) with the BFD notification (`10:15:42.301`). The delta is approximately **300ms** — matching the `100ms × 3` detection timer exactly.

Without BFD, the BGP keepalive hold-time (default 90 seconds) would delay detection until `10:16:12` — a **90-second** gap during which training ranks would be blocked at AllReduce.

### Step 5 — Show BGP convergence after BFD triggers withdrawal

```bash
docker exec clab-bfd-nccl-spine1 vtysh -c "show bgp summary"
```

```
IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.1, local AS number 65000 vrf-id 0

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.0.1.1        4      65001        14        14        7    0    0 00:00:03  Idle (Admin)       0 N/A
10.0.2.1        4      65002        13        13        7    0    0 00:03:01            1        2 N/A
```

Leaf1's neighbor is now `Idle (Admin)` and route count dropped from 3 to 2 (leaf1's prefix is withdrawn). Restore the link and watch reconvergence:

```bash
docker exec clab-bfd-nccl-spine1 ip link set eth1 up
sleep 1
docker exec clab-bfd-nccl-spine1 vtysh -c "show bgp summary"
```

```
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
10.0.1.1        4      65001        16        16        9    0    0 00:00:04            1        3 N/A
10.0.2.1        4      65002        15        15        9    0    0 00:04:08            1        3 N/A
```

BGP session restored and prefix count back to 3 (including leaf1's reconnected prefix) within 1–2 seconds.

### Step 6 — Write fault_injector.py

```python
# fault_injector.py
"""
Bring spine1↔leaf1 link down/up on a schedule, measure BGP convergence time,
and write a CSV report: cycle,down_time,detect_time_ms,recover_time_ms
"""
import subprocess, time, csv, re, sys
from datetime import datetime

CONTAINER  = "clab-bfd-nccl-spine1"
INTERFACE  = "eth1"
NEIGHBOR   = "10.0.1.1"
CYCLES     = int(sys.argv[1]) if len(sys.argv) > 1 else 5
OUT_CSV    = "convergence_report.csv"

def vtysh(cmd):
    return subprocess.check_output(
        ["docker", "exec", CONTAINER, "vtysh", "-c", cmd],
        text=True, stderr=subprocess.DEVNULL)

def link(state):          # "down" or "up"
    subprocess.run(["docker", "exec", CONTAINER, "ip", "link", "set",
                    INTERFACE, state], check=True)

def peer_state():
    out = vtysh("show bgp summary")
    m = re.search(rf"{re.escape(NEIGHBOR)}\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+\s+"
                  r"\d+\s+\d+\s+\d+\s+[\d:]+\s+(\S+)", out)
    return m.group(1) if m else "unknown"

def wait_for_state(target, timeout=10.0, poll=0.1):
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        if target.lower() in peer_state().lower():
            return time.monotonic()
        time.sleep(poll)
    raise TimeoutError(f"Peer did not reach state '{target}' within {timeout}s")

results = []
print(f"Running {CYCLES} failure cycles — output: {OUT_CSV}")

for cycle in range(1, CYCLES + 1):
    t_down = time.monotonic()
    link("down")
    try:
        t_detected = wait_for_state("idle", timeout=5.0)
        detect_ms = (t_detected - t_down) * 1000
    except TimeoutError:
        detect_ms = -1

    link("up")
    try:
        t_recovered = wait_for_state("established", timeout=15.0)
        recover_ms = (t_recovered - t_down) * 1000
    except TimeoutError:
        recover_ms = -1

    row = dict(cycle=cycle,
               down_time=datetime.utcnow().isoformat(),
               detect_ms=round(detect_ms, 1),
               recover_ms=round(recover_ms, 1))
    results.append(row)
    print(f"  Cycle {cycle}: detect={detect_ms:.0f}ms  recover={recover_ms:.0f}ms")
    time.sleep(5)   # settle between cycles

with open(OUT_CSV, "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=results[0].keys())
    w.writeheader()
    w.writerows(results)

print(f"\nReport written to {OUT_CSV}")
avg_detect  = sum(r["detect_ms"]  for r in results if r["detect_ms"]  > 0) / CYCLES
avg_recover = sum(r["recover_ms"] for r in results if r["recover_ms"] > 0) / CYCLES
print(f"Average detect:  {avg_detect:.0f}ms")
print(f"Average recover: {avg_recover:.0f}ms")
```

Run it:

```bash
source .venv/bin/activate
python fault_injector.py 5
```

Expected output:

```
Running 5 failure cycles — output: convergence_report.csv
  Cycle 1: detect=312ms  recover=1843ms
  Cycle 2: detect=298ms  recover=1921ms
  Cycle 3: detect=305ms  recover=1784ms
  Cycle 4: detect=311ms  recover=1866ms
  Cycle 5: detect=302ms  recover=1798ms

Report written to convergence_report.csv
Average detect:  306ms
Average recover: 1842ms
```

### Step 7 — Write a 2-rank AllReduce script and run with torchrun

Create `allreduce_test.py`:

```python
# allreduce_test.py
"""
Minimal 2-rank AllReduce over gloo backend.
Writes a heartbeat file each step so a watchdog can monitor liveness.
"""
import os, time, torch
import torch.distributed as dist

HEARTBEAT = "/tmp/training_heartbeat"
STEPS     = 1000
SLEEP_PER_STEP = 0.5    # simulate compute time

dist.init_process_group(backend="gloo")
rank  = dist.get_rank()
world = dist.get_world_size()
print(f"[rank {rank}] init_process_group OK — world_size={world}")

tensor = torch.ones(1024 * 1024)    # 4 MB tensor

for step in range(STEPS):
    # Simulate forward+backward pass
    time.sleep(SLEEP_PER_STEP)

    # AllReduce
    dist.all_reduce(tensor, op=dist.ReduceOp.SUM)

    # Update heartbeat
    if rank == 0:
        with open(HEARTBEAT, "w") as f:
            f.write(str(time.time()))

    if rank == 0 and step % 10 == 0:
        print(f"[rank 0] step {step} AllReduce sum={tensor[0].item():.0f}")

dist.destroy_process_group()
print(f"[rank {rank}] training complete")
```

Set the NCCL timeout and launch with `torchrun`:

```bash
export NCCL_TIMEOUT=10000   # 10 seconds
torchrun \
    --nproc-per-node=2 \
    --nnodes=1 \
    --rdzv-backend=c10d \
    --rdzv-endpoint=localhost:29400 \
    allreduce_test.py
```

Expected output (first few steps):

```
[rank 0] init_process_group OK — world_size=2
[rank 1] init_process_group OK — world_size=2
[rank 0] step 0 AllReduce sum=2097152
[rank 0] step 10 AllReduce sum=2097152
[rank 0] step 20 AllReduce sum=2097152
```

### Step 8 — Kill rank 1 mid-AllReduce and observe timeout

In a second terminal, while the torchrun job is running, kill the rank 1 process:

```bash
# Find rank 1's PID (it will be the second torchrun worker)
pgrep -f "allreduce_test.py" | tail -1 | xargs kill -9
```

After `NCCL_TIMEOUT=10000ms` (10 seconds), rank 0 logs:

```
[rank 0] step 30 AllReduce sum=2097152
Traceback (most recent call last):
  File "/home/user/allreduce_test.py", line 29, in <module>
    dist.all_reduce(tensor, op=dist.ReduceOp.SUM)
  File ".../torch/distributed/distributed_c10d.py", line 2191, in all_reduce
    work = group.allreduce([tensor], opts)
torch.distributed.DistBackendError: GLOO connect timeout after PT_GLOO_TIMEOUT_MILLISECONDS=10000ms
GLOO error: Encountered error in barrier. Abort is now being called.

ERROR:torch.distributed.elastic.multiprocessing.api:failed (exitcode: 1)
  [1]: time=2026-04-22_10:23:55 worker_id=1 exit_code=-9
  [0]: time=2026-04-22_10:23:55 worker_id=0 exit_code=1
Traceback (most recent call last):
  ...
torch.distributed.elastic.multiprocessing.api.ProcessGroupFailure:
  failures={
    1: ProcessFailure(local_rank=1, pid=54321, exitcode=-9, ...),
    0: ProcessFailure(local_rank=0, pid=54320, exitcode=1, ...)
  }
```

The stack trace shows `DistBackendError` after exactly 10 seconds, confirming `NCCL_TIMEOUT=10000` is respected.

### Step 9 — Enable automatic restart with torchrun --max-restarts

Modify `allreduce_test.py` to save and reload a checkpoint, then relaunch with `--max-restarts=2`:

```bash
export NCCL_TIMEOUT=10000
torchrun \
    --nproc-per-node=2 \
    --nnodes=1 \
    --max-restarts=2 \
    --rdzv-backend=c10d \
    --rdzv-endpoint=localhost:29400 \
    allreduce_test.py
```

Kill rank 1 again. This time, instead of terminating, `torchrun` restarts both workers:

```
[rank 0] step 47 AllReduce sum=2097152
ERROR:torch.distributed.elastic.multiprocessing.api:failed (exitcode: 1)
  [1]: time=2026-04-22_10:24:31 worker_id=1 exit_code=-9
WARNING:torch.distributed.elastic.agent.server.api:[default] Worker group failed, will retry.
INFO:torch.distributed.elastic.agent.server.api:[default] Restarting worker group (attempt 1 of 2).
INFO:torch.distributed.rendezvous.dynamic_rendezvous:[default] Rendezvous complete for 2 participants.
[rank 0] init_process_group OK — world_size=2
[rank 1] init_process_group OK — world_size=2
[rank 0] step 0 AllReduce sum=2097152
[rank 0] step 10 AllReduce sum=2097152
```

The job re-rendezvous and resumes from step 0 (or from a saved checkpoint if `load_checkpoint()` is implemented). The key log line is `Restarting worker group (attempt 1 of 2)` — `torchrun` consumed one of the two restart credits.

### Step 10 — Monitor RDMA error counters and export to Prometheus

Run `rdma stat` in a loop to watch for errors during fault injection:

```bash
watch -n 2 "rdma stat show 2>/dev/null | grep -E 'local_ack|rnr|out_of_seq'"
```

Start the Prometheus exporter in a background terminal:

```bash
source .venv/bin/activate
python rdma_exporter.py &
```

```
RDMA exporter listening on :9101
```

Verify the metrics endpoint:

```bash
curl -s http://localhost:9101/metrics | grep rdma_local_ack
```

```
# HELP rdma_local_ack_timeout_err_total RDMA counter: local_ack_timeout_err
# TYPE rdma_local_ack_timeout_err_total gauge
rdma_local_ack_timeout_err_total{dev="mlx5_0",port="1"} 3.0
```

Now inject a link fault while the exporter runs:

```bash
docker exec clab-bfd-nccl-spine1 ip link set eth1 down
sleep 2
curl -s http://localhost:9101/metrics | grep rdma_local_ack
```

```
rdma_local_ack_timeout_err_total{dev="mlx5_0",port="1"} 11.0
```

The counter jumped from 3 to 11 — 8 additional timeout errors triggered by the 2-second link outage. A Prometheus `rate()` alert on this counter crossing 1/s would fire within one 30-second evaluation window, giving the operations team an automated alert correlated with the BFD event.

### Step 11 — Cleanup

```bash
# Kill background exporter
kill %1 2>/dev/null

# Restore any down links
docker exec clab-bfd-nccl-spine1 ip link set eth1 up 2>/dev/null
docker exec clab-bfd-nccl-spine1 ip link set eth2 up 2>/dev/null

# Destroy Containerlab topology
sudo containerlab destroy --topo bfd-lab.yaml --cleanup
```

Expected output:

```
INFO[0000] Parsing & checking topology file: bfd-lab.yaml
INFO[0001] Destroying lab: bfd-nccl
INFO[0001] Removed container: clab-bfd-nccl-leaf2
INFO[0001] Removed container: clab-bfd-nccl-leaf1
INFO[0001] Removed container: clab-bfd-nccl-spine1
INFO[0001] Removing containerlab host entries from /etc/hosts file
INFO[0001] 🎉 done!
```

```bash
# Deactivate Python environment
deactivate

# Remove generated files
rm -f bfd-lab.yaml spine1-frr.conf leaf1-frr.conf leaf2-frr.conf \
      allreduce_test.py fault_injector.py rdma_exporter.py \
      convergence_report.csv /tmp/training_heartbeat
```

---

## Summary

- Network faults kill training jobs through three mechanisms: RDMA QP retry exhaustion (latent error), AllReduce barrier hang (blocking), and checkpoint corruption (data loss). BFD eliminates the first two by detecting link failures in milliseconds rather than minutes.
- BFD in FRR uses the `bfd profile` stanza to set `receive-interval`, `transmit-interval`, and `detect-multiplier`. With 100ms/100ms/3, failure detection completes in 300ms — over 300x faster than the BGP default hold-time.
- RDMA QP parameters `timeout`, `retry_cnt`, `rnr_retry`, and `min_rnr_timer` must be tuned together: a low `timeout` with high `retry_cnt` gives fast detection without false positives on transient congestion.
- The PFC watchdog in ConnectX NICs prevents lossless fabric deadlocks by auto-disabling PFC on a priority class after a configurable pause duration threshold is exceeded.
- `torchrun --max-restarts=N` combined with `NCCL_TIMEOUT` and application-level checkpointing provides end-to-end fault recovery: the network layer detects (BFD), the training framework times out (NCCL), and the launcher restarts and re-rendezvous (torchrun c10d elastic).
- Async checkpointing with atomic rename (`os.rename()` from `.tmp`) ensures checkpoint files are always complete or absent — never partially written.
- The circuit-breaker pattern prevents a partially-recovered cluster from entering a tight failure/retry loop that saturates rendezvous store infrastructure.
- Fault correlation requires joining three event streams: BFD state changes (gNMI on-change subscription), RDMA error counters (Prometheus rate alert), and NCCL timeout log lines. Timestamp proximity within `NCCL_TIMEOUT + BFD_detect` seconds is strong evidence of a network-induced training failure.

---

## References

- RFC 5880: Bidirectional Forwarding Detection (BFD) — datatracker.ietf.org/doc/html/rfc5880
- FRR BFD documentation: docs.frrouting.org/en/stable/bfd.html
- PyTorch Elastic Training: pytorch.org/docs/stable/elastic/run.html
- NCCL environment variables: docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html
- rdma-core and libibverbs: github.com/linux-rdma/rdma-core
- DMTCP: dmtcp.sourceforge.io
- Mellanox PFC watchdog: community.mellanox.com/s/article/understanding-pfc-storm-on-roce
- Prometheus alerting rules: prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- OpenConfig BFD YANG model: github.com/openconfig/public/tree/master/release/models/bfd


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).