---
title: "accurately generate SELinux policy with audit2allow"
date: 2025-12-09T01:30:00-00:00
draft: false
---

`audit2allow` is a powerful tool to turn a mess of SELinux audit logs into a policy that allows your application to do the stuff it needs to do. You can save time and headaches when attempting to fix SELinux access denial errors with a custom policy generated via `audit2allow` by:

- unmasking "unimportant" events that may be preventing your service from functioning
  - `semodule -DB`
- dropping SELinux into passive mode (preferably for just the specific context)
  - `setenforce 0` or `semanage permissive -a mycontext_t`
- wiping the audit log before a reproduce so you only feed `audit2allow` what is needed
  - `> /var/log/audit/audit.log`
- running the failing action/service/application and capturing a fresh set of events

```sh
# disable dontaudit so *every* selinux denial is logged
# some that the policy author thought were unimportant are probably masked
semodule -DB

# set selinux to 'passive' mode (audit but don't enforce MAC violations)
# this will let us log everything it's blocking (what we need to allow)
setenforce 0

# OR set just one context passive
# semanage permissive -a examplecontext_t
# to later set this "examplecontext_t" context back to enforcing:
# semanage permissive -d examplecontext_t

# clear the audit log so extra stuff doesn't find its way into our generated policy
> /var/log/audit/audit.log

# create audit events - run whatever is failing, e.g., systemd service 'mybrokenservice.service'
systemctl restart mybrokenservice

# capture all deny avc events into a policy template
cat /var/log/audit/audit.log | audit2allow -m mypolicy > mypolicy.te

# set SELinux back to 'enforcing' mode since we've probably collected our events now
setenforce 1
# OR if you just set one context: semanage permissive -d examplecontext_t

# reenable dontaudit so the log isn't filled with junk being intentionally blocked
semodule -B
```

Audit the generated `*.te` file (e.g., `mypolicy.te`) to confirm you're not allowing too much access.

Once satisfied, check, package and install the module:

```sh
checkmodule -M -m -o mypolicy.mod mypolicy.te
semodule_package -o mypolicy.pp -m mypolicy.mod
semodule -i mypolicy.pp
```

To remove a policy module, use `semodule -r`:

```sh
semodule -r mypolicy
```

Bonus: if trying to manually review audit logs, pipe them to `audit2allow -w` (`--why`) to get a much friendlier output.

You can also pipe `ausearch` (optionally filter for specific events, e.g., failures in the past 10 minutes with `-m avc -ts recent`) to `audit2allow` (`-w` if reviewing yourself).
