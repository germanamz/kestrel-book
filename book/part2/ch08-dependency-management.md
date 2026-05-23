# C++ Dependency Management

This is the only chapter in the book where you add an external library.
Kestrel's stack is otherwise raw POSIX plus the standard library — by design,
because the project is about C++, not about gluing libraries together. But you
will work in real C++ codebases that pull in dozens of third-party libraries,
and the way C++ handles dependencies is unlike anything in your Node or PyPI
muscle memory. So before we get back to Kestrel proper in Part 3, we spend
one chapter on how C++ dependencies actually work, and we use that chapter to
do something Kestrel needs anyway: replace the hand-rolled `CHECK` harness
from Chapter 2 with a real test framework.

The library we adopt is **doctest**, chosen because it is a single header,
a real and widely used framework, and tiny — perfect for teaching the
mechanics without the lesson getting buried under a build-system goliath. By
the end of this chapter you will have run a real package through CMake's
`FetchContent`, understood the trade-off against `find_package` and against
"package managers" like vcpkg and Conan, and rewritten every Kestrel test
against doctest's assertion macros.

---

## The no-npm reality

In Node and Python, dependency management is a solved problem in the sense
that you do not think about it. `npm install`, `pip install`, a `package.json`
or `requirements.txt`, a registry, a lockfile, and you move on. The runtime
loads the modules at run time; you never link anything, you never reason about
ABI compatibility, you almost never read the dependency's source.

C++ does not have any of that built in. The standard does not define a
package manager. The compiler does not know how to download a library. The
language has no module loader at run time (you remember Assumption 1). So
"using a library" in C++ is several distinct problems, all of which someone —
maybe you, maybe a tool — has to solve:

1. **Acquire the source or binary.** Clone a repo, download a tarball, install
   a system package, point at a local checkout. There is no canonical
   registry.
2. **Build it** (if you have source) or **find it** (if you have a binary
   installed system-wide).
3. **Tell the compiler where the headers are** (`-I/path/to/include`).
4. **Tell the linker where the library is** (`-L/path/to/lib -lfoo`).
5. **Match the ABI**: the library and your code must agree on layout,
   calling convention, exception model — usually settled by compiling them
   the same way, but a real pitfall when binaries cross compilers or standard
   library versions.

The reason there isn't *one* answer is that C++ targets too many environments
— Linux distros where the system already has the library, embedded targets
with no network, build farms with strict reproducibility rules — to fit
under one workflow. The result is a *menu* of tools that each pick a different
slice of those five problems.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`package.json` lives somewhere between problems (1) and (3): it lists names
and versions, and the runtime takes care of (3)–(5) for free. C++ does not
have problem (5) handled for you, ever; you must compile and link against a
library built compatibly with your code. That is the deepest difference, and
it is why C++ teams pick a build system and an acquisition method per
project, then live with that choice for years.

</div>

---

## The menu, briefly

You will encounter four families of approaches in real codebases. Understand
the trade-off of each; we will pick one for Kestrel.

### `find_package` + system / vendored install

The classic. The library is installed on the machine (via the OS package
manager — `apt`, `brew`, `dnf` — or built and installed by hand), and your
`CMakeLists.txt` calls `find_package(Foo REQUIRED)` to locate the headers
and the linker artifacts. `find_package` looks in well-known locations and
asks the library's own CMake module ("foo-config.cmake") where it landed.

Pros: zero configuration if the library is in the system; trivially the
fastest builds (the library is already compiled); ABI guaranteed-compatible
across the system's toolchain.

Cons: the library must already be there. Onboarding a new contributor means
"install these 12 things first." Reproducibility lives at the mercy of the
OS package versions.

Common pattern in Linux server projects, established C++ shops, and almost
all OS-level software.

### `FetchContent` (CMake itself fetches and builds the source)

CMake 3.11+ includes `FetchContent`, which clones (or downloads) a
dependency's source and adds it to your build like a subdirectory of your
own project. The whole thing — fetch, configure, compile, link — is one
`cmake -B build` away. No package manager required.

Pros: one command, no system installs, the dependency is built with *your*
toolchain and flags (ABI matches by construction). Excellent for small
projects and teaching code.

Cons: the build downloads source on the first configure; the dependency
contributes to your build time on every clean build. Not appropriate for
huge libraries (Boost, LLVM) where you'd want a binary.

This is what Kestrel will use. It fits the book: zero machine setup beyond
having a compiler and CMake, fully reproducible because the version is
pinned in `CMakeLists.txt`.

### `vcpkg` / `Conan` (real package managers)

Two competing ecosystems that look most like `npm` to someone arriving from
JavaScript. Each maintains a registry of recipes, each downloads and builds
binaries, each integrates with CMake. **vcpkg** is Microsoft-led and very
simple to start with; **Conan** is Python-based and more flexible (multiple
profiles, binary caches).

Pros: handle the "huge library" case well. Binary caching means you don't
rebuild Boost every clone. Treat dependencies as data, not as build code.

Cons: another tool to install and manage. Onboarding a contributor still
involves installing the package manager. Worth it for large, multi-team C++
projects; overkill for one library.

Kestrel will not use these, but you should be able to read a `vcpkg.json` or a
`conanfile.txt` when you encounter them.

### Header-only inclusion (just paste the header in)

The smallest libraries — doctest, nlohmann/json, fmt's header-only mode — are
distributed as a single `.hpp` file. The "dependency management" is "drop the
file in your repo, `#include` it, done." This is unfashionable but
unbeatable for small, leaf utilities where versioning churn is low and the
library is short enough to read.

Pros: no tooling at all. Maximum reproducibility.

Cons: doesn't scale beyond a handful of libraries. You take on the burden of
updating manually.

For doctest, header-only would work too. We'll use `FetchContent` instead,
because that's the more transferable lesson.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

The closest npm equivalent of each: `find_package` is "the package is
installed globally" (rare in JS, more like Python's system packages on Linux).
`FetchContent` is "the package is bundled as a git submodule and built with
the parent" — closer to `npm link` of a local checkout. `vcpkg`/`Conan` is
real `npm`. Header-only is `cat foo.js >> bundle.js` and pretending the rest
didn't happen. They all coexist in real C++ shops, often within the same
repo.

</div>

---

## Header-only vs compiled

A second axis cuts across the menu above. A C++ library can be:

- **Header-only**: the entire implementation lives in headers. Every
  translation unit that uses the library compiles a fresh copy of its code.
  The "linker" doesn't really care — there is no library to link. Fast to
  integrate; slower to compile *every* TU; no ABI questions; cannot share a
  single compiled object across consumers.
- **Compiled** (`.a` static or `.so` / `.dylib` dynamic): the library is a
  *separate* object file you link against. Headers carry only declarations;
  definitions live in the library. Faster to compile per-TU; the library is
  compiled once; ABI now matters — your binary and the library must agree on
  layouts, exception model, runtime library version.

Most modern small libraries (doctest, nlohmann/json, range-v3 for headers
only) are header-only because they exist in templates that *cannot* be
non-header anyway (Chapter 5's "templates live in headers"). Most large
libraries (OpenSSL, ICU, system libraries) are compiled because they predate
template-everything and because separate compilation is the only way to keep
build times sane.

**ABI** — the **A**pplication **B**inary **I**nterface — is the contract a
compiled library makes with its consumers about how a class is laid out in
memory, how a function expects its arguments, how exceptions are propagated.
Two binaries with *compatible* ABIs can call each other; two with
incompatible ABIs link and then crash. The most common C++ ABI break is
"library built with one version of `libstdc++`, your binary built with
another"; the second most common is "library built with exceptions disabled,
your binary built with them on." `FetchContent` sidesteps both because
everything is built with the *same* compiler invocation.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Mixing C++ standards (one TU compiled `-std=c++17`, another `-std=c++23`),
or mixing standard-library implementations (libstdc++ vs libc++), can produce
binaries that link but crash, because templates instantiate to subtly
different layouts. The rule that prevents this entire class of bug:
**build every TU in a project, including dependencies, with the same compiler
and the same flags.** This is exactly why we set CMake's `CXX_STANDARD` once
at the top level — it propagates to every target, including dependencies
fetched by `FetchContent`.

</div>

---

## Adopting doctest with FetchContent

Let's do it. The change to `src/CMakeLists.txt` is small. We pin doctest to a
known release tag (use a current release; the snippet below uses `2.4.11` as
a placeholder — pick the latest stable at the time you read this), and create
a top-level `doctest::doctest` target that any test can link against.

```cmake
# src/CMakeLists.txt  (additions)

include(FetchContent)

FetchContent_Declare(
  doctest
  GIT_REPOSITORY https://github.com/doctest/doctest.git
  GIT_TAG        v2.4.11        # pin to a tagged release for reproducibility
)
FetchContent_MakeAvailable(doctest)
```

`FetchContent_MakeAvailable` is the modern one-liner that fetches the
project, runs `add_subdirectory` on it, and exposes whatever targets the
upstream `CMakeLists.txt` defines. For doctest, that's an interface library
named `doctest::doctest`.

A doctest test program looks like this — a single TU that defines doctest's
implementation, plus your tests:

```cpp
// src/test/main_test.cpp
#define DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN
#include <doctest/doctest.h>
```

That `#define` triggers doctest's "I will be the main translation unit"
mode, which generates `main()` for you. Every other test TU in this binary
just `#include`s `<doctest/doctest.h>` without the define.

Rewriting the parser tests:

```cpp
// src/test/protocol_test.cpp
#include <string_view>
#include <variant>
#include <doctest/doctest.h>

#include "protocol/parse.hpp"
#include "protocol/serialize.hpp"

using namespace std::string_view_literals;

TEST_CASE("parses a full GET command") {
    using namespace kestrel::protocol;
    constexpr auto cmd = "*2\r\n$3\r\nGET\r\n$3\r\nkey\r\n"sv;
    std::size_t consumed = 0;
    auto r = parse(cmd, consumed);
    REQUIRE(r);                                  // not an error
    REQUIRE(r->has_value());                     // and a value, not "need more"
    CHECK(consumed == cmd.size());
    CHECK(std::holds_alternative<Array>(**r));
}

TEST_CASE("returns need-more for torn input") {
    constexpr auto half = "*2\r\n$3\r\nGE"sv;
    std::size_t consumed = 0;
    auto r = kestrel::protocol::parse(half, consumed);
    REQUIRE(r);
    CHECK_FALSE(r->has_value());
    CHECK(consumed == 0);
}

TEST_CASE("reports unknown-type errors") {
    constexpr auto bad = "?garbage\r\n"sv;
    std::size_t consumed = 0;
    auto r = kestrel::protocol::parse(bad, consumed);
    REQUIRE_FALSE(r);
    CHECK(r.error().code == kestrel::protocol::ErrorCode::UnknownType);
}
```

The CMake target:

```cmake
add_executable(protocol_test
  test/main_test.cpp
  test/protocol_test.cpp
  protocol/parse.cpp
  protocol/serialize.cpp
  core/buffer.cpp
)
target_link_libraries(protocol_test PRIVATE doctest::doctest)
add_test(NAME protocol_test COMMAND protocol_test)
```

`target_link_libraries(... PRIVATE doctest::doctest)` is how a target says "I
depend on this." The `PRIVATE` keyword means the dependency is *not*
transitive — anything that depends on `protocol_test` (nobody does) would not
also see doctest. The keywords `PRIVATE`, `PUBLIC`, and `INTERFACE` model the
transitivity of a dependency in the CMake target graph, and they are the
single most important thing to learn about CMake beyond `add_executable`. We
will use them throughout.

The same treatment applies to `version_test` and `value_test`: rename the
`CHECK`s, link `doctest::doctest`, and the rewrite is done. The hand-rolled
`test/check.hpp` file is now unused — delete it, and its mention from any
CMake target. We will not miss it; it served its purpose.

### What just happened on the first build

When you run `cmake -S src -B src/build` the first time after adding
doctest, you will see CMake clone the doctest repo into
`src/build/_deps/doctest-src` and configure it. Subsequent builds reuse the
cache — the clone happens once per build directory. Delete `src/build/` and
it reclones; that's expected.

Inspect `src/build/_deps/doctest-src/doctest/doctest.h` once. It is one
header, ~7000 lines, the entire framework — including how it captures
expressions, formats failures, and provides its main. Reading the *first
thousand lines* is well worth it — it's a master class in modern C++
template machinery, and it demystifies the macros you'll be writing in tests
every day.

<div class="callout callout-experiment">

🧪 **Experiment**

Confirm `FetchContent` is doing what you think. Delete `src/build/`,
configure with `cmake -S src -B src/build -DCMAKE_BUILD_TYPE=Debug ...`, and
*watch the output*. You will see a `-- Fetching doctest` line and then the
configure step for doctest itself. Now build: doctest contributes only the
header; nothing of doctest's source is compiled into your binary because it
is an interface (header-only) library. Run `ctest --test-dir src/build
--output-on-failure`. The output format is now doctest's — colored, with
each `TEST_CASE` printed by name.

</div>

---

## Two harder questions worth knowing the answers to

### Versioning and lockfiles

In Node, `package-lock.json` pins every transitive dependency. In C++ there
is no built-in equivalent. Best practices:

- For `FetchContent`, pin a **`GIT_TAG`** to a tag or commit hash, never to a
  branch like `main`. A branch can change under you; a tag/hash cannot.
- For `find_package`, the *system* is the lock — different systems get
  different versions. Document the required version in your README.
- For vcpkg / Conan, use the **manifest mode** (`vcpkg.json`, `conanfile.txt`)
  with version constraints. Both ecosystems now have lockfile equivalents.

Kestrel pins doctest by tag and that's enough for our purposes.

### License compatibility

The thing nobody covers in a tutorial and that you must know in real life:
C++ libraries come under various licenses (MIT, BSD, Apache-2, LGPL, GPL,
proprietary), and *linking* — especially statically linking — propagates
license obligations. Read each dependency's `LICENSE` file before adopting
it; doctest is MIT, which is permissive and combines freely with any project.
GPL libraries require your binary to be GPL (or LGPL-style dynamic linking).
This is not a C++ technicality, it is a real legal constraint your
employer's lawyers care about.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's my updated `src/CMakeLists.txt` after pulling in doctest via
> `FetchContent` and my rewritten `test/protocol_test.cpp`. Review the CMake
> changes: is `FetchContent_MakeAvailable` the right way to wire this in
> today, am I pinning the version correctly, and are `target_link_libraries`
> using `PRIVATE`/`PUBLIC` keywords correctly? Walk me through what would
> change if I switched to `find_package(doctest REQUIRED)` instead.

</div>

---

## Where Part 2 ends

Kestrel now speaks RESP, has a typed error path, and uses a real test
framework. The repo also carries one external dependency, declared in CMake
and pinned, which is exactly the form your day-job C++ projects will take.

The lesson generalizes. Most C++ third-party adoption boils down to:

1. **Decide your acquisition strategy** (FetchContent, find_package, vcpkg,
   header-only) per dependency, weighing build time, ABI risk, and
   reproducibility.
2. **Build everything with the same toolchain and flags.** ABI lives or dies
   here.
3. **Use CMake's `PRIVATE`/`PUBLIC`/`INTERFACE` keywords** to express which
   dependencies are part of your public API vs internal.
4. **Pin versions** by tag or hash, never by branch.
5. **Read the license**, every time.

Part 3 returns to Kestrel proper: a hand-written hash table (Chapter 9),
expiry and eviction (Chapter 10), and the list/sorted-set/hash data types
(Chapter 11) that fill in the placeholders in `core/value.hpp`. From here
out, every chapter is C++ in service of a real algorithm. The framework you
just adopted is what you'll catch each algorithm's bugs with.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and tests must pass under ASan/UBSan, using doctest now:

```bash
cmake -S src -B src/build -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

You should be able to:

- Name the four mainstream approaches to C++ dependencies and the trade-off
  of each.
- Explain what ABI is and the most common ways it breaks.
- Distinguish header-only from compiled libraries and predict which form a
  template-heavy library will take.
- Read a `target_link_libraries(... PRIVATE ...)` line and explain the
  transitivity rule it sets.
- Pin a `FetchContent` dependency to a specific tag and say why pinning to a
  branch is a bug.

Hand it to Claude:

> Review my Part 2 final state: `src/CMakeLists.txt` (with `FetchContent` for
> doctest), the rewritten tests, and the deletion of `test/check.hpp`. Are
> there any leftover references to the old harness? Are my `target_link_libraries`
> using the right `PRIVATE`/`PUBLIC` for each target? Anything else I should
> tidy before Part 3?

When everything is green, Part 3 begins — and we build the heart of Kestrel,
the hash table, by hand.

</div>
