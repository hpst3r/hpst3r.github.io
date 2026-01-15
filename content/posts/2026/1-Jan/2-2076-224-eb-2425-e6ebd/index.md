---
title: "IBM V7000 Expansion Module 2076-224 24SFF SAS2 disk shelves (Xyratex EB-2425-E6EBD)"
date: 2026-01-14T20:45:00-00:00
draft: false
---

I recently decommissioned an IBM V7000 SAN (dual controller, each is commodity Xeon, probably LGA 1366, running what is, as best as I can tell, Linux with a Java webapp for management). Good riddance.

I had a feeling that the disk shelves were just SAS, so I grabbed them for some science. Well, turns out, the Gen 1 IBM V7000 expansion modules are perfectly normal SAS2 disk shelves. Nothing special about them. They are white-labeled Xyratex EB-2425-E6EBD enclosures (as sold by Pure, NetApp, Dell, HPE, Cray, Seagate.. so, if you need parts, they're very easy to find) and pretty quiet for 24x 600gb 10K spinnies per shelf. I hooked them up to a Dell H200E (LSI 2008) HBA and the disks came right up.

A fine home for some future SAS SSDs.. Anyway, here's all 46 600gb 10K drives, in all their glory:

```txt
wporter@3888t0:~$ lsblk -e7
NAME               MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0 558.9G  0 disk
sdb                  8:16   0 558.9G  0 disk
sdc                  8:32   0 558.9G  0 disk
sdd                  8:48   0 558.9G  0 disk
sde                  8:64   0 558.9G  0 disk
sdf                  8:80   0 558.9G  0 disk
sdg                  8:96   0 558.9G  0 disk
sdh                  8:112  0 558.9G  0 disk
sdi                  8:128  0 558.9G  0 disk
sdj                  8:144  0 558.9G  0 disk
sdk                  8:160  0 558.9G  0 disk
sdl                  8:176  0 558.9G  0 disk
sdm                  8:192  0 558.9G  0 disk
sdn                  8:208  0 558.9G  0 disk
sdo                  8:224  0 558.9G  0 disk
sdp                  8:240  0 558.9G  0 disk
sdq                 65:0    0 558.9G  0 disk
sdr                 65:16   0 558.9G  0 disk
sds                 65:32   0 558.9G  0 disk
sdt                 65:48   0 558.9G  0 disk
sdu                 65:64   0 558.9G  0 disk
sdv                 65:80   0 558.9G  0 disk
sdw                 65:96   0 558.9G  0 disk
sdx                 65:112  0 558.9G  0 disk
sdy                 65:128  0 558.9G  0 disk
sdz                 65:144  0 558.9G  0 disk
sdaa                65:160  0 558.9G  0 disk
sdab                65:176  0 558.9G  0 disk
sdac                65:192  0 558.9G  0 disk
sdad                65:208  0 558.9G  0 disk
sdae                65:224  0 558.9G  0 disk
sdaf                65:240  0 558.9G  0 disk
sdag                66:0    0 558.9G  0 disk
sdah                66:16   0 558.9G  0 disk
sdai                66:32   0 558.9G  0 disk
sdaj                66:48   0 558.9G  0 disk
sdak                66:64   0 558.9G  0 disk
sdal                66:80   0 558.9G  0 disk
sdam                66:96   0 558.9G  0 disk
sdan                66:112  0 558.9G  0 disk
sdao                66:128  0 558.9G  0 disk
sdap                66:144  0 558.9G  0 disk
sdaq                66:160  0 558.9G  0 disk
sdar                66:176  0 558.9G  0 disk
sdas                66:192  0 558.9G  0 disk
sdat                66:208  0 558.9G  0 disk
```

```txt
wporter@3888t0:~$ lsscsi
[0:0:0:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdc
[0:0:1:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdb
[0:0:2:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sda
[0:0:3:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdd
[0:0:4:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sde
[0:0:5:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdf
[0:0:6:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdg
[0:0:7:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdh
[0:0:8:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdi
[0:0:9:0]    disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdj
[0:0:10:0]   disk    IBM-207x HUC109060CSS60   J2E6  /dev/sdk
[0:0:11:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdl
[0:0:12:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdm
[0:0:13:0]   enclosu XYRATEX  EB-2425-E6EBD    2126  -
[0:0:29:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdab
[0:0:31:0]   disk    IBM-207x ST600MM0006      B56S  /dev/sdad
[0:0:33:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdaf
[0:0:35:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdah
[0:0:36:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdai
[0:0:38:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdak
[0:0:40:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdam
[0:0:41:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdan
[0:0:42:0]   disk    IBM-207x HUC109060CSS60   J2E6  /dev/sdao
[0:0:46:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdas
[0:0:47:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdat
[0:0:50:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdn
[0:0:51:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdo
[0:0:52:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdp
[0:0:53:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdq
[0:0:54:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdr
[0:0:55:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sds
[0:0:56:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdt
[0:0:57:0]   disk    IBM-207x HUC109060CSS60   J2E6  /dev/sdu
[0:0:58:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdv
[0:0:59:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdw
[0:0:60:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdx
[0:0:61:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdy
[0:0:62:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdz
[0:0:63:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdaa
[0:0:64:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdac
[0:0:65:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdae
[0:0:66:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdag
[0:0:67:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdaj
[0:0:68:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdal
[0:0:69:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdap
[0:0:70:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdaq
[0:0:71:0]   disk    IBM-207x HUC109060CSS60   J2E9  /dev/sdar
[0:0:72:0]   enclosu XYRATEX  EB-2425-E6EBD    2126  -
```
