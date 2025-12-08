---
title: "basic Postfix config"
date: 2025-12-08T00:30:00-00:00
draft: false
---

AlmaLinux 10.1 6.12.0 on amd64 (host "3060t0") with Postfix 3.8.5-8.el10

Postfix is a Mail Transfer Agent (MTA) - the component of a "mail server" that primarily handles routing and relaying email between mail servers. In other words, it is a lightweight utility capable of routing (and sending) mail.

Postfix does:

- Send mail
- Receive mail
- Route mail between systems
- Queue messages for delivery
- Retry failed deliveries
- Relay outbound mail

Postfix doesn't:

- Authenticate users for mailbox access
- Provide a webmail interface
- Do spam filtering

Postfix is the de-facto standard for a MTA on a Linux system - it's secure (by design), easy to configure, rather lightweight, and widely available. This all makes it an excellent choice for sending mail out from infrastructure for things like reporting, alerts, or notifications.

While hosting your own mail server can be an undertaking, getting Postfix configured to listen on localhost and send mail alerts from your systems is rather straightforward.

## Quick resources

### Command reference

```sh
sudo postfix check                    # validate configuration
sudo postconf parameter               # show the value of a config parameter
sudo postconf -e 'parameter = value'  # set a parameter in the configuration

sudo postqueue -p                     # show the queue
sudo postqueue -f                     # flush (send all in) the queue
sudo postsuper -d QUEUE_ID            # delete a message
sudo postcat -q QUEUE_ID              # view a message

sudo tail -f /var/log/maillog         # follow the log file
sudo pflogsumm /var/log/maillog       # log summary - requires addl. package
```

### tl;dr config for outbound mail

```sh
sudo dnf install -y postfix # install postfix mta
sudo systemctl enable --now postfix # start and enable the service
sudo postconf -e "myhostname = $(hostname -f)" # the system FQDN
sudo postconf -e "myorigin = $(hostname -d)" # the system domain name - adjust to suit
sudo systemctl reload postfix # update the running config
```

Don't forget to configure DMARC and SPF.

## Detailed setup

I'll be installing this on an AlmaLinux 10 system (RHEL-like).

Installing the `postfix` package will install the MTA, provide the `/usr/sbin/sendmail` utility, associated management tools, populate the `/etc/postfix` configuration directory, and generate the `/var/spool/postfix` queue directories.

```sh
sudo dnf install postfix
```

Enable and start the service:

```sh
sudo systemctl enable --now postfix
```

For a minimal configuration that allows you to effectively send outbound mail from one system, you're already set!

However, I like to edit the `/etc/postfix/main.cf` configuration file to specify the hostname (`myhostname` - this shows in the headers and makes them easier to read) and origin (`myorigin` - this sets the implicit domain in sender addresses, which will make mail come from user@myorigin rather than user@localdomain or something, which will get black-holed, if you forget to specify the full address).

I'll set `myorigin` via `$mydomain`, a convenience variable, just to keep things a little organized.

```ini
myhostname = 3060t0.lab.wporter.org
mydomain   = lab.wporter.org
myorigin   = $mydomain
```

If your server is exposed to the Internet and you do not want it to receive mail, I would recommend setting the `mynetworks` variable so that Postfix listens only on localhost.

If you want to open up your machine to the local network as a SMTP server, you'll obviously want to do the inverse and specify your internal network.

I would recommend configuring authentication (and TLS, of course) if you intend to open up Postfix to more than localhost so your server doesn't become a spam bot.

```ini
inet_interfaces = all
mynetworks = 127.0.0.0/8
```

If you want all mail to go through another mail server rather than shooting straight out to the Internet, configure the `relayhost` property. You may have to configure SASL authentication and TLS (`smtp_use_tls = yes`) for this, depending on the server you're relaying to.

```txt
relayhost = [mailserver.purelymail.com]:587
```

## Email authentication

I will not be going into SPF and DMARC in great detail here; I'll just be briefly touching on them since they're relevant (without them, your mail probably won't be going anywhere).

If you do not know why this is, have never configured SPF or DMARC before, and would like to learn about them, Cloudflare has a nice brief and information on how each type of authentication record (SPF, DKIM, DMARC) works [accessible here (cloudflare.com)](https://www.cloudflare.com/learning/email-security/dmarc-dkim-spf/).

Postfix doesn't have built-in support for DKIM (cryptographically signing emails so recipients know the contents haven't been tampered with), so I don't tend to bother with it for this use case.

### SPF

The "Sender Policy Framework" allows you to say what Internet addresses are allowed to send mail for your domain. This definition is stored in DNS as a TXT record, with a format that looks something like:

```txt
"v=spf1 include:_spf.purelymail.com ip4:64.224.254.164 -all"
```

This essentially says "only servers with an IP address in PurelyMail's `_spf` record or 64.224.254.164 are allowed to send email from this domain, and everything else is invalid."

For most use cases, creating a restrictive SPF record for a subdomain (e.g., my infrastructure can only send out as "lab.wporter.org") that specifies your Internet addresses is sufficient.

Alternatively, a clean way to restrict sending to your mailservers and reduce repetition is to specify a SPF record targeting `mx`:

```txt
"v=spf1 mx -all"
```

This is equivalent to the A records that correspond to any MX records for the domain.

SPF is inherited *if* nothing is configured for the subdomain. If a SPF record *is* configured for a subdomain, that record overrides anything configured at a higher level.

### DMARC

"Domain-based Message Authentication, Reporting, and Conformance", or DMARC, is a security standard that essentially lets you tell mail servers that receive your email what they should do in case an unauthenticated email arrives - for example, reject, quarantine, or do nothing. It can do a bit more, but that's outside the scope of this quick writeup.

All you need to know is that:

- a SPF record pointed at your mail server is sufficient to get your email to pass DMARC
- you should probably tell recipients getting unauthenticated mail from you to reject it

To do the latter, it'll be something along the lines of a TXT record for your domain of choice called `_dmarc` with the contents:

```txt
"v=DMARC1; p=reject;"
```

Easy! This says:

- I'm a DMARC record
- If a message purportedly from this domain (or any subdomains) doesn't pass authentication, you should reject it

## Sending a test mail

You can use the `sendmail` utility to send a mail from your shell. For example:

```sh
sendmail liam@wporter.org <<EOF
From: William Porter <liam@lab.wporter.org>
To: William Porter <liam@wporter.org>
Subject: Test mail

Hi there,

This is a test mail from Postfix.

EOF
```

Include headers and a subject - without them, your mail is likely to be marked as spam, even if it passes authentication.

### Greylisting

If you don't immediately receive your test mail, check the maillog and postqueue.

```sh
$ sudo postqueue -p

120AD84081E     293 Sun Dec  7 23:27:14  wporter@lab.wporter.org
(host mailserver.purelymail.com[18.204.123.63] said: 451 4.7.1 Try again later (greylist) (in reply to RCPT TO command))
                                         liam@wporter.org
```

```sh
$ sudo tail /var/log/maillog

Dec  7 23:27:14 3060t0 postfix/smtp[131449]: 120AD84081E: to=<liam@wporter.org>, relay=mailserver.purelymail.com[18.204.123.63]:25, delay=0.24, delays=0.03/0.03/0.11/0.07, dsn=4.7.1, status=deferred (host mailserver.purelymail.com[18.204.123.63] said: 451 4.7.1 Try again later (greylist) (in reply to RCPT TO command))

```

Your messages will likely show `status=deferred` and "Try again later (greylist)" when you start sending outbound mail. This is a standard behavior and nothing to be concerned about.

When a receiving mail server first sees a new combination of sending mailserver, sender, and recipient, it'll likely ask your mail server to please wait and try again later. Spamming mail servers are less likely to behave properly and obey this request.

### Headers

Here are some example headers showing mail flow:

```txt
X-Spam-Status: No, score=1.19 required=5 tests=DMARC_NONE=0.898,RCVD_IN_VALIDITY_RPBL=1.284,RCVD_IN_VALIDITY_SAFE=-2,SPF_HELO_NONE=0.001,TVD_RCVD_IP=0.001,TVD_RCVD_IP4=0.001,TXREP=0.221,PURELYMAIL_IP_REPUTATION=0.79 autolearn=no autolearn_force=no version=4.0.1
X-Pm-Spam-PURELYMAIL-IP-REPUTATION: 0.79;EffectiveIp=64.224.254.164,SpamProbability=0.28
Return-Path: <wporter@lab.wporter.org>
Delivered-To: liam@wporter.org
X-Pm-Original-To: liam@wporter.org
Authentication-Results: purelymail.com; spf=pass (domain of lab.wporter.org designates 64.224.254.164 as permitted sender) smtp.mailfrom=lab.wporter.org; dmarc=pass (p=reject) header.from=liam@lab.wporter.org
Received: from 64.224.254.164 (EHLO 3060t0.lab.wporter.org) ([64.224.254.164])
          by smtp.purelymail.com (Purelymail SMTP) with ESMTPS id -1073040572
          for <liam@wporter.org>
          (version=TLSv1.3 cipher=TLS_AES_256_GCM_SHA384);
          Mon, 08 Dec 2025 05:21:03 +0000 (UTC)
Received: by 3060t0.lab.wporter.org (Postfix, from userid 1000)
  id 395F6840815; Mon,  8 Dec 2025 00:21:03 -0500 (EST)
```

Read from the bottom up:

- The message was received by Postfix from user 1000 (`wporter@3060t0.lab.wporter.org`)
- The message was received by `smtp.purelymail.com` from `64.224.254.164` (who is registered in `lab.wporter.org`'s SPF record, said hello as `3060t0.lab.wporter.org`, passed DMARC, and sent along a message that says it's from `liam@lab.wporter.org`)

## That's all

You should now have a way to send mail from your machine (or machines, if you decide to open it up). Wasn't that easy?

You can use the `sendmail` command from the shell, in a script, or wherever else you may fancy on this machine you've set up with Postfix.
