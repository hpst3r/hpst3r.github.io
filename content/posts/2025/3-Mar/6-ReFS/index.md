---
title: "ReFS sucks and you shouldn't use it, but here's how you can configure ReFS deduplication and compression (if you really want to do this to yourself)"
date: 2025-03-22T12:13:59-00:00
draft: false
---

Microsoft's ReFS is their (terrible) answer to ZFS - a "modern" filesystem with fancy features like deduplication and checksumming.

While the implementation of said features (specifically deduplication) frankly sucks, and I have not found it to be anywhere near production-ready, I have set ReFS up on my own machines to save some disk space when I'm being cheap and need Windows.

If at all possible, you should switch to Linux and use ZFS or BTRFS instead. You will most likely have a bad time with ReFS.

That said, here is how you can configure ReFS deduplication and compression with a weekly schedule:

```txt
# enable deduplication and compression
Enable-ReFSDedup -Volume D: -Type DedupAndCompress

# start an initial deduplication job
# compression level 6 is a good middle ground
# we get into diminishing returns past this
Start-ReFSDedupJob -Volume D: -Duration (New-Timespan -Hours 5) -CompressionFormat zstd -CompressionLevel 6

# there is really no good way to check on active jobs
# you will have to look at DedupStatus and watch changes
Get-ReFSDedupStatus D: | Format-List

# Configure a ReFS dedup schedule
Start-ReFSDedupSchedule -Volume D: -Start '2/24/2025 00:00:00' -Days 'Monday' -Duration (New-TimeSpan -Hours 6) -CompressionFormat zstd -CompressionLevel 6
```

Fun note - do not Receive-Job a ReFS compression job. The machine will die! Server 2025, version 26100.3476 (KB5053598).

```txt
[liam-c612-0]: PS C:\Users\liam\Documents> Receive-Job 3
WARNING: The network connection to liam-c612-0 has been interrupted. Attempting to reconnect for up to 4 minutes...
WARNING: Attempting to reconnect to liam-c612-0 ...
WARNING: Attempting to reconnect to liam-c612-0 ...
WARNING: Attempting to reconnect to liam-c612-0 ...
```

As a closing note, and I cannot say this enough, **DO NOT use ReFS with deduplication or compression for anything even remotely important.** Expect the system to go down at random and hand you a nice minidump that looks something like:

```txt
SYMBOL_NAME: ReFS!RefsFreeIoContext+5e

MODULE_NAME: ReFS

IMAGE_NAME:  ReFS.SYS
```

Anyway, that's all. Have a good one (hopefully without ReFS involvement)!