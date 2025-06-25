---
title: "Enumerating user profiles on a device with PowerShell"
date: 2025-06-25T13:30:00-00:00
draft: false
---

I was working on a script to set values in all existing profiles' HKEY_USER hives (more on that soon) and wound up coming up with a quick function to enumerate all the users that have a profile on a computer.

It would probably be fairly simple to compare the profiles for which no SID to name mapping was possible and delete their C:\Users directories to free up disk, but I won't be doing that just yet.

```PowerShell
# enumerate user profiles and translate SIDs to usernames when possible
# handles local, domain (5) and AAD (12) accounts
Function Get-Profiles {

  Get-CimInstance Win32_UserProfile |
  Where-Object SID -match '^S-1-(5|12)-\d{1,2}-\d+-\d+-\d+-.*' |
  ForEach-Object {
    $SID = [System.Security.Principal.SecurityIdentifier]$_.SID
    try {
        $SID.Translate([System.Security.Principal.NTAccount]).Value
    } catch {
        $SID.Value
    }
  }

}
```

You can find this in [my 'MiscWindowsScripts' repository on GitHub](https://github.com/hpst3r/MiscWindowsScripts).

Example output:

```PowerShell
PS C:\WINDOWS\system32> Get-Profiles
WP14G5I\localadmin
S-1-5-21-98111195-3049186222-2150619400-1002
WP14G5I\wporter
```
