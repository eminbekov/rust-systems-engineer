<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.5 Creating processes: fork, exec and wait](05-fork-exec-and-wait.md) · [📋 Phase 13](README.md) · [13.7 Signals in your programs →](07-signals-in-your-programs.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/06-pipes-and-redirection-inside.md) <!-- nav -->

# Lesson 13.6 — Pipes and redirection from the inside

> **Phase 13 — Operating systems from the inside** · Lesson 6 of 18 · ⏱ about 135 minutes

## What you will learn

- What a pipe is inside the kernel: a small buffer with a read end and a write end.
- How end of file works for pipes, and why forgetting to close one descriptor makes a program hang forever.
- How `dup2` rewires standard input and output, and how the shell builds `ls | wc -l` and `> file 2>&1` from `pipe`, `fork`, `dup2` and `exec`.
- What `SIGPIPE` and `EPIPE` are, and why a Rust program piped into `head` can panic.
- How `std::process::Stdio` does the same work safely.

## Before you start

- You finished [Lesson 13.5 — Creating processes: fork, exec and wait](05-fork-exec-and-wait.md).
- You know the three kernel tables, `dup` and `O_CLOEXEC` from [Lesson 13.4](04-files-and-file-descriptors.md).
- From [Lesson 11.3 — Text tools, pipes and redirection](../11-linux-shell-and-containers/03-text-tools-pipes-and-redirection.md) you can *use* `|`, `>`, `<` and `2>&1` in the shell. Now you build them.

## New words

| Word | Meaning |
|---|---|
| **pipe** | A one-way channel inside the kernel: bytes written into its write end come out of its read end, in the same order. |
| **read end / write end** | The two file descriptors that `pipe` returns: one only for reading, one only for writing. |
| **pipe buffer** | The kernel memory that holds bytes written but not yet read; on Linux 64 KiB by default. |
| **blocking** | A call that puts the process to sleep until it can finish: `read` on an empty pipe, `write` on a full one. |
| **`dup2`** | A system call that makes descriptor number B point to the same open file description as A, closing B first if needed. |
| **redirection** | Changing what descriptors 0, 1 or 2 of a program point to before it starts. |
| **`EPIPE`** / **`SIGPIPE`** | The error / the signal you get when you write into a pipe whose read ends are all closed ("broken pipe"). |
| **`EAGAIN`** | "Try again": the error a non-blocking call returns instead of waiting. |
| **PID**, **KiB**/**MiB**, **I/O** | Known from earlier lessons: Process IDentifier; kibibyte and mebibyte (1,024 bytes, 1,024 KiB); Input/Output. |

## 1. A pipe is a buffer with two ends

Think of a pipe as a narrow tube between two rooms: notes dropped in at one end come out at the other, in the same order.
If nobody takes notes out, the tube fills up and the writer must wait.

The system call `pipe` creates such a tube inside the kernel and returns **two** descriptors. Create `src/bin/pipe-basic.rs`:

```rust
use nix::unistd::{pipe, read, write};
use std::os::fd::AsRawFd;

fn main() -> nix::Result<()> {
    let (reader, writer) = pipe()?;
    println!(
        "read end: fd {}, write end: fd {}",
        reader.as_raw_fd(),
        writer.as_raw_fd()
    );

    write(&writer, b"through the kernel\n")?;
    let mut buf = [0_u8; 64];
    let n = read(&reader, &mut buf)?;
    print!("got {n} bytes: {}", String::from_utf8_lossy(&buf[..n]));

    drop(writer); // close the only write end
    let n = read(&reader, &mut buf)?;
    println!("after closing the write end, read returns {n}: end of file");
    Ok(())
}
```

```text
read end: fd 3, write end: fd 4
got 19 bytes: through the kernel
after closing the write end, read returns 0: end of file
```

**What happens:**

1. `pipe()` returns two `OwnedFd`s: fd 3 can only be read, fd 4 can only be written.
2. `write` copies 19 bytes into the pipe buffer inside the kernel; `read` copies them out again. Bytes leave in the
   order they came in, and every byte is read once.
3. After `drop(writer)` no write end exists any more. A `read` on an empty pipe with no writers returns **0**: end of
   file, the same signal as at the end of a disk file.

The rules of a pipe, which explain everything in this lesson:

| Situation | What the kernel does |
|---|---|
| `read`, pipe has data | returns up to the requested bytes at once |
| `read`, pipe empty, a write end is still open **somewhere** | the reader **blocks** (sleeps) until data arrives |
| `read`, pipe empty, **all** write ends closed | returns 0 (end of file) |
| `write`, there is space in the buffer | copies the bytes and returns |
| `write`, buffer full (64 KiB) | the writer **blocks** until the reader makes space |
| `write`, **all** read ends closed | fails: `SIGPIPE` and `EPIPE` (Section 5) |

The word "somewhere" in the third row is the key. The kernel counts the write ends in **all processes**. As long as one
process anywhere still holds a write end, the reader waits.

> ⚡ **Performance:** The blocking rule gives pipelines automatic flow control. If `wc` is slower than `ls`, the buffer
> fills, `ls` sleeps, and memory use stays at 64 KiB however much data flows. This is a simple form of **backpressure**:
> a slow consumer slows down the producer instead of being flooded.

## 2. A pipe between two processes

A pipe in one process is not very useful. The point is that `fork` copies the fd table, so after `fork` the parent
and the child **both** hold both ends. Each side closes the end it does not need. Create `src/bin/pipe-fork.rs`:

```rust
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, fork, pipe, write};
use std::fs::File;
use std::io::{BufRead, BufReader};

fn main() -> nix::Result<()> {
    let (reader, writer) = pipe()?;

    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            drop(reader); // the child only writes
            for i in 1..=3 {
                write(&writer, format!("message {i} from the child\n").as_bytes())?;
            }
            std::process::exit(0); // exiting closes the child's write end
        }
        ForkResult::Parent { child } => {
            drop(writer); // the parent only reads: close its copy of the write end
            let lines = BufReader::new(File::from(reader)).lines();
            for line in lines {
                println!("parent got: {}", line.unwrap());
            }
            println!("parent: end of file, the child is done");
            waitpid(child, None)?;
        }
    }
    Ok(())
}
```

```text
parent got: message 1 from the child
parent got: message 2 from the child
parent got: message 3 from the child
parent: end of file, the child is done
```

**What happens:**

1. After `fork`, two write ends exist: the parent's and the child's. Both point to the same pipe.
2. The parent drops **its** write end at once. Now only the child holds one.
3. `File::from(reader)` turns the `OwnedFd` into a `File`, so we can use `BufReader::lines()` from Phase 04.
4. When the child exits, the kernel closes its descriptors. The last write end is gone, the parent's `read` returns 0,
   the `lines()` loop ends.

> ⚠️ **Common mistake:** Not closing the parent's write end. Delete `drop(writer);` in the parent arm and run the program:
> it prints the three messages and then hangs forever. The parent is waiting for end of file, but it holds a write end
> itself, so end of file can never come. Press Ctrl-C to stop it. Every hanging pipe program you will ever debug has
> this cause: some process still holds a write end.

## 3. `dup2`: rewiring descriptors 0, 1 and 2

Programs like `ls` and `wc` do not know about pipes. `ls` writes to descriptor 1, `wc` reads from descriptor 0. To connect
them, the child must make **its descriptor 1 point to the pipe** before it calls `exec`. That is the job of `dup2`.

`dup2(old, new)` makes descriptor `new` a copy of `old`: after the call, both numbers point to the same open file
description (Lesson 13.4). If `new` was open, it is closed first. `nix` has `dup2_stdin(fd)`, `dup2_stdout(fd)` and
`dup2_stderr(fd)` for the three standard numbers.

```text
 child of "ls", before dup2_stdout(writer)     after dup2_stdout(writer)
   fd 1 -> terminal                              fd 1 -> pipe (write end)
   fd 4 -> pipe (write end)                      fd 4 -> pipe (write end)  (closed by O_CLOEXEC at exec)
```

Now the shell's `ls /etc | wc -l`, built by hand. Create `src/bin/pipeline.rs`:

```rust
use nix::fcntl::OFlag;
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, Pid, dup2_stdin, dup2_stdout, execvp, fork, pipe2};
use std::ffi::CString;
use std::os::fd::OwnedFd;

/// Forks a child that connects the given descriptors to its stdin/stdout and runs `argv`.
fn spawn(argv: &[&str], stdin: Option<&OwnedFd>, stdout: Option<&OwnedFd>) -> nix::Result<Pid> {
    let args: Vec<CString> = argv.iter().map(|a| CString::new(*a).unwrap()).collect();
    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            if let Some(fd) = stdin {
                dup2_stdin(fd)?; // fd 0 now points to the pipe's read end
            }
            if let Some(fd) = stdout {
                dup2_stdout(fd)?; // fd 1 now points to the pipe's write end
            }
            let err = execvp(&args[0], &args).unwrap_err();
            eprintln!("{}: {err}", argv[0]);
            std::process::exit(127);
        }
        ForkResult::Parent { child } => Ok(child),
    }
}

fn main() -> nix::Result<()> {
    // ls /etc | wc -l
    // O_CLOEXEC: the original pipe descriptors close by themselves when a child calls exec.
    // Only the copies made by dup2 (fd 0 and fd 1) survive into `ls` and `wc`.
    let (reader, writer) = pipe2(OFlag::O_CLOEXEC)?;
    let left = spawn(&["ls", "/etc"], None, Some(&writer))?;
    let right = spawn(&["wc", "-l"], Some(&reader), None)?;
    drop(reader); // the parent must close both ends,
    drop(writer); // or `wc` never sees end of file
    println!("ls: {:?}", waitpid(left, None)?);
    println!("wc: {:?}", waitpid(right, None)?);
    Ok(())
}
```

```bash
./target/debug/pipeline
ls /etc | wc -l
```

```text
94
ls: Exited(Pid(379), 0)
wc: Exited(Pid(380), 0)
94
```

**What happens:**

1. `pipe2(OFlag::O_CLOEXEC)` is `pipe` with flags: both ends get `FD_CLOEXEC`.
2. The first child connects fd 1 to the write end and becomes `ls`. Its output goes into the pipe, not to the terminal.
3. The second child connects fd 0 to the read end and becomes `wc -l`, which counts the lines arriving through the pipe.
4. `dup2` does **not** copy the `FD_CLOEXEC` flag. So at `exec`, the original pipe descriptors (3 and 4) are closed
   automatically, while the copies on 0 and 1 stay open. Each program ends up with exactly the one end it needs.
5. The parent closes both ends and waits for both children. Our program and the real shell print the same 94.

Under `strace` you will see `dup3(4, 1, 0)` instead of `dup2`: arm64 Linux has no separate `dup2` system call, and the C library uses `dup3` with flags 0, which does the same.

> ⚠️ **Common mistake:** Forgetting `O_CLOEXEC` here. Replace `pipe2(OFlag::O_CLOEXEC)` with `pipe()` and run it with a
> time limit:
>
> ```bash
> timeout 3 ./target/debug/pipeline; echo "exit code $?"
> ```
>
> ```text
> ls: Exited(Pid(385), 0)
> exit code 124
> ```
>
> `ls` finishes, but `wc` never does, and `timeout` kills the program after 3 seconds (exit code 124). The reason: when the
> parent forked `wc`, it still held `writer`, so `wc` inherited a copy of the write end on fd 4, and without `FD_CLOEXEC`
> that copy survives `exec`. Now `wc` itself holds a write end of the pipe it reads from, and end of file can never come.

## 4. Redirection: `>`, `<` and `2>&1`

Redirection to a file is the same trick with a file instead of a pipe: open the file in the child, `dup2` it onto 0, 1
or 2, then `exec`. The shell command `ls -l /etc/hostname /nope > /tmp/out.txt 2>&1` becomes `src/bin/redirect.rs`:

```rust
use nix::fcntl::{OFlag, open};
use nix::sys::stat::Mode;
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, dup2_stderr, dup2_stdout, execvp, fork};
use std::ffi::CString;

fn main() -> nix::Result<()> {
    // The shell command:  ls -l /etc/hostname /nope > /tmp/out.txt 2>&1
    let args: Vec<CString> = ["ls", "-l", "/etc/hostname", "/nope"]
        .iter()
        .map(|a| CString::new(*a).unwrap())
        .collect();

    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            let flags = OFlag::O_WRONLY | OFlag::O_CREAT | OFlag::O_TRUNC | OFlag::O_CLOEXEC;
            let file = open("/tmp/out.txt", flags, Mode::from_bits_truncate(0o644))?;
            dup2_stdout(&file)?; // > /tmp/out.txt : fd 1 -> the file
            dup2_stderr(&file)?; // 2>&1           : fd 2 -> the same open file description
            let err = execvp(&args[0], &args).unwrap_err();
            eprintln!("exec failed: {err}");
            std::process::exit(127);
        }
        ForkResult::Parent { child } => {
            println!("child status: {:?}", waitpid(child, None)?);
            print!("{}", std::fs::read_to_string("/tmp/out.txt").unwrap());
        }
    }
    Ok(())
}
```

```text
child status: Exited(Pid(245), 2)
ls: cannot access '/nope': No such file or directory
-rw-r--r-- 1 root root 13 Nov 18 07:35 /etc/hostname
```

**What happens:**

1. The child opens the file **after** `fork`, so the parent's own descriptors stay untouched. The flags are exactly what
   the shell uses for `>`: write only, create, truncate.
2. `dup2_stdout(&file)` makes fd 1 point to the file; `dup2_stderr(&file)` makes fd 2 point to the **same open file
   description**, so both streams share one offset (Lesson 13.4) and their lines follow each other.
3. `ls` writes its error to fd 2 and its listing to fd 1; both end up in `/tmp/out.txt`. `ls` exits with code 2 because
   one path did not exist.

The other redirections follow the same pattern:

| Shell | In the child, before `exec` |
|---|---|
| `> file` | `open(file, O_WRONLY \| O_CREAT \| O_TRUNC)`, then `dup2` onto 1 |
| `>> file` | `open(file, O_WRONLY \| O_CREAT \| O_APPEND)`, then `dup2` onto 1 |
| `< file` | `open(file, O_RDONLY)`, then `dup2` onto 0 |
| `2>&1` | `dup2(1, 2)`: fd 2 becomes a copy of whatever fd 1 is **now** |
| `2> /dev/null` | `open("/dev/null", O_WRONLY)`, then `dup2` onto 2 |

> 💡 **Tip:** Order matters for `2>&1`, because it copies fd 1 **at that moment**. `cmd > file 2>&1` sends both streams to
> the file. `cmd 2>&1 > file` first copies fd 1 (the terminal) to fd 2, then points fd 1 to the file: errors still go to the
> terminal.

## 5. When the reader goes away: `SIGPIPE` and `EPIPE`

What happens if a program writes into a pipe and the reader has exited? Try it with an endless Rust program
(`src/bin/endless.rs`) piped into `head`:

```rust
fn main() {
    for i in 1..=u64::MAX {
        println!("line {i}");
    }
}
```

```bash
./target/debug/endless | head -2
echo "status: ${PIPESTATUS[0]}"
```

```text
line 1
line 2

thread 'main' (248) panicked at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/std/src/io/stdio.rs:1209:9:
failed printing to stdout: Broken pipe (os error 32)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
status: 101
```

**What happens:**

1. `head -2` prints two lines and exits. Its read end of the pipe is closed; no reader is left.
2. The next `write` of our program hits the rule "all read ends closed". The kernel sends the process the signal
   `SIGPIPE`, and the `write` fails with `EPIPE` ("broken pipe", error 32).
3. The default action of `SIGPIPE` is to kill the process silently. That is what happens to C programs like `cat`.
   But the Rust runtime sets `SIGPIPE` to "ignore" before `main` (you saw the `rt_sigaction` line in Lesson 13.3), so the
   program survives and gets the error `EPIPE` instead.
4. `println!` panics when writing fails, so the program dies with a panic message and exit code 101.
   `${PIPESTATUS[0]}` is a bash variable with the exit code of the first command of the last pipeline.

The fix for a command-line tool: write with `writeln!`, which returns a `Result`, and treat `BrokenPipe` as a normal end
(`src/bin/endless-quiet.rs`):

```rust
use std::io::{ErrorKind, Write};

fn main() {
    let mut out = std::io::stdout().lock();
    for i in 1.. {
        if let Err(e) = writeln!(out, "line {i}") {
            if e.kind() == ErrorKind::BrokenPipe {
                return; // the reader has gone away: stop quietly, like `yes` or `cat`
            }
            eprintln!("write error: {e}");
            std::process::exit(1);
        }
    }
}
```

```text
line 1
line 2
status: 0
```

> 🔒 **Security:** Ignoring `SIGPIPE` is the safer default for servers: a network client that disconnects must not kill the
> whole server with a signal. Lesson 13.7 shows how to change signal behaviour yourself.

## 6. The standard library way: `Stdio`

`std::process::Command` does all of this for you. `Stdio::piped()` creates a pipe, and `Stdio::from(...)` hands one end to
the next program:

```rust
use std::process::{Command, Stdio};

fn main() -> std::io::Result<()> {
    // ls /etc | wc -l, the standard library way
    let mut ls = Command::new("ls")
        .arg("/etc")
        .stdout(Stdio::piped())
        .spawn()?;
    let ls_out = ls.stdout.take().expect("stdout was piped");
    let wc = Command::new("wc")
        .arg("-l")
        .stdin(Stdio::from(ls_out))
        .output()?;
    ls.wait()?;
    print!("wc says: {}", String::from_utf8_lossy(&wc.stdout));
    Ok(())
}
```

```text
wc says: 94
```

`spawn` starts `ls` without waiting; `take()` moves its stdout pipe out of the `Child`, so it is closed in our process once
`wc` has it; `output()` runs `wc`, collects its output through another pipe and waits. `std` creates every pipe with
`O_CLOEXEC`, so the hang of Section 3 cannot happen. Save this file as `src/bin/std-pipeline.rs`.

## Common mistakes

> ⚠️ **Common mistake:** Waiting for the first command before starting to read. If the parent calls `waitpid` on a child
> that writes more than 64 KiB into a pipe, and only then reads, both wait forever: the child for space in the buffer, the
> parent for the child to end. Read first (or in another thread), then wait.

## Try it yourself

1. In `pipe-fork`, delete the parent's `drop(writer);` and watch it hang. Press Ctrl-C. Put the line back.
2. Run `ls -l /proc/self/fd | cat`. Which descriptor of `ls` is a pipe? (`1 -> pipe:[...]`.)
3. Change `redirect.rs` to implement `>>` (append) and run it twice. The file must contain both outputs.
4. Run `./target/debug/endless-quiet | head -3` and `yes | head -3`. Both stop quietly.

## Exercises

### Exercise 1 — How big is a pipe? (easy)

Create a pipe with `pipe2(OFlag::O_NONBLOCK)` and write 1 KiB chunks into it without reading, until `write` fails with
`Errno::EAGAIN`. Print how many bytes fit.

<details>
<summary>Hint</summary>

With `O_NONBLOCK`, a `write` into a full pipe does not sleep; it returns `Err(EAGAIN)`. Keep the read end alive
(`let (_reader, writer) = ...`), otherwise you get `EPIPE` instead.

</details>

<details>
<summary>Answer</summary>

```rust
use nix::errno::Errno;
use nix::fcntl::OFlag;
use nix::unistd::{pipe2, write};

fn main() -> nix::Result<()> {
    // O_NONBLOCK: when the pipe is full, write fails with EAGAIN instead of waiting.
    let (_reader, writer) = pipe2(OFlag::O_NONBLOCK)?;
    let chunk = [b'x'; 1024];
    let mut total = 0;
    loop {
        match write(&writer, &chunk) {
            Ok(n) => total += n,
            Err(Errno::EAGAIN) => break,
            Err(e) => return Err(e),
        }
    }
    println!(
        "the pipe was full after {total} bytes ({} KiB)",
        total / 1024
    );
    Ok(())
}
```

```text
the pipe was full after 65536 bytes (64 KiB)
```

Why it works: the default pipe buffer on Linux is 64 KiB. A program can make it bigger with `fcntl(F_SETPIPE_SZ)`, up
to the limit in `/proc/sys/fs/pipe-max-size` (1 MiB by default). Non-blocking I/O returns later, in Lesson 13.13.

</details>

### Exercise 2 — `capture`, safe for tests (medium)

Write `fn capture(argv: &[&str]) -> nix::Result<(String, i32)>` that runs a program with its standard output connected to a
pipe and returns everything it printed plus its exit code (a small `Command::output`). Use the careful child of
Lesson 13.5, Exercise 3: prepare everything before `fork`, and in the child call only `libc::dup2`, `libc::execvp` and
`libc::_exit`. Write three tests: `echo hi there` gives `("hi there\n", 0)`, `false` gives `("", 1)`, and
`seq 1 100000` (more than 64 KiB of output) gives 100,000 lines.

<details>
<summary>Hint</summary>

In the parent: drop the write end, read the read end to the end (`File::from(reader).read_to_string`), **then**
`waitpid`. Reading first is what makes the 100,000-line test pass. Use `pipe2(OFlag::O_CLOEXEC)`: tests run in threads,
so other tests' children could otherwise inherit your write end and keep the pipe open.

</details>

<details>
<summary>Answer</summary>

```rust
use nix::fcntl::OFlag;
use nix::sys::wait::{WaitStatus, waitpid};
use nix::unistd::{ForkResult, fork, pipe2};
use std::ffi::{CString, c_char};
use std::io::Read;
use std::os::fd::AsRawFd;

/// Runs `argv` and returns everything it wrote to standard output, plus its exit code.
fn capture(argv: &[&str]) -> nix::Result<(String, i32)> {
    let c_args: Vec<CString> = argv.iter().map(|a| CString::new(*a).unwrap()).collect();
    let mut ptrs: Vec<*const c_char> = c_args.iter().map(|a| a.as_ptr()).collect();
    ptrs.push(std::ptr::null());
    let (reader, writer) = pipe2(OFlag::O_CLOEXEC)?;
    let write_fd = writer.as_raw_fd();

    // SAFETY: the child only calls dup2, execvp and _exit, which are async-signal-safe.
    match unsafe { fork() }? {
        ForkResult::Child => unsafe {
            libc::dup2(write_fd, 1);
            libc::execvp(ptrs[0], ptrs.as_ptr());
            libc::_exit(127);
        },
        ForkResult::Parent { child } => {
            drop(writer); // keep only the child's copy of the write end
            let mut output = String::new();
            std::fs::File::from(reader)
                .read_to_string(&mut output)
                .map_err(|_| nix::errno::Errno::EIO)?;
            let code = match waitpid(child, None)? {
                WaitStatus::Exited(_, code) => code,
                _ => -1,
            };
            Ok((output, code))
        }
    }
}

fn main() -> nix::Result<()> {
    let (out, code) = capture(&["uname", "-s"])?;
    println!("uname printed {out:?} and exited with {code}");
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn captures_standard_output() {
        assert_eq!(
            capture(&["echo", "hi", "there"]).unwrap(),
            ("hi there\n".to_string(), 0)
        );
    }

    #[test]
    fn reports_the_exit_code_and_empty_output() {
        assert_eq!(capture(&["false"]).unwrap(), (String::new(), 1));
    }

    #[test]
    fn output_larger_than_the_pipe_buffer_arrives_completely() {
        let (out, code) = capture(&["seq", "1", "100000"]).unwrap();
        assert_eq!(code, 0);
        assert_eq!(out.lines().count(), 100_000);
    }
}
```

```text
uname printed "Linux\n" and exited with 0
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Why it works: the child's fd 1 is a copy of the write end, and its original write end closes at `exec`. The parent closed
its own copy, so when the child exits, no write end is left and `read_to_string` ends. Because the parent reads while the
child writes, the 64 KiB buffer never blocks the child for long.

</details>

### Exercise 3 — Read the shell's own pipeline (hard)

Run `strace -f -o /tmp/t -e trace=pipe2,clone,dup3,close,execve bash -c 'ls /etc | wc -l'` and read `/tmp/t`
(`-o` writes the trace into a file; each line starts with the PID). Answer: does bash use `O_CLOEXEC` for its pipe? How does
it make sure that `wc` sees end of file? In which process, and in which order, are 3 and 4 closed?

<details>
<summary>Hint</summary>

Look at the flags of `pipe2`, then follow each PID separately: `grep -E '^[0-9]+ +(pipe2|clone|dup3|close\((3|4)\)|execve)' /tmp/t`.
The library-loading `close(3)` lines at the start belong to bash itself; ignore them.

</details>

<details>
<summary>Answer</summary>

The interesting lines, sorted by process and shortened:

```text
10    pipe2([3, 4], 0)                  = 0
10    clone(...)                        = 11
10    close(4)                          = 0
10    clone(...)                        = 12
10    close(3)                          = 0
11    close(3)                          = 0
11    dup3(4, 1, 0)                     = 1
11    close(4)                          = 0
11    execve("/usr/bin/ls", ["ls", "/etc"], ...) = 0
12    dup3(3, 0, 0)                     = 0
12    close(3)                          = 0
12    execve("/usr/bin/wc", ["wc", "-l"], ...) = 0
```

Bash (PID 10) creates the pipe **without** `O_CLOEXEC` (flags `0`) and closes every extra descriptor by hand instead:

1. Right after forking the left child (11), bash closes its write end 4. So when it forks the right child (12), there is no
   write end left to inherit — the hang of Section 3 cannot happen.
2. After forking the right child, bash closes the read end 3. Bash itself now holds no end of the pipe.
3. The left child closes the read end it does not need, makes fd 1 a copy of 4, closes 4, and becomes `ls`.
4. The right child makes fd 0 a copy of 3, closes 3, and becomes `wc`.

Both approaches work: close by hand in the right order, or let `O_CLOEXEC` close the originals at `exec`. The shell of
Lesson 13.8 must handle pipelines of any length, so choose one approach and apply it to every pipe.

</details>

## Quiz

1. A pipe is empty. When does `read` block, and when does it return 0?

<details>
<summary>Answer</summary>

It blocks while at least one write end is still open in any process. It returns 0 (end of file) when all write ends are closed.

</details>

2. What does `dup2(4, 1)` do?

<details>
<summary>Answer</summary>

It makes descriptor 1 point to the same open file description as descriptor 4, closing what fd 1 pointed to before. After that,
everything the program writes to standard output goes where fd 4 goes.

</details>

3. Why must the shell call `dup2` in the child and not in itself?

<details>
<summary>Answer</summary>

Because `dup2` changes the fd table of the process that calls it. In the child it prepares only the new program's descriptors;
in the shell it would redirect the shell's own input or output.

</details>

4. A program reads from a pipe and hangs after all data has arrived. What is the most likely cause?

<details>
<summary>Answer</summary>

Some process still holds a write end of the pipe — often the reader itself or a sibling that inherited it without
`FD_CLOEXEC`. While any write end is open, end of file cannot come.

</details>

5. Why does a Rust program get `EPIPE` instead of being killed by `SIGPIPE` like a C program?

<details>
<summary>Answer</summary>

The Rust runtime sets `SIGPIPE` to "ignore" before `main` runs. An ignored `SIGPIPE` does not kill the process, so the `write`
just returns the error `EPIPE`.

</details>

## Summary

- A pipe is a kernel buffer (64 KiB) with a read end and a write end; `read` blocks when it is empty, `write` blocks when it is full.
- End of file comes only when **all** write ends in all processes are closed; one forgotten descriptor makes the reader hang.
- After `fork` both processes hold both ends; each closes the end it does not use.
- `dup2` makes 0, 1 or 2 point to a pipe or a file; `dup2` does not copy `FD_CLOEXEC`, so create pipes with `O_CLOEXEC` and the originals close at `exec`.
- Redirection `>`, `>>`, `<`, `2>&1` is `open` plus `dup2` in the child before `exec`.
- Writing to a pipe without readers gives `SIGPIPE` and `EPIPE`; Rust ignores `SIGPIPE`, so handle `ErrorKind::BrokenPipe` in command-line tools.
- `Stdio::piped()` and `Stdio::from` do the same safely in `std`.

## Sources and further reading

- [pipe(7) — Linux manual page](https://man7.org/linux/man-pages/man7/pipe.7.html) — capacity, blocking, `SIGPIPE` and atomic writes.
- [pipe(2) — Linux manual page](https://man7.org/linux/man-pages/man2/pipe.2.html) — `pipe` and `pipe2`.
- [dup(2) — Linux manual page](https://man7.org/linux/man-pages/man2/dup.2.html) — `dup2` and `dup3`.
- [GNU Bash Reference Manual — Redirections](https://www.gnu.org/software/bash/manual/html_node/Redirections.html) — every redirection the shell knows.
- [std::process::Stdio — Rust documentation](https://doc.rust-lang.org/std/process/struct.Stdio.html) — pipes in the standard library.

---

[🏠 Start](../START-HERE.md) · [← 13.5 Creating processes: fork, exec and wait](05-fork-exec-and-wait.md) · [📋 Phase 13](README.md) · [13.7 Signals in your programs →](07-signals-in-your-programs.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
