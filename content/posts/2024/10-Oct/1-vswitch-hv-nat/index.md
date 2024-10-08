---
title: "Create a vSwitch and a NetNat object to avoid Default Switch DHCP"
date: 2024-10-07T12:34:56-00:00
draft: false
---

Problem: Need a more controlled local testing environment, double NAT/routing hijinks are a bit much

Solution: This plus DHCP running on an internal DC works for now

Tested on: Windows 11 24H2 Ent LTSC with PS 7.4.5

```PowerShell
# create the switch
$SwitchAndNatName = "internal-nat"
$Switch = New-VMSwitch -SwitchName $SwitchAndNatName -SwitchType Internal

# configure addressing for the gateway/hypervisor
$IPAddress = "192.0.2.1"
$PrefixLength = 24
$NetAdapter = Get-NetAdapter | Where Name -eq "vEthernet ($($Switch.Name))"

New-NetIPAddress -InterfaceIndex $NetAdapter.ifIndex -IPAddress $IPAddress -PrefixLength $PrefixLength

# create a new NAT object
New-NetNat -Name $SwitchAndNatName -InternalIPInterfaceAddressPrefix "$($IPAddress)/$($PrefixLength)"
```