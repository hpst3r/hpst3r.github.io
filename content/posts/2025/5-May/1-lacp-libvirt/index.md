---
title: "Configuring LACP bonds & VLAN bridges with nmcli"
date: 2025-05-03T18:30:00-00:00
draft: false
---

LACP is awesome if you're not terminating L3 on your servers. It's also super easy to configure bonds with NetworkManager.

## Configuring LACP (802.3ad) bonds with nmcli

To configure a bond, you'll need to:
- Create the bond interface
- Assign connections as slaves to the bond
- Bring the bond up

This is pretty simple:

```txt
[liam@t3 ~]$ nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=802.3ad"
```

This will create a `bond0` device, and a `bond0` connection - you can see them in the output of `nmcli con` or `nmcli dev`:

```txt
[liam@t3 ~]$ nmcli con
NAME         UUID                                  TYPE      DEVICE
System eth0  5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03  ethernet  eth0
bond0        ae6f842b-fc01-4328-a38f-415dab1c990a  bond      bond0
lo           f2d72c88-ccfa-4515-8231-66ce858b5788  loopback  lo
eth0         445b248e-2f8f-4809-ac9d-485e81196fc5  ethernet  --
[liam@t3 ~]$ nmcli dev
DEVICE  TYPE      STATE                                  CONNECTION
eth0    ethernet  connected                              System eth0
bond0   bond      connecting (getting IP configuration)  bond0
lo      loopback  connected (externally)                 lo
```

The bond will be hanging out failing to do much as it's not associated with any interfaces. Let's fix that by assigning our `eth0` to be a slave to said bond:

```txt
[liam@t3 ~]$ sudo nmcli con mod eth0 master bond0 slave-type bond
[liam@t3 ~]$ sudo nmcli con up eth0
```

Bam! Now we have a logical interface handling our addressing. You could add more ethernet interfaces to the bond if you'd like by setting a `master` and `slave-type` for them, too. Alternatively, you can create new connections for the raw devices. Couple of ways to skin a cat.

```txt
[liam@t3 ~]$ nmcli con
NAME         UUID                                  TYPE      DEVICE
bond0        ae6f842b-fc01-4328-a38f-415dab1c990a  bond      bond0
eth0         445b248e-2f8f-4809-ac9d-485e81196fc5  ethernet  eth0
lo           7e0af8dd-bdec-46af-9af6-d7b7210b6a13  loopback  lo
System eth0  5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03  ethernet  --
[liam@t3 ~]$ nmcli dev
DEVICE  TYPE      STATE                   CONNECTION
bond0   bond      connected               bond0
eth0    ethernet  connected               eth0
lo      loopback  connected (externally)  lo
```

### Config files for bonds

Here are the generated `.nmconnection` files for bond0 and eth0 in this test VM (located at `/etc/NetworkManager/system-connections/`).

```txt
[liam@t3 NetworkManager]$ sudo cat system-connections/bond0.nmconnection
[sudo] password for liam:
[connection]
id=bond0
uuid=ae6f842b-fc01-4328-a38f-415dab1c990a
type=bond
interface-name=bond0

[bond]
mode=802.3ad

[ipv4]
method=auto

[ipv6]
addr-gen-mode=default
method=auto

[proxy]
```

```txt
[liam@t3 NetworkManager]$ sudo cat system-connections/eth0.nmconnection
[connection]
id=eth0
uuid=445b248e-2f8f-4809-ac9d-485e81196fc5
type=ethernet
autoconnect-priority=-100
autoconnect-retries=1
controller=bond0
interface-name=eth0
port-type=bond
timestamp=1746324368

[ethernet]

[bond-port]

[user]
org.freedesktop.NetworkManager.origin=nm-initrd-generator
```

## Configuring bridges on VLANs on bonded interfaces for VM switches

In this scenario, I'm using a Lenovo M920q. I'll be using a quad-port NIC for our VLAN bridges and the built-in i219 for management (so I can SSH in to configure this).

My end goal is attaching VMs to a vSwitch so the host can tag their traffic (without any configuration of the VM itself). This is how VMware ESXi hosts tend to work without NSX in the picture, and I find it to be a nice way to do things.

In short, we'll need to:
- Create a bond
- Slave our physical interfaces to the logical bond
- Create a bridge
- Slave a VLAN on the bond of our physical interfaces to the logical bridge

A graphic of the interface hierarchy here might be helpful:

{{< figure src="images/bondage.png" >}}

First, delete the default connections for the four interfaces we're bonding. We'll be recreating these as slaves later. Note that this is not necessary; you could just modify the existing connections.

```txt
[root@m920q1 ~]# nmcli con del enp1s0f0
Connection 'enp1s0f0' (7d56d716-19b4-4a1e-b889-822c6bba58ed) successfully deleted.
[root@m920q1 ~]# nmcli con del enp1s0f1
Connection 'enp1s0f1' (9d81332a-c6a5-4678-ad73-c5b21e1768f9) successfully deleted.
[root@m920q1 ~]# nmcli con del enp1s0f2
Connection 'enp1s0f2' (1afce241-dac6-438c-9878-00466a15f281) successfully deleted.
[root@m920q1 ~]# nmcli con del enp1s0f3
Connection 'enp1s0f3' (159c9341-317b-4fa7-8ccb-d69222ea3e88) successfully deleted.
```

Let's have a look at our connections and devices before we start.

```txt
[root@m920q1 ~]# nmcli dev
DEVICE    TYPE      STATE                   CONNECTION
eno1      ethernet  connected               eno1
lo        loopback  connected (externally)  lo
enp1s0f3  ethernet  disconnected            --
enp1s0f0  ethernet  unavailable             --
enp1s0f1  ethernet  unavailable             --
enp1s0f2  ethernet  unavailable             --
[root@m920q1 ~]# nmcli con
NAME  UUID                                  TYPE      DEVICE
eno1  2555d2b2-2e0e-3c0e-be18-bc5a242527fc  ethernet  eno1
lo    c4b2de4b-a075-4353-b463-540af3810e7a  loopback  lo
```

Well, let's create the bond. We'll need to create it first, then slave our interfaces to it.

```txt
[root@m920q1 ~]# nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=802.3ad,xmit_hash_policy=layer2+3" ipv4.method disabled ipv6.method ignore
Connection 'bond0' (97921ace-0085-48a2-8b86-7cebee1816e2) successfully added.
```

Now let's create connections for the four enp1s0f* interfaces as slaves to the bond.

```txt
[root@m920q1 ~]# nmcli con add type ethernet con-name bond0-slave0 ifname enp1s0f0 master bond0 slave-type bond
Connection 'bond0-slave0' (d3fb54ea-75d5-4b4c-8533-d505b8865fc6) successfully added.
[root@m920q1 ~]# nmcli con add type ethernet con-name bond0-slave1 ifname enp1s0f1 master bond0 slave-type bond
Connection 'bond0-slave1' (9954aead-c5de-4df4-bf2f-eae2c1160aa3) successfully added.
[root@m920q1 ~]# nmcli con add type ethernet con-name bond0-slave2 ifname enp1s0f2 master bond0 slave-type bond
Connection 'bond0-slave2' (d829e451-bc52-4646-a38f-f437454f2886) successfully added.
[root@m920q1 ~]# nmcli con add type ethernet con-name bond0-slave3 ifname enp1s0f3 master bond0 slave-type bond
Connection 'bond0-slave3' (72cb1f33-175a-4457-b13d-11691f39b772) successfully added.
```

Your LACP bond is now ready!

Now, let's configure a bridge, and slave a tagged VLAN interface that's a child of the bond interface to it as a carrier:

```txt
[root@m920q1 ~]# nmcli con add type bridge con-name br254 ifname br254 ipv4.method disabled ipv6.method ignore
Connection 'br254' (8a8ed4ac-aa39-4578-9410-0563f7104eea) successfully added.

[root@m920q1 ~]# nmcli con add type vlan con-name v254 ifname bond0.254 dev bond0 id 254 master br254 slave-type bridge
Connection 'v254' (3b7868de-3d8d-45c6-a519-f999c0b1c780) successfully added.
```

That's about it! What does this look like? Well.. lots of connections!

```txt
[root@m920q1 ~]# nmcli con
NAME          UUID                                  TYPE      DEVICE
eno1          2555d2b2-2e0e-3c0e-be18-bc5a242527fc  ethernet  eno1
bond0         97921ace-0085-48a2-8b86-7cebee1816e2  bond      bond0
bond0-slave3  72cb1f33-175a-4457-b13d-11691f39b772  ethernet  enp1s0f3
br254         8a8ed4ac-aa39-4578-9410-0563f7104eea  bridge    br254
v254          3b7868de-3d8d-45c6-a519-f999c0b1c780  vlan      bond0.254
lo            c4b2de4b-a075-4353-b463-540af3810e7a  loopback  lo
bond0-slave0  d3fb54ea-75d5-4b4c-8533-d505b8865fc6  ethernet  --
bond0-slave1  9954aead-c5de-4df4-bf2f-eae2c1160aa3  ethernet  --
bond0-slave2  d829e451-bc52-4646-a38f-f437454f2886  ethernet  --
[root@m920q1 ~]# nmcli dev
DEVICE     TYPE      STATE                   CONNECTION
eno1       ethernet  connected               eno1
bond0      bond      connected               bond0
br254      bridge    connected               br254
enp1s0f3   ethernet  connected               bond0-slave3
bond0.254  vlan      connected               v254
lo         loopback  connected (externally)  lo
enp1s0f0   ethernet  unavailable             --
enp1s0f1   ethernet  unavailable             --
enp1s0f2   ethernet  unavailable             --
```

There's no connectivity on the host because we told it to not do that (with our arguments `ipv4.method disabled ipv6.method ignore` for the bridge) but, assuming you spin up a guest, get it an IP somehow, and the rest of your network works, you should have connectivity via that tagged VLAN on your bond.

```txt
[liam@vm-on-v254-br254 ~]$ ip a | grep -E "(eth0|inet .*eth0)"
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 172.27.254.30/24 brd 172.27.254.255 scope global dynamic noprefixroute eth0

[liam@vm-on-v254-br254 ~]$ ping -c 1 1.1.1.1 | grep from
64 bytes from 1.1.1.1: icmp_seq=1 ttl=55 time=9.81 ms
```
### Scripting it

Who would want to do this manually for a bunch of subnets? Certainly not me! I wrote a quick Bash script to save some typing. Find the latest version [on GitHub](https://github.com/hpst3r/bridge_builder).

```sh
#!/bin/bash

# bond interface/connection name
bond_name="bond0"

# interfaces to slave to bond
interfaces=("enp1s0f0" "enp1s0f1" "enp1s0f2" "enp1s0f3")

# VLANs to add
vlans=(254 244 243 100 1925 1935)

bond_exists=$(nmcli con | grep -v "$bond_name")

if [[ ! "$bond_exists" ]]; then

  echo "Bond '${bond_name}' was not found - creating it."  

  # create the bond
  nmcli con add type bond \
    con-name "$bond_name" \
    ifname "$bond_name" \
    bond.options "mode=802.3ad" \
    ipv4.method disabled \
    ipv6.method disabled

else

  echo "Bond '${bond_name}' already exists - no changes were made."

fi

# slave physical interfaces to the bond
for interface in "${interfaces[@]}"; do
  
  connection_is_mastered=$(nmcli con sh "$interface" | grep master)
  
  if [[ ! "$connection_is_mastered" ]]; then

    echo "Interface '${interface}' is not mastered by bond '${bond_name}' - slaving it."

    nmcli con mod "$interface" \
      slave-type bond \
      master "$bond_name"

  else
    
    echo "Interface '${interface}' was already mastered by bond '${bond_name}' - no changes were made."

  fi

done

# create bridges and slave requested VLANs to them
for vlan in "${vlans[@]}"; do

  vlan_name="vlan${vlan}"

  vlan_exists=$(nmcli con | grep "$vlan_name")

  if [[ ! "$vlan_exists" ]]; then

    echo "VLAN ${vlan} connection '${vlan_name}' does not exist. Creating it and its bridge."

    bridge_name="bridge${vlan}"
  
    # add a disassociated bridge
    nmcli con add type bridge \
      con-name "$bridge_name" \
      ifname "$bridge_name" \
      ipv4.method disabled \
      ipv6.method ignore
  
    # add desired vlan to bond, slave it to the bridge
    nmcli con add type vlan \
      con-name "$vlan_name" \
      ifname "$bond_name"."$vlan" \
      dev "$bond_name" \
      id "$vlan" \
      master "$bridge_name" \
      slave-type bridge

  else

    echo "VLAN ${vlan} connection '${vlan_name}' already exists - no changes were made."

  fi
  
done
```

### Example NetworkManager config files

```txt
[root@m920q1 ~]# cat /etc/NetworkManager/system-connections/enp1s0f3.nmconnection
[connection]
id=enp1s0f3
uuid=c6cd876c-101d-40db-a498-ee79e97729c0
type=ethernet
controller=bond0
interface-name=enp1s0f3
port-type=bond

[ethernet]
```

```txt
[root@m920q1 ~]# cat /etc/NetworkManager/system-connections/bond0.nmconnection
[connection]
id=bond0
uuid=96527623-4564-46ab-92a6-a3c369224c85
type=bond
autoconnect-ports=1
interface-name=bond0

[ethernet]
cloned-mac-address=B4:96:91:8A:E7:BB

[bond]
downdelay=0
miimon=100
mode=802.3ad
updelay=0

[ipv4]
method=disabled

[ipv6]
addr-gen-mode=default
method=disabled

[proxy]
```

```txt
[root@m920q1 ~]# cat /etc/NetworkManager/system-connections/vlan1.nmconnection
[connection]
id=vlan1
uuid=5ffc8298-af81-472d-8c45-df714994398b
type=vlan
controller=vlanbr1
interface-name=vlan1
port-type=bridge
timestamp=1746326414

[ethernet]

[vlan]
flags=1
id=1
parent=bond0

[bridge-port]
```

```txt
[root@m920q1 ~]# cat /etc/NetworkManager/system-connections/vlanbr1.nmconnection
[connection]
id=vlanbr1
uuid=ba8966a8-7b23-453f-b6e9-29d313bac738
type=bridge
autoconnect-ports=1
interface-name=vlanbr1

[ethernet]

[bridge]
stp=false

[ipv4]
method=disabled

[ipv6]
addr-gen-mode=default
method=disabled

[proxy]
```

## Bonus - Configuring a per-VLAN bridge for your VMs using Cockpit

Cockpit is super nice and can be used to save you some typing for one-offs once you understand the base concepts here. Yes, you can do this from a web UI! Isn't that great?

Log on to your server with Cockpit. Navigate to the Networking tab.

{{< figure src="images/0-cockpit-networking-start.png" >}}

First, we'll create a bond. To do so, click "Add bond", then:
- Select the interfaces you'll bond
- Optionally, select a MAC for the bond to use
- Select the bond mode - 802.3ad is LACP
- Click "Add"

{{< figure src="images/1-cockpit-networking-addbond.png" >}}

Your interfaces list will change a bit - the interfaces slaved to the bond will be replaced with the bond. Isn't that nice?

{{< figure src="images/2-cockpit-networking-withbond.png" >}}

In my case, this bond is stuck trying to get an IP from a native VLAN that doesn't exist on the switch, because, by default, your bond will be set to `ipv4.method auto` and `ipv6.method auto`. Let's disable both of those - click on the blue link to the interface config page:

{{< figure src="images/3-cockpit-networking-editbond.png" >}}

And, on the interface config screen, set IPv4 and IPv6 to "Disabled". Head back to the Networking page when you're done.

{{< figure src="images/4-cockpit-networking-ipvdisabled.png" >}}

Now, it's time to add us a VLAN.

{{< figure src="images/5-cockpit-networking-addvlan.png" >}}

Go ahead and click "Add VLAN"!

Select the bond as the Parent, name it whatever, and select whatever VLAN ID you'd like. Then, once you're done, click "Add".

{{< figure src="images/6-cockpit-networking-addvlan.png" >}}

Look at that! We made ourselves a VLAN interface!

{{< figure src="images/7-cockpit-networking-addedvlan.png" >}}

Same deal here (as with the bond) with addressing - by default, your VLAN interface will try to get itself an IPv4 and IPv6 address. In my case, tagged VLAN 1 *does* have a way to get to a DHCP server, so the VLAN interface on the bond grabbed an IP.

You can disable that addressing the same way you disabled it for the bond itself earlier on - click into the interface and change IPv4 and IPv6 methods to Disabled. Or don't, if host connectivity to those networks is desired. Up to you.

Whew. Almost there! Let's create a bridge, so we can attach VMs to this VLAN. On the main Networking page, click "Add bridge".

Name the bridge whatever you'd like. Select the VLAN on the bond (in my case, this is `bond0.1`). Optionally, toggle STP (it's generally normal to leave this off on hypervisors) and then click "Add".

{{< figure src="images/8-cockpit-networking-addbridge.png" >}}

If you disabled networking on the VLAN, the bridge will have it off by default, so we're good to go!

{{< figure src="images/9-cockpit-networking-addedbridge.png" >}}

You should now be able to drop a VM on this bridge and go to town! Rinse and repeat for any other VLAN you'd like to use.

## Conclusion

That's all for today! Slightly more involved than the VMware method, but not too bad, all things considered. 

Now, back to cramming FRR in places it shouldn't be...