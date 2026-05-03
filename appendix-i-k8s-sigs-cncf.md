# Appendix I — Kubernetes SIGs & CNCF Project Landscape

*The AI Cluster Network Stack — Building High-Performance Fabrics for GPU Clusters from First Principles*

---

> **Scope note.** This appendix is a curated reference for engineers building or operating AI cluster infrastructure on Kubernetes — not an exhaustive catalogue of all Kubernetes SIGs or CNCF projects. CNCF graduation statuses are verified as of **May 2026**. Projects that are not formally CNCF-hosted but are widely deployed in AI cluster infrastructure are included in the tables with a **Non-CNCF** label and their actual governance home in parentheses.

---

## Part 1 — Kubernetes Special Interest Groups

Kubernetes governance is structured around Special Interest Groups (SIGs) and Working Groups (WGs). SIGs own persistent areas of the codebase and specifications; WGs are time-bounded cross-SIG efforts that disband once their goals are achieved. The ten groups below are the most relevant to AI cluster networking and infrastructure.

---

### sig-network

**SIG Network** owns all in-cluster networking concerns: the **Container Network Interface (CNI)** specification (implemented by Cilium, Multus, Calico, and others — see Ch. 12, 13), **NetworkPolicy** semantics, **Service** and **EndpointSlice** APIs, **DNS** via **CoreDNS**, **kube-proxy** (iptables and IPVS modes), and the **Gateway API** (`GatewayClass`, `Gateway`, `HTTPRoute`, `GRPCRoute`, `TCPRoute`). For AI inference infrastructure, Gateway API's `GRPCRoute` resource is used to route disaggregated prefill/decode traffic (Ch. 29, 32). SIG Network maintains the **Network Policies** KEPs that define east-west isolation between training jobs.

Key deliverables:
- CNI specification (`github.com/containernetworking/cni`)
- Gateway API (`github.com/kubernetes-sigs/gateway-api`)
- NetworkPolicy v1 and AdminNetworkPolicy KEPs
- CoreDNS integration and DNS autoscaling
- kube-proxy IPVS and nftables backends

GitHub: <https://github.com/kubernetes/community/tree/master/sig-network>  
Docs: <https://sig-network.sigs.k8s.io/>

---

### sig-node

**SIG Node** owns the **kubelet** agent, the **Container Runtime Interface (CRI)**, **RuntimeClass** (for selecting GPU-optimized or confidential runtimes), **cgroup v2** integration, and the **Device Plugin API** (`k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1`). The Device Plugin API is the mechanism by which `nvidia-device-plugin` advertises `nvidia.com/gpu` resources to the scheduler (Ch. 32). SIG Node is also the primary owner of **Dynamic Resource Allocation (DRA)** in the kubelet, coordinating with WG Device Management on the API surface. The shift from cgroup v1 to **cgroup v2** unified hierarchy is a prerequisite for per-container memory and CPU pressure accounting in multi-tenant GPU nodes.

Key deliverables:
- kubelet source and lifecycle management
- CRI API (used by containerd and CRI-O — see Part 2)
- Device Plugin API v1beta1 and DRA kubelet integration
- RuntimeClass resource for hardware-specific runtime selection
- cgroup v2 resource management and memory QoS

GitHub: <https://github.com/kubernetes/community/tree/master/sig-node>  
Docs: <https://sig-node.sigs.k8s.io/>

---

### sig-scheduling

**SIG Scheduling** owns the **kube-scheduler** framework, the plugin interface (`PreFilter`, `Filter`, `Score`, `Reserve`, `Bind` extension points), and topology-awareness extensions. For AI training, the critical extension is **gang scheduling** — the ability to admit a PodGroup only when all pods can be scheduled simultaneously, preventing resource deadlock. **Coscheduler** (implemented as a kube-scheduler plugin) and **Volcano** (Ch. 29) both use the scheduler framework. SIG Scheduling also maintains the **Descheduler** project for rebalancing running workloads.

Key deliverables:
- kube-scheduler plugin framework (`github.com/kubernetes-sigs/scheduler-plugins`)
- TopologySpreadConstraints for rack-awareness
- Coscheduler / PodGroup gang scheduling plugin
- Descheduler (`github.com/kubernetes-sigs/descheduler`)
- Scheduling Framework KEPs (preemption, resource fit, binpacking)

GitHub: <https://github.com/kubernetes/community/tree/master/sig-scheduling>  
Docs: <https://kubernetes.io/docs/concepts/scheduling-eviction/>

---

### sig-storage

**SIG Storage** owns the **Container Storage Interface (CSI)** specification, Persistent Volume (PV) and PersistentVolumeClaim (PVC) lifecycle, **StorageClass**, **VolumeSnapshot**, and volume health monitoring. For AI cluster infrastructure, fast checkpoint I/O to distributed storage (Ceph/Rook, Longhorn — see Part 2) flows through CSI drivers. The `NodeStageVolume`/`NodePublishVolume` RPC lifecycle (Ch. 32) determines how NVMe-oF volumes (Ch. 6) and parallel filesystems attach inside containers.

Key deliverables:
- CSI specification and conformance tests (`github.com/kubernetes-sigs/csi-driver-nfs` et al.)
- VolumeSnapshot API (`github.com/kubernetes-csi/external-snapshotter`)
- `StorageClass` and dynamic provisioning
- `VolumeAttributesClass` for dynamic I/O throttling (KEP-3751)
- Volume health monitoring and node-side failure reporting

GitHub: <https://github.com/kubernetes/community/tree/master/sig-storage>  
Docs: <https://kubernetes.io/docs/concepts/storage/>

---

### sig-instrumentation

**SIG Instrumentation** owns the **Metrics API** (`metrics.k8s.io`), the **Custom Metrics API** (consumed by HPA for GPU utilization-based autoscaling), **structured logging** across core components, and the integration of **OpenTelemetry** tracing into Kubernetes control plane components. The observability pipeline described in Ch. 16 (OpenTelemetry → Prometheus → Grafana) builds on the metric-server and custom metrics infrastructure this SIG defines.

Key deliverables:
- `metrics-server` (`github.com/kubernetes-sigs/metrics-server`)
- Custom Metrics API (`github.com/kubernetes-sigs/custom-metrics-apiserver`)
- Structured logging migration across kube-apiserver, kubelet, and scheduler
- OpenTelemetry trace context propagation in kube-apiserver
- Logging format standards and log sanitization

GitHub: <https://github.com/kubernetes/community/tree/master/sig-instrumentation>  
Docs: <https://kubernetes.io/docs/concepts/cluster-administration/logging/>

---

### sig-auth

**SIG Auth** owns **Role-Based Access Control (RBAC)**, **ServiceAccount** token projection (bound tokens, audience restriction), **Pod Security** admission, and coordination with **cert-manager** for in-cluster PKI. In AI cluster deployments, RBAC scopes which service accounts can schedule GPU workloads; ServiceAccount tokens provide workload identity for SPIFFE/SPIRE mTLS (Ch. 26). Pod Security Standards (Baseline, Restricted) govern privilege levels of GPU containers.

Key deliverables:
- RBAC API (`ClusterRole`, `RoleBinding`, etc.) and audit logging
- ServiceAccount token projection with bounded audiences
- Pod Security Admission and Pod Security Standards
- `CertificateSigningRequest` API for in-cluster certificate issuance
- Node authorization mode (limiting kubelet API access to own node objects)

GitHub: <https://github.com/kubernetes/community/tree/master/sig-auth>  
Docs: <https://kubernetes.io/docs/reference/access-authn-authz/>

---

### sig-multicluster

**SIG Multicluster** defines APIs for operating federated Kubernetes environments — relevant when AI training spans multiple clusters (e.g., cross-region model-parallel jobs) or when inference is load-balanced globally. The SIG owns the **Multi-Cluster Services (MCS) API** (`ServiceImport`/`ServiceExport`) that extends Services across clusters within a **ClusterSet**, the **ClusterProfile** (formerly ClusterProperty) API for cluster self-description, and the **Work API** for deploying workloads across clusters. **Liqo** and **Submariner** (Part 2) implement these APIs.

Key deliverables:
- MCS API v1alpha1 (`github.com/kubernetes-sigs/mcs-api`)
- ClusterProfile / ClusterSet API (`github.com/kubernetes-sigs/about-api`)
- Work API (`github.com/kubernetes-sigs/work-api`)
- Cluster Inventory KEP (KEP-4322)
- Reference implementations and conformance tests

GitHub: <https://github.com/kubernetes/community/tree/master/sig-multicluster>  
Docs: <https://multicluster.sigs.k8s.io/>

---

### sig-cluster-lifecycle

**SIG Cluster Lifecycle** owns **kubeadm** (the canonical cluster bootstrap tool), **Cluster API (CAPI)** (declarative infrastructure provisioning using provider plugins for bare-metal, AWS, GCP, Azure, etc.), and **ClusterClass** (templated cluster topologies). For AI infrastructure teams, CAPI enables reproducible provisioning of GPU node pools with specific NUMA topology, RDMA NIC, and SR-IOV configuration baked into `MachineDeployments`.

Key deliverables:
- kubeadm (`github.com/kubernetes/kubeadm`)
- Cluster API (`github.com/kubernetes-sigs/cluster-api`)
- ClusterClass and managed topology
- CAPI provider ecosystem (bare-metal via Metal3 — see Part 2)
- `clusterctl` CLI for provider management

GitHub: <https://github.com/kubernetes/community/tree/master/sig-cluster-lifecycle>  
Docs: <https://cluster-api.sigs.k8s.io/>

---

### wg-device-management

**WG Device Management** (a cross-SIG working group spanning SIG Node, SIG Scheduling, and SIG Network) owns the **Dynamic Resource Allocation (DRA)** API — the successor to the Device Plugin API for hardware resources that require structured parameters, pooled allocation, and topology constraints. DRA is critical for AI cluster infrastructure: a GPU node may expose `nvidia.com/gpu` devices via DRA with NUMA affinity, NVLink topology, and MIG profile selection as structured parameters, rather than the binary present/absent model of the Device Plugin API. **DRA graduated to GA in Kubernetes v1.34** (September 2025). The working group produced prototypes and experiments in `github.com/kubernetes-sigs/wg-device-management` before folding its results into core Kubernetes. WG Device Management continues to coordinate enhancements such as `DRAConsumableCapacity` (alpha in v1.34) for fine-grained device sharing across workloads.

Key deliverables:
- DRA API (`ResourceClaim`, `ResourceClaimTemplate`, `DeviceClass`) — GA in v1.34
- Structured parameters for GPU/NIC/FPGA allocation
- Topology-aware device allocation (NUMA, NVLink)
- `DRAConsumableCapacity` for fractional device sharing (alpha, v1.34)
- Extended resource mapping (DRA resources advertised as extended resources)

GitHub: <https://github.com/kubernetes-sigs/wg-device-management>  
Community: <https://github.com/kubernetes/community/blob/master/wg-device-management/README.md>

---

### wg-serving *(concluded February 2026)*

**WG Serving** was a cross-SIG working group chartered to standardize Kubernetes as a first-class inference serving platform, covering LLM serving APIs, disaggregated prefill/decode orchestration (Ch. 29, NVIDIA Dynamo), and inference gateway standardization. The working group **concluded in February 2026** after successfully establishing common understanding of inference workload requirements and seeding improvements across SIG Node (model loader sidecars), SIG Scheduling (topology-aware inference placement), SIG Network (inference gateway as a request scheduler), and WG Device Management (DRA for GPU slice allocation). Ongoing inference-serving work has transitioned to these SIGs. The charter and meeting notes remain archived for reference.

Key accomplishments:
- Defined inference gateway standardization (adopted by SIG Network)
- Established LLM serving API patterns across Kubernetes deployments
- Produced workload classification for prefill vs. decode phases
- Seeded agent networking standardization work in SIG Network
- Work transitioned to SIG Node, SIG Scheduling, SIG Network, WG Device Management

Charter (archived): <https://github.com/kubernetes/community/blob/master/wg-serving/charter.md>

---

## Part 2 — CNCF Projects Relevant to AI Cluster Infrastructure

The tables below group projects by infrastructure layer. The **Status** column uses CNCF's official maturity levels — **Graduated**, **Incubating**, **Sandbox** — or **Non-CNCF** for projects that are widely deployed in AI cluster infrastructure but governed outside CNCF (with the actual governance home noted in parentheses). Projects marked Non-CNCF are included because omitting them would leave a significant gap in the operational landscape described in this book.

---

### Networking

| Project | Status | Description | URL | Book Chapter |
|---|---|---|---|---|
| **Cilium** | Graduated | eBPF-native CNI with NetworkPolicy, kube-proxy replacement, BGP control plane, and Hubble observability. The primary CNI for AI cluster Kubernetes deployments in this book. | <https://github.com/cilium/cilium> | Ch. 12 |
| **Hubble** | Graduated (part of Cilium) | Per-flow observability layer built on Cilium's eBPF datapath; exposes forwarded/dropped flows via gRPC and a web UI. Ships as part of the Cilium project. | <https://github.com/cilium/hubble> | Ch. 12 |
| **Multus CNI** | Non-CNCF (k8snetworkplumbingwg) | Meta-CNI plugin enabling pods to attach multiple network interfaces via `NetworkAttachmentDefinition` CRDs. Enables GPU pods to combine a management CNI (Cilium) with a high-speed secondary NIC (SR-IOV/RDMA). | <https://github.com/k8snetworkplumbingwg/multus-cni> | Ch. 13 |
| **SR-IOV Network Operator** | Non-CNCF (k8snetworkplumbingwg) | Kubernetes operator that configures SR-IOV Virtual Functions on nodes and exposes them as device plugin resources. Automates VF provisioning for RDMA-capable GPU pods. | <https://github.com/k8snetworkplumbingwg/sriov-network-operator> | Ch. 13 |
| **NVIDIA Network Operator** | Non-CNCF (NVIDIA/Mellanox) | Helm-based Kubernetes operator that deploys and manages the full NVIDIA networking stack: MOFED drivers, RDMA device plugin, NV-IPAM, Multus, and SR-IOV operator. | <https://github.com/Mellanox/network-operator> | Ch. 13 |
| **Whereabouts IPAM** | Non-CNCF (k8snetworkplumbingwg) | Cluster-wide CNI IPAM plugin that allocates unique IP addresses for secondary interfaces across all nodes using etcd or Kubernetes CRDs as backend. | <https://github.com/k8snetworkplumbingwg/whereabouts> | Ch. 13 |
| **Submariner** | Sandbox | Cross-cluster L3 networking: establishes encrypted tunnels between clusters using IPsec or WireGuard, and federates Services using the MCS API (ServiceImport/ServiceExport). | <https://github.com/submariner-io/submariner> | Ch. — |
| **Antrea** | Sandbox | CNI plugin based on Open vSwitch and eBPF; supports NetworkPolicy, IPAM, ClusterNetworkPolicy, and Antrea-native multicast. Alternative to Cilium for OVS-based deployments. | <https://github.com/antrea-io/antrea> | Ch. 11 |
| **Kube-OVN** | Sandbox | CNI plugin using OVN/OVS as the data plane; supports L2/L3 topologies, VPC isolation, QoS, and IPAM with subnet CRDs. Common in telco and enterprise multi-tenant clusters. | <https://github.com/kubeovn/kube-ovn> | Ch. 11 |

---

### Observability

| Project | Status | Description | URL | Book Chapter |
|---|---|---|---|---|
| **Prometheus** | Graduated | Pull-based metrics collection and alerting. The standard for GPU node metrics (DCGM exporter), NIC counters, and network device scraping in AI cluster deployments. | <https://github.com/prometheus/prometheus> | Ch. 16 |
| **OpenTelemetry** | Incubating | Vendor-neutral telemetry SDK and collector for metrics, traces, and logs; integrates with Prometheus, Jaeger, and Grafana. Used for AI workload observability pipelines. | <https://github.com/open-telemetry/opentelemetry-collector> | Ch. 16 |
| **Jaeger** | Graduated | Distributed tracing system; receives OTLP traces from OpenTelemetry and provides trace search and dependency graphs for multi-service AI inference paths. | <https://github.com/jaegertracing/jaeger> | Ch. 16 |
| **Thanos** | Incubating | Highly available Prometheus with long-term storage; supports cross-cluster query federation — important when monitoring distributed GPU clusters. | <https://github.com/thanos-io/thanos> | Ch. 16 |
| **Grafana** | Non-CNCF (Grafana Labs) | Visualization platform for Prometheus, Loki, Tempo, and OpenTelemetry data. Grafana Labs is a CNCF Platinum member; Grafana itself is not a CNCF-hosted project. | <https://github.com/grafana/grafana> | Ch. 16 |
| **VictoriaMetrics** | Non-CNCF (independent) | High-performance time-series database compatible with the Prometheus query API; widely used as a drop-in Prometheus replacement for large AI cluster deployments. Not a CNCF project. | <https://github.com/VictoriaMetrics/VictoriaMetrics> | Ch. 16 |
| **Pixie** | Sandbox | Auto-instrumentation observability for Kubernetes using eBPF; captures pod-level network flows, latency, and errors without code changes. Useful for debugging NCCL connectivity. | <https://github.com/pixie-io/pixie> | Ch. 7, 12 |

---

### Storage

| Project | Status | Description | URL | Book Chapter |
|---|---|---|---|---|
| **Rook (Ceph)** | Graduated | Kubernetes operator for Ceph distributed storage; exposes RBD block volumes and CephFS shared filesystems as CSI-backed PVCs for AI checkpoint I/O. | <https://github.com/rook/rook> | Ch. 18 |
| **Longhorn** | Incubating | Lightweight distributed block storage built natively for Kubernetes; replicates volumes across nodes and provides snapshot/backup support for training checkpoints. | <https://github.com/longhorn/longhorn> | Ch. 18 |
| **OpenEBS** | Sandbox | Container-attached storage (CAS) platform; provides Local PV (fastest, for GPU-local scratch), Replicated PV (Mayastor/NVMe-oF backend), and legacy backends. Re-accepted as Sandbox in October 2024 after significant architectural revision. | <https://github.com/openebs/openebs> | Ch. 18 |

---

### Security

| Project | Status | Description | URL | Book Chapter |
|---|---|---|---|---|
| **SPIFFE** | Graduated | Specification for workload identity (SVID — SPIFFE Verifiable Identity Document) using X.509 certificates or JWTs; the identity substrate for zero-trust AI clusters. | <https://github.com/spiffe/spiffe> | Ch. 26 |
| **SPIRE** | Graduated | SPIFFE runtime implementation; issues SVIDs to workloads via node attestation and workload attestation. Used with Cilium transparent encryption and mTLS between GPU pods. | <https://github.com/spiffe/spire> | Ch. 26 |
| **Falco** | Graduated | Runtime security using eBPF to detect anomalous syscalls, container escapes, and unexpected network connections in real time across GPU nodes. | <https://github.com/falcosecurity/falco> | Ch. 26 |
| **OPA / Gatekeeper** | Graduated | Open Policy Agent policy engine; Gatekeeper is the Kubernetes admission controller implementation. Enforces deployment constraints (e.g., only signed images, required resource limits) for GPU workloads. | <https://github.com/open-policy-agent/gatekeeper> | Ch. 26 |
| **cert-manager** | Graduated | Kubernetes-native certificate lifecycle manager; integrates with Let's Encrypt, Vault, and internal CAs. Provides TLS certificates for management APIs and mTLS between cluster services. | <https://github.com/cert-manager/cert-manager> | Ch. 26 |
| **Keycloak** | Incubating | Identity and Access Management platform; provides OIDC/OAuth2 for Kubernetes API authentication and AI platform user management. | <https://github.com/keycloak/keycloak> | Ch. 26 |

---

### Scheduling & Workload

| Project | Status | Description | URL | Book Chapter |
|---|---|---|---|---|
| **Volcano** | Incubating | Kubernetes batch scheduling system with gang scheduling, queue-based fairness, and HPC job semantics (`Job`, `Queue`, `PodGroup` CRDs); the standard choice for distributed training job orchestration. | <https://github.com/volcano-sh/volcano> | Ch. 29 |
| **Kueue** | Non-CNCF (kubernetes-sigs subproject) | Job queueing and admission control for AI/ML batch workloads; manages `ResourceFlavor`, `ClusterQueue`, and `LocalQueue` to enforce quota and fairness across teams sharing GPU pools. A `kubernetes-sigs` subproject, not a separately CNCF-hosted project. | <https://github.com/kubernetes-sigs/kueue> | Ch. 29 |
| **Argo Workflows** | Graduated (part of Argo) | Workflow engine for Kubernetes; used for multi-step ML pipelines (data prep → training → evaluation → export). Argo project (Argo Workflows, Argo CD, Argo Rollouts, Argo Events) graduated December 2022. | <https://github.com/argoproj/argo-workflows> | Ch. — |
| **Argo CD** | Graduated (part of Argo) | GitOps continuous delivery controller; syncs Kubernetes cluster state from Git repositories. Used to deploy AI infrastructure manifests (Volcano, GPU operator, network operator) declaratively. | <https://github.com/argoproj/argo-cd> | Ch. — |
| **Flux** | Graduated | GitOps toolkit for Kubernetes; source controller, Helm controller, Kustomize controller. Alternative to Argo CD for declarative AI cluster configuration management. | <https://github.com/fluxcd/flux2> | Ch. — |
| **KubeVirt** | Incubating | Extends Kubernetes to manage virtual machines alongside containers using the KVM hypervisor. Enables GPU passthrough (VFIO-PCI) and SR-IOV VF assignment to VMs. | <https://github.com/kubevirt/kubevirt> | Ch. 31 |
| **Liqo** | Non-CNCF (independent, liqotech) | Dynamic multi-cluster workload offloading; transparently schedules pods from one cluster to another over encrypted tunnels, implementing the MCS API. Featured in CNCF blog content. | <https://github.com/liqotech/liqo> | Ch. — |

---

### Runtimes

| Project | Status | Description | URL | Book Chapter |
|---|---|---|---|---|
| **containerd** | Graduated | Industry-standard container runtime; manages container lifecycle via CRI. The default runtime for Kubernetes GPU nodes; containerd shim v2 API enables custom runtimes via `RuntimeClass`. | <https://github.com/containerd/containerd> | Ch. 32 |
| **CRI-O** | Graduated | Lightweight CRI implementation optimized for Kubernetes; alternative to containerd used in OpenShift and some HPC deployments. | <https://github.com/cri-o/cri-o> | Ch. 32 |
| **Kata Containers** | Non-CNCF (Open Infrastructure Foundation) | Secure container runtime using hardware virtualization (KVM/QEMU) for strong workload isolation; relevant for multi-tenant GPU cluster security. Not a CNCF project — governed by the Open Infrastructure Foundation. | <https://github.com/kata-containers/kata-containers> | Ch. 31 |
| **gVisor** | Non-CNCF (Google, Apache 2.0) | User-space kernel sandbox that intercepts syscalls to isolate container workloads; lower isolation overhead than Kata, but limited GPU passthrough support. Not a CNCF project. | <https://github.com/google/gvisor> | Ch. 31 |
| **WasmEdge Runtime** | Sandbox | WebAssembly runtime for cloud-native environments; enables lightweight, sandboxed inference serving at the edge. CNCF Sandbox since April 2021. | <https://github.com/WasmEdge/WasmEdge> | Ch. — |

---

### Service Mesh & Ingress

| Project | Status | Description | URL | Book Chapter |
|---|---|---|---|---|
| **Envoy** | Graduated | High-performance L7 proxy and data plane; the engine behind Istio, Contour, and many inference gateway implementations. Supports gRPC, HTTP/2, and xDS control plane protocol. | <https://github.com/envoyproxy/envoy> | Ch. 29 |
| **Contour** | Incubating | Kubernetes ingress controller built on Envoy; implements Gateway API and HTTPProxy CRDs for AI serving endpoint exposure. | <https://github.com/projectcontour/contour> | Ch. — |
| **Istio** | Graduated | Service mesh providing mTLS, traffic management, and observability across microservices; applicable to AI inference serving clusters where sidecar overhead is acceptable. | <https://github.com/istio/istio> | Ch. 26 |
| **Linkerd** | Graduated | Lightweight service mesh using a Rust-based micro-proxy; lower overhead than Istio, suitable for latency-sensitive inference paths. | <https://github.com/linkerd/linkerd2> | Ch. 26 |
| **NGINX Gateway Fabric** | Non-CNCF (F5/NGINX) | Gateway API implementation using NGINX as the data plane. Not a CNCF-hosted project; maintained by F5/NGINX. Implements `GatewayClass`, `HTTPRoute`, and `GRPCRoute` for AI model serving endpoints. | <https://github.com/nginx/nginx-gateway-fabric> | Ch. 32 |

---

### AI/ML Infrastructure

| Project | Status | Description | URL | Book Chapter |
|---|---|---|---|---|
| **KServe** | Incubating | Kubernetes-native model serving framework supporting multi-framework inference (TensorFlow, PyTorch, ONNX, Triton) with autoscaling and canary deployments. Accepted as CNCF Incubating in November 2025. | <https://github.com/kserve/kserve> | Ch. 29 |
| **BentoML** | Non-CNCF (independent) | Model packaging and serving framework supporting multiple runtimes; received "Adopt" rating in CNCF Technology Radar (Q4 2025) for inference serving, but is not a CNCF-hosted project. | <https://github.com/bentoml/BentoML> | Ch. — |
| **Ray** | Non-CNCF (PyTorch Foundation) | Distributed computing framework for AI workloads: data processing, model training, and inference serving. Joined the PyTorch Foundation in 2025. Not a CNCF project; referenced throughout Ch. 20 for its fabric demands. | <https://github.com/ray-project/ray> | Ch. 20 |
| **MLflow** | Non-CNCF (LF AI & Data Foundation) | ML experiment tracking, model registry, and deployment platform. Governed by the Linux Foundation AI & Data Foundation, not CNCF. Widely used alongside Kubernetes-native scheduling. | <https://github.com/mlflow/mlflow> | Ch. — |

---

## Quick Cross-Reference: Projects by Book Chapter

| Chapter | Projects Covered |
|---|---|
| Ch. 7 — eBPF & XDP | Pixie (eBPF-based observability) |
| Ch. 11 — Overlay Networks | Antrea, Kube-OVN |
| Ch. 12 — Cilium | Cilium (Graduated), Hubble |
| Ch. 13 — Multus & SR-IOV | Multus CNI, SR-IOV Network Operator, NVIDIA Network Operator, Whereabouts |
| Ch. 16 — Observability | Prometheus, OpenTelemetry, Jaeger, Thanos, Grafana, VictoriaMetrics |
| Ch. 18 — Distributed Storage | Rook/Ceph, Longhorn, OpenEBS |
| Ch. 20 — Training Runtimes | Ray |
| Ch. 26 — Network Security | SPIFFE, SPIRE, Falco, OPA/Gatekeeper, cert-manager, Keycloak, Istio, Linkerd |
| Ch. 29 — AI Scheduling | Volcano, Kueue, KServe, Envoy (inference gateway) |
| Ch. 31 — GPU Virtualization | KubeVirt, Kata Containers, gVisor |
| Ch. 32 — Extending Kubernetes | containerd, CRI-O, Gateway API (sig-network), NGINX Gateway Fabric |

---

## Kubernetes SIG & Working Group Resources

| Resource | URL |
|---|---|
| Kubernetes Community GitHub | <https://github.com/kubernetes/community> |
| Kubernetes Enhancements (KEPs) | <https://github.com/kubernetes/enhancements> |
| SIG Index | <https://github.com/kubernetes/community/blob/master/sig-list.md> |
| CNCF Project Landscape | <https://landscape.cncf.io/> |
| CNCF Graduated & Incubating Projects | <https://www.cncf.io/projects/> |
| CNCF Sandbox Projects | <https://www.cncf.io/sandbox-projects/> |
| CNCF Technology Radar | <https://www.cncf.io/radar/> |
| DRA v1.34 Release Notes | <https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/> |
| WG Serving Conclusion Announcement | <https://www.cncf.io/blog/2026/02/26/kubernetes-wg-serving-concludes-following-successful-advancement-of-ai-inference-support/> |

---

*© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
