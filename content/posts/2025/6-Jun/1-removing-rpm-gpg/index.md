---
title: "Listing and removing RPM GPG (PGP) keys"
date: 2025-06-03T18:30:00-00:00
draft: false
---

## List keys

Our first goal will be to list the installed PGP public keys in the format we need, with a human-readable summary, as follows:

```sh
# structure:
gpg-pubkey-version-release My Organization <email@address.org> public key
# ex:
gpg-pubkey-105ef944-65ca83d1 Fedora (42) <fedora-42-primary@fedoraproject.org> public key
```

To do so, you can use the `rpm` command with arguments '`--query --queryformat`' (this may be abbreviated to '`-q --qf`').

We'll be querying RPM for `gpg-pubkey`(s), and we'll need to specify a queryformat that will get us the rpm-format shorthand identifier for a key:

```sh
fedora42:~$ rpm -q gpg-pubkey --queryformat "%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n"
gpg-pubkey-e99d6ad1-64d2612c    Fedora (41) <fedora-41-primary@fedoraproject.org> public key
gpg-pubkey-957f5868-5e5499b8    Tailscale Inc. (Package repository signing key) <info@tailscale.com> public key
gpg-pubkey-105ef944-65ca83d1    Fedora (42) <fedora-42-primary@fedoraproject.org> public key
gpg-pubkey-2012ea22-591e021e    Code signing for 1Password <codesign@1password.com> public key
```

## Remove keys

Now that we have our identifiers, we can attempt to remove a key.

I'm going to remove the `gpg-pubkey-2012ea22-591e021e` key from 1Password. To do so, we'll use `rpm --erase`:

```sh
fedora42:~$ sudo rpm --erase --allmatches gpg-pubkey-2012ea22-591e021e
```

We can confirm it's gone:

```sh
fedora42:~$ rpm -q gpg-pubkey --qf "%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n"
gpg-pubkey-e99d6ad1-64d2612c    Fedora (41) <fedora-41-primary@fedoraproject.org> public key
gpg-pubkey-957f5868-5e5499b8    Tailscale Inc. (Package repository signing key) <info@tailscale.com> public key
gpg-pubkey-105ef944-65ca83d1    Fedora (42) <fedora-42-primary@fedoraproject.org> public key
```

## Query format arguments

To explain the queryformat arguments, we'll need to see a key:

```sh
fedora42:~$ gpg --show-keys --with-colons /etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-41-primary
pub:-:4096:1:D0622462E99D6AD1:1691509036:::-:::escESC::::::23::0:
fpr:::::::::466CF2D8B60BC3057AA9453ED0622462E99D6AD1:
uid:-::::1691509036::7C99785914FC918D16935F0E5205B36A06F5B5F3::Fedora (41) <fedora-41-primary@fedoraproject.org>::::::::::0:
```

The `--queryformat` arguments used to create our shorthand `gpg-pubkey-x-y` here are:

1. `NAME`, the type of key (`gpg-pubkey`)
2. `VERSION`, the last eight hex digits of the fingerprint (last 32 bits)
   1. `E99D6AD1` is the last section of the public key above
3. `RELEASE`, a timestamp for the key's creation in hex (`1691509036` -> `64D2612C`), found
   1. At the end of the first part of the `pub` line (`:1691509036:::`)
   2. In the first part of the UID (`uid:-::::1691509036)
   3. This one means 8/8/2023
4. `SUMMARY`, a human-readable label for the key

You can see a full list of available `--queryformat` arguments with `rpm --querytags`:

```sh
fedora42:~$ rpm --querytags | head -n 3
ARCH
ARCHIVESIZE
ARCHSUFFIX
```
