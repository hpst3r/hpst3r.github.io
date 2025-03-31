---
title: "The setuid, setgid, and sticky special file permissions bits"
date: 2025-03-30T12:12:59-00:00
draft: false
---

[Red Hat blog](https://www.redhat.com/en/blog/suid-sgid-sticky-bit)

These are the leading zero!

The setuid and setgid properties are used to allow a trusted executable to be run with elevated permissions (either can do this - as user, or as group respectively) or to control the group assigned to newly created children of a directory (setgid).

The setuid and setgid bits can be set with either `chmod (u|g)+s` or a leading digit, when using three-digit-style permissions (possibly platform-dependent).

The sticky bit restricts file deletion (so that only the owner or root can remove the file).

The values of the leading digit are:
setuid(4)
setgid(2)
sticky(1)

Examples:

```txt
chmod u+s file # setuid
chmod g+s file # setgid
chmod 4770 file # setuid
chmod 2770 file # setgid
chmod 6770 file # setgid and setuid
chmod 1770 file # sticky
```

You can see the special bits in the output of `ls-l`, where it takes the place of the user or group execute "x".

If the file owner or group does not have execute permissions to the file, the (S)pecial bit will be capitalized. If the file owner *does* have execute permissions, it will be lowercase.

In the case of the sticky bit, this will be an upper or lowercase "t".

```txt
[liam@liam-p1g4i-0 ~]$ ls -l
total 357220
-rw-r--r-- 1 liam liam 292546292 Feb 16 11:17 disassembledvmk.asm
-rw-r--r-- 1 liam liam  47809778 Feb 16 11:21 hex
-rw-r--r-- 1 liam liam   6216218 Sep  4  2023 K.B00
-rw-r--r-- 1 liam liam   2737875 Feb 16 11:11 stringsout
-rwxr-xr-x 1 liam liam  16468828 Feb 16 02:46 vmvisor64-vmkernel
[liam@liam-p1g4i-0 ~]$ sudo chgrp root hex
[sudo] password for liam:
[liam@liam-p1g4i-0 ~]$ chmod 7644 hex
[liam@liam-p1g4i-0 ~]$ ls -l
total 357220
-rw-r--r-- 1 liam liam 292546292 Feb 16 11:17 disassembledvmk.asm
-rwSr--r-T 1 liam root  47809778 Feb 16 11:21 hex
-rw-r--r-- 1 liam liam   6216218 Sep  4  2023 K.B00
-rw-r--r-- 1 liam liam   2737875 Feb 16 11:11 stringsout
-rwxr-xr-x 1 liam liam  16468828 Feb 16 02:46 vmvisor64-vmkernel
[liam@liam-p1g4i-0 ~]$ sudo chmod 7644 hex
[liam@liam-p1g4i-0 ~]$ ls -l
total 357220
-rw-r--r-- 1 liam liam 292546292 Feb 16 11:17 disassembledvmk.asm
-rwSr-Sr-T 1 liam root  47809778 Feb 16 11:21 hex
-rw-r--r-- 1 liam liam   6216218 Sep  4  2023 K.B00
-rw-r--r-- 1 liam liam   2737875 Feb 16 11:11 stringsout
-rwxr-xr-x 1 liam liam  16468828 Feb 16 02:46 vmvisor64-vmkernel
[liam@liam-p1g4i-0 ~]$ sudo chmod 0644 hex
[liam@liam-p1g4i-0 ~]$ ls -l
total 357220
-rw-r--r-- 1 liam liam 292546292 Feb 16 11:17 disassembledvmk.asm
-rw-r--r-- 1 liam root  47809778 Feb 16 11:21 hex
-rw-r--r-- 1 liam liam   6216218 Sep  4  2023 K.B00
-rw-r--r-- 1 liam liam   2737875 Feb 16 11:11 stringsout
-rwxr-xr-x 1 liam liam  16468828 Feb 16 02:46 vmvisor64-vmkernel
```

Files will typically be highlighted red if setuid is set or orange if setgid is set. Setuid highlighting takes precedence. See the following image.

{{< figure src="images/specialbits.png" alt="Screenshot of a terminal showing typical suid sgid coloring." >}}

Some examples of common binaries that make use of setuid:

```txt
su
sudo
umount
newgrp
passwd
mount
gpasswd
chage
```