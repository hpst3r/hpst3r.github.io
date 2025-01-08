---
title: "JNCIA lab: simple route redistribution in Junos"
date: 2025-01-06T10:10:10-00:00
draft: false
---

Topology:

{{< figure src="topo.png" alt="A picture representing a lab topology" >}}


Images used:

IOSv L2 15.2 (as dumb hosts)
IOSv L3 15.9
JOSv (vMX) 23.2R1.15

Configure interfaces' addressing as shown.

Configure a mesh of four routers in an OSPF area:
JR1, JR2, JR3 and CR1

OSPF should advertise these four routers' loopback addresses.
Any interface not facing another OSPF router should be declared passive.

Configure RIP on JR3 and JR4. Modify the default export policy so they can talk.
Redistribute the RIP routes to 169.254.0.16/30 and 192.51.100.0/24 into the OSPF area. Redistribute JR3's OSPF routes into RIP on JR3 and JR4. Make sure CS1 can ping any of the loopback addresses in the topology with just a default route configured to JR4.

Configure a floating static route for 0.0.0.0/0 with next hop 169.254.0.22 (CS2) on JR2. Redistribute it into OSPF. Test it by pinging CS2's loopback address (that will not be advertised.) This should be the default route for JR4.

## Cisco router 1 config (CR1)

```txt
en
conf t
hostname CR1
router ospf 1
auto-cost reference-bandwidth 100000
router-id 192.0.2.3
passive-interface lo0
int g0/1
ip address 169.254.0.13 255.255.255.252
ip ospf 1 area 0
no shut
int g0/0
ip address 169.254.0.10 255.255.255.252
ip ospf 1 area 0
no shut
int lo0
ip address 192.0.2.3 255.255.255.255
ip ospf 1 area 0
no shut
end
wr
```
## Cisco switch 1 config (CS1)

```txt
en
conf t
hostname CS1
ip route 0.0.0.0 0.0.0.0 192.51.100.254
int g0/0
no sw
ip addr 192.51.100.200 255.255.255.0
no shut
end
wr
```
## Cisco switch 2 config (CS2)

```txt
en
conf t
hostname CS2
ip route 0.0.0.0 0.0.0.0 169.254.0.21
ip routing
int vlan 1
ip addr 169.254.0.22 255.255.255.252
no shut
int lo0
ip addr 192.0.2.255 255.255.255.255
no shut
end
wr
```
## Basic Junos router config (JR1)

Assign interface IPs and assign interfaces to OSPF generically. Set the OSPF reference bandwidth to 100g.

```txt
[edit]
root# set interfaces ge-0/0/0 unit 0 family inet4 address 169.254.0.1/30

[edit]
root# set interfaces ge-0/0/0 unit 0 family inet4 address 169.254.0.14/30

[edit]
root# set interfaces lo0 unit 0 family inet4 address 192.0.2.0/32

[edit]
root# edit protocols ospf area 0

[edit protocols ospf area 0.0.0.0]
root# set interface ge-0/0/0

[edit protocols ospf area 0.0.0.0]
root# set interface ge-0/0/1

[edit protocols ospf area 0.0.0.0]
root# set interface lo0 passive

[edit protocols ospf area 0.0.0.0]
root# top

[edit]
root# set system host-name JR1
```

```txt
[edit]
root# show | compare
[edit system]
+  host-name JR1;
[edit interfaces]
+   ge-0/0/0 {
+       unit 0 {
+           family inet {
+               address 169.254.0.1/30;
+           }
+       }
+   }
+   ge-0/0/1 {
+       unit 0 {
+           family inet {
+               address 169.254.0.12/30;
+               address 169.254.0.14/30;
+           }
+       }
+   }
+   lo0 {
+       unit 0 {
+           family inet {
+               address 192.0.2.0/32;
+           }
+       }
+   }
[edit protocols]
+   ospf {
+       area 0.0.0.0 {
+           interface ge-0/0/0.0;
+           interface ge-0/0/1.0;
+           interface lo0.0 {
+               passive;
+           }
+       }
+   }

[edit protocols ospf area 0.0.0.0]
```
## Configure static default route on JR2

```txt
[edit]
root@JR2# set interfaces ge-0/0/2 unit 0 family inet4 address 169.254.0.21/30

[edit]
root@JR2# set routing-options static route 0.0.0.0/0 next-hop 169.254.0.22 preference 254
```

```txt
[edit]
root@JR2# top show | compare
[edit interfaces]
+   ge-0/0/2 {
+       unit 0 {
+           family inet {
+               address 169.254.0.21/30;
+           }
+       }
+   }
[edit]
+  routing-options {
+      static {
+          route 0.0.0.0/0 {
+              next-hop 169.254.0.22;
+              preference 254;
+          }
+      }
+  }
```
### Verify

```txt
[edit]
root@JR2# run ping 192.0.2.255 rapid
PING 192.0.2.255 (192.0.2.255): 56 data bytes
!!!!!
--- 192.0.2.255 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.773/2.176/2.914/0.409 ms
```
## Configure redistribution of the static default route to OSPF on JR2

Configure a policy statement that selects this specific static route in addition to the Junos default connected route redistribution.

```txt
[edit]
root@JR2# edit policy-options policy-statement export-static-default-p254 term match

[edit policy-options policy-statement export-static-default-p254 term match]
root@JR2# set from route-filter 0.0.0.0/0 exact

[edit policy-options policy-statement export-static-default-p254 term match]
root@JR2# set from protocol static

[edit policy-options policy-statement export-static-default-p254 term match]
root@JR2# set from preference 254

[edit policy-options policy-statement export-static-default-p254 term match]
root@JR2# set from next-hop 169.254.0.22

[edit policy-options policy-statement export-static-default-p254 term match]
root@JR2# set then accept

[edit policy-options policy-statement export-static-default-p254 term match]
root@JR2# show
from {
    protocol static;
    next-hop 169.254.0.22;
    preference 254;
    route-filter 0.0.0.0/0 exact;
}
then accept;

[edit policy-options policy-statement export-static-default-p254 term match]
root@JR2# top

[edit]
root@JR2# show policy-options policy-statement export-static-default-p254
term match {
    from {
        protocol static;
        next-hop 169.254.0.22;
        preference 254;
        route-filter 0.0.0.0/0 exact;
    }
    then accept;
}
term catch-all {
    then reject;
}

[edit]
root@JR2# set protocols ospf export export-static-default-p254
```

```txt
[edit]
root@JR2# show | compare
[edit]
+  policy-options {
+      policy-statement export-static-default-p254 {
+          term match {
+              from {
+                  protocol static;
+                  next-hop 169.254.0.22;
+                  preference 254;
+                  route-filter 0.0.0.0/0 exact;
+              }
+              then accept;
+          }
+      }
+  }
[edit protocols ospf]
+   export export-static-default-p254;
```
### Verify

Confirm that R2's static default route is redistributed into the OSPF topology (and, thus, neighbors' routing tables) as an external route and CS2's non-advertised loopback is now reachable.

```txt
root@JR3> show route

inet.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[OSPF/150] 00:03:18, metric 0, tag 0
                    >  to 169.254.0.5 via ge-0/0/0.0

root@JR3> ping 192.0.2.255 rapid
PING 192.0.2.255 (192.0.2.255): 56 data bytes
!!!!!
--- 192.0.2.255 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.949/3.134/3.599/0.239 ms

root@JR3> show ospf route
Topology default Route Table:

Prefix             Path  Route      NH       Metric NextHop       Nexthop
                   Type  Type       Type            Interface     Address/LSP
192.0.2.0          Intra Router     IP          200 ge-0/0/0.0    169.254.0.5
                                                    ge-0/0/1.0    169.254.0.10
192.0.2.1          Intra AS BR      IP          100 ge-0/0/0.0    169.254.0.5
192.0.2.3          Intra Router     IP          100 ge-0/0/1.0    169.254.0.10
0.0.0.0/0          Ext2  Network    IP            0 ge-0/0/0.0    169.254.0.5
169.254.0.0/30     Intra Network    IP          200 ge-0/0/0.0    169.254.0.5
169.254.0.4/30     Intra Network    IP          100 ge-0/0/0.0
169.254.0.8/30     Intra Network    IP          100 ge-0/0/1.0
169.254.0.12/30    Intra Network    IP          200 ge-0/0/1.0    169.254.0.10
192.0.2.1/32       Intra Network    IP          100 ge-0/0/0.0    169.254.0.5
192.0.2.2/32       Intra Network    IP            0 lo0.0
192.0.2.3/32       Intra Network    IP          101 ge-0/0/1.0    169.254.0.10
```

Interestingly, here it seems like the Cisco R1 is adding 1 to its metric for the hop from g0/0 to lo0.
## Configure RIP on JR3

We'll also need to configure OSPF as a passive interface on the ge-0/0/2 interface if we want that route advertised.

```txt
[edit]
root@JR3# set interfaces ge-0/0/2 unit 0 family inet4 address 169.254.0.17/30

[edit]
root@JR3# set protocols ospf area 0 interface ge-0/0/2 passive

[edit]
root@JR3# set protocols rip group rip-group neighbor ge-0/0/2
```

```txt
[edit]
root@JR3# show | compare
[edit interfaces]
+   ge-0/0/2 {
+       unit 0 {
+           family inet {
+               address 169.254.0.17/30;
+           }
+       }
+   }
[edit protocols]
+   rip {
+       group rip-group {
+           neighbor ge-0/0/2.0;
+       }
+   }
[edit protocols ospf area 0.0.0.0]
      interface lo0.0 { ... }
+     interface ge-0/0/2.0 {
+         passive;
+     }
```
### Verify

For now, just confirm that the RIP process on R3 has picked up the correct interface IP.

```txt
root@JR3> show rip neighbor
                  Local  Source          Destination     Send   Receive   In
Neighbor          State  Address         Address         Mode   Mode     Met
--------          -----  -------         -----------     ----   -------  ---
ge-0/0/2.0           Up 169.254.0.17    224.0.0.9       mcast  both       1
```
## Configure RIP on JR4

```txt
[edit]
root@JR4# set protocols rip group rip-group neighbor ge-0/0/0
```

```txt
[edit]
root@JR4# show | compare
[edit protocols]
+   rip {
+       group rip-group {
+           neighbor ge-0/0/0.0;
+       }
+   }
```
### Verify

Since the Junos default is to not export anything via RIP, no routes or updates will be floating around yet. Again, just check that the process has the desired interface.

```txt
root@JR4> show rip neighbor
                  Local  Source          Destination     Send   Receive   In
Neighbor          State  Address         Address         Mode   Mode     Met
--------          -----  -------         -----------     ----   -------  ---
ge-0/0/0.0           Up 169.254.0.18    224.0.0.9       mcast  both       1
```
## Configure redistribution into RIP on JR3

Configure an export policy for RIP to redistribute directly connected and OSPF routes from JR3 into the RIP topology.

```txt
[edit]
root@JR3# edit policy-options policy-statement export-ospf-direct-to-rip term match

[edit policy-options policy-statement export-ospf-direct-to-rip term match]
root@JR3# set from protocol [ ospf direct ]

[edit policy-options policy-statement export-ospf-direct-to-rip term match]
root@JR3# set then accept

[edit policy-options policy-statement export-ospf-direct-to-rip term match]
root@JR3# top

[edit]
root@JR3# set protocols rip group rip-group export export-ospf-direct-to-rip
```

```txt
[edit]
root@JR3# show | compare
[edit]
+  policy-options {
+      policy-statement export-ospf-direct-to-rip {
+          term match {
+              from protocol [ ospf direct ];
+              then accept;
+          }
+      }
+  }
[edit protocols rip group rip-group]
+    export export-ospf-direct-to-rip;
```
### Verify

Confirm that JR4 has learned routes originating from the OSPF topology.

```txt
root@JR4> show route

inet.0: 13 destinations, 13 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

0.0.0.0/0          *[RIP/100] 00:00:37, metric 2, tag 0
                    >  to 169.254.0.17 via ge-0/0/0.0
169.254.0.0/30     *[RIP/100] 00:00:37, metric 2, tag 0
                    >  to 169.254.0.17 via ge-0/0/0.0
169.254.0.4/30     *[RIP/100] 00:00:37, metric 2, tag 0
                    >  to 169.254.0.17 via ge-0/0/0.0
169.254.0.8/30     *[RIP/100] 00:00:37, metric 2, tag 0
                    >  to 169.254.0.17 via ge-0/0/0.0
169.254.0.12/30    *[RIP/100] 00:00:37, metric 2, tag 0
                    >  to 169.254.0.17 via ge-0/0/0.0
169.254.0.16/30    *[Direct/0] 01:09:56
                    >  via ge-0/0/0.0
169.254.0.18/32    *[Local/0] 01:09:56
                       Local via ge-0/0/0.0
192.0.2.1/32       *[RIP/100] 00:00:37, metric 2, tag 0
                    >  to 169.254.0.17 via ge-0/0/0.0
192.0.2.2/32       *[RIP/100] 00:00:37, metric 2, tag 0
                    >  to 169.254.0.17 via ge-0/0/0.0
192.0.2.3/32       *[RIP/100] 00:00:37, metric 2, tag 0
                    >  to 169.254.0.17 via ge-0/0/0.0
192.51.100.0/24    *[Direct/0] 01:09:56
                    >  via ge-0/0/1.0
192.51.100.254/32  *[Local/0] 01:09:56
                       Local via ge-0/0/1.0
224.0.0.9/32       *[RIP/100] 00:36:33, metric 1
                       MultiRecv

root@JR4> show rip statistics
RIPv2 info: port 520; holddown 120s.
    rts learned  rts held down  rqsts dropped  resps dropped
              8              0              0              0

ge-0/0/0.0:  8 routes learned; 0 routes advertised; timeout 180s; update interval 30s
Counter                         Total   Last 5 min  Last minute
-------                   -----------  -----------  -----------
Updates Sent                        0            0            0
Triggered Updates Sent              0            0            0
Responses Sent                      0            0            0
Bad Messages                        0            0            0
RIPv1 Updates Received              0            0            0
RIPv1 Bad Route Entries             0            0            0
RIPv1 Updates Ignored               0            0            0
RIPv2 Updates Received              1            0            0
RIPv2 Bad Route Entries             0            0            0
RIPv2 Updates Ignored               0            0            0
Authentication Failures             0            0            0
RIP Requests Received               0            0            0
RIP Requests Ignored                0            0            0
RIP Update Acks Received            0            0            0

```

JR4 will not yet be able to reach anything past JR3, as JR3 is not advertising the 169.254.0.16/30 network anywhere yet (OSPF is not running on 169.254.0.17, JR3's ge-0/0/2.)
## Configure redistribution into RIP on JR4

Configure an export policy for RIP to redistribute directly connected routes from JR4 into the RIP topology.

```txt
[edit]
root@JR4# edit policy-options policy-statement export-direct-to-rip term match

[edit policy-options policy-statement export-direct-to-rip term match]
root@JR4# set from protocol direct

[edit policy-options policy-statement export-direct-to-rip term match]
root@JR4# set then accept

[edit policy-options policy-statement export-direct-to-rip term match]
root@JR4# top

[edit]
root@JR4# set protocols rip group rip-group export export-direct-to-rip
```

```txt
[edit]
root@JR4# show | compare
[edit]
+  policy-options {
+      policy-statement export-direct-to-rip {
+          term match {
+              from protocol direct;
+              then accept;
+          }
+      }
+  }
[edit protocols rip group rip-group]
+    export export-direct-to-rip;
```
### Verify

Confirm JR3 now has a RIP route to 192.51.100.0/24.

```txt
root@JR3> show route
...
192.51.100.0/24    *[RIP/100] 00:02:37, metric 2, tag 0
                    >  to 169.254.0.18 via ge-0/0/2.0
```
## Configure redistribution into OSPF on JR3

We are advertising the direct route JR3 has to 169.254.0.16/30 without OSPF running on ge-0/0/2, plus the RIP routes learned from JR4. You could lock this down more with `from next-hop 169.254.0.18`.

```txt
[edit]
root@JR3# edit policy-options policy-statement export-rip-direct-to-ospf term match

[edit policy-options policy-statement export-rip-direct-to-ospf term match]
root@JR3# set from protocol [rip direct]

[edit policy-options policy-statement export-rip-direct-to-ospf term match]
root@JR3# set then accept

[edit policy-options policy-statement export-rip-direct-to-ospf term match]
root@JR3# top

[edit]
root@JR3# set protocols ospf export export-rip-direct-to-ospf
```

```txt
[edit]
root@JR3# show | compare
[edit policy-options]
+   policy-statement export-rip-direct-to-ospf {
+       term match {
+           from protocol [ rip direct ];
+           then accept;
+       }
+   }
[edit protocols ospf]
+   export export-rip-direct-to-ospf;
```
### Verify

You should now have full L3 reachability. To verify this, try pinging CS2's unadvertised loopback address from CS1.

```txt
CS1#ping 192.0.2.255
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.0.2.255, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/4/5 ms
```

This success means that RIP is working, OSPF is working, the RIP route to 192.51.100.0/24 is being redistributed into OSPF, the direct route to 169.254.0.16/30 is being redistributed into OSPF, and the static default route to CS2 is being redistributed into OSPF.

Try to run a traceroute from CS2 to JR1. See which path it takes.

```txt
CS1>traceroute 192.0.2.0
Type escape sequence to abort.
Tracing the route to 192.0.2.0
VRF info: (vrf in name/id, vrf out name/id)
  1 192.51.100.254 2 msec 2 msec 1 msec
  2 169.254.0.17 3 msec 3 msec 3 msec
  3 169.254.0.5 3 msec 4 msec 3 msec
  4 192.0.2.0 5 msec 5 msec 6 msec
```