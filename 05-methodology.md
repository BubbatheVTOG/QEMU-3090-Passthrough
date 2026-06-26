# Methodology

This document describes the process and strategies used throughout the project. These practices are recommended for any AI agent working on similar VFIO/libvirt configurations.

## Snapshot-Based Rollback

### Why

VM configuration changes are destructive and hard to undo. A bad XML edit, a driver install that bluescreens, or a Windows update that breaks something can leave the VM unbootable. Without snapshots, you're left manually reverting XML edits and hoping the guest state is recoverable.

### How

Use libvirt's internal snapshot mechanism:

```bash
# Create a snapshot before any change
virsh snapshot-create-as win11 "descriptive-name" "description of what state this captures"

# List snapshots
virsh snapshot-list win11

# Revert to a snapshot (discards current state)
virsh snapshot-revert win11 "snapshot-name"

# Delete a snapshot no longer needed
virsh snapshot-delete win11 "snapshot-name"
```

### Rules

1. **Snapshot before every destructive change.** A "destructive change" is anything that modifies the guest OS state (driver installs, registry edits, service configuration) or the VM XML in a way that could prevent boot.
2. **Use descriptive snapshot names.** Names like `working-2560x1440` or `vdd-installed` are self-documenting. Avoid numeric timestamps alone.
3. **Test after each change before making the next one.** If the change works, snapshot the working state. If it doesn't, revert immediately.
4. **Don't conflate changes.** Make one change, snapshot, test. If you make three changes at once and something breaks, you don't know which one caused it.
5. **The snapshot chain is a linked list.** Each snapshot has a parent. The backing file chain in the qcow2 images mirrors this. Don't manually delete qcow2 files in the chain — use `virsh snapshot-delete`.

### Snapshot Chain Used

```
pre-vdd-fix (baseline before VDD work)
  ├── working-2560x1440 (VDD working at 2560x1440)
  │   └── working-144hz (refresh rate set to 144Hz)
  └── vdd-installed (VDD driver installed and configured)
      └── working-final (final working state with all fixes)
```

## Guest Agent Remote Management

### Why

When the VM has no accessible display (no SPICE display, no VNC, Looking Glass not yet working), you need another way to run commands inside the guest. The QEMU guest agent provides this via the `virsh` command.

### How

1. Add the guest agent channel to VM XML:
   ```xml
   <channel type='unix'>
     <target type='virtio' name='org.qemu.guest_agent.0'/>
   </channel>
   ```
2. Install the guest agent in Windows (via virtio-win ISO, QEMU Guest Agent installer)
3. Set the service to Automatic
4. Execute commands remotely:
   ```bash
   virsh qemu-agent-command win11 '{"execute":"guest-exec","arguments":{"path":"cmd.exe","arg":["/c","whoami"],"capture-output":true}}'
   ```

### Critical Limitation: Session 0

The guest agent service runs in **session 0** (the non-interactive service session). This means:

- ✅ File operations, registry edits, service management, process launching — all work
- ❌ Display APIs (`ChangeDisplaySettingsEx`, `EnumDisplayDevices`, `ChangeDisplaySettings`) — **do NOT work**
- ❌ Anything that requires a user interactive session (GUI automation, display enumeration)

**Workaround:** Use auto-login + startup scripts for anything that needs an interactive session. Configure Windows to auto-login, then place a script in the `Startup` folder or `HKLM\...\Run` that performs the interactive-session work.

### Auto-Login Configuration

Registry settings under `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`:

```
AutoAdminLogon = 1
DefaultUserName = <username>
DefaultPassword = <password>
```

This is insecure (plaintext password in registry) but necessary for headless VM operation. Only use on VMs that are already isolated from physical/network access.

## Incremental Change Strategy

### The Process

1. **Identify the goal** (e.g., "get audio working")
2. **Research the approach** (check what tools exist, what others have done)
3. **Snapshot the current state** before any changes
4. **Make ONE change** (install one driver, edit one XML element, change one setting)
5. **Test** (boot VM, verify the change had the intended effect)
6. **If working:** snapshot the working state, move to next change
7. **If not working:** diagnose, try to fix, or revert and try a different approach
8. **Document** what was tried and what happened (see [audio attempts](04-audio-attempts.md))

### Why One Change at a Time

The USB headset passthrough failure (Attempt 5 in the audio doc) is the cautionary tale. We added the USB device AND removed SPICE at the same time. The VM failed to boot, and we couldn't determine which change caused it. We had to revert both and lost the work.

If we had made only one change, we would know exactly what broke.

## Documentation Practice

Document failures as carefully as successes. The audio attempts document captures every approach tried, including:
- What the approach was
- What was specifically configured
- What went wrong (exact errors)
- What was tried to fix it
- Why it was abandoned
- Lessons learned

This prevents future agents from repeating the same dead ends.
