---
title: "Migrating Libvirt VMs to Proxmox VE by importing qcow2 disks to Ceph"
date: 2026-02-22T23:00:00-00:00
draft: false
---

I'm migrating from a Libvirt hypervisor (AlmaLinux 10) to Proxmox VE.

In my case, my images are backed by a cloud-init template:

```txt
[root@3060t0 ~]# qemu-img info --backing-chain /var/lib/libvirt/images/UniFi_OS.qcow2 
image: /var/lib/libvirt/images/UniFi_OS.qcow2
file format: qcow2
virtual size: 32 GiB (34359738368 bytes)
disk size: 5.09 GiB
cluster_size: 65536
backing file: /srv/iso/AlmaLinux-10-GenericCloud-10.0-20250528.0.x86_64.qcow2
backing file format: qcow2
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: true
    refcount bits: 16
    corrupt: false
    extended l2: false
Child node '/file':
    filename: /var/lib/libvirt/images/UniFi_OS.qcow2
    protocol type: file
    file length: 6.51 GiB (6987579392 bytes)
    disk size: 5.09 GiB

image: /srv/iso/AlmaLinux-10-GenericCloud-10.0-20250528.0.x86_64.qcow2
file format: qcow2
virtual size: 10 GiB (10737418240 bytes)
disk size: 439 MiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false
Child node '/file':
    filename: /srv/iso/AlmaLinux-10-GenericCloud-10.0-20250528.0.x86_64.qcow2
    protocol type: file
    file length: 439 MiB (460062720 bytes)
    disk size: 439 MiB
```

So, before I move the VMs off this host (with the backing file on it) I had to rebase them (consolidate the backing file and changes to a new QCOW). This can be done by "converting" the current VM disk with `qemu-img`.

```sh
qemu-img convert -O qcow2 -f qcow2 /var/lib/libvirt/images/UniFi_OS.qcow2 /home/wporter/UniFi_OS.qcow2
```

Then, I've copied the QCOW file over to local storage on one of the Proxmox VE hosts with `scp`.

In Proxmox VE, I'll create a VM with my desired characteristics (UEFI, CPU config, add to HA, etc). Once that's done, I'll run a `qm importdisk` (specifying the VM ID and storage target). This will convert the QCOW to RAW format and copy it to Ceph.

```sh
qm importdisk 100 ./UniFi_OS.qcow2 ceph
```

This will give me an 'unused disk' on the VM, so I'll attach it to the SCSI controller. My guest was already running on a modern Linux system with paravirtualized devices, so I do not need to update the initramfs to include the VirtIO modules.

> Be sure to use the VirtIO SCSI single controller! It's the most performant and best-supported option. The emulated options (VMware PVSCSI, LSI, etc), SATA, and IDE controllers perform significantly worse. The VirtIO Block option lacks some of the newer features added to the VirtIO SCSI single controller and may be deprecated soon.

{{< figure src="images/0-vm-hw.png" >}}

When attaching the disk, be sure to toggle writeback caching (if safe in your environment), the I/O thread, and both discard and SSD emulation (assuming the backing volume is flash and not spinning rust). Each will significantly improve performance in most cases, with exceptions:

- Writeback caching is not safe for older OSes that do not support sending flushes to push sync writes to disk
- Writeback caching MUST be used with Ceph if the RBD cache is enabled on the host, otherwise flushes will not propagate to Ceph and *everything* will be cached
- The I/O thread allows QEMU to parallelize things more by splitting I/O operations out into another process. This is good unless you are running into thread contention on the host.
- SSD emulation tells the guest that it's not on rotational media, which allows the guest's I/O scheduler to optimize for flash. This will dramatically improve random I/O and should always be enabled if you're using SSDs for VM disks (you are, right?)
- Discard tells the guest that the discard command is supported, which allows your SSDs to free up space that was used then freed in the guest. On cheaper drives this can significantly extend lifespan and improve performance.

{{< figure src="images/1-vm-disk.png" >}}

I'll also have to edit the boot order (VM > Options > Boot Order > Edit) and tell the VM to boot off the newly imported disk:

{{< figure src="images/2-vm-boot-order.png" >}}

Check the newly attached SCSI disk to select it as an eligible boot method, then click OK and fire up the VM.

{{< figure src="images/3-vm-boot-order.png" >}}
