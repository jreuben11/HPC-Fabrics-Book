# Chapter 32 — Extending Kubernetes for AI Infrastructure: CNI, Device Plugins, Operators & Gateway API

**Part VI: Platform Engineering** | ~25 pages

---

## Introduction

**Kubernetes** was designed as an extensible platform, not a fixed product. Its designers anticipated that operators, CNI plugins, storage drivers, hardware accelerators, and custom routing controllers would all need first-class integration — not hacks. The result is a collection of well-defined extension points: binary API contracts for network attachment (**CNI**), hardware device exposure (**Device Plugin API**), block and file storage (**CSI**), runtime shim selection (**CRI**), declarative application controllers (**Operators**), and structured L4/L7 ingress (**Gateway API**). Together these extension points define the seams through which AI infrastructure — GPUs, InfiniBand NICs, RDMA device plugins, custom schedulers, and inference serving endpoints — is integrated into the Kubernetes control plane.

This chapter provides a deep technical walkthrough of the extension points most relevant to AI cluster networking, with particular emphasis on the four that require custom implementation work: **CNI** plugins (network attachment), the **Device Plugin API** (hardware exposure), **Operator SDK** / **controller-runtime** (control-plane automation), and **Gateway API** (structured ingress/egress routing). **CRI** and **CSI** receive shorter orientation treatments, as their AI relevance is covered in the storage chapters. The chapter concludes with a design walkthrough of a `RDMANetwork` operator that ties all four active extension points together, and a lab that exercises the complete stack on a local **Kind** cluster.

Readers of this chapter should already be comfortable with the CNI and SR-IOV mechanics covered in Chapter 13 (Multus, SR-IOV, NVIDIA Network Operator), the Cilium CNI model from Chapter 12, and the Kubernetes AI scheduling infrastructure from Chapter 29 (Volcano, Kueue, topology-aware scheduling). Cross-references appear throughout using *(see Chapter N)* notation.

---

## Installation

### Go toolchain

```bash
go version
# go version go1.22.x linux/amd64
# Go 1.21+ required for controller-runtime v0.17+
```

### Operator SDK

```bash
export OPERATOR_SDK_VERSION=v1.37.0
curl -Lo /tmp/operator-sdk \
  https://github.com/operator-framework/operator-sdk/releases/download/${OPERATOR_SDK_VERSION}/operator-sdk_linux_amd64
chmod +x /tmp/operator-sdk
sudo mv /tmp/operator-sdk /usr/local/bin/operator-sdk
operator-sdk version
# operator-sdk version: "v1.37.0", commit: "...", go version: "go1.22.x"
```

### controller-gen (CRD manifest generator)

```bash
go install sigs.k8s.io/controller-tools/cmd/controller-gen@latest
controller-gen --version
# Version: v0.16.x
```

### Gateway API CRDs

```bash
# Install standard-channel CRDs (GatewayClass, Gateway, HTTPRoute, GRPCRoute, ReferenceGrant)
kubectl apply -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
# Install experimental-channel CRDs (TCPRoute, TLSRoute)
kubectl apply -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/experimental-install.yaml
```

### Kind cluster with Cilium

```bash
kind create cluster --name ai-ext --config - <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: true   # let Cilium take sole ownership
  podSubnet: "10.244.0.0/16"
EOF

# Install Cilium with Gateway API support
# gatewayAPI.hostNetwork.enabled=true makes the Gateway listener bind directly
# on the node IP instead of requiring a LoadBalancer Service — required on Kind
# where no cloud provider / MetalLB is present to assign an external IP.
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.16.x \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set gatewayAPI.enabled=true \
  --set gatewayAPI.hostNetwork.enabled=true \
  --set l7Proxy=true
```

---

## 32.1 Kubernetes Extension Points — A One-Page Map

Before diving into each API, it helps to see them side by side. Table 32.1 shows the six primary extension seams, the interface mechanism, the upstream specification, and which chapters of this book cover the implementation details.

**Table 32.1 — Kubernetes Extension Points**

| Extension Point | Interface Mechanism | Upstream Spec / API Group | Current Stable Version | Book Coverage |
|---|---|---|---|---|
| **CNI** (Container Network Interface) | Binary + stdin/stdout JSON | `github.com/containernetworking/cni` SPEC.md | 1.1.0 | Ch12 (Cilium), Ch13 (Multus), §32.2 |
| **Device Plugin API** | gRPC over Unix socket | `k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1` | v1beta1 | Ch13 (NVIDIA plugin), §32.3 |
| **CRI** (Container Runtime Interface) | gRPC — `RuntimeService` | `k8s.io/cri-api/pkg/apis/runtime/v1` | v1 | §32.6 |
| **CSI** (Container Storage Interface) | gRPC — `NodeServer` | `github.com/container-storage-interface/spec` | v1.10 | Ch6 (NVMe-oF), Ch18 (Ceph/Lustre), §32.6 |
| **Operator / controller-runtime** | In-cluster controller watching CRDs | `sigs.k8s.io/controller-runtime` | v0.19.x | §32.4, §32.7 |
| **Gateway API** | CRDs in `gateway.networking.k8s.io` | `sigs.k8s.io/gateway-api` | v1.2.0 | Ch12 (Cilium impl.), §32.5 |

**This chapter owns:** CNI spec depth (§32.2), Device Plugin API (§32.3), Operator SDK / controller-runtime (§32.4), Gateway API (§32.5), brief CRI/CSI orientation (§32.6), and the integrating `RDMANetwork` operator walkthrough (§32.7).

The key insight is that CNI, Device Plugin, CRI, and CSI are all **synchronous call-response protocols** invoked by the kubelet during pod lifecycle events — they must complete quickly and deterministically. Operators are **asynchronous reconcilers** that run as long-lived controllers watching for state drift. Gateway API occupies a different plane entirely: it is a declarative configuration API consumed by a gateway controller (such as Cilium), not a per-pod lifecycle hook.

---

## 32.2 CNI Spec in Depth

### Why CNI Uses a Binary Protocol

**CNI (Container Network Interface)** was deliberately designed around UNIX process invocation rather than gRPC. The kubelet executes a CNI plugin binary from a well-known path (`/opt/cni/bin/`), passes configuration and environment variables, reads JSON from stdout, and exits. This design makes CNI plugins easy to write in any language, easy to inspect with `strace`, and completely independent of the kubelet's own gRPC infrastructure. It does, however, make chaining and error propagation more complex than a long-lived gRPC service would be.

The current CNI specification is **version 1.1.0**, defined in the `SPEC.md` file of the `containernetworking/cni` repository. Version 1.1.0 added the `GC` (garbage collect) and `STATUS` commands to the 1.0 baseline.

### 32.2.1 Command Lifecycle: ADD / DEL / CHECK / GC / STATUS

The CNI_COMMAND environment variable selects the operation. The kubelet passes:

| Variable | Meaning |
|---|---|
| `CNI_COMMAND` | `ADD`, `DEL`, `CHECK`, `GC`, or `STATUS` |
| `CNI_CONTAINERID` | Opaque string identifying the sandbox |
| `CNI_NETNS` | Path to the network namespace (`/proc/<pid>/ns/net`) |
| `CNI_IFNAME` | Interface name to create inside the container (e.g., `eth0`) |
| `CNI_ARGS` | Semicolon-separated key=value pairs from the runtime |
| `CNI_PATH` | Colon-separated directories to search for plugin binaries |

**ADD**: Create the interface inside the container namespace, configure IP addressing (possibly via an IPAM sub-plugin), install routes, and return a result JSON object on stdout.

**DEL**: Tear down the network attachment. DEL must be idempotent — if the attachment was never created or was already deleted, DEL must return success.

**CHECK**: Verify that the container's network state still matches what ADD configured. A plugin that finds stale state (e.g., a missing ARP entry) should return an error. CHECK is called in the same forward order as ADD, and each plugin receives the `prevResult` from the preceding plugin.

**GC** (1.1.0+): Release any state the plugin holds for container IDs that are no longer active. Called with a `validAttachments` list so plugins can compare against their local state and prune orphaned entries.

**STATUS** (1.1.0+): A lightweight healthcheck — does the plugin believe it can satisfy new ADD requests? Returns success or an error with a human-readable message.

### 32.2.2 CNI Result JSON Schema

A successful ADD returns a JSON object on stdout:

```json
{
  "cniVersion": "1.1.0",
  "interfaces": [
    {
      "name": "eth0",
      "mac": "0a:58:0a:f4:00:06",
      "mtu": 1500,
      "sandbox": "/proc/12345/ns/net"
    }
  ],
  "ips": [
    {
      "address": "10.244.0.6/24",
      "gateway": "10.244.0.1",
      "interface": 0
    }
  ],
  "routes": [
    { "dst": "0.0.0.0/0", "gw": "10.244.0.1" }
  ],
  "dns": {
    "nameservers": ["10.96.0.10"],
    "search": ["cluster.local"]
  }
}
```

The `interface` field in each IP entry is an index into the `interfaces` array, allowing IPAM results to be mapped to specific physical interfaces.

### 32.2.3 IPAM Delegation

Rather than implementing IP address management directly, a CNI plugin delegates to a separate **IPAM** sub-plugin by invoking it with `CNI_COMMAND=ADD` and passing the full network configuration as stdin. The IPAM plugin returns an abbreviated result — containing `ips`, `routes`, and `dns` but no `interfaces` array. The parent plugin merges this IPAM output with its own interface configuration to produce the final result.

Standard IPAM plugins include `host-local` (per-node subnet from a local file), `dhcp` (leases from a DHCP server), and **Whereabouts** (cluster-wide IP allocation backed by etcd or Kubernetes leases *(see Chapter 13)*).

### 32.2.4 Plugin Chaining via conflist

A `conflist` (configuration list) allows multiple CNI plugins to run in sequence, each receiving the previous plugin's result as `prevResult`:

```json
{
  "cniVersion": "1.1.0",
  "name": "ai-node-net",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "ipam": {
        "type": "host-local",
        "subnet": "10.244.0.0/24"
      }
    },
    {
      "type": "portmap",
      "capabilities": { "portMappings": true }
    },
    {
      "type": "bandwidth",
      "ingressRate": 100000000,
      "egressRate": 100000000
    }
  ]
}
```

**Multus** *(see Chapter 13)* implements chaining by reading the `NetworkAttachmentDefinition` CRD and delegating to secondary plugins in order, carrying forward the `prevResult` between each call. **Cilium** *(see Chapter 12)* operates as a standalone plugin rather than a chained plugin — it handles the full datapath via eBPF and does not rely on conflist chaining for its primary interface.

### 32.2.5 Writing a Minimal CNI Plugin in Go

The `github.com/containernetworking/cni/pkg/skel` package provides the entry point skeleton. As of v1.3.0 (April 2025), the preferred API uses `skel.PluginMainFuncs` with the `CNIFuncs` struct (older `PluginMain` with three separate function arguments is deprecated). Use `github.com/containernetworking/plugins` for the `types` and `ipam` helper packages.

```go
// Module: github.com/example/minimal-cni-plugin
// go get github.com/containernetworking/cni@v1.3.0
// go get github.com/containernetworking/plugins@v1.5.0

package main

import (
    "encoding/json"
    "fmt"
    "net"

    "github.com/containernetworking/cni/pkg/skel"
    "github.com/containernetworking/cni/pkg/types"
    current "github.com/containernetworking/cni/pkg/types/100" // CNI result v1.0.0+ schema
    "github.com/containernetworking/cni/pkg/version"
)

// NetConf is the plugin's network configuration, unmarshalled from stdin.
type NetConf struct {
    types.NetConf               // embeds CNIVersion, Name, Type, IPAM, DNS
    ExampleParam  string `json:"exampleParam"`
}

func parseConf(data []byte) (*NetConf, error) {
    conf := &NetConf{}
    if err := json.Unmarshal(data, conf); err != nil {
        return nil, fmt.Errorf("parsing netconf: %w", err)
    }
    return conf, nil
}

// cmdAdd is called by the kubelet to attach the container to the network.
func cmdAdd(args *skel.CmdArgs) error {
    conf, err := parseConf(args.StdinData)
    if err != nil {
        return err
    }
    _ = conf // use conf.ExampleParam, conf.IPAM, etc.

    // --- IPAM delegation ---
    // In a real plugin: exec the IPAM plugin, get back ips/routes/dns.
    // Here we synthesize a static result for illustration.
    result := &current.Result{
        CNIVersion: conf.CNIVersion,
        Interfaces: []*current.Interface{
            {
                Name:    args.IfName,
                Mac:     "0a:58:0a:f4:00:01",
                Sandbox: args.Netns,
            },
        },
        IPs: []*current.IPConfig{
            {
                Interface: current.Int(0),
                Address:   net.IPNet{IP: net.ParseIP("10.244.1.5"), Mask: net.CIDRMask(24, 32)},
                Gateway:   net.ParseIP("10.244.1.1"),
            },
        },
    }
    return types.PrintResult(result, conf.CNIVersion)
}

// cmdDel is called when the pod is deleted. Must be idempotent.
func cmdDel(args *skel.CmdArgs) error {
    // Clean up interfaces, routes, IPAM leases.
    // Return nil even if the attachment never existed.
    return nil
}

// cmdCheck verifies that the container's network state matches the ADD result.
func cmdCheck(args *skel.CmdArgs) error {
    // Verify interface exists in args.Netns, IP is configured, routes present.
    return nil
}

func main() {
    // PluginMainFuncs is the modern (v1.2.0+) entry point; the older
    // PluginMain(cmdAdd, cmdCheck, cmdDel, ...) signature is deprecated.
    skel.PluginMainFuncs(
        skel.CNIFuncs{
            Add:   cmdAdd,
            Del:   cmdDel,
            Check: cmdCheck,
            // GC and Status are optional; omit for minimal plugins
        },
        version.PluginSupports("1.0.0", "1.1.0"),
        "minimal-cni-plugin v0.1.0",
    )
}
```

Build and install:

```bash
go build -o minimal-cni-plugin .
sudo cp minimal-cni-plugin /opt/cni/bin/
```

Write a `conflist` referencing this plugin as its `type`, and the kubelet will invoke it for any pod scheduled on the node.

---

## 32.3 Device Plugin API

### Why a Separate API for Hardware

**Kubernetes** resource accounting — CPU, memory — is built into the core scheduler. But custom hardware (GPUs, FPGAs, InfiniBand HCAs, SR-IOV Virtual Functions) cannot be represented as CPU millicores. The **Device Plugin API** provides a generic gRPC contract through which hardware vendors advertise resources to the kubelet using fully-qualified resource names (e.g., `nvidia.com/gpu`, `rdma/hca_shared_devices_a`). The kubelet then exposes these resources to the scheduler as allocatable capacity, and the device plugin is consulted at pod admission time to perform any vendor-specific allocation work (configuring cgroups, binding device files, setting up CDI entries).

### 32.3.1 Registration Protocol

A device plugin follows this lifecycle:

1. Start a gRPC server listening on a Unix socket at `/var/lib/kubelet/device-plugins/<plugin-name>.sock`.
2. Call `RegisterRequest` on the kubelet's own registration socket at `/var/lib/kubelet/device-plugins/kubelet.sock`, advertising the socket path, API version, and resource name.
3. The kubelet connects back to the plugin's socket and calls `GetDevicePluginOptions`, then starts a streaming `ListAndWatch` call.
4. The plugin streams device lists via `ListAndWatch` for the lifetime of the connection.
5. When a pod requesting the resource is scheduled, the kubelet calls `Allocate` on the plugin.

The current stable API is **`k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1`** (the API has been stable at v1beta1 since Kubernetes 1.10; promotion to v1 has been in progress but not yet complete as of Kubernetes 1.31).

### 32.3.2 gRPC Service Definition

The `DevicePluginServer` interface a plugin must implement:

```go
// Package: k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1
type DevicePluginServer interface {
    // Return plugin capabilities to the kubelet.
    GetDevicePluginOptions(context.Context, *Empty) (*DevicePluginOptions, error)

    // Stream device list updates. Called once; must block and send updates
    // whenever device health changes. The kubelet reconnects on stream failure.
    ListAndWatch(*Empty, DevicePlugin_ListAndWatchServer) error

    // Called before container creation. Optional; only if GetDevicePluginOptions
    // returns PreStartRequired: true.
    PreStartContainer(context.Context, *PreStartContainerRequest) (*PreStartContainerResponse, error)

    // Return a preferred subset of the requested devices for locality-aware
    // scheduling (e.g., prefer GPUs on the same NUMA node).
    GetPreferredAllocation(context.Context, *PreferredAllocationRequest) (*PreferredAllocationResponse, error)

    // Perform vendor-specific allocation work and return environment variables,
    // host mounts, and device file paths to inject into the container.
    Allocate(context.Context, *AllocateRequest) (*AllocateResponse, error)
}
```

### 32.3.3 How `nvidia-device-plugin` Exposes `nvidia.com/gpu`

The **NVIDIA device plugin** (`github.com/NVIDIA/k8s-device-plugin`) *(see Chapter 13)* uses **NVML** (NVIDIA Management Library) to discover GPUs and registers them under the resource name `nvidia.com/gpu`. Its `ListAndWatch` implementation:

1. Calls `nvml.DeviceGetCount()` to enumerate physical GPUs.
2. Constructs a `Device` for each: `ID` = UUID string (e.g., `GPU-12345678-...`), `Health = "Healthy"`.
3. Starts a goroutine watching NVML events for Double-Bit ECC errors and Xid faults.
4. On a health event, marks the affected device `"Unhealthy"` and sends a new `ListAndWatchResponse` stream update.

The `Allocate` response for each GPU sets:
- `Envs["NVIDIA_VISIBLE_DEVICES"]` to the GPU UUID
- A `DeviceSpec` for `/dev/nvidiaX` (character device, permissions `"rw"`)
- A `DeviceSpec` for `/dev/nvidia-uvm`, `/dev/nvidia-uvm-tools`, `/dev/nvidiactl`

Optionally, with Container Device Interface (**CDI**) enabled, the response includes a `CDIDevice` entry (`nvidia.com/gpu=<uuid>`) instead of raw device paths, letting the container runtime resolve the full device spec from the CDI registry.

### 32.3.4 How `rdma-shared-device-plugin` Exposes RDMA Resources

The **rdma-shared-device-plugin** (`github.com/Mellanox/k8s-rdma-shared-dev-plugin`) *(see Chapter 13)* uses a JSON configuration file to define resource pools. The resource name format is `{resourcePrefix}/{resourceName}`, where both components are user-configurable:

```json
{
  "periodicUpdateInterval": 300,
  "configList": [
    {
      "resourceName": "hca_shared_devices_a",
      "resourcePrefix": "rdma",
      "rdmaHcaMax": 1000,
      "devices": ["ib0", "ib1"]
    }
  ]
}
```

This exposes `rdma/hca_shared_devices_a` as an allocatable resource. Unlike the NVIDIA plugin where each device maps to a unique hardware unit, the RDMA shared plugin allows multiple pods to share the same HCA — up to `rdmaHcaMax` simultaneous allocations — by using the kernel RDMA character device (`/dev/infiniband/uverbsX`) and relying on the hardware's own isolation mechanisms.

### 32.3.5 Writing a Stub Device Plugin

The following stub demonstrates the gRPC server structure without requiring real hardware. It advertises a fake `example.com/widget` resource.

```go
// Module: github.com/example/stub-device-plugin
// go get k8s.io/kubelet@v0.31.0
// go get google.golang.org/grpc@v1.64.0

package main

import (
    "context"
    "fmt"
    "log"
    "net"
    "os"
    "path/filepath"
    "strings"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    pluginapi "k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1"
)

const (
    resourceName   = "example.com/widget"
    socketName     = "widget-plugin.sock"
    kubeletSocket  = pluginapi.KubeletSocket // /var/lib/kubelet/device-plugins/kubelet.sock
    pluginDir      = pluginapi.DevicePluginPath
)

// StubPlugin implements pluginapi.DevicePluginServer.
type StubPlugin struct {
    pluginapi.UnimplementedDevicePluginServer
    devices []*pluginapi.Device
}

func newStubPlugin() *StubPlugin {
    // Advertise four synthetic "widget" devices.
    devices := make([]*pluginapi.Device, 4)
    for i := range devices {
        devices[i] = &pluginapi.Device{
            ID:     fmt.Sprintf("widget-%d", i),
            Health: pluginapi.Healthy,
        }
    }
    return &StubPlugin{devices: devices}
}

func (p *StubPlugin) GetDevicePluginOptions(_ context.Context, _ *pluginapi.Empty) (*pluginapi.DevicePluginOptions, error) {
    return &pluginapi.DevicePluginOptions{
        PreStartRequired:                false,
        GetPreferredAllocationAvailable: false,
    }, nil
}

// ListAndWatch streams the device list; blocks until the context is cancelled.
func (p *StubPlugin) ListAndWatch(_ *pluginapi.Empty, stream pluginapi.DevicePlugin_ListAndWatchServer) error {
    // Send initial list.
    if err := stream.Send(&pluginapi.ListAndWatchResponse{Devices: p.devices}); err != nil {
        return err
    }
    // Block; a real plugin would watch for health changes here.
    <-stream.Context().Done()
    return nil
}

// Allocate is called per-container; return env vars and device paths.
func (p *StubPlugin) Allocate(_ context.Context, req *pluginapi.AllocateRequest) (*pluginapi.AllocateResponse, error) {
    responses := make([]*pluginapi.ContainerAllocateResponse, len(req.ContainerRequests))
    for i, cr := range req.ContainerRequests {
        responses[i] = &pluginapi.ContainerAllocateResponse{
            Envs: map[string]string{
                "WIDGET_IDS": strings.Join(cr.DevicesIds, ","),
            },
        }
    }
    return &pluginapi.AllocateResponse{ContainerResponses: responses}, nil
}

func register(socketPath string) error {
    conn, err := grpc.NewClient(
        kubeletSocket,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithContextDialer(func(ctx context.Context, addr string) (net.Conn, error) {
            return (&net.Dialer{}).DialContext(ctx, "unix", addr)
        }),
    )
    if err != nil {
        return err
    }
    defer conn.Close()
    client := pluginapi.NewRegistrationClient(conn)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    _, err = client.Register(ctx, &pluginapi.RegisterRequest{
        Version:      pluginapi.Version,
        Endpoint:     filepath.Base(socketPath),
        ResourceName: resourceName,
    })
    return err
}

func main() {
    socketPath := filepath.Join(pluginDir, socketName)
    os.Remove(socketPath) // clean up stale socket

    lis, err := net.Listen("unix", socketPath)
    if err != nil {
        log.Fatalf("listen: %v", err)
    }
    srv := grpc.NewServer()
    pluginapi.RegisterDevicePluginServer(srv, newStubPlugin())
    go srv.Serve(lis)

    if err := register(socketPath); err != nil {
        log.Fatalf("register: %v", err)
    }
    log.Printf("stub device plugin registered: %s", resourceName)
    select {} // block forever; handle SIGTERM in production
}
```

Request the fake resource in a pod spec:

```yaml
resources:
  limits:
    example.com/widget: "1"
```

The kubelet will call `Allocate` and inject `WIDGET_IDS` into the container environment.

---

## 32.4 Operator SDK / controller-runtime

### Why Operators Exist

Kubernetes CRUD operations — `kubectl apply`, `kubectl delete` — are sufficient for stateless applications. But AI infrastructure components (GPU device plugins, SR-IOV Virtual Function pools, network topology databases) have operational logic that requires ongoing reconciliation: if a node is added, VFs must be provisioned; if a NetworkAttachmentDefinition is deleted, the device plugin config must be updated. **Operators** encode this operational knowledge in a controller that watches Kubernetes resources and continuously reconciles observed state to desired state.

The **Operator SDK** (from Red Hat / Operator Framework) scaffolds Go operators using **controller-runtime** (`sigs.k8s.io/controller-runtime`), which provides the reconcile loop, client caching, leader election, and metrics instrumentation. The SDK adds code generation, OLM packaging, and scorecard testing on top.

### 32.4.1 Scaffolding a New Operator

Choose a consistent API group for all resources in this chapter's examples: `net.aicluster.io`.

```bash
mkdir network-topology-operator && cd network-topology-operator

# The API group is composed as: <--group>.<--domain>
# Using --domain aicluster.io and --group net produces net.aicluster.io/v1alpha1
operator-sdk init \
  --domain aicluster.io \
  --repo github.com/example/network-topology-operator

operator-sdk create api \
  --group net \
  --version v1alpha1 \
  --kind NetworkTopology \
  --resource \
  --controller
```

This generates:
- `api/v1alpha1/networktopology_types.go` — CRD struct with kubebuilder markers
- `internal/controller/networktopology_controller.go` — reconcile loop stub
- `config/crd/bases/` — kustomize bases, populated by `make manifests`
- `config/rbac/` — ClusterRole with RBAC markers derived from controller

### 32.4.2 CRD Type Definition with Kubebuilder Markers

**controller-gen** reads kubebuilder marker annotations embedded as comments to generate CRD OpenAPI schemas and RBAC rules:

```go
// api/v1alpha1/networktopology_types.go
package v1alpha1

import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

// NetworkTopologySpec describes the desired rack-to-node affinity mapping.
type NetworkTopologySpec struct {
    // Rack is the physical rack label this topology entry describes.
    // +kubebuilder:validation:MinLength=1
    // +kubebuilder:validation:MaxLength=63
    Rack string `json:"rack"`

    // Nodes lists the Kubernetes node names in this rack.
    // +kubebuilder:validation:MinItems=1
    Nodes []string `json:"nodes"`

    // BandwidthGbps is the intra-rack bisection bandwidth.
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=3200
    // +optional
    BandwidthGbps int32 `json:"bandwidthGbps,omitempty"`
}

// NetworkTopologyStatus reports observed rack membership as derived from node labels.
type NetworkTopologyStatus struct {
    // Conditions holds the latest available observations of the topology state.
    // +patchMergeKey=type
    // +patchStrategy=merge
    // +listType=map
    // +listMapKey=type
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // ObservedGeneration is the generation from spec the controller last reconciled.
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`
}

// NetworkTopology is the Schema for the networktopologies API.
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Rack",type=string,JSONPath=`.spec.rack`
// +kubebuilder:printcolumn:name="Bandwidth",type=integer,JSONPath=`.spec.bandwidthGbps`
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
// +kubebuilder:resource:scope=Cluster,shortName=nt
type NetworkTopology struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   NetworkTopologySpec   `json:"spec,omitempty"`
    Status NetworkTopologyStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type NetworkTopologyList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []NetworkTopology `json:"items"`
}

func init() {
    SchemeBuilder.Register(&NetworkTopology{}, &NetworkTopologyList{})
}
```

Generate CRD manifests from the markers:

```bash
make manifests   # runs: controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
```

### 32.4.3 Reconcile Loop

The reconciler reads the desired `NetworkTopology` spec and annotates each listed node with rack-affinity labels, which the topology-aware scheduler in Chapter 29 then uses for pod placement *(see Chapter 29, §29.6)*.

```go
// internal/controller/networktopology_controller.go
package controller

import (
    "context"
    "fmt"
    "time"

    corev1 "k8s.io/api/core/v1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/client-go/util/workqueue"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/builder"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller"
    "sigs.k8s.io/controller-runtime/pkg/handler"
    "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/predicate"

    netv1alpha1 "github.com/example/network-topology-operator/api/v1alpha1"
)

// +kubebuilder:rbac:groups=net.aicluster.io,resources=networktopologies,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=net.aicluster.io,resources=networktopologies/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=core,resources=nodes,verbs=get;list;watch;update;patch

type NetworkTopologyReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

const rackLabel = "topology.aicluster.io/rack"

func (r *NetworkTopologyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // 1. Fetch the NetworkTopology object.
    var nt netv1alpha1.NetworkTopology
    if err := r.Get(ctx, req.NamespacedName, &nt); err != nil {
        if apierrors.IsNotFound(err) {
            // Object deleted — nothing to do.
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, fmt.Errorf("get NetworkTopology: %w", err)
    }

    // 2. Annotate each node listed in spec.nodes with the rack label.
    var reconcileErr error
    for _, nodeName := range nt.Spec.Nodes {
        var node corev1.Node
        if err := r.Get(ctx, client.ObjectKey{Name: nodeName}, &node); err != nil {
            logger.Error(err, "node not found", "node", nodeName)
            reconcileErr = err
            continue
        }

        patch := client.MergeFrom(node.DeepCopy())
        if node.Labels == nil {
            node.Labels = map[string]string{}
        }
        node.Labels[rackLabel] = nt.Spec.Rack
        if err := r.Patch(ctx, &node, patch); err != nil {
            logger.Error(err, "patch node labels", "node", nodeName)
            reconcileErr = err
        }
    }

    // 3. Update status to reflect observed generation.
    condStatus := metav1.ConditionTrue
    condReason := "Reconciled"
    condMsg := fmt.Sprintf("rack label %q applied to %d nodes", nt.Spec.Rack, len(nt.Spec.Nodes))
    if reconcileErr != nil {
        condStatus = metav1.ConditionFalse
        condReason = "NodePatchFailed"
        condMsg = reconcileErr.Error()
    }

    meta.SetStatusCondition(&nt.Status.Conditions, metav1.Condition{
        Type:               "Ready",
        Status:             condStatus,
        Reason:             condReason,
        Message:            condMsg,
        ObservedGeneration: nt.Generation,
    })
    nt.Status.ObservedGeneration = nt.Generation
    if err := r.Status().Update(ctx, &nt); err != nil {
        return ctrl.Result{}, fmt.Errorf("status update: %w", err)
    }

    return ctrl.Result{}, reconcileErr
}

// SetupWithManager registers the controller and watch predicates.
func (r *NetworkTopologyReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&netv1alpha1.NetworkTopology{}).
        // Also re-reconcile when node labels change (external modification).
        Watches(
            &corev1.Node{},
            handler.EnqueueRequestsFromMapFunc(r.nodeToTopologies),
            builder.WithPredicates(predicate.LabelChangedPredicate{}),
        ).
        WithOptions(controller.Options{
            MaxConcurrentReconciles: 4,
            RateLimiter: workqueue.NewItemExponentialFailureRateLimiter(
                500*time.Millisecond, 60*time.Second,
            ),
        }).
        Complete(r)
}
```

**Watch predicates** — `predicate.LabelChangedPredicate{}` filters the node watch to fire only when labels actually change, preventing a reconcile storm from every node heartbeat. The `MaxConcurrentReconciles` setting of 4 allows parallel reconciliation of different `NetworkTopology` objects.

**Rate limiting** — the exponential backoff rate limiter prevents a failing node patch from flooding the API server with retries.

### 32.4.4 Deploying the Operator

```bash
# Build and push the controller image
make docker-build docker-push IMG=example/network-topology-operator:v0.1.0

# Deploy CRDs and the controller Deployment
make deploy IMG=example/network-topology-operator:v0.1.0

# Verify the controller is running
kubectl get pods -n network-topology-operator-system
# NAME                                         READY   STATUS    RESTARTS   AGE
# network-topology-operator-controller-...     1/1     Running   0          30s

# Apply a NetworkTopology object
cat <<EOF | kubectl apply -f -
apiVersion: net.aicluster.io/v1alpha1
kind: NetworkTopology
metadata:
  name: rack-a
spec:
  rack: rack-a
  nodes:
    - kind-worker
    - kind-worker2
  bandwidthGbps: 400
EOF

# Verify node labels were applied
kubectl get node kind-worker -o jsonpath='{.metadata.labels.topology\.aicluster\.io/rack}'
# rack-a
```

---

## 32.5 Gateway API

### Why Gateway API Replaces Ingress

**Kubernetes Ingress** was designed in 2015 for HTTP load balancing and accumulated vendor-specific annotation sprawl to handle every other use case (TCP, gRPC, TLS passthrough, header manipulation). **Gateway API** is its structured replacement: a set of CRDs in the `gateway.networking.k8s.io` API group that express routing rules in a role-aware, protocol-aware, multi-tenant model.

The core design insight is **role separation**: infrastructure providers define `GatewayClass` objects; cluster operators deploy `Gateway` instances; application developers write `Route` objects. Each role has its own CRDs, its own namespace scope, and its own RBAC. This separation is essential for AI clusters where GPU job namespaces are tenant-isolated but share a physical load balancer.

### 32.5.1 Resource Model

**Table 32.2 — Gateway API Resource Channels (v1.2.0)**

| Resource | API Version | Channel | Scope | Managed by |
|---|---|---|---|---|
| `GatewayClass` | `gateway.networking.k8s.io/v1` | Standard | Cluster | Infrastructure provider |
| `Gateway` | `gateway.networking.k8s.io/v1` | Standard | Namespaced | Cluster operator |
| `HTTPRoute` | `gateway.networking.k8s.io/v1` | Standard | Namespaced | App developer |
| `GRPCRoute` | `gateway.networking.k8s.io/v1` | Standard | Namespaced | App developer |
| `TLSRoute` | `gateway.networking.k8s.io/v1alpha2` | Experimental (Standard since v1.5) | Namespaced | App developer |
| `TCPRoute` | `gateway.networking.k8s.io/v1alpha2` | Experimental | Namespaced | App developer |
| `ReferenceGrant` | `gateway.networking.k8s.io/v1beta1` | Standard | Namespaced | Namespace owner |

> **Note on TCPRoute**: `TCPRoute` remains in the Experimental channel as of Gateway API v1.2.0. It requires installing `experimental-install.yaml` and is subject to change. Both Gateway API upstream and Cilium's implementation mark it as experimental.

### 32.5.2 GatewayClass — Cilium's Implementation

**Cilium** implements Gateway API via its operator. When `gatewayAPI.enabled=true` is set in the Helm chart, Cilium auto-creates a `GatewayClass` with `controllerName: io.cilium/gateway-controller` *(see Chapter 12)*:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
  description: "The default Cilium GatewayClass"
```

Under the hood, Cilium translates `Gateway` and `Route` objects into `CiliumEnvoyConfig` resources that configure its embedded Envoy proxy. The Envoy listener is exposed either as a `LoadBalancer` Service or, on Cilium 1.16+, directly on the host network with `gatewayAPI.hostNetwork.enabled=true`.

### 32.5.3 Gateway — Defining a Listener

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ai-inference-gateway
  namespace: inference
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Selector      # only allow routes from labeled namespaces
          selector:
            matchLabels:
              gateway-access: "true"
    - name: grpc
      protocol: HTTP          # gRPC over HTTP/2 — same listener as HTTP
      port: 50051
      allowedRoutes:
        namespaces:
          from: Same          # only routes in the same namespace
```

`allowedRoutes.namespaces.from` controls which `Route` objects can attach to this listener — a critical multi-tenant boundary in shared AI clusters.

### 32.5.4 HTTPRoute — Routing Inference Traffic

**HTTPRoute** routes HTTP traffic based on host, path, headers, and query parameters. For AI inference serving, the primary use case is routing prefill requests to prefill workers and decode requests to decode workers (disaggregated inference as described in Chapter 29 *(see Chapter 29, §29.9)*):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: dynamo-prefill-route
  namespace: inference
spec:
  parentRefs:
    - name: ai-inference-gateway
      sectionName: http      # attach to the "http" listener
  hostnames:
    - "inference.ai-cluster.internal"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /prefill
          headers:
            - name: X-Worker-Type
              value: prefill
      backendRefs:
        - name: dynamo-prefill-svc
          port: 8080
          weight: 100
    - matches:
        - path:
            type: PathPrefix
            value: /decode
      backendRefs:
        - name: dynamo-decode-svc
          port: 8080
          weight: 100
```

### 32.5.5 GRPCRoute — Model Serving over gRPC

**GRPCRoute** (Standard, GA since v1.1.0) provides gRPC-aware routing with method-level matching, which is cleaner than header-based routing for protocol-native services:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: triton-grpc-route
  namespace: inference
spec:
  parentRefs:
    - name: ai-inference-gateway
      sectionName: grpc
  rules:
    - matches:
        - method:
            service: inference.GRPCInferenceService
            method: ModelInfer
      backendRefs:
        - name: triton-server-svc
          port: 8001
    - matches:
        - method:
            service: inference.GRPCInferenceService
            method: ServerLive
      backendRefs:
        - name: triton-server-svc
          port: 8001
```

### 32.5.6 TCPRoute — L4 Routing (Experimental)

**TCPRoute** provides L4 TCP routing for protocols that are not HTTP or gRPC. In AI clusters, this is relevant for forwarding raw NCCL rendezvous ports or custom binary protocols. Because TCPRoute is in the Experimental channel, install the experimental CRDs and be prepared for API changes:

```yaml
# Requires: kubectl apply -f .../experimental-install.yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: nccl-rendezvous-route
  namespace: training
spec:
  parentRefs:
    - name: ai-inference-gateway
      sectionName: tcp          # listener must have protocol: TCP
  rules:
    - backendRefs:
        - name: nccl-rendezvous-svc
          port: 29500
```

> Note: to use `TCPRoute`, the corresponding `Gateway` listener must use `protocol: TCP`. Cilium's TCPRoute support is also experimental as of Cilium 1.16.

### 32.5.7 ReferenceGrant — Cross-Namespace Routing

By default, a `Route` in namespace `A` cannot reference a backend `Service` in namespace `B`. **ReferenceGrant** explicitly permits this: the owner of namespace `B` creates a `ReferenceGrant` that names which kinds of objects in which namespaces are allowed to reference resources in namespace `B`.

This is the mechanism that enables an inference gateway in the `ingress` namespace to forward traffic to backends in the `inference` namespace without granting the ingress operator blanket cross-namespace access:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-inference-routes
  namespace: inference          # the namespace being referenced (backend side)
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: ingress        # the namespace where routes live (route side)
  to:
    - group: ""
      kind: Service             # allow references to Services in this namespace
```

Without this grant, the Gateway API implementation MUST treat the cross-namespace reference as invalid and NOT route traffic. Removal of a `ReferenceGrant` immediately revokes all previously-granted cross-namespace references — there is no grace period.

---

## 32.6 CRI and CSI Orientation

### Container Runtime Interface (CRI)

The **CRI (Container Runtime Interface)** is a gRPC API in `k8s.io/cri-api/pkg/apis/runtime/v1` through which the kubelet manages pod sandboxes and containers. The kubelet calls `RunPodSandbox`, `CreateContainer`, `StartContainer`, and related methods; the CRI implementation (typically **containerd** or **CRI-O**) executes the actual container lifecycle.

For AI GPU workloads, the relevant extension point is **RuntimeClass**. A `RuntimeClass` object maps a runtime handler name to a specific OCI-compatible shim:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia-crun          # pods reference this name
handler: nvidia-crun         # matches containerd's runtime handler config
overhead:
  podFixed:
    memory: "128Mi"
    cpu: "250m"
scheduling:
  nodeSelector:
    gpu.nvidia.com/cuda: "true"
```

The **containerd shim v2 API** (`containerd/containerd/api/runtime/task/v2`) is what a custom shim implements; for NVIDIA GPUs, the NVIDIA Container Toolkit provides the `nvidia-container-runtime-hook` that intercepts container creation and injects GPU devices.

### Container Storage Interface (CSI)

The **CSI (Container Storage Interface)** is defined in `github.com/container-storage-interface/spec` (v1.10 as of 2024). The per-node interface the kubelet calls for volume operations:

- `NodeStageVolume` — mount a device or remote volume to a "staging" directory on the node (once per volume, shared across pods on the node)
- `NodePublishVolume` — bind-mount from the staging path into the pod's volume directory
- `NodeUnpublishVolume` / `NodeUnstageVolume` — reverse of the above

For AI checkpoint I/O *(see Chapter 6)*, a fast NVMe-oF CSI driver uses `NodeStageVolume` to connect the NVMe-oF target and present the namespace as a local block device, then `NodePublishVolume` to mount it into the training pod's `/checkpoints` directory. For distributed filesystems *(see Chapter 18)*, the **Ceph CSI** driver mounts CephFS via kernel or FUSE at stage time, while **Lustre** uses a kernel module-based CSI driver with `NodeStageVolume` managing the Lustre mount.

---

## 32.7 Putting It Together: The RDMANetwork Operator

### Design Intent

The previous sections introduced four independent extension points. This section shows how they interact in a realistic AI cluster scenario: a `RDMANetwork` custom operator that provisions SR-IOV Virtual Functions, creates `NetworkAttachmentDefinition` objects for Multus, and updates the device plugin configuration — all in response to a single declarative `RDMANetwork` CRD object.

This operator pattern is the automation layer that replaces manual SR-IOV provisioning steps described in Chapter 13 *(see Chapter 13, §13.3)*. The goal is that a cluster operator creates one `RDMANetwork` object and the controller handles everything downstream.

### 32.7.1 CRD Design

```go
// api/v1alpha1/rdmanetwork_types.go
package v1alpha1

// RDMANetworkSpec defines the desired SR-IOV and device plugin state.
type RDMANetworkSpec struct {
    // PhysicalFunction is the PCI address of the SR-IOV capable NIC.
    // +kubebuilder:validation:Pattern=`^[0-9a-fA-F]{4}:[0-9a-fA-F]{2}:[0-9a-fA-F]{2}\.[0-9a-fA-F]$`
    PhysicalFunction string `json:"physicalFunction"`

    // NumVFs is the number of Virtual Functions to create.
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=127
    NumVFs int32 `json:"numVFs"`

    // ResourceName is the device plugin resource name suffix.
    // Full resource: rdma/{ResourceName}
    // +kubebuilder:validation:MinLength=1
    ResourceName string `json:"resourceName"`

    // NodeSelector targets which nodes this RDMA network is provisioned on.
    NodeSelector map[string]string `json:"nodeSelector,omitempty"`
}

// RDMANetworkStatus tracks operator-observed state.
type RDMANetworkStatus struct {
    // ProvisionedVFs is the count of successfully configured VFs.
    ProvisionedVFs int32 `json:"provisionedVFs,omitempty"`
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Cluster,shortName=rnet
// +kubebuilder:printcolumn:name="PF",type=string,JSONPath=`.spec.physicalFunction`
// +kubebuilder:printcolumn:name="NumVFs",type=integer,JSONPath=`.spec.numVFs`
// +kubebuilder:printcolumn:name="Resource",type=string,JSONPath=`.spec.resourceName`
type RDMANetwork struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   RDMANetworkSpec   `json:"spec,omitempty"`
    Status RDMANetworkStatus `json:"status,omitempty"`
}
```

### 32.7.2 Reconcile Loop Walkthrough

The reconciler orchestrates four Kubernetes-native operations. Each step is idempotent — running the reconciler twice produces the same result.

```go
func (r *RDMANetworkReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var rnet netv1alpha1.RDMANetwork
    if err := r.Get(ctx, req.NamespacedName, &rnet); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // ── Step 1: Ensure SriovNetworkNodePolicy exists ──────────────────────
    // The SR-IOV Network Operator (Chapter 13) watches SriovNetworkNodePolicy
    // objects and calls the sriov-dp-config DaemonSet to configure VFs on matching nodes.
    if err := r.reconcileSriovPolicy(ctx, &rnet); err != nil {
        return ctrl.Result{}, err
    }

    // ── Step 2: Ensure NetworkAttachmentDefinition exists ─────────────────
    // Multus (Chapter 13) reads NADs to configure secondary interfaces.
    // The NAD references the SR-IOV CNI plugin with the VF resource pool name.
    if err := r.reconcileNAD(ctx, &rnet); err != nil {
        return ctrl.Result{}, err
    }

    // ── Step 3: Ensure rdma-shared-device-plugin ConfigMap is current ─────
    // Updating the ConfigMap triggers the device plugin DaemonSet to reload,
    // which re-runs ListAndWatch with the new device list.
    if err := r.reconcileDevicePluginConfig(ctx, &rnet); err != nil {
        return ctrl.Result{}, err
    }

    // ── Step 4: Update status ─────────────────────────────────────────────
    rnet.Status.ProvisionedVFs = rnet.Spec.NumVFs
    meta.SetStatusCondition(&rnet.Status.Conditions, metav1.Condition{
        Type:               "Ready",
        Status:             metav1.ConditionTrue,
        Reason:             "Reconciled",
        ObservedGeneration: rnet.Generation,
    })
    return ctrl.Result{}, r.Status().Update(ctx, &rnet)
}
```

**Step 1 — `reconcileSriovPolicy`** creates or updates a `SriovNetworkNodePolicy` object in the `sriov-network-operator` namespace:

```go
func (r *RDMANetworkReconciler) reconcileSriovPolicy(ctx context.Context, rnet *netv1alpha1.RDMANetwork) error {
    desired := &sriovv1.SriovNetworkNodePolicy{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "rdma-" + rnet.Name,
            Namespace: "sriov-network-operator",
        },
    }
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, desired, func() error {
        desired.Spec = sriovv1.SriovNetworkNodePolicySpec{
            ResourceName: rnet.Spec.ResourceName,
            NodeSelector: rnet.Spec.NodeSelector,
            NumVfs:       int(rnet.Spec.NumVFs),
            NicSelector:  sriovv1.SriovNetworkNicSelector{PfNames: []string{rnet.Spec.PhysicalFunction}},
            DeviceType:   "netdevice",
        }
        return controllerutil.SetControllerReference(rnet, desired, r.Scheme)
    })
    return err
}
```

**Step 2 — `reconcileNAD`** creates a `NetworkAttachmentDefinition` so Multus can attach RDMA interfaces to pods:

```go
func (r *RDMANetworkReconciler) reconcileNAD(ctx context.Context, rnet *netv1alpha1.RDMANetwork) error {
    nadConfig := fmt.Sprintf(`{
        "cniVersion": "1.0.0",
        "name": "%s",
        "type": "sriov",
        "ipam": {}
    }`, rnet.Name)

    desired := &nadv1.NetworkAttachmentDefinition{
        ObjectMeta: metav1.ObjectMeta{
            Name:      rnet.Name,
            Namespace: "default",
        },
    }
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, desired, func() error {
        desired.Spec.Config = nadConfig
        return controllerutil.SetControllerReference(rnet, desired, r.Scheme)
    })
    return err
}
```

**Step 3 — `reconcileDevicePluginConfig`** updates the rdma-shared-device-plugin's ConfigMap. The plugin watches for ConfigMap changes and reloads its device list:

```go
func (r *RDMANetworkReconciler) reconcileDevicePluginConfig(ctx context.Context, rnet *netv1alpha1.RDMANetwork) error {
    config := map[string]interface{}{
        "periodicUpdateInterval": 300,
        "configList": []map[string]interface{}{
            {
                "resourceName":   rnet.Spec.ResourceName,
                "resourcePrefix": "rdma",
                "rdmaHcaMax":     rnet.Spec.NumVFs * 10, // allow sharing
                "devices":        []string{rnet.Spec.PhysicalFunction},
            },
        },
    }
    configJSON, _ := json.Marshal(config)

    desired := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "rdma-device-plugin-config",
            Namespace: "kube-system",
        },
    }
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, desired, func() error {
        if desired.Data == nil {
            desired.Data = map[string]string{}
        }
        desired.Data["config.json"] = string(configJSON)
        return nil
    })
    return err
}
```

**Owner references** — `controllerutil.SetControllerReference` sets the `RDMANetwork` object as the owner of the `SriovNetworkNodePolicy` and `NetworkAttachmentDefinition`, so that deleting the `RDMANetwork` triggers garbage collection of all downstream resources via Kubernetes' built-in cascading delete.

### 32.7.3 Using the RDMANetwork Operator

```yaml
apiVersion: net.aicluster.io/v1alpha1
kind: RDMANetwork
metadata:
  name: training-rdma
spec:
  physicalFunction: "0000:03:00.0"    # PCI address of the ConnectX-7 NIC
  numVFs: 8
  resourceName: hca_training_vfs
  nodeSelector:
    role: gpu-worker
```

After `kubectl apply`, the operator:
1. Creates `SriovNetworkNodePolicy/rdma-training-rdma` → SR-IOV Operator provisions 8 VFs on `0000:03:00.0` on all `role=gpu-worker` nodes
2. Creates `NetworkAttachmentDefinition/training-rdma` → Multus can attach SR-IOV interfaces to pods using `k8s.v1.cni.cncf.io/networks: training-rdma` annotation
3. Updates `ConfigMap/rdma-device-plugin-config` → device plugin now advertises `rdma/hca_training_vfs`

Training pods then request both the device plugin resource and the Multus attachment:

```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: training-rdma
spec:
  containers:
    - resources:
        limits:
          rdma/hca_training_vfs: "1"
          nvidia.com/gpu: "8"
```

---

## 32.8 Lab: CNI Plugin + Device Plugin + Operator + Gateway API on Kind

This lab runs entirely on a local **Kind** cluster. No SR-IOV hardware is required — the CNI plugin and device plugin are stubs that demonstrate the API contracts without configuring real interfaces or exposing real devices.

### Prerequisites

- Kind cluster with Cilium and Gateway API CRDs installed (see Installation section above)
- Go 1.22+, operator-sdk v1.37.0, controller-gen installed
- `kubectl` pointed at the Kind cluster

### Lab Step 1 — Build and Deploy the Minimal CNI Plugin

**1.1** Create the plugin binary from the code in §32.2.5:

```bash
mkdir minimal-cni && cd minimal-cni
go mod init github.com/example/minimal-cni-plugin
go get github.com/containernetworking/cni@v1.3.0
go get github.com/containernetworking/plugins@v1.5.0
# (copy main.go from §32.2.5)
go build -o minimal-cni-plugin .
```

**1.2** Copy the binary into the Kind node:

```bash
# Get the kind worker node container ID
WORKER=$(docker ps --filter "name=ai-ext-worker" --format "{{.ID}}" | head -1)

# Copy the plugin binary into the CNI bin directory inside the kind node
docker cp minimal-cni-plugin ${WORKER}:/opt/cni/bin/minimal-cni-plugin
docker exec ${WORKER} chmod +x /opt/cni/bin/minimal-cni-plugin
```

Expected: no errors; `ls` in the node shows the binary.

**1.3** Test the plugin binary directly with `cnitool`:

> **Scope note:** On a Kind cluster, Cilium already owns the primary CNI configuration in `/etc/cni/net.d/`. Writing a second conflist there does *not* cause kubelet to invoke the minimal plugin for ordinary pods — kubelet uses only the lexically first conflist. To exercise the binary API without disturbing Cilium, use `cnitool` directly in the Kind node container (it calls any named conflist explicitly). For integration with Kubernetes pod scheduling, the correct approach is to add the plugin as a Multus `NetworkAttachmentDefinition` delegate (see Chapter 13).

```bash
# Build cnitool on the host (requires Go 1.22+) and copy it into the Kind node.
# cnitool is not published as a pre-built binary; it must be compiled from source.
go install github.com/containernetworking/cni/cnitool@v1.3.0
docker cp $(go env GOPATH)/bin/cnitool ${WORKER}:/usr/local/bin/cnitool

# Write the test conflist to a scratch directory inside the node
# (NOT /etc/cni/net.d/ — that is owned by Cilium)
docker exec ${WORKER} bash -c 'mkdir -p /tmp/cni-test && cat > /tmp/cni-test/99-minimal-test.conflist <<'"'"'EOF'"'"'
{
  "cniVersion": "1.1.0",
  "name": "minimal-test",
  "plugins": [
    {
      "type": "minimal-cni-plugin",
      "exampleParam": "hello-from-lab"
    }
  ]
}
EOF'

# Create a test network namespace and invoke ADD
docker exec ${WORKER} bash -c '
  ip netns add cni-test-ns
  NETCONFPATH=/tmp/cni-test CNI_PATH=/opt/cni/bin \
    cnitool add minimal-test /var/run/netns/cni-test-ns
  # expected: JSON result with "cniVersion":"1.1.0" on stdout
  ip netns del cni-test-ns'
```

### Lab Step 2 — Deploy the Stub Device Plugin

**2.1** Build the device plugin image from §32.3.5:

```bash
cd stub-device-plugin
docker build -t stub-device-plugin:latest .
kind load docker-image stub-device-plugin:latest --name ai-ext
```

**2.2** Deploy as a DaemonSet:

```yaml
# stub-dp-ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: stub-device-plugin
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: stub-device-plugin
  template:
    metadata:
      labels:
        app: stub-device-plugin
    spec:
      hostNetwork: true
      containers:
        - name: stub-device-plugin
          image: stub-device-plugin:latest
          imagePullPolicy: Never
          securityContext:
            privileged: true
          volumeMounts:
            - name: device-plugin
              mountPath: /var/lib/kubelet/device-plugins
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
```

```bash
kubectl apply -f stub-dp-ds.yaml
kubectl -n kube-system rollout status daemonset/stub-device-plugin
# daemon set "stub-device-plugin" successfully rolled out
```

**2.3** Verify the resource is visible:

```bash
kubectl get node kind-worker -o json | jq '.status.allocatable["example.com/widget"]'
# "4"
```

Expected output: `"4"` (the four synthetic widget devices the stub advertises).

**2.4** Run a pod that requests the resource:

```bash
kubectl run widget-consumer --image=busybox --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"widget-consumer","image":"busybox","command":["env"],"resources":{"limits":{"example.com/widget":"1"}}}]}}'

kubectl logs widget-consumer | grep WIDGET_IDS
# WIDGET_IDS=widget-0
```

### Lab Step 3 — Deploy the NetworkTopology Operator via Operator SDK

**3.1** Scaffold, build, and deploy:

```bash
cd network-topology-operator
make manifests generate
make docker-build IMG=network-topology-operator:latest
kind load docker-image network-topology-operator:latest --name ai-ext
make deploy IMG=network-topology-operator:latest

kubectl -n network-topology-operator-system rollout status deployment \
  network-topology-operator-controller-manager
# deployment "network-topology-operator-controller-manager" successfully rolled out
```

**3.2** Apply a `NetworkTopology` object and verify node labels:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: net.aicluster.io/v1alpha1
kind: NetworkTopology
metadata:
  name: rack-a
spec:
  rack: rack-a
  nodes:
    - kind-worker
    - kind-worker2
  bandwidthGbps: 400
EOF

# Wait for reconciliation
kubectl wait networktopology/rack-a --for=condition=Ready --timeout=30s
# networktopology.net.aicluster.io/rack-a condition met

kubectl get node kind-worker --show-labels | grep rack
# topology.aicluster.io/rack=rack-a
```

### Lab Step 4 — Expose a Service via Cilium Gateway API HTTPRoute

> **Kind note:** Cilium is installed with `gatewayAPI.hostNetwork.enabled=true` (see Installation), so the Gateway listener binds directly on the Kind node's host IP instead of requiring a `LoadBalancer` Service external IP. This makes the lab work on Kind without MetalLB or a cloud provider.

**4.1** Deploy a toy inference service:

```bash
kubectl create namespace inference
kubectl label namespace inference gateway-access=true

# Use `kubectl create deployment` (not `kubectl run`) so a Deployment and
# ReplicaSet are created — `kubectl run` creates a bare Pod which has no
# rollout status and cannot be reliably targeted by a Service selector.
kubectl -n inference create deployment triton-stub \
  --image=nginx:1.27 --replicas=1
# nginx listens on port 80, not 8080
kubectl -n inference expose deployment triton-stub \
  --port=80 --target-port=80 --name=triton-stub

kubectl -n inference rollout status deployment/triton-stub
# deployment "triton-stub" successfully rolled out
```

**4.2** Create the Gateway and HTTPRoute:

```bash
kubectl apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ai-inference-gateway
  namespace: inference
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: triton-route
  namespace: inference
spec:
  parentRefs:
    - name: ai-inference-gateway
      sectionName: http
  rules:
    - backendRefs:
        - name: triton-stub
          port: 80
EOF
```

**4.3** Wait for the Gateway to be programmed and retrieve the node IP:

```bash
kubectl -n inference wait gateway/ai-inference-gateway \
  --for=condition=Programmed --timeout=60s
# gateway.gateway.networking.k8s.io/ai-inference-gateway condition met

# With hostNetwork mode the gateway address is the node's internal IP.
# On Kind this is the docker bridge IP (typically 172.18.0.x).
GW_IP=$(kubectl -n inference get gateway ai-inference-gateway \
  -o jsonpath='{.status.addresses[0].value}')
echo "Gateway IP: ${GW_IP}"
# Gateway IP: 172.18.0.2  (Kind node IP; reachable from the host)
```

**4.4** Verify end-to-end routing:

```bash
curl -s -o /dev/null -w "%{http_code}" http://${GW_IP}/
# 200
```

Expected: HTTP 200 from the nginx stub — confirming the Cilium Gateway API pipeline is active end-to-end.

### Lab Step 5 — Deploy the RDMANetwork Operator (stub mode)

In a Kind cluster without real SR-IOV hardware, deploy the operator with the SR-IOV Network Operator CRDs installed but no actual VF provisioning daemonset running. The operator's reconcile loop will create the `SriovNetworkNodePolicy` and `NetworkAttachmentDefinition` objects but the actual VF provisioning step will be a no-op.

```bash
# Install SR-IOV Network Operator CRDs only (no daemonsets)
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/sriov-network-operator/master/deploy/crds

# Deploy the RDMANetwork operator
cd rdmanetwork-operator
make manifests generate
make docker-build IMG=rdmanetwork-operator:latest
kind load docker-image rdmanetwork-operator:latest --name ai-ext
make deploy IMG=rdmanetwork-operator:latest

# Apply an RDMANetwork object
cat <<EOF | kubectl apply -f -
apiVersion: net.aicluster.io/v1alpha1
kind: RDMANetwork
metadata:
  name: training-rdma
spec:
  physicalFunction: "0000:03:00.0"
  numVFs: 8
  resourceName: hca_training_vfs
  nodeSelector:
    role: gpu-worker
EOF

# Verify downstream resources were created
kubectl get sriovnetworknodepolicy
# NAME                 AGE
# rdma-training-rdma   5s

kubectl get network-attachment-definitions.k8s.cni.cncf.io
# NAME             AGE
# training-rdma    5s

kubectl -n kube-system get configmap rdma-device-plugin-config -o yaml
# data:
#   config.json: '{"configList":[{"devices":["0000:03:00.0"],"rdmaHcaMax":80,...}],...}'
```

---

## Summary

This chapter walked through the six Kubernetes extension points — CNI, Device Plugin, CRI, CSI, Operators, and Gateway API — with deep coverage of the four that require custom implementation in AI infrastructure. The CNI spec's binary protocol, command lifecycle, IPAM delegation, and conflist chaining were illustrated with a minimal Go plugin using the modern `skel.PluginMainFuncs` API. The Device Plugin API's gRPC service definition, `ListAndWatch` streaming pattern, and `Allocate` response structure were shown through a stub implementation, with concrete examples of how NVIDIA's GPU plugin and Mellanox's RDMA shared device plugin use these contracts. The Operator SDK and controller-runtime reconcile loop model was applied to a `NetworkTopology` operator that maintains rack-affinity labels consumed by the topology-aware scheduler in Chapter 29. Gateway API's structured, role-aware resource model — `GatewayClass`, `Gateway`, `HTTPRoute`, `GRPCRoute` (GA), and `TCPRoute` (Experimental) — was demonstrated with Cilium as the implementation, including the `ReferenceGrant` mechanism for secure cross-namespace routing.

The `RDMANetwork` operator capstone tied all four active extension points together: a single CRD drives SR-IOV policy creation (Chapter 13), NetworkAttachmentDefinition provisioning (Chapter 13), and device plugin configuration (§32.3), producing a fully declarative SR-IOV RDMA network from one Kubernetes object.

The lab confirmed each component in isolation — CNI plugin binary deployed to a Kind node, stub device plugin advertising `example.com/widget` resources, `NetworkTopology` operator labeling nodes with rack affinity, and Cilium Gateway API routing HTTP traffic end-to-end.

---

## References

- **CNI Specification v1.1.0**: https://www.cni.dev/docs/spec/
- **CNI Spec Upgrades**: https://www.cni.dev/docs/spec-upgrades/
- **containernetworking/cni — skel package**: https://pkg.go.dev/github.com/containernetworking/cni/pkg/skel
- **containernetworking/cni — libcni package**: https://pkg.go.dev/github.com/containernetworking/cni/libcni
- **containernetworking/plugins**: https://github.com/containernetworking/plugins
- **Kubernetes Device Plugin API**: https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/
- **k8s.io/kubelet deviceplugin/v1beta1**: https://pkg.go.dev/k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1
- **NVIDIA k8s-device-plugin**: https://github.com/NVIDIA/k8s-device-plugin
- **Mellanox k8s-rdma-shared-dev-plugin**: https://github.com/Mellanox/k8s-rdma-shared-dev-plugin
- **Operator SDK Go Tutorial**: https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/
- **controller-runtime**: https://pkg.go.dev/sigs.k8s.io/controller-runtime
- **controller-gen markers**: https://book.kubebuilder.io/reference/markers
- **Gateway API Introduction**: https://gateway-api.sigs.k8s.io/
- **Gateway API API Overview**: https://gateway-api.sigs.k8s.io/concepts/api-overview/
- **Gateway API HTTPRoute**: https://gateway-api.sigs.k8s.io/api-types/httproute/
- **Gateway API ReferenceGrant**: https://gateway-api.sigs.k8s.io/api-types/referencegrant/
- **Gateway API v1.1 Release**: https://kubernetes.io/blog/2024/05/09/gateway-api-v1-1/
- **Cilium Gateway API documentation**: https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/
- **Cilium GatewayClass template**: https://github.com/cilium/cilium/blob/main/install/kubernetes/cilium/templates/cilium-gateway-api-class.yaml
- **CRI API**: https://pkg.go.dev/k8s.io/cri-api/pkg/apis/runtime/v1
- **CSI Spec**: https://github.com/container-storage-interface/spec
- **RuntimeClass**: https://kubernetes.io/docs/concepts/containers/runtime-class/
- **SR-IOV Network Operator**: https://github.com/k8snetworkplumbingwg/sriov-network-operator
- **Multus CNI**: https://github.com/k8snetworkplumbingwg/multus-cni
- **Kind (Kubernetes-in-Docker)**: https://kind.sigs.k8s.io/

---

*© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
