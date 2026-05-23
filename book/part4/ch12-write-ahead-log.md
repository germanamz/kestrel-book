# The Write-Ahead Log

Up to this point, Kestrel exists entirely in memory: shut the process and
every key vanishes. Real key-value stores survive a crash and a restart by
writing each mutation to disk *before* (or alongside) applying it to memory.
The mechanism is called a **write-ahead log** — WAL — and it is the
foundation of every durable database from SQLite to Postgres to Redis itself
(in AOF mode). This chapter builds Kestrel's WAL.

The algorithm is straightforward. The C++ around it is not, because *durable
I/O* has more failure modes than the language tends to give you words for.
You will meet **`std::filesystem`**, the modern path-and-file APIs;
**`std::ofstream`**, why its default buffering quietly lies to you about
durability, and `fsync` — the system call that finally tells the operating
system to actually write your bytes to the disk's metal. You will write a
small **CRC32** checksum to catch torn writes, encode 64-bit lengths with
explicit endianness, and design a record format that is correct on a partial
write.

By the end, `src/persistence/` will contain a WAL that records every Kestrel
mutation and is robust to mid-write crashes. Chapter 13 will write the
snapshot/recovery flow that complements it.

---

## What "durable" actually means

When your program calls `write(fd, buf, n)` and the call returns success, the
bytes have been *handed to the kernel*. They are not yet on the disk; they
sit in the kernel's page cache, scheduled to be flushed at the kernel's
discretion (typically within seconds, but on a crash, before they reach the
platter — gone). The kernel returns success because that is the kernel's
contract: the data is now its problem. Whether *the disk* has it is a
separate, stronger guarantee.

The system call that turns "in the kernel" into "on the disk" is **`fsync`**
(POSIX) or `FlushFileBuffers` (Windows). It is slow — milliseconds, easily —
because it actually waits for the disk's physical state to match the bytes
you handed in. Production databases pay this cost selectively, usually once
per "commit point," not once per write.

So a WAL has three things to do, every time it records a mutation:

1. **Encode the mutation** into a self-describing record (length-prefixed,
   checksummed).
2. **Write the record** to the log file.
3. **`fsync` the log file**, *before* acknowledging the mutation to the
   client.

Without (3), a Kestrel that crashes after step (2) but before the kernel
flushed loses the very last writes — which is the exact case a WAL exists to
prevent. With (3), the cost of every mutation includes one disk sync. Redis
exposes this as `appendfsync always` (most durable, slowest) vs `everysec`
(syncs once per second; lose at most one second of writes; default and
much faster). Kestrel will support both modes through a configuration; we
start with `always` for safety.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`fs.writeFile` in Node and `open(path, 'w').write(data)` in Python both
default to "the bytes went somewhere; trust me." Neither defaults to fsync.
The Python `os.fsync(fileno)` and Node's `fs.fsyncSync(fd)` exist precisely
because of the gap above. You probably learned the words *file flush* and
*fsync* somewhere in your career, then promptly forgot them because
high-level languages defaulted to "good enough." A WAL is a small enough
contract that we cannot afford "good enough" — losing the last second of
writes after a crash is exactly what we are paid to prevent.

</div>

---

## `std::filesystem` and the modern path

C++17 introduced `<filesystem>` — paths as a proper type, directory
iteration, file metadata, all the things you have used `os.path` and
`fs.promises` for. Use it. The pre-C++17 alternatives (manually building
strings, calling POSIX directly) are riddled with portability landmines.

```cpp
#include <filesystem>
namespace fs = std::filesystem;

fs::path data_dir = "/var/lib/kestrel";
fs::path wal_path = data_dir / "kestrel.wal";
fs::create_directories(data_dir);
if (fs::exists(wal_path)) {
    auto sz = fs::file_size(wal_path);
    // ...
}
```

A path is *not* a string. The `/` operator does the right thing per
platform; `path::string()` and `path::generic_string()` give you string
views; functions like `parent_path()`, `stem()`, `extension()` save you from
hand-written tokenizers. We will use `fs::path` for every file path Kestrel
touches.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Several `<filesystem>` operations have *two* overloads — one that throws on
error and one that takes a `std::error_code&` and returns a status. The
throwing form is the default. For startup paths where a missing directory is
truly exceptional, throw. For runtime paths where a missing file is a
predictable case, take the error-code form and turn it into an
`std::expected<..., PersistenceError>`. Mixing the two leads to either
unexpected exceptions or silent failures.

</div>

---

## File streams and their honest behavior

C++'s file stream is `std::ofstream` (output) / `std::ifstream` (input). They
inherit from `std::ostream` / `std::istream`, so `<<`, `read`, `write` all
work. They handle buffering, formatting, conversion, and they are *not* the
right tool for a WAL.

The reason: streams buffer in **user space**, on top of the kernel's
buffering. A call to `os << "hello"` may end up in the stream's internal
buffer, never reaching the kernel until the buffer fills or you flush. Even
calling `os.flush()` only flushes the stream to the kernel — not the kernel
to the disk. For a WAL you want neither: you want the bytes in the kernel
*immediately* on write, and on the disk *immediately* on fsync. So we
bypass streams for the write path and call POSIX `write()` and `fsync()`
directly:

```cpp
// src/persistence/wal.hpp
#pragma once

#include <expected>
#include <filesystem>
#include <span>
#include <string>

namespace kestrel::persistence {

enum class FsyncMode { Always, EverySecond, Never };

struct PersistenceError { std::string what; };

class Wal {
public:
    static std::expected<Wal, PersistenceError>
    open(std::filesystem::path path, FsyncMode mode);

    Wal(const Wal&) = delete;
    Wal& operator=(const Wal&) = delete;
    Wal(Wal&&) noexcept;
    Wal& operator=(Wal&&) noexcept;
    ~Wal();

    std::expected<void, PersistenceError> append(std::span<const std::uint8_t> record);
    std::expected<void, PersistenceError> sync_now();

private:
    Wal(int fd, FsyncMode mode);
    int fd_ = -1;        // owned POSIX file descriptor
    FsyncMode mode_;
    std::chrono::steady_clock::time_point last_sync_;
};

}  // namespace kestrel::persistence
```

Two design notes already.

**`Wal` owns a file descriptor.** The destructor closes it; the move
operators transfer it; copy operations are deleted. This is RAII over a
POSIX resource, exactly as Chapter 1 promised. Chapter 14 will spin this
out into a generic `FileDescriptor` wrapper when sockets demand the same
pattern; for now, `Wal` does it inline.

**`open` is a static factory returning `expected`.** Constructors cannot
return errors except by throwing, which is wrong here — a missing data
directory at startup is exceptional, but Kestrel's runtime opens may fail
predictably (disk full, EROFS), and the type system should reflect that.
The factory pattern gives us a fallible constructor with a clean signature.

### The implementation

```cpp
// src/persistence/wal.cpp
#include "persistence/wal.hpp"

#include <fcntl.h>
#include <unistd.h>
#include <chrono>
#include <cstring>
#include <utility>

namespace kestrel::persistence {

std::expected<Wal, PersistenceError>
Wal::open(std::filesystem::path path, FsyncMode mode) {
    int fd = ::open(path.c_str(),
                    O_WRONLY | O_CREAT | O_APPEND,
                    0644);
    if (fd < 0) {
        return std::unexpected(PersistenceError{
            std::string("open ") + path.string() + ": " + std::strerror(errno)});
    }
    return Wal(fd, mode);
}

Wal::Wal(int fd, FsyncMode mode)
    : fd_(fd), mode_(mode), last_sync_(std::chrono::steady_clock::now()) {}

Wal::Wal(Wal&& o) noexcept
    : fd_(o.fd_), mode_(o.mode_), last_sync_(o.last_sync_) {
    o.fd_ = -1;
}

Wal& Wal::operator=(Wal&& o) noexcept {
    if (this == &o) return *this;
    if (fd_ >= 0) ::close(fd_);
    fd_ = o.fd_; mode_ = o.mode_; last_sync_ = o.last_sync_;
    o.fd_ = -1;
    return *this;
}

Wal::~Wal() {
    if (fd_ >= 0) ::close(fd_);
}

std::expected<void, PersistenceError>
Wal::append(std::span<const std::uint8_t> record) {
    std::size_t off = 0;
    while (off < record.size()) {
        ssize_t n = ::write(fd_, record.data() + off, record.size() - off);
        if (n < 0) {
            if (errno == EINTR) continue;             // interrupted; retry
            return std::unexpected(PersistenceError{
                std::string("write: ") + std::strerror(errno)});
        }
        off += static_cast<std::size_t>(n);
    }

    if (mode_ == FsyncMode::Always) return sync_now();
    if (mode_ == FsyncMode::EverySecond) {
        auto now = std::chrono::steady_clock::now();
        if (now - last_sync_ >= std::chrono::seconds(1)) return sync_now();
    }
    return {};
}

std::expected<void, PersistenceError> Wal::sync_now() {
    if (::fsync(fd_) != 0 && errno != EINVAL) {
        return std::unexpected(PersistenceError{
            std::string("fsync: ") + std::strerror(errno)});
    }
    last_sync_ = std::chrono::steady_clock::now();
    return {};
}

}  // namespace kestrel::persistence
```

A close read of the WAL's `append` repays the time. The loop around
`::write` accounts for **short writes**: POSIX is allowed to write fewer
bytes than you asked for, particularly when interrupted by a signal
(`EINTR`). Code that doesn't loop on short writes silently drops data when
signals fire — a classic Unix bug that gets you only after a long deploy.

`O_APPEND` makes writes atomic with respect to concurrent writers — the
kernel positions and writes in one step, so two threads writing the same fd
cannot interleave. Kestrel is single-threaded through Part 5, but the WAL
will outlive that constraint; `O_APPEND` is the right default regardless.

`fsync` can fail with `EINVAL` on file types that don't support it (some
test pseudo-files); we tolerate that for development.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`fsync` failure is a real, well-known problem (the "fsync gate" of 2018):
on some Linux configurations, `fsync` may return failure *and* clear the
in-kernel dirty state, meaning a retry returns success without the data
having been written. The genuinely safe response to `fsync` failure in a
WAL is to *crash the server* — anything else risks acknowledging writes
that aren't durable. Kestrel as written returns an error; a production
hardening would terminate the process on `fsync` failure. Either choice
should be deliberate.

</div>

---

## Record format

The bytes in the WAL need a self-describing structure so a recoverer can
parse them back. A small, robust record format:

```
+--------+--------+----------+-----------+
| length | crc32  | payload  |           |
| (u32)  | (u32)  | (length bytes)       |
+--------+--------+----------+-----------+
```

- `length` (4 bytes, big-endian): number of bytes in `payload`.
- `crc32` (4 bytes, big-endian): CRC-32 of the payload.
- `payload` (`length` bytes): the encoded command (we'll define the
  encoding in Chapter 13's snapshot work; for now, treat the payload as an
  opaque byte sequence the caller provides).

Big-endian on the wire because that is the network byte order convention
even on disk; once you make that choice across your project, future you
benefits from never having to remember which one this particular file uses.

The CRC32 is what makes the WAL **crash-safe** during a write. If the
process dies mid-write — say, half of `length`, all of `crc32`, none of the
payload — the next-startup recoverer will read the broken record, fail
the CRC check, and **truncate the log at that point**. Without the CRC, you
either have to write your record atomically (impossible past one block) or
trust the partial bytes (broken). With it, you have a self-validating
boundary every reader can find.

The C++ for it:

```cpp
// src/persistence/record.hpp
#pragma once

#include <bit>
#include <cstdint>
#include <span>

namespace kestrel::persistence {

std::uint32_t crc32(std::span<const std::uint8_t> bytes);

inline std::uint32_t to_be32(std::uint32_t x) {
    if constexpr (std::endian::native == std::endian::big) return x;
    else return std::byteswap(x);
}
inline std::uint32_t from_be32(std::uint32_t x) { return to_be32(x); }

}  // namespace kestrel::persistence
```

`std::byteswap` is C++23; before C++23 you wrote a manual swap.
`std::endian` is C++20. The `if constexpr` is resolved at compile time:
on big-endian hosts the function compiles to `return x`; on little-endian
it compiles to a single byte-swap instruction.

The CRC32 implementation itself is short. The classic polynomial table
fits in 1KB; the loop is one xor and one table lookup per byte. We will
not paste 256 hex constants into the book; the canonical table from RFC
1952 or a copy from any open-source database is fine. The point is that
CRC32 is *not* a cryptographic checksum — it catches accidental
corruption (torn writes, bit flips) with very high probability. For
intentional tampering you'd reach for a MAC; for crash safety, CRC32 is
exactly the right tool.

Writing a record:

```cpp
// in the dispatch layer that owns the Wal
auto write_record(Wal& wal, std::span<const std::uint8_t> payload) {
    std::vector<std::uint8_t> buf(8 + payload.size());
    std::uint32_t len_be = to_be32(static_cast<std::uint32_t>(payload.size()));
    std::uint32_t crc_be = to_be32(crc32(payload));
    std::memcpy(buf.data() + 0, &len_be, 4);
    std::memcpy(buf.data() + 4, &crc_be, 4);
    std::memcpy(buf.data() + 8, payload.data(), payload.size());
    return wal.append(buf);
}
```

Reading the WAL back at recovery time runs the inverse — read 8 bytes, check
the length is sensible (less than the file's remaining size), read that many
bytes, verify the CRC. If anything fails, that's the truncation point.
Chapter 13 will integrate this into the recovery flow.

<div class="callout callout-algo">

📐 **Algorithm**

**CRC32.** A linear-feedback shift register over GF(2) modulo the
polynomial `0xEDB88320` (reflected form of `0x04C11DB7`, the IEEE 802.3
polynomial). Implementation: 256-entry table of precomputed shifts; loop one
byte at a time, `crc = (crc >> 8) ^ table[(crc ^ byte) & 0xFF]`. O(N) over
the payload; about 1 GB/sec on modern CPUs without hardware acceleration.
Hardware-accelerated CRC32 instructions exist on x86-64 and ARMv8; we'll
note the optimization without taking it.

</div>

---

## A note on `std::ofstream` and when it's right

We bypassed file streams above. There are still uses for them in
persistence-adjacent code:

- **Reading the WAL** at startup — `std::ifstream` with `read()` is fine; we
  don't need byte-level control during recovery, only correctness.
- **Reading or writing config files** — text in, text out, line buffering
  is fine.
- **Test fixtures** that write known bytes to a temporary file.

The rule is: when *durability* matters, go straight to POSIX. When you just
want bytes on disk and don't care if they survive a power cut by 50ms,
streams are fine and more ergonomic.

<div class="callout callout-experiment">

🧪 **Experiment**

Demonstrate the cost of `fsync`. Run two WAL benchmarks: one with
`FsyncMode::Always` and one with `FsyncMode::Never`, each writing a
million small records (say, 64 bytes each). Predict the ratio: how much
slower is `Always`? On a typical SSD you should see `Always` deliver
single-digit thousands of records per second, and `Never` deliver
millions. Then switch to `EverySecond` and measure again — you should see
durability cost most of what `Never` saved you. This is the durability
tradeoff in concrete numbers, and it is what production database
operators argue about.

</div>

<div class="callout callout-ask">

💬 **Ask Claude**

> Here is my `Wal` and record format (`persistence/wal.{hpp,cpp}`,
> `persistence/record.{hpp,cpp}`). Review: (1) the `::write` loop — am I
> handling `EINTR` and short writes correctly? (2) the move semantics — is
> there any path where two `Wal` instances could share an fd or where I'd
> double-close it? (3) the record format — is 4-byte length enough, or
> should I use 8? (4) is `fsync` the right call on macOS, or should I be
> using `fcntl(F_FULLFSYNC)` for stronger durability?

</div>

---

## Where this leaves Kestrel

`src/persistence/wal.cpp` writes Kestrel's first byte to disk. Every
mutating command — `SET`, `DEL`, `EXPIRE`, `LPUSH`, etc. — will, at the
dispatch layer, serialize its arguments into a record and append it to the
WAL before applying to the in-memory keyspace. The WAL is the source of
truth: if the process dies, what's in the WAL is what the keyspace
remembers; what isn't, isn't.

Chapter 13 makes that recoverable end-to-end. It builds:

- A **snapshot format** that dumps the entire keyspace to disk as a single
  consistent file.
- **Log compaction**: rotate the WAL after a snapshot so the file doesn't
  grow forever.
- **Replay**: on startup, load the latest snapshot, then apply each WAL
  record after the snapshot to reach the pre-crash state.

After Chapter 13, Kestrel is durable. After that, Part 5 connects it to a
network — and the lessons of this chapter (RAII over file descriptors,
fallible operations as `expected`, blocking on I/O) carry directly to
sockets.

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and tests green under ASan/UBSan:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

Write tests that:

- Create a `Wal` in a temporary directory, append a few records, close it,
  and verify the file's on-disk size equals `8 * count + sum_of_lengths`.
- Corrupt the last byte of the file by hand and confirm a parser that reads
  it back rejects the last record (CRC mismatch) while accepting the
  earlier records.

You should be able to:

- Explain why `write` returning success does not mean the data is on disk.
- Describe what `O_APPEND` guarantees and why it matters here.
- Defend `fsync` mode `Always` versus `EverySecond` in terms of the
  durability/throughput tradeoff.
- Read the WAL record format and explain why each field is there.

Hand it over:

> Review my Chapter 12 WAL (`src/persistence/wal.{hpp,cpp}`,
> `persistence/record.{hpp,cpp}`). Are my error paths returning typed
> `PersistenceError`s consistently? Is the move-semantics of `Wal` correct
> (no double-close, valid moved-from state)? Are there places I should be
> calling `fsync` that I'm not? Would you change the record format before
> committing to it for Chapter 13?

When that's clean, Chapter 13 builds snapshots and recovery on top.

</div>
