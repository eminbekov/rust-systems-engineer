<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.1 How disks and files work](01-how-disks-and-files-work.md) · [📋 Phase 17](README.md) · [17.3 Binary file formats and serialization →](03-binary-file-formats.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/02-durability-and-fsync.md) <!-- nav -->

# Lesson 17.2 — Durability: fsync and crash safety

> **Phase 17 — Storage and database internals** · Lesson 2 of 15 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The three kinds of crash, and which data each of them destroys.
- What `File::sync_all` and `File::sync_data` really do on Linux and on macOS (where Rust uses `F_FULLFSYNC`).
- How much durability costs, and how *group commit* makes it affordable.
- Why a failed `fsync` must never simply be retried.
- What a *torn write* is, and how to simulate a crash in the middle of a write inside a normal test.

## Before you start

- You finished [Lesson 17.1 — How disks and files work](01-how-disks-and-files-work.md) and have the package
  `code/p17-storage`.
- You know `fsync` and the atomic-replace pattern (temporary file, `sync_all`, `rename`, sync the directory) from
  [Lesson 13.12 — File systems from the inside](../13-operating-systems/12-file-systems-inside.md). This lesson does not
  repeat that pattern; it explains what lies under it.
- You know signals and `abort` from [Lesson 13.7 — Signals in your programs](../13-operating-systems/07-signals-in-your-programs.md),
  and how to implement a trait such as `Write` for your own type ([Lesson 05.2](../05-rust-intermediate/02-traits.md)).

## New words

| Word | Meaning |
|---|---|
| **durable** | Saved so that it survives a crash or a power loss. A durable write has reached storage that keeps data without power. |
| **process crash** | Only your program dies (a panic with `abort`, `kill -9`, a bug). The kernel and its page cache keep running. |
| **OS crash** (Operating System crash) | The kernel itself stops, or the power goes off. Everything in RAM (Random Access Memory) is lost, including the page cache. |
| **`fdatasync`** | A system call like `fsync`, but it may skip metadata that is not needed to read the data back (for example the modification time). Rust: `File::sync_data`. |
| **`F_FULLFSYNC`** | A macOS `fcntl` command that makes the drive itself write its cache to permanent storage. Plain `fsync` on macOS does not do that. |
| **acknowledge** | To tell the caller "your data is saved". A storage engine must not acknowledge a write before it is durable. |
| **group commit** | Making many waiting writes durable with **one** `fsync`, so that each of them pays only a part of its cost. |
| **torn write** | A write that was interrupted by a crash, so that only its first part reached the storage. |
| **fault injection** | Making failures happen on purpose in a test (a full disk, a crash after N bytes) to check how the code reacts. |

## 1. Three kinds of crash

"The program crashed" can mean three very different things. The question is always the same: which of the layers from
Lesson 17.1 lose their contents?

| What happens | Your buffers (`BufWriter`) | Page cache (kernel RAM) | Drive's write cache | Flash / platters |
|---|---|---|---|---|
| **Process crash**: panic, `abort`, `kill -9` | lost | **survives** | survives | survives |
| **OS crash**: kernel panic, the VM (virtual machine) is killed | lost | lost | survives (usually) | survives |
| **Power loss** | lost | lost | lost, unless flushed | survives |

The good news is in the first row. When a `write` system call has returned, the data belongs to the kernel. If only your
process dies, the kernel still writes those pages to the device later. Only power loss and OS crashes need `fsync`.

The bad news is in the last row. A drive acknowledges a write when the data is in its own fast RAM cache, not when it is
in the flash chips. On power loss that cache can be lost too, unless somebody asked the drive to flush it.

## 2. A process crash, shown

Put this into `src/bin/crash.rs`:

```rust
use std::fs::File;
use std::io::{BufWriter, Write};

fn main() -> std::io::Result<()> {
    let mode = std::env::args().nth(1).unwrap_or_default();
    let file = File::create("crash.log")?;

    if mode == "buffered" {
        let mut out = BufWriter::new(file);
        out.write_all(b"line 1\nline 2\nline 3\n")?;
        // The three lines are still in the BufWriter's buffer, in this process's memory.
        std::process::abort();
    } else {
        let mut out = file;
        out.write_all(b"line 1\nline 2\nline 3\n")?;
        // The write system call has finished: the lines are in the kernel's page cache.
        std::process::abort();
    }
}
```

**What happens:**

1. `std::process::abort()` kills the process at once with the signal `SIGABRT`. No destructors run, so a `BufWriter`
   gets no chance to write its buffer. This is the same as a crash.
2. In `buffered` mode the 21 bytes are only copied into the `BufWriter`'s buffer; no system call has happened.
3. In the other mode `write_all` on the `File` makes a `write` system call, which returns before the abort.

Run both modes. Your shell reports each abort (zsh prints a line with `abort`); then look at the file:

```bash
cargo run --release --bin crash -- buffered
wc -c crash.log
cargo run --release --bin crash -- direct
cat crash.log
```

```text
       0 crash.log
line 1
line 2
line 3
```

The buffered lines are gone; the direct lines survived the crash, although nobody called `fsync`. The page cache is
outside your process. So the first rule of crash safety: data that is only in your process's memory is lost on any crash,
even the mildest one.

## 3. What `fsync` really promises

From Lesson 13.12, `File::sync_all` waits until the file's data and metadata are on the storage device. Here are the
details that matter for a database.

**On Linux**, `sync_all` is the `fsync` system call and `sync_data` is `fdatasync`. Both write the dirty pages of this file
to the device and then ask the device to flush its write cache, so that the data is really on the flash. The difference:
`fdatasync` may skip metadata that you do not need to read the data back, such as the modification time. It must still
write the file size if the file grew. For a log that grows, the two cost about the same.

You can watch this with `strace` in your VM `dev` (or in a container from the `rust:1.100` image). The program
`src/bin/sync_calls.rs`:

```rust
use std::fs::File;
use std::io::Write;

fn main() -> std::io::Result<()> {
    let mut file = File::create("sync.log")?;
    file.write_all(b"hello\n")?;
    file.sync_data()?; // Linux: fdatasync
    file.sync_all()?; // Linux: fsync
    std::fs::remove_file("sync.log")?;
    println!("done");
    Ok(())
}
```

Build it in the VM and run `strace -e trace=openat,write,fsync,fdatasync ./target/release/sync_calls`. The important lines
(the loading of libraries at the start is cut):

```text
openat(AT_FDCWD, "sync.log", O_WRONLY|O_CREAT|O_TRUNC|O_CLOEXEC, 0666) = 3
write(3, "hello\n", 6)                  = 6
fdatasync(3)                            = 0
fsync(3)                                = 0
```

**On macOS** the system call `fsync` is weaker. Apple's manual page for `fsync` says that it moves the data from the
computer *to the drive*, but the drive may keep it in its cache and write it later, even in a different order. To ask the
drive to write everything to permanent storage, macOS has a separate command: `fcntl(fd, F_FULLFSYNC)`.

Rust's standard library chooses the safe option. In its source code (`library/std/src/sys/fs/unix.rs`), on Apple systems
**both** `sync_all` and `sync_data` call `fcntl(fd, F_FULLFSYNC)`; on Linux they call `fsync` and `fdatasync`. The
documentation of `File::sync_all` only says that it "attempts to sync all OS-internal file content and metadata to disk";
the platform detail is in the source. So on your Mac, `sync_all` and `sync_data` are equally strong and equally slow.

> 🔒 **Security:** durability is part of *integrity*, one of the three goals from
> [Lesson 10.1](../10-security-fundamentals/01-security-mindset-and-the-cia-triad.md). A bank that says "payment saved"
> and loses it after a power cut has broken a promise as surely as one that was hacked.

## 4. The price of durability

How slow is "slow"? Put this into `src/bin/fsync_cost.rs`. It appends 1,000 records of 100 bytes and syncs in different
ways:

```rust
use std::fs::{self, File, OpenOptions};
use std::io::Write;
use std::os::fd::AsRawFd;
use std::time::{Duration, Instant};

const RECORDS: usize = 1000;
const RECORD: [u8; 100] = [b'r'; 100];

/// A function that makes a file durable, such as `File::sync_data`.
type SyncFn = fn(&File) -> std::io::Result<()>;

/// Appends RECORDS records to a new file and calls `sync` after every `every` records.
fn run(every: Option<usize>, sync: SyncFn) -> std::io::Result<Duration> {
    let mut file = OpenOptions::new()
        .create(true)
        .truncate(true)
        .write(true)
        .open("cost.log")?;
    let start = Instant::now();
    for i in 1..=RECORDS {
        file.write_all(&RECORD)?;
        if every.is_some_and(|n| i % n == 0) {
            sync(&file)?;
        }
    }
    let elapsed = start.elapsed();
    fs::remove_file("cost.log")?;
    Ok(elapsed)
}

/// The plain fsync() system call, without F_FULLFSYNC.
fn plain_fsync(file: &File) -> std::io::Result<()> {
    // SAFETY: fsync only reads the descriptor number of a file that is open.
    if unsafe { libc::fsync(file.as_raw_fd()) } == 0 {
        Ok(())
    } else {
        Err(std::io::Error::last_os_error())
    }
}

fn main() -> std::io::Result<()> {
    let cases: [(&str, Option<usize>, SyncFn); 4] = [
        ("no sync", None, File::sync_data),
        ("sync_data every record", Some(1), File::sync_data),
        ("sync_data every 100", Some(100), File::sync_data),
        ("plain fsync every record", Some(1), plain_fsync),
    ];
    for (name, every, sync) in cases {
        println!("{name:<26} {:?}", run(every, sync)?);
    }
    Ok(())
}
```

**What happens:**

1. `type SyncFn = fn(&File) -> …` gives a short name to a *function pointer* type: any function that takes a `&File` and
   returns `io::Result<()>`. `File::sync_data` and our `plain_fsync` both fit, so `run` can take either.
2. `every.is_some_and(|n| i % n == 0)` is true when a sync interval is given and record `i` is a multiple of it.
3. `plain_fsync` calls the C function `fsync` directly through `libc`, to compare the weak macOS `fsync` with Rust's
   `F_FULLFSYNC`. On failure, `last_os_error` turns the C error number into an `io::Error`.
4. Each case uses a fresh file, so the cases do not disturb each other.

`cargo run --release --bin fsync_cost` on the Mac:

```text
no sync                    2.482958ms
sync_data every record     5.211412959s
sync_data every 100        46.045ms
plain fsync every record   44.335167ms
```

Read this table slowly, because it explains a lot about databases:

- Without sync, 1,000 appends take 2.5 ms: they only go to the page cache.
- With a real sync after every record, they take **5.2 seconds**: about 5 ms per `F_FULLFSYNC`, two thousand times slower.
  A program that does this can save only about 200 records per second.
- Syncing once per 100 records costs 10 syncs instead of 1,000, so it is about 100 times faster.
- Plain `fsync` on macOS is a hundred times faster than `F_FULLFSYNC`, because it does not wait for the drive's cache. That
  speed is exactly what you give up for real durability.

A storage engine cannot simply sync less: each acknowledged write must be durable. The trick is **group commit**. When
many clients write at the same moment, the engine appends all their records, calls `fsync` **once**, and then acknowledges
all of them:

```text
client A ─ write ─┐
client B ─ write ─┼─► append A, B, C ─► one fsync (5 ms) ─► "saved" to A, B and C
client C ─ write ─┘
```

Each client still waits for a full sync, but the engine does 100 times more work per sync. PostgreSQL, SQLite and the
storage engines you will build use this idea. Exercise 3 builds a small version of the policy behind it.

> ⚡ **Performance:** the speed of a durable write is set by the sync, not by the bytes. Measure with syncs, or your
> benchmark measures RAM.

## 5. When `fsync` fails

`sync_all` returns a `Result`, and it can be an error: the device has a hardware fault, the space ran out, or a network
file system lost its connection. What should a program do then?

The natural idea is to try again. That is wrong. In 2018 the PostgreSQL developers found (in a discussion now called
"fsyncgate") that on Linux, after a failed `fsync`, the kernel may mark the dirty pages as clean, or even throw them away.
A second `fsync` then returns success, although the data never reached the device. The error is reported only once.

Since then, PostgreSQL stops (it *panics*) when `fsync` fails, and on restart it rebuilds its state from its write-ahead
log, which you meet in Lesson 17.4. The rule for your own code, in `src/bin/save.rs`:

```rust
use std::fs::File;
use std::io::Write;

fn save(file: &mut File, record: &[u8]) -> std::io::Result<()> {
    file.write_all(record)?;
    file.sync_data()?; // on error: return it; never loop and retry
    Ok(())
}

fn main() {
    let mut file = File::create("save.log").expect("cannot create save.log");
    if let Err(e) = save(&mut file, b"put a=1\n") {
        // We do not know what is on the disk now. Stop, and let recovery decide on restart.
        eprintln!("fatal: could not make the write durable: {e}");
        std::process::exit(1);
    }
    println!("saved");
    std::fs::remove_file("save.log").expect("cannot remove save.log");
}
```

**What happens:**

1. `save` returns the error of `sync_data` with `?` instead of hiding it. It never claims success.
2. `main` treats the error as fatal: it does not acknowledge anything and exits. On the next start, recovery reads the
   file and finds out what really is there.
3. When everything works, the program prints:

```text
saved
```

Also remember from Lesson 17.1 that dropping a `File` or a `BufWriter` ignores errors. Durable code always ends with an
explicit `flush` and sync whose `Result` it checks.

## 6. Torn writes, and a crash you can test

A write of 100 bytes is not one step for the hardware. If the power goes off in the middle, the device may keep the first
part and lose the rest: a **torn write**. Devices promise at most that a single sector (512 bytes or 4 KiB, kibibytes of 1,024 bytes) is not torn,
and many promise not even that. And, as Apple's manual says, without a full flush the drive may even write later data
before earlier data.

So after a crash, the end of a log file may contain half a record. The code that reads the file on restart must
recognise it. Lesson 17.3 gives you the tools (length prefixes and checksums). First, let us make such a file on purpose.

You cannot pull the power cord in a test, but you can simulate the effect with **fault injection**: a writer that
"crashes" after a given number of bytes. Create `src/crash.rs`:

```rust
use std::io::{self, Write};

/// A writer that pretends the machine crashes after `budget` bytes.
/// It passes at most `budget` bytes to `inner`; after that every write fails.
pub struct CrashAfter<W> {
    inner: W,
    budget: usize,
}

impl<W: Write> CrashAfter<W> {
    pub fn new(inner: W, budget: usize) -> Self {
        CrashAfter { inner, budget }
    }

    pub fn into_inner(self) -> W {
        self.inner
    }
}

impl<W: Write> Write for CrashAfter<W> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        if self.budget == 0 {
            return Err(io::Error::other("simulated crash"));
        }
        let n = buf.len().min(self.budget);
        let written = self.inner.write(&buf[..n])?;
        self.budget -= written;
        Ok(written)
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn crash_in_the_middle_of_a_record_leaves_half_of_it() {
        let mut disk = CrashAfter::new(Vec::new(), 20);
        disk.write_all(b"put a=1\n").unwrap(); // 8 bytes
        disk.write_all(b"put b=2\n").unwrap(); // 16 bytes
        let result = disk.write_all(b"put c=3\n"); // crash after 4 more bytes
        assert!(result.is_err());
        assert_eq!(disk.into_inner(), b"put a=1\nput b=2\nput ");
    }
}
```

and declare it by replacing the content of `src/lib.rs` (the exercise of Lesson 17.1 is done; from now on `lib.rs` lists the
modules of the phase):

```rust
pub mod crash;
```

**What happens:**

1. `CrashAfter<W>` wraps any writer: a `Vec<u8>` in tests (an "in-memory disk"), or a real `File`.
2. `write` may write fewer bytes than asked; that is allowed by the `Write` trait. `write_all` keeps calling `write`
   until everything is written or an error comes. When the budget is used up, `write` returns an error: the "crash".
3. `io::Error::other` creates an error of kind `Other` with a message.
4. The test writes two complete records and starts a third. After the "crash", the disk holds the first 20 bytes:
   two good records and the torn start of the third one, `put `.

`cargo test --lib crash`:

```text
running 1 test
test crash::tests::crash_in_the_middle_of_a_record_leaves_half_of_it ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

This writer is a small model of a crash: it shows what the file looks like, so recovery code can be tested against
every possible crash point (Exercise 2). It does not model a drive that reorders writes; that would need a fake disk
that keeps unsynced writes separately and drops some of them. Real projects also run kill tests: start the program as a
child process, kill it with `SIGKILL` at a random moment, restart it and check the data. Stretch goal 2 of the
project in Lesson 17.14 is exactly such a test.

> 🧪 **Testing:** crash safety is tested by making crashes cheap. Wrap your file in a fault-injecting writer, try every
> crash point in a loop, and check one property: after recovery, every acknowledged record is there, and nothing
> half-written is visible.

## 7. A durability checklist

Everything from Lessons 13.12, 17.1 and this lesson in one list:

1. Data in your process (a `Vec`, a `BufWriter`) is lost on any crash. `flush` before you rely on it.
2. Data after `write` survives a process crash, but not an OS crash or power loss.
3. Call `sync_data` (or `sync_all`) before you acknowledge a write. On macOS, Rust's sync calls use `F_FULLFSYNC`.
4. A new file, a rename or a delete changes the **directory**: sync the directory too (Lesson 13.12).
5. If a sync fails, do not retry and continue. Stop, and recover on restart.
6. Expect a torn last record after a crash; make the reader detect it (Lessons 17.3 and 17.4).
7. Batch: one sync for many records (group commit).

## Common mistakes

1. **Benchmarking without syncs and then promising durability.** The numbers of "no sync" are not the numbers of a
   database.
2. **Calling plain `libc::fsync` on macOS for durability.** It does not flush the drive's cache. Use `File::sync_data`
   or `File::sync_all`, which use `F_FULLFSYNC`.
3. **Retrying a failed sync.** The second call can return `Ok` for data that is lost. Treat the first error as fatal.
4. **Acknowledging before syncing.** "Saved" must come *after* `sync_data` returns `Ok`, never before or in parallel.
5. **Thinking a process crash test proves power-loss safety.** `kill -9` keeps the page cache, so such a test passes even
   with no sync at all. It checks your recovery code, not your durability.

## Try it yourself

1. Run `fsync_cost` twice. Compute the time of one `F_FULLFSYNC` and of one plain `fsync` from your numbers.
2. Change `Some(100)` to `Some(10)` and predict the new time before you run it.
3. In `crash.rs`, call `out.flush()?` before `abort()` in the buffered branch. Run it again: the lines now survive. Why?

## Exercises

### Exercise 1 — Which data survives? (easy)

A program appends a record with `write_all` on a `File`, does **not** call `sync_data`, and prints "saved". For each event,
say whether the record survives: (a) the program panics a moment later and is aborted; (b) somebody runs `kill -9` on it;
(c) the laptop battery dies; (d) the program calls `sync_data`, gets `Ok`, and then the battery dies.

<details>
<summary>Hint</summary>

Use the table in Section 1. After `write`, the record is in the page cache.

</details>

<details>
<summary>Answer</summary>

(a) Survives: a process crash keeps the page cache. (b) Survives, for the same reason. (c) Probably lost: the page cache
is RAM, and without a sync the kernel may not have written it yet. (d) Survives: `sync_data` returned `Ok`, so the data
and the file size are on permanent storage (on macOS through `F_FULLFSYNC`).

</details>

### Exercise 2 — Test every crash point (medium)

Add a test to `src/crash.rs` that writes the 24 bytes `b"put a=1\nput b=2\nput c=3\n"` through `CrashAfter` with every
budget from 0 to 24. For each budget, check that the disk holds exactly `budget` bytes and that they are the start of the
data.

<details>
<summary>Hint</summary>

`write_all` returns an error when the budget is too small; ignore it with `let _ = …`. A slice has the method
`starts_with`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn every_crash_point_leaves_a_prefix() {
        let data = b"put a=1\nput b=2\nput c=3\n";
        for budget in 0..=data.len() {
            let mut disk = CrashAfter::new(Vec::new(), budget);
            let _ = disk.write_all(data); // may fail: that is the crash
            let left = disk.into_inner();
            assert_eq!(left.len(), budget);
            assert!(data.starts_with(&left), "budget {budget}: {left:?}");
        }
    }
```

`cargo test --lib crash` now shows `2 passed`. Why it works: an append-only file that crashes during a write always holds
a **prefix** of what was written. That is the property recovery code relies on, and the test checks it for all 25 crash
points, not just one.

</details>

### Exercise 3 — A sync policy (hard)

Create `src/synclog.rs` (and `pub mod synclog;` in `src/lib.rs`) with:

- `pub enum SyncPolicy { Always, EveryN(usize), Never }`;
- `pub struct SyncLog` that owns a `File`, a policy, a counter of unsynced records and a public counter `syncs`;
- `append(&mut self, record: &[u8])` that writes the record and syncs when the policy says so;
- `sync(&mut self)` that calls `sync_data` only if there are unsynced records.

Test: with `EveryN(10)`, 25 appends make 2 syncs; one `sync()` call makes it 3; another `sync()` call keeps it at 3.
(Testing `Always` and `Never` the same way is a good extra exercise.)

<details>
<summary>Hint</summary>

In `append`, compute `due` with a `match` on the policy: `Always => true`, `EveryN(n) => self.unsynced >= n`,
`Never => false`. Create the test file in `std::env::temp_dir()` and delete it at the end.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs::File;
use std::io::{self, Write};

/// When the log calls `sync_data`.
#[derive(Debug, Clone, Copy)]
pub enum SyncPolicy {
    /// After every record: slow, but nothing acknowledged is ever lost.
    Always,
    /// After every `n` records: up to `n - 1` acknowledged records can be lost.
    EveryN(usize),
    /// Never: fast, everything since the last OS flush can be lost.
    Never,
}

pub struct SyncLog {
    file: File,
    policy: SyncPolicy,
    unsynced: usize,
    pub syncs: usize,
}

impl SyncLog {
    pub fn new(file: File, policy: SyncPolicy) -> Self {
        SyncLog {
            file,
            policy,
            unsynced: 0,
            syncs: 0,
        }
    }

    pub fn append(&mut self, record: &[u8]) -> io::Result<()> {
        self.file.write_all(record)?;
        self.unsynced += 1;
        let due = match self.policy {
            SyncPolicy::Always => true,
            SyncPolicy::EveryN(n) => self.unsynced >= n,
            SyncPolicy::Never => false,
        };
        if due {
            self.sync()?;
        }
        Ok(())
    }

    /// Forces everything written so far to the device.
    pub fn sync(&mut self) -> io::Result<()> {
        if self.unsynced > 0 {
            self.file.sync_data()?;
            self.syncs += 1;
            self.unsynced = 0;
        }
        Ok(())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn log_in_temp(name: &str, policy: SyncPolicy) -> (SyncLog, std::path::PathBuf) {
        let path = std::env::temp_dir().join(name);
        let file = File::create(&path).unwrap();
        (SyncLog::new(file, policy), path)
    }

    #[test]
    fn every_n_syncs_once_per_n_records() {
        let (mut log, path) = log_in_temp("p17-every-n.log", SyncPolicy::EveryN(10));
        for _ in 0..25 {
            log.append(b"record\n").unwrap();
        }
        assert_eq!(log.syncs, 2); // after record 10 and record 20
        log.sync().unwrap(); // the last 5 records
        assert_eq!(log.syncs, 3);
        log.sync().unwrap(); // nothing new: no extra sync
        assert_eq!(log.syncs, 3);
        assert_eq!(std::fs::read(&path).unwrap().len(), 25 * 7);
        std::fs::remove_file(path).unwrap();
    }
}
```

`cargo test --lib` (with the tests of `crash.rs`):

```text
running 3 tests
test crash::tests::crash_in_the_middle_of_a_record_leaves_half_of_it ... ok
test crash::tests::every_crash_point_leaves_a_prefix ... ok
test synclog::tests::every_n_syncs_once_per_n_records ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

Why it works: the counter `unsynced` remembers how many records are not yet durable, and `sync` resets it. `EveryN` is the
policy behind group commit; the comments on each variant state honestly what a crash can lose. A real engine would
acknowledge a record only after the sync that covers it.

</details>

## Quiz

1. A process is killed with `kill -9` right after `write` returned. Is the data lost?

<details>
<summary>Answer</summary>

No. After `write` the data is in the kernel's page cache, which survives a process crash. The kernel writes it to the
device later.

</details>

2. What does `File::sync_data` call on Linux, and what on macOS?

<details>
<summary>Answer</summary>

On Linux it calls `fdatasync`. On macOS (all Apple systems) it calls `fcntl(fd, F_FULLFSYNC)`, the same as `sync_all`.

</details>

3. Why is plain `fsync` on macOS not enough for a database?

<details>
<summary>Answer</summary>

It only moves the data to the drive. The drive may keep it in its volatile cache and write it later or in another order,
so a power loss can still lose it. `F_FULLFSYNC` asks the drive to write its cache to permanent storage.

</details>

4. A sync takes 5 ms. About how many durable writes per second can a program do if it syncs after each write, and how many
   with group commit of 100 records per sync?

<details>
<summary>Answer</summary>

About 200 per second with one sync per write, about 20,000 per second with 100 records per sync (ignoring the cost of the
writes themselves).

</details>

5. `sync_data` returned an error. Your colleague suggests calling it again until it succeeds. What is wrong with that?

<details>
<summary>Answer</summary>

After a failed sync, the kernel may have dropped the dirty pages or marked them clean. A second call can return `Ok` although
the data never reached the device. The only safe reaction is to stop and recover from the log on restart.

</details>

6. What is a torn write, and what does the end of an append-only file look like after one?

<details>
<summary>Answer</summary>

A write interrupted by a crash, so that only its first part reached the storage. The file then ends with a prefix of the last
record: complete earlier records followed by a half record.

</details>

## Summary

- A process crash loses only your own buffers; an OS crash also loses the page cache; a power loss can also lose the drive's
  write cache.
- `sync_all` / `sync_data` are `fsync` / `fdatasync` on Linux. On macOS, Rust calls `fcntl(F_FULLFSYNC)` for both, because
  plain `fsync` there does not flush the drive.
- A real sync costs milliseconds (about 5 ms here), thousands of times more than a buffered write.
- Group commit makes many writes durable with one sync, so durability becomes affordable.
- Never retry a failed sync; stop and recover on restart.
- After a crash, an append-only file holds a prefix of what was written, possibly ending with a torn record.
- A fault-injecting writer such as `CrashAfter` lets ordinary tests try every crash point.

## Sources and further reading

- [std::fs::File](https://doc.rust-lang.org/std/fs/struct.File.html) — `sync_all`, `sync_data`, and the note that errors on
  drop are ignored.
- [Rust standard library source: library/std/src/sys/fs/unix.rs](https://github.com/rust-lang/rust/blob/main/library/std/src/sys/fs/unix.rs)
  — search for `F_FULLFSYNC` to see how `fsync` and `datasync` are implemented on Apple systems and on Linux.
- [fsync(2) — Linux manual page](https://man7.org/linux/man-pages/man2/fsync.2.html) — `fsync` and `fdatasync`, including
  the disk cache flush.
- [fsync(2) — macOS manual page (Apple)](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/fsync.2.html)
  — why `fsync` alone does not reach permanent storage, and what `F_FULLFSYNC` does.
- [PostgreSQL wiki — Fsync Errors](https://wiki.postgresql.org/wiki/Fsync_Errors) — the history of "fsyncgate 2018" and how
  each operating system handles `fsync` errors.
- [Operating Systems: Three Easy Pieces — Crash Consistency: FSCK and Journaling](https://pages.cs.wisc.edu/~remzi/OSTEP/file-journaling.pdf)
  — how file systems themselves survive crashes, the background for the WAL (write-ahead log) in Lesson 17.4.

---

[🏠 Start](../START-HERE.md) · [← 17.1 How disks and files work](01-how-disks-and-files-work.md) · [📋 Phase 17](README.md) · [17.3 Binary file formats and serialization →](03-binary-file-formats.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
