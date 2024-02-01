---
title: "Fedora 39 + T14 Gen 2 AMD"
date: 2024-01-31T01:30:00-05:00
draft: true
---

## Introduction

This is about my "new" laptop (I've had it for around a year now, and I bought it used) but there's some minor config shenanigans that are essential to get it performing as well as it should on Linux. And I like to talk about hardware.

{{< figure src="images/wallpaper.JPG" alt="T14 Gen 2 AMD Gnome 45 nothing on screen">}}

### Use Case
My only use-case that would even suggest a machine this powerful is necessary is virtualization (networking devices or interactive machines) on the go, but it's a nice-to-have; I can get away with using this as a desktop, so I don't have to deal with syncing half my user directory anymore, and I can work on whatever VM wherever I am.

I have servers running ESXi that I do use when I can, but I've found myself out somewhere, with a few hours at my disposal, unable to get continue with what I was actively thinking about and working on, which is a pain and, to me, is definitely worth spending a bit to fix.

I like to have a nice and responsive desktop experience with a reasonable amount of screen real estate. I do *some* "Development", with a capital D, but that consists of mostly writing and occasionally running a compiler so it can give me a bunch of warnings and errors.

I'm also a student who has had to sit in a lab for five hours at a time, and I wind up using my laptop heavily somewhere that is not my desk and might not have a charger for an extended period at least a few times a month, so battery life is fairly important to me. Luckily, the T14 has me mostly covered there.

## Hardware
### Specs & Performance
|Component | As Configured |
|-|-|
| CPU: | Ryzen 7 5850u (8 core, 15-28w), Vega iGPU
| Memory: | 2x16gb (half soldered) DDR4 3200
| Disk: | Samsung 980 Pro 2tb
| WiFi: | AX200 (replaced some Mediatek card)

{{< figure src="images/neofetch.png" alt="neofetch output" >}}

Overall? I'm extremely happy with it, as far as performance is concerned. The laptop rips through anything I throw at it with a tiny bit of fan noise. This is two+ years after Zen 3 was the New Hotness, with bog standard DDR4, and a rather plain cooling solution. I can run fairly large virtualized labs on it (with GNS3), toss whatever I want in a VM (ESXi!) with plenty of cores and memory, and that SSD comfortably contains all the disk images I could dream of (I don't exactly need a lot of storage). All this in a fairly cheap, fairly light 14" laptop with decent battery life (I think I'm $400 into it, weighs 3.3 lbs, 6+ hours on a charge respectively).

The most significant omissions from this machine are Thunderbolt (by way of USB4), Intel vPro, and a 16:10 screen with decent color, but:

1\. I can definitely live without vPro on my personal laptop (and it's worth noting that the Ryzen 7 **PRO** 5850u does support AMD PRO features that include remote management)

and

2\. Thunderbolt or USB4 would only offer an increased data rate and PCIE tunneling (because I still have DisplayPort 1.4 over 10 Gbps USB-C ports), so it's mostly an issue of this laptop not working with Thunderbolt docks, not one of capability. This machine can comfortably drive two 4k60hz or four 1440p60hz displays off one USB-C cable (tested on a Lenovo 40AS USB-C dock and 40AJ 'mechanical' USB-C dock) which is the part of Thunderbolt that I care about.

So, the third point remains, and my main pain point with the "spec" stuff is really just the existence of the 16:9, 250 nit, 62% sRGB (according to an old Spyder 3 colorimeter) 1920x1080 display - it's not much more than usable. I don't enjoy using this laptop as a laptop, but it must be said that the screen is still *much* better than my old T430 with a 1600x900 TN with a 0:1 contrast ratio and lacks the visible touchscreen matrix on my T480s. I *could* easily replace this screen with a nicer 400 nit, 100% sRGB, low power screen for another $100 and about half an hour of time (and I did that to a T480s in the past) but it really doesn't bother me that much, and will still be 16:9.

### Alder Lake P28 comparison

Not sure where to slot this section (it didn't fit before the complaints about the display), so it'll go here.

Comparing the R7 5850u T14 Gen 2 (DDR4 3200) to a Dell Precision 3470 with an i5 1250p and (admittedly slow) DDR5 4800, performance is closer than it should be - the i5 scores about 20% higher in Geekbench 6 singlecore and 40% higher in the multicore test (my results, on Debian Sid), but the 5850u is only marginally slower than Alder Lake P28 in "raw CPU performance" benchmarks that aren't so memory dependent.

This is mostly because of its 8 full-fat Cezanne cores (the Cinebench R23 multicore test [[per CPUMonkey](https://www.cpu-monkey.com/en/compare_cpu-intel_core_i5_1250p-vs-amd_ryzen_7_pro_5850u)] sees it about equal with a 1250p, which has 4 big and 8 little cores.) It's also worth noting that the Dell, with a battery that's 40% larger (64wh vs degraded 45wh) runs out of charge about an hour SOONER than the AMD 8 core. They both have similar displays and not-LPDDRwhatever. That means you could expect about *65% better endurance* from a similarly performant AMD device that also doesn't cook your lap or desk if you look at it wrong. So, yeah, Alder Lake mobile has a well-deserved reputation. By the time this post is up, I'll have gotten rid of the 3470.

### Build

I'm not impressed by the build quality; it leaves a lot to be desired when I compare it to my (now nearly ancient) T480s or (actually ancient) T430. The center of the case (beneath the touchpad) can bend quite an alarming amount. The top cover flexes so much that if any force is put on the lid, the screen and keyboard will rub - my T14 was never in great shape, but the screen has a much worse keyboard imprint now than it did a year ago (and I am relatively careful with my laptop - the worst treatment it gets is a cushioned laptop bag.)

Luckily, durability is a different story - I've dropped this laptop from about six feet up twice (in recent memory) and it's survived with zero damage.

The keyboard is, like durability, fantastic - much better than an X1 Yoga G6 or the aforementioned Dell 3470 thanks to the amount of travel. This is the last shout for the 1.8mm keyboard - the Gen 3, released in 2022, cut travel down to 1.5mm, which is a very noticeable change, and one that I'll say is a downgrade.

The T14's keyboard feels different than what's in my T480s, but this is likely down to Lenovo's component lottery - the T14 has what feels like a bit more travel, is springier and is less tactile. I don't think this is because of a redesign of the mechanism, just the notorious huge variance in Lenovo parts (I'm no stranger to swapping a keyboard and noticing that it feels completely different.) On a related, but slightly more positive note, it's still extremely easy to replace this keyboard. Just two captive screws underneath the TrackPoint's mouse buttons (removed by prying up from the bottom of the mouse button) hold the keyboard in place. Mine arrived (used) with dead left and right clickers, so I swapped a T480s's keyboard in for a week or so to use it before I sent the laptop off for a warranty repair.

## Software

{{< figure src="images/neofetch-straighton.JPG" alt="T14 with Neofetch in a Gnome Terminal window, VSCode in background showing this page's source" >}}

I haven't run Windows on this; it's been Debian or Fedora since I got it. I don't know how good the Mediatek WLAN card these normally ship with is because I pulled it out and threw it in a drawer the same day the machine showed up at my door (I did not enjoy dealing with a similar card in a HP 845 G9.) I would assume the Mediatek card is bad and would have some weird compatibility issue preventing the drivers from being included with upstream Linux because that's what I've encountered every other time I've used a Mediatek WiFi card.

After the WiFi hardware is dealt with, there are a few other important things to dink with.

The default configuration of the `amd_pstate` driver has been lacking since I got it, too. Out of the box, with a fresh Fedora 39 install, this machine chugs down the power, managing just about four hours out of the battery (with about 45wh of remaining capacity). By just setting all the files in the selection:

```/sys/devices/cpu/cpu*/cpufreq/energy_performance_preference```

to "power" instead of the default "performance", the processor will behave more conservatively, sucking down less power for about the same performance. I did run Geekbench a few times and, surprisingly, saw better results with `epp = power`, but I haven't had the time or will to go dig in and figure out why that is. Perhaps the machine throttles earlier when it's allowed to suck down 28w of juice.

Additional hiccups with amd_pstate include the GNOME performance toggle not actually applying scaling governor changes to the files in /sys/.../cpufreq/scaling_governor, though perhaps that just isn't how the GNOME performance toggle works and I'm forgetting things. TODO: figure this out. Recently, I've just left the governor set to `powersave` and performance_preference set to `power`, though performance_preference seems to be the only thing that makes a significant impact on performance or battery life.

The system remains very responsive with these set, benchmarks faster than it does with `scaling=powersave` and `epp=performance`, and battery life is much improved. On reset, epp defaults to `power`, so I wrote a service to keep it set to what I want after a reboot.

So the processor requires some tinkering to extract its full potential. So do the integrated graphics, though it's a much simpler issue.

If Uniform Memory Access (UMA) is set to `Auto` in the UEFI, GNOME plus a browser and some Electron apps will absolutely curbstomp the poor Vega 8. I'm talking shotgun -> watermelon level carnage. While I wasn't able to quickly figure out how much memory the iGPU was being given (`lspci` said I had 256mb of VRAM before setting UMA=4gb, but also says I have 256mb afterwards), I *was* able to notice that, after I permanently allocated 4gb of memory to it:
1. the animation that plays when entering GNOME Overview (the Super key zoom out thing) remains buttery smooth regardless of what I've got open on three 1440p displays
2. the mouse cursor no longer becomes a jittery mess EVER
3. windows can be dragged across the display at redraw rates in excess of ten frames per second even when I have 15gb of Firefox tabs open.

This also makes the few games I've tried to run on the laptop (mostly Ace Combat 7) *much* nicer to play, as you'd probably expect.

I've heard that the Steam Deck actually does support automatically allocating memory to its iGPU on Linux, so I'm a little surprised that it didn't work for me (perhaps I shouldn't have been so surprised, or perhaps it does work, just not well). It was an easy fix, so no biggie.. just took me putting up with awful performance for a while to get frustrated enough to fix it. Didn't spend a lot of time troubleshooting this, so I don't know where the fault lies.

So it definitely isn't as "batteries included" as a Kaby Lake laptop, as far as Linux is concerned. Those are in another *universe* on the "Just Works" scale. But it's newer than, faster than, and gets better battery life than a Kaby Lake laptop, so that's alright by me. After all, if I valued my time, I wouldn't be using a Linux desktop in the first place (joke).

## Conclusion

It's a laptop that runs Linux. It's fast, was cheap, and lasts a decent while on a charge. The fact that I've come to use this little laptop instead of my Xeon W desktop with four times as much memory and a 5700xt when I'm at my desk should be rather telling - the processor in this T14 is just as fast as that 120w Skylake 8 core *in multicore workloads*, and I can grab this computer and go sit on the couch with it. This is essentially the "do it all" machine, and outside of a few minor nitpicks and Linux Issues, it's done everything but play AAA games for me.

Would I prefer a MacBook Pro? Probably, the screens are nice, though asking if I'd prefer Mac OS would be something else entirely. And, as I mentioned before, this was only about $400, which is much less than any ARM-powered half-eaten fruit.

Would I prefer a $(Michael_Product) from the brand that starts with D? Probably not. I'm not a fan of their keyboards, the lack of the TrackPoint, or the lack of AMD options in their business devices. I sold on the Precision 3470 (Latitude 5431) I mentioned earlier because I was underwhelmed by the build quality of the device and disappointed by the processor's thirst. Maybe the Intel's problem will be solved with Meteor Lake, but I don't expect the Dell problem to change much at all.

I'll probably replace it (or, more likely, supplement it, and replace a T480s running Windows) with a T14 Gen 5 if they ship with a Strix Point 12 core.. but that's probably two years away from showing up used on eBay. I can dream, though.

Would I recommend picking up a T14 G2 AMD today? Probably not. They've held their value too well compared to the faster, more efficient T14 Gen 3s with a 16:10 display and a better build. I'd rather spend an extra $100 to get a R7 6850u and 16:10 display, or about the same $500 to get a R5 6650u instead of a R7 5850u. But I got a deal on my T14 back when a Gen 3 was $800, and I'm happy with it.