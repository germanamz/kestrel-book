# Error Handling

The parser from Chapter 6 has a known weakness, deliberately left in. Its
return type — `std::optional<Message>` — collapses two completely different
outcomes into the same `nullopt`: *"I need more bytes; come back when you have
some"* and *"this byte stream is malformed and will never become a valid RESP
message."* A real socket loop must distinguish them — one means "wait," the
other means "drop the connection and log." Conflating them is the kind of bug
that ships.

This chapter is about error handling in modern C++: the choice between
**exceptions** and **error values**, what `noexcept` means and when to use it,
how RAII keeps cleanup correct on failure paths, and — the headline feature —
**`std::expected<T, E>`** from C++23, the type-safe sum that finally gives C++
a clean way to return "either a value or a typed error" without smuggling
either past the type system.

By the end, every fallible function in Kestrel will say so in its signature.
The parser will return `Expected<ParseResult, ProtocolError>` and the caller
will be unable to ignore the failure case without the compiler noticing. This
is the discipline that makes systems C++ safe — not catching all errors, but
*forcing* every caller to confront each one.

---

## The C++ error-handling landscape

Three mechanisms coexist in C++ for reporting that an operation failed, and
choosing between them is a real design decision per layer of the code.

**Exceptions** (`throw`/`catch`) propagate up the call stack until something
catches them. They are the right tool for *truly exceptional* conditions —
out-of-memory, programmer errors, contract violations — that you don't expect
on the normal control-flow path. They are also free *while not thrown* on
mainstream platforms (zero-cost exceptions), and very expensive *when* they
are thrown.

**Error codes** (returning a `bool`, an `int`, or an `errno`-like enum) are
the C-style answer. Cheap, predictable, but easy to forget: the compiler is
happy to let you ignore a return value. The Linux kernel uses them; modern
C++ application code does not.

**Error values via sum types** (`std::optional`, `std::expected`,
hand-written `Result`) put the error *into the type system* — the return
type is "either a value or an error," and the caller cannot read the value
without first acknowledging the error case. C++23's `std::expected` is the
mainstream form. This is where Kestrel will live for most fallible code.

The right division of labor in modern C++ is roughly:

- **Use `std::expected`** for predictable, recoverable failures on a function's
  normal control-flow path: a parse failure, a "key not found," a `write()`
  that returned `EAGAIN`. The caller is expected to handle this; the type
  forces them to.
- **Use exceptions** for genuinely exceptional conditions you cannot reasonably
  handle locally: out-of-memory (`std::bad_alloc`), programmer errors caught by
  assertions, deep configuration mistakes. Throwing is the right tool when the
  honest answer is "I cannot continue and I don't know what *can*."
- **Avoid raw error codes** in the application layer. They are fine in the
  thin C-API boundary you cannot avoid (the POSIX wrappers in Chapter 14
  return `errno`-style values), but every layer above it should convert them
  to `expected` or to typed exceptions.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

You are used to exceptions being the default for everything — "throw on
failure, catch at the top." C++ rejects that as a *general* rule. The reason
isn't ideology, it's cost and clarity: a recoverable, expected failure
(parser needs more bytes; `open()` returned `ENOENT`) is not exceptional, and
treating it as such makes the control flow harder to read and the
optimizations harder. JavaScript's `Result<T, E>`-style libraries (neverthrow,
fp-ts) exist because experienced JS engineers reach this same conclusion;
C++23 just made the type built-in.

</div>

---

## `std::expected`: a value or a typed error

`std::expected<T, E>`, in C++23, is a type that holds *either* a `T` (success)
or an `E` (failure). The default-constructed state is the success case; you
write `std::unexpected(e)` to construct the failure case. The API mirrors
`std::optional`:

```cpp
#include <expected>

std::expected<int, std::string> parse_positive_int(std::string_view s) {
    if (s.empty()) return std::unexpected("empty input");
    int v;
    auto [p, ec] = std::from_chars(s.data(), s.data() + s.size(), v);
    if (ec != std::errc{} || p != s.data() + s.size())
        return std::unexpected("not an integer");
    if (v <= 0) return std::unexpected("must be positive");
    return v;
}

auto r = parse_positive_int("42");
if (r) {                              // contextually-bool: true iff success
    std::printf("got %d\n", *r);      // operator* yields the value
} else {
    std::printf("err: %s\n", r.error().c_str());
}
```

The pieces:

- `r.has_value()` / `(bool)r` — did it succeed?
- `*r` / `r.value()` — the successful value; `r.value()` throws
  `std::bad_expected_access` if `r` is the error case (parallels
  `optional::value()`).
- `r.error()` — the error value when `!r`.
- `r.value_or(fallback)` — value if present, else fallback. Common shorthand.

There is also a `.transform(...)` / `.and_then(...)` family for chaining,
just like in Rust's `Result`. We will use `and_then` once or twice in Kestrel
to keep parser composition tidy, but the explicit `if (r) { ... } else { ...
}` form is fine and more readable for short chains.

### A `ProtocolError` for Kestrel

The error type does not have to be a string. For Kestrel's parser, a small
struct with a code and a description is more useful — code lets the caller
branch, description lets the operator read the log:

```cpp
// src/protocol/error.hpp
#pragma once

#include <string_view>

namespace kestrel::protocol {

enum class ErrorCode {
    UnknownType,        // first byte not in +,-,:,$,*
    BadLength,          // non-numeric length, or negative length other than -1
    BadCRLF,            // expected \r\n at a known position, didn't find it
    OversizedLength,    // length larger than a configured limit
};

struct ProtocolError {
    ErrorCode        code;
    std::string_view what;        // static description; lifetime: string literal
};

constexpr ProtocolError unknown_type{ ErrorCode::UnknownType, "unknown RESP type marker" };
constexpr ProtocolError bad_length  { ErrorCode::BadLength,   "malformed length prefix"  };
constexpr ProtocolError bad_crlf    { ErrorCode::BadCRLF,     "expected CRLF"            };

}  // namespace kestrel::protocol
```

A handful of `constexpr` constants instead of constructing fresh
`ProtocolError`s in-line, because the descriptions are static — using
`string_view` over a string literal is safe forever, and `constexpr` instances
cost nothing.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

Notice the error type is `ProtocolError`, not `Error`. C++ does not have a
single root error class everything throws or returns; you design the error
domain per module. The cost is that error types don't unify; the benefit is
that the type at each layer says exactly what *kinds* of failures can occur
there, with no fall-through. Chapter 13 will define a `PersistenceError`;
Chapter 14 a `NetworkError`. They never collide because they don't share a
base.

</div>

---

## Refactoring the parser

Chapter 6's parser confused "incomplete" with "broken." With `expected` we
can fix that — but we still need a third state, *incomplete*. Two clean
shapes are commonly used:

1. `std::expected<std::optional<Message>, ProtocolError>` — outer says
   "definitely-error or not," inner says "have value or need more bytes."
2. A custom struct/enum: `struct ParseResult { enum { Ok, NeedMore, Error };
   ... };`

Option 1 is short and composes with `expected`'s methods; option 2 reads more
plainly. Kestrel will use option 1 because composability matters as the
parser grows. The signature becomes:

```cpp
// src/protocol/parse.hpp
#pragma once

#include <expected>
#include <optional>
#include <string_view>

#include "protocol/error.hpp"
#include "protocol/message.hpp"

namespace kestrel::protocol {

// Returns:
//   expected with optional(Message) -> a complete message, `consumed` set.
//   expected with nullopt           -> need more bytes, `consumed` = 0.
//   unexpected(ProtocolError)       -> malformed; the caller should drop the
//                                      connection.
using ParseResult = std::expected<std::optional<Message>, ProtocolError>;

ParseResult parse(std::string_view input, std::size_t& consumed);

}  // namespace kestrel::protocol
```

Internally, the helpers from Chapter 6 change from returning
`std::optional<Parsed>` to returning a similar three-state value. The
mechanical refactor is:

```cpp
// Before:
if (!parse_int(body, v)) return std::nullopt;

// After:
if (!parse_int(body, v)) return std::unexpected(bad_length);

// Before:
if (in.empty()) return std::nullopt;
// After (still nullopt — empty input is "need more"):
if (in.empty()) return std::optional<Parsed>{};   // need more
```

The two `nullopt`s of Chapter 6 split into two different return paths now: a
malformed input returns `std::unexpected(...)`; an incomplete input returns
the "ok, but no value yet" branch. The compiler will refuse to ignore either.

Callers look like:

```cpp
std::size_t consumed = 0;
auto r = parse(input, consumed);
if (!r) {
    log("protocol error: %s", r.error().what.data());
    drop_connection();
} else if (!r->has_value()) {
    // need more bytes; remain in the read loop
} else {
    dispatch(std::move(**r));
    advance_read_window(consumed);
}
```

That branching is the *point*. Each of the three outcomes has its own
behaviour, and the type system insists you have one for each.

<div class="callout callout-experiment">

🧪 **Experiment**

Wire your refactored parser into the Chapter 6 tests. Add a third case to the
tests: feed it `?garbage\r\n` (unknown type marker) and confirm `r` is in the
unexpected branch with `r.error().code == ErrorCode::UnknownType`. Then make
the same call but with only `?garb` and confirm — wait, what *should* happen?
The first byte is unknown, so the answer is "error, not 'need more'," even
though the input is short. Decide before you run. Most real parsers report
the error at the first byte; the alternative ("might be unknown after more
bytes") is almost never useful.

</div>

---

## `noexcept`: when a function won't throw

A second piece of error-handling vocabulary that pays off as Kestrel grows.
`noexcept` is a function specifier that *promises* the function will not
throw exceptions. The compiler does not check the promise — if a `noexcept`
function does throw, `std::terminate()` is called immediately, no
unwinding. Treat `noexcept` as a contract: "I have audited this and it
genuinely cannot escape an exception."

There are three reasons to write `noexcept`:

1. **Move operations should be `noexcept`** (Chapter 3 already touched this).
   `std::vector` only moves elements during reallocation if the move
   constructor is `noexcept`; otherwise it copies. Forgetting `noexcept` on
   moves silently halves your performance.
2. **Destructors are implicitly `noexcept`**. A destructor that throws is one
   of C++'s most dangerous corners — it can cause `std::terminate` during
   stack unwinding from another exception. The implicit `noexcept` is the
   language saving you from this. Don't fight it.
3. **APIs that callers want to call without try/catch** — small leaf functions
   (an accessor, a getter, a pointer-swap) — should be `noexcept` when they
   really can't throw. Code at higher layers can call them inside their own
   `noexcept` functions without the compiler producing exception-handling
   tables.

`noexcept` is not a free upgrade — adding it to a function you're not sure
about turns a potential `throw` into an unconditional terminate. Don't
sprinkle. Add it deliberately, with a comment if needed.

```cpp
// src/core/buffer.hpp  (illustrative)
class Buffer {
public:
    Buffer() noexcept = default;
    std::size_t size() const noexcept { return size_; }
    std::span<std::uint8_t> bytes() noexcept { return {data_.get(), size_}; }
    // resize() / reserve() can throw bad_alloc; they are NOT noexcept.
};
```

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

A function that says `noexcept` and *does* throw calls `std::terminate`
without unwinding the stack. Your RAII destructors do not run; sockets do not
close; files do not flush. This is one of the few places in C++ where the
penalty for a lie is total program death. The redeeming feature is that it
is loud — you get a clear `terminate` message, not the slow corruption of UB.

</div>

---

## RAII on the failure path

The full payoff of RAII (Chapter 1, Chapter 3) is felt here. When a function
throws — `bad_alloc`, anything — the stack unwinds, and every local object's
destructor runs in reverse order of construction. Your `Buffer`s release
their memory. Your future `FileDescriptor` wrapper closes the fd. Your
`Mutex` lock releases the lock. *You write no cleanup code on the throw
path* — because the destructors are the cleanup.

This is what gives C++ exception-safe code its peculiar shape. The pattern is
not `try/finally`; it is "put the resource in a class whose destructor
releases it." Compare:

```python
# Python: explicit cleanup with `with`
def write_log(record):
    with open("kestrel.wal", "ab") as f:
        f.write(record)
    # close() runs whether write() threw or not
```

```cpp
// C++: implicit cleanup via RAII
void write_log(std::span<const std::uint8_t> record) {
    File f("kestrel.wal", File::Append);   // RAII wrapper, closes on destruction
    f.write(record);                        // throws? f's destructor still runs
}
```

There is no `try/catch`, no `finally`, no `with`. The cleanup is the type's
business; the function reads as if nothing can go wrong. This is why
Chapter 14's first job will be a `FileDescriptor` RAII wrapper around POSIX
file descriptors — once that exists, every later piece of code that touches
an fd is exception-safe by construction.

<div class="callout callout-experiment">

🧪 **Experiment**

Demonstrate RAII-on-throw by yourself. Write a tiny class whose constructor
takes a label and whose destructor prints `~label`, then call a function that
constructs two of them and throws between them. Predict the output, then run
it: you should see `~second\n~first\n`, in reverse construction order, before
the exception escapes the function. That is the only "exception handling
code" you have to write for the resources to be released — none.

</div>

---

## When to throw, when to return

A practical rule for Kestrel — and for most modern C++ — settles which
mechanism each fallible operation uses.

**Return `expected` when:**

- The failure is part of the function's contract (parser, lookup,
  `try_parse_int`, "does this key exist?").
- The caller is expected to handle the failure (most failures in a server's
  request-handling path).
- The performance of the *failure* path matters as much as the success path.

**Throw an exception when:**

- The failure is *not* something normal callers should be coping with
  (allocation failure, configuration mistakes, programmer errors).
- Locally there is nothing meaningful to do but propagate up many frames.
- The error involves invariants whose violation means the program cannot
  reliably continue.

A function that violates its own preconditions — index out of bounds, null
where non-null was promised — should not throw a recoverable exception; it
should `assert` (debug) or `std::abort` (release) or, in C++26, contract-check.
Those are *bugs*, not errors. Throwing for them invites callers to "handle"
broken state, which is worse than crashing.

Kestrel's split:

- Parser, store lookup, key-found-or-not, socket EAGAIN, file ENOENT →
  `std::expected`.
- Out of memory, file open during startup that *must* succeed, a malformed
  command bytes the dispatch layer thinks is impossible → throw.
- Internal invariants ("we should never reach here") → `assert` and a comment
  explaining the invariant.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

This is the rule of thumb you have absorbed in JS/Python without naming:
`raise` for genuinely exceptional things; return a falsy value or a
`{ok: false, err}` shape for the everyday "didn't work." C++ formalizes that
intuition with the type system. The discipline that has to grow is *not*
reaching for exceptions when `expected` is the better answer.

</div>

---

## Where this leaves Kestrel

The parser is the first Kestrel subsystem to fully participate in the
error-handling design. Every later subsystem will follow the same shape:
`expected<T, ErrorType>` for predictable failures, RAII destructors for
cleanup, exceptions only for genuinely exceptional cases, `noexcept` on the
hot path where it is truthful.

Chapter 8 is the only chapter in the book that introduces an external
library — a test framework — and it does so on purpose, to teach you C++
dependency management before Kestrel needs hash tables, persistence, or
networking. After Chapter 8, the test harness we hand-rolled in Chapter 2
gives way to doctest or Catch2, and `version_test`, `value_test`,
`protocol_test` all get rewritten as cases in the new framework. The
hand-rolled harness was never going to be Kestrel's permanent answer; it was
your introduction to the fact that a test is just a program with an exit
code.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's my Chapter 7 refactor of the parser to return
> `expected<optional<Message>, ProtocolError>`. Walk me through whether the
> three-state shape is the right design. Are there cases where I should
> throw instead of returning `unexpected`? Are there any `noexcept` I should
> add to `Buffer`, `String`, or `Value` that I haven't? Show me one place in
> the parser where I can use `.and_then` to compose two fallible steps more
> cleanly.

</div>

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and test cleanly under ASan/UBSan:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

You should be able to:

- State the three error-handling mechanisms in C++ and when each is
  appropriate.
- Read a `std::expected<T, E>` and explain how the caller is forced to
  handle the error case.
- Describe what `noexcept` means at the language level and the three reasons
  to use it.
- Argue why a destructor that throws is dangerous and what C++ does when one
  does throw.
- Distinguish "need more bytes" from "malformed bytes" in the refactored
  parser, in terms of the return type.

Hand it to Claude:

> Review my Chapter 7 changes: `src/protocol/parse.{hpp,cpp}`,
> `src/protocol/error.hpp`. Is `expected<optional<T>, E>` idiomatic, or
> would a custom `ParseResult` be clearer? Are my error categories well
> chosen? Did I miss `noexcept` annotations on functions that genuinely
> cannot throw?

When that's clean, Chapter 8 introduces C++ dependency management and
replaces the hand-rolled harness with a real test framework.

</div>
