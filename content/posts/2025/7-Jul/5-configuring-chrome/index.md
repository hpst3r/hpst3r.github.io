---
title: "Hardening and customizing Google Chrome via policy (registry)"
date: 2025-07-13T17:35:00-00:00
draft: false
---

Quick one here. Just going to list some policies, discuss setting them, and link to the docs and a script for setting them.

We can manage Chrome via MDM (like Intune), Group Policy (if the machine is joined to an Active Directory domain and you've imported the relevant ADMX), via 'preferences' JSON config files (Mac or Linux), or via the Windows registry (my preferred option since it applies to any Windows PC, regardless of management infrastructure, and is easy to script).

For some policies to get you started, take a look at the [Security Configuration Guide](https://support.google.com/chrome/a/answer/9710898?hl=en).

These recommendations boil down to:

- Enforce secure defaults
  - Site Isolation, HTTPS upgrades, etc.
- Update Chrome frequently
- Disable legacy authentication protocols (basic, digest), keeping only NTLM and Negotiate
- Prevent users from managing CA certificates
- Do not allow users to disable their browser history

I'll add a recommendation to deploy a good adblocker, like uBlock Origin Lite. [The CISA recommends this, too](https://www.cisa.gov/sites/default/files/publications/Capacity_Enhancement_Guide-Securing_Web_Browsers_and_Defending_Against_Malvertising_for_Federal_Agencies.pdf) - you should really be deploying one. I have a script for this, too, with some usage examples - [find it on GitHub](https://github.com/hpst3r/BrowserPowerShell/blob/main/Set-Extension.ps1). You can likely adapt this for your scenario, or take a variable for ExtensionId with a small bit of effort.

We'll be making some tweaks to telemetry and generative AI features, too.

An aside about the DNS client option - keeping the Chrome DNS client enabled allows some security features (like [Encrypted Hello](https://blog.cloudflare.com/announcing-encrypted-client-hello/), which improves privacy by making it more difficult for someone with control of the network to inspect your traffic) and HTTPS Resource Records.

This DNS client does NOT send requests to Google; it just has some additional features.

If your DNS stack is incompatible with the Chrome DNS client, it may be reasonable to force Chrome to use the system resolver. Otherwise, you should use Chrome's resolver.

If you do not supply your users a password manager, it's probably not a good idea to disable the Chrome password manager.

If you *do* provide one, you can disable the built-in Chrome password manager with the [PasswordManagerEnabled](https://chromeenterprise.google/policies/#PasswordManagerEnabled) policy (set PasswordManagerEnabled 0).

This prevents annoying double prompts to save your password and can help control where your users' passwords go.

## Setting policies

To set a policy with PowerShell, all that's needed is to set a registry property under either the recommended or enforced key (`HKLM:\SOFTWARE\Vendor\Product` or `Vendor\Product\Recommended`, like `HKLM:\SOFTWARE\Google\Chrome\Recommended`).

Here is an example of a wrapper function to do this:

```PowerShell
function Set-Policy {
  param (
    [Parameter(Mandatory)]
    [string]$PolicyPath
    ,
    [Parameter(Mandatory)]
    [string]$PropertyName
    ,
    [Parameter(Mandatory)]
    $DesiredValue
    ,
    [Parameter(Mandatory)]
    [string]$Description
  )

  # get properties of the policy key
  $PolicyProperties = (Get-ItemProperty $PolicyPath)

  # if the property's value is not already set to the desired value, set it.
  $CurrentValue = $PolicyProperties.$PropertyName

  if ($CurrentValue -eq $DesiredValue) {

    Write-Host "Policy '$($PropertyName)' is already set to '$($DesiredValue)'. No changes will be made."

  } else {

    Write-Host "Setting policy '$($PropertyName)' to '$($DesiredValue)' ($($Description))"

    Set-ItemProperty `
      -Path $PolicyPath `
      -Name $PropertyName `
      -Value $DesiredValue

  }

}
```

This is fairly trivial, so I won't be going into it very much. This function works for Edge or Chrome policies. For an example of usage, [please see my 'BrowserPowerShell' GitHub repository](https://github.com/hpst3r/BrowserPowerShell). This contains all the tweaks listed below.

## Policies to set

So now that we know how to set a policy, what do we want to set?

### Disable nags and 'private' profiling

Chrome does very well with the nags at startup (at least, it does very well here compared to Edge).

However, there's some telemetry going on, and a startup prompt about it. To disable the nag at first startup and force Google's special, Chrome-specific 'private' advertisement telemetry and profiling off, set:

- [PrivacySandboxAdMeasurementEnabled](https://chromeenterprise.google/policies/#PrivacySandboxAdMeasurementEnabled) 0
  - Uses Google's "Topics" to measure performance of advertisements.
- [PrivacySandboxAdTopicsEnabled](https://chromeenterprise.google/policies/#PrivacySandboxAdTopicsEnabled) 0
  - Uses Google's "Topics" for tracking.
- [PrivacySandboxIpProtectionEnabled](https://chromeenterprise.google/policies/#PrivacySandboxIpProtectionEnabled) 1
  - Proxies some traffic through Google to 'preserve privacy'. If you turn Safe Browsing off, you should probably turn this off, too.
- [PrivacySandboxPromptEnabled](https://chromeenterprise.google/policies/#PrivacySandboxPromptEnabled) 0
  - This is the startup prompt about 'enhanced ad privacy'. Since we're disabling everything, we can kill the prompt, too.
- [PrivacySandboxSiteEnabledAdsEnabled](https://chromeenterprise.google/policies/#PrivacySandboxSiteEnabledAdsEnabled) 0
  - Disable PrivacySandbox "site-suggested advertisements".

### Configuring Safe Browsing

In a business environment, you probably *do* want Safe Browsing enabled and enforced.

- [DisableSafeBrowsingProceedAnyway](https://chromeenterprise.google/policies/#DisableSafeBrowsingProceedAnyway) 1
  - Disallow users to continue to a site marked phishing or malware by Safe Browsing. In an ideal world this would force them to submit a ticket.
- [SafeBrowsingDeepScanningEnabled](https://chromeenterprise.google/policies/#SafeBrowsingDeepScanningEnabled) 1
  - Enable sending suspicious downloads to Google for a malware scan.
- [SafeBrowsingExtendedReportingEnabled](https://chromeenterprise.google/policies/#SafeBrowsingExtendedReportingEnabled) 1
  - Send system information and page content to Google for the detection of malicious sites ("help improve Safe Browsing"). I do not believe this has anything to do with the function of Safe Browsing on the device, so consider disabling this.
- [SafeBrowsingProtectionLevel](https://chromeenterprise.google/policies/#SafeBrowsingProtectionLevel) 2
  - Configure Safe Browsing to use the 'enhanced' protection mode, which sends more telemetry to Google, but provides better protection from malicious sites.
- [SafeBrowsingProxiedRealTimeChecksAllowed](https://chromeenterprise.google/policies/#SafeBrowsingProxiedRealTimeChecksAllowed) 1
  - Configure Safe Browsing to send more regular URL hashes to Google via an Oblivious HTTP proxy to improve security. This is enabled with the Enhanced ProtectionLevel; this policy only takes effect if SafeBrowsing is on in 'standard mode'.
- [SafeBrowsingSurveysEnabled](https://chromeenterprise.google/policies/#SafeBrowsingSurveysEnabled) 0
  - Do not present surveys about Safe Browsing to the user.
- [SafeBrowsingAllowlistDomains](https://chromeenterprise.google/policies/#SafeBrowsingAllowlistDomains) (list of strings)
  - Configure domains that will NOT be subject to Safe Browsing. This only takes effect if the device is Entra- or AD joined, connected via MDM or MCX (Mac) or the browser is enrolled in Chrome Enterprise Core.

#### Password Protection

Chrome Password Protection will capture salted hashes of passwords so it can tell if a password has been reused somewhere. If you'd like to configure this, the following policies are relevant:

- [PasswordProtectionLoginURLs](https://chromeenterprise.google/policies/#PasswordProtectionLoginURLs) (list of strings)
  - Configure the URLs that Chrome will capture and protect passwords on. This only takes effect if the device is Entra- or AD joined, connected via MDM or MCX (Mac) or the browser is enrolled in Chrome Enterprise Core.
- [PasswordProtectionChangePasswordURL](https://chromeenterprise.google/policies/#PasswordProtectionChangePasswordURL) "mysignins.microsoft.com"
  - Configure the URL that users will be redirected to so they can change a reused password.
- [PasswordProtectionWarningTrigger](https://chromeenterprise.google/policies/#PasswordProtectionWarningTrigger) 1
  - Configure when a user is alerted of password reuse (0 - never, 1 - when password is used anywhere not protected, 2 - when password is used on a phishing site)

### Enforcing secure defaults

Some good defaults to enforce are:

- [SitePerProcess](https://chromeenterprise.google/policies/#SitePerProcess) 1
  - Enforce Site Isolation
- [BlockThirdPartyCookies](https://chromeenterprise.google/policies/#BlockThirdPartyCookies) 1
  - Disallows web elements that are not from the domain listed in the address bar from setting cookies. This is the default since Q3 2024.
- [HttpsUpgradesEnabled](https://chromeenterprise.google/policies/#HttpsUpgradesEnabled) 1
  - Force HTTPS upgrades (the default setting)
- [SavingBrowserHistoryDisabled](https://chromeenterprise.google/policies/#SavingBrowserHistoryDisabled) 0
  - Do not allow users to disable their browser history. This can be useful for research post-breach.

### Extra tweaks

- [ShoppingListEnabled](https://chromeenterprise.google/policies/#ShoppingListEnabled) 0
  - Disables the price-tracking feature.
- [CACertificateManagementAllowed](https://chromeenterprise.google/policies/#CACertificateManagementAllowed) 2
  - Do not allow users to manage certificates.
- [AuthSchemes](https://chromeenterprise.google/policies/#AuthSchemes) 'ntlm,negotiate'
  - Disable basic and digest HTTP authentication.
- [BackgroundModeEnabled](https://chromeenterprise.google/policies/#BackgroundModeEnabled) 0
  - Disable "background mode" (keep Chrome running after the last window is closed) to save on memory
- Disable GenAI features
  - [GenAILocalFoundationalModelSettings](https://chromeenterprise.google/policies/#GenAILocalFoundationalModelSettings) 1
    - Do not download LLMs for use locally
  - [AIModeSettings](https://chromeenterprise.google/policies/#AIModeSettings) 1
    - Disable 'AI Mode'
  - [GeminiSettings](https://chromeenterprise.google/policies/#GeminiSettings) 1
    - Disable Gemini integration in Chrome
  - [AutofillPredictionSettings](https://chromeenterprise.google/policies/#AutofillPredictionSettings) 2
    - Disable GenAI autofill and telemetry in Chrome. Set this to 1 if you'd like to keep the GenAI but disable telemetry.
  - [HelpMeWriteSettings](https://chromeenterprise.google/policies/#HelpMeWriteSettings) 2
    - Disable GenAI 'short-form' autofill and telemetry in Chrome. Set this to 1 if you'd like to keep the GenAI but disable telemetry.
  - [HistorySearchSettings](https://chromeenterprise.google/policies/#HistorySearchSettings) 2
    - Disable GenAI history search and telemetry in Chrome. Set this to 1 if you'd like to keep the GenAI but disable telemetry.
  - [TabCompareSettings](https://chromeenterprise.google/policies/#TabCompareSettings) 2
    - Disable GenAI tab compare and telemetry in Chrome. Set this to 1 if you'd like to keep the GenAI but disable telemetry.
- [DefaultBrowserSettingEnabled](https://chromeenterprise.google/policies/#DefaultBrowserSettingEnabled) 0
  - Disable the prompt to set Chrome as your default browser on startup. Set this to 1 if you'd like Chrome to try to set itself as the default browser.
- [BrowserSignin](https://chromeenterprise.google/policies/#BrowserSignin) 0
  - Disable Google Account usage in Chrome.
- [BrowserLabsEnabled](https://chromeenterprise.google/policies/#BrowserLabsEnabled) 0
  - Disable the Browser Labs link in the toolbar. This does not affect the link on the new tab page.

That's all for today. As mentioned a bit above, for examples of usage and some extra policies for Edge, consider having a look at [my 'BrowserPowerShell' Git repo](https://github.com/hpst3r/BrowserPowerShell), which contains a setter and a selection of policies and values to start from. In the future, Gecko (Firefox) policy will go here, too.
