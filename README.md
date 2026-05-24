# Kestrel — Learn C++ by Building a Networked Key-Value Store

A book that teaches **modern C++ (C++23)** by building one real, non-trivial
project from scratch: **Kestrel**, a networked, persistent, concurrent
key-value store, wire-compatible with `redis-cli`.

It is written for engineers who have shipped systems in higher-level languages
(JavaScript, Python, Ruby, Go) and want a guided tour through the things C++
gets *uniquely* right and uniquely wrong. The book is concept-first: each
chapter explains a C++ topic in depth, with Kestrel code as the worked example.
You write the implementation; Claude Code acts as your tutor and reviewer (see
[`CLAUDE.md`](./CLAUDE.md) for the contract).

Read the [introduction](./book/introduction.md) for the full pitch.

## Use this as a template

This repository is a [GitHub template](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template).
To read along and keep your own implementation progress in its own repo:

1. Click **"Use this template" → "Create a new repository"** at the top of the
   GitHub page.
2. Name it whatever you like (e.g. `kestrel`).
3. Clone your new repo and start working through the book. Your implementation
   lives under `src/` and is yours to commit, push, and break.

The book itself is also published as a website. Read it locally (below) or
follow the link in the repo's "About" section once it's deployed.

## Read the book locally

The book is written in [mdBook](https://rust-lang.github.io/mdBook/).

```bash
cargo install mdbook        # one-time; requires mdbook >= 0.5
mdbook serve                # live preview at http://localhost:3000
```

Static HTML is built to `docs/book/` (configured in `book.toml`).

## Build Kestrel (the project you're writing)

The reader's implementation lives in [`src/`](./src). It starts as a skeleton
and grows chapter by chapter. The standard build flow (introduced in
Chapter 2):

```bash
cmake -S src -B src/build -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

Target: C++23, raw POSIX + standard library only, plus one test framework
(doctest/Catch2, introduced in Chapter 8). macOS and Linux.

## Repository layout

```
book/        the book's source (Markdown, organized by part/chapter)
src/         where the reader implements Kestrel
theme/       mdBook theme overrides (callouts, etc.)
docs/        built HTML output (gitignored except as published)
AUTHORING.md guidelines for editing the book itself
CLAUDE.md    the tutor + reviewer contract for Claude Code
```

## License

[MIT](./LICENSE). Use it, fork it, teach from it.
