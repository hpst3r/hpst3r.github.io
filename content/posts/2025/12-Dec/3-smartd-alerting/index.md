---
title: "smartd > {eml,ntfy} on Alma 9 & 10, PVE 9"
date: 2025-12-08T01:30:00-00:00
draft: false
---

AlmaLinux 10.1 6.12.0 on amd64 (host "3060t0") with smartmontools 7.4

Proxmox VE 9.1.1 6.17.2-1-pve on amd64 (host "800g4m0") with smartmontools 7.4

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

The default smartd scan/alert setup is:

```sh
DEVICESCAN -H -m root -M exec /usr/libexec/smartmontools/smartdnotify -n standby,10,q
```

This will:

- (`-M exec`) Run the default notify script, sending
- (`-m`) mail to root
- (`-H`) if a SMART health event is logged
- (`-n standby,10,q`) and avoid waking up disks if possible.

To adjust this to use your email addresses, it's simplest to drop a wrapper script in and call that instead of the default alerting script (that always uses the hostname).

Here are the two wrapper scripts I use:

One to send an email. This is required to specify the From= address (by default, smartd doesn't support this). Sometimes I don't want to touch an existing mail setup on a server; generally my FQDNs don't match email domains or are unreliable (e.g., tailnet names).

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

One to send a push notification (via POST to ntfy.sh, with subscription string "foobar"):

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

## SELinux

SELinux will block the `udevadm` and `curl` calls if enforcing, since the default `smartdwarn_t` context is very restrictive (I think mail was working).

I believe the required packages are installed out of the box with most Alma 10 profiles. If not you'll likely need:

```sh
sudo dnf install -y checkpolicy policycoreutils
```

Here is a generated policy patch for EL10 that should allow the above scripts to be run in the `smartdwarn_t` context:

```sh
tee allow_smartdwarn_udev_curl_mail.te > /dev/null << 'EOT'

module allow_smartdwarn_udev_curl_mail 1.0;

require {
  type net_conf_t;
  type sysfs_t;
  type fsdaemon_t;
  type kernel_t;
  type postfix_cleanup_t;
  type security_t;
  type postfix_master_t;
  type file_context_t;
  type udev_var_run_t;
  type setroubleshootd_t;
  type system_mail_t;
  type selinux_config_t;
  type rpm_var_lib_t;
  type fixed_disk_device_t;
  type cert_t;
  type http_port_t;
  type postfix_postdrop_t;
  type hostname_t;
  type default_context_t;
  type udev_exec_t;
  type smartdwarn_t;
  type postfix_smtp_t;
  class process { noatsecure rlimitinh siginh };
  class file { execute execute_no_trans getattr map open read setattr write };
  class dir { getattr open read search };
  class blk_file getattr;
  class lnk_file { getattr read };
  class unix_dgram_socket { create getopt sendto setopt write };
  class udp_socket { connect create getattr read write };
  class netlink_route_socket { bind create getattr nlmsg_read read write };
  class tcp_socket { connect create getattr getopt name_connect read setopt write };
  class fifo_file { getattr write };
}

#============= fsdaemon_t ==============
allow fsdaemon_t smartdwarn_t:process { noatsecure rlimitinh siginh };

#============= hostname_t ==============
allow hostname_t fsdaemon_t:fifo_file write;

#============= postfix_master_t ==============
allow postfix_master_t postfix_cleanup_t:process { noatsecure rlimitinh siginh };
allow postfix_master_t postfix_smtp_t:process { noatsecure rlimitinh siginh };

#============= postfix_postdrop_t ==============
allow postfix_postdrop_t fsdaemon_t:fifo_file { getattr write };

#============= setroubleshootd_t ==============
allow setroubleshootd_t rpm_var_lib_t:file { setattr write };

#============= smartdwarn_t ==============
allow smartdwarn_t cert_t:dir { getattr open read search };
allow smartdwarn_t cert_t:file { getattr open read };
allow smartdwarn_t default_context_t:dir search;
allow smartdwarn_t file_context_t:dir search;
allow smartdwarn_t file_context_t:file { getattr open read };

#!!!! This avc can be allowed using the boolean 'domain_can_mmap_files'
allow smartdwarn_t file_context_t:file map;
allow smartdwarn_t fixed_disk_device_t:blk_file getattr;
allow smartdwarn_t hostname_t:process { noatsecure rlimitinh siginh };
allow smartdwarn_t http_port_t:tcp_socket name_connect;
allow smartdwarn_t kernel_t:unix_dgram_socket sendto;
allow smartdwarn_t net_conf_t:file { getattr open read };
allow smartdwarn_t net_conf_t:lnk_file read;
allow smartdwarn_t security_t:file { open read };

#!!!! This avc can be allowed using the boolean 'domain_can_mmap_files'
allow smartdwarn_t security_t:file map;
allow smartdwarn_t self:netlink_route_socket { bind create getattr nlmsg_read read write };
allow smartdwarn_t self:tcp_socket { connect create getattr getopt read setopt write };
allow smartdwarn_t self:udp_socket { connect create getattr read write };
allow smartdwarn_t self:unix_dgram_socket { create getopt setopt write };
allow smartdwarn_t selinux_config_t:file { getattr open read };
allow smartdwarn_t sysfs_t:file { getattr open read };
allow smartdwarn_t sysfs_t:lnk_file { getattr read };
allow smartdwarn_t system_mail_t:process { noatsecure rlimitinh siginh };
allow smartdwarn_t udev_exec_t:file { execute execute_no_trans getattr open read };

#!!!! This avc can be allowed using the boolean 'domain_can_mmap_files'
allow smartdwarn_t udev_exec_t:file map;
allow smartdwarn_t udev_var_run_t:dir { getattr search };
allow smartdwarn_t udev_var_run_t:file { getattr open read };

#============= system_mail_t ==============
allow system_mail_t postfix_postdrop_t:process { noatsecure rlimitinh siginh };
EOT
```

Here's something for EL9, which uses different, less restrictive SELinux policy out of the box:

```sh
tee allow_smartdwarn_udev_curl_mail.te > /dev/null << 'EOT'

module allow_smartdwarn_udev_curl_mail 1.0;

require {
  type http_port_t;
  type fsdaemon_t;
  type security_t;
  type selinux_config_t;
  class file { getattr map open read };
  class tcp_socket name_connect;
}

#============= fsdaemon_t ==============

#!!!! This avc can be allowed using the boolean 'nis_enabled'
allow fsdaemon_t http_port_t:tcp_socket name_connect;

#!!!! This avc can be allowed using the boolean 'smartmon_3ware'
allow fsdaemon_t security_t:file { map open read };
allow fsdaemon_t selinux_config_t:file { getattr open read };
EOT
```

To check, compile, and install the module:

```sh
sudo checkmodule -M -m -o allow_smartdwarn_udev_curl_mail.mod allow_smartdwarn_udev_curl_mail.te
sudo semodule_package -o allow_smartdwarn_udev_curl_mail.pp -m allow_smartdwarn_udev_curl_mail.mod
sudo semodule -i allow_smartdwarn_udev_curl_mail.pp
```

To remove the module (if needed later):

```sh
sudo semodule -r allow_smartdwarn_udev_curl_mail
```

## Test alerting

You can test your alerts with some parameters in `smartd.conf`. The location of this configuration file may vary:

- AlmaLinux: `/etc/smartmontools/smartd.conf`
- Proxmox VE: `/etc/smartd.conf`

To run the built-in `smartd` alerting self-test, add the `-M test` directive:

```sh
sudo tee /etc/smartmontools/smartd.conf > /dev/null << 'EOT'
DEVICESCAN -H -m @ALL -M test -n standby,10,q
EOT
```

Restart the service to send alerts. In EL & Co., this will be `smartd.service`. In PVE (and likely Debian), this is `smartmontools.service`.

To get real alerts, you can set the `smartd` temperature maximum threshold to 10 degrees C:

```sh
sudo tee /etc/smartmontools/smartd.conf > /dev/null << 'EOT'
DEVICESCAN -H -m @ALL -W 0,5,10 -n standby,10,q
EOT
```

Again, restart the SMART daemon to send alerts.

Here's an example of an email alert:

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

Then, once done testing, set your `smartd.conf` back to normal. Mine is just:

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

Here's a more thorough config:

```sh
sudo tee /etc/smartmontools/smartd.conf > /dev/null << 'EOT'
DEVICESCAN -a -o on -S on \
  -n standby,10,q \
  -s (S/../.././02|L/../../6/03) \
  -m @ALL
EOT
```

Per smartd.conf(5):

```txt
-a    Equivalent to turning on all of the following Directives:
     '-H' to check the SMART health status,
     '-f' to report failures of Usage (rather than Prefail) Attributes,
     '-t' to track changes in both Prefailure and Usage Attributes,
     '-l error' to report increases in the number of ATA errors,
     '-l selftest' to report increases in the number of Self-Test Log errors,
     '-l selfteststs' to report changes of Self-Test execution status,
     '-C 197' to report nonzero values of the current pending sector count,
     and '-U 198' to report nonzero values of the offline pending sector count.

-o VALUE
      [ATA only] Enables or disables SMART Automatic Offline Testing
      when smartd starts up and has no further effect.

      The valid arguments to this Directive are on and off.

      The delay between tests is vendor-specific, but is typically four hours.

-S VALUE
      Enables or disables Attribute Autosave when smartd starts up
      and has no further effect.

      The valid arguments to this Directive are on and off.

      Also affects SCSI devices.

-n POWERMODE[,N][,q]
      [ATA only] This 'nocheck' Directive is used to prevent a disk
      from being spun-up when it is periodically polled by smartd.

-s REGEXP
      Run Self-Tests or Offline Immediate Tests, at scheduled times.
      A Self- or Offline Immediate Test will be run at the end of
      periodic device polling, if all 12 characters of the string
      T/MM/DD/d/HH match the extended regular expression REGEXP. 

-m ADD
      Send a warning email to the email address ADD if the '-H',
      '-l error', '-l xerror', '-l selftest', '-f', '-C', '-U',
      or '-W' Directives detect a failure or a new error, or
      if a SMART command to the disk fails. This Directive only
      works in conjunction with these other Directives
      (or with the equivalent default '-a' Directive).
```

## Hardware RAID

RAID controllers will add complexity. Disks attached to them cannot be detected by smartd with `DEVICESCAN`.

In this case, I have a server with a HPE RAID card (P420i). This hardware RAID card masks info about the disks - the system only sees `/dev/sda`, a logical volume.

The SMART parameters vary by vendor (review the manpage for `smartctl` for details) but I think most modern RAID cards will still allow you to get info about the disks. In this case, you can `sudo smartctl -x -d cciss,$id /dev/sda` to open the `$id`th drive on a controller using the `hpsa`/`cciss` driver.

If you need something generic and scriptable (like DEVICESCAN), this is an easy way to go about things:

```sh
# set defaults and clear file
echo "DEFAULT -o on -S on -a -s (S/../.././02|L/../../6/03) -m @ALL" > smartd.conf
# iterate through disks; 
for i in {0..15}; do
  if sudo smartctl -i -d cciss,$i /dev/sda 2>&1 | grep -q "Serial number"; then
    echo "/dev/sda -d cciss,$i" >> smartd.conf
  fi
done
sudo cp smartd.conf /etc/smartmontools/smartd.conf
```

The end result:

```sh
DEFAULT -o on -S on -a -s (S/../.././02|L/../../6/03) -m @ALL
/dev/sda -d cciss,0
/dev/sda -d cciss,1
/dev/sda -d cciss,2
/dev/sda -d cciss,3
/dev/sda -d cciss,4
/dev/sda -d cciss,5
/dev/sda -d cciss,6
/dev/sda -d cciss,7
```

The "DEFAULT" parameter tells the SMART daemon to apply the same tests to all following drives UOS.

You may be able to mix DEVICESCAN and individual drive test patterns; I haven't had a reason to try this.
