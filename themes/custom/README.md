# personal — a minimal Zola theme

Warm, pastel, editorial. Built for a personal landing page + blog.

## Structure

```
themes/personal/
├── theme.toml
├── templates/
│   ├── base.html       # nav, footer, fonts, CSS
│   ├── index.html      # landing page (/)
│   ├── section.html    # blog listing (/blog/)
│   └── page.html       # single post
└── sass/
    └── main.scss       # all styles with CSS custom-property tokens
```

## Installation

```bash
git clone <this-repo> themes/personal
```

Then in your `config.toml`:

```toml
theme = "personal"
```

See `config.example.toml` for the full set of `[extra]` options.

## Content structure

```
content/
├── _index.md         # landing page (front matter only, body unused)
└── blog/
    ├── _index.md     # blog section — set title + description here
    ├── my-first-post.md
    └── ...
```

### blog/_index.md

```toml
+++
title       = "Writing"
description = "Occasional posts on software and ideas."
sort_by     = "date"
# paginate_by = 10   # uncomment to enable pagination
+++
```

### A blog post

```toml
+++
title       = "My post title"
date        = 2026-06-08
description = "One-sentence summary shown in the post list."

[taxonomies]
tags = ["rust", "tools"]
+++

Post body in Markdown...
```

## Customisation

All design tokens are CSS custom properties in `:root` inside `sass/main.scss`.
To override without touching the theme, create `static/custom.css` and load it
by adding to your `config.toml`:

```toml
# no built-in mechanism — add a <link> to base.html or use Sass @use
```

Or simply edit `sass/main.scss` directly — it is self-contained.

## Dark mode

A complete set of dark-mode token overrides is stubbed (commented out) near
the top of `sass/main.scss`. Uncomment and fill in the values to enable it.
Add a small JS snippet to your `base.html` to toggle `body.dark`.

## Fonts

Loads **Crimson Pro** (headings) and **Source Sans 3** (body) from Google Fonts.
To self-host, download the WOFF2 files, place them in `static/fonts/`, and
replace the `<link>` in `base.html` with `@font-face` rules in `main.scss`.
