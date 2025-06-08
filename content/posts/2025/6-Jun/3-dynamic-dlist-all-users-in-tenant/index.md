---
title: "Creating a dynamic 'all staff' distribution list targeting enabled user mailboxes"
date: 2025-06-05T20:00:00-00:00
draft: true
---

## Introduction

Get away from your manual allstaff@company distribution lists! Use dynamic lists!

It's slightly annoying at first, but you won't have to think about it again afterwards!

You will have to do at least some of this with PowerShell or via API (not covered here).
You can create the lists without PowerShell, but need PowerShell to configure the
recipient filter on the dynamic distribution list.

## Config

### Prerequisites

Exchange Administrator privileges and the ability to connect to Exchange Online PowerShell.

The [Exchange Online PowerShell Module](https://learn.microsoft.com/en-us/powershell/exchange/connect-to-exchange-online-powershell?view=exchange-ps),
which you can install with:

```PowerShell
Install-Module ExchangeOnlineManagement -Scope CurrentUser
```

### Overview

We will be doing two main things:

- Creating a plain old manual distribution list as an 'exclusions' group
- Creating a dynamic distribution list with filter rules selecting members

I'll go over doing this with PowerShell. You can do *some* parts through the
Exchange Online admin portal, but not all of it.

### Creating the distribution lists

I'm not going to cover creating non-dynamic DLs in much detail here.

Here is how you create a list:

```PowerShell
New-DistributionGroup `
  -Name 'Excluded Folks' `
  -Members 'bobby@tables.com','tabless@tables.com','baz@tables.com'
```

Here is how you add a user to said list:

```PowerShell
Add-DistributionGroupMember `
  -Identity 'Excluded Folks' `
  -Member 'bar@tables.com'
```

```PowerShell

~
❯ $ExclusionGroup = Get-Group "not-staff-not-shared@contoso.onmicrosoft.com"

~
❯ $AllStaffGroup = Get-DynamicDistributionGroup -Identity 'All-Staff@contoso.com'

~
❯ $PreRecipients = Get-Recipient -Filter ($AllStaffGroup.RecipientFilter)

~ took 8s
❯ $PreRecipients | Measure-Object -Sum 2> $null

Count             : 120
Average           :
Sum               :
Maximum           :
Minimum           :
StandardDeviation :
Property          :

~
❯ Set-DynamicDistributionGroup -Identity "allstaff@contoso.com" -RecipientFilter "((RecipientTypeDetails -eq 'UserMailbox') -and (MemberOfGroup -ne '$($ExclusionGroup.DistinguishedName)'))"

~
❯ $PostRecipients = (Get-Recipient -RecipientPreviewFilter $AllStaffGroup.RecipientFilter) | Select PrimarySmtpAddress

~
❯ $AllStaffGroup | fl Name,RecipientFilter

Name            : All-Staff
RecipientFilter : ((((RecipientTypeDetails -eq 'UserMailbox') -and (MemberOfGroup -ne 'CN=Generic
Inbox20250605183903,OU=contoso.onmicrosoft.com,OU=Microsoft Exchange Hosted
Organizations,DC=NAMPR00A000,DC=PROD,DC=OUTLOOK,DC=COM'))) -and (-not(Name -like
'SystemMailbox{_')) -and (-not(Name -like 'CAS\_{_')) -and (-not(RecipientTypeDetailsValue -eq
'MailboxPlan')) -and (-not(RecipientTypeDetailsValue -eq 'DiscoveryMailbox')) -and
(-not(RecipientTypeDetailsValue -eq 'PublicFolderMailbox')) -and
(-not(RecipientTypeDetailsValue -eq 'ArbitrationMailbox')) -and
(-not(RecipientTypeDetailsValue -eq 'AuditLogMailbox')) -and (-not(RecipientTypeDetailsValue
-eq 'AuxAuditLogMailbox')) -and (-not(RecipientTypeDetailsValue -eq
'SupervisoryReviewPolicyMailbox')))

```

Get-Recipient -Filter ($AllStaffGroup.RecipientFilter)
