---
title: "Maxxing out my P1 Gen 4 with an RTX A5000 16gb and an i9-11950H (for science)"
date: 2025-04-05T12:12:59-00:00
draft: false
---

## Update

That wonderful 16g of VRAM was on its way out. Back to the baby GPU for me.

## Introduction

My P1 Gen 4 is just about my favorite laptop. It's got a big, bright, reasonably high resolution 16" screen (2560x1600, 400 nits, 100% sRGB, 60hz.. more on this Soon™ - there are some real nice 4K 120hz 16" panels for cheap), no number pad, lots of memory, a nice glass trackpad, and it has a TrackPoint.

I do have a quibble with it, though. The i7-11800H, RTX A2000 4gb model I have (bought early 2024 for about $600) is loud, runs hot and can't put its eight Tiger Lake cores or baby Nvidia card to proper use, since it winds up thermal throttling constantly.

At first I'd bought a vapor chamber heatsink from the higher-end models to install on my existing machine, but the heatsink I got was damaged in shipping and one of the fans (integrated with the heatsink) is a bit larger on the vapor chamber model, which meant it needed some modification to fit my machine. I eventually wound up shelving the project since I didn't care to chop up my one good heatsink.

But then, one day, I was perusing eBay in search of a Gen 5 board with an A3000 or RTX 3060 so I could drop in a better heatsink and get some Alder Lake cores. And I stumbled across this:

{{< figure src="images/p1-mobo-ebay-listing.png" >}}

A $315 P1 Gen 4 board.. with a single M.2 slot, a big cutout for the left-side fan, and, upon further inspection, LOTS of VRAM.. No specs listed, past the part number, but the GPU die looked big, and I quickly found that 5B21D53577 is the range-topping i9-11950H, RTX A5000 16gb mainboard.

Well, a few days later, I had a new RTX A5000 mainboard and a new, less damaged vapor chamber heatsink for the P1 on my desk.

So let's get it going!! Well... teaser first:

{{< figure src="images/p1-11950h-a5000-fastfetch.png" >}}

## Upgrade process

### Bill of Materials

If you intend to do this yourself, now would be a great time to replace the keyboard/palmrest assembly, as you'll have the entire laptop torn down to swap the board.

1. A low-spec P1 Gen 4.
2. A higher-end P1 Gen 4, or, likely, a Gen 5 or Gen 6 mainboard. All three (Gen 4, Gen 5, Gen 6) share the same chassis, so it's likely the newer boards are compatible.
	1. Any of the models with something better than a RTX A3000/3000 Ada or RTX 30/4060 use the better vapor chamber heatsink. You can tell by the shape of the larger fan cutout, the lack of the second NVME M.2 slot and WWAN M.2 slot, and the physically larger GPU/larger visible quantity of VRAM.
	2. The part number for my RTX A5000 board is 5B21D53577.
3. A vapor chamber heatsink, P/N 5H41D34325. This will run you about $30
4. The longer trackpad cable, P/N 5C11D64960. This one will run you about $25 at the time of writing (ouch).
   1. If you don't see this in pictures it's because I didn't know that I needed it. You need it.
5. Some thermal pads, of 2mm and 0.5mm thickness. I got a random set off Amazon for $8.
6. A good screwdriver set
7. Some patience

### Procedure

Open up your laptop.

Unscrew the bottom cover, and gently pry up the rear and, if needed, the sides of the bottom cover. Remove the bottom cover (it hinges at the front).

{{< figure src="images/p1-bottom-a2000.png" >}}

Unplug assorted connectors.

{{< figure src="images/p1-a2000-labelled.png" >}}

1. Unplug the battery. The connector on the P1 is a bit funky - unseat the left (front) side, then pull it up to remove it from its socket on the mainboard.
2. Gently flip the keyboard ribbon cable latch up (slide a fingernail in above the ribbon cable, and gently lift up) then remove the keyboard ribbon cable.
3. Remove the speaker cable by sliding the connector out of its socket. Remove the speakers; they will be in the way. You should probably completely remove the battery here, so you have more room to work.
4. Remove the CMOS battery. It's got some adhesive on the bottom; just pry up.
5. Unplug the first fan connector. These are a little tricky; pry it out with your fingernails or use a pair of pry tools.
6. The embedded DisplayPort connector for the screen may need to wait until you remove the heatsink, so you have room to work without the fans in the way. It can be removed by prying up on the back of the connector (from the rear of the laptop). It will pop up and out with a little effort.
7. The DC input (Slim Tip) will need to be removed from the mainboard. Remove the little piece of plastic covering the socket and pry it backwards and out.
8. As with the keyboard ribbon cable, gently pry up the latch holding the trackpad ribbon cable in place, pull out the ribbon cable, and put the latch back down so it doesn't get broken.
9. Get the plastic sheet out of the way and unplug fan #2 the same way you unplugged fan #1.
10. Pry the WLAN antenna cables off the soldered WiFi card.
11. Flip the camera/microphone ribbon cable latch up and remove the cable.
12. Flip the fingerprint/power button ribbon cable latch up and remove the cable.

Now's a good time to remove your memory and disks while the board is still firmly in place. The P1 boards aren't the most rigid thing in the world thanks to their ginormous fan cutouts.

Once you're done with that, it's time to get the heatsink (and all the components Lenovo stuck to the board) off, then take the board out.

Remove the heatsink by undoing the screws highlighted yellow. Doesn't really matter what order you unscrew them in, but feel free to follow the order Lenovo suggests (54321). Some are hidden under plastic sheets.

Once you've removed the heatsink and cleaned off the CPU and GPU, remove the DC power input by removing the screws highlighted with red, then sliding the brace out. Put it aside for now. These screws are different than most (M.2 L5); keep track of them.

Remove the fingerprint sensor/power button PCB that's stuck to the top left of the system board by gently prying it away. The adhesive on mine had already failed, which is why it's loose in the picture below.

Remove the abundance of mainboard screws (M.2 L3) highlighted green.

Finally, remove the two M.2 L6 screws at the top of the system board, highlighted in darker blue. Note the spacers they sit on; you'll probably need to reuse these. It's easiest to poke them out when the board is no longer in your system.

Once you're done unscrewing and cleaning, slide the board out by wiggling and pulling up on one side (probably the left side - the right side will be harder because of the port density), then sliding it out.

{{< figure src="images/p1-a2000-screws.png" >}}

Flip the now-removed board over and remove the PC speaker from its underside by gently prying up on it until the adhesive gives.

Pop the M.2 L6 standoffs/spacers off your old board by pushing up on them.

{{< figure src="images/p1-a2000-board-bottom-highlighted.png" >}}

Get your new board and repeat the process in reverse, though I'd keep the spacers separate until it's in the chassis.

- Stick the PC speaker on the bottom, then install the mainboard in the system.
- Secure the mainboard with the oodles of M.2 L3 screws you removed (with the exception of the one that was right next to the WLAN slot - this will go over the heatsink).
- Pop the two M.2 L6 standoffs (yellow, in the image just above this text, dark blue in the 'unscrew' image above that) back in place, then install the M.2 L6 screws that secure the top of the board.
- Pop the DC jack's connector back into the mainboard, then screw it back in with the two M.2 L5 screws. You might need to mess with the DC jack a bit to get it in the right spot.
- Might as well plug the eDP connector back in while you're at it. It's kind of a pain to deal with when the heatsink is in the way.
- Install your drive and memory. Or wait for these two, doesn't really matter.
	- If you have a Lenovo M.2 heatsink, go ahead and reinstall it, using all three screws to secure it - it's on the opposite side of the system, far away from that giant copper Thing, so the main heatsink doesn't obstruct it.

Get your thermal pads ready! If your heatsink came with pristine ones, feel free to skip this step.

- Cut to fit some thermal pads for the VRAM and VRMs.
- The GPU VRMs and VRAM require 2mm pads.
- The CPU VRMs require 0.5mm pads.
- It might make sense to use a 2mm pad for the chipset, since the copper Thing will be a few mm away from it. I didn't do this, just applied some thermal paste (that did absolutely nothing).

{{< figure src="images/p1-thermalpads.png" >}}

Clean your new heatsink, apply thermal paste, install the heatsink and screw it down. Plug in the two fan connectors (the left one is in a slightly different spot now).

Reseat the ribbon cables. Reinstall the speakers. Install your new, longer, trackpad cable. Stick the adhered components (fingerprint PCB, CMOS battery) back to the board. Reinstall the battery, and screw it down with (hopefully) your last few M.2 L3 screws.

Finishing touches.. done! Now that's considerably more heatsink than we started with, isn't it?

{{< figure src="images/p1-bottom-a5000-vape.png" >}}

Don't reinstall the bottom cover, unless you want the system to not POST (joking). Test it out. It'll probably complain that time is all effed up and might complain that the serial number and/or system type are "INVALID" (if the board is a proper refurbished one). If it boots, slap the bottom cover back on and make sure the machine doesn't melt itself by taking a look at HWINFO or HWMonitor.

If the serial number is not set, you'll probably want to set it; you can follow the relevant Lenovo KB [here](https://support.lenovo.com/us/en/solutions/ht501039-how-to-update-the-machine-type-and-model-mtm-system-serial-number-sn-or-system-brand-id-of-system-bios-menu-thinkcentre-thinkstation) for instructions on how to do so with the Lenovo utility for the EFI shell or DOS. I think I've done this in Windows somehow but don't recall exactly how. For the time being I don't care enough to fix my own machine; all it does is stop Lenovo Vantage from finding drivers (I use the driver packs anyway).

## Performance, thermals, system info

The P1 sustains about 150w Total System Power with 80w reported by the GPU and 30w reported by the CPU while playing a game with uncapped framerate/max GPU stress (probably the hardest I'll ever be on it). Temperatures, after about 15 min, stabilize at around 85c CPU, 80c GPU, 92c GPU hotspot in this scenario.

Unfortunately, the 11950H will never get near its 135w PL2 or 109w PL1. With CPU-only loads, the processor will draw ~65w, then settle down to ~55w and sit around 3.8ghz in light loads, or closer to 2.9ghz in full AVX torture mode (Prime95 Small FFT). At 55w, it'll settle in around 95c with moderate fan noise. This is a substantial improvement over the approx. 2.3 - 2.4ghz I got out of the 11800H with a similar heavy AVX load (that poor thing thermal throttled so hard that it dropped below 3ghz in "light" all-core loads like normal compression).

As with most laptops, if you're playing games and want performance, you'll need to disable the iGPU in the BIOS. When I was using this system with its original board, I noticed that disabling the iGPU made everything from Windows animations to 

You will be limited by the 30w allotted to the CPU in high-framerate games. It might make sense to cap FPS and lower graphics settings to allow the CPU to use more power (by running the GPU below its 80w maximum) to reduce stutter.

All the following benchmarks were done with a 230w AC adapter (the P1 is faster with this, even if the 11800H didn't use anywhere near 150w on its best day) and were in Best Performance mode to take advantage of the highest power limits Lenovo lets these machines use.

### Geekbench

Geekbench is a terrible benchmark for anything but general desktop performance, but it's easy to run. A high single-core score depends on memory and cache - the more, the faster, the better, and older platforms are brutally punished for their limited memory bandwidth.

[In Geekbench 6.4, the i9-11950H performs admirably](https://browser.geekbench.com/v6/cpu/11384341), scoring 2252/10069 (Windows 11 LTSC IoT) - this lands it firmly in striking range of a 12700H, even with slower DDR4 3200 (as opposed to DDR5 4800+).

For comparison, my P14s Gen 5 Intel with a Core Ultra 155H [scores 2475/13587](https://browser.geekbench.com/v6/cpu/10170892) running the same build of IoT.

The same P1 with its original board scored its best, [2142/10017, running Windows 10 IoT LTSC](https://browser.geekbench.com/v6/cpu/7789877), which is plain better than W11 LTSC at Geekbench, on the order of 5%. Running a slightly older W11 IoT LTSC build, the 11800H scored just [2079/9492](https://browser.geekbench.com/v6/cpu/7046745).

This 155H is a 16 (6+8+2) core, 22 thread processor that can and will use up to 65 watts of power, and the P14s boasts two 32gb DDR5 5600 MT/s DIMMs for far greater memory bandwidth than the older P1.

Interestingly, despite the 155H having significantly more cache (the 11950H makes do with just 80K of L1$ and 1.25 MB of L2$ for each of its 8 Tiger Lake cores (a total of 10 MB L2), while the 155H has 112K L1 plus 2 MB L2 each for a total of 12 MB L2$ for the P cores, *plus* an extra 4 for the E cores, and both have 24 MB of L3$) and faster system memory (both things Geekbench cares quite a lot about) the 11950H is quite close. We'll see this same story in Passmark CPU Mark, and I'm not entirely sure why this is (I suppose the MTL uArch was just heavily optimized for idle power efficiency, though my 75wh P14s G5i doesn't really last much longer than the P1 with an aged 70wh battery when the latter's iGPU is enabled).

Anyway, way off topic. Maybe Intel's 10nm process is just awesome and takes big unicorn dumps all over TSMC N6, or maybe Tiger Lake is just magical. The frequency difference is marginal and we know MTL is slower than Alder Lake clock-for-clock.. but still. Ouch.

### Passmark

{{< figure src="images/p1-cpumark.png" >}}

The 11950H continues to stomp around and do quite well in Passmark, where its score of 22283 multi/3274 single puts it within striking distance of 45w Ryzen eight-core chips and its younger cousins (the 155H and 12700H) if we go by the overall average (see below).

Of course, the 7840HS runs away with the lead here (even the 15-28w 7840U would beat out the 11950H, though, to be fair, the 7840U is easily one of the best laptop chips to come out over the past few years), and Alder Lake puts up a good showing (much better than the 155H) at the expense of a whole lot of power draw.

Zen 3+'s best is about on par with the specific 11950H in my P1, with the 6900HS average being slightly better in multi-core and slightly worse in single-core performance. All in all, a healthy showing in these two admittedly unscientific benchmarks.

I'm not going to bother comparing this 11950H to a HX 370. My P1 will get sad if I do. The HX 370 would absolutely trounce this thing in every way.

{{< figure src="images/p1-passmark-11950h-155h-7840hs-6900hs-12700h.png" >}}

### HWINFO

So what does that performance cost us, in terms of heat and noise? It's not really that bad (well.. it's not GREAT.. but it's a 4lb laptop with an 11950H and RTX A5000!)

Here's what HWINFO stats look like on mine. The averages are not super useful since I was using the system for a couple of hours with HWINFO running for all kinds of stuff.

At idle, writing up this document, with an external display connected (to the A5000):

{{< figure src="images/hwinfo-idle.png" >}}

While running Prime95 Small FFT, external display connected and editing this document, mild fan noise and slightly warm:

{{< figure src="images/hwinfo-p95.png" >}}

While playing a game at 1440p, full GPU load:

{{< figure src="images/p1-max-cpu-gpu-cpu.png" >}}

{{< figure src="images/p1-max-cpu-gpu-gpu.png" >}}

### CPU-Z

{{< figure src="images/p1-11950h-cpuz.png" >}}

{{< figure src="images/p1-11950h-cpuz-bench.png" >}}

Validation: https://valid.x86.fr/22w44f

{{< figure src="images/p1-11950h-cpuz-valid.png" >}}

{{< figure src="images/p1-11950h-mainboard.png" >}}

{{< figure src="images/p1-11950h-a5000-cpuz-graphics.png" >}}

### GPU-Z

{{< figure src="images/p1-11950h-gpuz.png" >}}

## Conclusion

I love having 16gb of VRAM. I can talk to my laptop (with 13b parameters!) instead of going outside and talking to people. And then, when I do need to go outside, I can bring eight really fast cores and a 3070 with me. And it looks all professional, and fits right in my bag!

Jokes aside, this is a monstrous laptop, and such a great form factor. Love it.

That's all for now! Maybe we'll revisit this machine with a 4k 120hz display.. and, maybe, in the distant future, a Gen 5 or Gen 6 board (probably not).