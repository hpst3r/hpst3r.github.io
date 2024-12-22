---
title: "Hide 'sign into all apps' pop-up in Microsoft apps"
date: 2024-12-10T12:34:56-00:00
draft: false
---

## Problem

Users, when signing into Teams or Office, are prompted to "sign into all apps on this device" (Entra join a machine), which then fails with error "Your account was not set up on this device because device management could not be enabled" with code 80192EE7 (server msg: 0x80192ee7.)

We don't want users getting this prompt at all.

## Solution

Luckily, there's a way to do just that (block our users from getting this prompt.) It's only possible to do this directly from the registry, with key and value:

```txt
HKLM\SOFTWARE\Policies\Microsoft\Windows\WorkplaceJoin
"BlockAADWorkplaceJoin"=dword:00000001
```

You can deploy this with Reg:Create via GPO.

This hides the option to "Sign into all apps on this device" (AAD join) and prevents users from getting error messages when they inevitably click through on devices that cannot be Entra joined.