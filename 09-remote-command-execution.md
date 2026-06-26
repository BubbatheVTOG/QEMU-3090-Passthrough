# Remote Command Execution via Guest Agent

The QEMU guest agent allows running commands inside the Windows VM from the Linux host, without any network connection, display, or SSH access. This was essential for headless configuration of the VM.

## How It Works

1. The guest agent service (`qemu-ga`) runs inside Windows in **session 0** (the non-interactive service session)
2. It communicates with QEMU over a virtio-serial channel
3. The host sends JSON commands via `virsh qemu-agent-command`
4. The agent executes them and returns results

This is NOT SSH — it's a QEMU protocol that works even when the VM has no network, no display, and no SSH server.

## Prerequisites

### VM XML (host side)

```xml
<controller type='virtio-serial' index='0'/>

<channel type='unix'>
  <target type='virtio' name='org.qemu.guest_agent.0'/>
</channel>
```

### Guest side

Install `qemu-ga-x86_64.msi` from the virtio-win ISO. Set to Automatic start.

## Command Patterns

### Run a command and capture output

```bash
virsh qemu-agent-command win11 '{
  "execute": "guest-exec",
  "arguments": {
    "path": "cmd.exe",
    "arg": ["/c", "whoami"],
    "capture-output": true
  }
}'
```

Returns:
```json
{"return":{"pid":42}}
```

### Get the output of a running/completed command

```bash
virsh qemu-agent-command win11 '{
  "execute": "guest-exec-status",
  "arguments": {"pid": 42}
}'
```

Returns:
```json
{
  "return": {
    "exited": true,
    "exitcode": 0,
    "out-data": "dXNlclxu"  // base64-encoded stdout
  }
}
```

Decode the output:
```bash
echo "dXNlclxu" | base64 -d
# → user
```

### Real-world examples used in this project

#### Check if a driver is installed
```bash
virsh qemu-agent-command win11 '{
  "execute": "guest-exec",
  "arguments": {
    "path": "cmd.exe",
    "arg": ["/c", "pnputil /enum-drivers | findstr MTT"],
    "capture-output": true
  }
}'
```

#### Write a file to the guest (base64-encode content)
```bash
# Encode the file content
CONTENT=$(base64 -w0 /path/to/local/file.ps1)

virsh qemu-agent-command win11 "{
  \"execute\": \"guest-file-open\",
  \"arguments\": {
    \"path\": \"C:\\\\Scripts\\\\file.ps1\",
    \"mode\": \"w+\"
  }
}"  # Returns a handle, e.g. {"return":100}

virsh qemu-agent-command win11 "{
  \"execute\": \"guest-file-write\",
  \"arguments\": {
    \"handle\": 100,
    \"buf-b64\": \"$CONTENT\"
  }
}"

virsh qemu-agent-command win11 '{
  "execute": "guest-file-close",
  "arguments": {"handle": 100}
}'
```

#### Run a PowerShell script
```bash
virsh qemu-agent-command win11 '{
  "execute": "guest-exec",
  "arguments": {
    "path": "powershell.exe",
    "arg": ["-ExecutionPolicy", "Bypass", "-File", "C:\\Scripts\\Set-VddResolution.ps1"],
    "capture-output": true
  }
}'
```

#### Edit the registry
```bash
virsh qemu-agent-command win11 '{
  "execute": "guest-exec",
  "arguments": {
    "path": "reg.exe",
    "arg": ["add", "HKLM\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon",
           "/v", "AutoAdminLogon", "/t", "REG_SZ", "/d", "1", "/f"],
    "capture-output": true
  }
}'
```

#### Check display enumeration (WILL FAIL from session 0)
```bash
virsh qemu-agent-command win11 '{
  "execute": "guest-exec",
  "arguments": {
    "path": "powershell.exe",
    "arg": ["-Command",
           "Add-Type -TypeDefinition (Get-Content C:\\Scripts\\Display.cs -Raw);
            [Display]::EnumDisplayDevices()"],
    "capture-output": true
  }
}'
```

This returns empty results because the guest agent runs in session 0.

## What Works and What Doesn't

### Works from guest agent (session 0)

- ✅ File read/write (`guest-file-open`, `guest-file-write`, `guest-file-close`)
- ✅ Process execution (`guest-exec`)
- ✅ Registry edits (`reg.exe add/delete/query`)
- ✅ Service management (`sc.exe config/start/stop`)
- ✅ Driver installation (`pnputil`, `devcon`)
- ✅ Device enumeration via SetupAPI (but see below)
- ✅ PowerShell scripts (file operations, registry, services)

### Does NOT work from guest agent (session 0)

- ❌ `ChangeDisplaySettingsEx` — silently fails, no resolution change
- ❌ `EnumDisplayDevices` — returns no displays
- ❌ `ChangeDisplaySettings` — same as above
- ❌ Any API that requires a window station / interactive desktop
- ❌ GUI automation (SendInput, UIAutomation)
- ❌ Anything that needs to interact with the user's desktop session

### The Workaround

For display-related operations, use **auto-login + startup scripts**:

1. Configure auto-login (via guest agent — registry edits work from session 0)
2. Place a script in the Startup folder or `HKLM\...\Run` (via guest agent — file writes work)
3. Reboot the VM
4. Windows auto-logins → startup script runs in interactive session → display APIs work

This is the only reliable way to configure displays without RDP, VNC, or physical console access.

## Helper Script

A bash wrapper to simplify guest-agent command execution:

```bash
#!/bin/bash
# guest-run.sh — run a command in the VM via guest agent
# Usage: ./guest-run.sh 'cmd /c whoami'
# Usage: ./guest-run.sh 'powershell -Command "Get-Service qemu-ga"'

VM_NAME="win11"
CMD="$1"

# Execute the command
RESULT=$(virsh qemu-agent-command "$VM_NAME" "{
  \"execute\": \"guest-exec\",
  \"arguments\": {
    \"path\": \"cmd.exe\",
    \"arg\": [\"/c\", \"$CMD\"],
    \"capture-output\": true
  }
}")

PID=$(echo "$RESULT" | grep -o '"pid":[0-9]*' | grep -o '[0-9]*')

# Wait for completion and get output
sleep 1
STATUS=$(virsh qemu-agent-command "$VM_NAME" "{
  \"execute\": \"guest-exec-status\",
  \"arguments\": {\"pid\": $PID}
}")

# Decode and print output
echo "$STATUS" | grep -o '"out-data":"[^"]*"' | cut -d'"' -f4 | base64 -d
echo "$STATUS" | grep -o '"err-data":"[^"]*"' | cut -d'"' -f4 | base64 -d >&2
```

## Lessons

- **The guest agent is the primary tool for headless VM management.** Learn to use it well.
- **Session 0 is the fundamental limitation.** Anything interactive (displays, GUI) must be deferred to startup scripts.
- **File transfer via guest-file-write is reliable** but verbose. For complex scripts, base64-encode locally and write in chunks.
- **Always check `exitcode`** in the guest-exec-status response. A command can "succeed" (return output) but have a non-zero exit code.
- **The guest agent is stateless between reboots.** No persistent sessions, no environment variables carried over. Each `guest-exec` is a fresh process.
