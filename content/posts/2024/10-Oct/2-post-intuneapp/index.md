---
title: "POST an Intune app registration to the Microsoft Graph API"
date: 2024-10-03T12:34:56-00:00
draft: false
---
## Problem: "The selected app does not have a latest package version" error preventing app registration in Intune/Endpoint Admin Center
Attempting to register Adobe Acrobat Reader DC (XPDP273C0XHQH2) as an Intune app of type Windows Store (New) results in a "The selected app does not have a latest package version" error. Intune admin center says that "This app is not supported in preview."

This seems to be a version error that seems to be caused by PackageVersion: Unknown, according to Sander Rozemuller, a M365 blogger. Speaking of them, thanks to [Sander Rozemuller's blog post](https://rozemuller.com/windows-store-app-not-supported-in-preview-in-intune/) on the topic for showing me that I can get around this. Without that post I would probably have stayed lost.
## Intermediate steps
Run a query and see what PackageVersion is! Let's see if what Sander said is probable here.
```http
POST https://storeedgefd.dsx.mp.microsoft.com/v9.0/manifestSearch {"Query": {"KeyWord": "Adobe Acrobat Reader DC", "MatchType": "SubString"}}
```
returns
```json
{
    "$type": "Microsoft.Marketplace.Storefront.StoreEdgeFD.BusinessLogic.Response.ManifestSearch.ManifestSearchResponse, StoreEdgeFD",
    "Data": [
        {
            "$type": "Microsoft.Marketplace.Storefront.StoreEdgeFD.BusinessLogic.Response.ManifestSearch.ManifestSearchData, StoreEdgeFD",
            "PackageIdentifier": "XPDP273C0XHQH2",
            "PackageName": "Adobe Acrobat Reader DC",
            "Publisher": "ADOBE INC.",
            "Versions": [
                {
                    "$type": "Microsoft.Marketplace.Storefront.StoreEdgeFD.BusinessLogic.Response.ManifestSearch.ManifestSearchVersion, StoreEdgeFD",
                    "PackageVersion": "Unknown"
                }
            ]
        }
    ]
}
```
So sure! Might be that Versions\[PackageVersion] field!
Figure out what permissions are required with the [Graph API Permissions Reference](https://learn.microsoft.com/en-us/graph/permissions-reference).
Figure out what API endpoint to use with [Graph API reference docs](https://learn.microsoft.com/en-us/graph/api/overview?view=graph-rest-1.0)! It's https://graph.microsoft.com/v1.0/deviceAppManagement/mobileApps. [Docs are here](https://learn.microsoft.com/en-us/graph/api/resources/intune-apps-deviceappmanagement?view=graph-rest-1.0).
Read the docs to figure out how to use the [Microsoft Graph PowerShell SDK](https://learn.microsoft.com/en-us/powershell/microsoftgraph/get-started?view=graph-powershell-1.0)!
## Solution: let's use the API to register an app! But I basically only know PowerShell, so we're gonna do that!
Install the Microsoft.Graph Graph SDK PowerShell Core module (might require PS7! Install PS7 with `winget install Microsoft.PowerShell`

```pwsh
Install-Module Microsoft.Graph -Scope CurrentUser -Repository PsGallery -Force
```

Connect to a M365 tenant and grant delegated access. You do not have to register an application if you do it this way. Specify permissions in connection request. Since we'll be writing an app to the DeviceManagementApps API endpoint, use the correct scope.

```pwsh
Connect-MgGraph -Scopes "DeviceManagementApps.ReadWrite.All"
```

Create a request body that you will POST. The `ConvertTo-Json` cmdlet can be used to turn PowerShell dictionaries/hashtables into JSON, so I'll write the request in native PowerShell and immediately convert it.

```json
$body = @{
	"isFeatured" = $true
	"publisher" = "Adobe"
	"roleScopeTagIds" = @()
	"repositoryType" = "microsoftStore"
	"@odata.type" = "#microsoft.graph.winGetApp"
	"packageIdentifier" = "XPDP273C0XHQH2"
	"developer" = "Adobe"
	"installExperience" = @{
		"runAsAccount" = "User"
	}
	"privacyInformationUrl" = $null
	"largeIcon" = @{
		"@odata.type" = "microsoft.graph.mimeContent"
        "value" = $null
        "type" = "String"
	}
    "description" = "Adobe Acrobat Reader DC"
    "displayName" = "Adobe Acrobat Reader DC"
    "informationUrl" = $null
} | ConvertTo-Json
```

This is what the above hashtable looks like when converted to JSON:

```json
PS C:\Users\liam> $body
{
  "isFeatured": true,
  "informationUrl": null,
  "packageIdentifier": "XPDP273C0XHQH2",
  "publisher": "Adobe",
  "description": "Adobe Acrobat Reader DC",
  "privacyInformationUrl": null,
  "installExperience": {
    "runAsAccount": "User"
  },
  "@odata.type": "#microsoft.graph.winGetApp",
  "roleScopeTagIds": [],
  "developer": "Adobe",
  "repositoryType": "microsoftStore",
  "displayName": "Adobe Acrobat Reader DC",
  "largeIcon": {
    "@odata.type": "microsoft.graph.mimeContent",
    "value": null,
    "type": "String"
  }
}
```

Make the request.

```pwsh
$response = Invoke-MgGraphRequest `
    -Uri "https://graph.microsoft.com/beta/deviceAppManagement/mobileApps" `
    -Method POST `
    -Body $body
```

If you don't get a cmdlet error, look at the response:

```pwsh
PS C:\Users\liam> $response

Name                           Value
----                           -----
lastModifiedDateTime           10/3/2024 6:07:06 PM
isFeatured                     True
installExperience              {[runAsAccount, user]}
supersededAppCount             0
developer                      Adobe
@odata.type                    #microsoft.graph.winGetApp
description                    Adobe Acrobat Reader DC
manifestHash
largeIcon
@odata.context                 https://graph.microsoft.com/beta/$metadata#deviceAppManagement/mobileApps/$entity
supersedingAppCount            0
createdDateTime                10/3/2024 6:07:06 PM
id                             e3c1a51b-9f9c-40ae-8567-28629dcfe0b6
publisher                      Adobe
publishingState                processing
owner
privacyInformationUrl
displayName                    Adobe Acrobat Reader DC
roleScopeTagIds                {}
packageIdentifier              XPDP273C0XHQH2
informationUrl
isAssigned                     False
notes
dependentAppCount              0
uploadState                    2
```

In JSON:

```json
PS C:\Users\liam> $response | ConvertTo-Json
{
  "lastModifiedDateTime": "2024-10-03T18:07:06.9665022Z",
  "isFeatured": true,
  "installExperience": {
    "runAsAccount": "user"
  },
  "supersededAppCount": 0,
  "developer": "Adobe",
  "@odata.type": "#microsoft.graph.winGetApp",
  "description": "Adobe Acrobat Reader DC",
  "manifestHash": null,
  "largeIcon": null,
  "@odata.context": "https://graph.microsoft.com/beta/$metadata#deviceAppManagement/mobileApps/$entity",
  "supersedingAppCount": 0,
  "createdDateTime": "2024-10-03T18:07:06.9665022Z",
  "id": "e3c1a51b-9f9c-40ae-8567-28629dcfe0b6",
  "publisher": "Adobe",
  "publishingState": "processing",
  "owner": null,
  "privacyInformationUrl": null,
  "displayName": "Adobe Acrobat Reader DC",
  "roleScopeTagIds": [],
  "packageIdentifier": "XPDP273C0XHQH2",
  "informationUrl": null,
  "isAssigned": false,
  "notes": null,
  "dependentAppCount": 0,
  "uploadState": 2
}
```

In the Intune portal, this is what our new app registration looks like:

{{< figure src="intune-appreg.png" alt="A screenshot of the Intune admin portal showing an application registration." >}}

When you're done, disconnect from Microsoft Graph. This gives you some session info as an object, which is neat, I guess.

```pwsh
PS C:\Users\liam> Disconnect-MgGraph

ClientId               : censored
TenantId               : censored
Scopes                 : {Directory.ReadWrite.All, openid, profile, User.Read…}
AuthType               : Delegated
TokenCredentialType    : InteractiveBrowser
CertificateThumbprint  :
CertificateSubjectName :
SendCertificateChain   : False
Account                : censored
AppName                : Microsoft Graph Command Line Tools
ContextScope           : CurrentUser
Certificate            :
PSHostVersion          : 7.4.5
ManagedIdentityId      :
ClientSecret           :
Environment            : Global
```
