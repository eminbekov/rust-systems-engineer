<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.16 Checkpoint](../16-backend-services/16-checkpoint.md) · [17.1 How disks and files work →](01-how-disks-and-files-work.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/README.md) <!-- nav -->

# Phase 17 — Storage and database internals

> **Stage 4 — Build real systems** · ⏱ about 5 weeks at ~10 hours per week

## What this phase is about

In Phase 16 you used PostgreSQL as a black box: you sent SQL and trusted it to keep your data safe. This phase opens the box.
You will learn what really happens between "save this" and data that survives a power cut: how disks and SSDs work, why
`write` is not the same as "saved", and how `fsync`, checksums and a write-ahead log make data durable.

Then you build the data structures that every database is made of, in Rust and with tests: a hash index, a B-tree, and an
LSM (log-structured merge) tree with SSTables, compaction and Bloom filters. Each one is small enough to understand completely, and each one is
measured, so that you see with your own numbers why real systems are designed the way they are.

The second half explains how a database lets many users work at the same time without corrupting data: transactions and
ACID (atomicity, consistency, isolation, durability), isolation levels, MVCC (multi-version concurrency control) and how a SQL query is planned and executed. The phase ends
with your own crash-safe key-value store.

## What you will be able to do

- [ ] Explain the path of data from your program to the storage device, and measure a disk without the page cache hiding it.
- [ ] Make writes durable with `sync_all`/`sync_data` (including `F_FULLFSYNC` on macOS) and handle a failed `fsync` correctly.
- [ ] Design binary record formats with length prefixes and CRC checksums, and test them against torn and damaged data.
- [ ] Build a write-ahead log with crash recovery, and test it by simulating crashes in the middle of a write.
- [ ] Implement and compare a hash index, a B-tree and an LSM tree with SSTables, compaction and Bloom filters.
- [ ] Explain transactions, ACID, isolation levels and MVCC, and recognise the anomalies each level allows.
- [ ] Read a query plan and explain how a SQL query is executed.
- [ ] Build your own crash-safe key-value store with benchmarks.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 17.1 | [How disks and files work](01-how-disks-and-files-work.md) | 120 min |
| 17.2 | [Durability: fsync and crash safety](02-durability-and-fsync.md) | 2.5 h |
| 17.3 | [Binary file formats and serialization](03-binary-file-formats.md) | 2.5 h |
| 17.4 | [The write-ahead log (WAL)](04-write-ahead-log.md) | 135 min |
| 17.5 | [Indexes: a hash index](05-hash-indexes.md) | 135 min |
| 17.6 | [B-trees](06-b-trees.md) | 2.5 h |
| 17.7 | [LSM trees](07-lsm-trees.md) | 3 h |
| 17.8 | [Compaction and Bloom filters](08-compaction-and-bloom-filters.md) | 3 h |
| 17.9 | [The page cache and the buffer pool](09-page-cache-and-buffer-pool.md) | 3 h |
| 17.10 | [Transactions and ACID](10-transactions-and-acid.md) | 135 min |
| 17.11 | [Isolation levels](11-isolation-levels.md) | 120 min |
| 17.12 | [MVCC: multi-version concurrency control](12-mvcc.md) | 2.5 h |
| 17.13 | [How a SQL query runs](13-how-a-sql-query-runs.md) | 135 min |
| 17.14 | [Project: your own key-value store](14-project-key-value-store.md) | 15 h |
| 17.15 | [Checkpoint](15-checkpoint.md) | 120 min |

## Projects in this phase

- **`code/p17-storage`** (Lessons 17.1–17.8): one library that grows lesson by lesson: a crash simulator, a record format
  with checksums, a WAL-backed store, a hash-indexed log, a B-tree, SSTables, an LSM tree, compaction and Bloom filters,
  each with tests and a small measuring program in `src/bin/`.
- **One small package per lesson** (Lessons 17.9–17.13): `code/p17-bufpool`, `code/p17-txkv`, `code/p17-isolation`,
  `code/p17-mvcc` and `code/p17-executor`. Lessons 17.9 and 17.11–17.13 also use a throw-away PostgreSQL container,
  `p17-postgres`, which Lesson 17.9 starts.
- **Your own key-value store** (Lesson 17.14, `code/p17-kv`): a library plus a command-line tool or server with put, get and delete, built
  on a WAL and an LSM tree or a Bitcask-style log with compaction, with crash-recovery tests and benchmarks.
- **Checkpoint** (Lesson 17.15): a self-test quiz and practical tasks that show whether you are ready for Phase 18.

## How to study this phase

- **Type and run every example, and measure.** The numbers in the lessons come from one Mac; yours will differ. The point is
  to see the ratios yourself: page cache versus SSD, sync versus no sync, index versus scan.
- **Write the crash tests.** Every lesson has a test that damages a file or "crashes" in the middle of a write. These tests
  are the most valuable code of the phase; keep them green as the library grows.
- **Keep one library.** Lessons 17.3–17.8 build on each other's modules (`record`, `wal`, `sstable`, `lsm`). Do the exercises
  too: later lessons assume their code and test counts.
- **Draw before you code.** For B-tree splits, SSTable layouts and merges, draw the bytes or the nodes on paper first; most
  bugs are visible in the drawing.
- **Read one real system alongside.** The LevelDB implementation notes and the PostgreSQL WAL chapter, linked from the
  lessons, are short and show the same ideas at production scale.

---

[🏠 Start](../START-HERE.md) · [← 16.16 Checkpoint](../16-backend-services/16-checkpoint.md) · [17.1 How disks and files work →](01-how-disks-and-files-work.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
