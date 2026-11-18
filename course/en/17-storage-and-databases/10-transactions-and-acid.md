<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.9 The page cache and the buffer pool](09-page-cache-and-buffer-pool.md) · [📋 Phase 17](README.md) · [17.11 Isolation levels →](11-isolation-levels.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/10-transactions-and-acid.md) <!-- nav -->

# Lesson 17.10 — Transactions and ACID

> **Phase 17 — Storage and database internals** · Lesson 10 of 15 · ⏱ about 135 minutes

## What you will learn

- What each letter of ACID promises, and which part of a database keeps each promise.
- How atomic commit works: changes wait in a write set and reach the log as **one** checksummed record.
- How recovery throws away a transaction that was cut by a crash, and how to test a crash at every byte.
- What redo and undo information are, and why a buffer pool needs undo.
- Why ACID alone does not stop two clients from losing each other's updates.

## Before you start

- You used transactions in PostgreSQL (`BEGIN`, `COMMIT`, `ROLLBACK`) and saw the ACID table in
  [Lesson 16.7 — Migrations and transactions](../16-backend-services/07-migrations-and-transactions.md).
- You know `sync_data`, torn writes and group commit from [Lesson 17.2 — Durability: fsync and crash safety](02-durability-and-fsync.md),
  length-and-checksum records from [Lesson 17.3 — Binary file formats and serialization](03-binary-file-formats.md),
  and the WAL (write-ahead log) from [Lesson 17.4 — The write-ahead log (WAL)](04-write-ahead-log.md).
- You know the buffer pool and dirty pages from [Lesson 17.9 — The page cache and the buffer pool](09-page-cache-and-buffer-pool.md).

## New words

| Word | Meaning |
|---|---|
| **transaction** | A group of reads and writes that the database treats as one unit: all of its changes happen, or none. |
| **ACID** (Atomicity, Consistency, Isolation, Durability) | The four promises of a transaction: all-or-nothing, rules stay true, transactions do not disturb each other, committed data survives crashes. |
| **commit point** | The exact moment a transaction becomes permanent. Before it, a crash undoes the transaction; after it, a crash cannot. |
| **write set** | The changes a transaction has made but not yet committed, kept aside in memory. |
| **commit record** | The log entry whose durable presence means "this transaction committed". |
| **rollback** (abort) | Ending a transaction by throwing its changes away. |
| **invariant** | A rule about your data that must always be true, for example "the total amount of money never changes". |
| **redo information** | What is needed to *repeat* a committed change after a crash (the new value). |
| **undo information** | What is needed to *take back* an uncommitted change after a crash (the old value). |
| **CRC** (Cyclic Redundancy Check) | A short checksum computed from bytes. If any byte changes, the CRC almost certainly changes. Here we use CRC-32 from the crate `crc32fast`. |
| **ARIES** (Algorithms for Recovery and Isolation Exploiting Semantics) | The classic recovery method of disk-based databases; its log holds both redo and undo information. |
| **lost update** | Two transactions read the same value, both change it, and the second write silently erases the first. |

## 1. ACID from the inside

Lesson 16.7 showed what ACID looks like from the outside. Here is which part of a database keeps each promise:

| Letter | Promise | Who keeps it | Where in this course |
|---|---|---|---|
| **A**tomicity | all changes of a transaction, or none | the log: a transaction counts only if its commit record is complete | this lesson |
| **C**onsistency | the rules you declared still hold after the transaction | constraints in the database **and** your own code | this lesson, Section 5 |
| **I**solation | concurrent transactions do not see each other's half-done work | locks or multiple versions | Lessons 17.11 and 17.12 |
| **D**urability | after "commit OK", the change survives a crash | `sync_data` on the log before answering | Lessons 17.2 and 17.4 |

Atomicity and durability meet at one moment, the **commit point**: the moment the commit record is durable in the log.
Everything in this lesson is about putting that moment in the right place.

## 2. Atomic commit with one record

The idea fits in three rules:

1. While a transaction runs, its changes go into a **write set** in memory. Nothing touches the log or the data.
2. To commit, write the whole write set to the log as **one record** with a length and a CRC, then `sync_data`.
3. On startup, replay only records whose length and CRC are correct. A record cut by a crash fails the check and is
   dropped, together with everything after it.

```text
 bank.log:
 +-----+-----+------------------------------+-----+-----+--------------------------
 | len | crc | alice=100, bob=50            | len | crc | alice=70, bob=80 ...  <- crash here
 +-----+-----+------------------------------+-----+-----+--------------------------
 \_______ complete: replayed ______________/ \______ torn: CRC check fails, cut off __
```

Rollback is free: you simply never write the write set. A crash before the commit point leaves at most a torn record,
which recovery removes. So there is no state in which only half of a transaction is visible.

> 💡 **Tip:** many engines write a transaction as several log records (`BEGIN`, one record per change, `COMMIT`)
> instead of one big record. The rule is the same: recovery applies a transaction only if its `COMMIT` record is
> complete. One record per transaction is the simplest form; LevelDB and RocksDB use it for their "write batches".

## 3. A transactional key-value store in Rust

Create the project:

```bash
cd ~/projects/rust
cargo new --lib code/p17-txkv
cd code/p17-txkv
cargo add crc32fast@1
cargo add tempfile@3 --dev
```

Keys and values are `String`s, and the data lives in a `BTreeMap` (a sorted map, Lesson 07.11) in memory, rebuilt from
the log at startup. Replace `src/lib.rs` with:

```rust
use std::collections::BTreeMap;
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Write};
use std::path::Path;

/// The changes of one transaction: key -> Some(new value) or None (delete).
#[derive(Debug, Default)]
pub struct Tx {
    writes: BTreeMap<String, Option<String>>,
}

impl Tx {
    pub fn put(&mut self, key: &str, value: &str) {
        self.writes.insert(key.to_string(), Some(value.to_string()));
    }

    pub fn delete(&mut self, key: &str) {
        self.writes.insert(key.to_string(), None);
    }
}

pub struct Db {
    log: File,
    data: BTreeMap<String, String>,
}

impl Db {
    /// Opens the log, replays every complete transaction, cuts off a torn tail.
    pub fn open(path: &Path) -> io::Result<Db> {
        let mut log = OpenOptions::new()
            .read(true)
            .append(true)
            .create(true)
            .open(path)?;
        let mut bytes = Vec::new();
        log.read_to_end(&mut bytes)?;

        let mut data = BTreeMap::new();
        let mut pos = 0;
        while let Some((writes, len)) = decode_record(&bytes[pos..]) {
            apply(&mut data, writes);
            pos += len;
        }
        if pos < bytes.len() {
            log.set_len(pos as u64)?; // remove the unfinished transaction
            log.sync_all()?;
        }
        Ok(Db { log, data })
    }

    pub fn begin(&self) -> Tx {
        Tx::default()
    }

    /// Makes all changes of `tx` durable as ONE log record, then applies them.
    pub fn commit(&mut self, tx: Tx) -> io::Result<()> {
        if tx.writes.is_empty() {
            return Ok(());
        }
        self.log.write_all(&encode_record(&tx))?;
        self.log.sync_data()?; // the commit point
        apply(&mut self.data, tx.writes.into_iter().collect());
        Ok(())
    }

    pub fn get(&self, key: &str) -> Option<&str> {
        self.data.get(key).map(String::as_str)
    }
}

type Writes = Vec<(String, Option<String>)>;

fn apply(data: &mut BTreeMap<String, String>, writes: Writes) {
    for (key, value) in writes {
        match value {
            Some(v) => data.insert(key, v),
            None => data.remove(&key),
        };
    }
}

/// Record = [payload length: u32][CRC-32 of payload: u32][payload].
/// Payload = for each write: [key len: u32][key][tag: 1 = put, 0 = delete][value len: u32][value]
pub fn encode_record(tx: &Tx) -> Vec<u8> {
    let mut payload = Vec::new();
    for (key, value) in &tx.writes {
        put_bytes(&mut payload, key.as_bytes());
        match value {
            Some(v) => {
                payload.push(1);
                put_bytes(&mut payload, v.as_bytes());
            }
            None => payload.push(0),
        }
    }
    let mut record = Vec::with_capacity(8 + payload.len());
    record.extend_from_slice(&(payload.len() as u32).to_le_bytes());
    record.extend_from_slice(&crc32fast::hash(&payload).to_le_bytes());
    record.extend_from_slice(&payload);
    record
}

fn put_bytes(out: &mut Vec<u8>, bytes: &[u8]) {
    out.extend_from_slice(&(bytes.len() as u32).to_le_bytes());
    out.extend_from_slice(bytes);
}

/// Returns the writes and the record length, or None if the record is
/// incomplete or damaged.
fn decode_record(buf: &[u8]) -> Option<(Writes, usize)> {
    let len = u32::from_le_bytes(buf.get(0..4)?.try_into().ok()?) as usize;
    let crc = u32::from_le_bytes(buf.get(4..8)?.try_into().ok()?);
    let payload = buf.get(8..8 + len)?;
    if crc32fast::hash(payload) != crc {
        return None;
    }
    let mut writes = Vec::new();
    let mut p = payload;
    while !p.is_empty() {
        let key = take_bytes(&mut p)?;
        let (&tag, rest) = p.split_first()?;
        p = rest;
        let value = if tag == 1 {
            Some(take_bytes(&mut p)?)
        } else {
            None
        };
        writes.push((key, value));
    }
    Some((writes, 8 + len))
}

fn take_bytes(p: &mut &[u8]) -> Option<String> {
    let len = u32::from_le_bytes(p.get(0..4)?.try_into().ok()?) as usize;
    let bytes = p.get(4..4 + len)?;
    *p = &p[4 + len..];
    String::from_utf8(bytes.to_vec()).ok()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_crash_at_any_byte_of_a_commit_loses_the_whole_transaction() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("tx.log");
        let mut db = Db::open(&path).unwrap();
        let mut t1 = db.begin();
        t1.put("alice", "100");
        t1.put("bob", "50");
        db.commit(t1).unwrap();
        drop(db);
        let good_len = std::fs::metadata(&path).unwrap().len();

        let mut t2 = Tx::default();
        t2.put("alice", "70");
        t2.put("bob", "80");
        let record = encode_record(&t2);

        for cut in 0..record.len() {
            // "crash": only the first `cut` bytes of t2's record reached the file
            let mut f = OpenOptions::new().append(true).open(&path).unwrap();
            f.write_all(&record[..cut]).unwrap();
            drop(f);

            let db = Db::open(&path).unwrap();
            assert_eq!(db.get("alice"), Some("100"), "cut at byte {cut}");
            assert_eq!(db.get("bob"), Some("50"), "cut at byte {cut}");
            assert_eq!(std::fs::metadata(&path).unwrap().len(), good_len);
        }
    }
}
```

**What happens:**

1. `Tx` is only a write set: a sorted map from key to `Some(value)` (put) or `None` (delete). Putting the same key
   twice keeps the last value. A `Tx` does not borrow the `Db`, so you can hold several at once.
2. `commit` encodes the whole write set into one record and appends it. `sync_data` is the **commit point**: only after
   it do we change `data` and return `Ok`. A caller that got `Ok` may tell its user "saved".
3. Rollback has no method: a `Tx` that is dropped without `commit` simply disappears.
4. `encode_record` writes `[length][CRC-32][payload]`, little-endian, like the records of Lesson 17.3. Every string in
   the payload carries its own length, so the decoder knows where it ends.
5. `decode_record` returns `None` for **any** problem: too few bytes for the header, too few for the payload, a wrong
   CRC, a broken string. `buf.get(a..b)?` is the trick: `get` returns `None` instead of panicking when the range is
   outside the buffer, and `?` passes that `None` on.
6. `open` replays records until the first bad one, then `set_len(pos)` cuts the file there and syncs. Without the cut,
   the next commit would be appended **after** the garbage, and recovery would never reach it.
7. The test writes one good transaction, then simulates a crash at **every** byte of a second commit: it appends only
   the first `cut` bytes, reopens, and checks that the second transaction is completely gone and the file is repaired.

```bash
cargo test
```

```text
running 1 test
test tests::a_crash_at_any_byte_of_a_commit_loses_the_whole_transaction ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.16s
```

> 🧪 **Testing:** "crash at every byte" is a cheap and strong test. A real crash is just "the file ends somewhere", so
> appending every possible prefix of a record covers every place a crash can cut it. The project in Lesson 17.14 uses
> the same idea.

Now a program that uses it. Create `src/bin/bank.rs`:

```rust
use p17_txkv::Db;
use std::path::Path;

fn main() -> std::io::Result<()> {
    let _ = std::fs::remove_file("bank.log"); // start fresh every run
    let mut db = Db::open(Path::new("bank.log"))?;

    let mut setup = db.begin();
    setup.put("alice", "100");
    setup.put("bob", "50");
    db.commit(setup)?;
    show("after setup", &db);

    // a transfer: two changes, committed together
    let mut transfer = db.begin();
    transfer.put("alice", "70");
    transfer.put("bob", "80");
    db.commit(transfer)?;
    show("after transfer", &db);

    // a transaction that is dropped instead of committed = rollback
    let mut mistake = db.begin();
    mistake.delete("alice");
    drop(mistake);
    show("after rollback", &db);

    drop(db);
    let db = Db::open(Path::new("bank.log"))?; // like a restart
    show("after reopen", &db);
    println!("log size: {} bytes", std::fs::metadata("bank.log")?.len());
    Ok(())
}

fn show(label: &str, db: &Db) {
    let alice = db.get("alice").unwrap_or("-");
    let bob = db.get("bob").unwrap_or("-");
    println!("{label:<15} alice={alice} bob={bob}");
}
```

```bash
cargo run --bin bank
```

```text
after setup     alice=100 bob=50
after transfer  alice=70 bob=80
after rollback  alice=70 bob=80
after reopen    alice=70 bob=80
log size: 77 bytes
```

**What happens:**

1. The transfer changes two keys; both changes reach the log in one record, so no crash can leave "alice paid, bob not
   paid".
2. `mistake` is dropped: the delete never happened, and nothing was written.
3. Reopening replays two records (the setup and the transfer): 39 + 38 = 77 bytes.

Look at the first record with `xxd`, a tool that prints bytes in hexadecimal (base 16):

```bash
xxd bank.log | head -3
```

```text
00000000: 1f00 0000 ea6a 7fbe 0500 0000 616c 6963  .....j......alic
00000010: 6501 0300 0000 3130 3003 0000 0062 6f62  e.....100....bob
00000020: 0102 0000 0035 301e 0000 006e cc3c b905  .....50....n.<..
```

`1f00 0000` is the payload length 31 (0x1f) in little-endian; `ea6a 7fbe` is the CRC; then `0500 0000` and `alice`,
the tag `01` (put), `0300 0000` and `100`, and so on. At offset 0x27 the second record starts with its length `1e`
(30).

## 4. Redo, undo, and the buffer pool

Our store is simple because uncommitted changes never leave the `Tx`. The log holds only **redo information**: new
values of committed transactions, which recovery repeats.

A disk-based database with a buffer pool (Lesson 17.9) is less lucky. A long transaction may change more pages than fit
in the pool, so the pool must sometimes write a dirty page that contains **uncommitted** changes to the data file. If
the system then crashes, the data file holds changes of a transaction that never committed. Recovery must take them
back, so the log must also contain **undo information**: the old values.

| Question | If "yes" the database needs |
|---|---|
| May a dirty page with uncommitted changes be written to the data file? | undo information in the log |
| May "commit OK" be returned before all changed pages are in the data files? | redo information in the log |

Almost every real database answers "yes" to both, because the alternatives are slow: keeping all of a transaction in
memory, or writing every changed page at commit. The classic recovery method that combines redo and undo is called
**ARIES** (Algorithms for Recovery and Isolation Exploiting Semantics); you do not need its details now. PostgreSQL takes another path for undo: an update never overwrites the old
row but writes a new version next to it, so "undo" just means "this version's transaction did not commit, ignore it".
Lesson 17.12 shows those versions.

## 5. Consistency is partly your job

"C" is the odd letter. The database can check only the rules you **declare**: `NOT NULL`, `UNIQUE`, `CHECK (balance >=
0)`, foreign keys. If any of them fails, the whole transaction is rolled back (you saw this in Lesson 16.7).

Rules that are not declared are your job. "The total amount of money never changes" is an **invariant** that no
constraint expresses. A transfer keeps it only if your code changes both accounts in the **same** transaction:

```text
transfer(alice -> bob, 30):     alice 100 -> 70,  bob 50 -> 80     total 150 -> 150  OK
crash between two transactions: alice 100 -> 70,  bob 50           total 150 -> 120  money vanished
```

Atomicity guarantees that a transaction is all-or-nothing. It is your code that decides what goes into one transaction.

> 🔒 **Security:** a check like "is the balance high enough?" must happen **inside** the transaction that changes the
> balance, and under proper isolation (next lesson). If the check and the change are separate, an attacker can send many
> requests at the same moment and spend the same money several times.

## 6. What ACID in our store does not give: isolation

Two clients that run at the same time can still break the invariant. Create `src/bin/lost.rs`:

```rust
use p17_txkv::Db;
use std::path::Path;

fn main() -> std::io::Result<()> {
    let _ = std::fs::remove_file("lost.log");
    let mut db = Db::open(Path::new("lost.log"))?;
    let mut setup = db.begin();
    setup.put("alice", "100");
    db.commit(setup)?;

    // two "clients" both take 10 from alice at the same time
    let mut t1 = db.begin();
    let mut t2 = db.begin();
    let seen1: i64 = db.get("alice").unwrap().parse().unwrap();
    let seen2: i64 = db.get("alice").unwrap().parse().unwrap();
    t1.put("alice", &(seen1 - 10).to_string());
    t2.put("alice", &(seen2 - 10).to_string());
    db.commit(t1)?;
    db.commit(t2)?;

    println!("alice = {} (expected 80)", db.get("alice").unwrap());
    Ok(())
}
```

```bash
cargo run --bin lost
```

```text
alice = 90 (expected 80)
```

**What happens:**

1. Both transactions read `alice = 100` before either commits.
2. Each writes `100 - 10 = 90`. Both commits are atomic and durable, and still 10 is lost: the second write erased the
   first. This is a **lost update**.
3. Atomicity and durability are about crashes. This bug has no crash; it is about two transactions **overlapping in
   time**. Preventing it is the job of **isolation**, the topic of Lessons 17.11 and 17.12.

## Common mistakes

> ⚠️ **Common mistake:** writing a transaction as one log record per key without a commit marker. We tried it: with
> `commit` changed to append one record per key, the crash test fails, because a crash between the two records keeps
> alice's new value and loses bob's:
>
> ```text
> thread 'tests::a_crash_at_any_byte_of_a_commit_loses_the_whole_transaction' (6118007) panicked at src/lib.rs:177:13:
> assertion `left == right` failed: cut at byte 24
>   left: Some("70")
>  right: Some("100")
> ```

> ⚠️ **Common mistake:** changing the in-memory data (or answering "OK") **before** `sync_data` returns. If the sync
> fails or the machine crashes, the user was told "saved" about data that is gone. The order is: write, sync, then apply
> and answer.

> ⚠️ **Common mistake:** not cutting off a torn tail at startup. The next commit is appended after the garbage, recovery
> stops at the garbage, and every later transaction is lost on the next restart.

## Try it yourself

1. Run `cargo run --bin bank`, then cut the log in the middle of the second record and reopen it. On macOS:
   `head -c 60 bank.log > cut.log && mv cut.log bank.log`, then add a small program `src/bin/show.rs` that opens
   `bank.log` and prints alice and bob (without deleting the file first). You should see `alice=100 bob=50`, and
   `ls -l bank.log` shows 39 bytes: the torn record was cut off.
2. In `bank.rs`, put the same key twice in one transaction (`put("alice", "1")`, then `put("alice", "2")`). Which value
   wins, and how many writes does the record contain?

## Exercises

### Exercise 1 — Count the bytes (easy)

How many bytes does `encode_record` produce for a transaction with the single write `put("k", "v")`? Compute it from the
format, then confirm it with a test.

<details>
<summary>Hint</summary>

Header: 4 + 4 bytes. Payload: key length (4) + key + tag (1) + value length (4) + value.

</details>

<details>
<summary>Answer</summary>

8 + (4 + 1 + 1 + 4 + 1) = 19 bytes.

```rust
    #[test]
    fn record_size_of_one_small_put() {
        let mut tx = Tx::default();
        tx.put("k", "v");
        assert_eq!(encode_record(&tx).len(), 19);
    }
```

Why it works: the format has no hidden padding; every field has a fixed size except the strings, which carry their own
lengths.

</details>

### Exercise 2 — Test that a rollback leaves no trace (medium)

Write a test `rollback_leaves_no_trace`: commit `alice = 100`, remember the log's size, then start a transaction that
sets `alice = 0` and `eve = 1000` and drop it. Also commit an empty transaction. Check that the log did not grow and
that after reopening alice is still 100 and eve does not exist.

<details>
<summary>Hint</summary>

`std::fs::metadata(&path)?.len()` gives the file size. `db.commit(db.begin())` commits an empty transaction.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn rollback_leaves_no_trace() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("tx.log");
        let mut db = Db::open(&path).unwrap();
        let mut t1 = db.begin();
        t1.put("alice", "100");
        db.commit(t1).unwrap();
        let len_before = std::fs::metadata(&path).unwrap().len();

        let mut t2 = db.begin();
        t2.put("alice", "0");
        t2.put("eve", "1000");
        drop(t2); // rollback
        db.commit(db.begin()).unwrap(); // an empty commit writes nothing either

        assert_eq!(std::fs::metadata(&path).unwrap().len(), len_before);
        drop(db);
        let db = Db::open(&path).unwrap();
        assert_eq!(db.get("alice"), Some("100"));
        assert_eq!(db.get("eve"), None);
    }
```

Why it works: a `Tx` never touches the log; only `commit` writes, and it returns early for an empty write set.

</details>

### Exercise 3 — Read your own writes (hard)

Inside a transaction, a program expects to see its own changes: after `tx.put("a", "10")`, reading `a` *in this
transaction* should give 10, while everybody else still sees the old value. Add
`pub fn get_in<'a>(&'a self, tx: &'a Tx, key: &str) -> Option<&'a str>` to `Db` and test it with a put and a delete.

<details>
<summary>Hint</summary>

Look in `tx.writes` first. There are three cases: `Some(Some(v))` (written), `Some(None)` (deleted in this
transaction), `None` (not touched: ask the committed data). The method is in the same module as `Tx`, so it may read the
private field `writes`.

</details>

<details>
<summary>Answer</summary>

```rust
    /// Reads a key as transaction `tx` sees it: its own writes first.
    pub fn get_in<'a>(&'a self, tx: &'a Tx, key: &str) -> Option<&'a str> {
        match tx.writes.get(key) {
            Some(Some(value)) => Some(value),
            Some(None) => None, // deleted in this transaction
            None => self.get(key),
        }
    }
```

```rust
    #[test]
    fn a_transaction_reads_its_own_writes() {
        let dir = tempfile::tempdir().unwrap();
        let mut db = Db::open(&dir.path().join("tx.log")).unwrap();
        let mut setup = db.begin();
        setup.put("a", "1");
        setup.put("b", "2");
        db.commit(setup).unwrap();

        let mut tx = db.begin();
        tx.put("a", "10");
        tx.delete("b");
        assert_eq!(db.get_in(&tx, "a"), Some("10"));
        assert_eq!(db.get_in(&tx, "b"), None);
        assert_eq!(db.get("a"), Some("1")); // others still see the old value
        assert_eq!(db.get("b"), Some("2"));
    }
```

Why it works: the write set "covers" the committed data like a transparent sheet with some cells painted over. The
lifetime `'a` says that the returned `&str` may come from either the `Db` or the `Tx`, so both must live long enough.

</details>

## Quiz

1. What is the commit point of the store in this lesson, exactly?

<details>
<summary>Answer</summary>

The moment `sync_data` returns after the transaction's record was appended. Before it, a crash can leave at most a torn
record, which recovery removes; after it, the record is complete and durable and will be replayed.

</details>

2. A crash happens after 20 of the 38 bytes of a commit record reached the disk. What does `Db::open` do?

<details>
<summary>Answer</summary>

It replays the earlier complete records, fails to decode the torn one (not enough bytes, or a wrong CRC), cuts the file
at the end of the last good record with `set_len`, and syncs. The torn transaction never happened.

</details>

3. Why does a database with a buffer pool need undo information, while our store does not?

<details>
<summary>Answer</summary>

The buffer pool may write a dirty page with uncommitted changes to the data file. After a crash those changes must be
taken back, which needs the old values. Our store never writes uncommitted changes anywhere.

</details>

4. Which ACID letter is partly the responsibility of the application, and why?

<details>
<summary>Answer</summary>

Consistency. The database can only check declared rules (constraints). Invariants that are not declared, such as "the
total money is constant", hold only if the application puts all related changes into one transaction.

</details>

5. Both transactions in `lost.rs` committed atomically and durably. Why is the result still wrong?

<details>
<summary>Answer</summary>

They overlapped in time: both read 100 before either wrote. That is an isolation problem (a lost update), not a crash
problem, so atomicity and durability cannot help.

</details>

## Summary

- ACID: atomicity and durability come from the log and `sync_data`, isolation from locks or versions, consistency from
  constraints plus your own code.
- A transaction collects its changes in a write set; commit writes them as one checksummed record and syncs. That sync
  is the commit point.
- Recovery replays complete records and cuts off a torn tail; a cut transaction disappears completely.
- Test crash safety by writing every prefix of a record and reopening.
- Redo information repeats committed changes; undo information takes back uncommitted ones, which a buffer pool that
  writes dirty pages early needs.
- A transaction groups changes; deciding *which* changes belong together is the application's job.
- Atomic and durable transactions can still lose updates when they overlap: that is what isolation is for.

## Sources and further reading

- [PostgreSQL: Transactions (tutorial)](https://www.postgresql.org/docs/current/tutorial-transactions.html) — the bank transfer example in PostgreSQL's own words.
- [PostgreSQL: Write-Ahead Logging (WAL)](https://www.postgresql.org/docs/current/wal-intro.html) — how PostgreSQL makes commits durable.
- [SQLite: Atomic Commit In SQLite](https://www.sqlite.org/atomiccommit.html) — a detailed, readable walk through atomic commit with a rollback journal.
- [LevelDB implementation notes](https://github.com/google/leveldb/blob/main/doc/impl.md) — the log and write batches of a real key-value store.
- [crc32fast documentation](https://docs.rs/crc32fast/latest/crc32fast/) — the checksum crate used here.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — its chapter on transactions explains ACID and isolation very clearly.

---

[🏠 Start](../START-HERE.md) · [← 17.9 The page cache and the buffer pool](09-page-cache-and-buffer-pool.md) · [📋 Phase 17](README.md) · [17.11 Isolation levels →](11-isolation-levels.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
