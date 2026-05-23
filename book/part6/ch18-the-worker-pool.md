# The Worker Pool

Chapter 17 gave us the vocabulary; this chapter cashes it in. Kestrel's
event loop is fast, but every command's keyspace work — hash lookup,
value mutation, expiry checks, WAL append — runs on the same thread that
reads bytes and writes them. A single slow command blocks all other
connections from being serviced. The fix is the classic **worker thread
pool**: a small set of threads that pull commands off a shared queue and
execute them in parallel.

We will build this in three steps. First, a **task queue** with a mutex
and condition variable — the textbook design, correct and easy to read.
Second, the **worker pool** that owns N threads pulling from that queue.
Third, a **keyspace sharding** scheme that lets multiple workers operate
on the same logical store at once with bounded contention.

The C++ topics are condition-variable-based work distribution, mutex
choice (and abuse), thread-pool ownership patterns, and the design
trade-off between a single global lock and N per-shard locks.

By the end, Kestrel has a configurable pool of worker threads
contending only on per-shard locks, the event loop hands off command
execution to them, and the whole thing should be clean under
ThreadSanitizer.

---

## The simplest queue that works

The first cut is one queue, one mutex, one condition variable. Pushes
take the lock, append, and signal. Pops take the lock, wait until the
queue is non-empty, and consume.

```cpp
// src/concurrency/task_queue.hpp
#pragma once

#include <condition_variable>
#include <functional>
#include <mutex>
#include <optional>
#include <queue>

namespace kestrel::concurrency {

class TaskQueue {
public:
    using Task = std::function<void()>;

    void push(Task t);
    // Returns nullopt iff the queue is shutting down.
    std::optional<Task> pop();
    void shutdown();

private:
    std::mutex              mu_;
    std::condition_variable cv_;
    std::queue<Task>        q_;
    bool                    closed_ = false;
};

}  // namespace kestrel::concurrency
```

```cpp
// src/concurrency/task_queue.cpp
#include "concurrency/task_queue.hpp"

namespace kestrel::concurrency {

void TaskQueue::push(Task t) {
    {
        std::lock_guard guard(mu_);
        q_.push(std::move(t));
    }
    cv_.notify_one();
}

std::optional<TaskQueue::Task> TaskQueue::pop() {
    std::unique_lock guard(mu_);
    cv_.wait(guard, [&] { return closed_ || !q_.empty(); });
    if (q_.empty()) return std::nullopt;     // closed and drained
    auto t = std::move(q_.front());
    q_.pop();
    return t;
}

void TaskQueue::shutdown() {
    {
        std::lock_guard guard(mu_);
        closed_ = true;
    }
    cv_.notify_all();
}

}  // namespace kestrel::concurrency
```

Walk it carefully — every line earns its place.

**Lock dropped before `notify_one`.** Notifying inside the lock works but
wakes a waiter that then immediately blocks on the same mutex you still
hold. Releasing the lock first lets the waker enter the critical section
without re-parking. Microbenchmarks have shown both forms; for this code,
the release-then-notify form is the common idiom and we use it.

**`unique_lock` for the consumer.** Required by `condition_variable::wait`,
which needs to atomically unlock and re-lock the mutex around the parked
state.

**The wait predicate.** `[&] { return closed_ || !q_.empty(); }` covers
both wakeup cases: a new task arrived (`!q_.empty()`) or the queue is
shutting down (`closed_`). The wait loop deals with spurious wakeups
automatically.

**`notify_all` on shutdown.** All workers must wake, see the closed flag,
return `nullopt`, and exit. A `notify_one` would wake exactly one worker;
the rest would sit waiting forever.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`Promise.allSettled` plus a queue of `Promise`s is the JS rough analog.
`asyncio.Queue` in Python is closer. The C++ version is the same shape
made explicit: a thread-safe FIFO with a "wait for an item" primitive. The
single most important detail that doesn't translate from those languages
is the *predicate-based wait* — spurious wakeups are real in C++ and
your `wait` must always have a predicate.

</div>

---

## The pool itself

The pool starts N threads, each running a loop that pops and executes.
Shutdown is cooperative: close the queue, join all threads. RAII handles
the shutdown automatically.

```cpp
// src/concurrency/worker_pool.hpp
#pragma once

#include <thread>
#include <vector>

#include "concurrency/task_queue.hpp"

namespace kestrel::concurrency {

class WorkerPool {
public:
    explicit WorkerPool(unsigned n);
    ~WorkerPool();
    WorkerPool(const WorkerPool&) = delete;
    WorkerPool& operator=(const WorkerPool&) = delete;

    template <class F>
    void post(F&& f) { queue_.push(std::function<void()>(std::forward<F>(f))); }

private:
    TaskQueue                 queue_;
    std::vector<std::jthread> threads_;
};

}  // namespace kestrel::concurrency
```

```cpp
// src/concurrency/worker_pool.cpp
#include "concurrency/worker_pool.hpp"

namespace kestrel::concurrency {

WorkerPool::WorkerPool(unsigned n) {
    threads_.reserve(n);
    for (unsigned i = 0; i < n; ++i) {
        threads_.emplace_back([this](std::stop_token) {
            while (auto t = queue_.pop()) (*t)();
        });
    }
}

WorkerPool::~WorkerPool() {
    queue_.shutdown();
    // jthread destructors call request_stop() then join(); since we already
    // closed the queue, every pop() returns nullopt and the workers exit.
}

}  // namespace kestrel::concurrency
```

The destructor relies on `jthread`'s automatic join — once the queue is
closed, every worker finishes its current task, sees `nullopt` from the
next `pop`, exits, and the `jthread` destructor joins. The `stop_token`
is unused because we drive shutdown through the queue's close flag; you
could equally drive it through the token, but doing both is redundant.

`post` takes a callable by *forwarding reference* and constructs a
`std::function<void()>` from it. The forwarding (`F&&` + `std::forward`)
lets the caller pass either an lvalue or an rvalue without an extra copy.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

The lambda captured inside `post` may capture references to local
variables that don't outlive the task's execution. A common bug:

```cpp
for (auto& conn : connections) {
    pool.post([&] { handle(conn); });   // `conn` may be destroyed first
}
```

If `connections` is rebuilt or any element moves, the captured reference
dangles. The mechanical fix is to capture by value (or by `shared_ptr`)
exactly what the task needs. Kestrel's command tasks will capture a
`shared_ptr<Connection>` so the connection cannot vanish under the
worker — Chapter 19's topic.

</div>

---

## Connecting the pool to the event loop

The event loop from Chapter 15 currently calls `Connection::dispatch`
inline. With a worker pool, the dispatch path changes:

```cpp
// (sketch — Chapter 19 will replace this with a cleaner shape)
void Connection::parse_and_dispatch() {
    // ... parse loop produces a Message ...
    pool_.post([this, msg = std::move(*r)] {
        this->execute(msg);     // hash table mutation + WAL append
    });
}

void Connection::execute(const protocol::Message& msg) {
    // 1. WAL append (thread-safe due to O_APPEND).
    wal_.append(encode(msg));
    // 2. Mutate the keyspace.
    auto response = dispatch_command(kv_, msg);
    // 3. Hand the response back to the I/O thread.
    io_loop_.post_to_main([this, response = std::move(response)] {
        this->out_.insert(this->out_.end(), response.begin(), response.end());
        this->try_flush();
    });
}
```

The handback to the I/O thread (`io_loop_.post_to_main`) is the piece
Chapter 19 implements; for now, observe the structure. The keyspace work
runs on a worker; the socket work runs on the I/O thread; nothing
mutates a `Connection`'s buffers except its owning I/O thread.

That single rule — **only the I/O thread touches `Connection` buffers** —
is the design constraint that keeps the per-connection state from needing
its own mutex. The shared mutable state is the keyspace, and that is what
we synchronize.

---

## Keyspace sharding

If all N workers contend on one mutex around the entire `HashTable`,
adding workers doesn't help — they queue on the lock. The fix is to
**shard**: split the keyspace into K independent partitions, hash each
key to pick its partition, and lock only that partition.

```cpp
// src/store/sharded_store.hpp
#pragma once

#include <array>
#include <mutex>
#include <functional>

#include "core/value.hpp"
#include "store/hash_table.hpp"

namespace kestrel::store {

class ShardedStore {
public:
    static constexpr std::size_t kShards = 16;   // power of two

    template <class F>
    auto with_shard(std::string_view key, F&& f) {
        auto& sh = shard_for(key);
        std::lock_guard guard(sh.mu);
        return std::forward<F>(f)(sh.table);
    }

    bool insert_or_assign(std::string key, Value v);
    bool erase(std::string_view key);
    std::optional<Value> get_copy(std::string_view key);  // see note below

private:
    struct Shard {
        std::mutex mu;
        HashTable  table;
    };
    std::array<Shard, kShards> shards_;

    Shard& shard_for(std::string_view key) {
        return shards_[std::hash<std::string_view>{}(key) & (kShards - 1)];
    }
};

}  // namespace kestrel::store
```

Two design decisions deserve attention.

**`with_shard(key, lambda)` instead of returning a locked reference.**
The shard lock must be held only while the caller is inside the closure.
Returning a reference would leak the lock through the caller's lifetime
and make it impossibly easy to forget to unlock. The lambda form pins the
lock to the closure's execution and is the C++ idiom for "do something
under a lock without exposing the lock to a caller."

**No `get` returning `Value*`.** The keyspace `HashTable::find` returns a
pointer into the table's storage. Once the shard lock releases, that
pointer dangles — another thread could have rehashed the table in
between. So the sharded API returns *owned* copies (or, for read-only
peeks, takes a callback under the lock). For Kestrel's case, the entire
command runs inside the closure on the worker thread, so the pointer is
fine — but a caller who tries to keep a pointer past the closure is
doomed. The API steers callers into the safe pattern.

`kShards = 16` is a balance: low enough to keep the per-shard memory
overhead reasonable (16 mutexes, 16 hash tables), high enough that a 16-
core machine sees almost no contention. Production Redis (when run in
multi-threaded mode) uses similar numbers; some go to 32 or 64. The
right number for Kestrel is "more than your typical concurrent worker
count, less than the cache contention starts to matter."

<div class="callout callout-algo">

📐 **Algorithm**

**Keyspace sharding.** Partition K independent hash tables. For each
operation, hash the key, map to one shard (`hash(key) mod K`, where K
is a power of two so we can use `&` instead of `%`), take that shard's
mutex, operate. Operations on distinct shards proceed in parallel.
Expected contention scales as `N_threads / K` for uniformly distributed
keys; worst-case (single hot key) is `N_threads` (no improvement over a
single-lock keyspace).

</div>

<div class="callout callout-experiment">

🧪 **Experiment**

Microbenchmark the lock contention. Pick four configurations:

1. One global mutex, 1 worker thread.
2. One global mutex, 8 worker threads.
3. 16-way sharding, 1 worker thread.
4. 16-way sharding, 8 worker threads.

Each runs the same workload: 1M `SET`s and 1M `GET`s on random keys
across, say, 100k key names (so the load is spread). Measure operations
per second.

Predict the ranking. Then run it. You should see #2 worse than #1
(contention costs more than the parallelism gains), and #4 dramatically
better than #2. The sharding makes the worker pool actually useful.

</div>

---

## A note on lock-free MPSC queues

`TaskQueue` uses a mutex. For a typical worker pool that is fine — the
critical section is short, contention is moderate, and the simplicity
is worth the constant-factor cost. But once the I/O thread is the sole
producer and there are N consumer workers, you can do better: a
**lock-free MPSC** (multi-producer single-consumer; here SP-MC, but the
algorithms are the same family) queue avoids the mutex entirely, using
only atomic operations.

We are *not* going to implement one in Kestrel; it is a half-chapter on
its own. The vocabulary:

- **Lock-free** means no thread holding the queue's data structures can
  block another from making progress. (Stronger: **wait-free**, where
  every operation finishes in a bounded number of steps regardless of
  contention.)
- **MPSC** queues are simpler than MPMC (multi-consumer); the SP variant
  is simpler still. The Ph.D.-style canonical reference is the
  Michael-Scott queue; for SP-MC, a ring buffer with atomic head/tail is
  enough.
- **Why bother**: when contention is high (tens of thousands of
  operations per second per worker), the mutex's `acquire` syscall
  becomes a real cost; lock-free reduces that to a few atomic CAS
  operations.

For Kestrel's workload — single I/O thread producing, a few workers
consuming — the mutex version is fine; we will benchmark in Chapter 21
and switch only if profiling justifies it. **Build the boring version
first; profile; complicate only on evidence.**

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Lock-free algorithms are infamous for the **ABA problem**: thread A reads
a value, thread B changes it to something else and back to the original,
A's compare-and-swap succeeds based on the "original" comparison but
operates on conceptually-stale state. The standard fix is tagged
pointers or hazard pointers. Bring this up only when implementing a real
lock-free structure; for Kestrel's mutex-protected queue, it does not
apply.

</div>

---

## TSan as a regression test

Every threaded change Kestrel adds gets a TSan run. Chapter 17 promised
this; Chapter 18 is when we cash in. The pool's tests should run under:

```bash
cmake -S src -B src/build-tsan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=thread -fno-omit-frame-pointer"
cmake --build src/build-tsan
ctest --test-dir src/build-tsan --output-on-failure
```

Tests to add:

- `worker_pool_test`: post 10,000 tasks to a pool of 4 workers; each
  task increments a `std::atomic<int>` shared counter; assert the final
  count equals 10,000. TSan must report no races.
- `sharded_store_test`: 4 threads, each performing 10,000 `insert`s,
  `erase`s, and `find`s on overlapping key ranges. Assert the final
  `size()` matches the expected count. TSan must be silent.

If TSan reports a race, **fix it**, don't suppress it. The "intermittent
unit test" pattern in concurrent code is almost always TSan correctly
diagnosing a real bug that happens to manifest rarely.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's my `TaskQueue`, `WorkerPool`, and `ShardedStore`. Review:
> (1) is my condition-variable wait predicate correct? am I covering
> spurious wakeups and the closed-and-drained case? (2) is the `with_shard`
> callback pattern correctly preventing callers from holding pointers
> past the lock? (3) is the destructor of `WorkerPool` race-free —
> specifically, can a task post happen *after* `shutdown()` was called?
> (4) which path in Kestrel's hot loop would I attack first to reduce
> mutex contention, given a profile that says the worker pool's queue
> is the bottleneck?

</div>

---

## Where this leaves Kestrel

The threading model is now in place:

- The **I/O thread** runs the event loop, parses commands, posts tasks to
  the pool, and writes responses on its own buffer.
- The **worker pool** runs N threads pulling tasks off the queue.
- The **sharded store** lets workers operate on distinct shards in
  parallel.
- TSan is the regression test for every concurrent change.

What is *not* done yet: Chapter 19's cross-thread wakeup. Right now, when
a worker finishes a command, it needs to put the response into the
connection's outbound buffer and ask the I/O thread to flush. We
sketched it as `io_loop_.post_to_main(...)` but did not implement it.
Chapter 19 builds that primitive — an `eventfd` or pipe watched by the
loop, plus a small queue of "things to do back on the main thread" —
and finally connects every piece end to end.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and tests green under *both* ASan/UBSan *and* TSan:

```bash
cmake --build src/build && ctest --test-dir src/build --output-on-failure
cmake --build src/build-tsan && ctest --test-dir src/build-tsan --output-on-failure
```

`worker_pool_test` and `sharded_store_test` exercise the new code. You
should be able to:

- Defend the lock-then-notify ordering in `TaskQueue::push`.
- Explain why the consumer uses `unique_lock` rather than `lock_guard`.
- Argue why `with_shard(key, lambda)` is safer than
  `lock_shard(key) -> RAII guard exposing the table`.
- Predict the contention behaviour of your sharded store on a random
  workload and on a single-hot-key workload.

Hand it over:

> Review my Chapter 18 worker pool and sharded store. Are there race
> conditions TSan didn't catch (e.g., on the shutdown path)? Is my
> sharding granularity right for a 16-core target? Suggest the smallest
> change that would let the pool perform LOCKLESSLY in the steady state
> when only one worker is active.

When that's clean, Chapter 19 ties the worker pool back to the event
loop.

</div>
