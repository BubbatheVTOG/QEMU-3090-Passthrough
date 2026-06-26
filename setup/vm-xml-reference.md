# VM XML Configuration Reference

The complete libvirt VM XML for the working configuration. This is the final state after all fixes were applied. Annotations explain each section's purpose.

```xml
<domain type='kvm'>
  <name>win11</name>
  <memory unit='KiB'>25165824</memory>  <!-- 24GB pinned for VM -->
  <currentMemory unit='KiB'>25165824</currentMemory>
  <vcpu placement='static'>16</vcpu>  <!-- All 16 host threads -->

  <!-- CPU pinning: each vCPU pinned to corresponding host thread -->
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <!-- ... 2-14 ... -->
    <vcpupin vcpu='15' cpuset='15'/>
  </cputune>

  <os firmware='efi'>
    <type arch='x86_64' machine='pc-q35-11.0'>hvm</type>
    <!-- Secure Boot with OVMF (required for Win11 + NVIDIA driver) -->
    <firmware>
      <feature enabled='no' name='enrolled-keys'/>
      <feature enabled='yes' name='secure-boot'/>
    </firmware>
    <loader readonly='yes' secure='yes' type='pflash' format='raw'>
      /usr/share/edk2/x64/OVMF_CODE.secboot.4m.fd
    </loader>
    <nvram template='/usr/share/edk2/x64/OVMF_VARS.4m.fd'
          templateFormat='raw' format='raw'>
      /var/lib/libvirt/qemu/nvram/win11_VARS.fd
    </nvram>
    <boot dev='hd'/>
  </os>

  <!-- Hyper-V enlightenments (standard for Windows guests) -->
  <features>
    <acpi/>
    <apic/>
    <hyperv mode='custom'>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
      <vpindex state='on'/>
      <runtime state='on'/>
      <synic state='on'/>
      <stimer state='on'/>
      <frequencies state='on'/>
      <tlbflush state='on'/>
      <ipi state='on'/>
      <avic state='on'/>
    </hyperv>
    <vmport state='off'/>
    <smm state='on'/>
  </features>

  <!-- host-passthrough exposes real CPU to guest (prevents NVIDIA code 43) -->
  <cpu mode='host-passthrough' check='none' migratable='on'>
    <topology sockets='1' dies='1' clusters='1' cores='8' threads='2'/>
    <feature policy='require' name='topoext'/>
  </cpu>

  <clock offset='localtime'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
    <timer name='hypervclock' present='yes'/>
  </clock>

  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>

    <!-- Boot disk: virtio-blk, qcow2 with snapshot backing chain -->
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/win11.working-final'/>
      <target dev='vda' bus='virtio'/>
    </disk>

    <!-- virtio-win ISO (can be removed after driver installation) -->
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/virtio-win.iso'/>
      <target dev='sdb' bus='sata'/>
      <readonly/>
    </disk>

    <!-- Network: virtio-net on default (NAT) network -->
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>

    <!-- Virtio-serial bus for guest agent + SPICE channels -->
    <controller type='virtio-serial' index='0'/>

    <!-- Guest agent channel (remote command execution) -->
    <channel type='unix'>
      <target type='virtio' name='org.qemu.guest_agent.0'/>
    </channel>

    <!-- SPICE channel (used by Looking Glass for mouse/keyboard input) -->
    <channel type='spicevmc'>
      <target type='virtio' name='com.redhat.spice.0'/>
    </channel>

    <!-- SPICE tablet input (absolute mouse positioning) -->
    <input type='tablet' bus='usb'/>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>

    <!-- TPM (required for Windows 11) -->
    <tpm model='tpm-crb'>
      <backend type='emulator' version='2.0'/>
    </tpm>

    <!-- SPICE server: needed for LG input, NOT for display -->
    <!-- audio=none prevents QEMU from trying to handle audio -->
    <graphics type='spice' port='5900' autoport='no' listen='127.0.0.1'>
      <listen type='address' address='127.0.0.1'/>
      <image compression='off'/>
    </graphics>
    <audio id='1' type='none'/>

    <!-- Video: set to 'none' to fix mouse misalignment -->
    <!-- Do NOT set this to none until VDD is confirmed working -->
    <video>
      <model type='none'/>
    </video>

    <!-- GPU passthrough: VGA controller (RTX 3090) -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x0b' slot='0x00' function='0x0'/>
      </source>
    </hostdev>

    <!-- GPU passthrough: HDMI audio controller -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <driver name='vfio'/>
      <source>
        <address domain='0x0000' bus='0x0b' slot='0x00' function='0x1'/>
      </source>
    </hostdev>

    <!-- Looking Glass IVSHMEM (64MB for 2560x1440 framebuffer) -->
    <shmem name='looking-glass'>
      <model type='ivshmem-plain'/>
      <size unit='M'>64</size>
    </shmem>

    <!-- Scream IVSHMEM (2MB) - NOT in current XML, driver not working -->
    <!-- Uncomment if Scream is fixed -->
    <!--
    <shmem name='scream'>
      <model type='ivshmem-plain'/>
      <size unit='M'>2</size>
    </shmem>
    -->

    <watchdog model='itco' action='reset'/>
    <memballoon model='virtio'/>
  </devices>
</domain>
```

## Key Design Decisions in the XML

| Element | Value | Why |
|---------|-------|-----|
| `<memory>` | 24GB | Enough for Windows + GPU workload; host retains 22GB |
| `<vcpu>` | 16 | All host threads (8 cores × 2 SMT) |
| `machine` | `pc-q35-11.0` | Q35 required for PCIe passthrough |
| `firmware` | UEFI + Secure Boot | Win11 requirement + NVIDIA driver compatibility |
| `<cpu mode>` | `host-passthrough` | Prevents NVIDIA error code 43 |
| `<video model>` | `none` | Fixes mouse misalignment (see [mouse fix](../looking-glass/mouse-pointer-fix.md)) |
| `<graphics type>` | `spice` | Needed for LG input channel, not for display |
| `<audio type>` | `none` | Prevents QEMU audio backend conflicts |
| `shmem looking-glass` | 64MB | Framebuffer size for 2560x1440 |
| `hostdev` | GPU + audio | Both functions of the GPU must be passed through |

## Snapshot Backing Chain

The boot disk uses a qcow2 backing chain that mirrors the snapshot history:

```
win11.qcow2                          (base — fresh Windows install)
  └─ win11.1781735125                (virtio drivers + guest agent)
      └─ win11.vdd-installed          (VDD driver installed)
          └─ win11.working-final      (all config done, working state)
```

Each layer is a libvirt snapshot. The VM boots from `win11.working-final`, which transparently reads from all backing layers. Do NOT delete intermediate qcow2 files — use `virsh snapshot-delete` which manages the chain properly.
