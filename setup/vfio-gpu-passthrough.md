# VFIO GPU Passthrough

## What Worked

Standard VFIO GPU passthrough using libvirt/qemu with an NVIDIA RTX 3090 on an AMD host. The process was straightforward once the GPU was isolated.

### Prerequisites

- AMD host with IOMMU enabled (`amd_iommu=on iommu=pt` kernel parameters)
- Two identical GPUs (one for host, one for guest) — this made isolation trivial since they were already in separate IOMMU groups
- `vfio-pci`, `vfio_iommu_type1`, `vfio` kernel modules loaded

### The GPU That Was Passed Through

- Bound to `vfio-pci` driver on the host (confirmed via `lspci -nnk` showing "Kernel driver in use: vfio-pci")
- Both the VGA controller (`0b:00.0`) and its audio controller (`0b:00.1`) were passed through as separate `<hostdev>` entries
- The host's GPU (`0c:00.0`) remained on the `nvidia` driver

### Libvirt XML Configuration

Key XML elements for the passthrough:

```xml
<!-- GPU VGA controller -->
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio'/>
  <source>
    <address domain='0x0000' bus='0x0b' slot='0x00' function='0x0'/>
  </source>
  <address type='pci' domain='0x0000' bus='0x06' slot='0x00' function='0x0'/>
</hostdev>

<!-- GPU Audio controller (HDMI audio) -->
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio'/>
  <source>
    <address domain='0x0000' bus='0x0b' slot='0x00' function='0x1'/>
  </source>
  <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
</hostdev>
```

### Other VM Settings That Mattered

- **OVMF/UEFI firmware** with Secure Boot — required for Windows 11 and for NVIDIA driver compatibility
- **CPU mode `host-passthrough`** with `topoext` feature — prevents Windows/NVIDIA from complaining about unknown CPU
- **CPU pinning** — all 16 vCPUs pinned to corresponding host threads. Not strictly required for functionality but helps latency-sensitive workloads.
- **Hyper-V enlightenments** (`relaxed`, `vapic`, `spinlocks`, `synic`, `stimer`, etc.) — standard for Windows guests
- **TPM** via `tpm-crb` emulator — required for Windows 11
- **virtio-win drivers** for disk (virtio-blk) and network (virtio-net)

## What Didn't Work / Gotchas

- **NVIDIA driver error code 43** was never encountered. `host-passthrough` CPU mode alone was sufficient — it hides the hypervisor signature so the NVIDIA driver sees a real CPU. No `<vendor_id>` spoofing under `<hyperv>` was needed.
- **The GPU audio controller must be passed through alongside the GPU.** Initially only the VGA function was passed through; the HDMI audio function must also be in its own `<hostdev>` entry or Windows won't properly initialize the GPU.

### Host-Side Boot Configuration

The VFIO module loading, udev rule, and toggle script are covered in [host-boot-configuration.md](host-boot-configuration.md). That doc includes the full `mkinitcpio.conf` changes, the udev rule that targets the GPU by PCI address, and the `toggle-gpu` script that switches between VM mode and host mode.

## Lessons

- Having two identical GPUs (one for host, one for guest) is the ideal setup — IOMMU isolation is automatic, no driver fighting, no need to unbind the primary GPU.
- Pass through ALL functions of the GPU (VGA + audio), not just the VGA function.
- Use `managed='yes'` on hostdev entries — libvirt handles binding/unbinding to vfio-pci automatically.
