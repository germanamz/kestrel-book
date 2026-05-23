# The Event Loop

Chapter 14's echo server worked for one client at a time. Adding a second
client made the first wait for it; adding a thousand made the whole thing
useless. The classical fix — one thread per connection — works up to a point
and then collapses under context-switch cost, kernel stack memory, and lock
contention. Modern servers (Redis, nginx, Node.js, Go's runtime) sidestep
the problem by sharing a small number of threads across many connections,
**driving I/O through readiness notifications instead of blocking on each
one**. That pattern is the **reactor**, and Kestrel implements it now.

This chapter builds Kestrel's event loop. The reactor sits on top of one of
two kernel APIs: **`kqueue`** on macOS / BSD, **`epoll`** on Linux. They are
not identical but they are close enough that you can put one C++ interface
in front of both, and Kestrel's `EventLoop` does exactly that. The C++
topics are the **PIMPL idiom** (a clean way to hide a platform-dependent
implementation behind a stable interface), an **interface for callbacks**
(function objects, `std::function`, and the trade-offs), and a tour through
how to make sockets actually non-blocking.

By the end, the single-client echo server becomes a thousand-client echo
server with no extra threads, and you have the substrate Chapter 16 plugs
the RESP parser into.

---

## Why "readiness" instead of "blocking"

A blocking `read(fd, buf, n)` parks the calling thread until at least one
byte is available. With one connection, that is fine. With ten thousand,
you'd need ten thousand threads to park — each consuming hundreds of
kilobytes of stack and switching costs.

The reactor's idea inverts the control flow:

1. Put every socket in **non-blocking** mode. Now `read` returns
   immediately, either with bytes or with `EAGAIN`/`EWOULDBLOCK` (meaning
   "I'd have blocked").
2. Ask the kernel to *notify* you when a socket becomes readable or
   writable. The mechanism is `epoll_wait` on Linux, `kevent` on BSD/macOS.
3. Sit in a loop: ask the kernel which sockets are ready, do the cheap
   non-blocking I/O on those, then ask again. The loop is a single thread
   handling thousands of connections.

This is what Node's `libuv`, Redis's `aeMain`, and nginx all do underneath.
The cost of the abstraction is that *every read or write must be prepared
to do nothing yet*, and *every connection's parsing state must persist
across loop iterations*. The benefit is that one CPU can fully utilize a
NIC servicing tens of thousands of concurrent clients.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

Node and Python's `asyncio` are reactors. The `async`/`await` in your code
is sugar over the same readiness loop: when you `await socket.read()`, the
runtime registers your continuation against the kernel notification and
parks. C++ has coroutines as of C++20 (and they would fit beautifully on
this loop), but Kestrel uses the simpler callback-based reactor — it is
easier to teach and easier to debug. The mental model is identical to
Node's `epoll`-backed loop.

</div>

---

## `kqueue` and `epoll` in five operations

Both APIs offer the same conceptual operations under different names. We
build one C++ interface over both:

| Operation              | `epoll` (Linux)             | `kqueue` (macOS/BSD)                    |
|------------------------|-----------------------------|-----------------------------------------|
| Create the loop        | `epoll_create1(0)`          | `kqueue()`                              |
| Watch a fd             | `epoll_ctl(EPOLL_CTL_ADD)`  | `EV_SET(..., EV_ADD)`; `kevent()`       |
| Stop watching a fd     | `epoll_ctl(EPOLL_CTL_DEL)`  | `EV_SET(..., EV_DELETE)`; `kevent()`    |
| Modify what to watch   | `epoll_ctl(EPOLL_CTL_MOD)`  | second `EV_SET`                         |
| Wait for any to fire   | `epoll_wait(events, n, ms)` | `kevent(changes=null, events, n, ts)`   |

The C++ interface that hides both:

```cpp
// src/net/event_loop.hpp
#pragma once

#include <cstdint>
#include <functional>
#include <memory>

namespace kestrel::net {

enum class Watch : std::uint8_t { Read = 1, Write = 2, ReadWrite = 3 };

using Callback = std::function<void(Watch ready)>;

class EventLoop {
public:
    EventLoop();
    ~EventLoop();
    EventLoop(const EventLoop&) = delete;
    EventLoop& operator=(const EventLoop&) = delete;

    void add(int fd, Watch what, Callback cb);
    void modify(int fd, Watch what);
    void remove(int fd);

    // Block until at least one event fires (or timeout); dispatch callbacks.
    void poll(int timeout_ms = -1);

    void stop() { running_ = false; }
    void run() { running_ = true; while (running_) poll(); }

private:
    struct Impl;
    std::unique_ptr<Impl> impl_;
    bool running_ = false;
};

}  // namespace kestrel::net
```

The interesting C++ here is **PIMPL** — *Pointer to IMPLementation*. The
private `struct Impl` is *declared* in the header but *defined* in the
`.cpp`, where the platform-specific bits live. The owner type
`std::unique_ptr<Impl>` knows just enough to call `Impl`'s destructor (which
is supplied in the `.cpp` and so sees the full definition).

PIMPL's benefits:

1. **The header is platform-clean.** No `#ifdef __APPLE__` leaking into
   every caller. The cost moves to the one place that legitimately knows:
   the implementation file.
2. **The header is also ABI-stable.** Adding a field to `Impl` does not
   change `EventLoop`'s size; callers don't recompile.
3. **Compile time.** Including `<sys/event.h>` (kqueue) or
   `<sys/epoll.h>` (epoll) leaks heavy headers; the PIMPL contains them.

The cost is one heap allocation per `EventLoop` and one indirection per
call. For a loop that runs forever, neither matters.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

PIMPL plays a similar role to TypeScript's `interface` vs `class` split, or
Python's "use a class, hide the implementation in a private attribute"
discipline. The difference is that C++ requires the indirection to be in
the type system, not just by convention — the header must compile without
seeing the platform-specific structs, which means the implementation is
literally invisible to consumers of the header.

</div>

---

## The PIMPL `Impl` (per platform)

On macOS, the implementation owns a kqueue fd plus a map of fd → callback:

```cpp
// src/net/event_loop_kqueue.cpp   (compiled on macOS)
#include "net/event_loop.hpp"

#include <sys/event.h>
#include <sys/time.h>
#include <unistd.h>

#include <unordered_map>
#include <vector>

namespace kestrel::net {

struct EventLoop::Impl {
    int kq = -1;
    std::unordered_map<int, Callback> callbacks;
};

EventLoop::EventLoop() : impl_(std::make_unique<Impl>()) {
    impl_->kq = ::kqueue();
}
EventLoop::~EventLoop() {
    if (impl_->kq >= 0) ::close(impl_->kq);
}

static void add_event(int kq, int fd, short filter) {
    struct kevent kev;
    EV_SET(&kev, fd, filter, EV_ADD | EV_CLEAR, 0, 0, nullptr);
    ::kevent(kq, &kev, 1, nullptr, 0, nullptr);
}

void EventLoop::add(int fd, Watch what, Callback cb) {
    impl_->callbacks[fd] = std::move(cb);
    if (static_cast<int>(what) & static_cast<int>(Watch::Read))  add_event(impl_->kq, fd, EVFILT_READ);
    if (static_cast<int>(what) & static_cast<int>(Watch::Write)) add_event(impl_->kq, fd, EVFILT_WRITE);
}

void EventLoop::remove(int fd) {
    impl_->callbacks.erase(fd);
    // Removing the events is best-effort; closing the fd auto-removes them.
}

void EventLoop::poll(int timeout_ms) {
    struct kevent events[64];
    struct timespec ts;
    struct timespec* tsp = nullptr;
    if (timeout_ms >= 0) {
        ts.tv_sec  = timeout_ms / 1000;
        ts.tv_nsec = (timeout_ms % 1000) * 1'000'000;
        tsp = &ts;
    }
    int n = ::kevent(impl_->kq, nullptr, 0, events, 64, tsp);
    for (int i = 0; i < n; ++i) {
        int fd = static_cast<int>(events[i].ident);
        Watch w = (events[i].filter == EVFILT_READ) ? Watch::Read : Watch::Write;
        auto it = impl_->callbacks.find(fd);
        if (it != impl_->callbacks.end()) it->second(w);
    }
}

}  // namespace kestrel::net
```

On Linux, the equivalent over `epoll` is structurally the same — `epoll_create1`
instead of `kqueue`, `epoll_ctl` instead of `kevent` registration,
`epoll_wait` instead of `kevent` polling. The two `.cpp` files compile
exclusively (CMake selects one by platform).

`EV_CLEAR` puts the kqueue into **edge-triggered** mode: a notification
fires when the readiness changes, not while it persists. Edge-triggered is
what epoll's `EPOLLET` does too, and it forces a particular discipline on
your callbacks: when readable, read until `EAGAIN`; when writable, write
until `EAGAIN`. We codify that discipline in Chapter 16.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Edge-triggered loops are deadlock-prone if you read once and stop. If 1KB
arrives, you `read` 1KB, the buffer's level drops; the kernel notifies you
once and does not notify you again until the buffer level *crosses* the
readable threshold again. If you stop reading at 1KB but 4KB had arrived,
the remaining 3KB sits in the buffer indefinitely. The rule: in
edge-triggered mode, drain the fd to `EAGAIN` on every callback.

</div>

---

## Making a socket non-blocking

A socket is by default in blocking mode. Make it non-blocking with
`fcntl(F_GETFL)` + `fcntl(F_SETFL, ... | O_NONBLOCK)`. Wrap it in a tiny
helper so the rest of Kestrel doesn't think about flags:

```cpp
// src/net/fd_utils.hpp
#pragma once
#include <expected>
#include "net/error.hpp"

namespace kestrel::net {

std::expected<void, NetworkError> set_non_blocking(int fd);

}  // namespace kestrel::net
```

```cpp
// src/net/fd_utils.cpp
#include "net/fd_utils.hpp"
#include <fcntl.h>
#include <cstring>

std::expected<void, kestrel::net::NetworkError>
kestrel::net::set_non_blocking(int fd) {
    int flags = ::fcntl(fd, F_GETFL, 0);
    if (flags < 0) return std::unexpected(NetworkError{ std::string("fcntl GETFL: ") + std::strerror(errno), errno });
    if (::fcntl(fd, F_SETFL, flags | O_NONBLOCK) < 0)
        return std::unexpected(NetworkError{ std::string("fcntl SETFL: ") + std::strerror(errno), errno });
    return {};
}
```

The `Listener` from Chapter 14 should call this on both the listener fd
and every accepted client fd. From the application's perspective, after
the flip:

- `read` may return `-1` with `errno == EAGAIN` or `EWOULDBLOCK`. That's
  "no data right now, try again later"; the event loop will tell you
  when.
- `write` may also return `-1` with `EAGAIN`/`EWOULDBLOCK` or a smaller
  byte count than requested. That's "the send buffer is full"; queue the
  rest and re-watch the fd for writability.
- `accept` may return `-1` with `EAGAIN` when there's no pending
  connection — same idea.

Together, these turn every I/O call into a *cooperative* operation. Your
code does the work in front of it and yields to the loop the moment the
kernel says "no more."

<div class="callout callout-algo">

📐 **Algorithm**

**The reactor.** Maintain a set of file descriptors of interest. Block in
one syscall (`epoll_wait` / `kevent`) until any one becomes ready. For each
ready fd, dispatch a callback registered when the fd was added. Callbacks
do non-blocking I/O until they hit `EAGAIN`, then return. Loop forever.
Throughput is limited by the number of ready fds the kernel can deliver per
syscall (a few dozen by default — we read 64 at a time above) and by how
much work each callback does.

</div>

---

## Connection state machines

A blocking connection reads bytes into its parse buffer until a frame
arrives, then handles the frame. With non-blocking I/O, that loop is no
longer a function — it is a *state machine* that resumes whenever the
event loop says "more bytes arrived" or "the socket is writable again."

The minimum state per connection is:

- The fd (in a `FileDescriptor`).
- A **read buffer** of bytes accumulated since the last successful parse.
- A **write buffer** of bytes the server wants to send but the kernel hasn't
  yet accepted.
- A reference to the `EventLoop` and a self-pointer so callbacks can find
  the connection back.

A first cut (Chapter 16 will refine this when it plugs in the RESP parser):

```cpp
// src/net/connection.hpp  (expanded)
#pragma once

#include <vector>
#include <cstdint>

#include "net/event_loop.hpp"
#include "net/file_descriptor.hpp"

namespace kestrel::net {

class Connection {
public:
    Connection(EventLoop& loop, FileDescriptor fd);
    ~Connection();

    void on_readable();
    void on_writable();

    // Append bytes to the write buffer; the loop will eventually drain them.
    void send(std::span<const std::uint8_t> bytes);

private:
    void try_flush();

    EventLoop&            loop_;
    FileDescriptor        fd_;
    std::vector<std::uint8_t> in_;   // bytes read; pending parse
    std::vector<std::uint8_t> out_;  // bytes queued; pending write
    std::size_t out_off_ = 0;        // bytes of `out_` already written
};

}  // namespace kestrel::net
```

`on_readable` drains the socket into `in_` and hands `in_` to the parser
(Chapter 16). `on_writable` drains `out_` to the socket until it returns
`EAGAIN`. The `Connection` registers itself with the `EventLoop` in its
constructor and deregisters in its destructor — RAII applied to the
loop registration, exactly the same pattern as the file descriptor itself.

A pseudo-loop fragment for `on_readable`:

```cpp
void Connection::on_readable() {
    std::uint8_t scratch[4096];
    for (;;) {
        ssize_t n = ::read(fd_.get(), scratch, sizeof scratch);
        if (n > 0) { in_.insert(in_.end(), scratch, scratch + n); continue; }
        if (n == 0) { /* EOF: peer closed cleanly. drop this connection. */ break; }
        if (errno == EINTR) continue;
        if (errno == EAGAIN || errno == EWOULDBLOCK) break;  // edge-triggered: drained
        // any other errno: log and drop the connection
        break;
    }
    // hand `in_` to the parser; on a complete frame, generate a response
    // into `out_` and call try_flush() to drain.
}
```

The shape of `on_writable` is dual — drain `out_` until `EAGAIN`, then
return. If `out_` empties, ask the loop to *stop watching for writability*
on this fd (otherwise it'll keep firing on an empty write buffer). If
`out_` doesn't empty, keep watching.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

A real, subtle bug: removing an fd from the loop while a callback for that
fd is being dispatched. If your dispatch loop iterates over events and one
callback closes its own fd (calling `remove`), the next event in the same
batch *for the same fd* fires on a dead connection. Defensive structure:
the dispatch loop should be ready for a callback to invalidate other
pending events. Typical fix: a "to-delete" list applied at the end of the
batch.

</div>

---

## Tying it together

The server side now looks like (sketch):

```cpp
int main() {
    std::signal(SIGPIPE, SIG_IGN);

    auto listener_or = kestrel::net::Listener::bind(6379);
    if (!listener_or) { /* log and exit */ return 1; }

    kestrel::net::EventLoop loop;
    auto& listener = *listener_or;
    kestrel::net::set_non_blocking(listener.fd()).value();

    loop.add(listener.fd(), kestrel::net::Watch::Read, [&](auto) {
        for (;;) {
            auto c = listener.accept();
            if (!c) break;                                 // EAGAIN: done for now
            kestrel::net::set_non_blocking(c->get()).value();
            // construct a Connection that registers itself with the loop
            (void)new kestrel::net::Connection(loop, std::move(*c));   // simplified ownership
        }
    });

    loop.run();
}
```

That `new Connection(loop, std::move(*c))` is *deliberately* simplified — in
real Kestrel, connections live in an owned container indexed by fd (so we
can find one for `remove`), and they delete themselves out of it when they
close. Chapter 19, when threads enter, will revisit ownership of
`Connection` because the loop and the worker pool will both need to refer
to it.

<div class="callout callout-experiment">

🧪 **Experiment**

Run the event-loop echo server and connect with `nc localhost 6379` from
three terminals simultaneously. Type in each. Confirm all three are
echoed independently, with no thread per client. Now write a quick load
test: a tiny client that opens 1000 connections and sends a line on each.
A blocking-server version of this would crawl; the reactor version should
finish in under a second. The difference is the lesson.

</div>

---

## What the event loop bought you

The reactor turns "thousands of concurrent connections" from a thread-pool
problem into a single-thread, mostly-CPU-bound problem. The cost has
shifted: each connection now needs to maintain its parse state across
loop iterations, and the parse code must tolerate "any byte may arrive at
any time" — which is exactly the property the Chapter 6 incremental
parser was built for. Chapter 16 connects those two pieces into Kestrel's
actual request-handling path.

Topics deferred to later parts:

- **Timers** — TTL sweeps and snapshot rotation want to fire on a schedule.
  The loop integrates timer events through the same `kevent`/`epoll_wait`
  with a timeout; Chapter 18 will wire it in.
- **Cross-thread wakeup** — when a worker thread (Part 6) wants to tell the
  loop "I have results to send," it cannot just touch shared state and
  hope. The pattern is `eventfd` (Linux) or a pipe (macOS), watched by the
  loop. Chapter 19's job.
- **Multiple loops, one per core** — for very high throughput, run one
  event loop per CPU core, each owning a slice of the connections. Chapter
  21 mentions it in the capstone.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here is my `EventLoop` (`net/event_loop.hpp`, the per-platform `.cpp`s)
> and the updated `Connection` with `on_readable` / `on_writable`. Review:
> (1) is the PIMPL idiom set up correctly — am I forward-declaring `Impl`
> and putting the destructor in the right place? (2) is the edge-triggered
> reading code draining to `EAGAIN` correctly? (3) am I handling the
> "callback closes its own fd" case safely? (4) is `std::function` the
> right callback type, or would a non-allocating alternative be worth the
> complexity?

</div>

---

## Where this leaves Kestrel

`src/net/` now has the four pieces a server needs: `FileDescriptor`,
`Listener`, `Connection`, `EventLoop`. They depend on POSIX, expose typed
errors, and are RAII-safe. Together they form the substrate; the
*application* logic — what to do when a parsed command arrives — is the
next chapter.

Chapter 16 plugs the RESP parser into `Connection::on_readable`, runs
parsed commands through the keyspace from Part 3 (via a small dispatch
function), serializes responses with the writer from Chapter 6, and queues
them into `Connection::out_`. At that point Kestrel becomes a real Redis
server: byte arrives, parsed, dispatched, response queued, written when
writable. The Part 5 picture is complete.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and the multi-client echo demo running:

```bash
cmake --build src/build
./src/build/echo_server &
nc localhost 6379   # type from here
nc localhost 6379   # ...and from another terminal at the same time
```

Both terminals should be able to echo concurrently. You should be able to:

- Explain the difference between blocking and non-blocking I/O at the
  kernel level.
- Describe what `kqueue`/`epoll` give you and the three operations the
  loop needs (add, remove, wait).
- Defend PIMPL for `EventLoop` in terms of header hygiene and platform
  separation.
- Define edge-triggered vs level-triggered and the drain-to-EAGAIN rule
  that follows from edge-triggered.
- Sketch how a `Connection` keeps its parse and write state across loop
  iterations.

Hand it over:

> Review my Chapter 15 event loop and `Connection` state machine. Are my
> edge-triggered reads draining correctly? Is the PIMPL pattern set up
> idiomatically? Any places where a callback closing its own fd could
> crash the loop? Is `set_non_blocking` set everywhere it should be?

When that's green, Chapter 16 ties protocol parsing into the loop.

</div>
