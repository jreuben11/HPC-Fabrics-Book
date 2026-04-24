# Front Matter — *The Open AI Network Stack*
### *Building High-Performance Fabrics for GPU Clusters from First Principles*

---

## Preface

The network is the bottleneck.

Every large-scale AI training run eventually hits the same wall: GPUs sit idle, not because they lack compute, but because the fabric between them cannot deliver data fast enough. A 4096-GPU cluster running a 70B-parameter model spends more wall-clock time waiting for AllReduce gradients than it does on forward and backward passes. The difference between a well-tuned fabric and a mediocre one is not 5%—it is 2×, sometimes more.

This book exists because the tools to build a world-class AI networking stack are now entirely open source, but they are scattered across a dozen communities with different vocabularies, different hardware assumptions, and almost no cross-pollination. An engineer who knows InfiniBand may not know eBPF. An engineer who knows Kubernetes networking may not know RDMA verbs. A network engineer moving into AI infrastructure may know BGP cold but have never heard of DCQCN or SHARP.

*The Open AI Network Stack* is a vertical integration guide. It starts at the RDMA transport model—the substrate everything else depends on—and builds upward through kernel-bypass I/O, programmable fabrics, container networking, model-driven management, distributed storage, and finally the testing and simulation frameworks that let you validate the whole stack before touching production hardware. Every layer is treated as software: version-controlled, CI-tested, observable, and replaceable.

The organizing principle is **disaggregation**: the deliberate dismantling of the closed, monolithic network appliance into separable, open, composable layers. UCX and LibFabric decouple transport semantics from hardware; DPDK and SPDK eject the kernel from the I/O path; eBPF extends programmability deep into Linux without leaving it; SONiC and SR Linux replace proprietary NOSes with systems that expose the ASIC through open APIs; P4 makes the forwarding pipeline itself a software artifact. The result is a network fabric that can be expressed as code—and treated with the same engineering discipline as any other software system.

---

## How to Use This Book

The book is organized into seven parts, each building on the previous:

- **Part I — Foundations** covers the invariants every other chapter depends on: cluster topology, the RDMA transport model, clock synchronization, and interconnect abstractions.
- **Part II — Kernel-Bypass & I/O** removes OS overhead from both the network and storage paths.
- **Part III — Programmable Fabric** treats the network itself as software: open NOSes, P4, SmartNIC/DPU SDKs, and traffic engineering.
- **Part IV — Overlay & Kubernetes Networking** wires GPU pods into the fabric through container networking, BGP-EVPN overlays, and security primitives.
- **Part V — Management, Telemetry & Control** covers model-driven configuration (NETCONF/YANG, gNMI) and the observability pipeline (OpenTelemetry, Prometheus, Grafana).
- **Part VI — Storage & Adjacent Infrastructure** covers the storage fabrics that feed training pipelines and the collective communication interface.
- **Part VII — Testing, Emulation, Simulation & Resilience** closes the loop: validating the full stack in software and keeping it healthy in production.

**Reading paths by role:**

| Role | Recommended path |
|------|-----------------|
| Network engineer new to AI infra | Parts I → III → V → VII |
| Kubernetes/platform engineer | Parts I (ch1–2) → IV → V → VI |
| Systems/HPC researcher | Parts I → II → III → VII (ch23) |
| Security engineer | Part I (ch1–2) → Part IV (ch26) → Part V (ch15) |
| Full stack — read cover to cover | Parts I through VII in order |

Each chapter includes hands-on lab exercises. All labs are designed to run on a single laptop using Containerlab with container images pulled from public registries. No physical hardware is required.

Chapters 19 and 20 are intentionally brief: GPU collective communications and distributed training runtimes are treated in companion investigations that go into the same depth as this book. Cross-references are provided.

---

## Lab Environment Setup

All hands-on exercises use **Containerlab** to define and deploy multi-NOS network topologies as Docker containers. The three primary NOS images used throughout the book are:

| Image | Kind | Pull command |
|---|---|---|
| SR Linux 24.3.x | `srl` | `docker pull ghcr.io/nokia/srlinux:24.3.2` |
| SONiC-VS (master) | `sonic-vs` | `docker pull ghcr.io/sonic-net/sonic-vs:master` |
| FRRouting 9.1 | `linux` | `docker pull frrouting/frr:v9.1.0` |

### Prerequisites

```bash
# Install Docker (Ubuntu 22.04):
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # log out and back in

# Install Containerlab:
bash -c "$(curl -sL https://get.containerlab.dev)"
containerlab version

# Verify SR Linux image pulls correctly:
docker pull ghcr.io/nokia/srlinux:24.3.2
```

Containerlab requires Linux with a kernel ≥ 4.10. On macOS or Windows, use a Linux VM (UTM, Lima, Multipass, or WSL2). At least 8 GB RAM is recommended; 16 GB is comfortable for multi-node topologies. All topology files used in lab exercises are collected in Appendix A.

### Quickstart — First Topology

```yaml
# Save as hello-fabric.yaml
name: hello-fabric
topology:
  nodes:
    leaf1:
      kind: linux
      image: frrouting/frr:v9.1.0
    leaf2:
      kind: linux
      image: frrouting/frr:v9.1.0
  links:
    - endpoints: ["leaf1:eth1", "leaf2:eth1"]
```

```bash
containerlab deploy -t hello-fabric.yaml
docker exec -it clab-hello-fabric-leaf1 vtysh -c "show version"
containerlab destroy -t hello-fabric.yaml
```

### Soft-RDMA for Laptop Labs

Chapters 2, 13, 24, and 28 require RDMA. On laptops without real RDMA NICs, use the `rdma_rxe` kernel module to create a software RoCE device over any Ethernet interface:

```bash
sudo modprobe rdma_rxe
sudo rdma link add rxe0 type rxe netdev eth0
rdma link show    # confirm rxe0 appears
ibstat rxe0       # confirm CA 'rxe0' with State: Active
```

`rdma_rxe` is included in all Ubuntu/Debian and Fedora kernels since 5.6. It implements the full RoCEv2 verbs API and is sufficient for all labs in this book. Throughput is CPU-limited (expect 10–20 Gb/s on modern hardware) but latency and correctness behavior are representative.

---

## Companion Investigations

Two companion investigations cover adjacent topics in comparable depth to this book. Both are referenced from chapters 19 and 20, and from Appendix D:

**GPU Collective Communications** — NCCL internals, ring and tree AllReduce algorithm selection, SHARP in-network aggregation architecture, RCCL and oneCCL differences, GPU topology detection and `NCCL_TOPO_FILE` generation.

**Distributed Training Runtimes** — PyTorch Distributed (DDP, FSDP, pipeline parallelism), DeepSpeed ZeRO stages 1–3, Megatron-LM tensor and pipeline parallelism, Ray cluster internals and object store networking.

---

## Conventions Used in This Book

- `monospace` — commands, file paths, config keys, and code
- **bold** — technology names on first introduction
- *italic* — lab exercise titles and book/spec titles
- `# comment` — inline annotation in code blocks
- Containerlab node names follow the pattern `clab-<topology-name>-<node-name>`
- IP address ranges used in labs: `10.0.0.0/8` for underlay, `192.168.0.0/16` for overlay VTEPs, `172.16.0.0/12` for pod/container networks


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).