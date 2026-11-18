<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.15 Project: a concurrent job queue](15-project-concurrent-job-queue.md) · [📋 Phase 14](README.md) · [Phase 15 — Networking →](../15-networking/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/16-checkpoint-stage-3-complete.md) <!-- nav -->

# Lesson 14.16 — Checkpoint: Stage 3 complete

> **Phase 14 — Concurrency and async in depth** · Lesson 16 of 16 · ⏱ about 105 minutes

## What you will learn

- Whether you really own the material of Stage 3 (Phases 12–14): how data looks in memory, how the operating system runs your programs, and how concurrent and async code works inside.
- How to combine these topics in four small practical tasks, each with a test.
- Which lessons to review, with direct links.
- What Stage 4 (building real systems, starting with networking) expects from you.

## Before you start

- You finished [Lesson 14.15 — Project: a concurrent job queue](15-project-concurrent-job-queue.md) and merged the `p14-jobqueue` branch.
- Plan about 60 minutes for the quiz and 90 minutes for the tasks. Answer the quiz **from memory**, on paper or in your notes, and only then open the hidden answers.
- The tasks go into one new package, `~/projects/rust/code/p14-checkpoint`:

```bash
cd ~/projects/rust
cargo new --lib code/p14-checkpoint
cd code/p14-checkpoint
cargo add tokio@1 --features full
cargo add rayon@1
cargo add --dev tokio@1 --features test-util
mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A lesson that checks what you learned, instead of teaching something new. |
| **Stage 3** | "How computers really work": Phase 12 (computer systems), Phase 13 (operating systems from the inside) and Phase 14 (concurrency and async in depth). |
| **retrospective** | A short look back: what went well, what was hard, what to change. |

## Stage 3 at a glance

| Phase | What you learned | Your main project |
|---|---|---|
| 12 Computer systems | Bits and number systems, two's complement, floating point, bit tricks, UTF-8, how the CPU (Central Processing Unit) runs instructions, assembly, caches, memory layout of Rust types, linking, allocators | Your own memory allocator |
| 13 Operating systems from the inside | Processes, system calls, file descriptors, `fork`/`exec`/`wait`, pipes, signals, threads and scheduling, virtual memory, `mmap`, file systems, epoll, io_uring, namespaces, cgroups | Your own shell; a mini container runtime |
| 14 Concurrency and async in depth | Races and deadlocks, mutexes, condition variables, thread pools and rayon, atomics, memory ordering, a spin lock, lock-free structures, loom, futures and wakers, a tiny runtime, `Pin`, Tokio tasks and channels, cancellation and graceful shutdown | A concurrent job queue |

## Self-test quiz

Answer from memory. Each answer is hidden; open it after you wrote yours.

**Phase 12: computer systems**

1. How is the `i8` value −1 stored in memory, and what happens with `127_i8 + 1` in a debug build and in a release build?

<details>
<summary>Answer</summary>

In two's complement, −1 is all bits set: `0b1111_1111` (`0xFF`). `127_i8 + 1` overflows: a debug build panics with "attempt to add with overflow"; a release build wraps around to −128 (unless overflow checks are turned on). Use `checked_add`, `wrapping_add` or `saturating_add` to say what you want.

</details>

2. Why is `0.1 + 0.2 == 0.3` false for `f64`, and how do you compare floating-point numbers?

<details>
<summary>Answer</summary>

`0.1`, `0.2` and `0.3` have no exact binary representation, so each is rounded, and the rounding errors do not cancel. Compare with a tolerance: `(a - b).abs() < 1e-9` (choose the tolerance for your problem).

</details>

3. What is false sharing, and how do you avoid it?

<details>
<summary>Answer</summary>

Two threads write to *different* variables that lie in the *same* cache line. Each write takes the whole line away from the other core, so the line bounces between cores and both threads slow down, although they share no data. Fix: put each hot variable in its own cache line (pad or align it, for example with `crossbeam_utils::CachePadded`). Cache lines are 64 bytes on most x86-64 CPUs and 128 bytes on Apple M-series CPUs (`sysctl hw.cachelinesize` prints `128` on an M4 Mac).

</details>

4. Why is a struct with the fields `u8`, `u32`, `u16` 8 bytes in Rust, but 12 bytes with `#[repr(C)]`?

<details>
<summary>Answer</summary>

Each field must start at an address that is a multiple of its alignment (`u32`: 4). With `#[repr(C)]` the fields stay in the written order: `u8` + 3 bytes of padding + `u32` + `u16` + 2 bytes of padding to make the size a multiple of 4 = 12. Plain Rust may reorder fields (`u32`, `u16`, `u8`, 1 byte of padding) = 8. Practical task 1 checks this with a test.

</details>

**Phase 13: operating systems**

5. What is a system call? Name three.

<details>
<summary>Answer</summary>

A request from a program to the kernel, for something the program may not do itself: open a file, read, write, create a process, map memory. Examples: `openat`, `read`, `write`, `clone`/`fork`, `execve`, `mmap`. The CPU switches into kernel mode for the call and back afterwards, which costs time.

</details>

6. What is a file descriptor, and what are 0, 1 and 2?

<details>
<summary>Answer</summary>

A small integer that a process uses to refer to an open file, pipe or socket; it is an index into the process's table of open files kept by the kernel. 0 is standard input, 1 standard output, 2 standard error.

</details>

7. What do `fork`, `exec` and `wait` do, and what is a zombie process?

<details>
<summary>Answer</summary>

`fork` creates a copy of the current process. `exec` replaces the program in the current process with another program. `wait` lets the parent collect the exit status of a finished child. A zombie is a child that has finished but whose status nobody has collected yet; it keeps an entry in the process table until the parent waits for it.

</details>

8. What is a page fault, and is it always an error?

<details>
<summary>Answer</summary>

The CPU raises it when a program touches a virtual address whose page is not (yet) mapped to physical memory. It is usually not an error: the kernel loads the page (from a file, from swap, or a fresh zero page) and the program continues. Only an access to an address that the process may not use becomes a crash (a segmentation fault).

</details>

9. What problem does epoll solve?

<details>
<summary>Answer</summary>

A server with thousands of connections cannot block on each one, and asking each one in turn ("any data yet?") wastes CPU. epoll lets one thread register many file descriptors and sleep until **any** of them is ready, then tells it which ones. Async runtimes such as Tokio build their reactor on it (kqueue on macOS).

</details>

10. What is the difference between a namespace and a cgroup?

<details>
<summary>Answer</summary>

A namespace changes what a process can **see** (its own process ids, network, mount points, host name). A cgroup limits what a process can **use** (CPU time, memory, number of processes). A container uses both.

</details>

**Phase 14: concurrency and async**

11. What is the difference between a data race and a race condition, and which one does safe Rust prevent?

<details>
<summary>Answer</summary>

A data race: two threads access the same memory at the same time, at least one writes, and there is no synchronisation; it is undefined behaviour. Safe Rust prevents it at compile time (`Send`, `Sync`, borrowing rules). A race condition: the result depends on timing, for example a lost update between a `load` and a `store`; it can happen in perfectly safe code, and only design and testing (loom) catch it.

</details>

12. What do `Release` on a store and `Acquire` on a load guarantee, and why does it matter on your Mac?

<details>
<summary>Answer</summary>

If an `Acquire` load sees the value written by a `Release` store, then everything the writing thread did before the store is visible to the reading thread after the load. arm64 processors are weakly ordered: without this pair, another core may see the flag before the data. A test with `Relaxed` really fails on an M4 Mac now and then, while it might never fail on another machine.

</details>

13. What does loom do, and why must loom tests be small?

<details>
<summary>Answer</summary>

It runs a test once for every possible interleaving of the loom threads (and simulates the memory model for atomics), so it finds bugs that appear only in rare schedules. The number of schedules grows extremely fast with each extra thread or operation (4 iterations for two threads, 253 for three in Lesson 14.9), so models must be tiny, or use `LOOM_MAX_PREEMPTIONS`.

</details>

14. What must a future do before it returns `Poll::Pending`?

<details>
<summary>Answer</summary>

Make sure the waker from `cx.waker()` will be woken when progress is possible: store a clone where the event source will find it (and do the check and the store without a gap, so no wake-up is lost). The executor polls the task again only after a wake.

</details>

15. Why does `poll` take `self: Pin<&mut Self>`?

<details>
<summary>Answer</summary>

Futures from `async fn` may contain references to their own fields after the first poll (a local borrowed across an `.await`). Moving such a future would leave those references dangling. `Pin` promises that the future will not be moved again, so polling is safe.

</details>

16. A `select!` loop races `read_pair(&mut rx)` against a timer. Why can it lose messages, and what are two fixes?

<details>
<summary>Answer</summary>

When the timer wins, `select!` drops the `read_pair` future, together with a first message it may already have received: `read_pair` is not cancel-safe. Fixes: put only cancel-safe futures (like `rx.recv()`) into the `select!` and keep partial state in the loop; or create the future once, pin it, and pass `&mut` to `select!` so it is not dropped.

</details>

17. What is the difference between `token.cancel()` and `handle.abort()`?

<details>
<summary>Answer</summary>

`cancel()` on a `CancellationToken` is a request: tasks that watch `cancelled()` stop in their own way and can clean up. `abort()` forces one task: its future is dropped at its next `.await`. A graceful shutdown asks first, waits with a time limit, and aborts only what is left.

</details>

18. Name two ways to prevent a deadlock between two mutexes.

<details>
<summary>Answer</summary>

Take locks in one global order (for example always the lower account id first), or never hold one lock while taking another (copy the data you need, release, then take the next lock). Shorter critical sections and `try_lock` with a fallback also help. loom finds the deadlocks you missed.

</details>

## Practical tasks

Each task names the lessons it combines. Write the code yourself first; the answers are hidden.

### Task 1 — Predict memory layouts (Phase 12)

In `tests/layout.rs`, define the structs `Reordered { a: u8, b: u32, c: u16 }` and the same fields with `#[repr(C)]` as `InOrder`, and an enum `Shape { Dot(u8), Line(u64) }`. **First** write down your predictions for `size_of` of: `Reordered`, `InOrder`, `Option<Box<u8>>`, `Option<u32>`, `Shape`, `&str`, `String` (on your 64-bit Mac). Then turn the predictions into one test with `assert_eq!` and run `cargo test --test layout`.

<details>
<summary>Hint</summary>

Think about alignment and padding (Lesson 12.11), and about the *niche* of `Box`: a `Box` is never null, so `Option` can use the null value for `None` without any extra byte. `u32` has no such forbidden value. `&str` is a pointer plus a length; `String` is a pointer, a capacity and a length.

</details>

<details>
<summary>Answer</summary>

```rust
use std::mem::size_of;

#[allow(dead_code)]
struct Reordered {
    a: u8,
    b: u32,
    c: u16,
}

#[allow(dead_code)]
#[repr(C)]
struct InOrder {
    a: u8,
    b: u32,
    c: u16,
}

#[allow(dead_code)]
enum Shape {
    Dot(u8),
    Line(u64),
}

#[test]
fn layout_matches_my_predictions() {
    assert_eq!(size_of::<Reordered>(), 8);
    assert_eq!(size_of::<InOrder>(), 12);
    assert_eq!(size_of::<Option<Box<u8>>>(), 8);
    assert_eq!(size_of::<Option<u32>>(), 8);
    assert_eq!(size_of::<Shape>(), 16);
    assert_eq!(size_of::<&str>(), 16);
    assert_eq!(size_of::<String>(), 24);
}
```

`cargo test --test layout` shows `test result: ok. 1 passed`. Why it works: `Reordered` is reordered by the compiler to 4 + 2 + 1 + 1 padding = 8; `InOrder` keeps the order, 12 (quiz question 4). `Option<Box<u8>>` uses the null niche: 8. `Option<u32>` needs a separate tag, padded to the alignment of `u32`: 8. `Shape` has a tag and a `u64` aligned to 8: 16. `&str` is 2 × 8, `String` 3 × 8.

</details>

### Task 2 — A command with a time limit (Phases 13 and 14)

Write `async fn run_limited(program: &str, args: &[&str], limit: Duration) -> String` in `src/bin/run_limited.rs`. It starts the program as a child process with `tokio::process::Command`, captures standard output, and waits at most `limit`. It returns a line with the exit code and the output, or "gave up after … ms, child killed". Try `echo hello`, `sh -c "exit 3"` and `sleep 5` with a limit of 500 ms. Make sure the `sleep` process really dies.

<details>
<summary>Hint</summary>

`Command::new(program).args(args).stdout(Stdio::piped()).kill_on_drop(true).spawn()` gives a `Child`; `child.wait_with_output()` is a future. Wrap it in `tokio::time::timeout`. On timeout, the future (and with it the `Child`) is dropped; `kill_on_drop(true)` then sends a kill signal to the process (Lesson 13.7). Check with `pgrep -f "sleep 5"` right after the program ends.

</details>

<details>
<summary>Answer</summary>

```rust
use std::process::Stdio;
use std::time::Duration;
use tokio::process::Command;
use tokio::time::timeout;

/// Runs `program` with `args`, but at most for `limit`.
async fn run_limited(program: &str, args: &[&str], limit: Duration) -> String {
    let child = Command::new(program)
        .args(args)
        .stdout(Stdio::piped())
        .kill_on_drop(true) // dropping the future kills the child
        .spawn()
        .expect("could not start the program");
    match timeout(limit, child.wait_with_output()).await {
        Ok(Ok(output)) => format!(
            "exit code {:?}, output {:?}",
            output.status.code(),
            String::from_utf8_lossy(&output.stdout).trim()
        ),
        Ok(Err(error)) => format!("error: {error}"),
        Err(_) => format!("gave up after {} ms, child killed", limit.as_millis()),
    }
}

#[tokio::main]
async fn main() {
    let limit = Duration::from_millis(500);
    println!("{}", run_limited("echo", &["hello"], limit).await);
    println!("{}", run_limited("sh", &["-c", "exit 3"], limit).await);
    println!("{}", run_limited("sleep", &["5"], limit).await);
}
```

```text
exit code Some(0), output "hello"
exit code Some(3), output ""
gave up after 500 ms, child killed
```

Why it works: cancellation is dropping (Lesson 14.14), and `kill_on_drop(true)` connects the drop of the Rust value to the operating-system process. Without it, `pgrep -f "sleep 5"` still finds the `sleep` process after the program has ended: the child keeps running on its own, adopted by the system's first process (Lesson 13.5).

</details>

### Task 3 — One sum, three kinds of parallelism (Phase 14)

In `src/bin/parallel_sum.rs`, sum the numbers 1 to 1,000,000 (`u64`) in four ways: a plain iterator; `std::thread::scope` with 4 threads, each summing one chunk; `rayon`'s `par_iter().sum()`; and the rayon sum inside `tokio::task::spawn_blocking` from an async `main`. Print the four results and `assert!` that they are equal.

<details>
<summary>Hint</summary>

`numbers.chunks(n)` splits a slice; `div_ceil` computes the chunk size so that 4 chunks cover everything. Scoped threads may borrow `numbers` (Lesson 14.4). `spawn_blocking` needs a `'static` closure, so move the `Vec` into it.

</details>

<details>
<summary>Answer</summary>

```rust
use rayon::prelude::*;
use std::thread;

fn sum_sequential(numbers: &[u64]) -> u64 {
    numbers.iter().sum()
}

fn sum_scoped_threads(numbers: &[u64], threads: usize) -> u64 {
    let chunk = numbers.len().div_ceil(threads);
    thread::scope(|scope| {
        let handles: Vec<_> = numbers
            .chunks(chunk)
            .map(|part| scope.spawn(move || part.iter().sum::<u64>()))
            .collect();
        handles.into_iter().map(|h| h.join().unwrap()).sum()
    })
}

fn sum_rayon(numbers: &[u64]) -> u64 {
    numbers.par_iter().sum()
}

#[tokio::main]
async fn main() {
    let numbers: Vec<u64> = (1..=1_000_000).collect();
    let a = sum_sequential(&numbers);
    let b = sum_scoped_threads(&numbers, 4);
    let c = sum_rayon(&numbers);
    // spawn_blocking needs owned data ('static), so the Vec is moved in.
    let d = tokio::task::spawn_blocking(move || sum_rayon(&numbers))
        .await
        .unwrap();
    println!("sequential {a}, scoped {b}, rayon {c}, rayon in spawn_blocking {d}");
    assert!(a == b && b == c && c == d);
}
```

```text
sequential 500000500000, scoped 500000500000, rayon 500000500000, rayon in spawn_blocking 500000500000
```

Why it works: addition does not care about order, so any split gives the same sum. Scoped threads borrow the data and are joined at the end of `thread::scope`; rayon splits the work itself and steals work between its threads; `spawn_blocking` keeps the CPU-heavy work away from Tokio's worker threads (Lesson 14.13).

</details>

### Task 4 — A generic retry with a paused clock (Phase 14)

In `src/lib.rs`, write

```rust
pub async fn retry<T, E, F, Fut>(max_attempts: u32, backoff: Duration, operation: F) -> Result<T, E>
```

which calls `operation(attempt)` (attempt 1, 2, …) until it returns `Ok`, at most `max_attempts` times, and waits `backoff`, then twice as long, and so on, between attempts. It returns the last error if every attempt fails. Write two `#[tokio::test(start_paused = true)]`s: an operation that fails twice succeeds after exactly 300 ms with a 100 ms backoff; an operation that always fails returns the error of attempt 3 when `max_attempts` is 3.

<details>
<summary>Hint</summary>

The bounds are `F: FnMut(u32) -> Fut` and `Fut: Future<Output = Result<T, E>>`: `operation` is called again for each attempt and returns a new future each time. In the tests, the closure can be `|attempt| async move { … }`.

</details>

<details>
<summary>Answer</summary>

```rust
//! Stage 3 checkpoint tasks.

use std::future::Future;
use std::time::Duration;

/// Calls `operation(attempt)` until it succeeds, at most `max_attempts` times.
/// Waits `backoff`, then twice as long, … between attempts.
pub async fn retry<T, E, F, Fut>(
    max_attempts: u32,
    backoff: Duration,
    mut operation: F,
) -> Result<T, E>
where
    F: FnMut(u32) -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    let mut delay = backoff;
    let mut attempt = 1;
    loop {
        match operation(attempt).await {
            Ok(value) => return Ok(value),
            Err(error) if attempt >= max_attempts => return Err(error),
            Err(_) => {
                tokio::time::sleep(delay).await;
                delay *= 2;
                attempt += 1;
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use tokio::time::Instant;

    #[tokio::test(start_paused = true)]
    async fn retry_succeeds_on_the_third_attempt() {
        let start = Instant::now();
        let result: Result<&str, String> =
            retry(5, Duration::from_millis(100), |attempt| async move {
                if attempt < 3 {
                    Err(format!("attempt {attempt} failed"))
                } else {
                    Ok("done")
                }
            })
            .await;
        assert_eq!(result, Ok("done"));
        assert_eq!(start.elapsed(), Duration::from_millis(300)); // 100 + 200
    }

    #[tokio::test(start_paused = true)]
    async fn retry_returns_the_last_error() {
        let result: Result<(), String> = retry(3, Duration::from_secs(1), |attempt| async move {
            Err(format!("failure {attempt}"))
        })
        .await;
        assert_eq!(result, Err("failure 3".to_string()));
    }
}
```

`cargo test --lib` shows `test result: ok. 2 passed; 0 failed`. Why it works: each call of `operation` creates a fresh future, so a failed attempt leaves nothing behind. The paused clock makes the 100 ms + 200 ms of waiting take no real time and makes `assert_eq!` on the duration exact (Lesson 14.14). This is the retry part of your job queue, extracted into a reusable function.

</details>

## If you got something wrong

| Topic | Review this lesson |
|---|---|
| Two's complement and overflow | [Lesson 12.2 — Integers in memory: two's complement and overflow](../12-computer-systems/02-integers-in-memory.md) |
| Floating-point numbers | [Lesson 12.3 — Floating-point numbers](../12-computer-systems/03-floating-point-numbers.md) |
| Caches and false sharing | [Lesson 12.9 — The memory hierarchy and caches](../12-computer-systems/09-memory-hierarchy-and-caches.md), [Lesson 12.10 — Writing cache-friendly code](../12-computer-systems/10-cache-friendly-code.md) |
| Alignment, padding, niches | [Lesson 12.11 — Memory layout of Rust types](../12-computer-systems/11-memory-layout-of-rust-types.md) |
| Allocators | [Lesson 12.13 — How memory allocators work](../12-computer-systems/13-how-memory-allocators-work.md) |
| System calls | [Lesson 13.3 — System calls](../13-operating-systems/03-system-calls.md) |
| File descriptors | [Lesson 13.4 — Files and file descriptors](../13-operating-systems/04-files-and-file-descriptors.md) |
| `fork`, `exec`, `wait`, zombies | [Lesson 13.5 — Creating processes: fork, exec and wait](../13-operating-systems/05-fork-exec-and-wait.md) |
| Signals | [Lesson 13.7 — Signals in your programs](../13-operating-systems/07-signals-in-your-programs.md) |
| Virtual memory and page faults | [Lesson 13.10 — Virtual memory](../13-operating-systems/10-virtual-memory.md) |
| epoll and non-blocking I/O (input/output) | [Lesson 13.13 — Non-blocking I/O and epoll](../13-operating-systems/13-non-blocking-io-and-epoll.md) |
| Namespaces and cgroups | [Lesson 13.15 — Namespaces](../13-operating-systems/15-namespaces.md), [Lesson 13.16 — Control groups (cgroups)](../13-operating-systems/16-cgroups.md) |
| Data races, race conditions, deadlocks | [Lesson 14.1 — Why concurrency is hard: races and deadlocks](01-why-concurrency-is-hard.md), [Lesson 14.2 — Mutexes and read-write locks in depth](02-mutexes-and-rwlocks-in-depth.md) |
| Scoped threads and rayon | [Lesson 14.4 — Thread pools, scoped threads and rayon](04-thread-pools-scoped-threads-and-rayon.md) |
| Atomics, `Release`/`Acquire` | [Lesson 14.5 — Atomics](05-atomics.md), [Lesson 14.6 — Memory ordering](06-memory-ordering.md) |
| loom | [Lesson 14.9 — Testing concurrent code with loom](09-testing-concurrent-code-with-loom.md) |
| `Future`, `poll`, `Waker` | [Lesson 14.10 — How futures work: Future, poll and Waker](10-how-futures-work.md), [Lesson 14.11 — Build a tiny async runtime](11-build-a-tiny-async-runtime.md) |
| `Pin` and `Unpin` | [Lesson 14.12 — Pin and self-referential futures](12-pin-and-self-referential-futures.md) |
| Tokio tasks, `spawn_blocking`, channels | [Lesson 14.13 — Tokio in depth: tasks and channels](13-tokio-tasks-and-channels.md) |
| Cancellation, timeouts, shutdown | [Lesson 14.14 — Cancellation, timeouts and graceful shutdown](14-cancellation-timeouts-and-graceful-shutdown.md) |
| Putting it together | [Lesson 14.15 — Project: a concurrent job queue](15-project-concurrent-job-queue.md) |

If you missed more than five questions, do not rush into Stage 4. Review the lessons in the table, redo one exercise from each, and take the quiz again after a day and after a week (spaced repetition, from the Stage 1 checkpoint).

## What's next

**Stage 4 — Build real systems** starts with [Phase 15 — Networking](../15-networking/README.md). You will learn how packets travel, write TCP (Transmission Control Protocol) and UDP (User Datagram Protocol) programs, design a protocol, build HTTP (HyperText Transfer Protocol) and a DNS (Domain Name System) resolver from scratch, add TLS (Transport Layer Security), and finish with a TCP load balancer. Nearly all of it runs on Tokio, so this stage is your foundation: every connection will be a task, every server will need timeouts, backpressure and a graceful shutdown.

Before you start Phase 15:

1. **Write a short retrospective** in your learning journal: the three hardest topics of Stage 3, the project you are proudest of, and one habit to change.
2. **Tag the milestone** in your learning repository: `git tag -a stage-3-complete -m "Stage 3 complete"` and `git push --tags`.
3. **Check your projects** with the quality gate: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test` in every `code/p12-*`, `code/p13-*` and `code/p14-*` package.
4. **Keep the Linux VM (virtual machine) ready.** Phase 15 runs network tools (`ss`, `tcpdump`, `dig`) and Linux-only experiments in the VM `dev` from Phase 11, or in a small Docker image that Lesson 15.1 builds; Docker Desktop must therefore also be ready.
5. **Know the crates of Stage 4.** Do not install them now; each lesson tells you when. Versions were checked on 2026-11-17.

| Crate | Version | What it is for |
|---|---|---|
| `tokio` | 1.54 | the async runtime, TCP and UDP sockets |
| `bytes` | 1.12 | cheap byte buffers for network code |
| `hyper` | 1.12 | a low-level HTTP library |
| `reqwest` | 0.13 | an HTTP client |
| `axum` | 0.8 | an HTTP server framework (Phase 16) |
| `rustls` / `tokio-rustls` | 0.23 / 0.26 | TLS without OpenSSL |
| `hickory-resolver` | 0.26 | DNS resolution |
| `quinn` | 0.11 | QUIC (a modern transport protocol on top of UDP) and HTTP/3 |
| `tracing` | 0.1 | structured logs and spans |

## Sources and further reading

- [Rust Atomics and Locks](https://marabos.nl/atomics/) — by Mara Bos, free online; the best deep reference for Phase 14's low-level half.
- [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/) — the official async book, for the futures half.
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) — the official tutorial; its later chapters (framing, I/O) lead straight into Phase 15.
- [Computer Systems: A Programmer's Perspective — student site](https://csapp.cs.cmu.edu/) — the classic textbook behind much of Phase 12.
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — a free book that covers Phase 13 in depth.
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — the official guide to `unsafe` Rust, layout and atomics.

---

[🏠 Start](../START-HERE.md) · [← 14.15 Project: a concurrent job queue](15-project-concurrent-job-queue.md) · [📋 Phase 14](README.md) · [Phase 15 — Networking →](../15-networking/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
