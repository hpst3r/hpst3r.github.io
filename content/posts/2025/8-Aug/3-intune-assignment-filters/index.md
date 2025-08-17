---
title: "Microsoft Intune Assignment Filters"
date: 2025-08-17T16:30:00-00:00
draft: false
---

For info on designing filters for performance and manageability, [see MS docs](https://learn.microsoft.com/en-us/intune/intune-service/fundamentals/filters-performance-recommendations).

Device assignment filters allow you to select what devices a policy assigned to users will apply to, and let you select what devices in a device group are eligible for a policy.

App assignment filters allow you to select what devices specific app protection policies apply to.

They're both useful tools for scoping down your Intune configuration policies. Let's have a look at them!

## Device assignment filters

### Creating device assignment filters

To create an assignment filter, navigate to Tenant administration > Assignment filters, then click Create. Select the type of filter you'll be creating (device or app).

{{< figure src="images/0-create.png" >}}

Name your filter, add a description, and select a platform. In my case, I'll be filtering for devices running Windows 10, version 21H2 (10.0.19044), so I'll name it "Windows 10, version 21H2", and select the "Windows 10 or later" Platform. Then, click Next.

{{< figure src="images/1-win10.png" >}}

The properties you can filter by in a 'devices' assignment filter are:

- Device name (e.g., DESKTOP-ASDFGH)
- Manufacturer (e.g., LENOVO)
- Model (e.g., 20UJS02900)
- Device category (an Intune group, created at Devices > Manage devices > Device categories)
- OS version (e.g., 10.0.26100)
- Operating System version (same as above, but does not support partial values/-in)
- Ownership (personal or corporate)
- Enrollment profile name
- Device trust type (Entra join type)
- OS SKU (e.g., Enterprise LTSC 2021 or Windows 11 Professional)

In my case, I'll be including any 21H2 build, so I'll create an osVersion filter for StartsWith 10.0.19044. You could also filter for 'contains' 19044 if you were so inclined.

> The osVersion property is being deprecated, but still works. The newer operatingSystemVersion property does not allow you to use the StartsWith or Contains operators, but DOES allow you to use the GreaterThan operator.

When done, click Preview to see what your rule will select:

{{< figure src="images/2-create.png" >}}

For fun, let's add some more stuff.

I'll select any corporate-owned, Entra-joined Lenovo device running 21H2 (10.0.19044) with an AMD64 processor that was deployed with the 'User-led enrollment' Autopilot profile that has an OS SKU that is not Core (Home).

My test ThinkPad with an AMD64 processor running Enterprise LTSC still qualifies:

{{< figure src="images/3-preview-more.png" >}}

When you're satisfied with your selection, click Next, add a scope tag (if applicable), then create the filter.

### Using device assignment filters

Let's say I'm targeting all of my Windows devices with a 24H2 security baseline, but I'd like to exclude my 21H2 machines for whatever reason (or maybe my Windows Home machines, since they don't support the relevant CSPs).

I can add my new assignment filter (which will be listed as its name and the QL filter) to my assigned group, select 'exclude filtered devices', and apply it - the 21H2 machines will be excluded from the assignment.

{{< figure src="images/4-apply-to-assignment.png" >}}

Let's do a more useful example.

I've got a policy enabling the new style of UAC sandboxing available in Insider Preview (Administrator Protection, which is basically automatic runas a managed ADMIN_YourName account after a WHfB prompt).

I've assigned it to my Windows Insider Dev channel users, but I want it to only apply to devices that meet the following criteria:

- OS build is at least 25H2 (10.0.26200)
- Device is Azure AD joined
- Operating system SKU is not Home (Core)

To do so, I'll create the following assignment filter (via Tenant Admin > Assignment Filters):

```txt
(device.operatingSystemVersion -ge 10.0.26200) and (device.deviceTrustType -eq "Azure AD joined") and (device.operatingSystemSKU -ne "Core")
```

Then, I'll apply the device assignment filter to the user group assignment:

{{< figure src="images/5-apply-to-assignment-2.png" >}}

This means the policy will only apply to compatible devices (running 25H2, which, at the moment, means Insider Dev).

## App assignment filters

### Creating app assignment filters

App assignment filters are similar, but are used to filter for managed apps on a platform (rather than filtering for a device). For example, if I select app.deviceManagementType -eq "Android Enterprise", I'll see Teams, Outlook and Edge on a handful of my Android Enterprise devices:

{{< figure src="images/6-app-filter.png" >}}

{{< figure src="images/7-app-filter-preview.png" >}}

You can use this to assign a policy to users with, for example, a corporate Android device and a BYOD Android device, to control which app protection policy applies to each device (e.g., to apply a less restrictive policy to corporate devices or apps in an Android work profile).

For example, to cover the other devices (non Enterprise), we could target NotEquals AndroidEnterprise or Equals Unmanaged. You'll now see my Pixel 9 that doesn't have an Android Enterprise work profile:

{{< figure src="images/8-app-filter-preview-2.png" >}}

### Using app assignment filters

If I apply this app assignment filter (targeting Android devices with deviceManagementType Unmanaged) to a MAM policy targeting my users, it'll apply to their unmanaged (non-Android Enterprise) devices (in my case, my Pixel 9).

If I log on to a managed app on that device with an eligible Microsoft 365 account and am forced to use an APP via Conditional Access, I'll get the protection policy applied.

{{< figure src="images/9-app-filter-applied-to-app.png" >}}
