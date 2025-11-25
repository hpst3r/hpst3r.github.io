---
title: "Testing and stressing spinning drives"
date: 2025-11-21T20:30:00-00:00
draft: false
---

## Intro

When purchasing spinning drives from eBay, you'll usually want to run a torture test on them (so if they're going to die, they die during your return window).

I like to use two utilities for this:

- `badblocks(8)` is a utility to "search a device for bad blocks".
- `smartmontools` allows you to send commands to supported drives, e.g., to request that they perform a self-test.

`badblocks` isn't *perfect* for its original intended use case - e.g., it can't tell if your drives decide to use their massive reallocation areas and have fixed bad blocks themselves, which they often will - but it does give you an easy way to generate quite a lot of I/O (of course, you could do the same thing with [`fio`](https://github.com/axboe/fio) if you so please.)

You may be I/O limited if you run `badblocks` on loads of drives in a slow system (e.g., fewer than 8 half-decent threads). A good rule of thumb is a core per disk running `badblocks`.

Before getting started, make sure you have adequate cooling! Disks don't like heat. Standard server chassis are great options for a lot of enterprise drives in a small space, if a bit loud.

First, dump the SMART data for the drives. Make sure they're healthy! Look for uncorrectible errors, reallocated sectors, and pending sectors. If you see a predicted failure, the drive is done for! Send it back!

```sh
for disk in /dev/sd{c,d,e,f,g,h,i,j,k,l,m,n}; do
  sudo smartctl -x "$disk" > ./smart_${disk##*/}.txt
done
```

(Partial) example output from a healthy Seagate Exos:

```txt
SMART Attributes Data Structure revision number: 1
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAGS    VALUE WORST THRESH FAIL RAW_VALUE
  5 Reallocated_Sector_Ct   -O--CK   100   100   000    -    0
  9 Power_On_Hours          -O--CK   100   100   000    -    42821
 12 Power_Cycle_Count       -O--CK   100   100   000    -    266
170 Available_Reservd_Space PO--CK   100   100   010    -    0
171 Program_Fail_Count      -O--CK   100   100   000    -    0
172 Erase_Fail_Count        -O--CK   100   100   000    -    0
174 Unsafe_Shutdown_Count   -O--CK   100   100   000    -    140
175 Power_Loss_Cap_Test     PO--CK   100   100   010    -    633 (237 447)
183 SATA_Downshift_Count    -O--CK   100   100   000    -    0
184 End-to-End_Error        PO--CK   100   100   090    -    0
187 Reported_Uncorrect      -O--CK   100   100   000    -    0
190 Temperature_Case        -O---K   076   067   000    -    24 (Min/Max 19/33)
192 Unsafe_Shutdown_Count   -O--CK   100   100   000    -    140
194 Temperature_Internal    -O---K   100   100   000    -    35
197 Current_Pending_Sector  -O--CK   100   100   000    -    0
199 CRC_Error_Count         -OSRCK   100   100   000    -    0
225 Host_Writes_32MiB       -O--CK   100   100   000    -    6239339
226 Workld_Media_Wear_Indic -O--CK   100   100   000    -    29317
227 Workld_Host_Reads_Perc  -O--CK   100   100   000    -    17
228 Workload_Minutes        -O--CK   100   100   000    -    2567729
232 Available_Reservd_Space PO--CK   100   100   010    -    0
233 Media_Wearout_Indicator -O--CK   072   072   000    -    0
234 Thermal_Throttle        -O--CK   100   100   000    -    0/0
241 Host_Writes_32MiB       -O--CK   100   100   000    -    6239339
242 Host_Reads_32MiB        -O--CK   100   100   000    -    1370727
                            ||||||_ K auto-keep
                            |||||__ C event count
                            ||||___ R error rate
                            |||____ S speed/performance
                            ||_____ O updated online
                            |______ P prefailure warning
```

If it's supported (older, e.g., ~2015 and earlier drives tend to not have this), a useful first test is the SMART Conveyance test, which is designed to detect shipping damage.

I'm not entirely sure what this test typically does (it's allegedly manufacturer-specific and not documented well), but this usually only takes a few minutes. I'd do this before kicking off any of the tests that take umteen hours to complete.

You can fire off a short self-test at the same time (performs self-tests on the drive's components and reads/writes some data to verify functionality; the long test does the same thing, but is more thorough as it checks the *entire* disk); can't hurt to do so and might save you some time if it finds a fault.

If your drives don't support the test (you can see supported SMART tests in the `smartctl -a` output, but it's easier to just throw the test at them) you'll get a standard `smartctl` blurb like:

```txt
wporter@supermacro3:~$ sudo smartctl /dev/sdp
smartctl 7.4 2023-08-01 r5530 [x86_64-linux-6.12.0-55.40.1.el10_0.x86_64] (local build)
Copyright (C) 2002-23, Bruce Allen, Christian Franke, www.smartmontools.org

SCSI device successfully opened

Use 'smartctl -a' (or '-x') to print SMART (and more) information
```

Rather than a "test is running, please wait x minutes" status update.

Anyway, to queue up a conveyance and short test:

```sh
for disk in /dev/sd{c,d,e,f,g,h,i,j,k,l,m,n}; do
  sudo smartctl -t conveyance "$disk"
  sudo smartctl -t short "$disk"
done
```

When done, check the status of the disk with `sudo smartctl -x /dev/sdx` again. I like to dump this to a file, too.

Then, you can get `badblocks` going. This will generate a bunch of writes (write, read, verify) to confirm the disks are functional (similarly to the long SMART test, but from the OS, not the drive itself).

Arguments:

- w(rite-mode test): `badblocks` scans for bad blocks by writing some patterns (0xaa, 0x55, 0xff, 0x00) on every block of the device
- s(how progress): write out approximate completion percentage
- v(erbose): write read errors, write errors, and data corruption events to stderr
- b(lock size): 4096-byte blocks. Make this match sector size; adjust if you've got 512 byte sectors.
- c (number of blocks tested at once): increase from the default 64 for (slightly) quicker completion. 4096 is good enough on a bigger spinning drive. Uses slightly more memory, reduces syscalls (faster).

`badblocks` with these args **WILL WRITE OVER EVERY SECTOR OF THE DISK** (consider this your warning), then read it back to verify that the block device works. **DO NOT RUN THIS ON A DISK THAT CONTAINS DATA YOU CARE ABOUT**.

```sh
for disk in /dev/sd{c,d,e,f,g,h,i,j,k,l,m,n}; do
  sudo badblocks -wsv "$disk" -b 4096 -c 4096 -o ./badblocks_${disk##*/}.log > ./badblocks_status_${disk##*/}.log 2>&1 &
done
```

I also like to run a "long" SMART self-test. This tells the *drive* to do a full media scan of itself. This can be started at the same time as `badblocks`, but since this has a low I/O priority, it'll sit there doing nothing until `badblocks` is done.

```sh
for disk in /dev/sd{c,d,e,f,g,h,i,j,k,l,m,n}; do
  sudo smartctl -t long "$disk"
done
```

If these complete, your drive still works, and no SMART errors are raised, you're probably good to go!
