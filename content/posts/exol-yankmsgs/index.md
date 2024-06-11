---
title: "Deleting messages org-wide with Exchange Online PowerShell"
date: 2024-06-10T12:34:56-00:00
draft: false
---

# Why

Don't have Defender P2, can't Threat Explore or manually delete messages, but need to get rid of some phishing messages batched out to your users?

# How

```pwsh
PowerShell 7.4.2
PS C:\Users\liam> install-module exchangeonlinemanagement
PS C:\Users\liam> import-module exchangeonlinemanagement
PS C:\Users\liam> connect-ippssession -userprincipalname exchangeadmin@company.com
PS C:\Users\liam> $search=new-compliancesearch -name "sender@domain.com spam" -exchangelocation All -contentmatchquery '(From:sender@domain.com)'
PS C:\Users\liam> start-compliancesearch -identity $search.identity
PS C:\Users\liam> new-compliancesearchaction -searchname "sender@domain.com spam" -purge -purgetype harddelete
```

## Sources

[MS Learn - Search for and Delete Email Messages](https://learn.microsoft.com/en-us/purview/ediscovery-search-for-and-delete-email-messages)

[MS Learn - `New-ComplianceSearch`](https://learn.microsoft.com/en-us/powershell/module/exchange/new-compliancesearch?view=exchange-ps)

## Note

This will be deprecated as Microsoft switches to REST API only and drops support for RPS. Expect it to not work post July 2025. You can also probably expect an update from me before then, as this has saved my butt a few times.