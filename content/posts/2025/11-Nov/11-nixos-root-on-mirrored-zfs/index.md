---
title: "NixOS root on (mirrored) ZFS"
date: 2025-11-30T11:31:00-00:00
draft: false
---

NixOS 25.05 6.12.55 [8b9e30f](https://github.com/hpst3r/nixos/tree/8b9e30ffb471c1001f5739c8fd05a9bd99cfd21b) on amd64 (host "6028a")

Be sure to boot live media with the LTS kernel for ZFS support.

For more detail about partitioning & ZFS params, review [this post, derived from the NixOS wiki, where I go into more depth](https://wporter.org/nixos-root-on-natively-encrypted-zfs).

## Partitioning

Desired layout:

```txt
SSD #0
│─ /boot0 1G vfat
└─ rpool 1G 100% ZFS

SSD #1
│─ /boot1 1G vfat
└─ rpool 1G 100% ZFS
```

Fetch the ZFS package:

```sh
nix-shell -p zfs
```

Clear disks - adjust list to suit (**THIS WILL REMOVE THE PARTITION TABLE FROM ALL YOUR DISKS, MAKING ANY DATA STORED THERE INACCESSIBLE**):

```sh
for disk in /dev/sd{x,y,z}; do
  sgdisk -Z "$disk"
done
```

Partition boot SSDs. In my case these are port 1 & 2 off an internal SATA controller, so they come up as `sda` and `sdb`. Obviously, adjust to suit.

```sh
parted /dev/sda -- mklabel gpt
parted /dev/sda -- mkpart ESP fat32 1MB 1G
parted /dev/sda -- set 1 esp on
parted /dev/sda -- mkpart root 1G 100%

parted /dev/sdb -- mklabel gpt
parted /dev/sdb -- mkpart ESP fat32 1MB 1G
parted /dev/sdb -- set 1 esp on
parted /dev/sdb -- mkpart root 1G 100%
```

Helper variables:

```sh
boot1="/dev/disk/by-id/ata-INTEL_SSDSC2BB480G4_BTWL5223027M480QGN-part1"
boot2="/dev/disk/by-id/ata-INTEL_SSDSC2BB480G4_BTWL52220571480QGN-part1"
root1="/dev/disk/by-id/ata-INTEL_SSDSC2BB480G4_BTWL5223027M480QGN-part2"
root2="/dev/disk/by-id/ata-INTEL_SSDSC2BB480G4_BTWL52220571480QGN-part2"
```

```sh
zpool create \
  -O compression=zstd \
  -O mountpoint=none \
  -O xattr=sa \
  -O atime=off \
  -O acltype=posixacl \
  -o ashift=12 \
  rpool mirror "$root1" "$root2"
```

Validate ZFS pool config:

```txt
[nix-shell:~]# zfs list
NAME    USED  AVAIL  REFER  MOUNTPOINT
rpool   432K   430G    96K  none

[nix-shell:~]# zpool status
  pool: rpool
 state: ONLINE
config:

  NAME                                                  STATE     READ WRITE CKSUM
  rpool                                                 ONLINE       0     0     0
    mirror-0                                            ONLINE       0     0     0
      ata-INTEL_SSDSC2BB480G4_BTWL5223027M480QGN-part2  ONLINE       0     0     0
      ata-INTEL_SSDSC2BB480G4_BTWL52220571480QGN-part2  ONLINE       0     0     0

errors: No known data errors
```

Create volumes in, then mount the ZFS pool:

```sh
zfs create rpool/root
zfs create rpool/nix
zfs create rpool/var
zfs create rpool/home

mkdir -p /mnt/

mount -t zfs rpool/root /mnt -o zfsutil

mkdir -p /mnt/{nix,var,home,boot1,boot2}

mount -t zfs rpool/nix /mnt/nix -o zfsutil
mount -t zfs rpool/var /mnt/var -o zfsutil
mount -t zfs rpool/home /mnt/home -o zfsutil
```

Format & mount boot partitions:

```sh
mkfs.fat -F 32 -n boot "$boot1"
mkfs.fat -F 32 -n boot "$boot2"
#mlabel -i "$boot2" -n - to change partid of boot2 so they're not identical. not needed
mount "$boot1" /mnt/boot1
mount "$boot2" /mnt/boot2
```

Now we're ready to get the system installed! More or less.

## Configuration

First, copy your baseline config over or, if needed, generate it (`nixos-generate-config --root /mnt`).

Then you'll need to make some changes. For more info, reference [this host config in (my nixos repo - 8b9e30f)](https://github.com/hpst3r/nixos/tree/8b9e30ffb471c1001f5739c8fd05a9bd99cfd21b/modules/hosts/6028a). I'll just highlight the important stuff here.

Here's the bootloader configuration that I keep in my per-host config:

```nix
boot.loader.grub = {
  enable = true;
  zfsSupport = true;
  efiSupport = true;
  device = "nodev"; # tells grub to not install to mbr
  mirroredBoots = mkForce [ # override implicit /boot from boot.loader.grub.enable
    { devices = [ "nodev" ]; path = "/boot1"; efiSysMountPoint = "/boot1"; }
    { devices = [ "nodev" ]; path = "/boot2"; efiSysMountPoint = "/boot2"; }
  ];
};

# allow nixos to manage efi boot entries
boot.loader.efi.canTouchEfiVariables = true;

boot.supportedFilesystems = [ "zfs" ];

# prevents "multiple pools with same name" problem during boot
boot.zfs.devNodes = "/dev/disk/by-partuuid";
```

I had to `lib.mkForce` my `mirroredBoots` config, or NixOS would try to write to `/boot1`, `/boot2`, and then fail to write to (implicit) `/boot`.

Setting the `canTouchEfiVariables` setting to `true` means NixOS will manage boot entries in the EFI (your system will see "nixos-boot1" and "nixos-boot2").

And the relevant section of the hardware configuration (filesystem/fstab) for a system with mirrored boot volumes:

```nix
fileSystems."/boot1" = {
  device = "/dev/disk/by-id/ata-INTEL_SSDSC2BB480G4_BTWL5223027M480QGN-part1";
  fsType = "vfat";
  options = [ "fmask=0022" "dmask=0022" "umask=0077" "nofail" ];
};

fileSystems."/boot2" = {
  device = "/dev/disk/by-id/ata-INTEL_SSDSC2BB480G4_BTWL52220571480QGN-part1";
  fsType = "vfat";
  options = [ "fmask=0022" "dmask=0022" "umask=0077" "nofail" ];
};
```

Note the `nofail` option for each `/boot{1,2}`, which allows the system to continue booting if one `/boot` is missing.

## Install the system

Once your configuration is sorted out, install the system:

```sh
nixos-install --root /mnt --flake /mnt/etc/nixos#hostname
```

No major caveats here. One satisfied, reboot into the new system.

If the system doesn't come up, boot the live USB, import the ZFS pool, mount the volumes in the pool to `/mnt`, fix your config, and try to install again.
