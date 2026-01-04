---
title: "Windows Encrypted File System basics"
date: 2026-01-04T17:45:00-00:00
draft: false
---

BitLocker is great, but is primarily useful for preventing tampering when the system is offline. Once your system is online, a sufficiently privileged user could write to or read from anywhere on the disk. The Encrypted File System (EFS) exists to mitigate this.

EFS, a capability of NTFS (since Windows 2000) allows you to mark files or directories for transparent encryption.

This is not to be confused with Windows Personal Data Encryption, an Enterprise-edition feature tied to Windows Hello for Business introduced with Windows 11, version 24H2. PDE is similar, requires Windows 11 Enterprise (e.g., M365 E3/E5) and uses WHfB keypairs rather than user certificates (and thus requires Entra join and Windows Hello logon, and thus doesn't work via RDP).

When you first use EFS, Windows generates a RSA keypair saved on disk as a certificate in the User\My certificate store. It's unlocked when you log on (with your password).

> **CAUTION!!** This means that if your password is reset (e.g., an 'unkind' lusrmgr.mmc reset, or PowerShell) you may lose access to the private key! Make a backup of the private key and/or be sure you can restore encrypted files from a backup before storing anything important with EFS!

Again, as Windows will tell you, **BACK UP YOUR ENCRYPTION KEY** before storing anything important in EFS-encrypted files!

{{< figure src="images/efs-0-ui-warning.png" >}}

So, in essence, EFS is a NTFS/Windows syscall feature that transparently (to user/applications) decrypts entire files at read time so they are never directly accessible on disk without a keypair stored in the user's certificate store.

It *is* likely possible to get at these files without the system online/without having access to the user profile (see [this post on tinyapps.org](https://tinyapps.org/docs/decrypt-efs-without-cert-backup.html)) but it is obviously much more difficult to do so than if they were sitting on disk in plaintext.

This is not to be confused with two other Windows encryption features that are similar, but do not do exactly the same thing:

- Personal Data Encryption is a Windows Enterprise-exclusive feature that uses Windows Hello for Business for authentication (it does not generate a separate keypair, and it's easier to recover data thanks to WHfB's cloud authorization)
- BitLocker is a standard Windows feature that encrypts the drive itself. When a drive is encrypted with BitLocker, the entire drive is "unlocked" at boot, similarly to LUKS or ZFS native encryption on a Linux system (and unlike EFS, which transparently decrypts files as they're read).

EFS encryption doesn't hammer the computer; on a Ryzen 5 3500U running Windows 11 24H2 I saw about 5% CPU and 12 MB/s of disk from LSASS (highly scientific monitoring with Task Manager) after toggling the property on a ~3 GiB directory of small files.

## Encrypt a directory from Explorer

To toggle file encryption for a specific file or directory on a NTFS volume in Windows 10/11 from the GUI:

Right-click an item, and click Properties in the context menu:

{{< figure src="images/efs-1-toggle-explorer.png" >}}

In the Properties menu, click Advanced:

{{< figure src="images/efs-2-toggle-explorer.png" >}}

Then, under "Advanced Attributes", check the "encrypt contents to secure data" option and click OK (in the Advanced window, and on the Properties window).

{{< figure src="images/efs-3-toggle-explorer.png" >}}

Confirm your changes. Select your desired recursion setting.

{{< figure src="images/efs-4-toggle-explorer.png" >}}

Then, Windows will encrypt the directory. This may take a while depending on your machine and the directory's contents.

{{< figure src="images/efs-5-toggling.png" >}}

That's it! Encryption is now enabled.

Once EFS has been toggled, items in the directory will show a special 'locked' icon, e.g.:

{{< figure src="images/efs-6-locked.png" >}}

Otherwise, encryption is transparent, and handled by the EFS subsystem, which operates in the context of LSASS.

To decrypt the directory, do the same thing but uncheck the 'encrypt contents' box. Alternatively, you can quickly decrypt a directory or file from the Explorer context menu:

{{< figure src="images/efs-7-decrypt-explorer.png" >}}

{{< figure src="images/efs-8-decrypt-explorer.png" >}}

## cipher CLI utility

You can also encrypt files or directories with the `cipher` utility.

`cipher` will list items in a directory, and their encryption status. Use the `/?` parameter for help.

`cipher /E` will encrypt a file or directory (if you just encrypt a directory with `/E`, only new items will be encrypted). **If you want to recursively encrypt a directory, use `cipher /E /S`.

### Encryption

```txt
PS C:\Users\liam> cipher /e notes

 Encrypting files in C:\Users\liam\

notes               [OK]

1 file(s) [or directorie(s)] within 1 directorie(s) were encrypted.
```

```txt
PS C:\Users\liam> cipher

 Listing C:\Users\liam\
 New files added to this directory will not be encrypted.

U Contacts
U Desktop
U Documents
U Downloads
U Favorites
U Links
U Music
E notes
U Pictures
U Saved Games
U Searches
U Videos
```

```txt
PS C:\Users\liam> sl notes
PS C:\Users\liam\notes> cipher

 Listing C:\Users\liam\notes\
 New files added to this directory will be encrypted.

U .$opnsense-carp-one-ip-lab.png.bkp
U .gitignore
U .obsidian
U 802.1d 802.1w refresher.md
U Books to read.md
```

```txt
    /E        Encrypts the specified files or directories. Directories will be
              marked so that files added afterward will be encrypted. The
              encrypted file could become decrypted when it is modified if the
              parent directory is not encrypted. It is recommended that you
              encrypt the file and the parent directory.
```

### Recursive operations

The `/S` parameter tells cipher to perform the given operation recursively.

```txt
PS C:\Users\liam> cipher /E /S:notes

 Setting the directory notes to encrypt new files [OK]

 Encrypting files in C:\Users\liam\notes\

.$opnsense-carp-one-ip-lab.png.bkp [OK]
.gitignore          [OK]
.obsidian           [OK]
802.1d 802.1w refresher.md [OK]

 Encrypting files in C:\Users\liam\notes\.git\

config              [OK]
description         [OK]
HEAD                [OK]
hooks               [OK]
index               [OK]
info                [OK]
logs                [OK]
objects             [OK]
packed-refs         [OK]
refs                [OK]

-- snip --

2627 file(s) [or directorie(s)] within 346 directorie(s) were encrypted.

Converting files from plaintext to ciphertext may leave sections of old
plaintext on the disk volume(s). It is recommended to use command
CIPHER /W:directory to clean up the disk after all converting is done.
```

You should heed the warning at the end of a `cipher /E /S:dir` run and run `cipher /W:dir` afterwards to ensure no remnants of plaintext files are left on disk.

```txt
    /W        Removes data from available unused disk space on the entire
              volume. If this option is chosen, all other options are ignored.
              The directory specified can be anywhere in a local volume. If it
              is a mount point or points to a directory in another volume, the
              data on that volume will be removed.
```

### Decryption

To decrypt a file or directory, use `cipher /D`. This can, of course, be combined with `cipher /S:dir` for recursive decryption.

```txt
PS C:\Users\liam> cipher /D /S:notes

-- snip --

2627 file(s) [or directorie(s)] within 346 directorie(s) were decrypted.
```

### Managing keys

The certificate used by EFS is stored in the Current User\Personal (My) certificate store. Windows always uses the ***most recent*** certificate for "new" EFS operations (e.g., writing a new encrypted file). Existing files retain their original FEK until 'touched' (e.g., /U is run.)

```txt
PS C:\Users\liam> ls Cert:\CurrentUser\My

   PSParentPath: Microsoft.PowerShell.Security\Certificate::CurrentUser\My

Thumbprint                                Subject              EnhancedKeyUsageList
----------                                -------              --------------------
67767A40FBB66045B8A10386CBD5CB527573F23D  CN=liam              Encrypting File System
```

{{< figure src="images/efs-9-certlm-efs-cert.png" >}}

To print your active (default) EFS cert's thumbprint:

```txt
PS C:\Users\liam> cipher /y

EFS certificate thumbprint for computer PF1K1CRK:

  6776 7A40 FBB6 6045 B8A1 0386 CBD5 CB52 7573 F23D
```

As far as I am aware, there is no supported user-facing way to determine which EFS certificate was used to encrypt a given file.

#### Rekeying

To update your keypair, run `cipher /K` to generate a new one, then `cipher /U` to update all the encrypted files on disk with the new encryption key.

```txt
PS C:\Users\liam> ls Cert:\CurrentUser\My

   PSParentPath: Microsoft.PowerShell.Security\Certificate::CurrentUser\My

Thumbprint                                Subject              EnhancedKeyUsageList
----------                                -------              --------------------
67767A40FBB66045B8A10386CBD5CB527573F23D  CN=liam              Encrypting File System

PS C:\Users\liam> cipher /K

EFS certificate thumbprint for computer PF1K1CRK:

  1B24 9BE5 759B F0AD FF1E 9CAC D5FE D53F CAF4 A50E

PS C:\Users\liam> ls Cert:\CurrentUser\My

   PSParentPath: Microsoft.PowerShell.Security\Certificate::CurrentUser\My

Thumbprint                                Subject              EnhancedKeyUsageList
----------                                -------              --------------------
67767A40FBB66045B8A10386CBD5CB527573F23D  CN=liam              Encrypting File System
1B249BE5759BF0ADFF1E9CACD5FED53FCAF4A50E  CN=liam              Encrypting File System

PS C:\Users\liam> cipher /U
C:\$Recycle.Bin\S-1-5-21-25087275-3701594579-3890375949-1000\$RNGYPZP.txt: Encryption updated.

-- snip --
```

You should then be able to remove the previous keypair (certificate).

```txt
    /K        Creates a new certificate and key for use with EFS. If this
              option is chosen, all the other options will be ignored.

              Note: By default, /K creates a certificate and key that conform
                    to current group policy. If ECC is specified, a self-signed
                    certificate will be created with the supplied key size.
```

```txt
    /U        Tries to touch all the encrypted files on local drives. This will
              update user's file encryption key or recovery keys to the current
              ones if they are changed. This option does not work with other
              options except /N.
```

`/N`, since it's mentioned:

```txt
    /N        This option only works with /U. This will prevent keys being
              updated. This is used to find all the encrypted files on the
              local drives.
```

### Backing up keys

You can export EFS certificates with `certlm.msc`:

{{< figure src="images/efs-10-export-cert.png" >}}

**Be sure to export the private key!** The public key is useless on its own.

{{< figure src="images/efs-11-export-cert-privkey.png" >}}

{{< figure src="images/efs-12-export-pkcs.png" >}}

You should encrypt the private key with AES256-SHA256 rather than 3DES/SHA1. 3DES/SHA1 are deprecated and should not be used if at all avoidable as both algorithms are cryptographically insecure.

{{< figure src="images/efs-13-aes256-sha256.png" >}}

### Data Recovery Agents

The Data Recovery Agent (DRA) is a special EFS keypair that allows an administrator to decrypt EFS-encrypted files if a user’s EFS private key is lost.

This does NOT exist by default on a standalone (workgroup) system utilizing EFS that runs Windows XP or later. If you would like, you can create a separate DRA certificate. This Data Recovery Agent's public key will be used in addition to the user's public key to encrypt files so that if the user certificate is lost, their files may be restored.

Files encrypted before a DRA is configured cannot be recovered by that DRA unless they are re-encrypted.

This is NOT associated with the local Administator account and should not be stored on the machine.

You can use the `cipher /R` option to generate a Recovery Agent certificate:

```txt
    /R        Generates an EFS recovery key and certificate, then writes them
              to a .PFX file (containing certificate and private key) and a
              .CER file (containing only the certificate). An administrator may
              add the contents of the .CER to the EFS recovery policy to create
              the recovery key for users, and import the .PFX to recover
              individual files. If SMARTCARD is specified, then writes the
              recovery key and certificate to a smart card. A .CER file is
              generated (containing only the certificate). No .PFX file is
              generated.

              Note: By default, /R creates an 2048-bit RSA recovery key and
                    certificate. If ECC is specified, it must be followed by a
                    key size of 256, 384, or 521.
```

According to Microsoft, if a Windows system is joined to a domain, the first domain controller's built-in Administrator profile will contain a domain Data Recovery Agent keypair. However, this information is dated (up to Server 2008) and I have not determined whether it's accurate for Server 2022/2025 (FL 2016/2025) domains.

This is *only* stored on the first domain controller and may be exported from the Administrator account's Personal certificate store.

If that certificate is lost and no additional DRAs are configured, EFS recovery for the domain may be impossible.
