# Remaining Issues

## 1. Audio (Unresolved)

Audio from the Windows guest to the Linux host is **not working**. Every approach tried hit a wall:

- SPICE audio — LG client bug, always connects channels despite config
- Scream IVSHMEM — driver doesn't send data on Windows 26200
- Scream UDP — blocked by user constraint (no firewall changes)
- QEMU audio backends — libvirt-qemu user can't access desktop audio session
- USB headset passthrough — not properly tested (was conflated with other changes)

### Recommended Next Step

**Retry USB audio passthrough in isolation.** This is the only approach that was never definitively ruled out. The previous failure was almost certainly caused by conflating the USB device addition with SPICE removal.

Process:
1. Snapshot the current working state
2. Add ONLY the USB audio device to the VM XML (change nothing else)
3. Boot the VM and test audio
4. If it works, snapshot. If it doesn't, revert and diagnose.

```xml
<!-- Add only this, change nothing else -->
<hostdev mode='subsystem' type='usb' managed='yes'>
  <source>
    <vendor id='0x????'/>
    <product id='0x????'/>
  </source>
</hostdev>
```

### Alternative Approaches Not Yet Tried

- **PulseAudio over TCP:** Configure PulseAudio on the host to listen on a TCP socket, configure the guest to send audio to it. Requires opening a port but on a private/libvirt network, not the host firewall per se.
- **Scream with a newer driver build:** The Scream project may have updates for Windows 11 25H2. Check for newer releases.
- **PipeWire with system-wide instance:** Run PipeWire as a system service (not per-user) so libvirt-qemu can access it. More complex but avoids the user/session permission issue.
- **Physical DP/HDMI dummy plug + audio extractor:** A hardware dummy plug that also has an audio output. Would require purchasing hardware.

## 2. Boot-Time Display (Partial)

VDD only activates after Windows loads the driver and an interactive session starts. This means:

- During BIOS/UEFI boot, there is no display
- During Windows boot (before login), there is no display
- If auto-login fails (e.g., after a Windows update that changes login behavior), the VM is headless

### Potential Solutions

- **Hardware dummy plug:** Would provide a display at all times, including boot. The cleanest solution but requires hardware.
- **Keep virtio-vga as boot display, switch to VDD after login:** virtio-vga provides a display during boot; a startup script switches the primary display to VDD after login. Risk: re-introduces the mouse misalignment issue (see [mouse fix](looking-glass/mouse-pointer-fix.md)) unless virtio-vga is disabled after the switch.
- **VDD with boot-start driver:** If VDD can be configured to load as a boot-start driver (before user login), it could provide a display earlier. Not tested.

## 3. SPICE Audio Channel Conflict (Unresolved)

Even with `spice:audio=no` in the Looking Glass client config, the client connects RECORD and PLAYBACK SPICE channels, causing a protocol error crash. This limits the ability to use SPICE for input (mouse/keyboard) without audio side effects.

### Potential Solutions

- **Report upstream:** This appears to be a bug in Looking Glass B7. The `spice:audio=no` config should prevent audio channel connections.
- **Patch the client:** Modify the LG client source to not connect audio channels when `spice:audio=no`. Requires building from source.
- **Use a different input method:** Instead of SPICE for mouse/keyboard, use evdev passthrough or USB tablet passthrough. This would allow removing SPICE entirely, eliminating the audio channel conflict.

## 4. Scream Driver Diagnosis (Unresolved)

The Scream driver was installed and configured but never sent audio data. Root cause unknown.

### Diagnostic Steps Not Yet Taken

- Check Windows Event Viewer for Scream driver errors
- Check if the Scream driver service is actually running (not just installed)
- Verify the IVSHMEM PCI device is visible to the Scream driver (not just to Windows)
- Try Scream in network mode on the libvirt internal network (NAT) to isolate whether it's a driver issue or an IVSHMEM issue
- Try an older Windows build (24H2) to see if the driver works there
