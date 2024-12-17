---
title: "Updating ESXi 8 hosts from a depot file"
date: 2024-07-05T12:34:56-00:00
draft: false
---

Personal notes - I usually use Lifecycle Manager because it's easy.

Download the depot file from somewhere (of course I mean the Broadcom portal, you dirty pirate.)

Upload the depot file to a host datastore of some kind, either through the web UI browser or with SCP.

```txt
user@pc:~$ scp VMware-ESXi-8.0U3-24022510-depot.zip user@hv0.mydomain.internal:/vmfs/volumes/<datastore>
```

You can either use esxcli remotely, or ssh into the host.

Determine which profiles are available from the depot with esxcli once it's on accessible storage.

```txt
[user@hvx:~] esxcli software sources profile list --depot=/vmfs/volumes/<datastore>/VMware-ESXi-8.0U3-24022510-depot.zip
Name                          Vendor        Acceptance Level  Creation Time        Modification Time
----------------------------  ------------  ----------------  -------------------  -----------------
ESXi-8.0U3-24022510-no-tools  VMware, Inc.  PartnerSupported  2024-06-11T13:40:11  2024-06-11T13:40:11
ESXi-8.0U3-24022510-standard  VMware, Inc.  PartnerSupported  2024-06-11T13:40:11  2024-06-11T13:40:11
```

If you haven't already done so, enter maintenance mode.

```txt
[user@hvx:~] esxcli system maintenanceMode set --enable true
```

Update the host. Append `--dry-run` if you want to preview the result and see if a reboot is required, and append `--no-hardware-warning` if you're getting warnings (e.g., future unsupported CPUs, like.. Skylake! Yay.)

```txt
[user@hvx:~] esxcli software profile update --depot=/vmfs/volumes/<datastore>/VMware-ESXi-8.0U3-24022510-depot.zip --profile=ESXi-8.0U3-24022510-standard
```

Once the update has been installed, reboot the host.

```txt
[user@hvx:~] esxcli system shutdown reboot --reason="System update to 8.0U3"
```

Confirm that the new profile is active.

```txt
[user@hvx:~] esxcli software profile get
```

I noticed that the output of `software profile get` has a little bug - maybe Broadcom laid off the person who wrote the version descriptions.

```txt
----------
      The general availability release of VMware ESXi Server 8.0U2
      brings whole new levels of virtualization performance to
      datacenters and enterprises.
```