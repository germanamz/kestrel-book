# Tying Threads to the Loop

Chapter 18 left a gap. The worker pool runs commands on its threads; the
event loop runs on its own thread; and a worker that finishes a command
needs to *tell* the loop "I have a response ready — please write it to
the connection." We sketched that step as `io_loop_.post_to_main(...)`
and moved on. This chapter is the implementation.

The bridge is two things together: a **cross-thread wakeup** primitive
that lets a worker pull the event loop out of `epoll_wait`/`kevent`, and
a **mailbox** queue the worker drops continuations into for the loop to
pick up. The C++ topic alongside is **cross-thread ownership of
connections** — a `Connection` may have a pending task on a worker
*after* its socket has closed, and a careless ownership scheme leaves the
worker dereferencing freed memory. `std::shared_ptr` is the right
hammer, used carefully.

By the end, Kestrel's I/O thread and worker threads coordinate cleanly:
every byte on the wire travels through a single owner, every command
runs on a worker, every response gets back to the right connection's
buffer, and TSan stays silent. Part 6 — and Kestrel's server — is then
complete.

---

## Why the loop needs a wakeup mechanism

`epoll_wait` and `kevent` block until *one of the file descriptors they
are watching* becomes ready. Worker threads have no such fd — they live
inside the process and want to say "hey, I produced a response,
look at the queue." If they just push to a queue, the loop won't notice
until some socket event happens, which might be never.

The portable solution is to give the loop *another* fd, owned by the
loop, that workers can write to. Writing to that fd makes it readable;
the loop wakes up on its next iteration and drains the mailbox. The
content of the bytes written to the fd doesn't matter — the kernel
notification is what does the work.

There are three common implementations:

- **`eventfd`** on Linux — a single fd whose value is a counter. Writes
  add to the counter; reads return and reset it. Cheap.
- **A pipe** on macOS / BSD — `pipe2(SOCK_CLOEXEC)`. Two fds; write to
  one, read from the other.
- **`kevent` with `EVFILT_USER`** on macOS — a user-triggered event,
  natively in kqueue. We use this where available.

Kestrel hides the choice behind a `Notifier`:

```cpp
// src/concurrency/notifier.hpp
#pragma once

#include "net/file_descriptor.hpp"

namespace kestrel::concurrency {

class Notifier {
public:
    Notifier();
    ~Notifier();
    Notifier(const Notifier&) = delete;
    Notifier& operator=(const Notifier&) = delete;

    int  read_fd() const;     // fd to register with the EventLoop
    void notify();            // safe to call from any thread
    void drain();              // call on read_fd readiness

private:
    // Platform-specific bits behind PIMPL (like EventLoop).
    struct Impl;
    std::unique_ptr<Impl> impl_;
};

}  // namespace kestrel::concurrency
```

On Linux:

```cpp
// src/concurrency/notifier_eventfd.cpp  (compiled on Linux)
#include "concurrency/notifier.hpp"

#include <sys/eventfd.h>
#include <unistd.h>
#include <cstdint>

namespace kestrel::concurrency {

struct Notifier::Impl {
    kestrel::net::FileDescriptor fd;
};

Notifier::Notifier() : impl_(std::make_unique<Impl>()) {
    int f = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    impl_->fd = kestrel::net::FileDescriptor(f);
}
Notifier::~Notifier() = default;

int Notifier::read_fd() const { return impl_->fd.get(); }

void Notifier::notify() {
    std::uint64_t one = 1;
    (void)::write(impl_->fd.get(), &one, sizeof one);   // ignore EAGAIN
}

void Notifier::drain() {
    std::uint64_t v;
    while (::read(impl_->fd.get(), &v, sizeof v) > 0) {}   // drain
}

}  // namespace kestrel::concurrency
```

On macOS we'd use a pipe pair under the same interface. The PIMPL pattern
from Chapter 15's `EventLoop` lets us keep the header platform-clean.

The fd lives forever (one notifier per loop). The `notify()` write is
non-blocking; if the kernel's small `eventfd` counter is full (extremely
unlikely; `eventfd` holds a 64-bit counter), we drop the wake-up, which
is fine because someone else's notify will catch it.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`eventfd` semantics with `EFD_NONBLOCK`: a read on an unsignalled `eventfd`
returns `EAGAIN`. We don't error on `EAGAIN` in `drain()`; we just stop.
Forgetting that and treating `EAGAIN` as an error would crash the loop on
every spurious wakeup. The discipline is the same one Chapter 15
introduced for sockets, applied to a kernel-managed counter fd.

</div>

---

## The mailbox

Workers don't just want to wake the loop — they want to *tell it what to
do*. The companion to the notifier is a thread-safe queue of
continuations:

```cpp
// src/concurrency/mailbox.hpp
#pragma once

#include <functional>
#include <mutex>
#include <vector>

namespace kestrel::concurrency {

class Mailbox {
public:
    using Task = std::function<void()>;

    void post(Task t);
    // Returns all pending tasks and clears the mailbox. Called from the
    // EventLoop thread, typically right after Notifier::drain().
    std::vector<Task> drain();

private:
    std::mutex        mu_;
    std::vector<Task> q_;
};

}  // namespace kestrel::concurrency
```

```cpp
void Mailbox::post(Task t) {
    std::lock_guard guard(mu_);
    q_.push_back(std::move(t));
}

std::vector<Mailbox::Task> Mailbox::drain() {
    std::lock_guard guard(mu_);
    std::vector<Task> out;
    out.swap(q_);
    return out;
}
```

The mailbox uses `std::vector<Task>` because we want to drain *all*
tasks at once on each loop wakeup — avoid the "wake up, do one,
wake up, do one" pattern. `swap` into a fresh local vector lets the
critical section be O(1), and the actual task execution happens outside
the lock.

The combined dance — worker thread posts to mailbox, then writes one
byte to the notifier; loop sees the notifier fire, calls
`notifier.drain()`, calls `mailbox.drain()`, and runs every continuation
— is the entire cross-thread protocol.

```cpp
// In the event loop's setup:
loop.add(notifier.read_fd(), net::Watch::Read, [&](auto) {
    notifier.drain();
    for (auto& t : mailbox.drain()) t();
});

// From a worker thread:
mailbox.post([conn, response] {
    conn->out_.insert(conn->out_.end(), response.begin(), response.end());
    conn->try_flush();
});
notifier.notify();
```

That `conn->out_` mutation now happens *on the loop thread*, because
that's the thread the mailbox tasks run on. The single-owner-per-buffer
rule from Chapter 16 is preserved. The cross-thread plumbing is exactly
the notifier + mailbox pair.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

This is exactly `postMessage` from a worker to the main thread in Node /
the browser — the message goes into a queue, the runtime wakes up the
event loop, the loop dispatches the message. C++ doesn't have a built-in
"main thread post"; we built our own out of `eventfd` + a queue. After
this chapter, that postMessage primitive will feel inevitable.

</div>

---

## Cross-thread ownership: when does a `Connection` die?

The loop creates a `Connection` when a client arrives. The connection is
the I/O thread's resource — only it touches `out_`, only it calls
`try_flush`. But a worker thread may still be *holding a reference* to
it because it has a pending task ("execute this command and post a
response"). What happens if the client disconnects while the worker is
mid-execution?

The naive design — the loop owns the `Connection` via `unique_ptr`,
workers hold raw pointers — has a fatal flaw: when the loop destroys the
connection, the worker's raw pointer dangles. By the time the worker
posts a continuation that touches `out_`, the connection is freed.

The clean solution is **shared ownership via `std::shared_ptr`**:

```cpp
std::unordered_map<int, std::shared_ptr<Connection>> conns;
```

Each pending task captures a `std::shared_ptr<Connection>` by value:

```cpp
mailbox.post([conn = std::shared_ptr<Connection>(shared_from_this()),
              response = std::move(response)] {
    conn->out_.insert(...);
    conn->try_flush();
});
```

Now even if the loop removes the connection from `conns` while a worker
is mid-task, the task's captured `shared_ptr` keeps the connection alive
until the task runs. When the task finishes (and the worker releases its
copy), the last `shared_ptr` is gone and the connection is destroyed.

`shared_from_this()` requires the class to inherit from
`std::enable_shared_from_this<T>`:

```cpp
class Connection : public std::enable_shared_from_this<Connection> {
    // ...
};
```

This gives the class a `shared_from_this()` method that returns a
`shared_ptr<Connection>` to itself — but only if the object is already
managed by a `shared_ptr`. If you accidentally construct a `Connection`
on the stack or via `make_unique`, the method throws
`std::bad_weak_ptr`. The compile-time signal: you must construct with
`std::make_shared<Connection>(...)`.

Critical: the *handler* that runs on the worker must also know the
connection might be dead, in the *application* sense — the client may
have disconnected. So the connection carries a `bool closed_` and tasks
check it before mutating buffers:

```cpp
mailbox.post([conn, response = std::move(response)] {
    if (conn->closed_) return;     // client disconnected; drop the response
    conn->out_.insert(...);
    conn->try_flush();
});
```

`shared_ptr` keeps the *memory* alive; `closed_` carries the *logical*
state. They are different things; both matter.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`closed_` is touched from both the I/O thread (sets it on socket close)
and the worker thread (reads it before mutating). That is a data race
unless we mark it `std::atomic<bool>` or protect it with a mutex.
Kestrel uses `std::atomic<bool>` — the predicate is one byte, one read
and one write, no other state to coordinate. Mark it `noexcept`-friendly:

```cpp
std::atomic<bool> closed_{false};
```

</div>

---

## `std::atomic<std::shared_ptr<T>>`

C++20 added `std::atomic<std::shared_ptr<T>>` — an atomic shared pointer
specifically for the case where you need to swap one shared_ptr for
another from multiple threads. Use cases include:

- **Copy-on-write configuration**: the loop holds an
  `atomic<shared_ptr<Config>>` to the current config; a refresh thread
  prepares a new `Config` and `atomic::exchange`s it in. Readers see
  either the old or the new, never a torn mix.
- **Lock-free root pointer**: a tree-shaped structure where the root is
  swapped atomically while readers may hold older versions.

Kestrel will not need `atomic<shared_ptr>` in the basic server build —
all our cross-thread sharing of the keyspace is mutex-protected via
`ShardedStore`. We mention it because it is the C++20 answer to "I need
to share a `shared_ptr` between threads safely, and I want lock-free
read-side" and you will hear about it at every C++ conference for the
next decade.

The cheaper alternative when you have one writer and many readers is the
pair of `shared_ptr::store(...)` and `shared_ptr::load()` via
`std::atomic_store`/`atomic_load` (pre-C++20). Both forms internally use
atomic CAS loops.

---

## The complete server loop

Putting the threads together:

```cpp
int main(int argc, char** argv) {
    std::signal(SIGPIPE, SIG_IGN);

    auto wal_or = persistence::Wal::open("/var/lib/kestrel/kestrel.wal",
                                         persistence::FsyncMode::EverySecond);
    if (!wal_or) return 1;

    auto kv_or  = persistence::recover("/var/lib/kestrel");
    if (!kv_or) return 1;

    store::ShardedStore store;
    // (load `*kv_or` into `store` shards — small adapter not shown)

    net::EventLoop      loop;
    concurrency::Notifier notifier;
    concurrency::Mailbox  mailbox;
    concurrency::WorkerPool pool(std::thread::hardware_concurrency());

    loop.add(notifier.read_fd(), net::Watch::Read, [&](auto) {
        notifier.drain();
        for (auto& t : mailbox.drain()) t();
    });

    auto listener_or = net::Listener::bind(6379);
    if (!listener_or) return 1;
    net::set_non_blocking(listener_or->fd()).value();

    std::unordered_map<int, std::shared_ptr<net::Connection>> conns;

    loop.add(listener_or->fd(), net::Watch::Read, [&](auto) {
        for (;;) {
            auto c = listener_or->accept();
            if (!c) break;
            net::set_non_blocking(c->get()).value();
            int cfd = c->get();
            auto conn = std::make_shared<net::Connection>(
                loop, std::move(*c), store, *wal_or, pool, notifier, mailbox);
            conns[cfd] = conn;
            // Connection registers its own loop callbacks in its constructor.
        }
    });

    loop.run();
}
```

Connections, when their socket closes, remove themselves from `conns` —
which is the last shared_ptr the loop holds — and any pending worker task
keeps the connection alive for the duration of its closure. When that
task finishes, the connection's destructor runs (in the worker, or back
on the loop, depending on who released last). RAII handles the rest.

<div class="callout callout-experiment">

🧪 **Experiment**

Stress-test the threading. Write a small client that opens 100
connections in parallel, each issuing 10,000 random commands (`SET`,
`GET`, `DEL` against a key pool of 1000). Run Kestrel with
`hardware_concurrency()` worker threads, then with `1` worker (effectively
serialized), then with `2 * hardware_concurrency()` (over-subscribed).
Measure ops/second for each. Predict where the sweet spot lands; usually
it's close to `hardware_concurrency()`. Then run the same test under
TSan — confirm no reports. *That clean TSan run is the actual capstone of
Part 6.*

</div>

---

## What's left for "production"

Kestrel is now a real, threaded Redis-compatible server, but a serious
deployment would add:

- **A timer wheel for TTL sweeps.** Currently you'd run the keyspace's
  expiry sweep on a schedule — register a timer event with the loop, fire
  every 100ms. The loop's `poll(timeout_ms)` already accepts a timeout,
  so this is one more callback.
- **Snapshot scheduling.** Trigger snapshots in the worker pool on a
  cadence (every N commands or every M minutes), with the rotation
  protocol from Chapter 13. Make sure snapshot work doesn't block the
  WAL — Chapter 13's TODO.
- **Replication and clustering.** Out of scope for the book.

Chapter 20 builds `kvcli`, the client side of the protocol, so you can
talk to your server without `redis-cli`. Chapter 21 takes the whole thing
under a benchmark and a profiler and closes the loop on "did this design
actually deliver?"

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's my Chapter 19 cross-thread plumbing: `concurrency/notifier.{hpp,cpp}`,
> `concurrency/mailbox.{hpp,cpp}`, and the updated `Connection` with
> `enable_shared_from_this`. Review: (1) is `eventfd` configured
> correctly (non-blocking, CLOEXEC, etc.)? (2) is the
> `notifier.drain()` then `mailbox.drain()` order correct, or could a
> task race in between? (3) is `closed_` correctly atomic, and does the
> "check `closed_` at the top of every task" idiom cover all paths?
> (4) am I capturing the `shared_ptr<Connection>` by value in every
> task closure, with no aliasing back to raw pointers?

</div>

---

## Where this leaves Kestrel

Part 6 is done. Kestrel now has:

- A reactor I/O thread (Chapter 15).
- A worker pool (Chapter 18) with a sharded keyspace.
- A wakeup mechanism that ties them (this chapter).
- A clean ownership model via `shared_ptr<Connection>` that survives
  cross-thread tasks.
- TSan as a regression test that catches the bugs you cannot see.

Part 7 closes the book. Chapter 20 implements `kvcli` — the client
binary, wire-compatible enough with `redis-cli` to talk to our server.
Chapter 21 benchmarks, profiles, and reflects on what to do next:
pipelining, observability, the optional pub/sub system, and the
trade-offs that produced the design you now have.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build under both sanitizer profiles:

```bash
cmake --build src/build && ctest --test-dir src/build --output-on-failure
cmake --build src/build-tsan && ctest --test-dir src/build-tsan --output-on-failure
```

Stress test the server:

```bash
./src/build/kvd &
# parallel-clients.sh: 100 redis-clis × 10k operations each
./stress/parallel-clients.sh
kill %1
```

You should be able to:

- Explain why `epoll_wait` blocked threads need a wake-up fd to receive
  work from other threads.
- Defend `shared_ptr<Connection>` ownership across thread boundaries and
  explain how it interacts with `closed_`.
- Read TSan output and locate the offending unsynchronized access.
- Sketch how a worker thread's response makes it back to the I/O
  thread's `try_flush`.

Hand it over:

> Walk through my Chapter 19 implementation end to end. Are there
> reasons a notification could be missed (e.g., between `mailbox.post`
> and `notifier.notify`)? Is the connection lifetime sound under all
> shutdown orders — server exit, client disconnect, worker still running?
> Suggest one observability hook I'd want before deploying this.

When that's green, Part 7 begins — and we write the client.

</div>
