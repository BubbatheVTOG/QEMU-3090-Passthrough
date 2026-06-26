# Resources

External resources that were useful during this project. AI agents should consult these for deeper understanding or when troubleshooting.

## Core Documentation

| Resource | Description |
|----------|-------------|
| [libvirt Domain XML format](https://libvirt.org/formatdomain.html) | Complete reference for VM XML configuration elements |
| [QEMU PCI passthrough](https://wiki.qemu.org/Documentation/PCI-Passthrough) | QEMU's official VFIO passthrough guide |
| [Arch Wiki: PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) | The definitive Arch Linux VFIO guide — covers IOMMU groups, mkinitcpio, udev rules, libvirt XML |
| [Arch Wiki: Looking Glass](https://wiki.archlinux.org/title/Looking_Glass) | Arch-specific LG setup instructions |

## Looking Glass

| Resource | Description |
|----------|-------------|
| [Looking Glass Official Documentation](https://looking-glass.io/docs) | Official docs for B7 (host and client) |
| [Looking Glass GitHub](https://github.com/gnif/LookingGlass) | Source code, issues, releases |
| [Looking Glass B7 Release Notes](https://github.com/gnif/LookingGlass/releases) | Version-specific notes and known issues |

## VDD (Virtual Display Driver)

| Resource | Description |
|----------|-------------|
| [VDD (Virtual Display Driver) GitHub](https://github.com/M1cha/vdd) | The virtual display driver used to provide a display without a physical monitor |
| [VDD Releases](https://github.com/M1cha/vdd/releases) | Driver downloads and changelogs |

## Scream (Audio)

| Resource | Description |
|----------|-------------|
| [Scream GitHub](https://github.com/duncanthrax/scream) | Virtual audio driver for Windows (IVSHMEM and UDP modes) |
| [Scream Wiki](https://github.com/duncanthrax/scream/wiki) | Setup guides for IVSHMEM and network modes |

## virtio-win Drivers

| Resource | Description |
|----------|-------------|
| [Fedora virtio-win ISO](https://fedorapeople.org/groups/virt/virtio-win/direct-latest/) | Latest virtio-win ISO download (drivers for disk, network, balloon, guest agent, serial) |
| [virtio-win GitHub](https://github.com/virtio-win/virtio-win-pkg-scripts) | Build scripts and documentation |

## QEMU Guest Agent

| Resource | Description |
|----------|-------------|
| [QEMU Guest Agent Protocol](https://qemu-project.gitlab.io/qemu/interop/qemu-ga.html) | Full reference for guest-agent JSON commands |
| [libvirt: guest agent](https://libvirt.org/formatdomain.html#guest-agent-interface) | XML configuration for the guest agent channel |

## Windows Internals (for AI agents)

These explain concepts that are critical for headless VM management:

| Resource | Description |
|----------|-------------|
| [Windows Session 0 Isolation](https://learn.microsoft.com/en-us/windows/win32/services/session-0-isolation) | Why services can't interact with the desktop — explains the guest agent display API limitation |
| [ChangeDisplaySettingsEx API](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-changedisplaysettingsexw) | Win32 API for setting display resolution programmatically |
| [OOBE bypassnro](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-setup-automation-overview) | Windows setup automation — the `oobe\bypassnro` command skips Microsoft account requirement |
| [Windows Autologon](https://learn.microsoft.com/en-us/troubleshoot/windows-server/user-profiles-and-logon/turn-on-automatic-logon) | Registry configuration for automatic login |

## SPICE Protocol

| Resource | Description |
|----------|-------------|
| [SPICE protocol documentation](https://www.spice-space.org/documentation.html) | SPICE protocol spec — explains audio channel behavior |
| [SPICE user documentation](https://spice-space.org/docs.html) | Client and server configuration |

## Community Resources

| Resource | Description |
|----------|-------------|
| [r/VFIO](https://reddit.com/r/VFIO) | Community for GPU passthrough help and discussion |
| [PassthroughPOST](https://passthroughpo.st/) | VFIO passthrough blog and guides |
| [Level1Techs GPU Passthrough Forum](https://forum.level1techs.com/c/hardware/gpu-passthrough) | Active community forum for troubleshooting |

## Related Projects (not tried but may be useful)

| Resource | Description |
|----------|-------------|
| [ Sunshine + Moonlight](https://github.com/LizardByte/Sunshine) | Alternative to Looking Glass — streams via Moonlight protocol, has built-in audio support |
| [evdev passthrough](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_keyboard/mouse_via_evdev) | Alternative to SPICE for mouse/keyboard input — avoids the SPICE audio channel conflict |
| [DDU (Display Driver Uninstaller)](https://www.wagnardsoft.com/display-driver-uninstaller-ddu-) | Useful for clean NVIDIA driver reinstalls in the VM |
