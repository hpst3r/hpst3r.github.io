---
title: "Cleaning up ghost Saved-Critical Hyper-V VMs after nuking all their files"
date: 2025-04-06T00:30:59-00:00
draft: false
---

So, I decided to upgrade my laptop. Part of that upgrade was the very caring removal of my D: drive, since the new board only had a single M.2 slot. My D: drive happened to be a second SSD with.. my Hyper-V config and VM disk directories.

Once I'd powered the machine back on and jumped back in to try and get rid of my ghost VMs via PowerShell, I was greeted with a pleasant sight:

```txt
Administrator in ~
❯ get-vm
Get-VM: Hyper-V encountered an error trying to access an object on computer 'LIAM-P1G4I-0' because the object was not found. The object might have been deleted, or you might not have permission to perform the task. Verify that the Virtual Machine Management service on the computer is running. If the service is running, try to perform the task again by using Run as Administrator.
Get-VM: Hyper-V encountered an error trying to access an object on computer 'LIAM-P1G4I-0' because the object was not found. The object might have been deleted, or you might not have permission to perform the task. Verify that the Virtual Machine Management service on the computer is running. If the service is running, try to perform the task again by using Run as Administrator.
Get-VM: Hyper-V encountered an error trying to access an object on computer 'LIAM-P1G4I-0' because the object was not found. The object might have been deleted, or you might not have permission to perform the task. Verify that the Virtual Machine Management service on the computer is running. If the service is running, try to perform the task again by using Run as Administrator.
```

Yeah.. it doesn't like that.

The VMMS is running, of course:

```txt
Administrator in ~
❯ get-service vmms

Status   Name               DisplayName
------   ----               -----------
Running  vmms               Hyper-V Virtual Machine Management
```

Hyper-V Manager shows the VMs' names with state "Saved-Critical" - right-clicking a VM gives me no option to delete it.

`Get-VM | Remove-VM` also doesn't do anything:

```txt
Administrator in ~
❯ get-vm | remove-vm -force
Get-VM: Hyper-V encountered an error trying to access an object on computer 'LIAM-P1G4I-0' because the object was not found. The object might have been deleted, or you might not have permission to perform the task. Verify that the Virtual Machine Management service on the computer is running. If the service is running, try to perform the task again by using Run as Administrator.
Get-VM: Hyper-V encountered an error trying to access an object on computer 'LIAM-P1G4I-0' because the object was not found. The object might have been deleted, or you might not have permission to perform the task. Verify that the Virtual Machine Management service on the computer is running. If the service is running, try to perform the task again by using Run as Administrator.
Get-VM: Hyper-V encountered an error trying to access an object on computer 'LIAM-P1G4I-0' because the object was not found. The object might have been deleted, or you might not have permission to perform the task. Verify that the Virtual Machine Management service on the computer is running. If the service is running, try to perform the task again by using Run as Administrator.
```

So let's try to clean up the VM cache; it's probably the only place these things might be.

**DO NOT DO THE FOLLOWING** if you want to keep any VMs on the system and do not know exactly what you are doing. If you copy and paste this, you will delete **ALL** of your VM cache entries.

If you want to preserve VM cache entries that aren't screwed up, you will need to filter on the VM ID (`get-vm | select vmid`) and (probably) only remove the directories/vmcx files that do not have a match.


```txt
Administrator in ~
❯ stop-service vmms -f # stop the Virtual Machine Management service

Administrator in ~
❯ sl 'C:\ProgramData\Microsoft\Windows\Hyper-V\Virtual Machines Cache'

Administrator in Windows\Hyper-V\Virtual Machines Cache
❯ gci # get list of cached VMs

    Directory: C:\ProgramData\Microsoft\Windows\Hyper-V\Virtual
Machines Cache

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d----           3/23/2025    21:30                8E9C78FA-5C45-4903-AE
                                                  47-D3FCC1A082EE
d----            2/6/2025    21:54                A7C4AA4E-9A5F-4C42-A0
                                                  60-7D8DCB6D8F9A
-a---          11/22/2024    13:46          16418 12664FD0-C490-48A9-B2
                                                  26-88D7D037C318.vmcx
-a---           3/23/2025    22:01          24576 8E9C78FA-5C45-4903-AE
                                                  47-D3FCC1A082EE.vmcx
-a---            2/6/2025    21:56          24576 A7C4AA4E-9A5F-4C42-A0
                                                  60-7D8DCB6D8F9A.vmcx


Administrator in Windows\Hyper-V\Virtual Machines Cache
❯  gci | ri -fo -re # recursively remove all children of the VM Cache directory DO NOT DO THIS - if you want to keep your VMs, filter by VMID

Administrator in Windows\Hyper-V\Virtual Machines Cache
❯ gci

Administrator in Windows\Hyper-V\Virtual Machines Cache
❯ sl '..\Snapshots Cache'

Administrator in Windows\Hyper-V\Snapshots Cache
❯  gci | ri -fo -re # recursively remove all children of the SShot Cache directory DO NOT DO THIS - if you want to keep your snapshots, filter by VMID

Administrator in Windows\Hyper-V\Virtual Machines Cache
❯ start-service vmms

Administrator in Windows\Hyper-V\Virtual Machines Cache
❯ sl ~

Administrator in ~
❯ get-vm

Administrator in ~
❯
```

There's also some lingering config junk in the binary file `C:\ProgramData\Microsoft\Windows\Hyper-V\data.vmcx`. Here's a sample:

```txt
g���'_12664FD0-C490-48A9-B226-88D7D037C318_�C:\ProgramData\Microsoft\Windows\Hyper-V\Virtual Machines Cache\12664FD0-C490-48A9-B226-88D7D037C318.vmcx��
    4��z'_88FBFE78-1489-4A6B-BA0B-82BD3BCFE3E8_�D:\hv\vmcfg\Virtual Machines\88FBFE78-1489-4A6B-BA0B-82BD3BCFE3E8.vmcx�
```

We can make Windows recreate this file if we stop VMMS, get rid of the file, and restart VMMS:


```txt
Administrator in Microsoft\Windows\Hyper-V
❯ stop-service vmms -f

Administrator in Microsoft\Windows\Hyper-V
❯ mi data.vmcx data.vmcx.old

Administrator in Microsoft\Windows\Hyper-V
❯ start-service vmms

```

If you want to preserve any of your VMs, you'll have to import their config again, and this will require stopping the VM somehow (with guest access or by killing the relevant `vmwp`). If it's not in the cache, the VM won't show up in the MMC or be queried with your cmdlets.

It looks like we can import this...

{{< figure src="/images/missing-vm.png" >}}

But no! Failure! The MMC won't be able to do this at all, even if we stop the VM.

{{< figure src="/images/import-fail.png" >}}

Trying to import the VM with PowerShell also fails (while it's running):

```txt
Administrator in ~
❯ import-vm 'C:\vmcfg\Virtual Machines\D060821D-292A-41B4-946E-025D97DDFF0E.vmcx'
Import-VM: Import failed. Import task failed to copy file.

Import failed. Import task failed to copy file from 'C:\vmcfg\Virtual Machines\D060821D-292A-41B4-946E-025D97DDFF0E.vmgs' to '': The process cannot access the file because it is being used by another process. (0x80070020).
```

You can shut the VM down from a console or over the network - the Hyper-V guest console itself will still work, but the buttons in the control bar will not:

{{< figure src="images/fail-to-stop.png" >}}

If you don't have access to the console anymore and the machine isn't accessible over the network, you'll have to go hunting for VM worker processes (`vmwp.exe`). Be careful; if you're using VBS you might take down your machine (I know that tinkering with the `vmcompute` service will bring buffer overflows and all kinds of crashes, so I imagine killing your own VBS worker will do the same!)

To blow all your VM workers away, effectively forcing your VMs off, you can:

```txt
Administrator in ~
❯ get-process vmwp

 NPM(K)    PM(M)      WS(M)     CPU(s)      Id  SI ProcessName
 ------    -----      -----     ------      --  -- -----------
     28    11.64      31.39       0.00    2504   0 vmwp

Administrator in ~
❯ get-process vmwp | stop-process -force
```

Then, try importing the VM or VMs again. If you're only vaguely following along, note that the `vmms` needs to be running for this to work.

```txt
Administrator in ~
❯ import-vm 'C:\vmcfg\Virtual Machines\D060821D-292A-41B4-946E-025D97DDFF0E.vmcx'

Name      State CPUUsage(%) MemoryAssigned(M) Uptime   Status             Version
----      ----- ----------- ----------------- ------   ------             -------
sacrifice Off   0           0                 00:00:00 Operating normally 12.0
```

Anyway, problem solved! My system doesn't know about these VMs anymore. Mostly. I doubt Microsoft missed a chance to dump crap in the registry. But this install is good as dead anyway, and it's late. Maybe this deserves a part two when I have some caffiene.