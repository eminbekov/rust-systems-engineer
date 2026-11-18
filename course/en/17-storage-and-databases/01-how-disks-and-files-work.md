<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 17 — Storage and database internals](README.md) · [📋 Phase 17](README.md) · [17.2 Durability: fsync and crash safety →](02-durability-and-fsync.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/01-how-disks-and-files-work.md) <!-- nav -->

# Lesson 17.1 — How disks and files work

> **Phase 17 — Storage and database internals** · Lesson 1 of 15 · ⏱ about 120 minutes

## What you will learn

- The layers between your `write` call and the storage device, and where data waits on the way.
- How a hard disk and an SSD store data, and why both work with whole blocks instead of single bytes.
- What latency and throughput are, and why *random* access is much slower than *sequential* access.
- How to measure a real SSD from Rust without the page cache hiding it.
- Why storage engines append to logs, use fixed-size pages and write in large pieces.

## Before you start

- You know the page cache from [Lesson 13.11 — Memory-mapped files](../13-operating-systems/11-memory-mapped-files.md) and
  inodes and `fsync` from [Lesson 13.12 — File systems from the inside](../13-operating-systems/12-file-systems-inside.md).
- You know file descriptors and `read`/`write` ([Lesson 13.4](../13-operating-systems/04-files-and-file-descriptors.md)),
  system calls ([Lesson 13.3](../13-operating-systems/03-system-calls.md)) and the memory hierarchy
  ([Lesson 12.9](../12-computer-systems/09-memory-hierarchy-and-caches.md)).
- Create the package for the whole phase. It is a library (for the code that later lessons grow into a small
  storage engine) with a `src/bin/` folder (one small program per example):

```bash
cd ~/projects/rust
cargo new --lib code/p17-storage
cd code/p17-storage
mkdir src/bin
cargo add libc@0.2
```

Every example of this phase runs on your Mac. When a detail is different on Linux, the lesson says so. Run the
measuring programs with `--release`: measurements of a debug build tell you little.

## New words

| Word | Meaning |
|---|---|
| **storage device** | Hardware that keeps data when the power is off: a hard disk or an SSD. |
| **HDD** (Hard Disk Drive) | A storage device with spinning magnetic platters and a moving arm that reads and writes them. |
| **SSD** (Solid-State Drive) | A storage device made of flash memory chips, with no moving parts. Your Mac has one. |
| **NVMe** (Non-Volatile Memory Express) | The fast way modern SSDs talk to the computer, over PCIe (Peripheral Component Interconnect Express), the internal high-speed bus. "Non-volatile" means "keeps data without power". |
| **sector / block** | The smallest unit a device reads or writes: usually 512 bytes or 4 KiB. A file system also uses blocks, usually 4 KiB. |
| **flash page / erase block** | Inside an SSD, data is written in flash pages (a few KiB) but can only be erased in much larger erase blocks (many pages). |
| **FTL** (Flash Translation Layer) | Software inside the SSD that maps the block numbers the computer uses to real places in the flash chips. |
| **latency** | How long one operation takes, from request to answer. Measured in microseconds (µs) or milliseconds (ms). |
| **throughput** | How much data per second moves, for example 2,000 MiB/s (mebibytes per second). |
| **IOPS** (Input/Output Operations Per Second) | How many separate reads or writes a device finishes per second. |
| **sequential / random access** | Reading or writing neighbouring blocks one after another / jumping to blocks all over the device. |
| **read-modify-write** | To change a part of a block, the system must read the whole block, change it in memory and write the whole block back. |
| **write amplification** | When the device writes more bytes than the program asked for, for example a whole 4 KiB block for one changed byte. |

## 1. The road from `write` to the device

When your program calls `write`, the bytes do not fly straight to the SSD. They pass several layers, and in
most of them they can wait:

```text
 your program      Vec<u8>, String        ← your memory
      │
 BufWriter         8 KiB buffer           ← still your memory (Rust std)
      │  write() system call
 kernel            page cache             ← RAM owned by the kernel
      │  later, or on fsync
 file system       blocks, inodes, journal
      │
 device driver     NVMe commands
      │
 SSD controller    small RAM write cache  ← inside the device!
      │
 flash chips       data survives power loss
```

You know the middle part already. From Lesson 13.11, `write` returns as soon as the data is in the page cache.
From Lesson 13.12, the kernel writes "dirty" pages to the device later, and `fsync` waits until they are there.

Two layers are new. At the top, a `BufWriter` (from Rust's standard library) collects small writes in your own
memory, so that the program makes fewer system calls. At the bottom, the device itself has a small cache. The
next lesson shows what this means for durability. This lesson is about **speed**: every layer exists because the
layer below it is slow.

The first layer is easy to measure. Put this into `src/bin/small_writes.rs`:

```rust
use std::fs::File;
use std::io::{BufWriter, Write};
use std::time::Instant;

const COUNT: usize = 100_000;
const RECORD: &[u8] = b"key=value;12345\n"; // 16 bytes

fn main() -> std::io::Result<()> {
    // 1. One write system call per record.
    let start = Instant::now();
    let mut file = File::create("direct.log")?;
    for _ in 0..COUNT {
        file.write_all(RECORD)?;
    }
    println!("unbuffered: {:?}", start.elapsed());

    // 2. BufWriter collects records in an 8 KiB buffer and writes it in big pieces.
    let start = Instant::now();
    let mut file = BufWriter::new(File::create("buffered.log")?);
    for _ in 0..COUNT {
        file.write_all(RECORD)?;
    }
    file.flush()?;
    println!("buffered:   {:?}", start.elapsed());

    std::fs::remove_file("direct.log")?;
    std::fs::remove_file("buffered.log")?;
    Ok(())
}
```

**What happens:**

1. `File` has no buffer of its own: every `write_all` on it becomes one `write` system call, 100,000 in total.
2. `BufWriter::new` wraps the file in an 8 KiB buffer. `write_all` copies the 16 bytes into the buffer; only when
   the buffer is full does it make one system call with 8 KiB. That is about 200 system calls instead of 100,000.
3. `flush` writes what is left in the buffer. Without it, the last part would be written only when the
   `BufWriter` is dropped, and an error at that moment would be ignored.
4. Both versions only reach the page cache. No data has to be on the SSD yet.

Run it with `cargo run --release --bin small_writes`:

```text
unbuffered: 470.717958ms
buffered:   1.537375ms
```

The same 1.6 MB (megabytes) of data, about 300 times faster. Each system call costs about 4–5 µs here (the kernel must check the
descriptor, find the pages in the page cache and copy), and the buffered version pays that cost 500 times less often.
Your numbers will be different, but the ratio will be similar.

> ⚡ **Performance:** a system call is cheap compared with a disk, but expensive compared with copying 16 bytes.
> When a program writes many small pieces, put a `BufWriter` around the file. Storage engines go further and
> build whole blocks in memory before they write anything.

## 2. Two kinds of devices

**A hard disk (HDD)** is like a record player. Data lies in circles (tracks) on spinning platters, and an arm
with a read/write head moves over them. To read a block, the arm must first move to the right track (a *seek*)
and then wait until the platter turns the block under the head. Both are mechanical, so they take milliseconds.
After that, reading the next blocks on the same track is fast, because they simply pass under the head.

**An SSD** has no moving parts. It stores bits as electric charge in flash memory chips. Any block can be reached
in tens of microseconds. But flash has two strange rules:

1. You can write a **flash page** (a few KiB) only when it is empty. You cannot overwrite it.
2. You can empty pages only in large groups, an **erase block** (often several MiB), and each block survives only a
   limited number of erases.

So an SSD never changes data in place. When you overwrite block 1000, the **FTL** (Flash Translation Layer), a
small program inside the SSD, writes the new data to some empty page and remembers "block 1000 now lives here".
The old page becomes garbage. Later, the SSD copies the still-valid pages out of a mostly-garbage erase block and
erases it. This copying is a kind of **write amplification**: the device writes more than you asked for.

```text
                        HDD                          SSD (NVMe)
smallest unit           sector, 512 B or 4 KiB       flash page, 4–16 KiB
random 4 KiB read       5–10 ms (seek + rotation)    20–200 µs
sequential throughput   100–250 MB/s                 2,000–7,000 MB/s
overwrite in place      yes                          no (FTL writes elsewhere)
```

These are typical values, not exact ones. The important part is the shape: an HDD is about 100 times faster
sequentially than randomly, and an SSD is still several times faster sequentially, as you will measure in Section 4.

## 3. Everything happens in blocks

Neither device can store one byte. The device works in **sectors** (512 bytes or 4 KiB), and the file system on top
of it works in **blocks**, usually 4 KiB. A file always uses whole blocks. Put this into `src/bin/blocks.rs`:

```rust
use std::fs;
use std::os::unix::fs::MetadataExt;

fn main() -> std::io::Result<()> {
    fs::write("one-byte.txt", b"x")?;
    fs::write("five-thousand.txt", vec![b'a'; 5000])?;

    for name in ["one-byte.txt", "five-thousand.txt"] {
        let meta = fs::metadata(name)?;
        println!(
            "{name}: len = {} bytes, on disk = {} bytes, block size = {}",
            meta.len(),
            meta.blocks() * 512,
            meta.blksize()
        );
    }

    fs::remove_file("one-byte.txt")?;
    fs::remove_file("five-thousand.txt")?;
    Ok(())
}
```

**What happens:**

1. `MetadataExt` (Unix only) gives the raw fields of `stat`, which you met in Lesson 13.12.
2. `blocks()` counts 512-byte units, whatever the real block size is, so we multiply by 512.
3. `blksize()` is the block size that the file system recommends for efficient reads and writes.

`cargo run --release --bin blocks` on the Mac (file system APFS, Apple File System):

```text
one-byte.txt: len = 1 bytes, on disk = 4096 bytes, block size = 4096
five-thousand.txt: len = 5000 bytes, on disk = 8192 bytes, block size = 4096
```

One byte takes a whole 4 KiB block; 5,000 bytes take two blocks. An ext4 file system on Linux gives the same picture.

Blocks also explain an expensive pattern. If a program changes 10 bytes in the middle of a file, the kernel must
have the whole 4 KiB page in the page cache. If the page is not there, it first **reads** the page from the device,
then **modifies** 10 bytes in memory, and later **writes** the whole page back. This is **read-modify-write**:

```text
want: change bytes 5000..5010

  device block 1 (bytes 4096..8191)
  ┌──────────────────────────────┐
  │ read all 4096 bytes  ────────┼──► page cache: change 10 bytes
  │ write all 4096 bytes ◄───────┼─── (later, or on fsync)
  └──────────────────────────────┘
```

A database that changes small records in place pays this price all the time. That is why the designs in this phase
either append (never change old bytes) or keep data in pages whose size equals the block size.

## 4. Sequential and random reads, measured

Measuring a device is tricky, because the page cache answers from RAM (Random Access Memory, the computer's main memory) whenever it can. A file that you have just
written is still in the cache, and reading it says nothing about the SSD. macOS has a flag for one file descriptor,
`F_NOCACHE`, that tells the kernel not to keep the file's data in the page cache. Rust's standard library has no
method for it, so we call the C function `fcntl` (file control) from the `libc` crate. Put this into
`src/bin/seq_vs_random.rs`:

```rust
use std::fs::{self, File};
use std::io::Write;
use std::os::unix::fs::FileExt;
use std::time::Instant;

const FILE_SIZE: u64 = 256 * 1024 * 1024; // 256 MiB
const BLOCK: usize = 4096; // read 4 KiB at a time
const READS: u64 = 4096; // 4096 reads x 4 KiB = 16 MiB

/// A tiny pseudo-random number generator (xorshift). Good enough to pick offsets.
fn next_random(state: &mut u64) -> u64 {
    *state ^= *state << 13;
    *state ^= *state >> 7;
    *state ^= *state << 17;
    *state
}

/// Asks macOS not to keep this file's data in the page cache, so that reads go to the SSD.
#[cfg(target_os = "macos")]
fn bypass_page_cache(file: &File) {
    use std::os::fd::AsRawFd;
    // SAFETY: fcntl with F_NOCACHE only sets a flag on a descriptor that we own and that is open.
    let result = unsafe { libc::fcntl(file.as_raw_fd(), libc::F_NOCACHE, 1) };
    assert_ne!(result, -1, "F_NOCACHE failed");
}

#[cfg(not(target_os = "macos"))]
fn bypass_page_cache(_file: &File) {
    eprintln!("note: page cache not bypassed on this system; reads may come from RAM");
}

fn main() -> std::io::Result<()> {
    // 1. Create the test file once: 256 MiB of bytes.
    let mut out = File::create("big.bin")?;
    bypass_page_cache(&out);
    let chunk = vec![7u8; 1024 * 1024];
    for _ in 0..FILE_SIZE / chunk.len() as u64 {
        out.write_all(&chunk)?;
    }
    out.sync_all()?;
    drop(out);

    let file = File::open("big.bin")?;
    bypass_page_cache(&file);
    let mut buf = vec![0u8; BLOCK];
    let blocks_in_file = FILE_SIZE / BLOCK as u64;

    // 2. Sequential: block 0, 1, 2, ...
    let start = Instant::now();
    for i in 0..READS {
        file.read_exact_at(&mut buf, i * BLOCK as u64)?;
    }
    let seq = start.elapsed();

    // 3. Random: blocks in a random order all over the file.
    let mut state = 42;
    let start = Instant::now();
    for _ in 0..READS {
        let block = next_random(&mut state) % blocks_in_file;
        file.read_exact_at(&mut buf, block * BLOCK as u64)?;
    }
    let random = start.elapsed();

    println!("sequential: {READS} reads in {seq:?} ({:?} per read)", seq / READS as u32);
    println!("random:     {READS} reads in {random:?} ({:?} per read)", random / READS as u32);
    fs::remove_file("big.bin")?;
    Ok(())
}
```

**What happens:**

1. `next_random` is *xorshift*, a classic three-line generator. It is not good enough for security, but it spreads
   offsets well and gives the same sequence every run, so runs can be compared. No crate is needed.
2. `#[cfg(target_os = "macos")]` compiles the first `bypass_page_cache` only on macOS; on other systems the second
   version is compiled, which only prints a note. On Linux you would use `O_DIRECT`, which needs specially aligned
   buffers; that is outside this lesson.
3. The `unsafe` block is needed because `fcntl` is a C function: the compiler cannot check it. Our promise (written
   in the `SAFETY:` comment) is that the descriptor is open and that this `fcntl` command does not touch memory.
4. We set `F_NOCACHE` already **while writing** the file. Otherwise its pages would stay in the cache from the write,
   and the reads would still come from RAM.
5. `read_exact_at` (from `FileExt`) reads at a given offset without moving the file's offset: it is the system call
   `pread`. It is handy when you jump around a file.
6. Sequential reads use blocks 0, 1, 2, … Random reads pick 4,096 random blocks from all 65,536.

`cargo run --release --bin seq_vs_random` (it writes and deletes a 256 MiB file):

```text
sequential: 4096 reads in 182.11775ms (44.462µs per read)
random:     4096 reads in 502.914791ms (122.781µs per read)
```

A second run on the same Mac gave 57 µs and 223 µs. Measurements of a device are noisy, because other programs use it
too. But the result is clear: on an SSD a random read costs about **three times** a sequential one. The SSD and the
operating system notice the sequential pattern and read ahead; random reads cannot be predicted. On an HDD the
difference would be closer to a hundred times, because every random read needs a seek.

The size of each read matters even more. The program `src/bin/block_sizes.rs` in Exercise 3 reads 32 MiB randomly in
pieces of different sizes. One run gave:

```text
random reads of    4096 bytes:     11 MiB/s
random reads of   16384 bytes:    100 MiB/s
random reads of   65536 bytes:    359 MiB/s
random reads of 1048576 bytes:   2341 MiB/s
```

Every read has a fixed cost (a system call, a command to the device, waiting for the flash), so reading 1 MiB at once
is more than a hundred times faster per byte than reading it in 4 KiB pieces.

> ⚠️ **Common mistake:** measuring a file that is still in the page cache. If you remove both `bypass_page_cache`
> calls, every read takes less than 1 µs, and "random" looks almost as fast as "sequential". That is the speed of RAM, not of
> the SSD (Exercise 2).

## 5. What storage engines learn from this

Everything later in this phase follows from four facts you have now seen:

| Fact | What storage engines do |
|---|---|
| System calls cost microseconds | Collect many small changes in memory, then write them in one call. |
| Devices work in blocks | Store data in **pages** of 4, 8 or 16 KiB and read or write whole pages (B-trees, Lesson 17.6). |
| Random access is slow, sequential is fast | **Append** to the end of a file instead of changing old bytes (the WAL, write-ahead log, in Lesson 17.4, logs in Lessons 17.5 and 17.7). |
| Big reads and writes are cheaper per byte | Write large sorted files in one go, and merge them in large pieces (Lessons 17.7 and 17.8). |

There is one more fact, the most important one, and you cannot measure it with speed: data in any cache can be lost.
Lesson 17.2 is about that.

## Common mistakes

1. **Benchmarking a debug build.** `cargo run` without `--release` builds code without optimisation. For I/O
   (Input/Output) programs, the extra time is mostly in your code, not in the device, so the numbers lie.
2. **Measuring the cache, not the device.** See the warning in Section 4. A file you have just written or read is in
   the page cache.
3. **Forgetting `flush` on a `BufWriter`.** The last bytes stay in the buffer until the `BufWriter` is dropped, and an
   error during that drop is ignored. Call `flush()` (or `into_inner()`) and handle its `Result`.
4. **Trusting one run.** Other programs use the same SSD. Run a measurement several times and look at the range, as
   in [Lesson 09.11 — Benchmarking in depth](../09-testing-debugging-benchmarking/11-benchmarking-in-depth.md).

## Try it yourself

1. Run `small_writes` three times. Then change `COUNT` to `10_000` and check that the times shrink about ten times.
2. In `blocks.rs`, add a third file of exactly 4096 bytes. Predict its size on disk before you run the program.
3. Run `seq_vs_random` twice and write both results down. Which one changes more between runs, and why do you think
   that is?

## Exercises

### Exercise 1 — How many pages does a write touch? (easy)

Write `pub fn pages_touched(offset: u64, len: u64, page_size: u64) -> u64` in `src/lib.rs`. It returns how many pages
of `page_size` bytes a write of `len` bytes at `offset` touches (each of them is a possible read-modify-write).
Write a test with at least these cases: a write of 4096 bytes at offset 0 touches 1 page; 2 bytes at offset 4095
touch 2 pages; 10,000 bytes at offset 100 touch 3 pages; 0 bytes touch 0 pages.

<details>
<summary>Hint</summary>

Compute the number of the first page (`offset / page_size`) and of the page that holds the *last* byte
(`(offset + len - 1) / page_size`). Treat `len == 0` separately, or `len - 1` underflows.

</details>

<details>
<summary>Answer</summary>

```rust
/// Returns how many pages of `page_size` bytes a write of `len` bytes at `offset` touches.
pub fn pages_touched(offset: u64, len: u64, page_size: u64) -> u64 {
    if len == 0 {
        return 0;
    }
    let first = offset / page_size;
    let last = (offset + len - 1) / page_size;
    last - first + 1
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_pages() {
        assert_eq!(pages_touched(0, 4096, 4096), 1);
        assert_eq!(pages_touched(4095, 2, 4096), 2);
        assert_eq!(pages_touched(100, 10_000, 4096), 3);
        assert_eq!(pages_touched(8192, 1, 4096), 1);
        assert_eq!(pages_touched(5, 0, 4096), 0);
    }
}
```

`cargo test --lib`:

```text
running 1 test
test tests::counts_pages ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the last byte of the write is at `offset + len - 1`. Every page from the first page to the page of
that byte is touched. Bytes 100..10,100 lie in pages 0, 1 and 2.

</details>

### Exercise 2 — Measure the cache instead (medium)

Copy `seq_vs_random.rs` to `src/bin/cached.rs` and remove both calls to `bypass_page_cache` (and the two functions,
so that `cargo clippy` does not warn about dead code). Predict the result, then run it. What do you measure now?

<details>
<summary>Hint</summary>

The file was just written, so all 256 MiB are probably still in the page cache.

</details>

<details>
<summary>Answer</summary>

One run on the test Mac:

```text
sequential: 4096 reads in 2.272916ms (554ns per read)
random:     4096 reads in 3.391375ms (827ns per read)
```

Both are below 1 µs per read, about a hundred times faster than the SSD. Each `read_exact_at` is now a system call plus a 4 KiB copy
from the page cache, which is RAM. The SSD is not used at all. This is why database benchmarks always say whether the
data fitted in memory.

</details>

### Exercise 3 — Bigger reads (hard)

Write `src/bin/block_sizes.rs`: it creates the same 256 MiB file with the cache bypassed, then for each read size
4 KiB, 16 KiB, 64 KiB and 1 MiB it reads 32 MiB at random block-aligned offsets and prints the throughput in MiB/s.
Explain the result.

<details>
<summary>Hint</summary>

Keep `next_random`, both `bypass_page_cache` functions and the code that creates the file. For each size, the number
of reads is `32 MiB / size` and the number of blocks in the file is `FILE_SIZE / size`. Throughput is
`32.0 / seconds`.

</details>

<details>
<summary>Answer</summary>

Replace everything in `main` after `drop(out);` up to `fs::remove_file` with this loop (and delete the constants
`BLOCK` and `READS`):

```rust
    let file = File::open("big.bin")?;
    bypass_page_cache(&file);
    let total: u64 = 32 * 1024 * 1024; // read 32 MiB in every test
    for block in [4096usize, 16 * 1024, 64 * 1024, 1024 * 1024] {
        let mut buf = vec![0u8; block];
        let reads = total / block as u64;
        let blocks_in_file = FILE_SIZE / block as u64;
        let mut state = 42;
        let start = Instant::now();
        for _ in 0..reads {
            let n = next_random(&mut state) % blocks_in_file;
            file.read_exact_at(&mut buf, n * block as u64)?;
        }
        let secs = start.elapsed().as_secs_f64();
        let mib_per_sec = total as f64 / (1024.0 * 1024.0) / secs;
        println!("random reads of {:>7} bytes: {:>6.0} MiB/s", block, mib_per_sec);
    }
```

The output is shown in Section 4 (a second run gave 21, 120, 435 and 1515 MiB/s). Why it works: each read has a
fixed cost of tens of microseconds, whatever its size. With 4 KiB reads you pay that cost 8,192 times for 32 MiB;
with 1 MiB reads only 32 times, and the SSD can read many flash chips in parallel for one large request.

</details>

## Quiz

1. Your program calls `write` and gets `Ok`. Name two places where the data may still be, other than the flash chips.

<details>
<summary>Answer</summary>

In the kernel's page cache (RAM), and in the SSD's own write cache. (If you used a `BufWriter` without `flush`, the
data may even still be in your program's buffer.)

</details>

2. Why can an SSD not overwrite a block in place, and what does it do instead?

<details>
<summary>Answer</summary>

Flash pages can only be written when empty, and emptying happens only for a whole erase block. So the FTL writes the
new data to an empty page, updates its map from block number to physical place, and later cleans up the old pages.

</details>

3. A program changes 1 byte in the middle of a 1 GiB file that is not in the page cache. Roughly how many bytes must be
   read and written?

<details>
<summary>Answer</summary>

One block: 4 KiB is read, 1 byte changed in memory, and 4 KiB written back (read-modify-write). Inside the SSD even
more may be written, because of write amplification.

</details>

4. What is the difference between latency and throughput? Give one example of each from this lesson.

<details>
<summary>Answer</summary>

Latency is the time of one operation, for example about 45 µs for one sequential 4 KiB read. Throughput is data per
second, for example 2341 MiB/s for 1 MiB random reads.

</details>

5. Why did the cached measurement in Exercise 2 show almost no difference between sequential and random reads?

<details>
<summary>Answer</summary>

All the data was in RAM (the page cache), and RAM has no seeks and no flash pages. The time (under 1 µs) is mostly the cost of the
system call and the copy.

</details>

6. Which design idea does the result "random is slow, sequential is fast" lead to?

<details>
<summary>Answer</summary>

Append new data at the end of a file (a log) instead of changing old data in place, and write large pieces at once.

</details>

## Summary

- Data passes several layers on the way to the flash chips: your buffer, the page cache, the file system, the driver
  and the device's own cache. Each layer exists because the next one is slower.
- A `BufWriter` turns many small writes into a few system calls; here that was about 300 times faster.
- Devices and file systems work in blocks (usually 4 KiB). Changing a few bytes costs a read-modify-write of a
  whole block.
- An HDD pays milliseconds for each seek; an SSD never overwrites in place, and its FTL moves data around.
- Random reads were about three times slower than sequential reads on the Mac's SSD, and big reads were much cheaper
  per byte than small ones.
- To measure a device, make sure the page cache is not answering (`F_NOCACHE` on macOS).
- Storage engines therefore batch, append, use pages, and write large sorted pieces.

## Sources and further reading

- [std::io::BufWriter](https://doc.rust-lang.org/std/io/struct.BufWriter.html) — the buffer size, `flush` and why
  errors on drop are ignored.
- [std::os::unix::fs::FileExt](https://doc.rust-lang.org/std/os/unix/fs/trait.FileExt.html) — `read_at`,
  `read_exact_at` and `write_at` (positional reads and writes).
- [std::os::unix::fs::MetadataExt](https://doc.rust-lang.org/std/os/unix/fs/trait.MetadataExt.html) — `blocks` and
  `blksize`.
- [Operating Systems: Three Easy Pieces — Hard Disk Drives](https://pages.cs.wisc.edu/~remzi/OSTEP/file-disks.pdf) —
  how seeks and rotation work, with the maths behind the HDD numbers.
- [Operating Systems: Three Easy Pieces — Flash-based SSDs](https://pages.cs.wisc.edu/~remzi/OSTEP/file-ssd.pdf) —
  flash pages, erase blocks, the FTL and write amplification, explained step by step.
- [libc crate: fcntl](https://docs.rs/libc/latest/libc/fn.fcntl.html) — the binding used for `F_NOCACHE`.

---

[🏠 Start](../START-HERE.md) · [← Phase 17 — Storage and database internals](README.md) · [📋 Phase 17](README.md) · [17.2 Durability: fsync and crash safety →](02-durability-and-fsync.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
