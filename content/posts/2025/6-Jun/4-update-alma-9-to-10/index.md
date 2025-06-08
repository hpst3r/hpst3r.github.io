---
title: "Performing an in-place upgrade from Alma 9 to Alma 10 with LEAPP and ELevate"
date: 2025-06-07T20:00:00-00:00
draft: false
---

That time of year again! Time to break all my stuff!

## x86-64-v2 systems

Quick aside about x86-64-v2 systems (generally these are processors Ivy Bridge and earlier - 3000 series Intel, E3/E5/E7 v2 or older).

EL10 does NOT normally support older processors that do not implement AVX2.

Most notably, this excludes some fairly recent embedded/low power SKUs (any Intel "small cores" prior to [Gracemont](https://en.wikipedia.org/wiki/Gracemont_(microarchitecture)), introduced in 2021, like [Tremont](https://en.wikipedia.org/wiki/Tremont_(microarchitecture)) in the Lakefield or Jasper Lake products that were sold until quite recently) and Ivy Bridge (the Intel Core 3rd generation SKUs, or Xeon E3/E5/E7 v2 chips like the E5-2680 v2) processors.

This means that anything older than [Gracemont](https://en.wikipedia.org/wiki/Gracemont_(microarchitecture)) (Alder Lake E-cores), [Haswell](https://en.wikipedia.org/wiki/Haswell_(microarchitecture)) (4th generation Core, Xeon E3/E5/E7 v3), or [AMD Excavator](https://en.wikipedia.org/wiki/Excavator_(microarchitecture)) (direct predecessor to Zen, so Ryzen or Epyc CPUs all support v3) WILL NOT run mainstream EL10.

The only way to get EL10 running on them at present is with the special x86-64-v2 build of AlmaLinux 10.

Unfortunately, at the time of writing, ELevate does NOT support updating from AlmaLinux 9 to the x86-64-v2 version of AlmaLinux 10.

An attempt to update an unsupported (x86-64-v2) system with LEAPP will alert you with an "inhibitor" class failure in the precheck stage because the hardware is unsupported:

```txt
============================================================
                      REPORT OVERVIEW
============================================================

Upgrade has been inhibited due to the following problems:
    1. Current x86-64 microarchitecture is unsupported in RHEL10
```

At present, there is no way around this. The only 'supported' way to get EL10 on a v2 system is to reinstall the machine.

To see if your system is affected (does not support the x86-64-v3 instruction set) you can check `/proc/cpuinfo` for the presence of the avx2 compatibility flag:

```sh
grep -m1 'model name' /proc/cpuinfo; \
grep -q avx2 /proc/cpuinfo && \
echo Supports AVX2 || echo No AVX2
```

This quick snippet will list the model of your processor(s) and whether they do or do not support AVX2, which generally implies that it's v3. For example, here's an i5-8600T, that does support AVX2 (Coffee Lake, 8th generation Intel Core):

```sh
[wporter@m920q1 ~]$ grep -m1 'model name' /proc/cpuinfo; \
grep -q avx2 /proc/cpuinfo && \
echo Supports AVX2 || echo No AVX2
model name      : Intel(R) Core(TM) i5-8600T CPU @ 2.30GHz
Supports AVX2
```

Here's an E5-2680 v2 (Ivy Bridge, 3rd generation Intel Core):

```sh
[wporter@ELevate-el9-el10-v2-test ~]$ grep -m1 'model name' /proc/cpuinfo; \
grep -q avx2 /proc/cpuinfo && \
echo Supports AVX2 || echo No AVX2
model name      : Intel(R) Xeon(R) CPU E5-2680 v2 @ 2.80GHz
No AVX2
```

Again, if your system is x86-64-v2 or older, it WILL NOT be able to upgrade to AlmaLinux 10 from AlmaLinux 9. You might have to wait a little longer for the ELevate project to extend support for in-place updates. Anyway, back to the good stuff.

## Update procedure

See [the AlmaLinux Wiki guide here](https://wiki.almalinux.org/elevate/ELevating-CentOS7-to-AlmaLinux-10.html#upgrading-almalinux-9-to-almalinux-10).

### TL:DR

In a nutshell:

Update the system.

```sh
sudo dnf update -y
```

Restart if a kernel update was applied.

```sh
reboot now
```

Add the ELevate repository (`https://repo.almalinux.org/elevate/elevate-release-latest-el$(rpm --eval %rhel).noarch.rpm`).

```sh
sudo dnf install -y https://repo.almalinux.org/elevate/elevate-release-latest-el$(rpm --eval %rhel).noarch.rpm
```

Install the `leapp-upgrade` and `leapp-data-almalinux` packages from the ELevate repository.

```sh
sudo dnf install -y leapp-upgrade leapp-data-almalinux
```

Run a `leapp preupgrade` check to validate the system configuration and state (for, hopefully, a successful upgrade).

```sh
sudo leapp preupgrade
```

Remediate any blockers identified by the `leapp preupgrade`.

Update the system.

```sh
sudo leapp upgrade
```

Reboot the system and wait a short while for the offline update.

```sh
reboot now
```
  
(Hopefully) reconnect to your AlmaLinux 10 system.

Clean up the old kernel, LEAPP/ELevate packages, and anything else lingering around.

List lingering packages:

```sh
rpm -qa | grep el9
```

Remove the old kernel:

```sh
sudo dnf remove -y kernel-modules-core-5.14.0
```

Remove ELevate and LEAPP packages that can be removed with `dnf`:

```sh
sudo dnf remove -y elevate-release leapp-data-almalinux
```

Remove update packages that cannot be removed with `dnf`:

```sh
sudo rpm -e leapp python3-leapp leapp-upgrade-el9toel10
```

Remove any other packages that are left over on your system that were not left over on mine.

Once you're done, confirm that everything is gone:

```sh
rpm -qa | grep el9
```

Get rid of any orphaned dependencies that are no longer needed:

```sh
sudo dnf autoremove
```

Clear the `dnf` cache, which will have a bunch of stuff in it after the update:

```sh
sudo dnf clean all
```

### Introduction

I would probably recommend rebuilding instead of updating if your IaC game is in shape. This would let you test things at your own pace and easily roll back if necessary.

I won't be doing a rebuild for my hypervisors because it's a bit of a pain to reinstall them and I'm lazy. I will be doing a rebuild for most of my VMs.

Anyway, here's a step-by-step with console output.

### Guide

First, fully update the system. If you have nothing to install, great! If you do have things to install, install them, then reboot if necessary to apply any kernel updates.

```txt
[wporter@3060t0 ~]$ sudo dnf update -y
Last metadata expiration check: 0:01:10 ago on Sat Jun  7 13:56:34 2025.
Dependencies resolved.
Nothing to do.
Complete!
```

Next, add the "`elevate-release-latest-el$(rpm --eval %rhel).noarch.rpm`" repo (this will be the el9 release for Alma 9, obviously).

```txt
[wporter@3060t0 ~]$ sudo dnf install -y http://repo.almalinux.org/elevate/elevate-release-latest-el$(rpm --eval %rhel).noarch.rpm
Last metadata expiration check: 0:01:17 ago on Sat Jun  7 13:56:34 2025.
elevate-release-latest-el9.noarch.rpm                                                   144 kB/s | 9.7 kB     00:00
Dependencies resolved.
========================================================================================================================
 Package                         Architecture           Version                      Repository                    Size
========================================================================================================================
Installing:
 elevate-release                 noarch                 1.0-2.el9                    @commandline                 9.7 k

Transaction Summary
========================================================================================================================
Install  1 Package

Total size: 9.7 k
Installed size: 3.4 k
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                1/1
  Installing       : elevate-release-1.0-2.el9.noarch                                                               1/1
  Verifying        : elevate-release-1.0-2.el9.noarch                                                               1/1

Installed:
  elevate-release-1.0-2.el9.noarch

Complete!
```

Then, we can install the `leapp` packages:

```txt
[wporter@3060t0 ~]$ sudo dnf install -y leapp-upgrade leapp-data-almalinux
ELevate                                                                                 311 kB/s |  55 kB     00:00
Dependencies resolved.
========================================================================================================================
 Package                                Architecture     Version                              Repository           Size
========================================================================================================================
Installing:
 leapp-data-almalinux                   noarch           0.9-3.el9.20250505                   elevate             527 k
 leapp-upgrade-el9toel10                noarch           1:0.22.0-3.el9.elevate.1             elevate             966 k
Installing dependencies:
 leapp                                  noarch           0.19.0-2.el9                         elevate              25 k
 leapp-deps                             noarch           0.19.0-2.el9                         elevate             7.3 k
 leapp-upgrade-el9toel10-deps           noarch           1:0.22.0-3.el9.elevate.1             elevate              20 k
 libdb-utils                            x86_64           5.3.28-55.el9                        appstream           138 k
 pciutils                               x86_64           3.7.0-7.el9                          baseos               92 k
 python3-leapp                          noarch           0.19.0-2.el9                         elevate             186 k
 python3-pyudev                         noarch           0.22.0-6.el9                         baseos               76 k

Transaction Summary
========================================================================================================================
Install  9 Packages

Total download size: 2.0 M
Installed size: 34 M
Downloading Packages:
(1/9): leapp-0.19.0-2.el9.noarch.rpm                                                    266 kB/s |  25 kB     00:00
(2/9): leapp-deps-0.19.0-2.el9.noarch.rpm                                                75 kB/s | 7.3 kB     00:00
(3/9): leapp-upgrade-el9toel10-deps-0.22.0-3.el9.elevate.1.noarch.rpm                   396 kB/s |  20 kB     00:00
(4/9): leapp-data-almalinux-0.9-3.el9.20250505.noarch.rpm                               3.4 MB/s | 527 kB     00:00
(5/9): leapp-upgrade-el9toel10-0.22.0-3.el9.elevate.1.noarch.rpm                        8.3 MB/s | 966 kB     00:00
(6/9): python3-leapp-0.19.0-2.el9.noarch.rpm                                            2.8 MB/s | 186 kB     00:00
(7/9): libdb-utils-5.3.28-55.el9.x86_64.rpm                                             1.7 MB/s | 138 kB     00:00
(8/9): pciutils-3.7.0-7.el9.x86_64.rpm                                                  1.3 MB/s |  92 kB     00:00
(9/9): python3-pyudev-0.22.0-6.el9.noarch.rpm                                           1.0 MB/s |  76 kB     00:00
------------------------------------------------------------------------------------------------------------------------
Total                                                                                   3.8 MB/s | 2.0 MB     00:00
ELevate                                                                                 3.0 MB/s | 3.1 kB     00:00
Importing GPG key 0x81B961A5:
 Userid     : "ELevate <packager@almalinux.org>"
 Fingerprint: 74E7 F249 EE69 8A4D ACFB 48C8 4297 85E1 81B9 61A5
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-ELevate
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                1/1
  Installing       : python3-pyudev-0.22.0-6.el9.noarch                                                             1/9
  Installing       : pciutils-3.7.0-7.el9.x86_64                                                                    2/9
  Installing       : libdb-utils-5.3.28-55.el9.x86_64                                                               3/9
  Installing       : leapp-upgrade-el9toel10-deps-1:0.22.0-3.el9.elevate.1.noarch                                   4/9
  Installing       : leapp-deps-0.19.0-2.el9.noarch                                                                 5/9
  Installing       : python3-leapp-0.19.0-2.el9.noarch                                                              6/9
  Installing       : leapp-0.19.0-2.el9.noarch                                                                      7/9
  Installing       : leapp-upgrade-el9toel10-1:0.22.0-3.el9.elevate.1.noarch                                        8/9
  Installing       : leapp-data-almalinux-0.9-3.el9.20250505.noarch                                                 9/9
  Running scriptlet: leapp-data-almalinux-0.9-3.el9.20250505.noarch                                                 9/9
  Verifying        : leapp-0.19.0-2.el9.noarch                                                                      1/9
  Verifying        : leapp-data-almalinux-0.9-3.el9.20250505.noarch                                                 2/9
  Verifying        : leapp-deps-0.19.0-2.el9.noarch                                                                 3/9
  Verifying        : leapp-upgrade-el9toel10-1:0.22.0-3.el9.elevate.1.noarch                                        4/9
  Verifying        : leapp-upgrade-el9toel10-deps-1:0.22.0-3.el9.elevate.1.noarch                                   5/9
  Verifying        : python3-leapp-0.19.0-2.el9.noarch                                                              6/9
  Verifying        : libdb-utils-5.3.28-55.el9.x86_64                                                               7/9
  Verifying        : pciutils-3.7.0-7.el9.x86_64                                                                    8/9
  Verifying        : python3-pyudev-0.22.0-6.el9.noarch                                                             9/9

Installed:
  leapp-0.19.0-2.el9.noarch                                     leapp-data-almalinux-0.9-3.el9.20250505.noarch
  leapp-deps-0.19.0-2.el9.noarch                                leapp-upgrade-el9toel10-1:0.22.0-3.el9.elevate.1.noarch
  leapp-upgrade-el9toel10-deps-1:0.22.0-3.el9.elevate.1.noarch  libdb-utils-5.3.28-55.el9.x86_64
  pciutils-3.7.0-7.el9.x86_64                                   python3-leapp-0.19.0-2.el9.noarch
  python3-pyudev-0.22.0-6.el9.noarch

Complete!
```

Run the pre-upgrade checks. I've truncated this because it's quite long. If it's unsuccessful, `leapp preupgrade` will spit out a failure report:

```txt
[wporter@3060t0 ~]$ sudo leapp preupgrade
==> Processing phase `configuration_phase`
====> * ipu_workflow_config
        IPU workflow config actor
==> Processing phase `FactsCollection`
====> * vendor_repositories_mapping
        Scan the vendor repository mapping files and provide the data to other actors.
====> * system_facts
        Provides data about many facts from system.
Failed to set locale, defaulting to C.UTF-8
====> * check_enabled_vendor_repos
        Create a list of vendors whose repositories are present on the system and enabled.
====> * vendor_repo_signature_scanner
        Produce VendorSignatures messages for the vendor signature files inside the
====> * scan_vendor_repofiles
        Load and produce custom repository data from vendor-provided files.
Failed to set locale, defaulting to C.UTF-8
====> * scan_grub_config
        Scan grub configuration files for errors.

### SNIP ###

Debug output written to /var/log/leapp/leapp-preupgrade.log

============================================================
                      REPORT OVERVIEW
============================================================

Upgrade has been inhibited due to the following problems:
    1. Newest installed kernel not in use

HIGH and MEDIUM severity reports:
    1. Packages not signed by Red Hat found on the system
    2. Leapp detected loaded kernel drivers which are no longer maintained in RHEL 10.
    3. Detected custom leapp actors or files.
    4. Berkeley DB (libdb) has been detected on your system

Reports summary:
    Errors:                      0
    Inhibitors:                  1
    HIGH severity reports:       3
    MEDIUM severity reports:     1
    LOW severity reports:        2
    INFO severity reports:       1

Before continuing, review the full report below for details about discovered problems and possible remediation instructions:
    A report has been generated at /var/log/leapp/leapp-report.txt
    A report has been generated at /var/log/leapp/leapp-report.json

============================================================
                   END OF REPORT OVERVIEW
============================================================

Answerfile has been generated at /var/log/leapp/answerfile
```

Fix the issues it's raised. Or don't. The high severity issues in this installation were the presence of packages not signed by Red Hat (oh no!), old loaded kernel drivers (no idea what these were, probably Realtek something that's unused and was out-of-tree) and I have no idea what the third one means. The inhibitor (newest installed kernel not in use) was caused by a `dnf update` without a reboot.

I slapped this machine with a reboot, and ran the precheck again, which cleared that inhibitor. Then, I performed a `leapp upgrade`.

If the `leapp preupgrade` passes first try, it'll start building an environment for later chroot with EL10 packages, run a transaction check, then present a report. The full report on a plain AlmaLinux 9 system running little past Cockpit and Libvirt/QEMU/KVM looks something like this:

```txt
[wporter@m920q1 ~]$ sudo leapp preupgrade
### SNIP ###
============================================================
                      REPORT OVERVIEW
============================================================

HIGH and MEDIUM severity reports:
    1. Detected custom leapp actors or files.
    2. Leapp detected loaded kernel drivers which are no longer maintained in RHEL 10.
    3. Berkeley DB (libdb) has been detected on your system

Reports summary:
    Errors:                      0
    Inhibitors:                  0
    HIGH severity reports:       2
    MEDIUM severity reports:     1
    LOW severity reports:        3
    INFO severity reports:       1

Before continuing, review the full report below for details about discovered problems and possible remediation instructions:
    A report has been generated at /var/log/leapp/leapp-report.txt
    A report has been generated at /var/log/leapp/leapp-report.json

============================================================
                   END OF REPORT OVERVIEW
============================================================
[wporter@m920q1 ~]$ sudo cat /var/log/leapp/leapp-report.txt
Risk Factor: high
Title: Detected custom leapp actors or files.
Summary: We have detected installed custom actors or files on the system. These can be provided e.g. by third party vendors, Red Hat consultants, or can be created by users to customize the upgrade (e.g. to migrate custom applications). This is allowed and appreciated. However Red Hat is not responsible for any issues caused by these custom leapp actors. Note that upgrade tooling is under agile development which could require more frequent update of custom actors.
The list of custom leapp actors and files:
    - /usr/share/leapp-repository/repositories/system_upgrade/common/files/distro/almalinux/rpm-gpg/10/RPM-GPG-KEY-AlmaLinux-10
    - /usr/share/leapp-repository/repositories/system_upgrade/common/files/rpm-gpg/10/RPM-GPG-KEY-AlmaLinux-10
Related links:
    - Customizing your Red Hat Enterprise Linux in-place upgrade: https://red.ht/customize-rhel-upgrade
Remediation: [hint] In case of any issues connected to custom or third party actors, contact vendor of such actors. Also we suggest to ensure the installed custom leapp actors are up to date, compatible with the installed packages.
Key: 2064870018370ce2bde3f977cf753ed8c59848d0
----------------------------------------
Risk Factor: high
Title: Leapp detected loaded kernel drivers which are no longer maintained in RHEL 10.
Summary: The following RHEL 9 device drivers are no longer maintained RHEL 10:
     - ip_set

Key: fbb11ae5828d624c4e4c91e73d766c8e27b066d9
----------------------------------------
Risk Factor: medium
Title: Berkeley DB (libdb) has been detected on your system
Summary: Libdb was marked as deprecated in RHEL-9 and in RHEL-10 is not included anymore. There are a couple of alternatives in RHEL-10; the applications that depend on libdb will not work. Such applications must implement another type of backend storage. And migrate existing data to the new database format.
Related links:
    - Migrating to a RHEL 10 without libdb: https://access.redhat.com/articles/7099256
Remediation: [hint] Back up your data before proceeding with the data upgrade/migration. For the conversion, the tool db_converter from the libdb-utils rpm could be used. This database format conversion must be performed before the system upgrade. The db_converter is not available in RHEL 10 systems. For more information, see the provided article.
Key: fdc8f5b084e95922a4f59485a807a92cae2fc738
----------------------------------------
Risk Factor: low
Title: SElinux will be set to permissive mode
Summary: SElinux will be set to permissive mode. Current mode: enforcing. This action is required by the upgrade process to make sure the upgraded system can boot without beinig blocked by SElinux rules.
Remediation: [hint] Make sure there are no SElinux related warnings after the upgrade and enable SElinux manually afterwards. Notice: You can ignore the "/root/tmp_leapp_py3" SElinux warnings.
Key: 39d7183dafba798aa4bbb1e70b0ef2bbe5b1772f
----------------------------------------
Risk Factor: low
Title: Some enabled RPM repositories are unknown to Leapp
Summary: The following repositories with Red Hat-signed packages are unknown to Leapp:
- appstream
- epel
- elevate
- baseos
And the following packages installed from those repositories may not be upgraded:
- python3-ptyprocess
- libnvme
- systemd-udev
### snip what seems like the rest of my AlmaLinux packages ###
Remediation: [hint] You can file a request to add this repository to the scope of in-place upgrades by filing a support ticket
Key: 8e89e20c645cea600b240156071d81c64daab7ad
----------------------------------------
Risk Factor: low
Title: The subscription-manager release is going to be kept as it is during the upgrade
Summary: The upgrade is executed with the --no-rhsm option (or with the LEAPP_NO_RHSM environment variable). In this case, the subscription-manager will not be configured during the upgrade. If the system is subscribed and release is set already, you could encounter issues to get RHEL content using DNF/YUM after the upgrade.
Remediation: [hint] Set the new release (or unset it) after the upgrade using subscription-manager: subscription-manager release --set 10.0
Key: 01986584e27e85ea18929586faf614eee011a121
----------------------------------------
Risk Factor: info
Title: SElinux relabeling will be scheduled
Summary: SElinux relabeling will be scheduled as the status is permissive/enforcing.
Key: 8fb81863f8413bd617c2a55b69b8e10ff03d7c72
----------------------------------------
```

Anyway, once you've remediated any blockers or potential issues (once the system is ready for its update), run a `leapp upgrade`:

```txt
[wporter@m920q1 ~]$ sudo leapp upgrade
==> Processing phase `configuration_phase`
====> * ipu_workflow_config
        IPU workflow config actor
==> Processing phase `FactsCollection`
====> * vendor_repositories_mapping
        Scan the vendor repository mapping files and provide the data to other actors.
### SNIP ###
Total size: 915 M
DNF will only download packages, install gpg keys, and check the transaction.
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Complete!
====> * add_upgrade_boot_entry
        Add new boot entry for Leapp provided initramfs.
A reboot is required to continue. Please reboot your system.


Debug output written to /var/log/leapp/leapp-upgrade.log

============================================================
                      REPORT OVERVIEW
============================================================

HIGH and MEDIUM severity reports:
    1. Detected custom leapp actors or files.
    2. Leapp detected loaded kernel drivers which are no longer maintained in RHEL 10.
    3. Berkeley DB (libdb) has been detected on your system

Reports summary:
    Errors:                      0
    Inhibitors:                  0
    HIGH severity reports:       2
    MEDIUM severity reports:     1
    LOW severity reports:        3
    INFO severity reports:       1

Before continuing, review the full report below for details about discovered problems and possible remediation instructions:
    A report has been generated at /var/log/leapp/leapp-report.txt
    A report has been generated at /var/log/leapp/leapp-report.json

============================================================
                   END OF REPORT OVERVIEW
============================================================

Answerfile has been generated at /var/log/leapp/answerfile
Reboot the system to continue with the upgrade. This might take a while depending on the system configuration.
Make sure you have console access to view the actual upgrade process.
```

When it finishes (this will take a while) it'll tell you to reboot the system. Do so, and the actual upgrade will proceed. If it completes successfully, the system will come back up running AlmaLinux 10. If it does not, you will be having some fun.

When the system comes back up, it's time to clean up the bit of mess the upgrade made.

A quick aside - the upgrade log is located at `/var/log/leapp/leapp-upgrade.log` if you need it (though it is *quite* long).

First, we'll verify that it did update successfully:

```txt
[wporter@m920q1 ~]$ uname -a
Linux m920q1 6.12.0-55.9.1.el10_0.x86_64 #1 SMP PREEMPT_DYNAMIC Sat May 24 12:41:28 EDT 2025 x86_64 GNU/Linux
[wporter@m920q1 ~]$ cat /etc/os-release
NAME="AlmaLinux"
VERSION="10.0 (Purple Lion)"
```

We can look at what packages are left over from AlmaLinux 9 with `rpm -qa | grep el9`. In my case, it's not too bad, but I don't want to keep these around:

```txt
[wporter@m920q1 ~]$ rpm -qa | grep el9
kernel-modules-core-5.14.0-503.38.1.el9_5.x86_64
kernel-core-5.14.0-503.38.1.el9_5.x86_64
kernel-modules-5.14.0-503.38.1.el9_5.x86_64
kernel-5.14.0-503.38.1.el9_5.x86_64
kernel-modules-core-5.14.0-570.19.1.el9_6.x86_64
kernel-core-5.14.0-570.19.1.el9_6.x86_64
kernel-modules-5.14.0-570.19.1.el9_6.x86_64
kernel-5.14.0-570.19.1.el9_6.x86_64
elevate-release-1.0-2.el9.noarch
python3-leapp-0.19.0-2.el9.noarch
leapp-0.19.0-2.el9.noarch
leapp-upgrade-el9toel10-0.22.0-3.el9.elevate.1.noarch
leapp-data-almalinux-0.9-3.el9.20250505.noarch
```

Remove the old 5.14 kernel:

```txt
[wporter@m920q1 ~]$ sudo dnf remove kernel-modules-core-5.14.0
### SNIP ###
Removed:
  kernel-5.14.0-503.38.1.el9_5.x86_64                        kernel-5.14.0-570.19.1.el9_6.x86_64
  kernel-core-5.14.0-503.38.1.el9_5.x86_64                   kernel-core-5.14.0-570.19.1.el9_6.x86_64
  kernel-modules-5.14.0-503.38.1.el9_5.x86_64                kernel-modules-5.14.0-570.19.1.el9_6.x86_64
  kernel-modules-core-5.14.0-503.38.1.el9_5.x86_64           kernel-modules-core-5.14.0-570.19.1.el9_6.x86_64

Complete!
```

Cool! We did some damage!

```txt
[wporter@m920q1 ~]$ rpm -qa | grep el9
elevate-release-1.0-2.el9.noarch
python3-leapp-0.19.0-2.el9.noarch
leapp-0.19.0-2.el9.noarch
leapp-upgrade-el9toel10-0.22.0-3.el9.elevate.1.noarch
leapp-data-almalinux-0.9-3.el9.20250505.noarch
```

We can finish up by removing `elevate-release` and `leapp-data-almalinux` with `dnf`:

```txt
[wporter@m920q1 ~]$ sudo dnf remove -y elevate-release leapp-data-almalinux
```

Then getting rid of `leapp`, `python3-leapp`, and `leapp-upgrade-el9toel10` with `rpm`:

```txt
[wporter@m920q1 ~]$ sudo rpm -e leapp python3-leapp leapp-upgrade-el9toel10
```

If we have a look now, we can confirm that all the `el9` stuff is gone:

```txt
[wporter@m920q1 ~]$ rpm -qa | grep el9
# no output
```

Finally, we'll run a `dnf autoremove` for good measure. On my system, this'll grab a few things:

```txt
[wporter@m920q1 ~]$ sudo dnf autoremove
### SNIP ###
Removed:
  acl-2.3.2-4.el10.x86_64                                       boost-iostreams-1.83.0-5.el10.x86_64
  boost-system-1.83.0-5.el10.x86_64                             boost-thread-1.83.0-5.el10.x86_64
  daxctl-libs-80-3.el10.x86_64                                  fuse-common-3.16.2-5.el10.x86_64
  fuse-overlayfs-1.14-2.el10.x86_64                             fuse3-3.16.2-5.el10.x86_64
  gdk-pixbuf2-2.42.12-3.el10.x86_64                             grub2-tools-efi-1:2.12-14.el10_0.alma.1.x86_64
  grub2-tools-extra-1:2.12-14.el10_0.alma.1.x86_64              gvisor-tap-vsock-6:0.8.5-1.el10_0.x86_64
  gvisor-tap-vsock-gvforwarder-6:0.8.5-1.el10_0.x86_64          libappstream-glib-0.8.3-3.el10.x86_64
  libatomic-14.2.1-7.el10.alma.1.x86_64                         libdaemon-0.14-31.el10.x86_64
  libnl3-cli-3.11.0-1.el10.x86_64                               libpmem-2.1.0-2.el10_0.x86_64
  libtool-ltdl-2.4.7-13.el10.x86_64                             libuser-0.64-10.el10.x86_64
  ndctl-libs-80-3.el10.x86_64                                   python3-gpg-1.23.2-6.el10.alma.1.x86_64
  python3-psutil-5.9.8-5.el10.x86_64                            python3-setuptools-wheel-69.0.3-9.el10.noarch
  slirp4netns-1.3.2-1.el10.x86_64                               squashfs-tools-4.6.1-6.el10.x86_64
  systemd-rpm-macros-257-9.el10_0.1.alma.1.noarch               which-2.21-43.el10.x86_64

Complete!
```

And, to get rid of any files that are sitting around unused, clear the rpm cache with `dnf`:

```txt
[wporter@m920q1 ~]$ sudo dnf clean all
374 files removed
```

One more reboot for good measure, and we're all set! Alma 9 is gone! Our system is good for another billion years (well, at least until 2035).
