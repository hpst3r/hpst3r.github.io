---
title: "Snippet: using Winget from system context (NinjaRMM scripts)"
date: 2024-11-22T12:34:56-00:00
draft: false
---

Yes, NinjaRMM has features for Winget package management. No, I didn't look it up, and it wasn't enabled in our tenant.

```
$WingetPath = Resolve-Path "C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller*\winget.exe"

if (-not $WingetPath){

	Write-Error -Message "Winget path not found. Failure. Exiting."
	return 1
	
}

$WingetPath = Split-Path -Path $WingetPath -Parent

Set-Location $WingetPath

.\winget.exe install --exact --id Microsoft.Teams --silent --accept-package-agreements --accept-source-agreements
```