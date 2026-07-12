+++
title = "How I Wrote My Thesis with an Agent"
date = 2026-07-12
description = "How I built a workflow for writing my thesis: Claude Code on an Ubuntu VPS, source processing, batch review, and reproducible DOCX builds."
+++

# How I Wrote My Thesis with an Agent

This year I wrote my graduation thesis about `buildout` — a CLI and MCP server
for working with documentation in buildin.ai. I built the project itself with
Claude Code and Spec Kit. To write the explanatory report, I created a separate
repository called [thesis-writing-agent](https://github.com/wellWINeo/thesis-writing-agent).

This is not a story about an agent writing my thesis for me. It did not choose
the topic, stack, or architecture, formulate the requirements, or check whether
the implemented result actually matched the original idea. Its job was
different: turn completed engineering work, source documents, and university
requirements into coherent, verifiable, properly formatted text.

The result was a DOCX that I printed and then successfully defended. But getting
there was more of an engineering experiment than a single successful prompt.

## Where It Started

By the time I created the agent, I already had:

- a draft of the first chapter;
- reports on my academic and operational practical training;
- the university's guidelines;
- the source code of the project that was the subject of my thesis.

I had prepared the internship reports with Claude Cowork, while the project
itself was developed in Claude Code using Spec Kit. So the new agent was not
starting from a blank page, but neither was it starting with a carefully
prepared knowledge base. The source material came in different formats and was
of varying quality: DOCX, XLS, already-written text, methodological documents,
and source code.

The first step, then, was not writing chapters. It was packaging the context.

## Where the Agent Ran

Claude Code ran on an Ubuntu VPS rather than on my laptop. That mattered for two
reasons. First, the agent could keep working after I closed the laptop and
through the night while I slept. Second, long-running tasks made better use of
my Claude subscription: I did not have to keep my local machine on and wait for
each pass to finish.

I connected to the VPS over ordinary SSH. For quicker checks, I used Claude
Code Remote, which let me interact with the agent from my phone when I did not
have my laptop nearby. The setup was practical: the agent handled heavy tasks
in the background, and I connected when I needed to inspect the result, leave
comments, or start the next step.

Still, the agent being constantly available did not mean it could change the
final text without limits. Those limits came from the repository structure and
the manual approval rule, not from whether my laptop happened to be open.

## Why Claude Code

At first I considered several options. I could use OpenClaw or Hermes, or build
my own agent with LangChain. LangChain would have given me complete control over
the architecture, but I would first have had to build the agent system itself:
the execution loop, tool integration, memory, error handling, and interface. I
did not have time for that. I needed to write a thesis, not develop a platform
for writing one.

OpenClaw and Hermes were not a good fit either. They are designed as continuously
running systems, built around constant presence and interaction through Telegram,
Slack, and similar channels. My task was different.

The thesis workflow was mostly step-by-step, or turn-based: the agent prepared
the next section, recorded the result, saved context in `memory/`,
`CHANGELOG.md`, and a decision log, then moved on to the next step. It did not
need to remain in a permanent conversation or decide for itself what to do an
hour later.

What I needed from the agent was more practical:

- reliable work with files in the repository;
- the ability to load and use skills;
- the ability to read project instructions and follow the required structure;
- command execution and source-code access;
- context transfer between sessions through memory files.

Claude Code handled this well enough. It already provided the runtime,
tooling, and a way to work with a repository, so I only had to design the
domain-specific parts: thesis skills, a memory format, review rules, and a
build pipeline.

There is a broader lesson here. Before developing your own agent or an entire
platform, it is worth testing the idea with a ready-made tool like Claude Code.
That proof of concept helps determine whether you really need a continuously
autonomous system, a separate interface, and your own runtime, or whether the
task can be broken down into a sequence of steps inside a repository.

## The Repository as the Agent's Operating System

The `thesis-writing-agent` repository is not a collection of generic
instructions like “write a good chapter.” It defines the project structure, the
roles of individual skills, the rules for working with memory, and the boundary
between draft and approved text.

In simplified form, the working directory looks like this:

```text
CLAUDE.md
REQUIREMENTS.md
outline/OUTLINE.md
references/
memory/
style/STYLE_GUIDE.md
drafts/
chapters/
converter/
addons/software-thesis/
```

`CLAUDE.md` is the operating manual. It describes what the agent should do when
a session starts, which skills to use, and the order in which to carry out the
work.

Skills divide the process into roles:

- `thesis-planner` maintains the structure of the thesis;
- `thesis-researcher` finds and records sources;
- `thesis-writer` writes text to `drafts/`;
- `thesis-reviewer` looks for problems and records the review results;
- `thesis-editor` brings the text to its final form;
- `thesis-code-analyst` analyzes the project's source code for technical chapters.

There are also separate skills for renumbering references and humanizing text.

This separation is not there to make the architecture look nice. It limits the
responsibility of each step. For example, the writer should not move text into
the final directory on its own, and the reviewer should not silently edit the
original draft instead of leaving comments.

## Turning Documents into a Knowledge Base

The agent processed the source documents on its own. I did not manually convert
the DOCX and XLS files to text before getting started.

The processed material went into `references/`. The original files remained
there, but they were joined by machine-prepared summaries:

```text
references/
  summary-guidelines.md
  summary-practice-main.md
  summary-practice-exploitation.md
  summary-life-safety-guidelines.md
  summary-economy-guidelines.md
  BIBLIOGRAPHY.md
```

This is an important difference from an ordinary chat with a model. In a chat,
context gradually disappears or starts consuming the entire context window. In
a repository, it becomes an artifact: something you can read, check, correct,
and use in the next session.

`BIBLIOGRAPHY.md` became a single registry of sources, with annotations and
references to the sections where they were used. `memory/` stored the current
status, decisions, review notes, and contradictions found in the methodological
materials.

For example, while preparing the safety section, the agent found that different
parts of the same set of materials reached incompatible conclusions about how
often NO₂ concentration should be monitored. That contradiction should not
disappear inside a generated paragraph. It was moved into memory, so the
decision could be made explicitly and preserved as a separate decision for the
text.

## The Chapter Preparation Cycle

The main workflow looked like this:

```text
Explore → Plan → Write → Review → Fix → Edit → Human approval
```

First, the agent studied the sources, requirements, outline, and, when needed,
the project's code. It then wrote the chapter in `drafts/`. The
`thesis-reviewer` skill checked the content, references, fidelity to the sources,
and style. The agent fixed the issues and repeated the review until the text
was ready for editing.

`chapters/` was the area for approved content. Material moved from `drafts/`
there only after my explicit decision. This simple rule turned out to matter
more than many detailed instructions: a model can rewrite text quickly, but it
should not decide for itself what becomes part of the thesis.

## Why the First Pull Request Workflow Failed

At first I wanted to build the process around a workflow familiar to developers:

1. the agent prepares a chapter in the background;
2. it opens a pull request;
3. I leave review comments;
4. the agent makes changes and updates the PR;
5. I merge the result.

From an engineering perspective, this seemed natural. In practice, though, the
model worked faster than I could read. There was a lot of text, changes appeared
often, and reviewing individual diffs made it hard to keep several chapters
coherent in my head.

The bottleneck turned out to be neither the agent nor Git. It was my reading
capacity.

I replaced the PR workflow with two zones:

- the agent writes and freely reworks text in `drafts/`;
- `chapters/` contains only material that has passed my batch review.

Usually I reviewed two chapters per evening in Zed with Markdown preview, then
gave the agent a list of comments. This was slower at the level of individual
changes, but it fit my actual reading pace much better.

That is probably the main practical lesson: a human-in-the-loop process should
be designed around the person's cognitive load. Having an approval button does
not mean that approving the result is convenient.

## What the Review Covered

After several passes, the text was usually coherent, but that did not mean it
was ready to submit.

### Sources

Some sources looked formally suitable but did not meet the standard expected of
academic work. Other links were simply broken.

A particularly telling case is when a website serves a “404 Not Found” page but
still returns the HTTP status `200 OK`. If you check only the status code, the
link looks valid and can easily end up in the bibliography.

In cases like that, I sent the problem back to the agent for another search. But
the final judgment about whether a source was suitable remained my responsibility.
The model can find a publication that supports a sentence, but that does not
mean the publication is reliable enough for the particular thesis.

### Text That Was Missing or Unnecessary

Sometimes the agent added details that did not strengthen the argument. It was
ordinary “fluff,” just with good grammar.

Sometimes the opposite happened: the text was too brief, and it needed an
explanation, an example, a reference, or an explicit conclusion tied to the task.

So the review was not just an error check. I decided which claims should be
compressed, which needed support, and which deserved a fuller explanation.

### Connection to the Real Project

For the technical chapters, the agent used the project's source code. Here it
was especially important to prevent a description from sounding convincing
while disagreeing with what had actually been implemented.

That is why the system has a separate `thesis-code-analyst`: it analyzes the
code, records file paths, and ties fragments to a repository version. That is
more reliable than asking a language model to “remember” how the project works.

## DOCX Was a Separate Product

Markdown was convenient for the agent and for Git, but the university needed a
Word file. Converting it to DOCX quickly became a separate engineering project.

The build used Pandoc with Lua filters and a `reference.docx` template defining
the styles. More expensive Opus 4.7/4.8 models helped review and plan this
system, while Sonnet 4.6 handled most of the text work.

The most painful problems were:

- adding a properly formatted title page;
- Word styles for body text, captions, tables, and code;
- getting tables to render with proper borders;
- Mermaid diagrams that sometimes did not fit on the page;
- Markdown syntax that Pandoc handled differently than expected;
- inconsistent heading levels;
- numbered lists using `1)` instead of `1.`;
- unordered lists using `* ` instead of the expected `- ` format.

The agent could produce text that was valid Markdown but failed to build into a
document with the required structure or looked bad after rendering. That is why
the pipeline gained automatic checks and a separate visual inspection of the
final document.

This suggests another useful principle: generating text and building the final
artifact are different tasks. The first can be delegated to a language model;
the second is better made deterministic and testable.

## What Became Automated

The agent handled the routine parts well:

- extracting and structuring the contents of source documents;
- building a working bibliography;
- maintaining the project outline and memory;
- preparing descriptive sections about work that had already been completed;
- finding internal inconsistencies and places that needed clarification;
- running repeated editorial passes;
- helping maintain a consistent style and format;
- participating in the DOCX build and the defense presentation.

I still chose the project's topic, stack, and architecture, wrote the
requirements, worked through the architectural decisions, and checked the
result. Those parts cannot honestly be described as the agent's work.

The defense presentation was also prepared with Claude Code and Gamma, but that
did not change the division of responsibility: the tools helped format and
package the result; they did not make engineering decisions for me.

## Conclusion

The resulting system did not give me an “automatic thesis.” It gave me a
practical way to turn engineering experience into a formal document. The most
useful part was not any individual prompt, but the structure around the prompts:

- source material becomes a verifiable knowledge base;
- the agent's roles are split into separate skills;
- drafts are kept separate from approved text;
- a human controls the transition between them;
- final DOCX generation is handled by a reproducible pipeline;
- errors and decisions are stored in the repository instead of disappearing into
  chat history.

If I were building this system again, I would start by assessing the human
review workflow rather than opening a PR for every chapter. For long documents,
generation speed is not the main metric. What matters more is whether a person
can read, understand, and verify what the agent produced in time.

And, of course, an agent setup does not remove authorship. It is good at writing
about work that has already been done, but it does not free you from choosing
what to do, explaining why the decision makes sense, and proving that the result
works.
