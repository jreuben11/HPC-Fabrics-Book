# Chapter 31 — GPU Virtualization & Isolation: VFIO, MIG, MPS & KubeVirt

**Part VI: Advanced GPU & Network Integration** | ~25 pages

---

## Introduction

A physical GPU on a bare-metal host offers maximum performance: every compute unit, every byte of HBM, every PCIe lane, every NVLink channel belongs to a single process. The problem is that bare metal is expensive to share. When a 96-node cluster of H100 servers idles at 15% average utilization between training runs — a common pattern for fine-tuning pipelines and inference workloads — the capital cost is mostly wasted. GPU virtualization and partitioning exist to solve that utilization problem, but each solution trades some part of bare-metal performance for some degree of isolation or sharing flexibility.

This chapter maps the full taxonomy of GPU isolation models, explains the kernel and hardware mechanisms that underpin each one, and gives complete configuration walkthroughs for the options that appear most often in AI infrastructure. The decision matrix in §31.1 is the spine of the chapter — every subsequent section justifies a row in it.

Section 31.2 covers the Linux **VFIO** (**Virtual Function I/O**) subsystem: **IOMMU** groups, the **vfio-pci** driver, the ACS override patch and when it is actually needed, and ROM BAR handling. Section 31.3 translates VFIO into practice with **KVM**/**QEMU** GPU passthrough, including **OVMF** UEFI firmware, huge-page-backed guest RAM, and the `qemu-system-x86_64` device assignment flags. Section 31.4 goes one level up the stack to **libvirt**/**virsh**, covering the `<hostdev>` domain XML element and network backend choices. Section 31.5 covers **Proxmox VE** in depth: PCIe passthrough setup, the `qm` and `pvesh` management CLIs, the Proxmox networking stack, **Corosync** cluster quorum, HA fencing, and **Ceph** RBD storage integration.

Sections 31.6 and 31.7 cover the two primary GPU partitioning technologies: **NVIDIA MIG** (**Multi-Instance GPU**), which partitions a single physical GPU into fully isolated hardware slices, and **NVIDIA MPS** (**Multi-Process Service**), which multiplexes CUDA contexts at the software level with shared hardware. Section 31.8 covers **KubeVirt**, the Kubernetes extension that runs VMs as pods, and its GPU passthrough and SR-IOV networking capabilities. Section 31.9 examines networking consequences — particularly why RDMA and **GPUDirect** require careful configuration under each isolation model. The chapter closes with the lab.

This chapter builds on Chapter 2 (**RoCEv2** and **GPUDirect RDMA** fundamentals), Chapter 10 (**SR-IOV** Physical and Virtual Functions on DPUs), Chapter 13 (**SR-IOV** in Kubernetes pods, `NetworkAttachmentDefinition`, **Multus**), Chapter 19 (**NCCL** collective behavior), and Chapter 29 (GPU device scheduling in Kubernetes). Readers who have not read Chapter 13 should review its SR-IOV VF lifecycle and Multus annotation model before tackling §31.8.

---

## Installation

The lab in this chapter has two phases with different hardware requirements. **Phase A** (Proxmox + QEMU passthrough) requires a physical machine with a real GPU, IOMMU support in the CPU and motherboard, and Proxmox VE 8.x installed. **Phase B** (KubeVirt + Soft-RoCE verification) runs entirely on a development laptop using **Kind**, KubeVirt, and the `rdma_rxe` soft-RDMA module for RDMA device emulation — no GPU hardware is required.

### Phase A: Physical host prerequisites

```bash
# Verify IOMMU is active (requires intel_iommu=on or amd_iommu=on in cmdline)
dmesg | grep -e IOMMU -e AMD-Vi -e "Directed I/O"
# Expected: DMAR: IOMMU enabled / AMD-Vi: Interrupt remapping enabled

# Install Proxmox VE 8.x on a bare-metal host per https://www.proxmox.com/en/proxmox-virtual-environment/get-started
# Verify version
pveversion
# pve-manager/8.2.x (running kernel: 6.8.x-x-pve)
```

### Phase B: Laptop / CI prerequisites

```bash
# Kind, kubectl, Helm (as in Chapter 13)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64 \
  && chmod +x kind && sudo mv kind /usr/local/bin/

# Soft-RoCE for RDMA emulation inside KubeVirt VMIs
sudo modprobe rdma_rxe
lsmod | grep rdma_rxe

# KubeVirt CLI
export KUBEVIRT_VERSION=v1.3.0
curl -Lo virtctl https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/virtctl-${KUBEVIRT_VERSION}-linux-amd64 \
  && chmod +x virtctl && sudo mv virtctl /usr/local/bin/
virtctl version --client
# Client Version: v1.3.0
```

---

## 31.1 GPU Isolation Taxonomy and Decision Matrix

Before choosing an isolation model, it helps to understand what exactly is being isolated and at what layer. Four resources must be partitioned or shared: compute units (SMs/CUs), memory (HBM), the PCIe/NVLink interconnect, and the security context (faults, resets, errant DMA). Each model handles these four dimensions differently.

| Model | Isolation granularity | HBM sharing | GPUDirect RDMA | Live migration | Best for |
|---|---|---|---|---|---|
| Bare metal | None (full GPU per process) | No | Yes (full support) | No | Training, benchmarking |
| VFIO passthrough | VM boundary (full GPU) | No | Yes (with `nvidia-peermem`/DMA-BUF) | No | Single-tenant GPU VMs |
| NVIDIA vGPU | Time-sliced or MIG-backed vGPU | Time-slice | No (vGPU abstractions break P2P) | Yes (licensed) | VDI, inference |
| AMD MxGPU | SR-IOV VF (hardware) | No | Limited | Yes | Multi-tenant inference VMs |
| NVIDIA MIG | HW-partitioned slice (GI+CI) | No | Yes (per-slice) | No | Multi-tenant training, inference |
| NVIDIA MPS | Shared SM context, soft limits | Yes (shared heap) | Yes (same physical GPU) | No | MPI jobs, inference batching |
| Container (device plugin) | cgroup/namespace | Shared by default | Yes | N/A | Standard Kubernetes GPU pods |
| KubeVirt + passthrough | VM boundary inside Kubernetes | No | Yes (same as passthrough) | No | GPU VMs in Kubernetes |

**Key insight**: As isolation increases from container → MPS → MIG → passthrough, per-workload performance guarantees improve but resource sharing efficiency decreases and RDMA/GPUDirect capabilities become more constrained. Workloads that require the full GPUDirect peer-to-peer path (§31.9) must use bare-metal, MIG (within a slice), or VFIO passthrough — not vGPU.

---

## 31.2 VFIO and IOMMU: The Kernel Foundation

**VFIO** (Virtual Function I/O) is the Linux kernel subsystem that enables safe, user-space-controlled device assignment. It exposes physical PCIe devices to user-space programs (typically QEMU/KVM guests) through a character device interface, while relying on the platform **IOMMU** to prevent the assigned device from performing unauthorized DMA into host memory.

### 31.2.1 IOMMU Groups

The **IOMMU** maps DMA addresses from a device's perspective to physical host memory addresses, exactly as the MMU does for CPU virtual addresses. An **IOMMU group** is the set of PCIe devices that the IOMMU cannot isolate from each other — typically because they share a PCIe ACS (Access Control Services) domain or are physically connected through a switch that lacks ACS capability.

The critical rule: all devices in an IOMMU group must be assigned to the same VM together, or none at all. If a GPU and its companion HD Audio controller are in the same IOMMU group, both must be passed through.

```bash
# Enumerate all IOMMU groups on the host
for dir in /sys/kernel/iommu_groups/*/devices/*; do
    echo "Group $(basename $(dirname $(dirname $dir))): $(lspci -nns $(basename $dir))"
done
# Example output:
# Group 1: 00:01.0 PCI bridge [0604]: Intel Corporation 12th Gen Core Processor PCI Express x16 Controller [8086:460d]
# Group 24: 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [GeForce RTX 3090] [10de:2204]
# Group 24: 01:00.1 Audio device [0403]: NVIDIA Corporation GA102 High Definition Audio Controller [10de:1aef]
```

Ideal IOMMU group structure puts the GPU (slot 0) and its audio function (slot 1) in the same group, with no other devices. This is the common case on modern servers with properly configured motherboards.

### 31.2.2 ACS Override Patch

**ACS** (Access Control Services) is the PCIe mechanism that gates peer-to-peer DMA between devices in the same hierarchy. When ACS is disabled or absent on an intermediate switch, all endpoints behind that switch share one IOMMU group — making isolated passthrough impossible without moving devices.

The **ACS override patch** (originally by Alex Williamson) is an out-of-tree kernel patch that forces every PCIe device into its own IOMMU group by lying to the kernel about ACS capability. It is available in some distribution kernels (Proxmox's `pve-kernel` includes it; the Arch Linux AUR `linux-vfio` package applies it) but has never been accepted upstream.

> **Security warning**: The ACS override patch undermines the isolation guarantee that VFIO depends on. Devices that are logically separated into different IOMMU groups by the patch may still be able to perform peer-to-peer DMA into each other's memory. Use it only when no hardware ACS is available and you understand the risk.

On modern Intel 12th/13th-gen and AMD Zen 3/4 server platforms, the CPU root complex typically provides ACS, so the patch is often unnecessary. Check `lspci -vvv | grep ACSCtl` for each device in a shared group before resorting to the patch.

### 31.2.3 vfio-pci Driver Binding

The `vfio-pci` driver is the VFIO backend for PCI devices. Binding a device to `vfio-pci` detaches it from its host driver (e.g., `nvidia`) and makes it available for VFIO assignment.

```bash
# Step 1: Identify device PCI address and vendor:device ID
lspci -nn | grep -i nvidia
# 01:00.0 VGA [0300]: NVIDIA Corporation GA102 [10de:2204] (rev a1)
# 01:00.1 Audio [0403]: NVIDIA Corporation GA102 HD Audio [10de:1aef] (rev a1)

# Step 2: Unbind from current driver
echo "0000:01:00.0" | sudo tee /sys/bus/pci/devices/0000:01:00.0/driver/unbind
echo "0000:01:00.1" | sudo tee /sys/bus/pci/devices/0000:01:00.1/driver/unbind

# Step 3: Bind to vfio-pci
echo "10de 2204" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
echo "10de 1aef" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id

# Verify
lspci -k -s 01:00.0
# Kernel driver in use: vfio-pci
```

For persistent binding across reboots, use the kernel cmdline approach (boot-time binding is more reliable than initramfs scripts for GPU passthrough):

```bash
# /etc/default/grub — add to GRUB_CMDLINE_LINUX_DEFAULT:
# intel_iommu=on iommu=pt vfio-pci.ids=10de:2204,10de:1aef
# For AMD:
# amd_iommu=on iommu=pt vfio-pci.ids=10de:2204,10de:1aef

sudo update-grub
sudo update-initramfs -u -k all
```

The `iommu=pt` (passthrough) flag instructs the IOMMU to use identity mappings for devices not assigned to VMs, reducing translation overhead for host devices while still protecting guest-assigned devices.

### 31.2.4 ROM BAR Issues

Some GPUs — particularly consumer NVIDIA cards — have VBIOS ROMs that are locked or that the PCI ROM BAR exposes in a form QEMU cannot read at runtime. The symptom is a blank screen in the guest or a QEMU error like `romfile: file not found` or EFI not recognizing the GPU.

The standard workaround is to dump the ROM directly from the host before binding to `vfio-pci`:

```bash
# Dump GPU VBIOS while nvidia driver is still bound
echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.0/rom
sudo cat /sys/bus/pci/devices/0000:01:00.0/rom > /tmp/gpu.rom
echo 0 | sudo tee /sys/bus/pci/devices/0000:01:00.0/rom
sudo cp /tmp/gpu.rom /usr/share/kvm/gpu.rom
```

In QEMU/libvirt the ROM is then passed explicitly (`romfile=gpu.rom`), bypassing the runtime read entirely (§31.3).

---

## 31.3 KVM/QEMU GPU Passthrough

**KVM** (**Kernel-based Virtual Machine**) is the Linux hypervisor that runs as a kernel module, providing hardware-accelerated virtualization for x86 VMs. **QEMU** is the machine emulator that uses KVM for CPU virtualization while handling device emulation in user space. Together they form the most common open-source hypervisor stack for GPU passthrough.

### 31.3.1 OVMF UEFI Firmware

**OVMF** (**Open Virtual Machine Firmware**) is an UEFI firmware implementation for QEMU. Modern NVIDIA GPUs require UEFI boot for proper operation — SeaBIOS lacks the EFI GOP (Graphics Output Protocol) initialization that the driver expects. OVMF is required for GPU passthrough on any GPU released after 2016.

```bash
sudo apt install -y ovmf
ls /usr/share/OVMF/
# OVMF_CODE_4M.fd  OVMF_VARS_4M.fd  OVMF_CODE.fd  OVMF_VARS.fd
```

Use the `_4M` variants for modern VMs — they include support for secure boot and a larger variable store required for GPU GOP initialization.

### 31.3.2 Huge-Page-Backed Guest RAM

Allocating guest RAM on 1 GiB huge pages eliminates TLB pressure from the second-level address translation (EPT/NPT) that KVM performs on every guest memory access. For GPU workloads that pin large contiguous memory regions (CUDA allocations, DMA buffers), the performance improvement from huge pages is typically 1–3% on memory-intensive benchmarks.

```bash
# Reserve 64 × 1 GiB hugepages (for a 64 GiB guest)
echo 64 | sudo tee /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# Mount hugetlbfs
sudo mkdir -p /mnt/hugepages-1G
sudo mount -t hugetlbfs -o pagesize=1G none /mnt/hugepages-1G

# Verify
grep HugePages /proc/meminfo
# HugePages_Total:      64
# HugePages_Free:       64
```

### 31.3.3 QEMU Device Assignment Command

A minimal `qemu-system-x86_64` invocation for GPU passthrough:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -machine q35,accel=kvm \
  -cpu host,kvm=off \
  -m 64G \
  -mem-path /mnt/hugepages-1G \
  -mem-prealloc \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.fd \
  -drive if=pflash,format=raw,file=/var/lib/vms/gpu-vm-vars.fd \
  -device vfio-pci,host=01:00.0,multifunction=on,x-vga=on,romfile=/usr/share/kvm/gpu.rom \
  -device vfio-pci,host=01:00.1 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0 \
  -drive file=/var/lib/vms/gpu-vm.qcow2,if=virtio \
  -nographic
```

Key flags:
- `-machine q35`: required for PCIe topology (not ISA-style PCI)
- `-cpu host,kvm=off`: expose host CPU features; `kvm=off` hides KVM from guest CPUID (helps with NVIDIA driver's hypervisor detection on some versions)
- `-mem-path /mnt/hugepages-1G`: back guest RAM with 1 GiB hugepages
- `-mem-prealloc`: allocate all guest memory at VM start, avoiding runtime fault storms
- `vfio-pci,host=01:00.0,x-vga=on`: assign GPU with VGA arbitration
- `romfile=`: use the pre-dumped VBIOS (§31.2.4)
- `multifunction=on`: required when audio function (01:00.1) accompanies the GPU

---

## 31.4 libvirt/virsh: XML-Based GPU Passthrough

**libvirt** is the standard management API layer above QEMU/KVM. It stores VM configuration as XML domain definitions and exposes a unified API consumed by virt-manager, Proxmox, and OpenStack. **virsh** is its command-line client.

### 31.4.1 Detaching the Device from the Host

Before libvirt can attach the device to a VM, it must be detached from the host driver. With `managed='yes'` in the domain XML, libvirt handles this automatically at VM start. For manual control:

```bash
# List PCI devices in libvirt's node device view
virsh nodedev-list | grep pci

# Detach GPU from host driver (binds to vfio-pci)
virsh nodedev-detach pci_0000_01_00_0
virsh nodedev-detach pci_0000_01_00_1

# Re-attach after VM stops
virsh nodedev-reattach pci_0000_01_00_0
```

### 31.4.2 Domain XML: `<hostdev>` PCI Passthrough

```xml
<domain type='kvm'>
  <name>gpu-vm</name>
  <memory unit='GiB'>64</memory>
  <cpu mode='host-passthrough' check='none'>
    <cache mode='passthrough'/>
  </cpu>
  <os>
    <type arch='x86_64' machine='q35'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/OVMF/OVMF_CODE_4M.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/gpu-vm_VARS.fd</nvram>
  </os>
  <memoryBacking>
    <hugepages>
      <page size='1048576' unit='KiB'/>
    </hugepages>
    <locked/>
  </memoryBacking>
  <devices>
    <!-- GPU: PCI passthrough via VFIO -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
      </source>
      <rom file='/usr/share/kvm/gpu.rom'/>
    </hostdev>
    <!-- GPU audio: must travel with GPU (same IOMMU group) -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
      </source>
    </hostdev>
    <!-- Network: Linux bridge for management -->
    <interface type='bridge'>
      <source bridge='br0'/>
      <model type='virtio'/>
    </interface>
    <!-- SR-IOV VF for RDMA (see §31.9) -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <!-- Mellanox ConnectX-7 VF assigned for RDMA -->
        <address domain='0x0000' bus='0x82' slot='0x00' function='0x0'/>
      </source>
    </hostdev>
  </devices>
</domain>
```

### 31.4.3 Network Backend Choices

libvirt supports three network backends for GPU VMs, each with different performance and isolation characteristics:

| Backend | `<interface type>` | Performance | RDMA support | Notes |
|---|---|---|---|---|
| Linux bridge | `bridge` | Good | No (software path) | Default; simple; shares host bridge |
| macvtap | `direct` | Better | No | Direct NIC attachment; bypasses host bridge |
| SR-IOV VF direct | `hostdev` (type='pci') | Near-line-rate | Yes (with RDMA NIC VF) | See §31.9; requires RDMA NIC with SR-IOV |

For GPU VMs that need **GPUDirect RDMA**, SR-IOV VF direct assignment is the only viable option (*see Chapter 13 for VF provisioning details*).

---

## 31.5 Proxmox VE: GPU Passthrough in Production

**Proxmox VE** (**PVE**) is an open-source server virtualization platform that combines KVM/QEMU and LXC containers under a unified web UI, REST API (`pvesh`), and CLI (`qm`, `pveceph`, `pvecm`). Version 8.x ships with a 6.8.x PVE kernel that includes the ACS override patch and VFIO modules pre-integrated.

### 31.5.1 IOMMU and VFIO Kernel Setup

```bash
# /etc/default/grub — for Intel:
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
# For AMD:
# GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"

update-grub

# /etc/modules — add VFIO modules (note: vfio_virqfd removed in PVE 8+)
cat >> /etc/modules <<'EOF'
vfio
vfio_iommu_type1
vfio_pci
EOF

update-initramfs -u -k all
reboot
```

After reboot, verify:

```bash
dmesg | grep -e IOMMU -e "DMAR: IOMMU"
# DMAR: IOMMU enabled

# Confirm GPU is in its own IOMMU group
find /sys/kernel/iommu_groups/ -name "*.0" | sort
```

### 31.5.2 Creating a GPU VM with qm

**qm** is Proxmox's VM configuration CLI. Every `qm set` option maps to a line in the VM's configuration file at `/etc/pve/qemu-server/<VMID>.conf`.

```bash
# Create VM 100 with OVMF UEFI and q35 machine type
qm create 100 \
  --name gpu-vm \
  --memory 65536 \
  --cores 16 \
  --sockets 1 \
  --cpu host,hidden=1 \
  --machine q35 \
  --bios ovmf \
  --efidisk0 local-lvm:1,format=raw,efitype=4m,pre-enrolled-keys=0 \
  --net0 virtio,bridge=vmbr0

# Allocate hugepage-backed memory
qm set 100 --hugepages 1024
qm set 100 --keephugepages 1

# Attach GPU via PCIe passthrough
# pcie=1: use PCIe (vs older PCI) addressing — required for modern GPUs
# x-vga=1: expose VGA framebuffer
# rombar=1: expose ROM BAR (disable if causing conflicts)
qm set 100 --hostpci0 01:00,pcie=1,x-vga=1,rombar=1,romfile=gpu.rom

# Create boot disk
qm set 100 --virtio0 local-lvm:64,format=raw
qm set 100 --boot order=virtio0
```

The resulting `/etc/pve/qemu-server/100.conf`:

```ini
bios: ovmf
boot: order=virtio0
cores: 16
cpu: host,hidden=1
efidisk0: local-lvm:vm-100-disk-0,efitype=4m,pre-enrolled-keys=0,size=1M
hostpci0: 0000:01:00,pcie=1,rombar=1,romfile=gpu.rom,x-vga=1
hugepages: 1024
keephugepages: 1
machine: q35
memory: 65536
name: gpu-vm
net0: virtio=AA:BB:CC:DD:EE:FF,bridge=vmbr0
sockets: 1
virtio0: local-lvm:vm-100-disk-0,size=64G
```

### 31.5.3 PCIe Bifurcation for Multi-GPU Hosts

When a host has multiple GPUs on a single PCIe slot through a PCIe switch (bifurcation), each GPU appears as a separate IOMMU group on servers that support ACS on the switch. Assign them to separate VMs:

```bash
# Second GPU to VM 101
qm set 101 --hostpci0 02:00,pcie=1,x-vga=0,rombar=1
```

For training VMs that need multiple GPUs on NVLink, all GPUs must go to the same VM — NVLink P2P does not cross VM boundaries.

### 31.5.4 Proxmox Networking Stack

Proxmox VE supports three network modes for VMs:

1. **Linux bridge (default)**: `vmbr0` is a standard Linux bridge backed by a physical NIC. Suitable for management traffic but adds a software switching layer.

2. **Open vSwitch mode**: Replace the Linux bridge with **OVS** for VLAN-aware switching, LACP bonding, and OVSDB configuration. Enable in the Proxmox web UI under Network, or:
   ```bash
   apt install -y openvswitch-switch
   # Convert vmbr0 to OVS in /etc/network/interfaces
   ```

3. **SDN VXLAN/EVPN plugin**: Proxmox 8.x includes an SDN plugin framework that configures VXLAN/EVPN overlays using FRR as the BGP/EVPN daemon (see Chapter 11 for EVPN fundamentals). Configure via `pvesh`:
   ```bash
   pvesh create /cluster/sdn/zones --zone overlay --type vxlan
   pvesh create /cluster/sdn/vnets --vnet gpu-vnet --zone overlay --tag 1001
   pvesh set /cluster/sdn --apply 1
   ```

### 31.5.5 Corosync Cluster Quorum and HA

**Corosync** is the distributed membership and messaging daemon that underpins Proxmox cluster state. All cluster nodes share the Proxmox cluster filesystem (`pmxcfs`) replicated over Corosync. Quorum requires more than half the nodes to agree — a 3-node cluster tolerates one failure.

```bash
# Initialize cluster (on first node)
pvecm create ai-gpu-cluster

# Join additional nodes
pvecm add <first-node-ip>

# Check cluster status
pvecm status
# Quorate: Yes
# Nodes: 3 (online: 3)
```

HA fencing ensures a failed node is positively powered off before its VMs are restarted on surviving nodes, preventing split-brain data corruption. Proxmox uses **watchdog-based fencing** by default — the `pve-ha-crm` daemon manages a hardware or software watchdog through the `watchdog-mux` service. If the HA CRM loses quorum contact, it stops feeding the watchdog, which fires and resets the node.

```bash
# /etc/default/pve-ha-manager — select watchdog module
# Hardware watchdog (preferred): use the board's native WDT driver
# Software watchdog (fallback, for lab VMs that lack hardware WDT):
cat /etc/default/pve-ha-manager
# WATCHDOG_MODULE=softdog    # set to iTCO_wdt, nv_tco, etc. for hardware

# Verify watchdog-mux is running
systemctl status watchdog-mux

# Add a VM to HA management (three equivalent methods):

# Method 1: ha-manager CLI (recommended)
ha-manager add vm:100 --group ha-group --max_restart 2 --max_relocate 1

# Method 2: write directly to pmxcfs config
# cat /etc/pve/ha/resources.cfg
# vm:100
#   group ha-group
#   max_restart 2

# Method 3: Proxmox web UI → Datacenter → HA → Add

# Create an HA group (controls which nodes may run the VM)
ha-manager groupadd ha-group --nodes "pve-01:2,pve-02:1,pve-03:1" --restricted 1

# Show HA status
ha-manager status
# quorum OK
# master pve-01
# vm:100 (started) node=pve-01
```

> **Fencing note**: Proxmox does not expose an IPMI-based fencer through the `pvesh` REST API. External IPMI fencing (for guaranteed node isolation when watchdog is insufficient) is handled by configuring the BMC/IPMI device at the OS level and integrating it with the watchdog via `pve-ha-crm` configuration, or by using IPMI-capable hardware that auto-resets on watchdog timeout. In most deployments the default software watchdog (`softdog`) is adequate for lab environments; production clusters should use a hardware watchdog module matched to the server's chipset (e.g., `iTCO_wdt` for Intel platforms).

### 31.5.6 Ceph RBD Storage Integration

**Ceph** provides distributed block storage that makes GPU VM disks available from any cluster node, enabling HA failover. Without Ceph (or another shared storage), VM disks are local — a failed node's VMs cannot restart anywhere else.

```bash
# Install Ceph on Proxmox nodes (minimum 3 nodes)
pveceph init --network 10.0.1.0/24   # dedicated Ceph network
pveceph createmon
pveceph createosd /dev/nvme0n1

# Create RBD pool for VM disks
pveceph pool create vm-disks --size 3 --min_size 2

# Add to Proxmox storage
pvesh create /storage --storage ceph-rbd --type rbd \
  --pool vm-disks --monhost "10.0.1.1,10.0.1.2,10.0.1.3"

# Create VM disk on Ceph RBD
qm set 100 --virtio0 ceph-rbd:64,format=raw
```

With Ceph-backed VM disks, GPU VM failover works correctly: the HA manager fences the dead node and restarts the VM on another node, which mounts the same RBD volume.

> **Note**: GPU passthrough disables live migration. A GPU VM on Proxmox HA will restart cold on another node after fencing — not live-migrated. Live migration requires detaching the GPU first, or using vGPU/MIG instead of passthrough.

---

## 31.6 NVIDIA MIG: Hardware GPU Partitioning

**MIG** (**Multi-Instance GPU**) is a hardware feature introduced in NVIDIA A100 that partitions a single physical GPU into up to seven fully isolated GPU instances, each with dedicated compute units, memory bandwidth, and HBM capacity. Unlike software-only approaches, MIG isolation is enforced by the GPU hardware itself — faults, hangs, and ECC errors in one MIG instance do not affect others.

### 31.6.1 GI/CI Hierarchy

MIG uses a two-level hierarchy:

- **GPU Instance (GI)**: a hardware partition of the physical GPU, with dedicated SM clusters, memory controllers, and HBM. The GI profile determines compute and memory allocation.
- **Compute Instance (CI)**: a partition of a GI's compute resources, sharing the GI's memory. One GI can contain multiple CIs, allowing further multiplexing of compute while maintaining shared memory isolation at the GI level.

A single H100 80GB in `7g.80gb` mode is effectively the full GPU as one GI with one CI. In `1g.10gb` mode, the same GPU becomes seven independent GIs of 1/7 compute and 10 GiB HBM each.

### 31.6.2 MIG Profile Catalogue

MIG profiles follow the format `<C>g.<M>gb[+me]` where `C` is the number of GPU compute slices (out of 7) and `M` is the memory allocation in GiB. The `+me` suffix adds a memory encryption capability slice.

**A100 80GB SXM profiles** (the 40GB variant uses `1g.5gb`, `2g.10gb`, `3g.20gb`, `4g.20gb`, `7g.40gb` — note the halved memory numbers):

| Profile | SMs | HBM | Max instances | Notes |
|---|---|---|---|---|
| `1g.10gb` | 1/7 | 10 GiB | 7 | Smallest slice |
| `1g.10gb+me` | 1/7 | 10 GiB | 1 | With memory encryption |
| `1g.20gb` | 1/7 | 20 GiB | 4 | Double memory per slice |
| `2g.20gb` | 2/7 | 20 GiB | 3 | 2 SM clusters |
| `3g.40gb` | 3/7 | 40 GiB | 2 | Half the GPU |
| `4g.40gb` | 4/7 | 40 GiB | 1 | More compute, same memory as 3g |
| `7g.80gb` | 7/7 | 80 GiB | 1 | Full GPU as single MIG instance |

**H100 80GB SXM profiles** (same structure, same max 7 instances):

| Profile | SMs | HBM | Max instances |
|---|---|---|---|
| `1g.10gb` | 1/7 | 10 GiB | 7 |
| `1g.10gb+me` | 1/7 | 10 GiB | 1 |
| `1g.20gb` | 1/7 | 20 GiB | 4 |
| `2g.20gb` | 2/7 | 20 GiB | 3 |
| `3g.40gb` | 3/7 | 40 GiB | 2 |
| `4g.40gb` | 4/7 | 40 GiB | 1 |
| `7g.80gb` | 7/7 | 80 GiB | 1 |

**H100 94GB PCIe** and **H100 96GB (GH200)** use different memory numbers per slice (`12gb`, `24gb`, `47gb`/`48gb`, `94gb`/`96gb`). Always run `nvidia-smi mig -lgip` on the actual hardware to see available profiles for your specific SKU.

### 31.6.3 MIG CLI Walkthrough

```bash
# Step 1: Enable MIG mode on GPU 0 (requires root; needs driver reload on some systems)
sudo nvidia-smi -i 0 -mig 1
# Enabled MIG Mode for GPU 00000000:01:00.0

# Verify MIG is enabled
nvidia-smi --query-gpu=index,mig.mode.current --format=csv
# 0, Enabled

# Step 2: List available GPU instance profiles
nvidia-smi mig -lgip
# GPU instance profile ID 9 is MIG 3g.40gb for GPU 0
# GPU instance profile ID 19 is MIG 1g.10gb for GPU 0

# Step 3: Create GPU instances (here: two 3g.40gb slices on A100 80GB)
sudo nvidia-smi mig -cgi 3g.40gb,3g.40gb
# Successfully created GPU instance ID  1 on GPU  0 using profile MIG 3g.40gb (ID  9)
# Successfully created GPU instance ID  2 on GPU  0 using profile MIG 3g.40gb (ID  9)

# Step 4: List created GPU instances
nvidia-smi mig -lgi
# GPU instance ID 1: MIG 3g.40gb, GPU 0
# GPU instance ID 2: MIG 3g.40gb, GPU 0

# Step 5: Create compute instances within each GI
sudo nvidia-smi mig -cci -gi 1
sudo nvidia-smi mig -cci -gi 2

# Step 6: Verify full MIG device tree
nvidia-smi
# MIG devices visible as GPU-<uuid>/0/0 and GPU-<uuid>/1/0

# Cleanup: destroy all CIs then GIs
sudo nvidia-smi mig -dci && sudo nvidia-smi mig -dgi
sudo nvidia-smi -i 0 -mig 0   # disable MIG mode
```

### 31.6.4 MIG in Kubernetes: NVIDIA GPU Operator

The **NVIDIA GPU Operator** (see also Chapter 13 for its SR-IOV integration) manages MIG configuration cluster-wide through a `MIG Manager` component. Two MIG strategies are supported:

- `single`: all GPUs on a node use the same MIG profile
- `mixed`: different nodes (or GPUs on the same node) can use different profiles

```yaml
# ClusterPolicy — enable MIG with mixed strategy
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  mig:
    strategy: mixed
```

Label a node to configure its MIG geometry:

```bash
# Set all GPUs on node gpu-node-01 to 2x 3g.40gb
kubectl label node gpu-node-01 nvidia.com/mig.config=all-3g.40gb --overwrite
```

The MIG Manager watches this label, stops GPU pods, reconfigures the hardware, and restarts the device plugin. MIG devices are then advertised as:

```
nvidia.com/mig-3g.40gb: 2   # per-node resource count
```

Pods request specific MIG slices:

```yaml
resources:
  limits:
    nvidia.com/mig-3g.40gb: "1"
```

### 31.6.5 NCCL Behavior Across MIG Slices

MIG instances are fully isolated hardware devices. There is no NVLink or inter-slice P2P DMA between MIG instances on the same physical GPU — they cannot perform GPU-to-GPU peer memory copies through the GPU fabric. NCCL collectives across MIG slices (whether on the same GPU or different GPUs) must travel through the network like any cross-node collective. This has two practical consequences:

1. A training job distributed across two `3g.40gb` slices on one A100 pays network bandwidth costs that a bare-metal job using the full GPU avoids through NVSwitch.
2. NCCL automatically discovers MIG topology and selects the appropriate transport (PCIe-level or network), so no special configuration is needed — but bandwidth expectations must account for the network path.

For inference workloads where each MIG slice serves an independent model shard, the isolation is exactly the right model: no coordination is needed, and the slice boundaries prevent one tenant's workload from impacting another's latency.

---

## 31.7 NVIDIA MPS: Multi-Process Service

**MPS** (**Multi-Process Service**) takes the opposite approach to MIG: rather than partitioning hardware, it merges multiple CUDA contexts into one, allowing processes from different users or ranks to share the GPU's execution resources. MPS is designed for MPI workloads where many ranks run the same kernel simultaneously — with MPS, those ranks share the GPU's SM scheduler rather than time-slicing through CUDA's normal exclusive-process model.

### 31.7.1 CUDA Context Sharing Model

Without MPS, each CUDA process gets its own context, and the GPU's context switch overhead (10s of microseconds) serializes access. With MPS, all client processes connect through a single **nvidia-cuda-mps-server** process that presents one unified context to the hardware. Kernel launches from multiple clients are queued together and executed concurrently on the SM array.

This is most valuable for small-batch inference (many concurrent requests, each using a fraction of the SMs) and MPI training runs where 8 ranks share one GPU (common in CPU-limited HPC codes ported to GPU). It is counterproductive for large training jobs where each rank already fully occupies the GPU.

### 31.7.2 Starting the MPS Daemon

```bash
# Set GPU to exclusive-process mode (required for MPS)
sudo nvidia-smi -i 0 -c EXCLUSIVE_PROCESS

# Set environment variables for daemon location
export CUDA_MPS_PIPE_DIRECTORY=/tmp/nvidia-mps
export CUDA_MPS_LOG_DIRECTORY=/var/log/nvidia-mps
mkdir -p $CUDA_MPS_PIPE_DIRECTORY $CUDA_MPS_LOG_DIRECTORY

# Start control daemon (as root or GPU owner)
nvidia-cuda-mps-control -d

# Verify daemon is running
echo "get_server_list" | nvidia-cuda-mps-control
# Server PID: 12345

# Check server status
echo "get_default_active_thread_percentage" | nvidia-cuda-mps-control
# Active thread percentage: 100
```

### 31.7.3 Per-Client Memory and Thread Limits

On Volta and newer GPUs, MPS supports per-client resource limits — a critical safety feature for multi-tenant deployments:

```bash
# Set default GPU memory limit for all future MPS clients on device 0
# (8 GiB per client — on a 40 GiB GPU, allows ~5 concurrent clients)
echo "set_default_device_pinned_mem_limit 0 8G" | nvidia-cuda-mps-control

# Set default active thread percentage (% of SMs available to each client)
echo "set_default_active_thread_percentage 25" | nvidia-cuda-mps-control
# 25% of SMs per client → up to 4 concurrent clients at full SM utilization

# Verify
echo "get_default_device_pinned_mem_limit 0" | nvidia-cuda-mps-control
# Device 0 default pinned memory limit: 8 GiB
```

Clients can also set their own limits via environment variables before CUDA initialization:

```bash
# Per-process limit (set before launching CUDA application)
export CUDA_MPS_PINNED_DEVICE_MEM_LIMIT="0=8G"
# Format: <device_ordinal>=<limit>[,<device_ordinal>=<limit>]
```

### 31.7.4 Stopping the MPS Daemon

```bash
echo "quit" | nvidia-cuda-mps-control
# Daemon stopped

# Restore GPU to default (shared) mode
sudo nvidia-smi -i 0 -c DEFAULT
```

### 31.7.5 MPS vs MIG: Isolation vs Sharing Granularity

| Dimension | MIG | MPS |
|---|---|---|
| Isolation enforcement | Hardware (GPU fault boundaries) | Software (memory limits, % control) |
| HBM partitioning | Hard (dedicated per slice) | Soft (limits, but shared physical pool) |
| Context switch overhead | None (dedicated hardware) | None (shared context) |
| Fault containment | Full (slice fault doesn't affect others) | Partial (misconfigured client can OOM the shared pool) |
| Concurrent workloads | Up to 7 (H100/A100) | Up to 48 (hardware queue limit) |
| Kubernetes support | Yes (GPU Operator, native resources) | Limited (requires MPS device plugin or DaemonSet) |
| All-reduce latency impact | High (network path between slices) | Low (shared SM, but serialized SM access at high load) |

**Rule of thumb**: use MIG when you need hard isolation guarantees between tenants (security-conscious multi-tenancy, mixed SLA tiers). Use MPS when you need to multiplex many small CUDA workloads that individually underutilize the GPU (small-batch inference, MPI jobs with many ranks per GPU).

---

## 31.8 KubeVirt: GPU VMs Inside Kubernetes

**KubeVirt** extends Kubernetes to run full virtual machines as first-class workloads. A VM appears as a `VirtualMachine` (VM) or `VirtualMachineInstance` (VMI) custom resource; the `virt-launcher` pod runs QEMU/KVM under the hood. This model is valuable for workloads that require a full OS (kernel drivers, custom firmware, legacy software) while also needing Kubernetes orchestration.

### 31.8.1 Core CRDs and virtctl

KubeVirt introduces three key CRDs:
- `VirtualMachine` (VM): desired state with start/stop lifecycle management
- `VirtualMachineInstance` (VMI): the running instance; created by the VM controller
- `DataVolume` (DV): CDI (Containerized Data Importer) resource for VM disk provisioning

**virtctl** is the KubeVirt CLI that extends `kubectl` for VM operations:

```bash
# Start / stop a VM
virtctl start my-gpu-vm
virtctl stop my-gpu-vm

# SSH into a running VMI
virtctl ssh --local-ssh my-gpu-vm

# Initiate live migration (GPU VMs will be rejected — see §31.8.4)
virtctl migrate my-gpu-vm

# Console access
virtctl console my-gpu-vm
```

### 31.8.2 GPU Passthrough via `devices.gpus`

Before assigning a GPU to a VMI, the device must be listed in the KubeVirt CR's `permittedHostDevices` section. This is an **administrator-level** change to the KubeVirt CR itself — it cannot be set by individual users on a per-VM basis.

```yaml
# Patch KubeVirt CR to allow GPU passthrough
apiVersion: kubevirt.io/v1
kind: KubeVirt
metadata:
  name: kubevirt
  namespace: kubevirt
spec:
  configuration:
    permittedHostDevices:
      pciHostDevices:
      - pciVendorSelector: "10DE:2204"       # NVIDIA GA102 (RTX 3090)
        resourceName: "nvidia.com/GA102"
        externalResourceProvider: true       # NVIDIA GPU device plugin manages this
      - pciVendorSelector: "10DE:20B5"       # NVIDIA A100 80GB PCIe
        resourceName: "nvidia.com/A100_80GB"
        externalResourceProvider: true
```

A VirtualMachineInstance with GPU passthrough:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: gpu-vmi
spec:
  domain:
    cpu:
      cores: 16
    memory:
      guest: 64Gi
    features:
      smm:
        enabled: true      # required for OVMF UEFI
    firmware:
      bootloader:
        efi:
          secureBoot: false
    devices:
      disks:
      - name: rootdisk
        disk:
          bus: virtio
      gpus:
      - name: gpu1
        deviceName: nvidia.com/GA102   # matches pciHostDevices resourceName
  volumes:
  - name: rootdisk
    dataVolume:
      name: gpu-vmi-dv
```

### 31.8.3 DataVolume for VM Disk Provisioning

**DataVolume** is the CDI mechanism for importing, uploading, or cloning VM disk images into PVCs. It decouples disk provisioning from VM start-up.

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: gpu-vmi-dv
spec:
  source:
    http:
      url: "https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img"
  pvc:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 64Gi
    storageClassName: local-path
```

### 31.8.4 Network Binding Modes

KubeVirt supports three primary network binding modes for VMI interfaces:

1. **masquerade** (default): NAT-based connectivity through the pod network; simpler but adds NAT overhead
2. **bridge**: bridges the VMI into the pod network; requires the pod interface to be configured for passthrough
3. **SR-IOV** (`sriov: {}`): direct VF assignment for near-line-rate networking; required for RDMA in VMIs

A complete VMI with both primary (masquerade) and secondary SR-IOV NIC:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: gpu-vm
spec:
  running: true
  template:
    spec:
      domain:
        devices:
          interfaces:
          - name: default
            masquerade: {}      # primary: NAT through pod network
          - name: sriov-net
            sriov: {}           # secondary: SR-IOV VF direct assignment
          gpus:
          - name: gpu1
            deviceName: nvidia.com/GA102
      networks:
      - name: default
        pod: {}
      - name: sriov-net
        multus:
          networkName: default/sriov-rdma-net   # NetworkAttachmentDefinition name
```

The `NetworkAttachmentDefinition` `sriov-rdma-net` is created by the SR-IOV Network Operator (Chapter 13) and references an SR-IOV VF from a Mellanox ConnectX NIC with RDMA capability.

### 31.8.5 Live Migration Constraints

Live migration with VFIO-assigned devices (both GPU passthrough via `devices.gpus` and SR-IOV NIC via `sriov: {}`) is **not possible** in current KubeVirt versions. VFIO assignment transfers device state into the guest's address space in a way that cannot be extracted and replicated to a destination host. Attempting to migrate a VMI with assigned devices will fail:

```
virtctl migrate gpu-vm
# Error: LiveMigration is not permitted for VMI gpu-vm with assigned host devices
```

Implications for HA design:
- GPU VMIs should be treated like bare-metal nodes for scheduling purposes
- Use pod anti-affinity to avoid colocating GPU VMIs with critical workloads that need live migration
- For Proxmox-based clusters, fencing + cold restart (§31.5.6) is the correct HA mechanism
- vGPU or MIG-based VMs (where no VFIO assignment occurs at the VM level) can in principle support live migration, subject to vendor licensing

---

## 31.9 Networking Consequences: RDMA, GPUDirect, and Isolation

The choice of GPU isolation model has profound consequences for RDMA and **GPUDirect** capability. This section explains the physical mechanism and what each model preserves or breaks.

### 31.9.1 GPUDirect RDMA Mechanism

**GPUDirect RDMA** enables a NIC to DMA directly into or out of GPU HBM without staging through CPU DRAM. The mechanism relies on the GPU driver registering GPU memory regions with the kernel's peer memory framework, which allows the RDMA verbs stack to obtain DMA mappings to GPU memory when an application posts a Send/Receive with a GPU memory buffer as the scatter-gather element.

Two kernel-level implementations exist:
- **nvidia-peermem**: the legacy kernel module from the GPU driver package that registers GPU memory with the RDMA subsystem. Supported since CUDA 5.0, but requires explicit loading.
- **DMA-BUF**: the mainline Linux kernel mechanism (introduced in kernel 5.12) for sharing DMA buffers between devices. NVIDIA recommends DMA-BUF over `nvidia-peermem` on kernels 5.12+ as it integrates with the standard driver model without a separate module.

```bash
# Load nvidia-peermem (legacy, pre-5.12 kernels or explicit requirement)
sudo modprobe nvidia-peermem

# Verify DMA-BUF peer support (modern kernels)
nvidia-smi | grep -i "DMA"
# Look for "peer-to-peer" support flag

# Test GPUDirect: requires physical NIC with RDMA and GPU on same PCIe root complex
# (cannot be tested without hardware)
```

### 31.9.2 RDMA Device Visibility in VMs

For a VM to perform RDMA operations, the RDMA NIC must be visible inside the VM as a device the guest OS can program. There are two ways to achieve this:

**VFIO direct assignment of an RDMA NIC VF**: an SR-IOV Virtual Function from a Mellanox ConnectX NIC is passed through via VFIO (as the `<hostdev>` in libvirt XML or `sriov: {}` in KubeVirt). The guest loads `mlx5_core` and sees a full RDMA device:

```bash
# Inside the VM, after loading mlx5_core
ibv_devinfo
# hca_id:   mlx5_0
#   transport:                  InfiniBand (0)
#   fw_ver:                     28.39.1002
#   node_guid:                  ...
#   phys_port_cnt:              1
#   active_mtu:                 4096 (5)
#   active_speed:               25 Gb/sec (8)
```

**virtio-net (software path)**: standard paravirtual networking. No RDMA capability. NCCL falls back to socket-based transport. Bandwidth and latency are orders of magnitude worse for collective-intensive workloads.

The rule is simple: if a GPU VM needs GPUDirect RDMA, it must have an SR-IOV VF from an RDMA-capable NIC passed through via VFIO — virtio-net cannot carry RDMA semantics.

### 31.9.3 GPUDirect with VFIO Passthrough

When both the GPU and an RDMA NIC VF are assigned to the same VM via VFIO, the GPUDirect path works through the guest driver stack with one important constraint: the DMA-BUF or `nvidia-peermem` mechanism must work within the guest's view of the IOMMU. Specifically:

- The GPU and the RDMA NIC VF must be in the same PCIe domain visible to the IOMMU in the guest
- The guest IOMMU (if configured) must permit peer-to-peer DMA between the GPU and NIC
- `nvidia-peermem` or DMA-BUF must be loaded inside the guest, not just on the host

For KVM/QEMU with `iommu=pt` on the host, the guest sees the physical PCIe topology through the virtual IOMMU, and peer-to-peer DMA is permitted. For Hyper-V or ESXi hypervisors (not covered in this chapter), additional configuration may be required.

### 31.9.4 IOMMU and DMA-Remapping Impact on GPU P2P

The IOMMU adds one level of address translation to every DMA operation. For GPU-to-GPU peer-to-peer copies via NVLink (which bypass the PCIe bus entirely), the IOMMU is not involved — NVLink P2P has its own address space managed by the GPU driver. For GPU-to-NIC P2P (GPUDirect RDMA via PCIe), the IOMMU translates the NIC's DMA requests, adding a small latency (typically <100 ns) per request but not affecting throughput significantly.

With `iommu=pt` (passthrough mode) on the host, devices assigned to VMs are translated through the IOMMU, while host devices bypass translation. This is the recommended configuration for GPU passthrough hosts.

### 31.9.5 Why vGPU Abstractions Break GPUDirect

**NVIDIA vGPU** and **AMD MxGPU** virtualize the GPU interface in a way that is fundamentally incompatible with GPUDirect RDMA. The vGPU manager intercepts GPU memory allocation and remaps it through a virtual address space that the NIC cannot access directly — there is no contiguous physical GPU memory region that the RDMA stack can register as a peer memory region. The vGPU driver presents a virtual framebuffer, not a physical HBM window.

Concretely: if a guest CUDA application using vGPU calls `ibv_reg_mr()` with a GPU buffer, the call will fail or silently fall back to CPU-memory staging, eliminating the zero-copy path. This is not a software limitation that will eventually be fixed — it is a consequence of how virtual GPU framebuffer management works.

For AI training workloads that depend on GPUDirect RDMA (which is nearly all collective-communication-heavy training using NCCL), vGPU is not a viable isolation model. The only viable alternatives are:
- Bare metal (full GPUDirect)
- VFIO passthrough with SR-IOV NIC VF (full GPUDirect through virtual IOMMU)
- MIG slices (GPUDirect within a slice; network path for cross-slice)

---

## 31.10 Decision Matrix: Choosing the Right Model

| Criterion | Bare metal | VFIO passthrough | MIG | MPS | KubeVirt + passthrough | vGPU / MxGPU |
|---|---|---|---|---|---|---|
| **Perf overhead** | None | <1% | None (within slice) | ~2–5% at high contention | <1% | 10–30% |
| **Isolation strength** | None | Strong (IOMMU) | Strong (hardware) | Weak (soft limits) | Strong (IOMMU) | Moderate (driver-level) |
| **GPUDirect RDMA** | Full | Full (NIC VF passthrough) | Per-slice | Full (same GPU) | Full (NIC VF passthrough) | No |
| **Live migration** | No | No | No | No | No | Yes (licensed) |
| **Kubernetes-native** | Yes (device plugin) | Partial (VM-level) | Yes (GPU Operator) | Partial (DaemonSet) | Yes (KubeVirt) | Vendor-specific |
| **Tenants per GPU** | 1 | 1 | 2–7 | Up to 48 | 1 | 4–16 |
| **Workload fit** | LLM training | GPU VM, legacy apps | Multi-tenant inference, mixed training | MPI, small-batch inference | Kubernetes-managed GPU VMs | VDI, compliance-isolated inference |

**Concrete recommendations by workload type:**

- **Large-scale distributed training** (NCCL all-reduce dominated): bare metal. Any virtualization layer adds latency to the critical path.
- **Single-tenant GPU VM** (legacy app, Windows guest, custom OS): VFIO passthrough with SR-IOV NIC VF for RDMA.
- **Multi-tenant inference serving** (multiple models on one GPU, hard SLA isolation): MIG. Each model gets a fixed slice; no cross-tenant interference.
- **MPI fine-tuning with many ranks per GPU**: MPS. Shared SM scheduler eliminates context switch serialization between ranks.
- **GPU VMs in Kubernetes** (operator needs full OS, or KubeVirt is platform standard): KubeVirt + `devices.gpus` with SR-IOV NIC VF.
- **VDI / enterprise desktop**: NVIDIA vGPU or AMD MxGPU. Live migration and vMotion compatibility outweigh GPUDirect loss.

---

## Lab: Phase A — Proxmox GPU Passthrough (Requires Physical Hardware)

> **Hardware required**: a physical host with Intel VT-d or AMD IOMMU, a supported GPU (NVIDIA), and Proxmox VE 8.x installed. Steps are numbered and show expected output.

### Step 1: Enable IOMMU

Edit `/etc/default/grub` on the Proxmox host:

```bash
# Intel host
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet"/GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"/' \
  /etc/default/grub
update-grub && update-initramfs -u -k all
reboot
```

After reboot:

```bash
dmesg | grep -i iommu | head -5
# [    0.483321] DMAR: IOMMU enabled
# [    0.483325] DMAR-IR: Enabled IRQ remapping in x2apic mode
```

### Step 2: Load VFIO Modules

```bash
echo -e "vfio\nvfio_iommu_type1\nvfio_pci" >> /etc/modules
update-initramfs -u -k all
reboot

# Verify
lsmod | grep vfio
# vfio_pci         57344  0
# vfio_iommu_type1 40960  0
# vfio               36864  2 vfio_pci,vfio_iommu_type1
```

### Step 3: Identify GPU PCI Address

```bash
lspci -nn | grep -i nvidia
# 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [10de:2204] (rev a1)
# 01:00.1 Audio device [0403]: NVIDIA Corporation GA102 HD Audio [10de:1aef]
```

### Step 4: Dump ROM (if needed)

```bash
echo 1 > /sys/bus/pci/devices/0000:01:00.0/rom
cat /sys/bus/pci/devices/0000:01:00.0/rom > /usr/share/kvm/ga102.rom
echo 0 > /sys/bus/pci/devices/0000:01:00.0/rom
ls -lh /usr/share/kvm/ga102.rom
# -rw-r--r-- 1 root root 512K /usr/share/kvm/ga102.rom
```

### Step 5: Create and Configure GPU VM

```bash
qm create 100 --name gpu-passthrough-vm --memory 32768 --cores 8 \
  --machine q35 --bios ovmf --cpu host,hidden=1 \
  --efidisk0 local-lvm:1,format=raw,efitype=4m,pre-enrolled-keys=0 \
  --net0 virtio,bridge=vmbr0

qm set 100 --hostpci0 01:00,pcie=1,x-vga=1,rombar=1,romfile=ga102.rom
qm set 100 --virtio0 local-lvm:60,format=raw
qm set 100 --boot order=virtio0
qm set 100 --hugepages 1024

qm config 100 | grep hostpci
# hostpci0: 0000:01:00,pcie=1,rombar=1,romfile=ga102.rom,x-vga=1
```

### Step 6: Start VM and Verify GPU Inside

```bash
qm start 100
qm status 100
# status: running

# SSH into the VM (install drivers first via a console)
virtctl ssh ubuntu@100
# Inside VM:
lspci | grep -i nvidia
# 05:00.0 VGA compatible controller: NVIDIA Corporation GA102 [10de:2204]
nvidia-smi
# +-----------------------------------------------------------------------------+
# | NVIDIA-SMI 550.x    Driver Version: 550.x    CUDA Version: 12.4            |
# |---------------------------+----------------------+-------------------------+
# | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
# |   0  NVIDIA GeForce RTX 3090  Off | 00000000:05:00.0 Off |                  N/A |
```

---

## Lab: Phase B — KubeVirt VMI with SR-IOV Secondary NIC (No GPU Required)

This phase demonstrates the full KubeVirt VMI deployment with SR-IOV secondary NIC attachment and RDMA device verification using Soft-RoCE, runnable on any Linux laptop with Kind.

### Step 1: Create Kind Cluster and Install KubeVirt

```bash
# Create a Kind cluster with nested virtualization support
cat <<'EOF' > kind-kubevirt.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  extraMounts:
  - hostPath: /dev/kvm
    containerPath: /dev/kvm
EOF
kind create cluster --config kind-kubevirt.yaml --name kubevirt-lab

# Deploy KubeVirt operator and CR
export KUBEVIRT_VERSION=v1.3.0
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-cr.yaml

# Wait for all KubeVirt components
kubectl wait --for=condition=Available --timeout=300s -n kubevirt kv/kubevirt
# condition met
```

### Step 2: Install Multus and SR-IOV (Soft-RoCE simulation)

```bash
# Multus CNI (as in Chapter 13)
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml

# Load Soft-RoCE on the Kind worker node
docker exec kubevirt-lab-worker modprobe rdma_rxe

# Add a Soft-RoCE device on eth0 inside the worker
docker exec kubevirt-lab-worker rdma link add rxe0 type rxe netdev eth0

# Create a NetworkAttachmentDefinition referencing the Soft-RoCE device
kubectl apply -f - <<'EOF'
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: rdma-net
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "rdma-net",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24"
      }
    }
EOF
```

### Step 3: Deploy VMI with Secondary Network Attachment

```bash
# First, deploy CDI for DataVolume support
export CDI_VERSION=v1.59.0
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-operator.yaml
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-cr.yaml

# Create a DataVolume with a Cirros test image (small, fast download)
kubectl apply -f - <<'EOF'
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: cirros-dv
spec:
  source:
    http:
      url: "https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
  pvc:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
    storageClassName: standard
EOF

kubectl wait --for=condition=Ready --timeout=300s dv/cirros-dv
```

```bash
# Deploy VMI with primary masquerade + secondary network attachment
kubectl apply -f - <<'EOF'
apiVersion: kubevirt.io/v1
kind: VirtualMachineInstance
metadata:
  name: rdma-vmi
  annotations:
    k8s.v1.cni.cncf.io/networks: rdma-net
spec:
  domain:
    cpu:
      cores: 2
    memory:
      guest: 512Mi
    devices:
      disks:
      - name: rootdisk
        disk:
          bus: virtio
      - name: cloudinit
        disk:
          bus: virtio
      interfaces:
      - name: default
        masquerade: {}
      - name: rdma-net
        bridge: {}          # bridge to the macvlan attachment
  networks:
  - name: default
    pod: {}
  - name: rdma-net
    multus:
      networkName: default/rdma-net
  volumes:
  - name: rootdisk
    dataVolume:
      name: cirros-dv
  - name: cloudinit
    cloudInitNoCloud:
      userDataBase64: ""
EOF

kubectl wait --for=condition=Ready --timeout=300s vmi/rdma-vmi
# condition met
```

### Step 4: Verify Secondary NIC Attachment (SR-IOV Workflow Simulation)

> **Scope of this phase**: Phase B demonstrates the full KubeVirt SR-IOV network binding workflow on any laptop. Because Kind worker nodes run as containers (not physical machines with SR-IOV-capable NICs), RDMA device visibility **inside the VMI** cannot be verified here — that requires a physical host with a Mellanox/Intel SR-IOV NIC whose VF is passed through via VFIO (Phase A scenario). The steps below verify the secondary interface attachment and confirm the Soft-RoCE device on the host node, which is the closest simulation achievable in this environment.

```bash
# Console into the VMI
virtctl console rdma-vmi
# (login: cirros / gocubsgo)

# Inside VMI — verify secondary interface was attached by Multus
ip addr show
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...  (pod network)
# 3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> inet 192.168.100.x/24  (secondary)

# Exit VMI console (Ctrl+])

# On the Kind worker host — verify the Soft-RoCE device is visible at the host level
# In a real SR-IOV passthrough deployment, this step runs INSIDE the VMI
# after the mlx5_core driver loads on the passed-through VF
docker exec kubevirt-lab-worker ibv_devinfo
# hca_id: rxe0
#   transport:            InfiniBand (0)
#   fw_ver:               0.0.0
#   node_guid:            ...
#   phys_port_cnt:        1
#   active_mtu:           1024 (3)

# On a real node with a physical RDMA VF passed through to the VMI,
# the ibv_devinfo output would instead appear inside the VMI, e.g.:
# hca_id:  mlx5_0
#   transport:  InfiniBand (0)
#   fw_ver:     28.39.1002
#   ...
```

> **Production RDMA inside a VMI**: With a physical Mellanox ConnectX-5/6/7 NIC, the SR-IOV Network Operator allocates a VF, KubeVirt binds it via `sriov: {}`, and the VMI guest kernel loads `mlx5_core` on the passed-through VF. Inside the VMI, `ibv_devinfo` reports the physical NIC's RDMA capabilities and `ib_write_bw` benchmarks run at full NIC line rate. The Soft-RoCE (`rdma_rxe`) device in this lab is a functional substitute for verifying the binding workflow; it does not replicate the performance characteristics or GPUDirect capability of a physical passthrough VF. See Chapter 13 for the full SR-IOV Network Operator setup required for physical deployments.

### Step 5: Cleanup

```bash
kubectl delete vmi rdma-vmi
kubectl delete dv cirros-dv
kind delete cluster --name kubevirt-lab
```

---

## References

- VFIO Kernel Documentation: https://www.kernel.org/doc/html/latest/driver-api/vfio.html
- VFIO-PCI Device Assignment: https://www.kernel.org/doc/html/latest/driver-api/vfio-mediated-device.html
- ACS Override Patch (out-of-tree): https://github.com/benbaker76/linux-acs-override
- Proxmox VE PCIe Passthrough Wiki: https://pve.proxmox.com/wiki/PCI(e)_Passthrough
- Proxmox VE Administration Guide: https://pve.proxmox.com/pve-docs/pve-admin-guide.html
- Proxmox Ceph Integration: https://pve.proxmox.com/wiki/Deploy_Hyper-Converged_Ceph_Cluster
- NVIDIA MIG User Guide: https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html
- NVIDIA MIG Supported Profiles: https://docs.nvidia.com/datacenter/tesla/mig-user-guide/supported-mig-profiles.html
- NVIDIA GPU Operator with MIG: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html
- NVIDIA MPS Overview: https://docs.nvidia.com/deploy/mps/index.html
- NVIDIA MPS Tools Reference: https://docs.nvidia.com/deploy/mps/appendix-tools-and-interface-reference.html
- GPUDirect RDMA Documentation: https://docs.nvidia.com/cuda/gpudirect-rdma/
- NVIDIA GPU Operator GPUDirect RDMA: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-rdma.html
- KubeVirt User Guide — Host Devices: https://kubevirt.io/user-guide/compute/host-devices/
- KubeVirt User Guide — Interfaces and Networks: https://kubevirt.io/user-guide/network/interfaces_and_networks/
- KubeVirt User Guide — Live Migration: https://kubevirt.io/user-guide/compute/live_migration/
- NVIDIA GPU Operator with KubeVirt: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-kubevirt.html
- libvirt Domain XML Format: https://libvirt.org/formatdomain.html
- OVMF (UEFI for QEMU/KVM): https://github.com/tianocore/tianocore.github.io/wiki/OVMF
- QEMU Documentation: https://www.qemu.org/docs/master/system/qemu-manpage.html
- Corosync Cluster Engine: https://corosync.github.io/corosync/
- CDI (Containerized Data Importer): https://github.com/kubevirt/containerized-data-importer

---

© 2025 jreuben1. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
