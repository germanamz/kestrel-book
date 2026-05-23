# Benchmark, Profile, Capstone

The book has built a working Redis-compatible server. The chapters from
here back can be re-read in any order and the code keeps working. The
question this chapter asks is the one a real engineer always reaches
last: *how fast is it, where does the time go, and what would I change
if I had to make it ten times faster?*

This is the **measurement** chapter. We benchmark Kestrel against real
workloads, run it under a profiler to see where its cycles go,
implement **pipelining** to verify the reactor was worth building, and
look at the **observability** surface — the metrics, logs, and traces a
production system would actually want. We close with the design choices
worth revisiting, the features deliberately left out (pub/sub,
replication, clustering), and the C++ topics you can now read about in
papers and ship in production code.

No new C++ machinery enters here. Everything in this chapter is about
applying what you have already built and what you have already learned,
plus the discipline of "measure before you optimize."

---

## Benchmarking, the right way

Benchmarks lie if you let them. A benchmark that runs once for half a
second tells you about JIT warmup (oh wait — no JIT, we're in C++; but
about *page-cache warmup, branch predictor warmup, frequency scaling,
and ASan*). A benchmark that locks the server thread to one CPU on a
machine doing nothing else is a fair micro-benchmark; a benchmark on
your laptop running a video call is meaningless.

The rules, in order:

1. **Build the optimized binary.** Use `cmake -DCMAKE_BUILD_TYPE=Release`
   (or `RelWithDebInfo` for profiling). Sanitizers are off; `-O2` is on.
   You have been building Debug + ASan all book, and that is the right
   *development* build — it catches bugs. But you cannot benchmark a
   Debug build. The optimizer is the point.
2. **Pin and warm up.** On Linux, `taskset` pins the process to a CPU.
   On macOS, you can't pin perfectly, but you can close everything else
   and run several iterations of warmup before measurement.
3. **Measure throughput *and* latency.** Throughput is "how many ops/sec
   can I sustain?" Latency is "what's the time to one op?" They are
   different numbers; an op-per-second figure with no latency
   distribution hides tail latency, which is where users live.
4. **Multiple runs, look at variance.** A single 30-second run that
   says "120k ops/sec" is one data point. Five runs that all say
   118k–122k means something; five runs of 80k–200k means your benchmark
   is noisy and you should fix that before drawing conclusions.

A small benchmark harness, in a separate target:

```cmake
add_executable(bench
  bench/main.cpp
  # link against the same libraries kvd uses
)
target_link_libraries(bench PRIVATE doctest::doctest)
```

```cpp
// src/bench/main.cpp  (sketch)
#include <chrono>
#include <iostream>
#include <thread>
#include <vector>

#include "client/encode.hpp"
#include "net/connection.hpp"
// connect helper from Chapter 20

int main(int argc, char** argv) {
    // Args: host, port, command name, total ops, concurrency, value size.
    auto threads_n = 32;
    auto ops_per_thread = 50000;
    std::atomic<long long> total_ops = 0;
    auto start = std::chrono::steady_clock::now();

    std::vector<std::jthread> workers;
    workers.reserve(threads_n);
    for (int t = 0; t < threads_n; ++t) {
        workers.emplace_back([&]{
            auto conn = connect("127.0.0.1", 6379).value();
            std::string val(64, 'x');
            for (int i = 0; i < ops_per_thread; ++i) {
                auto enc = kestrel::client::encode_array({"SET", "k", val});
                conn.write_all(/*...*/).value();
                // Read response (simplified, see Chapter 20 for the loop).
                ++total_ops;
            }
        });
    }
    workers.clear();   // jthread destructors join

    auto end = std::chrono::steady_clock::now();
    auto secs = std::chrono::duration<double>(end - start).count();
    std::cout << total_ops << " ops in " << secs << "s = "
              << static_cast<long long>(total_ops / secs) << " ops/sec\n";
}
```

Two things this harness does *not* do, deliberately: histogram its
latency, and use pipelining. We will add both shortly; they are the
features the chapter is for.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`hyperfine`, `wrk`, `redis-benchmark`, `apache bench` — all do this
job in different shapes. Real `redis-benchmark` is in fact the right
tool to point at Kestrel; it talks RESP and produces clean numbers. If
you want a baseline number for "is my server fast?", `redis-benchmark
-h 127.0.0.1 -p 6379 -n 1000000 -c 50` is one command and you can
compare against the same on real Redis. We write a harness inside
Kestrel mostly for the discipline of doing it.

</div>

---

## Profiling, the right way

Once you have a number, you need to know where the cycles went. There
are three families of profilers and each has its place.

**Sampling profilers** interrupt the program at a fixed rate, record
the call stack, and aggregate. They are nearly free at runtime (so they
work on production builds), they show you where the program *actually*
spends time, and they are the right default. `perf` on Linux,
`Instruments` on macOS, `samply` cross-platform. Use them.

**Tracing / instrumentation profilers** (`callgrind`, `gprof`) inject
counting code at every function boundary. They give you exact call
counts but distort timing — if a small function gets called a billion
times, the instrumentation cost dominates. Useful for "is this hot
function being called as much as I think?", less useful for "where does
the wall time go?"

**Hardware counters** (`perf stat` on Linux, `dtrace` on macOS) expose
cache-miss rates, branch mispredictions, IPC. These are where you go
once the obvious wins are gone and you need to understand *why* a hot
function is slow — e.g., it's cache-miss-bound, not compute-bound.

A first pass with `perf`:

```bash
cmake -B src/build-rel -S src -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build src/build-rel

./src/build-rel/kvd &
perf record -g ./src/build-rel/bench 127.0.0.1 6379 SET 1000000 32 64
perf report
```

`-g` collects call-chains (you'll need frame pointers; we've kept them
on with `-fno-omit-frame-pointer` since Chapter 2). `perf report` opens
a curses UI ranking functions by time spent. The hot list for a typical
Kestrel run looks like:

```
  19.4%  protocol::parse_one(string_view)
  12.1%  std::hash<string_view>::operator()
  10.6%  ::write
   8.3%  ::read
   7.5%  HashTable::probe
   ...
```

That distribution tells you a story. The parser is the hottest spot,
the hashing is second, syscalls are next. Each is an answer to a
different design question, and each can be attacked:

- **Parser**: most cycles are in CRLF scanning. SIMD-accelerated
  newline scanning (`memchr` with explicit vectorization, or just
  `memmem`) is the classic speed-up; modern compilers already lower
  byte-by-byte loops to SIMD if you write them carefully.
- **Hashing**: production hash flooding defenses (XOR with a process
  seed) cost a few cycles; faster hash functions (xxHash) cost fewer
  cycles than `std::hash`. The cost of swapping the hash function for
  the whole keyspace is real; benchmark first.
- **Syscalls**: each `read`/`write` is an entry into the kernel. The
  next chapter's pipelining work amortizes them across many requests
  per syscall. The bigger move (`io_uring` on Linux,
  `kqueue/aio_*` on macOS) is out of scope but worth knowing.

---

## Pipelining: the reactor's payoff, again

A Redis client speaks one request at a time by default — send, wait for
response, send, wait for response. The latency per command equals the
round-trip time, even though the server could be servicing many
commands at once. **Pipelining** is the technique of sending many
requests before reading any responses, then reading them all in a batch.

Kestrel's server already supports it (Chapter 16's parse loop processes
all complete messages in `in_` before yielding back to the loop). The
client needs to *use* it.

The benchmark with pipelining:

```cpp
// In bench: send N requests, then read N responses.
for (int j = 0; j < batch; ++j) {
    auto enc = kestrel::client::encode_array({"SET", "k", val});
    out.append(enc);
}
conn.write_all(out_span).value();
for (int j = 0; j < batch; ++j) {
    // parse one reply from conn (see Chapter 20)
}
```

With a pipeline depth of 32, you should see throughput jump by a factor
of 5–10× on a same-machine Kestrel — most of the time was being spent
waiting for the kernel to deliver each tiny `write`. Pipelining
amortizes that across many commands per syscall.

The number you should look at: **redis-benchmark on real Redis vs your
Kestrel**, both with `-P 32`. Within a small constant factor of real
Redis on the operations you've implemented is a *very* good result for
a teaching codebase.

<div class="callout callout-experiment">

🧪 **Experiment**

Three runs of `redis-benchmark`, on the same machine, against your
Kestrel:

```bash
redis-benchmark -h 127.0.0.1 -p 6379 -t set,get -n 1000000 -c 50
redis-benchmark -h 127.0.0.1 -p 6379 -t set,get -n 1000000 -c 50 -P 16
redis-benchmark -h 127.0.0.1 -p 6379 -t set,get -n 1000000 -c 50 -P 64
```

Note the throughput jumps with `-P`. Then run the *same* against real
Redis (`redis-server` in another terminal). Compare numbers. A factor
of 2× vs real Redis is excellent; 3–5× is normal for a clean
single-thread implementation; 10× is a sign your build is still Debug
or you missed `-O2`. The capstone insight: real Redis is fast because
of accumulated tuning, not because of magic.

</div>

---

## Observability

A production server is not just fast — it is *legible*. When something
goes wrong at 3am you need to find out fast, and "did the WAL get
slower?", "is one shard hot?", "which command is failing?" should be
answerable in seconds. None of that costs much to add, and we did not
need any of it until now, so it lives in this chapter.

The three pillars:

**Logs**: structured, levelled, time-stamped. `kvd` emits info for
startup/shutdown, errors for `fsync` failures and protocol errors, debug
behind a flag for command-by-command tracing. A real codebase reaches
for `spdlog` or similar; if Kestrel stays dependency-light we hand-roll
a tiny logger with `<format>` (C++20).

**Metrics**: counters and histograms exposed for scraping. The minimum:

- `kestrel_commands_total{command="SET"}` — counter per command name.
- `kestrel_connections_active` — gauge.
- `kestrel_wal_fsync_duration_seconds` — histogram.
- `kestrel_keyspace_size{shard="3"}` — gauge per shard.

Prometheus's text format is the cheapest way to expose these; a small
HTTP endpoint on a separate port that prints them on GET. Real Redis
exposes `INFO` over RESP; either works.

**Traces** (distributed): out of scope for the book, but conceptually
the same as logs with an attached request-id and a span hierarchy.
`OpenTelemetry C++` is the standard library. For a single-node Kestrel
the marginal value is low; it matters once you have many services.

A minimum logger using `<format>`:

```cpp
// src/observability/log.hpp
#pragma once

#include <chrono>
#include <format>
#include <iostream>
#include <string_view>

namespace kestrel::log {

template <class... Args>
void info(std::string_view fmt, Args&&... args) {
    auto now = std::chrono::system_clock::now();
    std::cerr << std::format("[{:%FT%TZ}] INFO  {}\n",
                             std::chrono::floor<std::chrono::seconds>(now),
                             std::vformat(fmt, std::make_format_args(args...)));
}

}  // namespace kestrel::log
```

`std::format` is the C++20 type-safe printf; `std::vformat` is the
"format from a runtime string + args" version. The lookup is a couple
of nanoseconds; for an infrequent log line that is fine. We'd avoid
this in the per-op hot path.

---

## The features we did not build

The design spec called out things Kestrel deliberately does *not*
include, and it is worth naming them now:

- **Pub/sub.** Redis's publish/subscribe is one or two more command
  pairs (`SUBSCRIBE`, `PUBLISH`), a per-channel list of connections, and
  the realization that `SUBSCRIBE` puts the connection into a *modal*
  state (it no longer responds to ordinary commands). All the
  machinery — non-blocking I/O, the dispatcher, command tables — is
  already in Kestrel; one chapter of additional work would land it. We
  skipped it because the lessons are repeats.

- **Replication and clustering.** Replicas connect to the primary,
  request a snapshot, then stream the WAL — exactly the file format we
  already have. Adding this would be a real project, not a side
  exercise; the design surface (failover, split brain, consensus) is
  big enough to fill its own book. The vocabulary Kestrel teaches —
  WAL, snapshots, log replay — is the same vocabulary every distributed
  storage system uses.

- **A genuinely cryptographic hash and TLS.** Production servers need
  both; teaching code does not. Adding TLS is a `find_package(OpenSSL)`
  away and a non-trivial `Connection` wrapper that does the handshake.

- **The full Redis command set.** We've implemented enough to demonstrate
  every machinery layer; commands beyond that are mostly variations.
  The Redis docs list 200+ commands; ours covers a couple dozen. Adding
  one more is "write a handler, write a serializer for the new value
  shape if needed, write a test."

What you have built, instead, is **the spine** of a real database
server. Plug in TLS and pub/sub and you've duplicated a meaningful
fraction of real Redis. More importantly, you have written every layer
yourself and so you understand which layer to look at when something
goes wrong.

---

## What this book is, in one sentence per part

- **Part 0** established the four mental-model shifts that make C++ feel
  alien — no runtime, the abstract machine, undefined behavior, value
  semantics — and the toolchain that respects them.
- **Part 1** built ownership: `Value`, `Buffer`, the rule of five and
  the rule of zero, smart pointers, `std::span`, templates and
  `std::variant`.
- **Part 2** spoke on the wire: incremental RESP parsing, `string_view`
  zero-copy discipline, `std::expected` typed errors, and the C++
  dependency tooling that brought in doctest.
- **Part 3** built data structures by hand: an open-addressing hash
  table with intrusive LRU, lazy expiry, ring buffers, hashes, skip
  lists.
- **Part 4** made it durable: an fsync'd WAL, atomic snapshot
  publication via `tmp + rename`, recovery and replay.
- **Part 5** put it on a network: RAII over POSIX, the reactor, edge-
  triggered drain-to-EAGAIN, back to RESP at the connection layer.
- **Part 6** added threads: the memory model and TSan, the worker pool,
  sharded contention, and cross-thread wakeup via `eventfd` plus a
  mailbox.
- **Part 7** measured what was built and pointed at what's next.

The C++ topics covered are the ones professional C++ engineers use
every day: RAII, move semantics, smart pointers, templates and
concepts, `std::variant` and `std::visit`, `std::expected`, `<chrono>`,
`<filesystem>`, the memory model, `std::atomic`, sanitizers. You have
not used C++ that requires standards beyond C++23, and you have not
used any third-party library beyond a test framework. The point of the
constraint is that you can write all of this anywhere — embedded, on
servers, on macOS, on Linux, in any C++ shop that does not have a
specific library mandate.

---

## What to read next

Specific recommendations, all worth your time:

- **Effective Modern C++** (Scott Meyers). The reference for C++11/14
  idioms; everything Kestrel uses is documented there with deeper
  rationales.
- **C++ Concurrency in Action, 2nd edition** (Anthony Williams). The
  book on the C++ memory model and concurrent data structures.
- **Designing Data-Intensive Applications** (Martin Kleppmann). Not C++
  at all, but the right next step for what Kestrel's persistence and
  replication chapters point at.
- **CppCon talks** by Sean Parent, Herb Sutter, Andrei Alexandrescu,
  Bjarne Stroustrup. Watch a few; you'll see the conventions you've
  been building toward formalized.
- The **C++ standard itself** (the latest draft is free online). Less
  forbidding than its reputation; the parts you've used here are well
  written and easier to read than the cppreference summaries you've
  been consulting.

<div class="callout callout-ask">

💬 **Ask Claude**

> Walk through my full Kestrel implementation as it stands. Identify the
> three biggest design weaknesses and rank them by what I'd attack
> first. Suggest specific, tractable next steps for each — not "rewrite
> the parser in assembly," but "the parser allocates here on every
> message; the fix is to reuse a per-connection buffer."

</div>

---

## Closing

You wrote a Redis-compatible key-value store from scratch in C++. The
server is durable, concurrent, network-driven, and correct under
sanitizers. The client speaks the same protocol from the other side.
You understand every layer of the stack you built and every C++ idiom
that holds it together.

What was alien at the start of Chapter 1 — value semantics, lifetime,
undefined behavior, the abstract machine — is now the lens through
which you read C++ code. Every piece of advice you've absorbed
(`unique_ptr` by default, return by value and let the compiler elide,
write the boring lock first, RAII over every resource, `noexcept` only
when it's true, sanitizers always) is what working C++ teams already
argue from. You can now argue with them as a peer.

That is the entire goal of the book.

<div class="callout callout-checkpoint">

✅ **Checkpoint — the capstone**

You should be able to:

- Run Kestrel under `redis-benchmark -P 32` and get a defensible
  ops/sec number against real Redis.
- Take a `perf record` or Instruments capture and name the three hottest
  functions and why each is hot.
- Identify one design decision (e.g., "single global lock vs sharded
  keyspace") whose alternative you understand and could implement.
- Explain to a friend, in five minutes, what you built, what's in it,
  and what you'd add next.

The final hand-over:

> Read my repo top-to-bottom and tell me: where would you take Kestrel
> next? Pick the single highest-impact change — not the highest-effort,
> the highest-impact — and lay out the plan.

When you have asked that question and acted on the answer, you are
finished with this book. The next code you write in C++ will be
yours.

</div>
