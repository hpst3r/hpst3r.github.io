---
title: "Links and inodes!"
date: 2026-04-19T19:00:00-00:00
draft: false
---

Let's talk about links!

## inodes

But first, we'll need to go over the concept of **inodes**. These are an essential data structure Unix filesystems that store the attributes (metadata, permissions) and block locations for a particular file or directory.

These "index" nodes allow the kernel to find the file contents from the inode number - conceptually, think of reading memory from a memory address. An inode contains a "disk address" in an easily parsable data structure, with metadata information slapped on. The kernel can read the inode and quickly figure out where to go to read the contents of a file off the disk.

> inodes were traditionally stored in fixed-size areas on disk set up at filesystem creation time, so the maximum *number* of inodes (and thus files) was set when the filesystem was created.
>
> Newer filesystems like XFS (though XFS does restrict the number of inodes to a percentage of free space), ZFS, and btrfs allow a dynamic number of inodes to exist on the system (e.g., the inode table, or in these cases, the inode B-trees, aren't stored in a fixed location on disk) - resolving the inode exhaustion problem. However, if working with an ext4 volume, be aware of your inodes!

Now, why do we need inodes?

Couple notable reasons:

- allows you to quickly look up metadata (e.g., file names, permissions, file size when you `ls -la`) without touching data blocks (more I/O would be required) by decoupling metadata from the data.
- seek optimization - filesystem can read inodes, find where the extents it needs to read are located, and intelligently batch I/O to make the most of spinning rust.
- inode data structures enable efficient scanning - you no longer need to traverse directory trees (random I/O all over the place) to find files, e.g., with `find`
- easier to cache inode data structures in memory - they're typically quite compact and this helps further reduce disk I/O. See the kernel VFS inode cache.

So, an inode is a pointer to a specific file on disk, but an inode itself is not the file. What does this mean for our links?

Well, what are the two *types* of links we're dealing with?

## symlinks - soft links

First, let's go over symlinks (or "soft" links).

These are essentially an alias to a filesystem path. Soft links are an obvious "pointer" to another filesystem path, rather than an address on disk. They always have their own inodes and may have their own extents on disk (though this is ONLY to contain the path to the target).

For example, if we were to create a symlink `/home/user/bash`, pointed at `/bin/bash`:

```txt
[wporter@rhcsa0 ~]$ ln -s /bin/bash /home/wporter/bash
```

We can clearly tell that this is a link, or a sort of shortcut (but don't conflate a symlink with a Windows shortcut) pointing to `/bin/bash` if we inspect it with `ls -l`:

```txt
[wporter@rhcsa0 ~]$ ls -l bash
lrwxrwxrwx. 1 wporter wporter 9 Apr 19 21:37 bash -> /bin/bash
```

The operating system and filesystem can clearly tell that this is a distinct file from `/bin/bash` that's simply pointing you to the target. And note that `/bin/bash`'s link count is still 1:

```txt
[wporter@rhcsa0 ~]$ ls -l /bin/bash
-rwxr-xr-x. 1 root root 1414608 Oct 29  2024 /bin/bash
```

While there's no particularly easy way to view the contents of the symlink itself (as a symlink is essentially a function of the filesystem API), we can review the implementation of symlinks in [XFS](https://github.com/torvalds/linux/blob/master/fs/xfs/xfs_symlink.c) and [ext](https://www.kernel.org/doc/html/latest/filesystems/ext4/ifork.html) to assert that, typically, a symlink is JUST an inode and doesn't use 'data' extents on disk (assuming the path can fit in the inode itself). However, the implementation of inodes differs per filesystem, so this varies. For example:

- ext filesystems (2 - 4) store the pathname for a symlink in the `i_block` field of the inode if it can fit.
  - This field consists of 15 4-byte integers = up to 60 characters.
  - This is called a "fast symlink".
  - Otherwise, the inode is used as a pointer to an extent on disk (as with any other file) and the path is written to disk at the other end of the pointer.
  - The maximum length of a symlink is 4096 bytes on an ext filesystem.
- XFS filesystems store the pathname for a symlink either "locally", in the inode's data fork (the space after the inode core, which varies) or in extents on disk.
  - the XFS inode core is a fixed 176 bytes, and the default size of an inode in RHEL is 512 bytes, giving you a whopping 336 bytes in this scenario. However, the size of an inode is configurable.
  - If the path doesn't fit in the inode's data fork, it's put in an extent on disk (with a pointer to the extent in an inode).
  - The maximum length of a symlink is 1024 bytes on an XFS filesystem.

If we want to look at the inode number for our symlink and the `/bin/bash` file:

```txt
[wporter@rhcsa0 ~]$ ls -i bash
174267 bash
[wporter@rhcsa0 ~]$ ls -i /bin/bash
16909362 /bin/bash
```

We can examine these inodes further wtih `xfs_db`. First, let's look at the symlink's inode.

```txt
[wporter@rhcsa0 ~]$ sudo xfs_db -r /dev/sda4
xfs_db> inode 174267
xfs_db> p
core.magic = 0x494e
core.mode = 0120777
core.version = 3
core.format = 1 (local)
core.onlink = 0
core.uid = 1000
core.gid = 1000
core.nlinkv2 = 1
core.projid_lo = 0
core.projid_hi = 0
core.nextents = 0
core.atime.sec = Sun Apr 19 21:37:54 2026
core.atime.nsec = 581534419
core.mtime.sec = Sun Apr 19 21:37:52 2026
core.mtime.nsec = 265523634
core.ctime.sec = Sun Apr 19 21:37:52 2026
core.ctime.nsec = 265523634
core.size = 9
core.nblocks = 0
core.extsize = 0
core.naextents = 0
core.forkoff = 35
core.aformat = 1 (local)
core.dmevmask = 0
core.dmstate = 0
core.newrtbm = 0
core.prealloc = 0
core.realtime = 0
core.immutable = 0
core.append = 0
core.sync = 0
core.noatime = 0
core.nodump = 0
core.rtinherit = 0
core.projinherit = 0
core.nosymlinks = 0
core.extsz = 0
core.extszinherit = 0
core.nodefrag = 0
core.filestream = 0
core.gen = 3743657658
next_unlinked = null
v3.crc = 0xf338f4c2 (correct)
v3.change_count = 4
v3.lsn = 0x70000827d
v3.flags2 = 0x18
v3.cowextsize = 0
v3.crtime.sec = Sun Apr 19 21:37:52 2026
v3.crtime.nsec = 265523634
v3.inumber = 174267
v3.uuid = 99bf1bf0-97c8-472e-9e64-fc4fee387b2b
v3.reflink = 0
v3.cowextsz = 0
v3.dax = 0
v3.bigtime = 1
v3.nrext64 = 1
u3.symlink = "/bin/bash"
a.sfattr.hdr.totsize = 51
a.sfattr.hdr.count = 1
a.sfattr.list[0].namelen = 7
a.sfattr.list[0].valuelen = 37
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 1
a.sfattr.list[0].parent = 0
a.sfattr.list[0].name = "selinux"
a.sfattr.list[0].value = "unconfined_u:object_r:user_home_t:s0\000"
```

Note how the `core.format` of the symlink (`./bash`), inode 174267, is "local". This inode doesn't contain a pointer to an extent on disk (note how `core.nextents`, the number of extents is zero, and we don't have a `u3.bmx` array containing the location on disk), because it's a short symlink. Additionally, you can find the symlink target stored right in the inode: `u3.symlink = "/bin/bash"`.

Then, we examine inode 16909362 (`/bin/bash`):

```txt
xfs_db> inode 16909362
xfs_db> p
core.magic = 0x494e
core.mode = 0100755
core.version = 3
core.format = 2 (extents)
core.onlink = 0
core.uid = 0
core.gid = 0
core.nlinkv2 = 1
core.projid_lo = 0
core.projid_hi = 0
core.nextents = 1
core.atime.sec = Sun Apr 19 22:01:01 2026
core.atime.nsec = 630024245
core.mtime.sec = Tue Oct 29 00:00:00 2024
core.mtime.nsec = 0
core.ctime.sec = Tue Nov 25 09:08:59 2025
core.ctime.nsec = 77086981
core.size = 1414608
core.nblocks = 346
core.extsize = 0
core.naextents = 0
core.forkoff = 24
core.aformat = 1 (local)
core.dmevmask = 0
core.dmstate = 0
core.newrtbm = 0
core.prealloc = 0
core.realtime = 0
core.immutable = 0
core.append = 0
core.sync = 0
core.noatime = 0
core.nodump = 0
core.rtinherit = 0
core.projinherit = 0
core.nosymlinks = 0
core.extsz = 0
core.extszinherit = 0
core.nodefrag = 0
core.filestream = 0
core.gen = 3925614409
next_unlinked = null
v3.crc = 0x86185dfe (correct)
v3.change_count = 15
v3.lsn = 0x7000087f5
v3.flags2 = 0x1a
v3.cowextsize = 0
v3.crtime.sec = Tue Nov 25 09:08:59 2025
v3.crtime.nsec = 63086678
v3.inumber = 16909362
v3.uuid = 99bf1bf0-97c8-472e-9e64-fc4fee387b2b
v3.reflink = 1
v3.cowextsz = 0
v3.dax = 0
v3.bigtime = 1
v3.nrext64 = 1
u3.bmx[0] = [startoff,startblock,blockcount,extentflag]
0:[0,2116417,346,0]
a.sfattr.hdr.totsize = 48
a.sfattr.hdr.count = 1
a.sfattr.list[0].namelen = 7
a.sfattr.list[0].valuelen = 34
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 1
a.sfattr.list[0].parent = 0
a.sfattr.list[0].name = "selinux"
a.sfattr.list[0].value = "system_u:object_r:shell_exec_t:s0\000"
```

Note how its type is "extents", it shows `core.nextents = 1`, and it *does* have information about the startblock and blockcount:

```txt
u3.bmx[0] = [startoff,startblock,blockcount,extentflag]
0:[0,2116417,346,0]
```

## Hard links

Hard links, as opposed to soft links, simply provide another directory entry pointing to the same inode, same block address of some data on disk. There is no distinction between the "original" file created when you actually created a file, and a hard link you manually created later.

For example, let's create and examine a hardlink to `/bin/bash`. Immediately, we'll notice that a hard link requires write and execute on the target.

```txt
[wporter@rhcsa0 ~]$ ln /bin/bash hardbash
ln: failed to create hard link 'hardbash' => '/bin/bash': Operation not permitted
```

> These permissions are a security feature, namely `fs.protected_hardlinks` - this was intended to protect against time-of-check/time-of-use attacks and unprivileged users pinning (potentially vulnerable) setuid binaries for later exploitation (since even if the old binary is removed from its "proper" location, a remaining hard link means the file is still there and perfectly usable!)

So I'll have to elevate to create this link to `/bin`:

```txt
[wporter@rhcsa0 ~]$ sudo ln /bin/bash hardbash
```

If we attempt to examine the `hardbash` link, we'll see the same filesize and inode, but note that the link count is 2:

```txt
[wporter@rhcsa0 ~]$ ls -l hardbash
-rwxr-xr-x. 2 root root 1414608 Oct 29  2024 hardbash
[wporter@rhcsa0 ~]$ ls -i hardbash
16909362 hardbash
```

Also, note that filesystem permissions are shared between the hardlink and the original file.

If we examine the inode, we can see it's the same darn thing we looked at when we looked at `/bin/bash` itself. The only thing that tells us anything is different is the `core.nlinkv2` counter that now shows we have two links.

```txt
[wporter@rhcsa0 ~]$ sudo xfs_db -r /dev/sda4
xfs_db> inode 16909362
xfs_db> p
core.magic = 0x494e
core.mode = 0100755
core.version = 3
core.format = 2 (extents)
core.onlink = 0
core.uid = 0
core.gid = 0
core.nlinkv2 = 2
core.projid_lo = 0
core.projid_hi = 0
core.nextents = 1
core.atime.sec = Sun Apr 19 22:26:02 2026
core.atime.nsec = 12045678
core.mtime.sec = Tue Oct 29 00:00:00 2024
core.mtime.nsec = 0
core.ctime.sec = Sun Apr 19 22:25:26 2026
core.ctime.nsec = 343871406
core.size = 1414608
core.nblocks = 346
core.extsize = 0
core.naextents = 0
core.forkoff = 24
core.aformat = 1 (local)
core.dmevmask = 0
core.dmstate = 0
core.newrtbm = 0
core.prealloc = 0
core.realtime = 0
core.immutable = 0
core.append = 0
core.sync = 0
core.noatime = 0
core.nodump = 0
core.rtinherit = 0
core.projinherit = 0
core.nosymlinks = 0
core.extsz = 0
core.extszinherit = 0
core.nodefrag = 0
core.filestream = 0
core.gen = 3925614409
next_unlinked = null
v3.crc = 0x994672a7 (correct)
v3.change_count = 16
v3.lsn = 0x700008d7d
v3.flags2 = 0x1a
v3.cowextsize = 0
v3.crtime.sec = Tue Nov 25 09:08:59 2025
v3.crtime.nsec = 63086678
v3.inumber = 16909362
v3.uuid = 99bf1bf0-97c8-472e-9e64-fc4fee387b2b
v3.reflink = 1
v3.cowextsz = 0
v3.dax = 0
v3.bigtime = 1
v3.nrext64 = 1
u3.bmx[0] = [startoff,startblock,blockcount,extentflag]
0:[0,2116417,346,0]
a.sfattr.hdr.totsize = 48
a.sfattr.hdr.count = 1
a.sfattr.list[0].namelen = 7
a.sfattr.list[0].valuelen = 34
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 1
a.sfattr.list[0].parent = 0
a.sfattr.list[0].name = "selinux"
a.sfattr.list[0].value = "system_u:object_r:shell_exec_t:s0\000"
```

So... next question. Where are these links stored? Well, XFS stores its directory metadata (e.g., file names) in directory inodes' data forks (where our symlinks go). Let's have a peek at what the files in my home directory actually are. Pay special attention to the `u3.sfdir3` properties.

```txt
[wporter@rhcsa0 ~]$ ls -id /home/wporter
172085 /home/wporter
[wporter@rhcsa0 ~]$ sudo xfs_db -r /dev/sda4
xfs_db> inode 172085
xfs_db> p
core.magic = 0x494e
core.mode = 040700
core.version = 3
core.format = 1 (local)
core.onlink = 0
core.uid = 1000
core.gid = 1000
core.nlinkv2 = 3
core.projid_lo = 0
core.projid_hi = 0
core.nextents = 0
core.atime.sec = Sun Apr 19 22:33:23 2026
core.atime.nsec = 674203632
core.mtime.sec = Sun Apr 19 22:31:11 2026
core.mtime.nsec = 303556870
core.ctime.sec = Sun Apr 19 22:31:11 2026
core.ctime.nsec = 303556870
core.size = 156
core.nblocks = 0
core.extsize = 0
core.naextents = 0
core.forkoff = 24
core.aformat = 1 (local)
core.dmevmask = 0
core.dmstate = 0
core.newrtbm = 0
core.prealloc = 0
core.realtime = 0
core.immutable = 0
core.append = 0
core.sync = 0
core.noatime = 0
core.nodump = 0
core.rtinherit = 0
core.projinherit = 0
core.nosymlinks = 0
core.extsz = 0
core.extszinherit = 0
core.nodefrag = 0
core.filestream = 0
core.gen = 4076948918
next_unlinked = null
v3.crc = 0x14d4747 (correct)
v3.change_count = 40
v3.lsn = 0x700008f53
v3.flags2 = 0x18
v3.cowextsize = 0
v3.crtime.sec = Thu Apr  9 21:29:25 2026
v3.crtime.nsec = 296006145
v3.inumber = 172085
v3.uuid = 99bf1bf0-97c8-472e-9e64-fc4fee387b2b
v3.reflink = 0
v3.cowextsz = 0
v3.dax = 0
v3.bigtime = 1
v3.nrext64 = 1
u3.sfdir3.hdr.count = 9
u3.sfdir3.hdr.i8count = 0
u3.sfdir3.hdr.parent.i4 = 8389767
u3.sfdir3.list[0].namelen = 12
u3.sfdir3.list[0].offset = 0x60
u3.sfdir3.list[0].name = ".bash_logout"
u3.sfdir3.list[0].inumber.i4 = 172086
u3.sfdir3.list[0].filetype = 1
u3.sfdir3.list[1].namelen = 13
u3.sfdir3.list[1].offset = 0x78
u3.sfdir3.list[1].name = ".bash_profile"
u3.sfdir3.list[1].inumber.i4 = 172087
u3.sfdir3.list[1].filetype = 1
u3.sfdir3.list[2].namelen = 7
u3.sfdir3.list[2].offset = 0x98
u3.sfdir3.list[2].name = ".bashrc"
u3.sfdir3.list[2].inumber.i4 = 172088
u3.sfdir3.list[2].filetype = 1
u3.sfdir3.list[3].namelen = 4
u3.sfdir3.list[3].offset = 0xb0
u3.sfdir3.list[3].name = ".ssh"
u3.sfdir3.list[3].inumber.i4 = 8676089
u3.sfdir3.list[3].filetype = 2
u3.sfdir3.list[4].namelen = 13
u3.sfdir3.list[4].offset = 0xc0
u3.sfdir3.list[4].name = ".bash_history"
u3.sfdir3.list[4].inumber.i4 = 174093
u3.sfdir3.list[4].filetype = 1
u3.sfdir3.list[5].namelen = 8
u3.sfdir3.list[5].offset = 0xf8
u3.sfdir3.list[5].name = ".lesshst"
u3.sfdir3.list[5].inumber.i4 = 505217
u3.sfdir3.list[5].filetype = 1
u3.sfdir3.list[6].namelen = 9
u3.sfdir3.list[6].offset = 0x130
u3.sfdir3.list[6].name = "email.txt"
u3.sfdir3.list[6].inumber.i4 = 174268
u3.sfdir3.list[6].filetype = 1
u3.sfdir3.list[7].namelen = 4
u3.sfdir3.list[7].offset = 0x148
u3.sfdir3.list[7].name = "bash"
u3.sfdir3.list[7].inumber.i4 = 174267
u3.sfdir3.list[7].filetype = 7
u3.sfdir3.list[8].namelen = 8
u3.sfdir3.list[8].offset = 0x158
u3.sfdir3.list[8].name = "hardbash"
u3.sfdir3.list[8].inumber.i4 = 16909362
u3.sfdir3.list[8].filetype = 1
a.sfattr.hdr.totsize = 55
a.sfattr.hdr.count = 1
a.sfattr.list[0].namelen = 7
a.sfattr.list[0].valuelen = 41
a.sfattr.list[0].root = 0
a.sfattr.list[0].secure = 1
a.sfattr.list[0].parent = 0
a.sfattr.list[0].name = "selinux"
a.sfattr.list[0].value = "unconfined_u:object_r:user_home_dir_t:s0\000"
```

By golly, it's pointers all the way down. Note that, for one, we have three links to the directory - these will be:

1. `/home/wporter`
2. `/home/wporter/.ssh/..`
3. `/home/wporter/.`

> Those dot and double dot hardlinks are special. Normally, you can't create hard links in userspace - they have the potential to break the filesystem, mostly by allowing you to create "unresolvable" infinite loops that would break anything that relies on the directory graph being a tree (which is a core assumption).
>
> Why does this matter specifically for hard links, and not symlinks? Well, since you can't tell which file is "canonical" with a bunch of hard links, you can't resolve a canonical path as you can with a symlink. So, per a conscious Unix (POSIX) design decision, only some special hard links to directories are permitted to exist to allow for traversing of the directory tree.

Note the `filetype` fields - these show us our hard links, or normal files (`filetype = 1`), symlinks (`filetype = 7`) and directories (`filetype = 2`).

And here's that `hardbash` file's definition:

```txt
u3.sfdir3.list[8].namelen = 8
u3.sfdir3.list[8].offset = 0x158
u3.sfdir3.list[8].name = "hardbash"
u3.sfdir3.list[8].inumber.i4 = 16909362
u3.sfdir3.list[8].filetype = 1
```

Yep. That's it. Name and a pointer to *its* inode. So this file is effectively identical to `/bin/bash` (the source). There is no difference between hard links to files - `/bin/bash` on this system is effectively a hard link to inode 16909362, just like my `/home/wporter/hardbash` file.

One last thing to note - as hard links are simply another reference to an inode (which is a filesystem-level construct), you can't span filesystems or volumes with a hard link.

Think that's all for this one! Onwards!
