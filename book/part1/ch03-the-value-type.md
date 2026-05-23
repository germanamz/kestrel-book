# The Value Type

A key-value store has, in its very name, one type more important than any other:
the **value**. In Kestrel, the value is what the user stored — a sequence of
bytes that we hand back, intact, when they ask for it. By the end of the book it
will be a variant over strings, lists, hashes, and sorted sets, but the variant
is Chapter 5's lesson. This chapter is about the simplest possible Value — a
**binary-safe string** — and the C++ machinery that gives Kestrel ownership of
the bytes inside it.

That machinery is the heart of the language. Designing one type that owns memory
correctly forces you to touch nearly every concept Chapter 1 warned was alien:
stack versus heap, value semantics, copying, moving, deterministic destruction,
the rule of three, the rule of five, and the optimizer's invisible hand via copy
elision. If you finish this chapter able to write `Value` from scratch and
explain every special member function it has — or doesn't — the rest of Part 1
is variation on a theme.

We will write the type once, naively, and let the compiler tell us what is
missing. Each gap it exposes is a place where C++ has a rule you have not yet
met. The naive version will be wrong in instructive ways; the final version will
be small, correct, and idiomatic.

---

## What a Value has to do

Before any C++, settle the requirements. A `Value` holds a sequence of bytes —
not characters, *bytes*, because Redis values can contain `\0` and any other
byte. So no null-terminator nonsense; we carry an explicit length. The bytes
are variable-size and not known at compile time, so they must live on the heap.
The `Value` itself owns those bytes: when the `Value` is destroyed, the bytes
are freed; when the `Value` is copied, the new `Value` gets its own copy of the
bytes (value semantics, Chapter 1's Assumption 4 made concrete); when the
`Value` is moved, the new `Value` takes the bytes and the old one is left empty.
Nothing outside the `Value` ever has to worry about the buffer's lifetime.

That single paragraph — owns the bytes, deep-copies on copy, transfers on move,
frees on destruction — is the contract. Everything below is how C++ lets you
write it.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

A Python `bytes` object handles this contract for you: the interpreter
allocates, reference-counts, and frees the underlying buffer; you never see the
machinery. In C++ there is no interpreter standing behind your class
(Assumption 1), so the class itself *is* the machinery. Writing `Value` is
writing, from scratch, the thing CPython's `bytesobject.c` does for `bytes`.
That is not extra ceremony — it is the actual job.

</div>

---

## Stack versus heap: where storage comes from

A `Value` object — the thing the variable names — has a fixed-size layout known
at compile time: enough room for a pointer to the bytes, plus the length, plus
maybe a capacity. The bytes themselves are variable-size and only known at run
time. C++ gives you two regions of memory to put things in, and the choice is
not aesthetic; it follows from what you know at compile time.

The **stack** is the region the function call mechanism uses. Each call frame
holds the local variables for that call. Allocation is a register decrement
(O(1)), deallocation is automatic when the function returns, and the size of
every stack object must be known by the compiler. The `Value` object itself —
the pointer/length/capacity triple — fits perfectly on the stack: a fixed-size
struct with a known layout.

The **heap** is a much larger region you ask for explicitly with `new` (or, in
practice, with `std::make_unique`, `std::vector`, allocators — almost anything
*but* a raw `new`). Heap allocations live until you (or RAII on your behalf)
release them. They can be any size, decided at run time. The bytes a `Value`
points at — the actual variable-length data — must live on the heap, because
their size is not knowable when `main` is compiled.

So one `Value` straddles both regions: a fixed-size head on the stack (or
wherever the `Value` lives) and a variable-size tail on the heap. The job of
the class is to keep that arrangement consistent through copy, move, and
destruction.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

In Python and JavaScript, *everything* user-facing lives on the heap; the
variable on the stack is just a reference. C++ inverts that: by default, what
you write *is* the object, and only the parts you explicitly request (with
`new`, `std::vector`, `std::string`, etc.) escape to the heap. The discipline
of "what is on the stack, what is on the heap?" is something you will learn to
see at a glance. It is also what makes C++ programs fast — touching the stack
is essentially free.

</div>

---

## A first, naive Value

Let's write the simplest thing that fulfils the contract, header first:

```cpp
// src/core/value.hpp
#pragma once

#include <cstddef>   // for std::size_t
#include <cstdint>   // for std::uint8_t

namespace kestrel {

class Value {
public:
    Value();                              // empty value
    Value(const std::uint8_t* data, std::size_t size);  // copy from a range
    ~Value();                             // free the bytes

    const std::uint8_t* data() const { return data_; }
    std::size_t size() const { return size_; }

private:
    std::uint8_t* data_ = nullptr;
    std::size_t   size_ = 0;
};

}  // namespace kestrel
```

```cpp
// src/core/value.cpp
#include "core/value.hpp"

#include <cstring>   // std::memcpy
#include <new>       // operator new

namespace kestrel {

Value::Value() = default;

Value::Value(const std::uint8_t* data, std::size_t size)
    : data_(size ? new std::uint8_t[size] : nullptr),
      size_(size)
{
    if (size) std::memcpy(data_, data, size);
}

Value::~Value() {
    delete[] data_;   // delete[] for arrays; delete for single objects.
}

}  // namespace kestrel
```

Two ideas already deserve a moment.

The constructor uses a **member initializer list** (`: data_(...), size_(size)`)
rather than assigning inside the body. The rule, more strict than in most
languages: a class's members are constructed *before* the body runs. The
initializer list lets you initialize each member directly with the value you
want; assigning in the body would first default-construct, then overwrite —
slower for non-trivial types and impossible for `const` members and references.
Always reach for the initializer list first.

The destructor uses `delete[]`, not `delete`. The two are not interchangeable:
`new T[n]` and `delete[]` form one matching pair; `new T` and `delete` form
another. Mixing them is **undefined behavior**, not a warning — exactly the
silent class of bug Chapter 1 promised would haunt us. We are about to delete
both raw `new` and raw `delete[]` from this file in the next section by reaching
for `std::unique_ptr`, but you need to see the matching rule once.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`new`/`delete` and `new[]`/`delete[]` are *two different operators* with two
different allocation paths. `delete` on an array-new pointer (or vice versa) is
UB even though the compiler will happily emit either; ASan often catches it,
but you should not rely on noticing. This mismatch is one of the long-standing
reasons modern C++ tells you to almost never write a raw `new` or `delete` —
hand the ownership to a smart pointer (`std::unique_ptr<T[]>` for an array) and
let the destructor pick the right call for you. We'll do this in Chapter 4.

</div>

---

## What the compiler quietly does for you

`Value` as written compiles. It also misbehaves the moment you copy it. To see
why, write a tiny test (you do not need to keep it):

```cpp
const std::uint8_t bytes[] = {1, 2, 3};
kestrel::Value a(bytes, 3);
kestrel::Value b = a;            // (1)
// ...both go out of scope here...
```

Line (1) does not call any constructor you wrote. There is no
`Value(const Value&)` in your class. So why does it compile? Because **the
compiler synthesizes one for you** unless you say otherwise. A compiler-generated
copy constructor copies each member, *one by one* — for `data_` (a pointer) and
`size_` (a number), that is a literal copy of the pointer value and the integer.

The result is two `Value` objects whose `data_` pointers refer to **the same**
heap buffer. When the block ends, both destructors run, both call `delete[]`
on the same pointer, and your program enters undefined behavior — a classic
**double-free**. Under ASan it crashes with a clear report. Without ASan, it
might silently corrupt the heap and crash much later, in unrelated code, on
Tuesday in production.

This is one of the most important rules in the language. **If your class
manages a resource (memory, file, socket, lock), the compiler-generated copy
operations are almost always wrong.** The compiler doesn't know your `data_` is
an *owned* pointer rather than a non-owning observer; it just copies the bits.
You have to override the copy operations — and, as you are about to see, the
move operations and the destructor too. There is a *name* for the rule that
ties these together.

---

## The rule of three (and why it still matters)

The classical rule: **if you write any of the destructor, copy constructor, or
copy assignment operator, you almost certainly need all three.** They are
linked by ownership semantics — if cleanup is non-trivial (you wrote a
destructor), then copying probably needs to allocate a fresh buffer too, and
assigning over an existing object needs to free the old buffer and allocate a
new one.

Write them out for `Value`:

```cpp
// src/core/value.hpp  (additions inside class Value)

Value(const Value& other);              // copy constructor
Value& operator=(const Value& other);   // copy assignment
```

```cpp
// src/core/value.cpp  (additions)

Value::Value(const Value& other)
    : data_(other.size_ ? new std::uint8_t[other.size_] : nullptr),
      size_(other.size_)
{
    if (size_) std::memcpy(data_, other.data_, size_);
}

Value& Value::operator=(const Value& other) {
    if (this == &other) return *this;            // (a) self-assignment guard

    std::uint8_t* fresh = other.size_             // (b) allocate first
        ? new std::uint8_t[other.size_]
        : nullptr;
    if (other.size_) std::memcpy(fresh, other.data_, other.size_);

    delete[] data_;                               // (c) only now release old
    data_ = fresh;
    size_ = other.size_;
    return *this;
}
```

Three details that look like paranoia but are not:

- **(a) Self-assignment guard.** `v = v;` is rare in code you write deliberately
  but common through references and aliasing. Without the guard, you would
  `delete[]` `data_` and then `memcpy` from a buffer you just freed — UB.
- **(b) Allocate before releasing.** This is the **strong exception guarantee**
  habit. If `new` throws (out of memory), we have not yet touched our own
  state. The old `*this` is still valid; the assignment simply failed cleanly.
  If we had deleted first and then thrown on `new`, we would have corrupted a
  live object on the way to a failed assignment.
- **(c) Release after success.** Once allocation and copy succeeded, swap in
  the new buffer and free the old. This pattern — *allocate, then commit, then
  release* — is the spine of exception-safe resource management.

Now `Value b = a;` deep-copies. The double-free is gone. We are, however,
copying every byte every time, which is wasteful when the source is about to
disappear anyway.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`b = a` in Python rebinds the name; the underlying buffer is shared and
reference-counted. `b = a` in C++ here makes an entire deep copy of the bytes
(value semantics, Assumption 4). For small `Value`s this is fine; for large
buffers it is a real cost. C++'s way out is not "share by reference" but
*move semantics* — transfer ownership when the source is about to die anyway.

</div>

---

## Move semantics: stealing instead of copying

Consider `Value make_value() { ... }` returning a `Value`. The temporary
returned is about to be destroyed; nothing else will ever see it again. Copying
its bytes into the destination is correct but pointless — far better to *take*
its `data_` pointer, leave the temporary holding `nullptr`, and let it die.
That transfer is **move semantics**, and C++ gives it a separate syntax and two
new special member functions.

Add to the header:

```cpp
Value(Value&& other) noexcept;            // move constructor
Value& operator=(Value&& other) noexcept; // move assignment
```

That `&&` is an **rvalue reference** — it binds to objects the compiler knows
are about to expire (function return values, things you wrap in `std::move`).
The `noexcept` is not decoration: STL containers like `std::vector` will only
move your elements during a reallocation if your move operations are declared
`noexcept`; otherwise they fall back to *copying*, silently undoing your
optimization. Make moves `noexcept` or lose half the point.

Implementing them is, if anything, simpler than copying — moves swap pointers,
they do not allocate:

```cpp
Value::Value(Value&& other) noexcept
    : data_(other.data_), size_(other.size_)
{
    other.data_ = nullptr;
    other.size_ = 0;
}

Value& Value::operator=(Value&& other) noexcept {
    if (this == &other) return *this;
    delete[] data_;                  // free our own first
    data_ = other.data_;
    size_ = other.size_;
    other.data_ = nullptr;
    other.size_ = 0;
    return *this;
}
```

Two things to internalize.

**The moved-from object must remain destructible.** After a move, `other` may
be destroyed at any moment; if we left its `data_` pointing at the buffer we
just stole, its destructor would `delete[]` a pointer we have now claimed —
double-free again. So we *null out* `other.data_` and reset `other.size_`. A
moved-from object is in a "valid but unspecified" state: you can destroy it,
assign to it, ask its `size()` — but you should not read its bytes as if it
still held data.

**`std::move` does not move anything.** It is a cast, not an operation. It
turns an lvalue into an rvalue reference, which then *selects* the move
overload at compile time. So when you write:

```cpp
kestrel::Value sink;
sink = std::move(local);    // selects operator=(Value&&)
```

`std::move` produced no machine code at all. It only changed which constructor
or assignment the compiler picked. Naming it "move" was, in retrospect, a
mistake — it should have been "rvalue_cast." Remember the function name as a
piece of selection syntax.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Reading the *value* of a moved-from object (its bytes, its semantic content)
where the type's contract does not promise something specific is a bug. The
standard library promises only that moved-from standard types are
*destructible and assignable* — not empty, not zero, not anything in
particular. For your own `Value`, document what a moved-from `Value` looks
like (we choose: `size() == 0`, `data() == nullptr`) and stick to it; for
foreign types, assume nothing.

</div>

<div class="callout callout-experiment">

🧪 **Experiment**

Add a noisy `std::printf("copy\n")` to your copy constructor and
`std::printf("move\n")` to your move constructor, then run:

```cpp
kestrel::Value make() {
    const std::uint8_t bs[] = {1, 2, 3};
    return kestrel::Value(bs, 3);
}

int main() {
    kestrel::Value v = make();      // (A)
    kestrel::Value w = v;           // (B)
    kestrel::Value x = std::move(v);// (C)
}
```

Predict first: how many `copy` and how many `move` prints? Then run it. (A)
likely prints *nothing* — copy elision (next section) erases the move entirely.
(B) prints `copy`. (C) prints `move`. Remove the prints when you are done; we
are not committing instrumentation to Kestrel.

</div>

---

## Copy elision: the move that wasn't

You will notice the experiment above produced no message on the return from
`make()`, even though a temporary clearly traveled out of the function. The
compiler is allowed — and as of C++17, *required* in many cases — to construct
the returned object directly in the caller's storage, skipping both the copy
constructor and the move constructor entirely. This is **copy elision** (also
called **Return Value Optimization**, NRVO when the returned object has a
name).

The implication is liberating: writing functions that *return objects by value*
is the right default in modern C++, even when the object is large. The
compiler will not naively copy; it will construct in place. You should not
write `std::move` on a `return` of a local variable for performance — doing so
can *inhibit* NRVO in some cases. Just `return x;`.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

The fear of "returning a big object by value" is a leftover instinct from C,
where it really did copy. Modern C++ teaches you to return by value and let
the language elide. The result reads like Python — `auto v = make_value();` —
and runs without any of the copies the syntax seems to imply.

</div>

---

## The rule of five (and the rule of zero)

We have written five special member functions: destructor, copy constructor,
copy assignment, move constructor, move assignment. That set is the **rule of
five**: in modern C++, the rule of three becomes "if you write one of the
five, think about all five." For `Value`, all five are non-trivial and we
wrote them by hand.

But there is a *more important* rule, named in inversion to flatter the
preferred path: the **rule of zero**. It says: write classes that *don't* need
any of the five, by holding their resources in members that already manage
themselves. If `Value` held a `std::vector<std::uint8_t>` instead of a raw
`std::uint8_t*`, the compiler-generated everything would be correct — `vector`
already knows how to copy itself, move itself, and free itself. The body of
`Value` would shrink dramatically:

```cpp
// What a rule-of-zero Value would look like (illustrative)
class Value {
public:
    Value() = default;
    Value(const std::uint8_t* data, std::size_t size)
        : bytes_(data, data + size) {}

    const std::uint8_t* data() const { return bytes_.data(); }
    std::size_t         size() const { return bytes_.size(); }

private:
    std::vector<std::uint8_t> bytes_;
};
```

No destructor, no copy operators, no move operators — and it is correct. The
compiler does the right thing because `std::vector` does.

So why didn't we start there? Because the *point* of Chapter 3 is to know what
`std::vector` is doing for you. The hand-written `Value` taught you the rule of
five; the rule-of-zero version will be what you actually ship — after
Chapter 4, where we build our own `Buffer` and discuss `std::unique_ptr`. In
Kestrel, the final `Value` will hold a small owning buffer (an arena-aware
`Buffer`), but the lesson lands either way: in modern C++ you write the five
when you must, and you arrange your data members so you mustn't.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

There is no equivalent of the rule of five in JS or Python because there is no
deterministic destruction to plug into. In Python, `__del__` exists but is
discouraged precisely because nobody can tell you *when* it runs. C++ ties
cleanup to a known point — the closing brace — which is exactly what makes
rule-of-five classes powerful and rule-of-zero classes possible.

</div>

---

## Putting it together

Update `src/CMakeLists.txt` to compile `core/value.cpp` into a target you can
test (and against `kvd`, eventually). For now, just teach the test binary
about it:

```cmake
add_executable(value_test
  test/value_test.cpp
  core/value.cpp
)

add_test(NAME value_test COMMAND value_test)
```

A first test exercising the contract — empty, copy, move, content equality:

```cpp
// src/test/value_test.cpp
#include <cstring>
#include <utility>

#include "core/value.hpp"
#include "test/check.hpp"

int main() {
    using kestrel::Value;

    Value empty;
    CHECK(empty.size() == 0);
    CHECK(empty.data() == nullptr);

    const std::uint8_t bs[] = {1, 2, 3};
    Value a(bs, 3);
    CHECK(a.size() == 3);
    CHECK(std::memcmp(a.data(), bs, 3) == 0);

    Value b = a;                              // copy
    CHECK(b.size() == 3);
    CHECK(b.data() != a.data());              // independent buffer
    CHECK(std::memcmp(b.data(), bs, 3) == 0);

    Value c = std::move(b);                   // move
    CHECK(c.size() == 3);
    CHECK(std::memcmp(c.data(), bs, 3) == 0);
    CHECK(b.size() == 0);                     // moved-from contract
    CHECK(b.data() == nullptr);

    Value d;
    d = a;                                    // copy assignment
    CHECK(d.size() == 3 && std::memcmp(d.data(), bs, 3) == 0);

    d = std::move(c);                         // move assignment
    CHECK(d.size() == 3 && std::memcmp(d.data(), bs, 3) == 0);

    return kestrel::test::failures == 0 ? 0 : 1;
}
```

Build and run under sanitizers exactly as in Chapter 2:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

A clean run is your first real C++ object exercising every special member
function — under ASan, which would scream at the first leak or double-free.

<div class="callout callout-ask">

💬 **Ask Claude**

> Here is my `Value` class (`core/value.hpp`, `core/value.cpp`) and its test
> (`test/value_test.cpp`). Review it against the rule of five and the strong
> exception guarantee. In particular: is my copy assignment safe if `new`
> throws? Are my moves `noexcept`? Is there any path where the moved-from
> object could be observed in an invalid state? Where would `std::vector`
> change this code, and what would I lose by switching?

</div>

---

## Where this leaves Kestrel

The skeleton now has its first domain object. `Value` owns its bytes, copies
deeply, moves cheaply, and frees deterministically. Every later type in
Kestrel — `Buffer`, the store's slots, the protocol's responses — follows the
same shape: pick the right ownership model, write only the special members you
must, and let RAII do the cleanup.

The next chapter generalizes what we did here. `Value` hand-rolled a single
owning pointer; Chapter 4 introduces `std::unique_ptr`, `std::shared_ptr`, and
`std::weak_ptr` — the standard library's ownership primitives — and builds
Kestrel's `Buffer` on top of them. Once you know those, the hand-written
five-by-five will collapse into a rule-of-zero `Value` and you will see *why*
the rule-of-zero version is what professional C++ looks like day to day.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Before moving on, `value_test` must build and pass under ASan/UBSan with no
warnings (`-Wall -Wextra -Wpedantic`). Verify:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

You should also be able to:

- Explain why the compiler-synthesized copy operations are wrong for `Value`
  and what specifically goes wrong (double-free).
- State the rule of three and the rule of five, and *why* one implies the
  others — the link is ownership semantics, not coincidence.
- Distinguish copy from move at the source level (`b = a` vs
  `b = std::move(a)`) and at the cost level (deep copy vs pointer swap).
- Describe what `std::move` actually does (a cast, not an operation) and what
  state a moved-from `Value` is in.
- Sketch the rule-of-zero `Value` and say why it is preferable in real code.

Then hand it over for review:

> Review my Chapter 3 `Value`: `src/core/value.{hpp,cpp}` and
> `src/test/value_test.cpp`. Check the rule of five, exception safety in copy
> assignment, the moved-from invariant, and whether anything here would fight
> the language. I'm new to C++ — point out idioms I'm missing and antipatterns
> I've slipped into.

When that's clean, head to Chapter 4 — we replace the raw `new`/`delete[]`
with `std::unique_ptr` and build `Buffer`.

</div>
