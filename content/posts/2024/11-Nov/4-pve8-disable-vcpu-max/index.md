---
title: "Snippet: Disable max vCPUs = # of threads restriction in Proxmox VE 8.3"
date: 2024-11-27T12:34:56-00:00
draft: false
---

# Source:
[jknight comment on Proxmox forums](https://forum.proxmox.com/threads/override-max-vcpu-allowed-per-vm.39363/post-194758)

# Problem:

Needed to assign more than 64 threads to a VM for testing (just needed more than 64 threads, don't care about performance or lack of it.)

This was helpful and took a little bit of digging to find.

# Solution:

Open `/usr/share/perl5/PVE/QemuServer.pm` with your favorite text editor.

Remove lines 3772 - 3776 (if you're /ing around afterwards or have an otherwise modified config file, this will be right after line 3770: `my $vcpus = $conf->{vcpus} ? $conf->{vcpus} : maxcpus;`)

Diff:
```txt
-    my $allowed_vcpus = $cpuinfo->{cpus};
-
-    die "MAX $allowed_vcpus vcpus allowed per VM on this node\n"
-       if ($allowed_vcpus < $maxcpus);
-
```

Proxmox staff members' responses to forum threads asking about this, including the linked source, were disappointing.

That and the missing features (the ability to rename a VM or use DHCP) are seriously making me want to dump this platform entirely and return to using lots of ESXi or go all-in with OpenShift.