# Owning Bytes: The Buffer

Chapter 3 wrote `Value` with a raw `new[]` and a raw `delete[]`, and ended by
pointing at the rule of zero: a class that holds its resources in a member that
already knows how to manage itself, and so needs none of the five special
members. This chapter is about the members modern C++ gives you for exactly that
job — **smart pointers** — and about a small companion abstraction,
**`std::span`**, that lets functions look at someone else's bytes without
claiming any ownership of them.

We will use the chapter to build Kestrel's `Buffer`, the chunk of bytes that
every later part — the RESP parser in Chapter 6, the socket read-loop in
Chapter 14, the WAL in Chapter 12 — will read into and write out of. Along the
way, the hand-rolled `Value` from Chapter 3 collapses into a rule-of-zero
version backed by `Buffer`, and you see, in working code, the difference between
the C++ you wrote *to learn the rules* and the C++ you will actually ship.

There are three smart pointers in the standard library, and they are not
interchangeable. Each one models a different ownership relationship, and using
the wrong one is one of the most common ways professionals still mis-design
C++ code. So we go through them by *what they mean*, not by what their
templates look like.

---

## Ownership has a vocabulary

In Chapter 3 we said "the `Value` owns its bytes." That word *owns* has a
precise meaning in C++ design: the owning object is the one responsible for
freeing the resource. Nobody else may free it; everyone else either holds an
unowning reference, borrows briefly, or asks the owner for a copy. Get the
ownership graph right and lifetime bugs become impossible by construction;
get it wrong and no amount of testing will save you.

Most resources have exactly one owner — one heap allocation, one open file
descriptor, one held mutex. A few have shared ownership — many parts of a
program legitimately keep a thing alive together, and the last one out turns
off the lights. A *very* few need to observe a thing without keeping it alive
at all, breaking a cycle or modeling a non-essential reference.

That triad — one owner, many owners, observer — is exactly what the three
smart pointers correspond to:

| Smart pointer       | Models                                  | When to reach for it                     |
|---------------------|-----------------------------------------|------------------------------------------|
| `std::unique_ptr`   | Single, exclusive owner                 | Default. Use unless you have a reason.   |
| `std::shared_ptr`   | Shared ownership via reference count    | Genuinely concurrent multi-owner cases.  |
| `std::weak_ptr`     | Non-owning observer of a `shared_ptr`   | Cycles, caches, observer patterns.       |

The order matters: **default to `unique_ptr`**. Reach for `shared_ptr` only
when you can articulate the multiple owners. Reach for `weak_ptr` only when
`shared_ptr` left you with a cycle. The common antipattern in production C++
is `shared_ptr` everywhere "to be safe" — it is more expensive than
`unique_ptr`, hides the ownership graph, and turns lifetime bugs into races.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

Python's reference count + cycle collector is closer to `shared_ptr` than to
`unique_ptr` — every object is effectively shared. C++ asks you to think
*first*: who owns this? In nine cases out of ten the honest answer is "this
one thing," and that's `unique_ptr`. Treating the others as the default would
be like writing every Python class to use `weakref.WeakValueDictionary` —
possible, but you've made every reader of your code wonder why.

</div>

---

## `unique_ptr`: one owner, no overhead

`std::unique_ptr<T>` is a pointer that frees what it points to when it is
destroyed. It cannot be copied — that would imply two owners and contradict
its own contract — but it can be **moved**, which transfers the ownership
cleanly. It is the same size as a raw pointer for the common case (no custom
deleter), with zero runtime overhead: the destructor it runs is just `delete`,
and you'd have written that yourself.

A first use of it: replace Chapter 3's hand-rolled allocation with a
self-managing array. Use `std::make_unique<T[]>(n)` for arrays — it picks the
right `new[]/delete[]` pairing so you can't get it wrong:

```cpp
#include <memory>

auto bytes = std::make_unique<std::uint8_t[]>(size);
std::memcpy(bytes.get(), source, size);
// ...use bytes[i] / bytes.get()...
// bytes goes out of scope; delete[] runs automatically
```

There is no constructor to write, no destructor to write, no copy/move
assignment to write. The smart pointer is the resource manager; you just use
it. `.get()` returns the raw `T*` for interop (e.g. `memcpy`); `bytes[i]`
indexes; `bytes.reset()` releases manually; `bytes.release()` hands the
ownership *out* (rare). And because `unique_ptr` is moveable but not copyable,
the compiler stops you at compile time from accidentally creating two owners.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Never call `.release()` to "get the raw pointer" for use — it gives ownership
*away* without freeing. If you only want to peek at the pointer (e.g. to pass
to a C API), use `.get()`. The two methods read alike and do opposite things;
this is a real source of leaks in code by people who think they are clever.

</div>

### Building Kestrel's `Buffer` on `unique_ptr`

`Buffer` is the dynamic byte container Kestrel will use everywhere: parser
input, socket scratch, WAL records. It owns a heap block, exposes the bytes,
and grows when asked to. We want it to be rule-of-zero: no destructor, no
copy/move operators written by hand. Holding the bytes in a `unique_ptr<u8[]>`
*almost* gets us there — the issue is that `unique_ptr<T[]>` does not track
size or capacity, so we keep those as members.

```cpp
// src/core/buffer.hpp
#pragma once

#include <cstddef>
#include <cstdint>
#include <memory>
#include <span>     // C++20

namespace kestrel {

class Buffer {
public:
    Buffer() = default;
    explicit Buffer(std::size_t n)
        : data_(std::make_unique<std::uint8_t[]>(n)),
          size_(n), cap_(n) {}

    // Rule of zero: no destructor, no copy/move members. unique_ptr handles it.
    // Buffer is move-only because unique_ptr is move-only — exactly what we want.

    std::size_t size() const { return size_; }
    std::size_t capacity() const { return cap_; }

    std::uint8_t*       data()       { return data_.get(); }
    const std::uint8_t* data() const { return data_.get(); }

    std::span<std::uint8_t>       bytes()       { return {data_.get(), size_}; }
    std::span<const std::uint8_t> bytes() const { return {data_.get(), size_}; }

    void resize(std::size_t n);   // change logical size, growing capacity if needed
    void reserve(std::size_t n);  // grow capacity without changing logical size

private:
    std::unique_ptr<std::uint8_t[]> data_;
    std::size_t size_ = 0;
    std::size_t cap_  = 0;
};

}  // namespace kestrel
```

```cpp
// src/core/buffer.cpp
#include "core/buffer.hpp"

#include <algorithm>
#include <cstring>

namespace kestrel {

void Buffer::reserve(std::size_t n) {
    if (n <= cap_) return;
    auto fresh = std::make_unique<std::uint8_t[]>(n);
    if (size_) std::memcpy(fresh.get(), data_.get(), size_);
    data_ = std::move(fresh);      // unique_ptr move: old block freed here
    cap_  = n;
}

void Buffer::resize(std::size_t n) {
    if (n > cap_) reserve(std::max(n, cap_ * 2));
    size_ = n;
}

}  // namespace kestrel
```

Look at what *is not* in `buffer.cpp`: no destructor, no copy constructor, no
move constructor, no `delete[]`. The `unique_ptr` member supplies all of them
for free, and the compiler synthesizes `Buffer`'s move operations from its
members' moves. `Buffer` ends up move-only, which fits how Kestrel will use it
(a parse buffer or a socket buffer is never silently deep-copied; if you want
the bytes elsewhere, that is a deliberate action).

Notice also `reserve`'s exception safety: it allocates `fresh` first, copies
into it, and only then moves it into `data_`. If `make_unique` throws, `data_`
is still the original block — the same pattern Chapter 3's copy assignment
followed by hand.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

You are looking at the same `Uint8Array`-style growth strategy you have used a
hundred times — geometric capacity, copy on resize — written out by hand
because there is no language-level "dynamic array" hiding underneath.
`std::vector` exists and does exactly this; we wrote `Buffer` anyway because
Kestrel will eventually need control over the *layout* of its bytes (alignment,
custom allocators) that `vector` does not give cleanly. The principle of
"hold the resource in a smart pointer; let copy/move/free fall out" is the
same regardless.

</div>

---

## `std::span`: a window without ownership

Half the functions you write on byte buffers shouldn't care whether the bytes
live in a `Buffer`, a stack array, a `std::string`, or a fragment of someone
else's storage — they just want "a pointer + a length, please." Pre-C++20 you
would express this as a pair of parameters, or as a template, both of which
leak detail across interfaces. C++20 gave us **`std::span<T>`**: a non-owning
view of a contiguous sequence, exactly the size of two pointers, free to copy.

A function that consumes bytes should take `std::span<const std::uint8_t>`;
a function that produces them takes `std::span<std::uint8_t>`. The caller
constructs the span from whatever they have:

```cpp
void hex_dump(std::span<const std::uint8_t> bytes);

kestrel::Buffer buf(64);
hex_dump(buf.bytes());                                   // from Buffer
hex_dump(std::span(local_array));                        // from a C array
hex_dump({reinterpret_cast<const std::uint8_t*>(s.data()),
          s.size()});                                    // from a std::string
```

`span` is to bytes what `string_view` is to text (Chapter 6 will use both): a
read-only or read-write *handle* you pass around, with no ownership and
therefore no lifetime obligations of its own.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

A `span` is a view; it owns nothing and tracks nothing. If the underlying
storage moves, grows, or dies, the span is left **dangling** — using it is UB.
The most common form: take a span of a `Buffer`'s bytes, then `reserve()`
into the buffer; the underlying allocation moved, and your span now points at
freed memory. ASan catches this *if you exercise it*. Treat spans like
references — never outlive what they refer to.

</div>

---

## `shared_ptr`: when ownership genuinely splits

`shared_ptr` keeps a small heap-allocated **control block** alongside the
object. Each copy of the `shared_ptr` increments a reference count; each
destruction decrements it; when the count reaches zero, the object is freed.
That count is **atomic**, which lets `shared_ptr`s be shared across threads
safely with respect to lifetime — though *not* the object itself, which has
its own thread-safety story.

In Kestrel, you will see `shared_ptr` exactly once — in Chapter 19, when the
event loop and worker threads both legitimately need to keep a connection
alive until both are finished with it. That is the right kind of use: a small
number of distinct owners, each one a real, separate concern. Everywhere else
in Kestrel, `unique_ptr` or plain values do the job, and we will not pay
`shared_ptr`'s overhead.

The overhead is real:

- Each `shared_ptr` is *two* pointers (to the object and to the control block).
- Constructing one allocates the control block — unless you use
  `std::make_shared<T>(...)`, which allocates the object and the control block
  in one call (faster, cache-friendlier, and the form to prefer).
- Copies and destructions perform atomic increments/decrements, which on
  modern CPUs cost real cycles and contention when many threads share the same
  count.

```cpp
// Sketch, not Kestrel code — illustrative of when shared_ptr fits.
auto session = std::make_shared<Session>(...);
loop.register_(session);     // loop keeps a copy
pool.dispatch(session);      // worker pool keeps a copy
// session goes out of scope here; loop and pool each still hold one.
// The Session dies when whichever of those releases its copy last.
```

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`shared_ptr` is the closest C++ comes to Python's per-object reference count.
The difference is that `shared_ptr` only counts the *`shared_ptr`s* themselves
— not every reference to the underlying object. A raw pointer or reference
extracted from a `shared_ptr` does not bump the count. So you can still
dangle. C++ never gives you the "everything is automatically managed" world,
even with smart pointers — it gives you tools, and the discipline to use them
where they fit.

</div>

---

## `weak_ptr`: observing without keeping alive

`shared_ptr` has a fundamental hazard: **cycles**. If `A` shared-owns `B`, and
`B` shared-owns `A`, neither count ever reaches zero and both leak.
`std::weak_ptr<T>` breaks the cycle: it observes a `shared_ptr` *without*
contributing to the count. To use the observed object, you call `.lock()` on
the weak pointer, which atomically attempts to construct a `shared_ptr` — you
get one if the object still exists, or an empty `shared_ptr` if it has been
destroyed.

Kestrel will not need `weak_ptr` until very late (and may not need it at all);
mentioning it here is so you recognize it in the wild. The mental model:
`shared_ptr` is "I am keeping this alive"; `weak_ptr` is "I am noticing this is
alive, and willing to be told it isn't."

---

## Alignment and custom allocators (a quick map)

Two more topics live in this part of the language and we will only sketch
them here — the bytes Kestrel cares about are byte-aligned strings, so we do
not have to fight alignment in Part 1. You will need to know the words.

Every type has an **alignment requirement** — the addresses where instances of
that type may begin. `alignof(int)` is typically `4`; `alignof(double)` is `8`;
SIMD types and some structs require 16 or 32. The compiler enforces alignment
for stack variables and member layout automatically. When you allocate raw
storage yourself (`operator new`, an arena, a memory pool), you must respect
the alignment of whatever you intend to construct there, or you create
undefined behavior — sometimes a crash, sometimes silent corruption on
architectures less forgiving than x86. The keyword `alignas(N)` lets you ask
for stronger alignment than a type would otherwise have.

**Custom allocators** are how you ask the standard library to allocate memory
through your own mechanism — an arena, a pool, a slab — instead of the global
heap. Every standard container has an allocator template parameter
(`std::vector<T, Allocator>`). For Kestrel we will not write a custom
allocator; the only time the topic resurfaces is Chapter 9, where the hash
table allocates with explicit calls and could in principle take an allocator.
The takeaway for now: when you read that someone wrote "a custom allocator,"
they mean a small object that supplies `allocate(n)` / `deallocate(p, n)` for
a container — and they did it because the general-purpose heap was a perf
bottleneck.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Casting a `char*` (or `std::uint8_t*`) to a `Foo*` to "look at" raw bytes as a
`Foo` is UB unless the storage *actually* holds a `Foo` started by a `Foo`
constructor, *and* the address is aligned correctly. Network code is full of
this temptation; the safe ways are `std::memcpy` into a stack `Foo`, or
`std::bit_cast` (C++20) when the types are trivially copyable. The protocol
chapter (Chapter 6) revisits this when we parse RESP integers.

</div>

---

## The rule-of-zero `Value`

We have everything to collapse Chapter 3's hand-rolled `Value` into a clean
rule-of-zero version backed by `Buffer`. The semantics are identical from the
outside; the body has nothing in it but member layout.

```cpp
// src/core/value.hpp
#pragma once

#include <cstddef>
#include <cstdint>
#include <cstring>
#include <span>

#include "core/buffer.hpp"

namespace kestrel {

class Value {
public:
    Value() = default;
    explicit Value(std::span<const std::uint8_t> bytes)
        : storage_(bytes.size())
    {
        if (!bytes.empty()) {
            std::memcpy(storage_.data(), bytes.data(), bytes.size());
        }
    }

    std::span<const std::uint8_t> bytes() const { return storage_.bytes(); }
    std::size_t size() const { return storage_.size(); }
    const std::uint8_t* data() const { return storage_.data(); }

private:
    Buffer storage_;
};

}  // namespace kestrel
```

That is the whole class. No destructor: `Buffer`'s destructor handles cleanup.
No copy constructor: `Buffer` is move-only, so `Value` is move-only too, which
is what we *want* — copying a `Value` should be explicit through a named
function (`Value::clone()` if we ever need it), not happen accidentally on
function-call boundaries. No move constructor: `Buffer`'s move is fine, so the
compiler synthesizes the right thing.

Update `value.cpp` to be empty (or delete it and remove from CMake) — the new
`Value` has no out-of-line definitions to write. Update the test from Chapter
3 to use the new constructor (`Value(std::span)` instead of `Value(ptr,
size)`) and drop the copy expectations; copies are gone on purpose. Run it
under ASan. Everything green.

<div class="callout callout-experiment">

🧪 **Experiment**

Try, deliberately, to copy a `Value`: `Value b = a;` after constructing `a`.
The compiler will refuse with an error mentioning a deleted copy constructor.
*Read the error.* Modern compilers (Clang especially) trace the chain: `Value`
has no copy constructor because `Buffer` has none, because `Buffer`'s
`std::unique_ptr` member has none. This is the rule of zero working in
reverse: a class becomes move-only because its members are, and the diagnostic
explains exactly that. When you understand why C++ said no, you have absorbed
this chapter.

</div>

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's my new rule-of-zero `Value` (`core/value.hpp`) backed by `Buffer`
> (`core/buffer.{hpp,cpp}`). Walk me through which special members the
> compiler synthesizes for `Value`, in what order, and which ones it deletes.
> Why is `Value` now move-only, and would you change that for Kestrel? If I
> needed an explicit deep copy, what would `Value::clone()` look like and
> where would I place it?

</div>

---

## Where Kestrel stands

`src/core/` now has the two foundational types of Part 1:

```text
src/core/
  buffer.hpp / buffer.cpp     # owning, growable byte block (move-only)
  value.hpp                   # rule-of-zero wrapper around Buffer
  version.hpp / version.cpp
```

Every later chunk of Kestrel allocates bytes through `Buffer` and exposes them
through `std::span`. The RESP parser in Chapter 6 reads into a `Buffer`; the
socket layer in Chapter 14 reads into a `Buffer`; the WAL writes a `Buffer`'s
bytes to disk. The smart-pointer vocabulary you just met — `unique_ptr` as
default, `shared_ptr` only where multiple legitimate owners exist, `weak_ptr`
for cycles — is the vocabulary the rest of the book will use without restating
it.

Chapter 5 closes Part 1 by extending the type system itself: **templates**,
**concepts**, **type traits**, and **`std::variant`**, with `Value` becoming a
variant over strings, lists, hashes, and sorted sets. That is the last piece
of Kestrel's data model. After it, Part 2 starts speaking on the wire.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and test should be green under ASan/UBSan:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

Make sure you can:

- Name the three smart pointers and the ownership relationship each models.
- State *why* `unique_ptr` is the default.
- Explain how `Buffer` achieves rule-of-zero: which member supplies the
  destructor, the copy ban, the move.
- Define `std::span` in one sentence and name two ways to make one dangle.
- Show why the new `Value` is move-only and argue whether that is what you
  want for Kestrel.

Then ask Claude:

> Review my Chapter 4 code: `src/core/buffer.{hpp,cpp}` and the rewritten
> `src/core/value.hpp`. Check that `Buffer` is exception-safe in `reserve`
> and `resize`, that I'm using `make_unique` correctly, that my `std::span`
> overloads have the right `const`-ness, and that `Value` is genuinely
> rule-of-zero with the semantics I think I have. Flag any case where I've
> reached for `shared_ptr` when `unique_ptr` would have done.

When that's clean, on to Chapter 5 and the type system.

</div>
