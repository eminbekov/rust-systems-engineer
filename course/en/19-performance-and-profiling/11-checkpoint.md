<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.10 Project: make a slow program fast](10-project-make-a-slow-program-fast.md) · [📋 Phase 19](README.md) · [Phase 20 — Reliability and hardening →](../20-reliability-and-hardening/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/11-checkpoint.md) <!-- nav -->

# Lesson 19.11 — Checkpoint

> **Phase 19 — Performance and profiling** · Lesson 11 of 11 · ⏱ about 105 minutes

## What you will learn

- Whether you are ready for Phase 20: this lesson checks the whole phase, from percentiles to a 10× faster program.
- Which lessons to review if an answer surprises you.

Answer each question in your own words **before** you open the answer. Do the practical tasks without looking back at the lessons first; look only when you are stuck.

## Before you start

- You finished [Lesson 19.10 — Project: make a slow program fast](10-project-make-a-slow-program-fast.md), at least up to milestone M4.
- Create one package for the practical tasks: `cargo new --lib code/p19-checkpoint`. Task 2 goes into the package `code/p19-layout` from Lesson 19.6, as one more program in its `src/bin/` folder.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A lesson that checks what you learned in a phase, with questions and practical tasks instead of new material. |

## Self-test quiz

1. A service handles 2,000 requests per second, and each request takes 50 ms on average. How many requests are in progress at the same time, and which law tells you?

<details>
<summary>Answer</summary>

Little's law: requests in progress = throughput × latency = 2,000 /s × 0.05 s = **100**. It tells you, for example, how many connections or worker tasks the service needs for this load.

</details>

2. Why is the mean latency a bad summary, and what should you report instead?

<details>
<summary>Answer</summary>

A few very slow requests are hidden in the mean: 99 requests of 10 ms and one of 2 s give a mean of 30 ms, which no request actually had. Report percentiles (p50, p99, p99.9) and the maximum, together with the unit and the load at which you measured.

</details>

3. What is coordinated omission, and how do you avoid it?

<details>
<summary>Answer</summary>

A load tester that waits for each answer before it sends the next request sends *fewer* requests exactly when the server is slow, so the slow periods are measured with few samples and the system looks faster than it is. Avoid it by measuring from the planned start time of each request and using an open-model load generator (a fixed rate, for example `oha -q 300 --latency-correction`).

</details>

4. In a load test, what is the "knee", and why is the capacity of a service lower than its maximum throughput?

<details>
<summary>Answer</summary>

The knee is the load at which latency stops being flat and starts to grow fast, because a resource is saturated and requests queue. Capacity is the highest load that still meets the latency goal (for example p99 below 50 ms); near maximum throughput the queues are long, so latency is already far above the goal.

</details>

5. What do *self time* and *total time* of a function mean in a profile?

<details>
<summary>Answer</summary>

Self time: the samples in which this function itself was running (it was at the top of the stack). Total time: the samples in which it was anywhere on the stack, so it includes the time of every function it called.

</details>

6. Why do you profile a build made with a `[profile.profiling]` that inherits `release` and sets `debug = true`, and not a normal debug build?

<details>
<summary>Answer</summary>

A debug build is not optimized, so its time goes to places where a release build spends almost nothing; you would fix the wrong things. The extra debug info lets the profiler show function names and source lines, including functions that were inlined. The machine code stays the same as in `release`.

</details>

7. In a flame graph, what does the width of a box mean, and what does the x-axis *not* mean?

<details>
<summary>Answer</summary>

The width is the share of samples in which the function was on the stack (its total time). The x-axis is **not** time: boxes are sorted by name or size, not by when they ran. Wide boxes with nothing above them are the hot spots (self time).

</details>

8. Where do you run `perf`, and what is the difference between `perf record -g` and `perf record --call-graph dwarf`?

<details>
<summary>Answer</summary>

On Linux: in the VM (virtual machine) `dev` with `sudo`, or in a `--privileged` container, never as root on your own machine. `-g` follows frame pointers: cheap and small files, but chains break where code has no frame pointers. `--call-graph dwarf` copies part of the stack in every sample and unwinds it with debug info: it also shows inlined functions, but the files become large.

</details>

9. What two different questions does a heap profiler like `dhat` answer, and name two ways to allocate less.

<details>
<summary>Answer</summary>

How **many** allocations the program makes (the allocation rate, which costs CPU (Central Processing Unit) time), and how **much** memory it holds at its peak (t-gmax, which decides how much RAM, Random Access Memory, it needs). To allocate less: reuse one buffer instead of creating a new `String` per item, look up map keys with `&str` instead of building an owned key, borrow instead of `clone`, and stream input instead of reading it all at once (for the peak).

</details>

10. A loop sums only the `cents` field of 10 million 32-byte `Order` structs. Why can it become several times faster if `cents` is stored in its own `Vec<u32>`?

<details>
<summary>Answer</summary>

Memory moves in cache lines of 64 or 128 bytes. In the array of structs, every 4 useful bytes bring 28 bytes of other fields with them, so the loop reads 8 times more memory than it needs. In a struct of arrays, all loaded bytes are useful, and the compiler can also vectorize the sum: the hot data is small (Lesson 19.6). Practical Task 2 measures this.

</details>

11. Why does the compiler vectorize a sum of `u32` values but not a simple sum of `f32` values, and how can you get a vectorized float sum?

<details>
<summary>Answer</summary>

A SIMD (Single Instruction, Multiple Data) sum adds the numbers in a different order. For integers (with wrapping) the result is the same, so the compiler may do it. For floats the rounding depends on the order, and the compiler must not change your result. Write the order yourself: several running totals ("lanes"), for example an array of 8 sums filled through `as_chunks::<8>()`, if a slightly different result is acceptable.

</details>

12. Why is calling a function marked `#[target_feature(enable = "avx2")]` `unsafe` from normal code, and what is the usual safe pattern around it?

<details>
<summary>Answer</summary>

On a CPU without that feature, the function would execute unknown instructions: undefined behaviour. The caller must prove the feature is there. The usual pattern: a safe wrapper that checks at run time with `is_x86_feature_detected!("avx2")` (or `is_aarch64_feature_detected!` on arm64) and calls the fast function in an `unsafe` block only then, with a simple fallback otherwise.

</details>

13. Rank these build changes by the typical size of their effect on speed: `dev` → `release`, `lto = "fat"`, PGO (profile-guided optimization), `-C target-cpu=native`. Which one has a portability risk?

<details>
<summary>Answer</summary>

`dev` → `release` is by far the biggest (5 times for the calculator in Lesson 19.8). LTO (link-time optimization), PGO and `target-cpu=native` usually give a few percent, often less than the measurement noise, sometimes more for large programs. `target-cpu=native` can produce a binary that crashes with "illegal instruction" on a CPU without the same features.

</details>

14. What are the steps of PGO, and what is the most important thing about the training run?

<details>
<summary>Answer</summary>

Build an instrumented binary (`cargo pgo build`), run it on typical work so it writes counts, then build again with the merged profile (`cargo pgo optimize`). The training run must look like real use; a profile from unusual inputs optimizes the wrong paths and can make the program slower.

</details>

15. A program prints one million short lines with `println!`. How many `write` system calls does it make, and how do you reduce them?

<details>
<summary>Answer</summary>

One per line, a million (standard output is line-buffered; `strace -c` showed exactly 1,000,000). Wrap a locked standard output in a `BufWriter`, write through it, and `flush()` at the end: about 2,000 calls and about 50 times faster in Lesson 19.9.

</details>

16. An async handler awaits three independent calls of 20, 30 and 25 ms one after another. What is its latency, how do you shorten it, and what must you add when the number of calls depends on user input?

<details>
<summary>Answer</summary>

About 75 ms, the sum. With `tokio::join!` (or `join_all`) the calls run together and the latency is about 30 ms, the slowest one. When the number of calls comes from the user, add a concurrency limit (`buffer_unordered(n)`, `buffered(n)` or a `Semaphore`), otherwise one request can overload the other service.

</details>

17. A task computes for 400 ms without any `.await`. What happens to the other tasks on the same runtime thread, and name two fixes.

<details>
<summary>Answer</summary>

Tokio schedules cooperatively, so the task keeps the thread for the whole 400 ms; every other task on that thread (timers, network answers, other requests) waits: a stall. Fixes: move the work to `tokio::task::spawn_blocking` or a CPU thread pool such as `rayon`, or split it and call `tokio::task::yield_now().await` every few milliseconds.

</details>

18. In the project of Lesson 19.10, why did you write golden tests and save `logstat-orig` before changing any code? And what does Amdahl's law say about a fix for a part that takes 80% of the time?

<details>
<summary>Answer</summary>

The golden tests freeze the exact output, so every faster version is checked automatically for identical behaviour; `logstat-orig` lets `hyperfine` compare old and new under the same conditions. Amdahl's law: making that part infinitely fast gives at most 1 / (1 − 0.8) = 5×; to go further, you must speed up the remaining 20%.

</details>

## Practical tasks

### Task 1 — Percentiles by hand (19.1)

Ten requests took these times in milliseconds: `12, 15, 11, 250, 14, 13, 16, 12, 900, 15`. Compute the mean and, with the nearest-rank method, p50, p90 and p99. Which number would you put on a dashboard, and why?

<details>
<summary>Hint</summary>

Sort first. Nearest rank: rank = ⌈p / 100 × n⌉, then take the value at that rank (counting from 1).

</details>

<details>
<summary>Answer</summary>

Sorted: `11, 12, 12, 13, 14, 15, 15, 16, 250, 900`. Sum 1,258, so the **mean is 125.8 ms**, which no request had. p50: rank ⌈0.5 × 10⌉ = 5 → **14 ms**. p90: rank 9 → **250 ms**. p99: rank ⌈9.9⌉ = 10 → **900 ms**. Show p50 and p99 (and the maximum) together: p50 says a typical request is fast, p99 says one in a hundred users waits almost a second. With only ten values, p99 is simply the maximum; real dashboards need many more samples, which is why Lesson 19.1 used histograms.

</details>

### Task 2 — Array of structs or struct of arrays (19.6)

In `code/p19-layout`, write a program `src/bin/orders.rs` with an `Order` struct of five fields (`id: u64`, `customer: u64`, `created_at: u64`, `cents: u32`, `status: u32`; 32 bytes), a `Vec` of 10 million orders, and a function `revenue_aos` that sums `cents` as `u64`. Then add a struct-of-arrays version that keeps `cents` in its own `Vec<u32>`, sum it with `revenue_soa`, and time both three times in one `main` with `Instant` and `black_box`. Both sums must be equal.

<details>
<summary>Hint</summary>

Build the struct of arrays from the array of structs with `aos.iter().map(|o| o.cents).collect()`. The first round may be slower; compare the later rounds. Mark the unused fields with `#[allow(dead_code)]` so the compiler does not warn.

</details>

<details>
<summary>Answer</summary>

```rust
use std::hint::black_box;
use std::time::Instant;

/// One order as the original program stores it (32 bytes).
#[allow(dead_code)] // the other fields are used elsewhere in a real program
#[derive(Clone, Copy)]
struct Order {
    id: u64,
    customer: u64,
    created_at: u64,
    cents: u32,
    status: u32,
}

/// The hot field in its own vector ("struct of arrays").
struct Orders {
    cents: Vec<u32>,
}

fn revenue_aos(orders: &[Order]) -> u64 {
    orders.iter().map(|o| u64::from(o.cents)).sum()
}

fn revenue_soa(orders: &Orders) -> u64 {
    orders.cents.iter().map(|&c| u64::from(c)).sum()
}

fn main() {
    let n = 10_000_000u64;
    let aos: Vec<Order> = (0..n)
        .map(|i| Order {
            id: i,
            customer: i % 1000,
            created_at: i,
            cents: (i % 5000) as u32,
            status: 1,
        })
        .collect();
    let soa = Orders {
        cents: aos.iter().map(|o| o.cents).collect(),
    };
    println!("size_of::<Order>() = {}", std::mem::size_of::<Order>());
    for _ in 0..3 {
        let t = Instant::now();
        let a = black_box(revenue_aos(black_box(&aos)));
        let ta = t.elapsed();
        let t = Instant::now();
        let s = black_box(revenue_soa(black_box(&soa)));
        let ts = t.elapsed();
        println!("aos {a} in {ta:?}, soa {s} in {ts:?}");
    }
}
```

`cargo run --release --bin orders` on an M4:

```text
size_of::<Order>() = 32
aos 24995000000 in 6.37975ms, soa 24995000000 in 5.251958ms
aos 24995000000 in 4.467041ms, soa 24995000000 in 822.625µs
aos 24995000000 in 4.668875ms, soa 24995000000 in 683.292µs
```

Why it works: the array of structs is 320 MB (megabytes), and the loop must pull all of it through the caches to read 40 MB of `cents`. The struct of arrays reads only the 40 MB it needs, in order, and the sum is vectorized: about 6 times faster after the first round. The first round can be slower for reasons that have nothing to do with the code (Lesson 12.6 explains why measurements jump around), so compare the later rounds.

</details>

### Task 3 — An early exit that blocks SIMD (19.7)

Put this into `code/p19-checkpoint/src/lib.rs` and look at its assembly with `cargo asm`:

```rust
#[inline(never)]
pub fn all_positive(values: &[i32]) -> bool {
    for &v in values {
        if v <= 0 {
            return false;
        }
    }
    true
}
```

Is it vectorized? Write `all_positive_chunks`, which checks 16 values at a time with a reduction that the compiler can vectorize, but still stops early at the first group with a bad value. Write a test that puts a zero at every position for every length from 0 to 40, and time both versions on 16,384 positive values.

<details>
<summary>Hint</summary>

`values.as_chunks::<16>()` gives full groups and a rest. For a group, the **minimum** of the 16 values is a reduction without early exit inside the group, so it vectorizes (`smin.4s`); compare the minimum with 0 once per group. Check the rest with `iter().all(...)`.

</details>

<details>
<summary>Answer</summary>

The original loop reads one value per trip (`ldr w10, [x0], #4`) and checks it: not vectorized, because of the early `return`.

```rust
#[inline(never)]
pub fn all_positive_chunks(values: &[i32]) -> bool {
    let (chunks, rest) = values.as_chunks::<16>();
    for chunk in chunks {
        if chunk.iter().fold(i32::MAX, |m, &v| m.min(v)) <= 0 {
            return false;
        }
    }
    rest.iter().all(|&v| v > 0)
}

#[cfg(test)]
mod simd_tests {
    use super::*;

    #[test]
    fn finds_a_zero_at_every_position() {
        for len in 0..=40usize {
            let good: Vec<i32> = (1..=len as i32).collect();
            assert!(all_positive_chunks(&good), "len {len}");
            for bad in 0..len {
                let mut v = good.clone();
                v[bad] = 0;
                assert!(!all_positive_chunks(&v), "len {len}, zero at {bad}");
            }
        }
    }
}
```

The main loop of `cargo asm --lib p19_checkpoint::all_positive_chunks`:

```text
LBB4_2:
	ldp q0, q1, [x9, #32]
	ldp q2, q3, [x9], #64
	smin.4s v1, v3, v1
	smin.4s v0, v2, v0
	smin.4s v0, v0, v1
	sminv.4s s0, v0
	fmov w10, s0
	cmp w10, #1
	b.lt LBB4_19
	subs x8, x8, #64
	b.ne LBB4_2
```

For the timing, `examples/positive.rs`:

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

use p19_checkpoint::{all_positive, all_positive_chunks};

/// The best of five rounds of 1,000 calls, as time per call.
fn best(f: impl Fn() -> bool) -> Duration {
    (0..5)
        .map(|_| {
            let start = Instant::now();
            for _ in 0..1000 {
                black_box(f());
            }
            start.elapsed() / 1000
        })
        .min()
        .unwrap()
}

fn main() {
    let values: Vec<i32> = (1..=16_384).collect();
    println!("simple: {:?}", best(|| all_positive(black_box(&values))));
    println!(
        "chunks: {:?}",
        best(|| all_positive_chunks(black_box(&values)))
    );
}
```

`cargo run --release --example positive`:

```text
simple: 6.091µs
chunks: 367ns
```

Why it works: inside a group there is no early exit, so the compiler computes the minimum of 16 values with four `smin.4s` and one horizontal `sminv.4s`, and only then makes one decision per group. About 15 times faster, and still stopping early, at most 15 values after the first bad one.

</details>

### Task 4 — Fan-out with a limit, tested with a paused clock (19.9)

Add Tokio and futures to `code/p19-checkpoint`, plus Tokio's test utilities for tests only:

```bash
cargo add tokio@1 --features full
cargo add futures@0.3
cargo add --dev tokio@1 --features test-util
```

Write `pub async fn fetch_all(ids: &[u32]) -> Vec<u32>` that calls a private `async fn fetch(id: u32) -> u32` (which sleeps 20 ms with `tokio::time::sleep` and returns `id * 2`) for every id, **at most 4 at a time**, and returns the results **in the order of `ids`**. Write a test with `#[tokio::test(start_paused = true)]` that checks the results for the ids 1 to 30 and that the whole call takes exactly 160 ms of (virtual) time.

<details>
<summary>Hint</summary>

`stream::iter(ids.iter().copied()).map(fetch)` gives a stream of futures. `buffer_unordered(4)` would mix the order; its sister keeps it. With `start_paused = true`, Tokio's clock does not run on its own: when every task is waiting for a timer, it jumps straight to the next timer. So `tokio::time::Instant::now()` and `elapsed()` measure exact virtual time, and the test runs in microseconds.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::Duration;

use futures::stream::{self, StreamExt};

/// Pretends to load one record from another service: waits 20 ms.
async fn fetch(id: u32) -> u32 {
    tokio::time::sleep(Duration::from_millis(20)).await;
    id * 2
}

/// Loads all records, at most 4 at the same time, in the order of `ids`.
pub async fn fetch_all(ids: &[u32]) -> Vec<u32> {
    stream::iter(ids.iter().copied())
        .map(fetch)
        .buffered(4)
        .collect()
        .await
}

#[cfg(test)]
mod fetch_tests {
    use super::*;
    use tokio::time::Instant;

    #[tokio::test(start_paused = true)]
    async fn keeps_order_and_runs_four_at_a_time() {
        let ids: Vec<u32> = (1..=30).collect();
        let start = Instant::now();
        let records = fetch_all(&ids).await;
        let elapsed = start.elapsed();
        assert_eq!(records, ids.iter().map(|id| id * 2).collect::<Vec<_>>());
        // 30 records, 4 at a time: 8 rounds of 20 ms.
        assert_eq!(elapsed, Duration::from_millis(160), "took {elapsed:?}");
    }
}
```

```text
test fetch_tests::keeps_order_and_runs_four_at_a_time ... ok
```

Why it works: `buffered(4)` keeps 4 futures running and yields their results in the original order. 30 records in groups of 4 need ⌈30 / 4⌉ = 8 rounds of 20 ms. The same test catches the slow version: with a plain `for` loop and `.await`, it fails with

```text
assertion `left == right` failed: took 600ms
  left: 600ms
 right: 160ms
```

and with no limit at all (`join_all`) it would report 20 ms, so the test also proves that the limit works.

</details>

### Task 5 — Read the numbers and decide (19.3, 19.8)

A colleague sends you this, measured on a quiet machine, and wants to ship `fat` LTO and PGO because "PGO is 3% faster":

```text
| Command | Mean Wall Time [ms] | Change | Factor |
|:---|---:|---:|:---|
| `release` | 812.4 ± 41.7 |  |  |
| `fat` | 801.9 ± 38.2 | -1.3% |  |
| `pgo` | 788.0 ± 44.9 | -3.0% |  |
```

The profile of the same program shows 70% of the time in one function that looks up every input line in a `Vec` with `iter().position(...)`. What do you answer, and what would you do first?

<details>
<summary>Hint</summary>

Compare each difference with `σ`. Then think about Amdahl's law and Lesson 19.10.

</details>

<details>
<summary>Answer</summary>

The differences (10 ms and 24 ms) are smaller than the noise (σ ≈ 40 ms), so the table does not show that `fat` or PGO helps at all; it would need many more runs to say anything, and both settings have costs (slower builds, a training step in every release). The profile is much more interesting: 70% of the time is in a linear search. Replacing it with a `HashMap` (or a sorted `Vec` with binary search) can remove most of that 70%, which by Amdahl's law allows up to 1 / (1 − 0.7) ≈ 3.3× — far more than any build setting. Fix the algorithm first, protected by golden tests, and measure; only then try build settings, and keep them only if the gain is clearly larger than σ.

</details>

## If you got something wrong

| Topic | Review |
|---|---|
| Latency, throughput, Little's law, percentiles, coordinated omission | [19.1 — Latency, throughput and percentiles](01-latency-throughput-and-percentiles.md) |
| `oha`, closed and open model, the knee, capacity, blocking handlers | [19.2 — Load testing services](02-load-testing.md) |
| Sampling profilers, self and total time, `samply`, flame graphs | [19.3 — Profilers and flame graphs](03-profilers-and-flame-graphs.md) |
| `perf stat`, `perf record`, frame pointers and DWARF | [19.4 — Profiling on Linux with perf](04-profiling-on-linux-with-perf.md) |
| Allocation rate, peak memory, `dhat`, reusing buffers | [19.5 — Memory profiling and fewer allocations](05-memory-profiling-and-allocations.md) |
| Hot data, compact types, flat arrays, choosing a map and a hasher | [19.6 — Data layout for speed](06-data-layout-for-speed.md) |
| SIMD, auto-vectorization, NEON intrinsics, `#[target_feature]` | [19.7 — SIMD: one instruction, many data](07-simd.md) |
| Profiles, LTO, codegen units, `panic = "abort"`, `target-cpu`, PGO | [19.8 — Build settings for speed](08-build-settings-for-speed.md) |
| Buffered I/O, `strace`, fan-out with limits, stalls | [19.9 — Performance of async and I/O code](09-async-and-io-performance.md) |
| The whole optimization loop, golden tests, Amdahl's law | [19.10 — Project: make a slow program fast](10-project-make-a-slow-program-fast.md) |
| Caches, cache lines, memory layout of types | [12.9 — The memory hierarchy and caches](../12-computer-systems/09-memory-hierarchy-and-caches.md), [12.11 — Memory layout of Rust types](../12-computer-systems/11-memory-layout-of-rust-types.md) |

## What's next

In this phase you learned to make programs **fast** and to prove it with numbers. [Phase 20 — Reliability and hardening](../20-reliability-and-hardening/README.md) asks the other question every production system must answer: does it **keep working** when things go wrong? You will learn how a service should treat errors and panics in production (including the `unwind` or `abort` choice from Lesson 19.8), find bugs with fuzzing, Miri, sanitizers, loom and Kani, inject faults on purpose, and protect a service against overload, where the load tests and percentiles of this phase come back as limits and backpressure.

Before you continue, commit your work in `code/p19-checkpoint`, `code/p19-layout` and `code/p19-logstat`, and add a line to your learning journal: which guess about performance turned out most wrong when you measured it?

## Summary

- Measure latency as percentiles under a stated load, avoid coordinated omission, and find the capacity of a service below its knee.
- Profile optimized code with debug info; read self and total time and flame graphs; use `perf` on Linux.
- Fewer allocations and better data layout often give more than clever code; `dhat` and the profiler show where.
- SIMD comes mostly from loops the compiler can vectorize; intrinsics need `#[target_feature]`, runtime checks and tests.
- Build settings are the last few percent; `release` is the big one, and every gain must be larger than the noise.
- In I/O and async code, count system calls, wait for independent things together with a limit, and never block a runtime thread.
- The method always wins: freeze the output, measure, profile, change one thing, test, measure again.

## Sources and further reading

- [The Rust Performance Book](https://nnethercote.github.io/perf-book/) — a compact, practical guide to everything in this phase.
- [Brendan Gregg — Flame Graphs](https://www.brendangregg.com/flamegraphs.html) — the inventor of flame graphs explains how to read them.
- [Tokio — Topics: Testing](https://tokio.rs/tokio/topics/testing) — testing with a paused clock, as in Task 4.
- [The Rust Reference — Code generation attributes](https://doc.rust-lang.org/reference/attributes/codegen.html) — `#[inline]`, `#[target_feature]` and their rules.

---

[🏠 Start](../START-HERE.md) · [← 19.10 Project: make a slow program fast](10-project-make-a-slow-program-fast.md) · [📋 Phase 19](README.md) · [Phase 20 — Reliability and hardening →](../20-reliability-and-hardening/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
