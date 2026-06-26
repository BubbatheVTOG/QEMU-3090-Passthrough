# GPU Passthrough + Looking Glass: AI Agent Knowledge Base

This document is a knowledge base for AI agents working on VFIO GPU passthrough VMs with Looking Glass display capture. It captures what worked, what didn't, the reasoning behind each decision, and the process used to get there.

## The Problem

Set up a Windows VM with a passed-through dedicated GPU (NVIDIA RTX 3090), captured via Looking Glass at high refresh rate (2560x1440@144Hz), with working mouse input and audio — **without a physical monitor attached to the GPU**.

## Key Constraints Encountered

- **No physical dummy plug available** — had to use a software virtual display driver (VDD) instead.
- **No host firewall modifications allowed** — ruled out network-based audio streaming (Scream UDP).
- **Guest agent runs in session 0** — Windows display APIs (`ChangeDisplaySettingsEx`, `EnumDisplayDevices`) do NOT work from session 0; they require an interactive user session. This is a critical gotcha.
- **NVIDIA GPU requires an active display output** for Looking Glass host to capture its framebuffer. With no monitor and no VDD active, the host service logs "Failed to locate a valid output device."
- **SPICE audio channels conflict with Looking Glass client** — the LG client connects RECORD and PLAYBACK SPICE channels even when `spice:audio=no` is set, causing a "Protocol error. The server asked us to reconnect an already connected channel (RECORD)" crash.

## Current Status

| Component | Status | Notes |
|-----------|--------|-------|
| Host VFIO Configuration | Working | Toggle script switches GPU between VM/host modes |
| GPU Passthrough (VFIO) | Working | RTX 3090 bound to `vfio-pci` via udev rule |
| Looking Glass Video | Working | 2560x1440@144Hz via VDD virtual display |
| Mouse/Pointer Input | Working | SPICE tablet input, no misalignment |
| Audio | **Not working** | See [audio/audio-attempts.md](audio/audio-attempts.md) |
| Boot-time Display | Partial | VDD requires interactive login to activate (chicken-and-egg) |

## Document Index

### Setup
- **[setup/host-boot-configuration.md](setup/host-boot-configuration.md)** — mkinitcpio.conf changes, udev rules for VFIO binding, toggle-gpu script to switch GPU between VM and host modes
- **[setup/vfio-gpu-passthrough.md](setup/vfio-gpu-passthrough.md)** — VFIO setup, IOMMU, libvirt XML for hostdev passthrough, what worked
- **[setup/vm-xml-reference.md](setup/vm-xml-reference.md)** — Complete annotated VM XML with design decision table and backing chain
- **[setup/windows-guest-setup.md](setup/windows-guest-setup.md)** — Everything done inside Windows: OOBE bypass, guest agent, LG host, VDD, resolution script, auto-login, Scream, SPICE config

### Looking Glass
- **[looking-glass/looking-glass-and-vdd.md](looking-glass/looking-glass-and-vdd.md)** — Looking Glass host/client, VDD virtual display, the chicken-and-egg problem, resolution switching
- **[looking-glass/mouse-pointer-fix.md](looking-glass/mouse-pointer-fix.md)** — Mouse misalignment and the virtio-vga tradeoff

### Audio
- **[audio/audio-attempts.md](audio/audio-attempts.md)** — Every audio approach tried, why each failed, and lessons learned

### Operations
- **[operations/methodology.md](operations/methodology.md)** — Snapshot-based rollback, guest agent remote management, incremental change strategy
- **[operations/remote-command-execution.md](operations/remote-command-execution.md)** — Running commands in the VM headlessly via guest agent, what works vs session 0 limitations

### Reference
- **[resources.md](resources.md)** — Links to external documentation, tools, and community resources
- **[remaining-issues.md](remaining-issues.md)** — What's still broken and proposed next steps

## Quick Start

```bash
# 1. Set GPU to VM mode (requires reboot)
sudo toggle-gpu vm

# 2. Start the VM
virsh start win11

# 3. Wait for auto-login + resolution script (30-60 seconds)

# 4. Run Looking Glass client
looking-glass-client

# 5. To free GPU for host use later
sudo toggle-gpu host
```
