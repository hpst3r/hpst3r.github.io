---
title: "Install a root certificate on Proxmox VE, BS, DM"
date: 2025-11-28T20:30:00-00:00
draft: false
---

Proxmox uses the system certificate store, so install your certs the same way you would on any other Debian system:

```sh
scp root-ca-a.crt root@proxmox:~
ssh root@proxmox
mv root-ca-a.crt /usr/local/share/ca-certificates/
update-ca-certificates
```
