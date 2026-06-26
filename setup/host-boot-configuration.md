# Host Boot Configuration

Configuring the Arch Linux host to reserve the GPU for the VM via VFIO. This involves kernel module loading in the initramfs, a udev rule to bind the GPU to `vfio-pci`, and a toggle script to switch between VM mode and host mode.

## The Problem

The host has two identical NVIDIA RTX 3090 GPUs:
- `0b:00.0` — reserved for the Windows VM (bound to `vfio-pci`)
- `0c:00.0` — used by the host (bound to `nvidia`)

By default, both GPUs would bind to the `nvidia` driver at boot. The GPU intended for the VM must be bound to `vfio-pci` instead, so QEMU can claim it. This binding must happen at boot time (in the initramfs), before the `nvidia` driver grabs it.

## mkinitcpio.conf Changes

### Original (stock Arch)

```ini
MODULES=(amdgpu)
FILES=()
```

### Modified (VFIO enabled)

```ini
MODULES=(vfio vfio_iommu_type1 vfio_pci amdgpu)
FILES=(/etc/udev/rules.d/61-vfio-pci-bind.rules)
```

### What Changed

1. **Added VFIO modules to `MODULES`**: `vfio`, `vfio_iommu_type1`, `vfio_pci` are loaded in the initramfs before any other driver can claim the GPU. `amdgpu` was already present (the host also has an AMD APU/iGPU).
2. **Added udev rules file to `FILES`**: The udev rule that binds the specific GPU to `vfio-pci` is baked into the initramfs so it runs at boot.

### Why Not Use modprobe.conf?

The traditional approach is to set `options vfio-pci ids=10de:2204,10de:1aef` in a modprobe config file. This works but binds **all** devices matching those IDs to vfio-pci — which would grab both 3090s since they have the same vendor/device IDs. The udev rule approach is more precise: it targets specific PCI addresses (`0000:0b:00.0` and `0000:0b:00.1`).

## udev Rule

File: `/etc/udev/rules.d/61-vfio-pci-bind.rules`

```
ACTION=="add", SUBSYSTEM=="pci", KERNEL=="0000:0b:00.0", ATTR{driver_override}="vfio-pci", RUN+="/bin/sh -c 'echo 0000:0b:00.0 > /sys/bus/pci/drivers/vfio-pci/bind'"
ACTION=="add", SUBSYSTEM=="pci", KERNEL=="0000:0b:00.1", ATTR{driver_override}="vfio-pci", RUN+="/bin/sh -c 'echo 0000:0b:00.1 > /sys/bus/pci/drivers/vfio-pci/bind'"
```

### How It Works

- Matches PCI devices by their kernel name (PCI address)
- Sets `driver_override` to `vfio-pci` — tells the kernel this device should use vfio-pci
- Then explicitly binds the device to the vfio-pci driver
- This runs at boot from the initramfs (because the rule file is in `FILES=`)

### Why Two Rules?

The GPU has two PCI functions:
- `0000:0b:00.0` — VGA controller (the GPU itself)
- `0000:0b:00.1` — HDMI audio controller

Both must be bound to vfio-pci for the VM to use the GPU properly.

## Toggle Script

Since the GPU is sometimes needed on the host (e.g., for compute workloads when the VM isn't running), a toggle script switches between "VM mode" (GPU bound to vfio-pci) and "host mode" (GPU available to nvidia driver).

File: `/usr/local/bin/toggle-gpu`

```bash
#!/bin/bash
case "${1:-}" in
    vm)
        # Write udev rule to bind GPU to vfio-pci
        sudo tee /etc/udev/rules.d/61-vfio-pci-bind.rules > /dev/null << 'EOF'
ACTION=="add", SUBSYSTEM=="pci", KERNEL=="0000:0b:00.0", ATTR{driver_override}="vfio-pci", RUN+="/bin/sh -c 'echo 0000:0b:00.0 > /sys/bus/pci/drivers/vfio-pci/bind'"
ACTION=="add", SUBSYSTEM=="pci", KERNEL=="0000:0b:00.1", ATTR{driver_override}="vfio-pci", RUN+="/bin/sh -c 'echo 0000:0b:00.1 > /sys/bus/pci/drivers/vfio-pci/bind'"
EOF
        # Ensure mkinitcpio.conf includes the udev rule in FILES
        if ! grep -q 'FILES=(/etc/udev/rules.d/61-vfio-pci-bind.rules)' /etc/mkinitcpio.conf; then
            echo 'FILES=(/etc/udev/rules.d/61-vfio-pci-bind.rules)' | sudo tee -a /etc/mkinitcpio.conf
        fi
        # Regenerate initramfs
        sudo mkinitcpio -P > /dev/null 2>&1
        echo "GPU set to VM mode — reboot now"
        ;;
    host)
        # Remove udev rule
        sudo rm -f /etc/udev/rules.d/61-vfio-pci-bind.rules
        # Remove FILES line from mkinitcpio.conf
        sudo sed -i '/61-vfio-pci-bind.rules/d' /etc/mkinitcpio.conf
        # Regenerate initramfs
        sudo mkinitcpio -P > /dev/null 2>&1
        echo "GPU set to host mode — reboot now"
        ;;
    status)
        driver=$(lspci -nnk -s 0b:00.0 | grep "Kernel driver in use" | awk '{print $5}')
        case "$driver" in
            vfio-pci) echo "Current: VM mode (GPU bound to vfio-pci)" ;;
            nvidia)   echo "Current: Host mode (GPU on nvidia driver)" ;;
            *)        echo "Current: Unknown ($driver)" ;;
        esac
        ;;
    *)
        echo "Usage: sudo toggle-gpu {vm|host|status}"
        echo "  vm     - bind GPU to vfio-pci for Windows VM"
        echo "  host   - free GPU for host Linux use"
        echo "  status - show current GPU binding"
        exit 1
        ;;
esac
```

### Usage

```bash
# Reserve GPU for the VM
sudo toggle-gpu vm
# → reboots, GPU binds to vfio-pci

# Free GPU for host use
sudo toggle-gpu host
# → reboots, GPU binds to nvidia

# Check current state
toggle-gpu status
# → Current: VM mode (GPU bound to vfio-pci)
```

### Why a Reboot Is Required

The binding happens in the initramfs at boot. There's no clean way to unbind a GPU from `nvidia` and rebind to `vfio-pci` (or vice versa) at runtime without risking crashes — the `nvidia` driver holds references to the GPU memory and contexts. The script modifies the configuration and regenerates the initramfs; the actual binding change takes effect on next boot.

## Kernel Parameters

The boot loader must also have IOMMU enabled:

```
amd_iommu=on iommu=pt
```

- `amd_iommu=on` — enables AMD-Vi IOMMU (required for VFIO)
- `iommu=pt` — puts IOMMU in passthrough mode by default (better performance; only isolated devices get translated)

For Intel hosts, use `intel_iommu=on iommu=pt` instead.

## Lessons

- **Use PCI-address-based udev rules, not vendor/device ID modprobe options**, when you have two identical GPUs. This prevents accidentally grabbing the host's GPU.
- **Bake the udev rule into the initramfs** via `FILES=` in mkinitcpio.conf. Otherwise the rule runs too late (after `nvidia` has already claimed the GPU).
- **A toggle script is worth the effort.** Without it, switching between VM and host mode requires manual editing of multiple files and remembering the exact udev rule syntax.
- **Always regenerate the initramfs** (`mkinitcpio -P`) after changing mkinitcpio.conf or the udev rule. The changes don't take effect until the initramfs is rebuilt.
- **Reboot is the clean way to switch GPU modes.** Runtime rebinding is fragile and can crash the host.
