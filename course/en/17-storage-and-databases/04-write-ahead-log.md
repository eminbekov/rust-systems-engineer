<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.3 Binary file formats and serialization](03-binary-file-formats.md) · [📋 Phase 17](README.md) · [17.5 Indexes: a hash index →](05-hash-indexes.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/04-write-ahead-log.md) <!-- nav -->

# Lesson 17.4 — The write-ahead log (WAL)

> **Phase 17 — Storage and database internals** · Lesson 4 of 15 · ⏱ about 135 minutes

## What you will learn

- The one rule behind every crash-safe database: write the change to a log first, and only then apply it.
- How to build a small key-value store whose data survives restarts and crashes: a `HashMap` in memory plus a WAL on disk.
- How recovery works: replay the log, find the torn tail, and cut it off.
- Why forgetting to cut the torn tail silently loses *new* data.
- Why a log grows forever, and how compaction and checkpoints keep it small.

## Before you start

- You finished [Lesson 17.3 — Binary file formats and serialization](03-binary-file-formats.md): `src/record.rs` encodes
  and decodes records with lengths and a CRC (Cyclic Redundancy Check, a checksum).
- You know durability, `sync_data` and torn writes from [Lesson 17.2](02-durability-and-fsync.md), and the atomic-replace
  pattern from [Lesson 13.12](../13-operating-systems/12-file-systems-inside.md).
- You know `HashMap` ([Lesson 07.6 — How hash tables work](../07-algorithms-and-data-structures/06-hash-tables.md)) and
  transactions from [Lesson 16.7 — Migrations and transactions](../16-backend-services/07-migrations-and-transactions.md).
- Add the crate `tempfile` for tests. It creates a temporary directory that is deleted automatically:

```bash
cd ~/projects/rust/code/p17-storage
cargo add --dev tempfile@3
```

## New words

| Word | Meaning |
|---|---|
| **WAL** (Write-Ahead Log) | An append-only file where a storage engine writes every change *before* it applies the change anywhere else. |
| **log entry** | One change in the log, for example "put apple = red" or "delete kiwi". |
| **replay** | Reading the log from the start and applying every entry again, to rebuild the state after a restart. |
| **recovery** | Everything a storage engine does at start-up to get back to a correct state after a crash: here, replaying and cutting the torn tail. |
| **torn tail** | The incomplete or damaged bytes at the end of a log after a crash in the middle of an append. |
| **truncate** | Make a file shorter, throwing away everything after a given length. In Rust: `File::set_len`. |
| **compaction** | Rewriting a log or data file so that it keeps only the latest live data and drops old versions and deleted keys. |
| **checkpoint** | A moment when the engine saves its full state elsewhere, so that the log before that moment is no longer needed. |
| **dev-dependency** | A crate that is used only by tests, examples and benchmarks, not by the library itself (`cargo add --dev`). |

## 1. The idea: write ahead

Imagine a shopkeeper who keeps the stock count in their head. Every time something is sold, they first write a line into a
notebook ("sold 2 apples"), and only then change the number in their head. If they fall asleep and forget everything, they
can read the notebook from the first page and get the count back. The notebook only ever grows at the end, and writing a
line is quick.

A **WAL** (write-ahead log) is that notebook. The rule has three parts:

1. **Write the change to the log first**, at the end of the file (an append, which is fast, Lesson 17.1).
2. **Make the log entry durable** with `sync_data` (Lesson 17.2).
3. **Only then** apply the change to the real data (in memory, or in data files), and tell the caller "saved".

If the program crashes before step 2 finishes, the caller never heard "saved", so losing the change is allowed. If it
crashes after step 2, the change is in the log, and recovery will apply it again. In both cases the state after restart is
one that the callers could have seen. That is the whole trick.

```text
put(apple, red)
   │
   ├─► 1. append entry to store.wal ──► 2. sync_data ──► 3. map.insert ──► "ok"
   │
 crash here? ─ before 2: entry may be torn or missing  → caller never got "ok", fine
            ─ after 2:  entry is in the log            → replay restores it
```

PostgreSQL, SQLite (in WAL mode), MySQL's InnoDB, and the LSM (Log-Structured Merge) trees of Lesson 17.7 all work like this. In PostgreSQL, a
transaction from Lesson 16.7 is "committed" exactly at the moment its commit entry in the WAL is durable.

## 2. The log entry format

We reuse the record format of Lesson 17.3 (checksum, two lengths, key, value). To tell a put from a delete, the first byte
of the record's value is the **entry type**:

```text
put apple = red    → record { key: "apple", value: [1, 'r', 'e', 'd'] }   (1 = put)
delete kiwi        → record { key: "kiwi",  value: [0] }                  (0 = delete)
```

So the record layer still gives us lengths and a checksum for every entry, and the WAL layer only adds one byte.

## 3. The store

Create `src/wal.rs`. First the type and recovery:

```rust
use crate::record::{self, DecodeError};
use std::collections::HashMap;
use std::fs::{File, OpenOptions};
use std::io::{self, Write};
use std::path::Path;

const PUT: u8 = 1;
const DELETE: u8 = 0;

/// A key-value store that keeps everything in a HashMap
/// and makes every change durable in a write-ahead log first.
pub struct WalStore {
    map: HashMap<Vec<u8>, Vec<u8>>,
    log: File,
    /// How many log entries were replayed when the store was opened.
    pub replayed: usize,
    /// How many bytes of a torn tail were cut off when the store was opened.
    pub dropped_bytes: u64,
}

impl WalStore {
    /// Opens the log at `path` (creating it if needed) and replays it.
    pub fn open(path: &Path) -> io::Result<WalStore> {
        let log = OpenOptions::new()
            .create(true)
            .read(true)
            .append(true)
            .open(path)?;
        let bytes = std::fs::read(path)?;

        let mut map = HashMap::new();
        let mut pos = 0;
        let mut replayed = 0;
        while pos < bytes.len() {
            match record::decode(&bytes[pos..]) {
                Ok((rec, used)) => {
                    apply(&mut map, rec.key, &rec.value);
                    pos += used;
                    replayed += 1;
                }
                // A torn or damaged entry: everything from here on is not trusted.
                Err(
                    DecodeError::Truncated | DecodeError::BadChecksum | DecodeError::TooLarge(_),
                ) => {
                    break;
                }
            }
        }

        let dropped_bytes = (bytes.len() - pos) as u64;
        if dropped_bytes > 0 {
            log.set_len(pos as u64)?; // cut the torn tail off
            log.sync_all()?;
        }
        Ok(WalStore {
            map,
            log,
            replayed,
            dropped_bytes,
        })
    }
```

**What happens:**

1. `OpenOptions` opens the log for reading and **appending** and creates it if it does not exist. In append mode every
   `write` goes to the current end of the file, whatever happened before. A *newly created* log is a new name in its
   directory, so a fully crash-safe store also syncs the directory once after creating it (item 4 of the checklist in
   Lesson 17.2). We leave that out here to keep the code short; your project in Lesson 17.14 must do it.
2. `std::fs::read` loads the whole log into memory. That is fine for a small store; a big one would read it in pieces.
3. The loop is the **replay**: decode an entry, apply it to the map, move `pos` forward. Later entries overwrite earlier
   ones, so after the loop the map holds the latest state.
4. The first entry that does not decode stops the replay. All three errors mean "this and everything after it is not a
   complete, correct entry". After a crash this is the torn tail of Lesson 17.2.
5. `set_len(pos)` **truncates** the file: it throws away the torn bytes. `sync_all` makes the new length durable
   (`set_len` changes metadata, so we use `sync_all`, not `sync_data`). Section 4 shows why this step is essential.

Then reading and writing, still inside `impl WalStore`, and the helper `apply` after it:

```rust
    pub fn get(&self, key: &[u8]) -> Option<&[u8]> {
        self.map.get(key).map(|v| v.as_slice())
    }

    pub fn put(&mut self, key: &[u8], value: &[u8]) -> io::Result<()> {
        let mut entry = vec![PUT];
        entry.extend_from_slice(value);
        self.append(key, &entry)?; // 1. log first ...
        self.map.insert(key.to_vec(), value.to_vec()); // 2. ... then memory
        Ok(())
    }

    pub fn delete(&mut self, key: &[u8]) -> io::Result<()> {
        self.append(key, &[DELETE])?;
        self.map.remove(key);
        Ok(())
    }

    fn append(&mut self, key: &[u8], entry: &[u8]) -> io::Result<()> {
        let mut buf = Vec::new();
        record::encode(key, entry, &mut buf);
        self.log.write_all(&buf)?;
        self.log.sync_data() // durable before we return Ok
    }
}

/// Applies one log entry to the map. The first byte of the entry says what it is.
fn apply(map: &mut HashMap<Vec<u8>, Vec<u8>>, key: Vec<u8>, entry: &[u8]) {
    match entry.split_first() {
        Some((&PUT, value)) => {
            map.insert(key, value.to_vec());
        }
        Some((&DELETE, _)) => {
            map.remove(&key);
        }
        _ => {} // unknown entry type: ignore it
    }
}
```

**What happens:**

1. `get` never touches the disk: all data is in the map. `map(|v| v.as_slice())` turns `Option<&Vec<u8>>` into
   `Option<&[u8]>`.
2. `put` and `delete` follow the rule exactly: `append` (write + `sync_data`) first; if it fails, `?` returns the error and
   the map is not changed. Only after a durable append does the map change.
3. `append` encodes the entry into one buffer and writes it with one `write_all`, so the entry is written in one piece as
   far as possible.
4. `split_first` splits a slice into its first element and the rest. Matching on `Some((&PUT, value))` reads as "the first
   byte is `PUT`, the rest is the value". `PUT` and `DELETE` are constants, so they can be used as patterns.

Add `pub mod wal;` to `src/lib.rs`.

> ⚡ **Performance:** every `put` here pays one full `sync_data`, about 5 ms on a Mac (Lesson 17.2), so this store can do
> only about 200 writes per second. Real engines use group commit: one sync for all the entries that arrived meanwhile. The
> `SyncPolicy` of Lesson 17.2 is a first step in that direction.

## 4. Recovery, and why the tail must be cut off

Why `set_len`? Imagine we skip it. After a crash the file ends with 11 torn bytes. The replay stops there, correctly. Then
the next `put` appends a new, perfect entry **after** the torn bytes:

```text
[ entry 1 ][ entry 2 ][ torn 11 bytes ][ new entry 3 ]
                       ▲
                       next replay stops here, and entry 3 is never read
```

On the next restart, the replay again stops at the torn bytes, and entry 3, which the caller was told was saved, is lost.
Truncating the file at the end of the last good entry makes the log a clean prefix again, and new entries continue it.

This is a test worth writing first. Put the tests at the end of `src/wal.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn data_survives_a_restart() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("store.wal");

        let mut store = WalStore::open(&path)?;
        store.put(b"apple", b"red")?;
        store.put(b"kiwi", b"green")?;
        store.put(b"apple", b"yellow")?;
        store.delete(b"kiwi")?;
        drop(store); // "restart"

        let store = WalStore::open(&path)?;
        assert_eq!(store.replayed, 4);
        assert_eq!(store.get(b"apple"), Some(&b"yellow"[..]));
        assert_eq!(store.get(b"kiwi"), None);
        Ok(())
    }

    #[test]
    fn a_torn_tail_is_cut_off_and_new_writes_survive() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("store.wal");
        let mut store = WalStore::open(&path)?;
        store.put(b"apple", b"red")?;
        drop(store);

        // Simulate a crash in the middle of the next append: write half an entry.
        let mut half = Vec::new();
        record::encode(b"kiwi", &[PUT, b'g', b'r', b'e', b'e', b'n'], &mut half);
        let mut file = OpenOptions::new().append(true).open(&path)?;
        file.write_all(&half[..half.len() / 2])?;
        drop(file);

        let mut store = WalStore::open(&path)?;
        assert_eq!(store.replayed, 1);
        assert_eq!(store.dropped_bytes, (half.len() / 2) as u64);
        assert_eq!(store.get(b"kiwi"), None);

        store.put(b"plum", b"purple")?; // must not land behind the garbage
        drop(store);
        let store = WalStore::open(&path)?;
        assert_eq!(store.get(b"plum"), Some(&b"purple"[..]));
        assert_eq!(store.dropped_bytes, 0);
        Ok(())
    }
}
```

**What happens:**

1. `tempfile::tempdir()` creates a new empty directory; when `dir` is dropped at the end of the test, the directory and
   everything in it are deleted. Each test gets its own directory, so tests can run in parallel.
2. `drop(store)` closes the log; opening it again is a "restart" that must replay.
3. The second test makes the torn tail by hand: it encodes a real entry and appends only its first half, exactly what a
   crash in the middle of `write_all` can leave.
4. After recovery, `plum` is written and the store is opened once more. Only a correctly truncated log keeps `plum`.

`cargo test --lib wal`:

```text
running 2 tests
test wal::tests::data_survives_a_restart ... ok
test wal::tests::a_torn_tail_is_cut_off_and_new_writes_survive ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 13 filtered out; finished in 0.11s
```

(The 13 filtered-out tests are those of the earlier lessons and their exercises.) Now remove the line with `set_len` and
run the tests again. The second test fails exactly where the lost entry should be:

```text
thread 'wal::tests::a_torn_tail_is_cut_off_and_new_writes_survive' (6188123) panicked at src/wal.rs:147:9:
assertion `left == right` failed
  left: None
 right: Some([112, 117, 114, 112, 108, 101])
```

`[112, 117, 114, 112, 108, 101]` is `purple` as bytes. Put the line back.

> 🧪 **Testing:** a recovery test always has the same three steps: build a state, damage the file the way a crash would,
> reopen and check both the old data *and* that new writes still work. The last part is the one people forget.

Our recovery stops at the **first** bad entry, wherever it is. A bad entry in the middle of the log (a bit flip on the
device) would also drop all later, good entries. Real engines separate the two cases: a bad tail is normal after a crash,
but a bad entry followed by good ones is real corruption, which they report instead of silently dropping data.

## 5. Try it with a real crash

A small command-line program makes the store visible. Put this into `src/bin/wal_demo.rs`:

```rust
use p17_storage::{record, wal::WalStore};
use std::fs::OpenOptions;
use std::io::Write;
use std::path::Path;

const LOG: &str = "demo.wal";

/// Writes only the first half of a put entry and then dies, like a power cut during a write.
fn crash_put(key: &str, value: &str) -> std::io::Result<()> {
    let mut entry = vec![1u8]; // 1 = put
    entry.extend_from_slice(value.as_bytes());
    let mut buf = Vec::new();
    record::encode(key.as_bytes(), &entry, &mut buf);
    let mut file = OpenOptions::new().append(true).open(LOG)?;
    file.write_all(&buf[..buf.len() / 2])?;
    std::process::abort();
}

fn main() -> std::io::Result<()> {
    let args: Vec<String> = std::env::args().skip(1).collect();
    let args: Vec<&str> = args.iter().map(String::as_str).collect();

    let mut store = WalStore::open(Path::new(LOG))?;
    if store.dropped_bytes > 0 {
        println!(
            "recovery: cut off a torn tail of {} bytes",
            store.dropped_bytes
        );
    }

    match args.as_slice() {
        ["put", key, value] => {
            store.put(key.as_bytes(), value.as_bytes())?;
            println!("ok");
        }
        ["get", key] => match store.get(key.as_bytes()) {
            Some(value) => println!("{}", String::from_utf8_lossy(value)),
            None => println!("(not found)"),
        },
        ["delete", key] => {
            store.delete(key.as_bytes())?;
            println!("ok");
        }
        ["crash-put", key, value] => crash_put(key, value)?,
        _ => eprintln!("usage: wal_demo put K V | get K | delete K | crash-put K V"),
    }
    Ok(())
}
```

**What happens:**

1. Every run is a new process: it opens (and replays) `demo.wal`, does one command and exits. So every command after the
   first one tests recovery.
2. `args.as_slice()` lets `match` look at the whole list of arguments at once: `["put", key, value]` matches exactly three
   arguments with `put` first and binds the other two.
3. `crash-put` builds a real entry, appends only half of it and aborts, a power cut in miniature. Its return type says
   `io::Result<()>`, but it never returns: `abort` ends the process.

Run the commands one after another:

```bash
cargo run -q --bin wal_demo -- put apple red
cargo run -q --bin wal_demo -- put kiwi green
cargo run -q --bin wal_demo -- get apple
cargo run -q --bin wal_demo -- delete kiwi
cargo run -q --bin wal_demo -- get kiwi
cargo run -q --bin wal_demo -- crash-put plum purple
cargo run -q --bin wal_demo -- get plum
cargo run -q --bin wal_demo -- get apple
```

```text
ok
ok
red
ok
(not found)
recovery: cut off a torn tail of 11 bytes
(not found)
red
```

(Your shell also reports the abort of `crash-put`.) The log was 60 bytes after `delete kiwi`: 21 + 22 + 17 bytes for the
three entries. `crash-put` added 11 of its 23 bytes, and the next start cut them off. `plum` is not found, which is
correct: `crash-put` never said "ok". Delete `demo.wal` when you are done.

## 6. The log grows forever

Each `put` adds an entry, even when it overwrites a key. After a million updates of one counter, the log holds a million
entries, of which one matters. Replay at start-up gets slower and slower. There are two answers, and real systems use both:

- **Compaction**: rewrite the log so that it contains only the current state, one entry per live key. Exercise 3 does this
  with the atomic-replace pattern from Lesson 13.12.
- **Checkpoints**: the data also lives somewhere else (data pages in PostgreSQL, sorted files in an LSM tree). Every now
  and then the engine makes sure those are durable up to log position X; after that, the log before X is not needed and
  can be deleted or reused. PostgreSQL calls this a *checkpoint*.

In Lessons 17.5–17.8 the data moves out of the `HashMap` into files on disk, and the WAL becomes what it is in a real
database: a short-lived safety net between "the change arrived" and "the change is in the data files".

## Common mistakes

1. **Changing memory before the log.** If the map changes first and then the append fails, readers have seen data that is
   not durable and will disappear after a restart.
2. **Not truncating the torn tail.** New entries land behind the garbage and are lost on the next restart (Section 4).
3. **Treating every decode error as fatal.** A torn last entry is *normal* after a crash; a store that refuses to start
   then is not crash-safe. Treat the tail as "not written".
4. **Acknowledging before `sync_data` returns.** Then the WAL protects nothing; see Lesson 17.2.
5. **Testing recovery without a write after it.** Many bugs show up only when new data is appended to a recovered log.

## Try it yourself

1. Run the demo commands above, then `xxd demo.wal`, and find the entry type byte (`01` or `00`) of each entry.
2. Run `crash-put` twice in a row without anything between them. What does the next start report, and why only one torn
   tail?
3. Time 100 `put`s in a loop in a small test or program. Compare with the `sync_data` cost you measured in Lesson 17.2.

## Exercises

### Exercise 1 — Which order is right? (easy)

A colleague writes `put` like this: `self.map.insert(...)`, then `println!("ok")` to the client, then `self.append(...)`.
Describe one crash that makes the client see wrong behaviour, and give the correct order.

<details>
<summary>Hint</summary>

What does the client believe after it saw "ok", and what does the log contain if the program crashes right then?

</details>

<details>
<summary>Answer</summary>

The program prints "ok" and crashes before `append`. The client believes the value is saved, but the log has no entry, so
after the restart the value is gone. Even without a crash, another reader could see the new value in the map and then lose
it after a restart. The correct order: `append` (write + `sync_data`), then change the map, then answer "ok".

</details>

### Exercise 2 — A damaged last entry (medium)

Write a test `a_damaged_last_entry_is_dropped`: put `apple = red` and `kiwi = green`, close the store, flip all bits of the
**last byte** of the file, and reopen. Check that one entry was replayed, that 22 bytes were dropped, that `apple` is still
there and `kiwi` is not.

<details>
<summary>Hint</summary>

Read the file with `std::fs::read`, change `bytes[bytes.len() - 1] ^= 0xff`, and write it back with `std::fs::write`. The
kiwi entry is 12 + 4 + 6 = 22 bytes (header, key, type byte + value).

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn a_damaged_last_entry_is_dropped() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("store.wal");
        let mut store = WalStore::open(&path)?;
        store.put(b"apple", b"red")?;
        store.put(b"kiwi", b"green")?;
        drop(store);

        let mut bytes = std::fs::read(&path)?;
        let last = bytes.len() - 1;
        bytes[last] ^= 0xff; // damage the last byte of the kiwi entry
        std::fs::write(&path, &bytes)?;

        let store = WalStore::open(&path)?;
        assert_eq!(store.replayed, 1);
        assert_eq!(store.dropped_bytes, 22); // the whole kiwi entry
        assert_eq!(store.get(b"apple"), Some(&b"red"[..]));
        assert_eq!(store.get(b"kiwi"), None);
        Ok(())
    }
```

Why it works: all bytes of the kiwi entry are present, so `decode` does not report `Truncated`, but the CRC no longer
matches and it reports `BadChecksum`. The replay stops before the entry, and recovery cuts off all 22 bytes. A torn write
and a damaged write are handled the same way.

</details>

### Exercise 3 — Compaction (hard)

Add `pub fn compact(&mut self, path: &Path) -> io::Result<()>` to `WalStore` (pass the same path as to `open`). It writes
one put entry per live key into a temporary file next to the log, makes it durable, renames it over the log, syncs the
directory, and continues appending to the new file. Test: 50 puts to `counter`, a put and a delete of `gone`; after
`compact` and one more put, the file is smaller than before, a reopen replays exactly 2 entries, and all values are right.

<details>
<summary>Hint</summary>

Build all entries in one `Vec<u8>` with `record::encode`, like `append` does. Use `path.with_extension("compacting")`
for the temporary name. After `fs::rename`, open the directory (`path.parent()`) and call `sync_all` on it, then replace
`self.log` with the new file opened in append mode.

</details>

<details>
<summary>Answer</summary>

Inside `impl WalStore`:

```rust
    /// Rewrites the log at `path` (the same path as in `open`) so that it holds
    /// one put entry per live key, and nothing else.
    pub fn compact(&mut self, path: &Path) -> io::Result<()> {
        let tmp = path.with_extension("compacting");
        let mut buf = Vec::new();
        for (key, value) in &self.map {
            let mut entry = vec![PUT];
            entry.extend_from_slice(value);
            record::encode(key, &entry, &mut buf);
        }
        let mut file = File::create(&tmp)?;
        file.write_all(&buf)?;
        file.sync_all()?; // 1. the new log is durable
        drop(file);
        std::fs::rename(&tmp, path)?; // 2. atomically replace the old log
        let dir = path.parent().unwrap_or(Path::new("."));
        File::open(dir)?.sync_all()?; // 3. make the rename durable
        self.log = OpenOptions::new().append(true).open(path)?; // 4. append to the new file
        Ok(())
    }
```

In the tests:

```rust
    #[test]
    fn compact_shrinks_the_log_and_keeps_the_data() -> io::Result<()> {
        let dir = tempfile::tempdir()?;
        let path = dir.path().join("store.wal");
        let mut store = WalStore::open(&path)?;
        for i in 0..50 {
            store.put(b"counter", i.to_string().as_bytes())?;
        }
        store.put(b"gone", b"soon")?;
        store.delete(b"gone")?;
        let before = std::fs::metadata(&path)?.len();

        store.compact(&path)?;
        store.put(b"after", b"compaction")?;
        let after = std::fs::metadata(&path)?.len();
        drop(store);

        let store = WalStore::open(&path)?;
        assert!(after < before, "before {before}, after {after}");
        assert_eq!(store.replayed, 2);
        assert_eq!(store.get(b"counter"), Some(&b"49"[..]));
        assert_eq!(store.get(b"after"), Some(&b"compaction"[..]));
        assert_eq!(store.get(b"gone"), None);
        Ok(())
    }
```

`cargo test --lib wal` (with Exercise 2):

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 13 filtered out; finished in 0.27s
```

Why it works: a crash at any moment leaves either the complete old log (before the rename) or the complete new log (after
it); both replay to the same state. The new log has one entry per live key, so `gone` (deleted) and the 49 old counter
values disappear. Step 4 matters: the old `File` still points to the old, now nameless file, and appends to it would be
lost.

</details>

## Quiz

1. State the write-ahead rule in one sentence.

<details>
<summary>Answer</summary>

Make the change durable in the log before you apply it anywhere else or acknowledge it.

</details>

2. The program crashes after `sync_data` returned, but before `map.insert`. Is the change lost?

<details>
<summary>Answer</summary>

No. The entry is durable in the log, so the replay at the next start applies it to the map.

</details>

3. Why does replay stop at the first entry that does not decode, instead of skipping it?

<details>
<summary>Answer</summary>

After a damaged entry the reader no longer knows where the next entry begins (the lengths may be wrong), and a torn entry
is by definition the last thing that was written. Everything from there on is treated as "not written".

</details>

4. What goes wrong if recovery does not truncate the torn tail?

<details>
<summary>Answer</summary>

New entries are appended after the torn bytes. On the next restart the replay stops at the torn bytes again and never reads
the new entries, so acknowledged data is lost.

</details>

5. Why does `open` call `sync_all` after `set_len`, and not `sync_data`?

<details>
<summary>Answer</summary>

`set_len` changes the file size, which is metadata. `sync_all` makes data *and* metadata durable, so the shorter length is
guaranteed to survive another crash.

</details>

6. Name two ways real databases stop the WAL from growing forever.

<details>
<summary>Answer</summary>

Compaction (rewrite the log with only the live data) and checkpoints (make the data files durable up to a log position and
then drop or reuse the log before it).

</details>

## Summary

- The write-ahead rule: append the change to the log, make it durable, then apply it and acknowledge it.
- A tiny but crash-safe key-value store is a `HashMap` plus a WAL; `open` replays the log to rebuild the map.
- The log reuses the record format of Lesson 17.3, with one extra byte for the entry type (put or delete).
- Recovery stops at the first entry that does not decode and truncates the file there; without the truncation, new data is
  lost.
- Recovery tests build state, damage the file like a crash, reopen, and check both old data and new writes.
- The log grows with every change; compaction and checkpoints keep it small.

## Sources and further reading

- [PostgreSQL documentation — Write-Ahead Logging (WAL)](https://www.postgresql.org/docs/current/wal-intro.html) — the same
  rule in a real database, and how it makes commits durable.
- [SQLite — Write-Ahead Logging](https://www.sqlite.org/wal.html) — SQLite's WAL mode, checkpoints and their trade-offs.
- [LevelDB log format](https://github.com/google/leveldb/blob/main/doc/log_format.md) — the WAL of a real key-value store.
- [std::fs::OpenOptions](https://doc.rust-lang.org/std/fs/struct.OpenOptions.html) — `append`, `create` and the other
  options, and what append mode guarantees.
- [std::fs::File](https://doc.rust-lang.org/std/fs/struct.File.html) — `set_len`, `sync_all` and `sync_data`.
- [tempfile documentation](https://docs.rs/tempfile/latest/tempfile/) — temporary files and directories for tests.

---

[🏠 Start](../START-HERE.md) · [← 17.3 Binary file formats and serialization](03-binary-file-formats.md) · [📋 Phase 17](README.md) · [17.5 Indexes: a hash index →](05-hash-indexes.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
