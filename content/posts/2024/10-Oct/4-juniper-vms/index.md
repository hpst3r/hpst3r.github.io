---
title: "Snippet: setting up nested virt for Juniper (GNS3)"
date: 2024-10-20T12:34:56-00:00
draft: false
---

[vJunos-switch deployment guide](https://www.juniper.net/documentation/us/en/software/vJunos/vjunos-switch-deployment-guide-for-kvm/vjunos-switch-deployment-guide-for-kvm.pdf) - 5g, 4 cores

Juniper VMs nest the control plane inside a Linux VM emulating forwarding plane - for this reason, running virtualized Juniper devices in a VM can be problematic

I'd recommend not trying to do the above and using something dedicated

This is probably from an Ubuntu Server install:

See if nested virtualization is enabled

`grep -cw vmx /proc/cpuinfo`

Enable nested virtualization

`echo "options kvm-intel nested=1" >> /etc/modprobe.d/kvm.conf`
`modprobe -r kvm && modprobe kvm`
`systemctl restart libvirtd`

Interfaces on the switch should be named:
fxp0
ge000 - ge0xx

`-cpu host -smbios type=1,product=VM-VEX` kvm options are REQUIRED:
- without `-cpu host` or `-cpu $something-with-vtx` KVM won't work
- without smbios options the network interfaces won't be assigned ge-0/0/x IDs

Once the vJunos switch is up:

Log in to amnesiac (root/)
```txt
root@:~ # cli
root> edit
Entering configuration mode

[edit]
root# set system root-authentication plain-text-password
New password:
Retype new password:

[edit]
root# delete chassis auto-image-upgrade

[edit]
root# commit
commit complete
```