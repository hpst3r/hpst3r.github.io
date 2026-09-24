---
title: "Granting a principal API access to a single SharePoint site with Sites.Selected permissions - Graph API and PowerShell"
date: 2026-09-23T21:00:00-00:00
draft: false
categories:
  - M365
  - PowerShell
---


## Introduction

When granting a principal access to anything, it's best practice to scope that access down as much as possible. As of 2026, the only way to avoid granting full access to *every* SharePoint site to your application when you need to access *some* SharePoint data is to use the Sites.Selected Graph permission. Unfortunately, to grant Sites.Selected access to SharePoint Online, you'll need to make API requests; this permission is not available through the Entra app registration admin interface. You'll also need to create the read/write permission on the SharePoint site you want the application to have access to (Sites.Selected permissions are an implicit deny; the 'exclusions' are applied to the SharePoint site and also cannot be modified through the web interface).

The simplest way to connect with the required permissions is (in my opinion) to use delegated authentication with Graph PowerShell.

In other examples online, you'll often see devs maintain a "delegated admin app" with the required permissions so they can use a secret or certificate and their preferred SDK to make the same requests (and this works, too) but I find simply using the existing PowerShell module app with delegated authentication is easier and cleaner, assuming you're comfortable with PowerShell and won't be doing this too frequently.

## Dependencies

You'll need both PowerShell Core 7 (the `Microsoft.PowerShell` WinGet package) the `Microsoft.Graph.Authentication` module to continue; install it with `Install-Module Microsoft.Graph.Authentication -Scope CurrentUser`. This works with PS7 on Windows, mac OS, and Linux. Windows PowerShell 5 doesn't play nicely with the Graph module.

## Granting permissions

### Overview

To grant these Graph permissions , using your preferred method (e.g., Graph PowerShell or your own application), you'll need to:

- Connect to Graph with the `AppRoleAssignment.ReadWrite.All` and `Application.Read.All` permissions.
- Get the service principal ID for Microsoft Graph itself
- Get the target service principal (i.e., the application registration or managed identity) from the `servicePrincipals` API endpoint.
- Retrieve the role ID for the desired Sites.Selected global permission
- Grant the Sites.Selected permission, by ID, to our service principal with a request to the `servicePrincipals/id/appRoleAssignments` endpoint
- Then grant access to the specific SharePoint site to the client ID with a request to the `sites/site/permissions` endpoint.

This works with any Entra application; all you need is the client ID (for a Managed Identity, accessible in the Azure Portal at Managed Identity > Overview > Client ID; for a "traditional" Entra application registration, accessible in the Entra portal at App Registrations > Overview > Application (client) ID).

### PowerShell

You can accomplish this with the following PowerShell; sub in your own client ID and site GUID. If you need to find the site GUID, log in to Microsoft 365 with an account with access to the site and connect to `https://tenant.sharepoint.com/sites/sitename/_api/site/id` in your web browser.

> You need site admin permissions to configure permissions on the site itself. The simplest way to add these is through the SharePoint admin center.

```PowerShell
# find the client ID either under your application registration
# or, for managed identity, under the Overview for the resource
$ApiClientId = '14fafc67-0000-0000-0000-000000000000'
# find the site GUID at https://tenant.sharepoint.com/sites/sitename/_api/site/id
$SiteGUID = '0c0c6969-0000-0000-0000-000000000000'

Connect-MgGraph -Scopes "Sites.FullControl.All","AppRoleAssignment.ReadWrite.All","Application.Read.All"

$GraphSP  = Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/v1.0/servicePrincipals?`$filter=appId eq '00000003-0000-0000-c000-000000000000'"
$TargetSP = Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/v1.0/servicePrincipals?`$filter=appId eq '$ApiClientId'"

$GraphSPId  = $graphSP.value[0].id
$TargetSPId = $targetSP.value[0].id
$TargetSPDisplayName = $targetSP.value[0].displayName

# Find the Sites.Selected app role ID
$AppRoleId = ($GraphSP.value[0].appRoles | Where-Object value -eq 'Sites.Selected').id

# grant the Sites.Selected app role
$SitesSelectedPermissions = Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/v1.0/servicePrincipals/$($TargetSPId)/appRoleAssignments" `
    -ContentType "application/json" `
    -Body (@{
        principalId = $TargetSPId
        resourceId  = $GraphSPId
        appRoleId   = $AppRoleId
    } | ConvertTo-Json)

# grant access to the specific site
$SharePointSpecificSitePermissions = Invoke-MgGraphRequest -Method POST `
    -Uri "https://graph.microsoft.com/v1.0/sites/$($SiteGUID)/permissions" `
    -ContentType "application/json" `
    -Body (@{
        roles               = @("write")
        grantedToIdentities = @(@{
            application = @{
                id          = $ApiClientId
                displayName = $TargetSPDisplayName
            }
        })
    } | ConvertTo-Json -Depth 5)
```

### Details

When we get the service principal, we'll receive the following datastructure:

```PowerShell
PS C:\Windows\System32> $GraphSP.value[0]

Name                           Value
----                           -----
appDescription
description
isDisabled
info                           {[supportUrl, ], [marketingUrl, ], [privacyStatementUrl, ], [logoUrl, ]…}
appOwnerOrganizationId         f8cdef31-0000-0000-0000-000000000000
createdDateTime                11/21/2024 8:01:24 PM
oauth2PermissionScopes         {ebfcd32b-0000-0000-0000-000000000000, e4aa47b9-0000-0000-0000-000000000000, 5af8c3f5-b…
servicePrincipalNames          {00000003-0000-0000-c000-000000000000/ags.windows.net, 00000003-0000-0000-c000-00000000…
preferredTokenSigningKeyThumb…
deletedDateTime
verifiedPublisher              {[verifiedPublisherId, ], [addedDateTime, ], [displayName, ]}
homepage
displayName                    Microsoft Graph
loginUrl
samlSingleSignOnSettings
signInAudience                 AzureADMultipleOrgs
logoutUrl
disabledByMicrosoftStatus
appId                          00000003-0000-0000-c000-000000000000
servicePrincipalType           Application
notificationEmailAddresses     {}
alternativeNames               {}
resourceSpecificApplicationPe… {10d712aa-0000-0000-0000-000000000000, ff9d3910-0000-0000-0000-000000000000, 22748df0-b…
tokenEncryptionKeyId
createdByAppId
notes
appRoleAssignmentRequired      False
keyCredentials                 {}
appRoles                       {d07a8cc0-0000-0000-0000-000000000000, ef5f7d5c-0000-0000-0000-000000000000, 18228521-a…
id                             f3da322d-0000-0000-0000-000000000000
appDisplayName                 Microsoft Graph
preferredSingleSignOnMode
replyUrls                      {}
addIns                         {}
tags                           {}
applicationTemplateId
accountEnabled                 True
passwordCredentials            {}
```

Note that we're selecting the GUID for the service principal, rather than the client ID. These are separate to facilitate Azure/Entra multi-tenant applications - the service principal is the "thing" in your tenant, and the client ID is the "thing" in every tenant.

This is just how Entra works, so while in the context of single-tenant Managed Identity we're only using the client ID to look up the service principal ID so we can assign permissions to the service principal *in our tenant* (for Managed Identity, the client IDs are fully managed by Microsoft and we generally don't need to use them for anything) we do still have both IDs to deal with.

Note that we're granting permissions to the local Entra service principal to access SharePoint, then using the *client ID* for our operations in SharePoint (granting the global application registration for this Managed Identity permissions to specific SharePoint sites) since SharePoint is a (somewhat) bolted-on external service.

We also retrieve the DisplayName from the service principal so we don't need to find it manually.

In the response for the POST to grant Sites.Selected Graph permissions, we can see the role ID, permission ID, and the principal to which we applied the permission:

```PowerShell
PS C:\Windows\System32> $SitesSelectedPermissions

Name                           Value
----                           -----
resourceDisplayName            Microsoft Graph
appRoleId                      883ea226-0000-0000-0000-000000000000
id                             1JE36lngj0Ckkh9ulzPwQbVa-0000000000-0000000
@odata.context                 https://graph.microsoft.com/v1.0/$metadata#appRoleAssignments/$entity
createdDateTime                4/27/2026 6:40:57 PM
principalDisplayName           example-uami
principalType                  ServicePrincipal
deletedDateTime
resourceId                     f3da322d-0000-0000-0000-000000000000
principalId                    ea3791d4-0000-0000-0000-000000000000
```

The response to the SharePoint API request will include the permission object's ID, as well as information on the identities we granted said permission *to*. Nothing crazy going on here.

```PowerShell
PS C:\Windows\System32> $SharePointSpecificSitePermissions

Name                           Value
----                           -----
@odata.context                 https://graph.microsoft.com/v1.0/$metadata#sites('0c0c6969-0000-0000-0000-000000000000'…
grantedToIdentities            {System.Collections.Hashtable}
roles                          {write}
id                             f0000000000000000000000000000000000000000000000000000000000000000000000000000000000000…
grantedToIdentitiesV2          {System.Collections.Hashtable}
PS C:\Windows\System32> $SharePointSpecificSitePermissions.grantedToIdentities

Name                           Value
----                           -----
application                    {[id, 14fafc67-0000-0000-0000-000000000000], [displayName, example-uami]}

```
