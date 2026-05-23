# Snapshots & Recovery

The WAL from Chapter 12 captures every mutation, durably, in append order.
That is enough to recover a crashed Kestrel by *replaying* every record from
the beginning of time — but the WAL grows forever, and recovery time grows
with it. Production stores combine the WAL with periodic **snapshots**: a
point-in-time dump of the entire keyspace to a single file, after which the
WAL can be **compacted** (truncated to records that arrived *after* the
snapshot). Startup loads the most recent snapshot, then replays the tail of
the WAL to reach the state at the moment of crash.

This chapter builds three things on top of Chapter 12's foundation: a
**snapshot writer** that serializes the entire keyspace to disk, a
**recoverer** that loads a snapshot and replays a WAL, and the **rotation
protocol** that ties them together safely — including the "atomic rename"
trick that prevents a half-written snapshot from being mistaken for a good
one. The C++ topics are **binary serialization**, the **`std::expected`**
discipline applied at scale, and the design pattern of **atomic file
publication** through `rename`.

By the end, Kestrel survives a crash. Restart it, watch the snapshot load
and the WAL tail replay, and the keyspace is back where it was at the last
acknowledged write.

---

## What goes into a snapshot

A snapshot is the entire keyspace, serialized to one self-describing file.
It must contain enough information to recreate every `(key, Value)` pair,
including each value's *type* (variant alternative) and the expiry, if any.

A compact binary format that mirrors the WAL's record style works well:

```
+--------+--------+-------------------+
| magic  | version| record* (until EOF)|
| u32    | u32   |                    |
+--------+--------+-------------------+
```

Each record is one of:

```
record:
  +--------+--------+--------+
  | length | crc32  | body   |
  | u32    | u32    | length |
  +--------+--------+--------+

body:
  +-----+--------+--------+--------+-----------+--------+
  | type| key len| key    | exp_ns | value type| value  |
  | u8  | u32 BE | bytes  | u64 BE | u8        | varies |
  +-----+--------+--------+--------+-----------+--------+
```

- `magic` (4 bytes): an ASCII tag like `'K','S','N','P'` ("Kestrel
  Snapshot") so a recoverer can recognize the file format before trusting
  any of its bytes.
- `version` (4 bytes): the format version, so future Kestrel can refuse
  forward-incompatible snapshots cleanly.
- `type` of each record is reserved for future extension (multiple record
  kinds per snapshot); for now everything is `0x01` = "keyspace entry."
- `exp_ns` is the absolute steady-clock-equivalent time in nanoseconds, or
  `0` if no expiry. (Storing relative ms-from-now is wrong: snapshots may
  be loaded much later than they were taken. Storing
  steady-clock-equivalent absolute time has its own problem — `steady_clock`
  has no calendar relationship — so on the disk we actually convert TTLs to
  durations from "snapshot-taken-at" wall time and convert back on load.
  We'll outline this in the experiments.)
- `value type` is the `std::variant` index (`0 = String`, `1 = List`,
  `2 = Hash`, `3 = SortedSet`).
- `value` is the per-type encoding — for `String`, length-prefixed bytes;
  for `List`, length-prefixed count followed by length-prefixed strings;
  for `Hash`, count plus field/value bulks; for `SortedSet`, count plus
  member/score (`f64`) pairs.

The CRC32 per record means recovery can tell a clean truncation from a
torn last record — the same trick as the WAL.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

You may want to reach for JSON. Don't. JSON cannot represent arbitrary
bytes without base64 encoding (Kestrel values are binary-safe), is
self-describing in a way you do not need (the snapshot format is fixed),
and is 4–10× larger and slower than a hand-rolled binary form. JSON is
right for human-edited config; the snapshot is for the machine only.

</div>

---

## Serializing a `Value` (with `std::visit` finally paying off)

Chapter 5 set up `std::variant` and `std::visit` precisely so that this
function is small:

```cpp
// src/persistence/snapshot.cpp
void write_value(std::vector<std::uint8_t>& buf, const Value& v) {
    buf.push_back(static_cast<std::uint8_t>(v.index()));   // variant tag

    std::visit([&](auto const& alt) {
        using T = std::decay_t<decltype(alt)>;
        if constexpr (std::is_same_v<T, String>) {
            write_u32_be(buf, alt.size());
            append_bytes(buf, alt.data(), alt.size());
        } else if constexpr (std::is_same_v<T, store::List>) {
            write_u32_be(buf, alt.size());
            // Walk the list in order, writing each element as a bulk.
            // (Implementation detail of List: an iterator or
            // a public `for_each` method to walk it without exposing layout.)
        } else if constexpr (std::is_same_v<T, store::Hash>) {
            write_u32_be(buf, alt.size());
            for (auto const& [k, v] : alt.fields()) {
                write_u32_be(buf, k.size()); append_bytes(buf, k);
                write_u32_be(buf, v.size()); append_bytes(buf, v);
            }
        } else if constexpr (std::is_same_v<T, store::SortedSet>) {
            // similar: count, then (member, score) pairs
        } else {
            static_assert(sizeof(T) == 0, "write_value: unhandled variant");
        }
    }, v);
}
```

The `static_assert(sizeof(T) == 0)` in the else branch is the catch-all
that fires only if a new `Value` alternative slipped in without a handler —
the compiler check we built in Chapter 5. *This* function is the original
motivating use case.

Reading it back is the dual:

```cpp
std::expected<Value, PersistenceError>
read_value(std::span<const std::uint8_t>& cursor) {
    if (cursor.empty()) return std::unexpected(PersistenceError{"short read: tag"});
    std::uint8_t tag = cursor[0];
    cursor = cursor.subspan(1);
    switch (tag) {
        case 0: { /* String */
            std::uint32_t len;
            if (auto e = read_u32_be(cursor, len); !e) return std::unexpected(e.error());
            if (cursor.size() < len) return std::unexpected(PersistenceError{"short read: string"});
            String s(cursor.first(len));
            cursor = cursor.subspan(len);
            return Value{ std::move(s) };
        }
        // case 1: List, case 2: Hash, case 3: SortedSet — same shape
        default: return std::unexpected(PersistenceError{"unknown variant tag"});
    }
}
```

Two C++ habits worth noting.

**Cursor by value, mutated in place via reference.** `read_value` takes
`std::span<const std::uint8_t>&` and advances it as it consumes bytes. The
span is cheap to copy (two pointers), so passing by reference is purely so
the caller's cursor moves forward. This is the typical "read N bytes,
advance the cursor" pattern in C++.

**Switch on the tag.** No virtual dispatch, no visitor; just a switch.
Variant deserialization is one of the rare places `std::visit` *doesn't* fit
because the variant doesn't exist yet — we are constructing it. A switch over
the tag is exactly right.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

Every length you read from disk must be **bounds-checked against the
remaining buffer** before you trust it. A truncated record might have a
length field reading "4 billion" — and if you `subspan(len)` without
checking, you get a span into garbage memory the moment you read it.
Defensive reads:

```cpp
if (cursor.size() < len) return std::unexpected(PersistenceError{"short read"});
```

This applies to *every* length field in *every* binary format you read.
Network protocols (Chapter 6's parser had it too) and file formats both.
Forgetting these checks is the most common source of remote-exploitable bugs
in C/C++ daemons.

</div>

---

## Atomic publication via `rename`

Writing a snapshot directly over `kestrel.snap` is dangerous: a crash
midway through leaves a corrupt file that looks (to a startup-time reader)
like a valid snapshot, broken only when the CRC fails on a record. We can
do better — POSIX guarantees that **`rename` within the same directory is
atomic**. The pattern, used by every database and editor that publishes
files safely:

1. Write the new snapshot to a temporary file (`kestrel.snap.tmp`).
2. `fsync` the temporary file (so its bytes are durable).
3. `rename` it to the final name (`kestrel.snap`). Atomic.
4. `fsync` the *directory* (so the rename itself is durable).

After step 3, any reader sees either the *old* snapshot or the *new*
snapshot — never a torn intermediate. After step 4, the rename survives a
crash.

```cpp
// src/persistence/snapshot.cpp  (publish)
std::expected<void, PersistenceError>
publish_snapshot(const fs::path& final_path, const std::vector<std::uint8_t>& bytes) {
    fs::path tmp = final_path;
    tmp += ".tmp";

    int fd = ::open(tmp.c_str(), O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) return std::unexpected(...);

    // 1. Write bytes (with retry/EINTR loop as in Chapter 12).
    // 2. fsync the file.
    if (::fsync(fd) != 0) { ::close(fd); return std::unexpected(...); }
    ::close(fd);

    // 3. Rename atomically.
    std::error_code ec;
    fs::rename(tmp, final_path, ec);
    if (ec) return std::unexpected(PersistenceError{ec.message()});

    // 4. fsync the directory entry.
    int dfd = ::open(final_path.parent_path().c_str(), O_RDONLY);
    if (dfd >= 0) { ::fsync(dfd); ::close(dfd); }
    return {};
}
```

Step 4 is the one most code skips. Without it, a crash *after* the rename
but *before* the kernel flushed the directory inode may roll the rename
back; on next boot the file is back to its old name and the new snapshot is
lost. The mitigation is one extra `fsync` on the directory — cheap, and
the difference between "almost always durable" and "always durable."

<div class="callout callout-algo">

📐 **Algorithm**

**Atomic publication via tmp + rename.** Standard POSIX pattern for
publishing a file safely:

1. Write data to `path.tmp`.
2. `fsync(path.tmp)`.
3. `rename(path.tmp, path)` — atomic by POSIX guarantee.
4. `fsync(parent_directory)` to make the rename durable.

Used by `git`, by `rsync --inplace=false`, by most editors. Cost: one extra
syscall (the directory fsync). Benefit: a process crash at any point
leaves either the old file or the new file, never a torn mix.

</div>

---

## The rotation protocol

Snapshots and the WAL must be coordinated so that:

- A WAL record applied to the in-memory state is *either* captured in the
  next snapshot *or* still present in the WAL — never lost.
- A WAL record present in the WAL is *not also* present in the snapshot,
  or the recovery replay applies it twice.

The classic protocol:

1. At snapshot time, the server **freezes** (or snapshots a consistent view
   of) the keyspace.
2. The new WAL is **opened** as `kestrel.wal.new`; further mutating commands
   write to it.
3. The snapshot is written and published to `kestrel.snap` (atomic
   rename).
4. The old `kestrel.wal` is **deleted** (or kept as `.bak` for a while);
   `kestrel.wal.new` is renamed to `kestrel.wal`.

The invariant: at every moment, every committed mutation is recorded in
either the snapshot file or the WAL file (or both, briefly). Crash between
step 2 and step 4? On recovery: load the old snapshot, replay the old WAL
(which includes everything between snapshots), reapply. The new snapshot
file may or may not be present and is ignored if its CRC fails. Crash
between step 3 and step 4? On recovery: load the new snapshot, replay
the new WAL (which has only post-snapshot records). Either way, consistent.

A full implementation runs the snapshot in a **background thread** so the
foreground keeps serving — but threads do not exist in Kestrel until
Part 6. For now, snapshotting blocks all other work. We will revisit when
Part 6 introduces concurrency.

---

## Recovery: snapshot then WAL replay

Startup sequence:

```cpp
// src/persistence/recovery.cpp  (sketch)
std::expected<HashTable, PersistenceError>
recover(const fs::path& dir) {
    HashTable table;

    // 1. Load snapshot if present.
    auto snap = fs::path(dir) / "kestrel.snap";
    if (fs::exists(snap)) {
        auto bytes = read_whole_file(snap);
        if (!bytes) return std::unexpected(bytes.error());
        if (auto e = load_snapshot_into(table, *bytes); !e)
            return std::unexpected(e.error());
    }

    // 2. Replay the WAL on top.
    auto wal = fs::path(dir) / "kestrel.wal";
    if (fs::exists(wal)) {
        auto bytes = read_whole_file(wal);
        if (!bytes) return std::unexpected(bytes.error());
        if (auto e = replay_wal_into(table, *bytes); !e)
            return std::unexpected(e.error());
    }
    return table;
}
```

`replay_wal_into` walks the WAL one record at a time, validates each
record's CRC, and applies the encoded mutation to the table. The encoded
mutation is *the command itself* — `SET`, `DEL`, `EXPIRE`, etc. — in the
same encoding the command dispatcher uses; not a verbatim copy of the
client's RESP bytes but a normalized command struct. The natural shape is
to share an `apply_command(HashTable&, Command const&)` between the live
path (called from the command dispatcher in Chapter 16) and the recovery
path (called from `replay_wal_into`).

The first time a record's CRC fails — assume it's the torn last record —
we *stop*, not error. A torn last record is the expected signature of a
crash mid-write, and the WAL is **truncated** at that point. This is
exactly why the CRC is there.

<div class="callout callout-experiment">

🧪 **Experiment**

Simulate a crash. Start a Kestrel that takes a few commands, then `kill
-9` it (or just `std::abort()` from a test) mid-write. Restart and
recover. Confirm: every command up to the one before the kill is restored;
the killed command may or may not be present, depending on whether `fsync`
completed. Now hand-corrupt the middle of the WAL file (flip a byte in
the payload of record N) and recover. The recovery should truncate at
record N — record N-1 is restored, records N+1 and later are lost. That
"lose data after the corruption" behavior is correct: a WAL is an
ordered log; any record that fails its CRC is the boundary.

</div>

---

## A subtle problem: TTLs across restart

Storing `steady_clock` time points to disk and loading them back has a real
correctness problem. `steady_clock` is monotonic *within a process* — its
epoch is undefined across restarts. So a deadline stored as
`steady_clock::time_point` becomes garbage when the process restarts; the
new process's `steady_clock` has a different zero.

The fix: store TTLs on disk as **`std::chrono::system_clock` wall-clock
time** (which has a real, calendar-defined zero), and convert to and from
`steady_clock` at the boundaries. The conversion is one snapshot per
restart:

```cpp
// At snapshot/serialize time:
auto wall = std::chrono::system_clock::now() +
            (slot.expires_at - std::chrono::steady_clock::now());
write_u64_be(buf, std::chrono::system_clock::to_time_t(wall));   // unix epoch s

// At recovery/load time:
auto wall = std::chrono::system_clock::from_time_t(read_u64_be(...));
slot.expires_at = std::chrono::steady_clock::now() +
                  (wall - std::chrono::system_clock::now());
```

`steady_clock` has no `to_time_t` because it doesn't have a calendar
relationship; converting via `system_clock` is exactly the right escape
hatch. The risk is the system clock changing between snapshot and recovery
— but if that happens, the TTLs are off by exactly the clock skew, which
is rarely the worst thing happening at that moment.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

Node's `Date.now()` is essentially `system_clock`, and `process.hrtime()` is
`steady_clock`. The same problem exists there — `process.hrtime` doesn't
survive across process restarts. The C++ standard library is more explicit
about the distinction, which is good once you remember to think about it.

</div>

---

## Where this leaves Kestrel

Kestrel can crash and survive. Specifically, after Part 4:

- Every mutating command is durably logged before it is acknowledged (with
  `FsyncMode::Always`) or within a one-second window (with `EverySecond`).
- Periodic snapshots fold the in-memory state into a single file via the
  tmp-then-rename atomic publish.
- Startup loads the latest snapshot and replays the WAL on top to reach
  the pre-crash state.

The pieces are not yet automatic — Kestrel's *server loop*, which calls
into the WAL on every mutating command and triggers a snapshot on a
schedule, lives in Part 5 (the networking layer) and Part 6 (the threading
layer). What Part 4 provides is the *machinery*, used in a single-threaded
context where the server explicitly calls into it.

Part 5 connects Kestrel to the network. The first chapter (Chapter 14)
introduces Berkeley sockets and writes an RAII wrapper around file
descriptors, generalizing what `Wal` did inline. Chapter 15 introduces the
event loop. Chapter 16 ties protocol parsing to socket I/O, completing the
server-side path from "TCP byte" to "WAL append" to "memory mutation."

<div class="callout callout-ask">

💬 **Ask Claude**

> Here are my snapshot writer, recoverer, and rotation protocol
> (`src/persistence/snapshot.{hpp,cpp}`, `recovery.{hpp,cpp}`). Review:
> (1) the rotation steps — am I leaving any window where a crash would
> lose acknowledged data? (2) the bounds-checking in `read_value` and
> `replay_wal_into` — am I checking every length field against the
> remaining buffer? (3) my TTL conversion — does it survive a system
> clock change correctly? (4) is `fsync` on the parent directory portable
> the way I am using it?

</div>

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and tests green under ASan/UBSan:

```bash
cmake --build src/build
ctest --test-dir src/build --output-on-failure
```

Tests must cover:

- Round-trip: write a snapshot of a populated `HashTable`, load it into a
  fresh one, and confirm every key/value/expiry matches.
- Recovery: snapshot a state, mutate further, replay the WAL, and confirm
  the final state matches the live state.
- Corruption resilience: corrupt the middle of a WAL file and confirm the
  recoverer truncates at the corruption point without crashing.

You should be able to:

- Explain why a snapshot uses tmp+rename rather than overwriting in place.
- Defend the WAL rotation order (open new WAL, write snapshot, rename
  snapshot, switch WAL, delete old WAL) in terms of "what survives a crash
  at each step."
- Discuss why storing `steady_clock` time points across restart is a bug
  and how the chapter's TTL encoding avoids it.

Hand it to Claude:

> Walk me through my Chapter 13 persistence layer end-to-end. Are there
> race conditions I'm not seeing? Are my `expected`-returning APIs
> consistent in error handling? Suggest one test I haven't written that
> would catch a class of subtle bugs in snapshot/replay.

When that's green, Part 4 is finished and Part 5 connects Kestrel to a
network.

</div>
