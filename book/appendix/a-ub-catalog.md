# Appendix A — UB Catalog

A reference of the undefined behaviors most likely to bite Kestrel and
code like it. Each entry names the rule, gives a minimal example,
explains *why* the standard considers it UB, and lists the mitigation.
Use this as a checklist when reviewing your own code or someone else's.

The list is selective, not exhaustive — the C++ standard has hundreds
of UB clauses. These are the ones you will actually encounter.

---

## Memory & pointer hazards

### Reading past the end of an array

```cpp
int xs[3] = {1, 2, 3};
int v = xs[3];     // UB — index 3 is one past the last valid index (2).
```

**Why UB:** The standard places no constraints on the contents at `xs +
3`. The compiler is free to assume `i < 3` in surrounding code; ASan
catches a stack-buffer-overflow report.

**Mitigation:** `std::array<int, 3>` plus `.at(i)` for bounds-checked
access; `std::span` to carry pointer + length; ASan during testing.

### Use-after-free

```cpp
auto* p = new int(5);
delete p;
int v = *p;        // UB — p points at freed memory.
```

**Why UB:** Freed memory may be reused, the allocator may unmap the
page, the compiler may have proven the load never happens.

**Mitigation:** `std::unique_ptr`, `std::shared_ptr`. Avoid raw `new` /
`delete`. ASan catches use-after-free reliably.

### Double-free

```cpp
delete p;
delete p;          // UB — second delete on the same pointer.
```

**Why UB:** The allocator's free list is corrupted; the next `new` may
return a pointer to freed memory.

**Mitigation:** RAII. A `unique_ptr` that has been moved-from holds
`nullptr` and the second destructor is a no-op.

### Dangling pointer / reference / span

```cpp
std::string_view get_label() {
    std::string s = "hello";
    return s;      // UB — the string dies at the closing brace.
}
```

**Why UB:** The `string_view` points into storage that has been
released.

**Mitigation:** Return `std::string` by value; let copy elision
optimize. Treat `string_view` and `span` as non-owning *parameters*,
not as fields. If you must store one, document the lifetime contract.

### Mismatched `new` / `delete`

```cpp
auto* xs = new int[10];
delete xs;         // UB — should be `delete[] xs;`.
```

**Why UB:** Array `new` and scalar `new` use different allocation
paths; using the wrong free corrupts the allocator.

**Mitigation:** Use `std::vector<T>` or `std::unique_ptr<T[]>`; never
write raw `new[]`.

### Null pointer dereference

```cpp
int* p = nullptr;
int v = *p;        // UB.
```

**Why UB:** No memory at address 0 (typically); even when there is,
reading it is not defined behavior.

**Mitigation:** Check for null before dereferencing; prefer references
where null isn't a possibility; `std::optional<T>` to make "absence"
explicit.

### Uninitialized read

```cpp
int x;
int y = x + 1;     // UB — x is uninitialized.
```

**Why UB:** The value of `x` is "indeterminate" — anything, including
trap representations on some architectures.

**Mitigation:** Always initialize. Modern habits: `int x = 0;` or
`int x{};` or `auto x = compute();`. `-Wuninitialized` warns when the
compiler can see it.

---

## Arithmetic hazards

### Signed integer overflow

```cpp
int x = INT_MAX;
int y = x + 1;     // UB.
```

**Why UB:** The standard does not define what happens on overflow for
signed types (unlike unsigned, where wrap is defined).

**Mitigation:** Use unsigned for values that may legitimately overflow
(checksums, hash mixers, counters that count beyond `int`). Use
`std::saturate_cast` (C++26) or explicit bounds checks for arithmetic
on user-supplied numbers. UBSan catches signed overflow.

### Division by zero

```cpp
int x = 1 / 0;     // UB for integer; defined (inf) for floating-point.
```

**Mitigation:** Check the divisor before dividing.

### Shift by too much

```cpp
int x = 1 << 32;   // UB if int is 32 bits.
```

**Why UB:** The standard requires the shift count to be less than the
bit width of the type.

**Mitigation:** Mask with `& 31` (for 32-bit) or check the bound
explicitly; UBSan catches it.

---

## Object lifetime hazards

### Reading a moved-from object's value

```cpp
std::string a = "hello";
std::string b = std::move(a);
// Reading a's content here is technically defined for std::string
// (it'll be "valid but unspecified"), but for your own types the
// contract is yours to set. Reading beyond destruction is UB.
```

**Mitigation:** Don't read a moved-from value as if it held data.
Re-assign it before use, or treat it as destructible-only.

### Object destruction order

```cpp
struct A { ~A() { /* uses B */ } } a;
struct B { /* ... */ } b;
// b is destroyed before a at program exit (reverse declaration order),
// but if a's destructor uses b, it touches a dead object.
```

**Mitigation:** Don't have destructors of static objects depend on
other static objects. The "static initialization order fiasco" is a
known C++ trap.

### Returning reference to a temporary

```cpp
const int& f() {
    return 42;     // UB — the temporary `42` dies at the return.
}
```

**Mitigation:** Return by value. Compilers warn (`-Wreturn-local-addr`).

### Self-move into a non-self-aware operator

```cpp
v = std::move(v);  // UB if operator=(&&) does not guard against self-move.
```

**Mitigation:** Write `if (this == &other) return *this;` at the top
of move assignment, or design the operator to be self-safe (e.g.,
swap-based).

---

## Concurrency hazards

### Data race

```cpp
int counter = 0;
// Thread A:  ++counter;
// Thread B:  std::cout << counter;
// UB — non-atomic access to the same memory from two threads,
// at least one writing, not synchronized.
```

**Mitigation:** `std::atomic<int>` for shared counters; `std::mutex`
for compound state; TSan as a regression test.

### Non-atomic stop flags

```cpp
bool stop = false;
// Thread A loops: while (!stop) { ... }
// Thread B sets stop = true.
// UB — compiler may hoist the load and never see the update.
```

**Mitigation:** `std::atomic<bool>` for any flag toggled across
threads. `std::jthread::stop_token` covers the common case.

### Locking different orderings

```cpp
// Thread A: lock(a); lock(b);
// Thread B: lock(b); lock(a);
// Deadlock (not UB strictly, but a hang).
```

**Mitigation:** Acquire locks in a globally consistent order, or use
`std::scoped_lock(a, b)` which uses a deadlock-avoidance algorithm.

---

## Compilation & ODR hazards

### One Definition Rule violation

```cpp
// file1.cpp:  struct S { int x; };
// file2.cpp:  struct S { int x, y; };   // Same name, different layout — UB.
```

**Why UB:** The linker may pick either definition; callers compiled
against one may receive an object laid out according to the other.

**Mitigation:** Define types once in a header; include the header
everywhere. For type-private helpers, use anonymous namespaces or
`static`.

### Inconsistent build flags across TUs

```cpp
// One .cpp built with -std=c++17, another with -std=c++23.
// Templates may instantiate to subtly different layouts.
```

**Why UB:** Different language versions or `_GLIBCXX_DEBUG` settings
can change the size and layout of standard library templates.

**Mitigation:** Set flags once at the top of CMake; never pass
per-target overrides for language or runtime flags.

---

## Casting & aliasing hazards

### Strict aliasing violation

```cpp
float f = 1.0f;
int   i = *reinterpret_cast<int*>(&f);    // UB — strict aliasing.
```

**Why UB:** The standard forbids accessing memory of one type through
a pointer-to-different-type, except via `char*`, `unsigned char*`, and
`std::byte*`.

**Mitigation:** `std::memcpy(&i, &f, sizeof i);` (the compiler folds
this to one instruction). `std::bit_cast<int>(f)` from C++20 when both
are trivially copyable.

### Misaligned access

```cpp
char buf[8];
auto* p = reinterpret_cast<std::int64_t*>(buf + 1);  // probably misaligned
auto v = *p;     // UB on architectures that require alignment.
```

**Mitigation:** `std::memcpy` for unaligned loads; `alignas(8)` on the
buffer if you want it aligned at the source.

### Reinterpret-casting unrelated types

```cpp
struct A { int x; };
struct B { int x; };
A a;
B* b = reinterpret_cast<B*>(&a);    // UB on any read of *b.
```

**Mitigation:** Almost never `reinterpret_cast` outside of network /
file I/O paths. The exceptions (e.g., `sockaddr_in*` to `sockaddr*`)
are documented in the POSIX API.

---

## Standard library hazards

### Dereferencing an end iterator

```cpp
auto it = v.end();
auto x = *it;    // UB.
```

**Mitigation:** Check `it != v.end()` before dereferencing. Range-for
covers most cases.

### Iterator invalidation

```cpp
std::vector<int> v = {1,2,3};
auto it = v.begin();
v.push_back(4);   // may reallocate; `it` is now dangling.
auto x = *it;     // UB.
```

**Mitigation:** Don't hold iterators across mutations. Each
container's docs list which operations invalidate which iterators;
`vector::push_back` invalidates all of them on capacity growth.

### Calling member on null `std::optional` / `std::expected`

```cpp
std::optional<int> o;
int v = *o;       // UB — operator* on an empty optional.
```

**Mitigation:** Check `if (o)` before dereferencing; or use `o.value()`
which throws (not UB).

---

## The escape hatch: sanitizers

Almost every UB above is caught by **AddressSanitizer**,
**UndefinedBehaviorSanitizer**, or **ThreadSanitizer** at runtime
*provided the buggy path executes during the test*. The rule from
Chapter 2 holds for the whole book: run your test suite under
sanitizers; the bugs they cannot catch (logical bugs) are the ones your
tests should be designed to provoke.

For more, see Appendix B's sanitizer guide.
