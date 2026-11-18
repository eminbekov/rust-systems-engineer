<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.13 How a SQL query runs](13-how-a-sql-query-runs.md) · [📋 Phase 17](README.md) · [17.15 Checkpoint →](15-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/14-project-key-value-store.md) <!-- nav -->

# Lesson 17.14 — Project: your own key-value store

> **Phase 17 — Storage and database internals** · Lesson 14 of 15 · ⏱ about 15 hours (8 sittings)

## What you will learn

- How to combine the ideas of this phase into one storage engine: a durable log, an index, compaction and recovery.
- How to make a multi-step change (compaction) crash-safe with a single commit point, the `MANIFEST` file.
- How to prove crash safety with tests: a crash at every byte, a crash in the middle of compaction, and a model-based
  test against `BTreeMap`.
- How to measure your engine with `criterion` and explain the numbers.

This lesson has no full solution. It gives you a contract, milestones, small hints and the exact commands that show a
milestone is done. Plan for several sittings; the time above is only for reading the plan.

## Before you start

- You finished [Lesson 17.13 — How a SQL query runs](13-how-a-sql-query-runs.md). Keep these lessons open, the project
  uses them all: Lessons [17.2](02-durability-and-fsync.md), [17.3](03-binary-file-formats.md),
  [17.4](04-write-ahead-log.md), [17.5](05-hash-indexes.md),
  [17.7](07-lsm-trees.md), [17.8](08-compaction-and-bloom-filters.md) and
  [17.10](10-transactions-and-acid.md).
- You know `clap` from [Lesson 05.11 — Practical crates: serde, clap, thiserror and anyhow](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md),
  `assert_cmd` from [Lesson 09.2 — End-to-end tests for command-line programs](../09-testing-debugging-benchmarking/02-testing-command-line-programs.md),
  `proptest` from [Lesson 09.4 — Property-based testing](../09-testing-debugging-benchmarking/04-property-based-testing.md)
  and `criterion` from [Lesson 09.11 — Benchmarking in depth](../09-testing-debugging-benchmarking/11-benchmarking-in-depth.md).
- Create the project and a Git branch:

```bash
cd ~/projects/rust
git switch main
git switch -c project/p17-kv
cargo new code/p17-kv
cd code/p17-kv
cargo add crc32fast@1
cargo add clap@4 --features derive
cargo add --dev tempfile@3 proptest@1 assert_cmd@2 predicates@3 criterion@0.8
```

## New words

| Word | Meaning |
|---|---|
| **storage engine** | The part of a database that stores and finds bytes on disk: records, indexes, logs, compaction, recovery. |
| **`MANIFEST`** | A small file that names the data files that make up the current state of the store. Replacing it atomically is the commit point of compaction. |
| **generation** | A number that grows every time compaction produces new data files; it appears in their names (`data-000002.log`). |
| **tombstone** | A record that says "this key was deleted". It must survive as long as older records of the key might still be read. |
| **model-based test** | A test that runs random operations on your store **and** on a simple, obviously correct model (here a `BTreeMap`), and checks that both always give the same answers. |
| **acceptance test** | A command with an expected output that proves a milestone is done ("Done when"). |
| **write amplification** | How many bytes the engine writes to disk for each byte the user wrote. Compaction increases it. |

## The project

### What you will build

```text
  p17-kv put / get / delete / compact / stats        (CLI, src/main.rs)
             |
             v
  +----------------------- Store (src/lib.rs) ------------------------+
  |  index in RAM: key -> where the newest record is                  |
  |                                                                   |
  |  append-only data file  data-<generation>.log   (CRC per record)  |
  |  MANIFEST: which data file(s) are current                         |
  +-------------------------------------------------------------------+
             |
             v
     directory given by --dir (default ./data)
```

A command-line key-value store whose data survives restarts and crashes. Every `put` and `delete` is durable before
the command prints its answer. `compact` throws away stale records, and a crash in the middle of it never loses data.

You choose **one** of two designs. Both satisfy the same contract:

| | Design A: log + hash index (recommended) | Design B: LSM (Log-Structured Merge) tree |
|---|---|---|
| Writes | append a record to the data file, `sync_data` | append to the WAL (write-ahead log), `sync_data`, insert into the memtable |
| Index | `HashMap<key, (offset, length)>` in RAM (Lesson 17.5) | memtable + sorted SSTables, newest first (Lesson 17.7) |
| Compaction | rewrite live records into a new data file | merge SSTables into one, drop shadowed records and tombstones (Lesson 17.8) |
| Commit point of compaction | `MANIFEST` names the new file | `MANIFEST` lists the new SSTables |

Design A is shorter. Design B teaches more and supports range scans as a stretch goal. Write your choice and the
reason into the README.

### Contract

**Command line** (`clap` derive). Every command takes `--dir <DIR>` (default `data`); the directory is created if it
does not exist.

| Command | Prints on stdout | Exit code |
|---|---|---|
| `put <KEY> <VALUE>` | `OK` | 0 |
| `get <KEY>` | the value | 0; if missing: `not found` on **stderr**, exit 1 |
| `delete <KEY>` | `deleted` | 0; if missing: `not found` on stderr, exit 1 |
| `compact` | `compacted: <bytes before> -> <bytes after> bytes` | 0 |
| `stats` | `keys: <number of live keys>` and `disk bytes: <bytes of all data files>` on two lines | 0 |
| any error (input/output, bad input, corrupt data) | `error: <message>` on stderr | 2 |

**Library** (`src/lib.rs`), so that tests and benchmarks can use the store without the CLI (signatures only; the
bodies are your work):

```rust
pub struct Store { /* your fields */ }
pub struct Stats { pub keys: usize, pub disk_bytes: u64 }
pub enum Error { Io(std::io::Error), EmptyKey, TooLarge, Corrupt(String) }

impl Store {
    pub fn open(dir: &Path) -> Result<Store, Error>;
    pub fn put(&mut self, key: &[u8], value: &[u8]) -> Result<(), Error>;
    pub fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>, Error>;
    pub fn delete(&mut self, key: &[u8]) -> Result<bool, Error>; // false: the key did not exist
    pub fn compact(&mut self) -> Result<(), Error>;
    pub fn stats(&self) -> Stats;
}
```

`Error` implements `std::fmt::Display` and `std::error::Error`. (Design B may need `&mut self` for `get`; that is
allowed.)

**Rules:**

1. Keys are 1 to 1,024 bytes, values 0 to 1,048,576 bytes (1 MiB, one mebibyte). An empty key is `Error::EmptyKey`, anything
   bigger is `Error::TooLarge`.
2. **Durability:** when `put` or `delete` returns `Ok`, the change is on stable storage (`sync_data` was called).
3. **Integrity:** every record on disk has a CRC-32 (Cyclic Redundancy Check, 32 bits). Lengths read from disk are checked against the limits of rule 1
   before any buffer is allocated.
4. **Recovery:** `open` replays the current data, and a torn record at the end of the newest file is cut off (as in
   Lessons 17.4 and 17.10). A bad record anywhere else is `Error::Corrupt`: that file was complete once, so the damage
   is real.
5. **Atomic compaction:** a crash at any moment during `compact` leaves the store either exactly as before or exactly
   as after. `open` deletes leftover files that the `MANIFEST` does not name.
6. Deleting a missing key writes nothing.

**Tests** that must exist (the names are part of the contract, so that "Done when" can name them):

| Test | What it proves |
|---|---|
| `torn_tail_at_every_byte_is_cut_off` | after a crash at any byte of the last record, `open` keeps all earlier data and the file ends after the last complete record |
| `crash_during_compaction_keeps_old_or_new_state` | stopping compaction after each of its steps, then reopening, gives the same answers as before |
| `matches_btreemap_model` | random `put` / `delete` / `compact` / reopen sequences give the same answers as a `BTreeMap` |
| `put_then_get_in_a_new_process` (in `tests/cli.rs`) | the CLI really persists data between processes |
| `missing_key_exits_with_1` (in `tests/cli.rs`) | the exit code and stderr of the contract |

### Step-by-step plan

#### M1 — A durable log with an index, and the CLI

**Goal:** `put`, `get`, `delete` and `stats` work across separate runs of the program.

**Guidance:** start from your `HashLog` of Lesson 17.5 or your WAL store of Lesson 17.4, but give `open` a directory
instead of a file. For now the directory holds one data file, `data-000001.log`; M3 adds the `MANIFEST`. Use the record
format of Lesson 17.3 with a put/delete tag. Call `sync_data` after every append. In `main`, parse the arguments with
`clap`, call the library, print exactly the contract's text and return a `std::process::ExitCode`. Write the two CLI
tests with `assert_cmd`.

<details>
<summary>Hint</summary>

`fn main() -> ExitCode` with a helper `fn run(cli: Cli) -> Result<ExitCode, Error>` keeps error handling in one place:
`main` matches on the result, prints `error: {e}` and returns `ExitCode::from(2)` for an `Err`. For a missing key, `run`
itself prints `not found` with `eprintln!` and returns `Ok(ExitCode::from(1))`.

</details>

**Done when:**

```bash
cargo run -q -- --dir demo put apple red
cargo run -q -- --dir demo put kiwi green
cargo run -q -- --dir demo get apple
cargo run -q -- --dir demo get mango; echo "exit code: $?"
cargo run -q -- --dir demo delete kiwi
cargo run -q -- --dir demo get kiwi; echo "exit code: $?"
cargo run -q -- --dir demo stats
cargo run -q -- --dir demo put '' x; echo "exit code: $?"
```

```text
OK
OK
red
not found
exit code: 1
deleted
not found
exit code: 1
keys: 1
disk bytes: 60
error: the key must not be empty
exit code: 2
```

The number of disk bytes depends on your record format (60 is what a 13-byte header gives); everything else must be
identical. `cargo test --test cli` passes both CLI tests. Commit: `M1: durable log, index and CLI`.

#### M2 — Crash safety of the log

**Goal:** a crash in the middle of a write never damages earlier data, and a corrupt record is never trusted.

**Guidance:** make `open` stop at the first record that is incomplete or has a wrong CRC, cut the file there with
`set_len` and sync it. Before reading a record's key and value, check the lengths against the limits (rule 3): a random
4-byte length in a damaged file could otherwise ask for a 4 GiB (gibibyte) allocation. Write `torn_tail_at_every_byte_is_cut_off`:
commit one value, then for every `cut` in `0..record.len()` append the first `cut` bytes of a second record, reopen and
check the first value and the file length (the same idea as Lesson 17.10).

<details>
<summary>Hint</summary>

The test needs the bytes of a record without going through `put`. Make your encoder a `pub(crate) fn encode(...)` in
the library, so that the unit tests in the same crate can call it. `std::fs::metadata(path)?.len()` gives the file
length.

</details>

**Done when:**

```bash
cargo test torn_tail
```

```text
running 1 test
test tests::torn_tail_at_every_byte_is_cut_off ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out; finished in 0.07s
```

(The number of filtered-out tests depends on how many other tests you already have.) Commit: `M2: torn-tail recovery`.

#### M3 — Compaction with a `MANIFEST`

**Goal:** `compact` removes stale records and deleted keys, and a crash during compaction is harmless.

**Guidance:** compaction has several steps, and the trick from Lesson 17.10 applies: make exactly **one** step the
commit point. For Design A:

```text
1. write all live records to data-<generation+1>.log, sync_all       (crash here: old MANIFEST, new file ignored)
2. write MANIFEST.tmp naming generation+1, sync_all                    (crash here: still the old state)
3. rename MANIFEST.tmp -> MANIFEST, sync the directory   <- COMMIT POINT
4. delete the old data file, sync the directory                         (crash here: new state, a leftover file)
```

`open` reads `MANIFEST` (if it does not exist, create one for generation 1), opens only the file it names, and deletes
every other `data-*.log` and `*.tmp` file: they are leftovers of a crash before or after the commit point. Without a
`MANIFEST`, a crash between steps 3 and 4 would leave two data files, and a key deleted before compaction could come
back from the old file. For Design B, the `MANIFEST` lists the SSTables, and compaction writes merged SSTables first.
Write `crash_during_compaction_keeps_old_or_new_state`.

<details>
<summary>Hint</summary>

To "crash" in a test without killing the test process, give the library a test-only variant such as
`fn compact_with_crash(&mut self, stop_after: Option<u8>)` that returns early after step 1 or step 2, and make
`compact` call it with `None`. Drop the store, reopen it and compare answers and the list of files with
`std::fs::read_dir`.

</details>

**Done when:**

```bash
cargo build -q
KV=./target/debug/p17-kv
for i in $(seq 1 300); do $KV --dir demo2 put counter "$i" > /dev/null; done
$KV --dir demo2 put name kv
$KV --dir demo2 stats
$KV --dir demo2 compact
$KV --dir demo2 stats
$KV --dir demo2 get counter
ls demo2
```

```text
OK
keys: 2
disk bytes: 6811
compacted: 6811 -> 42 bytes
keys: 2
disk bytes: 42
300
MANIFEST
data-000002.log
```

Again your byte counts will differ, but compaction must shrink 301 records of two keys to 2 records, and only the
`MANIFEST` and one data file may remain (Design B: the files your `MANIFEST` names). Commit: `M3: atomic compaction`.

#### M4 — A model-based test

**Goal:** confidence that every sequence of operations is correct, not only the ones you thought of.

**Guidance:** with `proptest`, generate a `Vec` of operations: `Put(key, value)`, `Delete(key)`, `Compact` and
`Reopen` (drop the store and open it again). Use few keys (for example 8 one-byte keys), so that operations hit the same
keys often. Apply each operation to the store and to a `BTreeMap`; after each one, check `get` for every key, and check
that `delete` returns the same `bool` as `BTreeMap::remove(...).is_some()`.

<details>
<summary>Hint</summary>

An `enum Op` and a strategy built with `prop_oneof![...]` and `prop_map` describe the operations. Every `put` syncs to
disk, so keep the cases small: `ProptestConfig::with_cases(64)` and sequences of up to 40 operations take a few seconds.

</details>

**Done when:**

```bash
cargo test
```

shows all five contract tests (the order may differ, and your own extra tests appear too):

```text
test tests::torn_tail_at_every_byte_is_cut_off ... ok
test tests::crash_during_compaction_keeps_old_or_new_state ... ok
test tests::matches_btreemap_model ... ok
test put_then_get_in_a_new_process ... ok
test missing_key_exits_with_1 ... ok
```

Break something on purpose (for example, skip the tombstone during replay) and check that `matches_btreemap_model`
fails and prints a short failing sequence. Commit: `M4: model-based test`.

#### M5 — Benchmarks

**Goal:** know how fast your engine is, and why.

**Guidance:** create `benches/kv.rs` with `criterion` (and `[[bench]] name = "kv"`, `harness = false` in
`Cargo.toml`). Load a few thousand keys once, then measure `get` of existing keys and a synced `put`. Use a small sample
size (`group.sample_size(10)`), because synced writes are slow. Run with a short measurement time:
`cargo bench --bench kv -- --warm-up-time 1 --measurement-time 3`.

<details>
<summary>Hint</summary>

Loading 10,000 keys with one sync each takes about 40 seconds on a Mac, because every sync is a full
`F_FULLFSYNC` (Lesson 17.2). Load fewer keys, or add a `put_many` that syncs once for a whole batch (stretch goal 1).

</details>

**Done when:** `cargo bench` prints a time for both benchmarks. On the Mac used for this course the Design A
reference printed:

```text
kv/get                  time:   [530.77 ns 685.99 ns 914.79 ns]
kv/put (synced)         time:   [3.8685 ms 4.0176 ms 4.1678 ms]
```

Your numbers will differ. Write into the README what dominates each number (a `get` is a hash lookup plus one
`read_at` served from the page cache; a synced `put` is almost entirely the time of the sync) and how many synced puts
per second that means. Commit: `M5: benchmarks`.

#### M6 — README and the quality gate

**Goal:** a project another person can understand and trust.

**Guidance:** the README explains the design you chose, the on-disk format (a table of the fields of a record), the
commit point of compaction, what the tests prove and what they do not, and the benchmark results. Then run the quality
gate.

**Done when:**

```bash
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
```

finishes without warnings or failures. Commit: `M6: README and quality gate`.

### Stretch goals

1. **Group commit:** `put_many(&[(key, value)])` writes several records and syncs once; measure how many keys per
   second it reaches compared with single synced puts (Lesson 17.2).
2. **A real crash test:** an integration test that starts the CLI in a loop writing keys, kills it with `SIGKILL`
   after a random delay, reopens the store and checks that every key whose `OK` was printed is present.
3. **Segments:** start a new data file when the current one reaches a size limit, so that compaction can rewrite old
   segments while writes continue in the newest one. The `MANIFEST` then lists several files.
4. **Range scans (Design B):** `scan <FROM> <TO>` prints all keys in order, merging the memtable and the SSTables.
5. **A server:** a Tokio TCP (Transmission Control Protocol) server with a line protocol (`PUT k v`, `GET k`, `DEL k`) in front of the same `Store`,
   with one writer task that owns the store and receives commands through a channel (Lesson 14.13).
6. **Hint files (Design A):** at compaction, also write a small file with only keys and offsets, so that `open` does
   not have to read all values (Lesson 17.5).

## Common mistakes

> ⚠️ **Common mistake:** printing `OK` before `sync_data` returns. The contract says that `OK` means "durable"; a
> crash right after the print would make it a lie.

> ⚠️ **Common mistake:** compacting by rewriting the data file **in place**. A crash in the middle destroys the only
> copy. Always write the new state to new files first, then switch with one atomic step.

> ⚠️ **Common mistake:** forgetting to sync the **directory** after `rename` and after creating or deleting files.
> The file data is durable, but the directory entry that names it may not be (Lesson 13.12).

> 🔒 **Security:** the data directory is input too. A damaged or malicious file can contain any length field; check
> lengths before allocating, and return `Error::Corrupt` instead of panicking. A panic in a storage engine during
> `open` means a service that cannot start.

## Quiz

1. Why does `put` call `sync_data` before printing `OK`, and what does that cost on a Mac?

<details>
<summary>Answer</summary>

`OK` promises durability; without the sync the record could still be only in the page cache and be lost in a crash.
On a Mac every sync is an `F_FULLFSYNC` of several milliseconds, so single synced puts manage only a few hundred per
second.

</details>

2. What is the commit point of compaction, and what does `open` do with files from before and after it?

<details>
<summary>Answer</summary>

The atomic `rename` of `MANIFEST.tmp` to `MANIFEST`. Before it, the old `MANIFEST` names the old file, and the new
file is a leftover that `open` deletes. After it, the new `MANIFEST` names the new file, and the old file is the
leftover.

</details>

3. Why is a bad record at the end of the newest file cut off, but a bad record in the middle reported as corruption?

<details>
<summary>Answer</summary>

A crash can only tear the record that was being appended, at the end. A bad record in the middle of a file that was
complete and synced means real damage (a disk error or a bug); hiding it would silently lose data.

</details>

4. What does `matches_btreemap_model` find that the other tests may not?

<details>
<summary>Answer</summary>

Bugs in combinations of operations that you did not think of, for example a delete followed by a compaction and a
reopen. proptest also shrinks a failing sequence to a short one that is easy to debug.

</details>

## Summary

- A storage engine combines a durable append-only log, an index, recovery and compaction; this project builds all four.
- `OK` means durable: sync before answering.
- Every record carries a CRC; recovery cuts a torn tail and refuses real corruption.
- Multi-step changes get one commit point; for compaction it is the atomic replacement of the `MANIFEST`.
- Crash safety is proven with tests: every byte of a torn record, every step of compaction, and a model-based test.
- Benchmarks are only useful with an explanation of what dominates the numbers.

## Sources and further reading

- [Bitcask: A Log-Structured Hash Table for Fast Key/Value Data](https://riak.com/assets/bitcask-intro.pdf) — the short paper behind Design A, including hint files.
- [LevelDB implementation notes](https://github.com/google/leveldb/blob/main/doc/impl.md) — the log, SSTables, the `MANIFEST` and compaction of a real LSM engine (Design B).
- [Rust std: File](https://doc.rust-lang.org/std/fs/struct.File.html) — `sync_all`, `sync_data` and `set_len`.
- [tempfile documentation](https://docs.rs/tempfile/latest/tempfile/) — temporary directories for the tests.
- [Criterion.rs book](https://bheisler.github.io/criterion.rs/book/) — benchmark groups, sample size and measurement time.
- [SQLite: Atomic Commit In SQLite](https://www.sqlite.org/atomiccommit.html) — another engine's careful answer to "what if the crash happens here?".

---

[🏠 Start](../START-HERE.md) · [← 17.13 How a SQL query runs](13-how-a-sql-query-runs.md) · [📋 Phase 17](README.md) · [17.15 Checkpoint →](15-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
