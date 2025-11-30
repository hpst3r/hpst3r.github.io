---
title: "NixOS root on (natively encrypted) ZFS"
date: 2025-11-30T11:30:00-00:00
draft: false
---

NixOS 25.05 6.12.55 [8b9e30f](https://github.com/hpst3r/nixos/tree/8b9e30ffb471c1001f5739c8fd05a9bd99cfd21b) on amd64 (hosts = [ "p1g4" "t14g2a" ])

This is derived from the content on the NixOS wiki, with a few additions for personal ease-of-use. See [wiki/ZFS](https://wiki.nixos.org/wiki/ZFS).

I'll be installing NixOS 25.05 on ZFS with native encryption (passkey at boot).

I've downloaded a standard 25.05 live image from nixos.org and booted the LTS kernel (if you do not do this, you won't have ZFS support! you can probably manually build a live image with a newer kernel and ZFS, but the live:latest images don't have ZFS!)

Figure out what drive you're going to be installing to:

```txt
$ lsblk -e7
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 119.2G  0 disk
├─sda1        8:1    0 119.2G  0 part
│ ├─ventoy  254:0    0   3.8G  1 dm   /iso
│ └─sda1    254:1    0 119.2G  0 dm
└─sda2        8:2    0    32M  0 part
nvme0n1     259:0    0   1.8T  0 disk
├─nvme0n1p1 259:1    0   260M  0 part
├─nvme0n1p2 259:2    0    16M  0 part
└─nvme0n1p3 259:3    0   1.8T  0 part
```

In my case, my /dev/sda is a Ventoy disk I'm booted from. /dev/nvme0n1 is my internal SSD.

We're going to be doing a lot of privileged stuff, and you're on a live image anyway, so might as well elevate to root here. Assume everything else is privileged.

Clear existing disk label on your drive (this will erase record of ALL partitions):

```sh
parted /dev/nvme0n1 -- mklabel gpt
```

Create a boot (and /boot/efi) partition. Designate it as an ESP (by part number):

```sh
parted /dev/nvme0n1 -- mkpart ESP fat32 1MB 1G
parted /dev/nvme0n1 -- set 1 esp on
```

Create a swap partition. I'll do 64G, since my machine has 64G of RAM and I've got plenty of disk.

```sh
parted /dev/nvme0n1 -- mkpart swap linux-swap 1G 65G
```

Create your root partition. I'll use all the space from 65G to the end of the disk. Note that I've not labelled its type; this will probably default to ext2.

```sh
parted /dev/nvme0n1 -- mkpart root 65G 100%
```

Check fdisk or parted when done.

```txt
Model: Samsung SSD 990 EVO 2TB (nvme)
Disk /dev/nvme0n1: 2000GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  1000MB  999MB   fat32        ESP   boot, esp
 2      1000MB  65.0GB  64.0GB               swap  swap
 3      65.0GB  2000GB  1935GB               root



[root@nixos:~]# fdisk -l /dev/nvme0n1
Disk /dev/nvme0n1: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
Disk model: Samsung SSD 990 EVO 2TB
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 5A468CF5-B446-485F-B980-B8A32220D779

Device             Start        End    Sectors  Size Type
/dev/nvme0n1p1      2048    1953791    1951744  953M EFI System
/dev/nvme0n1p2   1953792  126953471  124999680 59.6G Linux swap
/dev/nvme0n1p3 126953472 3907028991 3780075520  1.8T Linux filesystem

```

OK! Time to create our pool. Since the 25.05 graphical installer doesn't include the `zfs` package by default, we'll need to start a `nix-shell` to get it:

```sh
nix-shell -p zfs
```

I'm going to set things up with /dev/disk/by-id labels so stuff doesn't change and break my system.

My partitions are:

```sh
boot_part="/dev/disk/by-id/nvme-Samsung_SSD_990_EVO_2TB_S7M4NL0Y206745K-part1"
swap_part="/dev/disk/by-id/nvme-Samsung_SSD_990_EVO_2TB_S7M4NL0Y206745K-part2"
root_part="/dev/disk/by-id/nvme-Samsung_SSD_990_EVO_2TB_S7M4NL0Y206745K-part3"
```

A few things to note:

- ashift=12 means 4096 bit block size. 12 = 2^12. If you have a disk w/ 512-bit block size (e.g., an old drive), set ashift=9 (2^9 = 512). Setting this to something too low = you will suffer.
- xattr=sa means set extended attr in the inodes, rather than a shitload of tiny files.
- compression=lz4, don't turn it off, lz4 is good and very fast.
- atime=off, saves you some IOPS because it saves you from bumping the accessed time attribute on files
- recordsize varies by workload. Since this will just be an OS disk you can probably get away with the default, 128K. If the disk were being used for large files (sequential reads) set recordsize larger, perhaps ~1M, to reduce IO load (thus better performance reading back) and improve compression.
  - HOWEVER, if your recordsize is TOO small, compression will suffer.
  - performance is highly workload-dependent; if you want to go crazy optimizing, you're going to wind up with extra zpools.
- SLOG is a special drive used to cache writes before pushing them to your primary storage. Don't have one here.
- L2ARC is a device for additional read cache. Uses a lot of memory to cache. Make sure it's worth losing primary ARC space for this. Don't have one here.

So let's create our pool!

Note that it is *not* possible to easily shrink a zpool.

```sh
zpool create \
    -O encryption=on \
    -O keyformat=passphrase \
    -O keylocation=prompt \
    -O compression=zstd \
    -O mountpoint=none \
    -O xattr=sa \
    -O atime=off \
    -O acltype=posixacl \
    -o ashift=12 \
    zpool "$root_part"
```

```sh
zfs create zpool/root
zfs create zpool/nix
zfs create zpool/var
zfs create zpool/home

mkdir -p /mnt/

mount -t zfs zpool/root /mnt -o zfsutil

mkdir -p /mnt/{nix,var,home,boot}

mount -t zfs zpool/nix /mnt/nix -o zfsutil
mount -t zfs zpool/var /mnt/var -o zfsutil
mount -t zfs zpool/home /mnt/home -o zfsutil

mkfs.fat -F 32 -n boot $boot_part
mount $boot_part /mnt/boot
```

```sh
mkswap -L swap "$swap_part"
swapon "$swap_part"
```

Now that we've got our filesystems configured, we're ready for the bones of our system.

**If needed** (e.g., you don't already have a configuration or hardware configuration), use the `nixos-generate-config` utility to run a hardware scan (generate `hardware-configuration.nix` for your machine) and seed a basic `configuration.nix`:

```sh
nixos-generate-config --root /mnt
```

**If you're cloning your configuration (e.g., future me) to a new/existing machine**, now is when you'd want to `git clone https://github/user/nixos` to `/mnt/etc/nixos`.

Since we're using ZFS, you'll need to edit the new system's (`/mnt/etc/nixos/`) `hardware-configuration.nix` to specify `options = [ "zfsutil" ];` for your zvols, and, optionally, set swap by-id and enable randomEncryption (randomEncryption wipes UUIDs at boot, so you need to change to by-id).

As an example, here's the hardware-configuration I'm using for a similar host (AMD machine with a different Intel NVME):

```nix
{ config, lib, pkgs, modulesPath, ... }:

{
  imports =
    [
      (modulesPath + "/installer/scan/not-detected.nix")
    ];

  boot.initrd.availableKernelModules = [ "nvme" "xhci_pci_renesas" "xhci_pci" "usb_storage" "uas" "sd_mod" "rtsx_pci_sdmmc" ];
  boot.initrd.kernelModules = [ ];
  boot.kernelModules = [ "kvm-amd" ];
  boot.extraModulePackages = [ ];

  fileSystems."/" = {
    device = "zpool/root";
    fsType = "zfs";
    options = [ "zfsutil" ];
  };

  fileSystems."/nix" = {
    device = "zpool/nix";
    fsType = "zfs";
    options = [ "zfsutil" ];
  };

  fileSystems."/var" = {
    device = "zpool/var";
    fsType = "zfs";
    options = [ "zfsutil" ];
  };

  fileSystems."/home" = {
    device = "zpool/home";
    fsType = "zfs";
    options = [ "zfsutil" ];
  };

  fileSystems."/boot" = {
    device = "/dev/disk/by-id/nvme-INTEL_SSDPEKNU010TZ_PHKA1474028J1P0B_1-part1";
    fsType = "vfat";
    options = [ "fmask=0022" "dmask=0022" ];
  };

  swapDevices = [{
    device = "/dev/disk/by-id/nvme-INTEL_SSDPEKNU010TZ_PHKA1474028J1P0B_1-part2";
    randomEncryption = true;
  }];

  # Enables DHCP on each ethernet and wireless interface. In case of scripted networking
  # (the default) this is the recommended approach. When using systemd-networkd it's
  # still possible to use this option, but it's recommended to use it in conjunction
  # with explicit per-interface declarations with `networking.interfaces.<interface>.useDHCP`.
  networking.useDHCP = lib.mkDefault true;
  # networking.interfaces.enp2s0f0.useDHCP = lib.mkDefault true;
  # networking.interfaces.enp5s0.useDHCP = lib.mkDefault true;
  # networking.interfaces.wlp3s0.useDHCP = lib.mkDefault true;

  nixpkgs.hostPlatform = lib.mkDefault "x86_64-linux";
  hardware.cpu.amd.updateMicrocode = lib.mkDefault config.hardware.enableRedistributableFirmware;
}
```

Once any host-specific hardware configuration is sorted, it'd be a good time to copy over your config.

Note that you'll need to set the `networking.hostId` in your new system's configuration. This is a random hex value. Should be unique between your machines but isn't significant if you don't have file shares. Your build will fail if it's not present as ZFS depends on it. Generate something with:

```sh
head -c 4 /dev/random | od -A none -t x4
```

(grab 4 characters from `/dev/random` then format them as a 4-byte hex unit with `od(1)`)

Configuration example:

```nix
{ config, ... }: {
  networking.hostId = "d98fe659";
}
```

Finally, *once your config is present* (if you reboot with the minimal config you'll be met with a plain tty), install the system.

If you're using a plain `configuration.nix` (e.g., the generated config):

```sh
nixos-install --root /mnt
```

If you're using flakes:

```sh
nixos-install --root /mnt --flake /mnt/etc/nixos#host
```

Remember to set the root password when prompted. Log in as `root` (e.g., on tty2) and set your user's password once you've rebooted into the new system.
