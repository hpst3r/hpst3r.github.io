---
title: "CompTIA Net+ Retrospective"
date: 2023-12-30T19:14:24-05:00
draft: false
---

## Introduction

I took and passed my N+ with around an 800 on 12/21/2023. I spent about thirty hours explicitly studying for the Net+ and taking practice tests, over the course of about a week. Here's what I thought of it, and what I learned.

## The Test

The Net+ isn't the most difficult test in the world, and success with it does not depend on having a comprehensive understanding of networking. That isn't to say that it's the easiest test I've ever taken, or that it's meaningless. It required me to do a good deal of memorization.

### Knowledge Required (the content)

You need to know the difference between the various 802.11 standards off the top of your head - will an 802.11g device connect to an 802.11n access point? What about an 802.11a device? What's the maximum bandwidth you can get out of a WiFi 6 access point? What 802.* standard is PoE? What's a VLAN, and what's a port configured as a VLAN trunk's purpose (and limitations)? What's a RADIUS server, and which wireless security protocol makes use of it? What's the most secure way to configure a wireless network that doesn't make use of a centralized AA server? What are common types of DNS records? What's the standard size of an Ethernet frame? What type of network would you make use of jumbo frames in?

Questions that could be solved with that knowledge alone probably made up 10-15% of my exam. Many questions could have been solved by picking the three wrong answers apart and selecting the only one that makes sense, but I would have found it difficult to do this without understanding either the three not-answers or the one real answer, so it makes little difference to outcome.

In retrospect, I think this baseline, definition-based knowledge is all important, and I cannot take issue with the Net+ covering it. You cannot troubleshoot a connectivity issue if you don't know DHCP, DNS or APIPA exist (or, at the very least, it would make your life much more difficult) and I realize that the Net+ is targeted squarely at entry-level professionals attempting to secure a badge showing that they have at least an entry-level understanding of common networking protocols and at least some kind of idea of how a network goes together.

### Asterisks (PBQs)

The PBQs were a big question mark when I came in to sit for my exam. I used Jason Dion's practice exams (Udemy, $15 purchase) to prep for the test. They did not have PBQs, and the 'simulated' PBQ style questions provided were not at all reminiscent of the actual test (I expected this going in, but this still meant I'd never seen one before.)

You do *not* need to know any command line interface to give the Net+ a shot. The PBQs feature an extremely simplistic command line that will tell you exactly what you need to do (type "help") to get the information you need to complete the question. If you have even a slight bit of experience with IOS command lines (you know, made by that company whose name is similar to Sysco) and if the PBQs are similar to the four or five I worked through, you'll do just fine. Each question is fairly limited in scope but might catch you off guard with an odd complication.

## Conclusion

### What I Learned

I feel like I came away from this exam with a slightly improved personal glossary. I could not list off what 802.11a meant beforehand, and I can now. However, preparing for this cert did not expose me to much novel information.

#### What I Disliked

I do not feel that the Net+ tests more than an elementary understanding of networking and adjacent technologies. I am notably disappointed in the complete absence of material involving L3 troubleshooting, though I suppose an in-depth understanding of routing isn't strictly necessary for an entry-level IT professional.

#### The Real Conclusion

Overall, I'm rather disappointed that I spent the time and money on the Net+. I am also disappointed that I spent hundreds of dollars on a test that had a number of typos in questions. Instead of putting my resources into the Net+, I feel like I should have used the time and money to get at least partway into a CCNA, which I likely could have finished fairly quickly (not within 30 hours, maybe 90 or 120?) and would have probably been both a better resume item *and* a more effective learning experience.. although, since I act like a saltwater fish in a freshwater pond when dropped into a Cisco CLI (/etc/network/interfaces is my preferred level of complexity), so perhaps labbing through the pain will be a summer project.