# A Hash Table From Scratch

Kestrel's whole reason to exist is to map keys to values. That mapping is a
hash table, and while `std::unordered_map<std::string, Value>` would technically
work, this is a C++ book and the algorithm at the heart of any KV store
deserves to be implemented by hand. It is also the cleanest place in the book
to address a question that has been waiting: when does the STL stop being the
right answer, and what does it cost to build your own when it does?

This chapter builds Kestrel's keyspace as an open-addressing hash table with
linear probing — the same family Redis itself uses (Redis uses incremental
rehashing with chaining, but the open-addressing variant is simpler and a
better teaching example). Along the way we will cover the algorithmic
choices — open addressing versus chaining, hash functions, load factor and
rehashing, iterator invalidation — and the C++ machinery they force: hashing
custom types, careful storage layout, and the difference between an interface
the STL gives you and the one Kestrel actually needs.

By the end, `src/store/` will contain a hash table that holds Kestrel's
`Value`s by string key, with fast lookup, insertion, and removal — and
Chapter 10 will plug expiry and eviction into it.

---

## Why not just use `std::unordered_map`?

It is worth defending the decision before we make it, because "write your own"
is exactly the wrong answer 95% of the time in C++. `std::unordered_map` is
correct, debugged, well-understood, and widely available. Reaching for a
custom container is a serious move.

Here are the *real* reasons Kestrel writes its own:

1. **Pedagogy.** Hash tables are the single most useful data structure in
   systems programming and you should be able to write one cold. Half of this
   chapter is teaching the algorithm, not the C++.
2. **Layout control.** `unordered_map` uses chaining (one heap node per
   entry), which scatters memory and has poor cache behavior under load.
   Kestrel will see millions of lookups per second; we want a flat, contiguous
   table that fits in cache. The standard library was designed before cache
   behavior dominated performance.
3. **Customizable insertion path.** Expiry (Chapter 10) and eviction
   (Chapter 10) need to do things at insertion and lookup that
   `unordered_map`'s interface does not expose cleanly — tracking access
   order for LRU, lazily expiring entries on touch.

The first reason is the strongest for this book; the second and third are
the real-world ones that would justify the same choice at a job. In every
*other* case — config maps, build-time lookups, anything not on the hot
path — use `unordered_map` or, better, `absl::flat_hash_map`.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

A Python `dict` is, internally, a sophisticated open-addressing hash table
with perturbation probing. JavaScript `Map`s are similar. You have been using
exactly this data structure forever; you just have not had to *write* one.
The mechanics below are the mechanics that have been hiding behind every
`d[k]` you have ever typed.

</div>

---

## Open addressing vs chaining

Hash tables resolve collisions in one of two families.

**Chaining.** Each bucket is the head of a linked list (or another small
container). On a collision, the new entry is appended to the list at that
bucket. Lookups walk the list. `std::unordered_map` does this. Easy to
implement; load factor can exceed 1; deletes are local. Downside: every
non-empty bucket means a separate heap allocation for the list node, and the
nodes' physical layout is scattered across the heap. The cache prefetcher
cannot help; every probe is a pointer chase.

**Open addressing.** Every bucket lives in the same flat array. On a
collision, the entry is placed in the next available slot according to some
probing sequence. Lookups follow the same sequence until they find the key
or hit an empty slot. Load factor must stay below 1 (we will keep ours at
0.7). Upside: the table is one contiguous allocation; lookups under low load
hit a single cache line.

There are several probing sequences — linear (`i+1, i+2, i+3, ...`),
quadratic (`i+1, i+4, i+9, ...`), double-hashing — each with their own
trade-offs against clustering. Linear is the simplest and gives the best
cache behavior because consecutive probes touch consecutive memory addresses.
Its drawback is primary clustering: long runs of occupied slots that grow
even longer. Keeping the load factor below 0.7 keeps clustering manageable.

We will use **linear probing with open addressing**, sized to a power of two,
and resize when 70% full.

<div class="callout callout-algo">

📐 **Algorithm**

**Open addressing with linear probing.** Bucket count `N` is a power of two.
`hash(key) mod N` gives the home bucket. On collision, probe slots
`(home+1) mod N`, `(home+2) mod N`, ... until an empty (or matching) slot is
found. Lookup: same probe sequence, stop on match or on an empty slot. Insert:
same sequence, take the first empty slot. Remove: cannot just mark "empty"
(that breaks lookups for entries that probed past it); instead mark the slot
**tombstoned**, and treat tombstones as "occupied for probing purposes,
available for insertion." Rehash on load factor crossing 0.7 (counting
tombstones in the load).

Average-case complexity: **O(1)** for all three operations when load factor
is bounded. Worst case **O(N)** if everything hashes to the same bucket
(which is why the hash function matters).

</div>

---

## Hashing in C++

C++ does not have a built-in `hash()` like Python's. Instead, the standard
library has **`std::hash<T>`**, a class template you specialize for your own
types (`std::unordered_map` and friends use it internally).

For Kestrel's keys we use `std::string` (we will revisit using `String` from
Chapter 5 in the experiments) so `std::hash<std::string>` is already
available — it is whatever the standard library's implementer chose, usually
a variant of FNV-1a, MurmurHash, or wyhash. It is good enough for a teaching
implementation; for production code you would consider hash flooding
resistance and other security concerns we explicitly skip here.

```cpp
#include <functional>   // std::hash
#include <string>

std::size_t h = std::hash<std::string>{}(my_key);
```

A few non-obvious properties of `std::hash` to remember:

- The result is `std::size_t`, *not* a fixed 64 bits. On 32-bit platforms it
  may be smaller; on 64-bit platforms it's 64 bits.
- The standard does **not** guarantee the same hash value across program
  runs, compilers, or platforms. Don't persist `std::hash` output to disk
  (Chapter 12 will need a stable hash for the WAL).
- For composite types, you write your own `std::hash` specialization or use
  `boost::hash_combine`-style mixing — there is no built-in hash for tuples or
  user-defined structs (C++ committee has discussed this for years).

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

A *bad* hash function — one that maps many keys to the same value — does not
cause UB in the language sense, but it can turn O(1) operations into O(N) and
expose a server to **hash flooding**: an attacker submits crafted keys
designed to collide, and the hash table degrades to a linear scan on every
lookup. Production Redis uses a randomized seed per process to defeat this;
Kestrel for now uses `std::hash` directly, which is good for development and
not adequate for hostile-input use. We acknowledge the gap; fixing it is a
two-line change (XOR the result with a per-process random seed).

</div>

---

## The slot type and the table

The table is an array of slots; each slot is *empty*, *occupied with a
key/value*, or *tombstoned*. We tag the state with an enum and keep the rest
of the fields valid regardless:

```cpp
// src/store/hash_table.hpp
#pragma once

#include <cstddef>
#include <functional>
#include <optional>
#include <string>
#include <utility>
#include <vector>

#include "core/value.hpp"

namespace kestrel::store {

enum class SlotState : std::uint8_t { Empty, Occupied, Tombstoned };

struct Slot {
    SlotState   state = SlotState::Empty;
    std::string key;
    Value       value;
};

class HashTable {
public:
    HashTable();   // initial capacity 16

    bool insert_or_assign(std::string key, Value value);
    bool erase(std::string_view key);
    const Value* find(std::string_view key) const;
    Value*       find(std::string_view key);

    std::size_t size() const { return size_; }

private:
    void rehash(std::size_t new_capacity);
    std::size_t probe(std::string_view key) const;

    std::vector<Slot> slots_;
    std::size_t       size_       = 0;   // occupied count
    std::size_t       tombstones_ = 0;
};

}  // namespace kestrel::store
```

A few design notes you can already see.

**`std::vector<Slot>` for storage.** Rule of zero again: `vector` does the
right thing on resize, on destruction, on move. We do not write a destructor.
We *could* squeeze out a little memory by storing keys and values in two
parallel arrays, or by storing `state` in its own bitset — for production
code, that matters. For learning, one struct per slot is clearer.

**Keys are `std::string`, not `String`.** Owning the key inside the slot is
non-negotiable (it has to outlive whatever called `insert`), and `std::string`
already gives us that for free. We could use Kestrel's own `String`, but
`std::string` plays nicer with `std::hash` and `string_view` here, and the
keyspace is the one place in Kestrel where a hash-friendly text string is the
right type. Values are `Value` (the variant), since they can be any Kestrel
type.

**`find` returns `Value*` (or nullptr).** A pointer is the cheapest way to say
"yes/here" or "no/null." We do not return `std::optional<Value&>` because
references in `optional` are not standard; we do not return a copy of `Value`
because `Value` is move-only.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

A Python `dict.get(key)` returns the value or `None`. C++'s
`find(key) -> Value*` is the closest analog without copying: `nullptr` is the
"not found" sentinel, and the caller dereferences if non-null. The pointer is
non-owning (it points into the table's storage), which means it dangles if
the table rehashes — that's the next gotcha.

</div>

---

## Probing, inserting, finding

The probe function returns the index where `key` lives or would live — the
"insertion point" in both senses. Linear probing means we walk slot by slot
until we either find the key, find an empty slot, or wrap around (which we
prevent by sizing the table so the load factor is bounded):

```cpp
// src/store/hash_table.cpp
#include "store/hash_table.hpp"

namespace kestrel::store {

namespace {
constexpr double kMaxLoadFactor = 0.7;
constexpr std::size_t kInitialCapacity = 16;

std::size_t hash_to_index(std::string_view key, std::size_t capacity) {
    // capacity is a power of two: use mask instead of modulo.
    return std::hash<std::string_view>{}(key) & (capacity - 1);
}
}  // namespace

HashTable::HashTable() {
    slots_.resize(kInitialCapacity);
}

std::size_t HashTable::probe(std::string_view key) const {
    const std::size_t mask = slots_.size() - 1;
    std::size_t i = hash_to_index(key, slots_.size());
    std::size_t first_tomb = slots_.size();   // sentinel: "none"

    for (;;) {
        const Slot& s = slots_[i];
        if (s.state == SlotState::Empty)         return first_tomb == slots_.size() ? i : first_tomb;
        if (s.state == SlotState::Occupied && s.key == key)  return i;
        if (s.state == SlotState::Tombstoned && first_tomb == slots_.size())
            first_tomb = i;
        i = (i + 1) & mask;
    }
}

const Value* HashTable::find(std::string_view key) const {
    if (slots_.empty()) return nullptr;
    std::size_t idx = probe(key);
    return slots_[idx].state == SlotState::Occupied ? &slots_[idx].value : nullptr;
}

Value* HashTable::find(std::string_view key) {
    return const_cast<Value*>(const_cast<const HashTable*>(this)->find(key));
}

bool HashTable::insert_or_assign(std::string key, Value value) {
    if ((size_ + tombstones_ + 1) > kMaxLoadFactor * slots_.size()) {
        rehash(slots_.size() * 2);
    }
    std::size_t idx = probe(key);
    Slot& s = slots_[idx];
    const bool was_new = (s.state != SlotState::Occupied);
    if (s.state == SlotState::Tombstoned) --tombstones_;
    s.state = SlotState::Occupied;
    s.key   = std::move(key);
    s.value = std::move(value);
    if (was_new) ++size_;
    return was_new;
}

bool HashTable::erase(std::string_view key) {
    if (slots_.empty()) return false;
    std::size_t idx = probe(key);
    if (slots_[idx].state != SlotState::Occupied) return false;
    slots_[idx].state = SlotState::Tombstoned;
    slots_[idx].key.clear();
    slots_[idx].value = Value{};
    --size_;
    ++tombstones_;
    return true;
}

void HashTable::rehash(std::size_t new_capacity) {
    std::vector<Slot> old = std::move(slots_);
    slots_.assign(new_capacity, Slot{});
    size_ = 0;
    tombstones_ = 0;
    for (auto& s : old) {
        if (s.state == SlotState::Occupied) {
            insert_or_assign(std::move(s.key), std::move(s.value));
        }
    }
}

}  // namespace kestrel::store
```

A close reading of three subtle choices.

**Power-of-two capacity and bitmask indexing.** `i & (N - 1)` is the same as
`i % N` when `N` is a power of two, and an order of magnitude faster on any
hardware. The price is that the table size is always a power of two; we
accept this happily.

**The tombstone bookkeeping.** Without `first_tomb`, repeated insertions and
deletions degenerate: the table fills up with tombstones, lookups have to
walk past them, and we never reuse the slot. By remembering the first
tombstone seen during a probe and returning it as the insertion point (when
the key isn't already present), inserts reclaim tombstoned space and probe
chains stay tight. We rehash when the *combined* fill (occupants +
tombstones) crosses 0.7 — tombstones cost lookup time too.

**`const_cast` between overloads.** A common C++ idiom: implement the
`const` version once, and write the non-const version by removing the const
twice (once on `this`, once on the result). This is one of the rare places
the cast is correct: we *know* the underlying table is mutable because the
non-const overload was called on a non-const `HashTable`.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Pointers and references returned by `find` are **invalidated by every
insertion that causes a rehash**, by every `erase` (the slot is tombstoned and
its value cleared), and by anything else that mutates the table's storage.
This is exactly the kind of iterator/reference invalidation hazard the STL
has — `vector::push_back` past capacity invalidates all references too. The
rule of thumb: do not hold a pointer from `find` across any mutation. Look
up, use immediately, discard.

</div>

---

## Load factor and rehashing

Open addressing pays for its cache-friendliness with sensitivity to load
factor. At 50% full, the average probe length is ~1.5; at 90% it's ~5.5; at
99% it's ~50. Below 70%, the table is reliably fast; above it, performance
degrades quickly. So we resize when load crosses 0.7 — doubling the
capacity each time.

Doubling makes rehashing **amortized O(1)** per insert: a single rehash of
`N` entries costs O(N), but `N` happens only after `0.7 * N / 2` inserts
since the last rehash, so the cost per insert averages to a constant.

A real production hash table (Redis itself, for instance) often rehashes
**incrementally**: keep both the old and new tables, migrate a handful of
entries per operation, until the old table is empty. This trades a slightly
more complex `find` (it must look in both tables) for the elimination of
the rehash latency spike. We do not implement incremental rehashing in this
chapter — single-shot rehash is fine for a server that is not under hard
real-time constraints — but it is a natural extension to attempt later.

<div class="callout callout-experiment">

🧪 **Experiment**

Instrument your hash table to record the maximum probe length seen during
`probe()`. Run a tight loop that inserts 100,000 keys (`"key0"`, `"key1"`,
...). Observe how the max probe length grows: it should stay small (< 10
typically) through the entire run, with brief spikes around each rehash.
Now intentionally use a terrible hash function: `return 0;` for every key.
Re-run. Watch insertion become quadratic. The hash function is *the* knob
that turns a hash table from useful to useless; it is also why production
implementations seed their hashes randomly.

</div>

---

## Wiring up and testing

The Kestrel command dispatch (built later) will call into `HashTable`
directly. For now, write tests that exercise the algorithm:

```cpp
// src/test/hash_table_test.cpp
#include <doctest/doctest.h>

#include "store/hash_table.hpp"
#include "core/string.hpp"

TEST_CASE("insert and find") {
    kestrel::store::HashTable t;
    CHECK(t.find("missing") == nullptr);

    const std::uint8_t bs[] = {'h', 'i'};
    t.insert_or_assign("greeting", kestrel::String(std::span(bs)));
    auto* v = t.find("greeting");
    REQUIRE(v != nullptr);
    REQUIRE(std::holds_alternative<kestrel::String>(*v));
    CHECK(std::get<kestrel::String>(*v).size() == 2);
}

TEST_CASE("erase tombstones and reuse") {
    kestrel::store::HashTable t;
    for (int i = 0; i < 100; ++i) {
        t.insert_or_assign("k" + std::to_string(i), kestrel::String{});
    }
    CHECK(t.size() == 100);
    for (int i = 0; i < 50; ++i) {
        CHECK(t.erase("k" + std::to_string(i)));
    }
    CHECK(t.size() == 50);
    for (int i = 0; i < 100; ++i) {
        const bool gone = i < 50;
        CHECK((t.find("k" + std::to_string(i)) == nullptr) == gone);
    }
}

TEST_CASE("survives rehash") {
    kestrel::store::HashTable t;
    for (int i = 0; i < 10000; ++i) {
        t.insert_or_assign("k" + std::to_string(i), kestrel::String{});
    }
    for (int i = 0; i < 10000; ++i) {
        CHECK(t.find("k" + std::to_string(i)) != nullptr);
    }
}
```

CMake:

```cmake
add_executable(hash_table_test
  test/main_test.cpp
  test/hash_table_test.cpp
  store/hash_table.cpp
  core/buffer.cpp
)
target_link_libraries(hash_table_test PRIVATE doctest::doctest)
add_test(NAME hash_table_test COMMAND hash_table_test)
```

Build under ASan/UBSan; a passing run with 10,000 inserts is a real exercise
of the rehash path and a strong signal that the bookkeeping is right.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's my `HashTable` (`src/store/hash_table.{hpp,cpp}`) and tests. Review:
> (1) is my tombstone handling correct — would a sequence of insert/erase/insert
> at the same key behave properly? (2) is my rehash threshold reasonable, and
> what would you change for a real production table? (3) does my `find` API
> design make the invalidation rules clear, or should I return something
> stronger than a raw pointer? (4) anything in this code that fights `std`
> idioms I should clean up?

</div>

---

## Where this leaves Kestrel

`src/store/` has the data-structure backbone: a hand-written, cache-friendly
hash table that holds Kestrel's `(key, Value)` pairs. The rest of Part 3
builds on it:

- Chapter 10 adds **TTL expiry** (each slot gains an optional expiry
  timestamp; lookups lazily prune expired entries) and **LRU eviction** (an
  intrusive linked list over the slots, with sampled LRU as an alternative).
- Chapter 11 fills in **`List`, `Hash`, `SortedSet`** with real bodies — the
  data-type implementations behind Redis commands like `LPUSH`, `HGET`,
  `ZADD`.

After Part 3, Kestrel's in-memory layer is complete; Part 4 makes it
persistent.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and tests green under ASan/UBSan:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

You should be able to:

- Explain the difference between chaining and open addressing, and why
  Kestrel chose open addressing.
- Describe what a tombstone is and why it is necessary in open addressing.
- State why the table's capacity is a power of two and what that lets you
  do in the probe code.
- Discuss invalidation: under exactly which operations do `find`'s pointers
  become unsafe?
- Argue the trade-off between single-shot and incremental rehashing.

Hand over:

> Review my Chapter 9 hash table: `src/store/hash_table.{hpp,cpp}` and the
> tests. Are there hidden bugs around tombstones, rehashing, or the
> insert-after-erase case? Is my probe sequence correct on the edge cases
> (table almost full, exactly-on-boundary load)? Suggest one concrete
> improvement I could make in Chapter 10 to set up expiry cleanly.

When that's clean, Chapter 10 plugs expiry and eviction into the table.

</div>
