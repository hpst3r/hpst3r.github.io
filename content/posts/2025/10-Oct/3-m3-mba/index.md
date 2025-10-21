---
title: "Quick thoughts on M3 15\" Mac Air"
date: 2025-10-20T23:30:00-00:00
draft: false
---

Imagine a picture of a Mac here. I was too lazy to take one.

## Backstory

Short version:

I broke the embedded DisplayPort connector on my ThinkPad P1 G4 (too much use?) and wound up buying a MacBook Air after briefly going back to Linux (and remembering how much I don't like Windows desktops).

Long version:

I swapped too many motherboards in and out of my P1 Gen 4. The embedded DP connector on my original (and only working) board, with a RTX A2000/i7-11800H, stopped holding the cable in properly. After some tape didn't really fix the issue and my screen kept flickering like mad, I switched to a T14s Gen 1 AMD that I've had on backup duty for a little while. It's far too slow to run Windows 11, so I wound up running Fedora 42 with the GNOME desktop environment.

I've used Fedora for a long time, on both servers and my desktops or laptops. It's never been as stable as I would like, but I think GNOME's QC has gone downhill as of late or there are some extra bugs in `amdgpu` this time around. Anyway, after the third time in about as many hours that my GNOME session crashed (taking all my work with it), I went to eBay and bought a Mac out of frustration.

(The T14s is now running NixOS + KDE, and it's a much better combo - really nice laptop, with the exception of the screen, but finding decent software to put on it is difficult.)

## Motivation

I like Linux. It's much more ergonomic to use than Windows is (traditional shell, much faster, less overcomplicated), updates are about twenty times faster, and there's much less telemetry sent off. I find both OSes equally unstable on the desktop, which I put up with for a while.

As of late, I've been very pressed for time to use for my own things, and I don't mind spending a bit of money to reclaim time that I could use for valuable things (like studying, reading, or, heaven forbid, going outside). Basically, I've found that the ROI on an hour of studying for a certification or working on a POC for a deployment in $RealWorld is far higher than the ROI on an hour of dinking with my PC.

I recently borrowed a family member's base model M1 Mac Air to do some managing of iOS. The little Mac was completely silent, relatively fast (for a five-year-old fanless laptop, it's a screamer) and I didn't experience a single bug, hiccup or crash during the week or two I wound up using it.

So, when I had some issues with my own laptops, rather than figure something out (the smart choice) I decided to light money on fire.

## The laptop

My machine is a 2024 15" MacBook Air with the M3 chip, 16 GB of memory, and 512 GB of disk. I'm used to having four times the memory and storage.

My main considerations were:

- single core performance
- good, fairly large display
- good battery life
- good, stable OS with as little driver trouble as possible (not Windows)
- working, stable display scaling

The alternatives I considered were:

- ThinkPad P1 Gen 7
  - I was looking for absolutely no Nvida, preferably AMD CPU + iGPU, nice 16" screen ThinkPad with a centered keyboard. They don't make such a thing. The Ultra 7 155H isn't particularly fast, and used P1 G7s are very expensive.
- Asus ProArt P16
  - Briefly considered it, then remembered it's an Asus, so it would probably last a week. And none of the shortcut keys will work if I put Linux on it.
- M2 Max MacBook Pro 16"
  - Would be nice if I could run more than two external displays. However, the internal mini-LED screen's backlight has a high-frequency flicker, which generally doesn't play nicely with my eyes.

The newer Mac Airs don't have the backlight flicker issue, so I turned my attention to them. I found this one for $800 - about $500 less than the cheapest M2 Max 16" I could find, though that M2 Max obviously had more memory.

## Thoughts at end of first day

The screen is good, but not quite as good as I was expecting. I guess I'm too used to the oversaturated lack-of-clamping Adobe RGB panels. It's easily as good as the P1, though.

The speakers are good, but not great. For some reason (design) there are no exposed speaker grilles, which certainly negatively impacts sound quality. They get pretty loud, though.

The enormous haptic trackpad with a glass surface is wonderful, though I do miss the TrackPoint and its physical buttons (when clicking and dragging, the physical left mouse button makes a big difference). The keyboard is OK - the layout isn't very good. Notch is distracting. No idea of battery life yet.

The OS is fine. I dislike window management.

Default terminal app isn't great but is much faster than Windows Terminal. iTerm2 is fine, though.

## Thoughts at end of first five(?) days

Speakers are actually not very good. My P1 G4 is better.

The Mac trackpad is still wonderful. I still haven't quite made my mind up on the keyboard - it's OK. Not good. The layout kind of sucks, and key travel is nonexistent, but keys feel uniform and are relatively tactile. I prefer my external keyboards or P14s.

The battery life and performance are both absolutely incredible. No fan, no heat for this level of performance is mindboggling. I've been getting between 10 - 16 hours on a charge.

I've installed a few things to take care of some Mac annoyances:

- Rectangle for window tiling
- AltTab to replace macOS's terrible per-app cmd+tab switcher
- Scroll Reverser so my mice work properly
- Stillcolor to get rid of the GPU dithering (rapidly flickering pixels to make it appear that the display shows more colors than it actually does).

Other software I've been using is:

- Obsidian
  - I live in this app
- Visual Studio Code
- Spotify
- 1Password
- Chrome
  - Main web browser. I don't like Safari.
- iTerm2
  - Much better than the built-in Terminal app.
- Anki
- Tailscale
- Remote Desktop Manager (Devolutions)
  - Use this to RDP to my Windows jumpboxes/workstations.
- IINA
  - Video player.
- Pages
  - Good enough for opening DOCX files, LibreOffice for Mac is a buggy mess
- Numbers
  - Good enough for opening CSV files; Office is gigantic so I'd rather not install it
- PowerShell Core
  - Works fine for what I need on my computer (management modules, Graph API)
- Python
  - Works great since I'm running Unix, no WSL required
- Podman
  - Desktop app, since it gives you a nice icon when the VM is running, plus the VM and tools
- Hugo

I love how all the menus, dialog boxes, and clickable widgets are consistent *everywhere*. On every other platform it's an inconsistent disaster.

The webcam is great. The microphone is very good for something built in to a laptop.

The vertical height on the built-in 15.3" screen is lovely. The taskbar/icon tray/file menus/clock are out of the way, up near the notch, and I'm left with a full 15" 16:10 screen for the things I care about. I don't notice the notch's prescence. The screen is noticeably larger than the 14.5" panel in my work laptop (a ThinkPad P14s Gen 5i) and it definitely helps me get more done. 15" isn't quite as luxurious as a 16" or 17" display, of course, but the 15" Mac Air is a *very* portable laptop.

Thank goodness there's no number pad and the keyboard isn't offset.

The device wakes instantly, never uses much power, and can sit in standby for what feels like an eternity then come back with 2% less charge.

## Thoughts at the end of the first month

I've had some time to use this device now.

It is, by far, the best laptop computer I've ever used.

Pros:

- Completely silent (since it has no fan)
- It's got endless battery life (I can reasonably expect 10 - 16 hours of use, depending mostly on display brightness - easily enough for a full day of work, plus enough to get me through the evening and most of the next morning)
- Is still faster than the Core Ultra 7 155H in my same-age Windows laptop (while getting 4x the battery life, with a smaller battery)
- Incredibly stable software
- Has an excellent screen
- Doesn't run Windows (major plus) and is Unix, which is good enough for me

Cons:

- Keyboard feel and layout
- Feels extremely fragile
- Impossible to repair or upgrade
- Expensive
- Doesn't run Linux with systemd (NixBook when?)
- No USB-A or HDMI (though I think it's probably too thin for either)

Rambly notes:

The macOS is much kinder on disk than Windows is (with MSI installers preserved forever, a gigantic Windows Update cache, and with just the OS and my software ballooning wildly to 200+ GB once I've run my setup script and updated it a few times). I found that my software and data only really use about 70 GB of disk (90 GB total, including the immutable system partition). 512 GB should be perfectly adequate.

I have not once had to think about drivers, suspend, finding a charger, or an OS-related bug. The only things I don't like about the OS are the (lack of) sensible keyboard shortcuts for window management and the horrid built-in terminal app (iTerm2 to the rescue).

Mac font rendering looks horrible on a display that isn't high DPI, but looks lovely on anything that has enough pixels (because it actually gives you the font, not some antialiased, pixel-aligned monster). I have had no complaints with how scaling looks or performs on external 4K monitors.

The keyboard is possibly my least favorite part. If Apple could poach the persons responsible for the design of the P14s G5i keyboard, that would be excellent. The layout sucks, the travel sucks, the feel sucks. But it's fine. I'd rather have this than a Latitude keyboard, and my older T14 G2s have much worse typing surfaces.

The trackpad is lovely. All there is to it. Tracks accurately, is gigantic, palm rejection works flawlessly.

No complaints about software. Have had zero issues with software compatibility. The only thing I needed Rosetta for was installing the Steam launcher so I could launch an Apple Silicon-native game (Disco Elysium) that I wanted to play while lying in bed.

This has quickly become my main computer. I prefer it to everything else I own, including:

- an i9-12900KS/64GB/4TB/6800 XT desktop running Windows 11 and Fedora Linux
  - The Mac feels faster and has fewer software issues, let alone the fact that it's a fanless laptop that draws practically no power.
- several Zen 2/Zen 3 Ryzen 7/32GB/1TB ThinkPads running Fedora
  - The Mac has a much nicer screen, much better battery life, and is much more stable than any of these.
- My work laptop, a ThinkPad with a 16 core Ultra 7 155H (6p+8e+2lpe), 64GB of memory, and a 3072x1920 120hz IPS display with great colors
  - The Mac feels much faster, has a nicer screen, runs a better OS, lasts far longer on a charge, wakes in about a tenth of the time, and has fewer software issues. The Nvidia driver on this P14s put me through hell every time I dared wake the device from suspend until 580 released a few weeks back.

And Bluetooth actually works, too.
