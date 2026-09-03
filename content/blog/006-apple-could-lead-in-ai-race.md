+++
title = "Apple is falling behind in the AI race. Is it forever?"
date = 2026-09-03
description = "Why Apple's lead in ecosystem integration could still outweigh its lag in foundation models."
+++

We're used to hearing that Apple is losing the AI race. The company has no
frontier-level foundation model of its own that can compete with Gemini, GPT,
or Claude. And even before modern LLMs appeared, Siri was more often a punchline
than an example of a good assistant.
The partnership with Google was widely seen as Apple's final admission of defeat.

But I wouldn't write Apple off just yet. It has arrived late to markets before,
then assembled a more coherent product — that was the story with Apple Watch
and AirPods. Siri's history shows that this doesn't always work. Still, perhaps
we're measuring the AI race by the wrong metric.

## We're Measuring the Wrong Race

Leadership is usually reduced to the quality of the foundation model: who scored
higher on a benchmark, solved a harder problem, or handled a longer context
better. That matters to model developers. Users don't need the winner of a
leaderboard; they need a product that helps them every day.

The most powerful LLM doesn't automatically become the most useful assistant. It
still needs context, tools, access to apps, and a clear permissions system. The
model is central, but it's only one part of the product.

So I see two races: one for the best frontier model, and another for a personal
AI layer between people, their apps, and their devices. Apple is behind in the
first one. In the second, it starts from an unusually strong position.

## Gemini as Optimization, Not Surrender

In January 2026, Apple and Google [announced a multiyear
partnership](https://blog.google/company-news/inside-google/company-announcements/joint-statement-google-apple/): the next generation of Apple Foundation Models will be based on Gemini models and Google's cloud technologies. It's easy to read that as a white flag: Apple couldn't do it, so now it's buying intelligence from a competitor.

I find another explanation more convincing. Developing and running frontier
models requires enormous computing resources. If Google already does that part
better, Apple doesn't necessarily need to burn money owning every layer of the
system. The partnership could lower costs and risks, shorten development time,
and let Apple focus on the part that directly shapes the product.

That's only my interpretation; the financial terms of the deal haven't been
disclosed. At the same time, Apple hasn't abandoned its own development — it
calls the new family [built in collaboration with
Google](https://machinelearning.apple.com/research/introducing-third-generation-of-apple-foundation-models), not a simple call to somebody else's API. But if intelligence becomes an infrastructure component, integrating it well may matter more than developing everything in-house.

## A Small Model May Be Enough

Raw power matters when a model has to solve a new and difficult problem. But a
personal agent's actions are often more mundane: understand the intent, choose a
tool, pass along the parameters, and ask for confirmation when necessary.

A small language model (SLM) can run right on the device in this setting — fast,
offline, and without sending personal context elsewhere. More complex requests
can go to a more powerful model in the cloud. This isn't a choice between a
small and a large LLM; it's a hierarchy. Each one steps in where its particular
capabilities are needed.

Apple already has a technical foundation for this approach. Its local model,
with roughly 3 billion parameters, supports structured output and tool calling,
as the company described in an [update to Apple Foundation
Models](https://machinelearning.apple.com/research/apple-foundation-models-2025-updates). It isn't frontier-level, but an agent often benefits more from reliably triggering the right action than from writing a perfect essay.

`Qwen3.8-27B` is another telling example. According to its [model
card](https://huggingface.co/Qwen/Qwen3.8-27B), it comes close to much larger systems on some tasks. 27 billion parameters is too much for an ordinary phone, and the results were published by the developer itself. This isn't a ready-made mobile solution. It's a sign of where things are heading: smaller models are gradually handling increasingly difficult tasks.

## AI Shouldn't Live in a Separate Window

Today, AI features often feel bolted onto the main product. You have to open
ChatGPT, Claude, or Gemini, bring the context there, get an answer, and move it
back yourself.

Apple could integrate AI more deeply because it controls the path from chip to
interface: Neural Engine, the operating system, system apps, permissions,
frameworks, and the rules developers follow. It already has Spotlight,
Shortcuts, quick actions, and [App Intents](https://developer.apple.com/documentation/appintents), through which apps describe the actions and entities available to the system.

These pieces don't yet form a single agent, and an action triggered by holding
an icon can't magically become a safe AI tool. But the necessary vocabulary is
already emerging: apps tell the OS not only about their screens, but also about
what can be done inside them. Apple now needs to connect those capabilities with
a shared model, context, and access rules.

This is where vertical integration becomes more important than another
percentage point on a benchmark. An assistant developer that doesn't control
the platform has to negotiate with the owners of the OS, apps, and devices.
Apple often gets to negotiate with itself.

## An Agent That Follows the Person

Picture a morning. I head out for a run with my Apple Watch and earbuds, ask
about the day's plans, and ask the agent to prepare materials for my first
meeting. I get home and use HomePod to clarify one of the items. When I sit down
at my Mac, the agent continues the task, opens the documents I need, and shows a
short summary before the meeting starts.

For now, that's a fantasy, not a working feature. Individual pieces exist, but
Apple doesn't have a reliable agent that carries an unfinished task across all
of its devices. Few companies, though, have the phone, watch, earbuds, speaker,
tablet, and computer integrated so tightly.

Continuity isn't an endless chat. The interface changes with the situation while
the intent stays the same: on a run, the agent answers by voice; at home, it
continues the conversation; at the desk, it shows the documents. AI becomes a
property of the whole system.

## Without Trust, None of This Works

An assistant like this would need access to your calendar, email, location,
documents, and authenticated sessions. The more useful the agent, the more data
and authority you have to entrust to it — and the more dangerous its mistakes
become. Users need to understand what the agent can see, what it can do, and
where it will stop without confirmation.

Apple is already trying to address the cloud side of this problem through
[Private Cloud Compute](https://security.apple.com/blog/private-cloud-compute/). The company says requests are processed on attested nodes, data isn't stored, and the software images are available to researchers for inspection. That's more concrete than the usual promise to “value your privacy.”

Mark Zuckerberg recently described a similar future. In his manifesto [*The
Future is for Everyone*](https://www.meta.com/thefutureisforeveryone/), he talks about a personal agent that understands a user's goals and works in their interest “24/7.” On the same day, Meta released the open weights for [Muse Glimmer-30B](https://research.meta.ai/blog/introducing-muse-glimmer-open-agentic-model) under Apache 2.0. Meta calls the model optimized for “always-on local agent workflows.” It's a local model for a powerful Mac or PC, not a ready-made mass-market assistant, but the direction is quite specific.

The idea itself sounds reasonable, even to the author of a critical column. Russell
Brandom [wrote in
TechCrunch](https://techcrunch.com/2026/08/10/mark-zuckerbergs-ai-manifesto-is-exactly-why-people-dont-like-ai/) that he agrees with Zuckerberg “in broad strokes.” His problem is the messenger: society doesn't trust technology-company executives to decide the consequences of new technologies. That skepticism toward Meta wasn't unfounded. In 2019, Facebook [agreed to pay a record
$5 billion fine](https://www.ftc.gov/news-events/news/press-releases/2019/07/ftc-imposes-5-billion-penalty-sweeping-new-privacy-restrictions-facebook) to settle FTC allegations that it had violated a 2012 privacy order.

Apple may have an advantage here: it backs up its privacy claims with the
concrete architecture of PCC and a reputation built over many years. But PCC
doesn't make an agent safe automatically. It won't protect against a
misunderstood command, excessive permissions, prompt injection in an email, or a
bug in an app. And trust in a brand can't be earned once and then spent forever.

## Why Apple Could Still Lose

The strongest argument against everything above is Apple itself. The company
spent years failing to make Siri genuinely useful. In March 2025, Apple
[acknowledged a delay](https://daringfireball.net/2025/03/apple_is_delaying_the_more_personalized_siri_apple_intelligence_features) to the personal-context and in-app-action features it had promised a year earlier. In June 2026, it [introduced the new Siri
AI](https://www.apple.com/newsroom/2026/06/apple-introduces-siri-ai-a-profoundly-more-capable-and-personal-assistant/), but it was still a beta at the time of publication. Control over the stack is useless if the team can't connect its parts into a working product.

There's a strategic risk, too. Google is now not only a partner but also a
supplier of critical technology, while it develops Gemini, Android, and its own
device ecosystem. Google and Samsung already have [Gemini with actions across
multiple apps](https://blog.google/products-and-platforms/platforms/android/google-ai-samsung-galaxy-s25/), so Apple's head start is far from unique.

A real agent also needs third-party developers to participate. They'll have to
describe their actions, work with permissions, and provide ways to undo
mistakes. Safety adds confirmations and limits; autonomy raises the cost of an
error. Finding the balance is harder than putting on a pretty demo at WWDC.

## In Conclusion

The winner of the consumer AI race may not be decided on a benchmark leaderboard.
It will be whoever makes an agent useful across devices and apps, while also
making it safe enough for real-world actions. Apple has an operating system,
system tools, on-device models, a private cloud, and users who have lived inside
its ecosystem for years.

The hardest part remains: bringing it all together. If Apple succeeds, its
current lag in foundation models will turn out not to be a death sentence, but just
one detail. If it doesn't, vertical integration will become another attractive
advantage Apple failed to use.
