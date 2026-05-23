# Appendix C — CMake Reference

The CMake bits Kestrel uses, in one place, with the rationale. This is
not a complete CMake reference — that would be a book — but it covers
every command that appears in `src/CMakeLists.txt` plus the small handful
you'd add for a real project.

CMake's syntax is its own little programming language with quirks; the
rules below are conservative and modern (the `target_*` style introduced
around CMake 3.0+, which is the only style you should write today).

---

## The shape of a CMakeLists.txt

A minimum Kestrel-style file:

```cmake
cmake_minimum_required(VERSION 3.20)
project(kestrel CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(-Wall -Wextra -Wpedantic)
include_directories(${CMAKE_SOURCE_DIR})

add_executable(kvd
  server/main.cpp
  core/version.cpp
)

enable_testing()

include(FetchContent)
FetchContent_Declare(
  doctest
  GIT_REPOSITORY https://github.com/doctest/doctest.git
  GIT_TAG        v2.4.11
)
FetchContent_MakeAvailable(doctest)

add_executable(version_test
  test/main_test.cpp
  test/version_test.cpp
  core/version.cpp
)
target_link_libraries(version_test PRIVATE doctest::doctest)
add_test(NAME version_test COMMAND version_test)
```

Read top to bottom — each command does exactly one thing. The pattern
scales to dozens of targets.

---

## Commands you use constantly

### `cmake_minimum_required(VERSION x.y)`

The minimum CMake version your script depends on. Setting this also
sets the *policy* level, which is how CMake guards against
backward-incompatible changes. Pick a version recent enough to use the
features you want; Kestrel needs 3.20+ for `FetchContent_MakeAvailable`
and a few language features.

### `project(name [LANGUAGES...])`

Names the project and declares which languages it uses. Naming the
languages (`CXX` for C++, `C` for C, `Fortran` for, well, Fortran) tells
CMake which compilers to look for. Omitting them means "C and C++."

### `set(VAR value)` / `set(CACHE_VAR value CACHE TYPE "docstring")`

Plain variable assignment. The `CACHE` form publishes the variable to
the user (visible in `ccmake` or `cmake -D`), with a type (`BOOL`,
`STRING`, `FILEPATH`, `PATH`) for the GUI hint.

The three Kestrel-relevant ones:

- `CMAKE_CXX_STANDARD` — the C++ standard version (17, 20, 23).
- `CMAKE_CXX_STANDARD_REQUIRED ON` — fail if the compiler can't deliver
  it (else CMake silently falls back).
- `CMAKE_CXX_EXTENSIONS OFF` — compile as ISO C++, not the compiler's
  GNU dialect.

### `add_executable(name source1.cpp source2.cpp ...)`

Declares an executable target. The sources are listed explicitly — no
globs (globs hide ABI hazards when files are added or removed silently).

### `add_library(name [STATIC|SHARED|INTERFACE] source.cpp ...)`

Declares a library target. `STATIC` = `.a`, `SHARED` = `.so`/`.dylib`,
`INTERFACE` = header-only (no compilation unit; the target carries
include paths and compile flags only). Kestrel doesn't define libraries
explicitly because we link `.cpp`s directly into binaries; for a bigger
project, factor shared code into a `STATIC` library and link binaries
against it.

### `target_link_libraries(target [PRIVATE|PUBLIC|INTERFACE] dep1 dep2 ...)`

Declare dependencies of a target. The keyword controls *transitivity*:

- **`PRIVATE`** — `target` uses the dep; nothing that links against
  `target` sees it. The default for most cases.
- **`PUBLIC`** — `target` uses the dep, *and* anything that links
  against `target` also gets it. Use when the dep appears in
  `target`'s public headers.
- **`INTERFACE`** — `target` doesn't use the dep itself, but its
  consumers do. Rare; mostly for INTERFACE libraries.

The single most-misunderstood thing in CMake is the
`PRIVATE`/`PUBLIC`/`INTERFACE` distinction. Get it right and your
dependency graph is exactly what you intended; get it wrong and you
either link too much (binary bloat) or too little (missing-symbol
errors).

### `target_include_directories(target PRIVATE|PUBLIC|INTERFACE dir1 ...)`

Per-target include paths. The modern alternative to the older
project-wide `include_directories`. For a clean design, use this; for
small projects, the project-wide `include_directories` Kestrel uses is
forgivable.

### `target_compile_options(target PRIVATE|PUBLIC|INTERFACE -flag ...)`

Per-target compile flags. The modern alternative to project-wide
`add_compile_options`. Same keywords as link libraries.

### `enable_testing()` + `add_test(NAME name COMMAND ...)`

Registers a test with CTest. The command can be a binary name (resolved
to the path of the target if it's an `add_executable` target) followed
by args.

### `include(Module)`

Loads a CMake module — either built-in (`FetchContent`, `CMakePackageConfigHelpers`,
`GoogleTest`) or one of your own from a `cmake/` subdirectory.

### `FetchContent_Declare` + `FetchContent_MakeAvailable`

The "fetch and integrate a third-party dependency" pattern. See
Chapter 8 for the worked example with doctest. Pin to `GIT_TAG` (never
`GIT_BRANCH`), or use `URL` for a downloaded tarball.

---

## Commands you should know exist

### `find_package(Foo REQUIRED)`

Locates an already-installed library. The other half of the
"acquisition strategy" decision Chapter 8 covers. Looks for
`fooConfig.cmake` or `Findfoo.cmake` in standard locations.

### `option(NAME "description" DEFAULT)`

A boolean configuration variable surfaced to the user. Equivalent to
`set(NAME DEFAULT CACHE BOOL "description")`. Use for features like
`option(KESTREL_BUILD_BENCH "Build benchmarks" OFF)`.

### `if() / elseif() / else() / endif()`

CMake's conditionals. Beware: variables are referenced as both `${VAR}`
and the bare name in some contexts — read the docs for the operator
you're using.

### `foreach() / endforeach()`

Loop. Useful for "add a test for each .cpp in test/", but explicit
listing is cleaner and avoids the glob hazard.

### `function() / endfunction()` and `macro() / endmacro()`

CMake's own functions. Use them to factor repetitive target setup.

### `install(TARGETS target DESTINATION dir)`

Defines what `cmake --install` does. Out of scope for Kestrel, which is
run from the build tree, but needed for any deployed library.

### `configure_file(input output)`

Substitutes `@VAR@` in a template file. Useful for generating
`version.hpp` from a CMake variable, or generating a config file from
CMake variables.

### `add_subdirectory(dir)`

Includes another `CMakeLists.txt` from a subdirectory. Used for
splitting a large project; `FetchContent_MakeAvailable` uses it
internally.

---

## CMake patterns Kestrel uses

### Setting language standard *globally*

```cmake
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
```

Set once, applied to every target including fetched dependencies. This
is *the* defense against ABI breaks from mixed standard versions.

### Sanitizer flags via `CMAKE_CXX_FLAGS`

```bash
cmake -B src/build -S src -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer"
```

Sanitizers as project-wide flags applied at configure time. The
alternative — `target_compile_options(target PRIVATE -fsanitize=...)`
— is *per target* and forgets to apply to dependencies fetched by
`FetchContent`. For a teaching project, project-wide is right.

### Multiple build types in parallel build dirs

```bash
cmake -B src/build       ...                  # ASan/UBSan
cmake -B src/build-tsan  ...-fsanitize=thread # TSan
cmake -B src/build-rel   ...-DCMAKE_BUILD_TYPE=Release
```

CMake supports any number of out-of-source build directories; switching
between them is just `cmake --build src/build-tsan`.

### One executable per test file

```cmake
add_executable(value_test test/main_test.cpp test/value_test.cpp core/value.cpp)
target_link_libraries(value_test PRIVATE doctest::doctest)
add_test(NAME value_test COMMAND value_test)
```

The pattern Kestrel uses: each test is a separate binary; doctest is
linked into each. Faster compiles (parallel), clearer test boundaries.
The alternative is one mega-binary with all tests; this scales worse
once you have many.

---

## Things to avoid

- **`file(GLOB)` for source lists.** It hides newly-added files until a
  reconfigure; introduces ordering instability into the build. Always
  list sources explicitly.
- **Variables for source lists.** `set(MY_SOURCES a.cpp b.cpp)` then
  `add_executable(foo ${MY_SOURCES})` works but obscures what each
  target builds; for a single target, inline the list.
- **`include_directories` at project level** (we use it, though,
  because it keeps Kestrel's CMake tiny). Prefer
  `target_include_directories(target PRIVATE ...)` for non-trivial
  projects.
- **`add_definitions`** is the legacy form of
  `target_compile_definitions`; the latter has correct transitivity
  rules and is what you want.
- **Mixing `cmake_minimum_required` with `cmake_policy`.** The
  `cmake_minimum_required` form alone sets policies to match the
  declared version. Don't fight it.

---

## A handful of `cmake` CLI invocations

```bash
# Configure
cmake -B build -S src

# Configure with options
cmake -B build -S src -DCMAKE_BUILD_TYPE=Debug -DKESTREL_BUILD_BENCH=ON

# Build (incremental)
cmake --build build

# Build with N parallel jobs
cmake --build build -j 8

# Build only one target
cmake --build build --target kvd

# Run tests
ctest --test-dir build --output-on-failure

# Run a single test
ctest --test-dir build -R value_test --output-on-failure -V
```

The `ctest` invocations live in your fingers by chapter 8. The build
flow doesn't change much from there — every new chapter adds two or
three CMake lines and the existing flow keeps working.

---

## See also

- Chapter 2 — Toolchain & Skeleton: first CMakeLists.txt, walked through
  line by line.
- Chapter 8 — Dependency Management: `FetchContent` introduced; the
  full `target_link_libraries` story.
- Chapter 20 — The kvcli Client: a second binary in the same project;
  example of factoring shared code.
