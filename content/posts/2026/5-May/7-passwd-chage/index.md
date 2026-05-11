---
title: "Managing password aging with /etc/login.defs, passwd, and chage"
date: 2026-05-11T19:15:00-00:00
draft: false
---

We can use the `/etc/login.defs` file and `chage` command to manage password aging for users on an EL system.

The `/etc/login.defs` controls defaults for new users - namely, the `PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_MIN_LEN`, and `PASS_WARN_AGE` parameters:

```ini
# Password aging controls:
#
#	PASS_MAX_DAYS	Maximum number of days a password may be used.
#	PASS_MIN_DAYS	Minimum number of days allowed between password changes.
#	PASS_MIN_LEN	Minimum acceptable password length.
#	PASS_WARN_AGE	Number of days warning given before a password expires.
#
PASS_MAX_DAYS	99999
PASS_MIN_DAYS	0
PASS_WARN_AGE	7

# this is in here too, but is generally ignored & controlled by PAM
PASS_MIN_LEN	8
```

These values are added to the user's `/etc/shadow` entry at creation time.

For reference, an `/etc/shadow` entry is structured like:

```txt
user:$id$salt$hash:lastchange:mindaystochange:maxdays:warn:inactive:expiration:
```

An entry with the default password policy might look something like:

```txt
wporter:$y$j9T$TJCRyqICjrcUUHxUnsr4o0$NkiiA0S8BRIsoELX4uTVR/cToYdG8kcrBGYbAx7x730:20583:0:99999:7:::
```

For existing local users, you must use `chage`, `passwd`, or your editor (via `vipw` for proper locking a la `visudo`) to modify password expiration timers per user.

With `chage`, you can use the `-m` (`--mindays`), `-M` (`--maxdays`), and `-W` (`--warndays`) parameters to modify existing users' password aging. For example:

```sh
$ sudo cat /etc/shadow | grep ^user
user:$y$j9T$Jyz5XePjyGfiyeCSNSxMP/$/3fJUg.SQ3XHznBtHqllwY03ZkU4ujSHYiH7YC2hLB3:20577:0:99999:7:::
$ sudo chage user --mindays 2 --maxdays 10 --warndays 5
$ sudo cat /etc/shadow | grep ^user
user:!$y$j9T$Jyz5XePjyGfiyeCSNSxMP/$/3fJUg.SQ3XHznBtHqllwY03ZkU4ujSHYiH7YC2hLB3:20577:2:10:5:::
```

With `passwd`, you can use the `-n` (`--mindays`), `-x` (`--maxdays`) and `-w` (`--warndays`) parameters:

```sh
$ sudo cat /etc/shadow | grep ^user
user:!$y$j9T$Jyz5XePjyGfiyeCSNSxMP/$/3fJUg.SQ3XHznBtHqllwY03ZkU4ujSHYiH7YC2hLB3:20577:2:10:5:::
$ sudo passwd user --mindays 0 --maxdays 99999 --warndays 7
$ sudo cat /etc/shadow | grep ^user
user:!$y$j9T$Jyz5XePjyGfiyeCSNSxMP/$/3fJUg.SQ3XHznBtHqllwY03ZkU4ujSHYiH7YC2hLB3:20577:0:99999:7:::
```

Any password event with `chage` or `passwd` is written to the secure log. Note that any adjustment of `/etc/shadow` with `passwd` is listed as a password change.

```sh
$ sudo cat /var/log/secure | grep password
May 10 18:08:54 rhcsa0 passwd[10681]: pam_unix(passwd:chauthtok): password changed for wporter
May 11 12:19:27 rhcsa0 passwd[34231]: password for 'user' changed by 'root'
May 11 15:45:46 rhcsa0 chage[37589]: changed password expiry for user
May 11 15:45:53 rhcsa0 chage[37597]: changed password expiry for user
May 11 15:45:58 rhcsa0 chage[37607]: changed password expiry for user
May 11 15:46:18 rhcsa0 chage[37628]: changed password expiry for user
May 11 15:51:46 rhcsa0 passwd[37732]: password for 'user' changed by 'root'
```

Additionally, the `chage` command may be used to view the status of a user. This doesn't require elevation (for the current user only).

```sh
$ chage -l wporter
Last password change					: Apr 09, 2026
Password expires					: never
Password inactive					: never
Account expires						: never
Minimum number of days between password change		: 0
Maximum number of days between password change		: 99999
Number of days of warning before password expires	: 7
```

You can disable a user's password with the `passwd -l` (lock) command - this prepends an exclamation mark to the password hash in the `/etc/shadow` file. If you then need to unlock the account's password, run `passwd -u` or remove the leading exclamation point.

```sh
$ sudo cat /etc/shadow | grep ^user
user:$y$j9T$Jyz5XePjyGfiyeCSNSxMP/$/3fJUg.SQ3XHznBtHqllwY03ZkU4ujSHYiH7YC2hLB3:20577:0:99999:7:::

$ sudo passwd -l user
passwd: password changed.

$ sudo cat /etc/shadow | grep ^user
user:!$y$j9T$Jyz5XePjyGfiyeCSNSxMP/$/3fJUg.SQ3XHznBtHqllwY03ZkU4ujSHYiH7YC2hLB3:20577:0:99999:7:::
```

However, login with other authentication methods (e.g., SSH keys) is still permitted.

To truly disable an account, you can use `chage -E 0` or `usermod --expiredate 1`. These set the account's expiration date to Jan 1, 1970, and Jan 2, 1970, respectively, immediately and globally preventing logon.

To prevent interactive logon, but keep the account enabled (e.g., for `su` or service use) you can change the account's shell to `/sbin/nologin` with `usermod -s` or `chsh`.
