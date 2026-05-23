# Appendix B — Sanitizer Guide

Sanitizers are the C++ equivalent of the runtime safety net JavaScript
and Python give you for free. They are not part of the language — they
are *compiler-injected instrumentation* that detects classes of bugs at
runtime and reports them with precise stack traces. They are the single
most important productivity tool a C++ engineer has.

This appendix is a concise reference for the three Kestrel uses
(AddressSanitizer, UndefinedBehaviorSanitizer, ThreadSanitizer), plus
a few honorable mentions and the operational patterns that make them
pay off.

---

## The three you use

### AddressSanitizer (ASan)

**Catches:** out-of-bounds reads/writes on heap, stack, and globals;
use-after-free; use-after-return; use-after-scope; double-free;
mismatched `new`/`delete`; some leaks.

**Doesn't catch:** uninitialized reads (use MSan); data races (use
TSan); UB that doesn't touch memory (signed overflow, etc. — use
UBSan).

**Build:**

```bash
cmake -B src/build -S src -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address -fno-omit-frame-pointer"
```

**Cost:** ~2× slowdown, ~2× memory. Acceptable for testing; not for
production.

**Reading a report:** The first lines tell you the kind of error
(`stack-buffer-overflow`, `heap-use-after-free`, etc.) and the
addresses. The first stack trace is *where the bug happened*; the
second (when present) is *where the relevant allocation/deallocation
happened*. For use-after-free, you see "freed by thread X here:"
followed by the free's stack, then "previously allocated by thread X
here:" with the malloc's stack — pin-pointing both events.

**Knobs (`ASAN_OPTIONS` env var):**

- `detect_leaks=1` — turn on LeakSanitizer at exit (Linux only by
  default).
- `halt_on_error=0` — continue after first error (useful for tests
  that intentionally trip many).
- `abort_on_error=1` — coredump for postmortem.
- `symbolize=1` (default) — convert addresses to function/line names.

### UndefinedBehaviorSanitizer (UBSan)

**Catches:** signed integer overflow; division by zero; out-of-range
shifts; misaligned pointers; null pointer dereferences (in some forms);
invalid `enum` values; invalid `bool` values; vptr corruption (with
RTTI).

**Doesn't catch:** strict aliasing violations (those are hard); memory
bugs (use ASan).

**Build:** combine with ASan as Kestrel does:

```bash
-fsanitize=address,undefined -fno-omit-frame-pointer
```

**Cost:** small (~1.1× slowdown). Always-on in tests; cheap enough that
some shops ship it in production builds with `-fsanitize-trap` (turn
UB into a trap instruction).

**Reading a report:** "runtime error: <description>" plus a single
stack trace. Less informative than ASan but enough to find the line.

**Knobs:**

- `UBSAN_OPTIONS=print_stacktrace=1` (off by default!) — *do* set
  this. Without it, reports are one line and you have to find the call
  site yourself.

### ThreadSanitizer (TSan)

**Catches:** data races (the central concurrency UB from Chapter 17);
sync primitive misuse (e.g., destroying a locked mutex); deadlocks
under some configurations.

**Doesn't catch:** memory bugs (use ASan); atomic ordering bugs
(beyond establishing that synchronization exists at all).

**Build:** mutually exclusive with ASan. Use a separate build dir:

```bash
cmake -B src/build-tsan -S src -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=thread -fno-omit-frame-pointer"
```

**Cost:** ~5–10× slowdown, several × memory. Tests slow but bearable;
servers under load: not acceptable.

**Reading a report:** Two stack traces — the two unsynchronized
accesses to the same location — plus the relevant synchronization
edges (mutex locks/unlocks, atomic operations) the tool observed.
*Both* sides are important; the bug is the absence of a
happens-before edge between them. The fix is to add one — a mutex, an
atomic with the right ordering, a join.

**Knobs:**

- `TSAN_OPTIONS=second_deadlock_stack=1` — better deadlock traces.
- `TSAN_OPTIONS=halt_on_error=1` — stop on first race.

---

## Honorable mentions

### MemorySanitizer (MSan)

**Catches:** uninitialized reads. The thing ASan deliberately does not.

**Why not in Kestrel:** MSan requires the entire program — including
the standard library — to be instrumented. That means a separately
built libc++ (`libc++.asan`-style). It's a heavy infrastructure lift
for a teaching project; in production codebases it's worth it.

**Cost:** ~3× slowdown.

### LeakSanitizer (LSan)

**Catches:** memory leaks at process exit. ASan includes LSan on
Linux by default; on macOS, LSan must be invoked explicitly.

**Reading a report:** A list of "leaked N bytes, allocated here:" each
with the allocation stack trace.

### Static analyzers

Outside the runtime-instrumentation family but worth pairing:

- **clang-tidy** with the `clang-analyzer-*`, `bugprone-*`, and
  `modernize-*` checks. Finds bugs the compiler doesn't, before
  runtime.
- **cppcheck** — broad but noisier.
- **include-what-you-use (IWYU)** — finds missing/redundant `#include`s.

Run clang-tidy in CI alongside the sanitizer tests; it's the cheapest
review tool you'll ever add.

---

## The operational pattern

Three build directories is the norm for a real C++ project:

```text
src/build         # Debug + ASan + UBSan  (default development)
src/build-tsan    # Debug + TSan          (concurrency audit)
src/build-rel     # RelWithDebInfo        (benchmark / profile)
```

CI runs tests against the first two; sometimes the third. Treat any
sanitizer failure as you would a normal test failure — the failure is
real even when it's intermittent. The temptation to *suppress* a TSan
warning rather than fix the underlying race is one of the cardinal
sins of concurrent C++.

For the benchmark / profile build, sanitizers must be **off** —
they distort timing too much to draw conclusions from. See Chapter 21.

---

## Sanitizer-friendly habits

A few code patterns that avoid annoying sanitizer reports:

- **Initialize everything**, even when you "know" you'll write later.
  `int x{};` costs nothing and silences MSan and UBSan.
- **Move ownership through `unique_ptr`**, not raw pointers. ASan's
  use-after-free reports are precise when the free site is a
  destructor; less so when it's a manual `delete`.
- **Use `noexcept` only on functions that truly cannot throw.** A
  lying `noexcept` calls `std::terminate`, which sanitizers report,
  but the fix is to remove the false annotation.
- **`std::array` over C arrays**; `std::vector::at` for bounds-checked
  access in debug builds.
- **One mutex per logical resource**; sanitizers struggle with
  "ad-hoc" patterns where a single mutex protects a complex graph and
  the relationships are implicit.

---

## Suppressing what you cannot fix

Occasionally a sanitizer reports something you genuinely cannot fix
— a known issue in a third-party library, for instance. Both ASan and
TSan accept *suppression files*:

```bash
ASAN_OPTIONS=suppressions=/path/to/asan.supp ./mytest
TSAN_OPTIONS=suppressions=/path/to/tsan.supp ./mytest
```

The file is a list of patterns to ignore. Commit it; review it
periodically; never use it to hide your own bugs. Suppressions are a
documented escape, not a habit.

---

## Reading a TSan output, in detail

A typical TSan report:

```
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 4 at 0x7b0400001234 by thread T2:
    #0 Counter::increment() counter.cpp:7 (mybinary+0x42abc)
    #1 worker(int) worker.cpp:12 (mybinary+0x42d34)
    ...

  Previous read of size 4 at 0x7b0400001234 by thread T1:
    #0 Counter::value() counter.cpp:11 (mybinary+0x42b22)
    #1 main main.cpp:25 (mybinary+0x4301e)
    ...

  Location is heap block of size 8 at 0x7b0400001230 allocated by main thread:
    #0 operator new(unsigned long) ...
    #1 main main.cpp:21 (mybinary+0x42fce)
    ...

  Mutex M5 (0x...) created at:
    ...
```

Five sections:

1. **The error header**: kind (`data race`), pid.
2. **The first access**: write or read, size, address, stack trace.
3. **The previous access**: the *other* one in the racing pair.
4. **Allocation site**: where the racing memory was created.
5. **Synchronization context**: mutexes / atomics involved.

To fix: identify which two accesses are racing (sections 2 and 3),
add a synchronization edge between them (mutex, atomic, join, etc.).
The synchronization context (section 5) tells you what TSan *did*
see; if a mutex is listed but the race still happened, both threads
must lock the *same* mutex on both sides of the access.

---

## See also

- Appendix A — UB Catalog: bugs sanitizers find, listed by category.
- Chapter 2 — Toolchain & Skeleton: first introduction; the ASan/UBSan
  build flags that have been on the whole book.
- Chapter 17 — Threads & the Memory Model: TSan's natural habitat.
