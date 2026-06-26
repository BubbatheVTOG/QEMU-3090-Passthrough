# Audio Attempts

This is a chronological log of every approach tried to get audio working in the VM. **None of them worked.** Each section documents the approach, what was tried, what went wrong, and why it was abandoned.

## The Goal

Stream audio from the Windows guest to the Linux host so that sound from the VM plays through the host's audio system. Ideally with low latency and without requiring a physical audio device passthrough.

---

## Attempt 1: SPICE Audio (via Looking Glass client)

### Approach

Looking Glass can use the SPICE audio channel to carry guest audio to the host. The SPICE server in QEMU provides audio playback/record channels, and the LG client receives and plays them.

### What Was Tried

- QEMU XML configured with SPICE graphics and audio backend
- Looking Glass client config set to `spice:audio=yes` initially
- Sound device (HDA) added to VM XML

### What Went Wrong

The Looking Glass client connects **both** RECORD and PLAYBACK SPICE channels even when `spice:audio=no` is set in the config. The SPICE server advertises these channels, the client connects them, and then the SPICE server tells the client to reconnect an already-connected channel:

```
Protocol error. The server asked us to reconnect an already connected channel (RECORD)
```

This crashes the LG client's SPICE thread. The PLAYBACK channel has the same issue.

### What Was Tried to Fix It

1. Set `spice:audio=no` in client config — did not prevent the channel connections
2. Set QEMU audio backend to `type='none'` (`<audio id='1' type='none'/>`) — did not remove the SPICE audio channel advertisements
3. Removed sound devices (`<sound>`) from VM XML — reduced but did not eliminate the channel conflict
4. Removed the SPICE `spicevmc` channel — but this is needed for LG input (mouse/keyboard)

### Why It Was Abandoned

The SPICE protocol always advertises audio channels when a SPICE server is running. Looking Glass client always tries to connect them. There is no clean way to have SPICE input (for mouse/keyboard) without SPICE audio channel conflicts. This appears to be a bug or design limitation in Looking Glass B7's SPICE integration.

### Lesson

**SPICE audio and Looking Glass are fundamentally incompatible in B7.** The client doesn't properly honor `spice:audio=no` for channel connection management. If you need SPICE for input (which LG does), you get audio channel conflicts for free.

---

## Attempt 2: Scream (IVSHMEM mode)

### Approach

Scream is a virtual audio driver for Windows that streams audio over either UDP (network) or IVSHMEM (shared memory). IVSHMEM mode is ideal because it requires no network and has very low latency. The host runs a Scream receiver that reads audio from shared memory and plays it through the host's audio system.

### What Was Tried

- Installed Scream driver (v10.8.53.619) in Windows — test-signed driver, required enabling test signing in Windows
- Configured Scream registry for IVSHMEM mode: `HKLM\SYSTEM\CurrentControlSet\Services\Scream\Options` with `UseIVSHMEM=2`
- Added a 2MB IVSHMEM shared memory device to VM XML:
  ```xml
  <shmem name='scream'>
    <model type='ivshmem-plain'/>
    <size unit='M'>2</size>
  </shmem>
  ```
- Set Scream as the default playback device in Windows
- Confirmed shared memory file exists on host at `/dev/shm/scream`

### What Went Wrong

**The Scream driver never sent any audio data.** The IVSHMEM shared memory file (`/dev/shm/scream`) remained all zeros even when audio was playing in the guest. No packets were received by any potential receiver.

### What Was Tried to Fix It

1. Verified IVSHMEM device was present in Device Manager — it was
2. Verified registry settings were correct — `UseIVSHMEM=2` was set
3. Verified Scream was the default audio device — it was
4. Played audio in the guest (system sounds, YouTube, etc.) — no data in shared memory
5. Checked Windows driver signature requirements — Windows build 26200 may not trust the Scream driver's catalog signature even with test signing enabled

### Why It Was Abandoned

The root cause was never definitively identified. Suspicions:
- Scream uses the MSVAD (Microsoft Virtual Audio Device) sample framework, which may have compatibility issues with newer Windows builds (26200+)
- The driver's catalog signature may not be trusted even with test signing
- The IVSHMEM device may not be presented to the driver in a way it expects (different PCI address, different BAR layout)

### Lesson

**Scream IVSHMEM mode may not work on recent Windows 11 builds.** The driver is based on old MSVAD samples and hasn't been updated for Windows 11 24H2+/25H2. If the shared memory stays all zeros, the driver isn't initializing its IVSHMEM client. Consider checking if the driver even loads (check Windows driver management, not just test signing).

---

## Attempt 3: Scream (UDP network mode)

### Approach

Scream can also stream audio over UDP to a receiver on the network. This avoids the IVSHMEM driver issue entirely.

### What Went Wrong

**User constraint: no host firewall modifications allowed.** Scream UDP mode requires opening a port on the host firewall to receive the audio stream. This was ruled out by the user's explicit constraint.

### Why It Was Abandoned

Hard constraint. Could not proceed.

### Lesson

Always check user constraints early. If firewall modifications are off the table, network-based audio streaming is not an option.

---

## Attempt 4: QEMU Audio Backends (PulseAudio / ALSA / PipeWire)

### Approach

Instead of streaming audio from the guest, use QEMU's built-in audio backend support to route guest audio to the host's audio system directly. QEMU supports multiple backends: PulseAudio, ALSA, PipeWire.

### What Was Tried

Configured QEMU audio in the VM XML with various backends:

```xml
<!-- PulseAudio attempt -->
<audio id='1' type='pulseaudio' serverName='/run/user/1000/pulse/native'/>

<!-- ALSA attempt -->
<audio id='1' type='alsa' device='default'/>

<!-- PipeWire attempt -->
<audio id='1' type='pipewire'/>
```

### What Went Wrong

**QEMU runs as the `libvirt-qemu` user, not as the desktop user.** This means:

- **PulseAudio:** The libvirt-qemu user has no access to the desktop user's PulseAudio socket. Even with `serverName` pointing to the correct socket path, permissions denied.
- **ALSA:** The default ALSA device was busy (held by PipeWire/PulseAudio on the host). QEMU couldn't open it.
- **PipeWire:** Same issue as PulseAudio — libvirt-qemu user has no access to the PipeWire daemon.

### What Was Tried to Fix It

1. Added libvirt-qemu user to the desktop user's audio group — did not help (socket access is per-user, not per-group)
2. Tried setting `PULSE_COOKIE` environment variable for libvirt-qemu — too fragile, requires copying pulse cookie
3. Considered running QEMU as the desktop user — would require reconfiguring libvirt daemon, security implications

### Why It Was Abandoned

The fundamental issue is that QEMU/libvirt runs as a system service under a dedicated user, while the audio system runs under the desktop user's session. Bridging this gap securely is non-trivial and would require significant host configuration changes.

### Lesson

**QEMU audio backends don't work out-of-the-box with libvirt system sessions.** The user/permission model is wrong. This approach would work with a libvirt **session** (user-mode libvirt), but not with the system libvirt daemon that's needed for VFIO passthrough. Don't waste time on this unless you're willing to reconfigure the libvirt daemon to run as the desktop user.

---

## Attempt 5: USB Headset Passthrough

### Approach

Pass through a physical USB headset to the VM using qemu-xhci USB controller and `<hostdev>` USB passthrough. The headset would appear as a native USB audio device in Windows, bypassing all streaming/permission issues.

### What Was Tried

- Added USB headset to VM XML as a `<hostdev type='usb'>` device
- Removed SPICE display at the same time (to test if USB audio would work without SPICE interference)

### What Went Wrong

**The VM failed to boot.** This was likely because removing SPICE at the same time broke the input/display chain, not because of the USB device itself. The change conflated two modifications, making it impossible to tell which one caused the failure.

### Why It Was Abandoned

We rolled back via snapshot. The failure was likely due to the conflated SPICE removal, not the USB passthrough itself. This approach was NOT properly tested in isolation.

### Lesson

**Never conflate multiple changes.** When something breaks, you need to know which change caused it. Make one change at a time, snapshot, test, then make the next change. See [methodology](../operations/methodology.md).

**This is the most promising untested approach.** USB audio passthrough should work in principle — it's a well-supported QEMU feature. It just needs to be tested properly, with only the USB device added and nothing else changed.

---

## Summary Table

| Approach | Root Cause of Failure | Resolvable? |
|----------|----------------------|-------------|
| SPICE audio (LG client) | LG client always connects audio channels despite `spice:audio=no` | No — appears to be an LG bug |
| Scream IVSHMEM | Driver doesn't send data on Windows 26200 | Unknown — may need driver update |
| Scream UDP | User constraint: no firewall changes | No — hard constraint |
| QEMU PulseAudio | libvirt-qemu user can't access desktop user's audio socket | Possible but requires major reconfiguration |
| QEMU ALSA | Device busy (held by PipeWire) | Same as above |
| QEMU PipeWire | Same permission issue as PulseAudio | Same as above |
| USB headset passthrough | VM failed to boot (but conflated with other changes) | Likely yes — needs isolated retest |

## Recommended Next Approach

**Retry USB audio passthrough in isolation.** It's the only approach that wasn't definitively ruled out. Add ONLY the USB device to the XML, change nothing else, snapshot first, and test.
