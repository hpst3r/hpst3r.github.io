---
title: "Poking at the Intel Management Engine (vPro Enterprise, AMT) for IPMI-like remote access on a M920q"
date: 2025-04-23T18:30:00-00:00
draft: false
---

I'm sure that if you're reading this post on my backwards sysadmin blog, you're at least vaguely familiar with the Intel Management Engine. For shoots and giggles, let's chat about it anyway.

Essentially, it's a tiny embedded computer (like your good old ASpeed BMC) that's tightly integrated with almost any recent (2008+) Intel Platform Controller Hubs (PCHs - the chipset on a modern Intel board) or on the SoC itself (for Atom mobile processors - these have an integrated Security Engine). It has the potential to give you full remote access to the system at a hardware level (potentially far greater access than a separate BMC). The ME runs its own operating system (MINIX, in this generation, though this changes every so often). It runs its own applications that do.. stuff - its full scope of access/functionality is not documented, but extra fun features have been used by malware in the past.

Of course this has some security and privacy implications. We won't really get into those here, but if you're interested, here's some further reading:
- [Here's a paper by Dmitriy Evdokimov on the exploitation of CVE-2017-5689](https://www.blackhat.com/docs/us-17/thursday/us-17-Evdokimov-Intel-AMT-Stealth-Breakthrough-wp.pdf), by which a malicious actor can enable AMT on many systems (including those that don't have vPro!) and use it to hide their C&C network traffic.
- [Here's a letter to Intel from Dr. Andrew Tanenbaum](https://www.cs.vu.nl/~ast/intel/), the author of MINIX, which highlights his concerns about the Management Engine.

Some PCs have a bit of the ME's functionality exposed as Intel Active Management Technology (AMT) for the benefit of system administrators (like ourselves). You can tell if you have one of these because they'll typically have a sticker advertising vPro. They tend to be higher-end office PCs with Q-series Intel mainboards, and some OEMs gate vPro settings behind an additional option (Dell - ME Disable vs ME Enable models of the 7020) or a specific model with little else to differentiate it (Lenovo - M920q and M720q). The M920qs are nice because they generally all support vPro.

These PCs have slightly special network cards (like the Intel I219) that enable separate network access for the ME. This allows the ME to work wholly independently of the host, sharing the NIC by either using a second MAC and second IP or stealing a few TCP ports from the host by taking advantage of hardware features in the NIC.

Again, the Management Engine is separate from the main CPU on an Intel platform. It's very similar to a traditional server BMC in that it works independently of the host system's power state, has its own networking, and runs its own OS - this BMC just happens to be built into the chipset and primary NIC in the system. Like a server BMC, the ME allows you to set the power state of the host system, jump into a remote console, 

What this means is that this can allow us to use an out-of-band management tool that wouldn't be out of place on a server on our old e-wasted office PCs, which is great. As a brief aside, AMD has their own alternative called DASH that we may take a look at in the future (because I do have a M715q kicking around).

I would HIGHLY recommend updating your ME firmware before getting started; there are quite a few severe vulnerabilities kicking about in older versions of ME code. It's probably easiest to do this by booting Windows and running Lenovo's firmware update utility for your platform.

For example, on a Lenovo PC, you can download and run (in Windows) the [Corporate Intel ME FW Update Utility](https://pcsupport.lenovo.com/us/en/products/desktops-and-all-in-ones/thinkcentre-m-series-desktops/thinkcentre-m920q/downloads/ds505029-corporate-intel-management-engine-firmware-update-tool-for-intel-cfl-platform-thinkcentre-thinkstation?category=Motherboard%20Devices%20%28Backplanes,%20core%20chipset,%20onboard%20video,%20PCIe%20switches%29) for your platform (link is for the M920q). I updated mine to 12.0.90-build 2072, published 2022, as it was the most recent firmware available at the time of writing (4/2025).

Assuming that's done..

### Enabling AMT

Let's enable this on my M920q and have a poke at it. The following screenshots are taken from AMT because it's easier than trying to make a photo of a screen look presentable.

First, let's see if AMT is enabled, and reset it if needed.

Enter the BIOS by pressing Enter, then F1. Navigate to Advanced > Intel Manageability.

Confirm that "Intel(R) Manageability Control" and "Press <Ctrl-P to Enter MEBx" are Enabled.

{{< figure src="images/amt-lenovo-uefi-me-control.jpg" >}}

If you do not have the Intel ME password and it has been changed from the default, reset it by enabling the "Intel(R) Manageability Reset" option. Reboot and approve the unprovision activity.

{{< figure src="images/amt-lenovo-uefi-me-reset.jpg" >}}

You should now be able to enter the AMT menu by pressing Ctrl + P at boot time. Log in with the default password ("admin"), and change it when prompted. I believe that the default behavior in the ME 12.x on this Q370 is to get itself a DHCP address that's shared with the host, but I set mine to get a unique IP instead.

There's a default webpage you can use to configure AMT on port 16992:

{{< figure src="images/amt-default-webpage.png" >}}

{{< figure src="images/amt-default-webpage-systeminfo.png" >}}

But it's not all that useful, past configuring very basic networking (static or dynamic IP). There's no way to use it for a remote console session, which is kind of what I'm after.

{{< figure src="images/amt-default-webpage-powerstate.png" >}}

There are two main ways we can get IPMI-like functionality from AMT.

## Using client software

Install [MeshCommander](https://www.meshcommander.com/) (still supported) or download and run [MeshMini](https://github.com/brytonsalisbury/mesh-mini) (portable version of the former that runs in a web browser). I'll be doing this on a Windows 11 24H2 system.

These are more or less the same program, so you can use either.

{{< figure src="images/amt-meshcommander-meshmini-sidebyside.png" >}}

To use MeshCommander, add a computer with your credentials and go to town. Bear in mind that said credentials will be floating around on the wire if you have not set up TLS.

Here's a couple examples of the MeshCommander interface connected to my M920q, for an idea of what you can get up to with it. It's pretty straightforward.

{{< figure src="images/amt-meshcommander-installing-win11.png" >}}

{{< figure src="images/amt-meshcommander-hwinfo.png" >}}

{{< figure src="images/amt-meshcommander-netconfig.png" >}}

## Loading MeshCommander into AMT firmware

The [MeshCommander Firmware Loader](https://www.meshcommander.com/meshcommander/firmware) can help you skip installing the MeshCommander application on a separate machine. By loading MeshCommander into the ME's local storage (using MeshCommander instead of the default AMT page), you can use the ME's built-in webserver to serve the remote desktop applet.

Note that you need a reasonably recent AMT firmware revision to do this - the original build on this M920q (from 2017) was too old. If you have not already updated the ME firmware on your device (you should have, because old versions have a whole slew of security issues) you will need to do so.

To configure this, download and run the utility. Provide credentials for your ME, and click through the handful of options.

{{< figure src="images/amt-meshcommander-firmware-1.png" >}}

{{< figure src="images/amt-meshcommander-firmware-2.png" >}}

{{< figure src="images/amt-meshcommander-firmware-3.png" >}}

{{< figure src="images/amt-meshcommander-firmware-4.png" >}}

{{< figure src="images/amt-meshcommander-firmware-5.png" >}}

You should now be able to access the MeshCommander interface via the AMT's internal webserver on the default port 16992.

{{< figure src="images/amt-meshcommander-firmware-webpage.png" >}}

You can now access a VNC console or serial console from the device without local prerequisites! Works well enough across a VPN, too.

To revert the change, you can run the MeshCommander Firmware utility again, selecting "Remove MeshCommander" at the "select an operation" prompt.

{{< figure src="images/amt-meshcommander-firmware-6.png" >}}

Easy as pie!

## Networking Configuration

The AMT is a bit funky about its networking. It will generally share the host's addressing. While this is configured, the MAC that AMT uses matches the host's vPro-enabled NIC (in this case, the built-in i219) and AMT will 'borrow' tcp/16992 through tcp/16995 from the vPro-enabled NIC (any traffic sent to 16992, 16993, 16994, or 16995 is diverted to the ME).

If you were to configure a separate IP, according to [Intel's documentation](https://meshcentral.com/meshcommander/ActivePlatformManagementDemystified/APMD-Chapter11.pdf):

> On most versions of Intel AMT, if set to static IP mode, Intel AMT uses a separate MAC address and IP address for out-of-band communication. As a result, inbound TCP connections on ports 16992 through 16995 on the operating system’s IP address will still work correctly.

I was unable to replicate this - my AMT on this M920q appears to be stuck in 'sync' mode. The below image is inaccurate - I'm unable to change anything.

{{< figure src="images/amt-meshcommander-netconfig-dialog.png" >}}

And, finally, the apparent lack of any ability to set the AMT interface to use a tagged VLAN means it's going to be quite difficult (or impossible) to effectively isolate your AMT interfaces to a management network.

## Certificates/basic encryption

Never properly got this working with MeshCommander. I imagine I probably need trusted certificates for this. I'll revisit this at a later time.

You cannot manage certificates from the minified version of MeshCommander loaded into AMT storage/firmware. To configure HTTPS with MeshCommander/AMT, you'll need to access the computer from a full MeshCommander server (e.g., mesh-mini or MeshCommander running on a desktop PC).

To configure TLS and enable HTTPS, we'll need to add a certificate. Connect to the PC via a freestanding MeshCommander server, navigate to Security Settings.

{{< figure src="images/amt-meshcommander-securitysettings.png" >}}

Issue a self-signed certificate or add a certificate. To issue a self-signed certificate, click "Issue Certificate" and fill in the CN, Organization, State/Province and Country fields. All you need is a TLS Server certificate.

{{< figure src="images/amt-meshcommander-issuecert.png" >}}

Once the certificate has been created, it'll appear in the "Manage Certificates" menu area.

{{< figure src="images/amt-meshcommander-issuedcert.png" >}}

You can now configure TLS by clicking the 'Disabled' link next to 'remote' or 'local' TLS, and enabling TLS, selecting a certificate. You can configure either server- or mutual-auth TLS here; we won't be getting into anything past server-auth with self-signed certificates right now.

You may have to power-cycle the host (unplug and plug back in to reset the ME) for the web server to start, well, serving HTTPS on port 16993. Mine was unhappy with me when I attempted to get it to do so; I had to roll back to the default AMT webpage and physically unplug and reset the host to get the webserver listening on 16993.

The MeshCommander Firmware utility fails to connect to the ME on this machine after the self-signed certificate was configured, so I shelved it here for the time being. I will revisit this at a later date.

{{< figure src="images/amt-meshcommander-selfsigned.png" >}}

## Hardware lacking vPro

There's potential that vPro can, to some degree, be enabled on unsupported hardware - according to some chatter around CVE-2017-5689, it's possible to enable vPro on unsupported CPUs/boards. However, with newer versions of vPro, this likely requires BIOS modification or cleverness that I don't possess and is going to be difficult on Dell/HP OEM PCs that don't use a standard AMI BIOS. Additionally, I've no idea how this would work with a Realtek NIC.

[Intel source - AMT security best practice PDF](https://cdrdv2-public.intel.com/840853/Intel_AMT_Security_Best_Practices_QA.pdf)

> Q6. Is Intel AMT disabled by default on Intel vPro platforms? If not, can it be disabled or have any default passwords changed by end users not part of the IT-supported network? A6. Depending on the customers’ request, **Intel vPro platforms can be delivered by OEMs in one of the following Intel AMT states: un-provisioned and ON in BIOS, un-provisioned and OFF in BIOS, and un-provisioned and permanently OFF in BIOS.** The Intel AMT firmware does not search for a configuration server when being plugged in the network for the first time, this capability was removed in AMT 6.0 and later platforms. Intel recommends when receiving a new Intel AMT capable platform with Intel AMT turned ON in the BIOS, that the default password be changed and Intel AMT be provisioned.