# Looking Glass and VDD (Virtual Display Driver)

## The Core Problem

Looking Glass captures the framebuffer of a GPU output. The NVIDIA GPU needs an **active display output** to render anything. Without a physical monitor attached, the GPU has no active output, and the Looking Glass host service logs:

```
Failed to locate a valid output device
```

Since no physical dummy plug was available, we used **VDD (Virtual Display Driver)** by MTT to create a virtual display on the NVIDIA GPU.

## What Worked

### Looking Glass Host (Windows side)

- Installed Looking Glass host service (B7) in Windows
- Set service to Automatic start (`sc config "Looking Glass (host)" start= auto`)
- IVSHMEM device configured in VM XML as a 64MB shared memory region:

```xml
<shmem name='looking-glass'>
  <model type='ivshmem-plain'/>
  <size unit='M'>64</size>
</shmem>
```

- Shared memory file appears on host at `/dev/shm/looking-glass`
- Host service connects to IVSHMEM and captures the framebuffer

### Looking Glass Client (Linux side)

- Installed `looking-glass` package (B7) on Arch Linux host
- Client config at `~/.looking-glass-client.ini`:
  ```ini
  [spice]
  audio=no
  ```
- The `spice:audio=no` setting was critical — see [audio attempts](../audio/audio-attempts.md) for why
- Client connects to `/dev/shm/looking-glass` and renders the framebuffer with very low latency

### VDD (Virtual Display Driver)

- Installed VDD v24.12.24 in Windows
- VDD creates a virtual display device that appears as `\\.\DISPLAY5` (or similar) in Windows display enumeration
- VDD settings configured via `C:\Windows\System32\vdd_settings.xml` to specify the NVIDIA GPU and target resolution (2560x1440) and refresh rate (144Hz)
- Once VDD is active and set as the display, Looking Glass host successfully captures it

### Resolution Switching

- Used `ChangeDisplaySettingsEx` Win32 API to programmatically set the VDD display to 2560x1440@144Hz
- This must be called from an **interactive user session**, NOT from a service or guest agent (see below)

## What Didn't Work / Gotchas

### The Chicken-and-Egg Problem (CRITICAL)

This was the single most time-consuming issue:

1. NVIDIA GPU needs an active display to render → VDD provides the virtual display
2. VDD only becomes active after Windows loads the driver → happens at login
3. Auto-login was configured, but the resolution script runs AFTER login
4. The guest agent (used for remote command execution) runs in **session 0**
5. `ChangeDisplaySettingsEx` and `EnumDisplayDevices` **do NOT work from session 0** — they silently fail or return no displays
6. Therefore: you cannot remotely configure the display via guest agent

**Workaround used:**
- Configured Windows auto-login (`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` with `AutoAdminLogon=1`, `DefaultUserName`, `DefaultPassword`)
- Placed a resolution-setting script in the Windows `Startup` folder AND in `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`
- When the VM boots, Windows auto-logins, the startup script runs in the interactive session, and VDD gets configured

**Remaining issue:** There is a window of time between boot and the startup script completing where the display is not yet configured. Looking Glass will show nothing during this period.

### VDD Device Creation

VDD doesn't just "appear" after installing the driver. The device must be explicitly created using `SetupDiCreateDeviceInfo` + `SetupDiCallClassInstaller` with the hardware ID `Root\MttVDD`. This was done programmatically.

### VDD Settings Persistence

VDD settings were written to two locations:
- `C:\Windows\System32\vdd_settings.xml` — global config
- Driver store registry — per-device settings

Even with settings persisted, VDD did not always honor the configured resolution on boot. The startup script that calls `ChangeDisplaySettingsEx` is what reliably sets the correct resolution.

## Lessons

- **Without a physical dummy plug, you need VDD** (or equivalent virtual display driver) for Looking Glass to work with a passed-through GPU.
- **Display configuration APIs require an interactive session.** This is a Windows limitation, not a VDD limitation. Any agent automating display setup must account for this — use auto-login + startup scripts, not guest agent commands.
- **The IVSHMEM size must be large enough** for the framebuffer. 64MB was sufficient for 2560x1440. Formula: `width * height * bytes_per_pixel * 2` (double-buffered). For 2560x1440@32bpp: ~17.6MB, so 64MB is comfortable.
- **Looking Glass B7** is the current stable series. The host and client versions must match.
