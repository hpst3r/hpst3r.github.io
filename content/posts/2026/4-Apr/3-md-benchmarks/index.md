---
title: "Linux md RAID benchmarks on 4 Kioxia PM5s"
date: 2026-04-18T14:30:00-00:00
draft: false
---

I had access to some machines with a load of brand new Kioxia PM5 (read-intensive SAS3) SSDs, so after finishing my production benchmarking I decided to grab some academic numbers to follow up on [my prior `md` RAID layouts writeup](https://wporter.org/linux-md-raid-layouts/).

I'll be benchmarking just four drives here, as the PCIE 3.0x8 HBA they're hooked up to is a limiting factor for anything wider. These drives really could use a NVME interface (that's what the CM5 is!)

I'm testing with default writeback caching enabled, but bypassing the page cache (`--direct=1`). This is perfectly fine and reflects any production workload on enterprise drives, as these PM5s have power-loss prevention - using their caches is safe, the drives guarantee they'll write the write cache to NAND even if power is lost.

These drives have a 12 Gb/s SAS3 interface, and are connected to a single LSI 3008 (so they're in single-port mode, not in dual-port mode with multipath - this means performance is about half what you'll see on the spec sheets/half of what it could be) via the chassis' SAS expander. The rest of the test machines are:

- Dell R740xd
- 2x Xeon Gold 6254 (3.9 GHz)
- 1DPC 64gb DDR4 2933 (768gb)
- latency-performance `tuned` profile (effectively stuck in C0)
- LSI 3008 (PERC HBA330)
- Proxmox VE 9.1.6

The testing node was completely empty (no guests or other workloads).

These results clearly illustrate the architecture of different `md` RAID layouts. Let's dive in!

## fio jobs

[See gist](https://gist.github.com/hpst3r/1085d73ae5e979eb491de2b561e4ea96). Essentially:

- bs=1M for sequential
- bs=4K for random
- using a 128 Gb LVM atop `/dev/mdx`, default LVM configuration
- numjobs = threads
- iodepth = queue depth
- ioengine = libaio
- 30s, time-based

## Single drive

We'll start off with a baseline.. a single 1.92 Tb Kioxia PM5 (KPM5XRUG1T92).

- Sequential read
  - SEQ1MQ8T8 read 1112 MB/s
  - SEQ1MQ8T1 read 1110 MB/s
- Sequential write
  - SEQ1MQ8T8 write 1123 MB/s
  - SEQ1MQ8T1 write 1123 MB/s
- Random read
  - RND4KQ32T8 read 180K IOPS (739 MB/s)
  - RND4KQ32T1 read 180K IOPS (738 MB/s)
  - RND4KQ1T1 read 17.3K IOPS (70.9 MB/s)
- Random write
  - RND4KQ32T8 write 169K IOPS (693 MB/s)
  - RND4KQ32T1 write 173K IOPS (708 MB/s)
  - RND4KQ1T1 write 29.5K IOPS (121 MB/s)

Here we see strong sequential I/O (about as good as can be expected from a drive on a single 12 Gb/s SAS interface), along with healthy random I/O performance. All in all, a decent result for some cheaper drives.

## 4-wide mirror

Our first RAID configuration will be a four-way mirror. This will allow us to parallelize reads between the four drives, but writes will be limited by the performance of a single disk. Our random reads will also parallelize nicely when we've got more than one thread doing the reading.

Note how our random Q1 T1 I/O (e.g., ad-hoc, just testing latency) is significantly degraded the second we throw the multiple device driver in between `fio` and the disks. This will be a consistent factor with all of our `md` RAID results.

```sh
mdadm --create /dev/md2 --level=1 --raid-devices=4 /dev/sdk /dev/sdl /dev/sdm /dev/sdn --bitmap none
```

- Sequential read
  - SEQ1MQ8T8 read 4433 MB/s
  - SEQ1MQ8T1 read 4006 MB/s
- Sequential write
  - SEQ1MQ8T8 write 910 MB/s
  - SEQ1MQ8T1 write 1027 MB/s
- Random read
  - RND4KQ32T8 read 713K IOPS (2786 MB/s)
  - RND4KQ32T1 read 202K IOPS (828 MB/s)
  - RND4KQ1T1 read 7604 IOPS (31.1 MB/s)
- Random write
  - RND4KQ32T8 write 154K IOPS (632 MB/s)
  - RND4KQ32T1 write 91.7K IOPS (375 MB/s)
  - RND4KQ1T1 write 24.5K IOPS (100 MB/s)

## 4-wide RAID 10

Next, we'll switch to striped mirrors. In this configuration, we'll be able to parallelize reads between all four drives, though RAID 10 incurs slightly more overhead than a dumb mirror for our reads.

And, when writing, we'll see an improvement - not quite double the performance of a single drive, but we can effectively use our two striped pairs for writing at once.

```sh
mdadm --create /dev/md3 --level=10 --raid-devices=4 --layout=n2 --chunk=256 /dev/sdo /dev/sdp /dev/sdq /dev/sdr --bitmap none
```

- Sequential read
  - SEQ1MQ8T8 read 4136 MB/s
  - SEQ1MQ8T1 read 4112 MB/s
- Sequential write
  - SEQ1MQ8T8 write 1648 MB/s
  - SEQ1MQ8T1 write 2098 MB/s
- Random read
  - RND4KQ32T8 read 709K IOPS (2902 MB/s)
  - RND4KQ32T1 read 201K IOPS (825 MB/s)
  - RND4KQ1T1 read 7286 IOPS (29.8 MB/s)
- Random write
  - RND4KQ32T8 write 276K IOPS (1130 MB/s)
  - RND4KQ32T1 write 143K IOPS (586 MB/s)
  - RND4KQ1T1 write 28.8K IOPS (118 MB/s)

## 4-wide RAID 0

Striping is great. We'll get darn near four times the performance of a single disk in just about everything but Q1T1 random I/O.

Remember, *everything* that's big enough is parallelized across as many drives as possible! RAID 0 is dead simple, and stupid fast as a result. If you don't mind restoring from backup or need a cache, this is definitely the way to go.

```sh
mdadm --create /dev/md0 --level=0 --raid-devices=4 --chunk=256 /dev/sdc /dev/sdd /dev/sde /dev/sdf
```

- Sequential read
  - SEQ1MQ8T8 read 4442 MB/s
  - SEQ1MQ8T1 read 4429 MB/s
- Sequential write
  - SEQ1MQ8T8 write 4211 MB/s
  - SEQ1MQ8T1 write 4199 MB/s
- Random read
  - RND4KQ32T8 read 663K IOPS (2590 MB/s)
  - RND4KQ32T1 read 217K IOPS (890 MB/s)
  - RND4KQ1T1 read 7899 IOPS (32.4 MB/s)
- Random write
  - RND4KQ32T8 write 613K IOPS (2513 MB/s)
  - RND4KQ32T1 write 225K IOPS (922 MB/s)
  - RND4KQ1T1 write 29.9K IOPS (122 MB/s)

## 4-wide RAID 5

Recall that RAID 5 is effectively a stripe with one set of parity data (erasure coding = XOR so we can figure out what we lost given the remaining data and parity bit). So we're going to be able to highly parallelize our read operations a la RAID 0 (data blocks live on all four drives), but we're going to be stuck with the performance of writing that one parity block.

And, while we're here, let's talk about write amplification. RAID 5 works by writing a full stripe, and a parity block for the stripe (say we have our four disks - we'll have three data blocks and one parity block per stripe), to disk simultaneously. The parity block contains the necessary data to reconstruct the stripe if a drive holding data blocks is lost, but it corresponds to the whole stripe, not, for example, a single 4K block in the data chunk on one drive.

This means that, when we're doing tiny I/Os (like our random 4K operations), to update the single block on disk, we must:

1. Read the old data block
2. Read the old parity block
3. Calculate the new parity block (old parity XOR old data XOR new data)
4. Write the new data block (including our changes)
5. Write the new parity block

As you might guess, this is slow. Much slower than a mirror, stripe, single drive, or striped mirrors.

In my example, we have `chunk=256`. This means each chunk of the stripe is 256K, and, thus, to update a 4K block in a chunk we must read back the full 256K data block, read back the full 256K parity block, calculate the new parity block, then write the new 256K data and parity blocks to disk again. So this is terrible! We've got to read and write 1M (read 512K, then write 512K) to write 4K to disk!

```sh
mdadm --create /dev/md0 --level=5 --raid-devices=4 --chunk=256 --bitmap=none /dev/sdc /dev/sdd /dev/sde /dev/sdf
```

- Sequential read
  - SEQ1MQ8T8 read 4438 MB/s
  - SEQ1MQ8T1 read 4436 MB/s
- Sequential write
  - SEQ1MQ8T8 write 985 MB/s
  - SEQ1MQ8T1 write 1327 MB/s
- Random read
  - RND4KQ32T8 read 651K IOPS (2668 MB/s)
  - RND4KQ32T1 read 217K IOPS (888 MB/s)
  - RND4KQ1T1 read 8704 IOPS (35.7 MB/s)
- Random write
  - RND4KQ32T8 write 58.3K IOPS (239 MB/s)
  - RND4KQ32T1 write 68.5K IOPS (280 MB/s)
  - RND4KQ1T1 write 5540 IOPS (22.7 MB/s)

Contrast the random write performance, even with a high queue depth and a few threads, to the performance a RAID 10 array can put out. We see a (276K/58.3K) 4.7x improvement in 4 kilobyte random write I/O operations per second with RAID 10 on these four drives!

> This is even worse on spinning rust.. but that's a topic for another post. I have two disk shelves full of 2.5" spinners waiting for that :)

## 4-wide RAID 6

Recall that RAID 6 has the same characteristics as RAID 5, but with an extra parity block (storing the Q syndrome).

This results in higher overhead for writes (though this system is so fast that isn't particularly apparent) and even greater write amplification on random I/O (though this is parallelized, so we don't see a performance degradation in these simple tests).

A RAID 6 "small block write" goes something like:

1. Read old data block
2. Read the old P parity block
3. Read the old Q parity block
4. Calculate new P and Q
5. Write new data block
6. Write new P parity block
7. Write new Q parity block

Again, as all three read/write operations are on different disks, these tests aren't sufficient to properly demonstrate the impact of RAID 6 on random I/O performance. These PM5s are good enough to make the difference between RAID 5 and RAID 6 into noise in this relatively low load scenario.

With our 256K chunks, we've got to read and write a whopping *1.5M* (768K read, 768K write) to write 4K to disk! Awful!

```sh
mdadm --create /dev/md1 --level=6 --raid-devices=4 --chunk=256 --bitmap=none /dev/sdg /dev/sdh /dev/sdi /dev/sdj
```

- Sequential read
  - SEQ1MQ8T8 read 4438 MB/s
  - SEQ1MQ8T1 read 4445 MB/s
- Sequential write
  - SEQ1MQ8T8 write 951 MB/s
  - SEQ1MQ8T1 write 947 MB/s
- Random read
  - RND4KQ32T8 read 644K IOPS (2639 MB/s)
  - RND4KQ32T1 read 214K IOPS (875 MB/s)
  - RND4KQ1T1 read 8897 IOPS (36.4 MB/s)
- Random write
  - RND4KQ32T8 write 65.4K IOPS (268 MB/s)
  - RND4KQ32T1 write 69.7K IOPS (285 MB/s)
  - RND4KQ1T1 write 8237 IOPS (33.7 MB/s)

> Note that this array and the RAID 5 array were on two different sets of four PM5s (I built four arrays at once, then let them sync simultaneously). I think the random I/O performance on the drives backing the RAID 6 array was particularly good. RAID 6 shouldn't outperform RAID 5 in random I/O.

## Summary table

### Overall numbers

| Test                           | Single drive | 4 drive RAID0 | 4 drive RAID1 | 4 drive RAID10 | 4 drive RAID5 | 4 drive RAID6 |
| ------------------------------ | ------------ | ------------- | ------------- | -------------- | ------------- | ------------- |
| Read - SEQ1MQ8T8 MB/s          | 1112         | 4442          | 4433          | 4136           | 4438          | 4438          |
| Read - SEQ1MQ8T1 MB/s          | 1110         | 4429          | 4006          | 4112           | 4436          | 4445          |
| Write - SEQ1MQ8T8 MB/s         | 1123         | 4211          | 910           | 1648           | 985           | 951           |
| Write - SEQ1MQ8T1 MB/s         | 1123         | 4199          | 1027          | 2098           | 1327          | 947           |
| Read - RND4KQ32T8 IOPS (MB/s)  | 180K (739)   | 663K (2590)   | 713K (2786)   | 709K (2902)    | 651K (2668)   | 644K (2639)   |
| Read - RND4KQ32T1 IOPS (MB/s)  | 180K (738)   | 217K (890)    | 202K (828)    | 201K (825)     | 217K (888)    | 214K (875)    |
| Read - RND4KQ1T1 IOPS (MB/s)   | 17.3K (70.9) | 7.90K (32.4)  | 7.60K (31.1)  | 7.29K (29.8)   | 8.70K (35.7)  | 8.91K (36.4)  |
| Write - RND4KQ32T8 IOPS (MB/s) | 169K (693)   | 613K (2513)   | 154K (632)    | 276K (1130)    | 58.3K (239)   | 65.4K (268)   |
| Write - RND4KQ32T1 IOPS (MB/s) | 173K (708)   | 225K (922)    | 91.7K (375)   | 143K (586)     | 68.5K (280)   | 69.7K (285)   |
| Write - RND4KQ1T1 IOPS (MB/s)  | 29.5K (121)  | 29.9K (122)   | 24.5K (100)   | 28.8K (118)    | 5.54K (22.7)  | 8.24K (33.7)  |

### RND4KQ32T8 write

Note the dramatic difference between RAID 5 and RAID 0 - we're at a measly 10% of the random write performance! Ouch!

| Array       | IOPS (1,000) | vs single disk | vs RAID 0 | vs RAID 10 |
| ----------- | ------------ | -------------- | --------- | ---------- |
| Single disk | 169          | 100%           | 28%       | 61%        |
| RAID 0      | 613          | 363%           | 100%      | 222%       |
| RAID 1      | 154          | 91%            | 25%       | 56%        |
| RAID 10     | 276          | 163%           | 45%       | 100%       |
| RAID 5      | 58.3         | 34%            | 10%       | 21%        |
| RAID 6      | 65.4         | 39%            | 11%       | 24%        |

### SEQ1MQ8T8 write

Note how RAID 1, 5 and 6 are limited to approximately the throughput of a single disk when writing.

| Array       | MB/s | vs single disk | vs RAID 0 | vs RAID 10 |
| ----------- | ---- | -------------- | --------- | ---------- |
| Single disk | 1123 | 100%           | 27%       | 68%        |
| RAID 0      | 4211 | 375%           | 100%      | 256%       |
| RAID 1      | 910  | 81%            | 22%       | 55%        |
| RAID 10     | 1648 | 147%           | 39%       | 100%       |
| RAID 5      | 985  | 88%            | 23%       | 60%        |
| RAID 6      | 951  | 85%            | 23%       | 58%        |
