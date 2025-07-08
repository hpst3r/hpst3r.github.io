---
title: "Why doesn't Windows support modern cryptography?"
date: 2025-07-01T19:35:00-00:00
draft: false
---

Try to feed Windows a certificate with an Ed25519 signature somewhere in the chain and it'll just FALL OVER because [CNG](https://learn.microsoft.com/en-us/windows/win32/seccng/) doesn't support Ed25519 AT ALL?

This is an operating system that is 'modern' and 'current' and 'has AI'.

Here! Watch it fall over when I try to do anything with a cert:

![System level error occurred while verifying trust X509 cert signed with Ed25519 Windows 11 24H2 .crt dialog](image.png)

God forbid you want to use something with an Ed25519 signature in the chain:

```cmd
C:\Users\liam>curl https://test/
curl: (35) schannel: next InitializeSecurityContext failed: SEC_E_INVALID_TOKEN (0x80090308) - The token supplied to the function is invalid
```

```sh
liam@liam-z790-0:~$ curl https://test
404 page not found
```

This is a "Modern Operating System" alright. I guess Microsoft [couldn't afford](https://www.google.com/search?q=microsoft+market+cap) to have one of their [extremely small team](https://news.microsoft.com/facts-about-microsoft/#EmploymentInfo) add features to their flagship product at any point during the past thirteen years.

But there's an AI button now!

God forbid we mention Windows Server, with its decaying corpse sitting over in one of the basements at Redmond.
