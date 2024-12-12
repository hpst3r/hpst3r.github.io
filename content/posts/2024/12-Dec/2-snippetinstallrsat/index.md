---
title: "Snippet: one-liners to install all RSAT utilities"
date: 2024-12-02T12:34:56-00:00
draft: false
---

### Windows Server 2025:

`Get-WindowsFeature | Where Name -like "*RSAT*" | Install-WindowsFeature`

### Windows 11:

`Get-WindowsCapability -Online | Where Name -like "*RSAT*" | Add-WindowsCapability -Online`