# Chapter 29 — Kubernetes Scheduling for AI Workloads

**Part IV: Overlay & Kubernetes Networking** | ~20 pages

---

## Introduction

**Kubernetes** was built for stateless microservices that can be scheduled independently, spread for availability, and bin-packed by CPU and memory. Distributed AI training jobs violate every one of those assumptions. They require all ranks to be running simultaneously before any can make progress (gang scheduling), they are topology-sensitive to the point where a single rack boundary can add 40% overhead to pipeline-parallel training (topology awareness), and their resource requirements — typically 8 GPUs per pod, hundreds of pods per job — make fragmentation a default state rather than an edge case. This chapter explains what breaks and how to fix it.

The chapter opens with the failure modes (§29.1) and the core concept of gang scheduling (§29.2), establishing the vocabulary of `PodGroup`, `minMember`, starvation, and deadlock. From there it covers three **Kubernetes**-native scheduling systems: **Volcano** (§29.3), the dominant production gang scheduler that introduces `Queue`, `PodGroup`, and `Job` CRDs along with **DRF**-based fairness and preemption; **Kueue** (§29.4), which acts as an admission controller above the default scheduler, managing quota without replacing it; and **Coscheduler** (§29.5), which implements gang scheduling as a `kube-scheduler` plugin for environments that want gang guarantees without deploying a separate scheduler binary.

Sections 29.6 and 29.7 move from scheduling policy to physical consequences. Topology-aware scheduling using `topologySpreadConstraints` and **NUMA** affinity keeps AllReduce traffic within a rack, eliminating spine traversal that can add 3µs per hop to collective latency. The quantitative analysis in §29.7 shows that for high-frequency pipeline-parallel communications with small tensors, cross-rack placement can increase collective overhead by 40–50%. Section 29.8 builds out the operational policy layer: priority classes, preemption order, and multi-team quota management.

Section 29.9 extends beyond pure **Kubernetes** to the three additional scheduling systems that dominate real AI infrastructure: **SLURM** for HPC-origin **MPI** workloads where topology-aware allocation is expressed through `topology.conf` and `--switches`; **RunAI** for GPU utilization governance with its Projects and Departments quota model; and **NVIDIA Dynamo** for disaggregated prefill-decode inference serving at scale, where the performance-critical path is the KV cache **RDMA** transfer between prefill and decode workers.

This chapter connects directly to Chapter 2 (**RoCEv2** fabric bandwidth that makes rank placement performance-critical), Chapter 12 (**Cilium** CNI that provides pod networking within which **NCCL** operates), Chapter 13 (**SR-IOV** and the NVIDIA GPU device plugin that exposes GPUs to pods), and Chapter 27 (adaptive routing that determines how cross-rack traffic is handled when topology-constrained placement is not possible).

## 29.1 Why Standard Kubernetes Scheduling Breaks for AI

The default Kubernetes scheduler (`kube-scheduler`) was designed for stateless microservices: schedule each pod independently, bin-pack by CPU and memory, and spread for availability. This model fails for distributed AI training jobs in three distinct ways.

### Gang Scheduling Requirement

A distributed training job with N ranks cannot make progress unless all N ranks are running simultaneously. If only 7 of 8 ranks start, the 7 running ranks call `init_process_group()` and block at the rendezvous barrier indefinitely, consuming GPU memory and CPU cycles while achieving zero useful work. The eighth rank may be Pending because its node ran out of memory, or because the default scheduler decided to defer it for bin-packing efficiency.

Gang scheduling solves this with an **all-or-nothing** guarantee: either all N pods in a group are schedulable simultaneously, or none are scheduled at all. Pods wait in a coordinated Pending state rather than being partially launched.

### Topology Blindness

The default scheduler knows about node labels, taints, and `topologySpreadConstraints`, but it has no built-in model of:
- Which nodes share an NVLink domain (GPU topology)
- Which nodes are under the same ToR switch (rack membership)
- Which NICs are closest to which GPUs (RDMA affinity)

A 4-rank job scheduled across 4 racks will produce exactly the same performance as one scheduled on 4 nodes in the same rack — from the scheduler's perspective. In reality, the cross-rack job may be 2–5x slower due to spine switch traversal latency and bandwidth competition.

### Resource Fragmentation

Without coordination, multiple independent schedulers (or the same scheduler across concurrent jobs) can fragment cluster resources such that no single job can be fully scheduled even though the total available resources would fit it. Consider two 4-GPU jobs running on a cluster with 8 GPUs (4 per node): if node A has 2 GPUs allocated and node B has 2 GPUs allocated, neither job can fit even though 4 total GPUs are free.

Gang scheduling with a global admission controller prevents fragmentation by treating each job's resource requirement atomically.

---

## Installation

The lab runs entirely on a local multi-node **Kubernetes** cluster created by **Kind**, which requires only **Docker** and the `kind` binary — no cloud provider access is needed. `kubectl` and **Helm** are prerequisites for every subsequent installation step: **Helm** deploys both the **Volcano** gang scheduler (which installs the `vcctl` CLI, the `vc-scheduler` binary, and the `PodGroup` and `Queue` CRDs) and the **Kueue** admission controller (which adds `ClusterQueue`, `LocalQueue`, and `ResourceFlavor` CRDs). **Coscheduler** is deployed as a second `kube-scheduler` plugin via its own **Helm** chart and operates alongside the default scheduler, requiring no removal of existing scheduler infrastructure. GPU resource sharing and quota management across all three systems depend on the **NVIDIA GPU device plugin**, which is deployed separately and is covered in Chapter 13; this chapter assumes it is already running in the cluster.

### System packages

```bash
sudo apt install -y kubectl curl git
kubectl version --client
# Client Version: v1.30.x
```

### Kind — local multi-node cluster

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/
kind version
# kind v0.23.0 go1.22.0 linux/amd64
```

### Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
# version.BuildInfo{Version:"v3.15.x", ...}
```

### vcctl — Volcano CLI

```bash
VOLCANO_VERSION=v1.9.0
curl -Lo vcctl \
    "https://github.com/volcano-sh/volcano/releases/download/${VOLCANO_VERSION}/vcctl-linux-amd64"
chmod +x vcctl
sudo mv vcctl /usr/local/bin/
vcctl version
# Client Version: v1.9.0
```

### Python environment

```bash
uv venv .venv && source .venv/bin/activate
uv pip install kubernetes pyyaml
python -c "import kubernetes, yaml; print('OK')"
```

---

## 29.2 Gang Scheduling Concepts

### PodGroup and min-member

A `PodGroup` is a custom resource that groups pods and declares the minimum member count required before any pod in the group may be scheduled. The scheduler holds all pods in the group in a `WaitForPodScheduled` state until enough nodes are available to satisfy `minMember` simultaneously.

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: training-job-pg
  namespace: default
spec:
  minMember: 8           # all 8 ranks must be schedulable simultaneously
  queue: training        # which Volcano queue to draw resources from
  priorityClassName: high-priority
  minResources:
    cpu: "64"
    memory: "256Gi"
    nvidia.com/gpu: "8"
```

### All-or-Nothing Semantics

The semantic guarantee is precise: the scheduler will not bind any pod in the group to a node until it has found valid node assignments for all `minMember` pods simultaneously. If the cluster cannot satisfy the group at a given moment, the group remains fully Pending and no resources are consumed.

### Starvation and Deadlock Risks

Gang scheduling introduces two failure modes that don't exist with per-pod scheduling:

**Starvation**: A large job (e.g., 128-GPU) may be permanently blocked if the cluster is perpetually occupied by smaller jobs that each satisfy their own gang. Without preemption or quota management, the large job starves indefinitely.

**Deadlock**: If two gangs each hold some resources and are each waiting for additional resources that the other gang holds, both gangs deadlock. Volcano addresses this with a global queue and priority-based preemption; the higher-priority job preempts the lower-priority one.

---

## 29.3 Volcano Scheduler

Volcano is the dominant production gang scheduler for Kubernetes AI workloads. It introduces three primary CRDs: `Queue`, `PodGroup`, and `Job`.

### Queue

A Queue is a scheduling domain with guaranteed and maximum resource limits:

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: training
spec:
  weight: 10            # relative scheduling weight across queues
  capability:
    cpu: "128"
    memory: "512Gi"
    nvidia.com/gpu: "16"
  reclaimable: true     # this queue can have resources reclaimed by higher-priority queues
```

### Volcano Job

The `vcjob` CRD (kind: `Job`) expresses a complete distributed training job with multiple task types (e.g., parameter servers and workers), retry policy, and PodGroup binding:

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: pytorch-training
spec:
  minAvailable: 8
  schedulerName: volcano
  queue: training
  plugins:
    env: []
    svc: []
  policies:
  - event: PodEvicted
    action: RestartJob
  tasks:
  - replicas: 8
    name: worker
    policies:
    - event: TaskCompleted
      action: CompleteJob
    template:
      spec:
        containers:
        - name: pytorch
          image: pytorch/pytorch:2.3.0-cuda12.1-cudnn8-devel
          command: ["torchrun", "--nproc-per-node=1", "train.py"]
          resources:
            limits:
              nvidia.com/gpu: "1"
              cpu: "8"
              memory: "32Gi"
```

### vcctl CLI

```bash
# List queues
vcctl queue list
```

```
Name       Weight    Pending    Running    Unknown    Inqueue    Binding    Allocated  Succeeded  Failed     Preemptable
default    1         0          0          0          0          0          0          0          0          false
training   10        0          1          0          0          0          8          0          0          false
```

```bash
# List jobs
vcctl job list -n default
```

```
Name               Creation         Phase       JobType    Replicas    Min         Pending    Running    Succeeded  Failed     Unknown    RetryCount
pytorch-training   2026-04-22...    Running     <nil>       8           8           0           8          0          0          0          0
```

### Bin-packing vs Spread Policies

Volcano supports multiple scheduling plugins. The two most relevant for AI are:

**Binpack**: Fills nodes as densely as possible, keeping some nodes completely free for large gang jobs. Use for training clusters where you want to preserve large contiguous GPU allocations.

**DRF (Dominant Resource Fairness)**: A multi-resource fair-sharing algorithm that identifies each job's "dominant resource" (the resource — CPU, GPU, memory — that it consumes most relative to cluster capacity) and allocates across jobs to equalize dominant-resource shares, preventing any single queue from monopolizing the cluster. Use in multi-tenant training clusters.

Configure in the Volcano scheduler config:

```yaml
# volcano-scheduler-config.yaml
actions: "enqueue, allocate, backfill, preempt"
tiers:
- plugins:
  - name: priority
  - name: gang
  - name: conformance
- plugins:
  - name: drf
  - name: predicates
  - name: proportion
  - name: nodeorder
  - name: binpack
    arguments:
      binpack.weight: 10
      binpack.cpu: 5
      binpack.memory: 1
      binpack.resources: nvidia.com/gpu
      binpack.resources.nvidia.com/gpu: 10
```

---

## 29.4 Kueue — Workload Queueing and Admission

Kueue takes a different approach from Volcano: rather than replacing `kube-scheduler`, it acts as an admission controller that decides *when* to allow a job to enter the scheduling pipeline. The actual pod scheduling is still done by `kube-scheduler`. This makes Kueue composable with any scheduler plugin.

### Core CRDs

**ResourceFlavor** maps to physical node characteristics. A flavor named `gpu-a100-rack1` can select nodes with `rack=rack-1` and `accelerator=a100`:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: gpu-a100-rack1
spec:
  nodeLabels:
    accelerator: a100
    rack: rack-1
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Exists"
    effect: "NoSchedule"
```

**ClusterQueue** defines the cluster-level resource budget across flavors:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cluster-queue
spec:
  namespaceSelector: {}    # admit from all namespaces
  queueingStrategy: BestEffortFIFO
  resourceGroups:
  - coveredResources: ["cpu", "memory", "nvidia.com/gpu"]
    flavors:
    - name: gpu-a100-rack1
      resources:
      - name: cpu
        nominalQuota: "128"
        borrowingLimit: "32"
      - name: memory
        nominalQuota: "512Gi"
      - name: nvidia.com/gpu
        nominalQuota: "16"
```

**LocalQueue** is a namespace-scoped handle that teams use to submit workloads:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: team-a-queue
  namespace: team-a
spec:
  clusterQueueName: cluster-queue
```

### Admission Flow

When a Job is created in namespace `team-a` with label `kueue.x-k8s.io/queue-name: team-a-queue`:
1. Kueue's admission controller suspends the Job (sets `spec.suspend=true`).
2. Kueue creates a `Workload` object representing the resource request.
3. The ClusterQueue evaluates whether the request fits within quota.
4. If quota is available, Kueue unsuspends the Job and the Job pods enter the scheduling pipeline.
5. If quota is exhausted, the Workload is queued and admitted when resources are released.

---

## 29.5 Coscheduler — Gang Scheduling as a Scheduler Plugin

The `scheduler-plugins` project (sig-scheduling) implements gang scheduling as an in-tree plugin for `kube-scheduler` rather than a replacement scheduler. This means all other scheduling features — affinity, taints, topology spread — continue to work transparently.

### Architecture Difference from Volcano

| | Volcano | Coscheduler |
|---|---|---|
| Deployment | Replacement scheduler | Plugin for kube-scheduler |
| CRD | `Job`, `PodGroup`, `Queue` | `PodGroup` only |
| Additional features | Queue management, DRF, backfill | Gang guarantee only |
| Integration | Separate deployment, separate API | Same scheduler, same node selectors |

Coscheduler is appropriate when you want gang scheduling for a specific workload type while keeping the default scheduler for everything else.

### PodGroup with Coscheduler

```yaml
apiVersion: scheduling.sigs.k8s.io/v1alpha1
kind: PodGroup
metadata:
  name: nccl-job-pg
spec:
  scheduleTimeoutSeconds: 60    # fail if not schedulable within 60s
  minMember: 4
```

Pods reference the group via annotation:

```yaml
metadata:
  annotations:
    scheduling.sigs.k8s.io/pod-group: "nccl-job-pg"
```

---

## 29.6 Topology-Aware Scheduling

### NUMA Awareness

The Linux kubelet's CPU Manager and Memory Manager enforce NUMA locality at the node level. For GPU training pods, the relevant NUMA domain is the one containing both the GPU and the NIC (to minimize PCIe hop count for GPUDirect RDMA):

```yaml
# node-level kubelet config
cpuManagerPolicy: static
memoryManagerPolicy: Static
topologyManagerPolicy: single-numa-node   # require GPU + NIC on same NUMA node
```

`topologyManagerPolicy: single-numa-node` will reject pods that cannot fit in a single NUMA domain. This is the correct setting for GPUDirect workloads: a GPU-to-NIC transfer that crosses a NUMA boundary adds one PCIe root-complex traversal to every RDMA DMA operation.

### GPU Topology — NVLink and PCIe

NVIDIA GPU topology within a node affects collective communication bandwidth:
- GPUs on the same NVLink domain can exchange data at 600+ GB/s (NVLink 4.0)
- GPUs on the same PCIe root complex (but no NVLink) exchange at ~64 GB/s
- GPUs on different NUMA nodes exchange at ~32 GB/s (PCIe cross-NUMA)

The NVIDIA device plugin annotates nodes with GPU topology info that can be queried via the node's `allocatable` field and `nvidia.com/gpu-topology` annotation.

### NIC Affinity for RDMA

In a multi-NIC node, each RDMA NIC has a closest GPU determined by PCIe topology. NCCL reads `/proc/driver/nvidia/gpus/*/information` and `/sys/bus/pci/devices/*/numa_node` to build its topology graph. `NCCL_TOPO_FILE` can override this with a custom XML topology:

```xml
<!-- nccl-topo.xml for a DGX A100 node -->
<system version="1">
  <cpu numaid="0" affinity="0x00000000ff" arch="x86_64">
    <pci busid="0000:00:00.0" class="0x060000" vendor="0x8086">
      <pci busid="0000:3b:00.0" class="0x030200" vendor="0x10de">
        <!-- GPU 0 -->
        <gpu rank="0" gputype="Ampere"/>
      </pci>
      <pci busid="0000:3c:00.0" class="0x020700" vendor="0x15b3">
        <!-- NIC 0 closest to GPU 0 -->
        <nic>
          <net name="mlx5_0" dev="0" speed="200000" port="1" guid="0xe41d2d0300650000"/>
        </nic>
      </pci>
    </pci>
  </cpu>
</system>
```

### topologySpreadConstraints

`topologySpreadConstraints` can enforce rack-level co-location. The constraint below requires all pods of a training job to land within a single rack (minimizing spine traversal):

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 0               # all pods must be on the same rack (zero skew across racks)
    topologyKey: rack        # node label that defines topology domains
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        job: pytorch-training
```

`maxSkew: 0` is strict co-location: the skew between the most-populated and least-populated topology domain must be 0. Combined with a `nodeAffinity` targeting a specific rack, this pins the entire job to one rack.

---

## 29.7 Network Implications of Scheduling Decisions

### How Pod Placement Determines AllReduce Ring Topology

NCCL builds an AllReduce ring (or tree) by ordering ranks according to the physical network topology. If ranks 0–3 are on the same node and ranks 4–7 are on the same node but different from ranks 0–3, NCCL will construct intra-node rings using NVLink (fast) and inter-node rings using RoCE/IB (slower). The ring order is:

```
Node A:  rank0 → rank1 → rank2 → rank3 → (RoCE link) → Node B: rank4 → rank5 → rank6 → rank7 → (RoCE link) → Node A
```

This minimizes the number of NIC-level transfers (only 2 per AllReduce step, one in each direction) and maximizes NVLink utilization within each node.

If instead ranks are scattered one per node across 8 nodes with no NVLink, every step of the ring requires a RoCE transfer. The ring becomes:

```
Node A: rank0 → (RoCE) → Node B: rank1 → (RoCE) → Node C: rank2 → ...
```

This pattern uses 8 NIC transfers per ring step instead of 2, and may cross the spine switch if the nodes are on different racks.

### Traffic Crosses Spine vs Not

For a rail-optimized fabric (where each ToR has 1 GPU NIC uplink to a dedicated rail spine):
- **Same rack, same ToR**: AllReduce traffic stays within the ToR switch — no spine traversal. Typical RTT: 1–2µs.
- **Different racks, same rail**: Traffic passes through two ToR switches and one spine switch — one additional hop. Typical RTT: 3–5µs.
- **Different racks, different rails**: Traffic passes through two ToR switches, crosses the spine tier, and enters a different rail — two additional hops. Typical RTT: 5–10µs.

For a 256-rank AllReduce with 10 GB tensors, crossing an extra spine hop adds:

```
Extra latency per step  ≈ 2 × (extra_RTT) × ring_steps
Ring steps (allreduce)  = 2 × (N-1) ≈ 510 steps for N=256
Extra latency per step  ≈ 2 × 3µs × 510 = 3.06ms per AllReduce
Steps per training hour ≈ 1000
Extra training time/hr  ≈ 3.06ms × 1000 = 3 seconds/hour
```

While 3 seconds/hour sounds small, at 1000-GPU scale, persistent cross-spine placement can accumulate to minutes per day in slower steps — directly impacting experiment iteration velocity.

### Rank-to-NIC Mapping

NCCL's built-in topology detection assigns each rank to its closest NIC based on PCIe affinity. When pods are scheduled without NIC affinity, ranks may end up pinned to the "wrong" NIC (one on the far side of the PCIe root complex). `NCCL_TOPO_FILE` overrides autodetection and allows the infrastructure team to express the correct rank-to-NIC mapping for the target node type.

---

## 29.8 Practical Scheduling Policy

### Priority Classes for Training vs Inference

```yaml
# High priority for production training jobs
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: training-critical
value: 1000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Production LLM training jobs — may preempt batch inference"

---
# Medium priority for batch inference
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: inference-batch
value: 500
globalDefault: false
preemptionPolicy: PreemptLowerPriority

---
# Low priority for experiment/development
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: experiment
value: 100
globalDefault: true
preemptionPolicy: Never   # experiments do not preempt anything
```

### Preemption Order

With Volcano's preemption plugin enabled, the scheduler will evict lower-priority jobs to satisfy higher-priority gang scheduling requests. The eviction order is:
1. Pods in queues with lower `weight`
2. Within a queue, pods with lower `priorityClassName`
3. Within a priority class, pods with the oldest creation timestamp (LRU)

Graceful preemption via `SIGTERM` allows training jobs to save a checkpoint before termination. Set `terminationGracePeriodSeconds` appropriately:

```yaml
spec:
  terminationGracePeriodSeconds: 300   # 5 minutes to save checkpoint on SIGTERM
```

### Quota Management Across Teams

A multi-team cluster should configure per-team `LocalQueue` objects feeding a shared `ClusterQueue` with per-team quota. Borrowing allows a team to use another team's unused quota, with the quota returned when the owning team submits a job:

```yaml
# team-a can use up to 8 GPUs, borrow up to 4 from the shared pool
- name: gpu-a100-rack1
  resources:
  - name: nvidia.com/gpu
    nominalQuota: "8"
    borrowingLimit: "4"     # can borrow up to 4 extra
    lendingLimit: "4"       # can lend up to 4 to others
```

---

## 29.9 Beyond Pure Kubernetes: SLURM, RunAI, and NVIDIA Dynamo

Kubernetes-native schedulers (Volcano, Kueue, Coscheduler) excel at multi-tenant cluster sharing. But three additional systems occupy important niches in real AI infrastructure: SLURM for HPC-origin workloads, RunAI for GPU utilization governance, and NVIDIA Dynamo for inference serving at scale.

---

### 29.9.1 SLURM — HPC Workload Manager

**SLURM** (Simple Linux Utility for Resource Management) is the dominant scheduler in supercomputing. Virtually every HPC center, and many hyperscaler AI training clusters, runs SLURM. Understanding it is mandatory for engineers who need to co-locate Kubernetes and HPC workloads or migrate MPI/PyTorch jobs between environments.

**Architecture**

SLURM uses a central `slurmctld` daemon that holds cluster state and a per-node `slurmd` daemon that executes jobs. Jobs are submitted to a partition (analogous to a Kubernetes node pool) and held in a priority queue.

```
slurmctld  ─────────── slurmd (node-001)
    │                  slurmd (node-002)
    └── slurmdbd       slurmd (node-003)   ...
         (job accounting DB)
```

**GPU allocation with GRES**

SLURM models GPUs as Generic Resources (GRES). The `gres.conf` file on each node declares the hardware; job scripts request it with `--gres`:

```bash
# /etc/slurm/gres.conf (on each GPU node)
# Name=gpu Type=a100 File=/dev/nvidia[0-7] COREs=0-63

# Submit an 8-GPU job across 4 nodes (32 GPUs total)
sbatch --nodes=4 --ntasks-per-node=8 --gres=gpu:a100:8 train.sh

# Or interactively:
srun --nodes=2 --ntasks-per-node=4 --gres=gpu:4 \
     --pty bash -l
```

**Topology-aware scheduling**

SLURM's `topology.conf` describes the switch tree. With `--switches=2` SLURM constrains the allocation to nodes reachable through at most 2 switch hops — essential for rail-optimized fabrics:

```ini
# /etc/slurm/topology.conf
SwitchName=spine1 Switches=leaf[1-4]
SwitchName=spine2 Switches=leaf[5-8]
SwitchName=leaf1  Nodes=gpu[001-016]
SwitchName=leaf2  Nodes=gpu[017-032]
```

```bash
# Request all nodes under the same leaf switch
sbatch --nodes=16 --switches=1 train.sh
```

**PMIx integration (cross-reference Ch. 4)**

SLURM uses PMIx (Process Management Interface for Exascale) — a standardized API for bootstrapping and coordinating MPI and collective communication libraries across cluster nodes — to bootstrap MPI and collective communication libraries. When `srun` launches a PyTorch DDP job, SLURM sets `RANK`, `WORLD_SIZE`, `MASTER_ADDR`, and `MASTER_PORT` via PMIx; the training script reads them without any cluster-specific code. See §4.3 for the full PMIx wire protocol.

**Container jobs: Pyxis + Enroot**

NVIDIA's **Pyxis** SLURM plugin and **Enroot** container runtime allow Docker/OCI images to run as SLURM jobs without a daemon:

```bash
# Pull and squash to an Enroot image
enroot import docker://nvcr.io/nvidia/pytorch:24.02-py3
enroot create --name pytorch_24.02 \
    nvcr+io+nvidia+pytorch+24.02-py3.sqsh

# SLURM batch script using Pyxis
#!/bin/bash
#SBATCH --nodes=4
#SBATCH --gres=gpu:8
#SBATCH --container-image=./pytorch_24.02.sqsh
#SBATCH --container-mounts=/data:/data

srun python3 train.py --nnodes=4 --nproc_per_node=8
```

**SLURM vs Kubernetes scheduling — decision matrix**

| Criterion | SLURM | Kubernetes + Volcano/Kueue |
|-----------|-------|---------------------------|
| MPI / bare-metal | Native | Requires MPIOperator |
| Container orchestration | Via Pyxis (add-on) | First-class |
| Multi-tenant quota | Partition + QOS | ClusterQueue borrowing |
| Topology-aware allocation | `topology.conf` + `--switches` | topologySpreadConstraints |
| Heterogeneous workloads (ML + data + serving) | Limited | Strong |
| Time-sharing / preemption | Preempt + requeue | Preemption via priority class |
| Community in HPC | Dominant | Rare |

---

### 29.9.2 RunAI — GPU Scheduling Platform

**RunAI** is a Kubernetes-native GPU scheduling and orchestration platform. It extends kube-scheduler with a proprietary bin-packing and fairness algorithm, adds a governance layer (Projects and Departments), and provides a workload management UI tuned for ML teams.

**Architecture**

RunAI installs as a set of Kubernetes controllers and a custom scheduler. The custom scheduler replaces (or coexists with) kube-scheduler for GPU workloads:

```
RunAI Scheduler
    │
    ├── Workload CRDs (Run, TrainingRun, InteractiveRun)
    ├── Project CRDs (per-team GPU quota)
    └── Department CRDs (per-org quota tree)
```

**Projects and Departments (quota model)**

RunAI organizes teams into **Projects** (leaf quota units) and **Departments** (parent quota pools):

```yaml
# RunAI Project — team-a gets 16 GPUs guaranteed, can over-provision to 24
apiVersion: run.ai/v1
kind: Project
metadata:
  name: team-a
spec:
  deservedGpus: 16
  maxAllowedGpus: 24
  nodePools:
    - default
```

**Submitting a training job**

```bash
# runai CLI — submit a distributed PyTorch job
runai submit distributed-train \
  --image nvcr.io/nvidia/pytorch:24.02-py3 \
  --gpu 8 \
  --workers 4 \
  --project team-a \
  -- python3 train.py --nnodes=4 --nproc_per_node=8

# Monitor jobs
runai list jobs -p team-a
runai describe job distributed-train -p team-a

# View GPU utilization across projects
runai top node
```

**GPU sharing features**

RunAI supports sub-GPU allocation via its fractional GPU feature (time-slicing or MIG-backed) and dynamic GPU memory limits, enabling inference and development workloads to share nodes with training jobs without wasting idle GPU cycles. MIG (Multi-Instance GPU) is an NVIDIA A100/H100 hardware feature that partitions a single GPU into up to seven independent GPU instances, each with isolated memory and compute slices, enabling multiple workloads to run on one physical GPU with hard performance isolation.

```bash
# Inference workload requesting 0.25 GPU
runai submit inference-server \
  --image my-inference:latest \
  --gpu 0.25 \
  --project team-a \
  -- python3 serve.py
```

**Integration with Kubernetes scheduling primitives**

RunAI respects standard Kubernetes constructs (node selectors, tolerations, resource quotas) while adding its own priority and fairness layer on top. Jobs submitted through RunAI coexist with jobs submitted via vanilla `kubectl apply`; RunAI workloads preempt lower-priority non-RunAI pods according to the configured fairness policy.

---

### 29.9.3 NVIDIA Dynamo — Inference Serving Framework

**NVIDIA Dynamo** is an open-source distributed inference serving framework designed for disaggregated prefill-decode architectures. Where SLURM and RunAI focus on scheduling, Dynamo focuses on the inference workload itself — routing requests across prefill workers and decode workers to maximize GPU utilization and minimize time-to-first-token (TTFT).

**Prefill-decode disaggregation**

Large language model inference has two phases:
- **Prefill**: process the prompt (compute-bound, parallelizable across sequence length)
- **Decode**: generate tokens autoregressively (memory-bandwidth-bound, low arithmetic intensity)

Running both phases on the same GPU wastes resources. Dynamo separates them:

```
Request Router (Dynamo)
      │
      ├── Prefill Workers  (high-compute GPUs: H100 SXM)
      │        │  KV Cache transfer (NVLink / RDMA)
      └── Decode Workers   (high-memory-bandwidth GPUs: H100 PCIe or A100)
```

**Architecture components**

```
dynamo-router      — HTTP/gRPC frontend; routes requests to prefill/decode pools
dynamo-worker      — vLLM-backed inference worker; registers with router
KV cache store     — shared distributed KV cache (RDMA or NVLink transfer)
Planner            — monitors GPU utilization, scales workers, migrates KV blocks
```

`vLLM` is a high-throughput LLM inference serving library that implements PagedAttention (efficient KV cache memory management) and continuous batching, making it the dominant open-source inference engine for per-token-optimized GPU serving.

**Deployment on Kubernetes**

Dynamo ships Helm charts and Kubernetes operators:

```bash
# Install Dynamo via Helm
helm install dynamo oci://nvcr.io/nvidia/dynamo/charts/dynamo \
  --namespace dynamo-system \
  --set model.name=meta-llama/Llama-3-70B \
  --set prefill.replicas=4 \
  --set decode.replicas=8 \
  --set router.gpuFraction=0.1

# Check serving status
kubectl get dynamo -n dynamo-system
kubectl get pods -n dynamo-system -l app=dynamo-router
```

**Network requirements**

Dynamo's KV cache transfer between prefill and decode workers is the performance-critical path. It uses:
- **NVLink** (within a node) for intra-node KV transfer
- **RoCEv2 RDMA** (across nodes) for cross-node KV transfer — the same fabric discussed in Ch. 2

KV transfer bandwidth directly limits the sustainable request rate. At 70B model scale, each prefill→decode KV transfer is ~512 MB; with 1000 req/s, that is 512 GB/s of cross-node RDMA traffic — close to the full fabric bandwidth of a 400 Gb/s rail. Proper rail-to-worker placement (§29.6) is critical.

**Grove — Scheduling Extension**

**Grove** is NVIDIA's scheduling component within the Dynamo ecosystem, responsible for placing prefill and decode workers onto nodes with the correct GPU topology, RDMA rail affinity, and NVLink connectivity. It extends the Kubernetes scheduler (via a webhook) with Dynamo-specific placement constraints:

```yaml
# DynamoWorker spec with Grove placement hints
apiVersion: dynamo.nvidia.com/v1alpha1
kind: DynamoWorker
metadata:
  name: prefill-worker-0
spec:
  role: prefill
  model: meta-llama/Llama-3-70B
  resources:
    limits:
      nvidia.com/gpu: "8"
  placement:
    grove.nvidia.com/rdma-rail: "rail-0"
    grove.nvidia.com/nvlink-domain: "domain-0"
    grove.nvidia.com/kv-peer: "decode-worker-0"
```

Grove queries the cluster topology graph (populated via DCGM — NVIDIA's Data Center GPU Manager, which exposes per-GPU health metrics, topology, and utilization via a Kubernetes device plugin and Prometheus exporter — and network discovery) and ensures that prefill and decode worker pairs are placed to minimize KV transfer latency — on the same NVLink domain if possible, or on the same RDMA rail if cross-node transfer is unavoidable.

---

### 29.9.4 Choosing Between Schedulers

| Scenario | Recommended scheduler |
|----------|----------------------|
| HPC-origin MPI jobs, bare-metal GPU nodes | SLURM |
| Multi-tenant Kubernetes, mixed workloads | Volcano + Kueue |
| Gang scheduling only, existing kube-scheduler | Coscheduler |
| ML team governance, GPU utilization dashboards | RunAI (on top of Kubernetes) |
| Disaggregated LLM inference at scale | NVIDIA Dynamo + Grove |
| Hybrid HPC + Kubernetes | SLURM as outer scheduler, Pyxis for containers |

---

## Lab Walkthrough 29 — Gang Scheduling with Volcano and Topology-Aware Placement

This lab creates a 3-node Kind cluster with rack labels, installs Volcano, demonstrates gang scheduling guarantees, installs Kueue, and writes a topology advisor script that analyzes pod placement relative to rack boundaries.

### Step 1 — Create a 3-node Kind cluster with node labels

Create `kind-ai-cluster.yaml`:

```yaml
# kind-ai-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: ai-sched-lab
nodes:
  - role: control-plane
    labels:
      rack: rack-0
      accelerator: cpu-only
  - role: worker
    labels:
      rack: rack-1
      accelerator: gpu
      node-type: gpu-worker
  - role: worker
    labels:
      rack: rack-2
      accelerator: gpu
      node-type: gpu-worker
```

Create the cluster:

```bash
kind create cluster --config kind-ai-cluster.yaml
```

Expected output:

```
Creating cluster "ai-sched-lab" ...
 ✓ Ensuring node image (kindest/node:v1.30.2) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-ai-sched-lab"
```

Verify node labels:

```bash
kubectl get nodes --show-labels
```

```
NAME                         STATUS   ROLES           AGE   VERSION   LABELS
ai-sched-lab-control-plane   Ready    control-plane   60s   v1.30.2   ...,rack=rack-0
ai-sched-lab-worker          Ready    <none>          40s   v1.30.2   ...,rack=rack-1,accelerator=gpu
ai-sched-lab-worker2         Ready    <none>          40s   v1.30.2   ...,rack=rack-2,accelerator=gpu
```

### Step 2 — Install Volcano via Helm

```bash
helm repo add volcano-sh https://volcano-sh.github.io/helm-charts
helm repo update
helm install volcano volcano-sh/volcano \
    --namespace volcano-system \
    --create-namespace \
    --version 1.9.0
```

Expected output:

```
NAME: volcano
LAST DEPLOYED: Tue Apr 22 10:00:00 2026
NAMESPACE: volcano-system
STATUS: deployed
REVISION: 1
```

Wait for components to be ready:

```bash
kubectl -n volcano-system rollout status deployment/volcano-scheduler
kubectl -n volcano-system rollout status deployment/volcano-controller-manager
kubectl get pods -n volcano-system
```

```
NAME                                         READY   STATUS    RESTARTS   AGE
volcano-admission-7c6b4b58c5-9x8kq           1/1     Running   0          45s
volcano-controller-manager-5d8f9d4f6-hqnbj   1/1     Running   0          45s
volcano-scheduler-674fc9d974-jkl2p            1/1     Running   0          45s
```

### Step 3 — Create a Volcano Queue with guaranteed capacity

Create `volcano-queue.yaml`:

```yaml
# volcano-queue.yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: training
spec:
  weight: 10
  capability:
    cpu: "4"
    memory: "8Gi"
  reclaimable: false
```

Apply and verify:

```bash
kubectl apply -f volcano-queue.yaml
vcctl queue list
```

```
Name       Weight    Pending    Running    Unknown    Inqueue    Binding    Allocated  Succeeded  Failed     Preemptable
default    1         0          0          0          0          0          0          0          0          false
training   10        0          0          0          0          0          0          0          0          false
```

### Step 4 — Submit a PodGroup and Job with minMember=2

Create `gang-job.yaml` — two worker pods that sleep for 30 seconds then exit:

```yaml
# gang-job.yaml
---
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: gang-pg
  namespace: default
spec:
  minMember: 2
  queue: training

---
apiVersion: batch/v1
kind: Job
metadata:
  name: gang-job
  namespace: default
spec:
  completions: 2
  parallelism: 2
  template:
    metadata:
      labels:
        volcano.sh/job-name: gang-job
        volcano.sh/pod-group: gang-pg
    spec:
      schedulerName: volcano
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.36
        command: ["sh", "-c", "echo rank $MY_POD_NAME started; sleep 30; echo done"]
        env:
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
```

Apply and observe:

```bash
kubectl apply -f gang-job.yaml

# Watch pod creation — both pods should appear simultaneously
kubectl get pods -w --selector=volcano.sh/job-name=gang-job
```

```
NAME              READY   STATUS    RESTARTS   AGE
gang-job-9k2bx    0/1     Pending   0          1s
gang-job-w7pnm    0/1     Pending   0          1s
gang-job-9k2bx    0/1     ContainerCreating   0   2s
gang-job-w7pnm    0/1     ContainerCreating   0   2s
gang-job-9k2bx    1/1     Running   0          4s
gang-job-w7pnm    1/1     Running   0          4s
```

Both pods transition from Pending to ContainerCreating within the same second — Volcano held both in Pending until it could schedule both simultaneously.

### Step 5 — Demonstrate gang scheduling guarantee with insufficient nodes

Create `gang-overprovision.yaml` requesting 3 pods on a cluster that only has 2 schedulable worker nodes:

```yaml
# gang-overprovision.yaml
---
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: oversize-pg
  namespace: default
spec:
  minMember: 3     # requires 3 nodes, but only 2 workers exist
  queue: training

---
apiVersion: batch/v1
kind: Job
metadata:
  name: oversize-job
  namespace: default
spec:
  completions: 3
  parallelism: 3
  template:
    metadata:
      labels:
        volcano.sh/pod-group: oversize-pg
    spec:
      schedulerName: volcano
      restartPolicy: Never
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values: ["gpu-worker"]   # only schedule on the 2 worker nodes
      containers:
      - name: worker
        image: busybox:1.36
        command: ["sleep", "60"]
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
```

```bash
kubectl apply -f gang-overprovision.yaml
sleep 10
kubectl get pods --selector=volcano.sh/pod-group=oversize-pg
```

```
NAME                  READY   STATUS    RESTARTS   AGE
oversize-job-4nkzp    0/1     Pending   0          10s
oversize-job-hq7mn    0/1     Pending   0          10s
oversize-job-vt9xr    0/1     Pending   0          10s
```

All 3 pods remain Pending. Without gang scheduling, the default scheduler would have started 2 of the 3 pods and left one in Pending — wasting GPU resources on two running pods that cannot form a complete training group.

Check the PodGroup status:

```bash
kubectl get podgroup oversize-pg -o yaml | grep -A5 "status:"
```

```yaml
status:
  conditions:
  - lastTransitionTime: "2026-04-22T10:05:00Z"
    message: '3/3 tasks in gang are not yet schedulable: 3 Pending, 0 Running, 0 Succeeded, 0 Failed'
    reason: NotEnoughResources
    status: "True"
    type: Unschedulable
  phase: Pending
  succeeded: 0
```

Clean up:

```bash
kubectl delete -f gang-overprovision.yaml
```

### Step 6 — Install Kueue and create ResourceFlavor, ClusterQueue, LocalQueue

Install Kueue:

```bash
KUEUE_VERSION=v0.7.0
kubectl apply --server-side \
    -f "https://github.com/kubernetes-sigs/kueue/releases/download/${KUEUE_VERSION}/manifests.yaml"

kubectl -n kueue-system rollout status deployment/kueue-controller-manager
```

```
deployment.apps/kueue-controller-manager successfully rolled out
```

Create `kueue-resources.yaml`:

```yaml
# kueue-resources.yaml
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: gpu-flavor-rack1
spec:
  nodeLabels:
    rack: rack-1
    accelerator: gpu

---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: gpu-flavor-rack2
spec:
  nodeLabels:
    rack: rack-2
    accelerator: gpu

---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cluster-queue
spec:
  namespaceSelector: {}
  queueingStrategy: BestEffortFIFO
  resourceGroups:
  - coveredResources: ["cpu", "memory"]
    flavors:
    - name: gpu-flavor-rack1
      resources:
      - name: cpu
        nominalQuota: "2"
      - name: memory
        nominalQuota: "4Gi"
    - name: gpu-flavor-rack2
      resources:
      - name: cpu
        nominalQuota: "2"
      - name: memory
        nominalQuota: "4Gi"

---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: team-a-queue
  namespace: default
spec:
  clusterQueueName: cluster-queue
```

```bash
kubectl apply -f kueue-resources.yaml
kubectl get clusterqueue cluster-queue -o wide
```

```
NAME            COHORT   PENDING WORKLOADS   ADMITTED WORKLOADS
cluster-queue            0                   0
```

### Step 7 — Submit a batch of jobs via Kueue and observe admission

Create `kueue-batch-jobs.yaml` — 4 independent batch jobs each requesting 500m CPU, labeled for the LocalQueue:

```yaml
# kueue-batch-jobs.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job-1
  namespace: default
  labels:
    kueue.x-k8s.io/queue-name: team-a-queue
spec:
  suspend: true      # Kueue will unsuspend when admitted
  completions: 1
  parallelism: 1
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.36
        command: ["sh", "-c", "echo job1 running; sleep 20"]
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
```

Apply 4 such jobs (create batch-job-2, batch-job-3, batch-job-4 similarly):

```bash
for i in 1 2 3 4; do
  sed "s/batch-job-1/batch-job-${i}/" kueue-batch-jobs.yaml | kubectl apply -f -
done
```

Observe admission:

```bash
kubectl get workloads -n default
```

```
NAME                  QUEUE           ADMITTED   LATEST TRANSITION   AGE
job-batch-job-1-abc   team-a-queue    True        10s                 12s
job-batch-job-2-def   team-a-queue    True        10s                 12s
job-batch-job-3-ghi   team-a-queue    False       10s                 12s
job-batch-job-4-jkl   team-a-queue    False       10s                 12s
```

Jobs 1 and 2 are admitted (fitting within quota); jobs 3 and 4 are queued. Check ClusterQueue status:

```bash
kubectl get clusterqueue cluster-queue -o wide
```

```
NAME            COHORT   PENDING WORKLOADS   ADMITTED WORKLOADS
cluster-queue            2                   2
```

As jobs 1 and 2 complete, jobs 3 and 4 are admitted automatically.

### Step 8 — Write topology_advisor.py

```python
# topology_advisor.py
"""
Reads kubectl node and pod data, identifies pod-to-rack placement,
and estimates whether AllReduce traffic crosses the spine layer.
Usage: python topology_advisor.py [--label-selector app=training-job]
"""
import json, subprocess, sys, argparse
from collections import defaultdict

def kubectl_json(args):
    result = subprocess.run(["kubectl"] + args + ["-o", "json"],
                            capture_output=True, text=True, check=True)
    return json.loads(result.stdout)

def get_node_rack_map():
    nodes = kubectl_json(["get", "nodes"])
    rack_map = {}
    for node in nodes["items"]:
        name   = node["metadata"]["name"]
        labels = node["metadata"].get("labels", {})
        rack   = labels.get("rack", "unknown")
        rack_map[name] = rack
    return rack_map

def get_pod_placement(selector=None):
    cmd = ["get", "pods", "--field-selector=status.phase=Running"]
    if selector:
        cmd += ["-l", selector]
    pods = kubectl_json(cmd)
    placements = []
    for pod in pods["items"]:
        name     = pod["metadata"]["name"]
        node     = pod["spec"].get("nodeName", "unscheduled")
        phase    = pod["status"].get("phase", "Unknown")
        pod_ip   = pod["status"].get("podIP", "")
        placements.append({"pod": name, "node": node, "phase": phase, "ip": pod_ip})
    return placements

def analyze_topology(placements, rack_map):
    # Map each pod to its rack
    pod_racks = {}
    rack_pods = defaultdict(list)
    for p in placements:
        rack = rack_map.get(p["node"], "unknown")
        pod_racks[p["pod"]] = rack
        rack_pods[rack].append(p["pod"])

    total_pods = len(placements)
    unique_racks = set(pod_racks.values())
    pods_on_largest_rack = max((len(v) for v in rack_pods.values()), default=0)

    print(f"\n{'='*60}")
    print(f"  Topology Analysis Report")
    print(f"{'='*60}")
    print(f"  Total running pods  : {total_pods}")
    print(f"  Racks used          : {len(unique_racks)} — {sorted(unique_racks)}")
    print()

    print(f"  {'Pod':<40} {'Node':<30} {'Rack'}")
    print(f"  {'-'*40} {'-'*30} {'-'*10}")
    for p in sorted(placements, key=lambda x: x["node"]):
        rack = pod_racks.get(p["pod"], "?")
        print(f"  {p['pod']:<40} {p['node']:<30} {rack}")

    print()
    print(f"  Rack distribution:")
    for rack, pods in sorted(rack_pods.items()):
        print(f"    {rack}: {len(pods)} pods — {pods}")

    print()
    if len(unique_racks) == 1:
        print("  AllReduce traffic assessment: INTRA-RACK ONLY")
        print("  => No spine traversal. All traffic stays within ToR switch.")
        print(f"     Estimated east-west RTT: ~1-2µs")
        spine_traversals = 0
    elif len(unique_racks) == 2:
        print("  AllReduce traffic assessment: CROSS-RACK (2 racks)")
        print("  => Traffic crosses spine switch once each way.")
        print(f"     Estimated east-west RTT: ~3-5µs (+2-3µs vs intra-rack)")
        spine_traversals = 1
    else:
        print(f"  AllReduce traffic assessment: MULTI-RACK ({len(unique_racks)} racks)")
        print("  => Traffic crosses spine switch. Consider rack-constrained placement.")
        print(f"     Estimated east-west RTT: ~5-10µs")
        spine_traversals = len(unique_racks) - 1

    # Estimate AllReduce time penalty for 10 GB tensor, 256 ranks (illustrative)
    if total_pods > 1 and spine_traversals > 0:
        EXTRA_RTT_US = spine_traversals * 3.0   # 3µs per extra spine hop
        RING_STEPS   = 2 * (total_pods - 1)
        EXTRA_MS     = (EXTRA_RTT_US * 2 * RING_STEPS) / 1000
        print(f"\n  Collective latency estimate ({total_pods} ranks, 10 GB tensor):")
        print(f"    Extra RTT per hop  : {EXTRA_RTT_US:.1f}µs")
        print(f"    Ring steps         : {RING_STEPS}")
        print(f"    Extra latency/step : {EXTRA_MS:.2f}ms")
        print(f"    (vs optimal intra-rack placement)")

    print(f"{'='*60}\n")
    return {"racks": len(unique_racks), "spine_traversals": spine_traversals}

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="AI training topology advisor")
    parser.add_argument("--label-selector", default=None,
                        help="kubectl -l selector for pods to analyze")
    args = parser.parse_args()

    rack_map   = get_node_rack_map()
    placements = get_pod_placement(selector=args.label_selector)

    if not placements:
        print("No running pods found matching the selector.")
        sys.exit(0)

    analyze_topology(placements, rack_map)
```

Run it against the current cluster:

```bash
source .venv/bin/activate
python topology_advisor.py
```

```
============================================================
  Topology Analysis Report
============================================================
  Total running pods  : 4
  Racks used          : 2 — ['rack-1', 'rack-2']

  Pod                                      Node                           Rack
  ---------------------------------------- ------------------------------ ----------
  batch-job-1-abc-xyz                      ai-sched-lab-worker            rack-1
  batch-job-2-def-xyz                      ai-sched-lab-worker2           rack-2
  gang-job-9k2bx                           ai-sched-lab-worker            rack-1
  gang-job-w7pnm                           ai-sched-lab-worker2           rack-2

  Rack distribution:
    rack-1: 2 pods — ['batch-job-1-abc-xyz', 'gang-job-9k2bx']
    rack-2: 2 pods — ['batch-job-2-def-xyz', 'gang-job-w7pnm']

  AllReduce traffic assessment: CROSS-RACK (2 racks)
  => Traffic crosses spine switch once each way.
     Estimated east-west RTT: ~3-5µs (+2-3µs vs intra-rack)

  Collective latency estimate (4 ranks, 10 GB tensor):
    Extra RTT per hop  : 3.0µs
    Ring steps         : 6
    Extra latency/step : 0.04ms
    (vs optimal intra-rack placement)
============================================================
```

### Step 9 — Apply topologySpreadConstraints to pin a 4-pod job to rack-1

Create `rack-pinned-job.yaml`:

```yaml
# rack-pinned-job.yaml
---
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: rack-pinned-pg
  namespace: default
spec:
  minMember: 2
  queue: training

---
apiVersion: batch/v1
kind: Job
metadata:
  name: rack-pinned-job
spec:
  completions: 2
  parallelism: 2
  template:
    metadata:
      labels:
        app: rack-pinned-training
        volcano.sh/pod-group: rack-pinned-pg
    spec:
      schedulerName: volcano
      restartPolicy: Never
      # Pin to rack-1 only
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: rack
                operator: In
                values: ["rack-1"]
      # Spread evenly within rack-1 (maxSkew=1 since rack-1 has only 1 node)
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: rack
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: rack-pinned-training
      containers:
      - name: worker
        image: busybox:1.36
        command: ["sh", "-c", "echo rack-pinned rank started; sleep 60"]
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
```

```bash
kubectl apply -f rack-pinned-job.yaml
kubectl wait --for=condition=Ready pod -l app=rack-pinned-training --timeout=60s
kubectl get pods -l app=rack-pinned-training -o wide
```

```
NAME                      READY   STATUS    RESTARTS   AGE   IP          NODE
rack-pinned-job-4xlnp     1/1     Running   0          8s    10.244.1.8  ai-sched-lab-worker
rack-pinned-job-9kqzm     1/1     Running   0          8s    10.244.1.9  ai-sched-lab-worker
```

Both pods land on `ai-sched-lab-worker` which carries label `rack=rack-1`. Now run the topology advisor filtered to this job:

```bash
python topology_advisor.py --label-selector app=rack-pinned-training
```

```
============================================================
  Topology Analysis Report
============================================================
  Total running pods  : 2
  Racks used          : 1 — ['rack-1']

  Pod                                      Node                           Rack
  ---------------------------------------- ------------------------------ ----------
  rack-pinned-job-4xlnp                    ai-sched-lab-worker            rack-1
  rack-pinned-job-9kqzm                    ai-sched-lab-worker            rack-1

  Rack distribution:
    rack-1: 2 pods — ['rack-pinned-job-4xlnp', 'rack-pinned-job-9kqzm']

  AllReduce traffic assessment: INTRA-RACK ONLY
  => No spine traversal. All traffic stays within ToR switch.
     Estimated east-west RTT: ~1-2µs
============================================================
```

`INTRA-RACK ONLY` confirms successful rack-pinned placement.

### Step 10 — Show the AllReduce topology difference quantitatively

The topology advisor already calculates the penalty. Here is a more detailed worked example for a realistic 4-node, 32-GPU training job (8 GPUs/node):

```
Scenario A: 4 nodes on same rack (no spine traversal)
  Intra-node AllReduce (NVLink):  8 GPUs × 600 GB/s NVLink = ~75 GB/s per GPU
  Inter-node AllReduce (RoCE):    4 nodes × 200 Gb/s NIC   = 100 Gb/s = 12.5 GB/s
  RTT within ToR:                ~1.5µs
  AllReduce time for 10 GB tensor: ~800ms (bandwidth-bound, NIC is bottleneck)

Scenario B: 4 nodes spread across 4 racks (spine traversal on each hop)
  Inter-node RTT (spine crossed):  ~4.5µs  (3µs extra vs intra-rack)
  Ring steps for 32 ranks:         2 × 31 = 62 steps
  Extra latency:                   2 × 3µs × 62 = 0.37ms per AllReduce
  Extra steps for 10 GB tensor:    (pipeline stages can hide this)
  Bandwidth-limited case:          same 12.5 GB/s per rail
  Latency-limited case:            +0.37ms × 1000 steps/hr = +370ms/hr

At 1000 training steps per hour:
  Scenario A: 800ms/step × 1000 = 800s/hr spent in AllReduce
  Scenario B: 800.37ms/step × 1000 = 800.37s/hr spent in AllReduce
  Difference: 0.37s/hr  (~0.05% overhead)

For latency-sensitive workloads with small tensors (pipeline parallelism, frequent barriers):
  Tensor size: 10 MB, steps: 100,000/hr
  Scenario A: ~0.8ms/step × 100,000 = 80s/hr
  Scenario B: ~1.17ms/step × 100,000 = 117s/hr  (+46% overhead)
```

The spine traversal penalty is most significant for **high-frequency, small-tensor** collectives (e.g., pipeline-parallel bubble communications) rather than large AllReduce operations. For large LLM training with FSDP (Fully Sharded Data Parallel — a PyTorch parallelism strategy that shards model parameters, gradients, and optimizer states across all ranks, reducing per-GPU memory by a factor of world size) and infrequent checkpoints, rack placement has a smaller impact. For dense pipeline-parallel training with many microbatch synchronizations, same-rack placement can reduce collective overhead by up to 46%.

```bash
# Quick verification: compare ping RTT within rack vs cross-rack
kubectl exec rack-pinned-job-4xlnp -- ping -c 10 -W 1 \
    "$(kubectl get pod rack-pinned-job-9kqzm -o jsonpath='{.status.podIP}')" \
    | tail -3
```

```
10 packets transmitted, 10 received, 0% packet loss, time 9012ms
rtt min/avg/max/mdev = 0.041/0.057/0.083/0.012 ms
```

Both pods on the same node (same Kind worker), so RTT is sub-100µs. In a real cluster with rack-level separation, the additional ~3µs RTT from spine traversal would appear here.

### Step 11 — Cleanup

```bash
# Delete all lab resources
kubectl delete -f rack-pinned-job.yaml 2>/dev/null
kubectl delete -f gang-job.yaml 2>/dev/null
kubectl delete -f kueue-resources.yaml 2>/dev/null
kubectl delete -f volcano-queue.yaml 2>/dev/null

# Uninstall Volcano and Kueue
helm uninstall volcano -n volcano-system
KUEUE_VERSION=v0.7.0
kubectl delete \
    -f "https://github.com/kubernetes-sigs/kueue/releases/download/${KUEUE_VERSION}/manifests.yaml" \
    2>/dev/null

# Destroy the Kind cluster
kind delete cluster --name ai-sched-lab
```

Expected output:

```
Deleting cluster "ai-sched-lab" ...
```

```bash
# Deactivate Python environment and remove generated files
deactivate
rm -f kind-ai-cluster.yaml volcano-queue.yaml gang-job.yaml gang-overprovision.yaml \
      kueue-batch-jobs.yaml kueue-resources.yaml rack-pinned-job.yaml topology_advisor.py
```

---

## Summary

- Standard `kube-scheduler` is fundamentally incompatible with distributed AI training because it schedules pods independently, is blind to GPU and rack topology, and creates resource fragmentation that prevents large gang jobs from ever being admitted.
- Gang scheduling (Volcano, Coscheduler) provides the all-or-nothing semantic: all pods in a PodGroup are either scheduled simultaneously or none are. This prevents partial launches that waste resources and block at the AllReduce barrier.
- Volcano introduces `Queue`, `PodGroup`, and `Job` CRDs. Queues support weighted fair sharing, borrowing, and preemption. The `vcctl` CLI provides operational visibility into queue depth and job phase.
- Kueue acts as an admission controller above `kube-scheduler`, managing when workloads enter the scheduling pipeline. `ResourceFlavor`, `ClusterQueue`, and `LocalQueue` implement topology-aware quota management without replacing the underlying scheduler.
- Coscheduler implements gang scheduling as a `kube-scheduler` plugin, making it composable with existing node affinity, taint, and topology spread features without deploying a separate scheduler binary.
- Topology-aware placement using `topologySpreadConstraints` and node labels (`rack`, `accelerator`) enables same-rack co-location, eliminating spine traversal for intra-job AllReduce traffic.
- Spine traversal adds approximately 3µs RTT per hop. For large tensor AllReduce this is negligible; for high-frequency pipeline-parallel communications with small tensors it can increase collective overhead by 40–50%.
- `NCCL_TOPO_FILE` allows operators to override NCCL's topology autodetection with an authoritative XML topology, ensuring correct rank-to-NIC affinity regardless of how pods are scheduled.
- Priority classes for training, inference, and experiment workloads, combined with Volcano preemption and Kueue quota management, enable multi-tenant AI clusters with predictable guarantees for production training jobs.

---

## References

- [Volcano documentation](https://volcano.sh/docs)
- [Volcano GitHub](https://github.com/volcano-sh/volcano)
- [Kueue documentation](https://kueue.sigs.k8s.io/docs)
- [Kueue GitHub](https://github.com/kubernetes-sigs/kueue)
- [scheduler-plugins (Coscheduler)](https://github.com/kubernetes-sigs/scheduler-plugins)
- [NCCL topology documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html#nccl-topo-file)
- [Kubernetes Topology Manager](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager)
- [Kind (Kubernetes in Docker)](https://kind.sigs.k8s.io)
- [Kubernetes PriorityClass](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption)
- [SLURM Workload Manager](https://slurm.schedmd.com/documentation.html)
- [PMIx — Process Management Interface for Exascale](https://pmix.org)
- [Pyxis — SLURM plugin for container jobs](https://github.com/NVIDIA/pyxis)
- [Enroot — container runtime for HPC](https://github.com/NVIDIA/enroot)
- [RunAI GPU scheduling platform](https://docs.run.ai)
- [NVIDIA Dynamo — distributed inference serving framework](https://github.com/ai-dynamo/dynamo)
- [vLLM — high-throughput LLM inference library](https://docs.vllm.ai)
- [DCGM — NVIDIA Data Center GPU Manager](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/index.html)
- [NVIDIA GPU device plugin for Kubernetes](https://github.com/NVIDIA/k8s-device-plugin)
- [Alibaba AI infrastructure blog](https://medium.com/alibaba-cloud/volcano-apache-rocketmq-e1b64f5c3d88)


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).