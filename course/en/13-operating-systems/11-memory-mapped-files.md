<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.10 Virtual memory](10-virtual-memory.md) · [📋 Phase 13](README.md) · [13.12 File systems from the inside →](12-file-systems-inside.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/11-memory-mapped-files.md) <!-- nav -->

# Lesson 13.11 — Memory-mapped files

> **Phase 13 — Operating systems from the inside** · Lesson 11 of 18 · ⏱ about 135 minutes

## What you will learn

- What it means to *map* a file into memory, and how this differs from `read` and `write`.
- How the page cache connects files, `read` and mappings.
- How to read and change a file through a mapping with the `memmap2` crate, and why creating a mapping is `unsafe`.
- The difference between shared, private and anonymous mappings.
- When `mmap` is faster than `read`, when it is not, and what SIGBUS means.

## Before you start

- You finished [Lesson 13.10 — Virtual memory](10-virtual-memory.md): pages, page faults, mappings and `/proc/self/maps`.
- You know file descriptors ([Lesson 13.4](04-files-and-file-descriptors.md)), `strace` ([Lesson 13.3](03-system-calls.md)) and slices ([Lesson 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)).
- Create the package for this lesson on the Mac and copy it to the VM `dev` exactly as in Lesson 13.10 (only the name changes):

```bash
cd ~/projects/rust
cargo new code/p13-mmap
cd code/p13-mmap
cargo add memmap2@0.9
rm src/main.rs && mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **memory-mapped file** | A file that the kernel makes visible as a range of memory in a process. Reading that memory reads the file; writing it can change the file. |
| **`mmap` / `munmap`** | The system calls that create and remove a mapping. `mmap` returns the start address of the new range. |
| **page cache** | RAM that the kernel uses to keep recently used file data. Both `read` and mappings get file data from it. |
| **shared mapping** (`MAP_SHARED`) | Changes made through the mapping go into the page cache and later to the file; every process that maps the file sees them. |
| **private mapping** (`MAP_PRIVATE`) | The process gets a copy-on-write view of the file. Its changes stay in its own memory and never reach the file. |
| **anonymous mapping** | A mapping with no file behind it: fresh zeroed memory. Allocators use it for large blocks. |
| **`msync` / flush** | Asking the kernel to write the changed pages of a mapping to the disk now, instead of later. |
| **SIGBUS** (bus error) | The signal a process gets when it touches a mapped page that has no data behind it anymore, for example because the file was shortened. |
| **zero-copy** | Using data where it already is, without copying it into another buffer first. |

## 1. Two ways to get file data

You know the usual way from [Lesson 13.4](04-files-and-file-descriptors.md): `open` the file, then call
`read` with your own buffer. The kernel **copies** bytes into that buffer.

There is a second way. You ask the kernel: "make this file part of my address space". The kernel creates a
**mapping** (Lesson 13.10) whose pages are the pages of the file. After that, the file *is* a slice of
memory. You read it with `data[i]`, without any more system calls.

An analogy: `read` is like asking a librarian to photocopy pages of a book for you. `mmap` is like getting
the book itself on your desk and opening it at any page you want.

Both ways go through the **page cache**: the kernel keeps file data that was recently used in RAM, so that
the second access does not touch the disk.

```text
            disk
              |
              v
   +---------------------+      (kernel memory)
   |     page cache      |  <- pages of the file, 4 KiB each
   +---------------------+
       |              |
  read() copies    mmap() maps the same frames
       |              |
       v              v
  +---------+   +-------------------+   (your process)
  | buffer  |   | mapping: &[u8]    |
  +---------+   +-------------------+
```

With `read`, data exists twice in RAM: in the page cache and in your buffer. With `mmap`, your page table
points directly at the page cache frames. This is called **zero-copy**.

How does the data get there? Through **page faults**. The new mapping starts with an empty page table, just
like the 256 MiB vector of Lesson 13.10. The first touch of each page faults; the kernel finds the page in
the page cache (minor fault) or reads it from the disk first (major fault), and maps it.

## 2. Reading a file through a mapping

Rust's standard library has no `mmap`. The crate `memmap2` wraps the system call in a small, careful API.
Create `src/bin/mcount.rs`:

```rust
use std::env;
use std::fs::File;

use memmap2::Mmap;

fn main() -> std::io::Result<()> {
    let path = env::args().nth(1).expect("usage: mcount <file>");
    let file = File::open(&path)?;

    // SAFETY: we assume no other process changes or shortens the file while it is mapped.
    let map = unsafe { Mmap::map(&file)? };

    let bytes: &[u8] = &map; // the whole file, as a byte slice
    let lines = bytes.iter().filter(|&&b| b == b'\n').count();
    println!("{path}: {} bytes, {lines} lines", bytes.len());
    Ok(())
}
```

**What happens:**

1. `File::open` opens the file for reading and gives us a file descriptor, as usual.
2. `Mmap::map(&file)` calls `mmap` and maps the whole file read-only. It is an `unsafe` function; Section 6
   explains why. The `// SAFETY:` comment states what we promise ([Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)).
3. `Mmap` implements `Deref<Target = [u8]>`, so `&map` is an ordinary `&[u8]`. Every slice method works.
4. When `map` goes out of scope, its `Drop` calls `munmap`. The mapping stays valid even after `file` is
   closed: it holds its own reference to the file.

In the VM:

```bash
printf 'first line\nsecond line\nthird line\n' > small.txt
cargo run --bin mcount -- small.txt
```

```text
small.txt: 34 bytes, 3 lines
```

`strace` shows the system calls behind it (run the built program directly):

```bash
strace -e trace=openat,mmap,munmap,close ./target/debug/mcount small.txt
```

The last lines:

```text
openat(AT_FDCWD, "small.txt", O_RDONLY|O_CLOEXEC) = 3
mmap(NULL, 34, PROT_READ, MAP_SHARED, 3, 0) = 0xffffb68bc000
small.txt: 34 bytes, 3 lines
munmap(0xffffb68bc000, 34)              = 0
close(3)                                = 0
+++ exited with 0 +++
```

Read the `mmap` arguments from left to right: `NULL` = "choose an address for me", `34` = length,
`PROT_READ` = read-only pages, `MAP_SHARED` = a shared mapping of the file, `3` = the file descriptor, `0` =
start at offset 0 of the file. There is no `read` call at all: counting the lines only touched memory.

## 3. Changing a file through a mapping

A writable shared mapping changes the file itself. Create `src/bin/mwrite.rs`:

```rust
use std::fs::OpenOptions;

use memmap2::MmapMut;

fn main() -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("counter.bin")?;
    file.set_len(16)?; // the file must be big enough before we map it

    // SAFETY: only this program uses counter.bin, and we never shorten it while mapped.
    let mut map = unsafe { MmapMut::map_mut(&file)? };

    map[..5].copy_from_slice(b"HELLO");
    map[15] = b'\n';
    map.flush()?; // ask the kernel to write the changed pages to the disk now

    println!("mapped {} bytes, wrote through memory", map.len());
    Ok(())
}
```

**What happens:**

1. The file is opened for reading **and** writing: a writable mapping needs both.
2. `set_len(16)` makes the file 16 bytes long (it calls `ftruncate`). A mapping cannot make a file grow:
   writing past the end of the file is not possible. You must set the size first.
3. `MmapMut::map_mut` creates a `PROT_READ|PROT_WRITE`, `MAP_SHARED` mapping. `MmapMut` dereferences to
   `&mut [u8]`.
4. Writing into the slice changes the pages in the page cache. Every other process that reads the file now
   sees `HELLO`, even before anything reaches the disk.
5. `flush()` calls `msync`, which waits until the changed pages are written to the disk. Without it, the
   kernel writes them later on its own (usually within about 30 seconds) or when the mapping is removed.

```bash
cargo run --bin mwrite
od -c counter.bin
```

```text
mapped 16 bytes, wrote through memory
0000000   H   E   L   L   O  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \n
0000020
```

`od -c` (octal dump, characters) prints every byte; `\0` is a zero byte. The new bytes of the file were
zeros, and we changed six of them.

> ⚡ **Performance:** `flush()` waits for the disk, which can take milliseconds. Call it when the data must
> survive a crash (Lesson 13.12 explains durability), not after every single write.

## 4. Private and anonymous mappings

**Private mappings.** Sometimes you want to change the data in memory but keep the file as it is. A
private mapping gives you exactly that: pages start as the file's pages and are copied on the first write,
the same copy-on-write trick as after `fork`. Create `src/bin/private.rs`:

```rust
use std::fs::{self, File};

use memmap2::MmapOptions;

fn main() -> std::io::Result<()> {
    fs::write("note.txt", "original text\n")?;
    let file = File::open("note.txt")?;

    // SAFETY: only this program uses note.txt.
    let mut copy = unsafe { MmapOptions::new().map_copy(&file)? };
    copy[..8].copy_from_slice(b"CHANGED ");

    println!("in memory: {:?}", String::from_utf8_lossy(&copy));
    println!("on disk  : {:?}", fs::read_to_string("note.txt")?);
    Ok(())
}
```

```text
in memory: "CHANGED  text\n"
on disk  : "original text\n"
```

`map_copy` uses `MAP_PRIVATE`. The file was opened read-only, and still we could write into the mapping:
the written page is our own copy. This is also how the loader maps the global variables of a program
(the `rw-p` lines of `/proc/self/maps`; `p` means private).

**Anonymous mappings.** A mapping without a file is simply new memory filled with zeros. You already saw
one in Lesson 13.10: `vec![0u8; 256 MiB]` became
`mmap(NULL, 268439552, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)`. The `-1` means "no file".
Memory allocators ([Lesson 12.13](../12-computer-systems/13-how-memory-allocators-work.md)) ask the kernel
for big blocks this way. `memmap2` offers it as `MmapMut::map_anon(len)`.

| Kind | Flags | Changes go to the file? | Typical use |
|---|---|---|---|
| shared file mapping | `MAP_SHARED` + file | yes | databases, editing large files, sharing data between processes |
| private file mapping | `MAP_PRIVATE` + file | no (copy-on-write) | loading programs and libraries |
| anonymous mapping | `MAP_ANONYMOUS` | there is no file | allocators, big buffers |

## 5. Is `mmap` faster than `read`?

People often say "mmap is fast". The honest answer is: *sometimes*. Let us measure. Create
`src/bin/compare.rs`:

```rust
use std::env;
use std::fs::File;
use std::io::Read;
use std::time::Instant;

use memmap2::Mmap;

fn count_with_read(path: &str) -> std::io::Result<usize> {
    let mut file = File::open(path)?;
    let mut buf = vec![0u8; 64 * 1024];
    let mut lines = 0;
    loop {
        let n = file.read(&mut buf)?; // the kernel copies up to 64 KiB into buf
        if n == 0 {
            return Ok(lines);
        }
        lines += buf[..n].iter().filter(|&&b| b == b'\n').count();
    }
}

fn count_with_mmap(path: &str) -> std::io::Result<usize> {
    let file = File::open(path)?;
    // SAFETY: nobody changes the file during the measurement.
    let map = unsafe { Mmap::map(&file)? };
    Ok(map.iter().filter(|&&b| b == b'\n').count())
}

fn main() -> std::io::Result<()> {
    let path = env::args().nth(1).expect("usage: compare <file>");
    for round in 1..=3 {
        let t = Instant::now();
        let a = count_with_read(&path)?;
        let read_ms = t.elapsed().as_millis();
        let t = Instant::now();
        let b = count_with_mmap(&path)?;
        let mmap_ms = t.elapsed().as_millis();
        assert_eq!(a, b);
        println!("round {round}: {a} lines | read: {read_ms:>4} ms | mmap: {mmap_ms:>4} ms");
    }
    Ok(())
}
```

Make a file of about 250 MB with `seq` (it prints the numbers from 1 to 30,000,000, one per line) and
measure in release mode:

```bash
seq 1 30000000 > big.txt
cargo run --release --bin compare -- big.txt
```

```text
round 1: 30000000 lines | read:  179 ms | mmap:   98 ms
round 2: 30000000 lines | read:  102 ms | mmap:   88 ms
round 3: 30000000 lines | read:   96 ms | mmap:   91 ms
```

**What happens:**

1. Round 1 is slower for `read`: some file pages were not yet in the page cache. Measuring the first run
   only would mislead you ([Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).
2. In rounds 2 and 3 the difference is small. `read` copies 250 MB, but copying memory that is in the cache
   is fast. `mmap` avoids the copy but pays for about 63,000 page faults instead.
3. Your numbers will differ. Delete `big.txt` when you are done (`rm big.txt`).

Rules of thumb:

- **`mmap` helps** when you jump around in a big file (random access, like a database index), when many
  processes read the same file (they share the page cache frames), or when you want to treat a file as a
  `&[u8]` without managing buffers.
- **`read` is as good or better** for reading a file once from start to end, for small files (setting up a
  mapping has a cost), and for pipes, sockets and terminals, which cannot be mapped at all.
- **`read` is safer:** it reports errors as `Err`, while a disk error inside a mapping arrives as a signal.

## 6. Why `Mmap::map` is `unsafe`: SIGBUS

A `&[u8]` in Rust promises that the bytes do not change while you hold the reference. A mapping cannot
keep that promise by itself: **another process can change or shorten the file at any time**, and the
change appears in your slice. The compiler cannot check other processes, so `memmap2` makes you write
`unsafe` and promise it yourself.

Shortening is the worst case. Pages beyond the new end of the file have no data behind them. Touching
them raises **SIGBUS**, and the process dies. Create `src/bin/sigbus.rs`, which does it on purpose:

```rust
use std::fs::OpenOptions;

use memmap2::Mmap;

fn main() -> std::io::Result<()> {
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .truncate(true)
        .open("shrinking.bin")?;
    file.set_len(8192)?;

    // SAFETY: deliberately broken! We shorten the file below while it is mapped.
    let map = unsafe { Mmap::map(&file)? };
    println!("byte 5000 before: {}", map[5000]);

    file.set_len(0)?; // the file is now empty, but the mapping is still 8192 bytes long
    println!("file shortened, reading byte 5000 again ...");
    println!("byte 5000 after: {}", map[5000]);
    Ok(())
}
```

```bash
cargo run --bin sigbus
echo "exit=$?"
```

```text
byte 5000 before: 0
file shortened, reading byte 5000 again ...
Bus error
exit=135
```

The index `5000` is inside the slice (its length is still 8,192), so Rust's bounds check passes. The crash
comes from the kernel: 135 = 128 + 7, and 7 is SIGBUS. In a real program, "another process" can be a log
rotation tool or a user who runs `truncate` on the file.

> 🔒 **Security:** never map files that other, less trusted users can write to, unless you copy the data
> out first and check it. A hostile writer could change the bytes after you validated them, or shorten the
> file to crash your service.

## Common mistakes

1. **Mapping before setting the size.** Mapping a new, empty file with `MmapMut::map_mut` gives a mapping
   of length 0, and `map[0] = 1` panics with `index out of bounds: the len is 0 but the index is 0`.
   *Fix:* call `file.set_len(n)` first.

2. **Opening the file read-only and mapping it writable.** `MmapMut::map_mut` on a file opened with
   `File::open` fails: `Error: Os { code: 13, kind: PermissionDenied, message: "Permission denied" }`.
   *Fix:* use `OpenOptions` with `.read(true)` and `.write(true)`, or `map_copy` if you do not want to change
   the file.

3. **Expecting `flush` on every change to be free.** Each `flush` waits for the disk. *Fix:* group
   changes and flush once, at the points where the data must be durable.

4. **Trying to map a pipe.** `cat small.txt | ./target/debug/mcount /dev/stdin` fails with
   `Error: Os { code: 19, kind: Uncategorized, message: "No such device" }`, because a pipe has no pages to
   map. *Fix:* use `read` for streams; use `mmap` only for regular files.

## Try it yourself

1. Run `mwrite`, then look at the same file in two ways: `od -c counter.bin` and `cat counter.bin`.
   `cat` shows `HELLO`, then invisible zero bytes, then a new line.
2. Watch the mappings of a running program. Add `std::thread::sleep(std::time::Duration::from_secs(30));`
   at the end of `mcount`, start it with `./target/debug/mcount big.txt &`, then run
   `grep big.txt /proc/$!/maps` (`$!` is the PID of the last background program). You see one line like
   `ffffa4dab000-ffffa6380000 r--s 00000000 00:34 320408  /home/.../big.txt`: read-only and **s**hared.
3. Run `cargo run --bin mcount -- /dev/stdin < small.txt`. Redirecting a regular file works, because
   `/dev/stdin` then points to the file itself. Now try `cat small.txt | cargo run --bin mcount -- /dev/stdin`
   and compare the error with Common mistake 4.

## Exercises

### Exercise 1 — Count with a tested function (easy)

Move the counting out of `main` into `pub fn count_byte(data: &[u8], needle: u8) -> usize` in `src/lib.rs`,
and test it with `b"a\nb\nc\n"`, an empty slice and a text without new lines. Use it in `mcount`.

<details>
<summary>Hint</summary>

The function does not need to know about mappings at all: a mapping *is* a `&[u8]`. In `mcount`, call
`p13_mmap::count_byte(&map, b'\n')`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Counts how many times `needle` occurs in `data`.
pub fn count_byte(data: &[u8], needle: u8) -> usize {
    data.iter().filter(|&&b| b == needle).count()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_newlines() {
        assert_eq!(count_byte(b"a\nb\nc\n", b'\n'), 3);
        assert_eq!(count_byte(b"", b'\n'), 0);
        assert_eq!(count_byte(b"no newline", b'\n'), 0);
    }
}
```

Why it works: because `Mmap` dereferences to `&[u8]`, all logic can be written and tested on plain slices.
Only `main` knows that the bytes come from a mapping.

</details>

### Exercise 2 — `mgrep` (medium)

Write `src/bin/mgrep.rs`: `mgrep <word> <file>` maps the file and prints the numbers (starting at 1) of the
lines that contain the word. Put the search into `pub fn matching_lines(data: &[u8], word: &[u8]) -> Vec<usize>`
with a test. An empty word matches nothing. For a file with the lines `apple pie`, `banana`, `pineapple` and
the word `apple`, the output is `1` and `3`.

<details>
<summary>Hint</summary>

`data.split(|&b| b == b'\n')` gives the lines as `&[u8]`. `line.windows(word.len())` gives every piece of
the line with the length of the word; compare each with `word`.

</details>

<details>
<summary>Answer</summary>

In `src/lib.rs`:

```rust
/// Returns the numbers (starting at 1) of the lines that contain `word`.
pub fn matching_lines(data: &[u8], word: &[u8]) -> Vec<usize> {
    data.split(|&b| b == b'\n')
        .enumerate()
        .filter(|(_, line)| !word.is_empty() && line.windows(word.len()).any(|w| w == word))
        .map(|(i, _)| i + 1)
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_line_numbers() {
        let text = b"apple pie\nbanana\npineapple\n";
        assert_eq!(matching_lines(text, b"apple"), vec![1, 3]);
        assert_eq!(matching_lines(text, b"cherry"), Vec::<usize>::new());
        assert_eq!(matching_lines(text, b""), Vec::<usize>::new());
    }
}
```

`src/bin/mgrep.rs`:

```rust
use std::env;
use std::fs::File;

use memmap2::Mmap;
use p13_mmap::matching_lines;

fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().collect();
    if args.len() != 3 {
        eprintln!("usage: mgrep <word> <file>");
        std::process::exit(2);
    }
    let file = File::open(&args[2])?;
    // SAFETY: we assume nobody shortens the file while we read it.
    let map = unsafe { Mmap::map(&file)? };
    for n in matching_lines(&map, args[1].as_bytes()) {
        println!("{n}");
    }
    Ok(())
}
```

```bash
printf 'apple pie\nbanana\npineapple\n' > fruit.txt
cargo run --bin mgrep -- apple fruit.txt
```

```text
1
3
```

Why it works: `windows(n)` slides a window of `n` bytes over the line, so a match anywhere in the line is
found. The empty-word check comes first because `windows(0)` panics.

</details>

### Exercise 3 — Upper-case a file in place (hard)

Write `pub fn upcase_file(path: &Path) -> io::Result<()>` that turns every ASCII (American Standard Code for Information Interchange, Lesson 12.5) letter of a file into a
capital letter **in place**, through a writable shared mapping, and a small binary `upcase <file>`. An empty
file must work. Write a test that creates a temporary file with `hello, mmap!\n`, calls the function and
checks the result `HELLO, MMAP!\n`.

<details>
<summary>Hint</summary>

`u8::make_ascii_uppercase()` changes one byte. For the test file use
`std::env::temp_dir().join(format!("upcase-test-{}.txt", std::process::id()))`, so parallel test runs do not
collide, and remove the file at the end.

</details>

<details>
<summary>Answer</summary>

In `src/lib.rs`:

```rust
use std::fs::OpenOptions;
use std::io;
use std::path::Path;

use memmap2::MmapMut;

/// Turns ASCII letters a-z into A-Z, in place.
pub fn upcase_in_place(bytes: &mut [u8]) {
    for b in bytes.iter_mut() {
        b.make_ascii_uppercase();
    }
}

/// Upper-cases a whole file in place through a shared, writable mapping.
pub fn upcase_file(path: &Path) -> io::Result<()> {
    let file = OpenOptions::new().read(true).write(true).open(path)?;
    if file.metadata()?.len() == 0 {
        return Ok(()); // nothing to change
    }
    // SAFETY: we assume no other process shortens the file while we work on it.
    let mut map = unsafe { MmapMut::map_mut(&file)? };
    upcase_in_place(&mut map);
    map.flush()
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::fs;

    #[test]
    fn upcases_a_real_file() -> io::Result<()> {
        let path = std::env::temp_dir().join(format!("upcase-test-{}.txt", std::process::id()));
        fs::write(&path, "hello, mmap!\n")?;
        upcase_file(&path)?;
        let result = fs::read_to_string(&path)?;
        fs::remove_file(&path)?;
        assert_eq!(result, "HELLO, MMAP!\n");
        Ok(())
    }
}
```

`src/bin/upcase.rs`:

```rust
use std::env;
use std::path::Path;

fn main() -> std::io::Result<()> {
    let path = env::args().nth(1).expect("usage: upcase <file>");
    p13_mmap::upcase_file(Path::new(&path))
}
```

`cargo test` prints `test result: ok.` for the library tests, and `upcase fruit.txt` turns the fruit file
into `APPLE PIE`, `BANANA`, `PINEAPPLE`.

Why it works: the mapping is `MAP_SHARED`, so changing the bytes changes the file in the page cache, and
`flush` writes it to the disk. Nothing was read into a buffer or written back with `write`.

</details>

## Quiz

1. What is the main difference between `read` and `mmap` for the data of a file?

<details>
<summary>Answer</summary>

`read` copies data from the page cache into a buffer of the program. `mmap` maps the page cache pages
directly into the address space, so the program reads them in place without a copy and without more system
calls.

</details>

2. Why must you call `set_len` before creating a writable mapping of a new file?

<details>
<summary>Answer</summary>

A mapping cannot make a file bigger. A new file has length 0, so its mapping would have length 0. The file
must already have the size you want to write into.

</details>

3. You change a private (`map_copy`) mapping. What happens to the file?

<details>
<summary>Answer</summary>

Nothing. The first write to each page creates a private copy (copy-on-write); the file and other processes
never see the changes.

</details>

4. Why is `Mmap::map` an `unsafe` function?

<details>
<summary>Answer</summary>

Because the resulting `&[u8]` can change or disappear behind Rust's back: another process may write to the
file or shorten it. Rust cannot check that, so the programmer must promise it. A shortened file even leads
to SIGBUS.

</details>

5. Name one situation where `mmap` is a good choice and one where `read` is better.

<details>
<summary>Answer</summary>

`mmap`: random access into a large file, or many processes sharing the same file data. `read`: reading a
file once from start to end, small files, and pipes or sockets, which cannot be mapped.

</details>

## Summary

- `mmap` makes a file part of the address space; its pages come from the page cache through page faults, without copying into a buffer.
- In Rust, `memmap2::Mmap` gives a `&[u8]` and `MmapMut` a `&mut [u8]`; `Drop` unmaps.
- Shared mappings change the file (`flush` = `msync` makes it durable); private mappings are copy-on-write; anonymous mappings are fresh zeroed memory used by allocators.
- A writable mapping needs a file opened for reading and writing and already of the right size.
- `mmap` is not automatically faster: measure. It shines for random access and sharing; `read` is fine for streaming.
- Creating a mapping is `unsafe` because other processes can change or shorten the file; touching pages past the end gives SIGBUS.

## Sources and further reading

- [mmap(2) — Linux manual page](https://man7.org/linux/man-pages/man2/mmap.2.html) — every flag of `mmap`, including `MAP_SHARED`, `MAP_PRIVATE` and the SIGBUS case.
- [msync(2) — Linux manual page](https://man7.org/linux/man-pages/man2/msync.2.html) — what flushing a mapping means.
- [memmap2 on docs.rs](https://docs.rs/memmap2/0.9.12/memmap2/) — the crate used in this lesson, with the safety notes for `Mmap::map`.
- [The Linux kernel documentation — Page Cache](https://docs.kernel.org/mm/page_cache.html) — how the kernel keeps file data in RAM.
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — the chapter "Files and Directories" and the virtual memory chapters give the background.

---

[🏠 Start](../START-HERE.md) · [← 13.10 Virtual memory](10-virtual-memory.md) · [📋 Phase 13](README.md) · [13.12 File systems from the inside →](12-file-systems-inside.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
