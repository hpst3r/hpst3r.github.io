---
title: "NTP - chrony and timedatectl"
date: 2026-05-09T20:15:00-00:00
draft: false
---

On EL systems, the `chrony` NTP client and server is used to synchronize the system's time.

The configuration file is `/etc/chrony.conf`; the `chronyc` and `timedatectl` commands can be used to monitor the chrony daemon and configure the system clock, respectively.

The default configuration typically point chrony at a NTP pool (distro-specific). Pools are defined with the `pool` configuration line:

```sh
$ cat /etc/chrony.conf | grep pool
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (https://www.pool.ntp.org/join.html).
pool 2.almalinux.pool.ntp.org iburst
```

Servers are defined with the `server` configuration line instead.

> If you're curious, the `iburst` option simply means "send an initial burst of NTP packets on first contact to sync at startup".

> Pools are DNS records resolved to multiple nearby NTP servers. Servers are single static hosts. These configuration options are separate as Chrony has additional plumbing to query multiple pool members simultaneously, score servers based on response time, stratum, and consistency, drop unreliable servers, and automatically replace dropped servers by querying the pool again - these are not needed if you have just a single defined server.

To enable `chrony`'s NTP server functionality, you can simply configure the IP allowlist (the `allow` config line, e.g., `allow 192.168.0.0/16`) and permit NTP in on your firewall (`--add-service=ntp` or permit UDP 123).

To list your NTP sources, run `chronyc sources`:

```sh
$ chronyc sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^- 38.28.93.135                  2  10   377   108    +40us[  +40us] +/-   28ms
^* 50.205.57.38                  1  10   377   694    +36us[-5791ns] +/- 6256us
^- 66.85.78.80                   2  10   377   138   -137us[ -137us] +/-   40ms
^- t1.time.gq1.yahoo.com         2  10   377   606   +498us[ +498us] +/-  107ms
^- _gateway                      2   8   377   254   +892us[ +892us] +/-   26ms
```

To see the details of your NTP daemon (e.g., time offset, skew, update interval), run `chronyc tracking`:

```sh
$ chronyc tracking
Reference ID    : 32CD3926 (50.205.57.38)
Stratum         : 2
Ref time (UTC)  : Sat Apr 25 17:08:24 2026
System time     : 0.000031090 seconds slow of NTP time
Last offset     : -0.000041857 seconds
RMS offset      : 0.000147579 seconds
Frequency       : 4.936 ppm slow
Residual freq   : -0.017 ppm
Skew            : 0.149 ppm
Root delay      : 0.012512143 seconds
Root dispersion : 0.000895039 seconds
Update interval : 1040.9 seconds
Leap status     : Normal
```

To force an immediate sync, you can run `chronyc makestep`.

`timedatectl status` will show you details about the current time zone, local time, and universal time:

```sh
$ timedatectl status
               Local time: Sat 2026-04-25 17:20:19 UTC
           Universal time: Sat 2026-04-25 17:20:19 UTC
                 RTC time: Sat 2026-04-25 17:20:19
                Time zone: UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

To enable NTP with `timedatectl`, run, `timedatectl set-ntp true`.

Perhaps the most general use of `timedatectl` is for setting your timezone, which you can do with the aptly-named `set-timezone` parameter. You can run the `list-timezones` parameter to enumerate available timezones.

```sh
$ timedatectl list-timezones | grep New
America/New_York
America/North_Dakota/New_Salem
Canada/Newfoundland

$ sudo timedatectl set-timezone America/New_York

$ timedatectl status
               Local time: Sat 2026-04-25 13:20:41 EDT
           Universal time: Sat 2026-04-25 17:20:41 UTC
                 RTC time: Sat 2026-04-25 17:20:41
                Time zone: America/New_York (EDT, -0400)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

```sh
$ date
Sat Apr 25 01:20:44 PM EDT 2026
```
