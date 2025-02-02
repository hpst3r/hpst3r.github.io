---
title: "Revisited: Working with Hyper-V NAT switches"
date: 2025-02-01T10:12:59-00:00
draft: false
---

## Foreword

This is an update to [a similar post from back in October 2024](/create-a-vswitch-and-a-netnat-object-to-avoid-default-switch-dhcp).

I've been going through and cleaning up a lot of my setup scripts and the like in pursuit of better, faster, more reproducible testing environments.

Occasionally, I want to configure NAT switches for small-scale, self-contained environments. I didn't have anything that made this process nice, so I decided to fix something up.

Of course, before I knew it I'd wound up redoing the original post linked above - so I'd might as well post it!

## Intro

There's a type of vSwitch that the Hyper-V MMC does not let you create - the NAT switch.

Why not? Well, it's not a VMSwitch type, per say - it would be more accurate to describe this as some configuration on the host to forward packets from a subnet on an interface that happens to be attached to an internal vSwitch.

You've probably seen this before if you've ever installed Hyper-V on a Windows 10 or 11 machine - the Default Switch is effectively a NAT switch with some Internet Connection Sharing stuff and address overlap detection going on behind the scenes.

Anyway, having NAT switches is super useful if you're running stuff on your laptop or in an environment you do not entirely control:

- Your wireless adapter probably can't be bridged to a vSwitch.
- Maybe you need an isolated subnet for something that still needs Internet access.
- Perhaps you just want more control over DHCP.
- Or maybe you're getting tired of the Default Switch deciding that it's a different network occasionally.

## Create the switch

Basic config requires an Internal VMSwitch, assigning an IP address to your host machine on said VMSwitch, and then creating a NetNat object on that network.

Here's an example config:

```pwsh
# parameters
$IPAddress = "192.0.2.1"
$PrefixLength = 24
$SwitchName = "vlab0-natswitch"

# create the switch

$Switch = New-VMSwitch `
	-SwitchName $SwitchName `
	-SwitchType Internal

# configure addressing for the gateway (hypervisor OS)

$NetAdapter = Get-NetAdapter |
	Where Name -eq "vEthernet ($($Switch.Name))"

New-NetIPAddress `
	-InterfaceIndex $NetAdapter.ifIndex `
	-IPAddress $IPAddress `
	-PrefixLength $PrefixLength

# create a new NAT object

New-NetNat `
	-Name $SwitchName `
	-InternalIPInterfaceAddressPrefix "$($IPAddress)/$($PrefixLength)"
```

Let's package this up into something a little nicer to use.

This can be found [on GitHub](https://github.com/hpst3r/New-NatVMSwitch).

```pwsh
<#
.SYNOPSIS
Create an internal VMSwitch and configure host to do NAT between it and the outside world.

Optionally, configure forwarding between this NAT network and the WSL VMSwitch.

.PARAMETER IPv4Address
String: The IPv4 address that will be configured on the host's vEthernet interface for the new VMSwitch. Default: '192.0.2.1'.

.PARAMETER PrefixLength
String: The prefix length that will be configured on the host's vEthernet interface for the new VMSwitch, e.g. 16. Default: '24'.

.PARAMETER Name
String: The name of the new VMSwitch. Default: 'Internal NAT'.

.PARAMETER EnableForwarding
Boolean: Enable forwarding on the host's vEthernet interface for this network. Default: $true.
This DOES NOT affect forwarding to things outside the host, just between VMSwitches.

.PARAMETER EnableWSLForwarding
Boolean: Enable forwarding on the WSL interface. Convenience option to allow Ansible or SSH from WSL. Default: $false.

.PARAMETER NetNatName
String: The name of the new NetNat object. Default: same as VMSwitch name ($Name).

.PARAMETER Force
Switch: Overwrite existing VMSwitch and NetNat objects, if applicable.

.EXAMPLE
New-NatVMSwitch `
    -IPv4Address '10.128.252.254' `
    -PrefixLength 24 `
    -Name 'Example Network' `
    -EnableForwarding $true `
    -RouteWSL $true `
    -Force
#>
Function New-NatVMSwitch {
	param(
		[string]$IPv4Address = '192.0.2.1',
		[short]$PrefixLength = 24,
		[string]$Name = 'Internal NAT',
        [bool]$EnableForwarding = $true,
        [bool]$EnableWSLForwarding = $false,
        [string]$NetNatName = $Name,
        [switch]$Force
	)

    try {

        # verify that a VMSwitch does not exist, and/or remove the existing one if -Force is set

        $ExistingVMSwitch = Get-VMSwitch `
            -Name $Name `
            -ErrorAction SilentlyContinue

        if ($ExistingVMSwitch) {

            if ($Force) {

                Write-Information `
                "Attempting to remove existing VMSwitch $($Name) because -Force is set."

                Remove-VMSwitch `
                    -Name $Name `
                    -Force `
                    -Confirm:$false `
                    -ErrorAction Stop
                
            } else {
                
                throw `
                    "A VMSwitch with name '$($Name)' already exists."

            }

        }

        # verify that a NetNat object does not exist, and/or remove the existing one if -Force is set

        $ExistingNetNat = Get-NetNat `
            -Name $NetNatName `
            -ErrorAction SilentlyContinue

        if ($ExistingNetNat) {

            if ($Force) {

                Write-Information `
                    "Attempting to remove existing NetNat object $($NetNatName) because -Force is set."

                Remove-NetNat `
                    -Name $NetNatName `
                    -Confirm:$false `
                    -ErrorAction Stop

            } else {

                throw `
                    "A NetNat object with the name '$($Name)' already exists."
                
            }
            
        }

        # create an internal VMSwitch with requested name

        try {
            
            $Switch = New-VMSwitch `
                -SwitchName $Name `
                -SwitchType Internal `
                -ErrorAction Stop

        } catch {

            throw `
                "Failed to create VMSwitch '$($Name)': $($_)"

        }

        # find the newly created vEthernet adapter (the host's network adapter on the new VMSwitch)

        $NetAdapter = Get-NetAdapter |
        Where-Object Name -eq "vEthernet ($($Switch.Name))"

        if (-not $NetAdapter) {
            
            throw `
                "The virtual network adapter 'vEthernet ($($Name))' could not be found."

        }

        # assign the requested IPv4 address and prefix length to the switch's vEthernet adapter

        try {
            
            New-NetIPAddress `
                -InterfaceIndex $NetAdapter.ifIndex `
                -IPAddress $IPv4Address `
                -PrefixLength $PrefixLength `
                -ErrorAction Stop

        } catch {

            throw `
                "Failed to create NetIPAddress on interface $($NetAdapter.ifIndex): $($_)"

        }

        # create a new NetNat object to configure the host to forward traffic from a network

        try {

            New-NetNat `
                -Name $NetNatName `
                -InternalIPInterfaceAddressPrefix "$($IPv4Address)/$($PrefixLength)" `
                -ErrorAction Stop
            
        } catch {
            
            throw `
                "Failed to create NetNat object: $($_)"

        }

        # Set the Forwarding property of a NetIPInterface to Enabled by alias.

        function Enable-Forwarding {
            param(
                [string]$InterfaceAlias
            )

            $Interface = Get-NetIPInterface `
                -InterfaceAlias $InterfaceAlias

            if (-not $Interface) {

                throw `
                    "The ifAlias $($InterfaceAlias)'s associated interface could not be found. Failed to enable forwarding on interface."

            }

            Set-NetIPInterface `
                -InputObject $Interface `
                -Forwarding Enabled
            
        }

        # if enabling forwarding (on the new VMSwitch, to other VMSwitches) is desired, do so using the above wrapper

        if ($EnableForwarding) {

            Enable-Forwarding `
                -InterfaceAlias "vEthernet ($($Name))"

        }

        # if enabling forwarding (WSL to other VMSwitches) is desired, do so using the above wrapper

        if ($EnableWSLForwarding) {

            Enable-Forwarding `
                -InterfaceAlias 'vEthernet (WSL (Hyper-V firewall))'

        }

    } catch {

        Write-Error `
            "Script terminated with error: $($_)"

    }

}
```

This script will create a VMSwitch and NetNat object, and can optionally reconfigure the WSL VMSwitch.

```pwsh
PS C:\Users\liam> get-netnat

Name                             : Internal NAT
ExternalIPInterfaceAddressPrefix :
InternalIPInterfaceAddressPrefix : 192.0.2.1/24
IcmpQueryTimeout                 : 30
TcpEstablishedConnectionTimeout  : 1800
TcpTransientConnectionTimeout    : 120
TcpFilteringBehavior             : AddressDependentFiltering
UdpFilteringBehavior             : AddressDependentFiltering
UdpIdleSessionTimeout            : 120
UdpInboundRefresh                : False
Store                            : Local
Active                           : True

PS C:\Users\liam> sudo pwsh
PowerShell 7.5.0
PS C:\Users\liam> get-vmswitch -Name 'Internal NAT'

Name         SwitchType NetAdapterInterfaceDescription
----         ---------- ------------------------------
Internal NAT Internal

PS C:\Users\liam> get-netipinterface | where InterfaceAlias -like *WSL* | select Forwarding

Forwarding
----------
  Enabled
  Enabled
```

To clean up after it:

```pwsh
# delete the NetNat object
PS C:\Users\liam> get-netnat -name 'Internal NAT' | remove-netnat -confirm:$false

# delete the VMSwitch
PS C:\Users\liam> get-vmswitch -Name 'Internal NAT' | remove-vmswitch -confirm:$false

# unset forwarding on the WSL interface, if you configured it
PS C:\Users\liam> get-netipinterface | where InterfaceAlias -like *WSL* | set-netipinterface -forwarding Disabled
```
