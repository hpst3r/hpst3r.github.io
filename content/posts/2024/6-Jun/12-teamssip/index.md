---
title: "Removing ghost SipAddress and SipAddressProxy values from M365 user accounts"
date: 2024-06-28T12:34:56-00:00
draft: false
---

Ran into this problem while attempting to remove a domain from a Microsoft tenant. Stale mailboxes that no longer had licenses were showing that they were still associated with a problem domain (as reported by `Get-MsolUser -DomainName "domain.tld"` and the Admin Center domain menu). Specifically, they had an "IM Address" associated with the domain, despite having all aliases cleared - I could find nothing in the GUI relating to this.

Some frantic Googling and PowerShelling later, [this blog post](https://blog.danielburrowes.com/2023/09/unable-to-remove-sip-aliases-microsoft-365.html) sorted me out.

As described by Daniel, these SIP aliases must be overridden to remove lingering association with their old domain. I had already stripped aliases from these users (so I did not need to do that again.)

Connecting to Teams PowerShell:

```txt
Get-CSOnlineUser "user@domaincorp.onmicrosoft.com" | Select SipProxyAddress,SipAddress | Format-List
```

and querying a user's SipProxyAddress,SipAddress values confirmed that I had these properties lingering.

Without applying a license, I could add a new SIP address to users that still existed in Exchange Online as shared mailboxes.

However, this never actually updated the SipProxyAddress and SipAddress field.

I wound up needing to apply a license that grants access to Teams, in this case Microsoft 365 Business Basic, then apply a new SIP address with a garbage value (I used the user principal name), remove the license, and carry on.

As long as the user was licensed at one point, the change seemed to propagate and wipe the lingering SipAddress values, and, eventually, I was able to clear up these associations and detach the domain.

Below is a loop that I threw together in the moment. Unfortunately, the Set-MsolUserLicense cmdlet was giving me an 'unknown error,' and it's deprecated (so I didn't want to waste time troubleshooting something that may or may not work anymore during downtime.) I wound up doing this process mostly manually.

```txt
foreach ($upn in (Get-MsolUser -DomainName "olddomain.tld").UserPrincipalName) {

    upn = "$($user)@domaincorp.onmicrosoft.com"

    Set-MsolUserLicense -UserPrincipalName $upn -AddLicenses `
    "domaincorp:O365_BUSINESS_ESSENTIALS"
    
    $address = Get-Mailbox -Identity $upn |
     Select-Object -ExpandProperty EmailAddresses |
     Where-Object {$_ -like "*olddomain.tld"}

    Set-Mailbox -Identity $upn -EmailAddresses @{Remove=$address}

    Set-Mailbox $upn -EmailAddresses @{add="SIP:$($upn)"}

    Set-MsolUserLicense -UserPrincipalName $upn -RemoveLicenses `
    "domaincorp:O365_BUSINESS_ESSENTIALS"

}
```