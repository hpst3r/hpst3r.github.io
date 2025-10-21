---
title: "Mellanox SX6036 basics (setup, licensing, noise, power, VPI)"
date: 2025-10-20T22:00:00-00:00
draft: false
---

## Intro

The Mellanox SX6036 is a 36-port 56 Gb/s FDR Infiniband/Ethernet switch. Its forwarding is handled by the Mellanox SwitchX-2 switch chip, and its management plane is a Big Endian PowerPC chip running MLNX-OS (Linux).

The SX6036 is interesting because:

- it supports both Infiniband and Ethernet simultaneously, including Ethernet routing (with a very little bit of effort to enable the required licenses)
- it's around $90 off eBay
- it has a friendly, Cisco-like CLI
- it runs Linux and you can get shell access pretty easily
- perhaps most importantly, because the SwitchX-2 (despite being nearly 15 years old) is rather efficient - 50w at idle with stock fans and no active connections. With DACs, this power figure only slightly increases.

 As for supporting hardware, QSFP 40/50 GBe Mellanox ConnectX-4 LX Ethernet NICs are about $20 all day at the time of writing, and the switch isn't particularly picky about who made the optics or DACs that you plug into it (though Mellanox DACs are quite cheap - I picked up a buttload of 1M and 3M 40/100GBe DACs and some breakouts for about $100).

It also supports active/active L2 failover (MLAG) which is *extremely* rare on things that are cheap, good, and use little power. Though I don't have a way to justify 72 40 GBe ports at the moment, I thought it was worth mentioning MLAG support since it's one of the features I always look for when buying network switches.

Anyway, this all makes the SX6036 a wonderful (affordable, fast, and good) switch for > 1 GBe connections in a homelab. It's so cheap and power efficient that it makes 10 GBe switches look terrible in comparison.

## Inventory

In this post, I'll be using:

- A Mellanox SX6036 (obviously)
  - standard Cisco-style USB to RJ45 serial cable
- Jumpbox/utility server (`m715q-0`) running a web server (serving system images)
  - AlmaLinux 10.0 (not that it matters)
- Juniper EX2200-C (management interface is hooked up to this)
- Another random server, a Coffee Lake HP Z2
  - AlmaLinux 10.0, Nvidia DOCA-OFED (the lightweight MLNX_OFED replacement)
  - CX-4 LX "MCX4131A-GCA_Ax"
- Assorted Mellanox 3M DACs, FS Mellanox-coded 1M and 3M DACs, Mellanox 3M QSFP to SFP breakout DACs

## Updating firmware

My SX6036 arrived running a version of MLNX-OS from 2012. Getting rid of that was obviously the first order of business.

[Here's a direct link to download MLNX-OS 3.6.8012, the final release for the SX1036/6036](https://www.mellanox.com/downloads/Software/image-PPC_M460EX-3.6.8012.img) from Nvidia.

[You can find many older versions of MLNX-OS from HP here](https://support.hpe.com/connect/s/softwaredetails?language=en_US&softwareId=MTX-061c24a638b44bd2826c344823&tab=releaseNotes). You'll need to grab a few of these to do an update from 3.2.x to 3.6.8012. I'll mirror them somewhere eventually.

From start to end, I flashed my switch with:

- 3.2.0330
- 3.3.4100 (image-PPC_M460EX-SX_3.3.4100.img)
- 3.4.0012 (image-PPC_M460EX-SX_3.4.0012.img)
- 3.4.3002 (image-PPC_M460EX-3.4.3002.img)
- 3.5.1006 (image-PPC_M460EX-3.5.1006.img)
- 3.6.1002 (image-PPC_M460EX-3.6.1002.img)
- 3.6.8012 (image-PPC_M460EX-3.6.8012.img)

I've read that you can skip 3.5.x and 3.6.1002 and just go straight to 3.6.8012 from 3.4.x. I read this while waiting for mine to update from 3.5.1006 to 3.6.1002, so I didn't try it.

I plugged a standard Cisco RJ45 console cable into the switch and into a Linux box on my desk, and got the switch on my network.

Here's the `sh inv` and `sh ver` right after getting it booted:

```txt
switch-b7b218 [standalone: master] # sh inventory
================================================================================
================
Module                Type                  Part number           Serial Number

================================================================================
================
CHASSIS               SX6036                MSX6036F-1SFR         MT1248U03520

MGMT                  SX6036                MSX6036F-1SFR         MT1248U03520
FAN                   SXX0XX_FAN            MSX60-FF              MT1247U00857
PS1                   SXX0XX_PS             MSX60-PF              MT1312X01851
PS2                   SXX0XX_PS             MSX60-PF              MT1247U01045
CPU                   CPU                   SA001203              MT1247U01375
switch-b7b218 [standalone: master] # sh ver
Product name:      SX_PPC_M460EX
Product release:   SX_3.2.0330-70
Build ID:          #1-dev
Build date:        2012-12-16 18:14:46
Target arch:       ppc
Target hw:         m460ex
Built by:          doront@fit15

Uptime:            1h 22m 32.880s

Product model:     ppc
Host ID:           0002C9B7B218
System memory:     105 MB used / 1922 MB free / 2027 MB total
Swap:              0 MB used / 0 MB free / 0 MB total
Number of CPUs:    1
CPU load averages: 0.00 / 0.00 / 0.00
```

This subnet doesn't have a DHCP server running right now, so I'll set `mgmt0` up manually - note that you must manually disable DHCP with the `no interface mgmt0 dhcp` command if it's enabled. If you don't disable DHCP and try to set a static IP, it won't be applied to the interface. The first Mellanox quirk I ran into.

```txt
Mellanox MLNX-OS Switch Management

switch-b7b218 login: admin
Password:
Last login: Thu Oct 16 00:47:21 on ttyS0

Mellanox Switch

switch-b7b218 [standalone: unknown] > en
switch-b7b218 [standalone: unknown] # conf t
switch-b7b218 [standalone: unknown] (config) # no interface mgmt0 dhcp
switch-b7b218 [standalone: unknown] (config) # int mgmt0
switch-b7b218 [standalone: unknown] (config interface mgmt0) # ip address 192.168.77.200 255.255.255.0
switch-b7b218 [standalone: unknown] (config interface mgmt0) # exit
switch-b7b218 [standalone: unknown] (config) # ip route 0.0.0.0 0.0.0.0 192.168.77.254
switch-b7b218 [standalone: unknown] (config) # ip name-server 172.27.254.254
switch-b7b218 [standalone: unknown] (config) # exit
switch-b7b218 [standalone: master] # wr mem
switch-b7b218 [standalone: master] # ping 192.168.77.254 -c 1
PING 192.168.77.254 (192.168.77.254) 56(84) bytes of data.
64 bytes from 192.168.77.254: icmp_seq=1 ttl=64 time=0.613 ms

--- 192.168.77.254 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.613/0.613/0.613/0.000 ms
switch-b7b218 [standalone: master] # sh ip route
Destination       Mask              Gateway           Interface   Source
default           0.0.0.0           192.168.77.254    mgmt0       static
192.168.77.0      255.255.255.0     0.0.0.0           mgmt0       interface
switch-b7b218 [standalone: master] # ping wporter.org -c 1
PING wporter.org (185.199.110.153) 56(84) bytes of data.
64 bytes from cdn-185-199-110-153.github.com (185.199.110.153): icmp_seq=1 ttl=56 time=6.76 ms

--- wporter.org ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 6.767/6.767/6.767/0.000 ms
```

Great! It's got connectivity. At this point, you should be able to SSH to the switch with the `admin` user. Let's get to updating.

I've got the files on a Linux box that's serving them over HTTP. I'll use the `image fetch` command on the switch to pull down those images and update the box. I think you can `scp` them from the switch (pull from remote) but HTTP was simpler in this case.

First, download the new image to the switch with the `image fetch` command:

```txt
switch-b7b218 [standalone: master] # image fetch http://192.168.77.254/mlnx/image-PPC_M460EX-SX_3.3.4100.img
 100.0%  [#################################################################]
```

Then, you can list the images on the box with `show images`:

```txt
switch-b7b218 [standalone: master] # sh im
Images available to be installed:
  image-PPC_M460EX-SX_3.3.4100.img
  SX_PPC_M460EX SX_3.3.4100 2013-09-16 18:27:29 ppc

Installed images:
  Partition 1:
  SX_PPC_M460EX SX_3.2.0330-70 2012-12-16 18:14:46 ppc

  Partition 2:
  SX_PPC_M460EX SX_3.2.0330-70 2012-12-16 18:14:46 ppc

Last boot partition: 1
Next boot partition: 1

Boot manager password is set.

No image install currently in progress.

Image signing: trusted signature always required
Admin require signed images: yes

Settings for next boot only:
   Fallback reboot on configuration failure: yes (default)
```

Install the image with the `image install` command. Optionally, specify a `location` (the partition to use) if you have a screwed up image on one of your two boot partitions and would like to overwrite it.

```txt
image install image-PPC_M460EX-SX_3.3.4100.img (location 1)
```

View the installed images again with `sh image`. The system keeps two OS images on flash, and recovering it if both are corrupted doesn't seem to be too difficult.

```txt
switch-b7b218 [standalone: master] (config) # sh image
Images available to be installed:
  image-PPC_M460EX-SX_3.3.4100.img
  SX_PPC_M460EX SX_3.3.4100 2013-09-16 18:27:29 ppc

Installed images:
  Partition 1:
  SX_PPC_M460EX SX_3.2.0330-70 2012-12-16 18:14:46 ppc

  Partition 2:
  SX_PPC_M460EX SX_3.3.4100 2013-09-16 18:27:29 ppc

Last boot partition: 1
Next boot partition: 1

Boot manager password is set.

No image install currently in progress.

Image signing: trusted signature always required
Admin require signed images: yes

Settings for next boot only:
   Fallback reboot on configuration failure: yes (default)
```

Set the switch to boot from the new image with the `image boot next` command:

```txt
switch-b7b218 [standalone: master] (config) # image boot next
```

Write memory and reload:

```txt
switch-b7b218 [standalone: master] (config) # wr mem
switch-b7b218 [standalone: master] (config) # reload
```

Confirm the new image is booted:

```txt
switch-b7b218 [standalone: master] > sh ver
Product name:      SX_PPC_M460EX
Product release:   SX_3.3.4100
Build ID:          #1-dev
Build date:        2013-09-16 18:27:29
```

Delete the image file you just installed:

```txt
switch-b7b218 [standalone: master] # image delete image-PPC_M460EX-SX_3.3.4100.img
```

And continue:

```txt
switch-b7b218 [standalone: master] > en
switch-b7b218 [standalone: master] #
switch-b7b218 [standalone: master] #
switch-b7b218 [standalone: master] # image delete image-PPC_M460EX-SX_3.3.4100.img
switch-b7b218 [standalone: master] # image fetch http://192.168.77.254/mlnx/image-PPC_M460EX-3.4.3002.img
 100.0%  [#################################################################]
switch-b7b218 [standalone: master] # image install image-PPC_M460EX-3.4.3002.img
Step 1 of 4: Verify Image
 100.0%  [#################################################################]
Step 2 of 4: Uncompress Image
 100.0%  [#################################################################]
Step 3 of 4: Create Filesystems
 100.0%  [#################################################################]
Step 4 of 4: Extract Image
   5.5%  [###                                                              ]
```

If you try to jump too far ahead, the image will usually fail to extract with an error. For example, I tried to hop from 3.3.4100 to 3.4.3002 at one point, which didn't work:

```txt
% /bin/tar: Unexpected EOF in archive
/bin/tar: Unexpected EOF in archive
/bin/tar: Error is not recoverable: exiting now
*** Could not extract files from /tmp/mnt_image_wi/tmpfs/unzip/image-PPC_M460EX-3.4.3002.tar : 2
```

All together:

```txt
switch-b7b218 [standalone: master] # image del image-PPC_M460EX-3.4.3002.img
switch-b7b218 [standalone: master] # image fetch http://192.168.77.254/mlnx/image-PPC_M460EX-3.5.1006.img
 100.0%  [#########################################################################################################]
switch-b7b218 [standalone: master] # image install image-PPC_M460EX-3.5.1006.img
Step 1 of 4: Verify Image
 100.0%  [#########################################################################################################]
Step 2 of 4: Uncompress Image
 100.0%  [#########################################################################################################]
Step 3 of 4: Create Filesystems
 100.0%  [#########################################################################################################]
Step 4 of 4: Extract Image
 100.0%  [#########################################################################################################]
switch-b7b218 [standalone: master] # conf t
switch-b7b218 [standalone: master] (config) # image boot next
switch-b7b218 [standalone: master] (config) # wr mem
switch-b7b218 [standalone: master] (config) # reload
```

To shut down the switch, issue the `reload halt` command, then yank the power once it's done shutting down the OS.

```txt
switch-b7b218 [standalone: master] # reload halt
Configuration has been modified; save first? [yes] yes
Configuration changes saved.
Halting system...
switch-b7b218 [standalone: master] #

System shutdown initiated -- logging off.



Mellanox MLNX-OS Switch Management

Stopping pm: [  OK  ]
Shutting down kernel logger: [  OK  ]
Shutting down system logger: [  OK  ]
Starting killall:  [  OK  ]
Sending all processes the TERM signal...
Sending all processes the KILL signal...
Remounting root filesystem in read-write mode:
Saving random seed:
Syncing hardware clock to system time
Running vpart script:
Unmounting file systems:
Remounting root filesystem in read-only mode:
Running vpart script:
Halting system...
Power down.
System Halted, OK to turn off power
```

## Unlocking shell access

The generic `LK2-RESTRICTED_CMDS_GEN2-88A1-NEWD-BPNB-1` license key unocks "access to restricted system functionality", including access to the Bash shell via the `_shell` command. Install it from `configuration` mode with:

```txt
switch-b7b218 [standalone: master] # en
switch-b7b218 [standalone: master] # conf t
switch-b7b218 [standalone: master] (config) # license install LK2-RESTRICTED_CMDS_GEN2-88A1-NEWD-BPNB-1
switch-b7b218 [standalone: master] (config) # exit
switch-b7b218 [standalone: master] # sh licenses
License 1: LK2-RESTRICTED_CMDS_GEN2-88A1-NEWD-BPNB-1
   Feature:          RESTRICTED_CMDS_GEN2
   Description:      Access to restricted system functionality
   Valid:            yes
   Active:           yes
switch-b7b218 [standalone: master] # wr mem
```

Then, you can access the shell on the SX6036 by typing `_shell` from `enable` mode, like so:

```txt
switch-b7b218 [standalone: master] # _shell
[admin@switch-b7b218 ~]# whoami
admin
[admin@switch-b7b218 ~]# uname -a
Linux switch-b7b218 3.10.94-MELLANOXuni-m460ex PPC_M460EX jenkins #1 2019-02-13 12:36:41 ppc GNU/Linux
[admin@switch-b7b218 ~]# which bash
/bin/bash
```

## License generator

Importantly, this allows us to get at the `genlicense` utility - this may be used to unlock other things, like Ethernet. This is actually included in most MLNX-OS images, including the one on your SX6036.

```txt
[admin@switch-b7b218 ~]# which genlicense
/opt/tms/bin/genlicense
```

Running `genlicense` with no args will spit out a help dialog.

```txt
[admin@switch-b7b218 ~]# genlicense
Usage: genlicense <license type ID> <type-specific options ...>

genlicense 1 <feature name> <start date> <end date>
             <tied info id> <tied info str> <secret>

    Feature name:   String of characters not including hyphen '-'
    Start date:     0 for no limit, or date like 2003/10/15
    End date:       0 for no limit, or date like 2003/10/15
    Tied info id:   1 for first MAC address
                    2 for host ID
                    3 for a license not tied to any specific machine
    Tied info str:  For tied info id 1: MAC like 11:33:55:77:99:AA
                    For tied info id 2: Host ID like 8bbf35f82b2c
                    For tied info id 3: ignored (use "-")
    Secret:         Shared secret, quoted or escaped if necessary

genlicense 2 <feature name> <secret> [-h <hash type>] [-u <length>] [-i]
             [-f] [-o <option tag id> <option value>]*

    Feature name:   String of capital letters, numbers, and '_'
    Secret:         Shared secret, quoted or escaped if necessary

    Hash type: defaults to hmac_sha256_48 if not specified.
    Use either the number or the string in parentheses to specify.
       1 (hmac_md5_full): HMAC MD5 with full 128-bit length
       2 (hmac_md5_96): HMAC MD5 truncated to 96 bits
       3 (hmac_md5_48): HMAC MD5 truncated to 48 bits
       5 (hmac_sha256_full): HMAC SHA256 with full 256-bit length
       6 (hmac_sha256_128): HMAC SHA256 truncated to 128 bits
       7 (hmac_sha256_96): HMAC SHA256 truncated to 96 bits
       8 (hmac_sha256_48): HMAC SHA256 truncated to 48 bits

    The '-u' option requests to embed a randomly generated number in the
    license, which serves to make it unique from other licenses with the same
    options.  Useful e.g. if you are using a cumulative informational option.
    The parameter is a length in bits, up to 64.

    The '-i' option requests to add a validator identifier to the license.
    This makes the license five characters longer, but may speed up license
    calculations in some cases.

    The '-f' option forces generation of a license that otherwise failed our
    validation criteria: either (a) it does not meet the current requirements
    for hash length and/or algorithm, or (b) it uses an option tag which is
    not relevant to the particular feature chosen.

    The '-o' option can be used zero or more times to specify options
    for the license.  Each option tag has an id (which encodes the data
    type and meaning of the tag) and a value.  There are two classes of
    option tags, which share the same id namespace.  Specify the id either
    by number, or by the string shown in parentheses after the number:

    Activation option IDs:
        1 (start_date): Start date: license not active before this date
        2 (end_date): End date: license not active after this date
        3 (tied_primary_mac): Tie to this MAC address on primary interface
        4 (tied_host_id): Tie to this host ID
        5 (tied_host_id_hex): Tie to this host ID (lowercase hexadecimal)
       49 (efm_sx_serial_num): Chassis serial number for license verification

    Informational option IDs:
       48 (efm_sx_max_num_hca_ports): Maximum number of HCA ports supported by this MLNX-OS SwitchX license
       50 (efm_sx_active_ports): Active ports number supported by this MLNX-OS SwitchX license
       51 (efm_sx_l2_enabled): Eth L2 enabled by this MLNX-OS SwitchX license
       52 (efm_sx_ib_enabled): IB enabled by this MLNX-OS SwitchX license
       53 (efm_sx_eth_enabled): Eth enabled by this MLNX-OS SwitchX license
       54 (efm_sx_gw_ports): GW ports number supported by this MLNX-OS SwitchX license
       55 (efm_sx_max_ufm_ports): Maximum number of UFM ports supported by this MLNX-OS SwitchX license
       56 (efm_sx_ib_speed_sw_limit): IB port SW speed limit enabled by this MLNX-OS SwitchX license
       57 (efm_sx_eth_speed_sw_limit): Eth port SW speed limit enabled by this MLNX-OS SwitchX license
       58 (efm_sx_l3_enabled): Eth L3 enabled by this MLNX-OS SwitchX license
       59 (efm_sx_fcf_enabled): FCF enabled by this MLNX-OS SwitchX license
       60 (oem_lic_10gbps_enable): 10 Gbps ports licensed by OEM license
       61 (oem_lic_25gbps_enable): 25 Gbps ports licensed by OEM license
       62 (oem_lic_100gbps_enable): 100 Gbps ports licensed by OEM license
```

Note the "informational" option IDs; these are the switch feature toggles you pass to `genlicense`.

1. 48 sets the maximum number of HCA (host channel adapter) ports on the switch. Not entirely sure what this does.
2. 50 sets the maximum number of active ports on the switch.
3. 51 specifies whether L2 Ethernet is enabled.
4. 52 specifices whether Infiniband is enabled.
5. 53 specifies whether Ethernet (in general) is enabled.
6. I *think* 54 specifies how many Virtual Protocol Interconnect (VPI, IB to Ethernet GW) ports are available.
7. 55 specifies the number of Unified Fabric Manager (UFM) ports that are available.
8. 56 and 57 specify the speed limit on Infiniband and Ethernet ports, respectively.
9. 58 specifies whether L3 Ethernet is enabled.
10. 59 specifies whether the Fibre Channel Fabric (FCF) feature is enabled.
11. 60, 61, 62 are OEM licenses that I don't think are strictly applicable to the SX6036.

Someone ran a trace on the dumplicense binary a long time ago and found that the shared secret is `m2l0n%0x9`. If you want to look at it yourself, `dumplicense` is in the same spot as `genlicense` and takes a license as input. You'll probably want to extract the x86 version (see the last part of this section) so you can run `ltrace` on it. Look for the magic number that `dumplicense` stores in memory.

Anyway.. to generate a generic license with no time or hardware restriction that will enable basically everything:

```txt
[admin@switch-b7b218 ~]# genlicense 2 EFM_SX m2l0n%0x9 -o 51 true -o 52 true -o 53 true -o 54 1024 -o 55 1024 -o 58 true -o 59 true
LK2-EFM_SX-5K11-5L11-5M11-5N31-005P-3100-5T11-5U11-88A1-2953-7H6W-T
```

That license key will work on any SX6036. Install your generated license on the switch with the config mode `license install` command:

```txt
switch-b7b218 [standalone: master] > en
switch-b7b218 [standalone: master] # conf t
switch-b7b218 [standalone: master] (config) # license install LK2-EFM_SX-5K11-5L11-5M11-5N31-005P-3100-5T11-5U11-88A1-2953-7H6W-T
License was installed successfully. Please wait 1 minute before further configurations.
switch-b7b218 [standalone: master] (config) # wr mem
switch-b7b218 [standalone: master] (config) # sh license
License 1: LK2-RESTRICTED_CMDS_GEN2-88A1-NEWD-BPNB-1
   Feature:          RESTRICTED_CMDS_GEN2
   Description:      Access to restricted system functionality
   Valid:            yes
   Active:           yes

License 2: LK2-EFM_SX-5K11-5L11-5M11-5N31-005P-3100-5T11-5U11-88A1-2953-7H6W-T
   Feature:          EFM_SX
   Description:      Generic SX license
   Valid:            yes
   Active:           yes
   Eth L2 enabled:   true
   IB enabled:       true
   Eth enabled:      true
   GW ports number:  1024
   Max num ufm ports supported: 1024
   Eth L3 enabled:   true
   FCF enabled:      true
```

### Extracting an OS image and running `genlicense` on an x86 machine

I did this to generate a host license before realizing that the shell access license I'd found online is generic (dummy me). You can generate it, but it'll always be the same:

```txt
[liam@m715q ~]$ /tmp/image/opt/tms/bin/genlicense 2 RESTRICTED_CMDS_GEN2 m2l0n%0x9
LK2-RESTRICTED_CMDS_GEN2-88A1-NEWD-BPNB-1
```

Anyway, I'm going to leave this here because I already wrote it up, and this is probably how you'd want to go about tracing `dumplicense` to find the magic number if you were so inclined.

Since the license generator is in the MLNX-OS tarball, we can pull down an x86 version of MLNX-OS (I'm using 3.6.8008 for the SB7800 from <https://www.mellanox.com/downloads/Software/image-X86_64-3.6.8008.img>), use `binwalk` to disassemble the `img` file and extract the filesystem tarball, then run the binary on any x86 machine.

I've copied the image ("image-X86_64-3.6.8008.img") to a Linux machine (Alma 10) that has `podman`, `curl`, and `git` installed (we'll need this to build and run Binwalk in a container).

First, clone the binwalk source:

```sh
git clone https://github.com/ReFirmLabs/binwalk && cd binwalk
```

Edit the `docker_build` script to alias `docker=podman` (yeah, you could also replace the single `docker` call):

```sh
#!/usr/bin/env bash

shopt -s expand_aliases
alias docker=podman

docker build --build-arg SCRIPT_DIRECTORY=$PWD -t binwalkv3 .
```

Run the build script (don't need to elevate this since Podman):

```sh
./docker_build.sh
```

Wait for the build to complete. I made the mistake of running it on a Zen 1 machine, so it took a little while.

Make a directory in `/tmp` for the container, copy the `.img` there:

```sh
mkdir /tmp/analysis
mv ~/image-X86_64-3.6.8008.img /tmp/analysis/
```

Run the container, with automatic private relabeling (:Z), and `keep-id` (so you don't have to `chown 1000:1000 /tmp/analysis`:

```sh
podman run --userns=keep-id -t -v /tmp/analysis:/analysis:Z binwalkv3 -Me /analysis/image-X86_64-3.6.8008.img
```

`binwalk` should spit out its analysis. In my case, it says that:

- the image is a ZIP archive
- there's a gzipped operating system image in the ZIP archive
  - the OS image is of Linux 3.10
- it found a tarball in there somewhere

You can look at what was extracted from the image by navigating to the `binwalk` output directory:

```txt
[liam@m715q ~]$ cd /tmp/analysis/extractions/image-X86_64-3.6.8008.img.extracted/0/
[liam@m715q 0]$ ls -l
total 585440
-rwxr-xr-x. 1 liam liam      7820 Jul  3  2018 build_version.sh
-rwxr-xr-x. 1 liam liam      5467 Jul  3  2018 build_version.txt
-rw-r--r--. 1 liam liam 573662040 Jul  3  2018 image-X86_64-x86_64-x86_64-20180703-203209.tbz
-rw-r--r--. 1 liam liam       212 Jul  3  2018 image_vars.sh
-rw-r--r--. 1 liam liam       502 Jul  3  2018 md5sums
-rw-r--r--. 1 liam liam  21543657 Jul  3  2018 mfg-initrd-X86_64-x86_64-x86_64-20180703-203209
-rw-r--r--. 1 liam liam   4228528 Jul  2  2018 mfg-kernel-X86_64-x86_64-x86_64-20180703-203209
drwxr-xr-x. 3 liam liam      4096 Oct 19 13:42 mfg-kernel-X86_64-x86_64-x86_64-20180703-203209.extracted
-rw-r--r--. 1 liam liam       363 Jul  3  2018 mfg_vars.sh
-rw-r--r--. 1 liam liam      2033 Jul  3  2018 tpkg-manifest
-rw-r--r--. 1 liam liam       455 Jul  3  2018 tpkg-manifest.sig
-rw-r--r--. 1 liam liam        49 Jul  3  2018 tpkg-vars
```

I'll extract the `image.tbz` tarball:

```sh
mkdir /tmp/image
tar xf image-X86_64-x86_64-x86_64-20180703-203209.tbz -C /tmp/image/
```

If you have a look, you'll see that this is a `MLNX-OS` filesystem:

```txt
[liam@m715q ~]$ ls /tmp/image/
IBswcountlimits.pm  bin  boot  bootmgr  config  data  debug_symbols  dev  etc  home  initrd  iptables  iptables-multi  iptables-restore  iptables-save  lib  lib64  media  mnt  opt  output  proc  root  run  sbin  share  srv  stats-files  stats-reports  sys  tmp  usr  var  vtmp
```

We can then `find` and run the `genlicense` binary.

```txt
[liam@m715q ~]$ find /tmp/image/ -name "genlicense" 2> /dev/null
/tmp/image/opt/tms/bin/genlicense
```

```txt
[liam@m715q ~]$ /tmp/image/opt/tms/bin/genlicense
Usage: genlicense <license type ID> <type-specific options ...>

genlicense 1 <feature name> <start date> <end date>
             <tied info id> <tied info str> <secret>

    Feature name:   String of characters not including hyphen '-'
    Start date:     0 for no limit, or date like 2003/10/15
    End date:       0 for no limit, or date like 2003/10/15
    Tied info id:   1 for first MAC address
                    2 for host ID
                    3 for a license not tied to any specific machine
    Tied info str:  For tied info id 1: MAC like 11:33:55:77:99:AA
                    For tied info id 2: Host ID like 8bbf35f82b2c
                    For tied info id 3: ignored (use "-")
    Secret:         Shared secret, quoted or escaped if necessary

genlicense 2 <feature name> <secret> [-h <hash type>] [-u <length>] [-i]
             [-f] [-o <option tag id> <option value>]*

    Feature name:   String of capital letters, numbers, and '_'
    Secret:         Shared secret, quoted or escaped if necessary

    Hash type: defaults to hmac_sha256_48 if not specified.
    Use either the number or the string in parentheses to specify.
       1 (hmac_md5_full): HMAC MD5 with full 128-bit length
       2 (hmac_md5_96): HMAC MD5 truncated to 96 bits
       3 (hmac_md5_48): HMAC MD5 truncated to 48 bits
       5 (hmac_sha256_full): HMAC SHA256 with full 256-bit length
       6 (hmac_sha256_128): HMAC SHA256 truncated to 128 bits
       7 (hmac_sha256_96): HMAC SHA256 truncated to 96 bits
       8 (hmac_sha256_48): HMAC SHA256 truncated to 48 bits

    The '-u' option requests to embed a randomly generated number in the
    license, which serves to make it unique from other licenses with the same
    options.  Useful e.g. if you are using a cumulative informational option.
    The parameter is a length in bits, up to 64.

    The '-i' option requests to add a validator identifier to the license.
    This makes the license five characters longer, but may speed up license
    calculations in some cases.

    The '-f' option forces generation of a license that otherwise failed our
    validation criteria: either (a) it does not meet the current requirements
    for hash length and/or algorithm, or (b) it uses an option tag which is
    not relevant to the particular feature chosen.

    The '-o' option can be used zero or more times to specify options
    for the license.  Each option tag has an id (which encodes the data
    type and meaning of the tag) and a value.  There are two classes of
    option tags, which share the same id namespace.  Specify the id either
    by number, or by the string shown in parentheses after the number:

    Activation option IDs:
        1 (start_date): Start date: license not active before this date
        2 (end_date): End date: license not active after this date
        3 (tied_primary_mac): Tie to this MAC address on primary interface
        4 (tied_host_id): Tie to this host ID
        5 (tied_host_id_hex): Tie to this host ID (lowercase hexadecimal)
        6 (tied_serialno): Tie to this serial number
        7 (tied_uuid): Tie to this UUID (must be RFC 4122 compliant)
       49 (efm_sx_serial_num): Chassis serial number for license verification

    Informational option IDs:
       48 (efm_sx_max_num_hca_ports): Maximum number of HCA ports supported by this MLNX-OS SwitchX license
       50 (efm_sx_active_ports): Active ports number supported by this MLNX-OS SwitchX license
       51 (efm_sx_l2_enabled): Eth L2 enabled by this MLNX-OS SwitchX license
       52 (efm_sx_ib_enabled): IB enabled by this MLNX-OS SwitchX license
       53 (efm_sx_eth_enabled): Eth enabled by this MLNX-OS SwitchX license
       54 (efm_sx_gw_ports): GW ports number supported by this MLNX-OS SwitchX license
       55 (efm_sx_max_ufm_ports): Maximum number of UFM ports supported by this MLNX-OS SwitchX license
       56 (efm_sx_ib_speed_sw_limit): IB port SW speed limit enabled by this MLNX-OS SwitchX license
       57 (efm_sx_eth_speed_sw_limit): Eth port SW speed limit enabled by this MLNX-OS SwitchX license
       58 (efm_sx_l3_enabled): Eth L3 enabled by this MLNX-OS SwitchX license
       59 (efm_sx_fcf_enabled): FCF enabled by this MLNX-OS SwitchX license
       60 (oem_lic_10gbps_enable): 10 Gbps ports licensed by OEM license
       61 (oem_lic_25gbps_enable): 25 Gbps ports licensed by OEM license
       62 (oem_lic_100gbps_enable): 100 Gbps ports licensed by OEM license
```

This is the same thing that you get on a SX6036, just compiled for x86. I was able to install a license generated with it on my switch.

If you want to trace `dumplicense`, install `gdb` and `ltrace`, then `ltrace dumplicense {license}`. Look for the repeated memory addresses in `memcmp` calls. Run `gdb` and examine the memory location that's being referenced. `gdb` is broken on the box I was using so I didn't go through the whole process, but I found a few addresses of interest.

## Changing the system profile to VPI mode (enable Infiniband AND Ethernet)

You can use the `system profile` commands to change the system from Infiniband mode to Ethernet or VPI (both) mode. The SX6036, being an Infiniband switch, will ship in `ib-single-switch` mode. I'll change mine to "Virtual Protocol Interconnect" (VPI) mode, which allows you to switch ports between IB and Ethernet mode.

```txt
switch-b7b218 [standalone: master] (config) # system profile vpi-single-switch
Warning - confirming will cause system reboot and all configuration will be deleted
Type 'yes' to confirm profile change: yes
```

The management configuration will *not* be wiped by this change. When the switch reloads, you can set specific ports (or all of them) to Ethernet mode.

[A manual for configuring the switch may be found here, from IBM](https://delivery04.dhe.ibm.com/sar/CMA/XSA/MLNX-OS_VPI_v3_4_3002_UM.pdf).

## Quieting the fans

You can run `fae mlxi2c` commands to slow the fans in the SX6036 from the CLI:

```txt
switch-b7b218 [standalone: master] # fae mlxi2c set_fan /FAN/FAN 1 30
switch-b7b218 [standalone: master] # fae mlxi2c set_fan /PS2/FAN 1 30
switch-b7b218 [standalone: master] # fae mlxi2c set_fan /PS1/FAN 1 30
```

This brings the switch down to a much more tolerable noise level comparable to my Cisco 3750X and 3850 1 GBe non-PoE switches (the UPOE multigig ones are much louder). This also drops power consumption on my switch to around 42w at idle (4 connections up).

On a related note, you can use the `show temperature` command to see sensor readouts from your switch:

```txt
switch-b7b218 [standalone: master] # sh temp
---------------------------------------------------------
Module      Component              Reg  CurTemp    Status
                                        (Celsius)
---------------------------------------------------------
MGMT        SX                     T1   32.00      OK
MGMT        QSFP_TEMP1             T1   25.50      OK
MGMT        QSFP_TEMP2             T1   26.50      OK
MGMT        QSFP_TEMP3             T1   26.00      OK
MGMT        BOARD_MONITOR          T1   31.00      OK
MGMT        CPU_BOARD_MONITOR      T1   30.00      OK
MGMT        CPU_BOARD_MONITOR      T2   62.00      OK
```

Note the switch chip (SX), and CPU temperatures. The little PowerPC chip gets quite warm.

MLNX-OS seems to use SysVinit, so, to make your fan tweaks persistent, you can write a script to set fan_speed and maximum at startup and either dump it in the `rc.local` file or as a service under `/etc/init.d`.

I've created a really simple `fanspeed` script as a service under `/etc/init.d` referencing [this post on the STH forums](https://forums.servethehome.com/index.php?threads/mellanox-sx6036-fan-mod.35316/post-325392). I found that I could just set my PSU fans to 10% and chassis fans to 20% without triggering the temp alarm and a fan speed spike; this may differ depending on your ambient conditions. This quiets my switch down nicely (barely louder than a 1G Cisco 3750X-48T/non-PoE switch that I've got kicking around, perfectly acceptable for a closet somewhere).

If you're feeling really fancy you could probably do something cool and write a PID control loop for fan control; I was not feeling really fancy, and the switch will ramp the fans on its own if it gets too hot. Maybe a future project.

To make any changes to the root partition, you'll need to remount it read/write.

```sh
mount -o rw,remount /
```

Then, dump your new script in:

```sh
tee /etc/init.d/fanspeed << 'EOT'
#!/bin/sh
#
# fanspeed.sh custom fan speed setter
#
# chkconfig: 2345 99 99
# description: waits for clusterd (system full boot) then sets fan speed minimums

start() {

    logger "fanspeed.sh: Waiting for clusterd"
    
    count=1
            
    while true; do
    
        if [ "$count" -gt 15 ]; then
            
            logger "fanspeed.sh: FAILED! Timed out waiting for clusterd (15 rep.); aborting."
            break
        
        fi
        
        if pgrep -f "/opt/tms/bin/clusterd" > /dev/null; then

            logger "fanspeed.sh: clusterd is running; system has booted. Starting final 30s wait."
            
            sleep 30
            
            logger "fanspeed.sh: Setting fan speed minimum to 20 (CHA), 10 (PS{1,2})"
                
            /opt/tms/bin/mdreq action /system/chassis/actions/set-fan-speed fan_module string "/FAN/FAN" fan_number int8 1 fan_speed int8 20 set_max uint8 100
            /opt/tms/bin/mdreq action /system/chassis/actions/set-fan-speed fan_module string "/PS1/FAN" fan_number int8 1 fan_speed int8 10 set_max uint8 100
            /opt/tms/bin/mdreq action /system/chassis/actions/set-fan-speed fan_module string "/PS2/FAN" fan_number int8 1 fan_speed int8 10 set_max uint8 100
            
            logger "fanspeed.sh: Set fan speeds. Exiting."
            
            break
            
        else
        
            count=$((count + 1))
            
            sleep 30
            
        fi
        
    done  
    
}

case "$1" in
    start)
        start
        ;;
    stop)
        echo "fanspeed.sh: nothing to stop."
        ;;
    restart)
        $0 stop
        $0 start
        ;;
    *)
        echo "usage: $0 {start|stop|restart}"
        exit 1
        ;;
esac

exit 0
EOT
```

Make the script executable, then add it to the init list with `chkconfig`.

```sh
chmod +x /etc/init.d/fanspeed
chkconfig --add fanspeed
chkconfig fanspeed on
```

You shouldn't get any errors here.

Confirm you can start the `fanspeed` script:

```txt
[admin@switch-b7b218 ~]# /etc/init.d/fanspeed start
```

And that it's registered with `chkconfig`:

```txt
[admin@switch-b7b218 ~]# chkconfig --list | grep fanspeed
fanspeed        0:off 1:off 2:on 3:on 4:on 5:on 6:off
```

Then, reboot to test the script. Once the switch boots up, the fans should quiet down.

## Switching ports to Ethernet

Since I'm running in VPI (Infiniband-or-Ethernet) mode, I'll have to switch ports from IB mode to Eth mode. This can be done with:

```txt
switch-b7b218 [standalone: master] (config) # sh ports type
InfiniBand: 1/1 1/2 1/3 1/4 1/5 1/6 1/7 1/8 1/9 1/10 1/11 1/12 1/13 1/14 1/15 1/16 1/17 1/18 1/19 1/20 1/21 1/22 1/23 1/24 1/25 1/26 1/27 1/28 1/29 1/30 1/31 1/32 1/33 1/34 1/35 1/36
switch-b7b218 [standalone: master] (config) # int ib 1/1-1/36
switch-b7b218 [standalone: master] (config interface ib 1/1-1/36) # shutdown
switch-b7b218 [standalone: master] (config interface ib 1/1-1/36) # exit
switch-b7b218 [standalone: master] (config) # port 1/1-1/36 type ethernet
switch-b7b218 [standalone: master] (config) # int eth 1/1-1/36
switch-b7b218 [standalone: master] (config interface ethernet 1/1-1/36) # no shut
switch-b7b218 [standalone: master] (config interface ethernet 1/1-1/36) # exit
switch-b7b218 [standalone: master] (config) # wr mem
switch-b7b218 [standalone: master] (config) # exit
switch-b7b218 [standalone: master] # sh ports type
Ethernet:   1/1 1/2 1/3 1/4 1/5 1/6 1/7 1/8 1/9 1/10 1/11 1/12 1/13 1/14 1/15 1/16 1/17 1/18 1/19 1/20 1/21 1/22 1/23 1/24 1/25 1/26 1/27 1/28 1/29 1/30 1/31 1/32 1/33 1/34 1/35 1/36
```

My `eth 1/1` port, hooked up to a CX4 LX, will link right up at 40 Gbps out of the box.

```txt
switch-b7b218 [standalone: master] # sh int eth 1/1

Eth1/1:
  Admin state                               : Enabled
  Operational state                         : Up
  Last change in operational status         : 0:00:48 ago (1 oper change)
  Boot delay time                           : 0 sec
  Description                               : N\A
  Mac address                               : 00:02:c9:8e:e6:ac
  MTU                                       : 1500 bytes (Maximum packet size 1522 bytes)
  Fec                                       : auto
  Flow-control                              : receive off send off
  Actual speed                              : 40 Gbps
```

If you have a CX3 Pro or CX4-non-LX, force the switchport to `speed 56000` to make the two link up at 56 Gbps (cabling permitting). This is a proprietary Mellanox thing so you need Mellanox cabling/optics and Infiniband HCAs that support Ethernet (since they have hardware support for the fourteen data rate signalling rate).

## Connecting a machine with a CX4

I hooked up a computer with a ConnectX-4 and another with a SFN7122F (with some breakouts) to verify that things were working.

The CX4-LX links up at 40 Gbps right out of the box (with the in-tree driver in both Fedora 42 and Alma 10).

This NIC doesn't support the proprietary 56 Gbps Ethernet mode that the full fat Infiniband HCAs (the CX4 non LX and some CX3s) do; maybe I'll pick up some CX3 Pros for my bigger servers (power consumption be damned).

The CX4 LX use much less power, produce much less heat, and support ASPM, so 16 Gbps that I won't saturate isn't such a large sacrifice to make anyway.

Anyway, here's `ethtool` output:

```txt
wporter@a10cx4:~$ sudo ethtool enp1s0np0
Settings for enp1s0np0:
 Supported ports: [ Backplane ]
 Supported link modes:   1000baseT/Full
                         1000baseKX/Full
                         10000baseKR/Full
                         40000baseKR4/Full
                         40000baseCR4/Full
                         40000baseSR4/Full
                         40000baseLR4/Full
                         25000baseCR/Full
                         25000baseKR/Full
                         25000baseSR/Full
                         50000baseCR2/Full
                         50000baseKR2/Full
 Supported pause frame use: Symmetric
 Supports auto-negotiation: Yes
 Supported FEC modes: None  RS  BASER
 Advertised link modes:  1000baseT/Full
                         1000baseKX/Full
                         10000baseKR/Full
                         40000baseKR4/Full
                         40000baseCR4/Full
                         40000baseSR4/Full
                         40000baseLR4/Full
                         25000baseCR/Full
                         25000baseKR/Full
                         25000baseSR/Full
                         50000baseCR2/Full
                         50000baseKR2/Full
 Advertised pause frame use: Symmetric
 Advertised auto-negotiation: Yes
 Advertised FEC modes: None
 Link partner advertised link modes:  Not reported
 Link partner advertised pause frame use: No
 Link partner advertised auto-negotiation: Yes
 Link partner advertised FEC modes: Not reported
 Speed: 40000Mb/s
 Lanes: 4
 Duplex: Full
 Auto-negotiation: on
 Port: Direct Attach Copper
 PHYAD: 0
 Transceiver: internal
 Supports Wake-on: d
 Wake-on: d
 Link detected: yes
```

For the sake of the example, I'm going to install the proprietary out-of-tree driver. The Ethernet-only "lightweight" driver is no longer supported; it's been supplanted by the in-tree driver (part of the Linux kernel). The MLNX-OFED driver has been replaced with the DOCA-OFED driver ([see the transition guide here (docs.nvidia.com)](https://docs.nvidia.com/doca/archive/2-9-0-cx8/mlnx_ofed+to+doca-ofed+transition+guide/index.html)), and the MLNX-OFED driver's final LTS release came out at the end of 2024 (no new kernel support).

Since I'm running Alma 10, I'll need to install [the DOCA-OFED driver for RHEL 10](https://developer.nvidia.com/doca-downloads?deployment_platform=Host-Server&deployment_package=DOCA-Host&target_os=Linux&Architecture=x86_64&Profile=doca-ofed&Distribution=RHEL-Rocky&version=10.0&installer_type=rpm_online). The more complete drivers don't support EL10 yet (not sure if they ever will - Nvidia really likes Ubuntu).

Install the EPEL repo and activate CRB:

```sh
sudo dnf install -y epel-release
sudo /usr/bin/crb enable
```

Add the GPG key:

```sh
wget http://www.mellanox.com/downloads/ofed/RPM-GPG-KEY-Mellanox-SHA256
sudo rpm --import RPM-GPG-KEY-Mellanox-SHA256
```

Nvidia disables `gpgcheck`... You can reenable it, and I have below - they do actually sign things with the GPG key you installed above.

```sh
sudo tee /etc/yum.repos.d/doca.repo << 'EOT'
[doca]
name=DOCA Online Repo
baseurl=https://linux.mellanox.com/public/repo/doca/3.1.0/rhel10.0/x86_64/
enabled=1
gpgcheck=1
EOT
```

Go ahead and install the `doca-ofed` package now:

```sh
sudo dnf install -y doca-ofed
```

When done, give your machine a reboot to load the new kernel modules. You should then be able to use the Mellanox utilities to work with your NIC.

```txt
wporter@a10cx4:~$ sudo mst start
Starting MST (Mellanox Software Tools) driver set
Loading MST PCI module - Success
Loading MST PCI configuration module - Success
Create devices
Unloading MST PCI module (unused) - Success
wporter@a10cx4:~$ sudo mlxfwmanager
Querying Mellanox devices firmware ...

Device #1:
----------

  Device Type:      ConnectX4LX
  Part Number:      MCX4131A-GCA_Ax
  Description:      ConnectX-4 Lx EN network interface card; 50GbE single-port QSFP28; PCIe3.0 x8; ROHS R6
  PSID:             MT_2430110032
  PCI Device Name:  /dev/mst/mt4117_pciconf0
  Base MAC:         98039b63e56c
  Versions:         Current        Available
     FW             14.20.1010     N/A
     PXE            3.5.0210       N/A

  Status:           No matching image found
```

If you want to update firmware ([available from Nvidia at network.nvidia.com/support](https://network.nvidia.com/support/firmware/connectx4lxen/)) to get ASPM support, you can do so now.

## Wrapping things up

I think that's about it. More advanced stuff coming soon.

This switch will be installed soon so it can start serving my clusters, but I like it so much that I might buy another.
