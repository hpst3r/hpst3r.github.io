---
title: "Requiring multiple, different factors for Windows Hello console authentication"
date: 2024-12-01T12:34:56-00:00
draft: false
---

NOTE: this works with normal Windows Hello without any backing infrastructure if you just want to require two Hello factors to sign on to a machine - this is how I use it (make a convenience PIN more secure.)
NOTE: yes, Hello for Biz is MFA on its own, kind of.

By setting up first factor to be biometric or a PIN, then setting the second factor to be proximity or a PIN, I can make sure that I'm able to sign in with Hello when my phone is dead by passing biometric authentication and then entering my PIN.

It also makes signing in with biometric authentication quick, since it uses my Dynamic Lock device (my smartphone) as a second factor.

# Group Policy
Computer Configuration > Administrative Templates > Windows Components > Windows Hello for Business
1. Configure device unlock factors:
	1. Enabled
	2. First unlock factor credential providers:
		`{D6886603-9D2F-4EB2-B667-1971041FA96B},{BEC09223-B018-416D-A0AC-523971B639F5},{8AF662BF-65A0-4D0A-A540-A338A999D36F}`
		`{PIN},{Fingerprint},{Face ID}`
	3. Second unlock factor credential providers:
		`{27FBDB57-B613-4AF2-9D7E-4FA7A66C21AD},{D6886603-9D2F-4EB2-B667-1971041FA96B}`
		`{Device Proximity (bluetooth dynamic lock device)},{PIN}`

# Registry
TBD - I have it written down somewhere or I can diff the registry. Someday.

Either https://petervanderwoude.nl/post/excluding-the-password-credential-provider/ (breaks stuff including UAC)
Or Interactive logon: Require Windows Hello for Business or smart card (Computer Configuration > Windows Settings > Security Settings > Local Policies > Security Options) REQUIRES ENTRA or YOU WILL BE LOCKED OUT (guess how I found out)

Or set the password to a 64 character string and rotate it without user input?

Don't know what would be best to use in production. Without making it a little friendlier to set up a smartphone or customers willing to pay up for Hello webcams and fingerprint sensors, I probably won't be using this, but it's neat.

I've just set my password to something long and complex and just use a combination of PIN and my smartphone (Dynamic Lock paired device). This has worked well for me and I've been using it on my main machine for a month or two.