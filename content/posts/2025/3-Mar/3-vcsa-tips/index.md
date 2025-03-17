---
title: "vCenter Server Appliance - regen certs, recover PWs, other tips and tricks"
date: 2025-03-16T11:12:59-00:00
draft: false
---

## Enabling SSH

Consider this a prerequisite for everything that follows.

If SSH was not enabled while you set up the VCSA (it's a toggle labeled "Enable SSH" with a tooltip about being required for HA - you probably left it off) you can enable it via the vCenter Server Management UI at port 5480.

Log on to the management UI. In the left menu, select "Access".

Click "Edit" in the top right of the screen, toggle "Activate SSH Login", then click "OK".

{{< figure src="images/VCSAAccessEdit.png" alt="vCenter Management UI Access settings" >}}

{{< figure src="images/VCSAActivateSSH.png" alt="vCenter Management UI Access settings - edit view" >}}

You should now be able to SSH to the VCSA on port 22:

```txt
~
❯ ssh root@vcenter.lab.wporter.org

VMware vCenter Server 8.0.3.00400

Type: vCenter Server with an embedded Platform Services Controller

(root@vcenter.lab.wporter.org) Password:
Last login: Sun Mar 16 22:42:41 2025 from 192.168.77.250
Connected to service

    * List APIs: "help api list"
    * List Plugins: "help pi list"
    * Launch BASH: "shell"

Command>
```

## Configuring addressing with vami_config_net

To reconfigure networking (e.g., force the VCSA to renew its DHCP lease), you can use `/opt/vmware/share/vami/vami_config_net` from the Bash shell.

```txt
root@vcenter [ ~ ]# /opt/vmware/share/vami/vami_config_net

 Main Menu

0)      Show Current Configuration (scroll with Shift-PgUp/PgDown)
1)      Exit this program
2)      Default Gateway
3)      Hostname
4)      DNS
5)      Proxy Server
6)      IP Address Allocation for eth0
Enter a menu number [0]: 0

Network Configuration for eth0
IPv4 Address:   192.168.77.220
Netmask:        255.255.255.0
IPv6 Address:
Prefix:

Global Configuration
IPv4 Gateway:   192.168.77.1
IPv6 Gateway:
Hostname:       vcenter.lab.wporter.org
DNS Servers:    127.0.0.1, 192.168.77.1
Domain Name:
Search Path:    lab.wporter.org
Proxy Server:

 Main Menu

0)      Show Current Configuration (scroll with Shift-PgUp/PgDown)
1)      Exit this program
2)      Default Gateway
3)      Hostname
4)      DNS
5)      Proxy Server
6)      IP Address Allocation for eth0
Enter a menu number [0]: 6
Type Ctrl-C to go back to the Main Menu

Configure an IPv6 address for eth0? y/n [n]: n
Configure an IPv4 address for eth0? y/n [n]: y
Use a DHCPv4 Server instead of a static IPv4 address? y/n [n]: y
IPv4 Address:   AUTOMATIC
Netmask:        AUTOMATIC

Is this correct? y/n [y]: y

Reconfiguring eth0...
net.ipv6.conf.eth0.disable_ipv6 = 1
Network parameters successfully changed to requested values

 Main Menu

0)      Show Current Configuration (scroll with Shift-PgUp/PgDown)
1)      Exit this program
2)      Default Gateway
3)      Hostname
4)      DNS
5)      Proxy Server
6)      IP Address Allocation for eth0
Enter a menu number [0]: 0

Network Configuration for eth0
IPv4 Address:   192.168.77.159
Netmask:        255.255.255.0
IPv6 Address:
Prefix:

Global Configuration
IPv4 Gateway:   192.168.77.1
IPv6 Gateway:
Hostname:       vcenter.lab.wporter.org
DNS Servers:    127.0.0.1, 192.168.77.1
Domain Name:
Search Path:    lab.wporter.org
Proxy Server:
```

## Renewing or regenerating VMCA self-signed certificates

https://knowledge.broadcom.com/external/article?legacyId=2112283

You can use the `/usr/lib/vmware-vmca/bin/certificate-manager` utility to regenerate or replace certificates (say, if they've expired or the hostname has changed) then restart vCenter services. Note that this requires an Administrator (SSO domain) password as well as the appliance root password.

## Resetting the root password without reboot (if you know a SSO password)

https://knowledge.broadcom.com/external/article/321369

## Resetting the root password without any passwords

The VCSA is a Linux appliance, so you can reset the root password by booting into single-user mode to recover the system. You'll need to interrupt the boot process by pressing the `e` key at the GRUB prompt, as shown below:

{{< figure src="images/VCSAPhotonGrub.png" alt="VCSA GRUB bootloader splash screen" >}}

Then tell GRUB to boot the machine into single-user mode by appending `rw init=/bin/bash` to the `$\systemd_cmdline` section of the boot parameters (press `ctrl` + `x` to boot the machine once this is done):

{{< figure src="images/VCSAPhotonGRUBCmdline.png" alt="VCSA GRUB editing arguments" >}}

Photon OS will boot, and dump you into a root shell. You can reset the root password with the `passwd` command, or go about repairing the system if that's what you're in here for.

{{< figure src="images/VCSASingleUser.png" alt="VCSA booted to single user mode" >}}

When you're done, restart the machine and let it boot normally.

## Resetting SSO user passwords without another SSO account

To reset a SSO password (e.g. administrator@vsphere.local), you can use the `/usr/lib/vmware-vmdir/bin/vdcadmintool`:

```txt
root@vcenter [ ~ ]# /usr/lib/vmware-vmdir/bin/vdcadmintool


==================
Please select:
0. exit
1. Test LDAP connectivity
2. Force start replication cycle
3. Reset account password
4. Set log level and mask
5. Set vmdir state
6. Get vmdir state
7. Get vmdir log level and mask
==================

3
  Please enter account UPN : administrator@vsphere.local
New password is -
@CQuEY1=YmN"4yn]51RL


==================
Please select:
0. exit
1. Test LDAP connectivity
2. Force start replication cycle
3. Reset account password
4. Set log level and mask
5. Set vmdir state
6. Get vmdir state
7. Get vmdir log level and mask
==================

0
```

## Disable SSO user password expiry

Perhaps you've just wound up locked out and want to *not* need to reset your VCSA SSO passwords again.

To do so, sign on to the vSphere Client interface.

Select the hamburger menu icon in the top left of the vSphere client, then click "Administration".

{{< figure src="images/vSphereHamburgAdmin.png" alt="vSphere Administration menu location" >}}

In the Administration menu, select "Configuration" under "Single Sign On".
The SSO configuration pane will display to the right of the menu.
Select "Local Accounts", then click "Edit" on "Password Policy".

{{< figure src="images/vSphereSSOLocalAccountConfig.png" alt="vSphere SSO Local Account configuration settings" >}}

To disable password expiry, set the "Maximum lifetime" to 0.

{{< figure src="images/vSphereSSOConfigLAPPML.png" alt="vSphere SSO Local Account configuration settings - edit" >}}