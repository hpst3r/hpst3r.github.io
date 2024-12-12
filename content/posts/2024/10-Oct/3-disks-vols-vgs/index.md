---
title: "PowerShell: some commands for working with disks, volumes, VGs"
date: 2024-10-15T12:34:56-00:00
draft: false
---

One-liner to format a new disk as a single NTFS partition:
```pwsh
Get-Disk $DiskId | Initialize-Disk | New-Partition -UseMaximumSize | Format-Volume -Filesystem "NTFS" -AsJob -DriveLetter "Z"
```

`Get-Partition` will get you volumes (code block is missing MSR & EFI for brevity):

```pwsh
PS C:\Users\liam\Documents\tftp> get-partition | fl

UniqueId             : {00000000-0000-0000-0000-000100000000}eui.0000000001000000E4D25CA7DB4F5501
AccessPaths          : {D:\, \\?\Volume{aa409295-7d41-48b2-82e9-2154277448e5}\}
DiskNumber           : 1
DiskPath             : \\?\scsi#disk&ven_nvme&prod_intel_ssdpeknu01#5&159106e5&0&000000#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}
DriveLetter          : D
Guid                 : {aa409295-7d41-48b2-82e9-2154277448e5}
IsActive             : False
IsBoot               : False
IsHidden             : False
IsOffline            : False
IsReadOnly           : False
IsShadowCopy         : False
IsDAX                : False
IsSystem             : False
NoDefaultDriveLetter : False
Offset               : 16777216
OperationalStatus    : Online
PartitionNumber      : 2
Size                 : 953.85 GB
Type                 : Basic

UniqueId             : {00000000-0000-0000-0000-501100000000}eui.002538BA11B7787A
AccessPaths          : {C:\, \\?\Volume{ebf986f5-f46c-4e19-81fc-7a27c989a895}\}
DiskNumber           : 0
DiskPath             : \\?\scsi#disk&ven_nvme&prod_samsung_mzvl21t0#5&27c54347&0&000000#{53f56307-b6bf-11d0-94f2-00a0c91efb8b}
DriveLetter          : C
Guid                 : {ebf986f5-f46c-4e19-81fc-7a27c989a895}
IsActive             : False
IsBoot               : True
IsHidden             : False
IsOffline            : False
IsReadOnly           : False
IsShadowCopy         : False
IsDAX                : False
IsSystem             : False
NoDefaultDriveLetter : False
Offset               : 290455552
OperationalStatus    : Online
PartitionNumber      : 3
Size                 : 953.01 GB
Type                 : Basic
```

`Get-PhysicalDisk` will give you serial numbers (the actual disks, if you can believe that)

Can also be used to remove all the partitions on a disk pretty easily:

`Get-Disk` series of cmdlets work with logical disks (kinda sorta a volume command?)

```pwsh
PS C:\Users\liam> get-disk | fl

UniqueId           : eui.0000000001000000E4D25CA7DB4F5501
Number             : 1
Path               : \\?\scsi#disk&ven_nvme&prod_intel_ssdpeknu01#5&159106e5&0&000000#{53f56307-b6bf-11d0-94f2-00a0c91e
                     fb8b}
Manufacturer       :
Model              : INTEL SSDPEKNU010TZ
SerialNumber       : 0000_0000_0100_0000_E4D2_5CA7_DB4F_5501.
Size               : 953.87 GB
AllocatedSize      : 1024208494592
LogicalSectorSize  : 512
PhysicalSectorSize : 4096
NumberOfPartitions : 2
PartitionStyle     : GPT
IsReadOnly         : False
IsSystem           : False
IsBoot             : False

UniqueId           : eui.002538BA11B7787A
Number             : 0
Path               : \\?\scsi#disk&ven_nvme&prod_samsung_mzvl21t0#5&27c54347&0&000000#{53f56307-b6bf-11d0-94f2-00a0c91e
                     fb8b}
Manufacturer       :
Model              : SAMSUNG MZVL21T0HCLR-00BL7
SerialNumber       : 0025_38BA_11B7_787A.
Size               : 953.87 GB
AllocatedSize      : 1024208494592
LogicalSectorSize  : 512
PhysicalSectorSize : 4096
NumberOfPartitions : 4
PartitionStyle     : GPT
IsReadOnly         : False
IsSystem           : True
IsBoot             : True
```

TODO: I have some BitLocker stuff to add here. Eventually. Maybe?

`get-physicaldisk -SerialNumber 21294T805133 | get-disk | get-partition | remove-partition`

Or reset disks:

`Get-PhysicalDisk -SerialNumber 21294T805133 | Reset-PhysicalDisk`

Disks may (always?) require a reset to toggle their CanPool state, even if there's nothing on them.

```pwsh
PS C:\Users\Administrator> get-physicaldisk -SerialNumber 21294T805133 | select CanPool

CanPool
-------
  False

PS C:\Users\Administrator> get-physicaldisk -SerialNumber 21294T805133 | reset-physicaldisk
PS C:\Users\Administrator> get-physicaldisk -SerialNumber 21294T805133 | select CanPool

CanPool
-------
   True
```
## Create a pool by resetting a PhysicalDisk for use as a PV then creating a VG

```pwsh
New-StoragePool -FriendlyName SsdDataPool0 -StorageSubsystemFriendlyName (Get-StorageSubsystem).FriendlyName -PhysicalDisks (Get-PhysicalDisk -CanPool $true)
```

```pwsh
$PoolMembers = Get-PhysicalDisk -SerialNumber 21294T804630
$PoolMembers | Reset-PhysicalDisk
New-StoragePool `
	-FriendlyName SsdDataPool0 `
	-StorageSubsystemFriendlyName (Get-StorageSubsystem).FriendlyName `
	-PhysicalDisks $PoolMembers
```
### Example output PS5.1 2K25
```pwsh
PS C:\Users\Administrator> Get-PhysicalDisk -SerialNumber 21294T804630

Number FriendlyName            SerialNumber MediaType CanPool OperationalStatus HealthStatus Usage            Size
------ ------------            ------------ --------- ------- ----------------- ------------ -----            ----
0      WDC  WDS100T2B0A-00SM50 21294T804630 SSD       True    OK                Healthy      Auto-Select 931.51 GB


PS C:\Users\Administrator> $PoolMembers = Get-PhysicalDisk -SerialNumber 21294T804630
PS C:\Users\Administrator> $PoolMembers | Reset-PhysicalDisk
PS C:\Users\Administrator> New-StoragePool `
>> -FriendlyName SsdDataPool0 `
>> -StorageSubsystemFriendlyName (Get-StorageSubsystem).FriendlyName `
>> -PhysicalDisks $PoolMembers

FriendlyName OperationalStatus HealthStatus IsPrimordial IsReadOnly      Size AllocatedSize
------------ ----------------- ------------ ------------ ----------      ---- -------------
SsdDataPool0 OK                Healthy      False        False      931.01 GB        256 MB
```

## Create a LV on the VG (pool)
Resiliency settings **may** be changed later
[MS Learn - New-VirtualDisk](https://learn.microsoft.com/en-us/powershell/module/storage/new-virtualdisk?view=windowsserver2022-ps)
[MS Learn - Set-ResiliencySetting](https://learn.microsoft.com/en-us/powershell/module/storage/set-resiliencysetting?view=windowsserver2022-ps)
_NumberofDataCopies_
_PhysicalDiskRedundancy_

```pwsh
PS C:\Users\Administrator> New-VirtualDisk -StoragePoolFriendlyName SsdDataPool0 -FriendlyName SsdDataLv0 -ResiliencySettingName Simple -UseMaximumSize

FriendlyName ResiliencySettingName FaultDomainRedundancy OperationalStatus HealthStatus   Size FootprintOnPool StorageEfficiency
------------ --------------------- --------------------- ----------------- ------------   ---- --------------- --------
SsdDataLv0   Simple                0                     OK                Healthy      929 GB          930 GB   99.89%
```

## Format the disk
This disk will now appear as a logical disk (`Get-Disk` cmdlet)

```pwsh
PS C:\Users\Administrator> Get-Disk | Format-List

Number            : 1
FriendlyName      : KINGSTON S...
SerialNumber      : 50026B7685045DAD
HealthStatus      : Healthy
OperationalStatus : Online
TotalSize         : 223.57 GB
PartitionStyle    : GPT

Number            : 2
FriendlyName      : SsdDataLv0
SerialNumber      : {2740b8eb-4b25-4010-99f8-497b...}
HealthStatus      : Healthy
OperationalStatus : Offline
TotalSize         : 929 GB
PartitionStyle    : RAW
```

Format it:
```pwsh
PS C:\Users\Administrator> Get-Disk 2 | Initialize-Disk -Passthru | New-Partition -AssignDriveLetter -UseMaximumSize | Format-Volume -Filesystem ReFS
```

## Rename the disk (or modify other properties):
[MS Learn - Set-VirtualDisk](https://learn.microsoft.com/en-us/powershell/module/storage/set-virtualdisk?view=windowsserver2022-ps)

```pwsh
PS C:\Users\Administrator> get-virtualdisk | set-virtualdisk -newfriendlyname SataSsdVmDataLv0
```
