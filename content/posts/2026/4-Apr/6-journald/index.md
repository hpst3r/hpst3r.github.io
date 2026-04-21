---
title: "Working with journald"
date: 2026-04-20T22:15:00-00:00
draft: false
---

Let's briefly dig into `systemd-journald` and go over the what, the why, and the how!

## What is journald?

`systemd-journald` is a logging service bundled with `systemd`. It provides a number of nice extra features past simple plaintext log entries (a la syslog) including, but not limited to:

- journald uses binary storage, with fully indexed data for better (much faster) search
- supports compression... though it's per entry, so it's pretty useless
  - per-entry compression is fine for large individual entries, but Linux log messages are typically quite small and small strings don't compress nearly as well as a large block of text, made up of, say, mostly redundant log entries would.
- structured logging is enforced
- access control, by default
- automatic log rotation

Generally, the output of any systemd unit is sent to the journal.

The main user configuration file is `/etc/systemd/journald.conf`. This may not exist by default on your machine - create it if it's missing; this file just overrides the default configuration at `/usr/lib/systemd/journald.conf`.

> Don't edit files in `/usr/lib/systemd` - they may be overwritten by updates. Same deal with everything else in `/usr`.

By default, on an Enterprise Linux system, the journal is kept mainly in memory (not written to disk for persistence). There's a small buffer at `/run/log/journal`, but the journal is NOT persistent.

Also by default, `systemd-journald` writes to `/var/log` via `rsyslog` with the `imjournal` input plugin - this is a sort of backwards compatibility feature; in the past, `rsyslog` received logs directly and wrote these files out; now it just receives logs from the journal and writes them out.

Files are written out to specific files in `/var/log` depending on which facility they're sent to (e.g., authentication logs are written to `/var/log/secure`). So *some* lines from the journal are written to disk by default; however, this isn't the full journal and, of course, this depends on `rsyslog`.

Other `journald` features of note:

- `journald` supports rate-limiting. By default, it'll drop all messages from a service past `RateLimitBurst` and `RateLimitIntervalSec` (10,000 and 30 seconds). This scales based on available disk.
- Log entries larger than 512 bytes are compressed by default. This is configurable with the `Compress` parameter.

Useful `journalctl` commands:

`journalctl`: show all entries, oldest to newest, paged with `less`

`--disk-usage`: show the current disk usage of the journal. Note that the system and user journals are separate - this is an access control feature.

```txt
[wporter@rhcsa0 ~]$ journalctl --disk-usage
Archived and active journals take up 16M in the file system.
[wporter@rhcsa0 ~]$ sudo journalctl --disk-usage
Archived and active journals take up 56M in the file system.
```

`--vacuum-size=1GB`: immediately clean up the journal, reducing it in size to the configured threshold.

`-n 5`: tail the journal, showing only the last 5 lines (still goes to `less`)

`--no-pager`: output to stdout, without going through `less`

`-f`: follow the journal, equivalent to `tail -f`

`--reverse`, or `-r`: page in reverse order, from newest to oldest

`--no-hostname` will omit the FQDN typically shown at the beginning of a line

`--grep` or `-g` lets you quickly grep the logs with a Perl-compatible regular expression

`-p` lets you select a specific syslog priority (e.g., debug, info, notice, warning, err, alert, crit, alert, emerg)

`--facility` lets you select a specific syslog facility (e.g., 0, kern)

`-x` will provide a brief explanation of supported events

`-u` will allow you to parse the logs for a specific unit

These can, of course, all be combined:

```txt
[wporter@rhcsa0 ~]$ sudo journalctl --facility 1 -p debug -r --no-hostname | head -n 3
Apr 21 00:04:11 qemu-ga[30498]: info: executing fsfreeze hook with arg 'thaw'
Apr 21 00:04:11 qemu-ga[30498]: info: guest-fsthaw called
Apr 21 00:04:10 qemu-ga[30498]: info: executing fsfreeze hook with arg 'freeze'
```

## Writing the journal to disk

First of all, let's figure out [how to write the journal to disk](https://access.redhat.com/solutions/696893).

With `[Journal]` `Storage=auto` in the default `systemd-journald` configuration, all you have to do is create the `/var/log/journal` directory and flush the journal to disk (`journalctl --flush`). That's it! Note that a flush with `Storage=auto` won't create the directory.

```txt
[wporter@rhcsa0 log]$ sudo journalctl --flush
[wporter@rhcsa0 log]$ ls /var/log/journal
ls: cannot access '/var/log/journal': No such file or directory
[wporter@rhcsa0 log]$ sudo mkdir /var/log/journal
[wporter@rhcsa0 log]$ sudo journalctl --flush
[wporter@rhcsa0 log]$ ls /var/log/journal
2abddc0ae628462ab8be021e4c98ab37
```

Alternatively, you can write an override config file at `/etc/systemd/journald.conf` containing:

```ini
[Journal]
Storage=persistent
```

With a restart of `systemd-journald` and a `journalctl --flush`, `systemd-journald` will write the contents of the log to disk, creating `/var/log/journal` if needed.

```txt
[wporter@rhcsa0 log]$ ls /var/log/journal
ls: cannot access '/var/log/journal': No such file or directory
[wporter@rhcsa0 log]$ sudo tee /etc/systemd/journald.conf <<EOT
> [Journal]
> Storage=persistent
> EOT
[Journal]
Storage=persistent
[wporter@rhcsa0 log]$ sudo systemctl restart systemd-journald
[wporter@rhcsa0 log]$ ls /var/log/journal
ls: cannot access '/var/log/journal': No such file or directory
[wporter@rhcsa0 log]$ sudo journalctl --flush
[wporter@rhcsa0 log]$ ls /var/log/journal
2abddc0ae628462ab8be021e4c98ab37
```

## Indexing, search and filtering

So we've got our journal.. how do we use it?

To filter by unit, use the `-u` arg.

### Keys and values

You can also filter by fields by specifying a key and value. The list of keys on your system can be dumped with `journalctl -N`:

```txt
[wporter@rhcsa0 ~]$ journalctl -N
SYSLOG_PID
_SYSTEMD_UNIT
_RUNTIME_SCOPE
TID
_CAP_EFFECTIVE
CODE_LINE
_BOOT_ID
MESSAGE_ID
USER_UNIT
_COMM
CODE_FILE
_SYSTEMD_USER_UNIT
_SYSTEMD_OWNER_UID
USER_INVOCATION_ID
_AUDIT_SESSION
_SELINUX_CONTEXT
_SOURCE_REALTIME_TIMESTAMP
_MACHINE_ID
_EXE
SYSLOG_IDENTIFIER
_CMDLINE
_SYSTEMD_SESSION
_SYSTEMD_USER_SLICE
MESSAGE
_AUDIT_LOGINUID
JOB_ID
JOB_RESULT
_UID
_GID
_HOSTNAME
_SYSTEMD_INVOCATION_ID
SYSLOG_FACILITY
PRIORITY
_PID
CODE_FUNC
USERSPACE_USEC
JOB_TYPE
SYSLOG_TIMESTAMP
_SYSTEMD_SLICE
_SYSTEMD_CGROUP
_TRANSPORT
```

```txt
[wporter@rhcsa0 ~]$ sudo journalctl -F _CMDLINE
/usr/sbin/auditd
/bin/sh -c "whoami > /tmp/cronjob.txt"
logger -p cron notice -t "run-parts[204366]" "(/etc/cron.hourly) starting 0anacron"
/usr/bin/python3 -s /usr/bin/dnf makecache --timer
```

You can also pull logs as JSON with the `--output` (`-o`) argument, and specifying format `json` or `json-pretty`, then query them with `jq`.

Other options include (but are not limited to) `short-iso`, for ISO 8601 timestamps, `export`, a binary format good for storage or transfer, and `cat`, which provides the message without metadata of any kind.

### Filtering by boot

To see your previous boots, run `journalctl --list-boots`:

```txt
[wporter@rhcsa0 ~]$ sudo journalctl --list-boots
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY
 -2 e24064ccc1374080a5a883cf743308b0 Mon 2026-04-20 02:46:52 UTC Tue 2026-04-21 00:29:11 UTC
 -1 e7b2642e353c4a24a47134f64a3e4e2c Tue 2026-04-21 00:29:20 UTC Tue 2026-04-21 00:30:42 UTC
  0 6c8de02393994f959a8d47d9ae7e1023 Tue 2026-04-21 00:31:01 UTC Tue 2026-04-21 00:31:17 UTC
```

To show logs for the current boot, use `journalctl -b`:

```txt
[wporter@rhcsa0 ~]$ sudo journalctl -b | head -n 3
Apr 21 00:31:01 localhost kernel: Linux version 6.12.0-124.49.1.el10_1.x86_64 (mockbuild@x64-builder03.almalinux.org) (gcc (GCC) 14.3.1 20250617 (Red Hat 14.3.1-2), GNU ld version 2.41-58.el10_1.2.alma.1) #1 SMP PREEMPT_DYNAMIC Thu Apr  9 00:52:33 EDT 2026
Apr 21 00:31:01 localhost kernel: Command line: BOOT_IMAGE=(hd0,gpt3)/vmlinuz-6.12.0-124.49.1.el10_1.x86_64 root=UUID=99bf1bf0-97c8-472e-9e64-fc4fee387b2b ro console=tty0 console=ttyS0,115200n8 no_timer_check biosdevname=0 net.ifnames=0 quiet
Apr 21 00:31:01 localhost kernel: BIOS-provided physical RAM map:
```

To go backwards, use either the offset (e.g., `-1` in the output of `--list-boots` above) or the boot ID (by filtering by the key, e.g., `_BOOT_ID=`):

```txt
[wporter@rhcsa0 ~]$ sudo journalctl -b -2 | head -n 3
Apr 20 02:46:52 rhcsa0.lab.wporter.org systemd[1]: serial-getty@ttyS0.service: Deactivated successfully.
Apr 20 02:46:53 rhcsa0.lab.wporter.org systemd-journald[200995]: /var/log/journal/2abddc0ae628462ab8be021e4c98ab37/system.journal: Journal file has been deleted, rotating.
Apr 20 02:46:53 rhcsa0.lab.wporter.org systemd-journald[200995]: Failed to create new system journal: No such file or directory
[wporter@rhcsa0 ~]$ sudo journalctl _BOOT_ID=e24064ccc1374080a5a883cf743308b0 | head -n 3
Apr 20 02:46:52 rhcsa0.lab.wporter.org systemd[1]: serial-getty@ttyS0.service: Deactivated successfully.
Apr 20 02:46:53 rhcsa0.lab.wporter.org systemd-journald[200995]: /var/log/journal/2abddc0ae628462ab8be021e4c98ab37/system.journal: Journal file has been deleted, rotating.
Apr 20 02:46:53 rhcsa0.lab.wporter.org systemd-journald[200995]: Failed to create new system journal: No such file or directory
```

### Filtering by time

To filter the logs for a time interval, you can simply specify the `--since` and/or `--until` parameters. For example, to filter from "now" and tail the logs for future events:

```txt
[wporter@rhcsa0 ~]$ date
Tue Apr 21 12:34:52 AM UTC 2026
[wporter@rhcsa0 ~]$ sudo journalctl --since "now" -f
Apr 21 00:35:04 rhcsa0.lab.wporter.org systemd[1]: serial-getty@ttyS0.service: Deactivated successfully.
Apr 21 00:35:04 rhcsa0.lab.wporter.org systemd[1]: serial-getty@ttyS0.service: Scheduled restart job, restart counter is at 23.
Apr 21 00:35:04 rhcsa0.lab.wporter.org systemd[1]: Started serial-getty@ttyS0.service - Serial Getty on ttyS0.
Apr 21 00:35:04 rhcsa0.lab.wporter.org agetty[1291]: could not get terminal name: -22
Apr 21 00:35:04 rhcsa0.lab.wporter.org agetty[1291]: -: failed to get terminal attributes: Input/output error
```

We can clearly see that my `agetty` service is currently having a bad time!

To filter for, say, a five-minute block on 4/20/26 at 23:00 - 23:05 UTC, we'd say `journalctl --since "2026-04-20 23:00:00" --until "2026-04-20 23:05:00"`. Note that the only date format supported is `YYYY-MM-DD HH:MM:SS`.

```txt
[wporter@rhcsa0 ~]$ sudo journalctl --since "2026-04-20 23:00:00" --until "2026-04-20 23:05:00" --no-hostname | head -n 5
Apr 20 23:00:01 systemd[1]: Created slice user-1000.slice - User Slice of UID 1000.
Apr 20 23:00:01 systemd[1]: Starting user-runtime-dir@1000.service - User Runtime Directory /run/user/1000...
Apr 20 23:00:02 systemd[1]: Finished user-runtime-dir@1000.service - User Runtime Directory /run/user/1000.
Apr 20 23:00:02 systemd[1]: Starting user@1000.service - User Manager for UID 1000...
Apr 20 23:00:02 systemd-logind[740]: New session 1466 of user wporter.
```

There are a few other friendly time values you can pass, including but not limited to:

- 15min ago
- now
- today
- today UTC
- yesterday
- tomorrow
- -30s
- @$unix_timestamp

### Filtering with `--grep`

To grep the logs, simply pass `-g` and specify a regular expression. For example, to match `sudo` with grep:

```txt
[wporter@rhcsa0 ~]$ sudo journalctl -g "sudo" --no-hostname -r | head -n 3
Apr 21 00:39:16 sudo[1381]: pam_unix(sudo:session): session opened for user root(uid=0) by wporter(uid=1000)
Apr 21 00:39:16 sudo[1381]:  wporter : TTY=pts/0 ; PWD=/home/wporter ; USER=root ; COMMAND=/bin/journalctl -g sudo --no-hostname -r
Apr 21 00:39:05 sudo[1374]: pam_unix(sudo:session): session closed for user root
```

To match only lines ending with "session closed for user root" (in this case we'll just grab the three most recent):

```txt
[wporter@rhcsa0 ~]$ sudo journalctl -g "session closed for user root$" --no-hostname -r | head -n 3
Apr 21 00:40:15 sudo[1403]: pam_unix(sudo:session): session closed for user root
Apr 21 00:39:16 sudo[1381]: pam_unix(sudo:session): session closed for user root
Apr 21 00:39:05 sudo[1374]: pam_unix(sudo:session): session closed for user root
```

### Filtering by unit

To filter by unit, pass the `-u` argument and specify the unit name. For example, to search for journald's logs:

```txt
[wporter@rhcsa0 ~]$ sudo journalctl -u systemd-journald -r --no-hostname | head -n 3
Apr 21 00:31:05 systemd-journald[581]: Received client request to flush runtime journal.
Apr 21 00:31:04 systemd-journald[581]: System Journal (/var/log/journal/2abddc0ae628462ab8be021e4c98ab37) is 56M, max 894.9M, 838.8M free.
Apr 21 00:31:04 systemd-journald[581]: Time spent on flushing to /var/log/journal/2abddc0ae628462ab8be021e4c98ab37 is 313.994ms for 1295 entries.
```

To search for the logs for the `pveproxy` service:

```txt
wporter@z2g4a:~$ sudo journalctl -u pveproxy -r --no-hostname | head -n 3
Apr 20 17:34:39 pveproxy[1220517]: worker exit
Apr 20 17:34:25 pveproxy[3100]: worker 1295349 started
Apr 20 17:34:25 pveproxy[3100]: starting 1 worker(s)
```

To search the logs for the three oldest log entries involving the `prometheus-node-exporter-smartmon` timer:

```txt
wporter@z2g4a:~$ sudo journalctl -u prometheus-node-exporter-smartmon.timer | head -n 3
Mar 31 12:42:32 z2g4a systemd[1]: prometheus-node-exporter-smartmon.timer: Deactivated successfully.
Mar 31 12:42:32 z2g4a systemd[1]: Stopped prometheus-node-exporter-smartmon.timer - Run smart metrics collection every 15 minutes.
-- Boot 2cc391ca20304da3bac955f52f2233d9 --
```

### Structured logging

Earlier, at the very beginning of this note, I said that systemd-journald "enforces structured logging". What does that mean, and why do we care?

Unstructured logs are text, typically made up of individual strings. They're good for people to read, but difficult to parse and search. You get into text parsing hell if you have to parse a bunch of unstructured log lines; it's costly (in terms of compute) to search, extremely costly to index, and a pain in the butt to deal with if you need to consistently extract a specific "field" from a bunch of different messages.

Syslog messages from different vendors or different products might not necessarily follow the same exact format, so they can be considered semi-structured.

Another example of semi-structured logs are Windows event logs - some fields are easy to retrieve, but you're in text-parsing hell for other information.

Structured logs typically take the format of something like JSON - fields are clearly labeled, often with defined data types, and can be easily retrieved by a machine. For example, to access the date field in a structured JSON log, it's trivial to grab Object.Date in a programming language.

This also makes it very easy to send logs to different systems - you get a consistent log from one and can consistently convert it for the other. journald is a structured system, with standard, defined key/value pairs in log entries for things like the log message, timestamp, UID, facility, process, hostname.. etc (think back to the `journalctl -N` list of keys).

In any enterprise with more than a single machine, you'll probably be making use of log aggregation, automatic parsing, and alerting of some kind. This makes structured logging preferable in many folks' opinions (including mine).

A downside is increased storage usage, and structured logs are typically more difficult for humans to read through.. think of getting a big ol' pile of JSON as a response.

### Access control

`systemd-journald` offers access control and a sort of multi-tenancy. Users can have their own journals that are separate from the system journals and can be used or managed without elevation.

### Log rotation

To prevent the journal from becoming unmanageably large, you can explicitly configure how much space it uses by tweaking some parameters in your `/etc/systemd/journald.conf` override, or the drop-ins:

> We'll specifically be talking about the parameters that control the size and aging of log files on disk, but you should know that there are RuntimeMaxUse and RuntimeKeepFree parameters to control the amount of memory used by a volatile journal, too.

- SystemMaxUse
  - the maximum amount of space the journals can occupy, total
  - default is up to 10% of the filesystem or 4G, whichever is smaller
- SystemKeepFree
  - the minimum amount of space to keep free on the filesystem
  - default is 15% of the filesystem or 4G, whichever is smaller
- SystemMaxFileSize
  - the maximum size of an individual journal file, after which rotation is triggered
  - default is 1/8 of the values configured for SystemMaxUse, or 128M, whichever is smaller
- SystemMaxFiles
  - the maximum **count** of (archived) journal files on disk
  - default is 100
- MaxFileSec
  - the maximum duration collected in a single journal file before rotation
  - default is 1 month
- MaxRetentionSec
  - the maximum duration of logs kept on disk
  - default is infinite, intended to be controlled by MaxUse and KeepFree

You can also manually (`journalctl --vacuum-size=5G`, `--vacuum-time=3days`, or `--vacuum-files=5`) vacuum up archived journal entries to hit a space or time target. This only ever applies to inactive journals (files that have already been rotated out).

Rotation is triggered automatically when a journal file reaches the configured SystemMaxFileSize, when the configured MaxFileSec elapses for the active journal, or when `journald` restarts.

Immediately after rotation, `journald` checks whether total usage exceeds SystemMaxUse or SystemKeepFree and vacuums old rotated files until it's back within bounds.

An example drop-in configuration to control the size of the journal as a whole and its individual files through both aging and a MaxUse parameter could look something like this:

```ini
[Journal]
Storage=persistent
SystemMaxUse=1G
SystemKeepFree=512M
SystemMaxFileSize=128M
MaxFileSec=1month
MaxRetentionSec=1year
```

### Quick bonus: `systemd-analyze cat-config`

You can quickly view the full effective journald config including any drop-ins, overrides, and defaults with `systemd-analyze cat-config`:

```bash
systemd-analyze cat-config systemd/journald.conf
```

```txt
[wporter@rhcsa0 ~]$ systemd-analyze cat-config systemd/journald.conf
# /usr/lib/systemd/journald.conf
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it under the
#  terms of the GNU Lesser General Public License as published by the Free
#  Software Foundation; either version 2.1 of the License, or (at your option)
#  any later version.
#
# Entries in this file show the compile time defaults. Local configuration
# should be created by either modifying this file (or a copy of it placed in
# /etc/ if the original file is shipped in /usr/), or by creating "drop-ins" in
# the /etc/systemd/journald.conf.d/ directory. The latter is generally
# recommended. Defaults can be restored by simply deleting the main
# configuration file and all drop-ins located in /etc/.
#
# Use 'systemd-analyze cat-config systemd/journald.conf' to display the full config.
#
# See journald.conf(5) for details.

[Journal]
#Storage=auto
#Compress=yes
#Seal=yes
#SplitMode=uid
#SyncIntervalSec=5m
#RateLimitIntervalSec=30s
#RateLimitBurst=10000
#SystemMaxUse=
#SystemKeepFree=
#SystemMaxFileSize=
#SystemMaxFiles=100
#RuntimeMaxUse=
#RuntimeKeepFree=
#RuntimeMaxFileSize=
#RuntimeMaxFiles=100
#MaxRetentionSec=0
#MaxFileSec=1month
#ForwardToSyslog=no
#ForwardToKMsg=no
#ForwardToConsole=no
#ForwardToWall=yes
#TTYPath=/dev/console
#MaxLevelStore=debug
#MaxLevelSyslog=debug
#MaxLevelKMsg=notice
#MaxLevelConsole=info
#MaxLevelWall=emerg
#MaxLevelSocket=debug
#LineMax=48K
#ReadKMsg=yes
Audit=
```

### Quick bonus: `logger`

You can use the `logger` utility to easily write something to the journal from a script or shell session. For example:

```txt
[wporter@rhcsa0 log]$ logger -p authpriv.crit "hello from logger!"
[wporter@rhcsa0 log]$ journalctl | tail -n 1
Apr 20 02:16:18 rhcsa0.lab.wporter.org wporter[200373]: hello from logger!
[wporter@rhcsa0 log]$ sudo cat /var/log/secure | tail -n 1
Apr 20 02:16:18 rhcsa0 wporter[200392]: hello from logger!
```
