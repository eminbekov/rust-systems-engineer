<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.8 Build settings for speed: profiles, LTO and PGO](08-build-settings-for-speed.md) · [📋 Phase 19](README.md) · [19.10 Project: make a slow program fast →](10-project-make-a-slow-program-fast.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/09-async-and-io-performance.md) <!-- nav -->

# Lesson 19.9 — Performance of async and I/O code

> **Phase 19 — Performance and profiling** · Lesson 9 of 11 · ⏱ about 135 minutes

## What you will learn

- Why many small reads and writes are slow, and how buffering (`BufWriter`, `BufReader`) fixes it.
- How to count system calls with `strace` in a Linux container.
- Why awaiting things one after another wastes time, and how to wait for many things together, with a limit.
- How to find out whether a task blocks the async runtime, and three ways to fix it.

## Before you start

- You finished [Lesson 19.8 — Build settings for speed](08-build-settings-for-speed.md).
- You did the load tests of [Lesson 19.2 — Load testing services](02-load-testing.md), including "waiting versus blocking" and `spawn_blocking`.
- You know system calls ([Lesson 13.3](../13-operating-systems/03-system-calls.md)), futures ([Lesson 14.10](../14-concurrency-and-async/10-how-futures-work.md)), Tokio tasks ([Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)) and axum ([Lesson 16.3](../16-backend-services/03-web-services-with-axum.md)).
- Create the package and add the dependencies:

```bash
cd ~/projects/rust
cargo new code/p19-io
cd code/p19-io
cargo add tokio@1 --features full
cargo add futures@0.3
mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **I/O** (Input/Output) | Reading and writing data outside the program's memory: files, the terminal, the network. |
| **buffer** | A block of memory where data is collected before it is written, or kept after it was read, so that one big system call replaces many small ones. |
| **buffered I/O** | I/O through a buffer. `BufWriter` collects writes; `BufReader` reads big blocks and hands out small pieces from memory. |
| **flush** | Writing out everything that is waiting in a buffer now. |
| **line buffering** | A buffer that is flushed at every newline. Rust's standard output works this way. |
| **strace** | A Linux tool that runs a program and shows (or counts) every system call it makes. |
| **fan-out** | Starting many independent operations (for example requests to other services) at the same time and then collecting their results. |
| **concurrency limit** | The highest number of operations allowed to run at the same time. |
| **stall** | A moment when the async runtime cannot run ready tasks because a worker thread is busy with one task that does not yield. |
| **cooperative scheduling** | A task runs until it reaches an `.await` that is not ready. The runtime cannot interrupt it in between; the task must give the thread back itself. |
| **yield** | To give the thread back to the runtime voluntarily, so that other tasks can run (`tokio::task::yield_now().await`). |

## 1. Every system call costs: buffered writing

Writing to a file, a pipe or the terminal is a **system call** (Lesson 13.3): the program asks the kernel to do it. A system call costs much more than a normal function call, because the CPU (Central Processing Unit) must switch into the kernel and back. Writing 10 bytes or 8,000 bytes costs almost the same, so the number of calls matters more than the number of bytes.

Here is one program that prints a million short lines in three ways. Put it into `src/bin/write_lines.rs`:

```rust
use std::io::{self, BufWriter, Write};

fn main() -> io::Result<()> {
    let mode = std::env::args().nth(1).unwrap_or_default();
    let n = 1_000_000;
    match mode.as_str() {
        // One `println!` per line.
        "println" => {
            for i in 0..n {
                println!("line {i}");
            }
        }
        // Lock stdout once, but still write each line straight through.
        "locked" => {
            let mut out = io::stdout().lock();
            for i in 0..n {
                writeln!(out, "line {i}")?;
            }
        }
        // Lock once and collect output in a 8 KiB buffer.
        "buffered" => {
            let mut out = BufWriter::new(io::stdout().lock());
            for i in 0..n {
                writeln!(out, "line {i}")?;
            }
            out.flush()?;
        }
        _ => eprintln!("usage: write_lines <println|locked|buffered>"),
    }
    Ok(())
}
```

**What happens:**

1. `println!` locks standard output, writes the line and unlocks it again, for every line. Standard output is **line-buffered**: at every `\n` it hands the line to the kernel with one `write` system call.
2. `io::stdout().lock()` takes the lock once for the whole loop. `writeln!` is `println!` for any writer. The line buffering is still there.
3. `BufWriter::new(...)` puts an 8 KiB buffer in front. `writeln!` now only copies bytes into memory; when the buffer is full, `BufWriter` writes all of it with one system call.
4. `out.flush()?` writes the last, partly filled buffer. A `BufWriter` also flushes when it is dropped, but then it cannot report an error, so flush yourself.

`hyperfine` throws away the output of the program by default (it sends it to `/dev/null`), so we measure only the program itself:

```bash
cargo build --release
hyperfine -N --warmup 2 --runs 10 -L mode println,locked,buffered './target/release/write_lines {mode}'
```

```text
Benchmark 1: ./target/release/write_lines println (10 runs)
  Wall Time    1.282 s   ±   0.204 s      0.984 s   …   1.597 s
Benchmark 2: ./target/release/write_lines locked (10 runs)
  Wall Time    1.429 s   ±   0.371 s      1.020 s   …   2.421 s            +11.4%
Benchmark 3: ./target/release/write_lines buffered (10 runs)
  Wall Time    0.026 s   ±   0.005 s      0.021 s   …   0.037 s            -98.0%  (49.1x faster)
```

(The memory lines of `hyperfine` are left out here and below.) The buffered version is **about 50 times faster**. Taking the lock only once changed nothing visible: the lock is cheap, the system calls are not.

### Counting the system calls

To *see* the system calls, use `strace`. It is a Linux tool, so run it in a container from the official Rust image (Lesson 11.12). `--cap-add SYS_PTRACE` allows `strace` to watch another process; `strace -c` counts calls instead of printing each one, and `-e trace=write` limits it to `write`:

```bash
docker run --rm --name course-19-09-strace --cap-add SYS_PTRACE \
  -v "$PWD":/work -w /work -v course-cargo-registry:/usr/local/cargo/registry \
  -e CARGO_TARGET_DIR=/tmp/target rust:1.100 bash -c '
    apt-get update -qq >/dev/null && apt-get install -y -qq strace >/dev/null 2>&1
    cargo build --release -q --bin write_lines
    for m in println buffered; do
      echo "== $m"
      strace -c -e trace=write /tmp/target/release/write_lines $m > /dev/null
    done'
```

```text
== println
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    4.655557           4   1000000           write
------ ----------- ----------- --------- --------- ----------------
100.00    4.655557           4   1000000           total
== buffered
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.005009           2      2172           write
------ ----------- ----------- --------- --------- ----------------
100.00    0.005009           2      2172           total
```

**What happens:**

1. `CARGO_TARGET_DIR=/tmp/target` keeps the Linux build inside the container, so it does not mix with your macOS `target/` folder.
2. `println`: exactly **1,000,000** `write` calls, one per line, even though the output goes to `/dev/null` and not to a terminal.
3. `buffered`: **2,172** calls for the same 11.9 MB (megabytes). (Not exactly 11.9 MB ÷ 8 KiB ≈ 1,450, because the line buffer of standard output behind the `BufWriter` sometimes splits a block into two calls; it does not matter.)
4. `strace` itself makes every system call much slower (4 µs each), so do not compare its times with `hyperfine`. Use it for **counts**.

> ⚡ **Performance:** a program that writes a lot of output (a log analyzer, a converter, a report) should write through one `BufWriter` around a locked `stdout`, created once in `main` and passed down. This is often the single cheapest big win in a command-line program.

## 2. Reading in big blocks

Reading has the same problem in the other direction. `Read::bytes()` on a `File` gives an iterator of bytes, and every byte is one `read` system call. Put this into `src/bin/count_lines.rs`:

```rust
use std::fs::File;
use std::io::{self, BufReader, Read};

fn main() -> io::Result<()> {
    let args: Vec<String> = std::env::args().collect();
    let (mode, path) = (args[1].as_str(), &args[2]);
    let file = File::open(path)?;
    let lines = match mode {
        // Every byte is one `read` system call.
        "raw" => file.bytes().filter(|b| matches!(b, Ok(b'\n'))).count(),
        // BufReader reads 8 KiB at a time and hands out bytes from memory.
        "buffered" => BufReader::new(file)
            .bytes()
            .filter(|b| matches!(b, Ok(b'\n')))
            .count(),
        _ => panic!("usage: count_lines <raw|buffered> FILE"),
    };
    println!("{lines} lines");
    Ok(())
}
```

Clippy (Lesson 05.6) knows this trap:

```bash
cargo clippy --bin count_lines
```

```text
warning: calling .bytes() is very inefficient when data is not in memory
  --> src/bin/count_lines.rs:10:18
   |
10 |         "raw" => file.bytes().filter(|b| matches!(b, Ok(b'\n'))).count(),
   |                  ^^^^^^^^^^^^
   |
   = help: consider using `BufReader`
```

Exercise 1 measures the difference. `BufReader` also gives you `read_line` and `lines()`, which you used since Phase 05. For the fastest reading of a whole file that fits in memory, `std::fs::read` (or `read_to_string`) reads it with a few big calls.

## 3. Waiting one after another, or together

The first two sections were about the CPU doing too many system calls. Async code has a different typical problem: it **waits** in the wrong way.

Imagine a handler that needs prices for 20 items from another service, and every request takes 50 ms. Put this into `src/bin/fan_out.rs`:

```rust
use std::time::{Duration, Instant};

use futures::stream::{self, StreamExt};

/// Pretends to ask another service for the price of one item: it just waits 50 ms.
async fn fetch_price(item: u32) -> u32 {
    tokio::time::sleep(Duration::from_millis(50)).await;
    item * 10
}

#[tokio::main]
async fn main() {
    let items: Vec<u32> = (1..=20).collect();

    // 1. One after another.
    let start = Instant::now();
    let mut total = 0;
    for &item in &items {
        total += fetch_price(item).await;
    }
    println!("one by one:     total {total} in {:?}", start.elapsed());

    // 2. All at the same time.
    let start = Instant::now();
    let prices = futures::future::join_all(items.iter().map(|&item| fetch_price(item))).await;
    let total: u32 = prices.iter().sum();
    println!("all at once:    total {total} in {:?}", start.elapsed());

    // 3. At most 5 at the same time.
    let start = Instant::now();
    let total: u32 = stream::iter(items.iter().copied())
        .map(fetch_price)
        .buffer_unordered(5)
        .fold(0, |acc, price| async move { acc + price })
        .await;
    println!("5 at a time:    total {total} in {:?}", start.elapsed());
}
```

`cargo run --release --bin fan_out`:

```text
one by one:     total 2100 in 1.03218175s
all at once:    total 2100 in 51.334334ms
5 at a time:    total 2100 in 206.643667ms
```

**What happens:**

1. The `for` loop awaits each future before it creates the next one. The waits add up: 20 × 50 ms = 1 second. The CPU is idle almost all of that time.
2. `join_all` takes all 20 futures and polls them together (Lesson 14.10): all 20 timers run at the same time, so the whole batch takes 50 ms. This needs no extra threads; one task waits for 20 things at once.
3. `stream::iter(...).map(fetch_price)` makes a stream of futures (not yet started). `buffer_unordered(5)` runs at most 5 of them at a time and yields the results in the order they finish. 20 items in groups of 5 take 4 × 50 ms ≈ 200 ms.
4. `fold` adds up the results as they arrive. All three versions give the same total.

The difference between 1 s and 50 ms is not a micro-optimization: it is the latency your users feel. When you look at a slow handler, first ask: "does it wait for independent things one after another?" For a fixed small number of different calls, `tokio::join!(a, b, c)` does the same as `join_all` (Exercise 3).

> 🔒 **Security:** "all at once" is dangerous when the number of items comes from the user. A request with 100,000 items would open 100,000 connections to the other service at the same moment, which can overload it (a denial of service caused by your own code). Use a **concurrency limit**: `buffer_unordered(n)`, or a `tokio::sync::Semaphore` shared by all requests.

## 4. A task that does not give the thread back

Lesson 19.2 showed that `std::thread::sleep` inside a handler makes the whole service slow. Heavy *computation* inside a task has the same effect, and it is easier to miss, because the code looks innocent.

Tokio uses **cooperative scheduling**: a task runs on a worker thread until it reaches an `.await` that is not ready. Between two such points, the runtime cannot take the thread away. A loop that computes for 300 ms without an `.await` keeps its worker thread for 300 ms, and every other task on that thread waits.

We can measure this with a **ticker**: a task that wants to wake up every 10 ms and records how late it actually woke. Put this into `src/bin/stall.rs`:

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

/// Heavy computation: pure CPU work, no waiting.
fn crunch(rounds: u64) -> u64 {
    let mut x = 1u64;
    for i in 0..rounds {
        x = black_box(x.wrapping_mul(6364136223846793005).wrapping_add(i));
    }
    x
}

/// Wakes up every 10 ms and returns the worst delay it saw, in milliseconds.
async fn ticker(ticks: u32) -> u128 {
    let mut worst = Duration::ZERO;
    for _ in 0..ticks {
        let start = Instant::now();
        tokio::time::sleep(Duration::from_millis(10)).await;
        worst = worst.max(start.elapsed().saturating_sub(Duration::from_millis(10)));
    }
    worst.as_millis()
}

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let mode = std::env::args().nth(1).unwrap_or_default();
    let rounds = 200_000_000;
    let start = Instant::now();
    let ticks = tokio::spawn(ticker(30));
    // Let the ticker run for a moment before the heavy work starts.
    tokio::time::sleep(Duration::from_millis(50)).await;
    let result: u64 = match mode.as_str() {
        "inline" => crunch(rounds),
        "blocking" => tokio::task::spawn_blocking(move || crunch(rounds))
            .await
            .unwrap(),
        "yield" => {
            let mut x = 0;
            for _ in 0..200 {
                x ^= crunch(rounds / 200);
                tokio::task::yield_now().await;
            }
            x
        }
        _ => panic!("usage: stall <inline|blocking|yield>"),
    };
    let worst = ticks.await.unwrap();
    black_box(result);
    println!(
        "{mode:<8} worst ticker delay {worst:>3} ms, total {:?}",
        start.elapsed()
    );
}
```

**What happens:**

1. `flavor = "current_thread"` runs the whole runtime on one thread (Lesson 14.13). It makes the effect easy to see; on the multi-threaded runtime it is the same per worker thread.
2. `crunch` is pure CPU work; `black_box` stops the compiler from removing it (Lesson 05.12).
3. `ticker` sleeps 10 ms thirty times and remembers the largest extra delay. On an idle runtime that is 1–2 ms.
4. `inline` calls `crunch` directly inside the async `main`: no `.await` for the whole computation.
5. `blocking` moves the computation to Tokio's separate pool of threads for blocking work with `spawn_blocking` (Lesson 19.2) and awaits the result.
6. `yield` splits the work into 200 pieces and calls `tokio::task::yield_now().await` after each piece. That `.await` gives the thread back to the runtime, which can run the ticker, and then continues the loop.

```bash
cargo build --release --bin stall
./target/release/stall inline
./target/release/stall blocking
./target/release/stall yield
```

```text
inline   worst ticker delay 446 ms, total 799.69675ms
blocking worst ticker delay   2 ms, total 501.711208ms
yield    worst ticker delay   4 ms, total 504.812083ms
```

**What happens:**

1. `inline`: the ticker was **446 ms** late. For almost half a second, no other task on this runtime could run: no timer, no network answer, nothing. In a web service, every request on that worker would have waited.
2. `blocking`: 2 ms, as on an idle runtime. The computation ran on another thread.
3. `yield`: 4 ms. The work stayed on the runtime thread, but in small pieces; each piece takes about 2 ms, so nobody waits longer than that.

Which fix to choose:

| Situation | Fix |
|---|---|
| A blocking call you cannot change (`std::fs`, a synchronous database driver, `std::thread::sleep`) | `tokio::task::spawn_blocking` |
| Short, heavy computation per request (hashing a password, compressing a response) | `spawn_blocking`, or a thread pool for CPU work such as `rayon` (Lesson 14.4) with a channel back |
| A long loop that you write yourself | Split it and `yield_now().await` every few milliseconds |
| Waiting for time or the network | Use the async version (`tokio::time::sleep`, `tokio::net`), never the blocking one |

> 🧪 **Testing:** stalls do not show up in normal tests, because each test has the runtime for itself. A ticker like the one above is a cheap stall detector: run it in a test next to the code under test and assert that the worst delay stays below, for example, 20 ms. In a running service, the tool `tokio-console` shows for every task how long its polls take, so a task that runs for a long time without yielding stands out.

## 5. A checklist for slow async and I/O code

| Symptom | Typical cause | What to do |
|---|---|---|
| A command-line program spends most of its time in the kernel | thousands of tiny writes or reads | `BufWriter` / `BufReader` (Sections 1–2) |
| A handler takes the *sum* of its downstream calls | `.await` one after another | `tokio::join!`, `join_all`, `buffer_unordered(n)` (Section 3) |
| All endpoints get slow together under load | a task blocks a worker thread | `spawn_blocking`, `yield_now`, async APIs (Section 4) |
| Latency grows with the number of items in a request | unbounded fan-out overloads a dependency | a concurrency limit (`buffer_unordered(n)`, `Semaphore`) |
| Every request pays for a new TCP and TLS connection | a new HTTP client per request | create one client (for example `reqwest::Client`) at start-up and share it; it keeps a pool of open connections |
| Many tiny network writes | writing a response piece by piece on a `TcpStream` | wrap the stream in a buffer (`tokio::io::BufWriter`) or build the message first and write it with one `write_all` |

HTTP (HyperText Transfer Protocol) is the protocol of web requests, and it runs over a TCP (Transmission Control Protocol) connection. TLS (Transport Layer Security) is the encryption layer of HTTPS, which is HTTP over TLS (Lesson 15.12). Setting up a new TCP and TLS connection costs several network round trips, which is why a shared client with a connection pool matters so much.

## Common mistakes

> ⚠️ **Common mistake:** ending the program with `std::process::exit` while data is still in a `BufWriter`. `exit` stops the process at once, `Drop` does not run, and the output is lost:
>
> ```rust
> use std::io::{self, BufWriter, Write};
>
> fn main() -> io::Result<()> {
>     let mut out = BufWriter::new(io::stdout().lock());
>     writeln!(out, "important result: 42")?;
>     std::process::exit(0); // ends the process at once: `out` is never dropped
> }
> ```
>
> This prints **nothing**. Call `out.flush()?` before `exit`, or return from `main` normally.

> ⚠️ **Common mistake:** `eprintln!` for progress messages in a hot loop. Standard error has no buffer at all: every call is a system call. Print progress every few thousand items, not every item.

> ⚠️ **Common mistake:** assuming `buffer_unordered` keeps the order. It yields results in the order they **finish**. If the order matters, use `buffered(n)` (same limit, original order) or keep the index with each result.

> ⚠️ **Common mistake:** "fixing" a slow handler by adding worker threads. If a task blocks, more threads only move the problem: with enough load, all of them block again. Remove the blocking instead.

## Try it yourself

1. Build `write_lines` and repeat the `hyperfine` comparison. Then run `./target/release/write_lines buffered > lines.txt` and check the file with `wc -l lines.txt`.
2. Change the buffer size: `BufWriter::with_capacity(64 * 1024, ...)`. Is it faster than 8 KiB?
3. In `fan_out.rs`, replace `buffer_unordered(5)` with `buffer_unordered(10)` and with `buffered(5)`. Predict the times before you run it.
4. In `stall.rs`, change `#[tokio::main(flavor = "current_thread")]` to `#[tokio::main(flavor = "multi_thread", worker_threads = 2)]`. What happens to the `inline` delay, and why?

## Exercises

### Exercise 1 — Measure the unbuffered reader (easy)

Create `lines.txt` with `./target/release/write_lines buffered > lines.txt` (one million lines, 11.9 MB). Build `count_lines` and compare `raw` and `buffered` with `hyperfine` (use `--runs 3`: the slow one takes seconds). How many times faster is `buffered`, and how many `read` calls does `raw` make?

<details>
<summary>Hint</summary>

`hyperfine -N --warmup 1 --runs 3 -L mode raw,buffered './target/release/count_lines {mode} lines.txt'`. For the number of calls, think: one call per byte.

</details>

<details>
<summary>Answer</summary>

```text
Benchmark 1: ./target/release/count_lines raw lines.txt (3 runs)
  Wall Time    3.668 s   ±   0.095 s      3.597 s   …   3.775 s
Benchmark 2: ./target/release/count_lines buffered lines.txt (3 runs)
  Wall Time    0.029 s   ±   0.000 s      0.028 s   …   0.029 s            -99.2%  (127.0x faster)
```

Both print `1000000 lines`. Why it works: `raw` makes one `read` system call per byte, so about 11.9 million calls (plus one at the end that returns 0, meaning "end of file"). `BufReader` asks for 8 KiB per call, about 1,450 calls, and serves the bytes from memory. 127 times faster, with a one-word change.

</details>

### Exercise 2 — Write to any writer, and test it (medium)

Write a library function `pub fn write_squares<W: Write>(out: &mut W, n: u64) -> io::Result<()>` in `src/lib.rs` that writes the squares of 1 to `n`, one per line. Write a unit test that writes into a `Vec<u8>` and checks the text for `n = 4`. Then call it from `src/main.rs` with a `BufWriter` around the locked standard output.

<details>
<summary>Hint</summary>

`Vec<u8>` implements `Write`. `String::from_utf8(out).unwrap()` turns the bytes into text for `assert_eq!`. The package name `p19-io` becomes the crate name `p19_io`.

</details>

<details>
<summary>Answer</summary>

`src/lib.rs`:

```rust
use std::io::{self, Write};

/// Writes the squares of 1..=n, one per line, to any writer.
pub fn write_squares<W: Write>(out: &mut W, n: u64) -> io::Result<()> {
    for i in 1..=n {
        writeln!(out, "{}", i * i)?;
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn writes_one_square_per_line() {
        let mut out = Vec::new();
        write_squares(&mut out, 4).unwrap();
        assert_eq!(String::from_utf8(out).unwrap(), "1\n4\n9\n16\n");
    }
}
```

`src/main.rs`:

```rust
use std::io::{self, BufWriter, Write};

use p19_io::write_squares;

fn main() -> io::Result<()> {
    let mut out = BufWriter::new(io::stdout().lock());
    write_squares(&mut out, 5)?;
    out.flush()
}
```

`cargo test --lib` and `cargo run --release --bin p19-io`:

```text
test tests::writes_one_square_per_line ... ok
```

```text
1
4
9
16
25
```

Why it works: the function does not decide *where* the output goes or *how* it is buffered; the caller does. The test uses a `Vec<u8>` (fast, no files), and `main` uses a buffered `stdout`. This design makes output code both testable and fast.

</details>

### Exercise 3 — Load-test a handler that waits in sequence (hard)

Add axum (`cargo add axum@0.8`) and write `src/bin/report.rs`: a service on `127.0.0.1:3021` with a route `/report` whose handler calls three fake services one after another (20 ms, 30 ms and 25 ms, using `tokio::time::sleep`) and returns `user=ok orders=ok stock=ok`. Measure it with `oha -z 5s -c 20 --no-tui` (Lesson 19.2). Then add `/report2` that makes the same three calls with `tokio::join!` and measure again.

<details>
<summary>Hint</summary>

Write one helper `async fn lookup(name: &'static str, ms: u64) -> String`. `tokio::join!(a, b, c)` returns a tuple `(a_result, b_result, c_result)`. Start the service in one terminal and run `oha` in another.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::Duration;

use axum::{Router, routing::get};

/// Pretends to call another service: waits `ms` milliseconds, then answers.
async fn lookup(name: &'static str, ms: u64) -> String {
    tokio::time::sleep(Duration::from_millis(ms)).await;
    format!("{name}=ok")
}

/// Calls the three services one after another.
async fn report_sequential() -> String {
    let user = lookup("user", 20).await;
    let orders = lookup("orders", 30).await;
    let stock = lookup("stock", 25).await;
    format!("{user} {orders} {stock}\n")
}

/// Calls the three services at the same time.
async fn report_concurrent() -> String {
    let (user, orders, stock) = tokio::join!(
        lookup("user", 20),
        lookup("orders", 30),
        lookup("stock", 25)
    );
    format!("{user} {orders} {stock}\n")
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/report", get(report_sequential))
        .route("/report2", get(report_concurrent));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3021").await.unwrap();
    println!("listening on http://127.0.0.1:3021");
    axum::serve(listener, app).await.unwrap();
}
```

```bash
cargo run --release --bin report
# in a second terminal:
oha -z 5s -c 20 --no-tui http://127.0.0.1:3021/report | grep -E 'Requests/sec|50.00%|99.00%'
oha -z 5s -c 20 --no-tui http://127.0.0.1:3021/report2 | grep -E 'Requests/sec|50.00%|99.00%'
```

```text
  Requests/sec:	251.7851
  50.00% in 80.4035 ms
  99.00% in 87.2549 ms
  Requests/sec:	631.1875
  50.00% in 31.5434 ms
  99.00% in 42.8443 ms
```

Why it works: in sequence, each request waits 20 + 30 + 25 = 75 ms (plus timer overhead: p50 80 ms). With `join!`, it waits only for the slowest call, 30 ms (p50 32 ms). With 20 connections, throughput follows Little's law (Lesson 19.1): 20 ÷ 0.080 s ≈ 250 req/s, and 20 ÷ 0.032 s ≈ 630 req/s. The CPU did almost nothing in both cases; only the waiting changed. Stop the service with Ctrl+C.

</details>

## Quiz

1. Why is printing a million lines with `println!` slow, even when the output goes to `/dev/null`?

<details>
<summary>Answer</summary>

Standard output is line-buffered: each line is one `write` system call, so a million lines are a million system calls. `strace -c` showed exactly 1,000,000.

</details>

2. Why should you call `flush()` on a `BufWriter` yourself?

<details>
<summary>Answer</summary>

To write out the last, partly filled buffer and to get an error if writing fails. Flushing on drop cannot report errors, and with `std::process::exit` the drop never happens at all.

</details>

3. What does `strace -c` show, and why do you not trust its times?

<details>
<summary>Answer</summary>

A table of how many times each system call was made, with errors and time. `strace` slows every system call down a lot, so its times are not realistic; the counts are what you use.

</details>

4. Twenty independent requests of 50 ms each: how long do they take awaited one after another, with `join_all`, and with `buffer_unordered(5)`?

<details>
<summary>Answer</summary>

About 1 s (20 × 50 ms), about 50 ms (all at once), and about 200 ms (4 groups of 5).

</details>

5. What is cooperative scheduling, and what does it mean for a long computation inside a task?

<details>
<summary>Answer</summary>

A task keeps its worker thread until it reaches an `.await` that is not ready; the runtime cannot interrupt it. A long computation without `.await` holds the thread the whole time, so all other tasks on that thread wait (a stall).

</details>

6. Name three ways to stop heavy work from stalling the runtime.

<details>
<summary>Answer</summary>

`tokio::task::spawn_blocking`; a separate CPU thread pool such as `rayon` with a channel back; splitting the loop and calling `tokio::task::yield_now().await` regularly. (And for waiting, using the async API (Application Programming Interface: the functions a library offers) instead of a blocking one.)

</details>

## Summary

- System calls are expensive; the number of calls matters more than the number of bytes. Standard output makes one `write` per line.
- Wrap output in `BufWriter` (around `stdout().lock()`) and input in `BufReader`; flush yourself. One million lines: 1,000,000 → 2,172 system calls, 50 times faster.
- `strace -c` in a Linux container counts system calls; use its counts, not its times.
- Awaiting independent operations one after another adds up their latencies. Wait for them together (`join!`, `join_all`), and limit the number at once (`buffer_unordered(n)`, `Semaphore`).
- Tokio schedules cooperatively: a task without `.await` holds its thread. Heavy work goes to `spawn_blocking` or a CPU pool, or yields regularly.
- A ticker task is a simple stall detector; `oha` shows the effect on latency under load.

## Sources and further reading

- [`std::io::BufWriter`](https://doc.rust-lang.org/std/io/struct.BufWriter.html) — the buffered writer, its capacity and flushing rules.
- [`std::io::Stdout`](https://doc.rust-lang.org/std/io/struct.Stdout.html) — standard output, its lock and its line buffering.
- [Tokio — CPU-bound tasks and blocking code](https://docs.rs/tokio/latest/tokio/index.html) — the section of the Tokio documentation about blocking work (search the page for "CPU-bound").
- [`tokio::task::spawn_blocking`](https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html) and [`tokio::task::yield_now`](https://docs.rs/tokio/latest/tokio/task/fn.yield_now.html) — the two fixes from Section 4.
- [`futures::stream::StreamExt`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html) — `buffer_unordered`, `buffered` and the other stream adapters.
- [strace(1) manual page](https://man7.org/linux/man-pages/man1/strace.1.html) — all options of `strace`, including `-c` and `-e trace=`.
- [The Rust Performance Book — I/O](https://nnethercote.github.io/perf-book/io.html) — short advice on locking and buffering.

---

[🏠 Start](../START-HERE.md) · [← 19.8 Build settings for speed: profiles, LTO and PGO](08-build-settings-for-speed.md) · [📋 Phase 19](README.md) · [19.10 Project: make a slow program fast →](10-project-make-a-slow-program-fast.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
