# The kvcli Client

A server is only half of a protocol. `redis-cli` exists; you have been
using it to test Kestrel through every chapter since Chapter 16. But
writing our own client teaches the *other* half of RESP — the half that
sends commands instead of receiving them — and it is the cleanest place
in the book to demonstrate how a simple synchronous TCP client looks in
modern C++.

`kvcli` is intentionally small. It connects to a Kestrel server, reads a
line from the user, parses the line into a RESP array of bulks, writes
those bytes, reads the response, prints it, and loops. No event loop,
no thread pool, no buffer juggling — a blocking TCP client of the kind
nobody bothers to over-engineer. The result is wire-compatible with
`redis-cli` against any RESP2 server, including the real Redis.

The C++ topics are **command-line argument handling**, **`std::optional`
in a CLI shape**, and the discipline of **using the same parser /
serializer for both ends** of a protocol — Chapter 6's code does most
of the work for us.

By the end, `src/client/main.cpp` is a working `kvcli` you can drop into
any Redis-shaped environment. It is also the chapter where you see, with
fresh eyes, how much smaller a *client* implementation is than its
server — a useful piece of architectural perspective.

---

## What `kvcli` has to do

The behavior:

```
$ kvcli -h 127.0.0.1 -p 6379
127.0.0.1:6379> SET hello world
OK
127.0.0.1:6379> GET hello
"world"
127.0.0.1:6379> DEL hello
(integer) 1
127.0.0.1:6379> GET nope
(nil)
127.0.0.1:6379> ^D
$
```

So: connect, prompt, read a line, send a RESP array, decode a RESP
response, print a human-friendly rendering, repeat. The server-side
parser from Chapter 6 already does the hardest part — reading RESP from
a byte stream — so `kvcli` reuses it. The new work is the *sending*
side: turning `SET hello world` into the byte sequence
`*3\r\n$3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n`.

---

## Splitting a command line

The simplest tokenization that handles real input: whitespace separates
tokens; double-quoted spans preserve spaces inside them; backslash
escapes the next character. This is what `redis-cli` does. A handful of
lines:

```cpp
// src/client/lex.hpp
#pragma once

#include <string>
#include <vector>

namespace kestrel::client {

// Split a command line into argument tokens. Honours double quotes and
// backslash escapes. Returns the tokens; never throws.
std::vector<std::string> tokenize(std::string_view line);

}  // namespace kestrel::client
```

```cpp
// src/client/lex.cpp
#include "client/lex.hpp"

namespace kestrel::client {

std::vector<std::string> tokenize(std::string_view line) {
    std::vector<std::string> out;
    std::string cur;
    bool in_quotes = false;
    bool seen      = false;        // whether cur contains anything yet

    for (std::size_t i = 0; i < line.size(); ++i) {
        char c = line[i];
        if (c == '\\' && i + 1 < line.size()) {
            cur.push_back(line[++i]);
            seen = true;
            continue;
        }
        if (c == '"') { in_quotes = !in_quotes; seen = true; continue; }
        if (!in_quotes && (c == ' ' || c == '\t')) {
            if (seen) { out.push_back(std::move(cur)); cur.clear(); seen = false; }
            continue;
        }
        cur.push_back(c);
        seen = true;
    }
    if (seen) out.push_back(std::move(cur));
    return out;
}

}  // namespace kestrel::client
```

The tokenizer's only subtlety: an empty `""` is a *real* (empty) token,
not "no token." So we track `seen` separately from `cur.empty()` —
`seen` becomes true the moment we open a quote, even if no character
gets written to `cur` before the close.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

Shell-style argument splitting is what `shlex.split` (Python) or a
`yargs` parser (Node) does. This 30-line C++ version handles the
RESP-realistic subset: spaces, double quotes, backslash. It is purposely
small enough that you could hand it to a junior reviewer and they'd
understand every edge case. Don't reach for a CLI argument library here;
the cost of a dependency exceeds the cost of writing the tokenizer.

</div>

---

## Encoding the array

Once we have tokens, encoding them as a RESP array of bulks is two
loops:

```cpp
// src/client/encode.hpp
#pragma once

#include <string>
#include <string_view>
#include <vector>

namespace kestrel::client {

std::string encode_array(const std::vector<std::string>& argv);

}  // namespace kestrel::client
```

```cpp
// src/client/encode.cpp
#include "client/encode.hpp"

#include <charconv>

namespace kestrel::client {

static void append_int(std::string& out, long long v) {
    char buf[24];
    auto [p, ec] = std::to_chars(buf, buf + sizeof buf, v);
    out.append(buf, p);
}

std::string encode_array(const std::vector<std::string>& argv) {
    std::string out;
    out.reserve(argv.size() * 16);
    out += '*';
    append_int(out, static_cast<long long>(argv.size()));
    out += "\r\n";
    for (auto const& s : argv) {
        out += '$';
        append_int(out, static_cast<long long>(s.size()));
        out += "\r\n";
        out += s;
        out += "\r\n";
    }
    return out;
}

}  // namespace kestrel::client
```

`std::to_chars` again, just as the server used in Chapter 6 — no
exceptions, no allocation, no locale. The encoder hands back a
`std::string` of bytes; the connect-and-write code passes those bytes
to `write()`.

For binary-safe values (bytes containing `\r`, `\n`, or `\0`), the
encoder is already correct — bulks are length-prefixed, so the content
is opaque. The *user-facing prompt* would need a way to enter such
values (probably by `\xNN` escape sequences in the tokenizer), but that
is a nicety we'll leave to a real reader to add.

---

## Connecting and talking

The connect-and-talk loop uses the blocking `Connection` from Chapter
14 — that was its motivating use case. We don't need the event loop, we
don't need worker threads. One thread, one socket, blocking reads and
writes:

```cpp
// src/client/main.cpp  (sketch)
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#include <cstring>
#include <iostream>
#include <string>

#include "client/encode.hpp"
#include "client/lex.hpp"
#include "net/connection.hpp"
#include "net/file_descriptor.hpp"
#include "protocol/parse.hpp"

namespace {

std::expected<kestrel::net::Connection, std::string>
connect(const std::string& host, std::uint16_t port) {
    kestrel::net::FileDescriptor sock(::socket(AF_INET, SOCK_STREAM, 0));
    if (!sock.valid()) return std::unexpected("socket");

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(port);
    if (::inet_pton(AF_INET, host.c_str(), &addr.sin_addr) <= 0)
        return std::unexpected("invalid host");

    if (::connect(sock.get(), reinterpret_cast<sockaddr*>(&addr), sizeof addr) != 0)
        return std::unexpected(std::strerror(errno));
    return kestrel::net::Connection(std::move(sock));
}

}  // namespace

int main(int argc, char** argv) {
    // 1. Parse -h/-p flags.
    std::string host = "127.0.0.1";
    std::uint16_t port = 6379;
    for (int i = 1; i < argc; ++i) {
        std::string a = argv[i];
        if      (a == "-h" && i + 1 < argc) host = argv[++i];
        else if (a == "-p" && i + 1 < argc) port = static_cast<std::uint16_t>(std::atoi(argv[++i]));
        else if (a == "--help") { /* print usage */ return 0; }
    }

    // 2. Connect.
    auto conn_or = connect(host, port);
    if (!conn_or) {
        std::fprintf(stderr, "connect: %s\n", conn_or.error().c_str());
        return 1;
    }
    auto& conn = *conn_or;

    // 3. REPL.
    std::string line;
    std::string prompt = host + ":" + std::to_string(port) + "> ";
    std::vector<std::uint8_t> in;
    while (std::cout << prompt, std::getline(std::cin, line)) {
        if (line.empty()) continue;
        auto argv = kestrel::client::tokenize(line);
        if (argv.empty()) continue;

        auto enc = kestrel::client::encode_array(argv);
        auto wr = conn.write_all(std::span(
            reinterpret_cast<const std::uint8_t*>(enc.data()), enc.size()));
        if (!wr) { std::fprintf(stderr, "write: %s\n", wr.error().what.c_str()); return 1; }

        // 4. Read until one complete RESP message is parsed.
        while (true) {
            std::size_t consumed = 0;
            std::string_view view(reinterpret_cast<const char*>(in.data()), in.size());
            auto r = kestrel::protocol::parse(view, consumed);
            if (!r) { std::fprintf(stderr, "protocol: %s\n", r.error().what.data()); return 1; }
            if (r->has_value()) {
                print_reply(**r);
                in.erase(in.begin(), in.begin() + consumed);
                break;
            }
            std::uint8_t buf[4096];
            auto n = conn.read(buf);
            if (!n) { std::fprintf(stderr, "read: %s\n", n.error().what.c_str()); return 1; }
            if (*n == 0) { std::fprintf(stderr, "server closed\n"); return 1; }
            in.insert(in.end(), buf, buf + *n);
        }
    }
    return 0;
}
```

The structure should look familiar — every piece reuses code from earlier
chapters:

- `Connection::write_all` (Chapter 14) for the blocking write.
- `Connection::read` (Chapter 14) for the blocking read.
- `kestrel::protocol::parse` (Chapter 6, refined in Chapter 7) for the
  incremental parse. We're using the *exact same parser* the server uses
  to read commands; here we use it to read responses. RESP's symmetric
  shape lets one parser serve both directions.

That reuse is the architectural point. The protocol code does not know
or care which side it sits on. The parser doesn't care if the bytes came
from a client's request or a server's response; it just sees RESP.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

This is *the* canonical "TCP client" loop in any language — connect,
write line, read response, repeat. The Node and Python forms are
prettier (no `reinterpret_cast<sockaddr*>`, no manual `connect`), but
they are the same algorithm. The C++ version is twice as long; almost
all of it is the syscall plumbing. Once you have the wrappers (and we do,
from Chapter 14), the client logic is shorter than you'd expect.

</div>

---

## Pretty-printing the reply

A response is a `protocol::Message` (a `std::variant`); `kvcli` renders
it like `redis-cli`:

```cpp
void print_reply(const protocol::Message& m) {
    using namespace kestrel::protocol;
    std::visit([](auto const& alt) {
        using T = std::decay_t<decltype(alt)>;
        if constexpr (std::is_same_v<T, SimpleString>) {
            std::cout << alt.text << '\n';
        } else if constexpr (std::is_same_v<T, Error>) {
            std::cout << "(error) " << alt.text << '\n';
        } else if constexpr (std::is_same_v<T, Integer>) {
            std::cout << "(integer) " << alt.value << '\n';
        } else if constexpr (std::is_same_v<T, BulkString>) {
            if (alt.is_null) std::cout << "(nil)\n";
            else             std::cout << '"' << alt.bytes << "\"\n";
        } else if constexpr (std::is_same_v<T, Array>) {
            if (alt.is_null) { std::cout << "(nil)\n"; return; }
            for (std::size_t i = 0; i < alt.items.size(); ++i) {
                std::cout << (i + 1) << ") ";
                print_reply(alt.items[i]);    // recurse
            }
        }
    }, m);
}
```

Once more `std::visit` does the dispatch, and once more it would refuse
to compile if a new `Message` alternative slipped in unhandled. The
`if constexpr` inside the generic lambda is the lighter-weight idiom for
this shape; the alternative is a `struct` of overloaded `operator()`s
(Chapter 5's other form), equally good.

For binary-unsafe rendering (a bulk containing `\0` or `\n`), the above
is wrong — `std::cout << alt.bytes` prints the bytes as-is, which can
mangle a terminal. Real `redis-cli` escapes non-printable bytes as
`\x..`. Adding the escape is a one-page exercise; we leave it as the
chapter's experiment.

<div class="callout callout-experiment">

🧪 **Experiment**

Run `kvcli` against your Kestrel server. Try:

- `SET key value` → `OK`
- `GET key` → `"value"`
- `LPUSH list a b c` → `(integer) 3`
- `LRANGE list 0 -1` → `1) "c"  2) "b"  3) "a"`
- `DEL nothere` → `(integer) 0`

Now run `kvcli` against real Redis: `redis-server` in another terminal,
then `kvcli`. Everything should work identically. *That* is wire
compatibility. Then break a value with a binary byte:
`SET bin "abc\x00def"` and watch your unescaped `print_reply` mangle
the output. Add escaping. Re-run, see clean output.

</div>

---

## Reading from stdin: non-interactive mode

A useful nicety: when stdin is not a tty (e.g., `echo "PING" | kvcli`),
skip the prompt and don't bother with line editing. POSIX's
`isatty(fileno(stdin))` is the predicate. We will not implement readline-
style line editing (history, ctrl-arrow navigation) — that's a real
library's worth of code. `redis-cli` uses `linenoise`; Kestrel's `kvcli`
uses `std::getline` and accepts that the user types one line at a time
without history. Adding `linenoise` is left as a real-world exercise in
applying Chapter 8's dependency-management tools.

```cpp
bool interactive = ::isatty(::fileno(stdin));
while (true) {
    if (interactive) std::cout << prompt;
    if (!std::getline(std::cin, line)) break;
    // ...send/receive as above...
}
```

This is a small thing that meaningfully changes how the binary behaves
when used in scripts — pipelines like
`echo "GET hello" | kvcli | grep world` should not print prompts to
stdout.

---

## Tests for the client

Two small tests cover the client-only logic — the tokenizer and the
encoder. The full connect-and-talk flow we test by running it against
the server (an integration test).

```cpp
// src/test/client_lex_test.cpp
#include <doctest/doctest.h>
#include "client/lex.hpp"

TEST_CASE("simple tokens") {
    auto t = kestrel::client::tokenize("SET key value");
    REQUIRE(t.size() == 3);
    CHECK(t[0] == "SET");
    CHECK(t[1] == "key");
    CHECK(t[2] == "value");
}

TEST_CASE("quoted span") {
    auto t = kestrel::client::tokenize(R"(SET key "hello world")");
    REQUIRE(t.size() == 3);
    CHECK(t[2] == "hello world");
}

TEST_CASE("empty quoted is a token") {
    auto t = kestrel::client::tokenize(R"(SET key "")");
    REQUIRE(t.size() == 3);
    CHECK(t[2] == "");
}

TEST_CASE("backslash escape") {
    auto t = kestrel::client::tokenize(R"(GET "a\"b")");
    REQUIRE(t.size() == 2);
    CHECK(t[1] == "a\"b");
}
```

```cpp
// src/test/client_encode_test.cpp
#include <doctest/doctest.h>
#include "client/encode.hpp"

TEST_CASE("encode three-arg command") {
    auto s = kestrel::client::encode_array({"SET", "k", "v"});
    CHECK(s == "*3\r\n$3\r\nSET\r\n$1\r\nk\r\n$1\r\nv\r\n");
}

TEST_CASE("encode binary-safe value") {
    auto s = kestrel::client::encode_array({"SET", "k", std::string("a\0b", 3)});
    CHECK(s == std::string("*3\r\n$3\r\nSET\r\n$1\r\nk\r\n$3\r\na\0b\r\n", 27));
}
```

Both stay clean under ASan/UBSan. Integration tests where `kvcli` talks
to a real Kestrel are a CMake target that starts the server, runs the
client, asserts on output — useful for CI but a touch heavyweight for
this book.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's `kvcli` (`client/lex.{hpp,cpp}`, `client/encode.{hpp,cpp}`,
> `client/main.cpp`). Review: (1) is the tokenizer correctly handling
> empty tokens, backslashes, and mismatched quotes? (2) is `connect()`'s
> error handling complete — what about DNS, IPv6, refused connections?
> (3) does `print_reply` correctly escape non-printable bytes for a
> terminal? (4) is `std::cout` flushed at the right points so the prompt
> never gets ahead of the output?

</div>

---

## Where this leaves Kestrel

Kestrel is a complete client/server pair. `kvd` listens; `kvcli` connects.
The protocol module is symmetric — same parse, same serialize, both
directions — and the design from Chapter 6 onwards has been preparing
for this. Chapter 21 takes the whole thing under measurement: benchmarks
(how many ops/sec?), profiling (where are the cycles going?), and
reflection (what would you change for production?).

The client is also the first time you've seen "the same C++ codebase
built as multiple binaries from one CMake project." `kvd` and `kvcli`
share `core/`, `protocol/`, and `net/` as a static library; only their
`main`s differ. `target_link_libraries` lets each binary include only
the modules it needs — `kvcli` doesn't link `store/` or `persistence/`,
which it would never touch.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build, run the integration:

```bash
cmake --build src/build
./src/build/kvd &
./src/build/kvcli                # talks to your kvd
./src/build/kvcli -h 127.0.0.1 -p 6379 -- SET hi there   # optional one-shot mode
kill %1
```

You should be able to:

- Sketch the entire `kvcli` flow in five steps.
- Defend why `kvcli` reuses the server's RESP parser instead of writing
  its own.
- Identify the precise places in the code where Chapter 14's blocking
  `Connection` is the right tool (and the precise places it would not be
  for `kvd`).
- Explain why `target_link_libraries(kvcli PRIVATE ...)` does *not*
  include `store/` or `persistence/`.

Hand it over:

> Review my Chapter 20 client. Are there CLI edge cases I'm missing —
> stdin EOF, broken pipe to stdout when piped into `head`, signal
> handling? Are my unit tests for `tokenize` covering the right cases?
> Is the integration-test pattern I'm using (start kvd, run kvcli,
> assert) sound in CI?

When that's green, Chapter 21 closes the book with benchmarks and the
capstone.

</div>
