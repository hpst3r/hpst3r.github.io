---
title: "Resolving ESXi 8.0 esxcli software profile MemoryError with hostupdate depot"
date: 2025-03-16T11:13:59-00:00
draft: false
---

## Problem

VMware ESXi 8 host fails to query the hostupdate (hostupdate.broadcom.com) depot's contents with a `MemoryError` when an `esxcli software profile update` command is issued.

This also prevents ESXi from being updated (does not just affect querying the depot).

```txt
[root@vmware:~] esxcli software profile update -p ESXi-8.0U3c-24414501-standard -d https://hostupdate.broadcom.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml
 [MemoryError]
 Please refer to the log file for more details.
```

## Verify

Search the `/usr/lib/vmware/esxcli-software` file for the `mem=` argument being passed to the Python interpreter. The default 300 Mb is insufficient for querying the chunky ESXi 8 hostupdate depot, though 7.0 and 6.7 will still work fine.


```txt
[root@vmware:~] cat /usr/lib/vmware/esxcli-software | grep -i mem=
#!/usr/bin/python ++group=esximage,mem=300
```

If you see `mem=300` and received a MemoryError, continue.

# Solution

```txt
# disable VisorFS immutability
esxcli system settings advanced set -o /VisorFS/VisorFSPristineTardisk -i 0

# make backup and working copy of esxcli-software
cp /usr/lib/vmware/esxcli-software /usr/lib/vmware/esxcli-software.bak
cp /usr/lib/vmware/esxcli-software /usr/lib/vmware/esxcli-software.old

# replace string mem=300 with mem=500 in working copy using sed
sed -i 's/mem=300/mem=500/g' /usr/lib/vmware/esxcli-software.bak

# verify change
# should look like: #!/usr/bin/python ++group=esximage,mem=500
cat /usr/lib/vmware/esxcli-software.bak | grep -i mem=

# replace original file with working copy
mv -f /usr/lib/vmware/esxcli-software.bak /usr/lib/vmware/esxcli-software

# reenable VisorFS immutability
esxcli system settings advanced set -o /VisorFS/VisorFSPristineTardisk -i 1
```

The `esxcli software profile update` command should now succeed (assuming this was the culprit for you, too).

This change will be overridden at the next boot, so you don't have to undo anything to get the system back to its original state.

Tested on: lots of flavors of ESXi 8 at this point, incl. up to 8.0U3c.

Reference: [William Lam (VMware engineer) blog post referencing internal disc.](https://williamlam.com/2024/03/quick-tip-using-esxcli-to-upgrade-esxi-8-x-throws-memoryerror-or-got-no-data-from-process.html)