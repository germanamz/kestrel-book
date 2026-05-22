# Kestrel — Learn C++ by Building a Networked Key-Value Store

## Overview

An mdbook that teaches C++ to experienced developers (11+ years in high-level
languages such as JavaScript/Python) by building one complex project from
scratch: **Kestrel**, a networked, persistent, concurrent key-value store.

The book is **concept-first**: chapters explain C++ concepts in depth and use
Kestrel code as the worked example to show *why* things work — not step-by-step
dictation. The book is **interactive with Claude Code**: the reader writes the
code; Claude acts as tutor and code reviewer.

## Audience

- Experienced software engineers fluent in higher-level languages.
- Assume strong general programming/CS background; do not re-teach concepts
  they already know (loops, recursion, sockets-as-a-concept, etc.).
- Teach what is C++-specific or has no analog in JS/Python.

## Goal

Full C++ mastery via the project. Coverage must include, driven by project need:

- Value vs reference semantics; copy vs move semantics; `std::move`; copy elision.
- RAII; rule of 0/3/5; ownership; smart pointers; custom allocators; alignment.
- Templates; concepts; type traits; `std::variant`; `std::span`.
- The compilation & linking model; preprocessor; ODR; translation units.
- Undefined behavior and the abstract machine.
- Error handling: `std::expected`, exceptions vs error codes, `noexcept`.
- STL containers, iterators (and invalidation), ranges.
- Concurrency: threads, atomics, the memory model (`memory_order`), data races.
- Tooling: CMake, sanitizers (ASan/UBSan/TSan), gdb/lldb.

## Technical decisions

- **Language standard:** C++23.
- **Dependencies:** raw POSIX + standard library only. No third-party libraries.
- **Platforms:** macOS and Linux (POSIX). No Windows.
- **Build system:** CMake.
- **Concurrency model:** evolve across the book — blocking → event loop
  (kqueue/epoll behind one interface) → event loop + worker thread pool.
- **No reference implementation.** All code the reader needs is illustrated
  inline in chapters. The reader builds their own `kvd`/`kvcli` in their own
  working directory.

## Project: Kestrel

Final feature set the reader builds:

- Custom binary-safe wire protocol over TCP.
- In-memory store with value types: strings, lists, sorted sets, hashes.
- TTL expiry and LRU eviction.
- Durability: write-ahead log + snapshots + crash recovery.
- Non-blocking event loop (kqueue/epoll) feeding a worker thread pool.
- CLI client (`kvcli`); server (`kvd`).
- Benchmarking and profiling.
- std-only, CMake, clean under ASan/UBSan/TSan.

Module layout the reader produces:

```
core/         Value type, Buffer, ownership & memory primitives
protocol/     wire format: serialize / parse (template-driven)
store/        hash table, expiry, eviction, data-type impls
persistence/  WAL, snapshot, recovery, compaction
net/          socket RAII wrappers, event loop, connections
concurrency/  thread pool, task queue, sharded keyspace
server/       kvd
client/       kvcli
test/         hand-rolled assertion + test-runner harness
```

Data flow: `socket → connection buffer → protocol parse → dispatch to worker →
store op → persistence → response serialize → socket`.

## Interaction model

- The reader writes Kestrel themselves, chapter by chapter.
- Concept chapters explain the *why* with illustrative code examples.
- Claude Code is tutor + reviewer (see CLAUDE.md contract below).
- Each chapter ends with a checkpoint the reader asks Claude to review.

## CLAUDE.md tutor + reviewer contract

A `CLAUDE.md` at the repo root defines Claude's behavior:

**Tutor:**
- Explain concepts deeply; relate answers back to the relevant chapter.
- Give a conceptual nudge before the full answer when the reader is stuck.
- Compile and run the reader's actual code before diagnosing.
- Default to running under ASan/UBSan/TSan.
- Do not race ahead of the reader's current chapter unless asked.

**Reviewer:** when the reader has written code, review it for:
- Antipatterns.
- Bad practices.
- Code structure.
- Code clarity.
- Simplicity.
- That it compiles (default under sanitizers).

## Algorithms covered

Hashing + open addressing; exact LRU and sampled (Redis-style) LRU; skip list;
ring buffer / deque; log-structured storage + compaction; CRC checksums; the
reactor pattern; MPSC queue; keyspace sharding.

## mdbook conventions

- `book.toml` with `src = "book"` (source directory named `book/`, not `src/`).
- `book/SUMMARY.md` plus one file per chapter under `book/partN/`.
- Code examples are tagged with their file path.
- Chapters explain concepts with code as illustration, not dictation.
- Callout boxes (via `mdbook-admonish`):
  - 🟦 **Coming from JS/Python** — contrast with the reader's mental model.
  - ⚠️ **UB / Gotcha** — undefined behavior and footguns.
  - 📐 **Algorithm** — the algorithm being implemented, with complexity.
  - 💬 **Ask Claude** — a ready-to-paste prompt at a common sticking point.
  - 🧪 **Experiment** — change X, predict output, then ask Claude why.
  - ✅ **Checkpoint** — what must compile/pass; ask Claude to review against it.

## Repo layout

```
book.toml
book/                 # mdbook chapters (SUMMARY.md + partN/*.md)
CLAUDE.md             # tutor + code-review contract
```

## Chapter outline (20 chapters, 8 parts)

**Part 0 — The C++ mental model & toolchain**
1. Why C++ feels alien — compilation/linking model, the abstract machine,
   undefined behavior, value vs reference semantics.
2. Toolchain & skeleton — compiler, CMake, sanitizers, debugger; build a tiny
   test harness from scratch; first runnable binary.

**Part 1 — Values & memory**
3. The `Value` type — stack vs heap, RAII, rule of 0/3/5, copy vs move
   semantics, `std::move`, copy elision.
4. Owning bytes (`Buffer`) — `unique_ptr`/`shared_ptr`/`weak_ptr`, custom
   allocators, alignment, `std::span`.
5. The type system — templates, concepts, type traits, `std::variant`.

**Part 2 — Protocol & streams**
6. Wire protocol & parsing — streams, `string_view`, zero-copy parsing,
   endianness, byte handling, a generic serializer via templates.
7. Error handling — `std::expected`, exceptions vs error codes, `noexcept`,
   RAII cleanup.

**Part 3 — The store & algorithms**
8. Hash table from scratch — open addressing vs chaining, hashing, load
   factor/rehash, iterator invalidation, vs STL containers.
9. Expiry & eviction — TTL, exact LRU (intrusive list + map), sampled LRU.
10. More data types — lists (ring buffer/deque), sorted sets (skip list).

**Part 4 — Persistence (filesystem & streams)**
11. Write-ahead log — append-only I/O, `fsync`, `std::filesystem`, file
    streams, CRC checksums, crash safety.
12. Snapshots & recovery — serialization format, log compaction, replay on
    startup.

**Part 5 — Networking**
13. Sockets from scratch — Berkeley sockets, RAII fd wrapper, blocking server,
    connection lifecycle.
14. The event loop — non-blocking I/O, kqueue/epoll behind one interface, the
    reactor pattern, connection state machines.
15. Buffered connection I/O — partial reads/writes, backpressure,
    protocol-meets-socket.

**Part 6 — Concurrency**
16. Threads & the memory model — `std::thread`/`jthread`, atomics,
    `memory_order`, data races, what TSan catches.
17. The worker pool — task queue, mutex/condition_variable, MPSC queue,
    sharding the keyspace to cut contention.
18. Tying threads to the loop — dispatch, cross-thread ownership/lifetime,
    atomic `shared_ptr`.

**Part 7 — Finishing**
19. The `kvcli` client.
20. Benchmark, profile, capstone — pipelining, optional pub/sub; observability.

**Appendices** — UB catalog; sanitizer guide; CMake reference; JS/Python→C++
glossary.

## Out of scope

- Windows support.
- Third-party libraries (including Asio, test frameworks).
- A maintained reference implementation.
