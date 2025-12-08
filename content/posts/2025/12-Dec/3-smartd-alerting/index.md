---
title: "smartd alerting > {eml,ntfy}"
date: 2025-12-08T01:30:00-00:00
draft: false
---

AlmaLinux 10.1 6.12.0 on amd64 (host "3060t0") with smartmontools 7.4-8.el10

See [smartd.conf(5)](https://linux.die.net/man/5/smartd.conf).

You probably want to know when a disk fails. smartd (typically included with the smartmontools package) can tell you.

If you don't have smartmontools, get it:

```sh
sudo dnf install -y smartmontools
```

Confirm the `smartd` service is running:

```txt
$ systemctl status smartd
○ smartd.service - Self Monitoring and Reporting Technology (SMART) Daemon
     Loaded: loaded (/usr/lib/systemd/system/smartd.service; enabled; preset: enabled)
     Active: inactive (dead)
       Docs: man:smartd(8)
             man:smartd.conf(5)
```

If needed, enable and start it:

```sh
sudo systemctl enable --now smartd
```

The configuration file for `smartd` is, by default, `/etc/smartmontools/smartd.conf`.

By default, this gets you:

```txt
$ ls -l /etc/smartmontools/
total 20
-rw-r--r--. 1 root root 6132 Jul 15 20:00 smartd.conf
drwxr-xr-x. 2 root root 4096 Jul 15 20:00 smartd_warning.d
-rwxr-xr-x. 1 root root 6714 Jul 15 20:00 smartd_warning.sh
```

`smartd.conf` is the configuration file; `smartd_warning.sh` is a script that will fire off an email in case of a warning (if `smartd.conf` is properly configured), and `smartd_warning.d` is where you can put custom warning scripts (if desired).

There is a dependency on a MTA to send mail; `smartd` doesn't have anything built-in. You can likely simply install and enable the Postfix package, but will need to configure SPF and DMARC if you want your emails to have any chance of reaching you.

If you want more info on getting Postfix (my preferred MTA) set up, [check out this post](https://wporter.org/basic-postfix-config).

```sh
sudo dnf install -y postfix; sudo systemctl enable --now postfix
```

By default, `smartd`'s alert script looks for `mail`, so consider installing `mailx`, too - the system will still use Postfix as a MTA, `mailx` just provides a mail user agent (a client-side interface) for your mail transfer agent (the server).

```sh
sudo dnf install -y mailx
```

The default smartd scan/alert setup is:

```sh
DEVICESCAN -H -m root -M exec /usr/libexec/smartmontools/smartdnotify -n standby,10,q
```

To adjust this to use your email addresses, it's simplest to drop a wrapper script in and call that instead of the default alerting script (that always uses the hostname).

Here are the two wrapper scripts I use:

One to send an email. This is required to specify the From= address. Sometimes I don't want to touch an existing mail setup on a server; generally my FQDNs don't match email domains.

```sh
sudo tee /etc/smartmontools/smartd_warning.d/mail.sh > /dev/null << 'EOT'
#!/bin/bash
# smartd notification wrapper

# query the disk by-id with SN
DISK=$(udevadm info --query=property --name="$SMARTD_DEVICE" | grep "^ID_SERIAL=" | cut -d= -f2)

FROM="$(hostname)@lab.wporter.org"
TO="noc@wporter.org"
SUBJECT="${SMARTD_FAILTYPE} for ${DISK} on $(hostname -f)"
MESSAGE="${SMARTD_MESSAGE}"

# Send via sendmail with explicit From header
/usr/sbin/sendmail -t <<EOF
From: ${FROM}
To: ${TO}
Subject: SMART alert: ${SUBJECT}

Hi there,

$(hostname -f)'s ${DISK} (${SMARTD_DEVICE}) triggered a SMART alert:

${MESSAGE}

EOF
EOT
```

One to send a push notification (via POST to ntfy.sh):

```sh
sudo tee /etc/smartmontools/smartd_warning.d/ntfy.sh > /dev/null << 'EOT'
#!/bin/bash
# smartd push notification wrapper

# query the disk by-id with SN
DISK=$(udevadm info --query=property --name="$SMARTD_DEVICE" | grep "^ID_SERIAL=" | cut -d= -f2)

# send a POST to ntfy for a push notif
curl -d "SMART alert: ${SMARTD_FAILTYPE} for ${DISK} on host $(hostname -f)" \
    https://ntfy.sh/foobar
EOT
```

Make both executable:

```sh
sudo chmod +x /etc/smartmontools/smartd_warning.d/{mail,ntfy}.sh
```

### Test alerting

You can test your alerts with some parameters in `smartd.conf`.

To run the built-in `smartd` alerting self-test, add the `-M test` directive:

```sh
sudo tee /etc/smartmontools/smartd.conf > /dev/null << 'EOT'
DEVICESCAN -H -m @ALL -M test -n standby,10,q
EOT
```

To get real alerts, you can set the `smartd` temperature maximum threshold to 10 degrees C:

```sh
sudo tee /etc/smartmontools/smartd.conf > /dev/null << 'EOT'
DEVICESCAN -H -m @ALL -W 0,5,10 -n standby,10,q
EOT
```

Restart `smartd` after either to get test alerts sent to you.

Here's an example of the email alert:

```txt
From: 3060t0@lab.wporter.org
To: noc@wporter.org
Subject: SMART alert: Temperature for Samsung_SSD_860_EVO_250GB_S3YHNX0K851966K on 3060t0.ayu-trench.ts.net
Message-Id: <20251208065117.3B87D840815@3060t0.lab.wporter.org>
Date: Mon,  8 Dec 2025 01:51:17 -0500 (EST)

Hi there,

3060t0.ayu-trench.ts.net's Samsung_SSD_860_EVO_250GB_S3YHNX0K851966K (/dev/sda) triggered a SMART alert:

Device: /dev/sda [SAT], Temperature 25 Celsius reached critical limit of 10 Celsius (Min/Max ??/25)
```

Here's an example of the ntfy alert:

{{< figure src="smartd-ntfy.png" >}}

Once done testing, set your `smartd.conf` back to normal. Mine is just:

```sh
sudo tee /etc/smartmontools/smartd.conf > /dev/null << 'EOT'
DEVICESCAN -H -m @ALL -n standby,10,q
EOT
```

- `DEVICESCAN` tells `smartd` to scan all the disks in the system,
- `-H` enables overall health monitoring (if a disk reports failing health, a warning will be issued)
- `-m @ALL` means "if tripped, run all scripts in `/etc/smartmontools/smartd_warning.d/*`"
- `-n` is the "power mode directive". This tells `smartd` to not wake spun-down disks.
  - `standby` = skip tests if disk is in standby
  - `10` = maximum check interval, poll the disk to see if still asleep
  - `q` = do not alert if a test is skipped due to standby
