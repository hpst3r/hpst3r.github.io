---
title: "ipmitool"
date: 2025-11-23T20:30:00-00:00
draft: false
---

## TL;DR

Configure networking:

```sh
sudo dnf install -y ipmitool || nix-shell -p ipmitool
sudo ipmitool lan set 1 ipsrc static
sudo ipmitool lan set 1 netmask 255.255.255.0
sudo ipmitool lan set 1 defgw ipaddr 0.0.0.0
sudo ipmitool lan set 1 ipaddr 192.0.2.10
sudo ipmitool mc reset warm
```

List users:

```sh
sudo ipmitool user list
```

Set a password:

```sh
read -s password
sudo ipmitool user set password 1 "$password"
```

Get network info:

```sh
sudo ipmitool lan6 print
sudo ipmitool lan print
```

Get reported power consumption:

`ipmitool` is a utility for interacting with system BMC/IPMI controllers/management interfaces.

## Detailed

### Installation

On a RHEL-like platform, it can be installed with:

```sh
sudo dnf install -y ipmitool
```

On NixOS, you can get it in a `nix-shell`:

```sh
nix-shell -p ipmitool
```

### Channels

HPE G8 iLOs typically represent the Ethernet interface as channel 0x2:

```txt
[wporter@server ~]$ sudo ipmitool channel info 0x2
Channel 0x2 info:
  Channel Medium Type   : 802.3 LAN
  Channel Protocol Type : IPMB-1.0
  Session Support       : multi-session
  Active Session Count  : 0
  Protocol Vendor ID    : 7154
  Volatile(active) Settings
    Alerting            : enabled
    Per-message Auth    : disabled
    User Level Auth     : enabled
    Access Mode         : always available
  Non-Volatile Settings
    Alerting            : enabled
    Per-message Auth    : disabled
    User Level Auth     : enabled
    Access Mode         : always available
```

All available channels are `0x0` ... `0x9`, then `0xE, 0xF`.

### Resetting the BMC

For a 'warm' restart of the BMC (don't reboot the server), use the `ipmitool mc reset warm` command:

```txt
$ sudo ipmitool mc reset warm
Sent warm reset command to MC
```

This will usually take a few minutes.

For a full "cold" restart of server and BMC:

```txt
$ sudo ipmitool mc reset cold
```

### Finding network info

To get IPv4 info from the OS, use `ipmitool lan print`:

```txt
$ sudo ipmitool lan print
Set in Progress         : Set Complete
Auth Type Support       :
Auth Type Enable        : Callback :
                        : User     :
                        : Operator :
                        : Admin    :
                        : OEM      :
IP Address Source       : Static Address
IP Address              : 0.0.0.0
Subnet Mask             : 0.0.0.0
MAC Address             : 58:20:b1:ff:ff:ff
SNMP Community String   :
BMC ARP Control         : ARP Responses Enabled, Gratuitous ARP Disabled
Default Gateway IP      : 0.0.0.0
802.1q VLAN ID          : Disabled
802.1q VLAN Priority    : 0
RMCP+ Cipher Suites     : 0,1,2,3
Cipher Suite Priv Max   : XuuaXXXXXXXXXXX
                        :     X=Cipher Suite Unused
                        :     c=CALLBACK
                        :     u=USER
                        :     o=OPERATOR
                        :     a=ADMIN
                        :     O=OEM
Bad Password Threshold  : Not Available
```

For v6 info, use `ipmitool lan6 print`:

```txt
$ sudo ipmitool lan6 print
Getting parameter(s)...
IPv6/IPv4 Support:
    IPv6 only: no
    IPv4 and IPv6: yes
    IPv6 Destination Addresses for LAN alerting: no
IPv6/IPv4 Addressing Enables: both
IPv6 Status:
    Static address max:  4
    Dynamic address max: 8
    DHCPv6 support:      yes
    SLAAC support:       yes
IPv6 Static Address 0:
    Enabled:        no
    Address:        ::/0
    Status:         disabled
IPv6 Static Address 1:
    Enabled:        no
    Address:        ::/0
    Status:         disabled
IPv6 Static Address 2:
    Enabled:        no
    Address:        ::/0
    Status:         disabled
IPv6 Static Address 3:
    Enabled:        no
    Address:        ::/0
    Status:         disabled
IPv6 Dynamic Address 0:
    Source/Type:    SLAAC
    Address:        fe80::5a20:ffff:feff:ffff/64
    Status:         active
IPv6 Dynamic Address 1:
    Source/Type:    SLAAC
    Address:        ::/0
    Status:         disabled
IPv6 Dynamic Address 2:
    Source/Type:    SLAAC
    Address:        ::/0
    Status:         disabled
IPv6 Dynamic Address 3:
    Source/Type:    SLAAC
    Address:        ::/0
    Status:         disabled
IPv6 Dynamic Address 4:
    Source/Type:    DHCPv6
    Address:        ::/0
    Status:         disabledsc
IPv6 Dynamic Address 5:
    Source/Type:    DHCPv6
    Address:        ::/0
    Status:         disabled
IPv6 Dynamic Address 6:
    Source/Type:    DHCPv6
    Address:        ::/0
    Status:         disabled
IPv6 Dynamic Address 7:
    Source/Type:    DHCPv6
    Address:        ::/0
    Status:         disabled
IPv6 DHCPv6 Timing Configuration Support: not supported
IPv6 Router Address Configuration Control:
    Enable static router address:  yes
    Enable dynamic router address: yes
IPv6 ND/SLAAC Timing Configuration Support: not supported
```

### Configuring addressing

To set an IPv4 address:

```txt
$ sudo ipmitool lan set 1 ipsrc static

$ sudo ipmitool lan set 1 ipaddr 192.0.2.10
Setting LAN IP Address to 192.0.2.10

$ sudo ipmitool lan set 1 netmask 255.255.255.0
Setting LAN Subnet Mask to 255.255.255.0

$ sudo ipmitool lan set 1 defgw ipaddr 192.0.2.1
Setting LAN Default Gateway IP to 192.0.2.1
```

### Setting passwords

To reset the password for the `admin` user (some vendors like HP set a random password on the pull-tag on the front of the server; you may not have access to these):

First, list users. The specific channel varies; on a HPE G8 it's 2. On most other servers, it's 1.

```txt
$ sudo ipmitool user list 2
ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
1   Administrator    true    false      true       ADMINISTRATOR
2   (Empty User)     true    false      false      NO ACCESS
3   (Empty User)     true    false      false      NO ACCESS
4   (Empty User)     true    false      false      NO ACCESS
5   (Empty User)     true    false      false      NO ACCESS
6   (Empty User)     true    false      false      NO ACCESS
7   (Empty User)     true    false      false      NO ACCESS
8   (Empty User)     true    false      false      NO ACCESS
9   (Empty User)     true    false      false      NO ACCESS
10  (Empty User)     true    false      false      NO ACCESS
11  (Empty User)     true    false      false      NO ACCESS
12  (Empty User)     true    false      false      NO ACCESS
```

For best results, use a password without special characters that's at most 19 characters long (< 20 bytes). To set the user's password:

```txt
$ sudo ipmitool user set password 1 foobarbaz
Set User Password command successful (user 1)
```

If you want to not have the password hanging out in your bash history:

```sh
read -s password
sudo ipmitool user set password 1 "$password"
```

### Read sensors

HPE:

```txt
$ sudo ipmitool sdr list
UID Light                | 0x00              | ok
Sys. Health LED          | no reading        | ns
01-Inlet Ambient         | 22 degrees C      | ok
02-CPU 1                 | 40 degrees C      | ok
03-CPU 2                 | 40 degrees C      | ok
04-P1 DIMM 1-3           | 37 degrees C      | ok
05-P1 DIMM 4-6           | 39 degrees C      | ok
06-P1 DIMM 7-9           | 39 degrees C      | ok
07-P1 DIMM 10-12         | 39 degrees C      | ok
08-P2 DIMM 1-3           | 31 degrees C      | ok
09-P2 DIMM 4-6           | 32 degrees C      | ok
10-P2 DIMM 7-9           | 34 degrees C      | ok
11-P2 DIMM 10-12         | 36 degrees C      | ok
12-HD Max                | 42 degrees C      | ok
13-Chipset               | 55 degrees C      | ok
14-P/S 1                 | 34 degrees C      | ok
15-P/S 2                 | 32 degrees C      | ok
16-P/S 2 Zone            | 33 degrees C      | ok
17-VR P1                 | 42 degrees C      | ok
18-VR P2                 | 34 degrees C      | ok
19-VR P1 Mem             | 40 degrees C      | ok
20-VR P1 Mem             | 39 degrees C      | ok
21-VR P2 Mem             | 33 degrees C      | ok
22-VR P2 Mem             | 35 degrees C      | ok
23-VR P1Vtt Zone         | 41 degrees C      | ok
24-VR P2Vtt Zone         | 33 degrees C      | ok
25-HD Controller         | 84 degrees C      | ok
26-iLO Zone              | 40 degrees C      | ok
27-LOM Card              | 62 degrees C      | ok
28-PCI 1                 | disabled          | ns
29-PCI 2                 | disabled          | ns
30-PCI 3                 | disabled          | ns
31-PCI 4                 | disabled          | ns
32-PCI 5                 | disabled          | ns
33-PCI 6                 | disabled          | ns
34-PCI 1 Zone            | 36 degrees C      | ok
35-PCI 2 Zone            | 37 degrees C      | ok
36-PCI 3 Zone            | 38 degrees C      | ok
37-PCI 4 Zone            | disabled          | ns
38-PCI 5 Zone            | disabled          | ns
39-PCI 6 Zone            | disabled          | ns
40-I/O Board 1           | 41 degrees C      | ok
41-I/O Board 2           | disabled          | ns
42-VR P1 Zone            | 35 degrees C      | ok
43-BIOS Zone             | 54 degrees C      | ok
44-System Board          | 42 degrees C      | ok
45-SuperCap Max          | 29 degrees C      | ok
46-Chipset Zone          | 41 degrees C      | ok
47-Battery Zone          | 38 degrees C      | ok
48-I/O Zone              | 43 degrees C      | ok
49-Sys Exhaust           | 39 degrees C      | ok
50-Sys Exhaust           | 38 degrees C      | ok
Fan 1                    | 6.27 percent      | ok
Fan 2                    | 6.27 percent      | ok
Fan 3                    | 6.27 percent      | ok
Fan 4                    | 6.27 percent      | ok
Fan 5                    | 6.27 percent      | ok
Fan 6                    | 7.45 percent      | ok
Power Supply 1           | 50 Watts          | ok
Power Supply 2           | 85 Watts          | ok
Power Meter              | 156 Watts         | ok
Power Supplies           | 0x00              | ok
Fans                     | 0x00              | ok
Memory                   | 0x00              | ok
C1 P1I Bay 1             | 0x01              | ok
C1 P1I Bay 2             | 0x01              | ok
C1 P1I Bay 3             | 0x01              | ok
C1 P1I Bay 4             | 0x01              | ok
C1 P2I Bay 5             | 0x01              | ok
C1 P2I Bay 6             | 0x01              | ok
C1 P2I Bay 7             | 0x01              | ok
```

```txt
$ sudo ipmitool sdr type Temperature
01-Inlet Ambient         | 03h | ok  | 64.1 | 22 degrees C
02-CPU 1                 | 04h | ok  | 65.1 | 40 degrees C
03-CPU 2                 | 05h | ok  | 65.2 | 40 degrees C
04-P1 DIMM 1-3           | 06h | ok  | 32.1 | 38 degrees C
05-P1 DIMM 4-6           | 07h | ok  | 32.2 | 39 degrees C
06-P1 DIMM 7-9           | 08h | ok  | 32.3 | 39 degrees C
07-P1 DIMM 10-12         | 09h | ok  | 32.4 | 39 degrees C
08-P2 DIMM 1-3           | 0Ah | ok  | 32.5 | 31 degrees C
09-P2 DIMM 4-6           | 0Bh | ok  | 32.6 | 32 degrees C
10-P2 DIMM 7-9           | 0Ch | ok  | 32.7 | 34 degrees C
11-P2 DIMM 10-12         | 0Dh | ok  | 32.8 | 36 degrees C
12-HD Max                | 0Eh | ok  |  4.1 | 42 degrees C
13-Chipset               | 0Fh | ok  | 66.1 | 54 degrees C
14-P/S 1                 | 10h | ok  | 10.4 | 34 degrees C
15-P/S 2                 | 11h | ok  | 10.5 | 32 degrees C
16-P/S 2 Zone            | 12h | ok  | 10.6 | 33 degrees C
17-VR P1                 | 13h | ok  | 19.1 | 42 degrees C
18-VR P2                 | 14h | ok  | 19.2 | 34 degrees C
19-VR P1 Mem             | 15h | ok  | 19.3 | 40 degrees C
20-VR P1 Mem             | 16h | ok  | 19.4 | 39 degrees C
21-VR P2 Mem             | 17h | ok  | 19.5 | 33 degrees C
22-VR P2 Mem             | 18h | ok  | 19.6 | 35 degrees C
23-VR P1Vtt Zone         | 19h | ok  | 19.7 | 41 degrees C
24-VR P2Vtt Zone         | 1Ah | ok  | 19.8 | 33 degrees C
25-HD Controller         | 1Bh | ok  | 66.2 | 84 degrees C
26-iLO Zone              | 1Ch | ok  |  6.1 | 40 degrees C
27-LOM Card              | 1Dh | ok  | 11.1 | 62 degrees C
28-PCI 1                 | 1Eh | ns  | 11.2 | Disabled
29-PCI 2                 | 1Fh | ns  | 11.3 | Disabled
30-PCI 3                 | 20h | ns  | 11.4 | Disabled
31-PCI 4                 | 21h | ns  | 11.5 | Disabled
32-PCI 5                 | 22h | ns  | 11.6 | Disabled
33-PCI 6                 | 23h | ns  | 11.7 | Disabled
34-PCI 1 Zone            | 24h | ok  | 16.1 | 36 degrees C
35-PCI 2 Zone            | 25h | ok  | 16.2 | 37 degrees C
36-PCI 3 Zone            | 26h | ok  | 16.3 | 38 degrees C
37-PCI 4 Zone            | 27h | ns  | 16.4 | Disabled
38-PCI 5 Zone            | 28h | ns  | 16.5 | Disabled
39-PCI 6 Zone            | 29h | ns  | 16.6 | Disabled
40-I/O Board 1           | 2Ah | ok  | 16.7 | 41 degrees C
41-I/O Board 2           | 2Bh | ns  | 16.8 | Disabled
42-VR P1 Zone            | 2Ch | ok  | 19.9 | 34 degrees C
43-BIOS Zone             | 2Dh | ok  | 34.1 | 53 degrees C
44-System Board          | 2Eh | ok  | 66.3 | 42 degrees C
45-SuperCap Max          | 2Fh | ok  | 40.1 | 29 degrees C
46-Chipset Zone          | 30h | ok  | 66.4 | 41 degrees C
47-Battery Zone          | 31h | ok  | 40.2 | 38 degrees C
48-I/O Zone              | 32h | ok  | 66.5 | 43 degrees C
49-Sys Exhaust           | 33h | ok  | 13.1 | 39 degrees C
50-Sys Exhaust           | 34h | ok  | 13.2 | 38 degrees C
```

```txt
$ sudo ipmitool sdr type 'Power Supply'
Power Supply 1           | 3Bh | ok  | 10.1 | 50 Watts, Presence detected
Power Supply 2           | 3Ch | ok  | 10.2 | 90 Watts, Presence detected
Power Supplies           | 3Eh | ok  | 10.3 | Fully Redundant
```

Supermicro X10:

```txt
$ sudo ipmitool sdr
CPU1 Temp        | 47 degrees C      | ok
CPU2 Temp        | 38 degrees C      | ok
PCH Temp         | 47 degrees C      | ok
System Temp      | 26 degrees C      | ok
Peripheral Temp  | 43 degrees C      | ok
MB_NIC_Temp1     | 61 degrees C      | ok
MB_NIC_Temp2     | 58 degrees C      | ok
Vcpu1VRM Temp    | 37 degrees C      | ok
Vcpu2VRM Temp    | 31 degrees C      | ok
VmemABVRM Temp   | 27 degrees C      | ok
VmemCDVRM Temp   | 29 degrees C      | ok
VmemEFVRM Temp   | 27 degrees C      | ok
VmemGHVRM Temp   | 26 degrees C      | ok
P1-DIMMA1 Temp   | 31 degrees C      | ok
P1-DIMMA2 Temp   | 31 degrees C      | ok
P1-DIMMA3 Temp   | no reading        | ns
P1-DIMMB1 Temp   | 32 degrees C      | ok
P1-DIMMB2 Temp   | 31 degrees C      | ok
P1-DIMMB3 Temp   | no reading        | ns
P1-DIMMC1 Temp   | 30 degrees C      | ok
P1-DIMMC2 Temp   | 29 degrees C      | ok
P1-DIMMC3 Temp   | no reading        | ns
P1-DIMMD1 Temp   | 29 degrees C      | ok
P1-DIMMD2 Temp   | 30 degrees C      | ok
P1-DIMMD3 Temp   | no reading        | ns
P2-DIMME1 Temp   | 29 degrees C      | ok
P2-DIMME2 Temp   | 27 degrees C      | ok
P2-DIMME3 Temp   | no reading        | ns
P2-DIMMF1 Temp   | 28 degrees C      | ok
P2-DIMMF2 Temp   | 28 degrees C      | ok
P2-DIMMF3 Temp   | no reading        | ns
P2-DIMMG1 Temp   | 26 degrees C      | ok
P2-DIMMG2 Temp   | 26 degrees C      | ok
P2-DIMMG3 Temp   | no reading        | ns
P2-DIMMH1 Temp   | 27 degrees C      | ok
P2-DIMMH2 Temp   | 28 degrees C      | ok
P2-DIMMH3 Temp   | no reading        | ns
FAN1             | 1900 RPM          | ok
FAN2             | 1900 RPM          | ok
FAN3             | no reading        | ns
FAN4             | no reading        | ns
FAN5             | no reading        | ns
FAN6             | no reading        | ns
FAN7             | 2900 RPM          | ok
FAN8             | 2800 RPM          | ok
FAN9             | no reading        | ns
12V              | 12.13 Volts       | ok
5VCC             | 4.97 Volts        | ok
3.3VCC           | 3.33 Volts        | ok
VBAT             | 3.13 Volts        | ok
Vcpu1            | 1.82 Volts        | ok
Vcpu2            | 1.82 Volts        | ok
VDIMMAB          | 1.19 Volts        | ok
VDIMMCD          | 1.18 Volts        | ok
VDIMMEF          | 1.19 Volts        | ok
VDIMMGH          | 1.19 Volts        | ok
5VSB             | 5.05 Volts        | ok
3.3VSB           | 3.42 Volts        | ok
1.5V PCH         | 1.52 Volts        | ok
1.2V BMC         | 1.21 Volts        | ok
1.05V PCH        | 1.05 Volts        | ok
Chassis Intru    | 0x01              | ok
PS2 Status       | 0x01              | ok
```

### Logging

To retrieve the system event log from the BMC:

```txt
$ sudo ipmitool sel list
  23 | 12/06/2024 | 10:06:50 PM EST | Fan #0x41 | Lower Critical going low  | Asserted
  24 | 12/06/2024 | 10:06:50 PM EST | Fan #0x41 | Lower Non-recoverable going low  | Asserted
  25 | 12/06/2024 | 10:06:52 PM EST | Fan #0x42 | Lower Critical going low  | Asserted
  26 | 12/06/2024 | 10:06:52 PM EST | Fan #0x42 | Lower Non-recoverable going low  | Asserted
  27 | 12/06/2024 | 10:06:55 PM EST | Fan #0x47 | Lower Critical going low  | Asserted
  28 | 12/06/2024 | 10:06:55 PM EST | Fan #0x47 | Lower Non-recoverable going low  | Asserted
  29 | 12/06/2024 | 10:06:56 PM EST | Fan #0x48 | Lower Critical going low  | Asserted
  2a | 12/06/2024 | 10:06:56 PM EST | Fan #0x48 | Lower Non-recoverable going low  | Asserted
  2b | 12/06/2024 | 10:07:58 PM EST | Fan #0x48 | Lower Non-recoverable going low  | Deasserted
  2c | 12/06/2024 | 10:07:58 PM EST | Fan #0x48 | Lower Critical going low  | Deasserted
  2d | 12/06/2024 | 10:07:59 PM EST | Fan #0x47 | Lower Non-recoverable going low  | Deasserted
  2e | 12/06/2024 | 10:07:59 PM EST | Fan #0x47 | Lower Critical going low  | Deasserted
  2f | 12/06/2024 | 10:08:01 PM EST | Fan #0x42 | Lower Non-recoverable going low  | Deasserted
  30 | 12/06/2024 | 10:08:01 PM EST | Fan #0x42 | Lower Critical going low  | Deasserted
  31 | 12/06/2024 | 10:08:02 PM EST | Fan #0x41 | Lower Non-recoverable going low  | Deasserted
  32 | 12/06/2024 | 10:08:02 PM EST | Fan #0x41 | Lower Critical going low  | Deasserted
  33 | 12/08/2024 | 09:31:35 PM EST | Unknown #0xff |  | Asserted
  34 | 12/08/2024 | 09:31:41 PM EST | Unknown #0xff |  | Asserted
  35 | 12/08/2024 | 09:31:56 PM EST | Physical Security #0xaa | General Chassis intrusion () | Asserted
  65 | 09/11/2025 | 06:28:51 PM EDT | OS Boot | Installation started () | Asserted
  66 | 09/11/2025 | 06:50:38 PM EDT | OS Boot | Installation completed () | Asserted
  67 | 09/11/2025 | 06:57:23 PM EDT | Unknown #0xff |  | Asserted
  68 | 09/11/2025 | 06:57:38 PM EDT | Physical Security #0xaa | General Chassis intrusion () | Asserted
```

```txt
 532 | 08/14/2023 | 11:25:37 | Power Supply #0x3c | Failure detected | Asserted
 533 | 08/14/2023 | 11:25:39 | Power Supply #0x3c | Power Supply AC lost | Asserted
 534 | 08/14/2023 | 11:25:39 | Power Supply #0x3e | Redundancy Lost | Asserted
 535 | 08/14/2023 | 11:25:41 | Power Supply #0x3e | Redundancy Lost | Deasserted
 536 | 08/14/2023 | 11:25:42 | Power Supply #0x3c | Failure detected | Asserted
 537 | 08/14/2023 | 11:25:44 | Power Supply #0x3c | Power Supply AC lost | Asserted
 538 | 08/14/2023 | 11:25:44 | Power Supply #0x3e | Redundancy Lost | Asserted
 539 | 08/14/2023 | 11:28:06 | Power Supply #0x3e | Redundancy Lost | Deasserted
 ```

```txt
 547 | 03/16/2025 | 02:18:29 | Drive Slot / Bay #0x44 | Predictive Failure | Asserted
 548 | 03/26/2025 | 14:11:40 | Drive Slot / Bay #0x44 | Drive Fault | Asserted
```

To see info about the system event log (e.g., capacity):

```txt
$ sudo ipmitool sel info
SEL Information
Version          : 1.5 (v1.5, v2 compliant)
Entries          : 104
Free Space       : 8160 bytes
Percent Used     : 16%
Last Add Time    : 09/11/2025 06:57:38 PM EDT
Last Del Time    : Not Available
Overflow         : false
Supported Cmds   : 'Reserve' 'Get Alloc Info'
# of Alloc Units : 512
Alloc Unit Size  : 20
# Free Units     : 408
Largest Free Blk : 408
Max Record Size  : 20
```

### Chassis

```txt
$ sudo ipmitool chassis status
System Power         : on
Power Overload       : false
Power Interlock      : inactive
Main Power Fault     : false
Power Control Fault  : false
Power Restore Policy : always-on
Last Power Event     :
Chassis Intrusion    : inactive
Front-Panel Lockout  : inactive
Drive Fault          : false
Cooling/Fan Fault    : false
```

```txt
$ sudo ipmitool chassis identify
Chassis identify interval: default (15 seconds)

$ sudo ipmitool chassis identify 60
Chassis identify interval: 60 seconds
```

```txt
$ sudo ipmitool fru print
FRU Device Description : Builtin FRU Device (ID 0)
 Chassis Type          : Unknown
 Chassis Serial        : C829UAE38B00096
 Board Mfg Date        : Unspecified
 Board Mfg             : Supermicro
 Board Serial          : OM158S018596
 Board Part Number     : X10DRU-i+
 Product Manufacturer  : Supermicro
 Product Serial        :
```
