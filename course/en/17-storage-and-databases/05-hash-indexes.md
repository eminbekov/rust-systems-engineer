<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.4 The write-ahead log (WAL)](04-write-ahead-log.md) · [📋 Phase 17](README.md) · [17.6 B-trees →](06-b-trees.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/05-hash-indexes.md) <!-- nav -->

# Lesson 17.5 — Indexes: a hash index

> **Phase 17 — Storage and database internals** · Lesson 5 of 15 · ⏱ about 135 minutes

## What you will learn

- What an index is, and why a database without one must read everything to find anything.
- How a log-structured store with an in-memory hash index works (the design of Bitcask): values on disk, only keys and
  positions in RAM (Random Access Memory).
- How a lookup becomes one positioned read, and how deletes become *tombstones*.
- How to measure the difference between a lookup with and without an index.
- The limits of a hash index: all keys in RAM, no range queries, a slow start-up and a file that keeps growing.

## Before you start

- You finished [Lesson 17.4 — The write-ahead log (WAL)](04-write-ahead-log.md). This lesson reuses its entry format
  (record + type byte) and its recovery.
- You know hash tables from [Lesson 07.6 — How hash tables work](../07-algorithms-and-data-structures/06-hash-tables.md),
  positioned reads (`read_exact_at`) from [Lesson 17.1](01-how-disks-and-files-work.md), and SQL (Structured Query Language) indexes from
  [Lesson 16.5 — SQL and PostgreSQL basics](../16-backend-services/05-sql-and-postgresql-basics.md).

## New words

| Word | Meaning |
|---|---|
| **index** | An extra data structure that tells you *where* data is, so that you do not have to search all of it. Like the index at the back of a book. |
| **full scan** | Finding data by reading all of it from start to end. |
| **log-structured storage** | A design where the data file itself is an append-only log: nothing is ever changed in place. |
| **hash index** | An index kept in a hash table: key → position of the latest record in the file. |
| **offset** | The position of a byte in a file, counted from 0 (you know it from Lesson 13.4). |
| **tombstone** | A special record that says "this key was deleted". In a log you cannot remove old records, so you add a tombstone instead. |
| **stale data** | Records in the file that are no longer the latest version of their key (overwritten or deleted). They use space but are never read. |
| **range query** | A question about a range of keys, for example "all users from `user:000100` to `user:000199`". |

## 1. Why an index?

The store of Lesson 17.4 kept every value in a `HashMap` in RAM. That is fast, but RAM is small and expensive compared with
an SSD (solid-state drive). A real store must keep values on disk. Then a new question appears: given a key, where in the file is its value?

Without help, the only answer is a **full scan**: read the file from the start, decode every record, and remember the last
one with this key (the last one, because later records overwrite earlier ones). For a 25 MB (megabyte) file, that means reading
25 MB to find 100 bytes, on every lookup.

An **index** is the help. It is the index at the back of a book: "fsync … page 214". You look up the word in a small,
sorted list and jump straight to the page. A storage index maps each key to the **offset** of its record in the file.
In SQL you met indexes in Lesson 16.5 as `CREATE INDEX`; now you build one yourself.

The simplest index is a hash table. This design is called **Bitcask** (after the storage engine of the Riak database,
which made it known):

```text
 RAM: hash index                     disk: data.log (append-only)
 ┌──────────────┬─────────────────┐  offset
 │ "apple"      │ offset 43, 24 B │     0  [ put apple = red    ]  ← stale
 │ "kiwi"       │ offset 21, 22 B │    21  [ put kiwi  = green  ]
 └──────────────┴─────────────────┘    43  [ put apple = yellow ]
```

- **Writes** append a record to the end of the file (fast, sequential) and update the index.
- **Reads** look up the key in the index and do **one positioned read** of exactly the right bytes.
- **Values stay on disk.** RAM holds only the keys and two numbers per key.

## 2. A log with a hash index

Create `src/hashlog.rs` and add `pub mod hashlog;` to `src/lib.rs`:

```rust
use crate::record;
use std::collections::HashMap;
use std::fs::{File, OpenOptions};
use std::io::{self, Write};
use std::os::unix::fs::FileExt;
use std::path::Path;

const PUT: u8 = 1;
const DELETE: u8 = 0;

/// Where a record lives in the data file.
#[derive(Debug, Clone, Copy)]
struct Pos {
    offset: u64,
    len: u32,
}

/// A log-structured key-value store with an in-memory hash index:
/// values live only in the file, the index maps each key to its latest record.
pub struct HashLog {
    file: File,
    index: HashMap<Vec<u8>, Pos>,
    end: u64,
}

impl HashLog {
    pub fn open(path: &Path) -> io::Result<HashLog> {
        let file = OpenOptions::new()
            .create(true)
            .read(true)
            .append(true)
            .open(path)?;
        let bytes = std::fs::read(path)?;

        let mut index = HashMap::new();
        let mut pos = 0;
        while let Ok((rec, used)) = record::decode(&bytes[pos..]) {
            let here = Pos {
                offset: pos as u64,
                len: used as u32,
            };
            match rec.value.first() {
                Some(&PUT) => {
                    index.insert(rec.key, here);
                }
                Some(&DELETE) => {
                    index.remove(&rec.key);
                }
                _ => {}
            }
            pos += used;
        }
        if pos < bytes.len() {
            file.set_len(pos as u64)?; // torn tail, exactly as in the WAL
            file.sync_all()?;
        }
        Ok(HashLog {
            file,
            index,
            end: pos as u64,
        })
    }
```

**What happens:**

1. `Pos` is what the index stores per key: where the record starts and how long it is. With the length, a read needs no
   guessing.
2. `open` scans the file once, like the replay of Lesson 17.4. But instead of keeping values, it keeps only the position
   of each key's latest record.
3. `while let Ok(...) = ...` runs as long as decoding succeeds and stops at the first error: the end of the file or a torn
   tail. It is a shorter way to write the `match` of Lesson 17.4.
4. A delete record removes the key from the index. The torn tail is cut off exactly as in the WAL.
5. `end` remembers where the file ends, so that `put` knows the offset of the record it appends.

Then the methods, still inside `impl HashLog`:

```rust
    /// Number of live keys.
    pub fn len(&self) -> usize {
        self.index.len()
    }

    pub fn is_empty(&self) -> bool {
        self.index.is_empty()
    }

    pub fn get(&self, key: &[u8]) -> io::Result<Option<Vec<u8>>> {
        let Some(pos) = self.index.get(key) else {
            return Ok(None); // not in the index: no disk access at all
        };
        let mut buf = vec![0u8; pos.len as usize];
        self.file.read_exact_at(&mut buf, pos.offset)?; // one positioned read
        let (rec, _) =
            record::decode(&buf).map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
        Ok(Some(rec.value[1..].to_vec()))
    }

    pub fn put(&mut self, key: &[u8], value: &[u8]) -> io::Result<()> {
        let mut entry = vec![PUT];
        entry.extend_from_slice(value);
        let pos = self.append(key, &entry)?;
        self.index.insert(key.to_vec(), pos);
        Ok(())
    }

    pub fn delete(&mut self, key: &[u8]) -> io::Result<()> {
        if self.index.contains_key(key) {
            self.append(key, &[DELETE])?; // a tombstone
            self.index.remove(key);
        }
        Ok(())
    }

    fn append(&mut self, key: &[u8], entry: &[u8]) -> io::Result<Pos> {
        let mut buf = Vec::new();
        record::encode(key, entry, &mut buf);
        self.file.write_all(&buf)?;
        self.file.sync_data()?;
        let pos = Pos {
            offset: self.end,
            len: buf.len() as u32,
        };
        self.end += buf.len() as u64;
        Ok(pos)
    }
}
```

**What happens:**

1. `get` uses `let … else` (Lesson 04.5): if the key is not in the index, the answer is "not found" without touching the
   disk. Otherwise `read_exact_at` reads exactly `pos.len` bytes at `pos.offset`, one system call.
2. The record is decoded again, so the checksum is checked on every read: a damaged byte on disk becomes an error, not a
   wrong value. `map_err` turns our `DecodeError` into an `io::Error` of kind `InvalidData`.
3. `rec.value[1..]` skips the type byte.
4. `put` follows the write-ahead rule: append and sync first, then update the index.
5. `delete` cannot erase the old record, because the file is append-only. It appends a **tombstone**, a record that says
   "deleted". Without it, the next `open` would find the old put and bring the key back to life. Deleting a key that does
   not exist writes nothing.
6. Clippy asks for `is_empty` whenever a type has a public `len`; that is a Rust convention.

> 📏 **Convention:** a type with `pub fn len(&self)` also gets `pub fn is_empty(&self)`. Clippy's lint `len_without_is_empty`
> reminds you.

## 3. Testing it

Add the tests at the end of `src/hashlog.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn get_reads_the_latest_value_from_disk() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("data.log");
        let mut db = HashLog::open(&path)?;
        db.put(b"apple", b"red")?;
        db.put(b"kiwi", b"green")?;
        db.put(b"apple", b"yellow")?;
        assert_eq!(db.get(b"apple")?, Some(b"yellow".to_vec()));
        assert_eq!(db.get(b"plum")?, None);
        assert_eq!(db.len(), 2);
        Ok(())
    }

    #[test]
    fn the_index_is_rebuilt_on_open() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("data.log");
        let mut db = HashLog::open(&path)?;
        db.put(b"apple", b"red")?;
        db.put(b"kiwi", b"green")?;
        db.delete(b"kiwi")?;
        db.put(b"apple", b"yellow")?;
        drop(db);

        let db = HashLog::open(&path)?;
        assert_eq!(db.get(b"apple")?, Some(b"yellow".to_vec()));
        assert_eq!(db.get(b"kiwi")?, None);
        assert_eq!(db.len(), 1);
        Ok(())
    }
}
```

`cargo test --lib hashlog`:

```text
running 2 tests
test hashlog::tests::get_reads_the_latest_value_from_disk ... ok
test hashlog::tests::the_index_is_rebuilt_on_open ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.03s
```

The second test is the important one: after a restart, `kiwi` stays deleted only because of its tombstone.

## 4. Measuring the index

How much does the index help? Put this into `src/bin/hash_index.rs`. It builds a file with 200,000 records, then compares
a lookup through the index with a full scan:

```rust
use p17_storage::{hashlog::HashLog, record};
use std::fs::{self, File};
use std::io::{BufWriter, Write};
use std::path::Path;
use std::time::Instant;

const KEYS: usize = 200_000;

fn key(i: usize) -> String {
    format!("user:{i:06}")
}

/// Finds a key the slow way: decode every record of the file and keep the last match.
fn scan_for(bytes: &[u8], wanted: &[u8]) -> Option<Vec<u8>> {
    let mut pos = 0;
    let mut found = None;
    while let Ok((rec, used)) = record::decode(&bytes[pos..]) {
        if rec.key == wanted {
            found = Some(rec.value[1..].to_vec());
        }
        pos += used;
    }
    found
}

fn main() -> std::io::Result<()> {
    let path = Path::new("bench.log");

    // 1. Build a data file quickly: one BufWriter, no sync per record.
    let mut out = BufWriter::new(File::create(path)?);
    let mut buf = Vec::new();
    for i in 0..KEYS {
        buf.clear();
        let mut entry = vec![1u8]; // put
        entry.extend_from_slice(format!("profile of user {i}, {}", "x".repeat(80)).as_bytes());
        record::encode(key(i).as_bytes(), &entry, &mut buf);
        out.write_all(&buf)?;
    }
    out.into_inner()?.sync_all()?;
    println!("data file: {} MB", fs::metadata(path)?.len() / 1_000_000);

    // 2. Open = scan the whole file once and build the index.
    let start = Instant::now();
    let db = HashLog::open(path)?;
    println!(
        "open (rebuild index of {} keys): {:?}",
        db.len(),
        start.elapsed()
    );

    // 3. Look up 10,000 keys through the index.
    let start = Instant::now();
    for i in (0..KEYS).step_by(KEYS / 10_000) {
        assert!(db.get(key(i).as_bytes())?.is_some());
    }
    println!("get with index:    {:?} per key", start.elapsed() / 10_000);

    // 4. One lookup without an index: read and decode the whole file.
    let start = Instant::now();
    let bytes = fs::read(path)?;
    assert!(scan_for(&bytes, key(KEYS - 1).as_bytes()).is_some());
    println!("get without index: {:?} per key", start.elapsed());

    fs::remove_file(path)?;
    Ok(())
}
```

**What happens:**

1. To build the test file fast, the program writes records with a `BufWriter` and one sync at the end, instead of 200,000
   calls of `put` with 200,000 syncs (that would take about 17 minutes at 5 ms each).
2. `{i:06}` pads the number with zeros to six digits: `user:000042`.
3. `step_by(KEYS / 10_000)` takes every 20th key, 10,000 lookups in total.
4. `scan_for` is the store without an index: it must decode every record, because the latest version of a key could be
   anywhere.

`cargo run --release --bin hash_index`:

```text
data file: 25 MB
open (rebuild index of 200000 keys): 23.47975ms
get with index:    731ns per key
get without index: 9.314375ms per key
```

With the index, a lookup takes less than a microsecond: one hash lookup and one `pread` of about 125 bytes from the page
cache. Without it, 9 ms: more than ten thousand times slower, and it grows with the file. The price of the index is the
23 ms at start-up, when `open` scans the file once.

> ⚡ **Performance:** the file was still in the page cache (it was just written), so `get` did not wait for the SSD. With a
> cold cache, each `get` would cost one SSD read (tens of microseconds, Lesson 17.1); still one read, never a scan.

## 5. The limits of a hash index

Bitcask is simple and very fast, and real systems use it. But look at what it cannot do:

1. **All keys must fit in RAM.** The index holds every key plus about 16 bytes of `Pos` and the hash table's own overhead.
   With 100 million keys of 20 bytes, that is several gigabytes of RAM.
2. **No range queries.** A hash table has no order (Lesson 07.6). "All keys from `user:000100` to `user:000199`" means
   checking every key in the index.
3. **Slow start-up.** `open` must scan the whole file. For 25 MB that took 23 ms; for 100 GB (gigabytes) it would take minutes.
   Bitcask solves this with *hint files*: a small file with just the index, written when a data file is closed.
4. **The file only grows.** Every overwrite and every delete leaves **stale data** behind. A key updated a million times
   uses a million records' worth of space. The answer is compaction, as in Exercise 3 of Lesson 17.4; Lesson 17.8 does it
   for many files at once.

The next two lessons remove limits 1 and 2 in two different ways. A **B-tree** (Lesson 17.6) keeps keys sorted in pages on
disk, so the index itself lives on disk and supports ranges. An **LSM tree** (Log-Structured Merge tree, Lesson 17.7) keeps the append-only idea but
writes **sorted** files.

## Common mistakes

1. **Forgetting the tombstone.** Removing a key only from the index works until the next restart; then the old record
   brings the key back.
2. **Starting `end` at 0 after reopening.** New records are appended at the real end of the file, but the index would
   point to offset 0. Exercise 2 tests exactly this.
3. **Trusting a read without the checksum.** A positioned read can return damaged bytes; decode and check them every time.
4. **Benchmarking only a warm cache.** The 731 ns are the page cache; say so when you report such numbers.

## Try it yourself

1. Change `KEYS` to 1,000,000 in `hash_index.rs`. How do the open time and the scan time change? Does the indexed `get`
   change much?
2. In `hash_index.rs`, overwrite every key once more after building the file, then measure the file size. How much of it is
   stale?
3. Write a test that deletes a key, puts it again, reopens, and checks that the new value is there.

## Exercises

### Exercise 1 — How much RAM? (easy)

A store has 50 million keys, 24 bytes long on average. Each index entry needs the key, 16 bytes for `Pos`, and roughly
32 more bytes of overhead (the `Vec` header and the hash table's empty slots). Estimate the RAM for the index. Would it fit
on a server with 4 GB of RAM?

<details>
<summary>Hint</summary>

Bytes per entry × number of keys. 1 GB is about 10⁹ bytes.

</details>

<details>
<summary>Answer</summary>

24 + 16 + 32 = 72 bytes per key; 72 × 50,000,000 = 3.6 × 10⁹ bytes, about 3.6 GB. It would barely fit, leaving almost no
room for the page cache and the program. This is limit 1 of Section 5: a hash index needs RAM proportional to the number of
keys, whatever the size of the values.

</details>

### Exercise 2 — Writes after a reopen (medium)

Write a test `writes_after_reopen_land_at_the_end`: put `apple = red`, close, reopen, put `kiwi = green`, delete the
missing key `plum`. Check that both values can be read, and that the file is exactly 21 + 22 bytes long (no tombstone for
`plum`).

<details>
<summary>Hint</summary>

`std::fs::metadata(&path)?.len()` gives the file size. The apple record is 12 + 5 + 1 + 3 bytes.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn writes_after_reopen_land_at_the_end() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("data.log");
        let mut db = HashLog::open(&path)?;
        db.put(b"apple", b"red")?;
        drop(db);

        let mut db = HashLog::open(&path)?;
        db.put(b"kiwi", b"green")?;
        db.delete(b"plum")?; // not there: must write nothing
        assert_eq!(db.get(b"apple")?, Some(b"red".to_vec()));
        assert_eq!(db.get(b"kiwi")?, Some(b"green".to_vec()));
        assert_eq!(std::fs::metadata(&path)?.len(), 21 + 22);
        Ok(())
    }
```

Why it works: if `open` set `end` to 0 instead of the file length, the kiwi record would still be appended at offset 21
(append mode), but the index would say offset 0, and `get(b"kiwi")` would read the apple record. The test catches that bug,
and the size check catches a needless tombstone.

</details>

### Exercise 3 — Count the stale bytes (hard)

A store should compact itself when, say, half of its file is stale. Add a field `stale: u64` and a method
`pub fn stale_bytes(&self) -> u64` that returns how many bytes belong to overwritten or deleted records (count a tombstone
as stale too: it is not live data). Keep the count correct in `put`, `delete` **and** `open`. Test it with this sequence:
put `apple = red` (21 bytes), put `apple = green` (23 bytes), put `kiwi = x` (18 bytes), delete `kiwi` (a 17-byte
tombstone); the result must be 21 + 18 + 17, before and after a reopen.

<details>
<summary>Hint</summary>

`HashMap::insert` returns the old value, `Some(old)`, if the key was there; `remove` does too. Add `old.len` to `stale` in
those cases. In `delete`, copy the old `Pos` first with `self.index.get(key).copied()`.

</details>

<details>
<summary>Answer</summary>

The changed parts of `src/hashlog.rs`:

```rust
pub struct HashLog {
    file: File,
    index: HashMap<Vec<u8>, Pos>,
    end: u64,
    stale: u64,
}
```

In `open`, the loop and the result:

```rust
        let mut index: HashMap<Vec<u8>, Pos> = HashMap::new();
        let mut stale = 0;
        let mut pos = 0;
        while let Ok((rec, used)) = record::decode(&bytes[pos..]) {
            let here = Pos {
                offset: pos as u64,
                len: used as u32,
            };
            match rec.value.first() {
                Some(&PUT) => {
                    if let Some(old) = index.insert(rec.key, here) {
                        stale += old.len as u64;
                    }
                }
                Some(&DELETE) => {
                    if let Some(old) = index.remove(&rec.key) {
                        stale += old.len as u64;
                    }
                    stale += here.len as u64; // the tombstone itself is not live data
                }
                _ => {}
            }
            pos += used;
        }
```

```rust
        Ok(HashLog {
            file,
            index,
            end: pos as u64,
            stale,
        })
```

The new method and the changed `put` and `delete`:

```rust
    /// Bytes in the file that belong to overwritten or deleted records.
    pub fn stale_bytes(&self) -> u64 {
        self.stale
    }
```

```rust
    pub fn put(&mut self, key: &[u8], value: &[u8]) -> io::Result<()> {
        let mut entry = vec![PUT];
        entry.extend_from_slice(value);
        let pos = self.append(key, &entry)?;
        if let Some(old) = self.index.insert(key.to_vec(), pos) {
            self.stale += old.len as u64;
        }
        Ok(())
    }

    pub fn delete(&mut self, key: &[u8]) -> io::Result<()> {
        if let Some(old) = self.index.get(key).copied() {
            let tombstone = self.append(key, &[DELETE])?; // a tombstone
            self.index.remove(key);
            self.stale += old.len as u64 + tombstone.len as u64;
        }
        Ok(())
    }
```

The test:

```rust
    #[test]
    fn stale_bytes_count_overwrites_and_deletes() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("data.log");
        let mut db = HashLog::open(&path)?;
        db.put(b"apple", b"red")?; // 21 bytes, live
        db.put(b"apple", b"green")?; // 23 bytes, live; the 21 become stale
        db.put(b"kiwi", b"x")?; // 18 bytes
        db.delete(b"kiwi")?; // 17-byte tombstone; 18 + 17 stale
        assert_eq!(db.stale_bytes(), 21 + 18 + 17);
        drop(db);

        let db = HashLog::open(&path)?;
        assert_eq!(db.stale_bytes(), 21 + 18 + 17);
        Ok(())
    }
```

`cargo test --lib hashlog` (with Exercise 2):

```text
running 4 tests
test hashlog::tests::writes_after_reopen_land_at_the_end ... ok
test hashlog::tests::get_reads_the_latest_value_from_disk ... ok
test hashlog::tests::the_index_is_rebuilt_on_open ... ok
test hashlog::tests::stale_bytes_count_overwrites_and_deletes ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.05s
```

Why it works: a record becomes stale exactly when the index stops pointing at it, and `insert` and `remove` tell you the
moment that happens. `open` repeats the same accounting while it scans, so the number survives a restart. With
`stale_bytes() * 2 > file length` a store could decide to compact.

</details>

## Quiz

1. What does a hash index map, and what stays out of RAM?

<details>
<summary>Answer</summary>

It maps each key to the position (offset and length) of the key's latest record in the data file. The values stay on disk.

</details>

2. How many disk reads does `get` need for an existing key, and how many for a missing key?

<details>
<summary>Answer</summary>

One positioned read for an existing key; zero for a missing key, because the index already says it is not there.

</details>

3. Why must `delete` write a tombstone instead of only removing the key from the index?

<details>
<summary>Answer</summary>

The old put record is still in the file. On the next `open`, the scan would find it and bring the key back. The tombstone,
being later in the file, tells the scan that the key was deleted.

</details>

4. Why can't a hash index answer "all keys between `a` and `c`" efficiently?

<details>
<summary>Answer</summary>

A hash table stores keys in an order that depends on their hashes, not on the keys themselves. To find a range you must
check every key.

</details>

5. The measured `get` took 731 ns. Why is that not the speed of the SSD?

<details>
<summary>Answer</summary>

The file had just been written and was still in the page cache, so `read_exact_at` copied from RAM. A read from the SSD
would take tens of microseconds.

</details>

6. Name the four limits of the Bitcask design.

<details>
<summary>Answer</summary>

All keys must fit in RAM; no range queries; start-up must scan the file (unless there are hint files); and the file grows
with stale data until it is compacted.

</details>

## Summary

- An index tells you where data is; without one, every lookup is a full scan.
- A Bitcask-style store appends records to a log and keeps a hash index key → (offset, length) in RAM.
- `get` is one hash lookup plus one positioned read; a missing key needs no disk access at all.
- Deletes are tombstones, because records in an append-only file are never removed.
- On open, the index is rebuilt by scanning the log, with the same torn-tail handling as the WAL.
- Measured: under 1 µs per indexed lookup (from the page cache) versus 9 ms for a full scan of 25 MB.
- Limits: keys in RAM, no ranges, slow start-up, stale data. B-trees and LSM trees address them.

## Sources and further reading

- [Bitcask: A Log-Structured Hash Table for Fast Key/Value Data](https://riak.com/assets/bitcask-intro.pdf) — the short
  original paper of the design in this lesson, including hint files and merging.
- [std::collections::HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — `insert` and `remove`
  return the old value, which Exercise 3 uses.
- [std::os::unix::fs::FileExt](https://doc.rust-lang.org/std/os/unix/fs/trait.FileExt.html) — `read_exact_at`, the
  positioned read behind `get`.
- [PostgreSQL documentation — Index Types](https://www.postgresql.org/docs/current/indexes-types.html) — the hash and
  B-tree indexes of a real database, and when each one is used.
- [Clippy lints](https://rust-lang.github.io/rust-clippy/stable/index.html) — search for `len_without_is_empty`, the
  convention from Section 2.

---

[🏠 Start](../START-HERE.md) · [← 17.4 The write-ahead log (WAL)](04-write-ahead-log.md) · [📋 Phase 17](README.md) · [17.6 B-trees →](06-b-trees.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
