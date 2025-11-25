---
title: "Testing and stressing spinning drives"
date: 2025-11-21T20:30:00-00:00
draft: true
---

## Intro

When purchasing spinning drives from eBay, you'll usually want to run a torture test on them (so if they're going to die, they die during your return window).

I like to use two utilities for this:

- `badblocks(8)` is a utility to "search a device for bad blocks".
- `smartmontools` allows you to send commands to supported drives, e.g., to request that they perform a self-test.

`badblocks` isn't *perfect* for its original intended use case - e.g., it can't tell if your drives decide to use their massive reallocation areas and have fixed bad blocks themselves, which they often will - but it does give you an easy way to generate quite a lot of I/O (of course, you could do the same thing with [`fio`](https://github.com/axboe/fio) if you so please.)

You may be I/O limited if you run `badblocks` on loads of drives in a slow system (e.g., fewer than 8 half-decent threads). A good rule of thumb is a core per disk running `badblocks`.

A useful first test is the SMART Conveyance test, which is designed to detect shipping damage. This usually only takes a few minutes. I'd do this before kicking off any of the tests that take umteen hours to complete:

```sh
for disk in /dev/sd{e,f,g,h,i,j,k,l,m,n,o,p}; do
  sudo smartctl -t conveyance "$disk"
done
```

When done, check the status of the disk with `sudo smartctl -a /dev/sdx`.

Then, you can get `badblocks` going.

Arguments:

- w(rite-mode test): `badblocks` scans for bad blocks by writing some patterns (0xaa, 0x55, 0xff, 0x00) on every block of the device
- s(how progress): write out approximate completion percentage
- v(erbose): write read errors, write errors, and data corruption events to stderr

`badblocks` with these args **WILL WRITE OVER EVERY SECTOR OF THE DISK** (consider this your warning), then read it back to verify that the block device works. **DO NOT RUN THIS ON A DISK THAT CONTAINS DATA YOU CARE ABOUT**.

```sh
for disk in /dev/sd{e,f,g,h,i,j,k,l,m,n,o,p}; do
  sudo badblocks -wsv "$disk" -b 4096 -o ./badblocks_${disk##*/}.log > ./badblocks_status_${disk##*/}.log 2>&1 &
done
```

I also like to run a "long" SMART self-test. This tells the *drive* to do a full media scan of itself. This can be run at the same time as `badblocks`, but, while a background-mode SMART test runs with a low I/O priority, it *does* use I/O, so it'll slow things down a bit.

```sh
for disk in /dev/sd{e,f,g,h,i,j,k,l,m,n,o,p}; do
  sudo smartctl -t long "$disk"
done
```

If these complete, your drive still works, and no SMART errors are raised, you're probably good to go for at least a little while.
