---
title: "Notes on adding Trusted Locations via registry/rant about M365 Business"
date: 2025-10-21T23:30:00-00:00
draft: false
---

My example will be Excel, but this applies to any Office apps.

If you have Office for Business (e.g., M365 Business Premium), your apps don't support ADMX anymore (thanks, Microsoft). This is an "Office for Enterprise" (E3/E5) or Office LTSC feature only.

Sometimes, the Office admin center (config.office.com) will apply policy, but it's very unreliable and very difficult to troubleshoot - even in environments with hybrid Azure AD join configured, you'll only get a handful of your installations calling home. Don't waste your time with it. You will need to set the applicable options in the registry. This can be done through Group Policy or your preferred RMM/logon script (easy!) or Intune (perhaps as a remediation).

Anyway, Microsoft has gotten more aggressive about disabling VBA. I think you can see where this is going - folks have written their masterpiece in VBA, and it stops working when they're upgraded to subscription Office 365. You can't reenable macros on the office file server in a more "proper" way because Microsoft doesn't let you use the ADMX.

Good news! Setting things in the registry (either machine or user hives) does apply to Office for Business, and in environments that are (majority) domain-joined, this is a simple way around the issue.

To add a trusted location to a *machine*, create a key for your trusted location under `HKLM:\SOFTWARE\Microsoft\Office\16.0\$App\Security\Trusted Locations` or `HKLM:\SOFTWARE\Microsoft\Office\16.0\Common\Security\Trusted Locations`. Name it anything you'd like.

You may need to create the `Security\Trusted Locations` registry path if doing this with PowerShell; if you're doing things through Group Policy, any parent keys required will be created automatically.

Under the key, create a REG_SZ corresponding to the UNC path of the trusted location or a REG_EXPAND_SZ if, for example, you're using an environment variable like `%USERPROFILE%` in the path.

If you need to set the `AllowSubFolders` property (default is off, 0), create a DWORD and set it to 1.

If you would like, you can create another string, `Description`, with a description. This will be displayed in the Trust Center in Excel.

For example, here's what a key and its properties might look like:

```txt
HKLM:\SOFTWARE\Microsoft\Office\16.0\Common\Security\Trusted Locations
    MyLocationName (key)
        Path (SZ) = "\\FILESERVER\UNC"
        AllowSubFolders (DWORD) = 1
        Description (SZ) = "My fileserver"
```

If you would like to set a Trusted Location per-user instead (it can be changed or removed by the user, but will be recreated next time Group Policy applies) you can create a key under `HKCU:\Software\Microsoft\Office\16.0\Excel\Security\Trusted Locations\` with the same properties as a location defined in the HKLM hive.

The default naming scheme for user- or system-added Trusted Locations (like the Office default locations) is "Location n", where "n" is an integer, 1++, but you can name yours whatever and Excel will still read them. For example:

```txt
HKCU:\Software\Microsoft\Office\16.0\Excel\Security\Trusted Locations
    MyLocationName (key)
        Path (SZ) = "\\FILESERVER\UNC"
        AllowSubFolders (DWORD) = 1
        Description (SZ) = "My fileserver"
```
