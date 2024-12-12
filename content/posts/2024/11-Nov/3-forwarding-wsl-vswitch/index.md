---
title: "Snippet: enabling forwarding between Hyper-V NAT vSwitches (WSL & int. NAT)"
date: 2024-11-26T12:34:56-00:00
draft: false
---

https://superuser.com/a/1764704

In my case, with NAT switch 'vlab0-natswitch' and the default WSL NAT switch (so I can run Ansible playbooks against VMs behind a different NAT switch) the following one-liner does the trick. Adjust interfacealias values to suit.

```pwsh
get-netipinterface | where {$_.interfacealias -eq 'vEthernet (vlab0-natswitch)' -or $_.interfacealias -eq 'vEthernet (WSL (Hyper-V firewall))'} | set-netipinterface -forwarding Enabled -verbose
```

Works!

```txt
liam@liam-p1g4i-0:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.255.255.254/32 brd 10.255.255.254 scope global lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:b7:bc:42 brd ff:ff:ff:ff:ff:ff
    inet 172.28.133.19/20 brd 172.28.143.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::215:5dff:feb7:bc42/64 scope link
       valid_lft forever preferred_lft forever
liam@liam-p1g4i-0:~$ uname -r
5.15.167.4-microsoft-standard-WSL2
liam@liam-p1g4i-0:~$ ping 192.0.2.10
PING 192.0.2.10 (192.0.2.10) 56(84) bytes of data.
64 bytes from 192.0.2.10: icmp_seq=1 ttl=127 time=1.41 ms
```
