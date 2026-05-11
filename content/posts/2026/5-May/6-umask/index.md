---
title: "Setting default file permissions with umask"
date: 2026-05-09T22:30:00-00:00
draft: false
---

`umask` is a shell command that sets the value that controls which file permissions are set by default.

The OS starts with the set of maximum allowed permissions (`777` for directories, `666` for files, as the execute bit is required to traverse directories, but is only required to execute files - something you probably don't want by default) and the umask bitmask is applied to remove permissions.

For example:

The default `umask` is 022, which means files are created with mode 644 by default (and directories are created with mode 755).

```sh
$ umask
0022

$ touch example
$ ls -l example
-rw-r--r--. 1 wporter wporter 0 May 10 12:56 example

$ mkdir exampledir
$ ls -lZ | grep exampledir
drwxr-xr-x. 2 wporter wporter unconfined_u:object_r:user_home_t:s0               6 May 10 12:56 exampledir
```

To create new files read-only for the owner and group, you might apply a `umask` of 227:

```sh
$ umask 0227
$ touch example
$ mkdir exampledir

$ ls -lZ | grep "example$\|exampledir$"
-r--r-----. 1 wporter wporter unconfined_u:object_r:user_home_t:s0               0 May 10 13:10 example
dr-xr-x---. 2 wporter wporter unconfined_u:object_r:user_home_t:s0               6 May 10 13:10 exampledir
```

If you elevate with `sudo`, your umask settings will carry over. However, if you switch users with `su`, they will not.

```sh
[wporter@rhcsa0 ~]$ umask 227
[wporter@rhcsa0 ~]$ su
Password:
[root@rhcsa0 wporter]# umask
0022
[root@rhcsa0 wporter]# exit
exit
[wporter@rhcsa0 ~]$ umask
0227
[wporter@rhcsa0 ~]$ sudo su
[root@rhcsa0 wporter]# umask
0022
```

This umask setting removes the write bit (2) for the owner and group, and removes all of the read, write, and execute bits (4 + 2 + 1) for everyone else.

To make a umask change persistent, you can add it to your bashrc:

```sh
sudo tee -a ~/.bashrc << EOT
umask 022
EOT
```

The umask is a shell feature, and there's no easy single toggle to set it everywhere for every shell session and user.

The closest you can get is adding a drop-in to the global profile.d (e.g., `/etc/profile.d/umask.sh` containing `umask 022`) to set the umask for any login shell. For example:

```sh
sudo tee /etc/profile.d/umask.sh << EOT
umask 022
EOT
```
