# Appendix G — RDMA & NCCL Troubleshooting Cookbook

This appendix is a runbook-style reference for diagnosing and resolving the most common failure modes in AI cluster networking. Each section follows the same structure: problem statement, symptoms, diagnostic commands with expected output, root cause analysis, and remediation steps. Commands are written for Linux with MOFED installed and assume root or a user with `CAP_NET_ADMIN`.

Prerequisite tools referenced throughout:
- `ib_write_bw`, `ib_read_lat` — from `perftest` package (MOFED or distro)
- `ibstat`, `ibstatus`, `ibdiagnet`, `perfquery` — from MOFED `infiniband-diags`
- `mlnx_qos` — from MOFED `mlnx-tools`
- `rdma` — from `iproute2` (>= 5.2)
- `ethtool` — standard Linux
- `gnmic` — `gnmic` binary from OpenConfig gnmic project
- `birdc`, `vtysh` — from BIRD / FRRouting

---

## G.1 RDMA Bandwidth Below Expected

### Problem Statement

RDMA throughput measured by `ib_write_bw` or `ib_send_bw` is significantly below the rated line rate of the link. A ConnectX-7 NDR IB link (400 Gb/s) that produces only 150–180 Gb/s, or a 400GbE RoCEv2 link delivering less than 200 Gb/s, indicates a configuration or fabric problem rather than a hardware limit.

### Symptoms

- `ib_write_bw` reports BW < 50% of line rate at large message sizes (e.g., 1 MB)
- GPU-to-GPU NCCL bandwidth tests (via `nccl-tests`) show similar underperformance
- No errors reported in dmesg or application logs
- CPU utilization is low, ruling out CPU-bound processing

### Diagnostic Commands

**Step 1: Check link status and speed**

```bash
ibstat mlx5_0
```

Expected (good):
```
CA 'mlx5_0'
  Port 1:
    State: Active
    Physical state: LinkUp
    Rate: 400
    Link layer: InfiniBand
```

Bad (degraded):
```
    State: Active
    Physical state: LinkUp
    Rate: 100          # NDR link negotiated at HDR rate — cable/transceiver issue
```

**Step 2: Verify MTU**

```bash
ibstatus mlx5_0
# For RoCEv2, check Ethernet MTU:
ip link show eth0 | grep mtu
```

Expected (good): MTU 4096 for IB, MTU 9000 (jumbo frames) for RoCEv2.

Bad:
```
mtu 1500       # Default Ethernet MTU — RDMA performance degrades severely at 1500B MTU
```

Fix: `ip link set eth0 mtu 9000` (and set switch port MTU to match, typically 9216 to accommodate headers).

**Step 3: Check PFC status on RoCEv2 fabric**

```bash
mlnx_qos -i eth0
```

Expected (good):
```
PFC enabled: 01110000    # Priority 3 enabled (binary), this is the RDMA priority
```

Bad:
```
PFC enabled: 00000000    # PFC fully disabled — RoCEv2 will drop under load
```

**Step 4: Check ECN thresholds**

```bash
# On the switch (SONiC):
redis-cli -n 4 HGETALL "BUFFER_PG|Ethernet0|3"
# Or via sysfs on host:
cat /sys/class/net/eth0/ecn_enable
```

**Step 5: Check ring buffer sizing**

```bash
ethtool -g eth0
```

Expected (good):
```
Ring parameters for eth0:
Pre-set maximums:
RX:             8192
TX:             8192
Current hardware settings:
RX:             8192
TX:             8192
```

Bad: RX/TX at default (256 or 512) — small rings cause packet drops under burst.

Fix: `ethtool -G eth0 rx 8192 tx 8192`

**Step 6: Run `ib_write_bw` with verbose output**

```bash
# On receiver:
ib_write_bw -d mlx5_0 -q 8 --report_gbits

# On sender:
ib_write_bw -d mlx5_0 -q 8 --report_gbits <receiver_IP>
```

Expected (good, NDR IB):
```
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]
 1048576    1000           388.5              387.2
```

Bad:
```
 1048576    1000           182.3              180.1     # ~45% of line rate
```

### Root Causes and Fixes

| Root Cause | Diagnostic Signal | Fix |
|---|---|---|
| MTU mismatch (1500 vs 9000) | `ip link show` shows mtu 1500 | `ip link set eth0 mtu 9000`; set switch port MTU 9216 |
| PFC disabled on RDMA priority | `mlnx_qos` shows PFC off | Enable PFC on CoS 3: `mlnx_qos -i eth0 --pfc 0,0,0,1,0,0,0,0` |
| DCQCN misconfigured (ECN threshold too high) | No CNP packets in `tcpdump` | Lower ECN marking threshold in switch buffer profile |
| Wrong transport mode (UC instead of RC) | `ib_write_bw -t uc` used accidentally | Default RC is correct; verify `-t` flag not overriding |
| Link speed negotiated at wrong rate | `ibstat` shows Rate: 100 instead of 400 | Reseat cable; check transceiver compatibility; `ibportstate ... reset` |
| Small ring buffers | `ethtool -g` shows 256 | `ethtool -G eth0 rx 8192 tx 8192` |

---

## G.2 RDMA Link Errors and Retransmissions

### Problem Statement

RDMA connections are established but packets are being dropped or corrupted at the physical layer, causing retransmissions that inflate latency and reduce throughput. Link errors appear in InfiniBand port counters and may be intermittent.

### Symptoms

- `perfquery` shows incrementing `RcvErrors`, `XmtDiscards`, or `SymbolErrors`
- Latency spikes visible in `ib_read_lat` histogram
- Application-level timeouts or connection resets
- Intermittent NCCL timeout errors during training runs

### Diagnostic Commands

**Step 1: Check IB port counters**

```bash
perfquery -d mlx5_0 -x 1    # -d selects CA device; -x clears counters; 1 is the port; run twice with interval to see rate
```

Expected (good):
```
PortRcvErrors:..........0
PortXmtDiscards:........0
PortRcvRemotePhysErrors:0
SymbolErrorCounter:.....0
```

Bad:
```
PortRcvErrors:..........1482     # Rising counter — physical layer problem
SymbolErrorCounter:.....893      # Electrical signal degradation
```

**Step 2: Run ibdiagnet for fabric-wide diagnostics**

```bash
ibdiagnet --pc -r    # --pc resets port counters; -r full report
```

This scans the entire IB subnet and flags ports with errors. Look for output like:
```
-E- Node: H100-node-07 Port 1 SymbolErrors: 1204  <- cable or optic issue
```

**Step 3: Check per-port counters via sysfs**

```bash
cat /sys/class/infiniband/mlx5_0/ports/1/counters/port_rcv_errors
cat /sys/class/infiniband/mlx5_0/ports/1/counters/port_xmit_discards
cat /sys/class/infiniband/mlx5_0/ports/1/counters/symbol_error
```

**Step 4: Check RDMA statistics via iproute2**

```bash
rdma stat show dev mlx5_0 port 1
```

Expected (good):
```
link mlx5_0/1 state ACTIVE
  rx_write_requests 2847432 rx_read_requests 0 rx_atomic_requests 0
  out_of_buffer 0 out_of_sequence 0 duplicate_request 0
  rnr_nak_retry_err 0 local_ack_timeout_err 0
```

Bad:
```
  out_of_sequence 892       # Packet reordering or drops
  rnr_nak_retry_err 412     # Receiver Not Ready — queue overflow
```

**Step 5: Capture and examine RDMA packets**

```bash
ibdump -d mlx5_0 -i 1 -o /tmp/rdma.pcap -- --snaplen 256
# Then open in Wireshark or:
tcpdump -r /tmp/rdma.pcap -c 50
```

### Root Causes and Fixes

| Root Cause | Diagnostic Signal | Fix |
|---|---|---|
| Faulty QSFP/cable | Rising SymbolErrors, high BER | Reseat or replace cable/transceiver; check DOM: `ethtool -m eth0` |
| Speed/duplex mismatch | Link at wrong rate in `ibstat` | Force port speed or check auto-negotiation settings on switch |
| Dirty fiber connector | Intermittent SymbolErrors | Clean fiber connectors with IEC 61300-3-35 cleaner |
| Switch buffer overflow (XmtDiscards) | Rising XmtDiscards on switch-facing ports | Tune PFC/ECN; check for microbursts with INT telemetry |
| RNR (Receiver Not Ready) | `rnr_nak_retry_err` incrementing | Increase receive queue depth; `ibv_exp_query_intf` to check QP params |

---

## G.3 NCCL AllReduce Hang

### Problem Statement

A distributed training job starts successfully, all ranks launch, but training halts indefinitely at the first or subsequent AllReduce barrier. No error message is printed; GPUs show zero utilization; the job does not fail, it just stops progressing.

### Symptoms

- Training process stuck; `nvidia-smi` shows 0% GPU utilization across all nodes
- `ps aux` on nodes shows training process in `S` (sleeping/blocking) state
- Timeout (if configured) fires after minutes or tens of minutes
- No exceptions or error stack traces in application logs

### Diagnostic Commands

**Step 1: Enable NCCL debug logging and re-run**

```bash
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,COLL,P2P,NET
torchrun --nproc_per_node=8 train.py 2>&1 | tee /tmp/nccl_debug.log
```

In the log, look for where initialization stops. Common stuck points:

Good (progressing through init):
```
[0] NCCL INFO Bootstrap: Using eth0:10.0.1.5<0>
[0] NCCL INFO NET/IB : Using mlx5_0:1/RoCE
[0] NCCL INFO ncclCommInitRank: rank 0 nranks 64 commHash ...
```

Bad (stuck here):
```
[0] NCCL INFO bootstrapRoot: Listening on 10.0.1.5:29400   # Rank 0 waiting
# No further output — other ranks cannot reach port 29400
```

**Step 2: Check rendezvous port connectivity**

```bash
# Check if port is open on rank 0 node:
ss -tnlp | grep 29400
```

Expected (good): `LISTEN  0  128  0.0.0.0:29400`

Bad: no output — process crashed before binding, or another process took the port.

**Step 3: Test inter-node connectivity on rendezvous port**

```bash
# From rank 1 node, test reaching rank 0:
nc -zv <rank0_ip> 29400
```

Expected (good): `Connection to <rank0_ip> 29400 port [tcp/*] succeeded!`

Bad: `Connection refused` or timeout — firewall or iptables blocking.

**Step 4: Check firewall rules**

```bash
iptables -L INPUT -n -v | grep -E "DROP|REJECT"
# In Kubernetes, check NetworkPolicy:
kubectl get networkpolicy -n <namespace>
```

**Step 5: Check for rank count mismatch**

```bash
# Verify all nodes launched the same number of ranks:
grep "nranks" /tmp/nccl_debug.log | sort -u
```

Bad: different ranks reporting different `nranks` values — job was not launched consistently.

**Step 6: Strace a hung rank**

```bash
strace -p <rank_pid> -e trace=network,ipc 2>&1 | head -50
```

Look for:
```
recvfrom(sockfd, ...   # Stuck waiting for data — peer unreachable or not sending
```

### Root Causes and Fixes

| Root Cause | Diagnostic Signal | Fix |
|---|---|---|
| Firewall blocking port 29400 | `nc` fails from non-zero rank | `iptables -A INPUT -p tcp --dport 29400 -j ACCEPT` (or adjust NetworkPolicy) |
| Wrong `MASTER_ADDR` / `MASTER_PORT` | NCCL bootstrap log shows wrong IP | Set `MASTER_ADDR` to rank-0 IP explicitly in launch script |
| Rank count mismatch | `nranks` differs across ranks | Ensure `--nproc_per_node` and `--nnodes` consistent across all nodes |
| Network namespace isolation in K8s | Pod cannot reach other pods on port 29400 | Check Multus interface routing; ensure NCCL interface env var (`NCCL_SOCKET_IFNAME`) matches the correct interface |
| Stale process from previous run | Port 29400 occupied | `fuser -k 29400/tcp` on rank 0 node |

---

## G.4 NCCL Bandwidth Collapse Under Incast

### Problem Statement

AllReduce throughput is acceptable at small scale (8–16 GPUs) but collapses to less than 10% of expected throughput at larger scales (128+ GPUs). GPU utilization as reported by DCGM shows GPUs spending most time waiting at barriers. The fabric appears healthy (no link errors), but collective performance is severely degraded.

### Symptoms

- `nccl-tests` AllReduce bandwidth scales sublinearly above 32–64 GPUs
- DCGM metric `DCGM_FI_PROF_NCCL_RX_BYTES` near zero while job is running
- Switch `out-discards` counter incrementing on downlink ports
- Wireshark/tcpdump shows ECN CE marks on nearly all RDMA packets

### Diagnostic Commands

**Step 1: Check switch port queue discards via gNMI**

```bash
gnmic -a <switch_ip>:57400 --insecure subscribe \
  --path "/interfaces/interface[name=Ethernet1]/state/counters/out-discards" \
  --mode once
```

Expected (good): `out-discards: 0` or very low and static.

Bad: `out-discards: 4892931` and increasing — queue overflow on server-facing port.

**Step 2: Measure AllReduce BW at different message sizes**

```bash
# On 8 GPUs (baseline):
python -m torch.distributed.launch --nproc_per_node=8 \
  nccl-tests/build/all_reduce_perf -b 1M -e 1G -f 2 -g 1

# At scale (128 GPUs across 16 nodes):
mpirun --hostfile hosts.txt -np 128 \
  nccl-tests/build/all_reduce_perf -b 1M -e 1G -f 2 -g 1
```

Compare `algbw` (algorithm bandwidth) at large sizes. A healthy fabric scales near-linearly (modulo ring bandwidth formula). A collapse at scale with good small-message bandwidth suggests incast, not a configuration error.

**Step 3: Check ECN marking rate**

```bash
# On host, check CNP (Congestion Notification Packet) send rate:
ethtool -S eth0 | grep cnp
```

Expected (good, low congestion):
```
tx_cnp_ignored: 0
tx_cnp_sent: 12      # Small number of CNPs sent
```

Bad:
```
tx_cnp_sent: 4289012   # Extremely high CNP rate — ECN firing constantly
```

**Step 4: Check DCQCN alpha convergence via counter**

```bash
# Inspect DCQCN parameters:
cat /sys/class/net/eth0/tc/tc0/stats | grep alpha
# Or via mlnx_qos:
mlnx_qos -i eth0 --trust dscp
```

**Step 5: Check PFC pause counts for HOL blocking**

```bash
ethtool -S eth0 | grep -E "rx_pfc|tx_pfc"
```

Expected (controlled): low and stable PFC pause counts.

Bad:
```
rx_pfc_3_pause_duration: 8293847  # Very long pause durations — HOL blocking chain
tx_pfc_3_pause: 1939284
```

### Root Causes and Fixes

| Root Cause | Diagnostic Signal | Fix |
|---|---|---|
| ECN threshold too high (no early marking) | High `tx_cnp_sent`, late congestion response | Lower `ECN_MAX_THRESHOLD` on switch buffer profile from 3MB to 300KB |
| PFC storm causing HOL blocking | Very long `rx_pfc_*_pause_duration` | Enable PFC watchdog: `mlnx_qos --pfc_wd enable`; verify DCQCN prevents PFC triggering at all |
| DCQCN alpha not converging | CNP storm with no rate recovery | Tune `dcqcn_rp_clamp_tgt_rate`, `dcqcn_rp_time_reset` via `/sys/kernel/debug/mlx5/...` |
| Spine ECMP hash collision | Specific ports overloaded, others idle | Verify 5-tuple hash includes L4 ports; check for non-unique source ports in collective traffic |

---

## G.5 BGP Session Flapping

### Problem Statement

BGP sessions between leaf switches and spine switches (or between routers and route reflectors) cycle between Established and Active/Idle states. Routes are being withdrawn and re-advertised, causing traffic forwarding disruptions visible as packet loss or path flaps.

### Symptoms

- `show bgp summary` (Arista/FRR) shows session uptime resetting repeatedly
- Route table churn visible in log: `%BGP-5-ADJCHANGE: neighbor x.x.x.x Down/Up`
- Traceroute shows path changing between successive runs
- Training jobs experience timeout errors coinciding with BGP flap events

### Diagnostic Commands

**Step 1: Check BGP session status (BIRD)**

```bash
birdc show protocols all
```

Expected (good):
```
Name       Proto      Table      State  Since         Info
bgp_spine1 BGP        ---        up     2024-10-01    Established
  BGP state:          Established
    Neighbor AS:      65001
    Hold timer:       90/90
    Keepalive timer:  28.9
```

Bad:
```
bgp_spine1 BGP        ---        start  2024-10-01    Active
  BGP state:          Active       # Session not established
    Last error:       Hold timer expired
```

**Step 2: Check BGP session status (FRR)**

```bash
vtysh -c "show bgp summary"
vtysh -c "show bgp neighbors <peer_ip>"
```

Look for `BGP state = Active` and `Last reset: due to: holdtimer expired`.

**Step 3: Enable keepalive debugging (FRR)**

```bash
vtysh -c "debug bgp keepalives"
vtysh -c "debug bgp neighbor-events"
# Monitor live:
tail -f /var/log/frr/bgpd.log
```

**Step 4: Capture BGP traffic on port 179**

```bash
tcpdump -i eth0 port 179 -w /tmp/bgp.pcap -c 500
# Open in Wireshark or:
tcpdump -r /tmp/bgp.pcap -A | grep -E "OPEN|KEEPALIVE|NOTIFICATION"
```

Look for: missing KEEPALIVE messages before session drop, or NOTIFICATION with error code `Hold Timer Expired`.

**Step 5: Check BFD timer vs BGP hold timer**

```bash
# In FRR:
vtysh -c "show bfd peers"
```

Expected (good): BFD `Detection Multiplier: 3` with `Receive Interval: 300ms` = 900ms detection. BGP hold timer should be at least 3× the BFD detection time.

Bad: BFD `Receive Interval: 50ms, Multiplier: 3` = 150ms detection on a loaded CPU. BGP hold timer is 30s — BFD fires and tears down the BGP session before the hold timer expires.

**Step 6: Check CPU load on routing daemon**

```bash
top -p $(pgrep bgpd)
# High CPU on bgpd means keepalives may not be sent within hold timer
```

### Root Causes and Fixes

| Root Cause | Diagnostic Signal | Fix |
|---|---|---|
| BFD timer too aggressive | BFD flapping on loaded system | Increase BFD min-rx to 300ms × 3; or disable BFD for non-critical sessions |
| MTU mismatch causing BGP OPEN fragmentation | TCPdump shows fragmented TCP on port 179 | Set BGP TCP MSS clamp: `neighbor <ip> dont-capability-negotiate` or fix MTU |
| CPU overload on soft NOS | `bgpd` at 90%+ CPU, delayed KEEPALIVE | Reduce number of BGP prefixes; use route summarization; move to dedicated routing CPU |
| Hold timer too short | `Last reset: holdtimer expired` in logs | Increase hold timer: `neighbor <ip> timers 30 90` (keepalive 30s, hold 90s) |
| NIC queue drop causing TCP loss | `ethtool -S` shows `rx_missed_errors` | Increase NIC ring buffer; `ethtool -G eth0 rx 4096` |

---

## G.6 PFC Deadlock in RoCE Fabric

### Problem Statement

All traffic in the fabric stops completely. No packets are forwarded. GPU training jobs hang. Switch port counters freeze. There are no error messages — the network appears to be in a state of suspended animation. This is a PFC deadlock: a cycle of back-pressure has formed where every device is pausing the device it depends on to receive.

### Symptoms

- Training job hangs with no GPU activity (identical to G.3 NCCL hang, but fabric-wide)
- All RoCEv2 traffic stopped, including non-RDMA management traffic on the RDMA VLAN
- Switch ports show identical byte counters across two polling intervals
- `ethtool -S` shows `rx_pfc_3_pause` counts very large and static (no new increments — port is paused and nothing is moving)
- Resolves only when affected switches or hosts are restarted

### Diagnostic Commands

**Step 1: Check PFC pause counters**

```bash
mlnx_qos -i eth0
```

Expected (healthy):
```
Priority  PFC
0         disabled
1         disabled
2         disabled
3         enabled
...
PFC pause counters: Rx prio 3: 120   Tx prio 3: 89   # Small, slowly incrementing
```

Bad (deadlock):
```
PFC pause counters: Rx prio 3: 18293847   Tx prio 3: 18293847   # Huge, static
```

**Step 2: Check interface PFC statistics via ethtool**

```bash
ethtool -S eth0 | grep -E "pfc|pause"
```

Bad:
```
rx_pfc_3_pause: 19284756   # Paused waiting for upstream to drain
tx_pfc_3_pause: 19284720   # Sending pause to downstream
rx_pfc_3_pause_duration: 99999999  # Maximum pause duration — deadlock
```

**Step 3: Identify the deadlock topology**

Map which switches are pausing which other switches. A deadlock requires a cycle:
- Node A pauses Switch-Leaf-1
- Switch-Leaf-1 pauses Switch-Spine-1
- Switch-Spine-1 pauses Switch-Leaf-2
- Switch-Leaf-2 pauses Node A

Collect PFC pause state from all switches in the suspected cycle.

**Step 4: Check PFC watchdog status**

```bash
mlnx_qos --pfc_wd status -i eth0
```

Expected (good): `PFC watchdog enabled, poll interval 100ms, action: drop`

Bad: `PFC watchdog disabled` — deadlock cannot self-resolve.

**Step 5: Check priority class assignment**

```bash
mlnx_qos -i eth0 --trust dscp
# Verify RDMA traffic (DSCP 26) maps to CoS 3 (lossless)
# All other traffic should map to lossy queues (CoS 0, CoS 2)
```

Bad: storage traffic (iSCSI) on same CoS 3 as RDMA — creates deadlock candidate even without a RDMA loop.

### Root Causes and Fixes

| Root Cause | Diagnostic Signal | Fix |
|---|---|---|
| Bi-directional PFC forming a cycle | Multiple switches in pause, cycle detectable in topology | Enable PFC watchdog on all hosts and switches |
| Missing PFC watchdog | `pfc_wd status` shows disabled | `mlnx_qos --pfc_wd enable --pfc_wd_poll_interval 100 --pfc_wd_action drop -i eth0` |
| Wrong priority class for RoCE traffic | Mixed lossless priorities; storage+RDMA on same CoS | Isolate RDMA on CoS 3 only; storage on separate CoS or lossy |
| Too many lossless priorities | Multiple priorities with PFC enabled | Limit PFC-enabled priorities to 1 (CoS 3 for RDMA); all others lossy |

**Emergency fix (break deadlock immediately):**

```bash
# Disable and re-enable PFC on the affected interface to flush pause state:
mlnx_qos -i eth0 --pfc 0,0,0,0,0,0,0,0
sleep 1
mlnx_qos -i eth0 --pfc 0,0,0,1,0,0,0,0
```

---

## G.7 Containerlab Lab Not Converging

### Problem Statement

A containerlab topology with BGP routing (e.g., SR Linux or FRR nodes) fails to establish BGP sessions after startup. Sessions stay in `Active` or `Connect` state. No routes are exchanged between nodes.

### Symptoms

- `birdc show protocols all` or `vtysh -c "show bgp summary"` shows all sessions in `Active`
- `docker exec <node> ping <peer_ip>` may or may not succeed
- BGP port 179 `tcpdump` shows SYN packets but no SYN-ACK
- Containerlab `graph` shows nodes connected but routing table is empty

### Diagnostic Commands

**Step 1: Basic IP connectivity test**

```bash
docker exec clab-lab01-leaf1 ping -c 3 192.168.1.2
```

Expected (good): 3 packets received, 0% loss.

Bad: 100% packet loss — routing is not the problem, IP connectivity itself is broken. Check interface addressing.

**Step 2: Verify interface addressing**

```bash
docker exec clab-lab01-leaf1 ip addr show eth1
docker exec clab-lab01-spine1 ip addr show eth1
```

Check that peer IP addresses are on the same subnet and assigned to the correct interface.

**Step 3: Check MTU on container interfaces**

```bash
docker exec clab-lab01-leaf1 ip link show eth1
```

Expected (good): `mtu 9500` (containerlab default for point-to-point links)

Bad:
```
mtu 1500     # Docker default — BGP OPEN message may exceed 1500B with many capabilities
```

Fix: Set MTU in containerlab topology YAML:
```yaml
links:
  - endpoints: ["leaf1:eth1", "spine1:eth1"]
    mtu: 9500
```

**Step 4: Capture BGP traffic inside container**

```bash
docker exec clab-lab01-leaf1 tcpdump -i eth1 port 179 -c 20 -w - | tcpdump -r - -A
```

Look for: repeated SYN without SYN-ACK (firewall or address mismatch), or NOTIFICATION packets with error codes.

**Step 5: Check BGP configuration (FRR example)**

```bash
docker exec clab-lab01-leaf1 vtysh -c "show running-config" | grep -A 5 "neighbor"
```

Verify:
- `neighbor <ip> remote-as <AS>` — correct peer AS
- `neighbor <ip> update-source <interface>` — if using loopback as source
- No typo in peer IP address

**Step 6: Check for IP address overlap**

```bash
# From host, inspect all container network namespaces:
for cid in $(docker ps -q); do
  echo "=== $(docker inspect $cid --format '{{.Name}}') ==="
  docker exec $cid ip route
done
```

Bad: two containers have the same IP address configured — common when copying topology YAML and forgetting to update addressing.

### Root Causes and Fixes

| Root Cause | Diagnostic Signal | Fix |
|---|---|---|
| IP addressing overlap or error | `ip addr` shows wrong subnet | Correct IP addressing in containerlab YAML; `clab destroy && clab deploy` |
| Wrong `peer-as` configured | Session shows `Notification: Bad Peer AS` in tcpdump | Correct AS number in BGP config |
| Docker MTU (1500) vs container link MTU | BGP OPEN truncated, session Reset | Set `mtu: 9500` in containerlab link config |
| BGP daemon not started | No process listening on port 179 | `docker exec <node> systemctl status bird` or `docker exec <node> ps aux | grep bgp` |
| Loopback source with no route to peer loopback | `Active` state with no SYN seen | Add `neighbor <ip> update-source eth1` or ensure underlay routes for loopbacks |

**Quick redeploy after config fix:**

```bash
sudo clab destroy -t lab01.yaml && sudo clab deploy -t lab01.yaml
```

---

## G.8 gNMI Subscribe Getting No Data

### Problem Statement

Running `gnmic subscribe` against a network device (SR Linux, SONiC, Arista EOS) either hangs indefinitely without producing output, or returns empty responses. Telemetry data is not flowing despite the device appearing to be reachable.

### Symptoms

- `gnmic subscribe` command blocks with no output
- `gnmic get` returns empty `notification` objects
- Prometheus gnmic scrape target shows no metrics
- No errors returned — the connection succeeds but data is empty

### Diagnostic Commands

**Step 1: Verify device capabilities first**

Always run `capabilities` before `subscribe` to confirm supported encodings and models:

```bash
gnmic -a <device_ip>:57400 --insecure capabilities
```

Expected (good):
```
gNMI version: 0.7.0
supported models:
  - openconfig-interfaces, OpenConfig working group, 2.5.0
  - openconfig-bgp, OpenConfig working group, 6.1.0
supported encodings:
  - JSON_IETF
  - PROTO
```

Bad:
```
Error: rpc error: code = Unauthenticated   # TLS or credential issue
```
or
```
Error: connection refused    # Wrong port or gNMI not enabled
```

**Step 2: Test a simple Get before Subscribe**

```bash
gnmic -a <device_ip>:57400 --insecure get \
  --path "/interfaces/interface[name=ethernet-1/1]/state/oper-status"
```

Expected (good):
```
[
  {
    "source": "<device_ip>:57400",
    "timestamp": 1698765432000000000,
    "time": "2024-10-01T12:30:32Z",
    "updates": [
      {
        "Path": "interfaces/interface[name=ethernet-1/1]/state/oper-status",
        "values": {
          "interfaces/interface[name=ethernet-1/1]/state/oper-status": "UP"
        }
      }
    ]
  }
]
```

Bad: empty `updates` array — path is wrong or model not supported.

**Step 3: Check path encoding (origin)**

Some devices require explicit origin in the path:

```bash
# Without origin (may fail on some devices):
gnmic -a <device_ip>:57400 --insecure subscribe \
  --path "/interfaces/interface/state/counters"

# With OpenConfig origin (try this if above gives no data):
gnmic -a <device_ip>:57400 --insecure subscribe \
  --path "openconfig:/interfaces/interface/state/counters"
```

**Step 4: Verify subscription mode (SAMPLE vs ON_CHANGE)**

```bash
# SAMPLE mode with 10-second interval:
gnmic -a <device_ip>:57400 --insecure subscribe \
  --path "/interfaces/interface/state/counters/in-octets" \
  --mode stream \
  --stream-mode sample \
  --sample-interval 10s

# ON_CHANGE mode (only sends updates when value changes — may appear empty if nothing changes):
gnmic -a <device_ip>:57400 --insecure subscribe \
  --path "/interfaces/interface/state/oper-status" \
  --mode stream \
  --stream-mode on-change
```

Expected (good, SAMPLE mode): one update every 10 seconds with current counter value.

Bad: no output even after 30s — either path is wrong or SAMPLE interval is below device minimum.

**Step 5: Check TLS certificate requirements**

```bash
# Check if device requires TLS:
gnmic -a <device_ip>:57400 capabilities    # No --insecure flag
```

If device requires TLS but gnmic is using `--insecure`, the connection may be accepted at transport layer but device rejects subscription.

Fix (using device self-signed cert):
```bash
gnmic -a <device_ip>:57400 \
  --tls-ca /path/to/device-ca.pem \
  --tls-cert /path/to/client.pem \
  --tls-key /path/to/client-key.pem \
  subscribe --path "/interfaces/interface/state/counters/in-octets" \
  --mode stream --stream-mode sample --sample-interval 10s
```

**Step 6: Check for SAMPLE interval minimum**

Some devices reject SAMPLE intervals below a minimum (e.g., 1 second). If you request 100ms, the device may silently ignore the subscription.

```bash
# Check device-specific min sample interval (SR Linux example):
docker exec clab-lab01-leaf1 sr_cli \
  "info /system/gnmi-server subscription-profiles"
```

### Root Causes and Fixes

| Root Cause | Diagnostic Signal | Fix |
|---|---|---|
| Wrong path prefix / unsupported model | `get` returns empty `updates` | Run `capabilities` to find supported model names; verify exact path from model YANG |
| SAMPLE interval too short | No data, no error | Increase `--sample-interval` to 10s or check device minimum |
| Device requires TLS but gnmic using insecure | Connection succeeds, no data or auth error | Provide `--tls-ca` and client certs, or enable insecure mode on device |
| ON_CHANGE path never changes | No output, correct behavior | Switch to SAMPLE mode; or generate a change event (bounce an interface) |
| Wrong gNMI port | Connection refused | Default is 57400; check device config (`gnmi-server listen-address`) |
| Path uses wrong key syntax | Empty response | Use exact interface name: `interface[name=ethernet-1/1]` not `interface[name=Ethernet1/1]` |

**Quick connectivity and path validation one-liner:**

```bash
gnmic -a <device_ip>:57400 --insecure get \
  --path "/system/state/hostname" && echo "gNMI OK"
```

---

## General Troubleshooting Tips

**Isolate before scaling.** Always reproduce the problem with the smallest possible test case: single-node `ib_write_bw` before multi-node NCCL, two-node BGP before full fabric, single subscription path before full telemetry pipeline.

**Baseline before deployment.** Run `ib_write_bw`, `ib_read_lat`, and `nccl-tests` at cluster acceptance and store the results. Regressions are only visible if a baseline exists.

**Counters need rates, not absolute values.** Always poll counters twice (with a known interval) and compute rates. An absolute counter of 10,000 errors could be from three years ago or from the last second.

**Log with timestamps.** All NCCL, RDMA, and routing debug logs should include microsecond-resolution timestamps. Many hangs and flaps last under 500 ms and are invisible without high-resolution logging.

**Check NUMA affinity.** A NIC on NUMA node 1 driving a GPU on NUMA node 0 creates a silent PCIe cross-NUMA bottleneck. Always verify with `lstopo` and `nvidia-smi topo -m`.

```bash
# One-liner NUMA affinity check:
lstopo --output-format console | grep -E "PCI|GPU|NIC"
nvidia-smi topo --matrix
```

---

*For hardware identification referenced in this cookbook, see Appendix E. For definitions of diagnostic terms (DCQCN, ECN, PFC, QP, MOFED, etc.), see Appendix F.*


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).