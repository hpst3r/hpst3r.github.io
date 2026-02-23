---
title: "PVE/Ceph hyperconverged cluster build/Ceph performance tuning"
date: 2026-02-22T22:00:00-00:00
draft: false
---

## Introduction

This is a continuation of (and hopefully the final rebuild for) my "2025 Q4" cluster build project (yeah, I went a little past my deadline there).

Long story short, I've been looking to eliminate single points of failure in my infrastructure (at least, where they're easy to fix), so I wound up with three HP Z2 G4 SFF workstations (LGA 1151) and started trying to build a Proxmox (as I'm now responsible for supporting it professionally and do not have the time to figure out the perfect hypervisor That Is Not Proxmox) cluster out of them.

After several [attempts to use LINSTOR](https://wporter.org/failing-to-build-a-proxmox--linstor-drbd-hci-cluster-ft.-rdma-and-ewaste/), then attempting to use plain DRBD and a diskless third node (shared LVM atop a replicated block device - I didn't document this process, but it's rather straightforward), and suffering all the way thanks to DRBD-induced storage failures, I conceded defeat and moved on.

I'd really hoped I could find something that supported RDMA as my nodes only have six threads (Coffee Lake, no hyperthreading), but distributed storage is hard, and so there's not a whole lot of choice - LINSTOR is just about the only option that *does* support RDMA as far as I'm aware.

GlusterFS would have been my next stop rather than a small Ceph deployment, but it's going the way of the dodo and the storage plugin was removed in PVE 9 (following a decision made by QEMU, upstream). Sad! It had worked pretty well when I used it in the past, though performance wasn't incredible.

So, after being emboldened by a friend who was also running Ceph on consumer NVME (but did not show me performance numbers), I tried to deploy Ceph on the same QLC NVME SSDs (2x Intel 660p 512gb, 1x much nicer Samsung PM981a 512gb). This was a mistake, for reasons I will get into shortly. Some of you may know where this is going (the Ceph beginner's rite of passage). I knew better, but I was short on drives and I guess I like to frustrate myself (so I went ahead with this anyway).

It went about as one might expect. I got awful performance (ballpark 800 4K write IOPS at Q1 or Q32 without caching), and only about 6,000 IOPS with caching, or 9,500 with unsafe caching. And performance degraded terribly over a very short period.

So, I butchered two other servers to piece together six good 480g SATA SSDs (a mix of Intel DC S3500 and S4510) and chucked them in, instead. We'll go over this in a bit, but the end result was a perfectly usable Proxmox VE cluster that I've been wholly unable to break!

I've even done stuff like reboot all the nodes simultaneously, randomly take down 2 of 3 OSDs then reboot the host running the third, and start rm -rf'ing OSDs before data actually moved to the new ones. All my stuff is still there! Quite impressive for a distributed storage solution.

Performance is incredibly impressive considering the approximate total cost (~$100 in disk, purchased years ago, three ~$200 machines, and a $100 network switch). I'm looking forward to moving my production workloads over to this cluster and beating on it for a while!

Anyway, into the good stuff.

## Build log

This will be generally identical to the LINSTOR post, because I set all the machines up the same way. I need to rebuild my nodes again to clean them up after some more science so I'll probably link an Ansible playbook here eventually (for now you will have to make do).

### Hardware BoM

3x HP Z2 G4 SFF, nearly identical hardware:

- CPU: E-2126G (6c6t @ 4.5 GHz, Coffee Lake, like i5-8600K but higher 1C clocks and ECC support)
- Memory:
- 2x32gb DDR4 3200
- Disk:
- 120gb Intel D3-S3510 SATA SSD for boot
- Assorted 512gb NVME SSDs (1 per node) for data at 3.0x4, off CPU
- Networking:
- Broadcom BCM5720 dual port 1GBASE-T NIC (HP 332T) at 2.0x1, off chipset
- Dual-port QSFP (40/56GBe/QDR/FDR Infiniband) Mellanox ConnectX-3 Pro CX314A (flashed to CX354A) at 3.0x4, off chipset

My switching needs will be handled by a Mellanox SX6036 Infiniband/Ethernet FDR/QDR/40GBe VPI switch and a Cisco 3850-48T gigabit Ethernet switch.

The BCM5720s will be configured as a two-port bond, carrying management and cluster traffic. The CX3 Pros are connected to a Mellanox SX6036 in Ethernet mode for storage traffic and a backup Corosync link.

Eventually, I'll configure the built-in i219 for vPro remote management so I can use it as a poor man's BMC ([shameless plug to my post about vPro from a ways back](https://wporter.org/poking-at-the-intel-management-engine-vpro-enterprise-amt-for-ipmi-like-remote-access-on-a-m920q/)), but that's not a pressing need.

Network topology:

{{< figure src="images/0-cluster-basic-diagram-eth.png" >}}

### Addressing

- "z2g4a"
  - 172.27.2.10 (mgmt - on gigE bond)
  - 169.254.60.10 (corosync - on gigE bond)
  - 169.254.50.10 (storage - on CX3)
- "z2g4b"
  - 172.27.2.11 (mgmt - on gigE bond)
  - 169.254.60.11 (corosync - on gigE bond)
  - 169.254.50.11 (storage - on CX3)
- "z2g4c"
  - 172.27.2.12(mgmt - on gigE bond)
  - 169.254.60.12 (corosync - on gigE bond)
  - 169.254.50.12 (storage - on CX3)

## Initial base setup - Ethernet, PVE, odds and ends

### Repositories

Disable Ceph and PVE Enterprise, add PVE no-subscription and Ceph no-subscription:

```txt
root@z2g4a:~# cat /etc/apt/sources.list.d/proxmox.sources
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
root@z2g4a:~# cat /etc/apt/sources.list.d/pve-enterprise.sources
Types: deb
URIs: https://enterprise.proxmox.com/debian/pve
Suites: trixie
Components: pve-enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
Enabled: false
root@z2g4a:~# cat /etc/apt/sources.list.d/ceph.sources 
Types: deb
URIs: https://enterprise.proxmox.com/debian/ceph-squid
Suites: trixie
Components: enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
Enabled: false
```

## Updates

Fully updated the three nodes w/ restart.

### Certificates

Copy internal root cert over:

```sh
scp root-ca-a.crt root@proxmox:~/
ssh root@proxmox
mv root-ca-a.crt /usr/local/share/ca-certificates/
update-ca-certificates
```

### Configure systemd-journal-upload

This will get logs flowing to my VictoriaLogs server - useful in case I need to troubleshoot something right off the bat.

```sh
apt install -y systemd-journal-remote
tee /etc/systemd/journal-upload.conf > /dev/null << 'EOT'
[Upload]
# Compression = zstd:3 # not supported until systemd 258, n/a for alma 10
# NetworkTimeoutSec is unset
ServerCertificateFile = -
ServerKeyFile = -
TrustedCertificateFile = -
URL = https://victorialogs.lab.wporter.org:19532/insert/journald
EOT
mkdir -p /etc/systemd/system/systemd-journal-upload.service.d
tee /etc/systemd/system/systemd-journal-upload.service.d/override.conf > /dev/null << 'EOT'
[Service]
Restart=always
EOT
systemctl daemon-reload
systemctl enable --now systemd-journal-upload
```

### Configure Ethernet NICs (bridges and bond)

I labeled my NICs during setup. These are systemd links so you can certainly change them later.

I like to make any port on the system a member of one big ol' bond to reduce risk of future layer 8 errors, even if they're not in use, so I've included the built-in i219 NIC here.

Worst case, if I need to swap a NIC, I can easily plug in the built-in NIC to get the machine back on the network.

The ConnectX-3 is not set up as a bond - holdover from my old Infiniband setup, and it doesn't really matter. At some point I want to try to separate the public and internal Ceph networks so I didn't "fix" that (but it won't matter since I'm CPU limited and nowhere near the limits of 40 GbE).

```sh
address="12"
tee /etc/network/interfaces > /dev/null << EOT
auto lo
iface lo inet loopback

auto bcm5720p0
iface bcm5720p0 inet manual

auto bcm5720p1
iface bcm5720p1 inet manual

iface i219 inet manual

# connectx-3 pro port 1 - storage iface
auto cx354ap1
iface cx354ap1 inet static
    address 169.254.50.$address/24
    mtu 9000

# bcm5720 bond
auto bond0
iface bond0 inet manual
    bond-slaves bcm5720p0 bcm5720p1
    bond-miimon 100
    bond-mode 802.3ad

# main management address, untagged on bond
auto bridge0
iface bridge0 inet static
    address 172.27.2.$address/24
    gateway 172.27.2.254
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# corosync VLAN
auto bridge0.60
iface bridge0.60 inet static
    address 169.254.60.$address/24

source /etc/network/interfaces.d/*
EOT
systemctl restart networking
```

### Create the cluster

In the Proxmox VE web UI, on one of the nodes, navigate to Datacenter > Cluster > Create Cluster. Then, paste the encoded join information into the "join cluster" dialog on the other two nodes. Since my two networks are already up, I designated the 40 GbE network as a second Corosync ring here.

### Note on ACME

I did not bother configuring ACME. My DNS/PKI currently relies on Tailscale, and I don't want to put Tailscale on these nodes. If you *did* want to use ACME:

Configure pveproxy ACME account.

```sh
pvenode acme account register internal noc@wporter.org --directory https://intermediate-ca.lab.wporter.org/acme/acme/directory
```

```sh
pvenode config set --acme account=internal --acmedomain0 $(hostname -f)
```

Order your first cert:

```sh
pvenode acme cert order
```

Then, update the daily update timer so cert updates run every four hours if needed (I run 24h lifespan certs).

```sh
mkdir -p /etc/systemd/system/pve-daily-update.timer.d
tee /etc/systemd/system/pve-daily-update.timer.d/override.conf > /dev/null << 'EOT'
[Timer]
OnCalendar=
OnCalendar=00/4:00
RandomizedDelaySec=0
EOT
systemctl daemon-reload
```

### Live migration network

As with the old Infiniband setup, I chose to use my storage network for live migration (why not, Ceph can't use it all anyway). Cluster > Options > Migration settings:

{{< figure src="images/1-migration-settings.png" >}}

### HA Setting - Migrate when a node is restarted

Same deal as last time. Right under the "Migration Settings" is the option to set your VMs to migrate off a node when you tell it to reboot. I like this option!

{{< figure src="images/2-ha-settings.png" >}}

OK! That's about it for the cluster.. he's happy! Now, Ceph time!

## Intro to Ceph

Ceph is a clustered, distributed storage system that allows you to toss replicas of your data out over the network. Notably, Ceph scales very well and can be used for block (via the RADOS Block Device interface), file (with CephFS) or object storage (by default, accessible through the RADOS Gateway (RGW) which exposes a S3 compatible API).

Ceph has a few moving parts. I'll be doing a very brief summary of each.. not really going to use it here, but why not?

- **Monitors** keep track of data in Ceph clusters, manage quorum, and maintain "maps" of the cluster state (e.g., the CRUSH map).
- Ceph **managers** handle balancing load across the Ceph cluster and host the Ceph web UI.
- **Object Storage Daemons**, or OSDs, are processes running on storage servers that handle storing objects. Typically these correspond to single disks (so there is one daemon per disk - though this is not always the case).
- **Pools** are effectively volume groups - they contain storage volumes (OSDs) and are where you configure object protection (e.g., replication or erasure coding/parity).
- **Placement groups** are an abstraction that groups RADOS objects into subsets to simplify tracking, replication, balancing and recovery of objects (reducing monitor and manager overhead). For example, a placement group might contain *n* blocks and is addressed as a whole.
- The **Reliable Autonomic Distributed Object Store**, or RADOS, is the core software-defined-storage layer of Ceph.
- This "distributed object store" ***is*** Ceph, and is responsible for containing and managing all data in a cluster.
- RADOS storage may be consumed with aforementioned interfaces like the RADOS Gateway (object) or RADOS Block Device (block).
- **CRUSH** is an algorithm used by Ceph to compute how to store and retrieve data from the cluster; because Ceph clients implement this algorithm, they can determine where data is stored and retrieve it without calling home to a centralized service.
- **The CRUSH map** of a cluster contains OSDs, their locations (e.g., host, rack), and rules (how Ceph distributes data - e.g., failure domain host, or rack). It can be boiled down to "a set of parameters that govern how data is arranged in the cluster".
- By default, Ceph will automagically distribute data between failure domains (which are individual hosts).

## Ceph configuration w/ PVE

I will admit, I used the web UI for this. I am weak.

I will update this article with images when I get around to rebuilding the cluster one last time. For now:

To seed your initial Ceph configuration, navigate to a node, click Ceph, and click Install. Once you've downloaded the first Ceph packages, you can seed an initial configuration.

Once this is done, install Ceph on the other two nodes. Then, navigate to Node > Ceph > Monitor for each of the two other nodes and Create a Monitor and Manager on them - this will give us our three-node cluster.

{{< figure src="images/3-ceph-man-mon.png" >}}

To add OSDs, navigate to Node > Ceph > OSD > Create OSD. Specify an unused device (e.g., /dev/sdb), then click Create. Repeat for your other OSDs (I did this six times).

{{< figure src="images/4-ceph-osd.png" >}}

Finally, navigate to Ceph > Pools and create a pool.

{{< figure src="images/5-ceph-create-pool.png" >}}

The Size will be the number of copies distributed among your cluster. The Min. Size determines the smallest number of nodes you may have up while the pool still accepts I/O requests.

You do not want to lower this to 1 unless you know what you are doing; this would effectively allow your Ceph cluster to split-brain itself.

{{< figure src="images/6-ceph-create-pool.png" >}}

When done, click Create. Wait a few moments, and your ready-made Ceph cluster should be done! You'll see it as a new storage object on your hosts.

## Performance tweaks for Ceph

I ran through a small set of tweaks and basic benchmarks for Ceph before promoting this cluster to "production status". I'll be including the dismal results with the QLC SSDs, too. I ran `rados bench` occasionally but primarily relied on `kdiskmark` as an easy GUI frontend for some simple `fio` tests in a VM, as I felt this would be more representative of the workload.

**I will be discussing performance figures from this VM, with a QCOW-backed disk on my Ceph pool using the paravirtualized virtio-scsi-single driver, unless otherwise specified**.

The `virtio-scsi-single` driver has been shown to be the most performant option across the board by the good folks at Blockbridge. Note that I have also enabled writeback caching (NOT unsafe writeback caching), the separate `virtio-scsi` I/O thread, and SSD emulation. If any of these are missing, performance will be hampered.

Relevant bits of said VM's configuration:

```txt
root@z2g4a:~# qm config 100 | grep -E "agent|bios|cores|cpu|memory|numa|ostype|scsi0|scsihw|sockets"
agent: 1
bios: ovmf
cores: 4
cpu: host
memory: 8192
numa: 0
ostype: l26
scsi0: ceph:vm-100-disk-1,cache=writeback,iothread=1,size=32G,ssd=1
scsihw: virtio-scsi-single
sockets: 1
```

I am CPU-limited by the E-2126Gs in my hosts; they do not have enough threads for an optimal storage+guest configuration. I think the network and disks could do a bit better. The four cores I have servicing IRQs are typically pinned during storage I/O, and the system I'm running the benchmark on (the host) sees CPU busy% climb up to near 80%.

In every benchmark, Ceph is configured with a three-way replica (data is written to three OSDs). CRUSH pins these to three different nodes.

### Caching - Ceph and QEMU

> Quick aside: I'm going to talk about sync writes and flushes in this section. They're two related concepts.
>
> A **sync write** (e.g., application calling `fsync()`) is the app's way of saying "write this fully to disk" when it needs something written to persistent media.
> Flushes (FLUSH CACHE) are a SCSI/ATA/NVME command that the kernel then sends to the disk to tell it that it must write the data in its volatile cache (e.g., DRAM on disk) to persistent NAND (or the platter).
>
> After an application calls `fsync()`, the kernel will typically issue a flush cache command to the relevant disk (to meet the application's demand to write the data to persistent storage). Then, once the disk has drained cache to persistent media, the acknowledgement is sent back up through the storage stack to the application.
>
> These operations beat the crap out of consumer SSDs or hard disks because they must then use their slow, persistent media, not their fast cache (be it host memory with a modern DRAMless NVME, the fast RAM on the disk with a nicer SSD, or even SLC cache in a properly-behaved SSD - though some will acknowledge the write as soon as it hits SLC...)
>
> And this gets us into why power-loss prevention (PLP) is really important for fast, safe storage. Disks with PLP are important for 'enterprise' storage because they have supercapacitors that allow them to acknowledge writes as soon as they hit cache/DRAM (they can guarantee that they'll be able to write out the cache to persistent storage even if power to the system is lost). This dramatically improves performance versus writing random I/O to TLC or QLC NAND because, among other things, it allows the drive to buffer I/O operations (think ZFS TXG flushes, SLOG device..)
>
> And finally, you might ask, "why is Windows so fast on my Samsung 990 Pro Max Ultra without power-loss prevention?" Well, because Windows doesn't give two hoots about how much data your applications might lose if it crashes! Ceph and ZFS do!

Writeback caching is typically the best choice for production VM workloads. You should consider not using it if your power is unreliable and your systems do not have a battery backup, but this writeback (not marked unsafe) will NOT lie to the guest and accept sync writes/flushes without writing them to disk, so it is rather safe.

In my environment, properly configured writeback caching provides an 8.6x improvement in low queue depth write operations and about a 3x improvement in low queue depth reads (when the data we're trying to read is in cache).

To properly enable caching in a RBD/PVE environment, you must configure both the RBD cache in the `[client]` section of your Ceph configuration and writeback caching on the VM's disk (set the `cache=writeback` property on the disk, per disk on every VM).

To configure Ceph to use the cache, you must enable the RBD cache in the `[client]` section of the `/etc/pve/ceph.conf` configuration file:

```ini
[client]
keyring = /etc/pve/priv/$cluster.$name.keyring
rbd_cache = true
rbd_cache_writethrough_until_flush = true
```

The `rbd_cache_writethrough_until_flush` property allows us to keep caching enabled when our guests might not support flush operations (legacy OSes or misconfigured guests).

> Caution! If you're using RBD as backing storage for VM, have enabled RBD caching, and have NOT set `writethrough_until_flush = true`, you MUST enable writeback caching on your VMs.
>
> If you do not, the **VMs will never send flushes and Ceph will cache writes** in memory - this means everything, including important sync writes from your VMs, will not be flushed to disk as the guest might expect and **can cause data loss!**

By toggling `rbd_cache_writethrough_until_flush`, we default to safe write-through caching (effectively write to cache and disk simultaneously, just accelerating read operations with memory) until we learn that the guest supports flush operations (can tell us when it **needs** something properly written to disk) and thus that we can safely use a more aggressive caching strategy (writeback from cache to disk, force a commit to disk when the VM sends a flush request).

### QLC NVME vs MLC SATA with PLP

So, we've already talked about sync writes, flushes, and power-loss prevention.

The QLC NVME performs horribly - think 3 MB/s low queue depth random write - because Ceph makes heavy use of `fsync` calls to ensure data is actually written to disk. The disks don't have power-loss prevention, so they must flush to the extremely slow QLC NAND rather than cache.

The SATA drives had far more consistent performance (the NVMEs showed extremely "bursty" performance in benchmarks - ranging from 10 MB/s on run 1 to 90 MB/s on run 2 to 70 MB/s on run 3 in SEQ1MQ1T1 Write, and from 900 to 600 MB/s in SEQ1MQ8T1 Write). I noticed random 4K write was especially inconsistent - during the same benchmark (over 5 runs back-to-back), performance ranged from 7 MB/s to 35 MB/s for RND4KQ32T1 write. This is more of a QLC problem than anything else.

Additionally, with sustained I/O on the NVMEs, their performance quickly degraded. Take Q32T1 random 4K write IOPS in ten back-to-back runs (with a break between 7 and 8), for example:

| Run | RND4KQ32T1 write IOPS |
| --- | --------------------- |
| 1   | 11252.44              |
| 2   | 9433.59               |
| 3   | 9182.13               |
| 4   | 7741.70               |
| 5   | 6660.16               |
| 6   | 6093.75               |
| 7   | 5915.53               |
| 8   | 9777.83               |
| 9   | 681.15                |
| 10  | 722.66                |

Even with discard enabled on the OSDs (`bdev_enable_discard = true`) and VM disk, it took about a *day* for performance to clear up again (with no other I/O load on the systems).

### `tuned` profiles - CPU states and buffers

Since these systems are heavily CPU-limited and Ceph is latency-dependent, we see significant improvements from the `latency-perf` or `net-latency` `tuned` profiles. The most significant change here is locking the CPUs to C-states from which they may exit within a few (1-3) microseconds (eliminating power-saving functionality, but drastically reducing latency).

> In a homelab environment it might make sense to change `tuned` profiles to reflect solar output or I/O activity to minimize power consumption. I only have a few data points, but:
>
> I found the HP Z2 G4 E-2126G machines here use about 30w at idle in the default throughput-performance profile and about 50w with the latency-performance profile set.
>
> In another environment, setting the latency-performance profile on a R740xd (dual Cascade Lake Xeon Gold 6254, 768gb, 18 SSDs, already in maximum performance BIOS profile) increased nearly idle power consumption from 360w to 400w.

The system governor is set to `performance` by default, but if it was not, this change would likely make an improvement, too. Going further and forcefully locking the CPUs to `C0` state did not make any noticeable impact (vs the baseline `latency-performance` profile) so I am not doing this for my production deployment.

I found the `latency-performance` profile, when compared to `network-latency` (a child profile that tweaks buffers in addition to the tweaks made by `latency-performance`) performs marginally better (in terms of IOPS during random I/O) at the possible expense of a small amount of sequential read performance (margin of error), so I stuck with the former.

I briefly tested a Red Hat Ceph Storage-specific `tuned` profile [published by Red Hat](https://cloud.redhat.com/learning/learn:how-create-and-scale-6000-virtual-machines-7-hours-red-hat-openshift-virtualization/resource/resources:hour-1-network-tuning-red-hat-ceph-storage). This seems very similar to the `network-latency` built-in profile and I didn't see a notable improvement with that profile, or when I adjusted buffers to suit my connection (32 Gigabit theoretical maximum), and I also saw a significant regression in low queue depth sequential and random reads from my guest.

`latency-performance` got me about 100 MB/s of sequential read (~750 to ~850), Q8 (~50 with Q1), a marginal improvement in sequential write, and then big gains in random I/O: approximately 40 MB/s in Q32 random 4K read (1.2x improvement), 20 MB/s of Q32 random 4K write (1.5x improvement), and a whopping 2.9x improvement in Q1T1 4K read (from 5 MB/s to 15 MB/s).

### Kernels

I'll preface this: the stock kernel performed reasonably well; I would likely not go this far in a production environment unless random I/O was *imperative*.

I made no effort to measure the Xanmod tuned or realtime kernels' effect on other aspects of system performance; I believe I found that Xanmod reduced the system's maximum throughput when compared to a "clean" Fedora kernel in the past, so tuning for Ceph may harm other performance. In my case, I didn't care.

Also worthy of note is the fact that I'll be losing ZFS support by running a bleeding-edge Xanmod kernel. These sacrifices were OK by me in pursuit of more IOPS.

I swapped the Proxmox kernel (a manglement of the Ubuntu kernel with ZFS support and some minor tweaks backported to Debian) for Xanmod, a high-performance kernel distribution built with a lot of optimizations for x86_64 and heavy workloads (lots of I/O, etc) that is nicely packaged for Debian systems.

The Xanmod kernel was worth approximately 12.5% more Q32 4K read IOPS (40,000 to 45,000) and 15% more Q32 4K write IOPS (17,500 to 20,000), in addition to slight bumps in sequential I/O (and seemingly improved consistency). Low queue-depth I/O improved too - reads by about 8% (3600 to 3900 IOPS) and writes by about 14% (6900 to 7800 IOPS). All in all, a healthy improvement.

I also tried out the Xanmod realtime kernel, but I found that it (the Xanmod realtime build) was not an improvement. Low queue depth random 4K operations were within margin of error; sequential writes (somewhat buffered) were within margin of error, and the realtime Xanmod kernel did seem to perform markedly worse in RND4KQ32T1 write (-7,500 IOPS - compared to the non-realtime Xanmod v3 kernel). In addition, the realtime kernel was marginally worse in both moderate and low queue depth sequential read (SEQ1MQ8T1 approx -60 MB/s; ~840 to ~780 MB/s, SEQ1MQ1T1 approx. -25 MB/s; ~285 to ~260 MB/s).

### NIC settings

I found that slightly bumping the RX/TX buffers to 2048 (from 1024) on my ConnectX-3 Pro cards gave me a modest improvement in throughput without impacting latency enough to make an impact on random I/O. Going any further (4096) caused a negative impact on throughput and a marginal negative impact on random I/O.

Moving the ConnectX-3 Pro cards from a chipset-attached PCI-E 3.0 x4 slot (maximum theoretical 32 Gbps, via x4 VPI link shared with other components including the dual-port Gigabit NIC) to a CPU-attached PCI-E 3.0 x16 slot (at x8) seemed to slightly improve throughput. Curiously, there was no measurable impact on random I/O. This is good news for me - I will likely be moving the NICs back to the chipset slot to free up the x16s for accelerators (probably Tesla P4s) at some point in the future.

Increasing the amount of RX/TX queues from 4 to 6 did not provide a measurable improvement. As such, I left the queues at their defaults.

I did not attempt to manually balance IRQs. They were distributed between four of my six cores out of the box; I saw fit to leave this as-is.

### Ceph settings

I was able to squeeze a little more performance out of my cluster (marginal improvement in sequential read - about 100 MB/s, from 870 to 970 MB/s - and random read) by increasing the memory allocation per OSD to 8 GiB.

Additionally, since I'm using SSDs in my pool, setting the `discard` option for the OSDs is important to maintain performance over time (this tells Ceph that it should let the disks know when they can release "used space" without a reformat).

To set both of these, edit the `[osd]` section of the Ceph configuration (the central config at `/etc/pve/ceph.conf`) then restart the `ceph-osd.target` on each node (do a rolling restart).

```ini
[osd]
osd_memory_target = 8589934592
bdev_enable_discard = true
```

### Final performance results

Here are some of my test results (the ones I bothered to write down). I usually ran these more than once; each 'run' is five runs averaged.

| Tweaks | SEQ1MQ8T1 Read (MB/s) | SEQ1MQ1T1 Read (MB/s) | SEQ1MQ8T1 Write (MB/s) | SEQ1MQ1T1 Write (MB/s) | RND4KQ32T1 Read (MB/s) | RND4KQ1T1 Read (MB/s) | RND4KQ32T1 Write (MB/s) | RND4KQ1T1 Write (MB/s) | Q32T1 4K Write IOPS | Q32T1 4K Read IOPS |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **QLC NVME, 6.17.9-1-pve** | | | | | | | | | | |
| No cache, SSD emu, rx/tx1024 | 403.79 | 336.61 | 46.78 | 52.88 | 6.14 | 4.17 | 0.57 | 0.63 | 139 | 1499 |
| Writeback, SSD emu, rx/tx1024 | 595.11 | 472.51 | 132.34 | 14.63 | 6.19 | 5.38 | 27.08 | 12.89 | 6611 | 1511 |
| Writeback (unsafe), SSD emu, rx/tx1024 | 425.74 | 380.64 | 265.26 | 146.35 | 54.63 | 4.05 | 38.54 | 28.99 | 9409 | 13337 |
| **MLC SATA w/ PLP, 6.17.9-1-pve** | | | | | | | | | | |
| Writeback | 228.79 | 235.52 | 508.48 | 517.82 | 5.17 | 4.92 | 27.54 | 28.34 | 6724 | 1262 |
| No cache, SSD emu, rx/tx1024 | 725.91 | 227.12 | 362.73 | 172.62 | 132.28 | 5.02 | 45.94 | 3.28 | 11216 | 32295 |
| Writeback, SSD emu, rx/tx1024 | 756.36 | 224.80 | 520.42 | 506.10 | 136.69 | 5.03 | 48.19 | 28.26 | 11765 | 33372 |
| WB, SSDE, net-latency, rx/tx1024 | 876.43 | 288.40 | 533.69 | 526.96 | 149.82 | 14.54 | 67.40 | 24.93 | 16455 | 36577 |
| WB, SSDE, latency-perf, rx/tx1024 | 860.82 | 283.06 | 537.94 | 527.35 | 163.01 | 14.56 | 71.77 | 28.29 | 17522 | 39797 |
| WB, SSDE, latency-perf, nocstate, rx/tx1024 | 852.43 | 286.22 | 533.12 | 522.19 | 163.14 | 14.97 | 71.32 | 28.30 | 17412 | 39829 |
| **MLC SATA w/ PLP, 6.18.8-x64v3-xanmod1** | | | | | | | | | | |
| WB, SSDE, latency-perf, rx/tx1024 | 841.95 | 284.42 | 551.25 | 523.97 | 182.29 | 15.59 | 81.40 | 32.51 | 19873 | 44504 |
| WB, SSDE, latency-perf, rx/tx2048 | 871.31 | 286.77 | 571.76 | 552.80 | 181.90 | 15.93 | 81.06 | 31.96 | 19790 | 44409 |
| WB, SSDE, latency-perf, rx/tx4096 | 830.51 | 286.58 | 567.98 | 558.63 | 182.62 | 16.05 | 78.60 | 31.86 | 19189 | 44585 |
| WB, SSDE, rhcs tuned, sysbuf, rx/tx2048 | 826.54 | 251.37 | 552.67 | 535.42 | 163.37 | 5.44 | 71.37 | 27.97 | 17424 | 39885 |
| WB, SSDE, latency-perf, rx/tx1024, nic off cpu | 880.53 | 293.79 | 573.84 | 580.40 | 148.64 | 16.85 | 29.53 | 31.78 | 7209 | 36289 |
| WB, SSDE, latency-perf, rx/tx2048, nic off cpu | 869.45 | 292.54 | 585.27 | 577.70 | 177.58 | 15.88 | 78.47 | 31.59 | 19158 | 43354 |
| **MLC SATA w/ PLP, 6.18.7-rt-x64v3-xanmod1** | | | | | | | | | | |
| WB, SSDE, latency-perf | 780.69 | 261.03 | 567.23 | 559.01 | 160.53 | 15.74 | 51.50 | 28.95 | 12573 | 39192 |

## Conclusion

Ceph was surprisingly painless! So far, I'm pretty happy with it, but I have a good track record of managing to obliterate Proxmox clusters (so we'll see how long it lasts).
