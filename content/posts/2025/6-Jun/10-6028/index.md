---
title: "Quick thoughts on & power consumption of Supermicro 6028 with X10DRU-TR4T+"
date: 2025-06-23T15:30:00-00:00
draft: false
---

I got a pair of these for storage servers. Still haven't used them for more than some quick tinkering yet. Bit of a waste.

## Some findings

You can drop a peripheral (Molex) to SATA power cable and SFF 8087 to SATA breakout cable in these to shove some SATA drives in the back of the box. I've done this with both of mine.

By combining this with some U.2 risers you could feasibly fit eight internal SATA drives, twelve spinners in the front, and twelve more NVMEs in PCI-E slots. Since the system has a quartet of built-in Intel X540 10gbe NICs, you could actually consider using *all* of the remaining PCI-E for drives.

PCI-E layout is fantastic for dual slot GPUs. The necessary power cables are EPS 12v female to EPS 12v male or EPS 12v female to PCI-E 12v.

The PSUs I got are only 1kw on 240v (800w on 120v.) They're a semi-standard Supermicro size, and I believe some machines with similar chassis came with 1.4kw units.

Idle power consumption is quite high once you start adding drives. See the "power consumption" section below.

The internal HBA slot is, in Supermicro fashion, just a standard PCI-E x8 slot. You can shove a HBA330 (my choice, a standard LSI 3008 HBA that can be found used for about $20) or any other standard RAID card you please into said internal slot. The backplane is cabled via SFF 8643.

The built-in NICs (the Intel X540) are supposedly unsupported in Server 2025, but you can install the driver anyway by pointing Windows at the ixgbe driver (they use the same driver as the X550, but the Intel driver installer throws a fit on anything that's not Server 2019.) ESXi 8 and Linux support the X540s out of the box with no hassle.

If you want >10gbe, it might be wise to get the variant of 6028 with built-in gigabit Ethernet instead, as it has an extra PCIE 3.0x8 slot on the 'NIC riser' (whose lanes are eaten up by the four X540s in this model).

I've experienced no driver-related pain with these NICs, but be warned that they only support 100m/1g/10g - no multi-gig here. You'll need a much newer card for that.

You WILL be stuck at DDR4 1866 (or 1600!) if you're running a v3 Xeon at 2DPC or 3DPC (respectively.) Which sucks. Rude awakening the first time I dropped 384gb in one of these.

Broadwell "v4" chips will run your memory at 2400 (1dpc), 2133 (2dpc), or 1866 (3dpc). I didn't test a Broadwell chip with two or more DIMMs per channel of 2133 MT/s memory to see if it clocks down because I don't have enough 2133 MT/s memory kicking around to do so.

## Power consumption

### Constants

- MB: X10DRU, 4 X540s, 0 up, BMC active
- Stock fans @ "optimal" speed
- Dell HBA330 (LSI 3008)
- Disk: 4x8tb 3.5" SATA HDDs, 4x480gb Intel DC S3510 2.5" SATA SSDs

### Run 1

- 2DPC - 16x16gb (256gb) dual rank DDR4 2400 RDIMM @ 1866 MT/s
- 2x E5-2698 v3 (Haswell 16c)

**~150w** idle

### Run 2

- 1DPC - 8x16gb (128gb) dual rank DDR4 2400 RDIMM @ 2133 MT/s
- 2x E5-2698 v3 (Haswell 16c)

**~140w** idle

### Run 3

- 1DPC - 4x16gb (64gb) dual rank DDR4 2400 RDIMM @ 2133 MT/s
- 1x E5-2698 v3 (Haswell 16c 135w TDP)

**~130w** idle

### Run 4

- 1DPC - 4x16gb (64gb) dual rank DDR4 2400 RDIMM @ 2400 MT/s
- 1x E5-2660 v4 (Broadwell 14c 105w TDP)

**~125w** idle

### Run 5

- 2DPC - 8x16gb (128gb) dual rank DDR4 2400 RDIMM @ 2133 MT/s
- 1x E5-2660 v4 (Broadwell 14c 105w TDP)

**~130w** idle

### Thoughts on power draw

This thing is thirsty! That ~125w minimum seems a bit high. For comparison, I have another X10:

- X10DRL-i (C612, 1DPC ATX Supermicro board with 2x i210)
- 2x 2698 v3
- 8x 32gb DDR4 2133 LRDIMMs (256gb)
- 2x DC P3600 1.6tb NVMEs
- 2x DC S3510 120gb SATA SSDs
- 2x DC S3510 480gb SATA SSDs
- 1x Quadro M2000
- 1x X540-AT2 (two copper X540s)

sitting under my desk and idling at around 90w.

It's mostly due to the spinning rust - without which (or, when it's spun down) the system idles around 90-100w with a single 2660 v4.

I was surprised by how small the difference between two Haswell CPUs and one Broadwell CPU is. I suppose the vast majority of the power consumption is thanks to the drives, NICs, PCH and fans, not the chips (at idle).

Bit of a tangent.. but I found my E5-2667 v4 8-core parts *do* draw about half as much power as the E5-2698 v3s under load - worth considering if you're CPU shopping and don't intend to have the machine sitting at idle 24/7. More data to come when I get my smart plugs set up and start collecting data from them (wink wink).
