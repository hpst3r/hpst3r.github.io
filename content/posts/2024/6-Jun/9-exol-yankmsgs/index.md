---
title: "Deleting messages org-wide with Exchange Online PowerShell"
date: 2024-06-10T12:34:56-00:00
draft: false
---

# Why

Don't have Defender P2 (so you can't Threat Explore or manually delete messages) and need to get rid of some phishing messages batched out to your users? Not a problem (while Exchange Online PowerShell is around.)

# How

```pwsh
PowerShell 7.4.2
$sender = "sender@domain.com"
$upn = "exchangeadmin@company.com"
$searchname = "${sender} spam"

Install-Module ExchangeOnlineManagement
Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline -UserPrincipalName $upn
Connect-IPPSSession -UserPrincipalName $upn

New-ComplianceSearch -Name $searchname -ExchangeLocation All -ContentMatchQuery '(From:${sender})'
Start-ComplianceSearch -Identity $searchname
New-ComplianceSearchAction -SearchName $searchname -Purge -PurgeType harddelete
```

Optionally, block the sender's address:

```pwsh
New-TenantAllowBlockListItems -ListType Sender -Block -Entries $sender -NoExpiration
```

## Sources

[MS Learn - Search for and Delete Email Messages](https://learn.microsoft.com/en-us/purview/ediscovery-search-for-and-delete-email-messages)

[MS Learn - New-ComplianceSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/new-compliancesearch?view=exchange-ps)
