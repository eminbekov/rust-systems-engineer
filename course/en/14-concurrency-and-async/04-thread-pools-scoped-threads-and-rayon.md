<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.3 Condition variables](03-condition-variables.md) · [📋 Phase 14](README.md) · [14.5 Atomics →](05-atomics.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md) <!-- nav -->

# Lesson 14.4 — Thread pools, scoped threads and rayon

> **Phase 14 — Concurrency and async in depth** · Lesson 4 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- How to split one big calculation over all CPU (Central Processing Unit) cores with scoped threads.
- Why starting a thread per small job is slow, and how a thread pool with a multi-consumer channel fixes it.
- Why equal-sized pieces of work can still finish at different times, and how *work stealing* balances them.
- How to use the `rayon` crate: parallel iterators, `fold`/`reduce`, `par_sort` and `join`.
- The rules for rayon code: no blocking, pieces not too small, and always measure.

## Before you start

- You finished [Lesson 14.3 — Condition variables](03-condition-variables.md).
- You know scoped threads from [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), iterators and their adapters from [Lesson 05.5](../05-rust-intermediate/05-iterators.md), and the thread pool of [Lesson 06.11](../06-rust-advanced/11-project-multithreaded-web-server.md).
- In `~/projects/rust/code/p14-concurrency`, add two crates:

```bash
cargo add rayon@1 crossbeam@0.8
```

Your `Cargo.toml` now has `rayon = "1"` and `crossbeam = "0.8"` under `[dependencies]`.

## New words

| Word | Meaning |
|---|---|
| **data parallelism** | Doing the *same* operation on many pieces of data at the same time, on several cores. |
| **chunk** | One piece of a slice. `slice.chunks(n)` cuts a slice into pieces of `n` elements (the last one may be shorter). |
| **`available_parallelism`** | A standard function that says how many threads this machine can really run at the same time (usually the number of cores). |
| **P-core / E-core** | *Performance* cores (fast) and *efficiency* cores (slower, use less power). Apple M chips have both kinds. |
| **load imbalance** | Some threads finish early and sit idle while others still work, so the whole job waits for the slowest thread. |
| **work stealing** | A scheduling method: each worker thread has its own queue of tasks; a worker with nothing to do *steals* tasks from a busy worker's queue. |
| **rayon** | A crate for data parallelism. It runs your iterator or closures on a global thread pool that uses work stealing. |
| **parallel iterator** | A rayon iterator (`par_iter`, `into_par_iter`) whose items are processed by several threads. |
| **`fold` / `reduce`** | Two steps of a parallel summary: each thread *folds* its items into its own partial result, then the partial results are *reduced* (combined) into one. |
| **MPMC** (multi-producer, multi-consumer) | A channel that may have many senders **and** many receivers. `crossbeam::channel` is one. |
| **task granularity** | How big one piece of parallel work is. Too small: the overhead of scheduling eats the gain. Too big: bad balance. |
| **I/O** (input/output) | Reading and writing data outside the program: files, the network, the terminal. A thread that waits for I/O does no CPU work. |
| **stack overflow** | A thread used up all of its stack memory, usually because of too deep recursion. The program is stopped. |

## 1. Split the work yourself: scoped threads

The simplest kind of parallelism: one big job, cut into one piece per core, each piece on its own thread, and the results added up at the end. Scoped threads (Lesson 05.10) are perfect for this, because the threads may borrow the data. Create `src/bin/chunks.rs`:

```rust
use std::thread;

fn main() {
    let cores = thread::available_parallelism().map(|n| n.get()).unwrap_or(1);
    println!("this machine can run {cores} threads in parallel");

    let numbers: Vec<u64> = (1..=1_000_000).collect();
    let chunk_size = numbers.len().div_ceil(cores);

    let total: u64 = thread::scope(|s| {
        let handles: Vec<_> = numbers
            .chunks(chunk_size)
            .map(|chunk| s.spawn(move || chunk.iter().map(|x| x * x).sum::<u64>()))
            .collect();
        handles.into_iter().map(|h| h.join().unwrap()).sum()
    });

    println!("sum of squares = {total}");
}
```

```text
this machine can run 12 threads in parallel
sum of squares = 333333833333500000
```

**What happens:**

1. `available_parallelism()` returns a `Result<NonZeroUsize>`; `.get()` turns it into a `usize`. If the OS (Operating System) cannot tell, we fall back to 1. The test machine has 12 cores (8 P-cores and 4 E-cores); yours may show a different number.
2. `div_ceil` divides and rounds up, so `cores` chunks cover all the numbers.
3. `chunks(chunk_size)` cuts the vector into slices that **borrow** from `numbers`. Each scoped thread gets one slice (`move` moves only the slice reference into the closure).
4. We first `collect` all handles, so that all threads start, and only then `join` them. If we joined inside the same `map`, each thread would be joined before the next one starts, and nothing would run in parallel.

This works well when every piece costs the same. Section 3 shows what happens when it does not.

## 2. Thread pools: do not start a thread per job

Starting a thread is not free: the OS must create a stack and a control block and schedule the new thread. Let us measure it. Create `src/bin/spawn_cost.rs`:

```rust
use std::thread;
use std::time::Instant;

fn main() {
    const JOBS: u32 = 10_000;

    // One new thread per tiny job.
    let start = Instant::now();
    for i in 0..JOBS {
        thread::spawn(move || i * 2).join().unwrap();
    }
    let per_thread = start.elapsed() / JOBS;
    println!("spawn + join one thread: {per_thread:?}");

    // The same tiny job without a thread.
    let start = Instant::now();
    let mut sum = 0u64;
    for i in 0..JOBS {
        sum += std::hint::black_box(i * 2) as u64;
    }
    let per_call = start.elapsed() / JOBS;
    println!("the job itself:          {per_call:?} (sum {sum})");
}
```

`cargo run --release --bin spawn_cost`:

```text
spawn + join one thread: 17.56µs
the job itself:          0ns (sum 99990000)
```

About 18 microseconds per thread, for a job that takes less than a nanosecond. A **thread pool** pays this price once: it starts N workers at the beginning and keeps feeding them jobs through a queue.

In Lesson 06.11 the workers shared one `mpsc::Receiver` through `Arc<Mutex<Receiver>>`, because a standard `mpsc` channel has only one receiver. The `crossbeam` crate has an **MPMC** channel: you can `clone` the receiver, so every worker gets its own. Create `src/bin/pool.rs`:

```rust
use crossbeam::channel::{self, Sender};
use std::thread::{self, JoinHandle};

type Job = Box<dyn FnOnce() + Send + 'static>;

struct ThreadPool {
    sender: Option<Sender<Job>>,
    workers: Vec<JoinHandle<()>>,
}

impl ThreadPool {
    fn new(threads: usize, queue: usize) -> Self {
        let (sender, receiver) = channel::bounded::<Job>(queue);
        let workers = (0..threads)
            .map(|_| {
                let receiver = receiver.clone(); // many receivers: no Mutex needed
                thread::spawn(move || {
                    for job in receiver {
                        job();
                    }
                })
            })
            .collect();
        ThreadPool {
            sender: Some(sender),
            workers,
        }
    }

    fn execute(&self, job: impl FnOnce() + Send + 'static) {
        // Blocks while the queue is full: backpressure.
        self.sender.as_ref().unwrap().send(Box::new(job)).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        drop(self.sender.take()); // close the channel: workers' loops end
        for worker in self.workers.drain(..) {
            worker.join().unwrap();
        }
    }
}

fn main() {
    let pool = ThreadPool::new(4, 16);
    for id in 0..8 {
        pool.execute(move || {
            let name = thread::current().id();
            println!("job {id} ran on {name:?}");
        });
    }
    drop(pool); // waits for all jobs
    println!("all jobs done");
}
```

```text
job 2 ran on ThreadId(5)
job 4 ran on ThreadId(5)
job 5 ran on ThreadId(5)
job 6 ran on ThreadId(5)
job 7 ran on ThreadId(5)
job 0 ran on ThreadId(4)
job 1 ran on ThreadId(2)
job 3 ran on ThreadId(3)
all jobs done
```

**What happens:**

1. A `Job` is a boxed closure that can be called once and sent to another thread: a trait object ([Lesson 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)).
2. `channel::bounded(16)` creates an MPMC channel that holds at most 16 jobs. Like the queue of Lesson 14.3, `send` blocks when it is full: backpressure.
3. Each worker owns a clone of the receiver and loops `for job in receiver`. The loop takes jobs one by one and ends when the channel is closed and empty.
4. `Drop` drops the only sender. That closes the channel; the workers finish the jobs that are left and leave their loops; then `join` waits for each of them. This is the graceful shutdown of Lesson 06.11 in a few lines.
5. Which worker gets which job is decided at run time. Here one worker happened to take five jobs. The thread IDs and the order will differ for you.

> 💡 **Tip:** `crossbeam::channel` also has `unbounded()`, timeouts (`recv_timeout`) and a `select!` macro that waits on several channels at once. When you need a channel between threads with more than one receiver, it is the usual choice.

## 3. Uneven work and work stealing

Equal-sized chunks are only fair when every element costs the same. Often it does not. Counting primes is a good example: checking whether a big number is prime takes longer than checking a small one. On top of that, an Apple M chip has fast P-cores and slower E-cores, so even equal work finishes at different times.

```text
 equal chunks, 4 threads (time →)
 thread 1  ███                       done, idle .........
 thread 2  ██████                    done, idle ......
 thread 3  █████████                 done, idle ...
 thread 4  ████████████              ← everybody waits for this one
```

This is **load imbalance**: the job is only finished when the slowest thread is finished.

**Work stealing** fixes it without you having to guess the costs. The work is cut into many *small* tasks. Each worker thread has its own queue of tasks. When a worker's queue is empty, it does not sit idle: it **steals** a task from the other end of a busy worker's queue.

```text
 worker A queue: [t1 t2 t3 t4 t5 t6]   ← A takes its next task from this end (t6, then t5, ...)
 worker B queue: [ ]                   ← B is idle, so it steals t1 from the OTHER end of A's queue
```

Taking from opposite ends means the owner and the thief rarely touch the same task, so stealing is cheap. The result: all cores stay busy until the very end. The `rayon` crate does exactly this for you.

## 4. rayon: parallel iterators

rayon turns a normal iterator into a parallel one: change `iter()` to `par_iter()` (or `into_iter()` to `into_par_iter()`), and the items are processed on rayon's global thread pool, which has one worker per core and uses work stealing. You need one import: `use rayon::prelude::*;`. Create `src/bin/rayon_tour.rs`:

```rust
use rayon::prelude::*;

fn fib(n: u32) -> u64 {
    if n < 2 {
        n as u64
    } else {
        fib(n - 1) + fib(n - 2)
    }
}

fn main() {
    // 1. par_iter: like iter, but the items are shared out between threads.
    let words = vec!["apple", "banana", "cherry", "date"];
    let lengths: Vec<usize> = words.par_iter().map(|w| w.len()).collect();
    println!("lengths = {lengths:?}");

    // 2. Ranges and sums.
    let total: u64 = (1..=100u64).into_par_iter().sum();
    println!("total = {total}");

    // 3. Sorting a slice in parallel.
    let mut numbers: Vec<i32> = (0..10).rev().collect();
    numbers.par_sort();
    println!("sorted = {numbers:?}");

    // 4. join: run two closures, maybe in parallel, and wait for both.
    let (a, b) = rayon::join(|| fib(30), || fib(31));
    println!("fib(30) = {a}, fib(31) = {b}");
}
```

```text
lengths = [5, 6, 6, 4]
total = 5050
sorted = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
fib(30) = 832040, fib(31) = 1346269
```

**What happens:**

1. `par_iter()` gives a parallel iterator over `&str` items. `map` and `collect` look like the normal ones. `collect` keeps the **original order**, even though the items were processed in any order on any thread.
2. `into_par_iter()` on a range splits the range into pieces; `sum` adds the partial sums.
3. `par_sort()` is a parallel version of `sort()` (stable, like `sort`); `par_sort_unstable()` also exists.
4. `rayon::join(a, b)` is the basic building block of rayon: it puts `b` into the current worker's queue, runs `a`, then runs `b` itself, unless another worker has stolen `b` in the meantime. Then it waits for both. Exercise 3 builds a parallel sort on top of it.

Now the honest comparison on uneven work. Create `src/bin/primes.rs`:

```rust
use rayon::prelude::*;
use std::thread;
use std::time::Instant;

/// Slow on purpose: trial division. Bigger numbers take longer.
fn is_prime(n: u64) -> bool {
    if n < 2 {
        return false;
    }
    let mut d = 2;
    while d * d <= n {
        if n.is_multiple_of(d) {
            return false;
        }
        d += 1;
    }
    true
}

const N: u64 = 5_000_000;

fn main() {
    let start = Instant::now();
    let count = (1..=N).filter(|&n| is_prime(n)).count();
    println!("sequential   : {count} primes in {:.1?}", start.elapsed());

    let threads = thread::available_parallelism().unwrap().get() as u64;
    let start = Instant::now();
    let count: usize = thread::scope(|s| {
        let handles: Vec<_> = (0..threads)
            .map(|t| {
                let (from, to) = (t * N / threads + 1, (t + 1) * N / threads);
                s.spawn(move || (from..=to).filter(|&n| is_prime(n)).count())
            })
            .collect();
        handles.into_iter().map(|h| h.join().unwrap()).sum()
    });
    println!("equal chunks : {count} primes in {:.1?}", start.elapsed());

    let start = Instant::now();
    let count = (1..=N).into_par_iter().filter(|&n| is_prime(n)).count();
    println!("rayon        : {count} primes in {:.1?}", start.elapsed());
}
```

`cargo run --release --bin primes`, three runs:

```text
sequential   : 348513 primes in 422.6ms
equal chunks : 348513 primes in 62.1ms
rayon        : 348513 primes in 51.4ms
sequential   : 348513 primes in 409.5ms
equal chunks : 348513 primes in 62.6ms
rayon        : 348513 primes in 51.3ms
sequential   : 348513 primes in 412.4ms
equal chunks : 348513 primes in 62.7ms
rayon        : 348513 primes in 54.2ms
```

**What happens:**

1. `is_prime` tries every divisor up to the square root, so numbers near 5,000,000 cost more than numbers near 1, and primes cost the most (no early exit).
2. **Equal chunks**: 12 threads, each with one range of the same length. The last ranges hold the most expensive numbers, and some threads run on E-cores. The result is about 6.6 times faster than sequential, not 12 times: some threads wait idle at the end.
3. **rayon**: one line of code, about 8 times faster than sequential and about 18% faster than our chunks, because work stealing keeps every core busy until the end.
4. All three versions find the same number of primes. A parallel version that gives a different answer is a bug, not a speed-up; Exercises 1 and 2 test exactly this.

> ⚡ **Performance:** Twelve cores never give a 12× speed-up. Some work cannot be split, threads must be started and joined, results must be combined, and E-cores are slower. Measure in release mode, compare with the sequential version, and keep the parallel version only if it is clearly faster.

## 5. Rules for rayon code

1. **The closures must be `Send + Sync`**, because many threads call them at once. The compiler checks it: use an `Rc<i32>` inside `par_iter().map(...)` and the build stops with ``error[E0277]: `Rc<i32>` cannot be shared between threads safely``, like in Lesson 05.10.
2. **Do not block inside rayon.** rayon has one thread per core. A closure that sleeps, waits for a lock for a long time or reads from the network keeps a core idle, and other tasks cannot use it. rayon is for **CPU-heavy** work. Waiting for I/O (input/output) is the job of async code (Lessons 14.10 to 14.14).
3. **Do not make tasks too small.** Summing 10 numbers in parallel is slower than sequential: the scheduling costs more than the work. rayon already splits iterators sensibly. In recursive code with `join`, switch to sequential code below a threshold, as Exercise 3 does.
4. **Control the number of threads when you need to.** The environment variable `RAYON_NUM_THREADS` sets the size of the global pool. On the test machine `RAYON_NUM_THREADS=1 cargo run --release --bin primes` printed `rayon : 348513 primes in 397.7ms` (no gain at all), and `RAYON_NUM_THREADS=4` printed `107.6ms`. For a separate pool inside a program, use `rayon::ThreadPoolBuilder::new().num_threads(4).build()`.
5. **Shared results need care.** Do not push into a `Mutex<Vec<_>>` from every item: all threads would wait for that lock (Lesson 14.2). Return values and let `collect`, `sum` or `fold`/`reduce` combine them, as in Exercise 2.

Which tool to use:

| Situation | Tool |
|---|---|
| Split a calculation over data you have, equal pieces | `thread::scope` + `chunks` |
| Same operation on many items, uneven cost | rayon `par_iter` |
| Divide-and-conquer recursion (sort, tree walk) | `rayon::join` |
| Many independent jobs that arrive over time | a thread pool with a channel |
| Many jobs that mostly *wait* (network, disk, timers) | async with Tokio (Lessons 14.13 and 14.14) |

> 🧪 **Testing:** For every parallel function, keep a simple sequential version and write a test that compares both on many inputs, including edge cases (empty input, one item, an input smaller than the number of threads). The sequential version is your *reference implementation*.

## Common mistakes

1. **Joining inside the loop that spawns.** `for c in chunks { s.spawn(...).join() }` runs the threads one after another. Spawn all, collect the handles, then join.
2. **Parallel code without a sequential check.** A wrong `reduce` or a race can give a slightly wrong result that nobody notices. Compare with the sequential version in a test.
3. **Blocking or sleeping in a rayon closure.** It blocks a whole core of rayon's pool. Use a dedicated thread or async code for waiting.
4. **Measuring in debug mode.** Debug builds are many times slower and change the picture. Always `--release` for speed measurements.
5. **Expecting a speed-up for tiny work.** Thread start-up and task scheduling cost microseconds; a job that takes nanoseconds gets slower when you parallelize it.

## Try it yourself

1. Run `cargo run --release --bin primes` with `RAYON_NUM_THREADS=1`, `2`, `4`, `8` and without the variable. Write down the times and see where the gain stops growing.
2. In `primes.rs`, rewrite the "equal chunks" part so that it joins each thread inside the `map`: replace `s.spawn(move || …)` with `s.spawn(move || …).join().unwrap()`, and replace the last line of the scope with `.sum()` directly after the `map`. The count is still right, but on the test machine the time went from about `62ms` to `426.1ms`, the same as sequential. Why?
3. In `pool.rs`, change `ThreadPool::new(4, 16)` to `ThreadPool::new(1, 16)`. Which thread IDs do you see now? (On the test machine every job ran on `ThreadId(2)`, in order 0 to 7.)

## Exercises

### Exercise 1 — Parallel sum of squares with a test (easy)

Write `src/bin/squares.rs` with `sum_of_squares_seq(n: u64) -> u64` (normal iterator) and `sum_of_squares_par(n: u64) -> u64` (rayon). `main` prints both for `n = 1_000_000`. Write two tests: the two functions agree for `n` in `[0, 1, 2, 10, 1000, 1_000_000]`, and the parallel result equals the formula `n(n + 1)(2n + 1) / 6`.

<details>
<summary>Hint</summary>

`(1..=n).into_par_iter().map(|x| x * x).sum()`. Put `use rayon::prelude::*;` at the top. In the test, add the value of `n` to the assert message, so a failure tells you which input failed.

</details>

<details>
<summary>Answer</summary>

```rust
use rayon::prelude::*;

fn sum_of_squares_seq(n: u64) -> u64 {
    (1..=n).map(|x| x * x).sum()
}

fn sum_of_squares_par(n: u64) -> u64 {
    (1..=n).into_par_iter().map(|x| x * x).sum()
}

fn main() {
    let n = 1_000_000;
    println!("parallel   {}", sum_of_squares_par(n));
    println!("sequential {}", sum_of_squares_seq(n));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parallel_equals_sequential() {
        for n in [0, 1, 2, 10, 1000, 1_000_000] {
            assert_eq!(sum_of_squares_par(n), sum_of_squares_seq(n), "n = {n}");
        }
    }

    #[test]
    fn matches_the_formula() {
        let n = 1_000_000u64;
        assert_eq!(sum_of_squares_par(n), n * (n + 1) * (2 * n + 1) / 6);
    }
}
```

```text
parallel   333333833333500000
sequential 333333833333500000
```

`cargo test --bin squares` shows both tests `ok`.

Why it works: addition of integers gives the same result in any order and any grouping, so the partial sums of the threads add up to the same total. (This is not true for floating-point numbers: there a parallel sum can differ in the last digits.)

</details>

### Exercise 2 — Parallel word count (medium)

Write `count_seq(lines: &[&str]) -> HashMap<String, usize>` and `count_par` (rayon) in `src/bin/word_count.rs`. Words are separated by whitespace and counted in lowercase. `count_par` must not use a `Mutex`: use `fold` to build one `HashMap` per thread and `reduce` to merge them. Test on 10,000 copies of one line that both versions give the same map.

<details>
<summary>Hint</summary>

`fold(HashMap::new, |mut counts, line| { …; counts })` — the first argument is a function that creates an empty map for each thread. `reduce(HashMap::new, |mut a, b| { for (word, n) in b { … } a })` merges two maps into one. `std::iter::repeat_n(text, 10_000).collect()` builds the test input.

</details>

<details>
<summary>Answer</summary>

```rust
use rayon::prelude::*;
use std::collections::HashMap;

fn count_seq(lines: &[&str]) -> HashMap<String, usize> {
    let mut counts = HashMap::new();
    for line in lines {
        for word in line.split_whitespace() {
            *counts.entry(word.to_lowercase()).or_insert(0) += 1;
        }
    }
    counts
}

fn count_par(lines: &[&str]) -> HashMap<String, usize> {
    lines
        .par_iter()
        .fold(HashMap::new, |mut counts, line| {
            for word in line.split_whitespace() {
                *counts.entry(word.to_lowercase()).or_insert(0) += 1;
            }
            counts
        })
        .reduce(HashMap::new, |mut a, b| {
            for (word, n) in b {
                *a.entry(word).or_insert(0) += n;
            }
            a
        })
}

fn main() {
    let lines = ["the cat sat", "The dog sat", "a cat and a dog"];
    let counts = count_par(&lines);
    assert_eq!(counts, count_seq(&lines));
    let mut pairs: Vec<_> = counts.into_iter().collect();
    pairs.sort();
    println!("{pairs:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parallel_count_equals_sequential_count() {
        let text = "one two three two three three Four four FOUR four";
        let lines: Vec<&str> = std::iter::repeat_n(text, 10_000).collect();
        let par = count_par(&lines);
        assert_eq!(par, count_seq(&lines));
        assert_eq!(par["four"], 40_000);
    }
}
```

```text
[("a", 2), ("and", 1), ("cat", 2), ("dog", 2), ("sat", 2), ("the", 2)]
```

Why it works: during `fold` every thread writes only to its own map, so there is nothing to lock and nothing to wait for. `reduce` then merges the maps pair by pair, also in parallel. A `HashMap` has no order, so `main` sorts the pairs before printing to get stable output.

</details>

### Exercise 3 — Parallel quicksort with `rayon::join` (hard)

Write `par_quicksort<T: Ord + Copy + Send>(v: &mut [T])` in `src/bin/par_quicksort.rs`. Split the slice around a pivot (a *partition*), then sort the two sides with `rayon::join`. Slices of 1,000 elements or fewer are sorted with `sort_unstable`. Test against the standard `sort` on pseudo-random data of several lengths (generate it yourself with a few lines of arithmetic, no extra crate), and add a test with 50,000 equal elements.

<details>
<summary>Hint</summary>

`split_at_mut(mid)` gives two non-overlapping `&mut` slices, which is what the borrow checker needs for `join(|| sort(left), || sort(right))`.

If your first version uses the last element as the pivot and puts everything "not smaller" on the right, run the equal-elements test. What happens to the size of the right side when all elements are equal? How deep does the recursion get?

</details>

<details>
<summary>Answer</summary>

A first version with a simple two-way partition passes the random test but crashes on the equal elements:

```text
thread '<unknown>' (5663817) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

With 50,000 equal elements, every partition moves just one element (the pivot) to its place, and the other 49,999 stay on one side. The recursion goes about 50,000 levels deep, and the thread runs out of stack. The fix is a **three-way partition**: smaller, equal, greater. The equal part is already in its final place and is not sorted again.

```rust
use std::cmp::Ordering;

/// Sorts `v` with quicksort; the two sides are sorted with `rayon::join`.
fn par_quicksort<T: Ord + Copy + Send>(v: &mut [T]) {
    if v.len() <= 1_000 {
        v.sort_unstable(); // small parts: not worth a parallel task
        return;
    }
    let (lt, gt) = partition3(v);
    let (left, rest) = v.split_at_mut(lt);
    let right = &mut rest[gt - lt..]; // skip the elements equal to the pivot
    rayon::join(|| par_quicksort(left), || par_quicksort(right));
}

/// Three-way partition around the middle element. Returns `(lt, gt)`:
/// `v[..lt]` < pivot, `v[lt..gt]` == pivot, `v[gt..]` > pivot.
fn partition3<T: Ord + Copy>(v: &mut [T]) -> (usize, usize) {
    let pivot = v[v.len() / 2];
    let (mut lt, mut i, mut gt) = (0, 0, v.len());
    while i < gt {
        match v[i].cmp(&pivot) {
            Ordering::Less => {
                v.swap(lt, i);
                lt += 1;
                i += 1;
            }
            Ordering::Greater => {
                gt -= 1;
                v.swap(i, gt);
            }
            Ordering::Equal => i += 1,
        }
    }
    (lt, gt)
}

fn main() {
    let mut v: Vec<u32> = (0..5000).rev().collect();
    par_quicksort(&mut v);
    println!("first: {:?}, last: {:?}", &v[..3], &v[4997..]);
    println!("sorted: {}", v.is_sorted());
}

#[cfg(test)]
mod tests {
    use super::*;

    /// A tiny pseudo-random number generator, so we need no extra crate.
    fn pseudo_random(len: usize, seed: u64) -> Vec<u64> {
        let mut x = seed;
        (0..len)
            .map(|_| {
                x = x
                    .wrapping_mul(6364136223846793005)
                    .wrapping_add(1442695040888963407);
                x >> 33
            })
            .collect()
    }

    #[test]
    fn sorts_like_the_standard_library() {
        for (len, seed) in [(0, 1), (1, 2), (999, 3), (100_000, 4), (1_000_000, 5)] {
            let mut ours = pseudo_random(len, seed);
            let mut expected = ours.clone();
            par_quicksort(&mut ours);
            expected.sort();
            assert_eq!(ours, expected, "len = {len}");
        }
    }

    #[test]
    fn handles_many_equal_elements() {
        let mut v = vec![7u8; 50_000];
        v.extend([3, 9, 1]);
        par_quicksort(&mut v);
        assert!(v.windows(2).all(|w| w[0] <= w[1]));
    }
}
```

```text
first: [0, 1, 2], last: [4997, 4998, 4999]
sorted: true
```

`cargo test --bin par_quicksort` shows both tests `ok`.

Why it works: `partition3` walks once over the slice and keeps three regions: `< pivot` at the front, `> pivot` at the back, `== pivot` in the middle. Only the outer regions are sorted further, so equal elements end the recursion at once. `split_at_mut` proves to the compiler that the two `&mut` slices do not overlap, so `rayon::join` may sort them on different threads. The threshold of 1,000 keeps the tasks big enough to be worth scheduling (task granularity). The pseudo-random generator is a *linear congruential generator*: multiply, add, keep the high bits.

</details>

## Quiz

1. Why do we collect all `ScopedJoinHandle`s before joining them?

<details>
<summary>Answer</summary>

So that all threads are started first and run at the same time. Joining a thread right after spawning it waits for it to finish before the next one starts, and the code becomes sequential.

</details>

2. Why is a thread pool faster than a new thread per job for small jobs?

<details>
<summary>Answer</summary>

Starting and joining a thread costs many microseconds (about 18 µs on the test machine). A pool starts its threads once and then only passes jobs through a channel, which is far cheaper.

</details>

3. What is work stealing, and what problem does it solve?

<details>
<summary>Answer</summary>

Each worker has its own task queue; an idle worker takes tasks from a busy worker's queue. It solves load imbalance: when tasks have different costs or cores have different speeds, nobody sits idle while others still have work.

</details>

4. Why should you not sleep or wait for the network inside `par_iter().map(...)`?

<details>
<summary>Answer</summary>

rayon has a fixed number of threads, about one per core, meant for CPU work. A blocked closure keeps one of them idle, and the other tasks cannot use that core. Waiting should happen on dedicated threads or in async code.

</details>

5. What do `fold` and `reduce` do in a rayon pipeline, and why is that better than one shared `Mutex<HashMap>`?

<details>
<summary>Answer</summary>

`fold` lets every thread build its own partial result without any locking; `reduce` combines the partial results into one. With a shared `Mutex<HashMap>` every item would take the same lock, and the threads would spend their time waiting for each other.

</details>

6. A parallel version is 6 times faster than the sequential one on 12 cores. Is something wrong?

<details>
<summary>Answer</summary>

Not necessarily. Part of the work is sequential, threads must be started and combined, and E-cores are slower than P-cores. A speed-up below the number of cores is normal. Check that the result is the same, and compare different splits (for example equal chunks against rayon) to see whether load imbalance is the cause.

</details>

## Summary

- `thread::scope` with `chunks` splits work over cores, borrowing the data. Spawn all threads first, then join.
- A thread costs about 18 µs to start. A thread pool starts workers once; with an MPMC channel (`crossbeam::channel`) every worker can own a receiver, and dropping the sender shuts the pool down gracefully.
- Equal pieces are not equal work. Work stealing lets idle workers take tasks from busy ones, so all cores stay busy until the end.
- rayon gives parallel iterators (`par_iter`, `into_par_iter`), `fold`/`reduce`, `par_sort` and `join` on a global work-stealing pool.
- rayon is for CPU-heavy work: closures must be `Send + Sync`, must not block, and tasks must not be too small. `RAYON_NUM_THREADS` sets the pool size.
- Always check a parallel result against a sequential reference and measure in release mode.

## Sources and further reading

- [rayon on docs.rs](https://docs.rs/rayon) — the crate documentation: parallel iterators, `join`, `scope` and `ThreadPoolBuilder`.
- [rayon FAQ](https://github.com/rayon-rs/rayon/blob/main/FAQ.md) — answers to common questions, for example how many threads rayon uses and why.
- [crossbeam::channel on docs.rs](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html) — bounded and unbounded MPMC channels and `select!`.
- [std::thread::scope](https://doc.rust-lang.org/std/thread/fn.scope.html) — the documentation of scoped threads.
- [std::thread::available_parallelism](https://doc.rust-lang.org/std/thread/fn.available_parallelism.html) — what the number means and when it can be wrong.

---

[🏠 Start](../START-HERE.md) · [← 14.3 Condition variables](03-condition-variables.md) · [📋 Phase 14](README.md) · [14.5 Atomics →](05-atomics.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
