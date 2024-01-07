---
title: "Why I Stopped Using Debian (on my desktops)"
date: 2024-01-02T21:57:32-06:00
draft: true
---

## Foreword

This is the ultimate expression of a first-world problem. I do not expect you, dear reader, to finish reading this and say "wow, this internet guy is a very reasonable individual with his priorities in order." But **it** annoyed me enough to get me to slowly phase out all of my Debian desktops and replace them with Fedora.

TLDR: "turning Debian into a rolling release desktop distro takes some effort and I'm lazy," or "use the best tool for the job" depending on your attitude

## What?

I've been a Debian devotee for the last couple of years. The combination of the sheer size of the community of Debian and its derivatives (so you can fix something when it goes wrong), what amounted to a *guarantee* of stability in a major release (so it doesn't go wrong in the first place), and a sensible choice of software (ifupdown, not networkmanager, in a minimal install, no snap by default, an unmolested GNOME desktop, so you don't have to change stuff to get it back to how it's supposed to work) made it perfect for me. Setup was a bit ardurous, sure, but once I had Debian set up it more or less just worked. Worked great. I still love Debian, and I still use Debian anywhere I need stability. I'll expand on that a bit.

"Once I had it set up." My setup script was 150 lines of garbled bash, and, while it was continuously expanded by yours truly every time I deployed it, that great ol' mess didn't do everything. Why was I using a great old mess of a setup script? Well, bear with me here.

See, when you use the Debian installer, if you select the GNOME desktop, it installs the GNOME desktop. All of it. Just about every GNOME project package. Plus LibreOffice, and a bunch of other programs. LibreOffice alone made apt upgrades take twice as long as I wanted them to.

Because Debian has a fairly long release schedule, one might ask "why does how long an update takes matter?"

Because Debian's release schedule and commitment to stability got in my way a few times, so I defaulted to setting up Sid on anything where I'd log in and try to work. Granted, Debian Sid is still impressively stable (I've had a better experience on Debian's bleeding edge, Sid with experimental packages, than I did with mainline non-LTS Ubuntu releases) but you pay for the latest versions of packages. And one of the victims of getting the latest versions of packages is running Debian Stable. Which starts to defeat the point.

At this point, I was installing Debian by:
- Installing a minimal system from either a PXE server or flash drive
- Running a first stage setup script to install prerequisites for my apps and a base desktop
- Rebooting the new machine
- Running a second stage setup script to install and configure my commonly used apps (download and install, set up services for the stuff I wanted to run at boot, etc) and configure GNOME via gsettings
- Manually finishing up my installation by configuring syncthing (for my password manager and a /sync directory), git, signing in to Github, my Google accounts, etc.

Frankly, this was NOT quick, which matters, because I reinstall a lot (I tend to switch my machines between Windows and Linux as needed.) Anyway, long list of minor niggles with Debian, mostly related to out-of-the-box config, even though it worked (and still works) great. So why the breakup album?

## Just Plain Ease of Use

I happened to install Fedora once. Oops. I suppose this is what Mac people mean when they argue for the productivity benefits.

What do you get out of the box?
- a nearly minimal GNOME desktop, with no 30 clones of games (I suppose LibreOffice can be excused, but it's still there)
- flatpak already set up
- dnf, the only package manager that I like as much as apt (it's so fancy!)
- a large community (albeit not as large as Debian's) who are generally more advanced users
- faster, generally stable, release intervals (interface changes only once every six months, but you still get updates)
- it's Red Hat (the RHCSA is worth something)

My install script was able to become:
1) a gsettings bit
2) an installing software bit

I don't need prep work anymore. Total install time is roughly halved. I'm happy, and it's not really much different than any other Linux distro. Plus, I have a good live environment now that matches the distro I use most.

I still think Debian is a better fit for a server (I don't care to spend the time setting up licensing on machines in my lab, where I don't pay for enterprise support, and I don't want to deal with a rolling-release CentOS) so it'll remain in day-to-day use.

Perhaps when I have the time to breathe I'll revisit Debian and put some work into the installer. And I'm staying on the debian-users mailing list, because it's fun to read sometimes. But, for now, Fedora Just Works(TM).