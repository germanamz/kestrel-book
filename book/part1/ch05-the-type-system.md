# The Type System

So far, a Kestrel `Value` is one thing: a bag of bytes. Real Redis values are
several things — strings, lists, hashes, sorted sets — and the store needs to
hold any of them under a single name. In Python you would shrug and put any
object into a `dict[str, Any]`; in JavaScript the value just *is*. C++ has no
runtime to track which variant a value happens to be at the moment you look at
it, so the language has to encode that choice statically, in the type system
itself.

This chapter is about the part of C++ that lives entirely at compile time:
**templates**, **concepts**, **type traits**, and **`std::variant`**. Each
solves a different problem, and together they are what makes it possible to
write a single function that handles strings, lists, hashes, and sorted sets
correctly — without runtime type checks, without virtual dispatch, and with
the compiler refusing every call that doesn't make sense.

By the end, `Value` will be a `std::variant` over the four Kestrel types, and
you will have written your first templates with constraints. Part 1 ends here;
Part 2 starts speaking RESP on the wire and immediately leans on the templates
we are about to introduce.

---

## Templates: generic code, generated per type

A C++ template is **not** a generic type in the Java/Rust sense — it is a
*code-generation directive* the compiler runs at compile time. You write a
template once; for every distinct set of template arguments you actually use,
the compiler stamps out a fresh, fully-typed copy. The result is that template
code is as fast as hand-written code (everything is resolved before the
program runs) and as flexible as duck-typed code (any type that satisfies the
template's textual requirements compiles).

The trade-off, which you will feel immediately: **errors happen where the
template is instantiated, not where it is defined.** A template that compiles
in isolation may produce a hundred-line error when you instantiate it with an
unexpected type. C++20's **concepts** exist precisely to turn those late
errors into early, readable ones; we will reach them in a moment.

A first template, the most useful one you'll write this chapter, picks the
smaller of two values:

```cpp
template <typename T>
T const& min(T const& a, T const& b) {
    return a < b ? a : b;
}
```

`typename T` is a *template parameter*. When you call `min(3, 5)`, the compiler
deduces `T = int` and generates a real function `min<int>` whose body is what
you wrote. Call `min(std::string("a"), std::string("b"))` and it generates
another, completely separate `min<std::string>`. Two functions in the
executable, one in the source.

The template will compile for *any* `T` that supports `operator<` — which is
both the strength and the weakness. Pass a type without `<` and the error
arrives buried in the template's body. Worse, pass a type with a `<` that
does the wrong thing (e.g. raw pointers, where `<` compares addresses, not
contents) and the template silently does the wrong thing.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

A Python `def min(a, b): return a if a < b else b` does the same job at run
time, with a `TypeError` if `<` isn't defined. The C++ version does it at
*compile* time and produces a separate, specialized function per type — no
dispatch, no boxing, no runtime cost. The price is that the error, when it
comes, comes from the compiler, not at the call site. Concepts (below) close
that gap.

</div>

### Templates over classes

Templates apply to classes too. Kestrel's `Buffer` from Chapter 4 stored
`std::uint8_t`. Most of the time that's what you want, but the store will
also need byte buffers parameterized differently in later chapters. The
generalization is mechanical:

```cpp
template <typename T>
class TypedBuffer {
public:
    explicit TypedBuffer(std::size_t n) : data_(std::make_unique<T[]>(n)), size_(n) {}
    T*       data()       { return data_.get(); }
    const T* data() const { return data_.get(); }
    std::size_t size() const { return size_; }

private:
    std::unique_ptr<T[]> data_;
    std::size_t size_ = 0;
};
```

`Buffer` is just `TypedBuffer<std::uint8_t>`. We will not actually rewrite
`Buffer` to be a template — there is no need yet — but the mental model is
that any place you see a concrete type in a class, you could swap in a
template parameter and stamp out variations.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Templates are usually defined entirely in headers. The reason traces back to
Chapter 1: the compiler can only generate code for `TypedBuffer<int>` when it
sees both the template definition *and* the instantiation. Splitting a
template's definition into a `.cpp` file works only if every instantiation it
needs is named in that same `.cpp`, which is brittle. The boring rule that
prevents the entire class of problems: write templates in headers (or in
`.ipp`/`.tpp` files included by the headers). We will do this throughout
Kestrel.

</div>

---

## Concepts: turning template errors into messages

C++20 lets you constrain templates with **concepts** — named predicates the
compiler checks before instantiation, producing one clean error instead of
the inscrutable cascade you get from a template body that fails partway.

A concept for "this type has `operator<`":

```cpp
#include <concepts>

template <typename T>
concept Ordered = requires(T a, T b) {
    { a < b } -> std::convertible_to<bool>;
};

template <Ordered T>
T const& min(T const& a, T const& b) {
    return a < b ? a : b;
}
```

The `requires` clause says: "for some `T a, T b`, the expression `a < b` must
be valid and its result must convert to `bool`." Call `min` with a type that
doesn't satisfy `Ordered`, and the compiler stops at the call site with a
message naming the concept and what failed — not a hundred lines deep in the
template body.

Concepts are *evaluated at compile time*, just like the rest of the template
machinery. They are how modern C++ writes generic code that fails loudly and
fast. We will use a small set in Kestrel — `std::ranges::range` for code that
walks any sequence, `std::integral` for code that takes any integer type — and
otherwise borrow concepts from `<concepts>` and `<ranges>` rather than invent
our own.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

If you have used TypeScript's `extends` constraints (`<T extends { foo: ... }>`)
or Python's `Protocol`, concepts will feel familiar — they're the static
duck-typing predicate you wished those languages enforced more strictly. The
difference is that the C++ check happens entirely at compile time and never
costs anything at run time. The template instantiation with a satisfying type
runs exactly as fast as the hand-written version.

</div>

---

## Type traits: asking the compiler about a type

The standard `<type_traits>` header is a bag of compile-time queries about
types: `std::is_integral_v<T>`, `std::is_same_v<T, U>`,
`std::is_trivially_copyable_v<T>`, and dozens more. Each is evaluated by the
compiler when you instantiate the template; the value is `true` or `false`
*at compile time*, available to `static_assert`, to `if constexpr`, and to
concept expressions.

A representative use, the kind you will write a few times in Kestrel:

```cpp
template <typename T>
T deserialize(std::span<const std::uint8_t> bytes) {
    static_assert(std::is_trivially_copyable_v<T>,
                  "deserialize<T> requires T to be trivially copyable");
    T value;
    std::memcpy(&value, bytes.data(), sizeof(T));   // safe iff trivially copyable
    return value;
}
```

`static_assert` fires at compile time if the predicate is false, with the
message you supplied. That is the kind of error you want: an explanation, at
the line of code that made the mistake, naming the assumption it violated.
This is the *opposite* of UB — a check the compiler is guaranteed to run.

C++ also gives you **`if constexpr`**, which is `if` evaluated by the
compiler: branches it discards are not even type-checked. This lets one
template body specialize its behavior based on its type:

```cpp
template <typename T>
std::size_t encoded_size(T const& v) {
    if constexpr (std::is_integral_v<T>) {
        return sizeof(T);
    } else if constexpr (std::is_same_v<T, std::string>) {
        return v.size() + 2;     // length prefix etc.
    } else {
        static_assert(sizeof(T) == 0, "encoded_size: unsupported type");
    }
}
```

The interesting trick is the `static_assert(sizeof(T) == 0, ...)` in the
`else` branch — `sizeof(T) == 0` is always false for real types, so the
assertion fires for any `T` not explicitly handled. Putting it *inside* an
`if constexpr` else means the assertion only fires if you actually instantiate
the unsupported branch, not for every translation unit that sees the template.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

This is `isinstance` lifted to compile time. Where Python checks the type at
the moment the line runs, C++ checks it while compiling and *removes* the
branches that don't apply. There is no runtime type tag, no dispatch — by the
time the binary exists, only the relevant branch is in it.

</div>

---

## `std::variant`: a tagged union you can trust

A Kestrel value can be one of four things: a string, a list, a hash, or a
sorted set. In C, you would write a tagged union — a `struct` with an `enum`
tag and a `union` of payloads, and discipline. C++ gives you that pattern as
a type: **`std::variant<Ts...>`**, a value that, at any moment, holds exactly
one of its alternatives, knows which one, and destroys the right alternative
when it dies.

The skeleton — we'll define the missing types, but you can already see the
shape:

```cpp
#include <variant>

namespace kestrel {

class String;       // forward declarations — defined below or in Part 3
class List;
class Hash;
class SortedSet;

using Value = std::variant<String, List, Hash, SortedSet>;

}  // namespace kestrel
```

`Value` here is no longer a class we wrote — it is a `std::variant` alias.
That's the typical move: collapse a hand-rolled tagged union into a `variant`
and let the standard library handle storage, destruction, and dispatch.

### Constructing and inspecting a variant

You construct a variant by passing one of its alternatives:

```cpp
Value v = String(/*...*/);            // holds a String
Value w = List(/*...*/);              // holds a List
```

You inspect it by index or by type:

```cpp
if (std::holds_alternative<String>(v)) {
    const String& s = std::get<String>(v);    // throws bad_variant_access if wrong
    // ...
}
```

`std::get<T>` throws if the variant currently holds a different alternative.
For non-throwing access, `std::get_if<T>(&v)` returns a pointer to the value
or `nullptr`. Use `get_if` when "wrong type" is a plausible runtime case;
`get` when you know — by checking — that the type is right.

### Visitors: dispatching at compile time

The idiomatic way to handle every alternative is `std::visit` with a callable
that overloads on the alternatives:

```cpp
struct ToWire {
    void operator()(const String& s)    const { /* serialize string */ }
    void operator()(const List& l)      const { /* serialize list */ }
    void operator()(const Hash& h)      const { /* serialize hash */ }
    void operator()(const SortedSet& z) const { /* serialize zset */ }
};

std::visit(ToWire{}, v);
```

The compiler picks the right overload from `ToWire` based on what `v` holds,
with no virtual dispatch. The dispatch is a jump table generated at compile
time. `visit` will refuse to compile if `ToWire` cannot accept *every*
alternative — meaning if you later add a fifth Kestrel type to `Value`, every
visitor in the codebase will fail to build until you handle it. That is the
correctness guarantee an `enum`+`union` pattern in C cannot give you and the
reason `variant` exists.

For one-off visitors there's a tidy idiom using a `struct` with multiple
`operator()` overloads (above), or `if constexpr` inside a single generic
lambda:

```cpp
std::visit([](auto const& x) {
    using T = std::decay_t<decltype(x)>;
    if constexpr (std::is_same_v<T, String>)      { /* ... */ }
    else if constexpr (std::is_same_v<T, List>)   { /* ... */ }
    // ... etc.
}, v);
```

Either form is fine. The `struct`-of-overloads form is more readable for
visitors of any real size; the generic-lambda form is convenient for one-off
inspection.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`std::variant<A, B, C>` is the typed cousin of Python's `Union[A, B, C]` —
except you can't *forget* to handle a case, because `std::visit` won't compile
without the full coverage. It's a sum type that the type checker enforces, in
a language that otherwise lets you do all the unsafe things. Use it whenever
you would have reached for a `dict[str, str]` to fake polymorphism.

</div>

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

A *moved-from* variant is in a "valueless by exception" state if a move
construction threw. Calling `std::get` on it then throws
`std::bad_variant_access`; calling `std::visit` is UB unless the variant
holds an alternative. The everyday rule: don't read a moved-from variant
without re-assigning it first. As with `Value` in Chapter 3, treat moved-from
values as destructible but unreadable.

</div>

---

## Kestrel's Value as a variant

We will define `String`, `List`, `Hash`, and `SortedSet` as concrete types
over the next few chapters; Chapter 11 builds the list and sorted set
properly, Chapter 9 builds the hash table. For Part 1 we want the *shape* of
`Value` settled so Part 2's RESP code can use it, with placeholder bodies for
the types we have not implemented yet.

The string variant we *have* built — it is Chapter 4's `Value`, renamed to
`String` to free the name `Value` for the variant:

```cpp
// src/core/string.hpp  (renamed from core/value.hpp; same body, new name)
#pragma once

#include <cstddef>
#include <cstdint>
#include <cstring>
#include <span>

#include "core/buffer.hpp"

namespace kestrel {

class String {
public:
    String() = default;
    explicit String(std::span<const std::uint8_t> bytes)
        : storage_(bytes.size())
    {
        if (!bytes.empty()) std::memcpy(storage_.data(), bytes.data(), bytes.size());
    }

    std::span<const std::uint8_t> bytes() const { return storage_.bytes(); }
    std::size_t size() const { return storage_.size(); }
    const std::uint8_t* data() const { return storage_.data(); }

private:
    Buffer storage_;
};

}  // namespace kestrel
```

The other three are *forward-declared* stubs for now — empty classes whose
real bodies live in Part 3:

```cpp
// src/core/value.hpp
#pragma once

#include <variant>

#include "core/string.hpp"

namespace kestrel {

// Implementations land in later chapters; rule-of-zero shells suffice for Part 1.
class List {};
class Hash {};
class SortedSet {};

using Value = std::variant<String, List, Hash, SortedSet>;

}  // namespace kestrel
```

This pattern — declare the shape now, fill in bodies later — is a habit C++
rewards. Once the variant's alternatives are named, every visitor in the
codebase has its set of cases pinned; Part 3 only fills in what's inside the
cases, never their identity.

<div class="callout callout-experiment">

🧪 **Experiment**

Write a test that constructs a `Value` holding a `String`, then `std::visit`s
a `struct` of overloads that returns the type's name as a `std::string_view`.
Confirm it prints `"String"`. Now add a fifth alternative (`class Foo {};`) to
the variant and rebuild without updating the visitor. Read the compiler error
carefully — it names the missing overload. Remove `Foo`; the build is clean.
That refusal to compile *is* the safety of `std::variant` working for you.

</div>

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's `core/string.hpp`, `core/value.hpp`, and my test using `std::visit`.
> Explain in your own words why `std::visit` is preferred over a chain of
> `holds_alternative` / `get` checks. Walk me through what happens at compile
> time when I `std::visit` over a `Value`: how does the compiler pick which
> overload to call, and is there any runtime cost? If I want to handle only
> `String` specially and pass the rest to a default, what's the idiomatic
> form?

</div>

---

## Where Part 1 ends

Kestrel's data model is now type-complete:

```text
src/core/
  buffer.{hpp,cpp}    # owning byte block
  string.hpp          # the bytes-variant (rule of zero)
  value.hpp           # std::variant<String, List, Hash, SortedSet>
```

Every later part of Kestrel speaks in terms of these types. Part 2 implements
the RESP protocol, where a serializer is a templated function over the
variant's alternatives — exactly the use case the type-system tour was for.
Part 3 fills in `List`, `Hash`, and `SortedSet` with real bodies. Part 4
serializes `Value`s to disk through the same template machinery. By the time
you reach Chapter 17, the data layer in front of you is *just types* — no
runtime tagging, no virtual dispatch, no boxing — and it is the type system
that holds it together.

You have also acquired the C++ tools that produce that result. Templates
generate code per type. Concepts fail loudly when a type is wrong. Type traits
and `static_assert` let you state invariants the compiler checks for free.
`std::variant` plus `std::visit` give you closed-set polymorphism with
exhaustiveness enforced. These four ideas reappear everywhere from here.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

You should be able to build and test cleanly under ASan/UBSan:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

Make sure you can:

- State, in one sentence each, what templates, concepts, type traits, and
  `std::variant` are *for* — and how they differ.
- Explain why templates live in headers, in terms of Chapter 1's translation
  units.
- Read a `requires` clause and say what it constrains.
- Use `std::visit` and explain how the dispatch is implemented.
- Predict what happens when a new alternative is added to a `variant` whose
  visitors do not yet handle it.

Hand it to Claude:

> Review my Chapter 5 code: `src/core/string.hpp`, `src/core/value.hpp`, and
> the visitor test. Am I using `std::variant` idiomatically? Are my forward
> declarations sound, or should `List`/`Hash`/`SortedSet` live in their own
> headers? Suggest one concept I should introduce now that will pay off in
> Part 2.

When that's clean, Part 1 is finished and Part 2 begins — Kestrel goes on the
wire.

</div>
