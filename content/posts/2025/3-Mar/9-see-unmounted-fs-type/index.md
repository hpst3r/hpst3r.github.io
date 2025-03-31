---
title: "Looking at filesystem types on disks from Linux"
date: 2025-03-30T12:13:59-00:00
draft: false
---

Not sure why I went through so many ways to do this. But I did. Was slightly curious about how I might go about this.. guess I'm happy now.

## Unmounted

### parted

`parted /device print` will give you a nice list of partitions, name, and filesystem.

```txt
[liam@800g4m0 ~]$ sudo parted /dev/sda print
Model: ATA WDC WDS100T2B0A (scsi)
Disk /dev/sda: 1000GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                          Flags
 1      1049kB  274MB   273MB   fat32        EFI system partition          boot, esp
 2      274MB   290MB   16.8MB               Microsoft reserved partition  msftres
 3      290MB   1000GB  999GB   ntfs         Basic data partition          msftdata
 4      1000GB  1000GB  631MB   ntfs                                       hidden, diag

```

You can also `parted /device print all` to get `parted` to look at every disk.

```txt
[liam@800g4m0 ~]$ sudo parted /dev/sda print all
Model: ATA WDC WDS100T2B0A (scsi)
Disk /dev/sda: 1000GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                          Flags
 1      1049kB  274MB   273MB   fat32        EFI system partition          boot, esp
 2      274MB   290MB   16.8MB               Microsoft reserved partition  msftres
 3      290MB   1000GB  999GB   ntfs         Basic data partition          msftdata
 4      1000GB  1000GB  631MB   ntfs                                       hidden, diag


Model: INTEL SSDPEKNU010TZ (nvme)
Disk /dev/nvme0n1: 1024GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  68.2MB  67.1MB  fat16        EFI System Partition  boot, esp
 2      68.2MB  5437MB  5369MB  xfs
 3      5437MB  143GB   137GB                                      lvm
```

### file

`file --special-files --dereference` (`-sL`) will give you information on a partition/LV's filesystem or the MBR on a disk. Can be used on mounted or unmounted partitions.

Here's what this looks like with.. something.. left behind by a W11 24H2 install:

```txt
[liam@800g4m0 ~]$ sudo file -sL /dev/sda
/dev/sda: DOS/MBR boot sector MS-MBR Windows 7 english at offset 0x163 "Invalid partition table" at offset 0x17b "Error loading operating system" at offset 0x19a "Missing operating system"; partition 1 : ID=0xee, start-CHS (0x0,0,2), end-CHS (0x30,254,63), startsector 1, 4294967295 sectors
```

Here's what this looks like with a protective MBR (note the single partition with type 0xee):

```txt
[liam@800g4m0 ~]$ sudo file -sL /dev/nvme0n1
/dev/nvme0n1: DOS/MBR boot sector; partition 1 : ID=0xee, start-CHS (0x0,0,2), end-CHS (0x3ff,255,63), startsector 1, 2000409263 sectors, extended partition table (last)
```

Here are three types of partitions:

1. an EFI boot partition (vfat)
2. an XFS partition (/boot)
3. a LVM partition (PV)

```txt
[liam@800g4m0 ~]$ sudo file -sL /dev/nvme0n1p1
/dev/nvme0n1p1: DOS/MBR boot sector, code offset 0x3c+2, OEM-ID "mkfs.fat", sectors/cluster 4, reserved sectors 4, root entries 512, Media descriptor 0xf8, sectors/FAT 128, sectors/track 32, heads 64, hidden sectors 2048, sectors 131072 (volumes > 32 MB), reserved 0x1, serial number 0x14409030, unlabeled, FAT (16 bit)

[liam@800g4m0 ~]$ sudo file -sL /dev/nvme0n1p2
/dev/nvme0n1p2: SGI XFS filesystem data (blksz 4096, inosz 512, v2 dirs)

[liam@800g4m0 ~]$ sudo file -sL /dev/nvme0n1p3
/dev/nvme0n1p3: LVM2 PV (Linux Logical Volume Manager), UUID: 4j6gw3-qqmW-kUme-ggMY-gknT-AOEs-8luOM9, size: 137447342080
```

Here's what a LV looks like:

```txt
[liam@800g4m0 ~]$ sudo file -sL /dev/800g4/var
/dev/800g4/var: Linux rev 1.0 ext4 filesystem data, UUID=e2dc8883-6158-46c7-870d-74bff4aedffd (needs journal recovery) (extents) (64bit) (large files) (huge files)
```

Finally, here's what a Windows install looks like.

1. `sda1` is an EFI partition
2. `sda2` is a MSR partition
3. `sda3` is a NTFS data partition
4. `sda4` is a Windows Recovery Environment partition

```txt
[liam@800g4m0 ~]$ sudo file -sL /dev/sda1
/dev/sda1: DOS/MBR boot sector, code offset 0x58+2, OEM-ID "MSDOS5.0", sectors/cluster 8, reserved sectors 7166, Media descriptor 0xf8, sectors/track 63, heads 255, hidden sectors 2048, sectors 532480 (volumes > 32 MB), FAT (32 bit), sectors/FAT 513, reserved 0x1, serial number 0x72eb3a0d, unlabeled

[liam@800g4m0 ~]$ sudo file -sL /dev/sda2
/dev/sda2: data

[liam@800g4m0 ~]$ sudo file -sL /dev/sda3
/dev/sda3: DOS/MBR boot sector, code offset 0x52+2, OEM-ID "NTFS    ", sectors/cluster 8, Media descriptor 0xf8, sectors/track 63, heads 255, hidden sectors 567296, dos < 4.0 BootSector (0x80), FAT (1Y bit by descriptor); NTFS, sectors/track 63, sectors 1951721471, $MFT start cluster 786432, $MFTMirror start cluster 2, bytes/RecordSegment 2^(-1*246), clusters/index block 1, serial number 0d040ed3d40ed2ac4; contains bootstrap BOOTMGR

[liam@800g4m0 ~]$ sudo file -sL /dev/sda4
/dev/sda4: DOS/MBR boot sector, code offset 0x52+2, OEM-ID "NTFS    ", sectors/cluster 8, Media descriptor 0xf8, sectors/track 63, heads 255, hidden sectors 1952288768, dos < 4.0 BootSector (0x80), FAT (1Y bit by descriptor); NTFS, sectors/track 63, sectors 1232895, $MFT start cluster 51370, $MFTMirror start cluster 2, bytes/RecordSegment 2^(-1*246), clusters/index block 1, serial number 0a6e0c069e0c04173; contains bootstrap BOOTMGR
```

### blkid

Alternatively, you could look at the 'type' fields in the output of `blkid`.

```txt
[liam@800g4m0 ~]$ sudo blkid
/dev/mapper/800g4-var: UUID="e2dc8883-6158-46c7-870d-74bff4aedffd" TYPE="ext4"
/dev/nvme0n1p3: UUID="4j6gw3-qqmW-kUme-ggMY-gknT-AOEs-8luOM9" TYPE="LVM2_member" PARTUUID="967c90be-76ab-4da1-88b4-c209e6eb84d6"
/dev/nvme0n1p1: SEC_TYPE="msdos" UUID="1440-9030" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="6186fdfa-530a-4f6c-b0a7-bbc238b03f2e"
/dev/nvme0n1p2: UUID="71d3600d-5a34-456a-bea1-01d776726d2b" TYPE="xfs" PARTUUID="8cfb4928-91ab-4795-b334-a776a147916a"
/dev/mapper/800g4-root: UUID="38484eee-b428-4b99-9629-53d16b2fb5bb" TYPE="ext4"
/dev/sda4: UUID="A6E0C069E0C04173" TYPE="ntfs" PARTUUID="2a61485a-e223-41d2-8bfa-514bbf60d684"
/dev/sda3: UUID="D040ED3D40ED2AC4" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="035dab53-9703-4196-a1e1-d1dbc2c15478"
/dev/sda1: UUID="72EB-3A0D" TYPE="vfat" PARTLABEL="EFI system partition" PARTUUID="91c3fe2c-5640-41ad-9a35-cfc4525baa39"
/dev/sda2: PARTLABEL="Microsoft reserved partition" PARTUUID="265af130-5fe5-4bdc-a673-c90609c29bb3"
```

It'll see most things without elevation but misses the MSR partition at `/dev/sda2` here when run without root. I'm sure there's a reason for this.. maybe I'll dig later.

```txt
[liam@800g4m0 ~]$ blkid
/dev/mapper/800g4-var: UUID="e2dc8883-6158-46c7-870d-74bff4aedffd" TYPE="ext4"
/dev/nvme0n1p3: UUID="4j6gw3-qqmW-kUme-ggMY-gknT-AOEs-8luOM9" TYPE="LVM2_member" PARTUUID="967c90be-76ab-4da1-88b4-c209e6eb84d6"
/dev/nvme0n1p1: SEC_TYPE="msdos" UUID="1440-9030" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="6186fdfa-530a-4f6c-b0a7-bbc238b03f2e"
/dev/nvme0n1p2: UUID="71d3600d-5a34-456a-bea1-01d776726d2b" TYPE="xfs" PARTUUID="8cfb4928-91ab-4795-b334-a776a147916a"
/dev/mapper/800g4-root: UUID="38484eee-b428-4b99-9629-53d16b2fb5bb" TYPE="ext4"
/dev/sda4: UUID="A6E0C069E0C04173" TYPE="ntfs" PARTUUID="2a61485a-e223-41d2-8bfa-514bbf60d684"
/dev/sda3: UUID="D040ED3D40ED2AC4" TYPE="ntfs" PARTLABEL="Basic data partition" PARTUUID="035dab53-9703-4196-a1e1-d1dbc2c15478"
/dev/sda1: UUID="72EB-3A0D" TYPE="vfat" PARTLABEL="EFI system partition" PARTUUID="91c3fe2c-5640-41ad-9a35-cfc4525baa39"
```

## Mounted only

### df

`df -T` will show you the type of mounted filesystems. Combine this with `-h` for a nicely formatted (size-wise) output.

```txt
[liam@800g4m0 ~]$ df -T
Filesystem             Type     1K-blocks    Used Available Use% Mounted on
devtmpfs               devtmpfs      4096       0      4096   0% /dev
tmpfs                  tmpfs      7940048       0   7940048   0% /dev/shm
tmpfs                  tmpfs      3176020    9076   3166944   1% /run
efivarfs               efivarfs       150      76        70  53% /sys/firmware/efi/efivars
/dev/mapper/800g4-root ext4      65478188 1370608  60735756   3% /
/dev/mapper/800g4-var  ext4      65478188  149748  61956616   1% /var
/dev/nvme0n1p2         xfs        5177344  357452   4819892   7% /boot
/dev/nvme0n1p1         vfat         65390    7198     58192  12% /boot/efi
tmpfs                  tmpfs      1588008       0   1588008   0% /run/user/1000
```

### fsck

`fsck -N` will show you the filesystems on mounted disks. Not sure if it would be my first choice to do this.. but it can be used for it.

```txt
[liam@800g4m0 ~]$ fsck -N
fsck from util-linux 2.37.4
[/usr/sbin/fsck.ext4 (1) -- /] fsck.ext4 /dev/mapper/800g4-root
[/usr/sbin/fsck.vfat (1) -- /boot/efi] fsck.vfat /dev/nvme0n1p1
[/usr/sbin/fsck.ext4 (1) -- /var] fsck.ext4 /dev/mapper/800g4-var
```

## Honorary mention

### fdisk

Probably the first thing to come to mind. Good old `fdisk`. Doesn't quite do the trick - you can't see what filesystem is on a disk at first glance, just the partition type value.

Quick peruse of the manpage and Google leads me to believe this will not look at filesystem metadata at all.

```txt
[liam@800g4m0 ~]$ sudo fdisk -l
Disk /dev/nvme0n1: 953.87 GiB, 1024209543168 bytes, 2000409264 sectors
Disk model: INTEL SSDPEKNU010TZ
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 5A164C26-DF9A-41B7-8411-FB5F0F40AB17

Device            Start       End   Sectors  Size Type
/dev/nvme0n1p1     2048    133119    131072   64M EFI System
/dev/nvme0n1p2   133120  10618879  10485760    5G Linux filesystem
/dev/nvme0n1p3 10618880 279070719 268451840  128G Linux LVM


Disk /dev/sda: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: WDC  WDS100T2B0A
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 794F080D-0A9B-432C-9BD8-2EA0AE01103A

Device          Start        End    Sectors   Size Type
/dev/sda1        2048     534527     532480   260M EFI System
/dev/sda2      534528     567295      32768    16M Microsoft reserved
/dev/sda3      567296 1952288767 1951721472 930.7G Microsoft basic data
/dev/sda4  1952288768 1953521663    1232896   602M Windows recovery environment


Disk /dev/mapper/800g4-root: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/800g4-var: 64 GiB, 68719476736 bytes, 134217728 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```