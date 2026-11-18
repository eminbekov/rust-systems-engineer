<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.7 LSM trees](07-lsm-trees.md) · [📋 Phase 17](README.md) · [17.9 The page cache and the buffer pool →](09-page-cache-and-buffer-pool.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/08-compaction-and-bloom-filters.md) <!-- nav -->

# Lesson 17.8 — Compaction and Bloom filters

> **Phase 17 — Storage and database internals** · Lesson 8 of 15 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Why an LSM tree must compact its SSTables, and what compaction gives back: faster reads and free disk space.
- How to merge many sorted runs into one with a heap (a k-way merge), keeping only the newest version of each key.
- The two classic compaction strategies, size-tiered and leveled, and what each one costs.
- When a tombstone may be dropped, and why that is a crash-safety question.
- How a Bloom filter answers "definitely not here" without reading a table, and how to choose its size.

## Before you start

- You finished [Lesson 17.7 — LSM trees](07-lsm-trees.md): `src/sstable.rs` and `src/lsm.rs` exist, and the demo showed
  26 block reads for one missing key.
- You know heaps and `BinaryHeap` from [Lesson 07.12 — Heaps and priority queues](../07-algorithms-and-data-structures/12-heaps-and-priority-queues.md),
  merging sorted lists from [Lesson 07.9](../07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md), hash
  functions from [Lesson 07.6](../07-algorithms-and-data-structures/06-hash-tables.md), and bit masks from
  [Lesson 12.4](../12-computer-systems/04-bitwise-operations.md).

## New words

| Word | Meaning |
|---|---|
| **sorted run** | A sequence of key-value pairs in key order, for example the contents of one SSTable. |
| **k-way merge** | Merging `k` sorted runs into one sorted run in a single pass, using a heap to find the smallest next key. |
| **size-tiered compaction** | A strategy that merges several tables of about the same size into one bigger table. |
| **leveled compaction** | A strategy that keeps tables in levels L0, L1, L2, …; each level is about ten times bigger than the one above, and inside a level the tables do not overlap. |
| **space amplification** | Using more disk space than the live data needs, because of old versions and tombstones. |
| **Bloom filter** | A small bit array that answers "is this key in the set?" with "definitely not" or "maybe". It never says "no" for a key that was added. |
| **false positive** | A "maybe" from a Bloom filter for a key that was never added. It costs a useless read, but never a wrong answer. |
| **manifest** | A small file in which a storage engine records which table files are currently valid. Changing it atomically is how a set of files changes in one step. |

## 1. Two bills to pay

The LSM tree of Lesson 17.7 writes fast, but it leaves two bills:

1. **Read amplification.** Every flush adds a table, and a lookup may read a block from every table: 26 reads for one
   missing key in the demo.
2. **Space amplification.** An overwritten key keeps its old versions in older tables, and a deleted key keeps its old
   value *and* its tombstone. The files hold much more than the live data.

**Compaction** pays the first bill and most of the second: it merges several tables into one, keeps only the newest version
of each key, and deletes the old files. **Bloom filters** pay the rest of the first bill: they let a lookup skip a table
without reading it.

## 2. Merging sorted runs

Every SSTable is sorted, so merging tables is like merging the sorted halves in merge sort (Lesson 07.9), but with `k`
inputs instead of two. Keep the next key of every input in a **min-heap** (Lesson 07.12): the smallest key is on top. Pop
it, write it out, and push the next key of the input it came from. Each step costs `O(log k)`, and the inputs are read
strictly in order, sequentially.

One detail is new: the same key can appear in several inputs. The newest version must win. So the heap orders by key
first and by **age** second (0 = newest run), and the newest version of a key always comes out first. Older versions of
the same key that follow are skipped.

```text
 run 0 (newest):  b=2   d=✝            heap pops: a=1 (run 1)
 run 1 (oldest):  a=1   b=1   d=1                 b=2 (run 0)   ← newest b
                                                  b=1 (run 1)   skip: older b
 output:          a=1   b=2   d=✝                 d=✝ (run 0)   ← tombstone wins
                                                  d=1 (run 1)   skip
```

Create `src/compaction.rs` and add `pub mod compaction;` to `src/lib.rs`:

```rust
use crate::sstable::Value;
use std::cmp::Reverse;
use std::collections::BinaryHeap;

/// Merges sorted runs into one sorted run. `runs[0]` is the newest.
/// For a key that appears in several runs, only the newest version is kept.
/// With `drop_tombstones`, deleted keys disappear completely.
pub fn merge(runs: Vec<Vec<(Vec<u8>, Value)>>, drop_tombstones: bool) -> Vec<(Vec<u8>, Value)> {
    let mut iters: Vec<_> = runs.into_iter().map(|run| run.into_iter()).collect();
    let mut heap = BinaryHeap::new();
    for (age, iter) in iters.iter_mut().enumerate() {
        if let Some((key, value)) = iter.next() {
            heap.push(Reverse((key, age, value))); // smallest key first, then the newest run
        }
    }

    let mut out: Vec<(Vec<u8>, Value)> = Vec::new();
    while let Some(Reverse((key, age, value))) = heap.pop() {
        if let Some((next_key, next_value)) = iters[age].next() {
            heap.push(Reverse((next_key, age, next_value)));
        }
        if out.last().is_some_and(|(last, _)| *last == key) {
            continue; // an older version of a key we already have
        }
        out.push((key, value));
    }
    if drop_tombstones {
        out.retain(|(_, value)| value.is_some());
    }
    out
}

#[cfg(test)]
mod tests {
    use super::*;

    fn run(entries: &[(&str, Option<&str>)]) -> Vec<(Vec<u8>, Value)> {
        entries
            .iter()
            .map(|(k, v)| (k.as_bytes().to_vec(), v.map(|v| v.as_bytes().to_vec())))
            .collect()
    }

    #[test]
    fn newest_version_wins_and_output_is_sorted() {
        let newest = run(&[("b", Some("2")), ("d", None)]);
        let oldest = run(&[("a", Some("1")), ("b", Some("1")), ("d", Some("1"))]);
        let merged = merge(vec![newest, oldest], false);
        assert_eq!(
            merged,
            run(&[("a", Some("1")), ("b", Some("2")), ("d", None)])
        );
    }

    #[test]
    fn tombstones_can_be_dropped() {
        let newest = run(&[("d", None)]);
        let oldest = run(&[("a", Some("1")), ("d", Some("1"))]);
        assert_eq!(merge(vec![newest, oldest], true), run(&[("a", Some("1"))]));
    }
}
```

**What happens:**

1. `BinaryHeap` is a max-heap; wrapping the items in `Reverse` turns it into a min-heap (Lesson 07.12).
2. Tuples compare element by element: first the key, then `age`. So for equal keys, the item from the newest run
   (smallest `age`) comes out first. The value is the third element only so that it travels with its key.
3. After each pop, the next item of the same run goes into the heap. The heap never holds more than `k` items, however long
   the runs are.
4. `out.last()` is the key written just before. If the popped key is the same, it is an older version and is skipped.
5. Tombstones are dropped only at the very end, with `retain`. Dropping a tombstone during the loop would be a bug: the
   older version that follows it would then be written instead, and the deleted key would come back.

> ⚡ **Performance:** a real compaction streams: it reads each input block by block and writes the output block by block, so
> it needs memory for one block per input, not for whole tables. Our `merge` takes whole `Vec`s to stay short; the heap logic
> is the same.

## 3. Which tables to merge: two strategies

Merging *all* tables every time would rewrite the whole database again and again. Real engines choose **which** tables to
merge, and there are two classic strategies:

```text
 size-tiered (used by Cassandra)            leveled (LevelDB, RocksDB's default)
 ───────────────────────────────            ────────────────────────────────────
 4 small tables ──merge──► 1 medium         L0:  new flushed tables (may overlap)
 4 medium       ──merge──► 1 large          L1:  ~10 MB,  tables do not overlap
 4 large        ──merge──► 1 huge           L2:  ~100 MB, tables do not overlap
                                            L3:  ~1 GB …
 + little rewriting (low write amp.)        + a key is in at most one table per level
 − several tables of each size to search    − data is rewritten at every level
 − needs free space for the big merges        (higher write amplification)
```

- **Size-tiered** compaction waits until there are several tables of about the same size and merges them into one bigger
  table. Each piece of data is rewritten only a few times, so writes stay cheap. But at any moment there are several tables
  of each size, so reads still check several tables, and a big merge temporarily needs as much free space again.
- **Leveled** compaction keeps a fixed structure. Inside each level (except L0) the tables cover **different key ranges**, so
  a lookup checks at most one table per level, and the key-range check of Exercise 3 in Lesson 17.7 skips all the others.
  The price: when level `n` is too big, one of its tables is merged into the overlapping tables of level `n + 1`, so data
  is rewritten once per level: more **write amplification**.

Again a trade-off: size-tiered favours writes, leveled favours reads and space. Our store will use the simplest possible
strategy, a **full compaction** that merges all tables into one, which is enough to see the effect.

## 4. When can a tombstone go?

A tombstone exists to hide older values of its key. It may be dropped only when no older value can exist anywhere below it.
In a full compaction of *all* tables that is true: the merge sees every version, keeps the tombstone (the newest), and there
is nothing older left to hide.

But there is a crash-safety catch. Compaction does two things: (1) write the new table, (2) delete the old tables. A crash
between the two leaves both on disk. If the new table has *dropped* a tombstone, the old tables still hold the key's old
value, the new table says nothing about the key, and after a restart the deleted key **comes back**. (If the tombstone is
kept in the new table, it still shadows the old value, and nothing bad happens.)

Real engines solve this with a **manifest**: a small file that lists the table files that are valid. Compaction writes the
new table, then replaces the manifest atomically (Lesson 13.12) with a list that contains the new table and not the old
ones, and only then deletes the old files. On restart, files that are not in the manifest are ignored and deleted. The
rename of the manifest is the single moment at which the set of tables changes. LevelDB and RocksDB call this file
`MANIFEST`.

Our store has no manifest, so its compaction keeps tombstones (`drop_tombstones = false`). That is always safe; the price is a
few bytes per deleted key. The project of Lesson 17.14 is a good place to add a manifest and drop them.

> 🔒 **Security:** "deleted" data that comes back is not only a bug. If a user deletes personal data and it reappears after a
> crash, that can break privacy rules. Also remember that an LSM tree keeps old versions on disk until compaction: deleted
> data is physically gone only after the files that held it are deleted.

## 5. Bloom filters: "definitely not here"

Compaction reduces the number of tables, but there are always several (one per level, plus L0, plus new flushes). For each
table, a lookup would like to know before reading: "could this key be in this table at all?"

A **Bloom filter** answers that with a few bits per key. It is a bit array of `m` bits, all 0 at first, and `k` hash
functions. To **add** a key, compute its `k` hashes, each giving a position in the array, and set those `k` bits to 1. To
**ask** about a key, compute the same `k` positions: if **any** of those bits is 0, the key was **definitely never added**;
if all are 1, the key was **maybe** added.

```text
 m = 16 bits, k = 3
 add "apple": positions 1, 6, 11          bits: 0100 0010 0001 0000
 add "kiwi":  positions 3, 6, 14          bits: 0101 0010 0001 0010
 ask "plum":  positions 3, 9, 14  → bit 9 is 0  → definitely not added
 ask "fig":   positions 1, 3, 11  → all are 1   → "maybe": a false positive
```

There are never false negatives: an added key always finds its own bits set. There can be **false positives**, when other
keys happened to set all of a key's bits. A false positive only costs one useless block read; the table itself then says
"not found". With the best choice of `k` (about `0.69 × bits per key`), the false-positive rate is about

```text
 p ≈ (1 − e^(−k·n/m))^k        n = keys, m = bits, k = hash functions
 10 bits per key, k = 7  →  p ≈ 0.8 %
```

So 10 bits (a bit more than one byte) per key remove more than 99% of the useless reads, whatever the size of the keys.

Create `src/bloom.rs` and add `pub mod bloom;` to `src/lib.rs`:

```rust
/// FNV-1a, a simple 64-bit hash, followed by a "finalizer" that mixes the bits well.
/// Unlike std's `DefaultHasher`, the output is fixed forever, so a filter saved to disk
/// today can be read by tomorrow's program.
fn hash(bytes: &[u8], seed: u64) -> u64 {
    let mut h = 0xcbf2_9ce4_8422_2325 ^ seed;
    for &b in bytes {
        h ^= u64::from(b);
        h = h.wrapping_mul(0x0000_0100_0000_01b3);
    }
    // The finalizer of splitmix64: spreads every input bit over all output bits.
    h ^= h >> 30;
    h = h.wrapping_mul(0xbf58_476d_1ce4_e5b9);
    h ^= h >> 27;
    h = h.wrapping_mul(0x94d0_49bb_1331_11eb);
    h ^ (h >> 31)
}

/// A Bloom filter: "definitely not in the set" or "maybe in the set".
pub struct Bloom {
    bits: Vec<u64>, // the bit array, 64 bits per word
    m: u64,         // number of bits
    k: u32,         // number of hash functions
}

impl Bloom {
    /// A filter for about `items` keys with `bits_per_key` bits each.
    pub fn new(items: usize, bits_per_key: usize) -> Bloom {
        let m = (items * bits_per_key).max(64) as u64;
        // The best number of hash functions is bits_per_key * ln 2 (about 0.69).
        let k = ((bits_per_key as f64) * std::f64::consts::LN_2)
            .round()
            .max(1.0) as u32;
        Bloom {
            bits: vec![0; m.div_ceil(64) as usize],
            m,
            k,
        }
    }

    /// The k bit positions of a key, made from two hashes ("double hashing").
    fn positions(&self, key: &[u8]) -> impl Iterator<Item = u64> + '_ {
        let h1 = hash(key, 0);
        let h2 = hash(key, 0x9e37_79b9_7f4a_7c15) | 1;
        (0..u64::from(self.k)).map(move |i| h1.wrapping_add(i.wrapping_mul(h2)) % self.m)
    }

    pub fn insert(&mut self, key: &[u8]) {
        let positions: Vec<u64> = self.positions(key).collect();
        for bit in positions {
            self.bits[(bit / 64) as usize] |= 1 << (bit % 64);
        }
    }

    /// `false` means the key was never inserted. `true` means it probably was.
    pub fn may_contain(&self, key: &[u8]) -> bool {
        self.positions(key)
            .all(|bit| self.bits[(bit / 64) as usize] & (1 << (bit % 64)) != 0)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn inserted_keys_are_always_found() {
        let mut filter = Bloom::new(10_000, 10);
        for i in 0..10_000 {
            filter.insert(format!("key{i}").as_bytes());
        }
        for i in 0..10_000 {
            assert!(filter.may_contain(format!("key{i}").as_bytes()), "key{i}");
        }
    }

    #[test]
    fn an_empty_filter_contains_nothing() {
        let filter = Bloom::new(100, 10);
        assert!(!filter.may_contain(b"anything"));
    }
}
```

**What happens:**

1. The filter is saved with the data in real engines, so its hash must never change. `std`'s `DefaultHasher` makes no such
   promise between Rust versions, so we write FNV-1a (Fowler–Noll–Vo, a classic simple hash) by hand. Its output alone
   spreads similar keys like `user:1` and `user:2` badly, so a short *finalizer* of multiplications and shifts mixes the
   bits. `wrapping_mul` multiplies and lets the result wrap around instead of panicking on overflow (Lesson 12.2).
2. Computing `k` independent hashes would be slow. **Double hashing** builds all `k` positions from two hashes:
   `h1 + i·h2` for `i = 0, 1, …, k-1`. Making `h2` odd (`| 1`) keeps the positions different from each other.
3. The bit array is a `Vec<u64>`: bit number `b` lives in word `b / 64`, at bit `b % 64`. `|=` with `1 << …` sets it, `&`
   tests it (Lesson 12.4).
4. `insert` first collects the positions into a `Vec`, because the iterator from `positions` borrows `self` while the loop
   wants to change `self.bits`; the borrow checker would not allow both at the same time.
5. `all` stops at the first bit that is 0: most absent keys are rejected after one or two bits.

How close are we to the formula? Put this into `src/bin/bloom_rate.rs`:

```rust
use p17_storage::bloom::Bloom;

const N: usize = 100_000;

fn main() {
    println!("bits/key  measured  formula");
    for bits_per_key in [5, 10, 15] {
        let mut filter = Bloom::new(N, bits_per_key);
        for i in 0..N {
            filter.insert(format!("user:{i}").as_bytes());
        }
        // Ask for N keys that were never inserted and count the false "maybe" answers.
        let false_positives = (0..N)
            .filter(|i| filter.may_contain(format!("other:{i}").as_bytes()))
            .count();
        let measured = false_positives as f64 / N as f64;

        let k = (bits_per_key as f64 * std::f64::consts::LN_2).round();
        let formula = (1.0 - (-k / bits_per_key as f64).exp()).powf(k);
        println!(
            "{bits_per_key:>8}  {:>7.2}%  {:>6.2}%",
            measured * 100.0,
            formula * 100.0
        );
    }
}
```

`cargo run --release --bin bloom_rate`:

```text
bits/key  measured  formula
       5     9.25%    9.18%
      10     0.78%    0.82%
      15     0.08%    0.07%
```

The measured rates match the formula. Each extra 5 bits per key make false positives about ten times rarer.

## 6. Putting it together

Now connect both ideas to `src/lsm.rs`. First, a filter for every table. Change the imports, the struct and add a helper:

```rust
use crate::bloom::Bloom;
use crate::compaction;
use crate::sstable::{SsTable, Value};
use std::collections::BTreeMap;
use std::fs;
use std::io;
use std::path::{Path, PathBuf};

/// A tiny LSM tree: a memtable in memory plus immutable SSTables on disk.
pub struct Lsm {
    dir: PathBuf,
    memtable: BTreeMap<Vec<u8>, Value>,
    memtable_bytes: usize,
    memtable_limit: usize,
    tables: Vec<SsTable>, // newest first
    filters: Vec<Bloom>,  // filters[i] belongs to tables[i]
    next_id: u64,
}

const BITS_PER_KEY: usize = 10;

/// Builds a Bloom filter with all keys of `table`.
fn build_filter(table: &SsTable) -> io::Result<Bloom> {
    let entries = table.scan()?;
    let mut filter = Bloom::new(entries.len(), BITS_PER_KEY);
    for (key, _) in &entries {
        filter.insert(key);
    }
    Ok(filter)
}
```

In `open`, build a filter for each table, and add `filters,` to the `Ok(Lsm { … })` after `tables,`:

```rust
        let mut tables = Vec::new();
        let mut filters = Vec::new();
        for id in &ids {
            let table = SsTable::open(&dir.join(format!("{id:06}.sst")))?;
            filters.push(build_filter(&table)?);
            tables.push(table);
        }
```

In `flush` and `get`:

```rust
    pub fn flush(&mut self) -> io::Result<()> {
        if self.memtable.is_empty() {
            return Ok(());
        }
        let path = self.dir.join(format!("{:06}.sst", self.next_id));
        SsTable::write(&path, self.memtable.iter())?;
        let table = SsTable::open(&path)?;
        self.filters.insert(0, build_filter(&table)?);
        self.tables.insert(0, table); // the newest table goes first
        self.next_id += 1;
        self.memtable.clear();
        self.memtable_bytes = 0;
        Ok(())
    }

    pub fn get(&self, key: &[u8]) -> io::Result<Option<Vec<u8>>> {
        if let Some(value) = self.memtable.get(key) {
            return Ok(value.clone()); // newest data; None means deleted
        }
        for (table, filter) in self.tables.iter().zip(&self.filters) {
            if !filter.may_contain(key) {
                continue; // definitely not in this table: skip it without reading
            }
            if let Some(value) = table.get(key)? {
                return Ok(value); // the first (newest) table that knows the key wins
            }
        }
        Ok(None)
    }
```

And the compaction, a new method in `impl Lsm`:

```rust
    pub fn compact(&mut self) -> io::Result<()> {
        if self.tables.len() < 2 {
            return Ok(());
        }
        let mut runs = Vec::new();
        for table in &self.tables {
            runs.push(table.scan()?); // newest first, as `merge` expects
        }
        // Keep tombstones: see "When can a tombstone go?" in the lesson.
        let merged = compaction::merge(runs, false);

        let id = self.next_id;
        let path = self.dir.join(format!("{id:06}.sst"));
        // 1. the new table is complete and durable before anything is deleted
        SsTable::write(&path, merged.iter().map(|(key, value)| (key, value)))?;
        let table = SsTable::open(&path)?;
        let filter = build_filter(&table)?;

        for entry in fs::read_dir(&self.dir)? {
            let entry = entry?;
            let name = entry.file_name().to_string_lossy().into_owned();
            let old = name
                .strip_suffix(".sst")
                .and_then(|n| n.parse::<u64>().ok())
                .is_some_and(|n| n < id);
            if old {
                fs::remove_file(entry.path())?; // 2. only now delete the old tables
            }
        }
        fs::File::open(&self.dir)?.sync_all()?;

        self.tables = vec![table];
        self.filters = vec![filter];
        self.next_id = id + 1;
        Ok(())
    }
```

**What happens:**

1. `build_filter` scans a table once and adds every key, deleted keys included: a tombstone is an answer the lookup must
   find. Real engines store the filter inside the SSTable file instead of rebuilding it at start-up (Exercise 3 prepares
   that).
2. `zip` walks tables and filters together. A table whose filter says "definitely not" is skipped without any read.
3. `compact` scans all tables newest first, merges them, and writes the result as a table with a **new, larger** number.
   `merged.iter().map(|(key, value)| (key, value))` turns `&(K, V)` items into the `(&K, &V)` pairs that `write` expects.
4. Only after the new table is durable are the old files deleted, and then the directory is synced. A crash in between
   leaves extra old tables, which is harmless here because the new table is newer and kept its tombstones (Section 4).

A test, in the tests of `src/lsm.rs`:

```rust
    #[test]
    fn compaction_keeps_the_newest_data() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 1_000_000)?;
        for round in 0..5 {
            for i in 0..100 {
                db.put(
                    format!("key{i:03}").as_bytes(),
                    format!("v{round}").as_bytes(),
                )?;
            }
            db.delete(format!("key{round:03}").as_bytes())?;
            db.flush()?;
        }
        assert_eq!(db.table_count(), 5);

        db.compact()?;
        assert_eq!(db.table_count(), 1);
        assert_eq!(db.get(b"key050")?, Some(b"v4".to_vec()));
        assert_eq!(db.get(b"key004")?, None); // deleted in the last round
        assert_eq!(db.get(b"key003")?, Some(b"v4".to_vec())); // deleted, then written again
        drop(db);

        let db = Lsm::open(dir.path(), 1_000_000)?;
        assert_eq!(db.table_count(), 1);
        assert_eq!(db.get(b"key050")?, Some(b"v4".to_vec()));
        Ok(())
    }
```

Run the tests of the three modules together (several filters after `--` are allowed):

```bash
cargo test --lib -- bloom compaction lsm
```

```text
running 9 tests
test compaction::tests::tombstones_can_be_dropped ... ok
test bloom::tests::an_empty_filter_contains_nothing ... ok
test compaction::tests::newest_version_wins_and_output_is_sorted ... ok
test bloom::tests::inserted_keys_are_always_found ... ok
test lsm::tests::scan_all_merges_everything ... ok
test lsm::tests::newest_value_wins_across_memtable_and_tables ... ok
test lsm::tests::a_tombstone_survives_flush_and_reopen ... ok
test lsm::tests::compaction_keeps_the_newest_data ... ok
test lsm::tests::flushed_data_survives_a_reopen ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 29 filtered out; finished in 0.15s
```

Finally the demo of Lesson 17.7 again, now with filters and a compaction. Put this into `src/bin/compaction_demo.rs`:

```rust
use p17_storage::lsm::Lsm;
use std::path::Path;
use std::time::Instant;

const KEYS: u64 = 100_000;

fn reads_for(db: &Lsm, key: &[u8]) -> std::io::Result<(bool, u64)> {
    let before = db.block_reads();
    let found = db.get(key)?.is_some();
    Ok((found, db.block_reads() - before))
}

fn main() -> std::io::Result<()> {
    let dir = Path::new("compaction-demo");
    let mut db = Lsm::open(dir, 256 * 1024)?;
    for i in 0..KEYS {
        let n = i * 7919 % KEYS; // the same scrambled order as in Lesson 17.7
        let value = format!("value of {n} {}", "-".repeat(40));
        db.put(format!("user:{n:06}").as_bytes(), value.as_bytes())?;
    }
    db.flush()?;

    println!("with Bloom filters, {} tables:", db.table_count());
    println!("  existing key: {:?}", reads_for(&db, b"user:050000")?);
    println!("  missing key:  {:?}", reads_for(&db, b"user:050000x")?);

    let start = Instant::now();
    db.compact()?;
    println!("compaction took {:?}", start.elapsed());

    println!("after compaction, {} table:", db.table_count());
    println!("  existing key: {:?}", reads_for(&db, b"user:050000")?);
    println!("  missing key:  {:?}", reads_for(&db, b"user:050000x")?);

    std::fs::remove_dir_all(dir)?;
    Ok(())
}
```

`cargo run --release --bin compaction_demo` (each pair is `(found, blocks read)`):

```text
with Bloom filters, 26 tables:
  existing key: (true, 1)
  missing key:  (false, 0)
compaction took 50.923625ms
after compaction, 1 table:
  existing key: (true, 1)
  missing key:  (false, 0)
```

Compare with Lesson 17.7: 14 and 26 block reads became 1 and 0. The filters alone did that, even with 26 tables: 25 of
them said "definitely not" for the existing key, and all 26 for the missing one. (With about 0.8% false positives, an
occasional run shows one extra read.) Compaction then reduced 26 files to 1, in about 50 ms for 8 MB (megabytes) of data. This demo writes
every key only once, so the total size stays about the same; with overwrites and deletes, compaction also frees space (Try it
yourself, step 1).

## Common mistakes

1. **Dropping a tombstone during the merge loop.** The older version that follows becomes the "newest" and the deleted key
   comes back. Decide about tombstones after the newest version of each key is chosen.
2. **Dropping tombstones without a manifest.** A crash between writing the new table and deleting the old ones brings
   deleted keys back (Section 4).
3. **Building a Bloom filter with `DefaultHasher` and saving it.** Its algorithm may change between Rust versions, and the saved
   filter would then answer "definitely not" for keys that are there: false negatives, real data loss on reads.
4. **Leaving tombstones out of the filter.** The lookup must find the tombstone to stop; if the filter skips that table, an
   older table returns the deleted value.
5. **Deleting old tables before the new one is durable.** A crash then loses data. Write, sync, and only then delete.

## Try it yourself

1. In `compaction_demo.rs`, print the sum of the `.sst` file sizes before and after `compact` (use `fs::read_dir` and
   `metadata().len()`). Then overwrite every key once more before compacting and compare again.
2. Change `BITS_PER_KEY` to 2 and run the demo several times. How often does the missing key cost a read now?
3. In `bloom_rate.rs`, replace the finalizer part of `hash` with just `h` (plain FNV-1a) and run it. What happens to the
   measured rates? (Put the finalizer back afterwards.)

## Exercises

### Exercise 1 — How many bits? (easy)

For a target false-positive rate `p`, the best filter needs `-ln(p) / (ln 2)²` bits per key. Add
`pub fn bits_per_key_for(p: f64) -> f64` to `src/bloom.rs` and test it: 1% needs about 9.585 bits per key, 0.1% about
14.378. How many megabytes does a 1% filter need for 100 million keys?

<details>
<summary>Hint</summary>

`p.ln()` is the natural logarithm, and `std::f64::consts::LN_2` is ln 2. Compare floating-point numbers with a tolerance
(Lesson 12.3): `(a - b).abs() < 0.001`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Exercise 1: bits per key needed for a false-positive rate `p` (with the best k).
pub fn bits_per_key_for(p: f64) -> f64 {
    -p.ln() / (std::f64::consts::LN_2 * std::f64::consts::LN_2)
}
```

```rust
    #[test]
    fn bits_for_a_rate() {
        assert!((bits_per_key_for(0.01) - 9.585).abs() < 0.001);
        assert!((bits_per_key_for(0.001) - 14.378).abs() < 0.001);
    }
```

100 million keys × 9.585 bits ≈ 958.5 million bits ≈ 120 MB. Why it works: every ten times lower rate costs only about
4.8 more bits per key, which is why real engines can afford filters for every table. 120 MB is small next to the data of
100 million keys, and much smaller than a hash index of them (Lesson 17.5).

</details>

### Exercise 2 — Test the false-positive rate (medium)

Write a test `false_positive_rate_is_about_one_percent`: add 10,000 keys `in:0` … `in:9999` to a filter with 10 bits per
key, ask for 10,000 other keys `out:0` … `out:9999`, and check that fewer than 150 of them (1.5%) get "maybe".

<details>
<summary>Hint</summary>

Count with `(0..10_000).filter(|i| filter.may_contain(...)).count()`. Why 1.5% and not exactly 0.82%? The result of one run
varies a little; a test needs a margin, or it becomes flaky.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn false_positive_rate_is_about_one_percent() {
        let mut filter = Bloom::new(10_000, 10);
        for i in 0..10_000 {
            filter.insert(format!("in:{i}").as_bytes());
        }
        let false_positives = (0..10_000)
            .filter(|i| filter.may_contain(format!("out:{i}").as_bytes()))
            .count();
        assert!(false_positives < 150, "{false_positives} false positives");
    }
```

Why it works: the hash is fixed, so this test gives the same count on every run and machine; it is not flaky. The margin
still matters: a small change to the hash would change the count, and the test should only fail when the filter really got
worse. The message shows the number when it fails.

</details>

### Exercise 3 — Save a filter (hard)

Real engines store each table's filter on disk, so that opening a table does not have to scan it. Add
`pub fn to_bytes(&self) -> Vec<u8>` and `pub fn from_bytes(bytes: &[u8]) -> Option<Bloom>` to `Bloom`. Format: `k` as a
`u32` LE, `m` as a `u64` LE, then the words as `u64` LE. `from_bytes` returns `None` for inputs that are too short, have
`k` or `m` equal to 0, or have the wrong number of word bytes for `m`. Test the round trip and two broken inputs.

<details>
<summary>Hint</summary>

The number of words is `m.div_ceil(64)`, so the word bytes must be exactly `m.div_ceil(64) * 8`. `bytes.get(0..4)` returns
`None` instead of panicking when the input is too short, and `?` passes that `None` on. `as_chunks::<8>()` splits a slice into
`[u8; 8]` arrays.

</details>

<details>
<summary>Answer</summary>

```rust
impl Bloom {
    /// Exercise 3: the filter as bytes: k (u32 LE), m (u64 LE), then the words (u64 LE).
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut out = Vec::with_capacity(12 + self.bits.len() * 8);
        out.extend_from_slice(&self.k.to_le_bytes());
        out.extend_from_slice(&self.m.to_le_bytes());
        for word in &self.bits {
            out.extend_from_slice(&word.to_le_bytes());
        }
        out
    }

    /// Exercise 3: the opposite of `to_bytes`; `None` if the bytes do not fit together.
    pub fn from_bytes(bytes: &[u8]) -> Option<Bloom> {
        let k = u32::from_le_bytes(bytes.get(0..4)?.try_into().ok()?);
        let m = u64::from_le_bytes(bytes.get(4..12)?.try_into().ok()?);
        let words = &bytes[12..];
        if k == 0 || m == 0 || words.len() as u64 != m.div_ceil(64) * 8 {
            return None;
        }
        let (chunks, _) = words.as_chunks::<8>(); // the length check above leaves no rest
        let bits = chunks.iter().map(|w| u64::from_le_bytes(*w)).collect();
        Some(Bloom { bits, m, k })
    }
}
```

```rust
    #[test]
    fn a_filter_survives_bytes() {
        let mut filter = Bloom::new(1_000, 10);
        for i in 0..1_000 {
            filter.insert(format!("key{i}").as_bytes());
        }
        let bytes = filter.to_bytes();
        let back = Bloom::from_bytes(&bytes).unwrap();
        assert_eq!(back.to_bytes(), bytes);
        assert!(back.may_contain(b"key7"));
        assert!(Bloom::from_bytes(&bytes[..bytes.len() - 1]).is_none());
        assert!(Bloom::from_bytes(&[1, 2, 3]).is_none());
    }
```

`cargo test --lib bloom` (with Exercises 1 and 2):

```text
running 5 tests
test bloom::tests::bits_for_a_rate ... ok
test bloom::tests::an_empty_filter_contains_nothing ... ok
test bloom::tests::a_filter_survives_bytes ... ok
test bloom::tests::false_positive_rate_is_about_one_percent ... ok
test bloom::tests::inserted_keys_are_always_found ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 36 filtered out; finished in 0.01s
```

Why it works: all lengths are checked before the words are decoded, so damaged bytes give `None`, not a panic or a filter
of the wrong size (the rule of Lesson 17.3). A wrong `m` would map keys to wrong bits and cause false negatives, the one
error a Bloom filter must never make. In a real SSTable these bytes would go into their own block, protected by a CRC (Cyclic Redundancy Check, the checksum of Lesson 17.3).

</details>

## Quiz

1. What two problems does compaction solve?

<details>
<summary>Answer</summary>

Read amplification (fewer tables to check) and space amplification (old versions, and with a manifest also tombstones, are
removed).

</details>

2. In a k-way merge of 10 runs with a heap, how many items are in the heap at most, and what does each step cost?

<details>
<summary>Answer</summary>

At most 10, one per run. Each pop and push costs O(log 10), however long the runs are.

</details>

3. Why must the merge prefer the newest run for equal keys?

<details>
<summary>Answer</summary>

The newest version is the current value of the key (or its tombstone). Taking an older version would bring back an old value
or a deleted key.

</details>

4. Compare size-tiered and leveled compaction in one sentence each.

<details>
<summary>Answer</summary>

Size-tiered merges tables of similar size into bigger ones: little rewriting, but several tables to search and big temporary
space needs. Leveled keeps non-overlapping tables in levels that grow ten times each: at most one table per level to search,
but data is rewritten at every level.

</details>

5. A Bloom filter says "maybe". What does the lookup do, and what is the worst outcome?

<details>
<summary>Answer</summary>

It reads the table's block. The worst outcome is a false positive: one useless read, after which the table says "not found".
The answer is never wrong.

</details>

6. Why does our compaction keep tombstones?

<details>
<summary>Answer</summary>

Without a manifest, a crash between writing the new table and deleting the old ones would leave old values on disk; a new table
without the tombstone would then let a deleted key come back. A kept tombstone keeps shadowing them.

</details>

## Summary

- Compaction merges SSTables into fewer, keeps only the newest version of each key, and frees space; it fixes most of the
  LSM's read and space amplification.
- A k-way merge with a min-heap merges sorted runs in one sequential pass; for equal keys the newest run wins.
- Size-tiered compaction favours writes; leveled compaction favours reads and space at the cost of more rewriting.
- A tombstone may go only when nothing older can exist and no crash can bring old tables back; real engines use a manifest.
- A Bloom filter (about 10 bits per key, `k` ≈ 7) answers "definitely not" or "maybe" with about 0.8% false positives and no
  false negatives.
- Its hash must be fixed forever; we use FNV-1a with a finalizer and double hashing.
- Together, filters and compaction turned 26 block reads for a missing key into 0.

## Sources and further reading

- [RocksDB wiki — Compaction](https://github.com/facebook/rocksdb/wiki/Compaction) — leveled, universal (size-tiered) and
  FIFO (first in, first out) compaction in the most widely used LSM engine.
- [RocksDB wiki — RocksDB Bloom Filter](https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter) — how a real engine
  stores filters per table and chooses bits per key.
- [LevelDB implementation notes](https://github.com/google/leveldb/blob/main/doc/impl.md) — levels, compactions and the
  `MANIFEST` file.
- [Less Hashing, Same Performance: Building a Better Bloom Filter](https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf)
  — Kirsch and Mitzenmacher's paper behind double hashing (advanced).
- [std::collections::BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) and
  [std::cmp::Reverse](https://doc.rust-lang.org/std/cmp/struct.Reverse.html) — the heap used by the merge.

---

[🏠 Start](../START-HERE.md) · [← 17.7 LSM trees](07-lsm-trees.md) · [📋 Phase 17](README.md) · [17.9 The page cache and the buffer pool →](09-page-cache-and-buffer-pool.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
