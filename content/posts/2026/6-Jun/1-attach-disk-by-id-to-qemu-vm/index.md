---
title: "Attach a physical disk to a QEMU VM"
date: 2026-06-13T20:00:00-00:00
draft: false
---

My desktop's Windows install broke today. As I already had a Fedora install on another disk, it was simplest to boot the Windows install as a VM and fix it that way.

Write a Libvirt-format XML device definition, e.g., in the file "nvme1n1.xml".

Note the `bus='sata'`. I'm booting a Windows VM from a second disk in the system that doesn't have VirtIO drivers.

```sh
tee nvme1n1.xml << EOT
<disk type='block' device='disk'>
    <driver name='qemu' type='raw'/>
    <source dev='/dev/disk/by-id/nvme-Samsung_SSD_980_PRO_2TB_S6B0NL0T921654N'/>
    <target dev='vda' bus='sata'/>
</disk>
EOT
```

Then, attach the disk to the VM:

```sh
sudo virsh attach-device win11 --file nvme1n1.xml --persistent
```

This will prepend the device configuration to the relevant section of the machine's configuration (e.g., the `<devices>` block):

```xml
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw'/>
      <source dev='/dev/disk/by-id/nvme-Samsung_SSD_980_PRO_2TB_S6B0NL0T921654N'/>
      <target dev='vda' bus='sata'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
```

To use the more performant (and efficient) paravirtualized driver:

1. install the VirtIO drivers and guest tools in your VM (while booted with the SATA driver)
2. add a second disk (not the boot disk) to the system as a VirtIO device (to convince Windows to load the driver at boot, e.g., add the driver to the initramfs for the Linux folks following along)
3. then shut the VM down and switch the boot device to the VirtIO bus (e.g., modify the `target bus` from `sata` to `virtio` in the device entry in the machine's XML configuration with `virsh edit`).
4. Finally, restart the VM. You should be using the VirtIO driver.
