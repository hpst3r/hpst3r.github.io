---
title: "Deleting messages org-wide with Exchange Online PowerShell"
date: 2024-06-10T12:34:56-00:00
draft: false
---

# Why

Don't have Defender P2 (so you can't Threat Explore or manually delete messages) and need to get rid of some phishing messages batched out to your users? Not a problem (while Exchange Online PowerShell is around.)
This set of cmdlets has been very helpful to have on hand.

# How

```pwsh
PowerShell 7.4.2
PS> install-module exchangeonlinemanagement
PS> import-module exchangeonlinemanagement
PS> connect-ippssession -userprincipalname exchangeadmin@company.com
PS> $search=new-compliancesearch -name "sender@domain.com spam" -exchangelocation All -contentmatchquery '(From:sender@domain.com)'
PS> start-compliancesearch -identity $search.identity
PS> new-compliancesearchaction -searchname $search.name -purge -purgetype harddelete
```

## Sources

[MS Learn - Search for and Delete Email Messages](https://learn.microsoft.com/en-us/purview/ediscovery-search-for-and-delete-email-messages)

[MS Learn - `New-ComplianceSearch`](https://learn.microsoft.com/en-us/powershell/module/exchange/new-compliancesearch?view=exchange-ps)
