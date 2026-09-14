---
title: "Offline domain join - djoin.exe"
date: 2026-09-09T20:00:00-00:00
draft: false
---

`djoin` allows you to join devices to Active Directory with a provisioning blob (a text file) that is generated somewhere with line-of-sight to a DC, then used anywhere (on an online or offline Windows image). This means you can:

- skip authentication *on the target device*
  - helps reduce privileges required on the endpoint - no longer need an account with permission to create the computer object to sign into the endpoint
- join devices while they're offline (e.g., at a remote site, or in a staging location without connectivity to the domain)
- join devices during provisioning without the need to save credentials capable of *many* or arbitrary domain joins

An offline domain-joined machine will still require line-of-sight to a domain controller for initial user authentication, Group Policy processing, Kerberos, certificate enrollment, et cetera.

To create a provisioning blob, you can generally:

```PowerShell
djoin.exe /provision `
  /domain ad.lab.wporter.org `
  /machine djoin-demo `
  /savefile C:\Users\wp.da\Desktop\djoin-demo.txt `
  /machineou "OU=Clients,OU=Computers,OU=Organization,DC=ad,DC=lab,DC=wporter,DC=org"
```

The provisioning blob is tied to a particular computer account, and should be considered a sensitive credential. Blobs should be created per-computer, used, and then removed after use.

Then, once you've gotten the provisioning blob over to your client, you can apply it by specifying a Windows installation to target and the path to the blob:

```PowerShell
djoin.exe /requestodj `
  /loadfile C:\Users\wporter\Desktop\djoin-demo.txt `
  /windowspath C:\Windows `
  /localos
```

`djoin` can also service offline Windows installations. Note the `/windowspath C:\Windows` and `/localos` flags - it is necessary to specify both the Windows path and tell `djoin` it is operating on the currently-running operating system.

You must specify both the local `%WINDIR%` and the `/localos` flag to act on the live image.

[More details on `djoin` can be found on Microsoft Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/ff793312(v=ws.11)). The documentation hasn't been updated in a while, but the utility hasn't changed in a while, either.
