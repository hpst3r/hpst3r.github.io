---
title: "Renaming a standalone Proxmox node with guests"
date: 2025-11-28T21:30:00-00:00
draft: false
---

Reference [docs](https://pve.proxmox.com/wiki/Renaming_a_PVE_node), [this forum post](https://forum.proxmox.com/threads/changing-hostname-and-ip-of-non-empty-pve-host.112068/).

I would really recommend shutting down your guests before doing this.

Modify `/etc/hosts` (OR your DNS record, if you're not using the hosts file), `/etc/hostname`, `/etc/mailname` (if applicable), `/etc/postfix/main.cf`:

```sh
sed -i 's/oldname/newname/g' /etc/{hosts,hostname,/postfix/main.cf}
```

Then, it's simplest to stop the cluster filesystem and edit the backing database to rename the `/etc/pve/nodes` directory for the server.

Yes, really, though fighting the cluster FS and moving, not copying, the files in mounted `/etc/pve/nodes/old` to `/etc/pve/nodes/new` (directory by directory - you cannot move/rename a non-empty directory on the pmxcfs, so you will have to move `/old/qemu-server/*`, etc) should also work if this makes you squeamish.

```sh
systemctl stop pve-cluster
systemctl stop corosync

sqlite3 /var/lib/pve-cluster/config.db "UPDATE tree SET name='newname' WHERE name='oldname';"

systemctl start corosync
systemctl start pve-cluster
```

Finally, copy `/var/lib/rrdcached/db/pve-{node,storage}/old` to `/var/lib/rrdcached/db/pve-{node,storage}/new`, then reboot.
