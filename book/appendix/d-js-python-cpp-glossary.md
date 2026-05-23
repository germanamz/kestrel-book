# Appendix D — JS/Python → C++ Glossary

A reference of the patterns you used in JavaScript and Python and the
closest C++ analogs. The point is not "they are the same" — they rarely
are — but to give you a starting place when a familiar instinct misfires.

The chapters this appendix complements: every "Coming from JS/Python"
callout in the book. This is the index.

---

## Variables, references, copies

| JS / Python                    | C++                                                 | Notes                                                                                  |
|--------------------------------|-----------------------------------------------------|----------------------------------------------------------------------------------------|
| `let a = obj; let b = a;`      | `auto a = obj; auto b = a;`                          | C++ *copies*; JS/Python *alias*. (Ch.1, Ch.3)                                          |
| Pass by reference (default)    | Pass by value (default)                             | C++ defaults to copy on function call. Pass `T&` or `const T&` for reference.          |
| `b = a.copy()` (Python)        | `auto b = a;`                                       | The copy is the default in C++; the alias is the special request.                      |
| `b = a` (alias)                | `auto& b = a;`                                      | A reference is C++'s aliasing tool.                                                    |
| `del obj`                      | (none — out-of-scope destroys it)                   | Variables die at the closing brace; no manual delete needed if you used RAII.          |
| `if (x is not None):` (Python) | `if (x.has_value()):` (`std::optional`)             | C++ has no built-in "nullable"; `std::optional<T>` makes it explicit.                   |

---

## Memory & ownership

| JS / Python                 | C++                                  | Notes                                                                |
|-----------------------------|--------------------------------------|----------------------------------------------------------------------|
| Garbage collector           | RAII + smart pointers                | Deterministic destruction; no GC. (Ch.1, Ch.4)                       |
| Object reference            | `std::shared_ptr<T>`                 | Reference-counted; closest analog to GC behavior.                    |
| Sole owner (you've never said this) | `std::unique_ptr<T>`         | Default in C++; nonexistent in JS/Python.                            |
| Weak reference              | `std::weak_ptr<T>`                   | Same idea.                                                            |
| `new` / `delete`            | `std::make_unique` / RAII            | Raw `new`/`delete` is a code smell in modern C++.                    |
| `class X { ~X() {} }`       | Destructor                           | Runs at scope exit, deterministically.                                |

---

## Strings & bytes

| JS / Python                  | C++                                              | Notes                                                                          |
|------------------------------|--------------------------------------------------|--------------------------------------------------------------------------------|
| `string`                     | `std::string` (owning) or `std::string_view`     | `string` owns; `string_view` is a non-owning view.                              |
| `bytes`                      | `std::vector<std::uint8_t>` or `std::span`       | Sequences of bytes; same ownership distinction.                                  |
| `+` concat                   | `string + string` (allocates)                    | `string::append` to grow in place is cheaper.                                  |
| `"foo" == "foo"` (text eq)   | `s1 == s2` for `std::string`                      | But `==` on `const char*` compares pointers, not contents.                      |
| `s.split()`                  | `std::ranges::views::split` or manual            | Less ergonomic; no built-in `split` on `string`.                                |
| `f"{x}"` (Python f-string)   | `std::format("{}", x)` (C++20)                    | Same idea, type-safe.                                                            |
| `print(...)`                 | `std::println("...")` (C++23) or `std::cout`     | Buffered to stdout; `\n` flushes when stdout is a TTY.                            |

---

## Collections

| JS / Python              | C++                                  | Notes                                                                  |
|--------------------------|--------------------------------------|------------------------------------------------------------------------|
| `Array` / `list`         | `std::vector<T>`                     | Dynamic array; the right default sequence container.                     |
| `Map` / `dict`           | `std::unordered_map<K, V>`           | Hash-table-based; not ordered.                                          |
| `SortedDict` (Py 3rd-party) | `std::map<K, V>`                  | Red-black tree; ordered by key.                                          |
| `Set` / `set`            | `std::unordered_set<T>` or `std::set<T>` | Same distinction.                                                    |
| `Tuple`                  | `std::tuple<T1, T2, ...>` or `struct` | Prefer a named `struct` when fields have meaning.                          |
| `dataclass`              | `struct`                             | Plain aggregate. Default copy, move, destructor.                          |
| `deque`                  | `std::deque<T>`                      | Chunked array; O(1) both ends.                                          |

---

## Control flow & error handling

| JS / Python                  | C++                                               | Notes                                                          |
|------------------------------|---------------------------------------------------|----------------------------------------------------------------|
| `try / except / finally`     | `try / catch` + RAII for "finally"                | RAII destructors run regardless. (Ch.7)                          |
| `raise SomeError`            | `throw SomeError{}`                               | Cost is large *when thrown*; zero when not.                       |
| `return None` on failure     | `return std::nullopt;` or `std::unexpected(err);` | Sum-type return; caller must handle. (Ch.7)                       |
| `with open(...) as f:`       | `File f(...);` (RAII)                              | Cleanup is implicit at scope exit.                                |
| Async/await                  | `std::future`, coroutines (C++20)                  | More machinery; we used callbacks instead in Kestrel. (Ch.15)     |

---

## Functions & callables

| JS / Python                  | C++                                            | Notes                                              |
|------------------------------|------------------------------------------------|----------------------------------------------------|
| `function` / `def`           | `T func(args)`                                  | Plain function.                                    |
| Arrow function / lambda      | `[](T x) { return ...; }`                       | Captures with `[=]` (by copy) or `[&]` (by ref).   |
| Closure capturing variables  | `[name](){ ... }`                               | Explicit capture list; no surprises about scope.   |
| Pass a callable              | `std::function<R(Args...)>` parameter            | Allocates for non-empty captures.                  |
| Pass a callable, zero alloc  | Template `template<class F> ... f` parameter    | Compile-time, no allocation. Preferred where size  |
|                              |                                                 | of binary isn't a problem.                          |
| Method on object             | Member function                                  | `obj.method()` syntax identical.                   |

---

## Types & generics

| JS / Python                       | C++                                | Notes                                                  |
|-----------------------------------|------------------------------------|--------------------------------------------------------|
| TypeScript `T extends Foo`        | `template <T> requires Foo<T>`     | C++20 concept-constrained template.                     |
| Python `Protocol`                 | concept                            | Static duck typing; checked at compile time.            |
| `Union[A, B]` (Py) / `A \| B` (TS) | `std::variant<A, B>`              | Tagged union; `std::visit` for exhaustive dispatch.     |
| Generic class                     | `template <class T> class Foo;`     | Per-instantiation code generation.                      |
| `instanceof` / `isinstance`       | `dynamic_cast<T*>(p)` or variant   | Variant: `std::holds_alternative<T>(v)`.                |

---

## Concurrency

| JS / Python                       | C++                                | Notes                                                 |
|-----------------------------------|------------------------------------|-------------------------------------------------------|
| Event loop                         | `std::condition_variable` + queue or hand-rolled loop | Built-in event loop; we built our own.    |
| `setTimeout(fn, ms)`              | (your event loop's timer API)      | Not built into the language; you wire it up.            |
| `Worker` / `multiprocessing`       | `std::jthread`                     | Shared-memory thread; no isolation.                     |
| `Promise` / `Future`              | `std::future`                       | Similar shape; `std::async` to launch.                 |
| `async/await`                     | C++20 coroutines (advanced)         | Out of scope for this book.                             |
| `Lock`                            | `std::mutex` + `std::lock_guard`    | Always pair with RAII.                                  |

---

## Build & dependencies

| JS / Python                  | C++                                    | Notes                                                       |
|------------------------------|----------------------------------------|-------------------------------------------------------------|
| `npm install` / `pip install`| (none — choose one of several)         | See Chapter 8.                                              |
| `package.json`               | `CMakeLists.txt`                       | Different problem statement; CMake is also a build system.  |
| `package-lock.json`          | `GIT_TAG` pin in `FetchContent`        | Less granular; no automatic transitive lock.                 |
| `node_modules`               | `_deps/` (FetchContent) or system pkgs | Less common to ship dependencies in-tree.                   |
| `import x from 'foo'`        | `#include "foo/header.hpp"`            | Header is the file; no module loader.                       |
| `from foo import bar`        | (with `using` from a header)           | Namespaces serve a similar grouping role.                   |

---

## Tooling

| JS / Python                  | C++                                | Notes                                       |
|------------------------------|------------------------------------|---------------------------------------------|
| `console.log` / `print`      | `std::println` / `std::cout`       | Same idea; flush is your responsibility.    |
| `node --inspect` / `pdb`     | `lldb` / `gdb`                     | More demanding but more capable.            |
| `eslint` / `pylint`           | `clang-tidy`                       | Both are static analyzers.                  |
| `prettier` / `black`         | `clang-format`                     | All format opinionated.                     |
| `jest` / `pytest`            | `doctest` / `Catch2` / `GoogleTest`| Per-binary tests are typical in C++.        |
| `Jest snapshot`              | `CHECK(... == "expected output")`  | No built-in snapshotting; do it by hand.    |
| (no analog)                  | sanitizers (ASan/UBSan/TSan)       | The big win C++ has via instrumentation.    |

---

## Things JS/Python have that C++ doesn't (yet)

- **A built-in package manager.** Pick one — vcpkg, Conan, or
  `FetchContent` — per project. See Chapter 8.
- **A module system that resolves at run time.** C++20 added *modules*
  (the `import std;` form), but tooling support is still patchy.
  Kestrel uses headers.
- **Reflection.** C++ has no `__dict__` / `Object.keys`. Compile-time
  reflection is a C++26 proposal; today, the closest is hand-written
  `std::variant`/`std::visit` patterns.
- **A blessed test framework.** Pick one and stick with it.
- **A canonical async runtime.** C++ has coroutines but no runtime;
  reactor patterns are the answer for now.

## Things C++ has that JS/Python don't

- **Deterministic destruction.** The closing brace runs your cleanup
  now, not "sometime later."
- **Value semantics + move semantics.** Copying is the default;
  transferring ownership is a single move.
- **Zero-cost abstractions.** Templates resolve at compile time;
  the binary contains no overhead for the abstraction layer you wrote.
- **`std::span`, `std::string_view`, `std::variant`, `std::expected`.**
  Composable type-system tools without runtime cost.
- **The memory model with explicit ordering.** Once you understand it,
  it gives you a precision other languages cannot.

---

## See also

- Chapter 1 — Why C++ Feels Alien. The four mental-model shifts that
  back this glossary.
- Every "Coming from JS/Python" callout in the book. Each is a worked
  contrast in context.
