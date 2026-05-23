# Introduction

This is a book about learning **modern C++** by building one real,
non-trivial project from scratch: **Kestrel**, a networked, persistent,
concurrent key-value store, wire-compatible with `redis-cli`. By the
end, you will have written a working server and client; you will
understand every layer of the stack you built; and you will be fluent
in the C++ idioms — RAII, ownership, move semantics, templates,
sanitizers, the memory model — that hold the whole thing together.

## Who this book is for

You have shipped systems in higher-level languages (JavaScript, Python,
Ruby, Go) for at least a few years. You know what a hash table is, what
a TCP socket is, what a thread is, what undefined behavior *should*
mean. You don't need to be re-taught what a `for` loop does or what
recursion is. What you need is a guided tour through the things C++
gets wrong from your point of view — and the deep reasons those wrong
things are right for the language.

You do not need prior C++ experience. The book introduces every C++
concept it uses, in depth, from a perspective that assumes you've been
shaped by the languages above. Each chapter contrasts the C++ behavior
with the JS/Python analog where the contrast helps.

## How the book works

The book is **concept-first**: each chapter explains a C++ topic in
depth, with Kestrel code as the worked example. It is *not*
step-by-step dictation. The reader writes the code; Claude Code acts as
the tutor and code reviewer (see `CLAUDE.md` in the repo root for the
contract).

Each chapter ends with a **Checkpoint** describing what your `src/`
should now do — that's where you ask Claude to review against the
chapter's standard before moving on. The checkpoints are the only
"correctness checks" in the book; the body teaches the *why*, the
checkpoint asks you to demonstrate the *what*.

## The trade-offs the book makes

- **Modern C++23**, not C++11/14. The constraint sharpens design and
  removes large categories of complexity from earlier standards.
- **Raw POSIX + the standard library**, with one exception (a test
  framework introduced in Chapter 8 to teach C++ dependency
  management). No Boost, no Asio, no HTTP libraries. The point of the
  constraint is that you build every layer yourself.
- **macOS and Linux only.** Windows support is out of scope; the
  POSIX-based design is what the book is teaching.
- **No reference implementation.** Code in chapters is illustrative —
  enough to show the shape, not enough to copy-paste into a working
  program. You build Kestrel in `src/` and Claude reviews it.

## What you will learn

By the time you finish, you will have hands-on familiarity with:

- Ownership: `unique_ptr`, `shared_ptr`, `weak_ptr`, and the rule of
  five.
- Value semantics: copy, move, copy elision, `std::move`.
- The type system: templates, concepts, type traits, `std::variant`,
  `std::visit`.
- Error handling: `std::expected`, exceptions, `noexcept`, RAII
  cleanup.
- `<chrono>`, `<filesystem>`, `<bit>`, `<span>`, `<string_view>`.
- The compilation and linking model — translation units, headers, ODR.
- Undefined behavior, the abstract machine, the as-if rule.
- The memory model, atomics, mutexes, condition variables.
- Sanitizers — ASan, UBSan, TSan — as a real safety net.
- CMake, `FetchContent`, target-based dependency management.

You will also have hand-implemented: a binary-safe value type, a
buffer, an open-addressing hash table with intrusive LRU, a ring
buffer, a skip list, an incremental RESP parser, a write-ahead log
with CRC32 and `fsync`, atomic snapshot publication, RAII wrappers
around POSIX file descriptors, an event loop over `kqueue`/`epoll`, a
worker pool with sharded keyspace, and cross-thread wakeup via
`eventfd`.

## The shape of the journey

- **Part 0** — *The C++ mental model & toolchain.* Why C++ feels alien,
  and the compiler / CMake / sanitizers / debugger / first-binary setup
  that pays off the chapter's claims.
- **Part 1** — *Values & memory.* `Value`, `Buffer`, smart pointers,
  templates, `std::variant`. The data model.
- **Part 2** — *Protocol, streams & dependencies.* RESP parsing,
  `std::expected`, and the dependency-management tour that brings in
  the test framework.
- **Part 3** — *The store & algorithms.* Hash table, expiry, eviction,
  lists, hashes, sorted sets.
- **Part 4** — *Persistence.* The WAL and snapshots.
- **Part 5** — *Networking.* Sockets, the event loop, buffered
  connection I/O — Kestrel becomes a server.
- **Part 6** — *Concurrency.* Threads, the memory model, the worker
  pool, cross-thread coordination.
- **Part 7** — *Finishing.* The `kvcli` client; benchmark, profile,
  capstone.
- **Appendices** — UB catalog, sanitizer guide, CMake reference,
  JS/Python → C++ glossary.

The chapters are sequential. Skipping is not recommended on a first
read — every chapter builds on the previous, and the C++ topic in each
chapter is taught at the moment Kestrel forces you to confront it.

## Setting up

Before Chapter 1, do nothing. Chapter 1 is purely conceptual — no
compiler required. Chapter 2 walks you through installing a recent
Clang or GCC and a recent CMake, opening Kestrel's first build, and
running its first test. If you want to verify your toolchain ahead of
time:

```bash
c++ --version       # need C++23 support: roughly Clang 17+ or GCC 13+
cmake --version     # need 3.20+
```

If those report new enough versions, you are ready.

Take a breath. The first chapter is one you can read without typing
anything. Start there.
