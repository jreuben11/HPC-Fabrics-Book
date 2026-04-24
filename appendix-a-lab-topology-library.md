# Appendix A — Lab Topology Library

Ready-to-run Containerlab topologies covering the three canonical AI cluster lab configurations used in exercises throughout this book. Each topology is self-contained: copy the YAML to a working directory, pull the referenced container images, and run `containerlab deploy -t <file>.yaml`. Startup configs (where shown inline) can be extracted to separate files and referenced via `startup-config:` paths.

All topologies assume Containerlab ≥ 0.54 and Docker ≥ 24. SR Linux images are pulled from `ghcr.io/nokia/srlinux`; SONiC-VS from `ghcr.io/sonic-net/sonic-vs`; FRR from `frrouting/frr`.

---

## A.1 Two-Spine / Four-Leaf BGP-EVPN Fabric

This topology models the underlay + overlay fabric introduced in Chapters 8, 11, and 21. Two SR Linux spine switches run eBGP with four FRR leaf nodes. VXLAN/EVPN overlays are established between leaves. Two Linux hosts are attached to leaf1 and leaf3 to validate end-to-end MAC/IP reachability.

**Use cases:** Chapters 8, 11, 14, 15, 21, 22.

```yaml
name: spine-leaf-evpn

topology:
  kinds:
    srl:
      image: ghcr.io/nokia/srlinux:24.3.2
    linux:
      image: frrouting/frr:v9.1.0

  nodes:
    spine1:
      kind: srl
      startup-config: configs/spine1.cfg

    spine2:
      kind: srl
      startup-config: configs/spine2.cfg

    leaf1:
      kind: linux
      startup-config: configs/leaf1-frr.cfg

    leaf2:
      kind: linux
      startup-config: configs/leaf2-frr.cfg

    leaf3:
      kind: linux
      startup-config: configs/leaf3-frr.cfg

    leaf4:
      kind: linux
      startup-config: configs/leaf4-frr.cfg

    host1:
      kind: linux
      image: alpine:3.19

    host2:
      kind: linux
      image: alpine:3.19

  links:
    # Spine1 uplinks to all leaves
    - endpoints: ["spine1:e1-1", "leaf1:eth1"]
    - endpoints: ["spine1:e1-2", "leaf2:eth1"]
    - endpoints: ["spine1:e1-3", "leaf3:eth1"]
    - endpoints: ["spine1:e1-4", "leaf4:eth1"]
    # Spine2 uplinks to all leaves
    - endpoints: ["spine2:e1-1", "leaf1:eth2"]
    - endpoints: ["spine2:e1-2", "leaf2:eth2"]
    - endpoints: ["spine2:e1-3", "leaf3:eth2"]
    - endpoints: ["spine2:e1-4", "leaf4:eth2"]
    # Host attachments
    - endpoints: ["leaf1:eth3", "host1:eth1"]
    - endpoints: ["leaf3:eth3", "host2:eth1"]
```

### Minimal FRR Leaf Configuration (leaf1-frr.cfg)

```
frr version 9.1
frr defaults datacenter
hostname leaf1
!
interface eth1
 ip address 10.0.1.2/30
!
interface eth2
 ip address 10.0.2.2/30
!
interface lo
 ip address 10.0.0.1/32
!
router bgp 65001
 bgp router-id 10.0.0.1
 no bgp ebgp-requires-policy
 neighbor 10.0.1.1 remote-as 65000
 neighbor 10.0.2.1 remote-as 65000
 !
 address-family l2vpn evpn
  neighbor 10.0.1.1 activate
  neighbor 10.0.2.1 activate
  advertise-all-vni
 exit-address-family
!
vrf TENANT1
 vni 10001
!
```

### Validation Commands

```bash
# Verify BGP sessions on a leaf
docker exec -it clab-spine-leaf-evpn-leaf1 vtysh -c "show bgp summary"

# Check EVPN routes
docker exec -it clab-spine-leaf-evpn-leaf1 vtysh -c "show bgp l2vpn evpn"

# VXLAN tunnel verification
docker exec -it clab-spine-leaf-evpn-leaf1 bridge fdb show | grep "00:00"

# End-to-end ping
docker exec -it clab-spine-leaf-evpn-host1 ping -c3 <host2-ip>

# gNMI stream from SR Linux spine (Chapter 15)
gnmic -a clab-spine-leaf-evpn-spine1 --insecure subscribe \
  --path "/interface[name=ethernet-1/1]/statistics/in-octets"
```

---

## A.2 Rail-Optimized GPU Cluster Fabric

This topology models the rail-optimized architecture described in Chapter 1: each GPU server has two NICs — a management NIC (eth0, connected to the management leaf) and a high-speed RDMA NIC (eth1, connected to a dedicated GPU rail leaf). All RDMA traffic stays on the rail fabric; control and storage traffic traverse the management fabric.

**Use cases:** Chapters 1, 2, 13, 19, 21, 27, 28.

```yaml
name: rail-optimized-gpu

topology:
  kinds:
    srl:
      image: ghcr.io/nokia/srlinux:24.3.2
    sonic:
      image: ghcr.io/sonic-net/sonic-vs:master
    linux:
      image: ubuntu:22.04

  nodes:
    # Management fabric
    mgmt-spine:
      kind: srl

    mgmt-leaf1:
      kind: srl

    mgmt-leaf2:
      kind: srl

    # Rail fabric — one leaf per GPU rail
    rail-leaf1:
      kind: sonic
      startup-config: configs/rail-leaf1-sonic.json

    rail-leaf2:
      kind: sonic
      startup-config: configs/rail-leaf2-sonic.json

    rail-spine1:
      kind: srl

    rail-spine2:
      kind: srl

    # GPU server nodes (simulate 4 × dual-NIC GPU servers)
    gpu1:
      kind: linux
      binds:
        - /dev/net/tun:/dev/net/tun  # for rdma_rxe soft-RDMA

    gpu2:
      kind: linux
      binds:
        - /dev/net/tun:/dev/net/tun

    gpu3:
      kind: linux
      binds:
        - /dev/net/tun:/dev/net/tun

    gpu4:
      kind: linux
      binds:
        - /dev/net/tun:/dev/net/tun

  links:
    # Management spine → management leaves
    - endpoints: ["mgmt-spine:e1-1", "mgmt-leaf1:e1-1"]
    - endpoints: ["mgmt-spine:e1-2", "mgmt-leaf2:e1-1"]

    # Rail spines → rail leaves (full mesh for non-blocking AllReduce)
    - endpoints: ["rail-spine1:e1-1", "rail-leaf1:Ethernet0"]
    - endpoints: ["rail-spine1:e1-2", "rail-leaf2:Ethernet0"]
    - endpoints: ["rail-spine2:e1-1", "rail-leaf1:Ethernet4"]
    - endpoints: ["rail-spine2:e1-2", "rail-leaf2:Ethernet4"]

    # GPU server management NICs → management leaves
    - endpoints: ["gpu1:eth0", "mgmt-leaf1:e1-2"]
    - endpoints: ["gpu2:eth0", "mgmt-leaf1:e1-3"]
    - endpoints: ["gpu3:eth0", "mgmt-leaf2:e1-2"]
    - endpoints: ["gpu4:eth0", "mgmt-leaf2:e1-3"]

    # GPU server RDMA NICs → rail leaves (rail assignment)
    - endpoints: ["gpu1:eth1", "rail-leaf1:Ethernet8"]
    - endpoints: ["gpu2:eth1", "rail-leaf1:Ethernet12"]
    - endpoints: ["gpu3:eth1", "rail-leaf2:Ethernet8"]
    - endpoints: ["gpu4:eth1", "rail-leaf2:Ethernet12"]
```

### Setting Up Soft-RDMA for Lab Testing

The GPU server containers lack real RDMA hardware. Use `rdma_rxe` (soft-RDMA over Ethernet) to emulate RoCE on the rail interfaces:

```bash
# On each gpu container
modprobe rdma_rxe
rdma link add rxe0 type rxe netdev eth1

# Verify RDMA device
rdma link show
ibstat rxe0

# Run bandwidth test between gpu1 (server) and gpu2 (client)
# On gpu1:
ib_write_bw -d rxe0 --report_gbits -s 65536 -n 1000

# On gpu2:
ib_write_bw -d rxe0 --report_gbits -s 65536 -n 1000 <gpu1-eth1-ip>
```

### Validate Rail Isolation

```bash
# Confirm rail traffic does NOT traverse management fabric
# RDMA ping: gpu1 rail → gpu3 rail (cross-leaf, uses rail spines)
docker exec -it clab-rail-optimized-gpu-gpu1 \
  ibping -d rxe0 -G <gpu3-GID>

# Management ping: confirm eth0 reaches mgmt network only
docker exec -it clab-rail-optimized-gpu-gpu1 ping -I eth0 <mgmt-leaf1-ip>

# SONiC show interfaces on rail-leaf1
docker exec -it clab-rail-optimized-gpu-rail-leaf1 \
  sonic-cli -c "show interfaces status"
```

---

## A.3 Multi-NIC GPU Pod Testbed

This topology is the Kubernetes-focused lab companion to Chapters 12, 13, and 29. A Kind-based Kubernetes cluster provides the management network (Cilium CNI); SR-IOV virtual functions on a secondary NIC provide the RDMA data path. Two GPU pod replicas run `nccl-tests` across the secondary interface.

**Use cases:** Chapters 12, 13, 26, 29.

> **Note:** This topology requires a host with SR-IOV-capable NICs for VF-based secondary interfaces, or falls back to `macvlan` secondary CNI for pure software simulation.

```yaml
name: multi-nic-gpu-pod

topology:
  kinds:
    linux:
      image: ubuntu:22.04

  nodes:
    # Kind Kubernetes control plane
    k8s-control:
      kind: linux
      cmd: /usr/local/bin/kind-entrypoint.sh

    # Worker nodes — each simulates a dual-NIC GPU server
    worker1:
      kind: linux
      binds:
        - /dev/net/tun:/dev/net/tun
        - /sys/bus/pci:/sys/bus/pci:ro

    worker2:
      kind: linux
      binds:
        - /dev/net/tun:/dev/net/tun
        - /sys/bus/pci:/sys/bus/pci:ro

    # ToR switch connecting worker management NICs
    tor-mgmt:
      kind: srl
      image: ghcr.io/nokia/srlinux:24.3.2

    # Rail leaf for RDMA secondary NICs
    tor-rdma:
      kind: sonic
      image: ghcr.io/sonic-net/sonic-vs:master

  links:
    # Management fabric
    - endpoints: ["k8s-control:eth0", "tor-mgmt:e1-1"]
    - endpoints: ["worker1:eth0",      "tor-mgmt:e1-2"]
    - endpoints: ["worker2:eth0",      "tor-mgmt:e1-3"]
    # RDMA fabric (secondary NICs)
    - endpoints: ["worker1:eth1",      "tor-rdma:Ethernet0"]
    - endpoints: ["worker2:eth1",      "tor-rdma:Ethernet4"]
```

### Kubernetes Setup Commands

```bash
# 1. Deploy Kind cluster (management CNI only at this stage)
kind create cluster --config kind-config.yaml

# 2. Install Cilium CNI
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.15.4 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<control-ip> \
  --set k8sServicePort=6443

# 3. Install SR-IOV Network Operator
helm repo add sriov https://k8snetworkplumbingwg.github.io/sriov-network-operator
helm install sriov-operator sriov/sriov-network-operator \
  --namespace sriov-network-operator --create-namespace

# 4. Install Multus CNI
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml

# 5. Define secondary RDMA network attachment
kubectl apply -f - <<'EOF'
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rdma-net
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/resourceName: rdma/rdma_shared_device_a
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "rdma-net",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24"
      }
    }
EOF

# 6. Launch NCCL test pod pair
kubectl apply -f nccl-test-job.yaml

# 7. Validate Multus secondary interface is present
kubectl exec -it nccl-worker-0 -- ip addr show net1

# 8. Run AllReduce test
kubectl exec -it nccl-worker-0 -- \
  /opt/nccl-tests/build/all_reduce_perf \
    -b 8 -e 256M -f 2 -g 1 \
    --nccl_socket_ifname=net1
```

### NetworkAttachmentDefinition Verification

```bash
# Confirm secondary IP assigned by Whereabouts
kubectl get pod nccl-worker-0 \
  -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/networks-status}' | jq .

# Check Hubble flow between pods (Chapter 12)
hubble observe --namespace default --from-pod nccl-worker-0 --to-pod nccl-worker-1

# Cilium identity for GPU pods
cilium endpoint list | grep nccl
```


---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).