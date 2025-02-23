---
title: "Cisco 3850 ignoring startup-config on boot, config-register stuck at 0x142  "
date: 2025-02-22T10:12:59-00:00
draft: false
---

## Problem

This appears to be [Cisco bug CSCui54490](https://quickview.cloudapps.cisco.com/quickview/bug/CSCui54490).

After a write erase (or, I believe, a format flash: or a PW recovery in this case?) a 3850 will ignore its startup-config and always boot a blank configuration.

My switch is running IOS-XE 16.12.11 (Gibraltar), date Sun 31-Mar-24 12:30 (a minor version behind at the time of writing), though this seems to be a long-standing issue.

`show version` output suggests config register is 0x142.

```txt
Switch#sh ver | inc register
Configuration register is 0x142 (will be 0x102 at next reload)
```

## Solution

In config mode, set, then unset, the `ignore startupconfig` boot variable (`SWITCH_IGNORE_STARTUP_CFG`).

```txt
Switch(config)#system ignore startupconfig switch all
Applying config on Switch 1...[DONE]
Switch(config)#no system ignore startupconfig switch all
Applying config on Switch 1...[DONE]
Switch(config)#end
Switch#reload
```

Once the switch comes back up, verify that it loaded its config and the configuration register is now 0x102:

```txt
3850-48t-lab-0#sh ver | inc register
Configuration register is 0x102
```

If you loaded the startup config and then `wr`'d during troubleshooting, you may need to regenerate your SSH host keys with `crypto key generate rsa mod x`.

You can determine if this is necessary by quickly reviewing the `show ip ssh` output:

```txt
3850-48t-lab-0#sh ip ssh
SSH Enabled - version 1.99
```

Once this boot variable is set, the switch should load its startup-config properly (until you `wr erase` it again).

## Attempted fixes

- Set `config-register 0x102` and reload (no change)
- Set `config-register 0x2102` and reload (no change)
- Unset `SWITCH_IGNORE_STARTUP_CFG` in rommon (I did not set `SWITCH_IGNORE_STARTUP_CFG` first, this may be why this did not work)