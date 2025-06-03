---
title: "SSH agent forwarding"
date: 2025-06-03T19:00:00-00:00
draft: false
---

SSH agent forwarding allows you to use a local SSH agent (your local keys, 1Password, etc) on a remote machine.

You can use this to "pass" authentication requests made by you on a remote session back to your local machine, or "delegate" authentication requests from your workstation to a jumpbox, depending on how you look at it.

To enable agent forwarding to a remote host, all you have to do is edit your local `~/.ssh/config` (or `~\.ssh\config`) file on your workstation with the keys, and add a `Host` declaration:

```txt
Host fedora42
    ForwardAgent yes
```

You can test this by trying to 'hop' across a jumpbox with your SSH key. If it's working, you'll be able to use your SSH key on the extra remote host.

I'll try this now, connecting to a jumpbox (`fedora42`) and then another server from it that only allows SSH key-based authentication (`alma10`).

First, without `ForwardAgent = Yes` for `fedora42`, I'll get rejected by `alma10`:

```sh
~
❯ hostname
windowsworkstation

~
❯ ssh fedora42

wporter@fedora42:~$ ssh alma10

wporter@alma10: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).

wporter@fedora42:~$ exit
logout
Connection to fedora42 closed.

~ took 7s
❯ hostname
windowsworkstation
```

Now, with `ForwardAgent = Yes` for `fedora42`:

```sh
~
❯ hostname
windowsworkstation

~ took 8s
❯ ssh fedora42

wporter@fedora42:~$ ssh alma10

[wporter@alma10 ~]$ exit
logout
Connection to alma10 closed.

wporter@fedora42:~$ exit
logout
Connection to fedora42 closed.

~ took 7s
❯ hostname
windowsworkstation
```

This works with any local SSH agent. For example, I'm using 1Password, so my keys can just live there and be happy. I'll get a Windows Hello prompt to authorize a key I try to use on a jumpbox. Pretty nice, huh?
