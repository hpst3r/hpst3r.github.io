---
title: "Working with SELinux (RHCSA-level)"
date: 2026-05-09T18:15:00-00:00
draft: false
---

## References

- [The Red Hat SELinux documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/index)
- [The Fedora SELinux documentation](https://docs.fedoraproject.org/en-US/quick-docs/selinux-getting-started/)
- [Open Source Watch SELinux primer](https://opensourcewatch.beehiiv.com/p/everything-wanted-know-selinux-afraid-run)
- [The Gentoo SELinux documentation](https://wiki.gentoo.org/wiki/SELinux)

RHCSA v10 EX-200 exam objectives:

- Manage security (5 of 8)
  - Set enforcing and permissive modes for SELinux
  - List and identify SELinux file and process context
  - Restore default file contexts
  - Manage SELinux port labels
  - Use boolean settings to modify system SELinux settings

We're going to go a bit past this and touch on users, roles, and senstivity in addition to managing file contexts.

## Introduction

SELinux (Security Enhanced Linux) adds an additional layer of access control to Linux systems. This supplements the traditional Unix permissions model that we all know and love with a more advanced mandatory access control system that, essentially, lets you control access by asking "may this *subject* do this *action* to this *object*?"

Standard Unix permissions are known as discretionary access control, or DAC - they are useful, but do not allow you to create granular, comprehensive security policy (e.g., allow a set of applications only read access to a specific set of files, while allowing another set to write those same files).

Additionally, with standard DAC permissions, the root user bypasses the permissions system and has full access to everything on the system. If you're running something as root, you have very little ability to prevent that thing from having full access to your system. Or, if you launch a process as your user, it has access to anything you have access to - so if you run a webserver as joe.admin, and that webserver gets knocked over, an adversary has full access to whatever Joe's admin account has access to.

Normally, this is.. semi-fine; just don't run random junk as root. But consider an accidental `chmod 777` here, a set-UID binary there, and a zero-day in the other corner.. and then your webserver has root.

So, for example, if Bob's webserver gets compromised, and Bob happened to be running that webserver as root, or for whatever reason the attacker is able to compromise the Apache server on that box and elevate whatever account is running the webserver to root.. Bob's machine is no longer Bob's machine.

However, were Bob to use SELinux, the webserver would be running in something like the `httpd_t` context. So, even if Bob decided to just run his webserver as root, SELinux would explicitly block access to anything that's not standard for a webserver that a process running in `httpd_t` tries to access - for example, you won't be able to get a shell, and won't be able to nuke the system, because SELinux will prevent a process running in the `httpd_t` context from gaining access to files in the `system_u` context (like everything under `/` in a standard installation) or launching a program in the `system_u:object_r:shell_exec_t` context.

End result? Bob still needs to figure out how these people keep getting into his machines, but the machine (hopefully) wasn't fully compromised.

Just above, I mentioned *contexts*. Every process, port, or file ("resource") in a SELinux-enabled system has a security label called a "SELinux context". These contexts (or labels - same thing) are solely security identifiers - this serves to provide a consistent way of referencing objects and removes ambiguity in identification (e.g., multiple hardlinks or bind mounts to a file).

SELinux uses these contexts to determine what "security scope" a given process, file, or port falls under. Relationships between contexts are defined in SELinux *policies*.

SELinux policies are an implicit-deny series of rules that define what processes are able to do with other resources. For example (sticking with our webserver example above), a process in `httpd_t` may be allowed by policy to bind to a common HTTP server port, like 80 or 443. `httpd_t` will NOT be allowed to bind to, say, port 445, 3389, 88, or ANYTHING else not explicitly defined in the policy. `httpd_t` will also NOT even be allowed to make outbound network connections, access files outside of the webroot directory, or modify its configuration files by default.

All this means that SELinux can enforce greater, more granular restrictions on processes, reducing the potential blast radius of an incident.

> SELinux policy rules are checked *after* traditional access control, so no SELinux denial is logged only if traditional permissions deny access.

## Contexts

Before we dig in, let's go over contexts. These are the labels you see when you run a `ls -lZ` or `ps -Z` that look something like `system_u:object_r:sysfs_t:s0`. These SELinux contexts are composed of four things that SELinux can use for access control:

1. The user, e.g., `system_u` or `unconfined_u`
2. The role (part of SELinux RBAC) - for a file, typically `object_r`
3. The type, e.g., `user_home_t`, `httpd_sys_content_t`, `sysfs_t`
4. The sensitivity level, e.g., `s0`

In the standard targeted policy shipped with RHEL, we don't typically have to worry about anything but types - the targeted policy is called Type Enforcement because it controls access mostly through determining which process may do what actions to which types of file.

This basic configuration is often sufficient, especially for a small team or a shop without complex systems or comprehensive security requirements, but SELinux can go a lot deeper!

> We'll talk about all of these different concepts but will only be going over managing types today, since this is what's on the exam.

## Users

SELinux users are a separate concept from Linux users, though each Linux user is mapped to a SELinux user with SELinux policy. SELinux "users" allow you to restrict Linux users to a certain level of access (rather, a certain selection of SELinux roles, more on that later) - during a session, the user will never be permitted to elevate past what roles (and, thus, what access) their policy allows.

> Additionally, with MLS configured, users are mapped to a specific sensitivity level - a little more on this later, too.

The primary benefit of SELinux users are enforced separation of privileges at the MAC level. SELinux users serve mostly as a vehicle to restrict which roles (and thus which services or privileges) a user may access - for example, you could prevent a specific user from ever elevating to root or changing users with `su`, even if they have the password for the respective account, by blocking setuid binaries for their "class" of user.

On a standard RHEL installation with a targeted SELinux policy, most users are mapped to `unconfined_u` by default - this means confinement is essentially off for your user accounts, and any user can do anything without being inhibited by SELinux. You can validate this with `semanage login -l`:

```sh
$ sudo semanage login -l

Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
```

And you can view your current SELinux user, role and type with `id -Z`:

```sh
$ id -Z
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

You can view a list of users on the system (and their eligible MLS ranges and SELinux roles) with `semanage user -l`:

```sh
$ sudo semanage user -l

                Labeling   MLS/       MLS/
SELinux User    Prefix     MCS Level  MCS Range                      SELinux Roles

guest_u         user       s0         s0                             guest_r
root            user       s0         s0-s0:c0.c1023                 system_r staff_r sysadm_r unconfined_r
staff_u         user       s0         s0-s0:c0.c1023                 system_r staff_r sysadm_r unconfined_r
sysadm_u        user       s0         s0-s0:c0.c1023                 sysadm_r
system_u        user       s0         s0-s0:c0.c1023                 system_r unconfined_r
unconfined_u    user       s0         s0-s0:c0.c1023                 system_r unconfined_r
user_u          user       s0         s0                             user_r
xguest_u        user       s0         s0                             xguest_r
```

The other role intended for human administrators (sysadm) is similarly widely scoped - this is effectively root access to the system in the SELinux context.

We also have some lesser-scoped users, namely staff, user, xguest and guest. These are intended for normal use. Staff may elevate to sysadm. User is a normal system user, with access to services, networking, et cetera. Guest users are least-privilege accounts for graphical kiosks (xguest) and basic terminal access (guest).

Some examples of the default SELinux users' capabilities, by user, are:

| User         | Role                   | Domain       | su  | sudo | Execute in home and /tmp | Networking   |
| ------------ | ---------------------- | ------------ | --- | ---- | ------------------------ | ------------ |
| unconfined_u | unconfined_r, system_r | unconfined_t | yes | yes  | yes                      | yes          |
| sysadm_u     | sysadm_r               | sysadm_t     | yes | yes  | yes                      | yes          |
| system_u     | system_r, unconfined_r | system_t     | N/A | N/A  | N/A                      | N/A          |
| staff_u      | staff_r, sysadm_r      | staff_t      | no  | yes  | yes                      | yes          |
| user_u       | user_r                 | user_t       | no  | no   | yes                      | yes          |
| xguest_u     | xguest_r               | xguest_t     | no  | no   | yes                      | Firefox only |
| guest_u      | guest_r                | guest_t      | no  | no   | yes                      | no           |

> Why all the "N/A" for system_u? Well, system_u is for services only, and the user/role are generally just a stamp to say "this was launched by systemd". Enforcement at this level is typically by type. Execmod/execute_no_trans, network access, su or sudo access are all possible (there's nothing tied to system_u that explicitly prevents this, in contrast to, say, guest_u), but per-service types are typically very strict and a "can httpd_t execute su_exec_t?" should come back as a "NO!"

### Unconfined

The default user, unconfined_u, is still technically managed by SELinux but has wide-ranging SELinux privileges (effectively not restricted). This effectively allows you, the human admin, to default to unrestricted access on a SELinux-enabled system, likely to prevent extra SELinux-related frustration.

### System

The SELinux system user is assigned to any process spawned by the init system (in modern times, systemd). It has access to a great variety of contexts, but those contexts are tightly scoped for each process on the system.

The SELinux-aware kernel manages transitioning processes to the relevant context according to policy - for example, if sshd were to launch via init execution of `/usr/sbin/sshd` either on boot or via a `systemctl start sshd`, the SSH daemon will transition from the init context to `sshd_t` once policy observes that the init system, in context `init_t`, executed the binary in context `sshd_exec_t`.

> It's notable that, in the past, if a user restarted a service by running its respective init script from an admin shell, this would cause the process to start in the user's context rather than the init context.
>
> systemd is a lot cleaner (as far as SELinux is concerned) because it stamps `system_u:system_r` on everything it spawns, including processes started via `systemctl`.

### Administrative roles

The administrative roles staff, sysadm, and secadm are intended for use by IT folk, like system and security administrators, who need privileged access to systems.

sysadm is prevented from graphical or SSH logon by default. I suspect this is so an attacker cannot SSH in to a privileged account directly; they would need to go through a non-privileged account first.

The intended workflow for a typical account is elevation from staff to sysadm via sudo - staff users are allowed to log on via SSH or graphically, and can then elevate to `staff_u:sysadm_t`, while an administrative-only account might be directly assigned to the `sysadm_u` role.

There's also the `secadm` role, that allows managing SELinux. By default, this privilege is assigned to the `sysadm` role, but if you have separate administration and security teams, you should certainly split these up.

This allows auditable separation of privileges - security admins can review the audit logs and manage SELinux, while system administrators can manage the rest of the system.

In this scenario, even if a sysadm-tier account were compromised, SELinux cannot be modified or disabled (though, without further restriction, this obviously doesn't mean too much - if someone has root, they can certainly work around SELinux).

### Guests

The user, guest, and xguest users can only run permitted setuid binaries if permitted by policy, thus locking them out of `su` and `sudo` so they can *never* become root.

The xguest user is an interesting one - it's actually intended for graphical logon to kiosks and has a number of booleans in the default policy to control access to the network, bluetooth, removable media, and the ability to execute programs. By default, its network access is restricted to Firefox only.

The guest user, on the other hand, isn't permitted to log on graphically, only via a terminal (it's a "least privilege terminal user") that additionally defaults to having network access and removable media access disabled.

### Creating and modifying users

You can add, remove, or modify SELinux users with the `semanage user` commands. For example, let's create a custom user with access to the staff_r and user_r roles:

```sh
$ sudo semanage user -a -R "staff_r user_r" example_u

$ sudo semanage user -l | grep example_u
                Labeling   MLS/       MLS/
SELinux User    Prefix     MCS Level  MCS Range                      SELinux Roles
example_u       user       s0         s0                             staff_r user_r
```

Then, let's modify that user to add the sysadm role:

```sh
$ sudo semanage user -m -R "sysadm_r staff_r user_r" example_u

$ sudo semanage user -l | grep example_u
example_u       user       s0         s0                             sysadm_r staff_r user_r
```

And we can delete the user:

```sh
$ sudo semanage user -d example_u
```

To assign a SELinux user to a Linux user, we can either:

Create the user with `useradd -Z`:

```sh
$ sudo useradd -m user -s /bin/bash -Z user_u # -m: create home -s: set shell -Z: set SELinux user

$ sudo semanage login -l

Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
user                 user_u               s0                   *
```

### UBAC

Figured this was worth a mention. There's an additional optional constraint (the `ubac_constrained_type` attribute) that's not typically enabled, but allows you to restrict processes or files from interacting depending on the SELinux user label (e.g., `system_u` and `staff_u` can't talk).

## Roles

SELinux roles simply control the domains a user is able to access. In other words, a role is a simple whitelist of domains. When your session is `user_r`, you can only ever enter domains that are whitelisted for `user_r`. You will never be able to run `sudo` or `su` from the `user_u:user_r` context, because `user_r` doesn't whitelist access to `sudo_exec_t`.

For example, let's try to sudo and su from the `user_u:user_r:user_t` context. I enter the correct root password and... nada. I sudo and.. immediate nada.

```sh
[user@rhcsa0 ~]$ id -Z
user_u:user_r:user_t:s0
[user@rhcsa0 ~]$ su
Password:
su: Authentication failure
[user@rhcsa0 ~]$ sudo
sudo: PERM_SUDOERS: setresuid(-1, 1, -1): Operation not permitted
sudo: unable to open /etc/sudoers: Operation not permitted
sudo: setresuid() [0, 0, 0] -> [1001, -1, -1]: Operation not permitted
sudo: error initializing audit plugin sudoers_audit
```

We can see the domains a role is permitted to access with `seinfo -r`. There's a lot of stuff there:

```sh
$ seinfo -r user_r -x

Roles: 1
   role user_r types { abrt_helper_t alsa_home_t audio_home_t auth_home_t bluetooth_helper_t cache_home_t cdrecord_t chfn_t chkpwd_t chrome_sandbox_home_t chrome_sandbox_nacl_t chrome_sandbox_t chronyc_t config_home_t cronjob_t crontab_t data_home_t dbus_home_t fdo_home_t fetchmail_home_t fsadm_t gconf_home_t gconfd_t git_session_t git_user_content_t gkeyringd_gnome_home_t gnome_home_t gpg_agent_t gpg_helper_t gpg_pinentry_t gpg_secret_t gpg_t gstreamer_home_t home_bin_t home_cert_t httpd_user_script_t icc_data_home_t iceauth_home_t iceauth_t irc_home_t irc_t irc_tmp_t irssi_home_t irssi_t journalctl_t kismet_home_t kmod_t krb5_home_t loadkeys_t local_login_home_t lpr_t lvm_t mail_home_rw_t mail_home_t mandb_home_t mount_t mozilla_home_t mozilla_plugin_config_t mozilla_plugin_t mpd_home_t mpd_user_data_t mysqld_home_t namespace_init_t newrole_t nscd_t obex_t oddjob_mkhomedir_t oddjob_t openshift_var_lib_t pam_timestamp_t passwd_t ping_t policykit_auth_t policykit_grant_t polipo_cache_home_t polipo_config_home_t polipo_session_t postfix_postdrop_t postfix_postqueue_t pppd_t procmail_home_t ptchown_t pulseaudio_home_t pulseaudio_t qmail_inject_t qmail_queue_t rpcd_t rssh_ro_t rssh_rw_t sandbox_file_t sandbox_min_client_t sandbox_min_t sandbox_net_client_t sandbox_net_t sandbox_web_client_t sandbox_web_t sandbox_x_client_t sandbox_x_t sandbox_xserver_t screen_home_t setfiles_t smbmount_t spamc_home_t speech_dispatcher_home_t ssh_home_t ssh_t svirt_home_t svirt_socket_t svirt_t svirt_tcg_t systemd_home_t targetclid_home_t telepathy_cache_home_t telepathy_data_home_t telepathy_gabble_cache_home_t telepathy_gabble_t telepathy_idle_t telepathy_logger_cache_home_t telepathy_logger_data_home_t telepathy_logger_t telepathy_mission_control_cache_home_t telepathy_mission_control_data_home_t telepathy_mission_control_home_t telepathy_mission_control_t telepathy_msn_t telepathy_salut_t telepathy_sofiasip_t telepathy_stream_engine_t telepathy_sunshine_home_t telepathy_sunshine_t texlive_home_t thumb_home_t thumb_t traceroute_t tvtime_home_t uml_ro_t uml_rw_t updpwd_t user_dbusd_t user_fonts_cache_t user_fonts_config_t user_fonts_t user_gkeyringd_t user_home_dir_t user_home_t user_mail_t user_screen_t user_seunshare_t user_ssh_agent_t user_t user_tmp_t user_wine_t utempter_t virt_bridgehelper_t virt_content_t virt_home_t vlock_t vmtools_helper_t vmtools_t vmware_conf_t vmware_file_t wine_home_t wireshark_home_t xauth_home_t xauth_t xdm_home_t };
```

However, what isn't there is anything to do with sudo:

```sh
$ seinfo -r user_r -x | grep sudo
# nada
```

Note that we also don't have any SELinux allow and type transition rules that would potentially allow user_t to become anything else with MAC privileges to actually do anything of interest once we do sudo (even if we *could* use sudo - we can't). Allows are evaluated in conjunction with the role whitelist - if either is missing, stuff won't work.

## Types

Types are labels for many things, chief among which are files and ports. Types for processes also exist, but are called domains. These are the primary "part" of a SELinux context on a standard EL system.

What does this mean, you might ask? Well, SELinux on a 'targeted' system is called "Type Enforcement" because the core access control functionality involves restricting which types a process can access.

We assign types to processes, files, and ports. Policy then defines what types of processes, files, and ports a process can access. For example, the `httpd_t` domain belongs to your webservers, like Apache or NGINX:

```sh
$ ps -eZ | grep httpd
system_u:system_r:httpd_t:s0     440961 ?        00:00:00 nginx
system_u:system_r:httpd_t:s0     440962 ?        00:00:00 nginx
system_u:system_r:httpd_t:s0     440963 ?        00:00:00 nginx
```

And the system policy says that this `httpd_t` domain (process type) is allowed to read the `httpd_sys_content_t` file type, and can write the `httpd_sys_rw_content_t` type. Both types are intended for webserver content, e.g., the default index:

```sh
$ ls -lZ /usr/share/nginx/html | grep index
lrwxrwxrwx. 1 root root system_u:object_r:httpd_sys_content_t:s0  25 Mar 30 20:00 index.html -> ../../testpage/index.html
lrwxrwxrwx. 1 root root system_u:object_r:httpd_sys_content_t:s0  37 Mar 30 20:00 system_noindex_logo.png -> ../../pixmaps/system-noindex-logo.png
```

And policy also says that the `httpd_t` domain is permitted to bind to `http_port_t`-type  ports:

```sh
$ sudo semanage port -l | grep ^http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
```

Note that all three of these types of resources (processes, files, and ports) have different types, and policies are written around these specific types. Don't assign the `httpd_t` type to a port or file - SELinux won't like it!

To examine said types, you can use the `semanage {resource} -l` commands. For example:

To find the type for a port, run `semanage port -l`:

```sh
$ sudo semanage port -l | grep ^mysql
mysqld_port_t                  tcp      1186, 3306, 63132-63164
mysqlmanagerd_port_t           tcp      2273
```

To see what domain has bound to a port (at the moment), you can use the `lsof -Z` command. Do note that this doesn't tell you what type the port is - just the context of the process binding to it.

To find the type for a file or directory, you can look through the merged file context table with `semanage fcontext -l`. As file contexts are written with regular expressions, note that an exact search might not find anything for you.

```sh
$ sudo semanage fcontext -l | grep /etc/nginx
/etc/nginx(/.*)?                                   all files          system_u:object_r:httpd_config_t:s0
```

To view the contexts on files in a directory (for example, to determine whether a file is mislabelled), you can run `ls -Z`:

```sh
$ ls -lZ ~
total 1404
lrwxrwxrwx. 1 wporter wporter unconfined_u:object_r:user_home_t:s0       9 Apr 19 17:37 bash -> /bin/bash
-rw-r--r--. 1 root    root    unconfined_u:object_r:user_home_t:s0     116 May  3 02:43 boolean
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:user_home_t:s0     125 Apr 19 14:03 email.txt
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:user_home_t:s0       0 Apr 21 21:01 file1.txt
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:user_home_t:s0       0 Apr 21 21:01 file2.txt
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:user_home_t:s0     314 Apr 21 21:02 files.zip
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:user_home_t:s0       0 Apr 21 20:33 file.txt
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:user_home_t:s0      14 Apr 21 20:32 file.txt.bz2
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:user_home_t:s0      29 Apr 21 20:31 file.txt.gz
-rwxr-xr-x. 2 root    root    system_u:object_r:shell_exec_t:s0    1414608 Oct 28  2024 hardbash
```

To determine what access a domain has to a certain type, you can use the `sesearch` utility:

```sh
$ sesearch --allow -s httpd_t -t httpd_sys_content_t -c file
allow domain file_type:file map; [ domain_can_mmap_files ]:True
allow httpd_t httpd_content_type:file { getattr ioctl lock map open read };
allow httpd_t httpdcontent:file { append create getattr ioctl link lock open read rename setattr unlink watch watch_reads write }; [ ( httpd_builtin_scripting && httpd_unified && httpd_enable_cgi ) ]:True
allow httpd_t httpdcontent:file { execute getattr ioctl map open read }; [ ( httpd_builtin_scripting && httpd_unified && httpd_enable_cgi ) ]:True
```

We'll go into a bit more depth about managing file and port contexts later.

## Multi-Level Security (MLS)

This is not in scope for the RHCSA, so we'll just do a quick overview.

Multi-Level Security is an additional SELinux configuration that adds sensitivity levels (levels of clearance) to the access control model. It's represented by the fourth field in a SELinux context (and is typically a global `s0` if not configured).

With a standard, targeted SELinux policy, everything is `s0` and the field is ignored. MLS is typically only used in DoD environments where data must be classified and completely restricted from or to certain users.

Sensitivity levels range from `s0` to `s15`. Higher levels indicate higher classification (`s0` is least sensitive, `s15` is most sensitive).

In MLS terminology, a user or process that acts on resources is called a *subject*. Their sensitivity level is their *clearance*. Resources like files and devices are called *objects*; their sensitivity levels are their *classifications*.

A process can read data at or below its sensitivity level, and can write only at or above it, to prevent the leaking of information to a less sensitive classification. This follows the Bell-La Padula model. On RHEL, processes can read at or below their sensitivity level and can write only at their sensitivity level - this is called "write equality" and prevents low-clearance users from writing to secret files. This behavior, of course, can be adjusted by editing the system's policy.

MLS evaluation occurs after the evaluation of DAC and type enforcement rules.

### Multi-Category Security (MCS)

Multi-Category Security is another SELinux feature that provides access control by *category* - by default, these categories range from `c0` to `c1023`, but you can name them if needed. MCS evaluation occurs after the evaluation of DAC and type enforcement rules.

Categories are added to SELinux contexts as a fifth label - for example, a full context might be: `system_u:system_r:system_t:s0:c1`.

MCS can be used, for example, to restrict access to a certain project's files to specific users and processes while keeping type labels and sensitivity intact. This, too, is out of scope for the RHCSA.

## Operational - using SELinux

### Enforcing and Permissive modes

SELinux can be in one of three "modes":

- Enforcing
  - SELinux is active and enforcing policy (will deny access to resources).
- Permissive
  - SELinux is active and monitoring activity, but will not deny access to resources. Useful for testing, or when you don't want SELinux enabled.
- Disabled
  - SELinux has been fully disabled. This is not an ideal state, as the system will no longer be able to manage policy. You'll have a hard time reenabling SELinux after the fact.

#### Check and set enforcing or permissive state globally

To check whether or not SELinux is enabled on your system, run the `getenforce` command. This is usually present on any EL-type machine, down to the minimal cloud-init templates.

```txt
[wporter@rhcsa0 ~]$ getenforce
Enforcing
```

To temporarily switch SELinux between permissive and enforcing mode, use the `setenforce` command, passing either Permissive (0) or Enforcing (1). For example:

```sh
$ getenforce
Enforcing
$ sudo setenforce Permissive; getenforce
Permissive
$ sudo setenforce Enforcing; getenforce
Enforcing
$ sudo setenforce 0; getenforce
Permissive
$ sudo setenforce 1; getenforce
Enforcing
```

To set permissive or enforcing at boot, edit the `/etc/selinux/config` file, changing `SELINUX=` to the mode you want to set:

```sh
$ sudo cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
#
SELINUX=enforcing
# SELINUXTYPE= can take one of these three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

And reboot the system. Then, confirm the change with `getenforce`.

#### Set enforcing or permissive state per domain

Rather than setting SELinux to permissive mode system-wide, it may be smart to keep SELinux enforcing on the system and toggle it to permissive for just one domain. To do so, you can run the `semanage permissive -a domain`  command, e.g.:

```sh
$ sudo semanage permissive -a httpd_t
```

You can view a list of domains set to permissive with `semanage permissive -l`. This will show you built-in permissive domains, and any domains you've explicitly set permissive. Note that you cannot set any built-in permissive types to enforcing.

```sh
$ sudo semanage permissive -l

Builtin Permissive Types

systemd_hibernate_resume_t
systemd_zram_generator_t
systemd_import_generator_t
virtqemud_t
coreos_liveiso_autologin_generator_t
systemd_generic_generator_t
virtvboxd_t
anaconda_generator_t
coreos_boot_mount_generator_t
qgs_t
systemd_nfs_generator_t
switcheroo_control_t
systemd_pcrlock_t
virtstoraged_t
coreos_installer_generator_t
systemd_user_runtimedir_t
systemd_tpm2_generator_t
coreos_sulogin_force_generator_t
virtsecretd_t
systemd_mountfsd_t
tuned_ppd_t
systemd_pcrextend_t
bootupd_t
ktlshd_t

Customized Permissive Types

httpd_t
```

To set a domain back to enforcing mode, simply `semanage permissive -d domain`:

```sh
$ sudo semanage permissive -d httpd_t
libsemanage.semanage_direct_remove_key: Removing last permissive_httpd_t module (no other permissive_httpd_t module exists at another priority).

$ sudo semanage permissive -l | grep httpd_t
$ # nada
```

### Booleans

SELinux booleans allow you to toggle portions of policies - for example, to allow a webserver like NGINX to make network connections (e.g., when you're using it as a reverse proxy). These can be managed with the `setsebool` command, e.g.:

```sh
setsebool foo_can_bar 1
```

Additionally, you can list the booleans on the system with the `getsebool -a` or `semanage boolean -l` commands - `getsebool` will simply return the names and current state of a queried bool, while `semanage boolean -l` will also include the default state and a brief description.

```sh
$ sudo semanage boolean -l | head -n 4
SELinux boolean                State  Default Description

abrt_anon_write                (off  ,  off)  Allow abrt to anon write
abrt_handle_event              (on   ,   on)  Allow abrt to handle event
```

```sh
$ getsebool -a | head -n 2
abrt_anon_write --> off
abrt_handle_event --> on
```

To persist a boolean, use the `setsebool -P` argument (`P` for persistence). Other parameters are `-N` for no policy reload, and `-V` for verbose error messages.

Some examples of booleans include:

The `httpd_can_network_connect` boolean, which can be used to allow webservers to make network connections (as described above):

```sh
sudo setsebool -P httpd_can_network_connect 1
```

The `grafana_can_tcp_connect_prometheus_port` boolean, allowing Grafana to connect to the Prometheus API on port 9090:

```sh
sudo setsebool -P grafana_can_tcp_connect_prometheus_port 1
```

The `deny_bluetooth` boolean allows you to prevent any user from using bluetooth:

```sh
sudo setsebool -P deny_bluetooth 1
```

### Managing file contexts

To set the context for a file, you'll primarily use the `restorecon` and `chcon` utilities.

#### restorecon

`restorecon` allows you to reset contexts to the defined defaults.

When you copy a file, say, from your home directory to the NGINX configuration or data directory, the web server won't be able to read the file (likely labeled `user_home_t` - there's no good reason for NGINX to read a file from a home directory, but the label won't be automatically adjusted by the move). So, you'll need to use `restorecon` to reset the file's context to the default for that path:

```sh
$ touch example.html

$ ls -lZ example.html
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:user_home_t:s0 0 May  5 21:54 example.html

$ sudo mv example.html /usr/share/nginx/html/

$ ls -lZ /usr/share/nginx/html/example.html
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:user_home_t:s0 0 May  5 21:54 /usr/share/nginx/html/example.html

$ sudo restorecon /usr/share/nginx/html/example.html

$ ls -lZ /usr/share/nginx/html/example.html
-rw-r--r--. 1 wporter wporter unconfined_u:object_r:httpd_sys_content_t:s0 0 May  5 21:54 /usr/share/nginx/html/example.html
```

Alternatively, you can change the context for your files by adjusting the label for the directory with `chcon`. However, note that this doesn't update the default context for the path, so this change will be overwritten if the file is ever relabeled - use this only for testing.

```sh
chcon -t httpd_sys_content_t /example/file
```

#### Permanently updating path mappings

To permanently adjust the context of a file or directory, you can use the `semanage fcontext -a` command to add a context entry to the file contexts table.

This adds a regular expression (to match a path) and respective context to the `/etc/selinux/targeted/contexts/files/file_contexts.local` file with the default targeted policy in play.

The `/etc/selinux/targeted/contexts/files/file_contexts` file, meanwhile, is the set of contexts shipped with the system. We can see that this is populated without any action on our part:

```txt
[wporter@rhcsa0 ~]$ cat /etc/selinux/targeted/contexts/files/file_contexts | head -n 3
/.* system_u:object_r:default_t:s0
/[^/]+ -- system_u:object_r:etc_runtime_t:s0
/a?quota\.(user|group) -- system_u:object_r:quota_db_t:s0
```

> `semanage fcontext` does *not* actually adjust the context of the file. You'll need to relabel your resources with `restorecon` *after* making changes to the context table to set labels on files.

To adjust the type of a specific file, we can use `semanage` to add an entry to the `file_contexts.local` file, then restore the file's context to what it should be according to the contexts on disk with `restorecon`:

```sh
$ sudo touch /example

$ ls -lZ /example
-rw-r--r--. 1 root root unconfined_u:object_r:etc_runtime_t:s0 0 May  3 17:31 /example

$ sudo semanage fcontext -a -t user_home_t /example

$ ls -lZ /example
-rw-r--r--. 1 root root unconfined_u:object_r:etc_runtime_t:s0 0 May  3 17:31 /example

$ cat /etc/selinux/targeted/contexts/files/file_contexts.local | grep /example
/example    system_u:object_r:user_home_t:s0

$ sudo restorecon /example

$ ls -lZ /example
-rw-r--r--. 1 root root unconfined_u:object_r:user_home_t:s0 0 May  3 17:31 /example
```

Alternatively, rather than grepping the file_contexts files (there are two), you can list entries in the merged contexts database with `semanage fcontext -l`. For example:

```sh
$ sudo semanage fcontext -l | grep /example
/etc/ipsec\.d/examples(/.*)?                       all files          system_u:object_r:etc_t:s0
/example                                           all files          system_u:object_r:user_home_t:s0
```

Note that we have two entries here - one from the default policy for example files in `/etc/ipsec.d`, and one for our `/example` file. Also note that this requires elevation (unlike grepping the files).

### Managing port types

We use the `semanage port` commands to work with port contexts.

Let's relabel port 8000 to the `http_port_t` context. We'll pass the arguments:

- `-a`, `--add`: add type
- `-t`, `--type`
- `-p`, `--proto`: protocol

The `-m` (`--modify`) argument also exists, but you can use `--add` to modify, so use whichever you'd like.

> As mentioned in the "types" section earlier on, you can list ports and contexts with `semanage port -l`. We'll use that for verification below.

```sh
$ sudo semanage port -l | grep ^http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
$ sudo semanage port -a -t http_port_t -p tcp 8000
Port tcp/8000 already defined, modifying instead
$ sudo semanage port -l | grep ^http_port_t
http_port_t                    tcp      8000, 80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
```

Alternatively, to delete a (custom) label for a port, pass the `-d` (`--delete`) argument to `semanage`. This will revert the port to its standard label - in this case, for port 8000, that's `soundd_port_t`.

```sh
$ sudo semanage port -d -t http_port_t -p tcp 8000
$ sudo semanage port -l | grep ^http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
http_port_t                    udp      80, 443
$ sudo semanage port -l | grep 8000
soundd_port_t                  tcp      8000, 9433, 16001
```

### Troubleshooting SELinux denials

SELinux logs denials (among many other actions) to the `/var/log/audit/audit.log` file. We can search this file for access vector cache (AVC) events with `ausearch`, or we can grep it directly.

We're not going to go too far in depth, but we'll talk about the audit log and `ausearch`, which you can use to troubleshoot SELinux denials.

Let's create a denial or two! First, we'll change the context of a file (the default index) to create a denial and break NGINX:

```sh
$ sudo chcon -t user_home_t /usr/share/nginx/html/index.html
```

Then, if we try to make a connection (and force the web server to load the page), it'll fail:

```sh
$ curl -k localhost:80
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.26.3</center>
</body>
</html>
```

This denial will be logged to the audit log:

```sh
$ sudo cat /var/log/audit/audit.log | grep AVC
type=AVC msg=audit(1778114213.582:386422): avc:  denied  { read } for  pid=474679 comm="nginx" name="index.html" dev="sda4" ino=16923512 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0
```

#### ausearch

You can search the log with `ausearch`. In this case, we're looking for recent access vector cache (AVC) events.

The `-m` argument means "return events matching type". The `-ts`, or 'time start' (`--start`) and `-te` ('time end', `--end`) arguments allow you to specify a date and time range. Additionally, a few words (`now`, `recent`, `this-hour`, `boot`, `today`, `yesterday`, to name a few) are usable.

For example, let's pull 'recent' audit messages. Recent means "ten minutes ago", so we'll get everything from -10m to now.

```sh
$ sudo ausearch -m avc -ts recent
----
time->Wed May  6 20:36:53 2026
type=PROCTITLE msg=audit(1778114213.582:386422): proctitle=6E67696E783A20776F726B65722070726F63657373
type=SYSCALL msg=audit(1778114213.582:386422): arch=c000003e syscall=257 success=no exit=-13 a0=ffffff9c a1=56407ef73d56 a2=800 a3=0 items=0 ppid=474677 pid=474679 auid=4294967295 uid=991 gid=991 euid=991 suid=991 fsuid=991 egid=991 sgid=991 fsgid=991 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1778114213.582:386422): avc:  denied  { read } for  pid=474679 comm="nginx" name="index.html" dev="sda4" ino=16923512 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0
```

To look for every event on May 6 2026 between 20:00:00 and 21:00:00:

```sh
$ sudo ausearch -m avc --start 5/6/2026 20:00:00 --end 5/6/2026 21:00:00
----
time->Wed May  6 20:36:53 2026
type=PROCTITLE msg=audit(1778114213.582:386422): proctitle=6E67696E783A20776F726B65722070726F63657373
type=SYSCALL msg=audit(1778114213.582:386422): arch=c000003e syscall=257 success=no exit=-13 a0=ffffff9c a1=56407ef73d56 a2=800 a3=0 items=0 ppid=474677 pid=474679 auid=4294967295 uid=991 gid=991 euid=991 suid=991 fsuid=991 egid=991 sgid=991 fsgid=991 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1778114213.582:386422): avc:  denied  { read } for  pid=474679 comm="nginx" name="index.html" dev="sda4" ino=16923512 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0
```

#### setroubleshoot

> Note: `setroubleshoot-server` installs the `setroubleshootd` daemon, which is activated on demand and does not run continuously.

Additionally, if you've got `setroubleshoot-server` installed, `setroubleshoot` will send helpful log messages with recommended fixes to the journal, like:

```txt
May 09 17:48:00 rhcsa0.lab.wporter.org setroubleshoot[1245]: SELinux is preventing /usr/sbin/nginx from read access on the file index.html. For complete SELinux messages run: sealert -l 73e185a9-1b71-4708-ac85-ff1e63a89af0
May 09 17:48:00 rhcsa0.lab.wporter.org setroubleshoot[1245]: SELinux is preventing /usr/sbin/nginx from read access on the file index.html.

                                                             *****  Plugin catchall_boolean (89.3 confidence) suggests   ******************

                                                             If you want to allow httpd to read user content
                                                             Then you must tell SELinux about this by enabling the 'httpd_read_user_content' boolean.

                                                             Do
                                                             setsebool -P httpd_read_user_content 1

                                                             *****  Plugin catchall (11.6 confidence) suggests   **************************

                                                             If you believe that nginx should be allowed read access on the index.html file by default.
                                                             Then you should report this as a bug.
                                                             You can generate a local policy module to allow this access.
                                                             Do
                                                             allow this access for now by executing:
                                                             # ausearch -c 'nginx' --raw | audit2allow -M my-nginx
                                                             # semodule -X 300 -i my-nginx.pp
```

Do note that a restart of the `auditd` service is required to pick up the `setroubleshoot` hook if you're just installing `setroubleshoot` to follow along. The `setroubleshootd` daemon is only run on demand after an audit event.

#### sealert

> Note: `sealert` depends on the `setroubleshoot-server` and `setroubleshootd`.

Note the log message in the `setroubleshoot` output:

```txt
For complete SELinux messages run: sealert -l 73e185a9-1b71-4708-ac85-ff1e63a89af0
```

You can get an extremely verbose summary of the AVC event by querying the event with `sealert`, as recommended:

```sh
$ sudo sealert -l 73e185a9-1b71-4708-ac85-ff1e63a89af0
SELinux is preventing /usr/sbin/nginx from read access on the file index.html.

*****  Plugin catchall_boolean (89.3 confidence) suggests   ******************

If you want to allow httpd to read user content
Then you must tell SELinux about this by enabling the 'httpd_read_user_content' boolean.

Do
setsebool -P httpd_read_user_content 1

*****  Plugin catchall (11.6 confidence) suggests   **************************

If you believe that nginx should be allowed read access on the index.html file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -X 300 -i my-nginx.pp


Additional Information:
Source Context                system_u:system_r:httpd_t:s0
Target Context                system_u:object_r:user_home_t:s0
Target Objects                index.html [ file ]
Source                        nginx
Source Path                   /usr/sbin/nginx
Port                          <Unknown>
Host                          rhcsa0.lab.wporter.org
Source RPM Packages           nginx-core-1.26.3-2.el10_1.1.x86_64
Target RPM Packages
SELinux Policy RPM            selinux-policy-targeted-42.1.7-1.el10_1.2.noarch
Local Policy RPM              selinux-policy-targeted-42.1.7-1.el10_1.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     rhcsa0.lab.wporter.org
Platform                      Linux rhcsa0.lab.wporter.org
                              6.12.0-124.49.1.el10_1.x86_64 #1 SMP
                              PREEMPT_DYNAMIC Thu Apr  9 00:52:33 EDT 2026
                              x86_64
Alert Count                   3
First Seen                    2026-05-09 17:47:58 EDT
Last Seen                     2026-05-09 17:47:59 EDT
Local ID                      73e185a9-1b71-4708-ac85-ff1e63a89af0

Raw Audit Messages
type=AVC msg=audit(1778363279.195:95): avc:  denied  { read } for  pid=1163 comm="nginx" name="index.html" dev="sda4" ino=16923512 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:user_home_t:s0 tclass=file permissive=0


type=SYSCALL msg=audit(1778363279.195:95): arch=x86_64 syscall=openat success=no exit=EACCES a0=ffffff9c a1=55f58dee8d56 a2=800 a3=0 items=0 ppid=1162 pid=1163 auid=4294967295 uid=991 gid=991 euid=991 suid=991 fsuid=991 egid=991 sgid=991 fsgid=991 tty=(none) ses=4294967295 comm=nginx exe=/usr/sbin/nginx subj=system_u:system_r:httpd_t:s0 key=(null)

Hash: nginx,httpd_t,user_home_t,file,read

```

Note that this summary includes:

- A concise summary of the event ("SELinux is preventing /usr/sbin/nginx from read access on the file index.html.")
- Booleans you can set to configure this behavior, if desired (where available). In this case, this is the `httpd_read_user_content` boolean that allows the `httpd_t` domain to read `user_home_t` files.
  - `sealert` provides the full command for toggling the boolean with `setsebool`.
- Instructions for using audit2allow to modify policy to allow the access, if desired.
  - Note that audit2allow often creates policy that is *too* permissive, so you might want to review the policy module before compiling and installing it.
- The source context (domain), in this case our web server's httpd_t
- The target context, in this case one pertaining to a file in a home directory
- The source process (nginx)
- The source package
- The policy package
- The policy type and enforcing mode
- The count, first and last seen dates for the alert
