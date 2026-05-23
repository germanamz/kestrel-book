# More Data Types

`Value` from Chapter 5 was a variant over four alternatives, of which only
one — `String` — had a real body. The placeholders for `List`, `Hash`, and
`SortedSet` were forward-declared so the rest of the protocol could
compile; they have been waiting for this chapter.

Now we fill them in. Each represents a *different* algorithmic problem and
each leads us to a different C++ tool: a **deque** (ring buffer) for lists, a
**smaller hash table** for the field-keyed hash type, and a **skip list** for
sorted sets. The point is not just to give Kestrel parity with Redis; it is
to give you three small data structures you can implement cold, written in
the C++ idioms the rest of the book has set up.

We are not going to walk you through every method. Each section sketches the
representation, gives you the two or three operations that pin the design,
and then names the rest as exercises — by Chapter 11 you should be implementing
methods like `LPUSH`, `HSET`, `ZADD` mostly on your own, with C++ semantics as
the only friction.

---

## Lists: a ring-buffer deque

Redis lists support push and pop at *both* ends in O(1) (`LPUSH`, `RPUSH`,
`LPOP`, `RPOP`) and indexed access in O(N). Internally Redis uses a
"quicklist" — a doubly linked list of small fixed-size ziplists — for memory
efficiency. Kestrel's first cut is simpler: a **ring buffer over a
`Buffer`-like dynamic byte block**, generalizable to a deque of `String`s.

A ring buffer stores a contiguous array of slots, plus two indices: `head`
(where the next front pop will come from) and `tail` (where the next back
push will land). When indices wrap around the end of the array, they
*modulo* into the front — physical contiguity, logical circularity. Push at
either end is O(1) amortized: capacity grows geometrically, the same as
`std::vector`.

```cpp
// src/store/list.hpp
#pragma once

#include <cstddef>
#include <memory>
#include <string>
#include <utility>

namespace kestrel::store {

class List {
public:
    List();

    // O(1) amortized.
    void push_front(std::string s);
    void push_back(std::string s);

    // O(1). Returns nullopt on empty.
    std::optional<std::string> pop_front();
    std::optional<std::string> pop_back();

    // O(N). Negative indices count from the back (Redis-style).
    std::optional<std::string_view> at(std::ptrdiff_t i) const;

    std::size_t size() const { return size_; }

private:
    void grow();

    std::unique_ptr<std::string[]> buf_;
    std::size_t cap_  = 0;
    std::size_t head_ = 0;
    std::size_t tail_ = 0;       // one past the last element
    std::size_t size_ = 0;
};

}  // namespace kestrel::store
```

A note on storage: `std::string` is the per-element type. Lists contain
binary-safe bulk strings on the wire, and Kestrel's own `String` would work
too — but `std::string` is move-cheap and equality-friendly, and the
keyspace's `HashTable` keys are already strings. Consistency wins.

The mechanics of push and pop:

```cpp
// src/store/list.cpp  (illustrative)
#include "store/list.hpp"

namespace kestrel::store {

namespace { constexpr std::size_t kInitial = 8; }

List::List() : buf_(std::make_unique<std::string[]>(kInitial)), cap_(kInitial) {}

void List::grow() {
    std::size_t new_cap = cap_ * 2;
    auto fresh = std::make_unique<std::string[]>(new_cap);
    for (std::size_t i = 0; i < size_; ++i) {
        fresh[i] = std::move(buf_[(head_ + i) % cap_]);
    }
    buf_ = std::move(fresh);
    head_ = 0;
    tail_ = size_;
    cap_  = new_cap;
}

void List::push_back(std::string s) {
    if (size_ == cap_) grow();
    buf_[tail_] = std::move(s);
    tail_ = (tail_ + 1) % cap_;
    ++size_;
}

void List::push_front(std::string s) {
    if (size_ == cap_) grow();
    head_ = (head_ + cap_ - 1) % cap_;
    buf_[head_] = std::move(s);
    ++size_;
}

std::optional<std::string> List::pop_back() {
    if (size_ == 0) return std::nullopt;
    tail_ = (tail_ + cap_ - 1) % cap_;
    auto out = std::move(buf_[tail_]);
    --size_;
    return out;
}

std::optional<std::string> List::pop_front() {
    if (size_ == 0) return std::nullopt;
    auto out = std::move(buf_[head_]);
    head_ = (head_ + 1) % cap_;
    --size_;
    return out;
}

}  // namespace kestrel::store
```

The mod-by-`cap_` is what makes the buffer ring. When `cap_` is a power of
two, you can swap modulo for a bitmask, the same trick Chapter 9 used. We
keep modulo here for clarity; the optimization is a five-character change.

<div class="callout callout-algo">

📐 **Algorithm**

**Ring buffer deque.** Two indices, `head` and `tail`, walk around a
fixed-size array. Push back: write at `tail`, advance `tail` modulo `cap`.
Push front: decrement `head` modulo `cap`, write there. Resize doubles the
array and copies elements in logical order, resetting `head = 0`. Push,
pop, and indexed access at the ends are O(1); arbitrary indexed access is
O(1) too (`buf[(head + i) % cap]`); insertion or removal in the middle is
O(N). Memory is one allocation, contiguous.

</div>

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`std::deque` exists in the standard library and is a "chunked" array
(many fixed-size blocks linked together) so iterator invalidation works
differently than `vector`. We avoided it for two reasons: chunked layout has
worse cache behavior than our flat ring buffer, and writing the ring buffer
by hand is exactly the kind of "see how it works" Chapter 9 set up. Use
`std::deque` in everyday C++ code; write your own when you need the layout
control.

</div>

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`buf_` is a `unique_ptr<std::string[]>` — note the brackets. The element
type is `std::string`, so each slot must be constructed (the array form of
`make_unique` default-constructs each element). Forgetting the brackets
(`unique_ptr<std::string>`) would call `delete` instead of `delete[]` and
free only the first element — exact same UB as Chapter 3's array/single
mismatch. The compiler will not warn; ASan often does.

</div>

---

## Hashes: a hash within the hash

A Redis `HASH` is a map from field names to string values, scoped under a
single key. `HSET user:42 name "Ada"` puts "Ada" under field "name" of the
hash at key `user:42`. From C++'s side it is just another `unordered_map<
std::string, std::string>` — a hash table inside the `Value`.

We have two choices:

1. **Build it on Chapter 9's `HashTable`**, generalized over a value type.
   Reuse the open-addressing implementation; pay one open-addressing table's
   overhead per `Hash`.
2. **Use `std::unordered_map<std::string, std::string>`**. Quick, correct,
   honest about the algorithm being chaining.

For pedagogy the first is more satisfying. For pragmatism the second is what
Kestrel actually ships. A `HASH` typically has tens or hundreds of fields;
the cache-locality advantage of open addressing barely shows up at that
size, and `unordered_map` is one line to write.

```cpp
// src/store/hash.hpp
#pragma once

#include <optional>
#include <string>
#include <string_view>
#include <unordered_map>

namespace kestrel::store {

class Hash {
public:
    // O(1) average.
    void set(std::string field, std::string value);
    bool del(std::string_view field);
    std::optional<std::string_view> get(std::string_view field) const;

    std::size_t size() const { return fields_.size(); }
    const std::unordered_map<std::string, std::string>& fields() const { return fields_; }

private:
    std::unordered_map<std::string, std::string> fields_;
};

}  // namespace kestrel::store
```

The implementation is direct:

```cpp
void Hash::set(std::string field, std::string value) {
    fields_.insert_or_assign(std::move(field), std::move(value));
}

bool Hash::del(std::string_view field) {
    auto it = fields_.find(std::string(field));    // heterogeneous lookup C++20
    if (it == fields_.end()) return false;
    fields_.erase(it);
    return true;
}

std::optional<std::string_view> Hash::get(std::string_view field) const {
    auto it = fields_.find(std::string(field));
    if (it == fields_.end()) return std::nullopt;
    return std::string_view(it->second);
}
```

That `std::string(field)` in `find` is the irritating part: pre-C++20,
`unordered_map<std::string, V>` could only be looked up by `std::string`, not
`string_view`, because the hasher and equality operators were specialized on
the key type. C++20 added **heterogeneous lookup** for ordered containers and
C++23 extended it to unordered containers — provided you opt in by defining
a transparent hasher and equality. The opt-in is verbose enough that we
accept the temporary string for now; the experiment below shows the
opt-in shape.

<div class="callout callout-experiment">

🧪 **Experiment**

Opt into heterogeneous lookup for the inner `Hash` by defining:

```cpp
struct StringHash {
    using is_transparent = void;     // the magic typedef
    std::size_t operator()(std::string_view s) const noexcept { return std::hash<std::string_view>{}(s); }
    std::size_t operator()(const std::string& s) const noexcept { return std::hash<std::string_view>{}(s); }
};
struct StringEq {
    using is_transparent = void;
    bool operator()(std::string_view a, std::string_view b) const noexcept { return a == b; }
    // ...other overloads...
};
using FieldMap = std::unordered_map<std::string, std::string, StringHash, StringEq>;
```

Now `fields_.find(field)` works with a `string_view` and allocates no
temporary string. Confirm with a benchmark that millions of `HGET` lookups
no longer touch the heap. This is a real C++20/23 idiom you will reach for
often once you know its shape.

</div>

---

## Sorted sets: skip lists, with rationale

A Redis `ZSET` is a sorted set keyed by a string `member` with a `double`
score, supporting:

- `ZADD key score member` — insert or update.
- `ZSCORE key member` — get the score for a member. O(1) — needs an
  auxiliary hash map.
- `ZRANGE key start stop` — return the members in score order between two
  ranks. O(log N + M) for M results.
- `ZRANK key member` — return the rank of a member. O(log N).
- `ZRANGEBYSCORE key min max` — range by score, not rank. O(log N + M).

The data structure needs two views over the same data: an *unordered* view
(by member, for O(1) score lookup) and an *ordered* view (by score, for
range and rank). The first is a hash table. The second is a sorted
container with O(log N) insert and *rank* support — and that is where the
choice is interesting.

A balanced BST (`std::set` is a red-black tree) supports insert and lookup in
O(log N) but does not support rank in O(log N) — getting the i-th element
requires walking. To support rank, BSTs have to be augmented with subtree
counts, which is not part of `std::set`'s API.

**Skip lists** are the data structure of choice. They support insert, delete,
search, *and rank* all in O(log N) expected time, with a simple
implementation that fits in a hundred lines, and they have a property useful
for Part 6: skip-list-style structures are easier to make concurrent than
red-black trees. Redis chose them for both reasons.

### The skip-list algorithm

A skip list is a stack of linked lists, where each higher level skips over
twice as many nodes on average. The bottom level contains every element in
order; the level above contains roughly half of them; the next level a
quarter; and so on, up to `log N` levels.

```text
L3:  ----------> 30 ---------> 100
L2:  --> 10 --> 30 --> 60 ---> 100
L1:  -> 10 -> 20 -> 30 -> 45 -> 60 -> 100
```

Search: start at the top-left. While the next node at this level exists and
is ≤ target, go forward; otherwise drop down a level. Stop when you reach
level 0 and the next node either matches or exceeds the target.

Insert: search for the position; flip a coin to decide if the new node also
participates at level 1; flip again for level 2; until heads. Splice the new
node into every level it participates in. This randomization keeps the
average level count balanced without explicit rebalancing.

```cpp
// src/store/sorted_set.hpp  (sketch)
#pragma once

#include <memory>
#include <random>
#include <string>
#include <unordered_map>

namespace kestrel::store {

class SortedSet {
public:
    bool add(std::string member, double score);     // returns true if new
    std::optional<double> score(std::string_view m) const;
    bool remove(std::string_view m);
    std::size_t size() const { return by_member_.size(); }

    // Range by rank, by score, count by score — exercises for the reader.

private:
    static constexpr int kMaxLevel = 32;
    static constexpr double kProb  = 0.5;

    struct Node {
        std::string member;
        double      score = 0.0;
        std::vector<Node*> next;   // size == this node's level
    };

    int random_level();

    std::unique_ptr<Node> head_ = std::make_unique<Node>();    // dummy head, level kMaxLevel
    int current_level_ = 1;
    std::unordered_map<std::string, Node*> by_member_;        // member -> node
    std::mt19937 rng_{std::random_device{}()};
};

}  // namespace kestrel::store
```

Two design notes worth your attention.

**The dummy head.** A sentinel node at the front simplifies the splice
logic: every "previous pointer" during a search either points at the dummy
head or at a real node, never at "nothing." This is a classic linked-list
trick that pays for itself in cleaner code.

**`by_member_` for O(1) score lookup.** The skip list alone makes
member→score lookup O(log N); pairing it with a hash map drops that to O(1)
and is what Redis does. The hash map's value is a raw `Node*` — non-owning,
the skip-list owns the node — and that means deletions from the skip list
must remember to remove the entry from `by_member_` too. Both views, kept
consistent on every mutation.

The full skip-list implementation is more lines than fits cleanly here. Two
realistic options:

1. **Implement it yourself** as a Chapter 11 exercise — search, insert,
   remove. Test by inserting random `(member, score)` pairs and confirming
   `ZRANGE 0 -1` returns them in score order.
2. **Stub it** for now and come back to it during Chapter 21 (the capstone)
   when you benchmark range queries.

We strongly recommend (1) — the skip list is a beautiful data structure and
its asymmetry with the balanced BST is one of those "the simpler one is
also faster in practice" lessons C++ engineers learn the hard way.

<div class="callout callout-algo">

📐 **Algorithm**

**Skip list.** A probabilistic data structure for sorted sequences. Each
element participates in level *k* with probability `p^k` (typically
`p = 0.5`). Search, insert, delete are O(log N) expected. Rank query is
O(log N) if you store "skip widths" alongside the level pointers. Memory:
about `1 / (1 - p)` pointers per element on average — `2` for `p = 0.5`.
Performance comparable to a red-black tree with simpler code, no
rebalancing, and easier concurrent variants (cf. the lock-free skip list).

</div>

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

There is no `SortedDict` or `SortedSet` in JavaScript or in Python's
standard library (`sortedcontainers` exists in PyPI but is third-party).
Redis's `ZSET` is one of the prime examples of a data structure you have
never had ergonomically; doing it in C++ — even if just understanding the
algorithm — is part of why a C++ engineer reaches for fewer libraries.

</div>

---

## Wiring the variants in

Update `core/value.hpp` to *include* the new types instead of forward-
declaring them:

```cpp
// src/core/value.hpp
#pragma once

#include <variant>

#include "core/string.hpp"
#include "store/list.hpp"
#include "store/hash.hpp"
#include "store/sorted_set.hpp"

namespace kestrel {

using Value = std::variant<String, store::List, store::Hash, store::SortedSet>;

}  // namespace kestrel
```

Notice the namespace mixing: `String` is in `kestrel`; `List`, `Hash`,
`SortedSet` are in `kestrel::store`. Either you bring everything under one
namespace, or you accept the mix. We accept the mix because the layering
*means* something: `String` is a core, foundational type; the container
types are store-specific. Namespace as documentation.

The serializer from Chapter 6 can now fill in its `if constexpr` branches:
`String` writes as a bulk; `List` writes as an `*N`-length array of bulks;
`Hash` writes as an array of alternating field/value bulks; `SortedSet` as an
array of alternating member/score bulks. The shape is dictated by RESP; the
templated visitor from Chapter 5 makes each case a four-line function.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here are my new `store/list.{hpp,cpp}`, `store/hash.{hpp,cpp}`, and
> `store/sorted_set.{hpp,cpp}`. Walk me through: (1) my `List`'s ring-buffer
> bookkeeping — am I handling empty/full conditions and the resize edge
> cases correctly? (2) is my `Hash` heterogeneous lookup set up correctly,
> and does it actually avoid the temporary string? (3) the skip list — is my
> level distribution correct, and have I forgotten the `by_member_` update
> on remove? Suggest one test per data type that would catch a class of
> bugs I'm likely missing.

</div>

---

## Where Part 3 ends

`Value` is now a real variant: a string, a deque-shaped list, a smaller
hash, or a sorted set. The keyspace can store any of them under any key,
expire any of them, and evict any of them. The protocol layer (Chapter 6's
serializer) can write any of them to the wire as the right RESP shape.

You have also written three data structures by hand: a hash table (Chapter
9), an intrusive doubly-linked list (Chapter 10), a ring buffer, and the
beginnings of a skip list (this chapter). Each one taught a C++ idiom in
service of a real algorithm: rule-of-zero containers, intrusive layout
control, `std::chrono` for time, transparent hashers for heterogeneous
lookup.

Part 4 makes Kestrel durable: a write-ahead log writes every mutating
command to disk, snapshots periodically dump the keyspace, and crash
recovery replays both back into the data structures you just built. The
algorithms behind that are the next two chapters; the C++ topic is
filesystem and stream I/O, which has its own surprises.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and tests green:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

You should have tests covering:

- `List`: push at both ends, pop at both ends, growth across the ring
  boundary, indexed access with negative indices.
- `Hash`: insert, overwrite, delete, missing-field lookup, the heterogeneous
  lookup path.
- `SortedSet`: insert, score lookup, ordering of returned members. (Range
  queries can wait if you stubbed them.)

You should be able to:

- Describe the ring buffer's invariants and explain how `head`, `tail`,
  `size`, and `cap` relate.
- Defend the use of `unordered_map` for `Hash` rather than reusing `HashTable`.
- Explain skip-list probability, average levels, and why it is a natural
  fit for `ZSET`.

Hand it to Claude:

> Review Part 3's data types: `store/list.{hpp,cpp}`, `store/hash.{hpp,cpp}`,
> `store/sorted_set.{hpp,cpp}`. Check correctness on the ring-boundary
> resize, the heterogeneous lookup path, and skip-list insert/remove
> consistency. Are there places where C++ semantics — say, exceptions
> thrown during a move — could leave a structure in a bad state? What would
> you change before Part 4 starts persisting these to disk?

When Part 3 is fully green, Part 4 — persistence — begins.

</div>
