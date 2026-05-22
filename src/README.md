# Kestrel implementation

This directory is where **you**, the reader, build Kestrel as you work through
the book. It starts empty by design — the book teaches the concepts and shows
illustrative code; you write the real implementation here, chapter by chapter,
with Claude Code as your tutor and reviewer (see `../CLAUDE.md`).

Expected layout as the book progresses:

```
src/
  CMakeLists.txt
  core/         Value type, Buffer, ownership & memory primitives
  protocol/     RESP serialize / parse
  store/        hash table, expiry, eviction, data types
  persistence/  WAL, snapshot, recovery, compaction
  net/          socket RAII wrappers, event loop, connections
  concurrency/  thread pool, task queue, sharded keyspace
  server/       kvd
  client/       kvcli
  test/         tests (hand-rolled harness, later a real framework)
```
