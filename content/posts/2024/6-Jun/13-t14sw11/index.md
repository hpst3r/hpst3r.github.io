---
title: "T14s Gen 1 AMD + Windows 11"
date: 2024-06-29T12:34:56-00:00
draft: false
---

## New laptop! Well, not really.

This machine isn't exactly new to me (or new at all, really.) It's been doing a lot of heavy lifting for me over the past few months, so I thought I'd take a moment to appreciate it a bit and give it a writeup.

This was a $300 eBay find that I used to replace a Latitude 3540 (to get myself a centered keyboard, TrackPoint, smaller device, and more cores/memory/disk) as my primary machine at my day job.

It's also slowly become my only Windows machine as all the others flee to Fedora 40 in the face of the upcoming EoS date for Windows 10 (though I'll probably turn something into a dedicated gaming PC to run racing sims sooner or later.)

I've spent a lot of hours working from this device - I pull it out any time I want to write some PowerShell or customize a Windows ISO, and it has no trouble hosting a few Hyper-V guests if I need to do any Windows testing. Overall, it's been one of the best experiences I've had with a Windows device (not to say that it's perfect.)

{{< figure src="fetch.png" alt="Fastfetch output" >}}

## Use case

Typical work entails:
- a ticketing system in a web browser
- the Microsoft Remote Desktop viewer
- usually a VPN client of some sort
- Some amount of Windows Terminal windows
- Notion (likely soon to be replaced with selfhosted [Silver Bullet](https://silverbullet.md) as a PWA - more on that in another post)
- a WSL VM
- one or a handful of Windows VMs for testing
- and n+1 windows of VScode (no large projects/intensive linting.)

I have four 1440p monitors, a keyboard, and a mouse at my desk in the office, driven through a Lenovo 40AJ USB-C dock.

At home, I have two 1440p monitors, a 4K monitor, a keyboard, a mouse, a USB microphone and a DAC driven through a Lenovo 40AS USB-C dock.

You'd probably think that this machine is massive overkill for my workload, but it's Windows, so it still feels a bit slow (though I must admit that Zen 2 is no longer new hotness, and this monolithic, cut down caricature of a R7 3700x with half the cache and a hard 25w PL1 package limit isn't exactly the finest Zen 2 has to offer.)

## Processor, heat, battery

The Ryzen 7 4750u is an 8 core monolithic Zen 2 part supporting SMT (16 threads) with 8 MB of L3 cache (a quarter of the 32 MB on a vaguely similar, but chiplet and Socket AM4 Ryzen 7 3700x, and half of the 16 MB present on its mobile Zen 3 successor, the R7 5850u.)

{{< figure src="cpu-z.png" alt="CPU-Z output" >}}

Mine tends to live around 3.5 GHz, with rare excursions closer to the 4.1 GHz rated boost clock.

I have observed what seems to be a 25w package power limit in the T14s Gen 1, and the device usually settles in at 95 degrees Celsius during CPU and/or GPU load, with about 15w going to the CPU and 10w going to the iGPU during normal use (if hwinfo is correct.)

Fan noise is limited. It's present, but not especially loud or overly annoying. It's much quieter than the Dell 3540 it shares my desk with, but much louder than my T14 Gen 2 (which has a much more substantial cooling solution and a more modern processor with substantially better power management.)

On the topic of power and heat, this T14s does lack some power management controls (does not support the amdpstate scaling governor in Linux), and does seem to consume significantly more power at idle, which is exhibited as decreased battery life - I only see about 5 hours out of the 50wh remaining of my 57wh battery in Windows 11, which is adequate, and, again, an improvement over my  1335u Dell Latitude, but nothing to write home about.

Performance is generally quite good, with the Ryzen 4750u generally a little faster than my desktop Xeon W-2140B in single-core performance. In multi-core performance, the 120w Xeon seems to be about 33% faster.

The Ryzen 7 4750u provides about 75% of the performance of a Ryzen 7 5850u in both single and multi-core workloads (the 5850u provides multi-core performance similar to the Xeon W-2140B, but far faster single-core performance than the Xeon W.)

When compared to the i5 1335u (2p8e12t @ 55w PL2 - 15w PL1) in my Dell 3540, the Ryzen 7 4750u (25-15w) in the T14s has similar (slightly worse during PL2) multi-core performance and significantly worse single-core performance, but is able to sustain this 'similar' level of multi-core performance for much longer than the U15 i5 (as the Ryzen 7 sits at 25w forever, and the i5 boosts to 55w briefly before throttling back down to the 20w range.)

Additionally, performance is far more consistent across the 8 Zen 2 cores than it is across the 2 Golden Cove 'performance' cores + 8 Gracemont 'efficiency' cores in the i5 - the system does not seem to 'bog down' when running intensive applications or several virtual machines that occupy the two P cores, because the 4750u has 8 P cores.

Any P28 Intel processors will soundly beat the 4750u (albeit with far higher power consumption, with a 60w PL2 and 28w PL1), with even an older i5 1250p (4p8e16t) being more comparable to the Ryzen 7 6850u than the 4750u or 5850u (the Zen 3 6nm refresh boasts much improved power efficiency, allowing that 15-28w package to produce more performance despite fairly similar IPC to the original Zen 3 uarch.)

The i5 1250p in the Dell 5431 that crossed my desk last year was a fantastic performer (at the expense of substantially higher power consumption, even when idle - the 5431's 63wh battery was exhausted in around 4 hours of light usage) with single core performance 10-20% better than that provided by the 5850u and multicore performance exceeding that provided by my 120w Skylake Xeon W.

Performance is generally good (even without considering the $300 price.) A few Windows VMs running in the background are easy to forget about. Performance with virtualized networks in GNS3 is similarly acceptable.

I believe the 4750u is a decent little chip that still stands up to modern (13th Generation) U15 Intel processors while drawing significantly less power. Consistency (in performance - the Ryzen cannot have something hog the two P cores and bring the interactive system to its knees, for example) is far better than modern Intel chips and makes the user experience much better (when loading up the system.) For the price I paid, I have no complaints, and I knew what I was getting into.

## Graphics, comparisons, adequacy

{{< figure src="gpu-z.png" alt="GPU-Z output" >}}

The integrated graphics in Ryzen 4000 chips, including the iGPU in my Ryzen 7 4750u (with its [Renoir "Vega 7" CU/448 SP](https://www.techpowerup.com/gpu-specs/radeon-graphics-448sp-mobile.c3510) iGPU), are significantly faster than era Intel iGPUs (which were still the UHD 620 on 10th gen U15 chips, or UHD 630 on H45 variants.)

In fact, the Vega 7 iGPU in this laptop performs similarly to the (low-end) dedicated Nvidia MX150 graphics chip that was found in many high-end configurations of Intel ultrabooks at the time. Make no mistake, these don't perform *well* when faced with a monitor wall or 3D acceleration, but the Vega graphics shipped in the Ryzen 4000 series are generally impressive for 2020-vintage iGPUs.

In the Ryzen 4000 stack, the 4750u's iGPU is only second to the range-topping R7 4800u's Vega 8 CU/512 SP chip and does not perform much worse than the latter. Light gaming would be possible, but that is not what I use this device for.

All I have to say on the iGPU front is that it puts in a surprisingly good effort when faced with the obscene number of pixels I ask it to push.

Animations do start to get choppy on occasion, but, once transparency is disabled, things do tend to be fairly smooth (and I haven't even tried reserving additional system memory for the iGPU yet - it's still at 512 MB.)

Graphics are comparable to the Iris XE 80eu iGPUs in modern i5s (1335u) but drivers seem to be far less buggy and I had far fewer problems with the older AMD graphics (compared to my Dell 3540 (i5 1335u/Iris 80eu), and a Dell 5431 (i5 1240p/Iris 80eu) that I was able to use for an extended period recently.)

## Memory, disk, WiFi

Memory is DDR4 3200 (JEDEC timings) soldered to the system board. No LPDDR4x here, and no expandability either.

Disk is a single M.2 2280 (PCIe 3.0x4). I do not believe that the WWAN slot supports a second M.2 SSD, and my unit is not pre-wired for WWAN (lacks antennas).

Wireless connectivity is provided by a socketed Intel AX200. Performance is excellent, as one might expect from an AX200.

## Chassis, keyboard, display

The chassis is the most disappointing part of this laptop. Thermal performance is poor. The right side of the laptop's magnesium chassis heats up to uncomfortable temperatures while on wall power. Fit and finish are subpar; the bottom cover (after replacement from Lenovo for the same problem) is a bit loose and this can be felt every time the laptop is handled. The base of the machine also exhibits an extreme amount of flex (you can bend it. A lot.)

The keyboard is quite good. Key travel is long, the backlight gets bright, key legends are highly legible with backlight on or off, and the layout is great (standard ThinkPad chiclet layout with dedicated navigation, uniform size arrow keys.) Key feel is uniform and actuation is fairly snappy.

The trackpoint works well. On the Gen 1 and Gen 2, the buttons for the pointer are still located directly under the keyboard, not spaced out, which makes them (subjectively - my thought) nicer to use.

The trackpad is sized well and tracks nicely. Gestures work well.

The display is standard business laptop fare. 45% NTSC/60% sRGB. Colors are washed out and it doesn't get very bright. It can be easily upgraded to another 30 pin eDP panel, though I likely won't do this.

## Docking

This AMD machine lacks Thunderbolt, as does the T14 Gen 2a I have. This means you're limited to USB-C with DP Alt Mode and Power Delivery, which limits you to 10 Gbps of data, display and power, and stops Thunderbolt-only docks from working.

This older T14s has some trouble with my Lenovo 40AS USB-C dock (it does not like doing MST + non MST displays) and will sometimes lose both of my MST displays entirely. This could be due to the 8 ft USB-C Gen 2x2 cable that I'm using with it, as it seems to behave better with a shorter USB4 certified cable. The T14 Gen 2a does not have any problems with the longer cable.

The Dell WD19S is similarly unable to drive MST and non MST displays simultaneously. The MST displays (daisy chained Dell U2716Ds) will only run at 1920x1080 or lower off either dock if additional displays that are not daisy-chained are attached. This is not a problem with the T14 Gen 2 AMD. The Dell 3540 is unable to drive more than two 1440p displays via USB-C (DP 1.2 only, vs 1.4 on the AMD ThinkPads), so I have not tested it.

## Overall

Overall, the T14s G1a is a decent laptop with adequate performance. The chassis is a letdown, but the keyboard is excellent, it's fairly quiet, and performance is consistent.

It's a good Windows laptop that can do just about anything. Similar Ryzen 4000 ThinkPads tend to be available for reasonably low prices ($300 range) online now that they're being discarded after their 3-year lifecycles. Similar HP EliteBooks (845 G7) are in the $200 range (the G8, with Zen 3 processors, is priced similarly to a Zen 2 ThinkPad - worthy of consideration if you need the ultimate bang for your buck.)

I'm almost happy with this laptop, but the chassis is killer. I'll probably wind up trying to resolve that by padding the spots where the bottom cover and palmrest don't quite fit together.

Oh - it runs Linux perfectly. Everything works out of the box (with the exception of the Windows Hello camera, but that's to be expected). But that's not what I intend to use mine for (well, not just yet.)