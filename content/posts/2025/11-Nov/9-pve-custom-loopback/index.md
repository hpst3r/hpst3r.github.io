---
title: "Point PVE system at a custom loopback interface to allow dynamic IPs without dependencies"
date: 2025-11-29T11:30:00-00:00
draft: false
---

PVE 9.1.1 6.17.2-2 (2025-11-26T12:33Z) on amd64

Proxmox VE's cluster services *heavily* rely on DNS (they need the node to resolve its hostname to a non-loopback IP address), even in a single-node scenario. This is just a byproduct of its "always-clustered", multi-master design. This poses a problem if you'd like a self-contained, portable PVE system without dependencies on external DNS or a static network configuration (e.g., a static entry in `/etc/hosts`, which is the default behavior).

The cluster services won't launch until the system can resolve itself to a non-loopback address. We can work around this by creating a new loopback address (e.g., `lo0`) with a non-loopback IP. Bear in mind that this will break clustering.

Here's my default `/etc/network/interfaces` in a simple configuration with a single bridge and a static IPv4 address:

```txt
auto lo
iface lo inet loopback

iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
  address 192.168.77.25/24
  gateway 192.168.77.254
  bridge-ports nic0
  bridge-stp off
  bridge-fd 0
```

To supplement this, we'll create a file in `/etc/network/interfaces.d`, e.g., `lo0`, to create a new loopback interface with a "non-loopback" IP address. In this case, we use a link-local IPv4 address in the 169.254.0.0/16 space as this isn't normally routed and won't conflict with most networks. Proxmox will use our new static, internal loopback interface rather than requiring an external or public IP for its cluster services.

We use the drop-in `interfaces.d` to ensure the configuration persists across updates and won't be overwritten by a service or administrator writing to the `/etc/network/interfaces` file (always try to use drop-in directories or override files when available).

Note that we're attempting to automatically create and remove the interface when needed/no longer needed with `pre-up` and `post-down` hooks.

```txt
auto lo0
iface lo0 inet static
  pre-up ip link add lo0 type dummy 2>/dev/null || true
  address 169.254.0.1/32
  post-down ip link del lo0 2>/dev/null || true
```

Add an entry to `/etc/hosts`. Replace "800g4m" with your hostname if applicable.

```txt
169.254.0.1 800g4m.lab.wporter.org 800g4m
```

Bring the link up:

```sh
ifup lo0
```

Confirm the interface exists, the system resolves itself to the link-local address, and it's reachable:

```txt
root@800g4m:~# getent hosts 800g4m
169.254.0.1     800g4m.lab.wporter.org 800g4m

root@800g4m:~# ping 800g4m.lab.wporter.org -c1
PING 800g4m.lab.wporter.org (169.254.0.1) 56(84) bytes of data.
64 bytes from 800g4m.lab.wporter.org (169.254.0.1): icmp_seq=1 ttl=64 time=0.024 ms

--- 800g4m.lab.wporter.org ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.024/0.024/0.024/0.000 ms
```

If you reboot the system, your cluster services should come back without issue, and you can change the system's external IP however you'd like.

Check `systemctl status pve-cluster.service` or the journal (`journalctl -u pve-cluster`) to see what IP PVE is using for itself at boot:

```txt
resolved node name '800g4m' to '169.254.0.1'
```
