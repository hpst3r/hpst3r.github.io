---
title: "Failing to build a Proxmox + LINSTOR (DRBD) HCI cluster (ft. RDMA and ewaste!)"
date: 2026-02-02T22:00:00-00:00
draft: false
---

## Introduction

I had a machine go down the other day. It was my sole "production" hypervisor (an old crappy Optiplex) running all my stuff. Yes, my redundant DNS... on one machine.

So I set about fixing it. I did some reading on distributed storage (because who do you think I am? I'm not going to waste machines on making my own SAN.. at least, I won't if I have to pay for power), scrounged through the drawers for RAM and SSDs, ordered some modest hardware, and here we are. This is (supposedly) my new production hypervisor cluster!

**TL;DR?** Didn't work - LINSTOR is far too fragile. Will be back for round 2. Figured I'd post this anyway.

## Hardware BoM

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

## Software

- Proxmox VE 9 (Debian 13, "no more analysis paralysis" choice, not perfect)
  - The closest thing to ESXi with good software RAID support and no VCSA. Plus, PBS is EXCELLENT!
- LINSTOR/DRBD for storage
  - Theoretically the perfect choice. Places=2, intended for small clusters, DRBD has been around for a while, low overhead. Simpler than Ceph, likely more performant, too.

## Usage

20 - 40 Linux VMs (varies widely)

5 - 15 Windows VMs (varies widely, generally just for lab use)

Some VMs need to be highly available (DNS, primary AD DS, VPN servers, some app servers). Some do not.

This is not my only cluster/hosting, so anything that needs availability will have service-level HA to another cluster. Unfortunately power and network are not redundant, but I'll work with what I've got. I do have the ability to locate machines at other sites, which I may do as time permits.

The BCM5720s will be configured as a two-port bond, carrying management and cluster traffic. The CX3 Pros are connected to a Mellanox SX6036 in Infiniband mode for storage traffic and a backup Corosync/LINSTOR link (IPoIB).

The network topology is something like this:

{{< figure src="images/0-cluster-basic-diagram.png" >}}

## Addressing

And the topology is a perfect segue to our addressing!

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

Disable Ceph and PVE Enterprise, add PVE no-subscription:

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

### Tailscale

My DNS currently relies on Tailscale, so I've installed it here.

### Certificates

Copy internal root cert over:

```sh
scp root-ca-a.crt root@proxmox:~/
ssh root@proxmox
mv root-ca-a.crt /usr/local/share/ca-certificates/
update-ca-certificates
```

### Configure systemd-journal-upload

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

```sh
address="12"
tee /etc/network/interfaces > /dev/null << EOT
auto lo
iface lo inet loopback

auto bcm5720p0
iface bcm5720p0 inet manual

auto bcm5720p1
iface bcm5720p1 inet manual

auto i219
iface i219 inet manual

auto bond0
iface bond0 inet manual
  bond-slaves bcm5720p0 bcm5720p1 i219
  bond-miimon 100
  bond-mode 802.3ad
  bond-xmit-hash-policy layer2+3

auto bridge0
iface bridge0 inet static
  address 172.27.2.$address/24
  gateway 172.27.2.254
  bridge-ports bond0
  bridge-stp off
  bridge-fd 0
  bridge-vlan-aware yes
  bridge-vids 2-4094
#bridge on gigE ports

auto bridge0.60
iface bridge0.60 inet static
  address 169.254.60.$address/24
#corosync

source /etc/network/interfaces.d/*
EOT
systemctl restart networking
```

### Join the nodes to a PVE cluster

In the Proxmox VE web UI, on one of the nodes, navigate to Datacenter > Cluster > Create Cluster. For the time being, I've just selected the VLAN60 bridge interface (bridge0.60, 169.254.60.0/24) as my sole cluster link - once we have IB working, we can edit the corosync configuration to add a redundant link.

Once the cluster is made, copy the join information (Datacenter > Cluster > Join Information) and join the other two nodes (Datacenter > Cluster > Join Cluster).

### Configure ACME

Configure pveproxy ACME account.

```sh
pvenode acme account register internal noc@wporter.org --directory https://intermediate-ca.lab.wporter.org/acme/acme/directory
```

```sh
pvenode config set --acme account=internal --acmedomain0 $(hostname -f)
```

Order a first cert:

```sh
pvenode acme cert order
```

Then, update the daily update timer so cert updates run every four hours (I run 24h lifespan certs).

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

## Networking - Infiniband

### Configure Infiniband

I'll be building an older version of mstflint from source, just to have.

```sh
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

Additonally, you'll need a few packages for Infiniband operation:

```sh
apt install -y infiniband-diags ibutils rdma-core rdmacm-utils ibverbs-utils
```

Once these are installed, restart the node. Then, try an `ibping`.

You will need to run an ibping server with `ibping -S` on one node, then try to reach it by LID or GUID.

By LID:

```txt
root@z2g4c:~# ibping -L 5 -c1
Pong from z2g4b.(none) (Lid 5): time 0.065 ms

--- z2g4b.(none) (Lid 5) ibping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 1000 ms
rtt min/avg/max = 0.065/0.065/0.065 ms
```

By GUID:

```txt
root@z2g4c:~# ibping -G 0x0200000000001112 -c1
Pong from z2g4b.(none) (Lid 5): time 0.050 ms

--- z2g4b.(none) (Lid 5) ibping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 1000 ms
rtt min/avg/max = 0.050/0.050/0.050 ms
```

### Configuring IPoIB

We'll now configure IP over Infiniband (IPoIB) addressing so we can run Corosync and live migrations over the Infiniband link (as a backup). Alternatively, one could use the second port on a VPI HCA in Ethernet mode for this (that might be a better idea, but I'm limited to 32 Gbps by my PCI-E interface anyway).

IPoIB performs worse than native Ethernet (I believe it's only slightly worse than non-RDMA 40GBe) and much worse than native Infiniband, but allows us to run pure RDMA traffic (DRBD replication) and IP applications that don't support RDMA (like Corosync and Proxmox live migration) side by side.

To nicely rename the IPoIB devices (their names change quite often at reboot), generate link files (the MAC is the GID for the IB device), e.g.:

```sh
tee /usr/local/lib/systemd/network/50-cx354ap0.link > /dev/null << EOT
[Match]
MACAddress=80:00:02:08:fe:80:00:00:00:00:00:00:02:00:00:00:00:00:11:11

[Link]
Name=cx354ap0
EOT
tee /usr/local/lib/systemd/network/50-cx354ap1.link > /dev/null << EOT
[Match]
MACAddress=80:00:02:08:fe:80:00:00:00:00:00:00:02:00:00:00:00:00:11:12

[Link]
Name=cx354ap1
EOT
```

Then reboot.

Configure addressing in `/etc/network/interfaces`:

```txt
auto cx354ap1
iface cx354ap1 inet manual
  address 169.254.50.12/24
```

Restart networking:

```sh
systemctl restart networking
```

Test an IP ping over Infiniband:

```txt
root@z2g4b:~# ping 169.254.50.10 -c1
PING 169.254.50.10 (169.254.50.10) 56(84) bytes of data.
64 bytes from 169.254.50.10: icmp_seq=1 ttl=64 time=0.411 ms

--- 169.254.50.10 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.411/0.411/0.411/0.000 ms

root@z2g4b:~# ping 169.254.50.12 -c1
PING 169.254.50.12 (169.254.50.12) 56(84) bytes of data.
64 bytes from 169.254.50.12: icmp_seq=1 ttl=64 time=0.484 ms

--- 169.254.50.12 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.484/0.484/0.484/0.000 ms
```

### Configure Corosync to use IPoIB interfaces as a redundant ring

In my environment, Ethernet and Infiniband switching for the main access bond and storage network are on separate switches and separate NICs, so this will give the cluster a bit of redundancy and protect against split brain in case a switch goes down (though we do have three nodes, so here the split-brain concern is moot unless I later decide to shut nodes down to save power).

To add the IPoIB interfaces as a second ring, edit the corosync configuration (`/etc/pve/corosync.conf`, NOT `/etc/corosync/corosync.conf` - the former will overwrite the latter), adding the IPoIB addresses (per node) as `ringX_addr` in `nodelist` and adding a new `totem { interface }` section, e.g.:

Before:

```txt
logging {
  debug: off
  to_syslog: yes
}

nodelist {
  node {
    name: z2g4a
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 169.254.60.10
  }
  node {
    name: z2g4b
    nodeid: 2
    quorum_votes: 1
    ring0_addr: 169.254.60.11
  }
  node {
    name: z2g4c
    nodeid: 3
    quorum_votes: 1
    ring0_addr: 169.254.60.12
  }
}

quorum {
  provider: corosync_votequorum
}

totem {
  cluster_name: pve0
  config_version: 3
  interface {
    linknumber: 0
  }
  ip_version: ipv4-6
  link_mode: passive
  secauth: on
  version: 2
}
```

After:

```txt
logging {
  debug: off
  to_syslog: yes
}

nodelist {
  node {
    name: z2g4a
    nodeid: 1
    quorum_votes: 1
    ring0_addr: 169.254.60.10
    ring1_addr: 169.254.50.10
  }
  node {
    name: z2g4b
    nodeid: 2
    quorum_votes: 1
    ring0_addr: 169.254.60.11
    ring1_addr: 169.254.50.11
  }
  node {
    name: z2g4c
    nodeid: 3
    quorum_votes: 1
    ring0_addr: 169.254.60.12
    ring1_addr: 169.254.50.12
  }
}

quorum {
  provider: corosync_votequorum
}

totem {
  cluster_name: pve0
  config_version: 3
  interface {
    linknumber: 0
  }
  interface {
    linknumber: 1
  }
  ip_version: ipv4-6
  link_mode: passive
  secauth: on
  version: 2
}
```

Changes should be applied automatically.

### Use IPoIB for guest migration

Set the Datacenter > Options > Migration Settings to the IPoIB link local network to use it for guest live migration.

{{< figure src="images/1-migration-settings.png" >}}

### Unrelated - set HA setting to Migrate

While we're here, set the Datacenter > Options > HA Settings to Migrate guests off a node when it's shut down or restarted:

{{< figure src="images/2-ha-settings.png" >}}

## LINSTOR

We can finally get to the storage! Yay.

We'll be configuring LINSTOR and DRBD Reactor to get a highly-available SDN solution a la Ceph or S2D, but at a smaller scale (though LINSTOR does make things markedly complex).

DRBD Reactor will allow us to fail the LINSTOR controller over between nodes.

There's a LINSTOR plugin for Proxmox VE that lets us easily manage LINSTOR-managed DRBD volumes from Proxmox VE's web UI, so that's nice.

Why not ZFS replication? I want realtime HA, not after-the-fact replication, and am willing to accept additional complexity.

Why not Ceph? Few drives, somewhat (not really) limited resources, and everyone else uses Ceph!

DRBD 9 natively supports RDMA as a transport.

### LINSTOR basics

**Resources** are storage objects presented to clients, like DRBD volumes presented to Proxmox VE as highly available shared storage. Think of them as LVMs.

**Volumes** are subsets of a resource that generally correspond to a storage device. A resource may be composed of many volumes. Think of them as LVM VGs.

**Storage pools** back volumes - think of them as LVM PVs. For example, you might create a storage pool of type lvmthin on two nodes, and use both for a redundant volume.

**Nodes** are indiviudal machines in the LINSTOR cluster. They may be a controller, satellite, combined, or auxilary. For our purposes, all of our nodes will be combined, as they'll all have both the **controller** (orchestrator/SDS controller) and **satellite** (storage interface/SDS endpoint) installed.

### DRBD basics

LINSTOR is capable of managing many different types of storage, but in our case, we'll only be using DRBD. DRBD ("Distributed Replicated Block Device") is, as its name suggests, a way to replicate block devices over the network. Since DRBD 9, it scales up to ~32 nodes (theoretically) but performs best in smaller configurations (evaluate Ceph at 5 - 9+ nodes). Traditionally, DRBD only allowed you to do a sort of networked RAID-1, but nowadays DRBD 9 supports places=2 like Ceph (we will be using this).

Replication over the network in this deployment will be Protocol C only, which is strictly synchronous (unlike Ceph, which is less safe). Think ZFS - a sync write will be OK'd only when it is written to disk on both the local node and the remote node.

This has some performance implications, particularly as we're using consumer-grade SSDs that lack power-loss prevention. I will get into this later.

### Configuring storage

Before we get into LINSTOR and DRBD, let's prep our storage.

My configuration will be a 512gb NVME PV per node, using lvm-thin. I'll simply make the entire NVME a PV, and use 95% of the disk for the thin pool (saving the rest for metadata).

```sh
pvcreate /dev/nvme0n1 && vgcreate linstor /dev/nvme0n1 && lvcreate -l 95%FREE -T linstor/thinpool
```

### Installation

```sh
wget https://packages.linbit.com/public/linbit-keyring.deb

dpkg -i linbit-keyring.deb

PVERS=9 && echo "deb [signed-by=/etc/apt/trusted.gpg.d/linbit-keyring.gpg] \
    http://packages.linbit.com/public/ proxmox-$PVERS drbd-9" \
    > /etc/apt/sources.list.d/linbit.list apt update

apt update

# install prerequisites
apt install -y proxmox-default-headers drbd-dkms drbd-utils

# install linstor & proxmox ve plugin
apt install -y linstor-controller linstor-satellite linstor-client linstor-proxmox

# optionally, grab the linstor-gui package too

# enable linstor satellite
systemctl enable linstor-satellite --now

# enable linstor controller only on one node once installed
systemctl disable linstor-controller --now
```

### Initial configuration - one controller

To start, we'll work with a single LINSTOR controller. I'll be putting mine on node "z2g4a".

Enable and start the controller:

```sh
systemctl enable --now linstor-controller
```

Confirm it responds:

```txt
root@z2g4a:~# linstor controller version
linstor controller 1.33.1; GIT-hash: 95da7940d6efb6a39ea303c5f37b03478a6fab0b
```

Then, we'll add nodes to the cluster.

Note that I am NOT using SSL, and for now, I'll just be using the gigabit management link.

```sh
linstor node create --node-type combined --communication-type plain z2g4a 169.254.60.10
linstor node create --node-type combined --communication-type plain z2g4b 169.254.60.11
linstor node create --node-type combined --communication-type plain z2g4c 169.254.60.12
```

Run `linstor node list` to confirm your nodes are online:

```txt
root@z2g4a:~# linstor node list
╭────────────────────────────────────────────────────────╮
┊ Node  ┊ NodeType ┊ Addresses                  ┊ State  ┊
╞════════════════════════════════════════════════════════╡
┊ z2g4a ┊ COMBINED ┊ 169.254.60.10:3366 (PLAIN) ┊ Online ┊
┊ z2g4b ┊ COMBINED ┊ 169.254.60.11:3366 (PLAIN) ┊ Online ┊
┊ z2g4c ┊ COMBINED ┊ 169.254.60.12:3366 (PLAIN) ┊ Online ┊
╰────────────────────────────────────────────────────────╯
```

Configure a storage pool, using the `linstor/thinpool` LVM on all three nodes:

```sh
linstor storage-pool create lvmthin z2g4a thinpool linstor/thinpool
linstor storage-pool create lvmthin z2g4b thinpool linstor/thinpool
linstor storage-pool create lvmthin z2g4c thinpool linstor/thinpool
```

List storage pools to confirm that you have three:

```txt
root@z2g4a:~# linstor sp list
╭───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ StoragePool          ┊ Node  ┊ Driver   ┊ PoolName         ┊ FreeCapacity ┊ TotalCapacity ┊ CanSnapshots ┊ State ┊ SharedName                 ┊
╞═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ DfltDisklessStorPool ┊ z2g4a ┊ DISKLESS ┊                  ┊              ┊               ┊ False        ┊ Ok    ┊ z2g4a;DfltDisklessStorPool ┊
┊ DfltDisklessStorPool ┊ z2g4b ┊ DISKLESS ┊                  ┊              ┊               ┊ False        ┊ Ok    ┊ z2g4b;DfltDisklessStorPool ┊
┊ DfltDisklessStorPool ┊ z2g4c ┊ DISKLESS ┊                  ┊              ┊               ┊ False        ┊ Ok    ┊ z2g4c;DfltDisklessStorPool ┊
┊ thinpool             ┊ z2g4a ┊ LVM_THIN ┊ linstor/thinpool ┊   452.86 GiB ┊    452.86 GiB ┊ True         ┊ Ok    ┊ z2g4a;thinpool             ┊
┊ thinpool             ┊ z2g4b ┊ LVM_THIN ┊ linstor/thinpool ┊   452.86 GiB ┊    452.86 GiB ┊ True         ┊ Ok    ┊ z2g4b;thinpool             ┊
┊ thinpool             ┊ z2g4c ┊ LVM_THIN ┊ linstor/thinpool ┊   452.86 GiB ┊    452.86 GiB ┊ True         ┊ Ok    ┊ z2g4c;thinpool             ┊
╰───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

Then, create a resource group (consuming the storage pools). Note that LINSTOR will implicitly use *all* storage pools with the same name as backing storage for the resource group by default.

```sh
linstor resource-group create -d "lvmthin places=2 pool for VM storage" --storage-pool thinpool --place-count 2 drbdthinpool --diskless-on-remaining
```

Edit your `/etc/pve/storage.cfg` to map the LINSTOR pool as a volume, so we can see the results of our handiwork:

```yaml
drbd: linstor-pool
        controller 169.254.60.10
        resourcegroup drbdthinpool
        content rootdir,images
```

{{< figure src="images/3-linstor-pool.png" >}}

Then, create new LINSTOR interfaces on the Infiniband fabric, and set the resource group for the pool to use them:

```sh
linstor node interface create z2g4a infiniband 169.254.60.10
linstor node interface create z2g4b infiniband 169.254.60.11
linstor node interface create z2g4c infiniband 169.254.60.12
linstor resource-group set-property drbdthinpool PrefNic infiniband
linstor resource-group set-property drbdthinpool DrbdOptions/Net/transport rdma
```

> DO NOT CHANGE THE TRANSPORT TYPE WHILE THINGS ARE RUNNING ON YOUR HOSTS.

A restart of the nodes or LINSTOR services may be required - without restarting, the LINSTOR PVE plugin was giving me "storage does not exist" errors when attempting to start a test VM.

### Highly-available controller

See the docs [(the LINSTOR user's guide).](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/#s-linstor_ha)

To run a highly-available LINSTOR cluster, we need to replicate the database. To do so conveniently, we can use DRBD, with DRBD Reactor to orchestrate failover.

Create a resource group for the database:

```sh
linstor resource-group drbd-options --auto-promote=no --quorum=majority --on-suspended-primary-outdated=force-secondary --on-no-quorum=io-error --on-no-data-accessible=io-error linstor-db
```

Create a volume group:

```sh
linstor volume-group create linstor-db
```

Disable the LINSTOR controller so we can move the database:

```sh
systemctl disable --now linstor-controller
```

Seed a systemd mount unit - DRBD Reactor will be used to mount this when the node becomes Primary:

```sh
tee /etc/systemd/system/var-lib-linstor.mount > /dev/null << 'EOT'
[Unit]
Description=Filesystem for the LINSTOR controller

[Mount]
# you can use the minor like /dev/drbdX or the udev symlink
What=/dev/drbd/by-res/linstor_db/0
Where=/var/lib/linstor
EOT
```

Then, move /var/lib/linstor (to have a copy of the old contents) and create an empty directory for use as a mount point:

```sh
mv /var/lib/linstor{,.orig}
mkdir /var/lib/linstor
```

Protect `/var/lib/linstor` from accidental deletion:

```sh
chattr +i /var/lib/linstor # only if on LINSTOR >= 1.14.0
```

Promote the node to the DRBD primary for the database volume, so we can mount it:

```sh
drbdadm primary linstor_db
```

Format the database volume:

```sh
mkfs.ext4 /dev/drbd/by-res/linstor_db/0
```

Mount the DRBD-backed volume:

```sh
systemctl start var-lib-linstor.mount
```

Restore the controller database to the new volume:

```sh
cp -r /var/lib/linstor.orig/* /var/lib/linstor
```

Finally, restart the controller, now using the DRBD-backed volume for its database.

```sh
systemctl start linstor-controller
```

```txt
root@z2g4a:~# sudo tee /etc/systemd/system/var-lib-linstor.mount > /dev/null << 'EOT'
[Unit]
Description=Filesystem for the LINSTOR controller

[Mount]
# you can use the minor like /dev/drbdX or the udev symlink
What=/dev/drbd/by-res/linstor_db/0
Where=/var/lib/linstor
EOT
root@z2g4a:~# ls /var/lib/linstor
linstordb.mv.db  linstordb.trace.db
root@z2g4a:~# mv /var/lib/linstor{,.orig}
root@z2g4a:~# ls /var/lib/linstor
ls: cannot access '/var/lib/linstor': No such file or directory
root@z2g4a:~# mkdir /var/lib/linstor
root@z2g4a:~# chattr +i /var/lib/linstor
root@z2g4a:~# drbdadm primary linstor_db
root@z2g4a:~# mkfs.ext4 /dev/drbd/by-res/linstor_db/0
mke2fs 1.47.2 (1-Jan-2025)
Discarding device blocks: done
Creating filesystem with 208812 1k blocks and 52208 inodes
Filesystem UUID: 4bffc78a-4f26-4a5e-aa86-205b2711dec2
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729, 204801

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

root@z2g4a:~# systemctl start var-lib-linstor.mount
root@z2g4a:~# cp -r /var/lib/linstor.orig/* /var/lib/linstor
root@z2g4a:~# systemctl start linstor-controller
```

Then, seed the `/var/lib/linstor` mount unit to the other standby controllers:

```sh
tee /etc/systemd/system/var-lib-linstor.mount > /dev/null << 'EOT'
[Unit]
Description=Filesystem for the LINSTOR controller

[Mount]
# you can use the minor like /dev/drbdX or the udev symlink
What=/dev/drbd/by-res/linstor_db/0
Where=/var/lib/linstor
EOT
```

Do NOT yet start the LINSTOR controller; DRBD Reactor will be used to do this.

If the LINSTOR database already exists on your new standby controllers, remove its contents. The replicated volume will fail over into place here, so we don't want whatever was originally there to be in play.

```sh
rm -rf /var/lib/linstor/*
```

#### Configuring DRBD Reactor

We will use DRBD Reactor to manage promotion and failover. This is the recommended configuration from LINBIT.

DRBD Reactor is a high availability management utility for DRBD resources that integrates tightly with DRBD to "react" or take actions (e.g., promote a LINSTOR controller) in response to DRBD device state changes (e.g., a node goes offline).

DRBD Reactor's function can be summarized as:

- If a resource can be promoted (e.g., not active elsewhere and we have quorum), promote it, and perform the specified actions (usually meaning start systemd units).
- If quorum is lost, stop services and demote the resource. Optionally, do something to fence the node in case of failure.

It is substantially simpler than the Pacemaker stack, which is the whole point.

In this case, we'll use DRBD Reactor to mount the /var/lib/linstor DRBD volume and start the LINSTOR controller when a node is promoted to the primary for the `linstor_db` DRBD resource.

Install Reactor on all nodes that will be running the controller:

```sh
apt install -y drbd-reactor
```

Configure DRBD Reactor with a promotion action (mount the DRBD-backed volume and start the LINSTOR controller):

```sh
tee /etc/drbd-reactor.d/linstor_db.toml > /dev/null << 'EOT'
[[promoter]]
[promoter.resources.linstor_db]
start = ["var-lib-linstor.mount", "linstor-controller.service"]
EOT
```

Then, start the service:

```sh
systemctl restart drbd-reactor
systemctl enable drbd-reactor
```

Set the LINSTOR Satellite service to not overwrite the database at launch:

```sh
# this **SHOULD** no longer be needed as of
# LINSTOR 1.33.0-rc.1 - 2025-11-11
mkdir -p /etc/systemd/system/linstor-satellite.service.d
tee /etc/systemd/system/linstor-satellite.service.d/override.conf > /dev/null << 'EOT'
[Service]
Environment=LS_KEEP_RES=linstor_db
EOT
```

In your `/etc/pve/storage.cfg` (on the pmxcfs, so only edit this once), add the additional controller addresses:

```txt
drbd: linstor-pool
        controller 172.27.2.10,172.27.2.11,172.27.2.12
        resourcegroup pve-lvm
        content rootdir,images
```

To test failover, take down the currently active LINSTOR controller and both that DRBD Reactor migrated the service (`drbd-reactorctl status`) and confirm that LINSTOR is still functional (e.g., run `linstor controller version` with `--controllers {all nodes}`):

```txt
root@z2g4a:~# drbd-reactorctl status
/etc/drbd-reactor.d/linstor_db.toml:
Promoter: Currently active on this node
● drbd-services@linstor_db.target
● ├─ drbd-promote@linstor_db.service
● ├─ var-lib-linstor.mount
● └─ linstor-controller.service
```

```txt
root@z2g4a:~# linstor --controllers 172.27.2.10,172.27.2.11,172.27.2.12 controller version
linstor controller 1.33.1; GIT-hash: 95da7940d6efb6a39ea303c5f37b03478a6fab0b
```

The Proxmox VE plugin will try all provided controllers, and LINSTOR should now survive node failures.

## Why I dropped LINSTOR

It was at this point that my LINSTOR controller exploded (for the third time) - just after finishing a clean rebuild and starting to migrate VMs over.

After something (God knows what) happened, all of my LINSTOR resources got dumped into "Disconnecting/Connecting" state, some of them got stuck in I/O error lock, and a bunch of leftover LVMs got stuck under DRBD volumes I couldn't remove.

I was no longer able to migrate or create VMs, and then "start" operations started failing, too:

{{< figure src="images/4-linstor-death.png" >}}

Like I said, this is the third time this happened to me (I rebuilt the cluster the first time, and cleaned everything up, starting from scratch, the second time). So, I gave up. I can't rely on something so fragile.

When LINSTOR was working properly, it was great! Excellent performance (from a VM, I was seeing 1500+ MB/s seq read/write, 50 - 70 MB/s random 4K write on a single QLC DRAMless SSD per node), flawless migrations...

When LINSTOR stopped working, it broke the entire cluster's storage completely, and I had to purge my resources, resource definitions, delete the resource group, and start over.

So, again, after three of these explosions, I gave up.

Stay tuned for the Ceph rebuild, coming soon! I'll be interested in seeing what the level of complexity is and just how it performs compared to DRBD.

If Ceph doesn't work out... well, there's always goats to farm.
