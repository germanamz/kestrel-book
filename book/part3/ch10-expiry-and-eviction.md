# Expiry & Eviction

A real key-value store does not just hold keys forever. Two policies decide
when a key leaves the table.

**Expiry** is the user's choice: when a client sets `EXPIRE key 60`, that key
should disappear sixty seconds from now. Kestrel must remember the deadline
and refuse to return the value after it has passed.

**Eviction** is the store's choice: when the table is full and a new key
arrives, the store must throw something out. The decision is a policy — *which*
key to throw out. The classic answer is **LRU**: discard the least-recently-used
key, on the theory that recently-touched keys are likely to be touched again.
Redis defaults to a *sampled* LRU because the exact form has overhead it
considers excessive at scale; Kestrel builds both so you can compare.

This chapter teaches the data structures behind both policies — an
**intrusive doubly-linked list** for exact LRU, **time points** from
`<chrono>` for expiry, and the **sampled LRU** Redis trick that trades exact
correctness for memory and speed. It also introduces a C++ topic the book has
been circling: when to reach for the standard library's container and when a
hand-built node-based structure wins.

By the end, `HashTable` from Chapter 9 grows two new responsibilities, the
keyspace becomes evictable, and Kestrel can implement `EXPIRE`, `TTL`, and
`maxmemory` policies that look like Redis.

---

## Expiry: timestamps, not timers

The naive approach is "set a timer for every key with an expiry." It's
quadratic and impossible: a million keys with TTLs would need a million
timers. Real systems instead **store the deadline alongside each key** and
check it on access — *lazy expiry* — supplemented by an occasional sweep that
removes already-expired keys nobody has touched recently.

Each `Slot` in the hash table gains an optional deadline:

```cpp
// src/store/hash_table.hpp  (additions inside Slot)
struct Slot {
    SlotState   state = SlotState::Empty;
    std::string key;
    Value       value;
    std::optional<std::chrono::steady_clock::time_point> expires_at;
};
```

Two choices in that one new field deserve unpacking.

**`std::chrono::steady_clock::time_point`** is C++'s strongly-typed,
type-safe time. There are three clocks in `<chrono>`:

- `system_clock` — wall-clock time, the user's settable clock; can jump
  backward when an admin runs `ntpdate`. **Wrong** for measuring durations.
- `steady_clock` — monotonic, never goes backward, no relationship to
  calendar time. **Right** for "when does this expire?" because we want
  "60 seconds *from now*" regardless of clock adjustments.
- `high_resolution_clock` — usually an alias for one of the above; prefer
  naming what you mean.

So Kestrel's expiries are `steady_clock` time points. When a client says
`EXPIRE key 60`, we compute `steady_clock::now() + std::chrono::seconds(60)`
and store that.

**`std::optional<...>`** because most keys do not have an expiry. The
`optional` is cheap (one extra `bool` per slot, in practice) and makes the
"no expiry" case the same as "no value" at the type level.

A lookup that respects expiry becomes:

```cpp
// in HashTable::find (illustrative; integrate into the body from ch.9)
Value* HashTable::find(std::string_view key) {
    if (slots_.empty()) return nullptr;
    std::size_t idx = probe(key);
    Slot& s = slots_[idx];
    if (s.state != SlotState::Occupied) return nullptr;
    if (s.expires_at && *s.expires_at <= std::chrono::steady_clock::now()) {
        // lazy expiry: erase on the way past
        s.state = SlotState::Tombstoned;
        s.key.clear();
        s.value = Value{};
        s.expires_at.reset();
        --size_;
        ++tombstones_;
        return nullptr;
    }
    return &s.value;
}
```

Notice the test is `<= now()` — at the exact moment of expiry, the key is
gone. Redis uses the same boundary.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

You may have used `setTimeout(() => delete map[key], 60_000)` in Node. That
works for a few keys but creates one timer object per expiry, which doesn't
scale to a million. The lazy-expiry pattern — check on read, sweep
periodically — is what databases actually do, and it has the nice property
that an expired-but-untouched key doesn't pay any CPU until someone looks for
it.

</div>

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`std::chrono::system_clock::time_point` arithmetic with negative durations is
defined; that is not the gotcha. The gotcha is *using `system_clock` for
TTLs*: if the system time jumps backward by an hour, every TTL within the
hour now lives an extra hour. This is a real production bug pattern. Always
use `steady_clock` for "happens N seconds from now" — the type system here
helps because mixing time points from different clocks is a compile error.

</div>

### The periodic sweep

Lazy expiry by itself leaves untouched expired keys lingering in memory
forever. A *background sweep* periodically samples random slots and prunes
expired ones — Redis runs this 10 times per second by default, sampling 20
keys each time and continuing until the fraction of expired keys in the
sample falls below 25%.

```cpp
// src/store/hash_table.cpp  (additions)
void HashTable::sweep_expired(std::size_t sample_size) {
    if (slots_.empty()) return;
    auto now = std::chrono::steady_clock::now();
    std::size_t expired = 0;
    for (std::size_t i = 0; i < sample_size; ++i) {
        std::size_t idx = random_index();        // implementation: PRNG, masked
        Slot& s = slots_[idx];
        if (s.state == SlotState::Occupied && s.expires_at && *s.expires_at <= now) {
            // same delete logic as in find()
            s.state = SlotState::Tombstoned;
            s.key.clear();
            s.value = Value{};
            s.expires_at.reset();
            --size_; ++tombstones_;
            ++expired;
        }
    }
    if (expired > sample_size / 4) {
        sweep_expired(sample_size);              // keep going if it's productive
    }
}
```

The recursion is the "while expired-fraction is high, sample again" loop. In
production code you'd unroll it or run it from a periodic task driven by the
event loop (Chapter 15); for now, a method the server calls from a timer is
plenty.

---

## Exact LRU: the intrusive list + map combination

LRU eviction requires two operations to be fast:

1. **On every access**, mark the touched key as "most recently used."
2. **On eviction**, find the "least recently used" key in O(1).

A `std::list` plus a `std::unordered_map<key, list::iterator>` gets you
there: the list maintains MRU-to-LRU order; the map gives O(1) lookup of the
list node by key. Both `std::list::splice` (to move an existing node to the
front) and `std::list::pop_back` (to remove the LRU node) are O(1).

This is the canonical pattern for exact LRU and the obvious next step over
the hash table. But it has two costs you should see clearly:

- **Two heap allocations per insert** (one for the `unordered_map` node, one
  for the `list` node), plus a third if `key` is a `std::string` that
  exceeds small-string-optimization.
- **Two pointer chases per access** (lookup → list node → splice).

For Kestrel's first cut we will not rebuild the hash table to use this
pattern. Instead, we embed LRU pointers *inside* the existing `Slot`s — an
**intrusive doubly-linked list** through the hash table's own storage. No
extra heap allocations, no `unordered_map`, and the LRU manipulation walks the
same memory the hash table already touches on access.

```cpp
// src/store/hash_table.hpp  (additions inside Slot)
struct Slot {
    // ...existing fields...
    std::size_t lru_prev = kNullIndex;   // index of previous slot in LRU list
    std::size_t lru_next = kNullIndex;   // index of next slot in LRU list
};
```

Plus, in `HashTable`:

```cpp
class HashTable {
    // ...existing members...
private:
    std::size_t lru_head_ = kNullIndex;   // most recently used
    std::size_t lru_tail_ = kNullIndex;   // least recently used

    void lru_touch(std::size_t idx);      // move idx to head
    void lru_unlink(std::size_t idx);     // remove idx from list
    std::size_t lru_pop_tail();           // returns and unlinks LRU
};
```

The implementation is the standard doubly-linked-list bookkeeping, except
that "next pointer" is an *index into the `slots_` array* rather than a
`Slot*`. That means a rehash (which moves slots into a freshly-sized vector)
must remap every index; we handle this by re-running `lru_touch` for each
occupied slot during `rehash`.

```cpp
// src/store/hash_table.cpp  (additions)
constexpr std::size_t kNullIndex = static_cast<std::size_t>(-1);

void HashTable::lru_unlink(std::size_t idx) {
    Slot& s = slots_[idx];
    if (s.lru_prev != kNullIndex) slots_[s.lru_prev].lru_next = s.lru_next;
    else lru_head_ = s.lru_next;
    if (s.lru_next != kNullIndex) slots_[s.lru_next].lru_prev = s.lru_prev;
    else lru_tail_ = s.lru_prev;
    s.lru_prev = s.lru_next = kNullIndex;
}

void HashTable::lru_touch(std::size_t idx) {
    lru_unlink(idx);
    Slot& s = slots_[idx];
    s.lru_prev = kNullIndex;
    s.lru_next = lru_head_;
    if (lru_head_ != kNullIndex) slots_[lru_head_].lru_prev = idx;
    lru_head_ = idx;
    if (lru_tail_ == kNullIndex) lru_tail_ = idx;
}

std::size_t HashTable::lru_pop_tail() {
    std::size_t idx = lru_tail_;
    if (idx == kNullIndex) return kNullIndex;
    lru_unlink(idx);
    return idx;
}
```

`find`, `insert_or_assign`, and `erase` each grow one line: `lru_touch(idx)`
on access and insert, `lru_unlink(idx)` on erase. Eviction is one new method:

```cpp
bool HashTable::evict_one() {
    std::size_t idx = lru_pop_tail();
    if (idx == kNullIndex) return false;
    Slot& s = slots_[idx];
    s.state = SlotState::Tombstoned;
    s.key.clear();
    s.value = Value{};
    s.expires_at.reset();
    --size_;
    ++tombstones_;
    return true;
}
```

The pattern of putting links *inside* the data they connect is called an
**intrusive container**. It is heavier syntax but pays off for two reasons:

- **One allocation per item, period.** The slot lives in the table; the link
  fields live with it; no separate list-node allocation.
- **No iterator invalidation surprises from the list.** The links are
  indices, valid until the table itself moves them — which it does on
  rehash, and we have already accounted for.

Boost has a whole library of intrusive containers; the standard library does
not. Knowing the pattern lets you write your own when the perf math
demands it.

<div class="callout callout-algo">

📐 **Algorithm**

**Exact LRU via intrusive doubly-linked list.** Maintain head/tail pointers
into the table's slots; each slot stores `prev`/`next` indices. On every
read or write of a key, **splice** the slot to the head (MRU position). On
eviction, **pop** the slot at the tail (LRU position). All operations O(1).
Memory cost: two `size_t` per slot (16 bytes on 64-bit). Drawback: every
access mutates state, which has implications for cache lines and for thread
safety we will revisit in Part 6.

</div>

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

If you've implemented an LRU cache in JS or Python, you probably reached for
`Map` (which preserves insertion order; deletes + reinserts to "touch" a key)
or `OrderedDict` (which has explicit `move_to_end`). Both rely on the
language's general-purpose ordered map. C++ doesn't have one for hash tables,
which is exactly why we built one — and why it's strictly faster, because
the link fields live in the same cache line as the data they link.

</div>

---

## Sampled LRU: approximate is enough

Exact LRU's downside is that every access mutates two pointer fields, every
write goes through the list, and the list state is contention for any
concurrent design (Part 6). Redis's solution is *sampled* LRU: instead of
tracking exact recency, store a small **idle timestamp** in each slot,
updated on access; when it's time to evict, sample N random slots, pick the
one with the oldest idle timestamp, and evict it.

The math is forgiving: with 16 samples and a million keys, you almost
always evict from the genuinely-oldest 5% of the keyspace. You give up
strict LRU semantics for:

- **No list mutations on access** — the access path is read-only with
  respect to LRU state, just an idle-timestamp write.
- **No back-pointer maintenance.**
- **Better cache behavior under multi-threaded access** (which Part 6 will
  thank us for).

The implementation is almost trivial:

```cpp
// In Slot:
std::chrono::steady_clock::time_point last_access;

// In find / insert_or_assign:
s.last_access = std::chrono::steady_clock::now();

// Eviction:
bool HashTable::evict_sampled(std::size_t k) {
    std::size_t best = kNullIndex;
    auto best_time = std::chrono::steady_clock::time_point::max();
    for (std::size_t i = 0; i < k; ++i) {
        std::size_t idx = random_index();
        Slot& s = slots_[idx];
        if (s.state != SlotState::Occupied) continue;
        if (s.last_access < best_time) {
            best_time = s.last_access;
            best = idx;
        }
    }
    if (best == kNullIndex) return false;
    // ...tombstone like evict_one()...
    return true;
}
```

`k = 5` is Redis's default for low-overhead approximate LRU; `k = 10` is
closer to exact. The chart in the Redis docs comparing exact vs sampled at
various `k` is worth a look.

You now have two eviction policies. The Kestrel server (Chapter 18 onward)
will pick one at startup; the hash table doesn't care which.

<div class="callout callout-experiment">

🧪 **Experiment**

Compare exact and sampled LRU on a workload. Insert 100,000 keys, then run
1,000,000 accesses with a Zipfian-like distribution (most accesses hit a
small "hot" set; a long tail hits cold keys once). Configure both eviction
policies so that after a threshold the table starts evicting. Measure the
hit rate on the hot set under each policy with `k=5`, `k=10`, exact.
Predict first; then check the numbers. The result is one of the most
satisfying "this approximation is shockingly good" lessons in caching.

</div>

---

## Putting it together: `maxmemory`

The keyspace now needs one new top-level decision: when does it start
evicting? Kestrel exposes a `maxmemory` configuration (we won't wire a real
config system until later; for now, hard-code an `size_t` limit in
`HashTable`). When `size_` crosses that limit, the next `insert_or_assign`
calls `evict_sampled` (or `evict_one`, depending on the policy) before
inserting.

```cpp
bool HashTable::insert_or_assign(std::string key, Value value) {
    while (max_entries_ != 0 && size_ >= max_entries_) {
        if (!evict_sampled(5)) break;          // table empty, nothing to evict
    }
    // ...existing insert logic...
}
```

There are dozens of refinements possible — evict only keys with TTL
(`volatile-lru`), evict based on least-frequently-used (LFU), evict only
when memory hits a byte limit rather than an entry limit, fall back to random
eviction when sampled fails. Redis implements all of them. Kestrel will
ship with two: exact LRU and sampled LRU. The architecture (eviction policy
is a method on `HashTable`) makes adding more a localized change.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here is my extended `HashTable` with TTL expiry and both exact + sampled
> LRU eviction. Review: (1) is my intrusive doubly-linked list correct in
> the edge cases — single element, head and tail being the same slot,
> unlinking the only element? (2) am I handling the LRU links correctly on
> `rehash`, where slot indices change? (3) is `steady_clock` the right
> choice everywhere I use it? (4) would you change the lazy-expiry behavior
> in `find` — for instance, should expired-key lookups be a separate code
> path?

</div>

---

## Where this leaves Kestrel

The keyspace is now feature-complete except for the data types behind
`List`, `Hash`, and `SortedSet` (Chapter 11's work). Kestrel can:

- Store, fetch, and erase `(key, Value)` pairs at O(1) expected time.
- Honour TTLs lazily on access plus periodically sweep expired entries.
- Evict under memory pressure with either exact or approximate LRU.

The structure has grown a lot — `HashTable` is now the largest class in
Kestrel and will not get much bigger. Reading it back end-to-end now is a
good practice: every special-member rule, the intrusive list pattern, the
chrono types, lazy expiry. Each piece earned its way in.

Chapter 11 fills in the remaining `Value` alternatives. `List` is a ring
buffer / deque; `Hash` is a (smaller) hash table; `SortedSet` is a skip
list. After it, Part 3 ends and Part 4 makes Kestrel durable.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and tests green under ASan/UBSan:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

Add a test that:

- Sets a key with TTL 50ms, sleeps 100ms, looks it up, and confirms the
  lookup returns nullptr.
- Fills the table to its `max_entries_` limit and confirms that an
  additional insert triggers eviction.

You should also be able to:

- Explain why expiry is checked lazily on read and supplemented by a periodic
  sweep, not implemented with one timer per key.
- Choose between `system_clock` and `steady_clock` and defend the choice.
- Describe the intrusive doubly-linked list pattern and what it saves
  compared to `std::list`.
- Argue for sampled LRU over exact LRU, and the conditions under which the
  trade-off flips.

Hand it over:

> Review my Chapter 10 additions: `src/store/hash_table.{hpp,cpp}` and the
> new tests. Are my LRU touches happening on every access path, including
> `find`? Is `rehash` correctly remapping the intrusive links? Are there
> race-condition shaped problems waiting for Part 6, and would you fix
> any of them now?

When that's clean, Chapter 11 fills in the data-type variants.

</div>
