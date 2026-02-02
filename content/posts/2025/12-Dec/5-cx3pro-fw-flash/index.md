---
title: "Flash Mellanox ConnectX-3 Pro firmware (CX314A-BCCT to CX354A-FCCT)"
date: 2026-01-31T19:00:00-00:00
draft: false
---

## Introduction

The ConnectX-3 Pro (MT27520) was sold in a few different configurations (e.g., dual SFP+, QSFP+, dual QSFP+, Ethernet only, Infiniband and Ethernet). However, all the ASICs are the same, so you can easily cross-flash them (It's possible to flash VPI firmware to an Ethernet card if you want to unlock Infiniband.)

You can also typically flash OEM cards to stock Mellanox firmware with little hassle.

The most recent batch I purchased are CX314 Ethernet-only cards (about $10 each) so I figure I'll crossflash them to the VPI cards before pressing them into service as IB HCAs (for that sweet, sweet RDMA).

If you're not sure what kind of NIC you have, you can try to set a port to Infiniband mode. If you try to set a port to Infiniband mode on an Ethernet-only firmware revision:

```txt
root@3040a:~# echo "ib" > /sys/bus/pci/devices/0000:01:00.0/mlx4_port1
-bash: echo: write error: Operation not supported
```

Standard disclaimer - you should probably not attempt to crossflash a card if you do not have the means to replace it or the time to attempt recovery (they're $10 so whatever). This may or may not kill your card if it gets mucked up somehow. Buy an extra HCA.

## Finding PCI addresses

Before we get started, you'll need to find the PCI address for your card. The address is the first portion of a `lspci` output line (e.g., `01:00.0` below) and corresponds to the address of the device on the PCI bus.

```txt
root@3040a:~# lspci | grep -i mell
01:00.0 Ethernet controller: Mellanox Technologies MT27520 Family [ConnectX-3 Pro]
```

This address may change depending on which other devices are in the system. In the sysfs, PCI addresses are prefixed with `0000:` (e.g., `0000:01:00.0`).

## Other resources

Useful threads:

- [ServeTheHome - flashing firmware](https://forums.servethehome.com/index.php?threads/flashing-stock-mellanox-firmware-to-oem-emc-connectx-3-ib-ethernet-dual-port-qsfp-adapter.20525/#post-198015)
- [Flashing Dell C6600 mezzanine CX314s to CX354s](https://forums.servethehome.com/index.php?threads/connectx-3-pro-mcx314a-mcx384a-bcaa-cards-flashed-to-mcx354a-pro-infiniband-vpi-firmware-dont-connect-to-other-infiniband-cards-or-switches.36003/)
- [XtremeOwnage blog - configuring ConnectX 3 and 4](https://static.xtremeownage.com/blog/2025/mellanox-configuration-guide/)
- If you have a card with secure firmware (e.g., Dell, HP OEM mezzanine versions), you may need to flash it over I2C. [See this ServeTheHome thread.](https://forums.servethehome.com/index.php?threads/changing-psid-is-unsupported-under-secure-fw.36493/)

ConnectX-3 Pro (IB, HCA) firmware is accessible from [network.nvidia.com (Mellanox firmware)](https://network.nvidia.com/support/firmware/connectx3proib/) or [network.nvidia.com (OEM firmware)](https://network.nvidia.com/support/oem-firmware-downloads/). The latest firmware revisions for ConnectX NIC/HCA that I've needed firmware for can still be found on Nvidia's site.

## On support, cards this applies to

Unfortunately, the version of mstflint provided by most distributions (Debian, NixOS, etc) no longer supports the ConnectX-3 Pro family (though they are still perfectly good cards!)

Models that this writeup directly applies to:

- ConnectX-3 Pro CX314A
  - Dual port QSFP, Ethernet only
- ConnectX-3 Pro CX313A
  - Single port QSFP, Ethernet only
- ConnectX-3 Pro CX354A
  - Dual port QSFP, VPI (Ethernet and Infiniband)
- ConnectX-3 Pro CX353A
  - Single port QSFP, VPI (Ethernet and Infiniband)

You can likely skip the next section if you're using a CX4 or newer (current mstflint 4.31 still supports the CX4-LX and CX4). The 'flashing firmware' part is still applicable.

If you're not sure whether your distribution provides a compatible version of mstflint, try to use it - you'll receive a segfault when you try to do anything with your cards:

```txt
# where address 03:00.0 is a ConnectX-3/Pro..:
root@z2g4c:~# mstflint -d 03:00.0 q
Segmentation fault
root@z2g4c:~# mstflint -v
mstflint, mstflint 4.31.0, Git SHA Hash: e94e92f
```

## Building older mstflint

You will need [a release of mstflint prior to 4.26](https://github.com/Mellanox/mstflint/issues/1157#issuecomment-2695402138) if you're working with a CX3 Pro.

I had some trouble building 4.25 (theoretically the latest with support) on PVE 9, but I tried 4.22 and that works fine. 4.21 is provided by oldstable (Debian 12) if you would prefer to go that route. If on NixOS, [see misumisumi's flake for 4.25 on GitHub](https://github.com/misumisumi/flakes/pull/156/changes/4b9d28d7d46559e2619b82f1f89f4fe3248b302e).

My manual compile will be on a Proxmox VE 9.1.4 host (freshly installed, for consistency).

To compile and install the package manually:

```txt
# Download & extract mstflint 4.22 source
wget https://github.com/Mellanox/mstflint/releases/download/v4.22.0-1/mstflint-4.22.0-1.tar.gz
tar -xzf mstflint-4.22.0-1.tar.gz
cd mstflint-4.22.0

# build dependencies
apt install -y build-essential libssl-dev zlib1g-dev libibverbs-dev libibmad-dev

# configure with OpenSSL support (required for firmware signing)
./configure --enable-openssl
make -j$(nproc)
make install
```

Then reload your PATH and confirm that mstflint can query your card:

```txt
root@3040a:~# lspci | grep -i mell
01:00.0 Ethernet controller: Mellanox Technologies MT27520 Family [ConnectX-3 Pro]
root@3040a:~# mstflint -d 01:00.0 q
Image type:            FS2
FW Version:            2.40.5032
FW Release Date:       16.1.2017
Product Version:       02.40.50.32
Rom Info:              type=PXE version=3.4.747
Device ID:             4103
Description:           Node             Port1            Port2            Sys image
GUIDs:                 ffffffffffffffff ffffffffffffffff ffffffffffffffff ffffffffffffffff
MACs:                                       e41d2d1f28c0     e41d2d1f28c1
VSD:
PSID:                  MT_1090111023
```

## Flashing firmware

The most recent firmware for the CX3 Pro CX354/353/314/313 can be found at [network.nvidia.com (Mellanox firmware)](https://network.nvidia.com/support/firmware/connectx3proib/) (same link as in the intro).

**The single-port 353 and 313 firmware WILL flash to a dual-port card and at best disable a port. Use the appropriate firmware version for your card.**

I'll be flashing the dual-port VPI firmware to an Ethernet-only CX314. Download it to a machine with `mstflint` installed with `wget`:

```sh
wget https://www.mellanox.com/downloads/firmware/fw-ConnectX3Pro-rel-2_42_5000-MCX354A-FCC_Ax-FlexBoot-3.4.752.bin.zip

# then extract the archive - you may need to install `unzip`
unzip fw-ConnectX3Pro-rel-2_42_5000-MCX354A-FCC_Ax-FlexBoot-3.4.752.bin.zip
```

Dump the original firmware for a backup. Worst case, you can put the card in recovery mode and reflash it (haven't had to do this yet, so I won't be covering it - if needed, see [cammckenzie.com](https://www.cammckenzie.com/blog/index.php/2017/05/24/mellanox-technologies-mt27500-connectx-3-flash-recovery-how-to-recover-a-broken-firmware-update/), [ServeTheHome thread](https://forums.servethehome.com/index.php?threads/borked-flash-of-a-mellanox-cx354a-connectx-3-missing-second-port.37878/)).

```sh
mstflint -d 03:00.0 dc orig_firmware_314a_2.40.5032.bin
```

Before - note the PSID (Parameter-Set IDentification - unique identifier for the configuration/SKU) and MACs, in case anything goes wrong.

```txt
root@3040a:~# mstflint -d 01:00.0 q
Image type:            FS2
FW Version:            2.40.5032
FW Release Date:       16.1.2017
Product Version:       02.40.50.32
Rom Info:              type=PXE version=3.4.747
Device ID:             4103
Description:           Node             Port1            Port2            Sys image
GUIDs:                 ffffffffffffffff ffffffffffffffff ffffffffffffffff ffffffffffffffff
MACs:                                       e41d2d1f28c0     e41d2d1f28c1
VSD:
PSID:                  MT_1090111023
```

Then, `burn` the new firmware to the card. **If you are crossflashing the card** include the `-allow_psid_change` argument to tell `mstflint` it's OK to write a new PSID to the card.

```txt
root@3040a:~# mstflint -d 01:00.0 -i fw-ConnectX3Pro-rel-2_42_5000-MCX354A-FCC_Ax-FlexBoot-3.4.752.bin -allow_psid_change burn

    Current FW version on flash:  2.40.5032
    New FW version:               2.42.5000


    You are about to replace current PSID on flash - "MT_1090111023" with a different PSID - "MT_1090111019".
    Note: It is highly recommended not to change the PSID.

 Do you want to continue ? (y/n) [n] : y
Burning FS2 FW image without signatures - OK
Restoring signature                     - OK
```

The card will immediately appear with the new PSID, but you'll need to restart the machine to load the new firmware and use new features (e.g., change port modes), so do that. Note the two `FW Version*` lines below:

```txt
root@3040a:~# mstflint -d 01:00.0 q
Image type:            FS2
FW Version:            2.42.5000
FW Version(Running):   2.40.5032
FW Release Date:       5.9.2017
Product Version:       02.42.50.00
Rom Info:              type=PXE version=3.4.752
Device ID:             4103
Description:           Node             Port1            Port2            Sys image
GUIDs:                 ffffffffffffffff ffffffffffffffff ffffffffffffffff ffffffffffffffff
MACs:                                       e41d2d1f28c0     e41d2d1f28c1
VSD:
PSID:                  MT_1090111019
```

After a reboot, confirm that the firmware revision is updated, and check the port modes. They will likely default to autosensing if you flash the card to VPI firmware.

```txt
root@3040a:~# mstflint -d 01:00.0 q
Image type:            FS2
FW Version:            2.42.5000
FW Release Date:       5.9.2017
Product Version:       02.42.50.00
Rom Info:              type=PXE version=3.4.752
Device ID:             4103
Description:           Node             Port1            Port2            Sys image
GUIDs:                 ffffffffffffffff ffffffffffffffff ffffffffffffffff ffffffffffffffff
MACs:                                       e41d2d1f28c0     e41d2d1f28c1
VSD:
PSID:                  MT_1090111019
```

```txt
root@3040a:~# # replace 0000:01:00.0 with your PCI address..
root@3040a:~# cat /sys/bus/pci/devices/0000:01:00.0/mlx4_port{1,2}
auto (ib)
auto (ib)
```

## Program GUIDs

Infiniband addressing comes in the form of local identifiers (16-bit addresses for intra-subnet traffic, assigned by the subnet manager), global identifiers (GIDs - 128-bit addresses used for routing between IB subnets, think IPv6) and globally unique identifiers (GUIDs - 64-bit hardware addresses, like MAC addresses, used as an input for GIDs). All Infiniband addresses are typically represented in hex. Each Infiniband device must have a unique GUID.

Since my HCAs were originally just Ethernet NICs, they don't have an Infiniband GUID (they're currently just "ff").

This poses a problem - multiple devices in an Infiniband network cannot share GUIDs, and if you have two devices with a shared GUID in a subnet, the subnet manager won't be able to handle them both (like a MAC address). Only one will come up.

So, we'll have to generate GUIDs and flash them to our cards to make them usable on an Infiniband fabric.

You can flash GUIDs with mstflint. Again, you can make up the GUID, but it must be unique in your fabric and is intended to be globally unique.

```txt
root@3040a:~# mstflint -d 03:00.0 -guid 0x0200000000001120 sg
-W- GUIDs are already set, re-burning image with the new GUIDs ...
    You are about to change the Guids/Macs/Uids on the device:

                        New Values              Current Values
        Node  GUID:     0200000000001120        ffffffffffffffff
        Port1 GUID:     0200000000001121        ffffffffffffffff
        Port2 GUID:     0200000000001122        ffffffffffffffff
        Sys.Image GUID: 0200000000001123        ffffffffffffffff
        Port1 MAC:          e41d2d128000            e41d2d128000
        Port2 MAC:          e41d2d128001            e41d2d128001

 Do you want to continue ? (y/n) [n] : y
Burning FS2 FW image without signatures - OK  
Restoring signature                     - OK
```

## Quick aside - power draw

HW: OptiPlex 3040 SFF (2x4gb DDR3L, i5-6500, 120gb DC S3510)

Baseline, no NIC: 12.5w from the wall

Original Ethernet-only firmware (2.40.5032 MT_1090111023), no links up: 17w from the wall. CPU hits C8.

Latest VPI firmware (2.42.5000 MT_1090111019), no links up: 17w from the wall. CPU hits C8.

Don't have the ability to easily put a kill-a-watt on my rack.
