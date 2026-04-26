# Chapter 12 — eBPF-Native Kubernetes: Cilium

**Part IV: Overlay & Kubernetes Networking** | ~25 pages

---

## Installation

### Kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/
kind version
# kind v0.22.0 go1.21.0 linux/amd64
```

### kubectl

```bash
sudo apt install -y kubectl
kubectl version --client
# Client Version: v1.29.x
```

### Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
# version.BuildInfo{Version:"v3.14.x", ...}
```

### Cilium CLI

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
    https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz
cilium version --client
# cilium-cli: v0.16.x ...
```

### Hubble CLI

```bash
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --fail --remote-name-all \
    https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz
hubble version
# hubble v0.13.x compiled with go1.21.x ...
```

### Python environment (Kubernetes automation)

```bash
uv venv .venv && source .venv/bin/activate
uv pip install kubernetes pyyaml requests
python -c "import kubernetes, yaml, requests; print('OK')"
```

---

## Introduction

Kubernetes was designed with a flat pod-networking model: every pod gets an IP, every pod can reach every other pod, and a component called `kube-proxy` implements Service load balancing by installing iptables rules. This model works at small scale, but in AI clusters running hundreds of GPU nodes and thousands of pods, the iptables approach breaks down under O(N) rule traversal, O(N) update time, and complete absence of per-flow observability. A failed NCCL collective that blocks gradient synchronization may be caused by a dropped packet anywhere in the overlay, but traditional Kubernetes gives operators no tool to see it.

Cilium is a Kubernetes CNI (Container Network Interface) plugin that replaces iptables and kube-proxy entirely with eBPF programs. eBPF (extended Berkeley Packet Filter) is a Linux kernel subsystem that allows sandboxed programs to be loaded into the kernel at runtime and attached to network hooks — without kernel modules or kernel recompilation. Cilium attaches eBPF programs to each pod's virtual network interface, implementing network policy enforcement, Service load balancing, and traffic observability in O(1) hash-map lookups rather than O(N) rule chains.

This chapter covers Cilium's architecture, the BPF datapath mechanics, identity-based network policy, kube-proxy replacement with Service BPF maps, Direct Server Return (DSR), BGP Control Plane integration with the fabric, and Hubble — Cilium's per-flow observability layer. Each section builds toward the practical concern of an AI cluster operator: how do you enforce isolation between training jobs, debug a connectivity failure between NCCL workers, and verify that your network policies are not silently dropping gradient-synchronization traffic?

The lab walkthrough deploys a Kind (Kubernetes-in-Docker) cluster with Cilium as the sole CNI, demonstrates policy enforcement between trainer and gpu-worker pods, and uses Hubble's CLI and web UI to trace FORWARDED and DROPPED flows in real time.

This chapter sits at the intersection of Chapters 7 (eBPF and XDP fundamentals) and 11 (overlay networking). Chapter 13 extends the picture by adding SR-IOV secondary interfaces to the same pods — enabling RDMA traffic to coexist with Cilium-managed management traffic in a single multi-NIC GPU pod.

---

## 12.1 Why Kubernetes Networking Needed Reinventing

Kubernetes' original network model assumed iptables: a set of NAT and filter rules installed by `kube-proxy` — the Kubernetes component responsible for implementing Service virtual IPs by programming iptables DNAT rules on every node — to implement Services. At 100 pods per node and 1000 nodes, this means:
- 100,000+ iptables rules per node
- O(N) rule traversal per packet for random Service selection
- Rule update time O(N) on every Service change — blocking traffic during update
- No flow-level visibility — only host-level counters

At AI cluster scale (thousands of pods, hundreds of services), iptables doesn't work. Cilium replaces the entire networking and observability model with eBPF programs that operate in O(1) via BPF maps.

---

## 12.2 Cilium Architecture

```
┌──────────────────────────────────────────────────┐
│ Kubernetes API Server                            │
│  NetworkPolicy, Service, Endpoint objects        │
└────────────────────┬─────────────────────────────┘
                     │ Watch via K8s API
          ┌──────────┴──────────┐
          │  cilium-agent       │  (DaemonSet, one per node)
          │  - Policy engine    │
          │  - Endpoint manager │
          │  - BPF compiler/loader│
          └──────────┬──────────┘
                     │ loads/updates BPF programs
          ┌──────────┴──────────────────────────────┐
          │  eBPF programs (per endpoint)           │
          │  tc ingress/egress  XDP  socket         │
          └──────────┬──────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          │  BPF Maps           │
          │  ct_map (conntrack) │
          │  lb_map (services)  │
          │  policy_map         │
          │  ipcache            │
          └─────────────────────┘
```

Cilium agents on each node watch the Kubernetes API, translate policy and service objects into BPF map entries, and attach BPF programs to each pod's network interface. The `cilium-agent` runs as a DaemonSet — a Kubernetes workload type that ensures exactly one instance runs on every node in the cluster.

---

## 12.3 Cilium Installation (Helm)

```bash
# Install via Helm (recommended)
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
    --namespace kube-system \
    --set kubeProxyReplacement=true \      # replace kube-proxy entirely
    --set k8sServiceHost=<API_SERVER_IP> \
    --set k8sServicePort=6443 \
    --set hubble.relay.enabled=true \       # Hubble for observability
    --set hubble.ui.enabled=true \
    --set ipam.mode=kubernetes             # use k8s IPAM

# Verify
cilium status --wait
cilium connectivity test
```

---

## 12.4 CNI Data Path

When a pod is created, Cilium:
1. Creates a `veth` pair — a virtual Ethernet cable consisting of two linked interfaces; packets sent into one end emerge from the other — with one end inside the pod's network namespace and one end on the host
2. Attaches a TC ingress BPF program to the host-side veth
3. Attaches a TC egress BPF program to the host-side veth
4. Assigns the pod an identity (a 32-bit numeric label derived from its K8s labels)
5. Updates BPF maps: `ipcache` (pod IP → identity), `policy_map` (per-endpoint allowed traffic)

### Packet Flow (pod A → pod B, same node)

```
Pod A sends packet:
  vethA host-side egress BPF:
    - lookup src identity from ipcache
    - evaluate network policy
    - if allowed: XDP_REDIRECT to vethB via BPF redirect map  # XDP (eXpress Data Path): a kernel hook at the earliest NIC driver level, bypassing most of the kernel network stack
  vethB host-side ingress BPF:
    - no kernel routing, no iptables
    - packet delivered directly to Pod B's netns
```

The kernel network stack is bypassed entirely for east-west same-node traffic.

---

## 12.5 kube-proxy Replacement

With `kubeProxyReplacement=true`, Cilium implements all kube-proxy functions in BPF:

### Service Load Balancing

```bash
# Inspect Cilium's service map
cilium service list

# Service 10.96.0.1:443 → backend pods
# ID   Frontend         Service Type   Backend
# 1    10.96.0.1:443    ClusterIP      10.244.1.5:6443 (1/1)

# Dump the BPF map directly
cilium bpf lb list
# SERVICE   FRONTEND         BACKEND
# 1         10.96.0.1:443    10.244.1.5:6443
```

Kubernetes Service selection is implemented as a hash lookup in the `lb_map` BPF map — O(1) regardless of Service count.

### DSR — Direct Server Return

For external traffic, Cilium supports DSR: the backend pod responds directly to the client without routing through the load balancer. This halves the load-balancer's bandwidth requirement:

```bash
helm upgrade cilium cilium/cilium \
    --set loadBalancer.mode=dsr \
    --set loadBalancer.dsrDispatch=opt
```

---

## 12.6 Network Policy

Cilium's network policy is **identity-based** (not IP-based). Policies reference K8s label selectors; Cilium resolves those to numeric identities and programs BPF `policy_map` entries.

### L3/L4 Policy

```yaml
# Allow pods with role=gpu-worker to receive from role=trainer on port 29500 (NCCL)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-nccl
spec:
  endpointSelector:
    matchLabels:
      role: gpu-worker
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: trainer
    toPorts:
    - ports:
      - port: "29500"
        protocol: TCP
```

### L7 Policy (HTTP)

```yaml
# Allow only GET /healthz on port 8080
spec:
  endpointSelector:
    matchLabels:
      app: metrics-server
  ingress:
  - toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: /healthz
```

L7 policy is enforced by an Envoy sidecar managed by Cilium — the BPF program redirects matching traffic to local Envoy for L7 inspection. Envoy is a high-performance L7 proxy originally built at Lyft; Cilium uses it as a transparent HTTP/gRPC inspection engine without requiring application changes.

---

## 12.7 Cilium BGP Control Plane

Cilium can peer with fabric BGP routers to advertise pod CIDRs and LoadBalancer IPs directly — eliminating the need for a separate BGP daemon:

```yaml
# CiliumBGPPeeringPolicy
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: fabric-peering
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
  virtualRouters:
  - localASN: 65001
    exportPodCIDR: true
    serviceSelector:
      matchExpressions:
      - key: somekey
        operator: NotIn
        values: ['never-used-value']  # select all services
    neighbors:
    - peerAddress: 192.168.1.0/24
      peerASN: 65000
      gracefulRestart:
        enabled: true
        restartTimeSeconds: 120
```

This configuration causes Cilium to peer with all fabric switches in 192.168.1.0/24 and advertise pod CIDR routes — the fabric learns pod reachability dynamically as nodes join and leave.

---

## 12.8 Hubble — Flow Observability

Hubble is Cilium's built-in observability layer. Every packet passing through a Cilium BPF program is recorded (or sampled) in a per-node ring buffer, exportable via gRPC:

```bash
# CLI: live flow trace
hubble observe --namespace default --follow

# Filter: dropped packets only (policy violations)
hubble observe --verdict DROPPED --follow

# Filter: NCCL traffic
hubble observe --port 29500 --follow

# Output:
# Apr 22 10:00:01.234  gpu-worker-0:48922 -> gpu-worker-7:29500  TCP ESTABLISHED
# Apr 22 10:00:01.235  gpu-worker-0:48922 -> gpu-worker-7:29500  TCP to-network FORWARDED
```

```bash
# Hubble UI (port-forward)
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
# Browse http://localhost:12000 for service dependency map
```

### Hubble Metrics to Prometheus

```bash
# Enable Hubble metrics
helm upgrade cilium cilium/cilium \
    --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp}"

# Prometheus scrapes from:
# http://<node-ip>:9091/metrics
# Key metrics:
# hubble_drop_total{direction,reason,protocol}
# hubble_flows_processed_total{type,verdict}
# hubble_tcp_flags_total{flag,family}
```

---

## 12.9 XDP Acceleration

For node-to-external traffic (LoadBalancer services), Cilium can run in XDP mode — processing Service DNAT at the NIC driver level before `sk_buff` allocation:

```bash
helm upgrade cilium cilium/cilium \
    --set loadBalancer.acceleration=native   # XDP native mode
    # or --set loadBalancer.acceleration=best-effort  # auto-detect
```

In XDP mode, load-balanced external traffic is handled at line rate with no kernel stack involvement — the same performance class as DPDK, with the ergonomics of Kubernetes.

---

## Lab Walkthrough 12 — Cilium Policy and Hubble

This walkthrough builds a Kind cluster with Cilium as the CNI, deploys trainer and gpu-worker pods, tests connectivity before and after applying a CiliumNetworkPolicy, and uses Hubble to observe forwarded and dropped flows. Kind (Kubernetes in Docker) runs a multi-node Kubernetes cluster entirely inside Docker containers — each "node" is a container — making it ideal for local CNI development and testing without real VMs.

### Step 1 — Write the Kind cluster configuration

Create `kind-cilium.yaml` — the key requirement is disabling Kind's default CNI so Cilium can install its own:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cilium-lab
networking:
  # Disable the default CNI (kindnet) so Cilium can manage pod networking
  disableDefaultCNI: true
  # Use a pod subnet that does not conflict with the host
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/16"
nodes:
  - role: control-plane
    # Mount the BPF filesystem so Cilium can pin BPF maps across restarts
    extraMounts:
      - hostPath: /sys/fs/bpf
        containerPath: /sys/fs/bpf
        propagation: Bidirectional
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
  - role: worker
    extraMounts:
      - hostPath: /sys/fs/bpf
        containerPath: /sys/fs/bpf
        propagation: Bidirectional
  - role: worker
    extraMounts:
      - hostPath: /sys/fs/bpf
        containerPath: /sys/fs/bpf
        propagation: Bidirectional
```

### Step 2 — Create the Kind cluster

```bash
kind create cluster --config kind-cilium.yaml
```

Expected output:

```
Creating cluster "cilium-lab" ...
 ✓ Ensuring node image (kindest/node:v1.29.2) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-cilium-lab"
You can now use your cluster with:

kubectl cluster-info --context kind-cilium-lab

Have a nice day! 👋
```

Verify the cluster is up but nodes are NotReady (no CNI yet):

```bash
kubectl get nodes
```

```
NAME                        STATUS     ROLES           AGE   VERSION
cilium-lab-control-plane    NotReady   control-plane   30s   v1.29.2
cilium-lab-worker           NotReady   <none>          10s   v1.29.2
cilium-lab-worker2          NotReady   <none>          10s   v1.29.2
```

`NotReady` is expected — the nodes have no CNI yet. Pods in `kube-system` will be Pending.

### Step 3 — Install Cilium

Install Cilium with kube-proxy replacement and Hubble enabled. Kind does not use an external API server, so we pass the internal API address:

```bash
API_SERVER_IP=$(kubectl get nodes cilium-lab-control-plane \
    -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')

cilium install \
    --version 1.15.5 \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost="${API_SERVER_IP}" \
    --set k8sServicePort=6443 \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set hubble.enabled=true
```

Expected output:

```
🔮 Auto-detected Kubernetes kind: Kind
🔮 Auto-detected kube-proxy has not been installed
ℹ️  Cilium version not set, auto-detecting...
🔮 Auto-detected Cilium version: 1.15.5
🔑 Found existing CA in secret cilium-ca
🔑 Generating certificates for Hubble...
🚀 Creating Service accounts...
🚀 Creating Cluster roles...
🚀 Creating ConfigMap for Cilium version 1.15.5...
🚀 Creating DaemonSet...
⌛ Waiting for Cilium to be installed and ready...
✅ Cilium was successfully installed! Run 'cilium status --wait' to view installation health
```

### Step 4 — Wait for Cilium to become ready

```bash
cilium status --wait
```

This command polls until all components are healthy. Expected final output:

```
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (Envoy not installed)
 \__/¯¯\__/    Hubble Relay:       OK
    \__/        ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             hubble-relay       Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-ui          Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium             Running: 3
                       cilium-operator    Running: 1
                       hubble-relay       Running: 1
                       hubble-ui          Running: 1
Cluster Pods:          6/6 managed by Cilium
Helm chart version:    1.15.5
Image versions         cilium             quay.io/cilium/cilium:v1.15.5@sha256:...
                       cilium-operator    quay.io/cilium/operator-generic:v1.15.5@sha256:...
                       hubble-relay       quay.io/cilium/hubble-relay:v1.15.5@sha256:...
                       hubble-ui          quay.io/cilium/hubble-ui:v0.13.1@sha256:...
```

All components show `OK` or `Running`. Now check nodes are Ready:

```bash
kubectl get nodes
```

```
NAME                        STATUS   ROLES           AGE   VERSION
cilium-lab-control-plane    Ready    control-plane   3m    v1.29.2
cilium-lab-worker           Ready    <none>          2m    v1.29.2
cilium-lab-worker2          Ready    <none>          2m    v1.29.2
```

### Step 5 — Deploy the trainer and gpu-worker pods

Create `pods.yaml` with both pods and a simple HTTP server so curl tests work:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: trainer
  namespace: default
  labels:
    role: trainer
    app: ai-workload
spec:
  containers:
  - name: trainer
    image: curlimages/curl:8.6.0
    # Keep the pod running and serve HTTP on port 8080
    command:
    - sh
    - -c
    - |
      # Start a minimal HTTP server on 8080
      while true; do
        echo -e "HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\ntrainer:alive" \
          | nc -l -p 8080 -q 1
      done
    ports:
    - containerPort: 8080
      name: http
---
apiVersion: v1
kind: Pod
metadata:
  name: gpu-worker
  namespace: default
  labels:
    role: gpu-worker
    app: ai-workload
spec:
  containers:
  - name: gpu-worker
    image: curlimages/curl:8.6.0
    command:
    - sh
    - -c
    - |
      while true; do
        echo -e "HTTP/1.1 200 OK\r\nContent-Length: 16\r\n\r\ngpu-worker:alive" \
          | nc -l -p 8080 -q 1
      done
    ports:
    - containerPort: 8080
      name: http
```

Apply and wait for pods to be Running:

```bash
kubectl apply -f pods.yaml
kubectl wait --for=condition=Ready pod/trainer pod/gpu-worker --timeout=60s
```

Expected output:

```
pod/trainer condition met
pod/gpu-worker condition met
```

Get the pod IPs:

```bash
kubectl get pods -o wide
```

```
NAME         READY   STATUS    RESTARTS   AGE   IP            NODE
gpu-worker   1/1     Running   0          45s   10.244.1.23   cilium-lab-worker
trainer      1/1     Running   0          45s   10.244.2.17   cilium-lab-worker2
```

### Step 6 — Test connectivity BEFORE applying any policy (both directions allowed)

Test trainer → gpu-worker:

```bash
GPUWORKER_IP=$(kubectl get pod gpu-worker -o jsonpath='{.status.podIP}')
kubectl exec trainer -- curl -s --max-time 5 http://${GPUWORKER_IP}:8080/
```

Expected output (connection succeeds):

```
gpu-worker:alive
```

Test gpu-worker → trainer:

```bash
TRAINER_IP=$(kubectl get pod trainer -o jsonpath='{.status.podIP}')
kubectl exec gpu-worker -- curl -s --max-time 5 http://${TRAINER_IP}:8080/
```

Expected output (connection succeeds in both directions before any policy):

```
trainer:alive
```

With no CiliumNetworkPolicy applied, Cilium allows all traffic by default (same as vanilla Kubernetes).

### Step 7 — Apply the CiliumNetworkPolicy

Create `policy.yaml` — allows trainer → gpu-worker on port 8080, implicitly denying everything else to gpu-worker:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-trainer-to-gpu-worker
  namespace: default
spec:
  # This policy applies to the gpu-worker endpoint
  endpointSelector:
    matchLabels:
      role: gpu-worker
  ingress:
  # Only allow ingress from pods with role=trainer
  - fromEndpoints:
    - matchLabels:
        role: trainer
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
```

Apply the policy:

```bash
kubectl apply -f policy.yaml
```

Expected output:

```
ciliumnetworkpolicy.cilium.io/allow-trainer-to-gpu-worker created
```

Verify the policy was accepted by Cilium:

```bash
kubectl get ciliumnetworkpolicy
```

```
NAME                          AGE
allow-trainer-to-gpu-worker   10s
```

Check the policy is programmed on the gpu-worker endpoint:

```bash
kubectl exec -n kube-system ds/cilium -- cilium endpoint list | grep gpu-worker
```

```
ENDPOINT   POLICY (ingress)   POLICY (egress)   IDENTITY   LABELS (source:key[=value])
1234       Enabled            Disabled          12345      k8s:role=gpu-worker
```

`POLICY (ingress): Enabled` confirms the ingress policy is active on this endpoint.

### Step 8 — Test connectivity AFTER applying policy

Test trainer → gpu-worker (should still be ALLOWED):

```bash
kubectl exec trainer -- curl -s --max-time 5 http://${GPUWORKER_IP}:8080/
```

Expected output (still succeeds — trainer is explicitly permitted):

```
gpu-worker:alive
```

Test gpu-worker → trainer (should be DROPPED — no policy permits this egress from gpu-worker's perspective, and trainer has no ingress policy but gpu-worker initiating is not in the allow rule):

```bash
kubectl exec gpu-worker -- curl -v --max-time 5 http://${TRAINER_IP}:8080/
```

Expected output (connection times out — BPF policy_map drops the packet silently):

```
*   Trying 10.244.2.17:8080...
* Connection timeout after 5001ms
* Closing connection 0
curl: (28) Connection timed out after 5001 milliseconds
command terminated with exit code 28
```

The connection hangs and eventually times out. Cilium's BPF program drops the SYN packet at the gpu-worker egress hook because there is no egress rule permitting it. Note: to make the block bidirectional and explicit for trainer as well, add a second CiliumNetworkPolicy targeting `role: trainer` with an egress rule. The current policy demonstrates that a policy on gpu-worker's ingress prevents gpu-worker from receiving unsolicited connections from itself as well as from other pods.

### Step 9 — Observe flows with Hubble

First, enable the Hubble port-forward so the hubble CLI can connect:

```bash
cilium hubble port-forward &
```

Expected output:

```
Forwarding from 127.0.0.1:4245 -> 4245
```

Now observe all flows in the default namespace:

```bash
hubble observe --namespace default --follow
```

While the observe command is running, in another terminal re-run the curl tests. You will see output like:

```
Apr 22 15:01:03.412  default/trainer:44312 -> default/gpu-worker:8080  TCP Flags: SYN
Apr 22 15:01:03.413  default/trainer:44312 -> default/gpu-worker:8080  TCP Flags: SYN-ACK
Apr 22 15:01:03.413  default/gpu-worker:8080 -> default/trainer:44312  HTTP/1.1 200 OK
Apr 22 15:01:03.414  default/trainer:44312 -> default/gpu-worker:8080  TCP Flags: FIN
Apr 22 15:01:08.700  default/gpu-worker:51023 -> default/trainer:8080  TCP Flags: SYN  DROPPED
Apr 22 15:01:13.702  default/gpu-worker:51023 -> default/trainer:8080  TCP Flags: SYN  DROPPED
Apr 22 15:01:18.704  default/gpu-worker:51023 -> default/trainer:8080  TCP Flags: SYN  DROPPED
```

Interpretation:
- `FORWARDED`: The BPF policy_map lookup succeeded — the packet's source identity is permitted by the destination's ingress policy. The `trainer → gpu-worker` flow is FORWARDED.
- `DROPPED`: The BPF policy_map lookup found no matching allow rule. The packet is silently dropped at the kernel hook — no TCP RST is sent, which is why `curl` sees a timeout rather than a connection refused. Three SYN retransmits are visible before curl gives up.

Filter to see only dropped flows:

```bash
hubble observe --namespace default --verdict DROPPED --follow
```

```
Apr 22 15:01:08.700  default/gpu-worker:51023 -> default/trainer:8080  TCP Flags: SYN  policy-denied  DROPPED
Apr 22 15:01:13.702  default/gpu-worker:51023 -> default/trainer:8080  TCP Flags: SYN  policy-denied  DROPPED
```

The `policy-denied` reason confirms this is a CiliumNetworkPolicy enforcement action, not a network connectivity failure.

### Step 10 — Port-forward the Hubble UI and interpret the service map

```bash
kubectl port-forward -n kube-system svc/hubble-ui 12000:80 &
```

Expected output:

```
Forwarding from 127.0.0.1:12000 -> 8081
```

Open a browser at `http://localhost:12000`. Select namespace `default` from the dropdown.

What to look for in the service map:

1. **Nodes**: Each pod appears as a node in the graph (`trainer` and `gpu-worker`). The node color indicates health — green for active, grey for idle.

2. **Edges — green (FORWARDED)**: A green directed arrow from `trainer` to `gpu-worker` on port 8080. Hovering shows flow count and bytes transferred. This represents the permitted policy path.

3. **Edges — red (DROPPED)**: A red directed arrow from `gpu-worker` toward `trainer`. Hovering shows the drop count and the reason `policy-denied`. This visually confirms the asymmetric policy enforcement.

4. **Service map filtering**: Use the top search bar to filter by label (`role=gpu-worker`) to isolate the gpu-worker's ingress flows. The map collapses to show only flows involving that pod.

5. **Flow timeline**: Click on an edge to open the flow detail panel on the right. Each individual flow record shows: timestamp, source IP:port, destination IP:port, protocol, verdict (FORWARDED/DROPPED), and the policy that matched (or denied) it.

The Hubble UI service map is the canonical tool for answering "why is pod A unable to reach pod B?" in a Cilium cluster — the DROPPED flows and their `policy-denied` reasons identify exactly which CiliumNetworkPolicy is enforcing the block.

### Step 11 — Clean up

```bash
# Stop port-forwards
kill %1 %2 2>/dev/null

# Delete the policy and pods
kubectl delete -f policy.yaml
kubectl delete -f pods.yaml

# Destroy the Kind cluster
kind delete cluster --name cilium-lab
```

---

## Summary

- Cilium replaces kube-proxy and iptables with eBPF programs attached per-endpoint, achieving O(1) Service lookup and enforcing policy without iptables rule traversal.
- Identity-based network policy (label-derived numeric identities, not IPs) is policy that follows pods regardless of IP address changes.
- Hubble provides per-flow visibility for every packet in the cluster — essential for debugging NCCL connectivity failures, policy violations, and performance anomalies.
- XDP acceleration and DSR enable Cilium to handle external load balancer traffic at near-line rate.
- Cilium BGP Control Plane integrates pod CIDR advertisement directly into the fabric, eliminating a separate BGP daemon.

---

## References

- Cilium documentation: docs.cilium.io
- Cilium GitHub: github.com/cilium/cilium
- Hubble: github.com/cilium/hubble
- Postfinance blog: *Cilium in production at PostFinance*, cilium.io/blog
- CNCF case studies: cncf.io/case-studies/?_sft_project=cilium


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).