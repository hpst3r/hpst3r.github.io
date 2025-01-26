---
title: "Getting started with Cloud-init on Hyper-V"
date: 2025-01-24T10:10:10-00:00
draft: false
---

# Intro

This will go over setting up an environment to build Cloud-init nocloud data drives (ISO, not vfat), converting nocloud disk images to VHDX, and then using those to hands-off provision VMs on Hyper-V hosts that can then be configured with Ansible.

More specifically, this will demonstrate using Cloud-init and the Alma Linux genericcloud image to stand up an Alma Linux VM on a Server 2025 host.

## What is Cloud-init?

Cloud-init is a Canonical project that's become the defacto standard for touchless configuration of Linux VMs over the years. Cloud-init is a package that can be installed on a Linux or Unix system that will read configuration files and provision a template VM on first boot for its environment.

It's a powerful tool, but, in this example, we're going to use the bare minimum functionality - I just want a machine with a hostname and user account so I can run Ansible against it.

I would highly recommend you give the [Cloud-init documentation](https://cloudinit.readthedocs.io/en/latest/) a visit if you would like a deeper dive.

## How is Cloud-init used?

In our case, we'll be feeding a Cloud-init-ready installation of Linux a "nocloud" disk image in Joliet ISO format. This disk will contain YAML that Cloud-init will read and do.. stuff with.

Since we're on Windows, we'll have to jump through one or two hoops to get going - all the software you'd normally use to manipulate VHDs and ISOs isn't quite so easy to come by when you're not on Linux.

## Why Hyper-V?

Because.

# Where do I get Generic Cloud images that I can follow along with?

- [AlmaLinux](https://wiki.almalinux.org/cloud/Generic-cloud.html)
- [Ubuntu](https://cloud-images.ubuntu.com/) - for 24.04 LTS, you might be able to skip some steps with the [Azure image that is already a VHDX](https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64-azure.vhd.tar.gz)
- [Debian](https://cdimage.debian.org/images/cloud/)
- [\*BSD, if you're feeling adventurous](https://bsd-cloud-image.org/)

# Dependencies

We'll need some software to work with VHDXes and ISOs, as previously mentioned.
1. qemu-img, a utility for working with disk images
2. genisoimage, mkisofs, xorriso, or oscdimg - tools for working with ISO images

## Installing Dependencies (Windows)

qemu-img alone can be downloaded from Cloudbase Solutions (company that supports cloudbase-init, which is basically cloud-init for Windows): https://cloudbase.it/qemu-img-windows/.

PowerShell:

```PowerShell
(irm https://cloudbase.it/downloads/qemu-img-win-x64-2_3_0.zip -outfile qemu-img.zip) |
extract-archive .\qemu-img.zip
```

This will bring with it a few DLLs:

```PowerShell
PS C:\Users\liam\projects\cloud-init-a9> gci qemu-img

    Directory: C:\Users\liam\projects\cloud-init-a9\qemu-img

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a---           9/18/2013  1:35 PM          86528 libgcc_s_sjlj-1.dll
-a---           3/30/2015  3:09 PM        2584872 libglib-2.0-0.dll
-a---           3/30/2015  3:10 PM          79707 libgthread-2.0-0.dll
-a---           3/30/2015  3:38 AM        1475928 libiconv-2.dll
-a---           3/30/2015  1:59 PM         464017 libintl-8.dll
-a---           9/18/2013  1:35 PM          18944 libssp-0.dll
-a---           6/16/2015  9:37 PM        5615492 qemu-img.exe
```

More complete QEMU binaries for Windows, including qemu-img, can be found at: https://qemu.weilnetz.de/w64/

oscdimg is a part of the Windows Assessment and Deployment Kit, which can be installed with the `winget` package manager, if you have that installed (24H2+, incl. Server 2025 have Winget working by default).

Note that installing the ADK this way will not prompt you to select which components to install - and, by default, the ADK has a 2gb footprint, which is a bit much for just a tool to build Joliet format ISOs.

```cmd
winget install Microsoft.WindowsADK
```

Alternatively, you can download and run the ADK installer directly from Microsoft. If you do it this way, you can select only the "Deployment Tools" feature and save some disk space, or download and run the oscdimg installer alone.

```PowerShell
(irm https://go.microsoft.com/fwlink/?linkid=2196127 -method GET -outfile adksetup.exe) |
.\adksetup.exe
```

{{< figure src="images/adk-dt.png" alt="A screenshot of the Windows ADK installer with only the Deployment Tools package selected." >}}

If you *install* the Deployment Tools, oscdimg will, by default, be at: `C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\amd64\Oscdimg\oscdimg.exe`.

If you *download* the Deployment Tools, you won't be prompted to select which features you want, but can then just run the oscdimg installer `Oscdimg (DesktopEditions)-x86_en-us.msi`.

If you downloaded the ADK to your Downloads directory, the path to the installer(s) including that for oscdimg would be `~\Downloads\ADK\Installers`.

I have an archive containing oscdimg and its required .cab files. You could, too. I'm not going to distribute it here, though, because Microsoft might get mad (probably not.)

For reference, those files are:

```txt
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         1/20/2025   8:15 PM          81130 52be7e8e9164388a9e6c24d01f6f1625.cab
-a----         1/20/2025   8:15 PM          80196 5d984200acbde182fd99cbfbe9bad133.cab
-a----         1/20/2025   8:15 PM          81299 9d2b092478d6cca70d5ac957368c00ba.cab
-a----         1/20/2025   8:16 PM          84314 bbf55224a0290f00676ddc410f004498.cab
-a----         1/19/2025  10:22 PM         417792 Oscdimg (DesktopEditions)-x86_en-us.msi
```

Finally, [here's a link to the Oscdimg help page at (on?) MS Learn.](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/oscdimg-command-line-options?view=windows-11)

## Installing Dependencies (Linux)

I typically use WSL on my workstation, so I figured I'd include this as well. It's a little easier to get going this way, but you might not want/be able to go the WSL route on a locked-down work laptop or Windows build box.

You can install both genisoimage and qemu-img on Debian 12 with:
```sh
# apt install genisoimage qemu-img
```
Both are in the default Debian 12 repositories.

On Enterprise Linux & friends (I use Alma), you'll have to install EPEL first.
```sh
# dnf install epel-release
# dnf install genisoimage qemu-img
```
# Converting a GenericCloud image to a VHDX

One command, once you've downloaded qemu-img somewhere.

```cmd
C:\Users\liam\projects\a9-ci-ex> .\qemu-img\qemu-img.exe convert -O vhdx C:\Users\liam\Downloads\AlmaLinux-9-GenericCloud-9.2-20230513.x86_64.qcow2 .\AlmaLinux-9-GenericCloud-9.2-20230513.x86_64.vhdx
```

# Creating cloud-init files

See the cloud-init documentation for more detail. I like to have a project directory set up as follows:

```PowerShell
PS C:\Users\liam\projects\a9-ci-ex> gci

    Directory: C:\Users\liam\projects\a9-ci-ex

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d----           1/18/2025  6:10 PM                cidata
-a---           1/18/2025  6:13 PM          55296 cloud-init.iso

PS C:\Users\liam\projects\cloud-init-a9> gci cidata

    Directory: C:\Users\liam\projects\cloud-init-a9\cidata

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a---           1/18/2025  3:27 PM             51 meta-data
-a---           1/18/2025  3:28 PM             81 user-data
```

If you're not using oscdimg, you don't need the contents of your image to be in a specific directory - you can point genisoimage at individual files.

Here's a sample meta-data file that provides an instance-id and hostname:
```yaml
instance-id: $VMName
local-hostname: $VMName
```

Here's a sample user-data file that creates a user, a password, and a ssh key:
```yaml
#cloud-config
users:
- name: Karl
  groups: users,wheel
  passwd: octopus77
  shell: /bin/bash
  lock_passwd: False
  ssh_authorized_keys:
   - 'id-ed25519 aabbccdd'

ssh_pwauth: False
```

I don't particularly care about more advanced configuration - I'd rather use Ansible for that.

# Creating the Cloud-init userdata ISO

With genisoimage (WSL):
```txt
genisoimage -output cloud-init.iso -volid cidata -joliet -rock ./cidata/user-data ./cidata/meta-data
```

With oscdimg:
```txt
C:\Program` Files` `(x86`)\Windows` Kits\10\Assessment` and` Deployment` Kit\Deployment` Tools\amd64\Oscdimg\oscdimg.exe -j1 -lcidata -r .\cidata .\cloud-init.iso
```

Explanation of options:
```txt
-j1 : encode Joliet Unicode file names AND ISO 9660 file names
-l : label
-r : resolve symbolic links
```

# Manually cloud-init a new VM with the Hyper-V MMC

Make a copy of the GenericCloud VHDX you created previously and intend to use. Save it somewhere on the storage you intend to boot the VM on. We'll use this shortly.

Open up the Hyper-V MMC (Hyper-V Management console, virtmgmt.msc) on something that can connect to the server or cluster you want to use.

Right-click a server and select New > Virtual Machine.

{{< figure src="images/mmc-ci-0.png" >}}

In the New Virtual Machine Wizard dialog, select:

A name

{{< figure src="images/mmc-ci-1.png" >}}

Generation 2 (UEFI)

{{< figure src="images/mmc-ci-2.png" >}}

An amount of memory. Ballooning can be enabled or disabled.

{{< figure src="images/mmc-ci-3.png" >}}

Optionally, connect the VM to a vSwitch.

{{< figure src="images/mmc-ci-4.png" >}}

On the "Connect Virtual Hard Disk" page, select "Use an existing virtual hard disk." Point this at the copy of your prepared GenericCloud or custom VM image.

In my example, I've dropped an Alma 9.5 VHDX in a D:\vhdx directory (in this case, D: lives on a Storage Spaces simple ReFS volume) that happens to be my default Hyper-V VM volume location, but this doesn't matter one bit.

{{< figure src="images/mmc-ci-5.png" >}}

Click "Finish" on the final page. Do not immediately launch the VM - there's a bit more prep work to do yet.

{{< figure src="images/mmc-ci-6.png" >}}

Back on the main Hyper-V Manager page, right-click the VM and select Settings.

{{< figure src="images/mmc-ci-7.png" >}}

Either disable Secure Boot or change the Secure Boot template to "Microsoft UEFI Certificate Authority." Since this example isn't a Windows VM, you'll need to allow third-party UEFI certificates or disable Secure Boot to let GRUB boot the system.

{{< figure src="images/mmc-ci-8.png" >}}

Select a SCSI controller, and add a DVD drive to it.

{{< figure src="images/mmc-ci-9.png" >}}

Point said DVD drive at your cloud-init ISO. In my case, I've already dropped this in my user's Downloads directory on the server.

Click OK when you're done.

{{< figure src="images/mmc-ci-10.png" >}}

Double-click the VM to bring up the remote console. Click Start.

Wait for the VM to boot. If the hostname is set, great! That means cloud-init worked.

{{< figure src="images/mmc-ci-11.png" >}}

# Manually cloud-init a new VM with Windows Admin Center

Make a copy of the GenericCloud VHDX you created previously and intend to use. Save it somewhere on the storage you intend to boot the VM on. We'll use this shortly.

Log on to Windows Admin Center. Access the server or cluster you want to create the VM on.

{{< figure src="images/wac-ci-0.png" >}}

Open the Virtual Machines snap-in.

Under the landing page (VM inventory), click Add > New.

{{< figure src="images/wac-ci-1.png" >}}

Configure processors, generation, networking to suit.

When you get to the Storage configuration, select Add...

{{< figure src="images/wac-ci-2.png" >}}

...then choose the VHDX you prepared earlier.

{{< figure src="images/wac-ci-3.png" >}}

Click Create.

{{< figure src="images/wac-ci-4.png" >}}

Wait for the VM to be created. The WAC VM plugin will update shortly.

Select the VM, then click Settings.

{{< figure src="images/wac-ci-5.png" >}}

Select Disks, then click Add Disk.

{{< figure src="images/wac-ci-6.png" >}}

Select "Use an use an existing virtual hard disk or ISO image file," then enter or browse to the path of your cloud-init data ISO. Click "Save disks settings."

{{< figure src="images/wac-ci-7.png" >}}

Once disk settings have been saved, enter the Security menu for the VM.

{{< figure src="images/wac-ci-8.png" >}}

Disable Secure Boot or change the Secure Boot template to "Microsoft UEFI Certificate Authority." Click "Save security settings," then click Close.

{{< figure src="images/wac-ci-9.png" >}}

Select the VM again, then expand the Power menu and click Start.

{{< figure src="images/wac-ci-10.png" >}}

You may need to refresh the page to see the state change. Do so if it does not refresh itself, then select Connect and click the Connect option.

{{< figure src="images/wac-ci-11.png" >}}

Enter your credentials when prompted.

You should now see a basic console with your chosen instance name. If you configured networking and/or ssh keys or users, you should be able to use them.

{{< figure src="images/wac-ci-12.png" >}}

# Scripting the unattended creation of a Hyper-V Alma 9 VM

The previous two methods are all well and good, but why would we go through all this effort to still need to click on things manually? That would be silly! And we're not silly! We're sysadmins, or, at the very least, we're pretending to be sysadmins!

So let's make things that don't require clicks! Terraform method coming Soon(TM).

Say I want to tell my computer what name, switch, VHD, username and public key I want it to use to give me a new VM:

```PowerShell
$ManicPgSQLParams = @{
	VMName = 'manictime-pgsql'
	VSwitchName = 'ext-untagged'
	OriginalVHDXPath = 'D:\VHDX\AlmaLinux-9-GenericCloud-122024.vhdx'
	Username = 'liam'
	PublicKey = 'ssh-ed25519 i-know-this-isn't-sensitive-but-you're-not-getting-it-free'
	Force = $true
}
```

Let's write a script!

This will:

- Generate basic cloud-init meta-data and user-data files to permit key-based ssh and passwordless sudo
- Turn said cloud-init files into an iso with oscdimg.exe
- Clone your desired template VHDX
- Cloud-init a VM from said template

Once this is done, assuming you have DHCP leases being registered, you can SSH in (or immediately kick off some Ansible!)

Pop over to [this script's associated Github repository](https://github.com/hpst3r/pwsh-hyper-v-ci-example) for more detailed usage instructions.

```PowerShell
Function CloudInit-VM {
	param (
		[string]$VMName = 'manictime-pgsql',
		[string]$VMSwitchName = 'ext-untagged',
		[string]$OriginalVHDXPath = 'D:\VHDX\AlmaLinux-9-GenericCloud-122024.vhdx',
		[string]$CloudInitMetadataPath = 'C:\tmp\cloud-init',
		[string]$CloudInitMetadataOutPath = 'C:\tmp\cloud-init.iso',
        [string]$Username = 'cloud-user',
        [string]$PublicKey,
        #[securestring]$HashedPassword = 'aabbccddeeff',
		[int]$vCPUs = 8,
		[long]$Memory = 8GB,
        [bool]$Force = $false
	)

	Function Set-MetadataFile {
		param (
			[string]$ParentPath,
			[string]$Content,
			[string]$MetadataType,
            [bool]$Force
		)

        Write-Host -Object `
            "Generating $($MetadataType)-data file for cloud-init at $($CloudInitMetadataPath)."

		if ( (-not ( Test-Path -Path "$($Path)\$($MetadataType)-data" )
            ) -or $Force) {
	
            $MetadataFile = @{
                Path = "$($CloudInitMetadataPath)\user-data"
                Value = $Content
            }
            
            Set-Content @MetadataFile
		
		} else {
		
			Write-Error -Message `
				"$($MetadataType)-data file in $($Path) already exists and -Force is not set. Exiting."
				
			return 1
			
		}

	}

	# populate the meta-data file

    $MetadataFile = @{
        ParentPath = $CloudInitMetadataPath
        Content = @"
instance-id: $VMName
local-hostname: $VMName
"@
        MetadataType = 'meta'
        Force = $Force
    }

    # populate the user-data file with ssh public key and username

    Set-MetadataFile @MetadataFile

    # if you want to pass a password, generate it with mkpasswd, and add, under your user:
    # hashed_passwd: $(ConvertFrom-SecureString -SecureString $HashedPassword -AsPlainText)
    # and set lock_passwd to False
    $UserdataFile = @{
        ParentPath = $CloudInitMetadataPath
        Content = @"
#cloud-config
users:
- name: $Username
  groups: users,wheel
  sudo: ALL=(ALL) NOPASSWD:ALL
  shell: /bin/bash
  lock_passwd: True
  ssh_authorized_keys:
   - $PublicKey

ssh_pwauth: False
"@
        MetadataType = 'user'
        Force = $Force
    }

    Set-MetadataFile @UserdataFile

    # convert metadata files to a Joliet cloud-init ISO
	
	& C:\Program` Files` `(x86`)\Windows` Kits\10\Assessment` and` Deployment` Kit\Deployment` Tools\amd64\Oscdimg\oscdimg.exe -j1 -lcidata -r $CloudInitMetadataPath $CloudInitMetadataOutPath
	
	# make a copy of the base vhdx - put it under the configured default vhdx path on the host

	$NewVHDXPath = "$((Get-VMHost).VirtualHardDiskPath)\$($VMName).vhdx"
	
	Write-Host -Object `
		"Checking for existence of path $($NewVHDXPath)."
		
	if (Test-Path -Path $NewVHDXPath) {
		# if the -Force flag IS NOT set, do not overwrite the existing VHDX and terminate.
		if (-not $Force) {
		
			Write-Error -Message `
				'Default new VHDX path is already occupied and -Force is not set. Exiting.'
			return 1
			
		}
		
		# if the -Force flag IS set, attempt to remove the existing VHDX.
		# This WILL NOT SUCCEED if a VM is using said VHDX.
		
		if ($Force) {
		
			Write-Host -Object `
				"-Force is set. Attempting to remove VHDX at $($NewVHDXPath)."
				
			try {
			
				Remove-Item `
					-Path $NewVHDXPath `
					-Force `
					-Confirm:$false `
					-ErrorAction 'Stop'
					
			} catch {
			
				Write-Error -Message @"
-Force is set, but failed to remove existing file at desired clone VHDX path $($NewVHDXPath).
$($_)
Terminating.
"@

                return 1

			}
		}
	}

    # copy the VM template VHDX for our new VM.

	Write-Host -Object `
		"Copying template VHDX from $($OriginalVHDXPath) to $($NewVHDXPath)."

	try {

		Copy-Item -Path $OriginalVHDXPath -Destination $NewVHDXPath

	} catch {

		Write-Error -Message `
			"Copy failed with error: $($_). Terminating."

        return 1

	}

	Write-Host -Object @"
Attempting to create VM $($VMName) with specifications:

Threads: $($vCPUs)
Memory (Mb): $($Memory/([Math]::Pow(2,20)))
Switch: $($VMSwitchName)

"@

	try {
		# create the VM
		$VMParams = @{
			Name = $VMName
			SwitchName = $VMSwitchName
			Generation = 2
			MemoryStartupBytes = "$($Memory)" # is there a reason for this?
			ErrorAction = 'Stop'
		}
		
		$VM = New-VM @VMParams
	} catch {

		Write-Error -Message @"
Failed to create VM $($VMName) with error:
$($_)
Terminating script.
"@
		return 1

	}
	
	# add the copy of the VHDX to the VM
    
    Write-Host -Object `
        "Assigning clone VHDX $($NewVHDXPath) to VM $($VMName)."

    try {

        $VMDiskParams = @{
            VM = $VM
            ControllerType = 'SCSI'
            ControllerNumber = 0
            ControllerLocation = 0
            Path = $NewVHDXPath
            ErrorAction = 'Stop'
        }
        
        Add-VMHardDiskDrive @VMDiskParams

    } catch {

        Write-Error -Message `
            "Assigning VHDX $($NewVHDXPath) to VM $($VMName) failed with error: $($_). Terminating."

        return 1

    }

	# configure Secure Boot to allow non-MS signatures
	# you can also set by GUID 272e7447-90a4-4563-a4b9-8e4ab00526ce

    Write-Host -Object `
        "Setting $($VMName)'s UEFI to allow third-party Secure Boot signatures."

	Set-VMFirmware -VM $VM -SecureBootTemplate 'MicrosoftUEFICertificateAuthority'
	
	# add the cloud-init metadata image to the VM

    Write-Host -Object `
        "Assigning the $($CloudInitMetadataOutPath) metadata image to VM $($VMName)."
    
    try {

        $VMDvdParams = @{
            VM = $VM
            ControllerNumber = 0
            ControllerLocation = 1
            Path = $CloudInitMetadataOutPath
            ErrorAction = 'Stop'
        }
        
        Add-VMDvdDrive @VMDvdParams

    } catch {

        Write-Error -Message `
            "Failed to assign $($CloudInitMetadataOutPath) to VM $($VMName) with error: $($_). Terminating."

        return 1

    }

    # make sure first boot device is the cloned VHDX

    Write-Host -Object `
        "Setting VM $($VMName)'s first boot device to its hard drive (cloned VHDX.)"

	Set-VMFirmware -VM $VM -FirstBootDevice (Get-VMHardDiskDrive -VM $VM)
	
	# start the VM

    Write-Host -Object `
        "Starting VM $($VMName)."

	try {

        $StartVM = @{
            VM = $VM
            ErrorAction = 'Stop'
        }
    
        Start-VM @StartVM

    } catch {

        Write-Error -Message `
            "Failed to start VM $($VMName) with error: $($_). Terminating."

        return 1

    }

    # wait for Cloud-init
    Write-Host -Object `
        'Waiting 30 seconds for cloud-init.'

	Start-Sleep(30)
	
	# remove the Cloud-init disk

    Write-Host -Object `
        "Removing cloud-init drive from VM $($VMName)."
        
	Remove-VMDVDDrive -VMName $VM.Name -ControllerNumber 0 -ControllerLocation 1

    Write-Host -Object @"

Successfully created VM $($VMName) with:

Threads: $($vCPUs)
Memory (Mb): $($Memory/([Math]::Pow(2,20)))
Switch: $($VMSwitchName)

Original VHDX: $($OriginalVHDXPath)
Cloned VHDX: $($NewVHDXPath)

Generated cloud-init ISO: $($CloudInitMetadataOutPath)

"@

    Return 0
}
```