# The RESP Protocol & Parsing

Kestrel's data model is settled. Now it has to talk. Real Redis clients send
commands as bytes on a TCP socket, and Redis sends bytes back; for `kvcli` to
be wire-compatible with `redis-cli`, Kestrel has to speak the same language as
the byte stream they exchange. That language is **RESP** — the REdis
Serialization Protocol — and it is small enough to implement from its
published spec in a single chapter.

This chapter is half protocol, half C++. Protocol-wise, you will read enough
of [the RESP2 spec](https://redis.io/docs/reference/protocol-spec/) to
implement parsing and serialization for the five wire types Kestrel needs.
C++-wise, the chapter introduces three tools the rest of the book leans on:
**`std::string_view`** for zero-copy text handling, the discipline of
**incremental parsing** (because TCP delivers bytes in arbitrary chunks), and
a templated **serializer** that visits the `Value` variant from Chapter 5.

By the end, `src/protocol/` will contain a parser that turns bytes into
typed commands and a serializer that turns Kestrel responses back into bytes,
and you will be ready for Chapter 7 to formalize what happens when either of
them fails.

---

## RESP in fifteen minutes

RESP is a textual, line-oriented protocol with a single binary type bolted on.
Every value starts with a one-byte type marker, ends with `\r\n` (the CRLF
line terminator), and is unambiguous to parse left-to-right. There are five
types in RESP2:

| Marker | Type           | Wire example                | Notes                                            |
|--------|----------------|-----------------------------|--------------------------------------------------|
| `+`    | Simple String  | `+OK\r\n`                   | Short, no CR/LF inside.                          |
| `-`    | Error          | `-ERR unknown command\r\n`  | Same shape as Simple String; semantic difference.|
| `:`    | Integer        | `:1000\r\n`                 | 64-bit signed.                                   |
| `$`    | Bulk String    | `$6\r\nfoobar\r\n`          | Length-prefixed binary; may contain any byte.    |
| `*`    | Array          | `*2\r\n+OK\r\n:42\r\n`      | Length-prefixed sequence of any RESP value.      |

Two facts about the wire matter immediately.

**Commands from the client are always arrays of bulk strings.** A `SET key
value` command from `redis-cli` arrives on the wire as
`*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n`. Whatever else RESP can
represent, *requests* always have that shape: array-of-bulks. This lets us
write the command-parsing path narrowly while still implementing the full
type set for replies.

**Bulk strings carry their length up front.** A bulk string's length is the
ASCII decimal between `$` and `\r\n`; the payload is exactly that many bytes,
followed by another `\r\n`. The payload is *not* null-terminated and *may*
contain any byte, including `\r`, `\n`, and `\0`. This is exactly why
`String` in Chapter 5 carries explicit length and stores raw bytes — a
null-terminated `char*` cannot represent every bulk string.

There is also a special **null bulk string** (`$-1\r\n`) and a **null array**
(`*-1\r\n`), each of which is what Redis returns for "key doesn't exist." We
support them by treating the length `-1` as a sentinel.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

If you have written an HTTP parser, this is far simpler — no headers, no
chunked encoding, no transfer-encoding negotiation. The closest analog in
shape is JSON-but-binary-and-length-prefixed. The C++-specific challenge is
not the protocol but how you parse it *incrementally* over a socket whose
reads chop messages in half. That incremental discipline is what the next
section is about.

</div>

---

## Bytes in, bytes out — the abstractions we need

Parsing on a real connection is not "give me a buffer, return a value." It is
"here are the bytes I have so far; tell me whether they spell a complete value
yet, and if so, what it is." A parser that can be called repeatedly as more
bytes arrive — and that does not lose work between calls — is an **incremental
parser**. Kestrel writes one because real TCP traffic doesn't respect message
boundaries: a `SET` command may arrive split between two reads, or two
commands may arrive in one read.

The state our parser needs to keep is small:

- The byte buffer it is parsing from (a `std::span<const std::uint8_t>`).
- A cursor — the position it has consumed up to.
- A signal back to the caller: *parsed value*, *need more bytes*, or *error*.

For now we model that signal as a `std::optional<Value>` — `nullopt` means
"need more." Chapter 7 will refine this to `std::expected<Value, ProtocolError>`
to distinguish "need more bytes" from "definitely malformed." It pays to
start simple.

We do *not* want the parser to take ownership of the bytes or copy them; that
buffer belongs to the socket layer, and we will look into it through a
`std::string_view` or `std::span` and produce values that may keep
`string_view`s referring back into the same buffer when zero-copy is safe.

### `std::string_view`: bytes you do not own

A `std::string_view` is to text what `std::span` was to bytes in Chapter 4: a
non-owning, fixed-size handle (pointer + length) over a contiguous sequence
of characters that lives elsewhere. It is dirt cheap to copy, comparable with
`==`, and indexable with `[]`, and it is the right type for "show me the
characters; I don't need to keep them."

The temptation to make `string_view` your only string type is real and
*wrong*. A `string_view` does not own; if the underlying storage moves or
dies, the view dangles, and you have UB. As a rough rule:

- **Function *parameter* that does not need to store the string** →
  `std::string_view`. Cheap, accepts anything.
- **Field of a class** → either `std::string` (owns) or be very sure about
  the lifetime relationship if you store a view.

For the parser we will receive bytes as `std::string_view`, return tokens
inside the same buffer as `std::string_view`, and only construct an owning
`String` (Chapter 5's type) when the value is about to outlive the buffer —
e.g. when storing it in the keyspace.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`std::string_view` over a `std::string` becomes dangling if the `string`
reallocates (e.g. you `push_back` past its capacity). The classic bug:
return a `string_view` of a temporary `std::string`. The temporary dies at
the semicolon; your view is freed memory before the caller reads it. ASan
catches this *if* the test runs the dangling path. Never return a
`string_view` of a local string.

</div>

---

## The parser, sketched

A header sketches the protocol types and the parser entry point. We are
parsing into a small variant — *not* Chapter 5's `Value`, because the wire
form is one-step removed from the stored form (a wire bulk-string becomes a
`String` value, but a wire array becomes an outer container the *commands*
are dispatched from). So `protocol/` gets its own message type:

```cpp
// src/protocol/message.hpp
#pragma once

#include <cstdint>
#include <string_view>
#include <variant>
#include <vector>

namespace kestrel::protocol {

struct SimpleString { std::string_view text; };
struct Error        { std::string_view text; };
struct Integer      { std::int64_t value; };
struct BulkString   { std::string_view bytes; bool is_null = false; };

struct Array;
using Message = std::variant<SimpleString, Error, Integer, BulkString, Array>;

struct Array {
    std::vector<Message> items;
    bool is_null = false;
};

}  // namespace kestrel::protocol
```

The variant lives at protocol-layer scope; the views point into the parse
buffer. The parser is a free function:

```cpp
// src/protocol/parse.hpp
#pragma once

#include <optional>
#include <string_view>

#include "protocol/message.hpp"

namespace kestrel::protocol {

// Parse one RESP value from `input` starting at byte 0.
// On success, returns the parsed Message and sets `consumed` to the number of
// bytes used. On "need more data," returns nullopt and leaves `consumed` at 0.
std::optional<Message> parse(std::string_view input, std::size_t& consumed);

}  // namespace kestrel::protocol
```

The `consumed` out-parameter is how the socket layer (Chapter 16) advances
its read cursor: parse, advance by `consumed`, parse again. Returning
`nullopt` with `consumed == 0` is the explicit "wait for more bytes" answer.

### Implementing the line-eater

The heart of every RESP parse is "find the next CRLF." Once we have that,
each type's parse is short:

```cpp
// src/protocol/parse.cpp
#include "protocol/parse.hpp"

#include <charconv>
#include <cstring>

namespace kestrel::protocol {

namespace {

// Locate the next "\r\n" at or after `pos`. Returns the index of the '\r',
// or std::string_view::npos if not present.
std::size_t find_crlf(std::string_view s, std::size_t pos) {
    for (std::size_t i = pos; i + 1 < s.size(); ++i) {
        if (s[i] == '\r' && s[i + 1] == '\n') return i;
    }
    return std::string_view::npos;
}

// Parse a base-10 integer from a string_view. Returns true on success.
bool parse_int(std::string_view s, std::int64_t& out) {
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), out);
    return ec == std::errc{} && ptr == s.data() + s.size();
}

}  // namespace

}  // namespace kestrel::protocol
```

Two small modern-C++ moves earn their place.

**`std::from_chars`** is the cheapest, exception-free way to parse a number
from a `string_view`. Unlike `std::stoi` or `std::stoll`, it does not allocate
and does not throw — it returns a result struct with an error code. For
parsing wire protocols at scale this matters; you do not want to throw
exceptions for every malformed length. We will see `std::from_chars`'s sibling
`std::to_chars` in the serializer.

**An anonymous namespace** (`namespace { ... }`) at the top of a `.cpp` makes
its contents private to this translation unit. It is the C++ replacement for
`static`-on-free-function and a vital tool against polluting the linker with
internal helpers — One Definition Rule again, in friendlier form. Use it for
every helper not declared in a header.

### Recursive descent over RESP

The top-level parse dispatches on the first byte and falls into per-type
helpers. Each helper either returns the parsed value and the number of bytes
it consumed, or returns `nullopt` for "incomplete":

```cpp
namespace {

struct Parsed { Message msg; std::size_t consumed; };

std::optional<Parsed> parse_one(std::string_view in);

std::optional<Parsed> parse_simple(std::string_view in, char marker) {
    // +OK\r\n  or  -ERR ...\r\n  or  :123\r\n
    auto crlf = find_crlf(in, 1);
    if (crlf == std::string_view::npos) return std::nullopt;
    std::string_view body = in.substr(1, crlf - 1);

    Message m;
    if      (marker == '+') m = SimpleString{body};
    else if (marker == '-') m = Error{body};
    else /* ':' */ {
        std::int64_t v;
        if (!parse_int(body, v)) return std::nullopt;   // (placeholder for "error")
        m = Integer{v};
    }
    return Parsed{m, crlf + 2};
}

std::optional<Parsed> parse_bulk(std::string_view in) {
    // $<len>\r\n<bytes>\r\n   or   $-1\r\n
    auto crlf = find_crlf(in, 1);
    if (crlf == std::string_view::npos) return std::nullopt;
    std::int64_t len;
    if (!parse_int(in.substr(1, crlf - 1), len)) return std::nullopt;

    if (len == -1) return Parsed{ BulkString{{}, /*is_null=*/true}, crlf + 2 };

    std::size_t start = crlf + 2;
    std::size_t end   = start + static_cast<std::size_t>(len);
    if (in.size() < end + 2) return std::nullopt;          // not all bytes here yet
    if (in[end] != '\r' || in[end + 1] != '\n') return std::nullopt;

    return Parsed{ BulkString{in.substr(start, len), false}, end + 2 };
}

std::optional<Parsed> parse_array(std::string_view in) {
    auto crlf = find_crlf(in, 1);
    if (crlf == std::string_view::npos) return std::nullopt;
    std::int64_t len;
    if (!parse_int(in.substr(1, crlf - 1), len)) return std::nullopt;

    std::size_t consumed = crlf + 2;
    if (len == -1) return Parsed{ Array{{}, /*is_null=*/true}, consumed };

    Array arr;
    arr.items.reserve(static_cast<std::size_t>(len));
    for (std::int64_t i = 0; i < len; ++i) {
        auto p = parse_one(in.substr(consumed));
        if (!p) return std::nullopt;
        arr.items.push_back(std::move(p->msg));
        consumed += p->consumed;
    }
    return Parsed{ std::move(arr), consumed };
}

std::optional<Parsed> parse_one(std::string_view in) {
    if (in.empty()) return std::nullopt;
    switch (in[0]) {
        case '+': case '-': case ':': return parse_simple(in, in[0]);
        case '$':                     return parse_bulk(in);
        case '*':                     return parse_array(in);
        default:                      return std::nullopt;   // protocol error
    }
}

}  // namespace

std::optional<Message> parse(std::string_view input, std::size_t& consumed) {
    consumed = 0;
    auto p = parse_one(input);
    if (!p) return std::nullopt;
    consumed = p->consumed;
    return std::move(p->msg);
}
```

Walk it once with the eye on the C++.

**No allocations on the parse path except the array.** Bulk strings are
`string_view`s into the caller's buffer — zero-copy. Simple strings, errors,
integers — all views. The only allocation is the `std::vector` an `Array`
needs for its items (and the small `Array` move when it lands in the variant).
On a hot wire that processes commands at line rate, this is the difference
between gigabit and ten-gigabit throughput.

**The `consumed` discipline.** Every helper returns *exactly* how many bytes
it used. The top-level parser propagates that back so the socket layer can
slide its read window. Bugs in this number are some of the worst protocol
bugs — a parser that under-reports `consumed` re-parses bytes and produces
duplicates; one that over-reports skips the next message's header. Test it
hard.

**Right-pre-allocation.** `arr.items.reserve(len)` before pushing is the
non-obvious perf habit. A growable vector doubles on growth; reserving the
exact known size avoids those reallocations and the moves they trigger.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

Returning a "parsed value or need-more-bytes" via `std::optional` is C++
idiom; in Node you might throw `IncompleteFrameError`, in Python you might
raise `EOFError`. C++ avoids exceptions on the *expected* path (incomplete
data is normal, not exceptional) and reserves them for genuinely abnormal
conditions. Chapter 7's `std::expected` will distinguish "not done yet" from
"malformed forever" — both legitimate cases that a single `optional` blurs.

</div>

<div class="callout callout-experiment">

🧪 **Experiment**

Drive the parser with a deliberately torn input: feed it the first 10 bytes of
`*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n`, then progressively reveal
more, calling `parse` after each step. Until you have all 32 bytes, every
call should return `nullopt` with `consumed == 0`. The moment the full frame
is present, you get the parsed `Array`. That is the contract a real socket
loop will rely on. If your parser ever reports `consumed > 0` *and* returns
`nullopt`, you have a bug.

</div>

---

## Endianness and binary integers

RESP is text — numbers go on the wire as ASCII decimal. Kestrel will *not*
spend this chapter on `htonl`/`ntohs`. We will, however, hit binary
serialization in Chapter 12 when the WAL writes 64-bit lengths and CRCs, and
this is the right place to plant the flag.

C++ does not normalize byte order. The standard says "an integer is some
number of bytes laid out in memory in implementation-defined order." On every
mainstream platform we target — x86-64, ARM64 on macOS — that order is
little-endian, but on a wire you control, the rule is **pick one byte order
and write the conversion explicitly**. The portable, modern way is C++23's
`std::byteswap` plus `if constexpr (std::endian::native == std::endian::big)`:

```cpp
#include <bit>
#include <cstdint>

std::uint32_t to_be32(std::uint32_t x) {
    if constexpr (std::endian::native == std::endian::big) return x;
    else return std::byteswap(x);
}
```

We will come back to this in Chapter 12. For now, the takeaway is: bytes on
the wire have a defined order, your CPU has its own order, and the conversion
is a one-line function you must call deliberately. Memcpying an `int` into a
network frame is the single most common UB-style protocol bug.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`reinterpret_cast<std::uint32_t*>(buf)` on a network buffer is doubly wrong:
the storage is not a `uint32_t` (so the access is UB) *and* the byte order may
not be what you think. The safe form is `std::uint32_t x; std::memcpy(&x, buf,
4); x = from_be32(x);`. The compiler routinely turns those three lines into
the same single CPU instruction your hand-cast would have been, with no UB.

</div>

---

## Serializing the other way

Writing RESP out is straightforward — it is the parser running backwards. The
serializer takes a `Value` (or a smaller wire-side type) and an output
`Buffer`, and appends bytes:

```cpp
// src/protocol/serialize.hpp
#pragma once

#include <cstdint>
#include <string_view>

#include "core/buffer.hpp"
#include "core/string.hpp"

namespace kestrel::protocol {

void write_simple(Buffer& out, std::string_view text);
void write_error(Buffer& out, std::string_view text);
void write_integer(Buffer& out, std::int64_t v);
void write_bulk(Buffer& out, std::span<const std::uint8_t> bytes);
void write_null_bulk(Buffer& out);

}  // namespace kestrel::protocol
```

A representative implementation, using `std::to_chars` for integer formatting
(the symmetric counterpart to `std::from_chars`: no allocation, no
exceptions, no locale):

```cpp
// src/protocol/serialize.cpp
#include "protocol/serialize.hpp"
#include <charconv>
#include <cstring>

namespace kestrel::protocol {

namespace {

void append(Buffer& out, std::string_view s) {
    const std::size_t old = out.size();
    out.resize(old + s.size());
    std::memcpy(out.data() + old, s.data(), s.size());
}

void append(Buffer& out, std::span<const std::uint8_t> b) {
    const std::size_t old = out.size();
    out.resize(old + b.size());
    std::memcpy(out.data() + old, b.data(), b.size());
}

void append_int(Buffer& out, std::int64_t v) {
    char buf[24];
    auto [ptr, ec] = std::to_chars(buf, buf + sizeof buf, v);
    append(out, std::string_view(buf, ptr - buf));
}

}  // namespace

void write_simple(Buffer& out, std::string_view text) {
    append(out, "+"); append(out, text); append(out, "\r\n");
}

void write_integer(Buffer& out, std::int64_t v) {
    append(out, ":"); append_int(out, v); append(out, "\r\n");
}

void write_bulk(Buffer& out, std::span<const std::uint8_t> bytes) {
    append(out, "$");
    append_int(out, static_cast<std::int64_t>(bytes.size()));
    append(out, "\r\n");
    append(out, bytes);
    append(out, "\r\n");
}

// write_error and write_null_bulk follow the same pattern.

}  // namespace kestrel::protocol
```

The serializer is a series of `Buffer::resize` + `memcpy` calls. Because
`Buffer::resize` grows geometrically (Chapter 4), repeated appends amortize to
linear cost. We will eventually want a *templated* `write_value(Value const&,
Buffer&)` that uses `std::visit` to dispatch over the `Value` variant; the
shape of that function is one of the experiments below.

<div class="callout callout-experiment">

🧪 **Experiment**

Write the templated dispatch:

```cpp
void write_value(Buffer& out, const Value& v) {
    std::visit([&](auto const& alt) {
        using T = std::decay_t<decltype(alt)>;
        if constexpr (std::is_same_v<T, String>) {
            write_bulk(out, alt.bytes());
        }
        // ...rest of the cases as their bodies fill in over Part 3...
    }, v);
}
```

You won't be able to compile every branch until Chapters 9–11 give bodies to
`List`, `Hash`, and `SortedSet`. For each not-yet-implemented case, use
`static_assert(sizeof(T) == 0, "TODO: serialize List")` inside the `else`
branch. Adding a new variant alternative later will break the build at the
visitor — exactly what you want.

</div>

---

## Wiring it up

Update `src/CMakeLists.txt`:

```cmake
add_executable(protocol_test
  test/protocol_test.cpp
  protocol/parse.cpp
  protocol/serialize.cpp
  core/buffer.cpp
)
add_test(NAME protocol_test COMMAND protocol_test)
```

A handful of tests, exercising the parser with full and torn inputs and the
serializer for a round-trip:

```cpp
// src/test/protocol_test.cpp
#include <string_view>
#include <variant>

#include "protocol/parse.hpp"
#include "protocol/serialize.hpp"
#include "test/check.hpp"

using namespace std::string_view_literals;

int main() {
    using namespace kestrel::protocol;

    {   // full frame
        constexpr auto cmd = "*2\r\n$3\r\nGET\r\n$3\r\nkey\r\n"sv;
        std::size_t consumed = 0;
        auto msg = parse(cmd, consumed);
        CHECK(msg.has_value());
        CHECK(consumed == cmd.size());
        CHECK(std::holds_alternative<Array>(*msg));
    }
    {   // torn frame
        constexpr auto half = "*2\r\n$3\r\nGE"sv;
        std::size_t consumed = 0;
        auto msg = parse(half, consumed);
        CHECK(!msg.has_value());
        CHECK(consumed == 0);
    }
    {   // round-trip a bulk through the serializer
        kestrel::Buffer out;
        const std::uint8_t bs[] = {'h', 'i'};
        write_bulk(out, std::span(bs));
        std::string_view wire(reinterpret_cast<const char*>(out.data()),
                              out.size());
        CHECK(wire == "$2\r\nhi\r\n");
    }

    return kestrel::test::failures == 0 ? 0 : 1;
}
```

Build under ASan/UBSan as usual and watch it pass. The build itself is the
first proof that your protocol module has clean boundaries: it depends on
`core/buffer` and `core/string`, *nothing* depends back on `protocol/`, and
no other module needs `string_view` of internal buffers.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's my `protocol/parse.cpp` and `protocol/serialize.cpp`. Review them
> for: (1) zero-copy discipline — am I copying bytes I don't have to?
> (2) lifetime safety of the `string_view`s in `Message` — can any of them
> outlive the parse buffer? (3) the `consumed` accounting — am I always
> reporting it correctly, including on incomplete reads? (4) `std::to_chars`
> / `std::from_chars` usage — am I checking the result codes properly?

</div>

---

## Where this leaves Kestrel

`src/protocol/` is the first real module of Kestrel — bytes in, typed
messages out; typed responses in, bytes out. It depends only on `core/`, has
no I/O (a parser doesn't read from a socket; the socket layer feeds it), and
is the layer Chapter 14's server will plug a socket *into* and Chapter 7 is
about to plug *error handling* into.

The chapter also introduced two C++ tools you will use without further
comment:

- **`std::string_view`** as the right type for "look at characters you don't
  own," with the warning that storing one is a lifetime contract.
- **`std::from_chars` / `std::to_chars`** as the fast, exception-free, locale-
  free way to parse and format integers — the right defaults for protocol code.

Chapter 7 formalizes what happens when a parse fails. Right now,
`parse_int(...)` failing produces a `nullopt` indistinguishable from "need
more bytes" — a bug we deliberately deferred. `std::expected<T, E>` is C++23's
sum-type-shaped answer to that, and getting it right is what Chapter 7 is
for.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

`protocol_test` must build and pass under ASan/UBSan with `-Wall -Wextra
-Wpedantic`:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

You should be able to:

- State the five RESP2 types and their wire markers.
- Explain why command parsing only ever sees arrays of bulks.
- Defend the choice of `std::string_view` over `std::string` in `Message`,
  and name one concrete way it could dangle.
- Show how the parser handles a torn frame (returns `nullopt`,
  `consumed == 0`).
- Sketch the templated `write_value` and predict what happens at compile time
  when a new `Value` alternative appears.

Then ask Claude:

> Review my Chapter 6 protocol code (`src/protocol/parse.{hpp,cpp}`,
> `src/protocol/serialize.{hpp,cpp}`, `src/test/protocol_test.cpp`). Focus on
> incremental-parse correctness, zero-copy, lifetime safety, and whether my
> module boundaries against `core/` are clean. What edge cases am I missing
> — empty bulks, negative lengths other than -1, oversized lengths?

When that's clean, Chapter 7 turns the parser's silent failures into typed
ones.

</div>
