---
title: "PowerShell Core (Microsoft Graph and Az) in a toolbox container on your Linux box"
date: 2025-08-24T22:00:00-00:00
draft: false
---

If you're like me (god, for your sake I hope you aren't) you love PowerShell and use Linux.

I'm also super lazy. And I don't like Microsoft adding 5,000 repositories to my system to keep PowerShell updated. The solution? A container!

Toolbox lets you very easily jump into a podman container for a development environment. In this container, you can typically install whatever you'd need as if it was on a bare metal install - no special anything required. Since all I need is PowerShell, we'll keep things simple.

First, create a Toolbox container:

```txt
wporter@wt14sg1a:~$ toolbox create pwsh
Image required to create Toolbx container.
Download registry.fedoraproject.org/fedora-toolbox:42 (367.9MB)? [y/N]: y
Created container: pwsh
Enter with: toolbox enter pwsh
```

Follow the helpful instructions and enter your container:

```txt
wporter@wt14sg1a:~$ toolbox enter pwsh

Welcome to the Toolbx; a container where you can install and run
all your tools.

 - Use DNF in the usual manner to install command line tools.
 - To create a new tools container, run 'toolbox create'.

For more information, see the documentation.

⬢ [wporter@toolbx ~]$ 
```

Then, install PowerShell! I'll be using the 'direct download' method ([see docs at learn.microsoft.com](https://learn.microsoft.com/en-us/powershell/scripting/install/install-rhel?view=powershell-7.5#installation-via-direct-download)).

```txt
⬢ [wporter@toolbx ~]$ sudo dnf install https://github.com/PowerShell/PowerShell/releases/download/v7.5.2/powershell-7.5.2-1.rh.x86_64.rpm
Updating and loading repositories:
Repositories loaded.
 https://github.com/PowerShell/PowerShe 100% |  23.9 MiB/s |  71.5 MiB |  00m03s
Package                 Arch   Version                  Repository          Size
Installing:
 powershell             x86_64 7.5.2-1.rh               @commandline   183.4 MiB
Installing dependencies:
 libicu                 x86_64 76.1-4.fc42              fedora          36.3 MiB

Transaction Summary:
 Installing:         2 packages

Total size of inbound packages is 82 MiB. Need to download 11 MiB.
After this operation, 220 MiB extra will be used (install 220 MiB, remove 0 B).
Is this ok [y/N]: y
[1/1] libicu-0:76.1-4.fc42.x86_64       100% |   9.0 MiB/s |  10.7 MiB |  00m01s
--------------------------------------------------------------------------------
[1/1] Total                             100% |   7.9 MiB/s |  10.7 MiB |  00m01s
Running transaction
[1/4] Verify package files              100% |   5.0   B/s |   2.0   B |  00m00s
[2/4] Prepare transaction               100% |   9.0   B/s |   2.0   B |  00m00s
[3/4] Installing libicu-0:76.1-4.fc42.x 100% | 135.6 MiB/s |  36.3 MiB |  00m00s
[4/4] Installing powershell-0:7.5.2-1.r 100% |  73.6 MiB/s | 183.6 MiB |  00m02s
Warning: skipped OpenPGP checks for 1 package from repository: @commandline
Complete!
⬢ [wporter@toolbx ~]$ pwsh
PowerShell 7.5.2
PS /home/wporter> whoami
wporter
```

Now, let's install the Azure, and Microsoft Graph Authentication modules.

If you'd like to not see the 'do you trust this repository' prompts, mark PSGallery as a trusted module repository with `Set-PSRepository`:

```PowerShell
Set-PSRepository PSGallery -InstallationPolicy Trusted
```

The Azure rollup module is called "`Az`". Install it with:

```PowerShell
Install-Module Az
```

To connect to your Azure account, use the `Connect-AzAccount` cmdlet. You'll have to use Device Authentication since your container can't open a browser:

```txt
PS /home/wporter> Connect-AzAccount -UseDeviceAuthentication
WARNING: You may need to login again after updating "EnableLoginByWam".
Please select the account you want to login with.

[Login to Azure] To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code AABBCCDDE to authenticate.
Retrieving subscriptions for the selection...
[Tenant and subscription selection]

No      Subscription name                       Subscription ID                             Tenant name                   
----    ------------------------------------    ----------------------------------------    --------------------------
[1]     Azure subscription 1                    00000000-0000-0000-0000-000000000000        Default Directory         
[2]     Concierge Subscription                  00000000-0000-0000-0000-000000000000        Microsoft Learn Sandbox   

Select a tenant and subscription: 1

Subscription name    Tenant
-----------------    ------
Azure subscription 1 Default Directory
```

Now you can use Azure PowerShell commands! How nice!

To install the Microsoft Graph Authentication module (one part of the Microsoft.Graph rollup module - just a wrapper for authentication/authorization when making API requests), it's one command:

```PowerShell
Install-Module Microsoft.Graph.Authentication
```

To connect, you'll need to `Connect-MgGraph` with the `-UseDeviceAuthentication` flag (just as you did with the Azure module).

When you're done with PowerShell, just `exit` the container or close your terminal. When you need your container again, `toolbox enter pwsh` to jump right back in!
