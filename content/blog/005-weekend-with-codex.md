+++
title = "A Weekend with Codex after Claude Code"
date = 2026-07-13
description = "My experience with Codex: subagents, skills, the interface, and switching to OpenCode with an OpenAI provider."
+++

With the release of the GPT-5.6 line, I decided to spend the weekend
testing Codex on some pet projects. So I decided to use Codex for that.

Somehow, I had managed to avoid it and never worked with it before —
here was a chance to fix that.

In the end, though, I spent the whole time using Codex and cursing at it.
Below, I’ll go through the problems that became especially unpleasant for me.

> Everything here is just my personal opinion
> and experience from 2 days of use. Maybe that wasn’t
> long enough to properly figure things out.
> Maybe everything will work fine for you.

## Subagents

Yes, Codex supports subagents, but the implementation is
simply awful.

Claude Code, for example, has two ways of running subagents:
- synchronous: the main agent starts them and waits for them to finish;
- background: the main session remains unblocked, and the agent is notified
  when the subagents finish.

Codex has something in between. At first, that seemed interesting,
but the way it works is just 🤮.

Subagents run in the background and don’t block the main session,
but the main agent doesn’t get notified when a subagent finishes.
Instead, it has to ping them periodically, wasting time, context, and tokens.

On top of that, it’s very unstable: every so often, the subagents
just die, and you have to restart them.

I also often ask Claude Code Opus in the main session to launch
Sonnet subagents for particular tasks. That’s a good way to use
the subscription limits more efficiently and avoid running Opus
on tasks where it’s overkill. Unfortunately, Codex can’t do that.

## Skills system

I get the impression that Codex isn’t good at picking up skills automatically.
Every now and then, I noticed that instead of loading the skill it needed,
it would search for it with `rg` and read it like an ordinary file.

It also doesn’t always find skills installed through plugins on the first try,
since they live in a different directory.

Yes, it does eventually find the skills you point it to (I often explicitly
name a skill so that it gets used; maybe automatic skill use is the problem),
but the agent still spends resources just locating the skills it needs.
That gets especially unpleasant — and expensive — with GPT-5.6 Sol, for example.

## Also

A few minor gripes. I wouldn’t call them outright problems, but still:

1. In one session, I noticed that the agent’s message was duplicated — it
was simply displayed twice, which was confusing.
2. There are no slash commands, such as `/do something`. Sure, you can just
use commands as skills, but it isn’t as convenient.

## The solution

My overall experience with Codex over those two days was sharply negative.
The models themselves, however, impressed me.

So, to keep playing around with them, I decided to use OpenCode with an
OpenAI provider instead — luckily, that’s possible there, unlike with Claude.

> UPD (30.08.2026)
> After using OpenCode for several weeks, I found that it didn’t work
> the way I wanted (it’s hard to explain; I simply wasn’t satisfied
> with the result).
> I assumed it was just too heavyweight (too many tools and a large
> system prompt). In the end, I switched to the much lighter
> [pi coding agent](https://pi.dev/) and am happy with it now.
