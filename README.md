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

## Documents

1. **[01-vfio-gpu-passthrough.md](01-vfio-gpu-passthrough.md)** — VFIO setup, IOMMU, what worked
2. **[02-looking-glass-and-vdd.md](02-looking-glass-and-vdd.md)** — Looking Glass host/client, VDD virtual display, the chicken-and-egg problem, resolution switching
3. **[03-mouse-pointer-fix.md](03-mouse-pointer-fix.md)** — Mouse misalignment and the virtio-vga tradeoff
4. **[04-audio-attempts.md](04-audio-attempts.md)** — Every audio approach tried, why each failed, and lessons learned
5. **[05-methodology.md](05-methodology.md)** — Snapshot-based rollback, guest agent remote management, incremental change strategy
6. **[06-remaining-issues.md](06-remaining-issues.md)** — What's still broken and proposed next steps
7. **[07-windows-guest-setup.md](07-windows-guest-setup.md)** — Everything done inside Windows: OOBE bypass, guest agent, LG host, VDD, resolution script, auto-login, Scream, SPICE config
8. **[08-vm-xml-reference.md](08-vm-xml-reference.md)** — Complete annotated VM XML and design decisions
9. **[09-remote-command-execution.md](09-remote-command-execution.md)** — Running commands in the VM headlessly via guest agent, what works vs session 0 limitations

## Summary of Outcomes

| Component | Outcome | Key Lesson |
|-----------|---------|------------|
| GPU Passthrough | Worked | Standard VFIO bind; isolate GPU in its own IOMMU group |
| Looking Glass Video | Worked | VDD provides the virtual display NVIDIA needs to render |
| VDD Resolution | Worked (with caveat) | `ChangeDisplaySettingsEx` requires interactive session, not guest agent |
| Mouse Input | Worked | Removing virtio-vga (`<model type='none'/>`) fixed misalignment |
| Audio | Failed | Every approach hit a wall — see audio doc |
| Boot-time Display | Partial | VDD only activates after interactive login (chicken-and-egg) |
