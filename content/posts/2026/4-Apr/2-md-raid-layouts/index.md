---
title: "Linux md RAID layouts"
date: 2026-04-11T21:15:00-00:00
draft: false
---

I was working on setting up some big mdadm RAID arrays, so I had to do some reading up on RAID layouts to make sure I was setting them up correctly. The result? This nice (if I say so myself) write-up! Yes, I went a little overboard.

If you're *not* familiar with RAID levels, let's go just deep enough:

- RAID 0 = **blocks "striped" across all drives for best performance, no redundancy**
  - Calling this RAID isn't strictly accurate.. it's not redundant at all!
  - Blocks of data are distributed between all member disks. This means excellent performance (both writes and reads can be highly parallelized).
- RAID 1 = **blocks "mirrored" between all drives for redundancy**
  - Mirrors one drive to n replicas. As such, you only get the capacity of a single drive.
  - Dumb mirror. Very simple, very fast reads (if you throw enough threads at it, you can read from n drives simultaneously), but writes will never be faster than one drive.
- RAID 4 = **one dedicated parity (redundancy) drive, data striped between the rest of the drives.**
  - One drive is reserved for redundancy. Parity data is all kept on one drive.
  - Parity bytes are an XOR combination of data bytes (e.g., P = a XOR b XOR c XOR d XOR e - you can figure out what "a" was given b, c, d, e and P) which is how you get redundancy without a mirror. This is called "erasure coding" - you can reconstruct missing data if you know exactly which blocks are gone. If a drive fails, you know which blocks are gone.
  - As parity data lives on one dedicated drive, performance of the array is limited by that drive.
  - Because RAID4 (and 5, and 6) rely on parity, small I/O is punished. If you write a small chunk, you can't just write. You have to read the old data block, read the old parity block, XOR old data, new data, old parity to get your new parity value, write the new data block, then write the new parity block.
  - Large sequential writes are less affected, because you don't need to read old data - you can just dump a new set of data blocks and parity block to disk all at once.
- RAID 5 = **single set of parity data distributed across the array, data striped between all drives**.
  - The equivalent of one drive's capacity is reserved for redundancy. Parity data is distributed across the array.
  - Like RAID4, but, as the parity blocks are distributed between all members, RAID5 allows more parallelism (all drives store parity and data blocks).
- RAID 6 = **two sets of parity data distributed across the array, data striped between all drives**.
  - The equivalent of two drives' capacity is reserved for redundancy. Parity data is distributed across the array.
  - While RAID5 offers just one set of parity blocks, RAID 6 has traditional XOR parity blocks (the "P" parity data) *and* a second set of parity data that's reversible with some more advanced algebra ("Q" parity data). If you want to read about how that second set of parity data works, [check out Igor Ostrovsky's blog post on the topic](https://igoro.com/archive/how-raid-6-dual-parity-calculation-works/). He does an excellent job at explaining it.
  - The result is that your machine can solve for the missing values using the two parity equations and live data, so you can always figure out what bytes you had on the two disks you're able to lose.
  - If two drives are lost and you need to use the Q parity to reconstruct, performance will suffer significantly, as every read will require reading from all surviving drives and doing relatively expensive calculations.
  - However, if you only lose one drive and can reconstruct the missing data from the P parity algorithm or live data (e.g., only had one drive failure) the array will continue to perform relatively well.
- RAID 10 = **striped mirrors = pairs of two drives are mirrored, then the mirrored pairs are all striped together.**
  - Half of your drives are used for redundancy. Data is mirrored across a configurable number of members of the array (by default, 2). As such, you'll lose at least 50% of your disk space.
  - Simpler than erasure coding. Very fast. Works just like you'd think it does with "near" distribution, which is what is typically used. More on that later.

ZFS and LVM support some other fun things like linear volumes or triple parity, but those are different beasts entirely.

The `md` ("multiple device") driver provides "virtual" devices for RAID, pseudo-RAID (linear, stripe/RAID0), and two non-RAID uses (multipath and 'faulty' devices, which we won't get into here).

Anyway. RAID layouts. We're going to be specifically discussing this in the context of Linux `md` software RAID.

These 'layouts' determine how the parity data is distributed (in a RAID-5 or RAID-6 array) or how the redundant blocks are distributed (in a RAID-10 array).

RAID 5 supports six layouts:

1. left-asymmetric
    1. logical block 0 in a stripe is always on disk 0; skip the parity block wherever it lands. stripe to the left, from disk -1 (the last disk) to disk 0 (the first disk).
2. left-symmetric
    1. logical block 0 in a stripe lands on the disk after the last parity block, wrapping around. stripe to the left, from disk -1 (the last disk) to disk 0 (the first disk).
3. right-asymmetric
    1. as #1, but from disk 0 (the first disk) to disk -1 (the last disk). No performance impact.
4. right-symmetric
    1. as #2, but from disk 0 (the first disk) to disk -1 (the last disk). No performance impact.
5. parity-first
    1. faux RAID4, first disk holds all parity data
6. parity-last
    1. faux RAID4, last disk holds all parity data

The default is left-symmetric.

Parity-first and parity-last are faux RAID4 layouts, where parity data is distributed to the first or last drive, respectively.

RAID6 supports the same four layouts for distributing the parity data as RAID5, plus a few interim options (to be used while converting from RAID5 to RAID6) that we won't be covering.

We can determine what the difference between the algorithms is by referring to the source ([drivers/md/raid5.c](https://github.com/torvalds/linux/blob/master/drivers/md/raid5.c)). Unfortunately there's no nice explainer in a manpage (as far as I can tell). Bear with me here, I'm not that great with C.

```c
 case 5:
  switch (algorithm) {
  case ALGORITHM_LEFT_ASYMMETRIC:
   pd_idx = data_disks - sector_div(stripe2, raid_disks);
   if (*dd_idx >= pd_idx)
    (*dd_idx)++;
   break;
  case ALGORITHM_RIGHT_ASYMMETRIC:
   pd_idx = sector_div(stripe2, raid_disks);
   if (*dd_idx >= pd_idx)
    (*dd_idx)++;
   break;
  case ALGORITHM_LEFT_SYMMETRIC:
   pd_idx = data_disks - sector_div(stripe2, raid_disks);
   // logical block number = (physical disk after parity + block number) modulo raid_disks (wrap around if we go past last)
   *dd_idx = (pd_idx + 1 + *dd_idx) % raid_disks;
   break;
  case ALGORITHM_RIGHT_SYMMETRIC:
   pd_idx = sector_div(stripe2, raid_disks);
   *dd_idx = (pd_idx + 1 + *dd_idx) % raid_disks;
   break;
  case ALGORITHM_PARITY_0:
   pd_idx = 0;
   (*dd_idx)++;
   break;
  case ALGORITHM_PARITY_N:
   pd_idx = data_disks;
   break;
```

The difference between the right and left placement algorithms is simply the "direction" in which blocks are mapped to disks - the "left" algorithms start from the last disk (moving left, from last to first). The "right" algorithms do the opposite (moving from the first disk to the last disk).

The difference between the symmetric and asymmetric algorithms is that, with an asymmetric layout, logical block 0 in each stripe is always mapped to the first disk. With a symmetric layout, logical block 0 in a stripe always maps to the disk immediately after the parity block (the starting point for data block numbering rotates with the parity block as data is written to the disks).

This means that, with an asymmetric layout, a sequential read spanning multiple stripes will sometimes hit the same physical disk twice in a row, so a symmetrical layout can have a slight sequential read advantage. Because of this, you should probably just stick with the default of `left-symmetric` for a RAID5 or RAID6 array.

The "special" faux-RAID4 options just use the first or last drive as a dedicated parity disk.

RAID10 supports three. You can specify the number of copies of data.

1. near
2. offset
3. far

The meanings of these layouts may be inferred from the md source, or from `md(4)`.

Near: multiple copies of one data block are at similar offsets ("as close as possible") on different devices. This is probably what you think of when you think of RAID 10. With an even number of devices, two devices are exactly "mirrored" in a pair. With an odd number of devices, blocks are distributed so that you get the configured amount of copies, and the copies are on "nearby" devices. For SSDs, near is ideal - without spindles, physical placement of data on disk is irrelevant, so the simpler layout wins. For spinning drives, near placement offers the best sequential write performance (fewest seeks) but the worst sequential read performance, as you can't benefit from parallelism during read.

Far: copies of a chunk are laid out "as far as reasonably possible" from each other. This means that, rather than mirroring two nearby disks, blocks are distributed amongst the array and to different (distant) positions on each disk. This means `md` can easily distribute sequential reads over the (spinning) devices, making this type of RAID 10 array perform more like a RAID 0 array when reading (when using spinning rust as backing devices). However, more seeking makes this layout slower for writes. For SSDs, this doesn't matter much.

Offset: rather than duplicating chunks, duplicate entire stripes and rotate them by one device so each stripe (and, thus, its contained blocks) is duplicated to a different device. Without getting into too much detail, this stripes consecutive blocks across devices, then "offsets" the stripe's mirror stripes to distribute them between the drives in the array (both distributing their positions, and ensuring that a block in a stripe and its mirrored block aren't on the same disk). This helps improve sequential read characteristics on spinning drives, similarly to the "far" layout, but at a lesser write penalty as, when writing, data is distributed rather closer together (as far as the spindles are concerned). Again, for SSDs, this doesn't matter much.

If you're using spinning drives and need read performance, you should consider using the `offset` layout. If you're using spinning drives and need write performance, you should use the `near` layout.

If you're using SSDs, you should just use `near`. The performance benefits of `far` and `offset` aren't applicable to nonrotational media and might cause stripe lock contention. For example, with the `far=2` layout, a write touches two stripe regions, so it needs two write locks, and can impact other IOs on the array. I wasn't able to see this in benchmarks but didn't go too crazy attempting to get it to happen.
