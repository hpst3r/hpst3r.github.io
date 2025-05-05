---
title: "Disabling systemd-resolved and letting NetworkManager control /etc/resolv.conf on Fedora 40"
date: 2024-06-23T12:34:56-00:00
draft: false
---

Resolved regularly fails to resolve things properly for no apparent reason, and was kind of a pain to troubleshoot when I first tried to do so.

Here is how to disable it and allow NetworkManager to manage the `/etc/resolv.conf` file instead. Shockingly, things stop randomly breaking once this is done.

- Reconfigure NetworkManager to permit it to manage DNS resolvers
- Stop and mask the resolved service
- Remove `/etc/resolv.conf` to clear out existing contents
- Restart NetworkManager to apply the configuration change

First, allow NetworkManager to configure the system's DNS resolvers by editing the `/etc/NetworkManager/NetworkManager.conf` file.

```txt
# nvim /etc/NetworkManager/NetworkManager.conf
# add the "dns=default" line under [main] like so:

[main]
#plugins=keyfile,ifcfg-rh
dns=default

:x
```

Alternatively, with `sed(1)`:

```sh
sed -i -e '/^dns=/d' \
  -e '/\[main\]/a dns=default' \
  /etc/NetworkManager/NetworkManager.conf
```

Next, stop and mask the resolved service. Note that this WILL break DNS resolution until you remove resolv.conf and restart NetworkManager. Masking the service will disable it.

```sh
systemctl stop systemd-resolved; systemctl mask systemd-resolved
```

Finally, remove your resolved-generated `/etc/resolv.conf`...

```sh
rm /etc/resolv.conf
```

And restart NetworkManager to apply the config change made earlier (tell it to manage resolvers).

```sh
systemctl restart NetworkManager
```

All in one:

```sh
# disable resolved
sudo systemctl stop systemd-resolved
sudo systemctl mask systemd-resolved

# remove anything ^dns= then
# insert the "dns=default" line in [main] section
sudo sed -i -e '/^dns=/d' \
    -e '/\[main\]/a dns=default' \
    /etc/NetworkManager/NetworkManager.conf
    
# clean out resolv config so NM will edit it
sudo rm /etc/resolv.conf
# restart NM to make it reread config
sudo systemctl restart NetworkManager
```