+++
title = "Hello!"
+++

## About me

I'm a network/sysadmin, currently employed at a small MSP. I spend my free time as a voluntary computer janitor and amateur stuff-breaker.

If you'd like to get in touch with me, I'm reachable by email at [contact@wporter.org](mailto:contact@wporter.org).

## About the site

This site is built with Hugo on GitHub Actions and is hosted on GitHub Pages (so I can add content and rebuild with a commit.) I am a big fan of this setup. Some day I'll move it on-prem.

The theme I'm using is [my fork of](https://github.com/hpst3r/etch) [Lukas Joswiak's etch](https://github.com/LukasJoswiak/etch).

The fork is available [on GitHub here](https://github.com/hpst3r/etch).

## My machines

### Clients

- Custom PC (i9-12900KS, 64gb, 5tb of NVME) - W11 LTSC 2024, NixOS unstable
- Macbook Air (M3/15"/16/512)
- Pixel 9 Pro XL (Tensor G4, 16gb, 512gb) - GrapheneOS

### Lab

In active use:

- Primary Proxmox VE cluster
  - 3x HP Z2 G4 SFF (E-2126G, 64gb, 480gb + 480gb OSDs + 120gb boot + 512gb scratch NVME, PVE)
  - 1x custom Supermicro tower (2x E5-2667 v4, 256gb, 3.2tb NVME + 480gb + 480gb OSDs + 120gb boot + 12tb backup, PVE)
  - 1x ThinkStation E32 backup server (i7-4770, 16gb, 240gb boot + 4tb data, PVE + PBS)
  - 1x ASRock/Datto NUC offsite backup server (Celeron, 16gb, 16gb boot + 5tb data, PBS)
- Secondary PVE cluster
  - 2x M920q firewalls (Proxmox VE + Opnsense)
  - Witness on primary cluster
- Tertiary PVE cluster (at remote site)
  - 2x Lenovo T14 Gen 2 AMD (R7 5850u, 48gb, 1tb NVME) w/o HA, with ZFS replication
- Miscellaneous
  - 2x HP 8th gen minis (i7-8700T, i5-8500, 32gb, 512gb of NVME)
- Networking
  - Lenovo M715q (R5 2400GE, 16gb, 128gb NVME)
  - 2x Cisco 3850-48T
  - Mellanox SX6036
- Proxies/public webservers
  - Oracle Cloud arm64 VPSes
  - IONOS VPSes

Out of circulation:

- 2x Supermicro CSE-813 + X10DRL-i (stripped)
- 3x Supermicro 6028 (dual 2698 v3, 256/128gb)
- ThinkStation C30 (dual 2680 v2, 128gb)
