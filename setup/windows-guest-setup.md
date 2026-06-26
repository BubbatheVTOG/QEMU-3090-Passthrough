# Windows Guest Setup

Everything done inside the Windows VM to make Looking Glass, VDD, audio, and remote management work. This is a step-by-step reference for AI agents reproducing this setup.

## Prerequisites

- Windows 11 (build 26200+) installed on the VM
- virtio-win drivers installed (disk, network, balloon, guest-agent, serial)
- NVIDIA driver installed for the passed-through GPU
- Test signing enabled (for Scream driver; may be needed for VDD on some builds)

---

## 0. Windows Installation: Bypass Microsoft Account Requirement

Windows 11 OOBE normally requires a Microsoft account to complete setup. For a VM, you want a local account. There is a built-in bypass.

### Method: `bypassnro` (OOBE Bypass)

1. Boot the Windows 11 installer and proceed through OOBE until you reach the network connection screen
2. Press `Shift+F10` to open a command prompt
3. Run:
   ```cmd
   oobe\bypassnro
   ```
4. The system reboots. On the next OOBE pass, you get the "I don't have internet" / "Continue with limited setup" option
5. Create a local account

### How It Works

`bypassnro` adds a registry key that tells OOBE to skip the network requirement:
```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\OOBE
  BypassNRO = 1 (DWORD)
```

This is the simplest method. There are others (autounattend.xml with `OfflineAccount` / `bypassNRO` in the answer file, or the `Shift+F10` + `ipconfig /release` trick to force offline), but the `oobe\bypassnro` command is the most reliable.

### Why This Matters for This Setup

The entire VDD + Looking Glass + auto-login chain depends on a **local account** with a known password. A Microsoft account would complicate auto-login (PIN-based login, 2FA, etc.). Getting a local account during install is the foundation.

---

## 1. QEMU Guest Agent

The guest agent enables remote command execution from the host via `virsh qemu-agent-command`. This is essential for headless management.

### VM XML (host side)

```xml
<controller type='virtio-serial' index='0'>
  <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
</controller>

<channel type='unix'>
  <target type='virtio' name='org.qemu.guest_agent.0'/>
  <address type='virtio-serial' controller='0' bus='0' port='2'/>
</channel>
```

### Guest Installation

Install from the virtio-win ISO: `virtio-win\guest-agent\qemu-ga-x86_64.msi`

Set the service to Automatic:
```cmd
sc config qemu-ga start= auto
sc start qemu-ga
```

### Usage from Host

```bash
# Run a command in the guest via agent
virsh qemu-agent-command win11 '{
  "execute": "guest-exec",
  "arguments": {
    "path": "cmd.exe",
    "arg": ["/c", "whoami"],
    "capture-output": true
  }
}'

# Get output (use the PID returned by guest-exec)
virsh qemu-agent-command win11 '{
  "execute": "guest-exec-status",
  "arguments": { "pid": 123 }
}'
```

### Critical Limitation

The guest agent runs in **session 0** (non-interactive service session). Display APIs (`ChangeDisplaySettingsEx`, `EnumDisplayDevices`) do NOT work from session 0. See [methodology](../operations/methodology.md) for the auto-login workaround.

---

## 2. Looking Glass Host Service

### Installation

Download Looking Glass B7 host installer for Windows. Install as a service:

```cmd
# The installer registers the service automatically
# Verify it's set to automatic
sc config "Looking Glass (host)" start= auto
sc start "Looking Glass (host)"
```

### IVSHMEM Device (host side VM XML)

```xml
<shmem name='looking-glass'>
  <model type='ivshmem-plain'/>
  <size unit='M'>64</size>
</shmem>
```

64MB is sufficient for 2560x1440 (double-buffered: ~17.6MB actual usage).

The shared memory appears on the Linux host at `/dev/shm/looking-glass`.

### Host Log

Located at: `C:\ProgramData\Looking Glass (host)\looking-glass-host.txt`

Check this log if capture isn't working. The key error to watch for:
```
Failed to locate a valid output device
```
This means the NVIDIA GPU has no active display — VDD needs to be configured.

---

## 3. VDD (Virtual Display Driver)

VDD creates a virtual display on the NVIDIA GPU so Looking Glass has something to capture. This replaces a physical dummy plug.

### Installation

Download VDD (Virtual Display Driver by MTT). Install the driver. After installation, the device does NOT appear automatically — it must be created programmatically.

### Device Creation

The VDD device uses hardware ID `Root\MttVDD`. Create it using the SetupAPI:

```powershell
# PowerShell - create VDD device
$deviceClass = [guid]"{4d36e968-e325-11ce-bfc1-08002be10318}"  # Display adapters
$deviceInfo = [Win32.SetupDi]::SetupDiCreateDeviceInfo(
    $deviceClass,
    "MTT Virtual Display Device",
    "Root\MttVDD",
    ...
)
```

Or via `pnputil` / Device Manager "Add legacy hardware" with the VDD INF.

After creation, a new display appears (e.g., `\\.\DISPLAY5`).

### VDD Settings File

Settings file: `C:\Windows\System32\vdd_settings.xml`

Example configuration for NVIDIA GPU, 2560x1440@144Hz:

```xml
<?xml version="1.0" encoding="utf-8"?>
<root>
  <device>
    <hardware_id>Root\MttVDD</hardware_id>
    <adapter>NVIDIA GeForce RTX 3090</adapter>
    <resolutions>
      <resolution width="2560" height="1440" refresh="144" />
    </resolutions>
  </device>
</root>
```

**Note:** The settings file alone does not reliably set the resolution on boot. A startup script (below) is needed to force the resolution via `ChangeDisplaySettingsEx`.

### Resolution Setting Script

Placed in the Windows Startup folder AND in `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`.

The script uses the Win32 `ChangeDisplaySettingsEx` API to set the VDD display to 2560x1440@144Hz. This must run in an interactive user session (not session 0).

PowerShell example:

```powershell
# Set-VddResolution.ps1
# Enumerates displays, finds the VDD display, sets 2560x1440@144Hz

$code = @"
using System;
using System.Runtime.InteropServices;

public class Display
{
    [StructLayout(LayoutKind.Sequential)]
    public struct DEVMODE
    {
        [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 32)]
        public string dmDeviceName;
        public ushort dmSpecVersion;
        public ushort dmDriverVersion;
        public ushort dmSize;
        public ushort dmDriverExtra;
        public uint dmFields;
        public int dmPositionX;
        public int dmPositionY;
        public uint dmDisplayOrientation;
        public uint dmDisplayFixedOutput;
        public short dmBitsPerPel;
        public uint dmPelsWidth;
        public uint dmPelsHeight;
        public uint dmDisplayFlags;
        public uint dmDisplayFrequency;
        public uint dmICMMethod;
        public uint dmICMIntent;
        public uint dmMediaType;
        public uint dmDitherType;
        public uint dmReserved1;
        public uint dmReserved2;
        public uint dmPanningWidth;
        public uint dmPanningHeight;
    }

    [DllImport("user32.dll", CharSet = CharSet.Ansi)]
    public static extern int EnumDisplaySettings(string deviceName, int modeNum, ref DEVMODE devMode);

    [DllImport("user32.dll", CharSet = CharSet.Ansi)]
    public static extern int ChangeDisplaySettingsEx(string deviceName, ref DEVMODE devMode, IntPtr hwnd, uint flags, IntPtr lParam);
}
"@

Add-Type -TypeDefinition $code

$deviceName = "\\.\DISPLAY5"  # VDD display - enumerate to find correct one
$dm = New-Object Display+DEVMODE
$dm.dmSize = [System.Runtime.InteropServices.Marshal]::SizeOf($dm)

# Find the mode matching 2560x1440@144
$mode = 0
while ([Display]::EnumDisplaySettings($deviceName, $mode, [ref]$dm) -ne 0) {
    if ($dm.dmPelsWidth -eq 2560 -and $dm.dmPelsHeight -eq 1440 -and $dm.dmDisplayFrequency -eq 144) {
        $dm.dmFields = 0x00080000 -bor 0x00100000 -bor 0x00400000  # PELSWIDTH|PELSHEIGHT|FREQUENCY
        [Display]::ChangeDisplaySettingsEx($deviceName, [ref]$dm, [IntPtr]::Zero, 0, [IntPtr]::Zero)
        break
    }
    $mode++
}
```

**Important:** The display number (`\\.\DISPLAY5`) is not fixed. Enumerate all displays using `EnumDisplayDevices` to find which one is the VDD display. The VDD display typically has a device name containing "MTT" or appears as a secondary display.

---

## 4. Auto-Login Configuration

Required so the resolution script runs in an interactive session at boot.

Registry settings under `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`:

```
AutoAdminLogon = 1
DefaultUserName = <username>
DefaultPassword = <password>
DefaultDomainName = <domain or computer name>
```

Can be set via guest agent (session 0 is fine for registry edits):

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon /t REG_SZ /d 1 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName /t REG_SZ /d "username" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword /t REG_SZ /d "password" /f
```

### Startup Script Placement

Place the resolution script in:
- `C:\Users\<username>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\` (per-user)
- AND/OR register in `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` (system-wide)

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v SetVddResolution /t REG_SZ /d "powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\Set-VddResolution.ps1" /f
```

---

## 5. Scream Audio Driver (Installed but Not Working)

Documented here for reference. See [audio attempts](../audio/audio-attempts.md) for why this didn't work.

### Installation

1. Enable test signing: `bcdedit /set testsigning on` (reboot required)
2. Install Scream driver (MSVAD-based virtual audio device)
3. Set registry for IVSHMEM mode:
   ```
   HKLM\SYSTEM\CurrentControlSet\Services\Scream\Options
     UseIVSHMEM = 2 (DWORD)
   ```
4. Set Scream as default playback device in Windows Sound settings

### IVSHMEM Device (host side VM XML)

```xml
<shmem name='scream'>
  <model type='ivshmem-plain'/>
  <size unit='M'>2</size>
</shmem>
```

Shared memory appears on host at `/dev/shm/scream`.

### Status

Driver installed but **never sends audio data**. Shared memory stays all zeros. Likely incompatible with Windows 11 build 26200.

---

## 6. SPICE Configuration (for Looking Glass Input)

Looking Glass uses the SPICE channel for mouse/keyboard input. SPICE must be configured in the VM XML even though the SPICE display is not used.

### VM XML (host side)

```xml
<graphics type='spice' port='5900' autoport='no' listen='127.0.0.1'>
  <listen type='address' address='127.0.0.1'/>
  <image compression='off'/>
</graphics>

<channel type='spicevmc'>
  <target type='virtio' name='com.redhat.spice.0'/>
</channel>
```

### Audio Conflict

The SPICE server always advertises audio channels (PLAYBACK, RECORD). The Looking Glass client connects these channels even with `spice:audio=no`, causing crashes. This is an unresolved issue — see [remaining issues](../remaining-issues.md).

### Looking Glass Client Config

`~/.looking-glass-client.ini`:

```ini
[spice]
audio=no
```

---

## 7. Video Device Configuration

The virtual video device must be set to `none` once VDD is confirmed working, to fix mouse misalignment:

```xml
<video>
  <model type='none'/>
</video>
```

See [mouse fix](../looking-glass/mouse-pointer-fix.md) for the reasoning.

**Order of operations:** Get VDD working FIRST, then set video to `none`. Setting video to none before VDD works leaves you with no display at all.
