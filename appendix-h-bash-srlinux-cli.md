# Appendix H — Bash & SR Linux CLI Reference

**Reference cheat sheet for infrastructure engineers running the book's Containerlab labs.**
Two parts: Linux/Bash utilities grouped by purpose, then SR Linux CLI patterns.

---

## Part 1 — Linux / Bash Utilities

### H.1 Network Inspection

**`ip link`** — List all network interfaces and their link-layer state.

```bash
ip -j link show                   # JSON output (iproute2 >= 5.7)
ip link show dev eth0             # Single interface
ip link set eth0 mtu 9000         # Set jumbo-frame MTU
ip link set eth0 up               # Admin-up an interface
```

**`ip addr`** — Show or manage IP addresses assigned to interfaces.

```bash
ip addr show                      # All interfaces
ip addr show dev eth0             # One interface
ip -4 addr show up                # Only IPv4, only up interfaces
ip addr add 10.0.0.1/24 dev eth0  # Assign address
```

**`ip route`** — Inspect and manipulate the kernel routing table.

```bash
ip route show                          # Main table
ip route show table all                # All routing tables
ip route get 10.128.0.1                # Exact next-hop lookup
ip route add 192.168.100.0/24 via 10.0.0.254 dev eth0
```

**`ip neigh`** — ARP/NDP neighbor table.

```bash
ip neigh show                          # All entries
ip neigh show dev eth0                 # Per-interface
ip neigh flush dev eth0                # Clear stale entries
```

**`ip netns`** — Linux network namespaces (used by containers and Containerlab).

```bash
ip netns list                          # List namespaces
ip netns exec clab-lab1-leaf1 ip link  # Run command inside namespace
ip netns add testns && ip netns del testns
```

**`ss`** — Socket statistics, replacement for `netstat`.

```bash
ss -tulpn                              # TCP+UDP listening sockets with process
ss -antp state established             # Established TCP, with process
ss -s                                  # Summary statistics
ss -o state fin-wait-1                 # Sockets in a specific TCP state
```

**`ethtool`** — NIC configuration and diagnostics.

```bash
ethtool eth0                           # Link speed, duplex, auto-neg
ethtool -S eth0                        # Driver statistics (rx_dropped, tx_errors…)
ethtool -i eth0                        # Driver name and version
ethtool -k eth0                        # Offload features (GRO, TSO, RSS…)
ethtool -G eth0 rx 8192 tx 8192        # Set ring buffer sizes
ethtool -A eth0 rx on tx on            # Enable flow control (pause frames)
ethtool -l eth0                        # Combined / RSS queue count
ethtool -L eth0 combined 16            # Set queue count
```

**`tc qdisc show`** — Traffic-control queuing disciplines (ECN, DSCP, PFC shaping).

```bash
tc qdisc show dev eth0                 # Show active qdiscs
tc -s qdisc show dev eth0             # With drop/byte counters
tc qdisc add dev eth0 root handle 1: prio bands 3  # Add prio qdisc
tc filter show dev eth0                # Show filters (DSCP matching, etc.)
```

**`tcpdump`** — Packet capture with BPF filtering.

```bash
tcpdump -i eth0 -n -e                  # No DNS resolution, show MAC addresses
tcpdump -i eth0 -s 128 -w /tmp/cap.pcap  # Capture to file, 128-byte snaplen
tcpdump -i eth0 'port 4791'            # RoCEv2 UDP traffic
tcpdump -i any -n 'host 10.0.0.1 and (tcp or udp)' -c 1000
```

**`tshark`** — Terminal Wireshark; dissects protocols `tcpdump` can only capture.

```bash
tshark -i eth0 -Y 'roce'              # Filter decoded RoCE frames
tshark -r /tmp/cap.pcap -T json -e frame.number -e ip.src -e ip.dst
tshark -i eth0 -q -z io,stat,1        # Per-second throughput summary
```

---

### H.2 Hardware Enumeration

**`lspci`** — PCI device inventory (GPUs, HCAs, NICs, NVMe).

```bash
lspci -vvv -d 10de:                   # NVIDIA devices, verbose
lspci -vvv -d 15b3:                   # Mellanox/NVIDIA HCAs
lspci -nn | grep -i 'network\|infiniband\|vga'
lspci -tv                             # Tree view of PCI topology
```

**`numactl`** — NUMA topology and CPU/memory affinity.

```bash
numactl --hardware                     # NUMA nodes, CPUs, distances
numactl --show                         # Current process NUMA policy
numactl --cpunodebind=0 --membind=0 -- ib_write_bw -d mlx5_0  # Pin to NUMA 0
```

**`lstopo`** (`hwloc`) — Visual + text hardware topology (CPU, cache, NUMA, PCI, GPU).

```bash
lstopo --of ascii                      # ASCII art to terminal
lstopo --output-format png -o topo.png # Save PNG
lstopo --no-io                         # Omit I/O devices, show CPU/NUMA/cache only
hwloc-calc --intersect PU NUMANode:0   # List PU (logical CPU) indexes in NUMA node 0
```

**`lshw`** — Detailed hardware inventory from DMI + sysfs.

```bash
lshw -class network                    # All network devices
lshw -class processor -short           # CPU summary
lshw -json | jq '.children[].children[].id'
```

**`dmidecode`** — BIOS/DMI data: memory type, speed, slots, system serial.

```bash
dmidecode -t memory                    # DIMMs: speed, size, ECC
dmidecode -t processor                 # CPU model, socket, core count
dmidecode -t 1                         # System information (serial, UUID)
```

---

### H.3 RDMA / InfiniBand Tools

**`ibstat`** — Per-port state, rate, and LID for each HCA port.

```bash
ibstat                                 # All HCAs
ibstat mlx5_0                          # One device
ibstat mlx5_0 1                        # Port 1 of mlx5_0
# Look for: State: Active, Rate: 400, LID assigned
```

**`ibstatus`** — Concise one-line per port status (good for scripting).

```bash
ibstatus                               # All ports
ibstatus mlx5_0                        # One device
```

**`ibdev2netdev`** — Map IB device names to Ethernet netdev names (RoCEv2).

```bash
ibdev2netdev                           # mlx5_0 ==> eth0 (Up)
ibdev2netdev -v                        # Verbose with PCI BDF
```

**`ibnetdiscover`** — Discover InfiniBand subnet topology from the SM.

```bash
ibnetdiscover                          # Full topology dump
ibnetdiscover -p                       # Show ports
ibnetdiscover --node-name-map /tmp/nodemap.txt  # Map GUIDs to hostnames
```

**`ibping`** — InfiniBand-layer ping (requires SM with `opensm` running).

```bash
ibping -S                              # Start server (on remote node)
ibping -G 0x0002c903001e2820           # Ping by GUID from client
ibping -L 5 -C mlx5_0 -P 1            # Ping LID 5 via mlx5_0 port 1
```

**`perfquery`** — Read performance counters from IB ports (bandwidth, errors).

```bash
perfquery                              # Local port counters
perfquery -x                           # Extended 64-bit counters
perfquery -G 0x0002c903001e2820 1      # Remote port by GUID
perfquery -R                           # Reset counters after read
```

**`rdma dev`** — List RDMA devices (iproute2 rdma tool).

```bash
rdma dev                               # Devices and their state
rdma dev show mlx5_0                   # Single device detail
rdma link                              # Logical RDMA link endpoints
rdma statistic                         # Per-device statistics
rdma statistic mode auto               # Enable auto-collection mode
```

**`mlxconfig`** — Persistent NIC firmware configuration (SRIOV, VPI mode, PFC).

```bash
mlxconfig -d /dev/mst/mt4129_pciconf0 query  # Dump all settings
mlxconfig -d /dev/mst/mt4129_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=32
mlxconfig -d /dev/mst/mt4129_pciconf0 set LINK_TYPE_P1=2  # 2=Ethernet
# Requires reboot or mlxfwreset to take effect
```

**`mlxlink`** — Live link diagnostics: FEC, errors, eye diagram, signal quality.

```bash
mlxlink -d /dev/mst/mt4129_pciconf0 -p 1         # Link state + FEC counters
mlxlink -d /dev/mst/mt4129_pciconf0 -p 1 -m      # Module (transceiver) info
mlxlink -d /dev/mst/mt4129_pciconf0 -p 1 --show_eye  # Eye diagram (PAM4 lanes)
mlxlink -d /dev/mst/mt4129_pciconf0 -p 1 --pc    # Clear error counters
```

---

### H.4 GPU Management

**`nvidia-smi`** — NVIDIA System Management Interface; most-used GPU tool.

```bash
nvidia-smi                             # One-shot health summary
nvidia-smi dmon -s pumet -d 1         # Device Monitor: power/util/memory/enc+dec+temp, 1 s
nvidia-smi topo -m                     # GPU-to-GPU connectivity matrix (NVLink, PCIe)
nvidia-smi nvlink --status -i 0       # NVLink status for GPU 0
nvidia-smi nvlink --errorcounters -i 0 # Per-lane error counters
nvidia-smi mig -lgip                   # List GPU instance profiles (A100/H100)
nvidia-smi mig -cgi 9,14 -C -i 0      # Create 2×MIG instances on GPU 0
nvidia-smi -q -d MEMORY -i 0          # Detailed memory query for GPU 0
nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used \
           --format=csv,noheader       # Scriptable CSV output
```

**`nvitop`** — Interactive, real-time GPU process monitor (like `htop` for GPUs).

```bash
nvitop                                 # Full-screen interactive view
nvitop -m compact                      # Compact single-line per GPU
nvitop --ascii                         # ASCII-only (SSH without UTF-8)
```

**`dcgmi`** — DCGM (Data Center GPU Manager) CLI: health, diagnostics, field watches.

```bash
dcgmi discovery -l                     # List GPUs managed by DCGM
dcgmi health -g 0 -c                   # Check health group 0
dcgmi diag -r 1 -g 0                   # Rapid diagnostic (level 1)
dcgmi dmon -e 1002,1003,1004 -d 1000   # Watch fields: SM clock, mem clock, temp
dcgmi stats -g 0 -e                    # Enable job stats for group
```

---

### H.5 Process & Performance

**`perf stat`** — Hardware performance counters for a command or PID.

```bash
perf stat -e cache-misses,cache-references,cycles,instructions \
          ./my_workload
perf stat -p $(pgrep nccl_test) -I 1000   # Attach to running PID, 1 s intervals
perf stat -a sleep 5                       # System-wide for 5 seconds
```

**`perf record` / `perf report`** — CPU profiling with call-graph support.

```bash
perf record -g -F 99 -p $(pgrep python3) -- sleep 30   # Sample at 99 Hz
perf report --stdio --no-children                       # Flat profile
perf report -g graph,0.5,caller                         # Caller-oriented call graph
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

**`strace`** — System call trace; useful for debugging driver/kernel interactions.

```bash
strace -p $(pgrep nccl) -e trace=network,ipc -f    # Network + IPC syscalls
strace -c ./my_app                                  # System call summary count
strace -T ./my_app 2>&1 | grep -E 'write|read'     # Show time per call
```

**`lsof`** — List open files, sockets, and device handles.

```bash
lsof -p $(pgrep python3)                     # All open files for a process
lsof -i :4791                                # Processes with RoCEv2 port open
lsof /dev/infiniband/uverbs0                 # Who has RDMA uverbs open
```

**`dmesg`** — Kernel ring buffer; first place to look for driver and link errors.

```bash
dmesg -T --level=err,warn | tail -40         # Recent errors with timestamps
dmesg -T -w                                  # Follow (like tail -f)
dmesg | grep -i 'mlx5\|roce\|infiniband'     # NIC/RDMA driver messages
```

**`journalctl`** — Systemd journal; covers services like `openibd`, `opensmd`.

```bash
journalctl -u openibd -n 50 --no-pager       # Last 50 lines from IB daemon
journalctl -k -S "1 hour ago"               # Kernel messages from last hour
journalctl -f -u containerd                  # Follow containerd logs
```

**`top` / `htop`** — Interactive process/CPU view.

```bash
top -H -p $(pgrep nccl_test)                 # Thread view for one PID
htop -d 5 -u root                            # 0.5 s refresh, root-owned procs
```

**`iotop`** — Per-process I/O bandwidth.

```bash
iotop -oPa                                   # Only active procs, accumulated
iotop -d 2 -n 10                             # 2 s interval, 10 iterations
```

---

### H.6 Data Wrangling

**`jq`** — JSON query and transformation on CLI output.

```bash
ip -j route show | jq '.[] | {dst: .dst, gw: .gateway}'
kubectl get pods -o json | jq '.items[] | {name: .metadata.name, ip: .status.podIP}'
jq -r '.[] | [.index, .name, .pci.bus_id] | @csv' <<< "$(nvidia-smi -q -d PCI --json)"
```

**`yq`** — YAML query (same syntax as `jq`).

```bash
yq '.nodes[].interfaces' topo.clab.yaml
yq -i '.spec.replicas = 4' deploy.yaml     # In-place edit
cat cilium-values.yaml | yq '.kubeProxyReplacement'
```

**`awk`** — Field-based text processing.

```bash
awk '{sum += $NF} END {print sum}' latencies.txt   # Column sum
awk -F',' 'NR>1 && $3 > 80 {print $1, $3}' gpu.csv # Filter rows
netstat -s | awk '/retransmit/{print $0}'
```

**`sed`** — Stream editor for substitutions.

```bash
sed -i 's/autonomous-system 65001/autonomous-system 65100/g' spine1.cfg
sed -n '/^interface/,/^!/p' router.cfg    # Print interface stanzas
```

**`grep`** — Pattern search with context.

```bash
grep -r 'RNR' /var/log/                   # Recursive search for IB retransmit errors
grep -E 'mlx5|roce' /proc/net/dev
dmesg | grep -C 3 'link down'             # 3 lines of context
```

**`sort` / `uniq`** — Sort and deduplicate output.

```bash
ibnetdiscover | grep 'Switch\|CA' | sort -u
sort -k3 -n latencies.txt | tail -10      # Top-10 highest latencies
```

**`column`** — Align output into a table.

```bash
ss -tulpn | column -t
cat /proc/net/dev | column -t
nvidia-smi --query-gpu=name,utilization.gpu --format=csv | column -t -s ','
```

**`watch`** — Repeat a command, highlight changes.

```bash
watch -n 1 -d 'ip -s link show eth0'      # 1 s refresh, changed values highlighted
watch -n 2 'nvidia-smi dmon -c 1'         # GPU metrics every 2 s
watch -n 0.5 'ss -s'                      # Socket summary at 500 ms
```

---

### H.7 Remote & Session

**`ssh`** — Secure shell with useful flags for lab work.

```bash
ssh -J bastion user@leaf1                 # Jump host
ssh -L 3000:localhost:3000 user@mgmt      # Local port forward (Grafana tunnel)
ssh -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=/dev/null \
    admin@clab-lab1-leaf1                 # Skip host-key check for lab VMs
ssh -i ~/.ssh/lab_key -p 2222 user@host   # Custom key and port
```

**`rsync`** — Efficient remote file synchronisation.

```bash
rsync -avz --progress ./configs/ user@spine1:/home/user/configs/
rsync -avz --exclude '*.log' ./results/ user@storage:/data/
rsync --dry-run -avz src/ dst/            # Preview without transferring
```

**`tmux`** — Terminal multiplexer; essential for multi-node lab sessions.

```text
tmux new -s lab                           # New session named "lab"
tmux attach -t lab                        # Re-attach
<Prefix> = Ctrl-b by default
Ctrl-b c        New window
Ctrl-b "        Horizontal split (new pane below)
Ctrl-b %        Vertical split (new pane right)
Ctrl-b o        Next pane
Ctrl-b z        Zoom/unzoom current pane
Ctrl-b [        Scroll mode (q to exit)
Ctrl-b d        Detach session
```

**`parallel`** — Run commands across nodes concurrently (GNU Parallel).

```bash
parallel -j8 ssh admin@{} 'show version' ::: leaf1 leaf2 spine1 spine2
parallel --eta scp configs/{}.cfg {}:/etc/frr/frr.conf ::: leaf1 leaf2
cat hostlist.txt | parallel -j4 'ibstat | grep Rate' # Pipe host list
```

**`xargs`** — Build command lines from stdin.

```bash
echo "leaf1 leaf2 spine1" | xargs -n1 -I{} ssh admin@{} 'uptime'
find /var/log -name '*.log' -mtime +30 | xargs -P4 gzip
kubectl get nodes -o name | xargs -I{} kubectl describe {}
```

---

### H.8 Container / Kubernetes Shortcuts

**`kubectl`** — Power-user patterns.

```bash
# JSONPath queries
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
kubectl get node gpu-node-01 -o jsonpath='{.status.capacity.nvidia\.com/gpu}'

# Field selectors
kubectl get pods --field-selector status.phase=Running -A
kubectl get events --field-selector reason=BackOff --sort-by=.lastTimestamp

# Watch for changes
kubectl get pods -w                             # Watch pod state transitions
kubectl rollout status deployment/nccl-test -w  # Watch rollout

# Exec into a GPU pod
kubectl exec -it $(kubectl get pod -l app=nccl -o name | head -1) \
    -- nvidia-smi topo -m

# Logs from all pods matching a label
kubectl logs -l app=nccl --all-containers --prefix --tail=50

# Resource usage
kubectl top pods -A --sort-by=memory
kubectl top nodes
```

**`docker stats`** — Live container resource usage.

```bash
docker stats --no-stream --format \
  "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
docker stats clab-lab1-leaf1            # Single container
```

**`crictl`** — CRI-compatible container runtime CLI (works with containerd/cri-o).

```bash
crictl ps                               # Running containers
crictl pods                             # Pod sandbox list
crictl stats -a                         # Resource stats for all containers
crictl logs <container-id>              # Container logs without Docker
crictl exec -it <container-id> sh       # Exec without Docker
```

---

## Part 2 — SR Linux CLI

SR Linux is Nokia's YANG-native network operating system, used throughout the book's Containerlab labs. Every configuration path corresponds to a YANG node; there is no free-form text configuration. The CLI exposes three datastores — **running**, **candidate**, and **state** — through a consistent, transactional interface.

### H.9 Datastores and Modes

| Mode | Prompt indicator | Purpose |
|------|-----------------|---------|
| **running** | `--{ running }--` | Default; read-only configuration view. Show and ping commands work here. |
| **candidate** | `--{ candidate shared default }--` | Staging area for configuration changes. `set` and block edits apply here. |
| **state** | `--{ state }--` | Configuration + live operational values (counters, BGP peer states, uptime). |

The asterisk `*` in the prompt (`--{ * candidate shared default }--`) indicates staged but uncommitted changes.

**Entering and exiting modes:**

```text
--{ running }--[  ]--
A:leaf1# enter candidate

--{ candidate shared default }--[  ]--
A:leaf1# enter state

--{ state }--[  ]--
A:leaf1# enter running

--{ running }--[  ]--
A:leaf1# quit
```

`exit` navigates one level up in the context hierarchy. `exit to /` returns to the root context. `quit` exits the CLI session entirely.

---

### H.10 Navigating the YANG Tree

SR Linux context paths mirror the YANG hierarchy. Tab-completion works at every level.

```text
--{ running }--[  ]--
A:leaf1# interface ethernet-1/1

--{ running }--[ interface ethernet-1/1 ]--
A:leaf1# subinterface 0

--{ running }--[ interface ethernet-1/1 subinterface 0 ]--
A:leaf1# exit

--{ running }--[ interface ethernet-1/1 ]--
A:leaf1# exit to /

--{ running }--[  ]--
A:leaf1#
```

Prefix a path with `/` for an absolute context jump from anywhere:

```text
A:leaf1# / network-instance default protocols bgp
--{ running }--[ network-instance default protocols bgp ]--
```

**`info`** — The primary inspection command. Behaviour depends on current mode.

```text
# In running mode: show running configuration at current context
A:leaf1# info

# Show a specific path without navigating there
A:leaf1# info /interface ethernet-1/1

# Limit output depth
A:leaf1# info depth 1

# Show both config and state without entering state mode
A:leaf1# info from state /interface ethernet-1/1

# Flat (set-syntax) format — useful for copy-paste into scripts
A:leaf1# info flat
```

**`info flat`** emits output as `set /path/to/leaf value` lines, making it easy to reproduce a configuration or diff two devices.

---

### H.11 Making Configuration Changes

All changes are staged in the candidate datastore. They become active only after a successful commit.

**Entering candidate mode and using `set`:**

```text
--{ running }--[  ]--
A:leaf1# enter candidate

--{ candidate shared default }--[  ]--
A:leaf1# set / interface ethernet-1/1 admin-state enable
A:leaf1# set / interface ethernet-1/1 description "spine1-link"
A:leaf1# set / interface ethernet-1/1 subinterface 0 ipv4 admin-state enable
A:leaf1# set / interface ethernet-1/1 subinterface 0 ipv4 address 192.168.11.1/30
```

The context-navigation style (no `set` prefix) is equivalent:

```text
--{ candidate shared default }--[  ]--
A:leaf1# interface ethernet-1/1
A:leaf1# admin-state enable
A:leaf1# description "spine1-link"
A:leaf1# exit
```

Block (declarative) syntax for multi-leaf changes:

```text
A:leaf1# / interface ethernet-1/1 {
    admin-state enable
    description "spine1-link"
    subinterface 0 {
        ipv4 {
            admin-state enable
            address 192.168.11.1/30 { }
        }
    }
}
```

**`diff`** — Compare candidate against the running datastore before committing.

```text
--{ * candidate shared default }--[  ]--
A:leaf1# diff

    interface ethernet-1/1 {
+       admin-state enable
+       description "spine1-link"
+       subinterface 0 {
+           ipv4 {
+               admin-state enable
+               address 192.168.11.1/30 { }
+           }
+       }
    }
```

Lines prefixed `+` are additions; `-` are deletions. Run `diff` before every commit in production.

**`commit now`** — Apply staged changes immediately.

```text
A:leaf1# commit now
All changes have been committed. Starting new transaction.
```

**`commit confirmed <minutes>`** — Apply changes with an automatic rollback timer. The commit becomes permanent only if explicitly confirmed within the timeout window; if nothing happens before the timer expires, the switch rolls back automatically. Safe for production changes.

```text
A:leaf1# commit confirmed 5
Commit confirmed, rolling back in 5 minutes unless confirmed.
# ... verify the change works as expected ...
A:leaf1# tools system configuration confirmed-accept
```

**`discard`** — Abandon all staged changes in the candidate datastore.

```text
--{ * candidate shared default }--[  ]--
A:leaf1# discard
```

`discard` exits to running mode after clearing staged changes. `discard stay` clears staged changes and keeps you in candidate mode, ready for a fresh edit session.

---

### H.12 Operational Commands

These run in any mode (running, state, or candidate). Full paths are required when not at root context; prefix with `/ ` (forward slash + space) to force root resolution.

**`show version`** — System software release and platform.

```text
A:leaf1# show version
------------------------------------------------------------------------------------------
Hostname             : leaf1
Chassis Type         : 7220 IXR-D2
Part Number          : 3HE13719AARB01
Serial Number        : NS2152M0001
System HW MAC Address: 1A:2B:3C:4D:5E:6F
OS                   : SR Linux
Software Version     : v24.10.1
Build Number         : 319-gd61ef2f29a
Architecture         : x86_64
Last Booted          : 2025-04-30T08:12:33.000Z
Total Memory         : 24575 MB
Free Memory          : 17842 MB
------------------------------------------------------------------------------------------
```

**`show interface`** — Interface counters, speed, and operational state.

```text
A:leaf1# show interface ethernet-1/1
==========================================================================
  ethernet-1/1 is up, speed 100G, type None
--------------------------------------------------------------------------
  ethernet-1/1.0 is up
    Network-instance  : default
    Encapsulation     : null
    Type              : routed
    IPv4 addr         : 192.168.11.1/30 (static, preferred)
--------------------------------------------------------------------------
  Traffic statistics:
    Input    :  4817219284 octets,   3205638 packets,  0 error
    Output   :  5103441921 octets,   3398124 packets,  0 error
==========================================================================
```

**`show network-instance`** — Network instance (VRF) summary.

```text
A:leaf1# show network-instance
+-------------------+-----------+------------------+---------+
| Name              | Type      | Admin state      | Oper    |
+===================+===========+==================+=========+
| default           | default   | N/A              | up      |
| mgmt              | ip-vrf    | enable           | up      |
+-------------------+-----------+------------------+---------+
```

**`show route-table`** — IP routing table for a network instance.

```text
A:leaf1# show network-instance default route-table ipv4-unicast summary
+-------------------+-------+---+-----------+--------+---------+
| Prefix            | Proto | P | Metric    | Weight | NextHop |
+===================+=======+===+===========+========+=========+
| 10.0.0.1/32       | local | 0 | 0         |        | system0 |
| 10.0.0.2/32       | bgp   | 0 | 0         |        | 192.168.11.2 |
+-------------------+-------+---+-----------+--------+---------+
```

**`show network-instance default protocols bgp summary`** — BGP session table.

```text
A:leaf1# show network-instance default protocols bgp summary
+---------------------+------+------------+----------+---------+-----------+
| Neighbor            | AS   | State      | Up/Down  | #Received | #Sent   |
+=====================+======+============+==========+===========+=========+
| 192.168.11.2        | 65100| Established| 02:14:07 | 4         | 4       |
+---------------------+------+------------+----------+-----------+---------+
```

**`tools interface`** — Operational actions on interfaces (clear stats, bounce).

```text
A:leaf1# tools interface ethernet-1/1 statistics clear
```

**`tools system configuration save`** — Write running config to the startup-config file.

```text
A:leaf1# tools system configuration save
/etc/opt/srlinux/config.json
```

---

### H.13 Output Formatting

SR Linux output modifiers are appended with a pipe (`|`) after any command.

```text
# JSON — machine-readable, pipe to jq
A:leaf1# show version | as json
{
  "hostname": "leaf1",
  "chassis_type": "7220 IXR-D2",
  "sw_version": "v24.10.1",
  ...
}

# YAML
A:leaf1# info /interface ethernet-1/1 | as yaml

# Table — aligned columns for wide output
A:leaf1# show network-instance | as table

# grep — filter lines
A:leaf1# show interface | grep -A3 "ethernet-1/4"

# Paginate long output
A:leaf1# info / | more

# jq on JSON output
A:leaf1# show version | as json | jq '.sw_version'
```

Chain modifiers for compound transforms:

```text
A:leaf1# info from state / | as json | jq '[.. | objects | select(.oper_state? == "down") | .name]'
```

---

### H.14 One-Shot Execution from Shell

`sr_cli` is the SR Linux CLI binary. When invoked with `-c`, it executes a single command string and exits — useful for scripting, Ansible, and Containerlab startup hooks.

```bash
# One-shot from the host (Containerlab lab node)
docker exec clab-lab1-leaf1 sr_cli -c "show version"

# Multiple commands separated by newlines
docker exec clab-lab1-leaf1 sr_cli -c "
enter candidate
set / interface ethernet-1/1 description lab-link
commit now
"

# Capture JSON for parsing
docker exec clab-lab1-leaf1 sr_cli -c "show version | as json" \
  | jq '.sw_version'

# SSH then one-shot
ssh admin@clab-lab1-leaf1 'sr_cli -c "show interface | as json"' \
  | jq '.interface[] | {name: .name, oper: .oper_state}'
```

---

### H.15 Key Differences from IOS / EOS

| Concept | Cisco IOS / Arista EOS | SR Linux |
|---------|----------------------|----------|
| Privilege escalation | `enable` → privileged EXEC | No `enable` mode; permissions are role-based at login |
| Configuration mode | `configure terminal` | `enter candidate` (transactional datastore) |
| Apply configuration | Writes immediately (`no shutdown`) | Staged — requires `commit now` or `commit confirmed` |
| Rollback | `archive` config + manual restore | `discard` in candidate; `commit confirmed` auto-rollback |
| Data model | Proprietary, line-oriented | 100% YANG; every path maps to a YANG node |
| State vs config | `show` commands mix config and state | Separate datastores (`running` vs `state`); `info from state` for counters |
| Scripting access | EEM / Tcl; IOS-XE Python | `sr_cli -c` one-shot; NDK for on-box apps; gNMI for streaming |
| Configuration format | Text, vendor-specific | Hierarchical block or flat `set /path value`; exports as JSON/YAML |
| Help system | `?` suffix on partial command | `?` after any keyword; Tab for autocompletion with typo correction |

---

*© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
