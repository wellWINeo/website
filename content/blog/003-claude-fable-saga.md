+++
title = "Claude Fable Saga, or How the World Is Splitting in Two"
date = 2026-06-15
description = "Breaking down the Claude Fable 5 and Mythos 5 shutdown — and trying to make sense of whether this was geopolitics or a hit on Anthropic."
+++

# Background

> Feel free to skip if you're already up to speed

- **April 7** — Anthropic [announced](https://www.anthropic.com/project/glasswing) Claude Mythos and the Project Glasswing program: model access restricted to ~50 selected partners (Amazon, Apple, Microsoft, and cybersecurity companies), because Mythos can autonomously find vulnerabilities and write exploits.
- **April 7, same day** — a Discord group [gained access](https://www.techradar.com/pro/security/mythos-accessed-by-unauthorized-users-as-anthropic-says-were-investigating-cracks-may-be-showing-in-project-glasswing-as-unknown-users-access-model-via-third-parties) to Mythos Preview by guessing the URL from Anthropic's naming conventions.
- **April 21** — [a second incident](https://thenextweb.com/news/anthropic-mythos-unauthorized-access-vendor-breach): a different group got in through a third-party vendor environment.
- **June 9** — [Claude Fable 5 launched](https://www.anthropic.com/news/claude-fable-5-mythos-5): a public, guardrailed version of Mythos. Dangerous requests are automatically redirected to Opus 4.8.
- **June 10** — [it emerged](https://fortune.com/2026/06/10/anthropic-accu-claude-fable-5-limits-capabilities-ai-researchers-developers/) that Fable 5 had also been quietly sabotaging competing model development — no warnings, just degraded output. Anthropic [acknowledged the mistake](https://www.engadget.com/2192004/anthropic-walks-back-policy-sabotaging-research/) and promised to make all restrictions explicit.
- **June 12** — Amazon researchers found a jailbreak, CEO Andy Jassy [reported it to the White House](https://techcrunch.com/2026/06/13/amazon-ceo-reportedly-raised-anthropic-model-concerns-before-government-crackdown/). Commerce Secretary Lutnick issued an [export control directive](https://www.tomshardware.com/tech-industry/artificial-intelligence/us-export-control-order-forces-anthropic-to-disable-claude-fable-5-and-mythos-5-worldwide): cut access for all foreign nationals, Anthropic employees included. No compliant solution existed, so [both models were taken offline](https://9to5mac.com/2026/06/12/anthropic-pulls-claude-mythos-5-and-claude-fable-5-following-us-government-directive/) globally. Anthropic [disagreed](https://www.tomshardware.com/tech-industry/artificial-intelligence/trump-adviser-david-sacks-says-anthropic-refused-to-fix-fable-5-jailbreak-before-us-export-controls) but complied.

# What's Actually Going On Here?

## Splitting the World: Restricting Access to Frontier Models

Back when ChatGPT 3.5 first dropped, it genuinely surprised me that anyone in the world could just use it. A technology that might define the whole decade. Maybe the century.

At the time, LLMs were basically glorified chatbots — sure, they could generate text, summarize things, spit out code snippets, answer quick questions. Useful enough, but not exactly earth-shattering.

The past year or two changed that. LLMs stopped being chatbots and became the reasoning core of agents — things that read situations, pull in context, organize data, make decisions, and act on them using tools.

Which makes it more than a little interesting that the US military used [Claude Opus 4.6 for planning military operations](https://www.scientificamerican.com/article/anthropics-safety-first-ai-collides-with-the-pentagon-as-claude-expands-into/). That's not a chatbot anymore. That's dual-use technology.

And yet — roughly equal access for everyone. A top military commander, a big-tech engineer, or some person in a remote corner of the world who scraped together $20 for Claude Pro (setting aside rate limits and token quotas). That's wild, when you actually think about it.

For a long time, this genuinely amazed me — and alongside that amazement was the quiet understanding that it couldn't last. AI is becoming the defining weapon of the 21st century, the engine of a new cold war. At some point, access to frontier models was going to be gated behind alliances.

A [Semafor piece](https://www.semafor.com/article/06/13/2026/white-house-move-to-limit-anthropic-linked-to-concerns-about-chinese-access-to-mythos) adds some weight to this reading: the real trigger for the directive wasn't the Fable 5 vulnerability itself, but fear that Chinese labs might reach Mythos and distill it. Not an unreasonable fear — back in February, [Anthropic accused DeepSeek, Moonshot, and MiniMax](https://techcrunch.com/2026/02/23/anthropic-accuses-chinese-ai-labs-of-mining-claude-as-us-debates-ai-chip-exports/) of harvesting data through 24,000 fake accounts. DeepSeek already proved you can get surprisingly close to frontier quality on a fraction of the resources, if you're willing to get creative about data. And Mythos, remember, can write exploits.


## Deliberate Obstacles for Anthropic

There's another angle. Anthropic and the US government have had a deeply strained relationship for months. On February 27, 2026, the Pentagon [added Anthropic to its national security threat list](https://www.cnbc.com/2026/04/08/anthropic-pentagon-court-ruling-supply-chain-risk.html) — a designation historically reserved for foreign companies like Huawei. Anthropic sued; the appeals court denied an injunction; the case grinds on.

Both major players — [OpenAI and Anthropic — are preparing to go public](https://finance.yahoo.com/markets/article/spacex-openai-and-anthropic-here-are-the-most-anticipated-ipos-in-2026-114439441.html) before year's end, both filing confidential S-1s in June. The Fable 5 launch was a serious positive driver for Anthropic's valuation.

And then, just 48 hours later, the same government that's suing Anthropic issues a directive gutting global access to their flagship model.

The trigger came from Amazon — Anthropic's largest investor and cloud provider. Andy Jassy personally called the White House to report the jailbreak. A company that put $4 billion into Anthropic effectively initiated the shutdown of its flagship product. Coincidence?

And then there's the Pentagon's own contradiction. It put Anthropic on the threat list, has been suing them for months — and [then used Claude to coordinate strikes on Iran anyway](https://www.washingtonpost.com/technology/2026/03/04/anthropic-ai-iran-campaign/). The model is apparently good enough to trust with real military operations. Just not safe enough for some developer in Berlin to use.
