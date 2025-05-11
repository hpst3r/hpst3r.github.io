---
title: "Lazy PowerShell ssh-copy-id for Windows with SSH keys stored in 1Password"
date: 2025-05-11T16:30:00-00:00
draft: false
---

## Introduction

The version of OpenSSH bundled with Windows does not include the `ssh-copy-id` utility for quickly copying a SSH key to a remote machine.

I use 1Password's SSH agent, and store my SSH key in 1Password, so I can unlock it with biometrics (Windows Hello) instead of a passphrase, easily synchronize it between my machines, and avoid storing it on local disks. However, this means I can't just copy my `id_ed25519.pub` from one of my boxes using the 1Password SSH agent (because that file isn't created!)

Fortunately, 1Password has a command-line utility that can be used to access your vault items - Seriously, this is the best password manager I've ever used. It can even pose as a ghetto secret manager. Props to AgileBits - easily one of the most useful pieces of software I use.

So, while I don't have an `id_ed25519.pub`, and I don't have `ssh-copy-id`, I *do* have `op` (the 1Pass CLI utility), and I do have PowerShell. You can probably guess where this is going.

## Prerequisites

You'll need the 1Password CLI, which can be obtained either via Winget (as below) or by manually installing the binary from [AgileBits](https://app-updates.agilebits.com/product_history/CLI2).

```txt
~
❯ winget install AgileBits.1Password.CLI
Found 1Password CLI [AgileBits.1Password.CLI] Version 2.31.0
This application is licensed to you by its owner.
Microsoft is not responsible for, nor does it grant any licenses to, third-party packages.
Downloading https://cache.agilebits.com/dist/1P/op2/pkg/v2.31.0/op_windows_amd64_v2.31.0.zip
  ██████████████████████████████  8.65 MB / 8.65 MB
Successfully verified installer hash
Extracting archive...
Successfully extracted archive
Starting package install...
Command line alias added: "op"
Successfully installed

```

To get to the documentation for 1PW CLI, click the three dots on the top left of your 1Password client and select "Install 1Password CLI".

{{< figure src="images/op-dots-install-cli.png" >}}

Once you've installed 1Password CLI, enable integration from the desktop app (Settings > Developer > Command-Line Interface (CLI) > Integrate with 1Password CLI) so you can interactively authenticate via the desktop app:

{{< figure src="images/op-settings-integrate.png" >}}

Then, validate things are working by opening up your preferred shell (I'll be using PowerShell 7) and running a `op` command, like `op vault list`.

If you get an error about "connecting to desktop app: the pipe is being closed", the 1Pass release team probably forgot to sign the 1Password CLI app again.

```txt
~
❯ op vault list
[ERROR] 2025/05/11 14:17:06 connecting to desktop app: write: The pipe is being closed.
```

 You can confirm this by reviewing the 1Password logs (at `$env:LOCALAPPDATA\1Password\logs\1Password_rCURRENT.log`) and looking for an untrusted certificate error.

```txt
~
❯ gc $env:LOCALAPPDATA\1Password\logs\1Password_rCURRENT.log | select-string UntrustedFileCertificate

ERROR 2025-05-11T18:23:25.547+00:00 runtime-worker(ThreadId(21)) [1P:native-messaging\op-native-core-integration\src\lib.rs:647] Failed to accept new connection.: PipeAuthError(UntrustedFileCertificate(WinApi(Error HRESULT(0x800B0100): No signature was present in the subject.)))
ERROR 2025-05-11T18:23:53.327+00:00 runtime-worker(ThreadId(24)) [1P:native-messaging\op-native-core-integration\src\lib.rs:647] Failed to accept new connection.: PipeAuthError(UntrustedFileCertificate(WinApi(Error HRESULT(0x800B0100): No signature was present in the subject.)))
ERROR 2025-05-11T18:27:25.371+00:00 runtime-worker(ThreadId(21)) [1P:native-messaging\op-native-core-integration\src\lib.rs:647] Failed to accept new connection.: PipeAuthError(UntrustedFileCertificate(WinApi(Error HRESULT(0x800B0100): No signature was present in the subject.)))
```

In my case, I was able to resolve this by removing the latest release of the 1PW CLI (2.31.0) and reinstalling the prior release (2.30.3), which was properly signed:

```txt
~
❯ winget remove AgileBits.1Password.CLI

~
❯ winget install AgileBits.1Password.CLI --version 2.30.3

~
❯ op vault list
ID                            NAME
bd47xrtpknghhqruss4cppqwu4    Network
tregvgg3zxzildruwc52wrhb6q    Private
mswku5ai6dyyubsy6vxc3xyscu    Shared
```

## Doing a thing

Well, now that you have the `op` utility, you should now be able to access your SSH keys (stored in 1Password) with an `op item get` command, specifying the pubkey field (replace id_ed25519 with whatever you've named your primary public key):

```txt
~
❯ op item get id_ed25519 --fields 'public key'
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIC7QvzfPqlU1OKcyF8FRVoPNdSl+dJWcePUWM/Rn2+nZ
```

Awesome! We're like, halfway there.

Now let's throw together a quick command to modify the `~/.ssh/authorized_keys` file on our remote host.

```txt
$Command = @"
key="$(op item get id_ed25519 --fields 'public key')"
ssh_dir="`$HOME/.ssh"
auth_keys="`$ssh_dir/authorized_keys"
mkdir -p "`$ssh_dir"
touch "`$auth_keys"
chmod 700 "`$ssh_dir"
chmod 600 "`$auth_keys"
grep -qF "`$key" "`$auth_keys" || echo "`$key" >> "`$auth_keys"
"@

ssh 192.0.2.100 bash -c "'$Command'"
```

Let's break it down!

Define $Command as a multi-line string

```txt
$Command = @"
```

Define Bash variable `$key` as the result of PowerShell evaluation of `op item get id_ed25519`

```txt
key="$(& op item get id_ed25519 --fields 'public key')"
```

More Bash variables - saving repetition (I'm being lazy)

```txt
ssh_dir="`$HOME/.ssh"
auth_keys="`$ssh_dir/authorized_keys"
```

If ~/.ssh doesn't exist, create it

```txt
mkdir -p "`$ssh_dir"
```

If ~/.ssh/authorized_keys doesn't exist, create it

```txt
touch "`$auth_keys"
```

Set permissions on ~/.ssh so only the owner can dink with it

```txt
chmod 700 "`$ssh_dir"
```

Set permissions on ~/.ssh/authorized_keys so only the owner can dink with it

```txt
chmod 600 "`$auth_keys" # 6 not 7 as we don't need x on a file
```

Try to pattern match (`grep`) our key in authorized_keys:
- `-q`: silent, exit 0 if any match found
- `-F`: don't interpret special characters (e.g. * or \)

if specified key is not found, add it

```txt
grep -qF "`$key" "`$auth_keys" || echo "`$key" >> "`$auth_keys"
```

```txt
# terminate the multi-line string
"@
```

SSH to the remote host, and have Bash run our commands:

```txt
ssh 192.0.2.100 bash -c "'$Command'"
```

Note the escape characters (backticks) in the multiline string preventing PowerShell from evaluating our \$Variables, and note that we're surrounding our multiline string in a set of single quotes (as we're passing the whole string as an argument to `bash -c`). You can drop those single quotes in the string itself if you prefer. That's about all the complexity here.

To make sure this is working right, you can write the multiline string out in PowerShell. This is what we'll be passing to Bash on the remote host:

```txt
~
❯ Write-Host $Command
key="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIC7QvzfPqlU1OKcyF8FRVoPNdSl+dJWcePUWM/Rn2+nZ"
ssh_dir="$HOME/.ssh"
auth_keys="$ssh_dir/authorized_keys"
mkdir -p "$ssh_dir"
touch "$auth_keys"
chmod 700 "$ssh_dir"
chmod 600 "$auth_keys"
grep -qF "$key" "$auth_keys" || echo "$key" >> "$auth_keys"
```

The argument itself is quoted as ``"'$Command'"`` in our SSH command. This is just a plain PowerShell expanded string that, as mentioned previously, surrounds our multi-line string with single quotes.

Now, let's give it a try...

```txt
~
❯ $Command = @"
∙ key="$(op item get id_ed25519 --fields 'public key')"
∙ ssh_dir="`$HOME/.ssh"
∙ auth_keys="`$ssh_dir/authorized_keys"
∙ mkdir -p "`$ssh_dir"
∙ touch "`$auth_keys"
∙ chmod 700 "`$ssh_dir"
∙ chmod 600 "`$auth_keys"
∙ grep -qF "`$key" "`$auth_keys" || echo "`$key" >> "`$auth_keys"
∙ "@

~
❯ ssh wt14g2a bash -c "'$Command'"
liam@wt14g2a's password:

~ took 4s
❯ ssh wt14g2a
Activate the web console with: systemctl enable --now cockpit.socket
ss
Last login: Sun May 11 15:42:17 2025 from 192.168.77.1
liam@wt14g2a:~$ cat .ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIC7QvzfPqlU1OKcyF8FRVoPNdSl+dJWcePUWM/Rn2+nZ
liam@wt14g2a:~$ cowsay -s awesome
 _________
< awesome >
 ---------
        \   ^__^
         \  (**)\_______
            (__)\       )\/\
             U  ||----w |
                ||     ||
liam@wt14g2a:~$
```

Sweet. Looks like it works. Let's package it up into a PowerShell function:

```PowerShell
Function Copy-PubKey {
    param (
        [Parameter(Position = 0,
            Mandatory = $true)]
		[string]$RemoteHost,
		    
        [Parameter(Position = 1)]
        [string]$KeyName = "id_ed25519"
    )

    $Command = @"
key="$(op item get $($KeyName) --fields 'public key')"
ssh_dir="`$HOME/.ssh"
auth_keys="`$ssh_dir/authorized_keys"
mkdir -p "`$ssh_dir"
touch "`$auth_keys"
chmod 700 "`$ssh_dir"
chmod 600 "`$auth_keys"
grep -qF "`$key" "`$auth_keys" || echo "`$key" >> "`$auth_keys"
"@

    & ssh $RemoteHost bash -c "'$Command'"

}
```

Let's try it out!

```txt
~
❯ Copy-PubKey wt14g2a
liam@wt14g2a's password:

~ took 4s
❯ ssh wt14g2a
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Sun May 11 16:11:51 2025 from 192.168.77.1
liam@wt14g2a:~$ exit
logout
Connection to wt14g2a closed.

~ took 3s
❯
```

Well, that's what I was after! I'll be including this in my PowerShell profile so I have an alternative to `ssh-copy-id` that integrates with 1Password on Windows.

Obviously, it's not particularly versatile, but it does what I need.

[Here's the link to the GitHub repository](https://github.com/hpst3r/Copy-PubKey). As usual, the script in the post here might be a little outdated if I forget to come back and update it.