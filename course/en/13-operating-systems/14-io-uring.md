<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.13 Non-blocking I/O and epoll](13-non-blocking-io-and-epoll.md) · [📋 Phase 13](README.md) · [13.15 Namespaces →](15-namespaces.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/14-io-uring.md) <!-- nav -->

# Lesson 13.14 — io_uring

> **Phase 13 — Operating systems from the inside** · Lesson 14 of 18 · ⏱ about 120 minutes

## What you will learn

- The difference between the *readiness* model of epoll and the *completion* model of io_uring.
- How io_uring works: two ring buffers in memory shared by your program and the kernel.
- How to submit reads and writes with the `io-uring` crate, and why pushing an operation is `unsafe`.
- How many operations can be in flight at once and finish in any order.
- Where io_uring is available, why it is sometimes switched off, and how it fits into async Rust.

## Before you start

- You finished [Lesson 13.13 — Non-blocking I/O and epoll](13-non-blocking-io-and-epoll.md).
- You know shared mappings ([Lesson 13.11](11-memory-mapped-files.md)), raw pointers and `// SAFETY:` comments ([Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)).
- Create the package on the Mac and copy it to the VM as in Lesson 13.10:

```bash
cd ~/projects/rust
cargo new --lib code/p13-uring
cd code/p13-uring
cargo add io-uring@0.7
mkdir src/bin
```

- Check in the VM that io_uring is allowed:

```bash
sysctl kernel.io_uring_disabled
```

Example output (yours will look similar):

```text
kernel.io_uring_disabled = 0
```

`0` means "allowed for everybody". If it is `2`, io_uring is switched off on this machine (Section 5).

## New words

| Word | Meaning |
|---|---|
| **io_uring** | A Linux interface (since kernel 5.1, 2019) for asynchronous I/O (input/output): you queue operations in shared memory and the kernel reports when each one is done. The name means "I/O user ring". |
| **completion model** | You ask the kernel to *do* an operation; later it tells you the operation is *finished* and gives the result. |
| **readiness model** | The kernel tells you when an operation *could* be done without waiting; you then do it yourself (epoll, Lesson 13.13). |
| **ring buffer** | A fixed-size array used as a queue whose end wraps around to the start. One side writes at the *tail*, the other reads at the *head*. |
| **SQ** (Submission Queue) | The ring where your program puts operations for the kernel. |
| **SQE** (Submission Queue Entry) | One operation in the SQ: what to do (read, write, open…), on which descriptor, with which buffer. |
| **CQ** (Completion Queue) | The ring where the kernel puts results. |
| **CQE** (Completion Queue Entry) | One result: your `user_data` number and a `result` (bytes or a negative error number). |
| **`user_data`** | A 64-bit number you attach to an SQE; the kernel copies it into the matching CQE, so you know which operation finished. |
| **in flight** | Submitted but not yet completed. |
| **seccomp** (secure computing mode) | A Linux feature that lets a process (or a container runtime) forbid certain system calls. |

## 1. From "ready" to "done"

With epoll (Lesson 13.13) the conversation with the kernel goes like this:

```text
program: "tell me when fd 5 has data"       (epoll_ctl, once)
program: "anything ready?"                   (epoll_wait)
kernel:  "fd 5 is ready"
program: "read fd 5 into this buffer"        (read)  <- one more system call per operation
kernel:  "here are 100 bytes"
```

Two weaknesses show up. Every operation still needs its own system call, and each system call costs time
to enter and leave the kernel. And regular files do not work at all, because a file on disk is always
"ready" even when reading it means waiting for the disk.

io_uring changes the conversation:

```text
program: "read fd 5 into this buffer, read fd 7 into that buffer, write fd 9 ..."  (queued in memory)
program: "go" (one system call for the whole batch, or even none)
kernel:  ... does the work, maybe in parallel ...
kernel:  "operation 1 done: 100 bytes; operation 3 done: 4096 bytes; ..."         (in memory)
```

You describe *what* you want; the kernel does it and reports when it is *done*. That is the **completion
model**. It works for files, sockets and pipes alike.

## 2. Two rings in shared memory

io_uring is built around two **ring buffers** that live in memory mapped into both the kernel and your
process (shared mappings, Lesson 13.11). Nobody needs to copy them, and often nobody needs a system call to
look at them.

```text
          your program                               kernel
   +------------------------------------------------------------------+
   | SQ (submission queue)                                            |
   |   tail -> you write SQEs here      head -> kernel takes them here|
   |   [ read fd4 buf0 off0 ud=0 ][ read fd4 buf1 off4096 ud=1 ][ ]   |
   +------------------------------------------------------------------+
   | CQ (completion queue)                                            |
   |   head -> you take CQEs here       tail -> kernel writes here    |
   |   [ ud=1 res=4096 ][ ud=0 res=4096 ][ ]                          |
   +------------------------------------------------------------------+
        system call io_uring_enter: "I added N entries" / "wait for M"
```

1. Your program writes **SQEs** at the tail of the **SQ** and moves the tail.
2. It calls `io_uring_enter` to tell the kernel "there are new entries" (and, if it wants, "wait until M
   are complete").
3. The kernel takes entries from the head, performs them, and writes a **CQE** for each at the tail of the
   **CQ**.
4. Your program reads CQEs from the head of the CQ. The `user_data` in each CQE tells it which SQE it belongs to.

A ring buffer is simply an array plus two counters: the writer increments the tail, the reader increments
the head, and positions are taken modulo the size. Because one side only writes and the other only reads
each counter, no lock is needed.

The `io-uring` crate wraps all of this: `IoUring::new` sets up the rings, `submission()` gives you the SQ,
`completion()` the CQ, and `opcode::*` builds SQEs.

## 3. The first read

Create `src/bin/first.rs`:

```rust
use std::fs::File;
use std::os::fd::AsRawFd;

use io_uring::{IoUring, opcode, types};

fn main() -> std::io::Result<()> {
    let mut ring = IoUring::new(8)?; // room for 8 submissions
    let file = File::open("/etc/os-release")?;
    let mut buf = vec![0u8; 64];

    // Describe the operation: "read up to 64 bytes from this fd into buf, at offset 0".
    let read = opcode::Read::new(
        types::Fd(file.as_raw_fd()),
        buf.as_mut_ptr(),
        buf.len() as u32,
    )
    .offset(0)
    .build()
    .user_data(42);

    // SAFETY: `buf` and `file` stay alive and are not moved until the completion arrives.
    unsafe {
        ring.submission()
            .push(&read)
            .expect("submission queue is full");
    }
    ring.submit_and_wait(1)?; // one system call: hand over the work and wait for 1 completion

    let cqe = ring.completion().next().expect("no completion");
    println!("user_data = {}, result = {}", cqe.user_data(), cqe.result());
    let n = cqe.result() as usize;
    println!("{:?}", String::from_utf8_lossy(&buf[..n]));
    Ok(())
}
```

**What happens:**

1. `IoUring::new(8)` calls `io_uring_setup` and maps the rings. The SQ has 8 entries (the CQ gets twice as
   many by default).
2. `opcode::Read::new(fd, pointer, length)` builds the description of a read. It takes a **raw pointer** to
   the buffer, not a `&mut [u8]`: the kernel will write there later, after this line has finished.
   `.offset(0)` says where in the file to read; `.user_data(42)` attaches our label.
3. `push` is `unsafe` because the compiler cannot check what the kernel will do with that pointer. We
   promise that `buf` (and the descriptor) live, and are not moved or reused, until the completion arrives.
   If `buf` were dropped too early, the kernel would write into freed memory: undefined behaviour.
4. `submit_and_wait(1)` calls `io_uring_enter` once: "here is new work, return when at least 1 operation is
   complete".
5. `completion().next()` takes the CQE. `result()` is the number of bytes read, like the return value of
   `read`; a negative number would be an error (Exercise 1).

In the VM, `cargo run --bin first`:

```text
user_data = 42, result = 64
"PRETTY_NAME=\"Debian GNU/Linux 13 (trixie)\"\nNAME=\"Debian GNU/Linu"
```

(Your VM runs Ubuntu, so the text differs.) `strace` shows the system calls. Below, the start-up lines are
cut and the long `io_uring_setup` line is shortened with `...`:

```bash
strace -e trace=io_uring_setup,io_uring_enter,mmap,openat ./target/debug/first
```

```text
io_uring_setup(8, {flags=0, sq_thread_cpu=0, sq_thread_idle=0, sq_entries=8, cq_entries=16, ...}) = 3
mmap(NULL, 512, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE, 3, 0x10000000) = 0xffff9e1d2000
mmap(NULL, 352, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE, 3, 0) = 0xffff9e1d1000
openat(AT_FDCWD, "/etc/os-release", O_RDONLY|O_CLOEXEC) = 4
io_uring_enter(3, 1, 1, IORING_ENTER_GETEVENTS, NULL, 128) = 1
user_data = 42, result = 64
```

**What happens** in the trace:

1. `io_uring_setup` creates the ring and returns a descriptor (3). The kernel answers with the real sizes:
   `sq_entries=8, cq_entries=16`.
2. Two `mmap` calls with `MAP_SHARED` on descriptor 3 map the rings into our address space: the one at
   offset 0 holds both rings (SQ and CQ), the one at offset `0x10000000` the array of SQEs. It is the same
   shared mapping as in Lesson 13.11, but backed by kernel memory instead of a file.
3. `io_uring_enter(3, 1, 1, IORING_ENTER_GETEVENTS, …)`: "1 new entry, wait for 1 completion". There is no
   `read` system call: the kernel did the read itself.

## 4. Many operations in flight

The real strength appears with many operations. Create `src/bin/many.rs`, which reads 16 blocks of a file
with **one** system call:

```rust
use std::fs::{self, File};
use std::os::fd::AsRawFd;

use io_uring::{IoUring, opcode, types};

const BLOCK: usize = 4096;
const COUNT: usize = 16;

fn main() -> std::io::Result<()> {
    // A test file: block 0 is full of 'A', block 1 of 'B', ... block 15 of 'P'.
    let data: Vec<u8> = (0..COUNT)
        .flat_map(|i| vec![b'A' + i as u8; BLOCK])
        .collect();
    fs::write("blocks.bin", &data)?;

    let file = File::open("blocks.bin")?;
    let mut ring = IoUring::new(COUNT as u32)?;
    let mut buffers = vec![vec![0u8; BLOCK]; COUNT];

    for (i, buf) in buffers.iter_mut().enumerate() {
        let read = opcode::Read::new(types::Fd(file.as_raw_fd()), buf.as_mut_ptr(), BLOCK as u32)
            .offset((i * BLOCK) as u64)
            .build()
            .user_data(i as u64); // remember which block this is
        // SAFETY: `buffers` and `file` live until all completions have been collected below.
        unsafe { ring.submission().push(&read).expect("queue full") };
    }
    let submitted = ring.submit_and_wait(COUNT)?; // ONE system call for all 16 reads
    println!("submitted {submitted} reads with one io_uring_enter");

    let mut order = Vec::new();
    for cqe in ring.completion() {
        assert_eq!(cqe.result(), BLOCK as i32, "a read failed or was short");
        order.push(cqe.user_data());
    }
    println!("completion order: {order:?}");
    let firsts: String = buffers.iter().map(|b| b[0] as char).collect();
    println!("first byte of each block: {firsts}");
    Ok(())
}
```

**What happens:**

1. The loop pushes 16 read SQEs, each with its own buffer, offset and `user_data` (the block number).
   Nothing has been sent to the kernel yet: we only wrote into shared memory.
2. `submit_and_wait(COUNT)` hands all 16 over in one `io_uring_enter` and waits until 16 are complete.
3. `ring.completion()` is an iterator over all CQEs that are ready. We collect their `user_data` to see the
   order.
4. Each buffer received its block: the first bytes spell `A` to `P`.

```text
submitted 16 reads with one io_uring_enter
completion order: [0, 2, 3, 1, 6, 4, 7, 5, 9, 8, 10, 11, 13, 12, 14, 15]
first byte of each block: ABCDEFGHIJKLMNOP
```

The completions arrived **out of order**, and the order changes from run to run. The kernel worked on
several reads at the same time. This is why `user_data` exists: never assume that completion *n* belongs to
submission *n*.

Count the system calls with `strace -c` (a summary table instead of a list):

```bash
strace -c -e trace=io_uring_enter,read,pread64 ./target/debug/many
```

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 89.05    0.000187         187         1           io_uring_enter
 10.95    0.000023           4         5           read
------ ----------- ----------- --------- --------- ----------------
100.00    0.000210          35         6           total
```

The five `read` calls come from the loader reading the program's libraries at start-up. All 16 reads of the
file went through a single `io_uring_enter`. With `pread` (a `read` at an offset) it would have been 16
system calls.

> ⚡ **Performance:** fewer system calls matter when a program does hundreds of thousands of small
> operations per second, as databases and proxies do. For a program that reads a few files, plain `read` is
> simpler and just as fast. As always, measure first.

## 5. Availability and security

io_uring is powerful, and that has a price. It is a large new part of the kernel that can do almost
everything ordinary system calls can, and it has had many serious security bugs since 2019. That is why:

- Some systems switch it off. The setting `kernel.io_uring_disabled` is `0` (allowed), `1` (only for root and
  members of one group, set in `kernel.io_uring_group`) or `2` (disabled for everyone).
- **Docker's default seccomp profile blocks it.** **seccomp** lets a container runtime forbid system calls,
  and `io_uring_setup` is forbidden by default. In a normal container, `first` fails at once:

  ```text
  Error: Os { code: 1, kind: PermissionDenied, message: "Operation not permitted" }
  ```

  The examples of this lesson were checked in a container started with `--security-opt seccomp=unconfined`.
  In your VM they work without any option.

> 🔒 **Security:** treat io_uring like any powerful feature: use it where it gives a real benefit, keep the
> kernel updated, and expect it to be disabled in hardened environments. Write code that can fall back to
> normal I/O.

**io_uring and async Rust.** Tokio uses epoll by default, not io_uring. Runtimes such as `tokio-uring` are built on
io_uring. They show one more lesson about ownership: because the kernel uses a buffer *after* the call that
submitted it, a completion-based API cannot just borrow `&mut [u8]` the way `read` does. The operation must
**own** the buffer until it is done, and give it back afterwards. That is why such APIs take a `Vec<u8>` by
value and return it together with the result.

## Common mistakes

1. **Assuming completions come in submission order.** They do not, as `many` showed. *Fix:* always use
   `user_data` to match a CQE to its SQE.

2. **Letting the buffer die too early.** If the buffer is a local variable of a function that returns
   before the completion arrives, the kernel writes into memory that belongs to something else. The
   compiler does not warn you; that is exactly what the `unsafe` block is about. *Fix:* keep buffers in a
   place that lives longer than the operation (as `buffers` in `many`), and collect all completions before
   dropping it.

3. **Ignoring negative results.** `result()` is `-2` for "no such file", `-9` for "bad file descriptor", and
   so on. Treating it as a byte count (`as usize`) gives a giant number. *Fix:* convert it first (Exercise 1).

4. **Testing in a default container.** `Operation not permitted` from `IoUring::new` usually means seccomp
   or `kernel.io_uring_disabled`, not a bug in your code.

## Try it yourself

1. Change `COUNT` in `many` to 64 (the ring size follows it). Run it a few times and look at the order.
2. Make the file one block shorter: change `(0..COUNT)` in the line `let data` to `(0..COUNT - 1)`, but
   still submit 16 reads. The assertion fails with `left: 0` and `right: 4096`: a read that starts at the end
   of the file completes with result 0, exactly like `read` at end of file.
3. Add `std::thread::sleep(std::time::Duration::from_secs(10));` before `Ok(())` in `first`, start it in the
   background with `./target/debug/first &` and run `ls -l /proc/$!/fd`. Descriptor 3 is shown as
   `anon_inode:[io_uring]`: the ring itself is a file descriptor, like the epoll instance in Lesson 13.13.

## Exercises

### Exercise 1 — Turn a CQE result into `io::Result` (easy)

Write `pub fn cqe_result(result: i32) -> io::Result<usize>` in `src/lib.rs`. A value of zero or more is a
byte count. A negative value is a negated error number (errno), for example `-2` for ENOENT ("no such file").
Write tests for `5`, `0` and `-2` (its `kind()` must be `NotFound`).

<details>
<summary>Hint</summary>

`io::Error::from_raw_os_error(code)` builds an error from a positive errno.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io;

/// Turns the `result` field of a completion into an `io::Result`:
/// a negative value is a negated error number (errno).
pub fn cqe_result(result: i32) -> io::Result<usize> {
    if result < 0 {
        Err(io::Error::from_raw_os_error(-result))
    } else {
        Ok(result as usize)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn positive_result_is_a_byte_count() {
        assert_eq!(cqe_result(5).unwrap(), 5);
        assert_eq!(cqe_result(0).unwrap(), 0);
    }

    #[test]
    fn negative_result_is_an_errno() {
        let err = cqe_result(-2).unwrap_err(); // 2 = ENOENT
        assert_eq!(err.kind(), io::ErrorKind::NotFound);
    }
}
```

Why it works: ordinary system calls return -1 and put the error number in `errno`; io_uring has no
`errno`, so it puts the negated error number straight into `result`. `from_raw_os_error` gives you the same
`io::Error` that `std` would have produced.

</details>

### Exercise 2 — Write a file with io_uring (medium)

Write `src/bin/uwrite.rs` that creates `out.txt`, writes `hello from io_uring\n` into it with
`opcode::Write`, prints the number of bytes written (use `cqe_result`) and then prints the file content read
back with `fs::read_to_string`.

<details>
<summary>Hint</summary>

`opcode::Write::new(fd, pointer, length)` takes a `*const u8`: `text.as_ptr()`. Everything else is like
`first`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs::{self, File};
use std::os::fd::AsRawFd;

use io_uring::{IoUring, opcode, types};
use p13_uring::cqe_result;

fn main() -> std::io::Result<()> {
    let text = b"hello from io_uring\n";
    let file = File::create("out.txt")?;
    let mut ring = IoUring::new(2)?;

    let write = opcode::Write::new(
        types::Fd(file.as_raw_fd()),
        text.as_ptr(),
        text.len() as u32,
    )
    .offset(0)
    .build()
    .user_data(1);
    // SAFETY: `text` is a constant and `file` lives until the completion is read.
    unsafe { ring.submission().push(&write).expect("queue full") };
    ring.submit_and_wait(1)?;

    let cqe = ring.completion().next().expect("no completion");
    let written = cqe_result(cqe.result())?;
    println!("wrote {written} bytes");
    print!("file now contains: {}", fs::read_to_string("out.txt")?);
    Ok(())
}
```

```text
wrote 20 bytes
file now contains: hello from io_uring
```

Why it works: a write is described exactly like a read, with a pointer to the data. `text` is a byte string
literal that lives for the whole program, so the safety promise is easy to keep.

</details>

### Exercise 3 — Read blocks in the order you asked (hard)

Write `pub fn read_blocks(file: &File, offsets: &[u64], len: usize) -> io::Result<Vec<Vec<u8>>>`. It
submits one read of `len` bytes per offset, all at once, and returns the buffers **in the order of
`offsets`**, even though completions arrive in any order. A read at the end of the file may be short; cut
that buffer to the real length. Test it with a file containing `aaaabbbbccccdd`, offsets `[8, 0, 12]` and
length 4: the result is `cccc`, `aaaa`, `dd`.

<details>
<summary>Hint</summary>

Use the index of the offset as `user_data`, and `buffers[cqe.user_data() as usize].truncate(n)`. The ring
needs room for all reads and at least one entry; the kernel works with powers of two, so ask for
`offsets.len().max(1).next_power_of_two()`.

</details>

<details>
<summary>Answer</summary>

Add to `src/lib.rs` (next to `cqe_result`):

```rust
use std::fs::File;
use std::os::fd::AsRawFd;

use io_uring::{IoUring, opcode, types};

/// Reads `len` bytes at each offset, all submitted at once.
/// The results come back in the order of `offsets`, whatever order the kernel finishes in.
pub fn read_blocks(file: &File, offsets: &[u64], len: usize) -> io::Result<Vec<Vec<u8>>> {
    let mut ring = IoUring::new(offsets.len().max(1).next_power_of_two() as u32)?;
    let mut buffers = vec![vec![0u8; len]; offsets.len()];
    for (i, (buf, &offset)) in buffers.iter_mut().zip(offsets).enumerate() {
        let read = opcode::Read::new(types::Fd(file.as_raw_fd()), buf.as_mut_ptr(), len as u32)
            .offset(offset)
            .build()
            .user_data(i as u64);
        // SAFETY: `buffers` is not touched or moved until every completion is collected.
        unsafe { ring.submission().push(&read).expect("queue full") };
    }
    ring.submit_and_wait(offsets.len())?;
    for cqe in ring.completion() {
        let n = cqe_result(cqe.result())?;
        buffers[cqe.user_data() as usize].truncate(n); // a read at the end may be short
    }
    Ok(buffers)
}
```

And the test, in the `tests` module:

```rust
#[test]
fn reads_blocks_in_the_order_asked() -> io::Result<()> {
    let path = std::env::temp_dir().join(format!("blocks-{}.bin", std::process::id()));
    fs::write(&path, b"aaaabbbbccccdd")?;
    let file = File::open(&path)?;
    let blocks = read_blocks(&file, &[8, 0, 12], 4)?;
    fs::remove_file(&path)?;
    assert_eq!(
        blocks,
        vec![b"cccc".to_vec(), b"aaaa".to_vec(), b"dd".to_vec()]
    );
    Ok(())
}
```

(Add `use std::fs;` at the top of the `tests` module.) `cargo test`:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `user_data` carries the position in `offsets`, so each completion finds its own buffer. While
the kernel works, the program does not touch `buffers`; the kernel writes into them through the raw
pointers. `truncate` happens only after `submit_and_wait` has returned, when all reads are complete.

</details>

## Quiz

1. What is the main difference between epoll and io_uring?

<details>
<summary>Answer</summary>

epoll reports *readiness*: you still perform each operation with its own system call. io_uring uses
*completion*: you describe operations, the kernel performs them and reports the results.

</details>

2. What are the SQ and the CQ, and who writes to each?

<details>
<summary>Answer</summary>

The submission queue, where the program writes operations (SQEs) for the kernel, and the completion queue,
where the kernel writes results (CQEs) for the program. Both are ring buffers in memory shared by both
sides.

</details>

3. Why is pushing an SQE `unsafe` in the `io-uring` crate?

<details>
<summary>Answer</summary>

The SQE contains raw pointers to buffers that the kernel will use later. The compiler cannot check that the
buffers stay alive and unmoved until the operation completes; the programmer must guarantee it.

</details>

4. Your program submits 10 reads and gets 10 CQEs. How does it know which result belongs to which read?

<details>
<summary>Answer</summary>

By `user_data`: each SQE carries a number chosen by the program, and the kernel copies it into the
matching CQE. The order of CQEs is not the order of submission.

</details>

5. `IoUring::new` fails with "Operation not permitted" in a Docker container. Why?

<details>
<summary>Answer</summary>

Docker's default seccomp profile forbids the io_uring system calls. (On a normal machine the sysctl
`kernel.io_uring_disabled` can also switch it off.)

</details>

## Summary

- io_uring is Linux's completion-based I/O interface: describe operations, let the kernel do them, collect results.
- Two ring buffers in shared memory: the SQ (program → kernel, SQEs) and the CQ (kernel → program, CQEs); `io_uring_enter` submits and waits.
- Many operations can be in flight and finish in any order; `user_data` matches results to requests.
- `result` is a byte count or a negative errno; buffers must live until their completion, which is why pushing is `unsafe`.
- One system call can submit a whole batch, which matters for very high operation rates.
- io_uring can be disabled (sysctl, seccomp in Docker); completion-based async APIs must own their buffers.

## Sources and further reading

- [io_uring(7) — Linux manual page](https://man7.org/linux/man-pages/man7/io_uring.7.html) — the overview of the interface, the rings and the entries.
- [io_uring_enter(2) — Linux manual page](https://man7.org/linux/man-pages/man2/io_uring_enter.2.html) — submitting and waiting, and every opcode.
- [io-uring on docs.rs](https://docs.rs/io-uring/0.7.16/io_uring/) — the Rust crate used in this lesson.
- [Efficient IO with io_uring](https://kernel.dk/io_uring.pdf) — the design paper by the author of io_uring, Jens Axboe.
- [Lord of the io_uring](https://unixism.net/loti/) — a long tutorial (examples in C) for going deeper.

---

[🏠 Start](../START-HERE.md) · [← 13.13 Non-blocking I/O and epoll](13-non-blocking-io-and-epoll.md) · [📋 Phase 13](README.md) · [13.15 Namespaces →](15-namespaces.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
