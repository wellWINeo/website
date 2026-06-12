# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal website built with [Zola](https://www.getzola.org/), a Rust-based static site generator. The dev environment is managed via a Nix flake; `direnv` activates it automatically (`use flake` in `.envrc`).

## Commands

```bash
zola serve          # local dev server at http://127.0.0.1:1111 with live reload
zola build          # build to public/
zola check          # validate internal links
```

## Architecture

```
zola.toml               # site config — base_url, feeds, theme, [extra] social links
content/
  _index.md             # landing page (empty body; layout driven by index.html)
  blog/
    _index.md           # blog section — title, description, sort_by
    *.md                # posts (TOML front matter: title, date, description, tags)
themes/custom/
  templates/
    base.html           # layout shell — nav, footer, Google Fonts, CSS link
    index.html          # landing page (avatar, bio, social links, recent posts)
    section.html        # blog listing
    page.html           # single post
  sass/main.scss        # all styles — CSS custom-property design tokens, no JS
static/
  me.webp               # avatar referenced in zola.toml [extra]
```

The theme is fully self-contained in `themes/custom/`. All design tokens (colors, fonts, spacing) live as CSS custom properties in the `:root` block at the top of `themes/custom/sass/main.scss`. Dark-mode token overrides are stubbed-out (commented) there.

Sass is compiled by Zola directly (`compile_sass = true`); there is no separate build step for CSS.

## Content front matter

Blog posts use TOML front matter:

```toml
+++
title       = "Post title"
date        = 2026-01-01
description = "One-sentence summary shown in the post list."

[taxonomies]
tags = ["tag1", "tag2"]
+++
```

Social links on the landing page are driven by `[[extra.links]]` entries in `zola.toml`.
