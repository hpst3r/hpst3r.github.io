---
title: "Renaming domain member detaches Computer Object"
date: 2024-11-20T12:34:56-00:00
draft: false
---

If renaming a domain member succeeds but the corresponding computer object is not updated, the trust relationship between the domain and the computer will break, preventing you from logging in, among other things.

Luckily, the fix is just modifying the `Name`, `SamAccountName` (mandatory) and `DNSHostName` (optional) properties of the computer object:

```pwsh
Get-ADComputer -Filter * | Where Name -eq 'OLDNAME' | Rename-ADObject -NewName 'NEWNAME' -PassThru | Set-ADComputer -SamAccountName 'NEWNAME$'
```

I'd probably rename the machine again afterwards to make sure nothing lingers. You can do this once the computer account name and AD object name have been updated.

An example computer object, for reference:

```txt
DistinguishedName : CN=RSAT-0,CN=Computers,DC=wporter,DC=lab
DNSHostName       : rsat-0.wporter.lab
Enabled           : True
Name              : RSAT-0
ObjectClass       : computer
ObjectGUID        : fa39a936-2ffe-428b-84f6-502e96db2344
SamAccountName    : RSAT-0$
SID               : S-1-5-21-3395636781-103295144-3172313150-1105
UserPrincipalName :
```

https://learn.microsoft.com/en-us/powershell/module/activedirectory/rename-adobject?view=windowsserver2025-ps