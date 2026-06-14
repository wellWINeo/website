---
description: "Translate a blog post between Russian and English with humanized output and peer review"
argument-hint: "<path/to/post.md>"
---

# Translate

Translate the blog post at `$ARGUMENTS`. Follow every step below — none are optional.

## Step 1 — Detect language and derive output path

Read the file at `$ARGUMENTS`.

**Language detection (by suffix):**
| Source suffix | Source language | Target language | Output path |
|---|---|---|---|
| `.ru.md` | Russian | English | Same name, drop `.ru` → `foo.ru.md` → `foo.md` |
| `.md` (no `.ru`) | English | Russian | Insert `.ru` → `foo.md` → `foo.ru.md` |

If the output file already exists, read it too — you will overwrite it.

## Step 2 — Invoke the humanize-text skill

**Before writing a single translated sentence**, invoke the `humanize-text` skill. Apply its principles throughout the translation.

For **English → Russian** output: additionally enforce the forbidden-words list from humanize-text (`таким образом`, `безусловно`, `данный`, `осуществляется`, `в рамках`, `на сегодняшний день`, etc.).

For **Russian → English** output: apply the general humanize-text principles (varied sentence lengths, natural connectives, no symmetric lists) — there is no forbidden-words list for English, but stilted phrasing is still banned.

## Step 3 — Translate

### Preserve verbatim — do not modify, translate, or paraphrase

- **Fenced code blocks** (` ```…``` `) — every character inside, including inline comments
- **Inline code spans** (`` `identifier` ``) — every character
- **All URLs** and hyperlink targets `[text](url)` — translate `text`, keep `url` as-is
- **Brand names and platform nouns**: .NET, C#, Go, Java, JVM, CLR, JIT, GC, TAP, P/Invoke, ASP.NET, async/await, Task, ValueTask, Span, Unsafe, etc.
- **Type/method/field names**: `AsyncTaskMethodBuilder`, `IAsyncStateMachine`, `MoveNext`, `AwaitUnsafeOnCompleted`, `ExecutionContext`, `SynchronizationContext`, `AsyncLocal<T>`, etc.
- **Markdown structure**: heading levels, table column layout, list nesting depth, image syntax `![alt](path)`
- **TOML frontmatter keys**: `title`, `date`, `description` — keys stay, only string values are translated

### Translate

- All Russian/English body text and heading text
- Table cell content that is prose (not identifiers)
- The string values of `title` and `description` in the TOML frontmatter block

### Precision rules

- Translate technical concepts by their established term in the target language. Do not invent calques.
  - e.g. "state machine" → "стейт-машина" (not "машина состояний" if the author already uses "стейт-машина")
  - e.g. "green threads" → "зелёные потоки" (established term)
  - e.g. "stack unwind" → "размотка стека" (keep consistent with how the source uses it)
- Match the author's own terminology choices in the source — if they use a specific translation of a term, use it consistently.
- Numbers, dates, and statistics: copy exactly; do not reformat.

## Step 4 — Write the output file

Write the complete translated document to the output path from Step 1.

## Step 5 — Spawn a review subagent (mandatory)

Spawn a subagent with the task below. Do not skip this step even if the file is short or the translation feels clean.

---

**Subagent task:**

You are a translation reviewer. Read both files in full, then report on each of the following:

1. **Missing content** — paragraphs, sections, list items, or sentences present in the source but absent or significantly condensed in the translation.
2. **Mis-translated identifiers** — technical identifiers, class names, method names, brand names, or platform nouns that were translated instead of preserved verbatim.
3. **Code block divergence** — any difference between source and translation inside fenced code blocks or inline code spans.
4. **Stilted or machine-sounding phrasing** — sentences in the translation that read as AI-generated: uniform sentence lengths, symmetric list structures, forbidden AI-marker words (for Russian: `таким образом`, `безусловно`, `данный`, `осуществляется`, `в рамках`, `на сегодняшний день`).
5. **Factual drift** — numbers, names, dates, or benchmark figures that differ between source and translation.

Source: `$ARGUMENTS`
Translation: `<output path from Step 1>`

Return a numbered list of issues found, with the source line or section quoted for each issue. If there are no issues, respond with: **No issues found.**

---

After the subagent returns, fix every reported issue in the translation file, then report the final output path to the user.
