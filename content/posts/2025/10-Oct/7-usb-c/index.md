---
title: "USB-C laptop docks"
date: 2025-10-29T20:30:00-00:00
draft: false
---

What a mess. I miss old port replicators.

## TL;DR

New laptop with Thunderbolt or USB4 + want 3+ high-res displays? Dell WD22TB4 or Lenovo 40B0

AMD without USB4? Lenovo 40AS, Dell WD19TB or WD19s

Mac? Caldigit/OWC, or 40AS/WD19TB/WD22TB4 to save a buck. Desperate for screens? DisplayLink. It's complicated

Cheap? Dell WD19TB/Lenovo 40AC

## Standards

USB-C w/ DisplayPort alt mode and power delivery: your laptop almost certainly has this, unless it is quite literally the cheapest thing on the planet.

One thing to keep in mind is that how *much* DisplayPort you get varies wildly by model of laptop; until very recently lower-end Dells (Latitude 3000 series) only supported DisplayPort 1.2 over USB-C, so they could push at most a single 4K, 60 Hz monitor over USB-C. The current Dell Pro series (excepting the very new Pro Essential machines that really, really suck) do not suffer from this limitation.

Thunderbolt: your laptop probably has this if it's got an 11th generation or newer Intel chip in it (the controller is built-in), or if it's a Mac with USB-C. Older Windows laptops with 7th generation chips and newer *may* have Thunderbolt 3, but it'll likely be capped to DisplayPort 1.2 thanks to an older external Alpine Ridge Thunderbolt controller and will not satisfy your single-cable, high-resolution demands. If this is the case, pick up a Dell 5420 or 7420 for $150 off eBay and carry on. Ports and cables are likely labeled with a lighning bolt.

USB4: non-certified (by Intel) Thunderbolt 4. Most commonly found in devices with a Ryzen 7000 series (Zen 4) or newer AMD processor (the controller is built-in), though some Ryzen 6000/Zen 3+ devices have USB4, too. YMMV. Most business-class notebooks from Lenovo or Dell using the AMD controller have a good implementation, and will work properly with Thunderbolt peripherals.

## Cables

Cable quality matters. A lot. You will have a very, very bad time if you do not use a good cable (do keep in mind that, if you're running Thunderbolt, you're trying to push 20 or 40 Gbps over this little cable). Get a certified Thunderbolt cable if possible. Do not get a long one unless you are willing to spend up for an active one (you will need to spend up to get an active cable).

If possible, get one with your dock. Even short Thunderbolt cables are pricey. USB4 cables can be a decent stand-in, but quality varies. You *may* be able to run Thunderbolt (likely at 20 Gbps) over a good 10 or 20 Gbps USB-C cable, but YMMV, and I wouldn't recommend it.

## Normal docks

The new 2025 docks haven't been out for long enough for me to memorize the number spaghetti, so I'm not going to talk about them. If you need a Thunderbolt 5 dock you're going to be spending $500.

### High resolution displays

The current "best dock" that doesn't cost a million dollars is a refurbished semi-modern one supporting Thunderbolt 4 and Display Stream Compression. These can do up to four 4K 60 Hz displays and usually have one or two Thunderbolt 4 downstream ports for other peripherals, like high-speed networking (though YMMV with the amount of bandwidth left over).

My choices would be either the **Lenovo 40B0 Universal Thunderbolt 4 dock** (power button seems to work with more brands of laptop, detachable cable, can add a special cable to charge 230w slim tip machines) or **Dell WD22TB4** (works better with machines that don't have Thunderbolt and rely on USB-C as the dock itself is USB-C only, power button/status light only works with Dells, much cheaper off eBay, two downstream Thunderbolt ports).

I have one of each (40B0 at the office, since I had a P1 Gen 4 and wanted a "single cable" solution, and a WD22TB4 at home, since I've got a bunch of AMD machines that lack Thunderbolt); both work well.

The 40B0 does *not* play nicely with non-Thunderbolt (or USB4) machines, and is missing a downstream TB4 port, so I would prefer the Dell.

### Cheap docks

If you need a cheap Thunderbolt dock, get a Lenovo 40AC (TB3 dock Gen 1) or Dell WD19TB (~$30).

The 40AC will NOT play nicely with USB-C only devices. The WD19TB may, but YMMV. The 40AC is also limited to two monitors via HDMI or DisplayPort; the third will need to use VGA or USB-C (via the downstream Thunderbolt port). The 40AC works great with Macs.

If you need a cheaper USB-C only (not Thunderbolt) dock for your AMD laptop that can do a pair of 4K monitors or 3 1440p monitors, get the Dell WD19S or Lenovo 40AS (USB-C dock Gen 2) (~$40).

### Special USB-C PD (and weird Lenovo break-apart cables)

If you have a proprietary 135w Dell or Lenovo (e.g., P14s, P16s, Precision 5000 series, XPS 15), get their dock. The other brand's offering won't give you enough juice to avoid the slow charger nag, since their 135w PD standards are proprietary.

If you have an older (not modern USB-C PD for all 180w) Dell workstation, you need the dual USB-C Dell WD19DC. If you have an older (read: not brand new, slim tip) Lenovo workstation, get a 40B0, the 4X91K16970 cable, and a 230+w AC adapter. These 'specialty' docks will probably run you around $100 - 150.

## Macs

Macs play nicely with Thunderbolt and less nicely with USB-C. You might wind up stuck mirroring your displays unless you can daisy-chain USB-C devices from your dock. This is an Apple Silicon limitation.

If you only have a Mac, and price is no object, get an OWC or Caldigit dock and plan on connecting things via USB-C.

The number of displays an Apple Silicon Mac supports varies wildly by the specific chip you have.

If "including the internal screen" is listed, the iGPU does not support shutting the laptop lid or disabling the internal screen to enable another external display. The M3 is the only chip to which this limitation does not apply. Desktops (e.g., the Mac Mini and Mac Studio) typically have this 'internal screen' display output routed to their HDMI port (so you must use the HDMI port to use the maximum number of independent displays).

### Base chips

- M1 and M2: 2 displays, including the internal screen
- M3: 2 displays
- M4: 3 displays, including the internal screen

### Pro chips

- M1, M2, M3, M4 Pro: 3 displays, including the internal screen
- M1, M2, M3, M4 Max: 5 displays, including the internal screen
- M1 Ultra: 5 displays
- M2, M3 Ultra: 8 displays

### Getting around display limitations

If you want to add screens to your lower-end Mac, you can do so with a DisplayLink dock like the Dell D6000. These will software-render (e.g., use the CPU rather than the GPU) to additional screens for you.

Any motion or graphics are probably a no-no here, and the DisplayLink Manager software is a bit buggy on Mac OS (works much better on Windows).

A good rule of thumb is that this won't be a good experience if the resolution you're trying to use requires scaling. Support for scaling is poor and performance is even worse. 4K displays are too much, and 1440p is probably pushing it. 1080p is more or less OK.
