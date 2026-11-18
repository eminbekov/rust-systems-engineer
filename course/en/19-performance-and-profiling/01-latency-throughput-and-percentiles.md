<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 19 — Performance and profiling](README.md) · [📋 Phase 19](README.md) · [19.2 Load testing services →](02-load-testing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/01-latency-throughput-and-percentiles.md) <!-- nav -->

# Lesson 19.1 — Latency, throughput and percentiles

> **Phase 19 — Performance and profiling** · Lesson 1 of 11 · ⏱ about 135 minutes

## What you will learn

- What **latency** and **throughput** are, and how they are connected (Little's law).
- Why the average time hides the slow requests, and how **percentiles** (p50, p99, p99.9) show them.
- How to compute percentiles yourself and how to record millions of values in a **histogram** with the `hdrhistogram` crate.
- What **coordinated omission** is: the measuring mistake that makes slow systems look fast.

## Before you start

- You know how to measure honestly: release mode, `black_box`, many runs ([Lesson 05.12 — Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md) and [Lesson 09.11 — Benchmarking in depth](../09-testing-debugging-benchmarking/11-benchmarking-in-depth.md)).
- You can start threads with `thread::scope` ([Lesson 14.4 — Thread pools, scoped threads and rayon](../14-concurrency-and-async/04-thread-pools-scoped-threads-and-rayon.md)).
- You have built a web service ([Phase 16 — Backend services](../16-backend-services/README.md)). This lesson has no service yet; [Lesson 19.2](02-load-testing.md) puts load on one.

**How code is organised in this phase.** Every lesson has its own package `~/projects/rust/code/p19-<topic>`. Small programs go into `src/bin/<name>.rs` and run with `cargo run --release --bin <name>`. Shared helpers go into `src/lib.rs`. Always run performance code with `--release`.

## New words

| Word | Meaning |
|---|---|
| **latency** | How long one operation takes, from the moment it is asked for until the answer is there. Measured in time: µs, ms. |
| **throughput** | How many operations are finished per unit of time, for example requests per second (req/s). |
| **µs** (microsecond) | One millionth of a second. 1 ms (millisecond) = 1000 µs. |
| **distribution** | The whole set of measured values and how often each value appears: many fast ones, a few slow ones. |
| **percentile** | The value below which a given share of measurements falls. p99 = 3 ms means 99% of requests took at most 3 ms. |
| **p50 / median** | The 50th percentile: half of the requests are faster, half are slower. |
| **tail latency** | The slowest part of the distribution: p99, p99.9, max. "The tail" for short. |
| **outlier** | One value far away from the others. In latency, outliers are real requests that real users waited for. |
| **histogram** | A data structure that counts how many values fall into each range ("bucket"). It uses fixed memory, no matter how many values you add. |
| **Little's law** | A rule from queueing theory: requests in progress = throughput × latency. |
| **queue** | A line of work that waits to be served. Waiting time in a queue is part of latency. |
| **coordinated omission** | A measuring mistake: the tester waits for a slow answer and therefore does not send (and does not measure) the requests that would have been slow too. |
| **SLO** (Service Level Objective) | A promise a team makes about its service, for example "p99 latency below 200 ms over 30 days". |

## 1. Two numbers: latency and throughput

Think of a small coffee shop with one barista. Two different questions describe how good it is:

- **Latency:** "How long do I wait for my coffee?" — for example 3 minutes.
- **Throughput:** "How many coffees does the shop make per hour?" — for example 20.

They are not the same thing. Hire a second barista and throughput doubles, but each coffee still takes 3 minutes. Buy a faster machine and both change. Let a long line form at the door and throughput stays at 20 per hour, but your *latency* grows, because now you wait in the queue before the 3 minutes even start.

The same is true for programs. A web service has a latency per request and a throughput in requests per second. A function that parses a file has a latency per call and a throughput in megabytes per second.

There is a simple rule that connects the two. It is called **Little's law**:

```text
requests in progress  =  throughput  ×  latency
        L             =      λ       ×     W

Example: 2000 req/s × 0.005 s (5 ms) = 10 requests in progress at any moment
```

(`λ`, "lambda", and `W` are the usual letters in books; you do not need them.) Little's law is useful in both directions. If your service may have at most 10 requests in progress (10 threads, 10 database connections) and each takes 5 ms, it can never do more than 2000 req/s. And if latency suddenly doubles while traffic stays the same, the number of requests in progress doubles too, and you may run out of threads or connections.

> 💡 **Tip:** When someone says "the service is fast", ask "which number?". Latency is what one user feels. Throughput is what the machine can carry. A change can improve one and make the other worse, for example collecting work into big batches gives more throughput but each item waits longer.

## 2. Measuring latency: why the average lies

Create the package for this lesson:

```bash
cd ~/projects/rust/code
cargo new --lib p19-latency
cd p19-latency
mkdir src/bin
```

First we need a way to compute a percentile. Put this into `src/lib.rs`:

```rust
//! Small helpers for measuring latency.

/// Returns the `p`-th percentile (0.0..=100.0) of a sorted slice,
/// using the "nearest rank" method.
///
/// Panics if `sorted` is empty or `p` is outside 0..=100.
pub fn percentile<T: Copy>(sorted: &[T], p: f64) -> T {
    assert!(!sorted.is_empty(), "no values");
    assert!((0.0..=100.0).contains(&p), "p must be in 0..=100, got {p}");
    // Rank = how many values must be at or below the answer.
    let rank = (p / 100.0 * sorted.len() as f64).ceil() as usize;
    // Ranks start at 1, indexes at 0. p = 0 gives rank 0: take the first value.
    let index = rank.saturating_sub(1);
    sorted[index]
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn percentiles_of_one_to_hundred() {
        let values: Vec<u32> = (1..=100).collect();
        assert_eq!(percentile(&values, 50.0), 50);
        assert_eq!(percentile(&values, 99.0), 99);
        assert_eq!(percentile(&values, 100.0), 100);
        assert_eq!(percentile(&values, 0.0), 1);
    }

    #[test]
    fn one_value_is_every_percentile() {
        assert_eq!(percentile(&[7], 0.0), 7);
        assert_eq!(percentile(&[7], 99.9), 7);
    }

    #[test]
    #[should_panic(expected = "no values")]
    fn empty_slice_panics() {
        let empty: [u32; 0] = [];
        percentile(&empty, 50.0);
    }
}
```

**What happens:**

1. The values must be **sorted** first. Then "the 99th percentile of 1000 values" is simply the value at rank 990: 990 values are at or below it.
2. `p / 100.0 * len` gives the rank as a fraction; `ceil` rounds it up, so the answer is always a real measured value (this is the "nearest rank" method; other tools may interpolate between two values and give slightly different numbers).
3. Ranks count from 1, slice indexes from 0, so we subtract 1. `saturating_sub` keeps 0 at 0 for `p = 0`.
4. The function is generic over `T: Copy`, so it works for `Duration`, `u64` or anything else that can be copied out of the slice.
5. The tests use 1..=100, where the right answer is easy to see: the 50th percentile is 50.

Run `cargo test`:

```text
running 3 tests
test tests::one_value_is_every_percentile ... ok
test tests::percentiles_of_one_to_hundred ... ok
test tests::empty_slice_panics - should panic ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Now a pretend request handler. Most requests are cheap, but every 50th request does 100 times more work, like a request that misses a cache and must go to the database. Create `src/bin/latency.rs`:

```rust
use p19_latency::percentile;
use std::hint::black_box;
use std::time::{Duration, Instant};

/// A pretend request handler. Most requests are cheap,
/// but every 50th request does 100 times more work.
fn handle_request(id: u64) -> u64 {
    let rounds = if id.is_multiple_of(50) {
        2_000_000
    } else {
        20_000
    };
    let mut x = id;
    for _ in 0..rounds {
        x = x.wrapping_mul(6364136223846793005).wrapping_add(1);
    }
    x
}

fn main() {
    let mut times: Vec<Duration> = Vec::with_capacity(1000);
    for id in 0..1000 {
        let start = Instant::now();
        black_box(handle_request(black_box(id)));
        times.push(start.elapsed());
    }

    let total: Duration = times.iter().sum();
    let mean = total / times.len() as u32;
    times.sort();

    println!("requests: {}", times.len());
    println!("mean   {mean:?}");
    for (name, p) in [("p50", 50.0), ("p90", 90.0), ("p99", 99.0), ("p99.9", 99.9)] {
        println!("{name:<6} {:?}", percentile(&times, p));
    }
    println!("max    {:?}", times[times.len() - 1]);
}
```

**What happens:**

1. `handle_request` does a fixed amount of arithmetic. `id.is_multiple_of(50)` is true for every 50th id, which gets 100 times more rounds.
2. `black_box` stops the optimizer from removing the work (Lesson 05.12).
3. We time each call separately with `Instant` and keep every duration in a `Vec`.
4. The **mean** (average) is the sum divided by the count. Then we sort and read the percentiles.

Run it with `cargo run --release --bin latency`. One run on an M4 Pro (your numbers will differ, and they change from run to run):

```text
requests: 1000
mean   103.625µs
p50    23.042µs
p90    23.417µs
p99    2.372417ms
p99.9  23.606875ms
max    23.606875ms
```

Read the numbers slowly:

- **p50 is 23 µs.** A typical request is fast.
- **The mean is 104 µs**, more than four times the median. *No request took 104 µs.* The mean mixes 980 fast requests with 20 slow ones and describes neither group.
- **p99 is 2.4 ms**: that is the slow, "cache miss" group, 100 times slower than the median.
- **p99.9 and max are 23 ms**, even slower than our slow path. Our code never does 23 ms of work. This was the operating system: it paused our thread to run something else (Lesson 13.9). Real users feel such pauses too, so they belong in the measurement.

This is why engineers talk about percentiles and not about averages. One more reason makes the tail important: a single page or a single user action often needs **many** requests. If a page calls 100 small services and each one is slower than its p99 in 1% of calls, then most pages will hit at least one slow call (you will compute exactly how many in Exercise 1). For big systems, p99 is what most users feel.

> ⚠️ **Common mistake:** throwing away the "outliers" because they "spoil" the measurement. In a micro-benchmark of one function (Lesson 09.11) that can be right, because you want to know the speed of the code. In a latency measurement of a service, the outliers *are* the result: they are real people who waited.

> 📏 **Convention:** write percentiles as p50, p90, p99, p99.9 and always give the time unit. Say what was measured ("server-side handler time", "time seen by the client") and under what load. A p99 without load and setup is not a useful number.

## 3. Histograms: keeping millions of values

Keeping every duration in a `Vec` and sorting it is fine for a thousand values. A service that handles 10 000 requests per second for a day sees 864 million values; we cannot keep them all.

A **histogram** solves this. Instead of the values, it keeps **counters**: "how many values were between 20 and 21 µs", "between 21 and 22 µs", and so on. To find p99, it walks the counters from the smallest until it has passed 99% of the total count. Memory stays the same however many values you add.

```text
bucket (µs)   count
  20..21         12  #
  21..22        610  ##############################
  22..23        320  ################
 ...
2300..2400        9
2400..2500        8
```

The `hdrhistogram` crate (HDR = High Dynamic Range: it covers tiny and huge values in one structure) chooses the buckets for you. You tell it the smallest value, the largest value and how many **significant digits** must be exact. With 3 digits, a value of 4251 µs is stored as something between 4250 and 4260: accurate enough for latency. This histogram has 17 408 counters of 8 bytes, about 136 KiB, whether it holds ten values or ten billion.

Add the crate. We do not need its optional features (they serialize histograms to files):

```bash
cargo add hdrhistogram@7.6 --no-default-features
```

`src/bin/histogram.rs`:

```rust
use hdrhistogram::Histogram;
use std::hint::black_box;
use std::time::Instant;

fn handle_request(id: u64) -> u64 {
    let rounds = if id.is_multiple_of(50) {
        2_000_000
    } else {
        20_000
    };
    let mut x = id;
    for _ in 0..rounds {
        x = x.wrapping_mul(6364136223846793005).wrapping_add(1);
    }
    x
}

fn main() {
    // Values from 1 µs to 60 s, with 3 significant digits of precision.
    let mut hist = Histogram::<u64>::new_with_bounds(1, 60_000_000, 3).unwrap();

    for id in 0..20_000 {
        let start = Instant::now();
        black_box(handle_request(black_box(id)));
        let micros = start.elapsed().as_micros() as u64;
        hist.record(micros).unwrap();
    }

    println!("count  {}", hist.len());
    println!("mean   {:.1} µs", hist.mean());
    for q in [0.5, 0.9, 0.99, 0.999] {
        println!("p{:<5} {} µs", q * 100.0, hist.value_at_quantile(q));
    }
    println!("max    {} µs", hist.max());
}
```

**What happens:**

1. `Histogram::<u64>::new_with_bounds(1, 60_000_000, 3)` makes a histogram for values from 1 to 60 000 000 (we record microseconds, so 1 µs to 60 s) with 3 significant digits. It returns a `Result`, because impossible bounds are an error.
2. `record` adds one value. A value above the highest bound is an error (see Common mistakes).
3. `value_at_quantile(0.99)` returns p99. A **quantile** is the same idea as a percentile, written as a fraction: 0.99 instead of 99.
4. `len`, `mean` and `max` give the count, the average and the largest value.

`cargo run --release --bin histogram` (one run):

```text
count  20000
mean   204.0 µs
p50    23 µs
p90    24 µs
p99    4251 µs
p99.9  25199 µs
max    76927 µs
```

The shape is the same as before: a fast median, a slow p99, and a long tail of operating-system pauses.

> ⚡ **Performance:** recording into a histogram costs a few nanoseconds and never allocates, so it is safe inside a hot loop or a request handler. Two histograms can be added together with `add`, which lets every thread keep its own histogram (no lock needed while measuring) and merge them at the end. Never compute "the average of p99s" from several machines: percentiles cannot be averaged. Merge the histograms, then read p99.

## 4. Throughput and Little's law in practice

Throughput is easy to measure: count finished operations during a fixed time. Let us run cheap requests on 1, 2 and 4 threads for one second each and check Little's law. `src/bin/throughput.rs`:

```rust
use std::hint::black_box;
use std::thread;
use std::time::{Duration, Instant};

/// A cheap request: always the same amount of work.
fn handle_request(id: u64) -> u64 {
    let mut x = id;
    for _ in 0..20_000 {
        x = x.wrapping_mul(6364136223846793005).wrapping_add(1);
    }
    x
}

/// Runs `workers` threads for `run_for`. Returns (requests done, total busy time).
fn run(workers: u64, run_for: Duration) -> (u64, Duration) {
    thread::scope(|s| {
        let handles: Vec<_> = (0..workers)
            .map(|w| {
                s.spawn(move || {
                    let deadline = Instant::now() + run_for;
                    let mut done = 0u64;
                    let mut busy = Duration::ZERO;
                    while Instant::now() < deadline {
                        let start = Instant::now();
                        black_box(handle_request(black_box(w * 1_000_000 + done)));
                        busy += start.elapsed();
                        done += 1;
                    }
                    (done, busy)
                })
            })
            .collect();
        handles
            .into_iter()
            .map(|h| h.join().unwrap())
            .fold((0, Duration::ZERO), |(n, b), (dn, db)| (n + dn, b + db))
    })
}

fn main() {
    let run_for = Duration::from_secs(1);
    for workers in [1, 2, 4] {
        let (done, busy) = run(workers, run_for);
        let throughput = done as f64 / run_for.as_secs_f64();
        let mean_latency = busy.as_secs_f64() / done as f64;
        // Little's law: requests in progress = throughput × latency.
        let in_progress = throughput * mean_latency;
        println!(
            "workers {workers}: {throughput:>7.0} req/s, mean latency {:>5.1} µs, in progress {in_progress:.2}",
            mean_latency * 1e6
        );
    }
}
```

**What happens:**

1. Each worker thread loops until its deadline, times every request and adds the time to `busy`.
2. `fold` adds up the counts and busy times of all threads.
3. Throughput = requests ÷ seconds. Mean latency = busy time ÷ requests.
4. `throughput × mean_latency` is Little's `L`: the average number of requests in progress. With `workers` threads that are always busy, it must be equal to `workers`.

`cargo run --release --bin throughput` (one run):

```text
workers 1:   23133 req/s, mean latency  43.2 µs, in progress 1.00
workers 2:   42371 req/s, mean latency  47.1 µs, in progress 1.99
workers 4:   73023 req/s, mean latency  54.8 µs, in progress 4.00
```

Little's law holds: 1, 2 and 4 requests in progress. Throughput grows with the number of workers, while the latency of one request stays almost the same. That is the coffee shop with more baristas. (Latency is higher than in Section 2 because this program calls `Instant::now()` twice per request and because the M4 Pro has fast and slower cores; the operating system decides which core runs each thread.)

When the same program ran a second time while the Mac was busy with other work, the 4-worker line said `25102 req/s, mean latency 159.3 µs`. The threads had to wait for a free core, so every request got slower and throughput fell. A real service behaves the same way when it has more work than cores: requests start to **queue**, and latency grows long before throughput reaches zero. You will see this clearly with a real service in the next lesson.

## 5. Coordinated omission: the tester that lies

Here is a trap that has fooled many load tests. Imagine a tester that should send **one request every 10 ms** and measure each one. It sends a request, waits for the answer, then sends the next one at the next planned moment.

Now the server freezes for 200 ms (for example, a long pause while it cleans up memory). The tester sends request 50 and waits 200 ms. During that time it does **not** send requests 51 to 69, although the plan says it should. When it continues, the server is fast again, so all later requests look fast. The tester measured *one* slow request. A real user population would have sent twenty requests during the freeze, and *all* of them would have waited.

```text
plan:      |  |  |  |  |  |  |  |  |  |  |  |     one request every 10 ms
server:    ..[====== frozen 200 ms ======]..
naive:     50 sent ............. answer 50, then 51, 52 ... look fast   -> 1 slow value
honest:    51, 52 ... 69 were planned during the freeze and had to wait -> 19 more slow values
```

The tester and the server "coordinate": the slow server slows the tester down, so the tester *omits* the bad measurements. Hence the name **coordinated omission**.

We can simulate this exactly, without any real timing. `src/bin/omission.rs`:

```rust
use p19_latency::percentile;

/// How long the pretend server needs for request `k`, in milliseconds.
/// Request 50 hits a 200 ms pause; all others take 1 ms.
fn service_ms(k: u64) -> u64 {
    if k == 50 { 200 } else { 1 }
}

fn main() {
    let interval = 10; // the plan: one request every 10 ms
    let mut naive = Vec::new(); // measured from the moment it was really sent
    let mut honest = Vec::new(); // measured from the moment it SHOULD have been sent
    let mut server_free_at = 0; // the time when the server finishes its current request

    for k in 0..200 {
        let planned = k * interval;
        // The tester cannot send while it still waits for the previous answer.
        let sent = planned.max(server_free_at);
        let finished = sent + service_ms(k);
        server_free_at = finished;

        naive.push(finished - sent);
        honest.push(finished - planned);
    }
    naive.sort();
    honest.sort();

    println!("            p50   p90   p99   max (ms)");
    for (name, v) in [("naive", &naive), ("honest", &honest)] {
        println!(
            "{name:<8} {:>5} {:>5} {:>5} {:>5}",
            percentile(v, 50.0),
            percentile(v, 90.0),
            percentile(v, 99.0),
            v[v.len() - 1]
        );
    }
    let slow = |v: &[u64]| v.iter().filter(|&&ms| ms > 10).count();
    println!(
        "slower than 10 ms: naive {}, honest {}",
        slow(&naive),
        slow(&honest)
    );
}
```

**What happens:**

1. Time is just a number of milliseconds; nothing sleeps, so the result is the same on every computer.
2. Request `k` is **planned** at `k × 10` ms. It is really **sent** at the planned time or, if the server is still busy, when the server becomes free.
3. The **naive** latency starts when the request was sent. The **honest** latency starts when it was planned, because a real user would have clicked at that moment and started waiting.
4. Request 51 is planned at 510 ms, but the server is busy until 700 ms. Naive: 1 ms. Honest: 191 ms.

`cargo run --release --bin omission`:

```text
            p50   p90   p99   max (ms)
naive        1     1     1   200
honest       1    20   182   200
slower than 10 ms: naive 1, honest 22
```

The naive tester reports p99 = 1 ms: "the service is perfect". The honest view says that one request in ten waited 20 ms or more and p99 is 182 ms. Same server, same freeze, completely different conclusions.

How to avoid it:

- Measure latency **from the planned start time**, not from the moment the request finally went out.
- Use an **open model** tester that keeps sending at the planned rate, even while earlier requests are still waiting. The load tester in the next lesson can do this.
- If you can only measure the naive way, `hdrhistogram` can add the missing values for you with `record_correct` (Exercise 3).

> 🧪 **Testing:** the simulation above is also a good pattern for tests. Real timings are noisy, so tests of percentile math, histograms and "what does my tool report" should use made-up, fixed numbers. Keep real timing for benchmarks.

## Common mistakes

1. **Reporting only the average.** It hides the slow requests. Report p50, p99 and max together.
2. **Measuring in debug mode.** Debug builds can be ten or more times slower and change the shape of the distribution. Always `--release`.
3. **Too few samples for the percentile.** p99.9 of 1000 values is the single largest or second largest value; it jumps from run to run. For p99.9 you want at least tens of thousands of values.
4. **Averaging percentiles.** "Server A has p99 = 10 ms and server B has p99 = 30 ms, so p99 is 20 ms" is wrong. Merge the histograms (or the raw values), then read p99.
5. **Recording a value above the histogram's highest bound.** A typical cause is the wrong unit. In `histogram.rs`, replace the `record` line with `hist.record(start.elapsed().as_nanos() as u64).unwrap();`: now one operating-system pause of 70 ms is 70 000 000 *nanoseconds*, above the bound 60 000 000, and the program stops:

   ```text
   thread 'main' (5916363) panicked at src/bin/histogram.rs:26:56:
   called `Result::unwrap()` on an `Err` value: ValueOutOfRangeResizeDisabled
   ```

   (The number in brackets is the thread's ID; yours differs.) Use one unit everywhere and write it in the variable name (`micros`), choose bounds that cover every possible value, or use `Histogram::new(3)`, which grows as needed.

## Try it yourself

1. Run `latency` five times in a row. Which numbers stay almost the same, and which jump? (You should see p50 stable, max jumping.)
2. In `latency.rs`, change `1000` requests to `100_000`. Does p99 still show the slow group? Does p99.9 become more stable?
3. In `omission.rs`, change the freeze to 50 ms (`if k == 50 { 50 }`). How many requests are slower than 10 ms now? (Answer from the code: request 50 and the four planned during the freeze, so 5 in total.)

## Exercises

### Exercise 1 — The tail at scale (easy)

A page needs `n` independent calls to back-end services. Each call is fast (below its p99) with probability 0.99. Write a program `src/bin/fanout.rs` that prints, for `n` = 1, 10, 50 and 100, the percentage of pages that wait for at least one slow call.

<details>
<summary>Hint</summary>

All `n` calls are fast with probability `0.99` multiplied by itself `n` times: `f64::powi(0.99, n)`. "At least one slow" is the opposite: `1 - that`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let fast = 0.99; // a single call is fast (below its p99) 99% of the time
    for calls in [1, 10, 50, 100] {
        let all_fast = f64::powi(fast, calls);
        println!(
            "{calls:>3} calls: {:>4.1}% of pages wait for at least one slow call",
            (1.0 - all_fast) * 100.0
        );
    }
}
```

```text
  1 calls:  1.0% of pages wait for at least one slow call
 10 calls:  9.6% of pages wait for at least one slow call
 50 calls: 39.5% of pages wait for at least one slow call
100 calls: 63.4% of pages wait for at least one slow call
```

Why it works: independent events multiply. With 100 calls, the "rare" 1% case becomes the normal case for most pages, which is why large systems care about p99 and p99.9 of every component.

</details>

### Exercise 2 — Test the percentile edges (easy)

Write two more tests for `percentile` in `src/lib.rs`: (a) for the values 1..=20, p95 is 19 and p95.1 is 20; (b) `p = 101.0` panics with a message that contains `p must be in 0..=100`.

<details>
<summary>Hint</summary>

For (b) use `#[should_panic(expected = "...")]`. Remember that `expected` only needs to be a part of the message.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod exercise_tests {
    use super::percentile;

    #[test]
    fn p95_of_twenty_values() {
        let values: Vec<u64> = (1..=20).collect();
        assert_eq!(percentile(&values, 95.0), 19);
        assert_eq!(percentile(&values, 95.1), 20);
    }

    #[test]
    #[should_panic(expected = "p must be in 0..=100")]
    fn p_above_100_panics() {
        percentile(&[1, 2, 3], 101.0);
    }
}
```

`cargo test` now shows 5 passed tests. Why it works: 95% of 20 is exactly rank 19; 95.1% of 20 is 19.02, which `ceil` rounds up to rank 20. Such "just above the border" cases are where off-by-one bugs hide.

</details>

### Exercise 3 — Correcting coordinated omission (hard)

`hdrhistogram` has `record_correct(value, expected_interval)`. When a value is larger than the expected interval, it also records the values that the missing requests would have seen: `value - interval`, `value - 2 × interval`, and so on. Rewrite the simulation in `src/bin/corrected.rs` with two histograms: `plain` gets `record(measured)`, `corrected` gets `record_correct(measured, 10)`, where `measured = finished - sent` (the naive value). Print count, p90, p99 and max of both.

<details>
<summary>Hint</summary>

Use `Histogram::<u64>::new(3)` for both. The loop is the same as in `omission.rs`; you only need the naive value.

</details>

<details>
<summary>Answer</summary>

```rust
use hdrhistogram::Histogram;

fn service_ms(k: u64) -> u64 {
    if k == 50 { 200 } else { 1 }
}

fn main() {
    let interval = 10;
    let mut plain = Histogram::<u64>::new(3).unwrap();
    let mut corrected = Histogram::<u64>::new(3).unwrap();
    let mut server_free_at = 0;

    for k in 0..200 {
        let planned = k * interval;
        let sent = planned.max(server_free_at);
        let finished = sent + service_ms(k);
        server_free_at = finished;

        let measured = finished - sent;
        plain.record(measured).unwrap();
        // Also adds the values that the missing requests would have seen.
        corrected.record_correct(measured, interval).unwrap();
    }

    for (name, h) in [("plain", &plain), ("corrected", &corrected)] {
        println!(
            "{name:<10} count {:>3}  p90 {:>3}  p99 {:>3}  max {:>3}",
            h.len(),
            h.value_at_quantile(0.90),
            h.value_at_quantile(0.99),
            h.max()
        );
    }
}
```

```text
plain      count 200  p90   1  p99   1  max 200
corrected  count 219  p90   1  p99 180  max 200
```

Why it works: for the one 200 ms value, `record_correct` adds 190, 180, … 10: the 19 requests the tester skipped. p99 jumps from 1 to 180 ms, close to the honest 182 ms. The correction is an estimate (the count is 219, not 200, and p90 is still 1), so measuring from the planned start time is better when you can do it.

</details>

## Quiz

1. A service handles 500 req/s and each request takes 40 ms. How many requests are in progress on average?

<details>
<summary>Answer</summary>

Little's law: 500 × 0.040 = 20 requests in progress.

</details>

2. The mean latency is 100 µs and p50 is 23 µs. What does that tell you about the distribution?

<details>
<summary>Answer</summary>

It is skewed: most requests are fast, and a small number of very slow ones pull the mean up. The mean describes no real request; look at p99 and max to see the slow group.

</details>

3. What does "p99 = 2.4 ms" mean in one sentence?

<details>
<summary>Answer</summary>

99% of the measured requests took 2.4 ms or less; 1% took longer.

</details>

4. Why should a latency measurement of a service keep the outliers?

<details>
<summary>Answer</summary>

Because every outlier is a real request that a real user waited for. Removing them makes the service look better than it is. (Removing noise is acceptable only in micro-benchmarks of pure code.)

</details>

5. Explain coordinated omission in two sentences.

<details>
<summary>Answer</summary>

When the system is slow, a tester that waits for each answer sends fewer requests, so the slow period produces only one measurement instead of many. The result hides most of the slow requests and makes the tail look much better than what users experience.

</details>

6. Two servers report p99 = 10 ms and p99 = 30 ms. Can you say that the p99 of both together is 20 ms?

<details>
<summary>Answer</summary>

No. Percentiles cannot be averaged. Combine the raw values or merge the histograms, then compute p99 again.

</details>

## Summary

- **Latency** is the time of one operation; **throughput** is operations per second. They are different, and changes can move them in different directions.
- **Little's law:** requests in progress = throughput × latency. It tells you how many threads or connections a load needs.
- The **mean hides the slow requests.** Report p50, p99, p99.9 and max, with the unit and the load.
- Tail latency matters more as systems grow, because one user action touches many components.
- `hdrhistogram` records millions of values in fixed memory, gives any percentile, and merges exactly.
- **Coordinated omission** makes slow systems look fast. Measure from the planned start time and use an open-model load tester.

## Sources and further reading

- [hdrhistogram crate documentation](https://docs.rs/hdrhistogram/) — the API used in this lesson: `record`, `value_at_quantile`, `add`, `record_correct`.
- [HdrHistogram project site](https://hdrhistogram.github.io/HdrHistogram/) — the original design, with an explanation of buckets and significant digits.
- [The Rust Performance Book — Benchmarking](https://nnethercote.github.io/perf-book/benchmarking.html) — how to measure Rust code well.
- [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) — why services are judged by percentiles, and how SLOs are written (SRE: Site Reliability Engineering, the job of keeping services reliable).
- [`std::time::Instant`](https://doc.rust-lang.org/std/time/struct.Instant.html) — the clock used for all measurements here.

---

[🏠 Start](../START-HERE.md) · [← Phase 19 — Performance and profiling](README.md) · [📋 Phase 19](README.md) · [19.2 Load testing services →](02-load-testing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
