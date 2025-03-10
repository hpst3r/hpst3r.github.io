---
title: "Updating standalone VMware ESXi hosts from the online hostupdate depot with esxcli"
date: 2025-03-08T10:12:59-00:00
draft: false
---

## Problem

You have a standalone VMware ESXi server, or a non-clustered (no vCenter available) set of servers that need to be updated.

## Solution

Before starting:

- **Verify that your IPMI/BMC is plugged in!** If it is not, you will not have a good time if your sole ESXi server fails to write the bootloader and wipes its config at a site three hours away. Make sure it's accessible somehow.

- **Check your server's DNS settings!** Name resolution won't work once you suspend VMs if the server is pointed at a guest it's running.

- **Check site clients' DNS settings!** You will not have a good time if something goes wrong and the only DNS servers at the site are down (because they were on your VMware box). Your RMM may not work. Don't ask how I know.

Now, on to the good stuff.

### TL:DR

#### Step by step update procedure with esxcli

```txt
# enable ssh via web UI or PowerCLI, then connect to the server via SSH first.
ssh root@server

# see what version of ESXi is currently running
esxcli software profile get

# enable outbound HTTP if you need to perform web requests to update the server
esxcli network firewall ruleset set -e true -r httpClient

# list available profiles
# if this fails with a MemoryError on 8.x, see the following section
esxcli software sources profile list -d https://hostupdate.broadcom.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml

# perform a dry run of the update.
# remember to replace the version (ESXi-7.0U3s-24585291-standard)
# with the desired profile from the output of the profile list command above.
esxcli software profile update -p ESXi-7.0U3s-24585291-standard -d https://hostupdate.broadcom.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml --dry-run

# suspend your VMs, enter maintenance mode
esxcli system maintenanceMode set -e true

# install the update
esxcli software profile update -p ESXi-7.0U3s-24585291-standard -d https://hostupdate.broadcom.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml

# disable outbound HTTP
esxcli network firewall ruleset set -e false -r httpClient

# restart the system to boot the new install
esxcli system shutdown reboot -r 'ESXi-7.0U3s-24585291-standard update'

```

#### MemoryError fix

```txt
# disable VisorFS immutability
esxcli system settings advanced set -o /VisorFS/VisorFSPristineTardisk -i 0

# make backup and working copy of esxcli-software
cp /usr/lib/vmware/esxcli-software /usr/lib/vmware/esxcli-software.bak
cp /usr/lib/vmware/esxcli-software /usr/lib/vmware/esxcli-software.old

# replace string mem=300 with mem=500 in working copy using sed
sed -i 's/mem=300/mem=500/g' /usr/lib/vmware/esxcli-software.bak

# verify change
cat /usr/lib/vmware/esxcli-software.bak | grep -i mem=
#!/usr/bin/python ++group=esximage,mem=500

# replace original file with working copy
mv -f /usr/lib/vmware/esxcli-software.bak /usr/lib/vmware/esxcli-software

# reenable VisorFS immutability
esxcli system settings advanced set -o /VisorFS/VisorFSPristineTardisk -i 1
```

### In-depth process

You can either:
- [manually update the hosts from a depot file on local, shared or network storage. This is a very similar process. See this older post (wporter.org) for more info.](/updating-esxi-8-hosts-from-a-depot-file)
- use Broadcom's online 'hostupdate' depot for easy online updates.

We'll be going over the latter here.

The full update process goes something like the following:

### Log into the web UI, enable SSH, and SSH to the server, or connect with PowerCLI.

To enable SSH via the web UI, click on Actions > Services > Enable SSH on the Host page, then use your preferred SSH client to connect to the server.

{{< figure src="images/actions-ssh.png" alt="The Actions > Enable SSH menu on a VMware ESXi Host UI." >}}

```txt
~
❯ ssh root@server.lab.wporter.org
The time and date of this login have been sent to the system logs.

WARNING:
   All commands run on the ESXi shell are logged and may be included in
   support bundles. Do not provide passwords directly on the command line.
   Most tools can prompt for secrets or accept them from standard input.

VMware offers powerful and supported automation tools. Please
see https://developer.vmware.com for details.

The ESXi Shell can be disabled by an administrative user. See the
vSphere Security documentation for more information.
[root@server:~]
```

To connect via PowerCLI, import the module, optionally allow untrusted certificates and connect to the server by IP or FQDN. You will then be able to issue PowerCLI commands.

```txt
~
❯ Import-Module VMware.PowerCLI -Prefix ESX
~
❯ Set-PowerCLIConfiguration -InvalidCertificateAction Prompt
~
❯ Connect-VIServer server.lab.wporter.org
```

To enable SSH with PowerCLI:

```txt
~
❯ Get-VMHostService | Where-Object { $_.Key -eq 'TSM-SSH' } | Start-VMHostService
```

### Verify the current version of VMware ESXi

With PowerCLI:

```txt
~
❯ Get-VMHost | Select Version, Build

Version Build
------- -----
8.0.3   24022510
```

With esxcli:

```txt
[root@server:~] esxcli software profile get
ESXi-8.0U3-24022510-standard
   Name: ESXi-8.0U3-24022510-standard
   Vendor: VMware, Inc.
```

### Check the depot for available releases

Enable outbound HTTP, then query the depot with the `esxcli software sources profile list` command.

```txt
[root@server:~] esxcli network firewall ruleset set -e true -r httpClient
[root@server:~] esxcli software sources profile list -d https://hostupdate.broadcom.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml
```

Look through the list and figure out which version of ESXi you would like to update to (e.g., ESXi-7.0U3s-24585291-standard).

#### MemoryError fix

If you are using VMware ESXi 8.x, you will likely have to increase the amount of memory that esxcli can use to enumerate the depot, as the 300 Mb default is not sufficient. If you run out of memory while querying the depot, you'll receive the following error:

```txt
[root@server:~] esxcli software profile update -p ESXi-8.0U3d-24585383-standard -d https://hostupdate.broadcom.com/softwa
re/VUM/PRODUCTION/main/vmw-depot-index.xml --no-hardware-warning
 [MemoryError]
 Please refer to the log file for more details.
```

The fix for this issue is to modify a line (below) in the esxcli-software script that tells the Python interpreter to limit memory usage to 300 mb.

```txt
[root@server:~] cat /usr/lib/vmware/esxcli-software | grep -i mem=
#!/usr/bin/python ++group=esximage,mem=300
```

To easily set this to 500 mb, you can, from esxcli:

```txt
# disable VisorFS immutability
esxcli system settings advanced set -o /VisorFS/VisorFSPristineTardisk -i 0

# make backup and working copy of esxcli-software
cp /usr/lib/vmware/esxcli-software /usr/lib/vmware/esxcli-software.bak
cp /usr/lib/vmware/esxcli-software /usr/lib/vmware/esxcli-software.old

# replace string mem=300 with mem=500 in working copy using sed
sed -i 's/mem=300/mem=500/g' /usr/lib/vmware/esxcli-software.bak

# verify change
cat /usr/lib/vmware/esxcli-software.bak | grep -i mem=
#!/usr/bin/python ++group=esximage,mem=500

# replace original file with working copy
mv -f /usr/lib/vmware/esxcli-software.bak /usr/lib/vmware/esxcli-software

# reenable VisorFS immutability
esxcli system settings advanced set -o /VisorFS/VisorFSPristineTardisk -i 1
```

Many thanks to William Lam for [his post on the issue from 2024/03, which I referenced the first time I ran into this.](https://williamlam.com/2024/03/quick-tip-using-esxcli-to-upgrade-esxi-8-x-throws-memoryerror-or-got-no-data-from-process.html)

This will be reset on reboot - you do NOT need to set this back to 300.

### Suspend running VMs

From the web UI, click on 'Virtual Machines' to access the Virtual Machines overview page, select the ones you'd like to suspend, and click Suspend in the actions bar above the VM list.

With esxcli:

```txt
vim-cmd vmsvc/getallvms
vim-cmd vmsvc/power.suspend VMID
```

With PowerCLI:

```txt
~
❯ Get-VM | Suspend-VM
```

### Enter maintenance mode

From the web UI:

{{< figure src="images/actions-maintenance.png" alt="The Actions > Enter Maintenance Mode option in the ESXi host client web UI." >}}

With esxcli:

```txt
[root@server:~] esxcli system maintenanceMode set -e true
```

With PowerCLI:

```txt
~
❯ set-vmhost -state Maintenance

Name         ConnectionState PowerState NumCpu CpuUsageMhz CpuTotalMhz   MemoryUsageGB   MemoryTotalGB Version
----         --------------- ---------- ------ ----------- -----------   -------------   ------------- -------
server       Maintenance     PoweredOn      20         114       51980           2.877         127.893   8.0.3
```

### Update the system

If you are using unsupported hardware or future unsupported hardware, you will need to append the `--no-hardware-warning` option to upgrade commands or they will fail.

There definitely is a way to do this whole process with PowerCLI, but I haven't gotten around to figuring it out yet. [See this link to someone updating ESXi 6.5 machines with PowerCLI, maybe?](https://www.codyhosterman.com/2017/12/upgrading-esxi-environment-with-powercli/)

Perform a dry run to see which VIBs will change:

```txt
[root@server:~] esxcli software profile update -p ESXi-8.0U3d-24585383-standard -d https://hostupdate.broadcom.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml --dry-run
```

Install the update:

```txt
[root@server:~] esxcli software profile update -p ESXi-8.0U3d-24585383-standard -d https://hostupdate.broadcom.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml
```

Disable the HTTP in firewall rule now that we're done with it:

```txt
[root@server:~] esxcli network firewall ruleset set -e false -r httpClient
```

Restart the machine to apply the update:

```txt
[root@server:~] esxcli system shutdown reboot -r 'ESXi-8.0U3d-24585383 update'
```