---
title: "Configuring silent OneDrive sign-in and Known Folders sync"
date: 2025-08-17T17:00:00-00:00
draft: false
---

[MS Learn link here](https://learn.microsoft.com/en-us/sharepoint/use-silent-account-configuration).

You can do this via Group Policy or Intune, depending on how your environment is configured, but the policies are the same.

## Deploying silent OneDrive sync with Intune

For successful, silent sync, you MUST configure these three policies:

1. Silently move known folders to OneDrive
2. Silently sign users in to OneDrive with their Windows credentials
3. Enable Files on Demand

Here is an example of this minimal configuration in Intune. Choose just these settings from the Settings Catalog and enable them:

{{< figure src="images/0-create-config.png" >}}

Without Files On Demand, the user will always be prompted to choose which folders to sync (even if this option is explicitly disabled via policy).

I would recommend also enabling the EnableSyncAdminReports policy [so you can see a report of OneDrive utilization in the Apps admin center under Health > OneDrive Sync](https://learn.microsoft.com/en-us/sharepoint/sync-health?tabs=windows).

Additionally, if MFA is required per Conditional Access "All Cloud Resources", you'll need the user to have already manually passed MFA (e.g., they've already been prompted when signing in to their email or Office) or you'll need Windows Hello for Business so your users pass MFA by signing in to the device (you should set this up if you don't already have it).

To reattempt a OneDrive silent sign-in, you'll need to log out of OneDrive and clear the OneDrive for Business silent config state registry keys (since silent ODfB will not set itself up again if the user has signed out). These keys are applicable if the sign-in did not run.

```cmd
reg delete HKCU\Software\Microsoft\OneDrive /v SilentBusinessConfigCompleted /f
reg delete HKCU\Software\Microsoft\OneDrive /v ClientEverSignedIn /f
reg delete HKCU\Software\Microsoft\OneDrive /v PersonalUnlinkedTimeStamp /f
reg delete HKCU\Software\Microsoft\OneDrive /v OneAuthUnrecoverableTimestamp /f
```

I would recommend *against* silently rolling out OneDrive if you're still using legacy Office (like 2016) as OneDrive often causes sync failures and **will** lose some of your users' work.

That said, this won't be a common problem for much longer (as Office 2016 and 2019 are EOL in October).

## Group Policy

### Finding ADMX

You can retrieve the OneDrive ADMX from a OneDrive installation on a client. The files are located in the OneDrive install directory, typcially `C:\Program Files\Microsoft OneDrive\version\adm`:

{{< figure src="images/1-admx-location-file-explorer.png" >}}

### Config

Copy the ADMX files to your central policy store, then configure a Group Policy targeting the same settings (Administrative Templates > OneDrive > Silently move Windows known folders to OneDrive, Silently sign in users to the OneDrive sync app with their Windows credentials, and Use OneDrive Files On-Demand).

I'm not going to go over this in too much detail; it's rather straightforward.
