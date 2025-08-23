---
title: "Microsoft MD-102 (Endpoint Administrator): Pass!"
date: 2025-08-23T12:00:00-00:00
draft: false
---

## My thoughts on the certification

I passed the Intune test!

It's been about a year since I started working at a shop that uses Intune (a bit - for iOS and Macs), so on a whim I decided I should get the cert. It took me about three weekends, plus a few hours of study during the week.

I had a decent bit of experience with Intune, but don't use it daily.

Overall, it was a pretty fair test. The MD-102 demands a broad *and* deep understanding of Intune (including the Intune Suite), across every common platform.

You will need to be curious and lab if you intend to pass the exam (though, really, you should always be curious and lab, lab, lab - no matter the cert you're chasing!)

There were, of course, some curveballs, and questions I wasn't sure of (I haven't used the Intune Suite much at all - it's ruinously expensive!)

Any of the 'unfair' questions can be remedied with a detour to RTFM (documentation for the product you're testing on is accessible during Microsoft exams via Microsoft Learn).

[The exam objectives](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/md-102#skills-measured-as-of-september-17-2024) are a good representation of the content covered.

In conclusion - nothing too difficult, though not particularly easy. The MD-102 is definitely a mid-level exam, and demonstrates a good understanding of the Intune product and Microsoft 365.

I would expect a MD-102 candidate to be able to do a MDM deployment solo, without mentoring or guidance - or jump right into an existing cloud-managed environment with no fuss and understand the day-to-day stuff.

One would also expect a MD-102 candidate to understand (more or less) how Microsoft 365 works, and have at least a basic grasp on Windows system administration (as that's probably what you'll be doing).

The exam doesn't test you on PowerShell and doesn't have much to do with on-prem Active Directory. That said, a prospective Windows system administrator who doesn't know PowerShell is pretty useless, so I'd recommend picking up PowerShell if you intend to administer Windows machines.

## Study resources

### Recommendations for hardware/licensing in your lab

Get at least a Microsoft 365 Business Premium trial. This should have all the licensing you realistically need.

You will want at least a Windows machine that can run a few Windows VMs. This might just mean getting some more memory for your desktop. One VM at a time would be fine.

You'll also need at least one kind of mobile device (iOS or supported Android Enterprise).

You can probably make one of either category work, but I'd recommend getting an Android if you have any kind of experience with iOS and Apple Device Enrollment (ABM or ASM) as the Android enrollment modes are neat, and very different from the single mode you get for iPhones.

I wouldn't bother messing with the Intune Suite unless you really want some hands-on with one of the products for a potential deployment at work.

### My resources

I used:

- The free Microsoft MD-102 Learning Modules (though they weren't really of much use and haven't been updated for the new version of the MD-102 from September 2024)
- The MeasureUp MD-102 practice test (outdated, but OK - not sure how much I liked it. This is not a glowing recommendation)
- Anki flashcards
- The Intune product documentation
- A lot of labbing with Windows and Android (VMs and a Pixel 6a).

### Study experience

It took me approximately three weekends to study for the exam. I passed with a 900/1000 and had plenty of time.

To study Windows management, I deployed machines with Autopilot, mucked about with Update Rings, Feature Update policies, security baselines and compliance policies, and had some fun with Defender for Endpoint.

I worked with a hybrid AD lab, deployed WHfB, the Intune connector for AD, and all that fun stuff to supplement the cloud-only knowledge.

Be sure you understand the ways you can join/register devices to Microsoft Entra ID. Understand how Windows Hello for Business works, and, ideally, do a hybrid WHfB deployment so you understand how it can be done. WHfB should be deployed EVERYWHERE in the real world - it's an excellent solution. Implement Windows LAPS (it'll only take a few minutes).

I did not study with Windows 365. I don't care for the cloud VDI stuff and they're just Windows machines in the end.

Also, be sure to take a look at Microsoft Defender for Endpoint - the basics are within the scope of the MD-102 certification. I configured automatic enrollment via Intune and GPO, ran some malware on a test device, ran some EDR testing tools on a test device, and mucked about with compliance policies - but didn't go too deep. Save that for your MS-102.

Defender for Endpoint seems like a good product - far easier to use than SentinelOne, and the relevant parts of the console are much more responsive and much less frustrating to use. Microsoft has successfully sold me on giving it a closer look if I need an endpoint security product.

For Android, I used a physical Pixel 6a - I wiped it a few times, set up MAM, lived with MAM in my work tenant for a little while (very nice to be able to do this), and deployed it in every Android Enterprise mode.

You can do *some* things with Android Studio emulated devices or Waydroid, but you'll really want a physical device to play around with the assorted fully managed deployment modes. If you can't get your hands on an Android, be sure to watch someone else lab it out and take notes.

You don't need two platforms to lab mobile application management - one is plenty. MAM works nearly identically on Android and iOS.

I've worked with iOS and Mac OS at $DayJob, so, while I didn't have a Mac and iPhone on hand to lab with, I knew the basics.

If you have no experience managing Mac OS or iOS with Intune, I *highly* recommend buying or borrowing at least an old iPhone or iPad (and a Mac that can run Apple Configurator so you can enroll the things), standing up an Apple Business Manager tenant, and going through the motions so you know how that works, too. At a bare minimum, watch someone else lab it out and take notes.

If you don't have experience deploying software via Intune or configuring Conditional Access, be sure to lab those out, too. They're not difficult topics, but both are quite important in the real world.

I did not lab MDE on mobile platforms, though I'd like to play around with that some day.

I felt I was adequately prepared for the exam. If I was to do it again, I'd probably have sat the exam a week earlier.

Anyway, I think that's enough for now. On to the next one!
