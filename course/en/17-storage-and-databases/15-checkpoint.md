<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.14 Project: your own key-value store](14-project-key-value-store.md) · [📋 Phase 17](README.md) · [Phase 18 — Distributed systems →](../18-distributed-systems/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/15-checkpoint.md) <!-- nav -->

# Lesson 17.15 — Checkpoint

> **Phase 17 — Storage and database internals** · Lesson 15 of 15 · ⏱ about 120 minutes

## What you will learn

This lesson teaches nothing new. It checks that you can use what Phase 17 taught, and it shows you where to go back if
something is still weak. You will:

- Answer 18 questions about disks and durability, file formats, logs, indexes, caches, transactions, isolation, MVCC (Multi-Version Concurrency Control) and
  query execution.
- Solve 5 practical tasks: two with code and tests, one in PostgreSQL, two on paper.
- Find the lessons you should repeat, with direct links.

## Before you start

- You finished [Lesson 17.14 — Project: your own key-value store](14-project-key-value-store.md). Your `p17-kv` passes
  its quality gate and is committed.
- Do the quiz **closed book**: no lessons, no compiler, no search. Write your answer first, then open the hidden answer.
- For the practical tasks you may use the compiler and the lessons, but try first without hints. Tasks 1 and 2 go into a
  new library crate:

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/17-15-checkpoint
cargo new --lib code/p17-checkpoint
cd code/p17-checkpoint
cargo add crc32fast@1
```

Put `pub mod lru;` and `pub mod record;` into `src/lib.rs`, and the code of the tasks into `src/record.rs` and
`src/lru.rs`. For Task 3 the container `p17-postgres` must be running.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** (lesson) | A pause at the end of a phase to test yourself before you go on. Not the database checkpoint of Lesson 17.9. |
| **closed book** | Answering from memory only, without looking anything up. |

## Self-test quiz

### Disks, durability and file formats (Lessons 17.1–17.3)

1. A program calls `write` and it returns success. Then the power fails. Can the data be lost? What would have prevented
   it?

<details>
<summary>Answer</summary>

Yes. `write` only puts the data into the page cache in RAM (Random Access Memory). `sync_data` / `sync_all` (`fdatasync` / `fsync`, on macOS
`fcntl(F_FULLFSYNC)`) waits until the data is on stable storage.

</details>

2. Why does a storage engine prefer to append to the end of a file instead of changing bytes in the middle?

<details>
<summary>Answer</summary>

Devices work in blocks: changing a few bytes costs a read-modify-write of a whole block, and random writes are slower
than sequential ones. Appending is sequential, and a crash can only damage the end of the file, which recovery can cut
off.

</details>

3. What is a torn write, and how does a reader recognise a torn record?

<details>
<summary>Answer</summary>

A write interrupted by a crash, so only its first part reached the disk. The reader sees that the record is shorter
than its length field says, or that the CRC (Cyclic Redundancy Check) of the payload does not match.

</details>

4. Why must a decoder check a length read from a file against a maximum **before** allocating a buffer?

<details>
<summary>Answer</summary>

A damaged (or malicious) file can contain any number, for example 4 GiB (gibibytes). Allocating that much could crash the program
or the machine. The decoder returns an error such as `TooLarge` instead.

</details>

### Logs and indexes (Lessons 17.4–17.8)

5. State the write-ahead rule in one sentence.

<details>
<summary>Answer</summary>

Append the change to the log and make the log durable **before** applying the change and acknowledging it.

</details>

6. Recovery finds a damaged record at the end of the log. Why must it truncate the file there, instead of just stopping
   reading?

<details>
<summary>Answer</summary>

New records are appended after the end of the file. If the garbage stays, the next recovery stops at the garbage again
and never reaches the new records: they would be lost.

</details>

7. Name two limits of a hash index (Bitcask design).

<details>
<summary>Answer</summary>

Any two of: all keys must fit in RAM; no range queries (keys are not sorted); start-up must scan the log to rebuild the
index; stale records waste disk space until compaction.

</details>

8. Why does a B-tree with a fanout of several hundred need only three or four page reads to find a key among billions?

<details>
<summary>Answer</summary>

Its height is the logarithm of the number of keys with the fanout as the base: 500³ is already 125 million, 500⁴ more
than 60 billion. Each level costs one page read, and the top levels are usually in the buffer pool anyway.

</details>

9. In an LSM (Log-Structured Merge) tree, why can a lookup of one missing key cost many reads, and what do compaction and Bloom filters do about
   it?

<details>
<summary>Answer</summary>

The key could be in any SSTable, so the lookup asks the memtable and then every table from newest to oldest. Compaction
merges tables, so there are fewer to ask; a Bloom filter per table answers "definitely not here" for most tables
without reading them.

</details>

### Caches and transactions (Lessons 17.9–17.12)

10. Give two reasons why a database keeps its own buffer pool instead of relying only on the page cache.

<details>
<summary>Answer</summary>

Any two of: it must control the order of writes (data pages only after their WAL (write-ahead log) records are durable); it knows which
pages are valuable; it decides when dirty pages are written (checkpoints), which keeps recovery short.

</details>

11. What is sequential flooding, and which two tricks do databases use against it?

<details>
<summary>Answer</summary>

A big scan through an LRU (Least Recently Used) cache that evicts exactly the pages needed next, so the cache gets no hits and pushes out the
hot pages. Databases give big scans a small private ring of buffers and use the clock algorithm with usage counters.

</details>

12. What is the commit point of a transaction in a log-based store?

<details>
<summary>Answer</summary>

The moment when the transaction's commit record (or its single batch record) is complete and durable in the log. Before
it, a crash erases the transaction; after it, recovery replays it.

</details>

13. Why does a database whose buffer pool may write uncommitted changes to the data files need undo information?

<details>
<summary>Answer</summary>

After a crash the data files can contain changes of transactions that never committed. Recovery must take them back,
which needs the old values.

</details>

14. Two transactions each read `stock = 1` and each write `stock = 0` after selling the last item. Which anomaly is this,
    and how does PostgreSQL's `REPEATABLE READ` react?

<details>
<summary>Answer</summary>

A lost update (one sale is not counted; the item is sold twice). At `REPEATABLE READ` the second `UPDATE` fails with
`could not serialize access due to concurrent update` (SQLSTATE `40001`, the standard SQL error code for a serialization failure), and the application must retry the whole
transaction.

</details>

15. Why does snapshot isolation allow write skew, and which PostgreSQL level stops it?

<details>
<summary>Answer</summary>

Snapshot isolation only detects two transactions writing the **same** row. In write skew they write different rows
after reading the same data. `SERIALIZABLE` (SSI, Serializable Snapshot Isolation) tracks read/write dependencies and aborts one of them.

</details>

16. A PostgreSQL row version has `xmin = 900` and `xmax = 950`. Transaction 950 committed. Which snapshots still see
    this version?

<details>
<summary>Answer</summary>

Snapshots for which 900 is committed and in the past, but 950 is not yet: snapshots taken before transaction 950
committed. Newer snapshots see the version that 950 created instead (or no row at all, if 950 deleted it).

</details>

### Queries and the project (Lessons 17.13–17.14)

17. `EXPLAIN ANALYZE` shows `rows=30` estimated and `rows=100020` actual for a scan. What is the most likely cause, and
    the first thing to try?

<details>
<summary>Answer</summary>

Old statistics: the data changed since the last `ANALYZE`. Run `ANALYZE` on the table and check why autovacuum did not.

</details>

18. Compaction writes a new data file, replaces the `MANIFEST` and deletes the old file. Why is the `MANIFEST` needed at
    all?

<details>
<summary>Answer</summary>

It gives the multi-step change one atomic commit point. Without it, a crash after writing the new file but before
deleting the old one would leave two files, and `open` could not know which one is the current state (a deleted key
could come back from the old file).

</details>

## Practical tasks

### Task 1 — A decoder that never panics (Lessons 17.3 and 17.4)

In `src/record.rs`, write `encode(payload: &[u8]) -> Vec<u8>` for records `[length: u32 LE][CRC-32: u32 LE][payload]`
and `decode(buf: &[u8]) -> Result<(&[u8], usize), DecodeError>` that returns the payload and the bytes used. Errors:
`Truncated`, `TooLarge(usize)` (payload over 1 MiB, one mebibyte) and `BadChecksum`. Write four tests: a round trip, every prefix of a
record is `Truncated`, a length of `u32::MAX` is `TooLarge`, a flipped bit is `BadChecksum`.

<details>
<summary>Hint</summary>

`buf.get(..8)` and `buf.get(8..8 + len)` return `None` instead of panicking; `.ok_or(DecodeError::Truncated)?` turns
that into the error. Check the length limit **before** `get(8..8 + len)`.

</details>

<details>
<summary>Answer</summary>

```rust
//! Task 1: a record decoder that never panics.

pub const MAX_PAYLOAD: usize = 1 << 20;

#[derive(Debug, PartialEq)]
pub enum DecodeError {
    Truncated,
    TooLarge(usize),
    BadChecksum,
}

/// Record = [payload length: u32 LE][CRC-32 of payload: u32 LE][payload].
pub fn encode(payload: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(8 + payload.len());
    out.extend_from_slice(&(payload.len() as u32).to_le_bytes());
    out.extend_from_slice(&crc32fast::hash(payload).to_le_bytes());
    out.extend_from_slice(payload);
    out
}

/// Returns the payload and the number of bytes the record used.
pub fn decode(buf: &[u8]) -> Result<(&[u8], usize), DecodeError> {
    let header = buf.get(..8).ok_or(DecodeError::Truncated)?;
    let len = u32::from_le_bytes(header[0..4].try_into().unwrap()) as usize;
    let crc = u32::from_le_bytes(header[4..8].try_into().unwrap());
    if len > MAX_PAYLOAD {
        return Err(DecodeError::TooLarge(len)); // check before using the length
    }
    let payload = buf.get(8..8 + len).ok_or(DecodeError::Truncated)?;
    if crc32fast::hash(payload) != crc {
        return Err(DecodeError::BadChecksum);
    }
    Ok((payload, 8 + len))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn round_trip() {
        let rec = encode(b"hello");
        assert_eq!(decode(&rec), Ok((&b"hello"[..], 13)));
    }

    #[test]
    fn every_prefix_is_truncated() {
        let rec = encode(b"hello");
        for cut in 0..rec.len() {
            assert_eq!(
                decode(&rec[..cut]),
                Err(DecodeError::Truncated),
                "cut {cut}"
            );
        }
    }

    #[test]
    fn a_huge_length_is_rejected() {
        let mut rec = encode(b"x");
        rec[0..4].copy_from_slice(&u32::MAX.to_le_bytes());
        assert_eq!(decode(&rec), Err(DecodeError::TooLarge(u32::MAX as usize)));
    }

    #[test]
    fn a_flipped_bit_is_caught() {
        let mut rec = encode(b"hello");
        rec[10] ^= 0b0000_0100;
        assert_eq!(decode(&rec), Err(DecodeError::BadChecksum));
    }
}
```

```bash
cargo test record
```

```text
running 4 tests
test record::tests::a_flipped_bit_is_caught ... ok
test record::tests::every_prefix_is_truncated ... ok
test record::tests::a_huge_length_is_rejected ... ok
test record::tests::round_trip ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 3 filtered out; finished in 0.00s
```

Why it works: every access to the buffer goes through `get`, so no input can cause a panic, and the checks run in a safe
order: header, length limit, payload, checksum. (The "3 filtered out" are the tests of Task 2.)

</details>

### Task 2 — Count LRU hits, test first (Lesson 17.9)

In `src/lru.rs`, write `lru_hits(capacity: usize, trace: &[u64]) -> usize`: simulate an LRU cache with `capacity`
slots for the page numbers in `trace` and return the number of hits. **Write the tests first**: the trace
`1, 2, 1, 3, 2, 1` with 2 slots gives 1 hit; three scans of pages 0 to 9 give 0 hits with 9 slots and 20 hits with 10
slots; a cache with 0 slots never hits.

<details>
<summary>Hint</summary>

A `Vec<u64>` ordered from least to most recently used is enough: on a hit, remove the page and push it to the end; on
a miss with a full cache, remove index 0 first. Then think about `capacity == 0` again.

</details>

<details>
<summary>Answer</summary>

```rust
//! Task 2: count the hits of an LRU cache for a trace of page numbers.

/// Simulates an LRU cache with `capacity` slots and returns the number of hits.
pub fn lru_hits(capacity: usize, trace: &[u64]) -> usize {
    if capacity == 0 {
        return 0; // nothing can ever be cached
    }
    let mut cache: Vec<u64> = Vec::new(); // least recently used first
    let mut hits = 0;
    for &page in trace {
        if let Some(i) = cache.iter().position(|&p| p == page) {
            hits += 1;
            cache.remove(i);
        } else if cache.len() == capacity {
            cache.remove(0); // evict the least recently used page
        }
        cache.push(page); // now the most recently used
    }
    hits
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn the_example_from_lesson_17_9() {
        assert_eq!(lru_hits(2, &[1, 2, 1, 3, 2, 1]), 1);
    }

    #[test]
    fn a_scan_one_page_bigger_than_the_cache_never_hits() {
        let trace: Vec<u64> = (0..3).flat_map(|_| 0..10).collect();
        assert_eq!(lru_hits(9, &trace), 0);
        assert_eq!(lru_hits(10, &trace), 20);
    }

    #[test]
    fn capacity_zero_never_hits() {
        assert_eq!(lru_hits(0, &[1, 1, 1]), 0);
    }
}
```

Why it works: the order of the `Vec` *is* the LRU order. The zero-capacity test matters: without the early `return`,
the first miss finds `cache.len() == capacity` (both 0) and calls `cache.remove(0)` on an empty vector, which panics
with `removal index (is 0) should be < len (is 0)`. Edge-case tests find exactly such bugs.

</details>

### Task 3 — Lose an update, then fix it (Lessons 17.11 and 17.12)

In two `psql` terminals, create `CREATE TABLE stock (item text PRIMARY KEY, n int NOT NULL);` with one row
`('book', 1)`. Reproduce a lost update at `READ COMMITTED` (both terminals read `n`, both write `n - 1` as a constant),
then show **two** fixes: one that keeps `READ COMMITTED`, and one that uses `REPEATABLE READ`. For each, write down the
timeline and what each terminal printed.

<details>
<summary>Hint</summary>

The fix at `READ COMMITTED` lets the database do the arithmetic, or locks the row while reading. At `REPEATABLE READ`
the second writer gets an error; what must its application do then?

</details>

<details>
<summary>Answer</summary>

```text
step  T1                                              T2
1     BEGIN;                                          BEGIN;
2     SELECT n FROM stock WHERE item = 'book';  -> 1
3                                                     SELECT n FROM stock WHERE item = 'book';  -> 1
4     UPDATE stock SET n = 0 WHERE item = 'book';
5     COMMIT;
6                                                     UPDATE stock SET n = 0 WHERE item = 'book';
7                                                     COMMIT;          -> n = 0: two sales, one counted
```

Fix 1 (`READ COMMITTED`): `UPDATE stock SET n = n - 1 WHERE item = 'book' AND n > 0;` in both terminals. The second one
waits for T1's row lock, then re-checks the row, finds `n = 0` and prints `UPDATE 0`: the application knows the item is
sold out. (`SELECT ... FOR UPDATE` before the write works too.)

Fix 2 (`REPEATABLE READ`): the same timeline with `BEGIN ISOLATION LEVEL REPEATABLE READ;`. Step 6 fails with
`ERROR:  could not serialize access due to concurrent update`; T2 runs `ROLLBACK`, starts again, reads `n = 0` and does
not sell.

Why it works: both fixes stop the second writer from acting on a value that is no longer true; the first makes it
re-check, the second makes it start over.

</details>

### Task 4 — Visibility on paper (Lesson 17.12)

A table has these versions of the row `alice`:

```text
 version | xmin | xmax | balance
---------+------+------+---------
 A       |  100 |  105 |     100
 B       |  105 |  110 |      90
 C       |  110 |    0 |      80
```

Transactions 100, 105 and 110 all committed. Which version does a snapshot see that was taken (1) after 105 committed
but while 110 was still running, (2) after 110 committed? Which versions can `VACUUM` remove in case (2), if no older
snapshot exists?

<details>
<summary>Hint</summary>

A version is visible if its `xmin` is committed in the snapshot's past and its `xmax` is not.

</details>

<details>
<summary>Answer</summary>

(1) Version B (balance 90): 105 is in the past, 110 is still running, so B's deletion does not count yet. (2) Version C
(balance 80). `VACUUM` can remove A and B: their deleters committed and no snapshot needs them any more.

Why it works: each snapshot sees exactly one version of the row, the one whose "lifetime" `[xmin, xmax)` contains the
snapshot's point in time.

</details>

### Task 5 — Find the problem in a plan (Lessons 17.9 and 17.13)

A query is slow. Its plan:

```text
 Aggregate  (cost=112.74..112.75 rows=1 width=8) (actual time=9.331..9.332 rows=1.00 loops=1)
   Buffers: shared hit=641
   ->  Bitmap Heap Scan on orders  (cost=4.53..112.67 rows=30 width=0) (actual time=1.400..6.671 rows=100020.00 loops=1)
         Recheck Cond: (customer_id = 7)
         Heap Blocks: exact=561
         Buffers: shared hit=641
```

Answer: (a) which node runs first, (b) how wrong the row estimate is, (c) whether the data came from the disk, and
(d) what you do first.

<details>
<summary>Hint</summary>

Compare `rows=` in the cost part with `rows=` in the actual part. `shared hit` refers to the buffer pool.

</details>

<details>
<summary>Answer</summary>

(a) The `Bitmap Heap Scan` (more indented; it also has a `Bitmap Index Scan` child, not shown) runs first; the
`Aggregate` counts its rows. (b) 30 expected, 100,020 actual: more than 3,000 times too few. (c) No: all 641 pages were
`shared hit`, found in the buffer pool. (d) Run `ANALYZE orders;` and look at the plan again; then find out why
autovacuum did not refresh the statistics. With a wrong estimate like this, any join above the scan may get a bad plan.

Why it works: a large gap between estimated and actual rows is the most common root cause of a bad plan, and the
statistics are the cheapest thing to fix.

</details>

## If you got something wrong

| Topic | Lesson to review |
|---|---|
| Layers between `write` and the device, blocks, sequential and random I/O | [Lesson 17.1 — How disks and files work](01-how-disks-and-files-work.md) |
| `sync_all`, `F_FULLFSYNC`, torn writes, group commit | [Lesson 17.2 — Durability: fsync and crash safety](02-durability-and-fsync.md) |
| Record formats, byte order, length limits, CRC | [Lesson 17.3 — Binary file formats and serialization](03-binary-file-formats.md) |
| The write-ahead rule, recovery, truncating the tail | [Lesson 17.4 — The write-ahead log (WAL)](04-write-ahead-log.md) |
| Bitcask, tombstones, the limits of a hash index | [Lesson 17.5 — Indexes: a hash index](05-hash-indexes.md) |
| B-tree height, splits, pages | [Lesson 17.6 — B-trees](06-b-trees.md) |
| Memtable, SSTables, shadowing, read amplification | [Lesson 17.7 — LSM trees](07-lsm-trees.md) |
| Merging tables, Bloom filters | [Lesson 17.8 — Compaction and Bloom filters](08-compaction-and-bloom-filters.md) |
| Page cache, buffer pool, LRU, clock, checkpoints | [Lesson 17.9 — The page cache and the buffer pool](09-page-cache-and-buffer-pool.md) |
| ACID (atomicity, consistency, isolation, durability), commit point, redo and undo | [Lesson 17.10 — Transactions and ACID](10-transactions-and-acid.md) |
| Anomalies, isolation levels, `40001` and retries | [Lesson 17.11 — Isolation levels](11-isolation-levels.md) |
| Versions, snapshots, visibility, `VACUUM` | [Lesson 17.12 — MVCC: multi-version concurrency control](12-mvcc.md) |
| Plans, costs, statistics, joins, the Volcano model | [Lesson 17.13 — How a SQL query runs](13-how-a-sql-query-runs.md) |
| `MANIFEST`, atomic compaction, model-based tests | [Lesson 17.14 — Project: your own key-value store](14-project-key-value-store.md) |

## What's next

Phase 18 takes the same questions ("what if it crashes here?", "what does each side see?") to many machines that talk
over a network: [Phase 18 — Distributed systems](../18-distributed-systems/README.md). Replication, consensus and
distributed transactions build directly on the log, the commit point and the isolation levels of this phase.

Clean up the containers of this phase when you no longer need them:

```bash
docker rm -f p17-postgres
docker volume rm p17-pagecache 2>/dev/null; true
```

`p17-postgres` had no volume, so removing it deletes its data. Commit your checkpoint work and merge the branch.

## Sources and further reading

- [Designing Data-Intensive Applications](https://dataintensive.net/) — its chapters on storage and on transactions cover this whole phase in one place.
- [CMU 15-445/645 Database Systems](https://15445.courses.cs.cmu.edu/) — CMU is Carnegie Mellon University; the free course for going deeper: buffer pools, indexes, concurrency control, recovery.
- [PostgreSQL: Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html) — isolation and MVCC in the database you used.
- [SQLite: Atomic Commit In SQLite](https://www.sqlite.org/atomiccommit.html) — a complete, careful example of crash safety in a real engine.

---

[🏠 Start](../START-HERE.md) · [← 17.14 Project: your own key-value store](14-project-key-value-store.md) · [📋 Phase 17](README.md) · [Phase 18 — Distributed systems →](../18-distributed-systems/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
