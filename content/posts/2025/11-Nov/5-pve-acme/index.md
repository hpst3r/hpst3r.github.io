---
title: "Proxmox VE & Backup Server built-in ACME cert renewal"
date: 2025-11-28T21:30:00-00:00
draft: false
---

PVE 9.1.1 6.17.2-2 (2025-11-26T12:33Z), PBS 4.1.0 6.17.2-1 (2025-10-21T11:55Z) on amd64

Relevant docs: [Proxmox VE docs - Certificate Management](https://pve.proxmox.com/wiki/Certificate_Management), [Proxmox BS docs - HTTPS Certificate Configuration](https://pbs.proxmox.com/wiki/HTTPS_Certificate_Configuration), `pvenode(1)`, `proxmox-backup-manager(1)`.

If you're using a custom ACME responder with a private (not publicly trusted) certificate, **be sure to install your root cert** and `update-ca-certificates` before trying to connect to the CA via HTTPS to request a cert.

## PVE

Create an ACME account on the PVE box:

```sh
pvenode acme account register acct_name email_addr --directory https://ca.domain.example/acme/directory
```

```txt
root@800g4m0:~# pvenode acme account register internal noc@wporter.org --directory https://intermediate-ca.lab.wporter.org/acme/acme/directory

Attempting to fetch Terms of Service from 'https://intermediate-ca.lab.wporter.org/acme/acme/directory'..
No Terms of Service found, proceeding.

Attempting to register account with 'https://intermediate-ca.lab.wporter.org/acme/acme/directory'..
Generating ACME account key..
Registering ACME account..
Registration successful, account URL: 'https://intermediate-ca.lab.wporter.org/acme/acme/account/ZPNt6whonOmBsc3CLUU1z7Cu0z6W411u'
Task OK
```

Configure the ACME 'requester' with account name (from above) and the domain name you'll be requesting a certificate for. In my case, this is `800g4m0.pve.lab.wporter.org`:

```sh
pvenode config set --acme account=internal --acmedomain0 800g4m.lab.wporter.org
```

To have a peek at your config, use `pvenode config get`:

```txt
root@800g4m0:~# pvenode config get
acme: account=internal
acmedomain0: 800g4m0.pve.lab.wporter.org
```

To request a cert from the CA, use `pvenode acme cert order`:

```txt
root@800g4m:~# pvenode acme cert order
Loading ACME account details
Placing ACME order
Order URL: https://intermediate-ca.lab.wporter.org/acme/acme/order/F25BAivKY9NODhZfKWfn2npNF1FFBUbR

Getting authorization details from 'https://intermediate-ca.lab.wporter.org/acme/acme/authz/JX0ov7kupqwdQX0hWhIN9XDLwsgLzs86'
The validation for 800g4m0.pve.lab.wporter.org is pending!
Setting up webserver
Triggering validation
Sleeping for 5 seconds
Status is 'valid', domain '800g4m0.pve.lab.wporter.org' OK!

All domains validated!

Creating CSR
Checking order status
Order is ready, finalizing order
valid!

Downloading certificate
Setting pveproxy certificate and key
Restarting pveproxy
Task OK
```

Certificates are stored at `/etc/pve/local/pveproxy-ssl.pem` (`/etc/pve/nodes/node/pveproxy-ssl.pem`) and are renewed by the `/usr/bin/pveupdate` Perl script, called by the `pve-daily-update.service` oneshot (which itself is called by the `pve-daily-update.timer`).

By default, this timer runs daily at 1 AM, with a random 5H delay:

```ini
## /usr/lib/systemd/system/pve-daily-update.timer
[Unit]
Description=Daily PVE download activities

[Timer]
OnCalendar=*-*-* 1:00
RandomizedDelaySec=5h
Persistent=true

[Install]
WantedBy=timers.target
```

Since my certificates only last a day, this timer is too long! To override it in a way that won't be overwritten by an update, create the `/etc/systemd/system/pve-daily-update.timer.d/override.conf` file:

```sh
mkdir -p /etc/systemd/system/pve-daily-update.timer.d
tee /etc/systemd/system/pve-daily-update.timer.d/override.conf > /dev/null << 'EOT'
[Timer]
OnCalendar=
OnCalendar=00/4:00
RandomizedDelaySec=0
EOT
```

This will:

- clear the default timing with `OnCalendar=`
- set the timer to run every four hours with `OnCalendar=00/4:00`
- clear the randomized delay with `RandomizedDelaySec=0`

Then, run `systemctl daemon-reload` to reload your units.

## PBS

Swap `pvenode` for `proxmox-backup-manager` - otherwise, it's pretty similar to Proxmox VE.

PBS currently asks you about ACME external account binding - if you don't know what this is, say **N**o.

```sh
proxmox-backup-manager acme account register acct_name email_addr --directory https://ca.domain.example/acme/directory
```

```txt
root@pbs:~# proxmox-backup-manager acme account register internal noc@wporter.org --directory https://intermediate-ca.lab.wporter.org/acme/acme/directory
Attempting to fetch Terms of Service from "https://intermediate-ca.lab.wporter.org/acme/acme/directory"
No Terms of Service found, proceeding.
Do you want to use external account binding? [y|N]: N
Attempting to register account with "https://intermediate-ca.lab.wporter.org/acme/acme/directory"...
Registration successful, account URL: https://intermediate-ca.lab.wporter.org/acme/acme/account/Qo6lhhqkZXvaYt8AhQDHWd03BOEvQ38b
```

```sh
proxmox-backup-manager node update --acme account=internal --acmedomain0 pbs.lab.wporter.org
```

PBS is a little different than PVE - you'll need to `--force` a cert update, or it will say that since the self-signed cert expires more than 30d in the future, everything is fine.

Try to request a cert:

```sh
proxmox-backup-manager acme cert order --force
```

If this hangs forever with no logs (the PBS CLI and ACME implementation like to do this), confirm that:

- DNS works (both ways)
- You have installed your root cert on PBS
- The CA can reach port 80 on PBS
- PBS can reach 443/acme/directory on the CA
- This directory exists (it didn't for me):
  - `/var/lib/proxmox-backup/.well-known/acme-challenge`
- Permissions on `/var/lib/proxmox-backup/.well-known` are `755` `backup:backup`:
  - `chown -R backup:backup /var/lib/proxmox-backup/.well-known`
  - `chmod -R 755 /var/lib/proxmox-backup/.well-known`

Certs are stored at `/etc/proxmox-backup/proxy.pem` and `/etc/proxmox-backup/proxy.key`.

The cert will be renewed by the `proxmox-backup-daily-update.service` (similarly to Proxmox VE).

Same deal with the timer - you can override it with:

```sh
mkdir -p /etc/systemd/system/proxmox-backup-manager.timer.d
tee /etc/systemd/system/proxmox-backup-manager.timer.d/override.conf > /dev/null << 'EOT'
[Timer]
OnCalendar=
OnCalendar=00/4:00
RandomizedDelaySec=0
EOT
```

## PDM

Haha. If you thought documentation for PBS or PVE is bad.. well, this is nonexistent. Not even going to try.

```txt
root@pdm:~# ls /usr/sbin/proxmox-*
/usr/sbin/proxmox-boot-tool  /usr/sbin/proxmox-datacenter-manager-admin

root@pdm:~# proxmox-datacenter-manager-admin
Error: no command specified.
Possible commands: help, remote, versions

Usage:

proxmox-datacenter-manager-admin help [{<command>}] [OPTIONS]
proxmox-datacenter-manager-admin remote add --authid <string> --id <string> --nodes <string> --token <string> --type pve|pbs [OPTIONS]
proxmox-datacenter-manager-admin remote list [OPTIONS]
proxmox-datacenter-manager-admin remote remove <id>
proxmox-datacenter-manager-admin remote update <id> [OPTIONS]
proxmox-datacenter-manager-admin remote version <id>
proxmox-datacenter-manager-admin versions [OPTIONS]

root@pdm:~# man proxmox-datacenter-manager-admin
No manual entry for proxmox-datacenter-manager-admin
```
