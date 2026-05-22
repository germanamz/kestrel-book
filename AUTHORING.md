# Authoring Guide

Read this before writing or editing any chapter. It is the shared contract that
keeps independently-authored chapters consistent. The full design is in
`docs/superpowers/specs/2026-05-21-kestrel-cpp-book-design.md` — read the
relevant chapter's entry there too.

## What this book is

A concept-first guide that teaches **modern C++ (C++23)** to experienced
engineers by building **Kestrel**, a networked, persistent, concurrent key-value
store. The reader writes the code; Claude Code tutors and reviews (see
`CLAUDE.md`). Each chapter explains C++ concepts in depth and uses Kestrel as the
running example.

## Audience — calibrate to this

- 10+ years in higher-level languages (JavaScript, Python).
- Strong on general programming and CS. Do **not** re-teach loops, recursion,
  hash tables as a concept, what a socket is, etc.
- New to C++ *semantics*. Go deep on what is C++-specific or has no analog in
  their languages (value/move semantics, RAII, templates, UB, the build model,
  the memory model).

## Voice & approach

- **Concept-first, not instruction-first.** Explain *why* the language works the
  way it does. Use Kestrel code to *illustrate* the concept — never reduce the
  chapter to "type these lines."
- **Motivate every feature from the project's needs.** Introduce a C++ concept at
  the moment Kestrel requires it.
- **The reader writes the code.** Show illustrative examples and explain them;
  do not present a copy-paste-complete solution. Prompt the reader to implement,
  then verify with Claude.
- Direct, concrete, technical. No filler, no hype. Respect the reader's time.

## Cumulative consistency

Chapters build on each other. Before authoring chapter N:

- Read chapters `1 … N-1` so terminology, type names, and the state of the
  Kestrel codebase are consistent.
- Reuse names exactly as earlier chapters defined them (e.g. `Value`, `Buffer`).
- Do not use concepts or APIs the book hasn't introduced yet. If you need one,
  either it belongs in an earlier chapter or you introduce it here deliberately.

## Code in chapters

- Target C++23, raw POSIX + standard library only (plus the one test framework
  introduced in Chapter 8). macOS and Linux only.
- Tag every code block with its file path under `src/`, e.g.
  `// src/core/value.hpp`.
- Show idiomatic, modern C++. Code shown must be code you would actually ship.
- Keep examples focused — show the part that illustrates the concept, not an
  entire file dump, unless the file is the point.

## Callout vocabulary (plain HTML, no preprocessor)

Callouts are raw HTML `<div>` blocks styled by `theme/callouts.css`. mdbook
passes HTML through and still renders Markdown inside it, **as long as you leave
a blank line after the opening `<div>` and before the closing `</div>`**. Syntax:

```markdown
<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

In JavaScript, objects are reference types; in C++, `Value` is copied by
default — assignment makes a *copy*, not an alias.

</div>
```

| Callout | `<div>` class | Lead line | Use for |
|---|---|---|---|
| Coming from JS/Python | `callout callout-jsts` | `🟦 **Coming from JS/Python**` | Contrast with the reader's mental model |
| UB / Gotcha | `callout callout-ub` | `⚠️ **UB / Gotcha**` | Undefined behavior and footguns |
| Algorithm | `callout callout-algo` | `📐 **Algorithm**` | The algorithm being implemented, with complexity |
| Ask Claude | `callout callout-ask` | `💬 **Ask Claude**` | A ready-to-paste prompt at a common sticking point |
| Experiment | `callout callout-experiment` | `🧪 **Experiment**` | "Change X, predict the output, then ask Claude why" |
| Checkpoint | `callout callout-checkpoint` | `✅ **Checkpoint**` | What must compile/pass before moving on |

Always start the callout body with the bold lead line from the table, then a
blank line, then the content.

Every chapter ends with a **✅ Checkpoint** describing what the reader's `src/`
should now do, framed as "ask Claude to review your code against this."

## Chapter structure (guideline, not rigid)

1. Short intro: what we're adding to Kestrel and which C++ concept it forces.
2. The concept, explained in depth, with illustrative Kestrel code.
3. 🟦 / ⚠️ / 📐 callouts woven in where relevant.
4. 💬 Ask Claude prompts at known sticking points.
5. 🧪 Experiment(s) to build intuition.
6. ✅ Checkpoint.

## Mechanics

- One file per chapter under `book/partN/` (paths already stubbed; see
  `book/SUMMARY.md`). Do not rename files or change `SUMMARY.md` ordering.
- Verify the book builds before committing: `mdbook build` (no errors).
- Commit per chapter: `git commit -m "book: write chapter NN — <title>"`.
