<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.12 File systems from the inside](12-file-systems-inside.md) · [📋 Phase 13](README.md) · [13.14 io_uring →](14-io-uring.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/13-non-blocking-io-and-epoll.md) <!-- nav -->

# Lesson 13.13 — Non-blocking I/O and epoll

> **Phase 13 — Operating systems from the inside** · Lesson 13 of 18 · ⏱ about 135 minutes

## What you will learn

- Why a normal `read` *blocks*, and why that is a problem for a program that serves many inputs at once.
- How to put a file descriptor into non-blocking mode, and what `WouldBlock` (EAGAIN) means.
- How `epoll` lets one thread wait for many descriptors and wake up only when one of them is ready.
- The difference between level-triggered and edge-triggered notification, and why edge-triggered code must drain.
- How this "readiness" model is the base of async runtimes such as Tokio.

## Before you start

- You finished [Lesson 13.12 — File systems from the inside](12-file-systems-inside.md).
- You know file descriptors ([Lesson 13.4](04-files-and-file-descriptors.md)), pipes ([Lesson 13.6](06-pipes-and-redirection-inside.md)) and threads ([Lesson 13.9](09-threads-and-scheduling.md)).
- You have seen `async`/`await` once ([Lesson 06.9](../06-rust-advanced/09-async-await-introduction.md)); this lesson shows what is underneath.
- Create the package on the Mac and copy it to the VM as in Lesson 13.10:

```bash
cd ~/projects/rust
cargo new --lib code/p13-epoll
cd code/p13-epoll
cargo add nix@0.32 --features event,fs
mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **blocking call** | A system call that puts the thread to sleep until it can finish, for example `read` on an empty pipe. |
| **non-blocking mode** (`O_NONBLOCK`) | A flag on a file descriptor: calls that would have to wait return at once with an error instead. |
| **EAGAIN / `WouldBlock`** | The error a non-blocking call returns when it would have to wait ("try again later"). In Rust it is `io::ErrorKind::WouldBlock`. |
| **I/O multiplexing** | I/O means input/output. I/O multiplexing is watching many descriptors with one thread and handling whichever is ready. |
| **readiness** | The state "a call on this descriptor would not block now": data to read, or space to write. |
| **epoll** | The Linux interface for I/O multiplexing: you register descriptors once, then ask "which are ready?". |
| **interest list** | The set of descriptors (and events) that an epoll instance watches. |
| **level-triggered** (LT) | epoll reports a descriptor *as long as* it is ready. The default. |
| **edge-triggered** (ET) | epoll reports a descriptor only when it *becomes* ready (new data arrives). |
| **busy waiting** | Asking again and again in a loop whether something is ready, burning CPU (Central Processing Unit) time. |
| **event loop** | The main loop of a program that waits for events and handles them one after another. |

## 1. The problem with blocking

When you call `read` on a pipe that is empty, the kernel puts your thread to sleep until data arrives. This
is a **blocking call**. For one input it is perfect: no CPU is used while waiting.

Now imagine a program that must handle many inputs: a chat server with 10,000 connected users, or a tool that
watches the output of several child processes. A blocked thread can wait for only **one** descriptor. While
it sleeps on user A, a message from user B is not handled.

There are three ways out:

| Way | Idea | Problem |
|---|---|---|
| a thread per input | each thread blocks on its own descriptor | 10,000 threads cost memory (a stack each) and scheduling work (Lesson 13.9) |
| non-blocking + loop | ask every descriptor in turn "anything for me?" | **busy waiting**: the loop spins at 100% CPU when nothing happens |
| **I/O multiplexing** | ask the kernel "wake me when *any* of these is ready" | none of the above; this lesson |

The third way needs two building blocks: non-blocking descriptors and `epoll`.

## 2. Non-blocking mode

Every open file description has flags. The flag `O_NONBLOCK` changes the rule: a call that would have to
wait returns at once with the error **EAGAIN** ("resource temporarily unavailable, try again"). Rust maps it
to `io::ErrorKind::WouldBlock`. The standard library has no function to set this flag on a pipe, so we use
`fcntl` ("file control") from `nix`. Create `src/bin/nonblock.rs`:

```rust
use std::fs::File;
use std::io::{ErrorKind, Read, Write};

use nix::fcntl::{FcntlArg, OFlag, fcntl};
use nix::unistd::pipe;

fn main() -> std::io::Result<()> {
    let (read_end, write_end) = pipe()?;
    fcntl(&read_end, FcntlArg::F_SETFL(OFlag::O_NONBLOCK))?; // switch the read end to non-blocking

    let mut reader = File::from(read_end);
    let mut writer = File::from(write_end);
    let mut buf = [0u8; 16];

    match reader.read(&mut buf) {
        Err(e) if e.kind() == ErrorKind::WouldBlock => println!("empty pipe: {e}"),
        other => println!("unexpected: {other:?}"),
    }

    writer.write_all(b"hi")?;
    let n = reader.read(&mut buf)?;
    println!(
        "after a write: {n} bytes, {:?}",
        String::from_utf8_lossy(&buf[..n])
    );
    Ok(())
}
```

**What happens:**

1. `pipe()` returns two `OwnedFd` values (Lesson 13.6). `nix` errors convert into `io::Error`, so `?` works in
   a function that returns `io::Result`.
2. `fcntl(..., F_SETFL(O_NONBLOCK))` sets the flag on the read end.
3. `File::from(fd)` wraps a descriptor in a `File`, so we can use the normal `Read` and `Write` traits.
4. The first `read` finds the pipe empty. Instead of sleeping, it returns `WouldBlock` (os error 11 = EAGAIN).
5. After the write, `read` returns the two bytes at once.

```text
empty pipe: Resource temporarily unavailable (os error 11)
after a write: 2 bytes, "hi"
```

Non-blocking mode alone does not solve our problem. It only tells us "not now". We still need a way to
sleep until "now" comes. That is `epoll`.

## 3. epoll: waiting for many descriptors at once

`epoll` works in three steps, each a system call:

```text
1. epoll_create1()           -> a new epoll instance (itself a file descriptor)
2. epoll_ctl(ADD, fd, events) -> put fd on the interest list, once per descriptor
3. epoll_wait(timeout)        -> sleep until at least one fd is ready; get the list of ready ones

           interest list                 ready list
   epoll  +------------------+  data   +-------------+
   fd 3   | fd 4  EPOLLIN  0 | ------> | fd 6 (id 1) |  -> returned by epoll_wait
          | fd 6  EPOLLIN  1 | arrives +-------------+
          | fd 8  EPOLLIN  2 |
          +------------------+
```

The kernel keeps both lists. When data arrives on a watched descriptor, the kernel puts it on the ready list
and wakes the waiting thread. Waiting costs nothing, and the work per wake-up depends on how many descriptors
are *ready*, not on how many are watched. That is why one thread can watch tens of thousands of connections.

Each registration carries a number of your choice (`data`), which `epoll_wait` gives back. We use it as an
index into our own table of readers. Create `src/bin/watch3.rs`:

```rust
use std::fs::File;
use std::io::{Read, Write};
use std::thread;
use std::time::{Duration, Instant};

use nix::sys::epoll::{Epoll, EpollCreateFlags, EpollEvent, EpollFlags, EpollTimeout};
use nix::unistd::pipe;

fn main() -> std::io::Result<()> {
    let start = Instant::now();
    let epoll = Epoll::new(EpollCreateFlags::empty())?;
    let mut readers = Vec::new();

    // Three "workers": each writes one message into its own pipe after a delay, then closes it.
    for (id, delay_ms) in [(0u64, 300u64), (1, 100), (2, 200)] {
        let (read_end, write_end) = pipe()?;
        epoll.add(&read_end, EpollEvent::new(EpollFlags::EPOLLIN, id))?;
        readers.push(File::from(read_end));
        thread::spawn(move || {
            let mut writer = File::from(write_end);
            thread::sleep(Duration::from_millis(delay_ms));
            writer
                .write_all(format!("hello from worker {id}").as_bytes())
                .unwrap();
        }); // `writer` is dropped here: the write end is closed
    }

    let mut open = readers.len();
    let mut events = [EpollEvent::empty(); 8];
    while open > 0 {
        let ready = epoll.wait(&mut events, EpollTimeout::NONE)?; // sleep until something is ready
        for event in &events[..ready] {
            let id = event.data() as usize;
            let mut buf = [0u8; 64];
            let n = readers[id].read(&mut buf)?;
            let t = start.elapsed().as_millis();
            if n == 0 {
                println!("{t:>4} ms  pipe {id}: closed");
                epoll.delete(&readers[id])?;
                open -= 1;
            } else {
                println!(
                    "{t:>4} ms  pipe {id}: {}",
                    String::from_utf8_lossy(&buf[..n])
                );
            }
        }
    }
    Ok(())
}
```

**What happens:**

1. `Epoll::new` creates the instance (`epoll_create1`). `Epoll` owns its descriptor and closes it on drop.
2. For each worker we make a pipe and register the read end with `EPOLLIN` ("tell me when there is something
   to read") and the number `id`. The threads only simulate slow sources; in a real server these would be
   network connections (Phase 15).
3. `epoll.wait(&mut events, EpollTimeout::NONE)` sleeps with no time limit and fills `events` with up to 8
   ready descriptors. It returns how many it filled.
4. `event.data()` gives back our `id`. We read from that pipe. Because epoll said "ready", this `read` does
   not block.
5. A `read` that returns 0 means end of file: the worker closed its end. We remove the descriptor from the
   interest list with `epoll.delete` and count down.

```text
 100 ms  pipe 1: hello from worker 1
 100 ms  pipe 1: closed
 202 ms  pipe 2: hello from worker 2
 202 ms  pipe 2: closed
 303 ms  pipe 0: hello from worker 0
 303 ms  pipe 0: closed
```

One thread handled three sources in the order in which they became ready, not in the order in which they
were registered. The milliseconds vary a little from run to run.

`strace -f` (`-f` follows the threads too) shows the system calls; here are the first lines:

```bash
strace -f -e trace=epoll_create1,epoll_ctl,epoll_pwait ./target/debug/watch3
```

```text
epoll_create1(0)                        = 3
epoll_ctl(3, EPOLL_CTL_ADD, 4, {events=EPOLLIN, data=0}) = 0
strace: Process 284 attached
[pid   283] epoll_ctl(3, EPOLL_CTL_ADD, 6, {events=EPOLLIN, data=0x1}) = 0
strace: Process 285 attached
[pid   283] epoll_ctl(3, EPOLL_CTL_ADD, 8, {events=EPOLLIN, data=0x2}) = 0
strace: Process 286 attached
[pid   283] epoll_pwait(3, [{events=EPOLLIN, data=0x1}], 8, -1, NULL, 8) = 1
 102 ms  pipe 1: hello from worker 1
[pid   283] epoll_pwait(3, [{events=EPOLLHUP, data=0x1}], 8, -1, NULL, 8) = 1
 102 ms  pipe 1: closed
[pid   283] epoll_ctl(3, EPOLL_CTL_DEL, 6, NULL) = 0
```

Two details:

- On arm64 Linux there is no `epoll_wait` system call; the C library implements it with `epoll_pwait`
  (with no signal mask, `NULL`). `-1` is "no timeout".
- The second wake-up reports `EPOLLHUP` ("hang up": the other end was closed), although we asked only for
  `EPOLLIN`. epoll always reports hang-ups and errors.

## 4. Level-triggered and edge-triggered

When should epoll report a descriptor? There are two answers:

- **Level-triggered** (the default): *as long as* there is unread data. If you read only part of it, the
  next `wait` reports the descriptor again. Like a doorbell that keeps ringing while someone is at the door.
- **Edge-triggered** (`EPOLLET`): only when the state *changes*, that is, when new data arrives. If you
  leave data unread, epoll stays silent until even more data comes. Like a doorbell that rings once.

Let us see the difference. Create `src/bin/triggers.rs`:

```rust
use std::fs::File;
use std::io::{Read, Write};

use nix::fcntl::{FcntlArg, OFlag, fcntl};
use nix::sys::epoll::{Epoll, EpollCreateFlags, EpollEvent, EpollFlags};
use nix::unistd::pipe;

fn run(name: &str, flags: EpollFlags) -> std::io::Result<()> {
    let (read_end, write_end) = pipe()?;
    fcntl(&read_end, FcntlArg::F_SETFL(OFlag::O_NONBLOCK))?;
    let epoll = Epoll::new(EpollCreateFlags::empty())?;
    epoll.add(&read_end, EpollEvent::new(flags, 0))?;
    let mut reader = File::from(read_end);
    File::from(write_end).write_all(b"0123456789")?; // 10 bytes, write end closed at once

    println!("{name}:");
    let mut events = [EpollEvent::empty(); 1];
    loop {
        let ready = epoll.wait(&mut events, 100u16)?; // wait at most 100 ms
        if ready == 0 {
            println!("  no event within 100 ms");
            return Ok(());
        }
        let mut buf = [0u8; 4]; // read only 4 bytes per wake-up
        let n = reader.read(&mut buf)?;
        println!(
            "  woke up, read {n} bytes: {:?}",
            String::from_utf8_lossy(&buf[..n])
        );
        if n == 0 {
            return Ok(());
        }
    }
}

fn main() -> std::io::Result<()> {
    run("level-triggered", EpollFlags::EPOLLIN)?;
    run("edge-triggered", EpollFlags::EPOLLIN | EpollFlags::EPOLLET)?;
    Ok(())
}
```

**What happens:**

1. Ten bytes are written, and the write end is closed in the same line (the temporary `File` is dropped).
2. The timeout `100u16` means "wait at most 100 ms"; `wait` returns 0 if nothing happened.
3. Each wake-up reads only 4 bytes, on purpose.

```text
level-triggered:
  woke up, read 4 bytes: "0123"
  woke up, read 4 bytes: "4567"
  woke up, read 2 bytes: "89"
  woke up, read 0 bytes: ""
edge-triggered:
  woke up, read 4 bytes: "0123"
  no event within 100 ms
```

With level-triggered mode, epoll kept reporting the pipe until everything, including the end of file, was
read. With edge-triggered mode it reported the pipe once; six bytes are still waiting, and no new event will
ever come, because no new data arrives. A program written like this would hang forever.

The rule for edge-triggered mode: **after each event, read until you get `WouldBlock`** (this is called
*draining*), and therefore always use non-blocking descriptors with it. Why would anyone choose it? With
many threads waiting on one epoll, or very busy descriptors, it avoids repeated wake-ups for the same data.
High-performance libraries such as `mio` use it. For your own code, start with level-triggered.

> ⚡ **Performance:** epoll's cost grows with the number of *ready* descriptors per wake-up, not with the
> number you watch. The older calls `select` and `poll` pass the whole list to the kernel on every call, so
> they get slower as the list grows.

## 5. From epoll to async Rust

Look at the loop in `watch3` again: wait for events, find out whose event it is, run a small piece of code
for it, go back to waiting. This shape is an **event loop**, and it is exactly what an async runtime does:

```text
   your async tasks:   task A        task B        task C
                     .await read   .await read   .await sleep
                          \             |             /
                    +------------------------------------+
                    | runtime (Tokio): event loop        |
                    |   mio -> epoll_wait (Linux)        |
                    |         kqueue    (macOS)          |
                    +------------------------------------+
```

- The crate `mio` ("metal I/O") wraps epoll on Linux and kqueue (the equivalent) on macOS behind one API (Application Programming Interface: the set of functions a library offers).
- Tokio is built on `mio`. When an `async` function does `.await` on a socket that is not ready, Tokio
  registers the socket with epoll and runs other tasks. When epoll reports it, Tokio continues the task.

You will build a tiny runtime yourself in Phase 14
([Lesson 14.11](../14-concurrency-and-async/11-build-a-tiny-async-runtime.md)). Note one limit of the
readiness model: epoll tells you *when* you may read; you still do the `read` yourself. The next lesson,
[13.14](14-io-uring.md), shows a different model where the kernel does the work and tells you when it is
*done*.

## Common mistakes

1. **Busy waiting on `WouldBlock`.** A loop like `loop { match file.read(..) { Err(WouldBlock) => continue, … } }`
   works but keeps one CPU core at 100% while waiting. *Fix:* when you get `WouldBlock`, go back to
   `epoll.wait`.

2. **Edge-triggered without draining.** As `triggers` showed, the rest of the data is never reported.
   *Fix:* read until `WouldBlock` after each event (Exercise 3), or use level-triggered mode.

3. **Registering a regular file.** epoll works with pipes, sockets, terminals and similar, but not with
   regular files, which are always "ready". Running the program of Exercise 2 with `tick < input.txt` fails:

   ```text
   Error: Os { code: 1, kind: PermissionDenied, message: "Operation not permitted" }
   ```

   This is EPERM ("operation not permitted") from `epoll_ctl`. *Fix:* read regular files normally (or with io_uring, Lesson 13.14).

4. **Forgetting to remove closed descriptors.** If you keep a closed pipe on the interest list (or keep
   reading it), level-triggered epoll reports the end of file again and again. *Fix:* on `read` returning 0,
   call `epoll.delete` and drop the reader.

## Try it yourself

1. Run `watch3` and change the delays so that two workers finish at the same moment, for example
   `[(0, 200), (1, 200), (2, 100)]`. Does one `wait` return both? Print `ready` to see.
2. Run `ls -l /proc/$(pgrep watch3)/fd` from a second terminal while `watch3` waits (make the delays longer
   first, for example 10 seconds). One descriptor is shown as `anon_inode:[eventpoll]`: the epoll instance
   itself.
3. In `nonblock`, remove the `fcntl` line and run again. What happens at the first `read`? (Stop it with
   Ctrl-C.)

## Exercises

### Exercise 1 — A tested `try_read` (easy)

Write `pub fn try_read(source: &mut impl Read, buf: &mut [u8]) -> io::Result<Option<usize>>` in
`src/lib.rs`: `Ok(None)` for `WouldBlock`, `Ok(Some(n))` for data or end of file, other errors unchanged.
Test it with a non-blocking pipe: empty → `None`; after writing `abc` → `Some(3)`; after the writer is
dropped → `Some(0)`.

<details>
<summary>Hint</summary>

Write a test helper `fn nonblocking_pipe() -> (File, File)` that creates the pipe, sets `O_NONBLOCK` on the
read end and wraps both ends in `File`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io::{self, ErrorKind, Read};

/// Reads without waiting: `Ok(None)` means "no data right now, try later".
pub fn try_read(source: &mut impl Read, buf: &mut [u8]) -> io::Result<Option<usize>> {
    match source.read(buf) {
        Ok(n) => Ok(Some(n)),
        Err(e) if e.kind() == ErrorKind::WouldBlock => Ok(None),
        Err(e) => Err(e),
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use nix::fcntl::{FcntlArg, OFlag, fcntl};
    use nix::unistd::pipe;
    use std::fs::File;
    use std::io::Write;

    /// A pipe whose read end is non-blocking: (reader, writer).
    fn nonblocking_pipe() -> (File, File) {
        let (r, w) = pipe().unwrap();
        fcntl(&r, FcntlArg::F_SETFL(OFlag::O_NONBLOCK)).unwrap();
        (File::from(r), File::from(w))
    }

    #[test]
    fn empty_pipe_gives_none_then_data_gives_some() {
        let (mut reader, mut writer) = nonblocking_pipe();
        let mut buf = [0u8; 8];
        assert_eq!(try_read(&mut reader, &mut buf).unwrap(), None);
        writer.write_all(b"abc").unwrap();
        assert_eq!(try_read(&mut reader, &mut buf).unwrap(), Some(3));
        assert_eq!(&buf[..3], b"abc");
    }

    #[test]
    fn closed_pipe_gives_end_of_file() {
        let (mut reader, writer) = nonblocking_pipe();
        drop(writer);
        let mut buf = [0u8; 8];
        assert_eq!(try_read(&mut reader, &mut buf).unwrap(), Some(0));
    }
}
```

Why it works: `Option` separates the two normal situations ("nothing yet" and "got n bytes") from real
errors. `impl Read` makes the function work with any reader, which keeps it easy to test.

</details>

### Exercise 2 — `tick`: input with a timeout (medium)

Write `src/bin/tick.rs`. It registers standard input with epoll and waits at most one second at a time. If
no line arrives within a second, it prints `tick`. Each line is echoed as `you typed: <line>`. At end of
input it prints `bye` and exits. Test it with a pipe that delivers lines slowly:

```bash
(sleep 2.5; echo hello; sleep 0.3; echo world) | ./target/debug/tick
```

<details>
<summary>Hint</summary>

`io::stdin()` implements `AsFd`, so you can pass `&stdin` to `epoll.add`. A timeout of 1000 ms is
`1000u16`. Read lines with `stdin.lock().lines()`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io::{self, BufRead};

use nix::sys::epoll::{Epoll, EpollCreateFlags, EpollEvent, EpollFlags};

fn main() -> io::Result<()> {
    let stdin = io::stdin();
    let epoll = Epoll::new(EpollCreateFlags::empty())?;
    epoll.add(&stdin, EpollEvent::new(EpollFlags::EPOLLIN, 0))?;

    let mut events = [EpollEvent::empty(); 1];
    let mut lines = stdin.lock().lines();
    loop {
        if epoll.wait(&mut events, 1000u16)? == 0 {
            println!("tick"); // a whole second without input
            continue;
        }
        match lines.next() {
            Some(line) => println!("you typed: {}", line?),
            None => {
                println!("bye");
                return Ok(());
            }
        }
    }
}
```

```text
tick
tick
you typed: hello
you typed: world
bye
```

Why it works: the timeout turns "nothing happened" into a normal result (0 events). One caution: `lines()`
reads through a buffer. If two lines arrive in one chunk, the second one sits in *your* buffer, and epoll,
which only knows the kernel's pipe buffer, does not report it. Here the lines come 0.3 s apart, so it does not
happen; real event loops keep their own buffers and check them before waiting again.

</details>

### Exercise 3 — Drain for edge-triggered mode (hard)

Write `pub fn drain(source: &mut impl Read) -> io::Result<Vec<u8>>` that reads everything available now
(using your `try_read` and a small 4-byte buffer) and stops at `WouldBlock` or end of file. Test that after
writing `0123456789` one `drain` returns all ten bytes, and a second `drain` returns nothing while the writer
is still open. Then change `triggers` so the edge-triggered run uses `drain` and reads everything.

<details>
<summary>Hint</summary>

Loop: `match try_read(source, &mut buf)? { Some(0) | None => return Ok(all), Some(n) => … }`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Reads everything that is available now from a non-blocking source.
/// With edge-triggered epoll you must drain like this after every event.
pub fn drain(source: &mut impl Read) -> io::Result<Vec<u8>> {
    let mut all = Vec::new();
    let mut buf = [0u8; 4];
    loop {
        match try_read(source, &mut buf)? {
            Some(0) | None => return Ok(all), // end of file, or nothing more for now
            Some(n) => all.extend_from_slice(&buf[..n]),
        }
    }
}
```

The test, in the same `tests` module as Exercise 1:

```rust
#[test]
fn drain_reads_everything_available() {
    let (mut reader, mut writer) = nonblocking_pipe();
    writer.write_all(b"0123456789").unwrap();
    assert_eq!(drain(&mut reader).unwrap(), b"0123456789");
    assert_eq!(drain(&mut reader).unwrap(), b""); // nothing left, writer still open
}
```

`cargo test`:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the loop stops only when the kernel says "nothing more right now" (`WouldBlock`) or "nothing
ever again" (0). After that, any new data is a new *edge*, so epoll will report it. A blocking descriptor
would make the second `drain` sleep forever, which is why edge-triggered mode needs non-blocking mode.

</details>

## Quiz

1. What does a non-blocking `read` return when there is no data yet, and how does Rust show it?

<details>
<summary>Answer</summary>

It returns the error EAGAIN at once ("try again"). In Rust it is an `io::Error` whose `kind()` is
`ErrorKind::WouldBlock`.

</details>

2. Why is "one thread per connection" a poor design for 10,000 connections?

<details>
<summary>Answer</summary>

Every thread needs its own stack and kernel bookkeeping, and the scheduler must switch between them. Most
of them only sleep. One thread with epoll can wait for all connections at almost no cost.

</details>

3. What are the three epoll system calls and what does each do?

<details>
<summary>Answer</summary>

`epoll_create1` creates an instance; `epoll_ctl` adds, changes or removes a descriptor on its interest list;
`epoll_wait` (on arm64: `epoll_pwait`) sleeps until some descriptors are ready and returns them.

</details>

4. You read only part of the available data. What happens next with level-triggered and with edge-triggered
   mode?

<details>
<summary>Answer</summary>

Level-triggered: the next `wait` reports the descriptor again, because data is still there. Edge-triggered:
nothing is reported until new data arrives, so the rest can wait forever. Edge-triggered code must drain.

</details>

5. How is epoll related to Tokio?

<details>
<summary>Answer</summary>

Tokio's event loop uses `mio`, which uses epoll on Linux. A task that awaits a socket that is not ready is
parked; its socket is watched by epoll, and the task continues when epoll reports it ready.

</details>

## Summary

- A blocking call sleeps until it can finish; good for one input, bad for many.
- `O_NONBLOCK` makes calls return `WouldBlock` (EAGAIN) instead of sleeping; alone it leads to busy waiting.
- epoll: create an instance, register descriptors with a number of your choice, then `wait` for the ready ones; the cost depends on ready descriptors, not watched ones.
- Hang-ups (`EPOLLHUP`) are always reported; `read` returning 0 means end of file, so remove the descriptor.
- Level-triggered repeats while data remains; edge-triggered fires once per change and requires draining until `WouldBlock`.
- epoll is the readiness engine under `mio` and Tokio; regular files cannot be watched with it.

## Sources and further reading

- [epoll(7) — Linux manual page](https://man7.org/linux/man-pages/man7/epoll.7.html) — the full description, including level- versus edge-triggered mode and its pitfalls.
- [epoll_ctl(2) — Linux manual page](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html) — all event flags and errors such as EPERM for regular files.
- [nix::sys::epoll on docs.rs](https://docs.rs/nix/0.32.0/nix/sys/epoll/index.html) — the safe wrapper used in this lesson.
- [mio on docs.rs](https://docs.rs/mio/1.2.5/mio/) — the cross-platform event library under Tokio.
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — the chapter "Event-based Concurrency (Advanced)".

---

[🏠 Start](../START-HERE.md) · [← 13.12 File systems from the inside](12-file-systems-inside.md) · [📋 Phase 13](README.md) · [13.14 io_uring →](14-io-uring.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
