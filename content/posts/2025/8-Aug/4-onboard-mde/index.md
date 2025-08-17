---
title: "Onboarding Microsoft Defender for Endpoint with Intune"
date: 2025-08-17T16:45:00-00:00
draft: false
---

## Licensing Prerequisites

Your tenant requires a license for Defender for Endpoint - either a Microsoft Defender for Endpoint P1 or P2, Microsoft Defender for Server, or Microsoft Defender for Business ("P1.5" edition for SMBs included with Microsoft 365 Business Premium).

Without one of these licenses, you will not see the Assets > Devices tab or the Settings > Endpoints menu, and cannot onboard devices to MDE.

**It may take up to 24 hours for your Defender tenant to be provisioned** once you have purchased and assigned the required licensing. Your admin account does **not** need to be licensed.

There is nothing you need to do to provision the Defender tenant for your org; it will happen automatically, but takes quite a bit of time.

If you're using Intune, obviously you'll need Intune licensing. This is, like MDE, included with Microsoft 365 Business Premium or M365 E3/E5.

## Configuration

### Intune connector

Navigate to the Defender admin center at security.microsoft.com, then expand System, select Settings, and select 'Endpoint Settings'.

{{< figure src="images/0-defender-settings-endpoint.png" >}}

In the 'advanced features' area, under 'General', scroll down and enable the 'Microsoft Intune connection':

{{< figure src="images/1-defender-intune-connection.png" >}}

This will allow the Defender service and Intune service to communicate and **enables most of the Defender for Endpoint features in the Intune admin center**, like risk evaluation in compliance policies. This change should take effect nearly immediately.

Then, hop over to intune.microsoft.com.

### Connect the platforms you'd like to onboard

**This next step is necessary**. The tooltips aren't particularly accurate. You must enable the 'Connect devices' option for the platforms you want to onboard with Intune; this is not just for risk sync.

Select Endpoint security > Setup > Microsoft Defender for Endpoint. Confirm the connection shows Available:

{{< figure src="images/2-endpoint-security-connector-state.png" >}}

Enable the connector services you'd like to use. Be sure to enable the 'Connect' option for the platform(s) you'll be onboarding with Intune. The 'create policy' option will be greyed out and/or you won't be able to onboard the devices you want to onboard if you do not.

In my tenant, I'll be enabling **Android and Windows risk sync**, and enabling the **'Allow Microsoft Defender for Endpoint to enforce Endpoint Security Configurations'** option (This allows Microsoft Defender for Endpoint to manage endpoint security configuration on devices that are not enrolled in MDM, which is a nice extra feature).

{{< figure src="images/3-select-platforms-to-sync-to-mde.png" >}}

**When done, click Save** at the top of the page, and confirm that the 'connection status' has updated to Enabled:

{{< figure src="images/4-mde-connector-enabled.png" >}}

### Create an EDR configuration policy

Next, let's start onboarding our devices. I'll be using Intune, with the EDR configuration template.

You **must** use the configuration template to automatically onboard devices - the option to pass a provisioning blob does not exist in the settings catalog as far as I'm aware.

Navigate to the Endpoint detection and response tab in the Endpoint Security area, confirm the connector is enabled, and click 'create policy'.

{{< figure src="images/5-create-edr-onboarding-policy.png" >}}

Give your policy a descriptive name, then set the 'configuration package type' to 'Auto from connector'. Intune will automatically populate the onboarding blob. Select a sample sharing option (I'd probably recommend 'all', but you can turn it off if you'd like). When done, click Next.

{{< figure src="images/6-configure-mde-policy.png" >}}

Configure any scope tags and assign the policy to a group, as you would with any other Intune configuration policy (this is just a fancy template in a different section of the portal, after all). When complete, Create the policy.

### Confirm devices are enrolled

On one of your devices, trigger an Intune sync (e.g., from the Company Portal app). Once the devices receive their MDE configuration package, they'll appear in the Assets > Devices view of the Defender admin center (security.microsoft.com) with a checkmark in the 'Onboarding status' field and 'Active' sensor health:

{{< figure src="images/7-mde-devices-populating.png" >}}

To confirm MDE is functioning, you can use the EICAR test file, which will be detected as malware on disk:

```PowerShell
Invoke-WebRequest "https://secure.eicar.org/eicar.com.txt" -OutFile "test.txt"
```

Or, you can run the 1.exe MDE test:

```cmd
powershell.exe -NoExit -ExecutionPolicy Bypass -WindowStyle Hidden $ErrorActionPreference = 'silentlycontinue';(New-Object System.Net.WebClient).DownloadFile('http://127.0.0.1/1.exe', 'C:\\test-MDATP-test\\invoice.exe');Start-Process 'C:\\test-MDATP-test\\invoice.exe'
```

The EICAR test file will pop up a Defender alert:

{{< figure src="images/8-defender-toast.png" >}}

The 1.exe test will not. Both will create an alert in the Defender admin center (under Incidents & alerts > Incidents).

{{< figure src="images/9-mde-admin-incident.png" >}}

To generate a more interesting incident to look at, I'd recommend something like [APTSimulator (GitHub)](https://github.com/NextronSystems/APTSimulator) (though it's been unmaintained for a while, it'll generate a bunch of events and can mark a device risky for your Intune labs).

### Configure email alerts in the Defender admin center

To get email alerts for incidents, configure notifications at Settings > Microsoft Defender XDR > Email notifications:

{{< figure src="images/10-xdr.png" >}}

{{< figure src="images/11-incident-notification-rule.png" >}}
