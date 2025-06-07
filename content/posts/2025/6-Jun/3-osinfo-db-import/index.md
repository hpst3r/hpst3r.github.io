---
title: "Updating the osinfo database on a Linux hypervisor"
date: 2025-06-07T17:00:00-00:00
draft: false
---

The osinfo database provides "all the information about an operating system that is required in order to provision and manage it in a virtualized environment." ([libosinfo.org](https://libosinfo.org/))

In a nutshell, it's metadata about guest OSes for use by hypervisors. I haven't poked at it more than this, but I like it when my guest OSes are recognized, there are two that I now use commonly that are not recognized: Windows Server 2025 and AlmaLinux 10.

To update the osinfo database, you can use `osinfo-db-import(1)` (provided by `osinfo-db-tools` in AlmaLinux).

If this is run elevated, it updates the database for all users (updates `/etc/osinfo`). If it is not, it updates the database for only the current user (updates `~/.config/osinfo`). You can override this behavior with the `--user` or `--local` arguments, or overwrite the system (packaged) `osinfo-db` with the `--system` argument.

To update to the latest release for all users (at the time of writing, this does *not* provide AlmaLinux 10 or Server 2025):

```sh
$ sudo osinfo-db-import --latest
```

To update to the nightly version of the osinfo database for all users:

```sh
$ sudo osinfo-db-import --nightly
```

Alternatively, you can get a new version of the osinfo database from [releases.pagure.org/libosinfo](https://releases.pagure.org/libosinfo/) and manually import it.

```sh
$ wget https://releases.pagure.org/libosinfo/osinfo-db-20250124.tar.xz
--2025-06-07 16:53:13--  https://releases.pagure.org/libosinfo/osinfo-db-20250124.tar.xz
Resolving releases.pagure.org (releases.pagure.org)... 8.43.85.76, 2620:52:3:1:dead:beef:cafe:fed8
Connecting to releases.pagure.org (releases.pagure.org)|8.43.85.76|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 159540 (156K) [application/x-xz]
Saving to: ‘osinfo-db-20250124.tar.xz’

osinfo-db-20250124.tar.xz       100%[======================================================>] 155.80K  --.-KB/s    in 0.05s   

2025-06-07 16:53:14 (2.78 MB/s) - ‘osinfo-db-20250124.tar.xz’ saved [159540/159540]

$ sudo osinfo-db-import --local osinfo-db-20250124.tar.xz
```

To verify your changes, you can use `osinfo-query` to query the database for recognized operating systems.

```sh
wporter@c300:~$ osinfo-query os | grep -i alma
 almalinux8           | AlmaLinux 8                                        | 8        | http://almalinux.org/almalinux/8        
 almalinux9           | AlmaLinux 9                                        | 9        | http://almalinux.org/almalinux/9        
wporter@c300:~$ sudo osinfo-db-import --latest
wporter@c300:~$ osinfo-query os | grep -i alma
 almalinux8           | AlmaLinux 8                                        | 8        | http://almalinux.org/almalinux/8        
 almalinux9           | AlmaLinux 9                                        | 9        | http://almalinux.org/almalinux/9        
wporter@c300:~$ sudo osinfo-db-import --nightly
wporter@c300:~$ osinfo-query os | grep -i alma
 almalinux-kitten10   | AlmaLinux Kitten 10                                | 10       | http://almalinux.org/almalinux-kitten/10
 almalinux10          | AlmaLinux 10                                       | 10       | http://almalinux.org/almalinux/10       
 almalinux8           | AlmaLinux 8                                        | 8        | http://almalinux.org/almalinux/8        
 almalinux9           | AlmaLinux 9                                        | 9        | http://almalinux.org/almalinux/9        
```

That was an easy one! Yay!

Here's a demo of what running `osinfo-db-import` elevated (or not) does:

```sh
[wporter@3060t0 ~]$ sudo rm -rf /etc/osinfo; rm -rf ~/.config/osinfo
[wporter@3060t0 ~]$ osinfo-query os | grep -i 'Windows Server 2025'
[wporter@3060t0 ~]$ # nada
[wporter@3060t0 ~]$ osinfo-query os | grep -i 'Windows Server'
 win2k12              | Microsoft Windows Server 2012                      | 6.3      | http://microsoft.com/win/2k12           
 win2k12r2            | Microsoft Windows Server 2012 R2                   | 6.3      | http://microsoft.com/win/2k12r2         
 win2k16              | Microsoft Windows Server 2016                      | 10.0     | http://microsoft.com/win/2k16           
 win2k19              | Microsoft Windows Server 2019                      | 10.0     | http://microsoft.com/win/2k19           
 win2k22              | Microsoft Windows Server 2022                      | 10.0     | http://microsoft.com/win/2k22           
 win2k3               | Microsoft Windows Server 2003                      | 5.2      | http://microsoft.com/win/2k3            
 win2k3r2             | Microsoft Windows Server 2003 R2                   | 5.2      | http://microsoft.com/win/2k3r2          
 win2k8               | Microsoft Windows Server 2008                      | 6.0      | http://microsoft.com/win/2k8            
 win2k8r2             | Microsoft Windows Server 2008 R2                   | 6.1      | http://microsoft.com/win/2k8r2          
[wporter@3060t0 ~]$ # this is the packaged osinfo-db (from AlmaLinux 10.0) - only up to 22
[wporter@3060t0 ~]$ osinfo-db-import --nightly # update my personal ~/.config/osinfo db
[wporter@3060t0 ~]$ osinfo-query os | grep -i 'Windows Server 2025'
 win2k25              | Microsoft Windows Server 2025                      | 10.0     | http://microsoft.com/win/2k25           
[wporter@3060t0 ~]$ su - temp
Password: 
Last login: Sat Jun  7 16:57:43 EDT 2025 on pts/1
[temp@3060t0 ~]$ osinfo-query os | grep -i 'Windows Server 2025'
[temp@3060t0 ~]$ # nada
[temp@3060t0 ~]$ exit
logout
[wporter@3060t0 ~]$ rm -rf ~/.config/osinfo # clear local config
[wporter@3060t0 ~]$ osinfo-query os | grep -i 'Windows Server 2025'
[wporter@3060t0 ~]$ # nada; back to osinfo packaged with Alma 10.0
[wporter@3060t0 ~]$ # import an update to the system database at /etc/osinfo
[wporter@3060t0 ~]$ sudo osinfo-db-import --nightly
[wporter@3060t0 ~]$ su - temp
Password: 
Last login: Sat Jun  7 17:04:11 EDT 2025 on pts/1
[temp@3060t0 ~]$ osinfo-query os | grep -i 'Windows Server 2025'
 win2k25              | Microsoft Windows Server 2025                      | 10.0     | http://microsoft.com/win/2k25           
[temp@3060t0 ~]$ # updated for all users
[temp@3060t0 ~]$ exit
logout
```
