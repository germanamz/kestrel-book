# Buffered Connection I/O

The reactor from Chapter 15 delivers readiness notifications; the parser
from Chapter 6 turns bytes into messages. The two have never met. Until
they do, Kestrel can echo bytes but cannot speak RESP, and the persistence
layer from Chapter 13 has nothing to plug into.

This chapter is the gluing chapter. It writes the **buffered connection**
that lives between socket and protocol — a state machine that:

1. Reads bytes from the socket whenever it is readable (Chapter 15).
2. Feeds them to the incremental RESP parser (Chapter 6) until one or
   more complete commands emerge.
3. Dispatches each command to the keyspace (Chapter 9) and writes the
   command into the WAL (Chapter 12).
4. Serializes responses through the RESP writer (Chapter 6) into an
   outbound buffer.
5. Drains that buffer to the socket whenever it is writable.

By the end, Kestrel will respond to `redis-cli` for the small command set
the in-memory layer supports: `SET`, `GET`, `DEL`, `EXPIRE`, `LPUSH`,
`LRANGE`, `HSET`, `HGET`, `ZADD`, plus the housekeeping `PING`. The
chapter's C++ topics are about **backpressure** (what to do when a slow
client makes `out_` grow without bound), **half-close** semantics, and the
correctness contract between an incremental parser and a non-blocking
socket.

---

## The pipeline, in one picture

```
            +--------+    bytes    +--------+    Message    +----------+
  socket -> | read   | ----------> | parser | -----------> | dispatch |
            +--------+             +--------+              +----------+
                                                                 |
                                                                 v
                                                            +--------+
                                                            |  WAL   |
                                                            +--------+
                                                                 |
                                                                 v
                                                            +--------+
            +--------+    bytes    +--------+    Message  |  apply  |
  socket <- | write  | <---------- | writer | <---------- |  to KV  |
            +--------+             +--------+              +--------+
```

Read direction is *pull*: the socket has bytes, the parser pulls until
either a complete message emerges or the buffer is exhausted, the
dispatcher pulls the parsed messages until none remain.

Write direction is *push*: a response is appended to the outbound buffer,
the writer pushes whenever the socket is writable, and the loop only
re-watches for writability when the buffer is non-empty.

The two directions share a `Connection` object that holds both buffers, the
parser's incremental state (none, in this design — the parser is pure),
and the loop registration. RAII manages the registration: a `Connection`'s
destructor calls `loop_.remove(fd_.get())` and then closes the fd, freeing
both halves cleanly.

---

## The full `Connection`

Picking up from Chapter 15's sketch, the connection grows the parser and
dispatch hooks:

```cpp
// src/net/connection.hpp
#pragma once

#include <cstdint>
#include <vector>

#include "core/buffer.hpp"
#include "net/event_loop.hpp"
#include "net/file_descriptor.hpp"
#include "protocol/parse.hpp"
#include "store/hash_table.hpp"
#include "persistence/wal.hpp"

namespace kestrel::net {

class Connection {
public:
    Connection(EventLoop& loop, FileDescriptor fd,
               store::HashTable& kv, persistence::Wal& wal);
    ~Connection();

    Connection(const Connection&) = delete;
    Connection& operator=(const Connection&) = delete;

    // Loop callbacks.
    void on_readable();
    void on_writable();

private:
    void parse_and_dispatch();
    void send(std::span<const std::uint8_t> bytes);
    void try_flush();
    void close();

    EventLoop&        loop_;
    FileDescriptor    fd_;
    store::HashTable& kv_;
    persistence::Wal& wal_;

    std::vector<std::uint8_t> in_;
    std::vector<std::uint8_t> out_;
    std::size_t out_off_ = 0;
    bool watching_write_ = false;
    bool closed_ = false;
};

}  // namespace kestrel::net
```

The implementation in three pieces.

### Reading and parsing

```cpp
void Connection::on_readable() {
    std::uint8_t scratch[8192];
    for (;;) {
        ssize_t n = ::read(fd_.get(), scratch, sizeof scratch);
        if (n > 0) { in_.insert(in_.end(), scratch, scratch + n); continue; }
        if (n == 0) { close(); return; }                       // EOF
        if (errno == EINTR) continue;
        if (errno == EAGAIN || errno == EWOULDBLOCK) break;    // drained
        close(); return;                                       // any other error
    }
    parse_and_dispatch();
}

void Connection::parse_and_dispatch() {
    using namespace kestrel::protocol;
    std::string_view view(reinterpret_cast<const char*>(in_.data()), in_.size());

    std::size_t total_consumed = 0;
    while (true) {
        std::size_t consumed = 0;
        auto r = parse(view.substr(total_consumed), consumed);
        if (!r) { close(); return; }                           // protocol error
        if (!r->has_value()) break;                            // need more bytes
        dispatch(**r);                                          // apply, append response to out_
        total_consumed += consumed;
    }
    in_.erase(in_.begin(), in_.begin() + total_consumed);
    try_flush();
}
```

Two design choices are worth defending.

**Read everything, then parse everything.** Inside `on_readable` we drain
the socket fully before parsing; inside `parse_and_dispatch` we keep
parsing as long as complete messages remain. This batches: a client that
pipelined `SET k v\nSET k2 v2\nGET k\n` arrives as one or two reads, and
all three messages are processed in one callback. Pipelining is one of the
main reasons RESP-style protocols are fast; the parser+loop design must
not erase that gain.

**`in_.erase(begin, begin+n)` to advance the buffer.** Linear-time per
advance, but typical message sizes are small and the buffer is short.
For higher throughput you'd switch to a ring buffer (Chapter 11's
algorithm applied here) or move the residual to the front only when the
buffer crosses a watermark. We'll measure in Chapter 21.

### Dispatching a parsed command

The dispatcher is a function (eventually a class with a command table)
that takes a parsed `Message` representing an array of bulks, looks at
the first bulk's bytes, and dispatches to the right handler:

```cpp
void Connection::dispatch(const protocol::Message& msg) {
    using namespace kestrel::protocol;

    const Array* arr = std::get_if<Array>(&msg);
    if (!arr || arr->is_null || arr->items.empty()) {
        write_error(buffer_view(out_), "ERR malformed command");
        return;
    }
    const BulkString* cmd = std::get_if<BulkString>(&arr->items[0]);
    if (!cmd) {
        write_error(buffer_view(out_), "ERR command must be a bulk string");
        return;
    }

    std::string_view name(cmd->bytes);
    if      (icase_eq(name, "PING"))   handle_ping(*arr);
    else if (icase_eq(name, "SET"))    handle_set(*arr);
    else if (icase_eq(name, "GET"))    handle_get(*arr);
    else if (icase_eq(name, "DEL"))    handle_del(*arr);
    // ... and so on through the supported command set
    else                                handle_unknown(name);
}
```

A real Kestrel will eventually have a *command table* — an `unordered_map`
from command name to a function pointer or `std::function`. For Part 5's
purposes, the `if/else` chain is clearer, and the compiler will turn it
into a jump table anyway given a recent compiler. We refactor to the
table when the command list grows past twenty.

Each handler does the same three things:

1. **Validate arity.** `SET` needs exactly three bulks (`SET`, key, value).
   Reply with an error on mismatch.
2. **Apply to the keyspace.** Possibly write to the WAL first if the
   command mutates.
3. **Serialize the response into `out_`.**

A representative pair, `SET` and `GET`:

```cpp
void Connection::handle_set(const protocol::Array& arr) {
    if (arr.items.size() != 3) { /* reply error and return */ }
    const auto* k = std::get_if<protocol::BulkString>(&arr.items[1]);
    const auto* v = std::get_if<protocol::BulkString>(&arr.items[2]);
    if (!k || !v) { /* reply error */ }

    // 1. WAL first (durability before ack).
    auto rec = encode_set(k->bytes, v->bytes);   // see Chapter 13's serializer
    auto wr = wal_.append(rec);
    if (!wr) { /* reply error and close */ return; }

    // 2. Apply in-memory.
    std::string key{k->bytes};                   // owning copy for the table
    kestrel::String val(
        std::span(reinterpret_cast<const std::uint8_t*>(v->bytes.data()),
                  v->bytes.size()));
    kv_.insert_or_assign(std::move(key), kestrel::Value{std::move(val)});

    // 3. Reply.
    protocol::write_simple(view_of(out_), "OK");
}

void Connection::handle_get(const protocol::Array& arr) {
    if (arr.items.size() != 2) { /* reply error */ return; }
    const auto* k = std::get_if<protocol::BulkString>(&arr.items[1]);
    if (!k) { /* reply error */ return; }

    auto* val = kv_.find(k->bytes);
    if (!val) { protocol::write_null_bulk(view_of(out_)); return; }

    if (auto* s = std::get_if<kestrel::String>(val)) {
        protocol::write_bulk(view_of(out_), s->bytes());
        return;
    }
    // The key is of a wrong type (e.g., LIST). Redis returns an error here.
    protocol::write_error(view_of(out_), "WRONGTYPE");
}
```

The persistence-then-memory order matters: a `SET` is acknowledged only
after the WAL has accepted it. With `FsyncMode::Always`, that
acknowledgement implies the bytes are on disk. If the WAL append fails,
Kestrel does *not* mutate the in-memory state — anything else risks
returning `OK` for a write the recoverer would not replay.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

In a Node Redis-like, a write handler `await wal.append(record)` then
mutates the in-memory map. The shape is identical here, just without the
`await`. The C++ analog is "block on the WAL, then mutate." Since the WAL
is local, fast, and small, this is fine; if it became a bottleneck, the
pattern is to queue WAL appends to a writer thread and not acknowledge
until the writer signals durability — Part 6's worker pool will give us
those tools.

</div>

### Writing back

`try_flush` drains as much of `out_` as the kernel will take in one
go, and toggles loop registration for writability:

```cpp
void Connection::try_flush() {
    while (out_off_ < out_.size()) {
        ssize_t n = ::write(fd_.get(), out_.data() + out_off_, out_.size() - out_off_);
        if (n > 0) { out_off_ += static_cast<std::size_t>(n); continue; }
        if (errno == EINTR) continue;
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            if (!watching_write_) {
                loop_.modify(fd_.get(), Watch::ReadWrite);
                watching_write_ = true;
            }
            return;
        }
        close(); return;
    }
    // Drained fully; collapse the buffer.
    out_.clear();
    out_off_ = 0;
    if (watching_write_) {
        loop_.modify(fd_.get(), Watch::Read);
        watching_write_ = false;
    }
}

void Connection::on_writable() { try_flush(); }
```

The "ask the loop to stop watching for writability when the buffer
empties" step is crucial. Edge-triggered loops will fire on writability
for as long as the buffer has room — without un-watching, the callback
runs in a tight loop after every flush, wasting CPU.

---

## Backpressure

What if a client is slow? Each `SET` writes the response into `out_`,
which grows; if the kernel can't drain it fast enough, the buffer
grows without bound, and a single misbehaving client can exhaust Kestrel's
memory.

Real Redis has two defenses:

- **Output buffer limits**: a per-client memory cap. When exceeded,
  Redis disconnects the client. Cheap, decisive, the right policy for a
  cache.
- **Replica / pub-sub special cases**: replicas get larger limits because
  they catch up after disconnect; pub-sub clients get a "soft" limit
  before the hard one.

For Kestrel we implement the simple bound: a constant `kMaxOutBufferBytes`,
and on exceeded, close the connection with an error.

```cpp
void Connection::send(std::span<const std::uint8_t> bytes) {
    if (out_.size() + bytes.size() > kMaxOutBufferBytes) { close(); return; }
    out_.insert(out_.end(), bytes.begin(), bytes.end());
    try_flush();
}
```

Buffering tradeoffs are not always so clean — but for an in-memory store
where the working set should fit in RAM and clients should be local,
"throttle by disconnecting" is the right default.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`close()` from inside an event-loop callback while the loop is in the
middle of dispatching events for this same fd is the bug Chapter 15
warned about. Concretely: if `try_flush` calls `close()`, the next
ready event in the same batch for this fd will deliver to a destroyed
`Connection`. Two mitigations: (1) check `closed_` at the start of every
callback and exit early; (2) defer destruction until end-of-batch (the
loop maintains a "to-delete" list). The simple defensive check inside
callbacks is enough for Kestrel:

```cpp
void Connection::on_readable() {
    if (closed_) return;
    // ...
}
```

</div>

---

## The full pipeline, end to end

Putting it together in `main`:

```cpp
int main(int argc, char** argv) {
    std::signal(SIGPIPE, SIG_IGN);

    // 1. Open WAL, recover keyspace.
    auto wal_or = kestrel::persistence::Wal::open(
        "/var/lib/kestrel/kestrel.wal", kestrel::persistence::FsyncMode::EverySecond);
    if (!wal_or) { /* fatal */ return 1; }
    auto kv_or = kestrel::persistence::recover("/var/lib/kestrel");
    if (!kv_or) { /* fatal */ return 1; }

    // 2. Listen, set up loop.
    auto listener_or = kestrel::net::Listener::bind(6379);
    if (!listener_or) { return 1; }

    kestrel::net::EventLoop loop;
    kestrel::net::set_non_blocking(listener_or->fd()).value();

    // 3. Connection table (owned, fd-indexed).
    std::unordered_map<int, std::unique_ptr<kestrel::net::Connection>> conns;

    loop.add(listener_or->fd(), kestrel::net::Watch::Read, [&](auto) {
        for (;;) {
            auto c = listener_or->accept();
            if (!c) break;
            kestrel::net::set_non_blocking(c->get()).value();
            int cfd = c->get();
            conns[cfd] = std::make_unique<kestrel::net::Connection>(
                loop, std::move(*c), *kv_or, *wal_or);
        }
    });

    // 4. Run.
    loop.run();
}
```

That `main` is the whole Kestrel binary up to the end of Part 5: WAL
opened, keyspace recovered, listener bound, loop driving everything. A
real `redis-cli` connects, sends `SET hello world\r\n`, sees `+OK\r\n`
come back. `GET hello` returns `$5\r\nworld\r\n`. The server is real.

<div class="callout callout-experiment">

🧪 **Experiment**

Run Kestrel and `redis-cli ping`. Then `set foo bar`. Then `get foo`.
Then restart Kestrel and `get foo` again — confirm the persistence layer
brought your value back. Now pipeline:

```bash
(echo "PING"; echo "SET a 1"; echo "GET a") | redis-cli --pipe
```

Confirm the connection handled three commands in a single read, sent one
batch of responses, and closed cleanly. This is the proof that pipelining
works through your incremental parser and your batched flush.

</div>

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's my `Connection` (`net/connection.{hpp,cpp}`) wired up to the
> parser, dispatch, and WAL. Review: (1) is the WAL-before-memory order
> enforced for every mutating command? (2) is `try_flush` correctly
> toggling the writable watch on and off? (3) is my backpressure handling
> on `kMaxOutBufferBytes` reasonable, or should it be byte limits, time
> limits, or both? (4) is there any way the dispatch logic could leave
> `out_` in a half-formed state (e.g., wrote the prefix then errored
> midway through serialization)?

</div>

---

## Where Part 5 ends

Kestrel speaks Redis. A `redis-cli` user can connect to it, issue
commands, see responses, and survive a restart. The architecture from
this point is:

- One thread driving the event loop, handling all I/O and dispatch.
- A blocking WAL `append` per mutating command.
- A keyspace, persistence layer, and protocol layer that have not changed
  in shape since Parts 3 and 4 — only their callers changed.

The bottleneck is now CPU on the single event-loop thread, plus the WAL
`fsync` latency. Part 6 attacks both. Chapter 17 covers the
threading-and-memory-model preliminaries; Chapter 18 introduces a
**worker thread pool** that takes computational work off the loop; Chapter
19 ties the loop and the pool together with cross-thread wakeup and
careful ownership.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

A working Kestrel server end-to-end:

```bash
cmake --build src/build
./src/build/kvd &
redis-cli ping            # PONG
redis-cli set foo bar     # OK
redis-cli get foo         # "bar"
kill %1
./src/build/kvd &         # recovery
redis-cli get foo         # still "bar"
```

You should be able to:

- Trace a `SET` command from byte 1 on the socket to the response back on
  the socket, naming every component touched.
- Explain the WAL-then-memory ordering and what would break if you
  reversed it.
- Defend the un-watching of writability when `out_` drains.
- Discuss what backpressure does on a slow client and which alternative
  designs you'd consider for a non-cache workload.

Hand it over:

> Walk through my Chapter 16 implementation. Are there ways a client can
> push Kestrel into an inconsistent state — for instance, by pipelining
> many writes faster than the WAL can sync? Is my dispatch logic
> resilient against malformed RESP (e.g., a `SET` with non-bulk args)?
> Suggest one stress test that would exercise the pipelining path
> aggressively.

When that's green, Part 5 is complete and Part 6 introduces threads.

</div>
