# Chapter 13 — Multi-NIC GPU Pods: Multus, SR-IOV & Network Operator

**Part IV: Overlay & Kubernetes Networking** | ~20 pages

---

## Introduction

GPU training pods have a networking problem that standard **Kubernetes** cannot solve out of the box: they need two fundamentally different network interfaces simultaneously. The first is a conventional container network interface for **Kubernetes** control plane traffic — DNS lookups, service discovery, health checks, and monitoring. The second is a high-performance **RDMA** interface directly attached to a physical NIC's Virtual Function, providing the near-zero-overhead data path that **NCCL** collective operations require for gradient synchronization. Standard **Kubernetes** assigns exactly one network interface per pod, and that interface belongs to the primary CNI plugin.

**Multus** CNI (Container Network Interface) solves this by acting as a meta-plugin: it delegates to other CNI plugins in sequence, attaching multiple interfaces to a single pod. The primary CNI — typically **Cilium** or **Flannel** — handles `eth0` for management traffic, while a secondary CNI configuration backed by **SR-IOV** creates `net1`, `net2`, and so on for **RDMA** data-plane traffic. Each secondary interface is defined by a `NetworkAttachmentDefinition` custom resource, and pods request them via a single pod annotation.

**SR-IOV** (Single Root I/O Virtualization) is the PCIe hardware feature that makes this practical at scale. A single physical NIC (the Physical Function) can expose up to 127 isolated Virtual Functions, each appearing as an independent PCIe device with its own hardware queues, MAC address, and **RDMA** capability. When a pod is assigned an **SR-IOV** VF, it has a direct hardware path to the NIC's **RDMA** engine — bypassing the host kernel networking stack and enabling **GPUDirect RDMA** transfers directly from GPU memory to the wire.

This chapter covers the complete **Kubernetes** network stack for GPU pods: **Multus** CNI for multi-NIC attachment, **SR-IOV** VF provisioning, the **SR-IOV Network Operator** for cluster-scale automated VF management, the **NVIDIA Network Operator** as an all-in-one deployment bundle, **Whereabouts** for cluster-wide **IPAM (IP Address Management)** on secondary networks, and **GPUDirect RDMA** configuration. The lab walkthrough uses **Kind** with **macvlan** and Soft-**RoCE** to simulate the full stack without real **SR-IOV** hardware.

- **Macvlan**: a Linux networking driver that allows virtual interfaces (such as Docker containers or VMs) to be assigned unique MAC addresses, making them appear as physical devices directly connected to the network. It enables containers to have their own IP addresses on the same subnet as the host, bypassing the host bridge and improving performance
- **WhereAbouts**: An IP Address Management (**IPAM**) CNI plugin that assigns IP addresses cluster-wide. Provides a way to assign IP addresses dynamically across all the nodes of a cluster. Whereabouts can be used for both IPv4 & IPv6 addressing.

This chapter builds directly on Chapter 12 (**Cilium** as the primary CNI managing `eth0`) and Chapter 2 (**RoCEv2** and **RDMA** fundamentals). Chapter 19 examines how **NCCL** uses the **RDMA** VFs this chapter provisions to implement ring-allreduce and tree-allreduce collective algorithms across the GPU fabric.

---

## Installation

**kubectl**, **Helm**, and **Kind** are the standard **Kubernetes** management tools used to stand up the lab cluster and deploy the **SR-IOV Network Operator** via **Helm** chart. The **rdma-core** and **libibverbs** packages provide the Linux **RDMA** userspace stack, including the `ibv_devinfo` and `ib_write_bw` utilities needed to verify **RDMA** device presence and measure bandwidth inside pods. The `rdma_rxe` kernel module (Soft-**RoCE**) emulates an **RDMA** device over a standard Ethernet interface, making it possible to exercise the full **Multus** annotation, `NetworkAttachmentDefinition`, and **RDMA** verification workflow in a **Kind** cluster without physical **Mellanox** NICs. This combination allows the chapter's lab to demonstrate multi-NIC pod attachment and **RDMA** path validation on any development machine, with a clear mapping to what changes when real **SR-IOV** Virtual Functions are present.

### System Packages (Ubuntu 24.04)

```bash
# kubectl
sudo apt install -y kubectl rdma-core libibverbs-dev ibverbs-utils

# Helm 3
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Kind (Kubernetes-in-Docker) for local lab
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64 \
  && chmod +x kind \
  && sudo mv kind /usr/local/bin/

# Soft-RoCE kernel module (RDMA emulation — no real InfiniBand hardware needed)
sudo modprobe rdma_rxe
# Verify the module loaded
lsmod | grep rdma_rxe
```

### Python Environment (uv)

```bash
uv venv .venv && source .venv/bin/activate
uv pip install kubernetes pyyaml
```

---

## 13.1 The Multi-NIC Problem

A GPU training pod needs at least two network interfaces:
1. **Management/default CNI (eth0):** Kubernetes control plane, DNS, service mesh, monitoring — handled by Cilium or another primary CNI. CNI (Container Network Interface) is the Kubernetes plugin API that network providers implement to allocate IP addresses and configure interfaces when pods start.
2. **RDMA data plane (net1, net2, ...):** High-bandwidth GPU-to-GPU collective traffic — must use the physical RDMA NIC with near-zero overhead; cannot share with management traffic

Standard Kubernetes assigns exactly one network interface per pod. Multus CNI breaks this limitation.

---

## 13.2 Multus CNI — Meta-Plugin for Multiple Interfaces

Multus is a CNI meta-plugin: it delegates to other CNI plugins. The primary CNI (Cilium) handles `eth0`; additional CNI configurations attached via `NetworkAttachmentDefinition` handle `net1`, `net2`, etc.

### 13.2.1 Installation

- Multus is installed as a DaemonSet: 
- 
```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml

# Verify
kubectl get pods -n kube-system | grep multus
# multus-ds-xxxxx   1/1   Running
```

### 13.2.2 NetworkAttachmentDefinition

```yaml
# Define a secondary network backed by SR-IOV
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-rdma-net
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/resourceName: nvidia.com/mlnx_sriov_rdma
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "sriov-rdma-net",
      "type": "sriov",
      "deviceID": "auto",
      "vlan": 0,
      "spoofchk": "off",
      "trust": "on",
      "link_state": "enable",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.200.0/24",
        "exclude": ["192.168.200.0/32", "192.168.200.255/32"]
      }
    }
```

### 13.2.3 Pod Annotation

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-trainer
  annotations:
    # Request two RDMA secondary interfaces
    k8s.v1.cni.cncf.io/networks: |
      [
        {"name": "sriov-rdma-net", "interface": "net1"},
        {"name": "sriov-rdma-net", "interface": "net2"}
      ]
spec:
  containers:
  - name: trainer
    image: nvcr.io/nvidia/pytorch:24.01-py3
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/mlnx_sriov_rdma: 2   # request 2 VF resources
    env:
    - name: NCCL_IB_HCA
      value: "net1,net2"      # tell NCCL which interfaces to use
```

After pod creation, `ip addr show` inside the pod shows `eth0` (Cilium), `net1`, and `net2` (SR-IOV VFs).

---

## 13.3 SR-IOV — Single Root I/O Virtualization

SR-IOV allows a single PCIe NIC (Physical Function, PF) to present multiple isolated virtual devices (Virtual Functions, VFs) to the OS. Each VF appears as an independent NIC with its own PCIe BARs, queues, and MAC address.

### 13.3.1 SR-IOV Mechanism

```
Physical Function (PF):  mlx5_core, full NIC driver
  ├── VF 0:  mlx5_core VF, appears as eth0 in pod A
  ├── VF 1:  mlx5_core VF, appears as net1 in pod A
  ├── VF 2:  mlx5_core VF, appears as net1 in pod B
  └── ...
```

VFs share the NIC ASIC (hardware queues, RDMA engine) but have isolated data paths. The hypervisor/host controls bandwidth limits and MAC/VLAN enforcement on each VF.

### 13.3.2 Enabling SR-IOV on a Mellanox NIC

```bash
# Check PF name and current VF count
ip link show
# eth0: flags=UP,BROADCAST,RUNNING,MULTICAST  mtu 9000

# Enable VFs (max depends on NIC model; ConnectX-6 supports 127)
echo 8 > /sys/class/net/eth0/device/sriov_numvfs

# Verify VFs created
ip link show eth0
# 4: eth0: ...
#    vf 0     link/ether aa:bb:cc:dd:ee:01 brd ff:ff:ff:ff:ff:ff, spoof checking off
#    vf 1     link/ether aa:bb:cc:dd:ee:02 brd ff:ff:ff:ff:ff:ff, spoof checking off
#    ...

# Enable SR-IOV in the NIC firmware (persistent across reboots)
mlxconfig -d /dev/mst/mt4123_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=8
```

### 13.3.3 RDMA in VFs

Mellanox ConnectX VFs expose full RDMA capability — each VF has its own RDMA device visible to the pod:

```bash
# Inside a pod with SR-IOV VF:
ibv_devinfo
# hca_id: mlx5_0    ← the VF appears as an independent RDMA device
# transport: InfiniBand (0)
# fw_ver: 22.32.1010
# node_guid: ...

# Run RDMA bandwidth test between two pods on different nodes
# Pod A (receiver):
ib_write_bw -d mlx5_0 -x 3

# Pod B (sender):
ib_write_bw -d mlx5_0 -x 3 192.168.200.5
```

---

## 13.4 SR-IOV Network Operator

Managing SR-IOV configuration across 1000+ nodes manually is impractical. The SR-IOV Network Operator automates VF provisioning via Kubernetes CRDs.

### 13.4.1 Installation

```bash
helm repo add sriov-network-operator https://sriov.github.io/sriov-network-operator/
helm install sriov-network-operator sriov-network-operator/sriov-network-operator \
    --namespace sriov-network-operator \
    --create-namespace
```

### 13.4.2 SriovNetworkNodePolicy

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: mlnx-rdma-policy
  namespace: sriov-network-operator
spec:
  resourceName: mlnx_sriov_rdma
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"
  numVfs: 8
  nicSelector:
    vendor: "15b3"          # Mellanox
    deviceID: "101d"        # ConnectX-6 Dx
    pfNames: ["eth1"]
  deviceType: netdevice      # or vfio-pci for DPDK
  isRdma: true               # expose VF as RDMA device in pod
```

The operator reads this policy and:
1. Creates VFs on all matching nodes
2. Registers `nvidia.com/mlnx_sriov_rdma` as an extended resource via device plugin
3. The scheduler uses these resource limits to place pods on nodes with available VFs

### 13.4.3 Checking Resource Availability

```bash
# See allocatable SR-IOV resources per node
kubectl get node gpu-node-01 -o json | \
    jq '.status.allocatable | with_entries(select(.key | contains("sriov")))'
# {
#   "nvidia.com/mlnx_sriov_rdma": "8"
# }

# See current allocations
kubectl get sriovnetworknodestates -n sriov-network-operator -o yaml
```

---

## 13.5 NVIDIA Network Operator

The NVIDIA Network Operator bundles all network components for GPU nodes into a single Helm chart: MOFED driver (Mellanox **OFED** — the vendor-optimized InfiniBand and Ethernet driver stack that unlocks full RDMA performance on ConnectX NICs), SR-IOV operator, secondary CNI, RDMA device plugin, and NV-IPAM.

### 13.5.1 Installation

```yaml
# values.yaml for NVIDIA Network Operator
deployCR: true
nfd:
  enabled: true        # Node Feature Discovery (detects GPU/NIC)
sriovNetworkOperator:
  enabled: true
rdmaSharedDevicePlugin:
  enabled: true
  resources:
  - name: rdma_shared_device
    vendors: ["15b3"]     # Mellanox
nvIpam:
  enabled: true
ofedDriver:
  deploy: true
  image: mofed
  version: "24.01-0.3.3.1"
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"
```

```bash
helm install network-operator nvidia/network-operator \
    --namespace network-operator \
    --create-namespace \
    -f values.yaml
```

### 13.5.2 NV-IPAM — Cluster-Wide IP Allocation

NV-IPAM manages IP address pools for secondary networks, integrating with Whereabouts for cluster-wide allocation:

```yaml
apiVersion: nv-ipam.nvidia.com/v1alpha1
kind: IPPool
metadata:
  name: rdma-pool
  namespace: default
spec:
  subnet: 192.168.200.0/24
  prefix: 24
  gateway: 192.168.200.1
  perNodeBlockSize: 10    # each node gets a /28 from this pool
```

---

## 13.6 GPUDirect RDMA in Kubernetes

With SR-IOV VFs and `nvidia-peermem` loaded — `nvidia-peermem` is a kernel module that allows the Mellanox RDMA driver to map GPU memory pages directly, enabling GPUDirect RDMA without a CPU-mediated bounce buffer — GPU memory can be registered as RDMA memory regions directly:

```python
# Inside the container, using PyTorch + NCCL
import torch
import os

# NCCL picks up the RDMA VFs via NCCL_IB_HCA
os.environ["NCCL_IB_HCA"] = "mlx5_0,mlx5_1"
os.environ["NCCL_IB_GID_INDEX"] = "3"           # RoCEv2 GID
os.environ["NCCL_NET_GDR_LEVEL"] = "SYS"        # GPUDirect for all peers
os.environ["NCCL_DEBUG"] = "INFO"

# Initialize process group with NCCL backend
import torch.distributed as dist
dist.init_process_group(backend="nccl")

# Tensors in GPU memory are transferred via GPUDirect RDMA
tensor = torch.ones(1024 * 1024, device="cuda")
dist.all_reduce(tensor, op=dist.ReduceOp.SUM)
```

NCCL's `NCCL_NET_GDR_LEVEL=SYS` instructs NCCL to use GPUDirect RDMA for all peers (not just same-PCIe-switch peers), maximizing GPU memory bandwidth utilization.

---

## 13.7 Whereabouts — Cluster-Wide IPAM

Standard CNI IPAM plugins (host-local) allocate IPs per-node, causing conflicts when pods move. **Whereabouts** maintains a cluster-wide IP allocation database (backed by etcd or the K8s API) to prevent double-allocation:

```yaml
# NetworkAttachmentDefinition with Whereabouts IPAM
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "rdma-net",
      "type": "sriov",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.200.0/24",
        "exclude": ["192.168.200.0/32", "192.168.200.255/32"],
        "leader_lease_duration": 1500,
        "leader_renew_deadline": 1000,
        "leader_retry_period": 500
      }
    }
```

---

## Lab Walkthrough 13 — Multi-NIC GPU Pod with SR-IOV and NCCL

This walkthrough uses Kind (Kubernetes-in-Docker) with macvlan as a stand-in for SR-IOV, and Soft-RoCE (`rdma_rxe`) as the RDMA layer, since Kind clusters do not have real SR-IOV Virtual Functions. The final steps note what would differ on bare-metal hardware with actual VFs.

### Step 1 — Verify Prerequisites

```bash
# Confirm kind, kubectl, and helm are installed
kind version
# kind v0.22.0 go1.21.6 linux/amd64

kubectl version --client
# Client Version: v1.29.x

helm version
# version.BuildInfo{Version:"v3.x.x", ...}

# Confirm Soft-RoCE module is loaded
lsmod | grep rdma_rxe
# rdma_rxe              114688  0

# If not loaded:
sudo modprobe rdma_rxe
```

### Step 2 — Create a Kind Cluster

Save the following as `kind-config.yaml`:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: multus-lab
nodes:
  - role: control-plane
  - role: worker
    extraMounts:
      - hostPath: /dev
        containerPath: /dev
networking:
  disableDefaultCNI: true   # we will install our own CNI (Flannel + Multus)
```

```bash
kind create cluster --config kind-config.yaml
# Creating cluster "multus-lab" ...
#  - Ensuring node image (kindest/node:v1.29.2) ...
#  - Preparing nodes ...
#  - Writing configuration ...
#  - Starting control-plane ...
#  - Installing StorageClass ...
#  - Joining worker nodes ...
# Set kubectl context to "kind-multus-lab"
# You can now use your cluster with: kubectl cluster-info --context kind-multus-lab

kubectl cluster-info --context kind-multus-lab
# Kubernetes control plane is running at https://127.0.0.1:<port>
```

### Step 3 — Install Flannel as Primary CNI

Flannel is a lightweight Kubernetes CNI plugin that provides pod-to-pod IP routing using a simple overlay (VXLAN by default). It handles `eth0` assignment here; Multus will layer SR-IOV secondary interfaces on top.

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Wait for Flannel pods to be Running
kubectl wait --for=condition=Ready pod -l app=flannel -n kube-flannel --timeout=120s
# pod/kube-flannel-ds-xxxxx condition met
# pod/kube-flannel-ds-yyyyy condition met
```

### Step 4 — Install Multus CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml

# Wait for Multus daemonset to roll out
kubectl rollout status daemonset/kube-multus-ds -n kube-system --timeout=120s
# daemon set "kube-multus-ds" successfully rolled out

# Confirm Multus pods are Running on every node
kubectl get pods -n kube-system -l app=multus
# NAME                  READY   STATUS    RESTARTS   AGE
# kube-multus-ds-abcd   1/1     Running   0          45s
# kube-multus-ds-efgh   1/1     Running   0          45s
```

### Step 5 — Install Whereabouts IPAM

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/crds/whereabouts.cni.cncf.io_ippools.yaml
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/crds/whereabouts.cni.cncf.io_overlappingrangeipreservations.yaml
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/install/daemonset-install.yaml

kubectl rollout status daemonset/whereabouts -n kube-system --timeout=120s
# daemon set "whereabouts" successfully rolled out
```

Verify the IPPool CRD is registered:

```bash
kubectl get crd | grep whereabouts
# ippools.whereabouts.cni.cncf.io                       2026-04-22T00:00:00Z
# overlappingrangeipreservations.whereabouts.cni.cncf.io 2026-04-22T00:00:00Z
```

### Step 6 — Create the NetworkAttachmentDefinition (macvlan stand-in for SR-IOV)

In this lab we use `macvlan` mode because Kind worker nodes are Docker containers with a virtual `eth0`. `macvlan` is a Linux kernel driver that creates virtual NICs sharing a physical interface but with distinct MAC addresses — it produces a secondary interface functionally similar to an SR-IOV VF but without hardware isolation or RDMA capability. On real hardware you would replace `"type": "macvlan"` with `"type": "sriov"` and add a `deviceID` field.

Save as `nad-macvlan.yaml`:

```yaml
# nad-macvlan.yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: secondary-net
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "secondary-net",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.200.0/24",
        "exclude": [
          "192.168.200.0/32",
          "192.168.200.255/32"
        ]
      }
    }
```

```bash
kubectl apply -f nad-macvlan.yaml
# networkattachmentdefinition.k8s.cni.cncf.io/secondary-net created

kubectl get network-attachment-definitions
# NAME            AGE
# secondary-net   5s
```

### Step 7 — Deploy Two Pods with Multus Annotation

Save as `pods-multus.yaml`:

```yaml
# pods-multus.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: '[{"name":"secondary-net","interface":"net1"}]'
spec:
  containers:
  - name: nettools
    image: nicolaka/netshoot:latest
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: '[{"name":"secondary-net","interface":"net1"}]'
spec:
  containers:
  - name: nettools
    image: nicolaka/netshoot:latest
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f pods-multus.yaml

# Wait for both pods to be Running
kubectl wait --for=condition=Ready pod/pod-a pod/pod-b --timeout=120s
# pod/pod-a condition met
# pod/pod-b condition met

kubectl get pods -o wide
# NAME    READY   STATUS    RESTARTS   AGE   IP           NODE
# pod-a   1/1     Running   0          30s   10.244.1.x   multus-lab-worker
# pod-b   1/1     Running   0          30s   10.244.1.y   multus-lab-worker
```

### Step 8 — Verify Secondary Interfaces Inside Each Pod

```bash
# Check interfaces in pod-a
kubectl exec pod-a -- ip addr show
# 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
#     inet 127.0.0.1/8
# 2: eth0@if...: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 ...
#     inet 10.244.1.x/24 brd 10.244.1.255       ← primary CNI (Flannel)
# 3: net1@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
#     inet 192.168.200.1/24 brd 192.168.200.255  ← secondary (Multus/macvlan)

# Check interfaces in pod-b
kubectl exec pod-b -- ip addr show
# ...
# 3: net1@eth0: ...
#     inet 192.168.200.2/24 brd 192.168.200.255  ← different IP, same pool
```

Record the `net1` IP of pod-b (shown as `192.168.200.2` above) for the iperf3 test.

Also confirm Multus annotated the pod with the actual IPs assigned:

```bash
kubectl get pod pod-a -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}' | python3 -m json.tool
# [
#   {
#     "name": "flannel",
#     "interface": "eth0",
#     "ips": ["10.244.1.x"],
#     "default": true
#   },
#   {
#     "name": "default/secondary-net",
#     "interface": "net1",
#     "ips": ["192.168.200.1"],
#     "mac": "..."
#   }
# ]
```

### Step 9 — Run iperf3 Between Pods via the Secondary Interface

```bash
# Start iperf3 server in pod-b, listening on the net1 address
kubectl exec pod-b -- iperf3 -s -B 192.168.200.2 -p 5201 &

# Give the server a moment to start, then run the client from pod-a
kubectl exec pod-a -- iperf3 -c 192.168.200.2 -p 5201 -t 10
# Connecting to host 192.168.200.2, port 5201
# [  5] local 192.168.200.1 port 54321 connected to 192.168.200.2 port 5201
# [ ID] Interval           Transfer     Bitrate
# [  5]  0.00-1.00   sec  1.23 GBytes  10.6 Gbits/sec
# [  5]  1.00-2.00   sec  1.25 GBytes  10.7 Gbits/sec
# ...
# - - - - - - - - - - - - - - - - - - - -
# [ ID] Interval           Transfer     Bitrate
# [  5]  0.00-10.00  sec  12.4 GBytes  10.7 Gbits/sec     sender
# [  5]  0.00-10.00  sec  12.4 GBytes  10.7 Gbits/sec     receiver

# Stop the background server
kubectl exec pod-b -- pkill iperf3
```

Traffic flows over `net1` (192.168.200.0/24), completely separate from the Flannel management plane on `eth0`.

### Step 10 — Minimal Distributed PyTorch Hello (NCCL via Socket Fallback)

Kind has no GPU or real RDMA hardware, so NCCL falls back to socket transport. This step confirms the distributed runtime wiring is correct end-to-end.

Save as `nccl-job.yaml`:

```yaml
# nccl-job.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: train-script
  namespace: default
data:
  train.py: |
    import os
    import torch
    import torch.distributed as dist

    dist.init_process_group(backend="gloo")   # gloo: Facebook's distributed communication library using CPU and TCP/IP sockets — the fallback when no GPU or RDMA hardware is present
    rank  = dist.get_rank()
    world = dist.get_world_size()

    t = torch.ones(4) * (rank + 1)
    print(f"[rank {rank}] before all_reduce: {t.tolist()}")
    dist.all_reduce(t, op=dist.ReduceOp.SUM)
    print(f"[rank {rank}] after  all_reduce: {t.tolist()}")
    # Expected: [world*(world+1)/2, ...] — sum of ranks+1 across all workers

    dist.destroy_process_group()
---
apiVersion: v1
kind: Pod
metadata:
  name: nccl-rank0
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: '[{"name":"secondary-net","interface":"net1"}]'
spec:
  restartPolicy: Never
  volumes:
  - name: script
    configMap:
      name: train-script
  containers:
  - name: trainer
    image: python:3.11-slim
    command:
    - sh
    - -c
    - |
      pip install uv -q && uv pip install torch --index-url https://download.pytorch.org/whl/cpu -q
      python /scripts/train.py
    env:
    - name: MASTER_ADDR
      value: "192.168.200.1"     # net1 IP of rank-0 pod
    - name: MASTER_PORT
      value: "29500"
    - name: RANK
      value: "0"
    - name: WORLD_SIZE
      value: "2"
    volumeMounts:
    - name: script
      mountPath: /scripts
---
apiVersion: v1
kind: Pod
metadata:
  name: nccl-rank1
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: '[{"name":"secondary-net","interface":"net1"}]'
spec:
  restartPolicy: Never
  volumes:
  - name: script
    configMap:
      name: train-script
  containers:
  - name: trainer
    image: python:3.11-slim
    command:
    - sh
    - -c
    - |
      pip install uv -q && uv pip install torch --index-url https://download.pytorch.org/whl/cpu -q
      python /scripts/train.py
    env:
    - name: MASTER_ADDR
      value: "192.168.200.1"     # connect to rank-0 over net1
    - name: MASTER_PORT
      value: "29500"
    - name: RANK
      value: "1"
    - name: WORLD_SIZE
      value: "2"
    volumeMounts:
    - name: script
      mountPath: /scripts
```

```bash
kubectl apply -f nccl-job.yaml

# Wait for both pods to complete
kubectl wait --for=condition=Succeeded pod/nccl-rank0 pod/nccl-rank1 --timeout=300s

# Check output from rank 0
kubectl logs nccl-rank0
# [rank 0] before all_reduce: [1.0, 1.0, 1.0, 1.0]
# [rank 0] after  all_reduce: [3.0, 3.0, 3.0, 3.0]   ← 1+2=3, correct

# Check output from rank 1
kubectl logs nccl-rank1
# [rank 1] before all_reduce: [2.0, 2.0, 2.0, 2.0]
# [rank 1] after  all_reduce: [3.0, 3.0, 3.0, 3.0]   ← 1+2=3, correct
```

### Step 11 — Delta: What Changes with Real SR-IOV VFs

On bare-metal hardware with Mellanox ConnectX-6 (or similar) and the SR-IOV Network Operator deployed, the only differences from this lab are:

| Lab (Kind + macvlan) | Production (bare-metal + SR-IOV) |
|---|---|
| CNI type: `macvlan` | CNI type: `sriov` + `deviceID` field |
| No resource limits | `resources.limits: nvidia.com/mlnx_sriov_rdma: 1` |
| NCCL backend: `gloo` (CPU socket) | NCCL backend: `nccl` with `NCCL_IB_HCA=net1` |
| No `ibv_devinfo` output | `ibv_devinfo` shows `mlx5_0` per pod |
| iperf3 bandwidth: ~10 Gbps (container network) | iperf3 bandwidth: 100–400 Gbps (RDMA) |
| Soft-RoCE (`rdma_rxe`) | Hardware RDMA via VF (`mlx5_core`) |

To verify real RDMA inside a pod with SR-IOV VFs:

```bash
# Inside pod with real SR-IOV VF:
ibv_devinfo
# hca_id: mlx5_0
# transport: InfiniBand (0)
# fw_ver: 22.32.1010

# RDMA bandwidth test (receiver on pod-b):
ib_write_bw -d mlx5_0 -x 3 --report_gbits

# RDMA bandwidth test (sender on pod-a, targeting pod-b net1 IP):
ib_write_bw -d mlx5_0 -x 3 --report_gbits 192.168.200.2
# ---------------------------------------------------------------------------------------
# #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
# 65536      5000           199.44             199.30               0.380
# ---------------------------------------------------------------------------------------
```

### Step 12 — Cleanup

```bash
kubectl delete -f nccl-job.yaml
kubectl delete -f pods-multus.yaml
kubectl delete -f nad-macvlan.yaml
kind delete cluster --name multus-lab
```

---

## Summary

- Multus CNI delegates to multiple CNI plugins per pod, enabling management (Cilium on eth0) and RDMA data plane (SR-IOV on net1) to coexist in the same pod.
- SR-IOV VFs give each pod a hardware-isolated NIC context with full RDMA capability, at near-zero overhead vs dedicated physical NICs.
- SR-IOV Network Operator automates VF provisioning across the cluster via CRDs, eliminating per-node manual configuration.
- NVIDIA Network Operator bundles all necessary components (MOFED, SR-IOV, RDMA device plugin, NV-IPAM) for GPU cluster deployments.
- GPUDirect RDMA in Kubernetes requires `nvidia-peermem`, SR-IOV VFs, and proper NCCL environment variables to achieve GPU-memory-to-fabric transfers without CPU involvement.

---

## References

- [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni)
- [SR-IOV Network Operator](https://github.com/k8snetworkplumbingwg/sriov-network-operator)
- [SR-IOV Network Device Plugin](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin)
- [Whereabouts IPAM](https://github.com/k8snetworkplumbingwg/whereabouts)
- [Network Plumbing Working Group — multi-net spec](https://github.com/k8snetworkplumbingwg/multi-net-spec)
- [NVIDIA Network Operator](https://docs.nvidia.com/networking/display/cokan)
- [NCCL documentation](https://docs.nvidia.com/deeplearning/nccl)
- [Kubernetes](https://kubernetes.io/docs/)
- [Kind (Kubernetes in Docker)](https://kind.sigs.k8s.io)
- [Helm](https://helm.sh)
- [Flannel CNI](https://github.com/flannel-io/flannel)
- [PyTorch](https://pytorch.org)
- [gloo (distributed communication library)](https://github.com/facebookincubator/gloo)
- [iperf3](https://iperf.fr)
- [nicolaka/netshoot](https://github.com/nicolaka/netshoot)
- [Mellanox OFED (MOFED)](https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).