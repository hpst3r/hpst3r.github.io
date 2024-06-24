---
title: "Disabling systemd-resolved and letting NetworkManager control /etc/resolv.conf on Fedora 40"
date: 2024-06-23T12:34:56-00:00
draft: false
---

Resolved does not have sane defaults, is not simple to configure, and makes troubleshooting more difficult. It also regularly fails to resolve things properly for no apparent reason.

Here is how to disable it and allow NetworkManager to manage the `/etc/resolv.conf` file instead. Shockingly, things stop randomly breaking once this is done. Please assume this has only been tested on Fedora 40.

```txt
# nvim /etc/NetworkManager/NetworkManager.conf
# add the "dns=default" line under [main] like so:

[main]
#plugins=keyfile,ifcfg-rh
dns=default

:x
# systemctl stop systemd-resolved
# systemctl mask systemd-resolved
Created symlink /etc/systemd/system/systemd-resolved.service → /dev/null.
# rm /etc/resolv.conf
# systemctl restart NetworkManager
```