---
title: "Migrating guests between incompatible versions of QEMU"
date: 2025-11-29T01:30:00-00:00
draft: false
---

You cannot live-migrate Libvirt VMs between different QEMU versions over SSH (sadly, since that feature is awesome). You can, however, easily perform an offline migration.

I was migrating from Alma 10 to Fedora 43; your QEMU machine types may vary.

First, stop the VM on the source server.

Dump the configuration for the domain:

```txt
root@machine0:~# virsh dumpxml file-pub-0.lab.wporter.org > file-pub-0-cfg.xml
```

Copy the VHD over (typically to /var/lib/libvirt/images), and copy the config over somewhere easily accessible (it's just going to be used to define the VM in the new spot). `scp` is nice and easy to use for this (and means your traffic is going over the wire encrypted).

```txt
root@machine0:~# scp /var/lib/libvirt/images/file-pub-0.lab.wporter.org.qcow2 machine1:/var/lib/libvirt/images/
file-pub-0.lab.wporter.org.qcow2                                                 100%   64GB  77.8MB/s   14:02
```

Once you've got your files on the destination machine, edit the configured machinetype in the configuration XML file from the unsupported version of QEMU, e.g., `pc-q35-rhel10.0.0`, to a supported version, e.g., `pc-q35-9.2`. Example: `sed -i 's/pc-q35-rhel10.0.0/pc-q35-9.2/g'`.

Once references to unsupported things or nonexistent network interfaces are removed, load the domain on your target system, start it, and off you go:

```txt
root@machine1:~# virsh define file-pub-0-cfg.xml
```
