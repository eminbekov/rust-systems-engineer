<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.2 Processes from the inside](02-processes-from-the-inside.md) · [📋 Phase 13](README.md) · [13.4 Files and file descriptors →](04-files-and-file-descriptors.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/03-system-calls.md) <!-- nav -->

# Lesson 13.3 — System calls

> **Phase 13 — Operating systems from the inside** · Lesson 3 of 18 · ⏱ about 120 minutes

## What you will learn

- What exactly happens when a program makes a system call, down to the CPU instruction.
- How one system call looks at four levels: assembly, the C library, the `nix` crate and the Rust standard library.
- How the kernel reports errors (`errno`), and how `libc`, `nix` and `std` show them to you.
- How to watch every system call of a program with `strace`.
- Why system calls cost time, and how buffering saves most of them.

## Before you start

- You finished [Lesson 13.2 — Processes from the inside](02-processes-from-the-inside.md).
- You can read a little arm64 assembly: registers `x0`…`x30` and the idea of an instruction ([Lesson 12.7 — Reading assembly](../12-computer-systems/07-reading-assembly.md)).
- You know `unsafe` and raw pointers ([Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)) and how to call C functions with FFI ([Lesson 06.10 — FFI: calling C](../06-rust-advanced/10-ffi-calling-c.md)).
- `strace` is installed in your VM (Lesson 13.1, Section 5).

## New words

| Word | Meaning |
|---|---|
| **system call number** | The number that tells the kernel which service you want: on arm64 Linux, 64 is `write` and 172 is `getpid`. |
| **`svc`** (SuperVisor Call) | The arm64 instruction that enters the kernel. On x86-64 the instruction is called `syscall`. |
| **ABI** (Application Binary Interface) | The rules at machine level: which register holds which argument, where the result goes. |
| **libc** (C library) | The library of small C functions (`write`, `open`, `fork` …) that every Linux program uses to make system calls. Ubuntu and Debian use glibc (the GNU C library). |
| **wrapper** | A small function around another one that makes it easier or safer to use. |
| **`errno`** (error number) | A per-thread variable where libc stores why the last call failed, for example 2 = `ENOENT`. |
| **`strace`** (system call trace) | A tool that runs a program and prints every system call it makes, with arguments and results. |
| **buffer** / **buffering** | A piece of memory where data is collected / collecting many small writes into one big write. |
| **vDSO** (virtual Dynamic Shared Object) | A tiny library that the kernel puts into every process, so that a few calls (such as reading the clock) do not need to enter the kernel. |
| **CPU**, **OS**, **PID**, **ELF**, **KiB**, **GNU** | Known from earlier lessons: Central Processing Unit; Operating System; Process IDentifier; Executable and Linkable Format (Lesson 12.12); kibibyte (1,024 bytes); GNU ("GNU's Not Unix"), the free-software project that makes glibc. |

## 1. A function call that crosses the border

Your program cannot simply call a function inside the kernel: the kernel's memory is protected (Lesson 13.1), and
jumping there would cause the same `SIGSEGV` as reading it. Instead, the CPU has **one special instruction** that
switches to the kernel's privilege level and jumps to a single entry point that the kernel chose at boot. On arm64
it is `svc #0`; on x86-64 it is `syscall`.

Because there is only one door, the program must say which service it wants. It puts a **system call number** and the
arguments into registers before the instruction. These rules are part of the Linux **ABI**:

| | arm64 (your VM) | x86-64 |
|---|---|---|
| instruction | `svc #0` | `syscall` |
| system call number | `x8` | `rax` |
| arguments 1–6 | `x0` … `x5` | `rdi`, `rsi`, `rdx`, `r10`, `r8`, `r9` |
| result | `x0` | `rax` |
| `write` / `getpid` | 64 / 172 | 1 / 39 |

The kernel looks the number up in its **system call table**, runs the matching kernel function (for example
`ksys_write`), puts the result into `x0` and returns to the instruction after `svc`. If something went wrong, the
result is a small **negative** number: −2 means "error 2", which is `ENOENT`.

Note that the numbers differ between CPU types. That is one reason why programs almost never write `svc` themselves.

## 2. One system call at four levels

Let us write one line of text to the screen (file descriptor 1, standard output, Lesson 11.3) in four ways, from
the lowest level to the highest. First add the `libc` crate (Rust declarations for the C library, Lesson 06.10) and
the `fs` feature of `nix`, which we use in Section 3:

```bash
cargo add libc@0.2
cargo add nix@0.32 --features fs
```

Create `src/bin/four-writes.rs`:

```rust
use std::io::Write;

/// Level 0: the raw system call, written in assembly (arm64 only).
#[cfg(target_arch = "aarch64")]
fn raw_write(fd: usize, buf: &[u8]) -> isize {
    let ret: isize;
    // SAFETY: `write` only reads `buf.len()` bytes from `buf`, which is a valid slice.
    unsafe {
        std::arch::asm!(
            "svc #0",                  // "supervisor call": enter the kernel
            in("x8") 64_usize,         // system call number: 64 = write on arm64
            inlateout("x0") fd => ret, // argument 1 in, result out
            in("x1") buf.as_ptr(),     // argument 2: address of the bytes
            in("x2") buf.len(),        // argument 3: how many bytes
            options(nostack),
        );
    }
    ret
}

fn main() {
    // Level 0: assembly.
    #[cfg(target_arch = "aarch64")]
    raw_write(1, b"0: raw svc instruction\n");

    // Level 1: the C library function write(), through FFI.
    let text = b"1: libc::write\n";
    // SAFETY: the pointer and the length describe the valid array `text`.
    unsafe { libc::write(1, text.as_ptr().cast(), text.len()) };

    // Level 2: nix, a safe wrapper around the libc function.
    nix::unistd::write(std::io::stdout(), b"2: nix::unistd::write\n").unwrap();

    // Level 3: the standard library.
    std::io::stdout().write_all(b"3: std::io::Write\n").unwrap();
    println!("4: println!");
}
```

```text
0: raw svc instruction
1: libc::write
2: nix::unistd::write
3: std::io::Write
4: println!
```

**What happens:**

1. **Level 0.** `std::arch::asm!` puts assembly directly into a Rust function. `in("x8") 64_usize` loads the number of
   `write` into `x8`; `x0`, `x1`, `x2` get the file descriptor, the address and the length. `inlateout("x0") fd => ret`
   means "`x0` is an input, and after the instruction read the result from it". `svc #0` enters the kernel. Inline
   assembly is always `unsafe`: the compiler cannot check it. `#[cfg(target_arch = "aarch64")]` compiles this part only
   on arm64.
2. **Level 1.** `libc::write` is the C library's function. Inside, it does the same as level 0 (on any CPU type), and on
   failure it stores the error number in `errno` and returns −1. It takes a raw pointer, so the call is `unsafe`.
3. **Level 2.** `nix::unistd::write` calls `libc::write` for you, takes a Rust slice and anything that has a file
   descriptor (`std::io::stdout()` has one), and returns a `Result`. No `unsafe` in your code.
4. **Level 3.** `std::io::Write` is what you normally use. `println!` ends in the same `write` system call.

All five lines took exactly the same road into the kernel. `strace` proves it (more on this tool in Section 4):

```bash
strace -e trace=write ./target/debug/four-writes > /dev/null
```

```text
write(1, "0: raw svc instruction\n", 23) = 23
write(1, "1: libc::write\n", 15)        = 15
write(1, "2: nix::unistd::write\n", 22) = 22
write(1, "3: std::io::Write\n", 18)     = 18
write(1, "4: println!\n", 12)           = 12
+++ exited with 0 +++
```

The kernel cannot tell which level made the call. `= 23` is the result: the number of bytes written.

> 📏 **Convention:** Use the highest level that does the job. `std` first; `nix` for Unix features that `std` does not
> have; `libc` only when `nix` lacks something; assembly never in normal code. You saw it here once, to know what is at
> the bottom.

## 3. When a system call fails: `errno`

Many system calls can fail: the file does not exist, you lack permission, the disk is full. The kernel returns a
negative number, and each level turns it into something different:

- **libc** returns −1 and puts the positive error number into `errno`. You must read `errno` right away, before another
  call changes it.
- **nix** returns `Err(Errno::ENOENT)`, an enum with one name per error.
- **std** returns `Err(io::Error)`, which keeps the number (`raw_os_error()`) and a portable `kind()`.

Create `src/bin/errors.rs`:

```rust
use nix::fcntl::{OFlag, open};
use nix::sys::stat::Mode;

fn main() {
    // libc: returns -1 and leaves the error number in `errno`.
    let fd = unsafe { libc::open(c"/no/such/file".as_ptr(), libc::O_RDONLY) };
    let err = std::io::Error::last_os_error();
    println!("libc: fd = {fd}, errno = {:?}, {err}", err.raw_os_error());

    // nix: returns Result<OwnedFd, Errno>.
    match open("/no/such/file", OFlag::O_RDONLY, Mode::empty()) {
        Ok(_) => println!("nix:  opened?!"),
        Err(e) => println!("nix:  {e:?} = {}", e.desc()),
    }

    // std: returns io::Result<File>; the OS error number is inside.
    match std::fs::File::open("/no/such/file") {
        Ok(_) => println!("std:  opened?!"),
        Err(e) => println!("std:  {e} (kind {:?})", e.kind()),
    }
}
```

```text
libc: fd = -1, errno = Some(2), No such file or directory (os error 2)
nix:  ENOENT = No such file or directory
std:  No such file or directory (os error 2) (kind NotFound)
```

**What happens:**

1. `c"/no/such/file"` is a **C-string literal**: a `&CStr` with a NUL byte at the end, made at compile time. It is
   what `CString::new` from Lesson 06.10 makes, without the allocation.
2. `libc::open` returns −1. `std::io::Error::last_os_error()` reads `errno` and wraps it: error 2.
3. `nix::fcntl::open` returns `Err(Errno::ENOENT)`; `desc()` gives the text. The result on success would be an
   `OwnedFd`, an owned file descriptor (Lesson 13.4).
4. `std::fs::File::open` shows the same error 2, plus the portable kind `NotFound`.

Common error names you will meet in this phase: `ENOENT` (no such file), `EACCES` (permission denied), `EBADF` (bad
file descriptor), `EINTR` (interrupted by a signal, Lesson 13.7), `EAGAIN` (try again later), `EPIPE` (broken pipe,
Lesson 13.6). The full list is in the manual page `errno(3)`.

> ⚠️ **Common mistake:** Reading `errno` too late. Any call between the failing call and `last_os_error()` — even a
> `println!` — may change `errno`, and you report the wrong error. With `nix` and `std` the error is part of the
> returned value, so this mistake is impossible.

## 4. `strace`: watching a program talk to the kernel

`strace` runs a program and prints each system call: its name, its arguments, and after `=` its result. It uses a
kernel feature called `ptrace` (process trace), the same one that debuggers use. Start with the smallest program,
`src/bin/hello.rs`:

```rust
fn main() {
    println!("hello");
}
```

```bash
cargo build -q
strace ./target/debug/hello
```

The output is about 65 lines. Here are the beginning and the end (real output, shortened in the middle):

```text
execve("./target/debug/hello", ["./target/debug/hello"], 0xfffff220e130 /* 11 vars */) = 0
brk(NULL)                               = 0xaaaaea1eb000
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/aarch64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0\267\0\1\0\0\0\200$\2\0\0\0\0\0"..., 832) = 832
mmap(0xffff975b0000, 1826832, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0) = 0xffff975b0000
close(3)                                = 0
...
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTART}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
...
write(1, "hello\n", 6hello
)                  = 6
exit_group(0)                           = ?
+++ exited with 0 +++
```

**What happens:**

1. `execve` — the kernel replaces the program of the process with `hello` (Lesson 13.5).
2. `openat`, `read`, `mmap`, `close` — the dynamic loader (Lesson 12.12) opens `libc.so.6`, reads its ELF header and
   maps it into memory. Result `= 3` is the new file descriptor.
3. `rt_sigaction(SIGPIPE, … SIG_IGN …)` — the Rust runtime tells the kernel to **ignore** `SIGPIPE` before `main`
   starts. Lesson 13.6 shows why that matters.
4. `write(1, "hello\n", 6)` — your `println!`. The program's own output `hello` appears in the middle of the line,
   because `strace` writes to standard error and the program to standard output, on the same terminal.
5. `exit_group(0)` — the process ends with exit code 0. It has no result, so `strace` prints `= ?`.

Your one-line program makes about 60 system calls, almost all of them for loading and setup. Useful options:

| Option | What it does |
|---|---|
| `-e trace=openat,read` | show only these calls |
| `-c` | count calls and time instead of printing each one |
| `-f` | also follow child processes (Lesson 13.5) |
| `-p PID` | attach to a process that is already running |
| `-o file` | write the trace into a file |

```bash
strace -e trace=openat ./target/debug/os-info 2>&1 | grep proc
```

```text
openat(AT_FDCWD, "/proc/self/maps", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/proc/version", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/proc/uptime", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/proc/meminfo", O_RDONLY|O_CLOEXEC) = 3
```

The last three lines are the files of your `os-info` from Lesson 13.1. The first one is read by the Rust runtime at
start (it finds the stack's limits there). Each file gets descriptor 3 again, because the previous one was closed.

> 💡 **Tip:** `strace` is the first tool to reach for when a program "does nothing" or fails without a good message.
> Look for the last calls before it stopped, and for results with `-1 E…`: they tell you the file it could not open or
> the call that failed.

> 🔒 **Security:** `strace` shows everything a program sends and receives, including passwords it reads. That is why a
> normal user may only trace their own processes, and why some container setups need the option
> `--cap-add SYS_PTRACE` before `strace` works inside them.

## 5. System calls cost time

A system call is much slower than a normal function call: the CPU must save state, switch privilege level, check the
arguments and switch back. A simple call costs somewhere around 100 nanoseconds to a few microseconds, which is
hundreds of normal function calls. Programs that make millions of tiny calls waste a lot of time at the border.

Rust's `stdout` collects text until the end of a line, so every `println!` is one `write` call. Compare with a
`BufWriter`, which collects 8 KiB before it writes. Create `src/bin/many-lines.rs`:

```rust
use std::io::{BufWriter, Write};

fn main() -> std::io::Result<()> {
    let buffered = std::env::args().any(|a| a == "--buffered");
    if buffered {
        let mut out = BufWriter::new(std::io::stdout().lock());
        for i in 0..1000 {
            writeln!(out, "line {i}")?;
        }
        out.flush()?;
    } else {
        for i in 0..1000 {
            println!("line {i}");
        }
    }
    Ok(())
}
```

```bash
cargo build -q
strace -c -e trace=write ./target/debug/many-lines > /dev/null
strace -c -e trace=write ./target/debug/many-lines --buffered > /dev/null
```

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.003099           3      1000           write
------ ----------- ----------- --------- --------- ----------------
100.00    0.003099           3      1000           total
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.000008           4         2           write
------ ----------- ----------- --------- --------- ----------------
100.00    0.000008           4         2           total
```

**What happens:**

1. Without a buffer: 1000 lines, 1000 `write` calls, about 3 ms inside the kernel (`strace` itself slows each call down,
   so the real numbers are smaller, but the ratio holds).
2. With `BufWriter`: the same 8,890 bytes go out in 2 calls, because 8 KiB of text fit into the first one.
3. `stdout().lock()` takes the lock of standard output once, instead of once per line.
4. `out.flush()?` writes the rest. A `BufWriter` also flushes when it is dropped, but then it cannot report an error,
   so call `flush` yourself.

> ⚡ **Performance:** Count system calls before you optimise: `strace -c` shows where a program crosses the border.
> Batching small reads and writes with `BufReader` and `BufWriter` is the easiest speed-up in systems code.

Some calls are so frequent that Linux avoids the border entirely. The kernel maps a tiny library, the **vDSO**, into every
process; reading the clock (`clock_gettime`, used by `std::time::Instant::now()`) runs there in user space. That is why
`strace` never shows those calls.

## Common mistakes

> ⚠️ **Common mistake:** Attaching with `strace -p PID` to a process that `strace` did not start. Ubuntu's security
> module Yama allows that only for root by default (the file `/proc/sys/kernel/yama/ptrace_scope` contains 1), so even
> for your own program you get:
>
> ```text
> strace: attach: ptrace(PTRACE_SEIZE, 22): Operation not permitted
> ```
>
> Start the program under `strace` instead, or use `sudo strace -p PID` in your VM.

> ⚠️ **Common mistake:** Using the system call numbers of another CPU type. `write` is 64 on arm64 but 1 on x86-64. In
> Rust, use the constants of the `libc` crate (`libc::SYS_write`), which are correct for the target you compile for.

> ⚠️ **Common mistake:** Forgetting that `write` may write **less** than you asked. The result is the number of bytes
> actually written, and the rest must be written in another call. `std`'s `write_all` loops for you; Exercise 3 builds
> such a loop.

## Try it yourself

1. Run `strace -c ./target/debug/hello > /dev/null` and find the total number of calls in the last line.
2. Run `strace -e trace=openat cat /etc/hostname`. Which files does `cat` open besides `/etc/hostname`?
3. Run `strace -e trace=write echo hi`. Compare it with your `hello` program.
4. Change `many-lines` to print 100,000 lines and compare the two versions with `time` (Lesson 11.3).

## Exercises

### Exercise 1 — What does `ls` open? (easy)

Use `strace -e trace=openat,getdents64 ls /tmp`. Find the call that opens the folder `/tmp` and the call that reads its
entries. What is the result of `getdents64`?

<details>
<summary>Hint</summary>

Look for `openat(AT_FDCWD, "/tmp", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3`. `getdents64` means "get directory
entries".

</details>

<details>
<summary>Answer</summary>

`ls` opens `/tmp` with the flag `O_DIRECTORY` and gets descriptor 3. Then it calls `getdents64(3, …)` until the result is
0. The first result is the number of bytes of directory entries the kernel copied into the buffer; the final 0 means
"no more entries". Before that, `ls` opens libraries and locale files, just like `hello` opened `libc.so.6`.

</details>

### Exercise 2 — `getpid` by number, with a test (medium)

Write `src/bin/raw-getpid.rs` with `fn raw_getpid() -> i64` that uses `libc::syscall(libc::SYS_getpid)`: the C
library's generic function that makes any system call by number. Print the number of `SYS_getpid` and both PIDs. Write a
test `raw_getpid_matches_std` that compares the result with `std::process::id()`.

<details>
<summary>Hint</summary>

`libc::syscall` is `unsafe` and returns a `c_long` (`i64` on 64-bit Linux). Compare with `i64::from(std::process::id())`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Asks the kernel for our PID with the generic `syscall` function of libc.
fn raw_getpid() -> i64 {
    // SAFETY: getpid takes no arguments and cannot fail.
    unsafe { libc::syscall(libc::SYS_getpid) }
}

fn main() {
    println!("SYS_getpid is number {} on this CPU", libc::SYS_getpid);
    println!("raw: {}, std: {}", raw_getpid(), std::process::id());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn raw_getpid_matches_std() {
        assert_eq!(raw_getpid(), i64::from(std::process::id()));
    }
}
```

```text
SYS_getpid is number 172 on this CPU
raw: 63, std: 63
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: both ask the same kernel the same question. `strace -e trace=getpid` shows two `getpid()` calls with the
same result. The test is a cheap guard: if you used a wrong number, it would fail.

</details>

### Exercise 3 — A `write_all` that handles partial writes (hard)

Write `fn write_all_fd<F: AsFd>(fd: F, buf: &[u8]) -> nix::Result<usize>` with `nix::unistd::write` in a loop. It must
write every byte, return how many `write` calls it needed, retry on `Errno::EINTR`, and return `Err(Errno::EIO)` if a
call writes 0 bytes. Write a test that writes 300,000 bytes into a file in `std::env::temp_dir()` and reads them back.

<details>
<summary>Hint</summary>

Keep `buf` as a shrinking slice: after `Ok(n)`, do `buf = &buf[n..]`. `AsFd` is in `std::os::fd`; a `&File` implements
it, so you can pass `&file`. Lesson 13.4 explains `AsFd` in detail.

</details>

<details>
<summary>Answer</summary>

```rust
use nix::errno::Errno;
use std::os::fd::AsFd;

/// Writes every byte of `buf`, even if the kernel accepts only a part per call.
fn write_all_fd<F: AsFd>(fd: F, mut buf: &[u8]) -> nix::Result<usize> {
    let mut calls = 0;
    while !buf.is_empty() {
        match nix::unistd::write(fd.as_fd(), buf) {
            Ok(0) => return Err(Errno::EIO), // the kernel accepted nothing: give up
            Ok(n) => {
                buf = &buf[n..]; // drop the part that was written
                calls += 1;
            }
            Err(Errno::EINTR) => continue, // interrupted by a signal: just try again
            Err(e) => return Err(e),
        }
    }
    Ok(calls)
}

fn main() -> nix::Result<()> {
    let calls = write_all_fd(std::io::stdout(), b"all bytes arrived\n")?;
    eprintln!("({calls} write call)");
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn writes_a_large_buffer_completely() {
        let path = std::env::temp_dir().join("p13-write-all-test.bin");
        let file = std::fs::File::create(&path).unwrap();
        let data = vec![7_u8; 300_000];
        write_all_fd(&file, &data).unwrap();
        drop(file);
        assert_eq!(std::fs::read(&path).unwrap(), data);
        std::fs::remove_file(&path).unwrap();
    }
}
```

```text
all bytes arrived
(1 write call)
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the loop only ends when the slice is empty, so a partial write simply leads to another call with the
remaining bytes. Files on a local disk usually take everything at once; pipes and network sockets (Lesson 13.6 and Phase 15)
often do not, and there this loop matters.

</details>

## Quiz

1. Why can a program not call a kernel function like a normal function?

<details>
<summary>Answer</summary>

The kernel's memory is protected and the CPU runs the program in the low privilege level. The only way in is the special
instruction (`svc` on arm64, `syscall` on x86-64), which switches the level and jumps to the kernel's fixed entry point.

</details>

2. On arm64 Linux, where do the system call number, the first argument and the result go?

<details>
<summary>Answer</summary>

The number goes into `x8`, the first argument into `x0`, and the result comes back in `x0`.

</details>

3. A libc function returned −1. Where do you find the reason, and when must you read it?

<details>
<summary>Answer</summary>

In `errno` (in Rust: `std::io::Error::last_os_error()`), immediately after the failing call, before any other call can
overwrite it.

</details>

4. What does `strace -c` show, and when is it useful?

<details>
<summary>Answer</summary>

A table of how many times each system call was made and how much time it took. It is useful to see whether a program
makes too many calls, for example one `write` per line.

</details>

5. Why does `BufWriter` make a program that prints many small lines faster?

<details>
<summary>Answer</summary>

It collects the lines in memory and sends them with a few large `write` calls instead of one call per line. Each system
call has a fixed cost, so fewer calls means less time at the border.

</details>

6. Why does `strace` not show a system call when Rust reads the clock with `Instant::now()`?

<details>
<summary>Answer</summary>

Because reading the clock runs in the vDSO, a small library the kernel maps into the process. The code runs in user space
and never crosses into the kernel.

</details>

## Summary

- A system call is a request to the kernel through one special instruction (`svc #0` on arm64, `syscall` on x86-64), with the number in `x8` and arguments in `x0`–`x5`.
- The same call can be made from assembly, `libc`, `nix` or `std`; use the highest level that does the job.
- Errors come back as negative numbers; libc puts them into `errno`, `nix` returns `Errno`, `std` returns `io::Error`.
- `strace` prints every system call of a program; `-e trace=`, `-c`, `-f` and `-p` are the most useful options.
- Even "hello" makes about 60 system calls, most of them for loading libraries.
- System calls are expensive; buffering with `BufWriter` and `BufReader` turns many small calls into a few big ones.

## Sources and further reading

- [syscall(2) — Linux manual page](https://man7.org/linux/man-pages/man2/syscall.2.html) — the register rules for every CPU type.
- [errno(3) — Linux manual page](https://man7.org/linux/man-pages/man3/errno.3.html) — every error name with its meaning.
- [strace(1) — Linux manual page](https://man7.org/linux/man-pages/man1/strace.1.html) — all options of `strace`.
- [The Rust Reference — Inline assembly](https://doc.rust-lang.org/reference/inline-assembly.html) — the full rules of `asm!`.
- [vdso(7) — Linux manual page](https://man7.org/linux/man-pages/man7/vdso.7.html) — which calls the vDSO provides.
- [nix crate documentation](https://docs.rs/nix/0.32.0/nix/) — the safe wrappers, grouped by feature.

---

[🏠 Start](../START-HERE.md) · [← 13.2 Processes from the inside](02-processes-from-the-inside.md) · [📋 Phase 13](README.md) · [13.4 Files and file descriptors →](04-files-and-file-descriptors.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
