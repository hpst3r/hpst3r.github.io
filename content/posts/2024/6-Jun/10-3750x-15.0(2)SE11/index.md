---
title: "Configuring SSH clients for use with IOS 15.0(2)SE11 on a Cisco 3750x"
date: 2024-06-15T12:34:56-00:00
draft: false
---

# Intro

The 3750x is a very old switch. It does not support newfangled things like 'secure cryptographic algorithms.'

I'm not going to throw away a perfectly adequate gigabit L3 switch (IP Services!) and I would like to be able to use SSH to manage it (instead of squatting on the floor in the basement next to it with a mini USB cable every time I need to modify the config.)

This is documentation for the configuration changes required for client and server to get SSH working in my situation. This does not magically bring the SSH server on the 3750x up to modern standards, but it's better than Telnet.

2025 update - this has been SUPER helpful to have documented. These, or very similar exclusions, will allow you to use the Windows OpenSSH client to connect to most network gear and Linux servers dating back to the first decade of the 2000s.

# Quick Fix

The easiest fix: use Putty. It is unbothered by silly things like "security" and works out of the box.

# Overrides for OpenSSH

Tested with OpenSSH_9.6p1 on 15 Jun 2024.

You can either connect by specifying a bunch of options, or modify your .ssh/config file and setting the options either for a host or globally (if you have a few hundred switches, I could understand setting less secure global defaults.)

Anyway, on to the fixes.

## Want a one-liner to connect to a device?

```txt
ssh -o KexAlgorithms=+diffie-hellman-group1-sha1 -o HostKeyAlgorithms=+ssh-rsa,ssh-rsa-cert-v01@openssh.com -o PubkeyAcceptedAlgorithms=+ssh-rsa,ssh-rsa-cert-v01@openssh.com -o Ciphers=+aes256-cbc user@10.0.0.2
```

## Want to update your `~/.ssh/config`?

```txt
$ nvim ~/.ssh/config

# Make changes only for this host.
# We don't want to use less secure options when they're not needed.
# These are disabled by default for a good reason.

Host 10.0.0.2
    # Allow the use of sha1 key exchange.
    KexAlgorithms +diffie-hellman-group1-sha1

    # Allow the use of rsa host keys.
    HostKeyAlgorithms +ssh-rsa,ssh-rsa-cert-v01@openssh.com
    PubkeyAcceptedAlgorithms +ssh-rsa,ssh-rsa-cert-v01@openssh.com

    # Allow the use of insecure cbc ciphers.
    Ciphers +aes256-cbc
```

Now, on to the problems encountered & explanations! If you care.

### Problem 1 - no matching key exchange method found

```txt
$ ssh user@10.0.0.2
Unable to negotiate with 10.0.0.2 port 22: no matching key exchange method found. Their offer: diffie-hellman-group-exchange-sha1,diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
```

sha1 produces an insecure 160 bit hash. The algorithm has been defeated and has been considered insecure since 2005. Collision attacks (chosen-prefix) have been practical for the last few years (as of 2024) and sha1 has been phased out in most places.

### Problem 2 - no matching host key type found

```txt
$ ssh user@10.0.0.2
Unable to negotiate with 10.0.0.2 port 22: no matching host key type found. Their offer: ssh-rsa
```

Deprecated since OpenSSH 8.8. Enable them for the host in your `~/.ssh/config` or add the `-o HostKeyAlgorithms=+ssh-rsa` option to your `ssh` command.

### Problem 3 -  no matching cipher found

```txt
$ ssh admin@10.0.0.2
Unable to negotiate with 10.0.0.2 port 22: no matching cipher found. Their offer: aes128-cbc,3des-cbc,aes192-cbc,aes256-cbc
```

cbc ciphers are insecure and have been deprecated from OpenSSH since ~2014?

Enable them for the host in your `~/.ssh/config`, or add the `-c aes256-cbc` option to your `ssh` command.

### Problem 4 - Bad server host key: Invalid key length

```txt
$ ssh user@10.0.0.2
Bad server host key: Invalid key length
```

OpenSSH 9.6's default minimum RSA key size is 1024 bits and this cannot be lowered.

Fortunately, the 3750x running IOS 15 supports generating keypairs of between 360 and 4096 bits in length.

Receiving this error probably means you haven't configured the RSA key on the switch properly (in this case, I probably forgot to `crypto key generate rsa`), and the switch either lacks a host key or has a sub-1024 bit one.

Here's how to reconfigure the switch.

Log in to the switch with `telnet` or a console cable.

Enter enable and config mode. Wipe existing keys.

```txt
sw(config)#crypto key zeroize
% All keys will be removed.
% All router certs issued using these keys will also be removed.
Do you really want to remove these keys? [yes/no]: y
```

Make sure hostname and domain name are set, then generate new 2048+ bit rsa keys.

```txt
sw(config)#crypto key generate rsa
The name for the keys will be: sw.domain.local
Choose the size of the key modulus in the range of 360 to 4096 for your
  General Purpose Keys. Choosing a key modulus greater than 512 may take
  a few minutes.

How many bits in the modulus [512]: 4096
% Generating 4096 bit RSA keys, keys will be non-exportable...
[OK] (elapsed time was 253 seconds)

```

As shown in the command output above, this may take a while. The 400 mhz GPCPU in this thing is not a powerhouse. At all. It takes about 40 seconds to generate 2048 bit keys; 4096 bit keys take a few minutes (done for science).

Fun factoid: when connecting to the device with a 4096 bit keypair, it takes about 16 seconds for `ssh` to bring up the password prompt.

```txt
$ time ssh user@10.0.0.2
(user@10.0.0.2) Password: 


real    0m16.588s
user    0m0.010s
sys    0m0.004s
```