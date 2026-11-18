<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.6 B-trees](06-b-trees.md) · [📋 Phase 17](README.md) · [17.8 Compaction and Bloom filters →](08-compaction-and-bloom-filters.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/07-lsm-trees.md) <!-- nav -->

# Lesson 17.7 — LSM trees

> **Phase 17 — Storage and database internals** · Lesson 7 of 15 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How an LSM tree combines the fast appends of a log with the sorted order of a tree.
- What a memtable and an SSTable are, and how writes, deletes and reads move through them.
- How to write and read an SSTable with a sparse index, so that a lookup reads one small block.
- How to build a small LSM store in Rust that keeps its data across restarts.
- Why reads get slower as SSTables pile up (read amplification), and how B-trees and LSM trees differ.

## Before you start

- You finished [Lesson 17.6 — B-trees](06-b-trees.md) and [Lesson 17.5 — Indexes: a hash index](05-hash-indexes.md).
  The LSM tree takes the append-only idea from 17.5 and the sorted order from 17.6.
- You know the WAL from [Lesson 17.4](04-write-ahead-log.md), the record format from [Lesson 17.3](03-binary-file-formats.md),
  and the atomic-replace pattern from [Lesson 13.12](../13-operating-systems/12-file-systems-inside.md).
- You know interior mutability (`RefCell`) from [Lesson 05.9 — Smart pointers](../05-rust-intermediate/09-smart-pointers.md).
  This lesson uses `Cell`, its simpler sibling for small `Copy` values such as counters: `get` reads the value, `set`
  replaces it, both through a shared reference.

## New words

| Word | Meaning |
|---|---|
| **LSM tree** (Log-Structured Merge tree) | A storage design that collects writes in a sorted in-memory table, writes it out as an immutable sorted file when it is full, and later merges those files. |
| **memtable** | The in-memory, sorted table that receives all new writes of an LSM tree. Here a `BTreeMap`. |
| **SSTable** (Sorted String Table) | An immutable file of key-value records sorted by key, with an index. "String" is historical: keys and values are just bytes. |
| **flush** | Writing the full memtable to disk as a new SSTable and starting an empty memtable. |
| **immutable** | Never changed after it is written. An SSTable is only ever created, read and (later) deleted. |
| **sparse index** | An index with one entry per *block* of records instead of one per record, so it stays small. |
| **shadowing** | A newer value for a key hides all older values of that key in older files. |
| **read amplification** | Reading more than one place (or more bytes than needed) to answer one lookup. |

## 1. Two ideas in one

The hash log of Lesson 17.5 writes fast (it only appends) but cannot answer range queries and needs all keys in RAM. The
B-tree of Lesson 17.6 is sorted but changes pages in place, so every insert is a random write of a whole page. An **LSM
tree** keeps the good part of both: it only ever writes **sequentially**, and it keeps the data **sorted**.

The trick is to sort in memory and write in big sorted pieces:

```text
 write path                                    read path (newest first)
 ──────────                                    ─────────
 put/delete ─► WAL (durable, Lesson 17.4)      get(key)
            └► memtable (BTreeMap, in RAM)        1. memtable
                   │ full?                        2. newest SSTable
                   ▼ flush                        3. older SSTable
               000003.sst  (newest)               4. ...
               000002.sst                         5. oldest SSTable
               000001.sst  (oldest)               the first one that knows the key wins
```

1. Every write goes into the **memtable**, a sorted map in memory. (And first, for durability, into a WAL; see Section 6.)
2. When the memtable is big enough, it is **flushed**: written out in one go, in key order, as a new **SSTable** file. Then
   a new empty memtable starts. The SSTable is **immutable**: it is never changed again.
3. A read looks in the memtable first, then in the SSTables from the newest to the oldest. The first place that knows the
   key has the newest value. This is **shadowing**.
4. A delete cannot remove a key from old, immutable files, so it writes a **tombstone** (Lesson 17.5), which shadows the
   older values.
5. Over time, many SSTables pile up. **Compaction** merges them in the background (Lesson 17.8).

LevelDB, RocksDB, Cassandra and many other databases are LSM trees. RocksDB alone runs under a large part of the world's
key-value storage.

## 2. The SSTable format

An SSTable is our record format from Lesson 17.3, sorted by key, plus an index at the end:

```text
 ┌──────────────────────── data ─────────────────────────┬──── index ────┬────────┐
 │ block 0: 16 records │ block 1: 16 records │ block 2 … │ (key0, off 0) │ index  │
 │ apple … bean        │ beet … corn         │           │ (beet, off X) │ offset │
 │                     │                     │           │ …             │ u64 LE │
 └─────────────────────┴─────────────────────┴───────────┴───────────────┴────────┘
                                                          ▲                  │
                                                          └──────────────────┘
```

- The **data** is all records in key order. Each record's value starts with the type byte of Lesson 17.4 (1 = put,
  0 = tombstone).
- Every 16 records start a new **block**. The **index** has one entry per block: the block's first key and its offset. This
  is a **sparse index**: 16 times smaller than one entry per record.
- The last 8 bytes, the **footer**, hold the offset where the index starts. A reader reads the footer first, then the
  index, and keeps the index in memory.

To find a key, binary-search the index for the last block whose first key is `<=` the key, read that one block (16 small
records, one positioned read), and scan it. Because the file is sorted, the key can only be in that block.

## 3. Writing and reading an SSTable

Create `src/sstable.rs` and add `pub mod sstable;` to `src/lib.rs`. The types and helpers:

```rust
use crate::record;
use std::cell::Cell;
use std::fs::{self, File};
use std::io::{self, Write};
use std::os::unix::fs::FileExt;
use std::path::Path;

const PUT: u8 = 1;
const DELETE: u8 = 0;
/// One index entry for every this many records.
const BLOCK_RECORDS: usize = 16;

/// A value, or `None` for a tombstone (the key was deleted).
pub type Value = Option<Vec<u8>>;

/// An immutable, sorted file of key-value records with a sparse index.
/// Layout: [data records][index records][index offset: u64 LE]
pub struct SsTable {
    file: File,
    index: Vec<(Vec<u8>, u64)>, // first key of each block, offset of the block
    data_end: u64,
    /// How many blocks were read from the file (for measuring).
    pub block_reads: Cell<u64>,
}

fn encode_value(value: &Value) -> Vec<u8> {
    match value {
        Some(v) => {
            let mut entry = vec![PUT];
            entry.extend_from_slice(v);
            entry
        }
        None => vec![DELETE],
    }
}

fn decode_value(entry: &[u8]) -> Value {
    match entry.split_first() {
        Some((&PUT, v)) => Some(v.to_vec()),
        _ => None,
    }
}

fn invalid(msg: &str) -> io::Error {
    io::Error::new(io::ErrorKind::InvalidData, msg.to_string())
}
```

`Value = Option<Vec<u8>>` is the whole idea of a tombstone in one type: `Some(bytes)` is a value, `None` is "deleted".
`block_reads` is a counter for measuring; it is a `Cell` because `get` takes `&self` and still needs to count.

Writing, in `impl SsTable`:

```rust
impl SsTable {
    /// Writes `entries` (which must be sorted by key, without duplicates) to a new file at `path`.
    pub fn write<'a>(
        path: &Path,
        entries: impl Iterator<Item = (&'a Vec<u8>, &'a Value)>,
    ) -> io::Result<()> {
        let mut data = Vec::new();
        let mut index = Vec::new();
        for (i, (key, value)) in entries.enumerate() {
            if i % BLOCK_RECORDS == 0 {
                // This record starts a new block: remember its key and offset.
                record::encode(key, &(data.len() as u64).to_le_bytes(), &mut index);
            }
            record::encode(key, &encode_value(value), &mut data);
        }
        let index_offset = data.len() as u64;
        data.extend_from_slice(&index);
        data.extend_from_slice(&index_offset.to_le_bytes());

        // Atomic replace (Lesson 13.12): a crash leaves no half-written table.
        let tmp = path.with_extension("tmp");
        let mut file = File::create(&tmp)?;
        file.write_all(&data)?;
        file.sync_all()?;
        fs::rename(&tmp, path)?;
        File::open(path.parent().unwrap_or(Path::new(".")))?.sync_all()
    }
```

**What happens:**

1. `entries` is any iterator of `(key, value)` pairs, for example `memtable.iter()` of a `BTreeMap`, which yields its
   entries in sorted order. The lifetime `'a` says the references live at least as long as the call.
2. Index entries are ordinary records too: the key, and the block's offset as 8 little-endian bytes as the value. So the
   index gets lengths and checksums for free.
3. The whole file is built in memory and written with one `write_all`: one large sequential write (Lesson 17.1).
4. The file is written under a temporary name, synced and renamed, and the directory is synced. A reader sees either no
   table or a complete one, never half of one.

Opening and reading, still in `impl SsTable`:

```rust
    pub fn open(path: &Path) -> io::Result<SsTable> {
        let file = File::open(path)?;
        let len = file.metadata()?.len();
        if len < 8 {
            return Err(invalid("file too short"));
        }
        let mut footer = [0u8; 8];
        file.read_exact_at(&mut footer, len - 8)?;
        let data_end = u64::from_le_bytes(footer);
        if data_end > len - 8 {
            return Err(invalid("bad footer"));
        }

        let mut raw = vec![0u8; (len - 8 - data_end) as usize];
        file.read_exact_at(&mut raw, data_end)?;
        let mut index = Vec::new();
        let mut pos = 0;
        while pos < raw.len() {
            let (rec, used) = record::decode(&raw[pos..]).map_err(|_| invalid("bad index"))?;
            let offset =
                u64::from_le_bytes(rec.value.try_into().map_err(|_| invalid("bad index"))?);
            index.push((rec.key, offset));
            pos += used;
        }
        Ok(SsTable {
            file,
            index,
            data_end,
            block_reads: Cell::new(0),
        })
    }

    /// `None`: this table knows nothing about the key.
    /// `Some(None)`: the key was deleted. `Some(Some(v))`: the key has value `v`.
    pub fn get(&self, key: &[u8]) -> io::Result<Option<Value>> {
        // The block to read is the last one whose first key is <= `key`.
        let n = self
            .index
            .partition_point(|(first, _)| first.as_slice() <= key);
        if n == 0 {
            return Ok(None); // smaller than the first key of the table
        }
        let start = self.index[n - 1].1;
        let end = self
            .index
            .get(n)
            .map_or(self.data_end, |(_, offset)| *offset);

        let mut block = vec![0u8; (end - start) as usize];
        self.file.read_exact_at(&mut block, start)?;
        self.block_reads.set(self.block_reads.get() + 1);

        let mut pos = 0;
        while pos < block.len() {
            let (rec, used) = record::decode(&block[pos..]).map_err(|_| invalid("bad record"))?;
            if rec.key.as_slice() == key {
                return Ok(Some(decode_value(&rec.value)));
            }
            if rec.key.as_slice() > key {
                break; // sorted: the key is not in this block
            }
            pos += used;
        }
        Ok(None)
    }

    /// Reads every record of the table, in key order.
    pub fn scan(&self) -> io::Result<Vec<(Vec<u8>, Value)>> {
        let mut data = vec![0u8; self.data_end as usize];
        self.file.read_exact_at(&mut data, 0)?;
        let mut out = Vec::new();
        let mut pos = 0;
        while pos < data.len() {
            let (rec, used) = record::decode(&data[pos..]).map_err(|_| invalid("bad record"))?;
            out.push((rec.key, decode_value(&rec.value)));
            pos += used;
        }
        Ok(out)
    }
}
```

**What happens:**

1. `open` reads the footer, checks that it points inside the file (a value from disk is input, Lesson 17.3), reads the
   index area and decodes it into a `Vec` of `(first key, offset)`. The data itself is not read.
2. `rec.value.try_into()` turns the 8-byte `Vec<u8>` into `[u8; 8]`; it fails if the length is wrong, which we report as
   damage.
3. `get` uses `partition_point` (a binary search on a sorted slice): it returns how many index entries have a first key
   `<= key`. The block to read is the last of them. If there is none, the key is smaller than everything in the table.
4. The block ends where the next block starts, or at `data_end` for the last block. One `read_exact_at` reads it.
5. Inside the block, the scan stops early when it passes the key, because the records are sorted.
6. `get` has three answers, so it returns `Option<Value>` = `Option<Option<Vec<u8>>>`: "don't know", "deleted", or a
   value. The LSM needs to tell "don't know, ask an older table" from "deleted, stop".
7. `scan` reads all data records; Lesson 17.8 uses it to merge tables.

A test, at the end of `src/sstable.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::BTreeMap;

    #[test]
    fn write_then_get_every_kind_of_key() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("000001.sst");
        let mut entries: BTreeMap<Vec<u8>, Value> = BTreeMap::new();
        for i in 0..100u32 {
            let key = format!("key{:03}", i * 2).into_bytes(); // even numbers only
            entries.insert(key, Some(i.to_string().into_bytes()));
        }
        entries.insert(b"key050".to_vec(), None); // a tombstone
        SsTable::write(&path, entries.iter())?;

        let table = SsTable::open(&path)?;
        assert_eq!(table.index.len(), 7); // 100 records, a block every 16
        assert_eq!(table.get(b"key000")?, Some(Some(b"0".to_vec())));
        assert_eq!(table.get(b"key198")?, Some(Some(b"99".to_vec())));
        assert_eq!(table.get(b"key050")?, Some(None)); // deleted
        assert_eq!(table.get(b"key051")?, None); // between two keys
        assert_eq!(table.get(b"aaa")?, None); // before the first key
        assert_eq!(table.get(b"zzz")?, None); // after the last key
        assert_eq!(table.scan()?.len(), 100);
        Ok(())
    }
}
```

`cargo test --lib sstable`:

```text
running 1 test
test sstable::tests::write_then_get_every_kind_of_key ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 27 filtered out; finished in 0.02s
```

The test checks the first and last key, a tombstone, a key between two keys (it would be in a block but is not), and keys
outside the table. These are the edge cases of any sorted lookup.

## 4. The LSM store

Now the store that ties memtable and SSTables together. Create `src/lsm.rs` and add `pub mod lsm;` to `src/lib.rs`:

```rust
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
    next_id: u64,
}

impl Lsm {
    /// Opens (or creates) a store in `dir`. The memtable is flushed when it holds
    /// about `memtable_limit` bytes of keys and values.
    pub fn open(dir: &Path, memtable_limit: usize) -> io::Result<Lsm> {
        fs::create_dir_all(dir)?;
        let mut ids = Vec::new();
        for entry in fs::read_dir(dir)? {
            let name = entry?.file_name().to_string_lossy().into_owned();
            if let Some(id) = name
                .strip_suffix(".sst")
                .and_then(|n| n.parse::<u64>().ok())
            {
                ids.push(id);
            }
        }
        ids.sort_unstable_by(|a, b| b.cmp(a)); // newest (largest id) first
        let mut tables = Vec::new();
        for id in &ids {
            tables.push(SsTable::open(&dir.join(format!("{id:06}.sst")))?);
        }
        Ok(Lsm {
            dir: dir.to_path_buf(),
            memtable: BTreeMap::new(),
            memtable_bytes: 0,
            memtable_limit,
            tables,
            next_id: ids.first().map_or(1, |newest| newest + 1),
        })
    }

    pub fn put(&mut self, key: &[u8], value: &[u8]) -> io::Result<()> {
        self.write(key, Some(value.to_vec()))
    }

    pub fn delete(&mut self, key: &[u8]) -> io::Result<()> {
        self.write(key, None) // a tombstone
    }

    fn write(&mut self, key: &[u8], value: Value) -> io::Result<()> {
        self.memtable_bytes += key.len() + value.as_ref().map_or(0, |v| v.len());
        self.memtable.insert(key.to_vec(), value);
        if self.memtable_bytes >= self.memtable_limit {
            self.flush()?;
        }
        Ok(())
    }

    /// Writes the memtable to a new SSTable and empties it.
    pub fn flush(&mut self) -> io::Result<()> {
        if self.memtable.is_empty() {
            return Ok(());
        }
        let path = self.dir.join(format!("{:06}.sst", self.next_id));
        SsTable::write(&path, self.memtable.iter())?;
        self.tables.insert(0, SsTable::open(&path)?); // the newest table goes first
        self.next_id += 1;
        self.memtable.clear();
        self.memtable_bytes = 0;
        Ok(())
    }

    pub fn get(&self, key: &[u8]) -> io::Result<Option<Vec<u8>>> {
        if let Some(value) = self.memtable.get(key) {
            return Ok(value.clone()); // newest data; None means deleted
        }
        for table in &self.tables {
            if let Some(value) = table.get(key)? {
                return Ok(value); // the first (newest) table that knows the key wins
            }
        }
        Ok(None)
    }

    pub fn table_count(&self) -> usize {
        self.tables.len()
    }

    /// Total number of blocks read from all SSTables so far.
    pub fn block_reads(&self) -> u64 {
        self.tables.iter().map(|t| t.block_reads.get()).sum()
    }
}
```

**What happens:**

1. SSTables are named by a growing number with six digits (`000001.sst`), so a bigger number means newer data. `open`
   collects the numbers from the directory and opens the tables newest first. Leftover `.tmp` files from a crash during a
   flush are ignored.
2. `put` and `delete` both go through `write`: a delete is just a write of `None`. `memtable_bytes` is a rough size
   estimate that decides when to flush.
3. `flush` writes the memtable as a new table, puts it at the front of `tables`, and starts a new memtable. Writing a
   memtable is one sequential write of sorted data, however random the order of the puts was.
4. `get` follows the read path of Section 1 exactly. `table.get` returns `Some(...)` as soon as a table knows the key, and
   that answer, a value or a tombstone, is final.

Two tests, at the end of `src/lsm.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn newest_value_wins_across_memtable_and_tables() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 1_000_000)?;
        db.put(b"apple", b"red")?;
        db.put(b"kiwi", b"green")?;
        db.flush()?; // table 1
        db.put(b"apple", b"yellow")?;
        db.delete(b"kiwi")?;
        db.flush()?; // table 2
        db.put(b"plum", b"purple")?; // still in the memtable

        assert_eq!(db.table_count(), 2);
        assert_eq!(db.get(b"apple")?, Some(b"yellow".to_vec())); // table 2 shadows table 1
        assert_eq!(db.get(b"kiwi")?, None); // the tombstone in table 2 hides table 1
        assert_eq!(db.get(b"plum")?, Some(b"purple".to_vec()));
        Ok(())
    }

    #[test]
    fn flushed_data_survives_a_reopen() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 100)?; // tiny limit: many flushes
        for i in 0..50 {
            db.put(format!("key{i:02}").as_bytes(), b"0123456789")?;
        }
        db.flush()?;
        let tables = db.table_count();
        drop(db);

        let db = Lsm::open(dir.path(), 100)?;
        assert_eq!(db.table_count(), tables);
        assert_eq!(db.get(b"key07")?, Some(b"0123456789".to_vec()));
        assert_eq!(db.get(b"key99")?, None);
        Ok(())
    }
}
```

`cargo test --lib lsm`:

```text
running 2 tests
test lsm::tests::newest_value_wins_across_memtable_and_tables ... ok
test lsm::tests::flushed_data_survives_a_reopen ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 28 filtered out; finished in 0.09s
```

## 5. The cost: read amplification

Every flush adds a table, and a lookup may have to look into all of them. Put this into `src/bin/lsm_demo.rs`:

```rust
use p17_storage::lsm::Lsm;
use std::path::Path;

const KEYS: u64 = 100_000;

fn main() -> std::io::Result<()> {
    let dir = Path::new("lsm-demo");
    let mut db = Lsm::open(dir, 256 * 1024)?; // flush every ~256 KiB

    // Write 100,000 keys in a scrambled order (7919 shares no factor with KEYS, so every key comes once).
    for i in 0..KEYS {
        let n = i * 7919 % KEYS;
        let value = format!("value of {n} {}", "-".repeat(40));
        db.put(format!("user:{n:06}").as_bytes(), value.as_bytes())?;
    }
    db.flush()?;
    println!("SSTables on disk: {}", db.table_count());

    let before = db.block_reads();
    let found = db.get(b"user:050000")?.is_some();
    println!(
        "get existing key: found = {found}, blocks read = {}",
        db.block_reads() - before
    );

    let before = db.block_reads();
    let found = db.get(b"user:050000x")?.is_some();
    println!(
        "get missing key:  found = {found}, blocks read = {}",
        db.block_reads() - before
    );

    std::fs::remove_dir_all(dir)?;
    Ok(())
}
```

**What happens:**

1. `i * 7919 % KEYS` visits every number from 0 to 99,999 exactly once, in a scrambled order, because 7919 shares no
   factor with 100,000. Every table therefore holds keys from the whole key range.
2. `user:050000x` does not exist, but it sorts between `user:050000` and `user:050001`, so it lies inside the key range of
   every table.
3. `block_reads` before and after a `get` shows how many blocks that one lookup read.

`cargo run --release --bin lsm_demo`:

```text
SSTables on disk: 26
get existing key: found = true, blocks read = 14
get missing key:  found = false, blocks read = 26
```

Writing was cheap: 26 large sequential writes. But reading `user:050000` had to read a block from 14 tables before the one
that held it, and the missing key cost a block read in **every one** of the 26 tables. This is **read amplification**, and
it gets worse with every flush. The two cures are the topic of Lesson 17.8: **compaction** merges many tables into few, and
**Bloom filters** let a lookup skip tables that surely do not contain the key, without reading them.

> ⚡ **Performance:** in an LSM tree, missing keys are the most expensive lookups, because the answer "not found" is only
> certain after the last table. Many workloads ask for missing keys all the time ("does this user exist yet?").

## 6. Durability, and B-tree versus LSM tree

Our memtable lives only in RAM: a crash loses everything written since the last flush. Real LSM trees put the WAL of
Lesson 17.4 in front of it: each write is appended to the WAL and synced, then put into the memtable. After a flush, the
data is safe in an SSTable, so the WAL up to that point can be deleted; on restart, the WAL is replayed into a fresh
memtable. If you choose the LSM design (Design B) for the project of Lesson 17.14, you connect the two there.

How does this compare with a B-tree?

| | B-tree (Lesson 17.6) | LSM tree |
|---|---|---|
| Write pattern | random, a page per insert, in place | sequential: WAL + large sorted files |
| Read of one key | one path of pages, about 3–4 reads | memtable + possibly every table (until compaction and Bloom filters help) |
| Range query | walk the leaves | merge the ranges of all tables |
| Write amplification | a whole page for a small change | data is rewritten again by each compaction |
| Typical users | PostgreSQL, SQLite, MySQL InnoDB | RocksDB, LevelDB, Cassandra |

Neither is "better". B-trees favour reads; LSM trees favour writes, especially many small random writes.

## Common mistakes

1. **Searching the tables oldest first.** The first answer must come from the newest data; otherwise an old value shadows
   a newer one.
2. **Treating "deleted" like "not found".** If a table says "deleted", the search must stop. Asking older tables would bring
   the old value back. That is why `SsTable::get` returns `Option<Option<…>>`.
3. **Flushing unsorted data.** The sparse index only works if every block is sorted and blocks follow each other in key
   order. A `BTreeMap` memtable guarantees it.
4. **Opening a half-written table after a crash.** Write tables under a temporary name and rename them, so that a table file
   either is complete or does not exist.

## Try it yourself

1. In `lsm_demo.rs`, change the memtable limit to 1 MiB (mebibyte, 1,048,576 bytes) and then to 64 KiB (kibibytes of 1,024 bytes). How do the number of tables and the blocks read
   for the missing key change?
2. Write the keys in sorted order instead (`let n = i;`). How many blocks does the missing key cost now, and why?
3. After running the demo without the last `remove_dir_all`, list the directory with `ls -l lsm-demo`. Are all tables the
   same size?

## Exercises

### Exercise 1 — A tombstone after a restart (easy)

Write a test `a_tombstone_survives_flush_and_reopen`: put `kiwi = green`, flush, delete `kiwi`, flush, close, reopen, and
check that `kiwi` is not found, although the older table still contains its value.

<details>
<summary>Hint</summary>

Use a big memtable limit (`1_000_000`) so that only your `flush` calls create tables.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn a_tombstone_survives_flush_and_reopen() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 1_000_000)?;
        db.put(b"kiwi", b"green")?;
        db.flush()?; // old table has the value
        db.delete(b"kiwi")?;
        db.flush()?; // new table has the tombstone
        drop(db);

        let db = Lsm::open(dir.path(), 1_000_000)?;
        assert_eq!(db.get(b"kiwi")?, None);
        Ok(())
    }
```

Why it works: after the reopen, `open` sorts the tables newest first by their number, so the table with the tombstone is
asked first and answers "deleted". The test would fail if `open` sorted the tables the wrong way round.

</details>

### Exercise 2 — Read everything (medium)

Add `pub fn scan_all(&self) -> io::Result<BTreeMap<Vec<u8>, Vec<u8>>>` to `Lsm`: all live keys with their newest values,
merged from all tables and the memtable, without deleted keys. Test it with: put `a = 1`, `b = 1`, flush, put `b = 2`, delete
`a`, flush, put `c = 3`. The result must be `{b: 2, c: 3}`.

<details>
<summary>Hint</summary>

Insert everything into one `BTreeMap<Vec<u8>, Value>`, starting with the **oldest** table, so that newer data overwrites
older data; the memtable comes last. At the end, keep only the entries whose value is `Some`.

</details>

<details>
<summary>Answer</summary>

In `impl Lsm`:

```rust
    /// Exercise 2: every live key and value, merged from all tables and the memtable.
    pub fn scan_all(&self) -> io::Result<BTreeMap<Vec<u8>, Vec<u8>>> {
        let mut merged: BTreeMap<Vec<u8>, Value> = BTreeMap::new();
        for table in self.tables.iter().rev() {
            // oldest first, so that newer tables overwrite older ones
            for (key, value) in table.scan()? {
                merged.insert(key, value);
            }
        }
        for (key, value) in &self.memtable {
            merged.insert(key.clone(), value.clone());
        }
        Ok(merged
            .into_iter()
            .filter_map(|(key, value)| value.map(|v| (key, v)))
            .collect())
    }
```

The test:

```rust
    #[test]
    fn scan_all_merges_everything() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let mut db = Lsm::open(dir.path(), 1_000_000)?;
        db.put(b"a", b"1")?;
        db.put(b"b", b"1")?;
        db.flush()?;
        db.put(b"b", b"2")?;
        db.delete(b"a")?;
        db.flush()?;
        db.put(b"c", b"3")?; // memtable only

        let all = db.scan_all()?;
        let expected = BTreeMap::from([
            (b"b".to_vec(), b"2".to_vec()),
            (b"c".to_vec(), b"3".to_vec()),
        ]);
        assert_eq!(all, expected);
        Ok(())
    }
```

`cargo test --lib lsm` (with Exercise 1):

```text
running 4 tests
test lsm::tests::a_tombstone_survives_flush_and_reopen ... ok
test lsm::tests::newest_value_wins_across_memtable_and_tables ... ok
test lsm::tests::scan_all_merges_everything ... ok
test lsm::tests::flushed_data_survives_a_reopen ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 28 filtered out; finished in 0.12s
```

Why it works: inserting oldest first and newest last is shadowing written as a loop. A tombstone (`None`) overwrites an older
value and is then dropped by `filter_map`. This merge reads all data into memory; Lesson 17.8 merges sorted tables without
doing that.

</details>

### Exercise 3 — Skip tables by their last key (hard)

`SsTable::get` already returns without reading when the key is smaller than the table's first key. Make it also return
without reading when the key is **larger than the table's last key**. Find the last key once in `open` (store it in a new
field `last_key`). Test with a table of `b` and `d`: `get(b"z")` and `get(b"a")` read no block, `get(b"c")` reads one.

<details>
<summary>Hint</summary>

The last key is the last record of the last block. In `open`, read the block from the last index offset to `data_end` and
decode its records; the key of the last one is the last key.

</details>

<details>
<summary>Answer</summary>

Add the field `last_key: Vec<u8>,` after `data_end` in `struct SsTable`. In `open`, replace the final `Ok(SsTable { … })`
with:

```rust
        // Exercise 3: find the last key by decoding the last block once.
        let mut last_key = Vec::new();
        if let Some((_, start)) = index.last() {
            let mut block = vec![0u8; (data_end - start) as usize];
            file.read_exact_at(&mut block, *start)?;
            let mut pos = 0;
            while pos < block.len() {
                let (rec, used) =
                    record::decode(&block[pos..]).map_err(|_| invalid("bad record"))?;
                last_key = rec.key;
                pos += used;
            }
        }
        Ok(SsTable {
            file,
            index,
            data_end,
            last_key,
            block_reads: Cell::new(0),
        })
```

In `get`, change the first check to:

```rust
        if n == 0 || key > self.last_key.as_slice() {
            return Ok(None); // outside the key range of the table: no read at all
        }
```

The test:

```rust
    #[test]
    fn keys_after_the_last_key_need_no_read() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("000001.sst");
        let entries = BTreeMap::from([
            (b"b".to_vec(), Some(b"1".to_vec())),
            (b"d".to_vec(), Some(b"2".to_vec())),
        ]);
        SsTable::write(&path, entries.iter())?;
        let table = SsTable::open(&path)?;
        assert_eq!(table.get(b"z")?, None);
        assert_eq!(table.get(b"a")?, None);
        assert_eq!(table.block_reads.get(), 0);
        assert_eq!(table.get(b"c")?, None); // inside the range: one block read
        assert_eq!(table.block_reads.get(), 1);
        Ok(())
    }
```

`cargo test --lib sstable` shows `2 passed`. Why it works: a sorted table knows its key range exactly, so keys outside
`[first, last]` cannot be in it. Run `lsm_demo` again: it still prints 26 blocks for the missing key, because
`user:050000x` lies *inside* every table's range. Key ranges help when tables cover different ranges (for example after
compaction into levels); for keys inside the range you need the Bloom filters of Lesson 17.8.

</details>

## Quiz

1. Where does a new write go first in an LSM tree, and when does it reach an SSTable?

<details>
<summary>Answer</summary>

Into the WAL (for durability) and the memtable. It reaches an SSTable when the memtable is full and is flushed.

</details>

2. Why does an LSM tree write only sequentially, even when the keys arrive in random order?

<details>
<summary>Answer</summary>

The memtable sorts the keys in memory. Disk writes are only appends to the WAL and whole SSTables written in one go; nothing
on disk is changed in place.

</details>

3. A key has a value in table 1 (old) and a tombstone in table 3 (new). What does `get` return, and why must it stop at
   table 3?

<details>
<summary>Answer</summary>

"Not found". The tombstone in the newer table shadows the older value; if `get` continued to table 1, the deleted value would
come back.

</details>

4. What is a sparse index, and what does it cost per lookup?

<details>
<summary>Answer</summary>

An index with one entry per block (here every 16 records) instead of per record. It is 16 times smaller, and a lookup must
read and scan one whole block instead of exactly one record.

</details>

5. Why is a lookup of a missing key the most expensive kind in an LSM tree?

<details>
<summary>Answer</summary>

"Not found" is only certain after the memtable and every table were checked, so the lookup may read a block in every table.

</details>

6. Name one advantage of a B-tree over an LSM tree and one the other way round.

<details>
<summary>Answer</summary>

A B-tree reads a key along one path of pages, without asking many files. An LSM tree writes only sequentially and handles many
small random writes much faster.

</details>

## Summary

- An LSM tree collects writes in a sorted memtable (behind a WAL) and flushes it as an immutable, sorted SSTable.
- Reads ask the memtable, then the SSTables from newest to oldest; the first answer wins (shadowing).
- Deletes are tombstones; "deleted" must stop the search, so a table's lookup has three answers.
- Our SSTable is records in key order, a sparse index of every 16th key, and an 8-byte footer; a lookup reads one block.
- Tables are written under a temporary name and renamed, so a crash never leaves half a table.
- Read amplification grows with every flush: 26 tables meant 26 block reads for one missing key.
- B-trees favour reads, LSM trees favour writes; compaction and Bloom filters (Lesson 17.8) reduce the LSM's read cost.

## Sources and further reading

- [The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf) — the original 1996 paper by
  O'Neil and others that introduced the idea.
- [LevelDB implementation notes](https://github.com/google/leveldb/blob/main/doc/impl.md) — memtables, the log, SSTable
  files and compaction in a real LSM tree, in a few pages.
- [RocksDB Overview](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview) — the architecture of the most widely used LSM
  engine.
- [Mini-LSM](https://skyzh.github.io/mini-lsm/) — a free course that builds a complete LSM storage engine in Rust, step by
  step: the natural next step after this phase.
- [Rust slice methods](https://doc.rust-lang.org/std/primitive.slice.html) — `partition_point` and `binary_search`, used by
  the SSTable lookup.

---

[🏠 Start](../START-HERE.md) · [← 17.6 B-trees](06-b-trees.md) · [📋 Phase 17](README.md) · [17.8 Compaction and Bloom filters →](08-compaction-and-bloom-filters.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
