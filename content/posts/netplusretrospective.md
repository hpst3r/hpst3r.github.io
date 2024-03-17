---
title: "CompTIA Net+ Retrospective"
date: 2023-12-30T19:14:24-00:00
draft: false
---

Updated post-CCNA, 3/2024.
Lines preceded by a '+' are new additions.

## Introduction

I took and passed my N+ with around an 800 on 12/21/2023. I spent about thirty hours explicitly studying for the Net+ and taking practice tests, over the course of about a week. Here's what I thought of it, and what I learned.

### Study Material

I used Jason Dion's practice tests and made flashcards. I was able to pass the Net+ with basic knowledge that I already had. No official study material and I didn't follow a prep book.

## The Test

The Net+ isn't the most difficult test in the world, and success with it does not depend on having a comprehensive understanding of networking (routing isn't covered well.) That isn't to say that it's the easiest test I've ever taken, or that it's meaningless.

### General Knowledge

For a general idea of the type of understanding I think a Net+ implies:

- You need to know the difference between the various 802.11 standards off the top of your head
    - will an 802.11g device connect to an 802.11n access point?
    - What about an 802.11a device?
    - What's the theoretical maximum bandwidth of WiFi 6?
- What 802.* standard is PoE? PoE+?
- What's a VLAN?
- Some "semi-practical" questions:
    - what's a port configured as a VLAN trunk's purpose (and limitations)?
    - What's a RADIUS server, and which wireless security protocol makes use of it?
    - What's the most secure way to configure a wireless network that doesn't make use of a centralized AA server?
    - What are common types of DNS records?
    - What's the standard size of an Ethernet frame?
        - What's a runt, what's a giant?
    - What type of network would you make use of jumbo frames in?

In retrospect, I think this baseline, definition-based knowledge is all important, and I cannot take issue with the Net+ covering it.

You cannot efficiently troubleshoot a connectivity issue if you don't know DHCP, DNS or APIPA exist and I realize that the Net+ is targeted squarely at entry-level professionals attempting to secure a badge showing that they have at least an entry-level understanding of common networking protocols and at least some kind of idea of how a network goes together.

+ Unfortunately, the Net+ doesn't really care about your actual ability to troubleshoot, as far as I'm concerned. I felt like the exam was more "trivia" and less implementation or understanding.
+ In contrast, the CCNA required the ability to fix broken things and configure basic network connectivity in a handful of scenarios. Additionally, the 'trivia' was much more difficult - I'd say it required a thorough understanding of the underlying topic.

### Asterisks (PBQs)

The PBQs were a big question mark when I came in to sit for my exam. I used Jason Dion's practice exams (Udemy, $15 purchase) to prep for the test. They did not have PBQs, and the 'simulated' PBQ style questions provided were not very accurate (I expected this going in, but this still meant I'd never seen one before.)

Being as general as possible to abide by the candidate agreement, you do *not* need to be familiar with any command line interface to give the Net+ a shot. Just remember that '`man man`' is the solution to all your problems. You'll do just fine if you have the knowledge required to pass the rest of the exam.

+ The Net+ is not at all comparable to the CCNA in this regard. The CCNA has simlet questions that are actually fairly similar to using real devices. The questions given on the CCNA require a much better grasp of fundamental networking concepts.

## Conclusion

### What I Learned

I feel like I came away from this exam with a slightly improved personal glossary. I could not list off what 802.11a meant beforehand, and I can now. However, preparing for this cert did not expose me to much "networking".

#### What I Disliked

I do not feel that the Net+ tests more than an elementary understanding of networking and adjacent technologies. I am notably disappointed in the absence of L3, though I suppose an in-depth understanding of routing isn't strictly necessary for an entry-level IT professional.

+ CCNA remedies this a bit.

#### The Real Conclusion

Overall, I'm rather disappointed that I spent the time and money on the Net+. Questions weren't all well thought out and the knowledgebase it covers was rather limited. Instead of putting my resources into the Net+, I feel like I should have used the time and money to get at least partway into a CCNA (though I would certainly not be able to finish it in 30 hours) as it would have probably been both a better resume item *and* a more effective learning experience.. although, since I feel somewhat like a saltwater fish in a freshwater pond when dropped into a Cisco CLI, perhaps labbing through the pain will be a summer project.

#### Conclusion Pt. 2 - Post CCNA

I should have used the time and money to start working on the CCNA instead of giving CompTIA any money. The Net+ is more or less meaningless in comparison to the Cisco cert, but hindsight is 20/20. Also, the IOS CLI isn't that bad, at all. Took a few days to get used to it. I would not recommend anyone spend the money on the Net+.