# Threads & the Memory Model

Kestrel as it stands runs everything — event loop, parsing, dispatch, WAL
appends — on a single thread. That single thread, on a single CPU core,
can already handle a remarkable amount of traffic, but it cannot use the
other cores in your machine. To get more throughput we need threads, and
the moment we add the second thread, we step into a part of C++ that is
genuinely alien even to engineers who have shipped concurrent JavaScript
or Python: the **memory model**.

This chapter is about thinking, not building. We are not going to add a
worker pool yet — that is Chapter 18's job. We are going to establish what
a *data race* is at the standard-language level, why simply "using a
mutex" or "using a lock" is too coarse a vocabulary in C++, and what
`std::atomic`, `memory_order`, `std::thread`, `std::jthread`, and the
**ThreadSanitizer** are *for*. By the end you will know why concurrent C++
is hard, what the language gives you for it, and what TSan will be telling
you when we start using it next chapter.

If you have done concurrent programming in other languages, the C++
specifics here will surprise you in two ways. First, *what counts as a
race* is more permissive than you may expect — atomic operations are not
the default; ordinary loads and stores from multiple threads to the same
variable are UB. Second, *what synchronization buys you* is more subtle —
locks and atomics establish a **happens-before** relationship between
threads that controls when one thread's writes become visible to another.
Both ideas land in this chapter.

---

## What is a thread, what is a process

A **process** is an isolated address space with one or more threads
running inside it. A **thread** is an independently scheduled instruction
stream sharing that address space with its siblings. Sharing the address
space is what makes threading powerful — passing a pointer between threads
is free — and dangerous — every variable not protected by synchronization
is a possible race.

C++11 added `std::thread`; C++20 added `std::jthread`, which is what you
should use unless you have a reason. `jthread`'s destructor *joins* the
thread automatically (a `thread` whose destructor runs while the thread is
still active calls `std::terminate`), and it supports cooperative
cancellation through a `std::stop_token`. It is the modern default.

```cpp
#include <thread>
#include <stop_token>
#include <chrono>

void worker(std::stop_token stop) {
    while (!stop.stop_requested()) {
        // do some unit of work
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}

int main() {
    std::jthread t(worker);                    // starts the thread
    std::this_thread::sleep_for(std::chrono::seconds(1));
    // t's destructor runs here: t.request_stop() then t.join()
}
```

That is the entire ceremony of "start a thread, stop it cleanly, wait for
it to exit." A bare `std::thread` with manual `join` is the older,
error-prone form; you'll see it in pre-C++20 code, but Kestrel uses
`jthread`.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

Node has one main thread plus a worker pool you do not directly drive;
Python has the GIL throttling true parallelism; both languages give
"concurrency" without "true parallelism" by default. C++ gives you true
threads, *with* shared memory, *without* the GIL safety blanket. Every
data structure you have ever shared between Node's main thread and a
worker thread (you used `MessagePort` or structured cloning) — in C++ you
share by pointer or reference and it just works *if* your synchronization
is correct, and corrupts memory silently otherwise. The cost is
discipline; the reward is using all your cores.

</div>

---

## The data race, formally

The C++ standard defines a **data race** as:

> Two threads access the same memory location, at least one of them
> writes, and at least one of them is not an atomic operation, and the
> accesses are not ordered by a happens-before relationship.

Every word of that definition does work, and every data race in your code
is **undefined behavior** at the language level — same severity as a
buffer overrun, in the standard's eyes. Three things follow.

**Reading a non-atomic variable from two threads is fine** if neither
writes. The contents are immutable; nobody can observe a torn or stale
value. This is the deep reason "shared read-only data" is one of the safe
concurrency patterns; immutability is the simplest synchronization
strategy.

**A single `std::atomic<int>` load from each thread is fine, even when one
thread is storing.** Atomic operations are the language's primitive
"synchronize-with" tool. They are slower than ordinary loads but defined.

**Two non-atomic accesses to the same variable, one of them a write, with
no synchronization between them, is UB.** No "you'll get *some* value"; no
"you'll see the old value or the new value." Anything is allowed,
including phantom values, including the optimizer assuming the race cannot
happen and removing your check.

That last fact is the one engineers from managed languages most often
miss. In Java, the JVM defines races as producing one of a small set of
observed values. In C++, it does not.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

A particularly nasty consequence: `bool stop_flag` toggled from one thread
and read from another **is UB** even though the variable is a single byte.
The compiler may hoist the read out of a loop:

```cpp
while (!stop_flag) { do_work(); }    // compiler: load once, jump forever
```

If `stop_flag` becomes `true` partway through, the loop never sees it,
because the compiler proved (under the assumption of no UB) that nothing
in `do_work()` modifies `stop_flag`, so the load needs to happen only
once. The fix is `std::atomic<bool>`, which forbids this optimization.
This is exactly what `std::jthread`'s `stop_token` does under the hood.

</div>

---

## `std::atomic` and the memory orders

`std::atomic<T>` is the C++ primitive for synchronized memory access.
Operations on it have **two** properties beyond just atomicity:

1. **Indivisibility**: the load or store happens as a whole. No torn
   reads, no half-updated values. Cheap on architectures with native
   atomic operations for `T` (most numeric types).
2. **Ordering**: the operation participates in the language's
   *happens-before* graph. This is the part you have to learn.

The default `std::atomic` operations use `memory_order_seq_cst`
(sequentially consistent) ordering — the strongest, most expensive, and
the right default. Under `seq_cst`, all atomic operations across all
threads appear to happen in *some single global total order*. The cost is
that the compiler and CPU must emit full memory barriers around each
atomic, which on x86 is cheap and on ARM is expensive.

```cpp
#include <atomic>

std::atomic<int> counter{0};

// In thread A:
counter.fetch_add(1, std::memory_order_seq_cst);

// In thread B:
int v = counter.load(std::memory_order_seq_cst);
```

For most code, **use sequential consistency** and stop worrying. The
hand-tuned weaker orderings (`acquire`, `release`, `relaxed`, `acq_rel`,
`consume`) exist for lock-free data structures where the perf math
genuinely justifies the complexity. We will need exactly two non-default
orderings in Kestrel, both for the MPSC queue in Chapter 18; everywhere
else, `seq_cst` is right.

The other three orderings, briefly:

- **`memory_order_acquire`** on a load: any reads/writes *after* this
  load (in program order) cannot be reordered to before it. Pairs with a
  release store to establish synchronizes-with.
- **`memory_order_release`** on a store: any reads/writes *before* this
  store cannot be reordered to after it. The "publish" side of a
  publish/consume pattern.
- **`memory_order_relaxed`**: no ordering, only atomicity. Use for things
  like statistics counters that don't synchronize anything else.

If those words don't immediately mean a picture in your head, you are in
the same place every engineer is when they meet the memory model for the
first time. The remedial reading is Herb Sutter's "atomic<> Weapons"
talks; the operational summary is: **start with `seq_cst`; only weaken
when a profiler says the atomic is your bottleneck.**

<div class="callout callout-algo">

📐 **Algorithm**

**Happens-before, intuitively.** Two threads' operations are ordered by
happens-before if one of: (a) they are in the same thread and one comes
before the other in program order; (b) a release-store in thread A
synchronizes-with an acquire-load in thread B reading the value the
release stored; (c) thread A's join is after thread B's exit; (d) the
chain is transitive. Anything not ordered by happens-before may appear to
either thread in any order. Synchronization primitives (mutex
lock/unlock, atomic acquire/release, condition-variable wait) exist to
extend the happens-before graph across threads.

</div>

---

## Mutexes: the workhorse

For 95% of Kestrel's concurrency, a mutex is the right tool. C++ has
several:

- **`std::mutex`** — basic exclusive lock. Cheap, correct, what you want.
- **`std::shared_mutex`** — many readers or one writer; useful when reads
  vastly outnumber writes and the critical sections are slow enough that
  the shared-lock overhead is worth it. Often a footgun; the constant
  factors are surprisingly bad and a plain mutex frequently wins.
- **`std::recursive_mutex`** — same thread can lock it twice. Almost
  always a sign of design confusion. Avoid.
- **`std::timed_mutex`** — `try_lock_for` with a timeout. Useful for
  deadlock-avoidance heuristics in higher-level systems.

`std::mutex` paired with the **RAII guard** `std::lock_guard` (or, for
condition variables, `std::unique_lock`) is what you reach for first:

```cpp
#include <mutex>

class Counter {
public:
    void increment() {
        std::lock_guard guard(mu_);
        ++count_;
    }
    int value() const {
        std::lock_guard guard(mu_);
        return count_;
    }
private:
    mutable std::mutex mu_;
    int count_ = 0;
};
```

Two things to absorb.

**The `mutable` keyword.** A `const` method (`value() const`) cannot
modify members — but locking the mutex *is* a modification to the mutex
object. `mutable` says "this member can be modified even from `const`
methods," which is exactly the right escape hatch for mutexes and other
"internal mechanism" state.

**`lock_guard` is RAII over the lock.** Constructor locks, destructor
unlocks. If the function throws, the destructor still runs, and the lock
is released — a critical property that hand-written `lock()`/`unlock()`
calls cannot reliably provide.

A subtle rule: **acquire locks in a consistent order across the whole
program.** Two threads acquiring `A then B` and `B then A` deadlock. The
fix is either a fixed global order (always lock the lower-address mutex
first), or `std::scoped_lock(a, b)` which uses a deadlock-avoidance
algorithm internally. We will use `scoped_lock` once or twice in Chapter
18 and otherwise stick to single-lock critical sections.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

JavaScript has no mutex; the language is single-threaded by design.
Python has `threading.Lock` but the GIL means most concurrency hazards are
covered already. The C++ mutex is a tool you must reach for *constantly*
once you have threads — every cross-thread piece of mutable state needs
one. RAII makes this bearable: a `std::lock_guard` is one line.

</div>

---

## Condition variables: waiting for state

Sometimes a thread needs to *wait* for something else to happen — the
queue gets non-empty, the WAL finishes flushing, the connection gets
data. A spin loop (`while (!ready) {}`) burns CPU; a `sleep` is wasteful
and slow. The right tool is a **condition variable**: a primitive that
parks a thread until another thread signals it, atomically with respect to
a mutex.

```cpp
std::mutex              mu;
std::condition_variable cv;
std::queue<Work>        q;
bool                    stop = false;

// Producer:
{
    std::lock_guard g(mu);
    q.push(std::move(w));
}
cv.notify_one();

// Consumer:
{
    std::unique_lock g(mu);
    cv.wait(g, [&] { return !q.empty() || stop; });
    if (stop && q.empty()) return;
    auto item = std::move(q.front());
    q.pop();
}
// ...process item outside the lock...
```

The `wait` predicate (`[&] { return !q.empty() || stop; }`) protects
against **spurious wakeups** — `cv.wait` may return for no reason at all
according to the standard, and the predicate is the loop that catches
that case and re-waits. Always pass a predicate; never call the
no-predicate overload in production code.

`unique_lock` rather than `lock_guard` because `wait` needs to unlock and
relock the mutex internally — only `unique_lock` exposes that capability.

Together, mutex + condition variable + a shared work queue is the entire
classical thread pool design pattern. Chapter 18 builds Kestrel's worker
pool with exactly these tools, plus an MPSC queue for the no-lock variant.

---

## ThreadSanitizer: catching what you cannot see

UB from data races is hard to diagnose by reading code. **ThreadSanitizer
(TSan)** is the dynamic tool that finds them at runtime — same family as
ASan and UBSan from Chapter 2, different instrumentation. TSan tracks
every memory access and every synchronization operation, and reports any
pair of unsynchronized accesses to the same location where at least one is
a write — exactly the standard's definition of a data race.

Build under TSan:

```bash
cmake -S src -B src/build-tsan -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=thread -fno-omit-frame-pointer"
cmake --build src/build-tsan
ctest --test-dir src/build-tsan --output-on-failure
```

TSan and ASan are mutually exclusive — they instrument memory in
incompatible ways — so you'll keep two build directories from Part 6
onward: `src/build` for ASan/UBSan (the default), `src/build-tsan` for
TSan (the threading audit).

When TSan fires, it prints both stack traces — the offending unsync'd
access and the most recent racing access — along with the relevant
synchronization edges it observed. Reading TSan output is a real skill;
the format is dense but precise. Practice on a deliberately-racy program
before you hit one for real.

<div class="callout callout-experiment">

🧪 **Experiment**

Write a deliberately racy program — two threads incrementing the same
`int count = 0;` a million times each, then printing `count`. Compile
under TSan, run, and read the report. Predict first: what value will
`count` have? (Not 2 million.) Then fix it with `std::atomic<int>` and
confirm TSan goes quiet *and* the result is now correct. Then fix it with
`std::mutex` and confirm the same. This twenty-line experiment is the
single best investment in your concurrent-C++ intuition you can make
before Chapter 18.

</div>

---

## What concurrency lets Kestrel do

Concretely, the threads Kestrel will use in Chapter 18 and Chapter 19:

- **One I/O thread** running the event loop from Chapter 15. Does the
  network reads and writes, runs the RESP parser, and queues commands
  for workers.
- **N worker threads** running a shared MPSC queue. Each worker pulls a
  command, applies it to the keyspace, writes a response back to the
  connection's outbound buffer, and signals the I/O thread.
- **One WAL writer thread** (optional) batching WAL appends and `fsync`s
  across many commands, dramatically reducing per-command durability
  cost.

Each handoff between threads is one happens-before edge, each shared
data structure is one synchronization decision. The decisions we make in
Chapter 18 — fine-grained locking on the keyspace, lock-free MPSC for
queues, atomic counters for stats — are concrete instances of the ideas
above.

We do not need to attack the *whole* keyspace at once. The simplest
sound design is "one big mutex around the entire `HashTable`," and we
will start there. Then, as the experiments show contention, we will
*shard*: split the keyspace into N independent hash tables, hash the key
to pick one, and lock per-shard. That alone gets us most of the way to
N-core scalability with minimal complexity.

<div class="callout callout-ask">

💬 **Ask Claude**

> I'm starting to add concurrency to Kestrel. Walk me through what TSan
> output looks like on a real race — what fields it prints, how to read
> the stack traces, how to map the "synchronization edges" it mentions
> back to my code. Also, give me your reasoning for `std::jthread` over
> `std::thread` in three sentences. Finally: if I want to add a single
> mutex around my `HashTable` as a starting point, where in the code
> should it go?

</div>

---

## Where this leaves Kestrel

Nothing in Kestrel's source has changed in this chapter — we are
preparing. You now know:

- What a data race is, at the standard-language level, and why it is UB.
- That ordinary loads and stores are not safe across threads; `std::atomic`
  is.
- That mutexes plus RAII guards are the workhorse, and condition variables
  the way to wait for state.
- That sequentially consistent ordering is the right default and that
  weaker orderings exist for specific lock-free designs.
- That ThreadSanitizer is the tool that turns the otherwise-invisible
  bugs into reports.

Chapter 18 builds the worker pool: an MPSC queue, a fixed number of
worker threads, the per-shard mutexes that let those workers run in
parallel against the keyspace. Chapter 19 then ties the workers to the
event loop with cross-thread wakeup and careful ownership.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

There is no source change for this chapter. The checkpoint is
*conceptual*. Make sure you can:

- State the standard's definition of a data race.
- Explain why a non-atomic `bool stop_flag` toggled from another thread
  is UB even though one byte fits in a register.
- Distinguish `std::mutex` from `std::atomic`: when each is the right
  tool, and what each does to the happens-before graph.
- Defend `std::jthread` over `std::thread`.
- Set up a TSan build of Kestrel and explain why it lives in a separate
  build directory from the ASan one.

To check yourself:

> Quiz me on C++'s memory model from the perspective of someone who has
> written concurrent JavaScript and Python but no C++ before. Cover: data
> races as UB, the difference between `atomic<int>` and `int` for a
> shared counter, why sequentially consistent ordering is the right
> default, and what a condition variable does that a sleep loop doesn't.
> Push back when my answers are hand-wavy.

When that's solid, Chapter 18 introduces the worker pool.

</div>
