# Toolchain & Skeleton

Chapter 1 was about rewiring how you think. This chapter is where that new model
becomes machinery you can touch. You are going to install a compiler, describe
Kestrel's build to CMake, produce the first runnable binary — a `kvd` that prints
its version and exits — turn on the sanitizers that claw back the runtime's old
safety net, meet the debugger, and write a tiny test harness from scratch. By the
end, `src/` will contain real, compiling, tested C++.

This is the chapter that pays off the promises Chapter 1 made. It said the
compile-and-link pipeline is ahead-of-time, per-file, and explicit; here you build
the tool that drives that pipeline. It said sanitizers catch undefined behavior at
the moment it happens; here you switch them on and watch one fire. It said
deterministic destruction and value semantics are coming in Chapter 3 — so we will
deliberately *not* touch `Value`, `Buffer`, or any of Part 1 yet. The goal now is a
skeleton: the smallest possible thing that compiles, links, runs, and is tested,
so that every later chapter has a real project to grow.

Because the build *is* a first-class part of C++ — there is no runtime to resolve
your code for you (Assumption 1) — learning the toolchain is not a chore you do
before the real work. It is the real work. A C++ engineer who cannot read a build
or a sanitizer report is as stuck as a JavaScript engineer who cannot read a stack
trace.

---

## The compiler is the system

In Node or CPython you install *a runtime* and feed it source. In C++ you install
*a compiler* and it feeds the bare machine your translated program. There is no
`node` sitting underneath the result at run time (Assumption 1), so the compiler's
identity and version matter enormously: it is the single tool that decides which
language features exist, how aggressively your code is optimized, and what
diagnostics you see.

Two compilers matter for this book:

- **Clang** (the LLVM compiler), default on macOS and excellent everywhere.
- **GCC** (the GNU compiler), default on most Linux distributions.

Either is fine; Kestrel targets both, plus macOS and Linux only. We use **C++23**,
so you need a recent toolchain: roughly **Clang 17+** or **GCC 13+**. Check what
you have:

```bash
c++ --version          # the generic "a C++ compiler" alias
clang++ --version      # Clang specifically
g++ --version          # GCC specifically
```

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`node --version` tells you which runtime will execute your code tonight in
production. `c++ --version` tells you which *translator* will build your program —
and after that the compiler is gone. The binary it emits has no dependency on the
compiler's version, only on the OS and C++ standard library it linked against. The
version you care about is a *build-time* fact, not a *run-time* one. This split
between build-time and run-time concerns is new, and it runs through everything
that follows.

</div>

On macOS, the `clang++` shipped by Apple inside Xcode's Command Line Tools
(`xcode-select --install`) usually suffices; if it is too old for C++23, install a
current LLVM with `brew install llvm` and use that `clang++`. On Linux, your
distribution's `g++` or `clang++` package will do; install a newer one if the
version check above falls short. If you are unsure whether your compiler is new
enough, the cleanest test is to make it build something — which is exactly what we
do next.

---

## Why a build system exists at all

In JavaScript, `import './store.js'` is resolved while your program runs: the
runtime finds the file, executes it, and hands you its exports. We established in
Chapter 1 that C++ has no such moment. Every connection between files — every
`#include`, every call from one translation unit into another — is wired up
*before* the program exists, by the compiler and the linker.

That means *someone* has to know the whole plan: which `.cpp` files exist, where
their headers live, which flags each must be compiled with, and how the resulting
object files link into `kvd`. For one file you could type the compiler command by
hand. For the dozens of files Kestrel will grow into, across two operating systems,
with sanitizer and warning flags that must stay consistent, you need that plan
written down and executed mechanically. That written-down plan is a **build
system**, and ours is **CMake**.

CMake does not compile anything itself. It is a *generator*: you describe your
project in a `CMakeLists.txt`, and CMake produces the actual build instructions
(typically a `Makefile` or a Ninja file) for your platform. You then run that
build. Two steps, always in this order:

1. **Configure** — CMake reads `CMakeLists.txt` and generates a build directory.
2. **Build** — the generated tool invokes the compiler and linker for real.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`CMakeLists.txt` sits in roughly the same slot as `package.json`, but its job is
the inverse of what you are used to. `package.json` mostly answers *"what do I
download?"* — npm fetches dependencies a runtime will later resolve. Kestrel
downloads almost nothing (raw POSIX + the standard library; one test framework
arrives in Chapter 8). `CMakeLists.txt` answers a question npm never has to:
*"what is the exact sequence of compile and link commands that turns this text
into a binary?"* There is no install step that defers the wiring to run time —
the wiring is the build.

</div>

### Kestrel's first `CMakeLists.txt`

Create `src/CMakeLists.txt`. Here is the whole thing for the skeleton; we will walk
through every line, because each one corresponds to a decision Chapter 1 forced.

```cmake
# src/CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(kestrel CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Warnings are not optional in this book: the compiler is your first reviewer.
add_compile_options(-Wall -Wextra -Wpedantic)

# Headers are included relative to src/, e.g. #include "core/version.hpp".
include_directories(${CMAKE_SOURCE_DIR})

# The server binary, kvd. One translation unit of its own today; many later.
add_executable(kvd
  server/main.cpp
  core/version.cpp
)

enable_testing()
```

Reading it top to bottom:

- `project(kestrel CXX)` names the project and declares it C++-only (no C, no
  Fortran) so CMake does not go hunting for compilers it will never use.
- The three `CMAKE_CXX_*` lines pin the language: standard **23**, *required*
  (fail loudly rather than silently fall back to an older standard), and
  **extensions off** so we compile against portable ISO C++ (`-std=c++23`) rather
  than a compiler's GNU dialect (`-std=gnu++23`). Portability between Clang and GCC
  is a project requirement, and this is where you enforce it.
- `add_compile_options(-Wall -Wextra -Wpedantic)` turns on the compiler's
  diagnostics. In a language with no runtime to catch your mistakes, every warning
  the compiler offers is free signal — treat warnings as the cheapest code review
  you will ever get.
- `include_directories(${CMAKE_SOURCE_DIR})` makes `src/` the root for header
  lookups, so `#include "core/version.hpp"` means the same thing from every file.
- `add_executable(kvd ...)` is the heart of it: it declares a binary and *lists
  the translation units that compose it*. This is the explicit plan — the thing a
  module loader would otherwise discover at run time — written out by hand.
- `enable_testing()` switches on CTest, CMake's test driver; we wire an actual
  test into it later in the chapter.

Notice that `add_executable` names two source files for one binary. That is
Chapter 1's diagram made literal: `main.cpp` and `version.cpp` are compiled
*independently* into object files, then the linker stitches them into `kvd`. We are
about to write both.

---

## The first binary

The skeleton needs the smallest amount of code that still exercises the
multi-file pipeline — because the pipeline, not the code, is the lesson here. So
`kvd` will print its version and exit, and the version will live in its own
translation unit. That split is not busywork: it is the concrete payoff of
Chapter 1's declaration-versus-definition distinction.

Create a header that *declares* the version function, and a `.cpp` that *defines*
it:

```cpp
// src/core/version.hpp
#pragma once

namespace kestrel {

// Returns Kestrel's build version as a C string literal.
const char* version();

}  // namespace kestrel
```

```cpp
// src/core/version.cpp
#include "core/version.hpp"

namespace kestrel {

const char* version() { return "0.0.1"; }

}  // namespace kestrel
```

And the entry point:

```cpp
// src/server/main.cpp
#include <cstdio>

#include "core/version.hpp"

int main() {
    std::printf("kvd (Kestrel) %s\n", kestrel::version());
    return 0;
}
```

Look at what the compiler sees when it compiles `main.cpp`. It reads the
`#include "core/version.hpp"`, which gives it the *declaration* `const char*
version();` — a promise that such a function exists somewhere with that signature.
That promise is enough to type-check the call `kestrel::version()`. The compiler
never sees `version.cpp` while compiling `main.cpp`; it does not need to. It emits
`main.o` with an unresolved reference, and the **linker** later connects that
reference to the definition in `version.o`. This is the entire reason headers exist,
and you just used it on purpose.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`#pragma once` is the C++ answer to a problem `import` never has. Because
`#include` is literal text substitution (Chapter 1's preprocessor), including the
same header twice in one translation unit would paste its declarations in twice —
a redefinition error. `#pragma once` tells the preprocessor "paste me at most once
per translation unit." A JS module is deduplicated by the loader for free; a C++
header is deduplicated by you, with one line at the top of every header you write.

</div>

A subtle but important detail: `version()` is declared in the header and *defined*
in the `.cpp`. Why not just write the body in the header? Because if two
translation units both included a header that *defined* the function, the linker
would see *two* definitions of `kestrel::version` and reject the program — a
violation of the **One Definition Rule**. The header carries the declaration (you
may repeat it freely); exactly one `.cpp` carries the definition. Keeping that
boundary clean is a habit you build now and rely on for the rest of the book.

### Configure, build, run

From the repo root, run the two-step dance — and add the sanitizer flags now, so
we never get used to building without them (more on what they do in the next
section):

```bash
cmake -S src -B src/build -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"
cmake --build src/build
```

- `-S src` points CMake at the source directory (where `CMakeLists.txt` lives).
- `-B src/build` is the **out-of-source** build directory — all generated files
  and object files land there, never mixed into your source tree. It is in
  `.gitignore` for exactly that reason; you can delete `src/build/` at any time and
  lose nothing but compile time.
- `-DCMAKE_BUILD_TYPE=Debug` selects an unoptimized, debuggable build (`-O0 -g`).
  We will live in Debug for almost the whole book; optimized builds come up only
  when we benchmark.

If it built, run it:

```bash
./src/build/kvd
# kvd (Kestrel) 0.0.1
```

That line of output is your first C++ program for Kestrel: compiled ahead of time
from two translation units, linked into one binary, running directly on the
processor with no interpreter beneath it.

---

## Sanitizers: renting back the safety net

Chapter 1 spent its longest section on undefined behavior, and ended it with a
promise: that we would build Kestrel under **sanitizers** to catch UB at the moment
it happens. Those `-fsanitize=...` flags you just passed are that promise kept.

A sanitizer is *instrumentation* the compiler weaves into your binary. It is not a
runtime in the JS/Python sense — it does not interpret your code or make UB defined
— but it does insert checks around the operations that are most dangerous, so that
when you cross a line, the program stops and prints a precise report instead of
silently corrupting itself. You are renting back, at build time and with some
runtime cost, a slice of the safety net the interpreter used to give you for free.

Three sanitizers matter for Kestrel:

- **AddressSanitizer (ASan)** — catches memory errors: reading or writing past the
  end of a buffer, use-after-free, and similar. The single highest-value tool you
  have against the bugs Chapter 1 warned about.
- **UndefinedBehaviorSanitizer (UBSan)** — catches a grab-bag of other UB: signed
  integer overflow (the very thing behind Chapter 1's `x * 2 / 2` experiment), bad
  shifts, misaligned pointers, null dereferences in some cases.
- **ThreadSanitizer (TSan)** — catches data races between threads.

You combine ASan and UBSan in one build (`-fsanitize=address,undefined`), which is
why we pass exactly that. TSan is mutually exclusive with ASan — they instrument
memory in incompatible ways — so it gets its *own* build, and we will not need it
until Kestrel has threads in Part 6. The `-fno-omit-frame-pointer` flag keeps stack
frames intact so the reports name your functions instead of raw addresses.

<div class="callout callout-experiment">

🧪 **Experiment**

Make a sanitizer fire, on purpose, so you recognize the report when a real bug
produces one. Temporarily edit `src/server/main.cpp` to read past the end of a
small stack array before printing:

```cpp
int main() {
    int xs[3] = {1, 2, 3};
    int oops = xs[3];          // index 3 is one past the end — UB
    std::printf("kvd (Kestrel) %s (%d)\n", kestrel::version(), oops);
    return 0;
}
```

Rebuild (`cmake --build src/build`) and run `./src/build/kvd`. Predict first: in
Python this would be an `IndexError`; what happens here? Without ASan, the most
likely outcome is that it prints *some* number and exits cleanly — the silent,
"works today" face of UB. With ASan compiled in, the program halts and prints a
`stack-buffer-overflow` report pointing at the exact line. Read the report top to
bottom, then paste it to Claude and ask it to walk you through each section. When
you are done, **delete the bad line** — we keep the skeleton clean.

</div>

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Sanitizers find UB *that your program actually executes*. They are dynamic checks,
not a proof: if your test never runs the buggy path, ASan never sees it. This is
the deep reason the book pairs sanitizers with tests — the sanitizer can only catch
what you exercise, so the more behavior your tests drive, the more UB the
sanitizers can surface. A clean ASan run on a code path you never triggered tells
you nothing about that path.

</div>

---

## The debugger: stopping the bare machine

A sanitizer tells you *that* something went wrong. A **debugger** lets you stop the
program mid-execution and look around: inspect variables, walk the call stack, step
one line at a time. In a language with no runtime narrating your program's
progress, the debugger is how you watch the bare machine code do its work.

Use **lldb** on macOS and **gdb** on Linux; they share most concepts. A first
session on `kvd`:

```text
$ lldb ./src/build/kvd
(lldb) b main                 # set a breakpoint at main
(lldb) run                    # start; it stops at the breakpoint
(lldb) p kestrel::version()   # evaluate an expression in the stopped program
(lldb) n                      # step to the next line
(lldb) c                      # continue to the end
```

(`gdb ./src/build/kvd` then `break main`, `run`, `print`, `next`, `continue` is the
same flow.) The breakpoint works because the **Debug** build carried debug info
(`-g`) — a map from machine instructions back to your source lines and variable
names. That map is what lets the debugger speak in terms of `main` and
`kestrel::version()` rather than raw addresses.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

This is recognizably the same activity as `debugger;` in Node DevTools or
`breakpoint()` in Python — pause, inspect, step. The difference is what you are
inspecting. There, the runtime hands the debugger a tidy object graph. Here, the
debugger reads debug info to reconstruct your variables from registers and stack
memory, because at run time there is no runtime holding that structure for it. It
is the same job done one layer lower.

</div>

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Debug under `Debug` builds, not optimized ones — and now you know why from Chapter
1. Under the as-if rule (Assumption 2), an optimizing compiler reorders statements,
folds computations away, and keeps variables in registers that never touch memory.
A debugger then reports "variable optimized out" or jumps between lines in an order
that looks insane, because the machine code no longer corresponds line-for-line to
your source. `-O0` keeps that correspondence intact so the debugger can tell you
the truth.

</div>

---

## A test harness from nothing

Kestrel needs tests from the start, and here we hit a fact that surprises everyone
arriving from npm or PyPI: C++ has no blessed, built-in test runner, and we are not
allowed a third-party one yet (dependency management is its own chapter — Chapter 8
— and that is where we adopt a real framework). So we build the smallest test
harness that could possibly work, from scratch. It will teach you more about how
C++ programs actually run than any framework would, and throwing it away later for
a real one is itself part of Chapter 8's lesson.

The core insight: **a test is just a program.** Its *exit code* is its verdict —
`0` means success, anything else means failure, by long Unix convention. CTest, run
the test program, looks at the exit code. That is the entire contract. No
framework, no magic, no runtime — just a `main` that returns the right number.

We need one helper: a way to assert a condition and record a failure. Create a tiny
header-only harness:

```cpp
// src/test/check.hpp
#pragma once

#include <cstdio>

namespace kestrel::test {

// Incremented by CHECK each time a condition is false. A test program returns
// 0 from main only if this stayed 0 — i.e. every check passed.
inline int failures = 0;

}  // namespace kestrel::test

// CHECK must be a macro, not a function: only the preprocessor can capture the
// source *text* of the condition (#cond) and the call site (__FILE__/__LINE__).
// A function would receive only the already-computed bool, with no way to print
// what was tested or where.
#define CHECK(cond)                                                  \
    do {                                                             \
        if (!(cond)) {                                               \
            std::fprintf(stderr, "FAIL %s:%d: CHECK(%s)\n",          \
                         __FILE__, __LINE__, #cond);                 \
            ++kestrel::test::failures;                               \
        }                                                            \
    } while (0)
```

Two details earn their place here:

- **Why a macro.** Macros are the preprocessor's blunt text tool, and modern C++
  avoids them — but this is one of the few jobs only a macro can do. `#cond`
  stringizes the condition's source text so the failure message can show exactly
  what was tested; `__FILE__` and `__LINE__` capture where. A normal function
  receives only the *value* `true` or `false`, having lost the expression and the
  location. (C++20's `std::source_location` recovers the location without a macro,
  but not the expression text — so the macro stays. We will not need that nuance
  yet.)
- **Why `inline int failures`.** An `inline` variable in a header has *one* shared
  definition across every translation unit that includes it — the One Definition
  Rule again, this time letting us safely put a global in a header. Without
  `inline`, each `.cpp` including this header would define its own `failures` and
  the linker would reject the duplicates.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

The `do { ... } while (0)` wrapper around the macro body is a classic C++ idiom,
not noise. It makes the multi-statement macro behave like a single statement, so
that `if (x) CHECK(y); else ...` parses correctly instead of the `else` binding to
a stray brace. Bare `{ }` would leave a trailing semicolon problem; `do/while(0)`
swallows the semicolon cleanly. You will see this pattern in every hand-rolled
C++ macro.

</div>

Now a first test. It exercises `version()`, which means the test binary links
against `core/version.cpp` just as `kvd` does — another concrete instance of the
linker stitching translation units together:

```cpp
// src/test/version_test.cpp
#include <cstring>

#include "core/version.hpp"
#include "test/check.hpp"

int main() {
    // Compare the string *contents*, not the pointers. In C++, == on two
    // const char* compares addresses, not characters — almost never what you want.
    CHECK(std::strcmp(kestrel::version(), "0.0.1") == 0);
    CHECK(kestrel::version()[0] == '0');

    return kestrel::test::failures == 0 ? 0 : 1;
}
```

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

That `strcmp` is guarding against a real trap. In JavaScript and Python, `==` on
two strings compares their *contents*, so `"0.0.1" == "0.0.1"` is naturally true.
In C++, `kestrel::version()` returns a `const char*` — a pointer — and `==` on two
pointers compares the *addresses* they hold, not the characters they point at. The
comparison might happen to be true (the compiler may pool identical literals) or
false, and either way it is asking the wrong question. `std::strcmp(...) == 0`
compares characters. Part 1's real string type will give us back content
comparison; until then, this is the seam to watch.

</div>

### Wire it into the build

Tell CMake to build the test binary and register it with CTest. Add to
`src/CMakeLists.txt`, after the `kvd` target:

```cmake
# The test harness is header-only (test/check.hpp); each test is its own
# program whose exit code is the verdict.
add_executable(version_test
  test/version_test.cpp
  core/version.cpp
)

add_test(NAME version_test COMMAND version_test)
```

Re-configure (CMake notices the changed `CMakeLists.txt` on the next build, but
running configure explicitly is clearer the first time), build, and run the tests:

```bash
cmake -S src -B src/build -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

You should see `version_test` pass. Because the whole thing was compiled with
`-fsanitize=address,undefined`, that passing run is also a clean sanitizer run over
everything the test exercised — the two tools working together exactly as the
gotcha above described.

<div class="callout callout-experiment">

🧪 **Experiment**

Confirm the harness actually fails when it should — a test that cannot fail is
worse than no test. Add a deliberately wrong check, e.g.
`CHECK(std::strcmp(kestrel::version(), "9.9.9") == 0);`, rebuild, and run `ctest
--output-on-failure`. Watch the `FAIL test/version_test.cpp:NN: CHECK(...)` line
appear and the test report as failed (because `main` returned `1`). Then remove the
bad check. You have now seen both verdicts the harness can produce, which is the
only way to trust it.

</div>

<div class="callout callout-ask">

💬 **Ask Claude**

> Here is my `src/CMakeLists.txt`, `src/test/check.hpp`, and
> `src/test/version_test.cpp`. I'm coming from JS/Python and this is my first
> CMake project. Walk me through what `cmake -S src -B src/build` actually
> generates in the build directory, what `cmake --build` then does with it, and how
> `ctest` decides my test passed. Where in this flow does the *linker* run, and
> what would a missing `core/version.cpp` in my `add_executable` look like when it
> failed?

</div>

---

## Where the skeleton stands

`src/` now holds a real, if tiny, project:

```text
src/
  CMakeLists.txt        # the build plan: kvd + version_test, C++23, warnings, CTest
  core/
    version.hpp         # declaration
    version.cpp         # definition
  server/
    main.cpp            # kvd entry point
  test/
    check.hpp           # the hand-rolled CHECK macro + failure counter
    version_test.cpp    # first test, run by CTest
```

Everything Chapter 1 described in the abstract is now something you can run: the
ahead-of-time pipeline (CMake → compiler → linker → `kvd`), the
declaration/definition split that makes headers necessary, the sanitizers that
catch UB the instant it executes, the debugger that stops the bare machine, and the
plain truth that a test is just a program returning an exit code. The next chapter
starts filling this skeleton with the heart of Kestrel — the `Value` type — and
that is where value semantics, RAII, and the rule of 0/3/5 stop being Chapter 1
vocabulary and become code you write.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Your `src/` should now build and test cleanly. Concretely, from the repo root, all
of this should succeed:

```bash
cmake -S src -B src/build -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"
cmake --build src/build
./src/build/kvd                              # prints: kvd (Kestrel) 0.0.1
ctest --test-dir src/build --output-on-failure   # version_test passes
```

The build must be **warning-free** (`-Wall -Wextra -Wpedantic` are on) and the test
run **clean under ASan/UBSan**. Make sure you can also:

- Explain why `kvd` lists *two* source files and what the linker does with them.
- Explain why `version()` is declared in the header but defined in the `.cpp`, in
  terms of the One Definition Rule.
- Set a breakpoint in `main` with lldb/gdb and print `kestrel::version()`.

When it all passes, hand your code to Claude:

> Review my Chapter 2 Kestrel skeleton: `src/CMakeLists.txt`,
> `src/core/version.{hpp,cpp}`, `src/server/main.cpp`, `src/test/check.hpp`, and
> `src/test/version_test.cpp`. I'm an experienced engineer new to C++. Check it for
> non-idiomatic C++, header hygiene, anything that won't build clean under
> `-Wall -Wextra -Wpedantic` with ASan/UBSan, and whether my CMake target setup is
> sane. Tell me what's already idiomatic and what you'd change, with the reason.

Once Claude signs off and everything above is green, you have a real C++ project —
on to Chapter 3 and the `Value` type.

</div>
