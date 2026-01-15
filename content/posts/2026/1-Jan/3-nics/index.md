---
title: "1, 10, 25, 40 gig NIC shopping list"
date: 2026-01-14T20:45:00-00:00
draft: false
---

## Gigabit

- BCM5719 (quad-port Broadcom PCIe 2.0x4, ~$15, SR-IOV, VMq)
- BCM5720 (dual-port Broadcom PCIe 2.0x1, ~$10, SR-IOV, VMq)

Cheap OEM models of Broadcom NICs:

- HP 331T (~$10 - 12)
  - HP 332T - BCM5720 (<$10, some for ~$5)
- Dell 0YGCV4 (not cheap, but included for completeness, ~$15)

- Intel i350-T4 (~5w, PCIe 2.0x4, supports SR-IOV and VMq, ~$30)
- i340-T4 (~5w, PCIe 2.0x4, lacks SR-IOV, supports VMq, ~$20)
- PRO/1000 ET(2) (12-15w, quad-port, 8-queue per-port VMq, SR-IOV, ~$15) if you MUST have Intel for cheap

## 10 Gigabit

X540-AT2 for Base-T. Get a later X550 if you need multigig. X520-DA2s are solid SFP+ cards but I prefer Mellanox (a ConnectX-4 LX).

HP 546SFP+ are ConnectX-3 Pros for cheap.

562SFP+ are X710-DA2 for cheap.

## 25 Gigabit

640SFP28 - HPE part number for the dual SFP28 CX4 LX. Normal SFP28 CX4 LX listings are a little pricey.

## FDR Infiniband and 40 Gigabit Ethernet (VPI)

ConnectX-3 Pros are the only cards cheap enough to be worth consideration. ~$15 for dual port 56g Infiniband/Ethernet. Utilities no longer support them, so you'll have to source older versions. Don't expect working ASPM.

HP calls them the "HP InfiniBand 544+" (544+FLR-QSFP)

## 40 Gigabit Ethernet

Single-port QSFP ConnectX-4 LX cards are cheap (~$20) and very good. Supposedly support ASPM with newer firmware.
