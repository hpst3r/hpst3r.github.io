---
title: "Dell T5810 ESXi 8 inaccurate TSC sync errors"
date: 2024-04-09T12:34:56-00:00
draft: false
---

## Problem

Both of my Dell T5810s are failing to boot ESXi 8.0U2 - meet PSOD with a TSC sync error. E5 2660 v4 and E5 2690 v4. Counter is off by same value every boot, and both machines experience no instability with Win/Linux, so bypass is necessary and reasonable to boot ESXi.

## Fix

Shift+O to add command line options to boot 1) installer and 2) install for the first time. Add command line options:

```tscSyncSkip=TRUE timerForceTSC=TRUE```

After first boot, SSH to the machine and either:

1) use esxcli:

```
esxcli system settings kernel set --setting=tscSyncSkip --value=TRUE

esxcli system settings kernel set --setting=timerForceTSC --value=TRUE
```

2) edit /bootbank/boot.cfg (kernelopt line)

```kernelopt=... tscSyncSkip=TRUE timerForceTSC=TRUE```

to set the above boot options.

## Source

[VMware community discussion](https://communities.vmware.com/t5/ESXi-Discussions/ESXi-8-x-Install-error-TSCs-are-out-of-sync-cpu1-gt-cpu27/td-p/2992745) ([Archive.org link](https://web.archive.org/web/20231111000308/https://communities.vmware.com/t5/ESXi-Discussions/ESXi-8-x-Install-error-TSCs-are-out-of-sync-cpu1-gt-cpu27/td-p/2992745))