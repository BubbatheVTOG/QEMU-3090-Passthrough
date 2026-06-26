# Mouse Pointer Fix

## The Problem

When both a virtual video device (virtio-vga) AND a passed-through GPU are present in the VM, Windows sees multiple displays. The SPICE tablet input (used by Looking Glass for mouse positioning) maps its coordinate space to the **total virtual desktop**, not to the display that Looking Glass is capturing.

This causes the mouse pointer to be **misaligned** — clicks land in the wrong place, the cursor appears offset, and in multi-monitor configurations the pointer may disappear entirely onto a display that isn't being captured.

## What Worked

### Removing virtio-vga

The fix was to remove the virtual video card entirely by setting the video model to `none`:

```xml
<video>
  <model type='none'/>
</video>
```

With no virtual video device, Windows only sees the displays attached to the passed-through NVIDIA GPU (i.e., the VDD virtual display). The SPICE tablet coordinate space now maps 1:1 to the captured display, and mouse alignment is perfect.

**Important:** We also kept the SPICE graphics server enabled (`<graphics type='spice'>`) because Looking Glass uses the SPICE channel for input (mouse/keyboard). The `<video>` element controls the virtual GPU; `<graphics>` controls the SPICE server. These are independent.

## What Didn't Work / Tradeoffs

### Loss of SPICE Display Fallback

With `<model type='none'/>`, there is no SPICE display fallback. If Looking Glass isn't running or the VDD isn't active, you have **no way to see the VM's screen**. This means:
- You cannot troubleshoot boot issues visually without Looking Glass
- If VDD fails to load, the VM boots to a black screen
- Windows Recovery Environment (WinRE) has no display

**Mitigation:** Keep snapshots of known-working states so you can roll back if the VM becomes inaccessible. See [methodology](05-methodology.md).

### virtio-vga as Secondary Display

We considered keeping virtio-vga as a secondary (non-primary) display and only using the NVIDIA/VDD display as primary. This did not reliably fix the mouse misalignment — the SPICE tablet still mapped to the combined desktop space. The only reliable fix was removing the virtual video device entirely.

## Lessons

- **Mouse misalignment in Looking Glass is almost always caused by multiple displays.** If the captured display is not the only display, the SPICE tablet coordinate mapping will be wrong.
- **`<model type='none'/>` is the correct fix**, but it removes your visual fallback. Only do this once VDD is confirmed working.
- **`<video>` and `<graphics>` are separate concepts.** You can have SPICE input (mouse/keyboard) without a SPICE display. Looking Glass needs the SPICE input channel but not the SPICE display.
- **Order of operations matters:** Get VDD working FIRST (so you have a display on the NVIDIA GPU), THEN remove virtio-vga. Doing it in the wrong order leaves you with no display at all.
