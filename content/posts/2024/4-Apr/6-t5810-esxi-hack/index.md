---
title: "ESXi 8 - ignoring a TSC mismatch PSOD"
date: 2024-04-09T12:34:56-00:00
draft: false
---

# Problem

Both of my Dell T5810s fail to boot ESXi 8.0U2 - they quickly meet a PSOD that calls out a TSC sync error.

The CPUs in these machines are an E5-2660 v4 and E5-2690 v4 (both the 14-core Broadwell die, incidentally.)

The TSC is off by same value every boot, and both machines experience no instability with Win/Linux (or ESXi after the counter error is bypassed.) YMMV with bypassing this yourself.

# Fix

Hit the Shift + O keys to edit your boot parameters while you're booting the installer, and (later on) your new installation for the first time.

Add the following arguments:

```txt
tscSyncSkip=TRUE timerForceTSC=TRUE
```

After first boot, SSH to the machine and set this parameter permanently with:

1) use esxcli:

```txt
esxcli system settings kernel set --setting=tscSyncSkip --value=TRUE
```
```txt
esxcli system settings kernel set --setting=timerForceTSC --value=TRUE
```

2) edit /bootbank/boot.cfg (kernelopt line)

```txt
kernelopt=... tscSyncSkip=TRUE timerForceTSC=TRUE
```

## Source

[VMware community discussion](https://communities.vmware.com/t5/ESXi-Discussions/ESXi-8-x-Install-error-TSCs-are-out-of-sync-cpu1-gt-cpu27/td-p/2992745) ([Archive.org link](https://web.archive.org/web/20231111000308/https://communities.vmware.com/t5/ESXi-Discussions/ESXi-8-x-Install-error-TSCs-are-out-of-sync-cpu1-gt-cpu27/td-p/2992745))