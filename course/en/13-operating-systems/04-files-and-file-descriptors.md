<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.3 System calls](03-system-calls.md) · [📋 Phase 13](README.md) · [13.5 Creating processes: fork, exec and wait →](05-fork-exec-and-wait.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/04-files-and-file-descriptors.md) <!-- nav -->

# Lesson 13.4 — Files and file descriptors

> **Phase 13 — Operating systems from the inside** · Lesson 4 of 18 · ⏱ about 120 minutes

## What you will learn

- What a file descriptor really is: a small number that points into a table inside the kernel.
- The three kernel tables behind every open file, and why they explain sharing and offsets.
- How to open, read, write, seek and close with the system calls themselves, using `nix`.
- How Rust makes descriptors safe: `OwnedFd`, `BorrowedFd`, `AsFd` and `RawFd`.
- What a descriptor leak is, how to see one in `/proc/self/fd`, and how Rust prevents it.

## Before you start

- You finished [Lesson 13.3 — System calls](03-system-calls.md) and know `strace`, `errno` and the `nix` crate.
- From [Lesson 11.3 — Text tools, pipes and redirection](../11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) you know the three streams: standard input (0), standard output (1) and standard error (2).
- From [Lesson 11.2 — The Linux file system](../11-linux-shell-and-containers/02-the-linux-file-system.md) you know that "everything is a file" and what permissions like `644` mean.

## New words

| Word | Meaning |
|---|---|
| **file descriptor** (fd) | A small non-negative number that a process uses to name one open file, pipe, socket or device. |
| **fd table** | The per-process table in the kernel; the file descriptor is an index into it. |
| **open file description** | The kernel record created by one `open` call: it holds the current **offset** and the flags (read-only, append …). Several descriptors can share one. |
| **inode** (index node) | The kernel's record of one file on disk: its type, size, owner, permissions and where its data lives. The file name is not inside it. |
| **offset** | The position in the file where the next `read` or `write` starts. |
| **seek** | Moving the offset with the system call `lseek`. |
| **`dup`** (duplicate) | A system call that makes a second descriptor for the same open file description. |
| **EOF** (End Of File) | The point where there is nothing more to read; `read` then returns 0. |
| **descriptor leak** | Forgetting to close descriptors, until the process reaches its limit and can open nothing more. |
| **`RawFd` / `OwnedFd` / `BorrowedFd`** | Rust types: a plain `i32` number / a descriptor you own and that is closed on drop / a borrowed one that you may use but not close. |
| **PID**, **KiB**, **I/O** | Known from earlier lessons: Process IDentifier; kibibyte (1,024 bytes); Input/Output, reading and writing data. |

## 1. A ticket for an open file

When you leave your coat in a theatre cloakroom you get a small ticket with a number. The ticket is not the coat; it
is only the number that lets the staff find your coat. A **file descriptor** is such a ticket. When a program opens a
file, the kernel keeps everything about the open file inside kernel memory and gives the program only a small number:
3, 4, 5 … Every later `read`, `write` or `close` names the file by that number.

Every process starts with three tickets already given out, by its parent (usually the shell):

| fd | Name | Usually connected to |
|---|---|---|
| 0 | standard input | the keyboard (the terminal) |
| 1 | standard output | the terminal |
| 2 | standard error | the terminal |

The kernel always gives out the **lowest free number**. That is why the first file a program opens gets 3.

A descriptor can point to much more than a disk file: a terminal, a pipe (Lesson 13.6), a network socket (Phase 15),
`/dev/null`, even a folder. For your program they all look alike: numbers you can `read` from and `write` to. This
is what "everything is a file" means in practice.

```bash
ls -l /proc/self/fd
```

Real output in a terminal (the line for fd 3 is the folder `ls` itself is reading):

```text
total 0
lrwx------ 1 root root 64 Nov 18 07:21 0 -> /dev/pts/0
lrwx------ 1 root root 64 Nov 18 07:21 1 -> /dev/pts/0
lrwx------ 1 root root 64 Nov 18 07:21 2 -> /dev/pts/0
lr-x------ 1 root root 64 Nov 18 07:21 3 -> /proc/1/fd
```

`/dev/pts/0` is your terminal ("pseudo-terminal slave number 0"). All three standard streams point to it.

## 2. Three tables in the kernel

Behind a descriptor there are three layers. Understanding them explains almost every surprise with files.

```text
 process A                        kernel                                       disk
 fd table                  open file descriptions                       inodes
 +----+---------+          +--------------------------+          +-------------------+
 | 0  | --------|--------->| /dev/pts/0, offset -, rw |--------->| terminal device    |
 | 1  | --------|----/     +--------------------------+          +-------------------+
 | 2  | --------|---/
 | 3  | --------|--------->| offset 7, flags O_RDWR   |---+----->| inode 1234        |
 | 4  | --------|---+      +--------------------------+   |      | size 14, rw-r--r--|
 +----+---------+   +----->| offset 0, flags O_RDONLY |---+      | data blocks ...   |
                           +--------------------------+          +-------------------+
```

1. **The fd table** belongs to one process. Entry 3 says "go to that open file description". It also holds one flag
   per descriptor, `FD_CLOEXEC` (Section 5).
2. **The open file description** is created by each `open` call. It remembers the **offset** (where the next read or
   write happens) and how the file was opened (read, write, append …).
3. **The inode** is the file itself: size, owner, permissions, data. File names live in folders and point to inodes
   (`ls -i` shows inode numbers).

In the picture, descriptors 3 and 4 opened the same file twice. They share the inode, so they see the same data, but
they have **different offsets**. Two descriptors can also share one open file description; you will see the difference
in Section 4.

## 3. Open, read, write, seek, close

`std::fs::File` does all of this for you, but it is worth doing it once with the system calls. Create
`src/bin/fd-basics.rs` (the `fs` feature of `nix` is already on from Lesson 13.3):

```rust
use nix::fcntl::{OFlag, open};
use nix::sys::stat::Mode;
use nix::unistd::{Whence, lseek, read, write};
use std::os::fd::AsRawFd;

fn main() -> nix::Result<()> {
    let flags = OFlag::O_RDWR | OFlag::O_CREAT | OFlag::O_TRUNC;
    let fd = open("/tmp/notes.txt", flags, Mode::from_bits_truncate(0o644))?;
    println!("open gave us descriptor {}", fd.as_raw_fd());

    let n = write(&fd, b"hello, kernel\n")?;
    println!(
        "wrote {n} bytes; offset is now {}",
        lseek(&fd, 0, Whence::SeekCur)?
    );

    lseek(&fd, 7, Whence::SeekSet)?; // move the offset back to byte 7
    let mut buf = [0_u8; 64];
    let n = read(&fd, &mut buf)?;
    println!(
        "read {n} bytes from offset 7: {:?}",
        String::from_utf8_lossy(&buf[..n])
    );

    let n = read(&fd, &mut buf)?;
    println!("next read gave {n} bytes: end of file");
    Ok(()) // `fd` is an OwnedFd: dropping it here calls close()
}
```

```text
open gave us descriptor 3
wrote 14 bytes; offset is now 14
read 7 bytes from offset 7: "kernel\n"
next read gave 0 bytes: end of file
```

**What happens:**

1. `open` takes a path, **flags** and a **mode**. `O_RDWR` = open for reading and writing, `O_CREAT` = create the file if
   it does not exist, `O_TRUNC` = make it empty if it does. The mode `0o644` (`rw-r--r--`, Lesson 11.4) is used only when
   the file is created. The `0o` prefix writes a number in octal (Lesson 12.1).
2. The result is an `OwnedFd` with number 3: 0, 1 and 2 were taken.
3. `write` puts 14 bytes at the offset 0 and moves the offset to 14. `lseek(&fd, 0, Whence::SeekCur)` means "move by 0
   from the current position" and returns the position: a common way to ask "where am I?".
4. `lseek(&fd, 7, Whence::SeekSet)` sets the offset to byte 7. `read` returns the 7 bytes from there to the end.
5. The next `read` returns **0**. For a file, 0 means EOF. It is not an error: it is how the kernel says "nothing more".
6. When `main` ends, `fd` is dropped, and `OwnedFd`'s destructor calls `close(3)`.

> ⚠️ **Common mistake:** Expecting `read` to fill the whole buffer. `read(fd, &mut buf)` returns **up to** `buf.len()`
> bytes: fewer if the file has fewer left, and often fewer for pipes and sockets. Always use the returned number, as in
> `&buf[..n]`, never the whole buffer.

## 4. Two opens versus one `dup`

Here is the experiment that shows why the middle table exists. Create `src/bin/offsets.rs`:

```rust
use std::fs::{self, File, OpenOptions};
use std::io::Write;

fn main() -> std::io::Result<()> {
    let path = "/tmp/offsets.txt";

    // Two separate open() calls: two open file descriptions, two offsets.
    let mut a = File::create(path)?;
    let mut b = OpenOptions::new().write(true).open(path)?;
    a.write_all(b"AAAA")?;
    b.write_all(b"BB")?;
    println!("two opens: {:?}", fs::read_to_string(path)?);

    // try_clone() calls dup(): two descriptors, ONE open file description, one shared offset.
    let mut c = File::create(path)?;
    let mut d = c.try_clone()?;
    c.write_all(b"AAAA")?;
    d.write_all(b"BB")?;
    println!("dup:       {:?}", fs::read_to_string(path)?);
    Ok(())
}
```

```text
two opens: "BBAA"
dup:       "AAAABB"
```

**What happens:**

1. `a` and `b` come from two `open` calls, so each has its own open file description and its own offset, both starting
   at 0. `a` writes `AAAA` at 0–3. `b` writes `BB` at **its** offset 0, over the first two `A`s: `BBAA`.
2. `File::try_clone` uses the `dup` system call (strictly, `fcntl` with `F_DUPFD_CLOEXEC`, which does the same plus the
   flag of Section 5). `c` and `d` are two numbers for **one** open file description, so they share one offset. After
   `c` writes 4 bytes, the shared offset is 4, and `d` continues there: `AAAABB`.

This is exactly what the shell relies on for `command > out.txt 2>&1`: standard output and standard error become two
descriptors for one open file description, so their texts follow each other instead of overwriting each other. Lesson 13.6
builds redirection like that.

> 💡 **Tip:** If several programs append to the same log file, open it with `O_APPEND` (`OpenOptions::new().append(true)`).
> Then the kernel moves the offset to the end **inside** every `write`, so lines from different processes never overwrite
> each other.

## 5. Descriptors in Rust: who closes them?

At the system call level a descriptor is just an `i32`. Nothing stops a C program from using a number after closing it,
closing it twice, or forgetting to close it. Rust puts descriptors under **ownership** (Phase 03), with four types in
`std::os::fd`:

| Type | What it is | Closes on drop? |
|---|---|---|
| `RawFd` | just `i32`, no rules | no |
| `OwnedFd` | a descriptor that you own | **yes** |
| `BorrowedFd<'a>` | a borrowed descriptor, valid for the lifetime `'a` | no |
| `File`, `TcpStream`, `Stdout` … | types that contain an `OwnedFd` (or borrow the process's 0, 1, 2) | as their owner says |

The trait `AsFd` means "you can borrow a `BorrowedFd` from me". `nix` functions take `impl AsFd`, which is why you could
pass `&fd`, `&file` or `std::io::stdout()` in the examples. Turning a raw number into an `OwnedFd` needs `unsafe`
(`OwnedFd::from_raw_fd`), because the compiler cannot check that you really own that number.

Watch ownership close descriptors. Create `src/bin/fd-table.rs`:

```rust
use std::fs::{self, File};

fn print_fds(when: &str) {
    println!("{when}:");
    let mut entries: Vec<_> = fs::read_dir("/proc/self/fd").unwrap().flatten().collect();
    entries.sort_by_key(|e| e.file_name().to_string_lossy().parse::<u32>().unwrap_or(0));
    for entry in entries {
        let target = fs::read_link(entry.path()).unwrap_or_default();
        println!(
            "  {} -> {}",
            entry.file_name().to_string_lossy(),
            target.display()
        );
    }
}

fn main() -> std::io::Result<()> {
    print_fds("at start");
    {
        let _passwd = File::open("/etc/passwd")?;
        let _hostname = File::open("/etc/hostname")?;
        print_fds("with two files open");
    } // both File values are dropped here, and their descriptors are closed
    print_fds("after the block");
    Ok(())
}
```

```text
at start:
  0 -> /dev/pts/0
  1 -> /dev/pts/0
  2 -> /dev/pts/0
  3 -> /proc/1/fd
with two files open:
  0 -> /dev/pts/0
  1 -> /dev/pts/0
  2 -> /dev/pts/0
  3 -> /etc/passwd
  4 -> /etc/hostname
  5 -> /proc/1/fd
after the block:
  0 -> /dev/pts/0
  1 -> /dev/pts/0
  2 -> /dev/pts/0
  3 -> /proc/1/fd
```

**What happens:**

1. Every entry of `/proc/self/fd` is a symbolic link from the number to what it points to; `fs::read_link` reads it.
2. The highest entry is the folder `/proc/self/fd` itself, opened by `read_dir` while we list it (here `self` resolved
   to PID 1, the program in the container).
3. Inside the block, the two files got 3 and 4, the lowest free numbers, and `read_dir` got 5.
4. At `}` the two `File` values go out of scope; their destructors call `close`, and the numbers are free again.

Run it with `./target/debug/fd-table < /etc/hostname 2>/dev/null` and fd 0 points to `/etc/hostname`, fd 2 to `/dev/null`:
the shell changed the table before starting the program. That is redirection, the topic of Lesson 13.6.

### Close on exec

When a process starts another program (Lesson 13.5), the new program **inherits** the fd table. Usually you do not want
to hand your open files, sockets or secrets to a child program. The flag `FD_CLOEXEC` ("close on exec") tells the
kernel to close that descriptor automatically when a new program is started. Rust's `std` sets it on every descriptor
it creates: that is the `O_CLOEXEC` you saw in every `openat` line of `strace` in Lesson 13.3.

> 🔒 **Security:** A leaked descriptor in a child process is a real class of security bug: a child program could read a
> secret file or write to a socket that only the parent should use. When you open descriptors with `nix` or `libc`, add
> `OFlag::O_CLOEXEC` unless the child really must inherit them.

## Common mistakes

> ⚠️ **Common mistake:** Leaking descriptors. In a long-running server, a descriptor that is never closed stays in the
> table forever. The kernel limits how many a process may have (`ulimit -n`, often 1024). This program leaks on purpose
> (`src/bin/leak.rs`):
>
> ```rust
> use std::fs::File;
>
> fn main() {
>     for i in 1.. {
>         match File::open("/etc/hostname") {
>             // BUG on purpose: forget() never runs the destructor, so close() is never called.
>             Ok(file) => std::mem::forget(file),
>             Err(e) => {
>                 println!("open number {i} failed: {e}");
>                 break;
>             }
>         }
>     }
> }
> ```
>
> ```bash
> (ulimit -n 10; ./target/debug/leak)
> ```
>
> ```text
> open number 8 failed: Too many open files (os error 24)
> ```
>
> Error 24 is `EMFILE`. With a limit of 10, descriptors 0–2 are taken and 3–9 are leaked by the first seven opens. The
> parentheses run the commands in a sub-shell, so the low limit does not stay in your shell. In normal Rust code, drop
> does the closing; leaks come from `mem::forget`, `into_raw_fd` or values kept forever in a collection.

> ⚠️ **Common mistake:** Using a descriptor after it was closed. In C this silently reads from whatever file got the same
> number later. In Rust a closed `OwnedFd` no longer exists, so the compiler rejects the use (`error[E0382]: borrow of
> moved value` after `drop(fd)`).

## Try it yourself

1. Run `ls -l /proc/self/fd` and then `ls -l /proc/self/fd < /etc/hostname > /tmp/out.txt; cat /tmp/out.txt`.
   Which descriptors changed?
2. Run `fd-basics`, then `ls -i /tmp/notes.txt` and `stat /tmp/notes.txt`. Find the inode number and the size 14.
3. Change `offsets.rs` to open `b` with `.append(true)` instead of `.write(true)` and explain the new result.
4. Run `strace -e trace=openat,close ./target/debug/fd-table`. Find the `close(3)` and `close(4)` at the end of the block.

## Exercises

### Exercise 1 — How many descriptors? (easy)

Write `src/bin/count-fds.rs` with `fn count_open_fds() -> usize` that counts the entries of `/proc/self/fd`, without
the one that `read_dir` itself uses. Print the count in `main`.

<details>
<summary>Hint</summary>

`fs::read_dir("/proc/self/fd").map(|d| d.count())` gives the number of entries, including the folder's own
descriptor. Subtract 1.

</details>

<details>
<summary>Answer</summary>

See Exercise 2: the same file, with a test added.

```text
open descriptors now: 3
```

Why it works: 0, 1 and 2 are open in every normal process; the descriptor of `read_dir` is subtracted.

</details>

### Exercise 2 — A test that proves `drop` closes (medium)

Add a test `a_file_uses_one_descriptor_until_dropped` to `count-fds.rs`: count the descriptors, open
`/proc/self/status`, check the count is one higher, drop the file, check the count is back.

<details>
<summary>Hint</summary>

Tests in one binary run in parallel threads of **one** process, and they share the fd table. With only one test in the
file this is fine; with several, another test could open a file at the same moment, and the test would sometimes fail.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs;

/// Counts the open file descriptors of this process.
fn count_open_fds() -> usize {
    // read_dir itself opens one descriptor for the folder; do not count it.
    fs::read_dir("/proc/self/fd")
        .map(|d| d.count())
        .unwrap_or(0)
        - 1
}

fn main() {
    println!("open descriptors now: {}", count_open_fds());
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::fs::File;

    #[test]
    fn a_file_uses_one_descriptor_until_dropped() {
        let before = count_open_fds();
        let file = File::open("/proc/self/status").unwrap();
        assert_eq!(count_open_fds(), before + 1);
        drop(file);
        assert_eq!(count_open_fds(), before);
    }
}
```

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `File::open` adds one entry to the fd table, and `drop` runs `close`, which removes it.

> 🧪 **Testing:** Tests that look at process-wide state (the fd table, the current directory, environment variables)
> can disturb each other, because `cargo test` runs tests in parallel threads. Keep such tests in their own binary, or
> run them with `cargo test -- --test-threads=1`.

</details>

### Exercise 3 — `cp` with system calls (hard)

Write `src/bin/copy.rs`: `copy <from> <to>` copies a file with `nix::fcntl::open`, `nix::unistd::read` and `write`, using a
64 KiB buffer. Put the loop into `fn copy_fd<S: AsFd, D: AsFd>(src: S, dst: D) -> nix::Result<(u64, u32)>` that returns the
number of bytes and the number of `read` calls. Write a test that copies 200,000 bytes and checks the content and that
exactly 5 `read` calls were needed. Then compare with `strace` against `std::fs::copy`.

<details>
<summary>Hint</summary>

Read until `read` returns 0. A `write` may write fewer bytes than asked, so write each chunk in an inner loop (as in
Lesson 13.3). 200,000 / 65,536 is 3.05, so there are four reads with data and one that returns 0.

</details>

<details>
<summary>Answer</summary>

```rust
use nix::fcntl::{OFlag, open};
use nix::sys::stat::Mode;
use nix::unistd::{read, write};
use std::os::fd::AsFd;

/// Copies everything from `src` to `dst` with a fixed buffer. Returns (bytes, read calls).
fn copy_fd<S: AsFd, D: AsFd>(src: S, dst: D) -> nix::Result<(u64, u32)> {
    let mut buf = vec![0_u8; 64 * 1024];
    let (mut total, mut reads) = (0_u64, 0_u32);
    loop {
        let n = read(src.as_fd(), &mut buf)?;
        reads += 1;
        if n == 0 {
            return Ok((total, reads)); // 0 means end of file
        }
        let mut chunk = &buf[..n];
        while !chunk.is_empty() {
            let written = write(dst.as_fd(), chunk)?;
            chunk = &chunk[written..];
        }
        total += n as u64;
    }
}

fn main() -> nix::Result<()> {
    let args: Vec<String> = std::env::args().collect();
    if args.len() != 3 {
        eprintln!("usage: copy <from> <to>");
        std::process::exit(2);
    }
    let src = open(args[1].as_str(), OFlag::O_RDONLY, Mode::empty())?;
    let flags = OFlag::O_WRONLY | OFlag::O_CREAT | OFlag::O_TRUNC;
    let dst = open(args[2].as_str(), flags, Mode::from_bits_truncate(0o644))?;
    let (bytes, reads) = copy_fd(&src, &dst)?;
    println!("copied {bytes} bytes with {reads} read calls");
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::fs;

    #[test]
    fn copies_a_file_larger_than_the_buffer() {
        let dir = std::env::temp_dir();
        let (from, to) = (dir.join("p13-copy-from"), dir.join("p13-copy-to"));
        let data: Vec<u8> = (0..200_000_u32).map(|i| (i % 251) as u8).collect();
        fs::write(&from, &data).unwrap();
        let src = fs::File::open(&from).unwrap();
        let dst = fs::File::create(&to).unwrap();
        let (bytes, reads) = copy_fd(&src, &dst).unwrap();
        assert_eq!(bytes, 200_000);
        assert_eq!(reads, 5); // 4 full or partial buffers + the final 0
        assert_eq!(fs::read(&to).unwrap(), data);
    }
}
```

```bash
head -c 1000000 /dev/urandom > /tmp/big
./target/debug/copy /tmp/big /tmp/big2
cmp /tmp/big /tmp/big2 && echo same
```

```text
copied 1000000 bytes with 17 read calls
same
```

With `strace`, a small program that calls `std::fs::copy("/tmp/big", "/tmp/big-copy")` shows no `read` and `write` for
the data at all:

```text
copy_file_range(3, NULL, 4, NULL, 1073741824, 0) = 1000000
copy_file_range(3, NULL, 4, NULL, 1073741824, 0) = 0
```

Why it works: your loop moves every byte through your buffer, in 17 rounds of `read` plus `write`. `std::fs::copy` uses
the Linux system call `copy_file_range`, which tells the kernel "copy from descriptor 3 to descriptor 4" and keeps the data
inside the kernel: one call instead of 34, and no copying into user space.

</details>

## Quiz

1. Your program opens two files. Which descriptor numbers does it most likely get, and why?

<details>
<summary>Answer</summary>

3 and 4. 0, 1 and 2 are already taken by the standard streams, and the kernel always gives the lowest free number.

</details>

2. What does `read` returning 0 mean for a file?

<details>
<summary>Answer</summary>

End of file: there is nothing more to read. It is not an error.

</details>

3. Two descriptors point to the same file. When do they share the offset, and when not?

<details>
<summary>Answer</summary>

They share it when they point to the same open file description, which happens after `dup` (or `try_clone`, or when a
child inherits it). Two separate `open` calls create two open file descriptions, each with its own offset.

</details>

4. What is the difference between `OwnedFd` and `BorrowedFd`?

<details>
<summary>Answer</summary>

`OwnedFd` owns the descriptor and closes it when dropped. `BorrowedFd<'a>` only borrows it for a lifetime; it can be used
but never closes the descriptor.

</details>

5. Why does `std` open every file with `O_CLOEXEC`?

<details>
<summary>Answer</summary>

So that the descriptor is closed automatically when the process starts another program. Otherwise the child program
would inherit open files and sockets it should not have.

</details>

6. A server fails after a few days with `Too many open files (os error 24)`. What is the likely bug, and how can you
   confirm it?

<details>
<summary>Answer</summary>

A descriptor leak: some descriptors are opened but never closed. Confirm it by watching `ls /proc/<pid>/fd | wc -l` grow
over time, and by looking at what the leaked entries point to.

</details>

## Summary

- A file descriptor is a small number, an index into the process's fd table; 0, 1 and 2 are the standard streams, and new descriptors get the lowest free number.
- Behind it are three layers: the fd table (per process), the open file description (offset and flags, created by `open`) and the inode (the file itself).
- `open`, `read`, `write`, `lseek` and `close` are the basic system calls; `read` may return fewer bytes than asked, and 0 means EOF.
- Two `open` calls give separate offsets; `dup` gives two descriptors with one shared offset.
- Rust's `OwnedFd` closes on drop, `BorrowedFd` borrows, and `AsFd` lets functions accept any type that has a descriptor.
- `FD_CLOEXEC` closes a descriptor when a new program starts; `std` sets it on everything it opens.
- `/proc/<pid>/fd` shows every open descriptor; it is the first place to look for a leak.

## Sources and further reading

- [open(2) — Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html) — all flags, and a clear explanation of open file descriptions.
- [dup(2) — Linux manual page](https://man7.org/linux/man-pages/man2/dup.2.html) — `dup`, `dup2` and `dup3`.
- [std::os::fd — Rust documentation](https://doc.rust-lang.org/std/os/fd/index.html) — `OwnedFd`, `BorrowedFd`, `AsFd` and the I/O safety rules.
- [RFC 3128 — I/O safety](https://rust-lang.github.io/rfcs/3128-io-safety.html) — the RFC (Request for Comments, a written proposal for a change to Rust) that explains why Rust treats descriptors like owned resources.
- [copy_file_range(2) — Linux manual page](https://man7.org/linux/man-pages/man2/copy_file_range.2.html) — copying inside the kernel.

---

[🏠 Start](../START-HERE.md) · [← 13.3 System calls](03-system-calls.md) · [📋 Phase 13](README.md) · [13.5 Creating processes: fork, exec and wait →](05-fork-exec-and-wait.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
