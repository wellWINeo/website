+++
title = "Most Software Will Be Completely Rewritten Over the Next Three Years"
date = 2026-11-05
description = "Why AI agents may make rewriting old systems cheaper than endlessly repairing them—and what will be left of developers’ work."
+++

The title is deliberately sensational and provocative, but I don't think it's far from the truth.

## Act 1. The vulnerability wave

Over the past six months or so, we've been hit by a massive wave of critical vulnerabilities that demanded an immediate response—patching where a fix was available, or simply disabling part of the functionality.

I see the rise of AI agents as the main reason. But it's not just that they introduce new vulnerabilities into code. They also make it faster to find bugs that have been there for years, if not decades, and to chain them into exploits with dangerous consequences.

For a sense of scale, I looked at how many CVE records are published each year, using data from the [CVE Program](https://www.cve.org/About/Metrics#PublishedCVERecords). These numbers include vulnerabilities of every severity, not just critical ones.

![CVE records published by year since 1999: 48,244 for all of 2025 and 72,812 for January–September 2026](/assets/whole-software-we-know-would-be-rewritten/cve-publications-by-year.svg)

*1999–2025 are full years; 2026 covers January–September only, with no projection for the final quarter.*

There were 28,961 CVE records published in 2023, 40,077 in 2024, and 48,244 in 2025. In the first nine months of 2026 alone, there were already 72,812. That's 105% more than in the same nine months of 2025, when 35,448 records were published.

![CVE records published by month from January through October 2026](/assets/whole-software-we-know-would-be-rewritten/cve-publications-by-month-2026.svg)

But these are publication counts, not the number of bugs introduced in a particular year or the number of attacks. This growth alone doesn't prove AI is having an impact: the program's scope and the process for registering vulnerabilities are changing too. For example, [the CVE Program itself attributes the increase in publications to the expansion of its CNA network](https://www.cve.org/Resources/Media/Cve25YearsAnniversaryReport.pdf)—the organizations that assign identifiers and publish CVE records.

So far, all of this came to a head in “Black October 2026,” which saw:

* two critical PHP vulnerabilities that allow RCE—[php2shell, CVE-2026-109618 and CVE-2026-109619](https://www.php.net/archive/2026.php#2026-10-21-5)
* a critical vulnerability in the Linux TCP stack, with RCE demonstrated on systems running MPTCP
* RCE in GitLab
* the compromise of eMudhra intermediate certificates (probably the biggest WebPKI incident in more than 30 years)
* the compromise of Railway
* and much more

And all of this happened not just in October, but over the course of about a week.

The pace of attacks is telling, too. On October 22, Cloudflare CEO Matthew Prince [wrote](https://x.com/eastdakota/status/2113079352682570051) that the company's network had recorded thousands of exploitation attempts as early as ten hours before php2shell was publicly announced. In the first 45 minutes after the announcement, the count climbed into the tens of thousands; after two hours, the hundreds of thousands. By the time he posted, it was in the millions.

Those were attempts, not millions of successful breaches. Still, we can no longer count on having a few quiet days between the announcement of a critical vulnerability and the start of mass attacks.

## Act 2. The problem with accumulated codebases

Most of the software we use has been developed over decades. I've long had this thought: any codebase tends to turn to shit as it grows. And you don't need to hire bad developers for that to happen.

Everyone may have made perfectly reasonable choices. Someone was saving memory; another team was up against a release deadline. Then backward compatibility became necessary, the protocol changed, or someone bought a company along with its product. Nobody removed the old solution—they just bolted the new one on top.

Twenty years later, we're left with dozens of layers of other people's compromises. No one can explain half of them anymore, but everybody knows not to touch them. We call it a mature, time-tested system. It might look something like this:

![A residential building with numerous additions](/assets/whole-software-we-know-would-be-rewritten/home.jpeg)

All of this already looked like a game of Jenga: the components seem to work, but pulling any one of them out feels dangerous. Now people are going to test this structure much more aggressively.

And that's no exaggeration. In [Azure API Connections](https://www.binarysecurity.no/posts/2026/10/one-root-case), a researcher bypassed the fixes again and again: the service had excessive permissions, and attempts to close the issue relied on URL checks. After one of the patches, an earlier bypass worked again.

Linux is a good example of a large codebase. The kernel has been developed for more than thirty years, with thousands of people contributing to it. It's tens of millions of lines of code supporting a huge number of systems, including critical ones. Its monolithic architecture was criticized from the very start—just think of Tanenbaum. A compromised kernel can hand an attacker the entire system, so mistakes at this level are especially costly. From here on, I'll be talking mostly about Linux.

## Act 3. Code costs nothing now

But for all the criticism of Linux, replacing it is on a completely different scale from simply writing your own small kernel. Supporting the hardware, protocols, and applications you need—and then making all of that the new standard—is another story altogether.

In 2008, the cost of redeveloping the Linux kernel from scratch was [estimated at $1.4 billion](https://www.linuxfoundation.org/press/press-release/linux-foundation-publishes-study-estimating-the-value-of-linux), and by 2011 it was [around $3 billion](https://dwheeler.com/essays/linux-kernel-cost.html). The codebase has grown several times over since then. Obviously, these estimates are rough, but they give you a sense of the scale of the cost.

Those estimates assumed people were writing the code. With agents, it costs less and gets done faster.

“But agents can fix old code, can't they?” Of course they can. And they will. But sometimes what needs fixing is the system's design itself. If a service is given access to everything, and its security depends on developers remembering to check every possible way of reaching it, we can keep collecting CVEs for a long time.

We're used to thinking that rewriting a large system from scratch is almost always a bad idea. It takes too long, costs too much, and too much will break. So we keep propping up the old structure. That was a perfectly reasonable choice when writing a new implementation by hand took years.

But it's time to run those numbers again. Maybe another ten years of repairs will cost more than a replacement. I think that's where we're headed.

In the end, I think the only complete solution to the endless stream of vulnerabilities is to rewrite most software from scratch—including critical systems.

An agent being the author doesn't fix anything by itself. Ask it to reproduce the old architecture, and the old problems can easily come along for the ride. The code will be fresh, though.

The rewrite has to account for what we've learned: where inputs can't be trusted, how to divide privileges, which classes of errors shouldn't be possible in the first place. Whether that means a monolithic kernel or a microkernel is another conversation. Simply giving the old mistakes a new author isn't enough.

All that code we've spent decades writing by hand really could be rewritten by agents. We can put it in a frame, shed a tear over the beautiful abstraction, and move on.

But what will be left of our work? The system's behavior. The decisions about what it should do, what it must remain compatible with, and what it must never allow under any circumstances. Specifications, ideally.

Only a lot of those specifications currently live in the source code. In some odd check that was added after an incident. In a call order you can't change, or everything crashes once a month. No one remembers why anymore, but they tested it—and it crashes.

And we'll have to tease that knowledge out. Figure out which of the workarounds accumulated over the years are still necessary, and which ones we're simply afraid to touch. Otherwise, the agent will dutifully rewrite them all.

Have we really spent all these years just typing colorful little characters into an editor? Or were we making systems useful to people and saving businesses money? If it's the latter, replacing the source code doesn't mean our work has gone in the trash.
