---
title: "Configuring the cloud Kerberos trust - Kerberos SSO to domain resources with WHfB for Entra-joined clients"
date: 2026-03-07T17:30:00-00:00
draft: false
---

The scenario: I have a hybrid AD environment (identity and password hash synchronization) with Entra-joined endpoints. I'd like to authenticate to both cloud resources and on-premise domain-joined resources (via Kerberos) with my Windows Hello keypair.

## Introduction

### Entra-joined devices authenticating to domain resources

Right out of the box, an Entra-joined workstation with a user who's synchronized from the on-premise domain and has signed in with a password will be able to authenticate to domain resources.

More accurately, the way this process works is that if the user attempts to authenticate to a resource requiring Kerberos or NTLM, the device will attempt to locate a DC and retrieve a Kerberos TGT or NTLM token for authentication. [See the documentation for more info](https://learn.microsoft.com/en-us/entra/identity/devices/device-sso-to-on-premises-resources).

However, this process depends on the user's password. This becomes a problem if you want to use Windows Hello for Business for more secure passwordless authentication. If you sign in with a PIN, the device is unable to retrieve a TGT, and you'll get a 'domain not available' error:

{{< figure src="images/0-no-tgt.png" >}}

The solution? Configuring a cloud Kerberos trust for your domain and Entra tenant!

### Cloud Kerberos trust background

Entra ID Kerberos joins your domain as a RODC computer object, then acts as a KDC (Key Distribution Center) in Entra to issue partial TGTs to Entra-joined clients (Entra only handles the initial authentication server (AS) exchange, and then your AD DS infrastructure handles the rest of the TGS request when needed).

So, when the client authenticates with WHfB to get its PRT, it receives a partial TGT from Entra as well, and can fire this off at a DC if the user needs to access resources via Kerberos.

With this authentication flow, the credential from the user unlocks their WHfB private key, which is used by the Cloud Authentication Provider to sign the authentication request.

The signed request is then sent to Microsoft Entra ID, which generates a PRT (encrypted to one of the device's keypairs), and sends the request along to Entra Kerberos, which provides a partial TGT.

This partial TGT simply includes the user SID - it serves the purpose of completing the AS exchange in Entra ID (effectively performing Kerberos preauthentication with WHfB rather than the user's password), but leaves authorization up to Active Directory, and is encrypted so that only the on-premises domain controllers can decrypt it with the `krbtgt_AzureAD` key to create the full TGT.

Then, Entra ID sends the PRT and partial TGT back to the client device. The client device can now authenticate to M365 resources with the PRT, but still needs to turn the partial TGT into a full TGT for SSO to Kerberos resources.

So, the device will then present the partial TGT to a domain controller as if it were any other Kerberos request, and the on-premises domain controller will decrypt and authorize the request, exchanging the partial TGT for a full Kerberos TGT for the user as if they'd signed in with a password.

## Configuration

### Microsoft Entra Kerberos

To configure the cloud Kerberos trust itself, all you need to do is install a PowerShell module and run a cmdlet from a PAW. You may need to provide credential objects if you're not signed in as a Domain Admin.

> In my case, I can't easily provide a credential for the M365 admin account since I've enforced strong authentication everywhere - providing the UPN instead of a credential as I've done here will cause an interactive cloud authentication prompt.

```PowerShell
Install-Module `
    -Name AzureADHybridAuthenticationManagement

# configure the domain and M365 for the cloud trust
# this will create a RODC in your DCs OU
Set-AzureADKerberosServer `
    -Domain "ad.wporter.org" `
    -UserPrincipalName "william.admin@portercomputing.com"
```

Then, confirm the cloud RODC has been created:

```txt
PS C:\Users\Administrator.AD> Get-ADComputer -Filter "Name -like '*Kerberos'"


DistinguishedName : CN=AzureADKerberos,OU=Domain Controllers,DC=ad,DC=wporter,DC=org
DNSHostName       :
Enabled           : True
Name              : AzureADKerberos
ObjectClass       : computer
ObjectGUID        : 55c6f6d3-c26d-4fa7-b5a3-67294206282e
SamAccountName    : AzureADKerberos$
SID               : S-1-5-21-882220983-1801536033-1692761748-1110
UserPrincipalName :
```

If you need to rotate the krbtgt keys:

```PowerShell
Set-AzureADKerberosServer -Domain $Domain -CloudCredential $CloudCred -DomainCredential $DomainCred -RotateServerKey
```

### Enabling Windows Hello for Business and "use cloud trust"

Now that it's configured, we'll need to tell devices to use the cloud Kerberos trust. We'll do this with an Intune policy.

{{< figure src="images/1-intune-new-policy.png" >}}

I'll choose "Platform" W10 and later, and "type" settings catalog:

{{< figure src="images/2-intune-settings-catalog.png" >}}

I'll choose a descriptive name for the configuration policy, then select the "Use Cloud Trust For On Prem Auth" setting from the "Windows Hello For Business" settings catalog category. Then, I'll Enable that setting.

Note that I've already enabled Windows Hello for Business elsewhere, so I don't need to do it again here. If you do not already have WHfB enabled, you should also enable the "Use Windows Hello For Business" setting (for the Device and/or User).

{{< figure src="images/3-create-policy.png" >}}

In this case, I won't be configuring scope tags and I'll just be assigning it to the All Devices group.

{{< figure src="images/4-confirm-create.png" >}}

Then, I'll Sync my test workstation.

In this scenario, my workstation already had Hello for Business configured (Entra-only, not migrating from key or cert hybrid trust). I found that I had to wipe out the Hello container (`certutil.exe -deleteHelloContainer`) and reprovision Hello to get the cloud TGT working right after configuring the cloud trust - without this, my client PC would never get a TGT.

> This is not a documented requirement, and Microsoft doesn't go into any detail that could clue me in to why this was required in the documentation I was reading.

To determine the current state of the CloudTGT configuration on a machine, you can review:

1. The output of `dsregcmd /status` (SSO State: CloudTgt : YES)
2. Event logs (Applications and Services logs > Microsoft > Windows > User Device Registration: "Cloud trust for auth policy is enabled")
3. `klist cloud_debug` - if your machine is unable to retrieve a full TGT from a DC, you'll see Cloud Primary TGT available: 0 as below:

> This is the broken state! If the cloud TGT is available: 1, you're good to go.

```txt
PS C:\Windows\system32> klist cloud_debug

Current LogonId is 0:0x14d718
Cloud Kerberos Debug info:
Cloud Kerberos enabled by policy: 0
AS_REP callback received: 1
AS_REP callback used: 0
Cloud Referral TGT present in cache: 0
SPN oracle configured: 0
KDC proxy present in cache: 0
Public Key Credential Present: 1
Password-derived Keys Present: 0
Plaintext Password Present: 0
AS_REP Credential Type: 2
Cloud Primary (Hybrid logon) TGT available: 0
```

Anyway, after wiping out the Hello container, signing out, and reprovisoning Hello for Business, I was able to confirm that the cloud TGT was available and authenticate to domain resources from my AAD-joined client:

```txt
PS C:\Windows\system32> sl \\dc0.ad.wporter.org\SYSVOL
PS Microsoft.PowerShell.Core\FileSystem::\\dc0.ad.wporter.org\SYSVOL> gci


    Directory: \\dc0.ad.wporter.org\SYSVOL


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d----l         2/15/2026   9:58 PM                ad.wporter.org


PS Microsoft.PowerShell.Core\FileSystem::\\dc0.ad.wporter.org\SYSVOL> whoami
ad\william
```

I'm going to cut this here for now.. there's some more digging I could do, but this was just a prerequisite for another lab.
