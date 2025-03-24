---
title: "Writing something like free in PowerShell"
date: 2025-03-23T12:13:59-00:00
draft: false
---

The `free` utility is great - just look at it!

```txt
[liam@m920q-1 ~]$ free -h
               total        used        free      shared  buff/cache   available
Mem:            30Gi       2.9Gi        11Gi       9.0Mi        17Gi        28Gi
Swap:           31Gi          0B        31Gi
```

Yeah! Woo!! Doesn't that look great?

But getting at it generally requires us to use a proper OS. Boo!! Not great!!

The Linux sysadmins are going to give us crap for not having this wonderful, amazing tool! How do we ~~copy it~~ create ~~the same thing~~ an *even better* tool on our lovely NT machines?

Well, querying the `Win32_OperatingSystem` WMI class for properties labelled "memory" gives us something that looks vaguely familiar, in kilobytes:

```txt
~
❯ Get-CimInstance Win32_OperatingSystem | Select *memory*

FreePhysicalMemory     : 45776976
FreeVirtualMemory      : 47923584
MaxProcessMemorySize   : 137438953344
TotalVirtualMemorySize : 76824044
TotalVisibleMemorySize : 66862572
```

If we rename this friendlier output with a `PSCustomObject`, then feed it to `Format-Table` to explicitly format it as a table:

```txt
~
❯ Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName | ForEach-Object {
∙ [PSCustomObject] @{
∙ ' ' = 'Mem:'
∙ 'total' = ($_.TotalVisibleMemorySize * (1KB/1))
∙ 'used' = (($_.TotalVisibleMemorySize - $_.FreePhysicalMemory) * (1KB/1))
∙ 'free' = ($_.FreePhysicalMemory * (1KB/1))
∙ 'available' = ($_.FreeVirtualMemory * (1KB/1))
∙ }} | Format-Table

           total        used        free   available
-          -----        ----        ----   ---------
Mem: 68467273728 24535154688 43932119040 45124259840
```

That looks like what we want..

What about swap? Well, Windows uses a pagefile for the stuff you'd think of as 'swap' (it does have a swapfile, but it's generally used for Metro components or UWP apps, not normal stuff). And, guess what - we can query the page file's properties pretty easily, as they're exposed via the `Win32_PageFileUsage` WMI class!

```txt
~
❯ Get-CimInstance Win32_PageFileUsage | Select *

Status                :
Name                  : C:\pagefile.sys
CurrentUsage          : 117
Caption               : C:\pagefile.sys
Description           : C:\pagefile.sys
InstallDate           : 7/4/2024 5:24:23 PM
AllocatedBaseSize     : 9728
PeakUsage             : 117
TempPageFile          : False
PSComputerName        :
CimClass              : root/cimv2:Win32_PageFileUsage
CimInstanceProperties : {Caption, Description, InstallDate, Name…}
CimSystemProperties   : Microsoft.Management.Infrastructure.CimSystemProperties
```

These values are in megabytes, of course, because why would you want to keep them consistent across the OS?

So.. let's do the same thing we did above, with our memory, to relabel them! Turn it into a PSCustomObject! But this time, let's convert everything to bytes so we have something uniform. Here's our page file:

```txt
~
❯ Get-CimInstance Win32_PageFileUsage | ForEach-Object {
∙     [PSCustomObject] @{
∙         ' ' = 'Page:'
∙         'total' = ($_.AllocatedBaseSize * (1MB/1))
∙         'used' = ($_.CurrentUsage * (1MB/1))
∙         'free' = (($_.AllocatedBaseSize - $_.CurrentUsage) * (1MB/1))
∙     }
∙ }

            total      used        free
-           -----      ----        ----
Page: 10200547328 122683392 10077863936
```

And here's our memory, now in bytes:

```txt
~
❯ Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName | ForEach-Object {
∙     [PSCustomObject] @{
∙         ' ' = 'Mem:'
∙         'total' = ($_.TotalVisibleMemorySize * (1KB/1))
∙         'used' = (($_.TotalVisibleMemorySize - $_.FreePhysicalMemory) * (1KB/1))
∙         'free' = ($_.FreePhysicalMemory * (1KB/1))
∙         'available' = ($_.FreeVirtualMemory * (1KB/1))
∙     }
∙ } | Format-Table

           total        used        free   available
-          -----        ----        ----   ---------
Mem: 68467273728 24447250432 44020023296 45184757760
```

So.. how do we present them together?

Well, we can feed both to Format-Table as an array, and let it figure them out! Turns out that it actually does a bang-up job at this!

```txt
~
❯ $Memory = Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName | ForEach-Object {
∙     [PSCustomObject] @{
∙         ' ' = 'Mem:'
∙         'total' = ($_.TotalVisibleMemorySize * (1KB/1))
∙         'used' = (($_.TotalVisibleMemorySize - $_.FreePhysicalMemory) * (1KB/1))
∙         'free' = ($_.FreePhysicalMemory * (1KB/1))
∙         'available' = ($_.FreeVirtualMemory * (1KB/1))
∙     }
∙ }

~
❯ $Page = Get-CimInstance Win32_PageFileUsage -ComputerName $ComputerName | ForEach-Object {
∙     [PSCustomObject] @{
∙         ' ' = 'Page:'
∙         'total' = ($_.AllocatedBaseSize * (1MB/1))
∙         'used' = ($_.CurrentUsage * (1MB/1))
∙         'free' = (($_.AllocatedBaseSize - $_.CurrentUsage) * (1MB/1))
∙     }
∙ }

~
❯ @($Memory, $Page)

          : Mem:
total     : 68467273728
used      : 24471482368
free      : 43995791360
available : 45188911104

      : Page:
total : 10200547328
used  : 122683392
free  : 10077863936


~
❯ @($Memory, $Page) | Format-Table

            total        used        free   available
-           -----        ----        ----   ---------
Mem:  68467273728 24471482368 43995791360 45188911104
Page: 10200547328   122683392 10077863936
```

Well, that looks familiar. Here's the non-human-friendly output of `free`, for reference:

```txt
liam@liam-z790-0:~$ free
               total        used        free      shared  buff/cache   available
Mem:        32745796     1075412    31798192        2564      251080    31670384
Swap:        8388608           0     8388608
```

`free` does things in kilobytes by default, and we're doing things in bytes, but whatever.

So, now, we want to be able to read this! Nobody thinks in bytes of memory nowadays. How might we go about that?

Well, if all we care about is presenting things in gigabytes, we could make a mess, and iterate over our PSCustomObjects with a rather nasty one-liner. If you hate the next guy's guts, you might as well just do this.

```txt
$Memory.PSObject.Properties.ForEach( { if ( $_.Value.GetType().Name -ne 'string' ) { $_.Value = ([string][math]::round($_.Value / 1GB, 1) + 'Gi') } } ) 
```

But we want to do things a *little* more cleanly:

```txt
Function Readable-IfyMe {
    param(
        [PSCustomObject]$Values
    )
    $Values.PSObject.Properties.ForEach({
        if ($_.Value -is [UInt64]) {
            $_.Value = switch ($_.Value) {
                { $_ -ge 1TB } { [string]([math]::Round($_ / 1TB, 1)) + 'TB'; break }
                { $_ -ge 1GB } { [string]([math]::Round($_ / 1GB, 1)) + 'GB'; break }
                { $_ -ge 1MB } { [string]([math]::Round($_ / 1MB, 1)) + 'MB'; break }
                { $_ -ge 1KB } { [string]([math]::Round($_ / 1KB, 1)) + 'KB'; break }
                default { [string]($_ + 'B') }
            }
        }
    })
}
```

Note that we're using the `PSObject.Properties.ForEach()` method, not the PowerShell ForEach-Object loop. This is because PSObjects are special and slightly annoying to work with - a ForEach-Object loop would give you everything in the PS object, and we'd need to pipe it to yet another ForEach-Object to get at individual values.

So now we can pass a pointer to our PSCustomObject to this (definitely legally named) function, and let it go to town:

```txt
~
❯ $Memory

          : Mem:
total     : 68467273728
used      : 24686641152
free      : 43780632576
available : 44895326208


~
❯ Readable-IfyMe($Memory)

~
❯ $Memory

          : Mem:
total     : 63.8GB
used      : 23GB
free      : 40.8GB
available : 41.8GB

~
❯ $Page

            total      used        free
-           -----      ----        ----
Page: 10200547328 122683392 10077863936


~
❯ Readable-IfyMe($Page)

~
❯ $Page

      total used  free
-     ----- ----  ----
Page: 9.5GB 117MB 9.4GB
```

Sweet!!

Now, let's put it all together, so we can call this with `free -h` and get a human-readable, automagically formatted output!

```txt
Function free {
    param(
        [switch]$HumanReadable,
        [string]$ComputerName = 'localhost'
    )

    Function Readable-IfyMe {
        param(
            [PSCustomObject]$Values
        )
        $Values.PSObject.Properties.ForEach({
            if ($_.Value -is [UInt64]) {
                $_.Value = switch ($_.Value) {
                    { $_ -ge 1TB } { [string]([math]::Round($_ / 1TB, 1)) + 'TB'; break }
                    { $_ -ge 1GB } { [string]([math]::Round($_ / 1GB, 1)) + 'GB'; break }
                    { $_ -ge 1MB } { [string]([math]::Round($_ / 1MB, 1)) + 'MB'; break }
                    { $_ -ge 1KB } { [string]([math]::Round($_ / 1KB, 1)) + 'KB'; break }
                    default { [string]($_ + 'B') }
                }
            }
        })
    }
    # values in kb converted to bytes
    $Memory = Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName | ForEach-Object {
        [PSCustomObject] @{
            ' ' = 'Mem:'
            'total' = ($_.TotalVisibleMemorySize * (1KB/1))
            'used' = (($_.TotalVisibleMemorySize - $_.FreePhysicalMemory) * (1KB/1))
            'free' = ($_.FreePhysicalMemory * (1KB/1))
            'available' = ($_.FreeVirtualMemory * (1KB/1))
        }
    }
    # values in mb converted to bytes
    $Page = Get-CimInstance Win32_PageFileUsage -ComputerName $ComputerName | ForEach-Object {
        [PSCustomObject] @{
            ' ' = 'Page:'
            'total' = ([UInt64]$_.AllocatedBaseSize * (1MB/1KB))
            'used' = ([UInt64]$_.CurrentUsage * (1MB/1KB))
            'free' = (([UInt64]$_.AllocatedBaseSize - $_.CurrentUsage) * (1MB/1KB))
        }
    }
    if ($HumanReadable) {
        Readable-IfyMe($Page)
        Readable-IfyMe($Memory)
    }
    @($Memory, $Page) | Format-Table
}
```

The final product:

```txt
~
❯ free -h

      total  used   free   available
-     -----  ----   ----   ---------
Mem:  63.8GB 23.1GB 40.6GB 41.6GB
Page: 9.5GB  117MB  9.4GB


~
❯ free

            total        used        free   available
-           -----        ----        ----   ---------
Mem:  68467273728 24854773760 43612499968 44700868608
Page: 10200547328   122683392 10077863936
```

Anyway, that's all! Just a fun thirty-minute doodle. Wasn't that grand?

Now I think you should probably install Windows 10 IOT LTSC on your smart toaster, join it to your domain, and run this over the network. Isn't PowerShell just the best?