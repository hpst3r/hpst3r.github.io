---
title: "Power Consumption"
date: 2024-01-07T10:10:10-00:00
draft: false
---

This is a writeup of some testing I did on C422/Skylake-W, dual socket (2S) C602/Ivy Bridge-EP, single socket (1S) C612/Broadwell-EP, 2S C612/Broadwell-EP and H87/Haswell-S prebuilt desktops, RDNA and Polaris GPUs, i350-t4 and SFN7122F NICs for power consumption.

So.. my power is in the $0.25/kwh range. Which sucks. I wanted to figure out how much my heaps of sand in nice metal cases cost me so I can make more informed decisions before replacing everything with more E5-2690 v4s.

This is not highly scientific because I only need somewhat accurate numbers to keep in the back of my head. I am not running a datacenter where a 3w difference in measurement will affect a million dollar purchasing decision or even cost $500 a year, so take these numbers with a grain of salt.

As processor power consumption is unlikely to change in CPU-only benchmarks I’ve skipped those that don’t focus on graphics power when I already ran the test on the same system with a different config. I don’t need the same data twice. This mostly applies only to the P520.

And this data is a work in progress; I'll update it as I go. I'm also fighting Hugo over the \<detail> tags in this file, so it may look sloppy.

Anyway, here you go, future me.

## Hardware Configuration(s)

{{<details "<h3 style='display:inline;'>Lenovo ThinkStation P520</h3>">}}

**Motherboard**

Lenovo 1036 (C422, LGA 2066)

**Processor**

Intel Xeon W-2140B (Skylake 8c16t, iMac Pro CPU, 120w PL1, 144w PL2)

**Memory**

4x32gb Kingston DDR4 2400 RDIMMs

**Storage**

1x Intel D3 S4510 (SSDDSC2KB480G8) (Windows boot drive)

2x Intel 670p 1tb, VROC array on onboard M.2 ports

**Graphics**

AMD RDNA configuration: AMD Radeon RX 5700 XT (Gigabyte GV-R57XTGAMING Rev 2)

AMD Polaris configuration: AMD RX 580X 8gb (Dell OEM card)

*All configs are driving two 2560x1440 60hz monitors*

**Peripherals**

FiiO K3 DAC/AMP (probably worth one or two watts, but included in all tests of all configs)

USB switch, bus powered, with a wireless (2.4ghz, 1000hz) mouse and wired USB keyboard (no backlight) attached

DVD drive I stole out of another computer. Not sure how much this adds.

**OS**

Windows 10 Enterprise IOT LTSC (21H2) with Hyper-V disabled, NOT a fresh install, but background tasks minimized and only about 5gb of memory “used”at idle. 

Ultimate Performance power plan, because it’s my desktop and not having Ultimate Performance enabled doesn’t seem to change much at all (idle and “wave the mouse around” power draw remain the same, and Ultimate Performance did increase Geekbench scores when I tested it a while ago, which is enough persuasion for me.)

**Power Supply**

900w 80+ Platinum proprietary unit

**Fans**

3x Delta 92mm (?, recheck) definitely not low power or quiet, incl. one on the CPU cooler
{{</details>}}

{{<details "<h3 style='display:inline;'>Lenovo ThinkStation E32</h3>">}}

**Motherboard**

Lenovo Sharkbay (H87, LGA 1150)

**Processor**

Intel Core i5 4670 (Haswell 4c4t)

**Memory**

2x4gb RAMAXEL DDR3 1600 UDIMMs

**Storage**

1x Intel D3-S4510 (SSDDSC2KB480G8)

**Graphics**

Intel Integrated HD 4600

*All configs are driving one 1024x768 monitor via VGA, unless indicated*

**Peripherals**

USB 2 dongle with a wireless keyboard (? hz) and mouse (1000hz)

*configurations with 10gb NIC only:* SolarFlare SFN7122F

*configurations with i350-t4 only:* Intel i350-t4

**OS**

Windows 10 Enterprise 2021 LTSC (21H2) with Hyper-V disabled. Fresh install. Default everything.

**Power Supply**

240w Bronze FSP proprietary unit

**Fans**

2x AVC 80 or 92mm, eyeballed. Front model number is DAZH0925R2U (9cm) and #2 is on the CPU cooler. DAZH0925R2U is rated 12v .6a so maximum of around 7w.

{{</details>}}

{{<details "<h3 style='display:inline;'>Lenovo ThinkStation C30</h3>">}}

**Motherboard**
Waters Corporation 11361K3 (C602, LGA 2011-3 2S platform)

**Processor**

2x Intel Xeon E5-2680 v2 (Ivy Bridge 10c20t)

**Memory**

8x16gb Samsung/Cisco DDR3 1600 RDIMMs

**Storage**

1x Intel DC S3500 (SSDSC2BB480G4)

**Graphics**

AMD FirePro W5100

*All configs are driving a 1024x768 monitor via a DisplayPort to VGA adapter unless indicated*

**Peripherals**

Same wireless mouse and keyboard as the E32

**OS**

Windows 10 Enterprise LTSC 2021 (21H2)

RHEL 9

VMware ESXi 8

{{</details>}}

{{<details "<h3 style='display:inline;'>Dell Precision T5810</h3>">}}

**Motherboard**

Dell 0HHV7N (C612, LGA 2011-3 1S)

**Processor**

E5-2690 v4 (Broadwell 14c28t 3.2ghz)

**Memory**

2x32gb Micron DDR4 2133 RDIMMs

**Storage**

1x Intel DC S3500 (SSDSC2BB480G4)

**Graphics**

Nvidia Quadro M2000

**Peripherals**

Wireless mouse and keyboard

**OS**

RHEL 9

Fedora 39

Windows 10 Enterprise LTSC 2021

Windows Server 2022

{{</details>}}

{{<details "<h3 style='display:inline;'>Lenovo ThinkStation P710</h3>">}}

**Motherboard**

Lenovo 1030 (C612, LGA 2011-3 2S)

**Processor(s)**

2x E5-2660 v4 (2x Broadwell 14c28t 2.6ghz)

**Memory**

8x8gb assorted DDR4 RDIMMs

**Storage**

2x 256gb SATA SSDs

4x recent Seagate HDDs

**Graphics**

RX560 for display output (it was $20, a waste because it is unused 99.9% of the time)

**Peripherals**

LSI 2008 (Dell H310 flashed to a generic HBA, still argues with the BIOS in this workstation)

**OS**

Proxmox 8

**Power Supply**

650w 80+ Platinum proprietary unit

**Fans**

3 Delta 92mm (if I remember correctly, including 2 CPU fans) and 2 Delta 80mm

{{</details>}}
# Testing

{{<details "<h3 style='display:inline;'>Testing Methodology (’controlled’ Windows tests with logs)</h3>">}}

Power on system (cold boot)

Wait approximately five minutes. Close G Hub, Parsec, Discord, Spotify background tasks.

Clear hwinfo averages and timer. Start logging. Wait sixty seconds. Stop logging.

Clear hwinfo averages and timer. Start logging. Move mouse around for thirty seconds. Stop logging.

Open Discord, Spotify, and a Firefox Developer Edition tab with uBlock, Unhook extensions, play LG OLED The Black on YouTube at 1440p 60fps. Clear averages and start logging. Wait sixty seconds. Stop logging.

Start Prime95 Blend test. Wait for Prime95 to load up system memory. Clear averages and timer. Start logging. Wait sixty seconds. Stop logging. Close Prime95 (stop test and exit.)

Start Furmark burn-in test. Clear averages and timer. Start logging. Wait sixty seconds. Stop logging.

Leave Furmark running. Start Prime95 Blend, wait for Prime95 to load up system memory. Clear averages and timer. Start logging. Wait sixty seconds. Stop logging.

{{</details>}}



{{<details "<h3 style='display:inline;'>ThinkStation P520 (Skylake Xeon W)</h3>">}}

#### Run 1: 5700xt

| Idle | Moving mouse | Electron & YT | P95 Blend | Furmark only | Furmark & P95 |
| --- | --- | --- | --- | --- | --- |
| 95w WALL | 115w WALL | 125w WALL | 248w WALL | 310w WALL | 410w WALL |
| 16w CPU | 33w CPU | 43w CPU | 120w CPU | 40w CPU | 120w CPU |
| 16w RAM | 20w RAM | 30w RAM | 110w RAM | 22w RAM | 110w RAM |
| 38w GPU | 38w GPU | 39w GPU | 38w GPU | 170w GPU | 155w GPU |
| 25w REM | 24w REM | 13w REM | -20w REM | 78w REM | 25w REM |

**Suspended:**

6.3w from the wall

Running a Fedora 39 live system and doing my very scientific “sit on the desktop” and “sit on the desktop and move the mouse around” stress tests, I see about 90w and 100w from the wall, respectively, but W10 is running the Ultimate Performance power plan and I didn’t touch intel_pstate or even the GNOME controls for scaling governor (or performance preference, I don’t remember what that toggle in the GUI does.) I don’t think 5-20w is worth crying over in this scenario, and I don’t care enough to try and get better Linux data right now because then I’d feel compelled to try a fresh Windows install, too. For my needs, that level of accuracy is not required.

#### Run 2: RX 580

| Power | Idle | Moving mouse | Electron & YT | Furmark only |
| --- | --- | --- | --- | --- |
| From wall (w) | 55 | 80 | 90 | 270 |
| CPU (w) | 15 | 33 | 42 | 35 |
| RAM (w) | 15 | 20 | 29 | 19 |
| GPU (w) | 6.2 | 7.3 | 8.7 | 145 |
| Remainder (w) | 19 | 18 | 10 | 71 |

#### Original config: with 5700xt, rougher results

140w power on

130w before POSTing (memory check)

95w typical near complete idle (Spotify, Discord, Notion open, after Windows has settled down (10 min post boot))

CPU package: 20w

total DRAM power: 20w

GPU: 38w

Rest: 17w

/tmp
120w typical idle ‘in use’ (moving mouse in near complete idle scenario - Spotify, Discord, Notion open, after Windows has settled down (10 min post boot))

CPU: 40w

DRAM: 25w

GPU: 38w

Rest: 17w

250w Snowrunner (no vsync)

150w booting two VMware Workstation VMs (Debian 4cpu 8gb 40gb on nvme raid 0) with 40 Firefox tabs loading

240w Prime95 Blend (8 workers w/HT)
hwm reported split:

CPU package: 118-120w max 122w (120w tdp) spending about 95% of time in C0 and 5% in C3

DRAM 105-110w (reserved 125/128gb)

GPU: 38w

Rest: at least -21w (huh?)

330-350w pl2 Furmark only (160-190w reported GPU power)
300w settled Furmark
hwinfo reported split:

- CPU package 50w
- dram 25w

405w settled P95 blend + furmark
split:

- Furmark reports 130-190w fluctuating constantly
- hwinfo reports:
- 120w CPU
- 113w dram

130w YouTube 1080p60 Firefox Dev Edition
50w CPU
30w dram

150w browsing, Spotify, Steam download

{{</details>}}

{{<details "<h3 style='display:inline;'>ThinkStation C30 (dual Ivy Bridge) - WIP</h3>">}}
{{</details>}}

{{<details "<h3 style='display:inline;'> ThinkStation E32 (H87, i5 4670) - WIP</h3>">}}

These have a USB card reader built into the front of the case. I tested with it disabled, though it didn’t seem to increase idle power usage.

What *does* increase idle power usage is the front 9cm fan, which is rated for 12v .6a. Unplugging it nets about 1.2w of idle savings when the system’s using its noise-optimized thermal profile, but the resulting fan error prevents it from booting normally. Unfortunately, I use these E32s as headless servers, making this a significant point of frustration. Perhaps this could be solved by attacking the BIOS; Lenovo tends to use mostly standard AMT firmware - but that takes effort away from more interesting projects, so until I get really bored I’ll live with the extra watt.

### Run 1: Empty

Idle: 20.5w from the wall

{{</details>}}

{{<details "<h3 style='display:inline;'> ThinkStation P710 (dual Broadwell) - WIP </h3>">}}

This thing is a little odd, but it’s working fine. ‘Production’ hypervisor running Proxmox 8 because this system didn’t like ESXi for some reason (don’t remember the specifics). The HBA isn’t compatible with ESXi, but I added that afterwards and would have passed it to a VM anway.

I like to leave well enough alone, so I won’t be pulling it out of its spot on a wire shelf tucked away from the world and putting Windows on it for benchmarking. I did slip my fake kill-a-watt on it overnight, though.

This bad boy only uses about 95w at idle, which is about 95w lower than I’d expected, considering it has four pieces of spinning rust in it and two CPUs. I suppose the newer architecture and lower capacity DIMMs are good on power and the processors spend 98% of their time in C6 at 1.2 GHz (yes, I checked.)

I could, and should, probably go at this one with `powertop` to try and cut a few more watts off by shutting down stuff like the card reader (Lenovo, card readers, and power consumption on Linux don’t go together nicely, as most ThinkPad T480s owners find out) but for now, just knowing how surprisingly good it’s being is enough for me. I would pull the second CPU, but then I’d lose 1/3 of my memory, and it’s already starved for memory as-is (a large portion is handed to ZFS for cache.) If I do go at it with powertop or swipe a processor out, I’ll probably update this page.

Regardless, this machine is going to be up for replacement shortly as it’s the only one on 24/7 that is not an i5 4670 (and I want it back in my lab because it’s beefy and has a ton of neat toggles in the UEFI to play with.)

{{</details>}}

## Results

### Graphics Cards

A RX 580 contributes about 6.5w of power consumption at idle and 10w when lightly loaded (2D tasks.)

A RX 5700 XT contributes about 38w of power consumption at idle. This is a RDNA issue that has been fairly widely addressed online.

### Conclusion on Platforms

Little Haswell boxes use less power than I thought; drawing 20w means they’ll stay in service for a LOT longer than I thought they would (most Aliexpress Special N100 boxes are around the 10w mark at idle)

Broadwell and Skylake HEDT platforms are surprisingly frugal. 55w for my desktop with 4 32gb DIMMs impressed me, and 95w for a dual 14 core machine with 4 spinners is also surprisingly good.

### Saving Power

It seems that fans are low-hanging fruit for power consumption on devices that don’t need a ton of airflow or can be easily retrofitted with Arctic Px fans (though it would take about four years for you to break even on each individual Arctic P12 if saving .8w per fan and buying a five pack, so.. not a great way to save money, but probably still beats investing the $6 per if the machines are on 24/7.)

Another good way to save power is by cutting down on RDIMMs - if I pull half out of my desktop, I can save ten watts at idle. Not bad, but I’m not going to halve my memory bandwidth for it. Perhaps lower capacity DIMMs are better in that regard - I’ll have to go back and test for that when I have time to do so.

### Generational Power Draw

Idle power consumption per generation should be more or less similar.

Skylake Xeon W: around 50w idle

Broadwell Xeon E5 single socket: around 60w idle

Broadwell Xeon E5 dual socket: probably around 80w idle

Haswell on a H87 in a crappy prebuilt: around 20w idle