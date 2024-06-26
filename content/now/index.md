+++
title = "/now"
+++

Saw [Jamie Tanna's /now page](https://www.jvt.me/now/) and liked the idea.

## What exactly am I doing?

Upd: Mar 17, 2024

### Goals

**Short-term:**

~~CCNA~~ Done! 3/14/2024

AZ-104

**Medium-term:**

RHCSA

### Interests

- Sim racing
- Fast computers
- Cool software
- Bulleted lists

### Hardware & Software

What kind of junk am I running now?

1. Desktop
    - $350 ThinkStation P520 build (bought 2023, hardware ~2016)
        - Xeon W-2140B, Radeon 5700xt, 4x32gb RDIMMs
        - Windows 10 Enterprise LTSC 2021 (broken, pending reinstall)
        - Fedora 39 (GNOME)
        - Getting a little long in the tooth, but still decently performant.
2. Laptop
    - ThinkPad T14 G2a (bought 2023, hardware ~2022)
        - R7 5850u, 2x16gb DDR4 3200
        - Fedora 39 (GNOME)
    - ThinkPad T480 (bought 2022, hardware ~2018)
        - i5 8350u, 2x16gb DDR4 2133
        - Windows 11
3. Phone
    - Pixel 6a (2022)
        - Tensor G1, 6gb
        - Google Android 15
4. Peripherals
    - Monitors
        - Dell P2418d x3 (1440p 24")
            - These are a good balance of size and resolution for me. 120ppi is sharp enough and looks decent while being usable without any scaling at the OS level. I quite like these, they provide a lot of screen real estate with a small footprint and the three of them cost me less than my one 4k monitor.
        - Dell S2722QC x1 (4k 27")
            - Fairly nice looking monitor, working around fractional scaling isn't fun though
    - Keyboard
        - Cherry Stream TKL low profile membrane board. Easier on my fingers than a mechanical keyboard
    - Mouse
        - Razer Basilisk X Hyperspeed - not playing FPS games often anymore so I don't mind the weight. Battery lasts forever.

### Home Network

- Firewall/gateway/hypervisor: ThinkStation E32
    - ZFS mirror on two Intel DC S-somethingorother
    - plain Debian 12 with some virtual networking and VMs to provide an Opnsense firewall and UniFi controller in one box. Has been solid, does what I need. No complaints.
- APs: UniFi U5 Pro
    - I would almost prefer an autonomous AP (because I'm only using one) but I can't complain about much considering how cheap used last-gen UniFi gear is and how easy it would be to add a second AP.
- Switching: a single HP managed gigabit L2.5 8 port from circa 2006. Anything 'lab' is virtualized or emulated