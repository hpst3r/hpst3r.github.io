---
title: "Changing vCenter Server DHCP address"
date: 2025-03-16T10:12:59-00:00
draft: false
---

# Problem

You would like to change the IP address for a vCenter Server appliance that is currently configured with DHCP by modifying the reservation.

Doing this the way you might first think of (by changing the reservation then forcing the VCSA to get a new IP) fails, because you must regenerate certificates afterwards and cannot do so when the SSO authentication service is not working (after an IP change).

# Solution

Tested with a VCSA ver 8.0.3.00000 on 16 Mar 2025. Fresh install, not a neglected appliance!

Make the change from the web UI by setting the VCSA's IP configuration to your new reservation as a static IP, then changing the interface settings back to DHCP.

Log on to the vCenter management UI at port 5480. Navigate to Networking. Click "Edit" in the top right corner of the "Network Settings" section.

{{< figure src="images/VCSA-Networking-Edit.png" alt="vCenter Server management UI Network pane with the edit button highlighted" >}}

Reconfigure your DNS settings so that your VCSA's hostname resolves properly to the new IP you'd like to use.

You could set this to a different IP in the meantime (not what you will eventually use as a reservation), but this will force the VCSA to regenerate all of its certificates when you switch back to DHCP (which will add another long wait) and there's no reason (that I can think of) to do this.

In my case, with a dnsmasq DHCP/DNS server, I did this by editing /etc/hosts and my reservations file under /etc/dnsmasq.d, then restarted dnsmasq to apply the changes. Obviously, this will differ with your flavor of DNS server.

Select the network adapter you would like to modify, then select "Enter DNS settings manually" and "Enter IPv4 settings manually" (or IPv6, if you so desire). Enter your desired network configuration (the new reservation) manually, then click Next.

{{< figure src="images/VCSA-Networking-NIC-Settings.png" alt="vCenter Management UI network NIC settings showing the config of a static IP" >}}

Provide your SSO credentials.

{{< figure src="images/VCSA-I-Promise-I-Made-A-Backup.png" alt="vCenter Management UI network reconfig final approval options" >}}

Check the "I acknowledge that I have made a backup" box (you have, haven't you?), then hit "Finish".

Wait for the VCSA to reconfigure itself. This may take some time. As directed, do not refresh the page. You may see an untrusted cert error when you reconnect to the VCSA with its new certificates.

Once the VCSA has sorted itself out with its new static IP, reconfigure your DHCP reservation, if you have not done this already.

When your reservation is sorted, return to the VCSA Management web UI's Networking settings. Click Edit on Network Settings again, and select the network adapter you rearranged.

Switch your NIC's IP settings to "automatic". Optionally, set your DNS settings to "automatic" as well.

{{< figure src="images/VCSA-NIC-DHCP.png" alt="vSphere Management UI NIC settings with DHCP options selected" >}}

Click "Next", provide SSO credentials again, then check the "I've made a backup" box and click "Finish".

Expect a very short wait. The operation will complete, and your VCSA will be at the new IP and using DHCP.

If, upon attempting to access your newly-moved VCSA, you receive a "vSphere Client service has stopped working" error, you can SSH to the VCSA and restart its services with `service-control --restart --all` or reboot the box.

{{< figure src="images/VCSA-VerySad.png" alt="vSphere Client has stopped working error message" >}}

