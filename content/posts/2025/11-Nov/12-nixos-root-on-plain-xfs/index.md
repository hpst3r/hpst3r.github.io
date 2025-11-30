---
title: "NixOS root on (plain) XFS"
date: 2025-11-30T11:29:00-00:00
draft: false
---

NixOS 25.05 6.12.55 [8b9e30f](https://github.com/hpst3r/nixos/tree/8b9e30ffb471c1001f5739c8fd05a9bd99cfd21b) on amd64 (host "xps9380")

Example of lazy single `/` partition setup. You could run a smaller ESP that is not also `/boot` and lump `/boot` into `/` if desired, but I like to be consistent.

Disk layout:

```txt
SSD #0
│─ boot 1G vfat (ESP, kernel)
│─ swap 1G 17G (16G) swapdevice
└─ root 17G 100% XFS
```

```shell
sudo -i
# partition disk
parted /dev/nvme0n1 -- mklabel gpt
parted /dev/nvme0n1 -- mkpart ESP fat32 1MB 1G
parted /dev/nvme0n1 -- set 1 esp on
parted /dev/nvme0n1 -- mkpart swap linux-swap 1G 17G
parted /dev/nvme0n1 -- mkpart root 17G 100%
# helper vars for easier copypaste
boot="/dev/disk/by-id/nvme-Samsung_SSD_960_EVO_250GB-part1"
swap="/dev/disk/by-id/nvme-Samsung_SSD_960_EVO_250GB-part2"
root="/dev/disk/by-id/nvme-Samsung_SSD_960_EVO_250GB-part3"
# enable swap
mkswap -L swap "$swap"
swapon "$swap"
# format root part
mkfs.xfs "$root"
mkdir /mnt
mount "$root" /mnt
# format boot part
mkfs.fat -F 32 -n boot "$boot"
mkdir /mnt/boot
mount "$boot" /mnt/boot
# gen config, if needed
nixos-generate-config --root /mnt
# if not needed, now is a great time to clone/copy over your config
nixos-install --root /mnt
# set the root password when prompted, then reboot to your new system
```

Relevant bits of hardware configuration:

```nix
fileSystems."/" = {
  device = "/dev/disk/by-id/nvme-Samsung_SSD_960_EVO_250GB_S3ESNX0K366033D-part3";
  fsType = "xfs";
};

fileSystems."/boot" = {
  device = "/dev/disk/by-id/nvme-Samsung_SSD_960_EVO_250GB_S3ESNX0K366033D-part1";
  fsType = "vfat";
  options = [ "fmask=0022" "dmask=0022" "umask=0077" ];
};

swapDevices = [
  { device = "/dev/disk/by-id/nvme-Samsung_SSD_960_EVO_250GB_S3ESNX0K366033D-part2"; }
];
```
