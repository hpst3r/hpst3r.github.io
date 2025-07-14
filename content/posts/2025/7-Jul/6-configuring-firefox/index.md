---
title: "Configuring Firefox via policy (registry)"
date: 2025-07-13T21:00:00-00:00
draft: false
---

Firefox supports a JSON configuration file on all platforms. If you're only working with Firefox and are dealing with a number of different platforms, this would probably be the preferred way to go about setting policy, since it's pretty easy to use.

However, since I'm primarily dealing with Windows desktops, we'll just use the registry. Mozilla's docs list the registry path as "GPO" for some reason, but they do show the registry path for all of their policies, so yay. Firefox policy works the same way as Chromium. Extensions are a bit different, but we'll get to that.

[Mozilla's policy docs can be found here.](https://mozilla.github.io/policy-templates)

## Policies of interest

You can find a sample implementation of these policies [in my BrowserPowerShell git repository](https://github.com/hpst3r/BrowserPowerShell). See the "ConfigureFirefox.ps1" script.

### Trust system root certificates

- [Certificates\ImportEnterpriseRoots](https://mozilla.github.io/policy-templates/#certificates--importenterpriseroots) 1
  - Trust certificates from the system's root certificate authority store.

### Automatically update Firefox

- [AppAutoUpdate](https://mozilla.github.io/policy-templates/#appautoupdate) 1
  - Automatically update Firefox without user approval.
- [BackgroundAppUpdate](https://mozilla.github.io/policy-templates/#backgroundappupdate) 1
  - Run background application updates for Firefox.

### Enable SSO with Entra credentials stored in Windows

- [WindowsSSO](https://mozilla.github.io/policy-templates/#windowssso) 1
  - Enable Microsoft single sign-on (SSO).

### Misc. features to disable

- [DisableFormHistory](https://mozilla.github.io/policy-templates/#disableformhistory) 1
  - Prevent information from being saved from web forms or the search bar.
- [DisableFirefoxStudies](https://mozilla.github.io/policy-templates/#disablefirefoxstudies)
  - Disable evaluation features.
- [DisableFirefoxAccounts](https://mozilla.github.io/policy-templates/#disablefirefoxaccounts) 1
  - Disable Firefox Sync account integration.
- [AutofillCreditCardEnabled](https://mozilla.github.io/policy-templates/#autofillcreditcardenabled) 0
  - Tell Firefox to not autofill your payment information.

### Disable the browser's password manager

- [PasswordManagerEnabled](https://mozilla.github.io/policy-templates/#passwordmanagerenabled) 0
  - Disable the password manager (e.g. if you've got something else).
- [OfferToSaveLoginsDefault](https://mozilla.github.io/policy-templates/#offertosaveloginsdefault) 0
  - Do not offer to save login credentials.

### Hide nags and splash screens

- [OverrideFirstRunPage](https://mozilla.github.io/policy-templates/#overridefirstrunpage) '' (empty string)
  - Disable the first-run page.
- [OverridePostUpdatePage](https://mozilla.github.io/policy-templates/#overridepostupdatepage) '' (empty string)
  - Disable the post-update landing page.
- [NoDefaultBookmarks](https://mozilla.github.io/policy-templates/#nodefaultbookmarks) 1
  - Disable the creation of default bookmarks if the profile has not yet been created.
- [DontCheckDefaultBrowser](https://mozilla.github.io/policy-templates/#dontcheckdefaultbrowser) 1
  - Do not prompt the user to set Firefox as their default browser.
- [UserMessaging](https://mozilla.github.io/policy-templates/#usermessaging)\ExtensionRecommendations 0
  - Don't recommend extensions to the user.
- UserMessaging\FeatureRecommendations 0
  - Don't recommend browser features to the user.
- UserMessaging\UrlbarInterventions 0
  - Don't offer 'Firefox-specific suggestions' in the URL bar.
- UserMessaging\SkipOnboarding 1
  - Skip the first run experience (import, etc).
- UserMessaging\MoreFromMozilla 0
  - No more from Mozilla! Leave me be!
- UserMessaging\FirefoxLabs 0
  - Disable Firefox Labs/Studies/Nimbus
- UserMessaging\Locked 0
  - Do not lock these settings (allow the user to change them if they'd like).

### Remove advertisements

- [FirefoxSuggest](https://mozilla.github.io/policy-templates/#firefoxsuggest)\WebSuggestions 0
  - Disable 'web suggestions' in Firefox.
- FirefoxSuggest\SponsoredSuggestions 0
  - Disable 'sponsored suggestions'
- FirefoxSuggest\ImproveSuggest 0
  - Disable suggestion telemetry
- FirefoxSuggest\Locked 1
  - Lock these settings so they may not be changed from the browser. To not lock these, set this to 0 or do not set it.
- [FirefoxHome](https://mozilla.github.io/policy-templates/#firefoxhome)\SponsoredTopSites 0
  - Disable sponsored 'top sites' links on the homepage
- FirefoxHome\Highlights 0
  - Disable (ad?) Highlights on the Firefox homepage
- FirefoxHome\Pocket 0
  - Disable Pocket on the Firefox homepage
- FirefoxHome\SponsoredPocket 0
  - Ditto, but ads
- FirefoxHome\Snippets 0
  - No idea what this is
- FirefoxHome\Locked 0
  - Do not lock the homepage settings. To lock these, set this to 1.

## Security and privacy options

- [HttpsOnlyMode](https://mozilla.github.io/policy-templates/#httpsonlymode) 'force_enabled'
  - Force HTTPS-only mode on.
- [DisableTelemetry](https://mozilla.github.io/policy-templates/#disabletelemetry) 1
  - Disable Mozilla telemetry.
- [DisableDefaultBrowserAgent](https://mozilla.github.io/policy-templates/#disabledefaultbrowseragent) 1
  - Prevent a telemetry scheduled task from calling home.
- [InstallAddonsPermission\Default](https://mozilla.github.io/policy-templates/#installaddonspermission) 0
  - Remove the ability for the user to install extensions.

### Cookies

- [Cookies](https://mozilla.github.io/policy-templates/#cookies)\Behavior 'reject-foreign'
  - Reject third-party cookies. Consider the less invasive 'reject-tracker-and-partition-foreign' if this is problematic, though it should not be.

### Tracking Protection

- [EnableTrackingProtection](https://mozilla.github.io/policy-templates/#enabletrackingprotection)\Value 1
  - Enable Tracking Protection.
- EnableTrackingProtection\Cryptomining 1
  - Block cryptomining scripts on websites.
- EnableTrackingProtection\Fingerprinting 1
  - Block fingerprinting scripts.
- EnableTrackingProtection\EmailTracking 1
  - Block email tracking pixels and scripts.
- EnableTrackingProtection\Locked 1
  - Allow the user to modify these settings? 1 = no, 0 = yes

### DoH

- [DNSOverHTTPS](https://mozilla.github.io/policy-templates/#dnsoverhttps)\Enabled 1
  - Enable DNS over HTTPS.
- DNSOverHTTPS\Fallback 1
  - Fall back to the system's DNS resolver if the DoH provider is unavailable.

### Safe Browsing

- [DisableSecurityBypass](https://mozilla.github.io/policy-templates/#disablesecuritybypass)\InvalidCertificate 0
  - Allow the user to bypass invalid certificate warnings.
- [DisableSecurityBypass](https://mozilla.github.io/policy-templates/#disablesecuritybypass)\SafeBrowsing 1
  - Do not allow the user to bypass Safe Browsing warnings.

## Installing extensions

[Mozilla docs here.](https://mozilla.github.io/policy-templates/#extensionsettings)

Firefox's ForceInstallList is a JSON object at Policies\ExtensionSettings. Here is the Mozilla example:

```json
{
  "*": {
    "blocked_install_message": "Custom error message.",
    "install_sources": ["https://yourwebsite.com/*"],
    "installation_mode": "blocked",
    "allowed_types": ["extension"]
  },
  "uBlock0@raymondhill.net": {
    "installation_mode": "force_installed",
    "install_url": "https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi"
  },
  "adguardadblocker@adguard.com": {
    "installation_mode": "force_installed",
    "install_url": "https://addons.mozilla.org/firefox/downloads/latest/adguardadblocker@adguard.com/latest.xpi"
  },
  "https-everywhere@eff.org": {
    "installation_mode": "allowed",
    "updates_disabled": false
  }
}
```

We can import it to a PSObject, make our changes, then export it again:

```PowerShell
PS C:\Users\liam\projects\BrowserPowerShell> $String = @"
>> {
>>   "*": {
>>     "blocked_install_message": "Custom error message.",
>>     "install_sources": ["https://yourwebsite.com/*"],
>>     "installation_mode": "blocked",
>>     "allowed_types": ["extension"]
>>   },
>>   "uBlock0@raymondhill.net": {
>>     "installation_mode": "force_installed",
>>     "install_url": "https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi"
>>   },
>>   "adguardadblocker@adguard.com": {
>>     "installation_mode": "force_installed",
>>     "install_url": "https://addons.mozilla.org/firefox/downloads/latest/adguardadblocker@adguard.com/latest.xpi"
>>   },
>>   "https-everywhere@eff.org": {
>>     "installation_mode": "allowed",
>>     "updates_disabled": false
>>   }
>> }
>> "@
PS C:\Users\liam\projects\BrowserPowerShell> $ExtensionSettings = $String | ConvertFrom-Json
PS C:\Users\liam\projects\BrowserPowerShell> $ExtensionSettings | fl

*                            : @{blocked_install_message=Custom error message.; install_sources=System.Object[];
                               installation_mode=blocked; allowed_types=System.Object[]}
uBlock0@raymondhill.net      : @{installation_mode=force_installed;
                               install_url=https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi}
adguardadblocker@adguard.com : @{installation_mode=force_installed;
                               install_url=https://addons.mozilla.org/firefox/downloads/latest/adguardadblocker@adguard.com/latest.xpi}
https-everywhere@eff.org     : @{installation_mode=allowed; updates_disabled=False}

PS C:\Users\liam\projects\BrowserPowerShell> $ExtensionSettings.'uBlock0@raymondhill.net'

installation_mode install_url
----------------- -----------
force_installed   https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi
```

This process will become checking for existence, verifying the installation_mode is set correctly, and verifying that the install_url is set correctly. I was lazy, so I just nuke the extension's settings and start over, since it's easier and I don't care about Firefox.

We're also converting everything to hashtables because PSObjects are slightly annoying and it's past my bedtime.

You must specify a full install_url for a Firefox extension, not just the ID. You can find IDs in about:debugging once you've installed the extension in question.

Here is my lazy setter to force install Firefox extensions:

```PowerShell
function Set-GeckoExtension {
  param (
    [Parameter(Mandatory)]
    [string]$ExtensionId
    ,
    [Parameter(Mandatory)]
    [ValidateSet(
      'allowed',
      'blocked',
      'force_installed',
      'normal_installed'
    )]
    [string]$InstallationMode
    ,
    [string]$InstallUrl = $null
  )

  $PolicyPath = "HKLM:\SOFTWARE\Policies\Mozilla\Firefox"
  $ValueName = "ExtensionSettings"

  # Ensure registry path exists
  if (-not (Test-Path $PolicyPath)) {
    New-Item -Path $PolicyPath -Force | Out-Null
  }

  # Read existing JSON from registry property

  $ExistingJson = (Get-ItemProperty -Path $PolicyPath -Name $ValueName).$ValueName

  # Attempt to deserialize. Create a new object if we don't have valid JSON at ExtensionSettings.
  try {

    if ($ExistingJson) {

      $Obj = $ExistingJson | ConvertFrom-Json -ErrorAction Stop

      # convert PSObjects from JSON to hashtables
      $Settings = @{}
      foreach ($Key in $Obj.PSObject.Properties.Name) {

        $Settings[$Key] = @{}
        
        foreach ($Prop in $Obj.$Key.PSObject.Properties.Name) {
          $Settings[$Key][$Prop] = $Obj.$Key.$Prop
        } # foreach

      } # foreach

    } # if
    else { $Settings = @{} } # if no existing JSON, use an empty hashtable

  } # try
  catch {

    Write-Warning "Existing ExtensionSettings do not contain valid JSON. Starting over."
    $Settings = @{}

  }

  # For simplicity, recreate the extension object with desired parameters

  $Settings[$ExtensionId] = @{
    installation_mode = $InstallationMode
  }
  if ($InstallationMode -eq "force_installed" -and $InstallUrl) {

    $Settings[$ExtensionId].install_url = $InstallUrl

  } else {

    Write-Warning "Extension $($ExtensionId) is being force_installed, but no install URL was specified. Installation will fail!"

  }

  $Json = $Settings | ConvertTo-Json -Depth 99 -Compress

  Set-ItemProperty -Path $PolicyPath -Name $ValueName -Value $Json

  Write-Host "Extension settings updated for '$($ExtensionId)'."

}
```

This, too, is in the BrowserPowerShell repository, [which you can find here](https://github.com/hpst3r/BrowserPowerShell).
