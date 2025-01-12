---
title: "Enabling Hyper-V in Windows Home editions"
date: 2025-01-11T10:10:10-00:00
draft: false
---

There's a batch script floating around that does this, but I was curious to see if it still worked and wanted to rewrite it in PowerShell.

Tested and working with a Media Creation Tool version of Windows 10 Home Single Language 22H2 and Windows 11 Home 24H2. 

PowerShell:

```pwsh
Function Enable-HyperVOnWinHome {
    [CmdletBinding()]
    param ()
    $ssp = "$($env:SystemRoot)\Servicing\Packages"
    Get-ChildItem -Path $ssp |
    Where-Object Name -like "*Hyper-V*.mum" |
    ForEach-Object { & dism.exe /online /norestart /add-package:$ssp\$_ }

    & dism.exe /online /enable-feature /featurename:Microsoft-Hyper-V /all
}

# this is not required, but you might want to enable the VM Platform while you're here
# if it's missing, WSL won't work.

& dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

Proof:

{{< figure src="w10h.png" alt="A picture showing Hyper-V Mangager MMC and a local VM console on screen beside winver on a Windows 10 Home Single Language virtual machine" >}}

{{< figure src="w11h.png" alt="A picture showing Hyper-V Mangager MMC and a local VM console on screen beside winver on a Windows 11 Home Single Language virtual machine" >}}
