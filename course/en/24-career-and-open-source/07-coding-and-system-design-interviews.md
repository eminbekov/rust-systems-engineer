<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 24.6 Resume and job search](06-resume-and-job-search.md) · [📋 Phase 24](README.md) · [24.8 Keep learning: books, papers and habits →](08-keep-learning.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/07-coding-and-system-design-interviews.md) <!-- nav -->

# Lesson 24.7 — Coding and system design interviews

> **Phase 24 — Career and open source** · Lesson 7 of 9 · ⏱ about 135 minutes

## What you will learn

- Which kinds of interviews a systems engineer usually meets, and what each one checks.
- How to solve a coding problem out loud, using the method from Phase 07, with a worked example in Rust.
- A seven-step method for system design interviews, and a complete worked example: a URL shortener, with the
  numbers computed by a small Rust program.
- How to prepare stories for behavioural questions with the STAR method, and how to practise.

## Before you start

- You finished [Lesson 24.6 — Resume and job search](06-resume-and-job-search.md).
- You know the problem-solving method of [Lesson 07.18 — Problem-solving practice](../07-algorithms-and-data-structures/18-problem-solving-practice.md).
  This lesson builds on it instead of repeating it.
- You built a URL shortener in [Lesson 16.15](../16-backend-services/15-project-url-shortener-service.md) and know
  caching ([Lesson 16.10](../16-backend-services/10-caching-with-redis.md)) and replication
  ([Lesson 18.6](../18-distributed-systems/06-replication.md)).

## New words

| Word | Meaning |
|---|---|
| **interview loop** | The series of interviews for one job, often on one day or over a few weeks. |
| **coding interview** | An interview where you solve a programming problem while the interviewer watches and asks questions. |
| **system design interview** | An interview where you design a larger system on a whiteboard or in a document and discuss the trade-offs. |
| **behavioural interview** | An interview about how you worked in the past: conflicts, mistakes, decisions. |
| **functional requirement** | What the system must *do*, for example "redirect a short link to the long address". |
| **non-functional requirement** | How well it must do it: speed, availability, durability, cost. |
| **back-of-the-envelope estimate** | A rough calculation, good to a factor of two or so, done quickly to see which design is needed. |
| **trade-off** | A choice where you win something and lose something else, for example faster reads but slower writes. |
| **STAR** (Situation, Task, Action, Result) | A structure for telling a story about your past work in an interview. |
| **URL** (Uniform Resource Locator) | A web address such as `https://example.com/page`. |
| **HTTP** (HyperText Transfer Protocol) | The protocol of the web (Lesson 15.8); a redirect is an HTTP response that sends the browser to another URL. |
| **API** (Application Programming Interface) | Here: the requests a service accepts and the responses it gives. |
| **MB / GB / TB** (megabyte / gigabyte / terabyte) | 10⁶, 10⁹ and 10¹² bytes. |
| **LSM tree** (log-structured merge tree) | A storage structure that collects writes in memory and writes them to disk in sorted batches; good for many writes (Lesson 17.7). |
| **p99** | The 99th percentile: the time that 99% of requests stay under (Lesson 19.1). |

## 1. The interview loop

Companies combine different interviews. The names and the order vary; the table shows the common kinds and what
they check.

| Interview | What happens | What they check | Where this course prepared you |
|---|---|---|---|
| Screening call | A short talk about your background and the role. | Motivation, communication, basic fit. | Lessons 24.1, 24.6 |
| Coding | Solve one or two problems in 45–60 minutes, in a shared editor. | Data structures, algorithms, clean code, testing, talking while thinking. | Phase 07, this lesson |
| Systems coding | Build a small real component: a rate limiter, a cache, a parser, a thread pool. | Practical Rust, concurrency, error handling. | Phases 05, 06, 14, 15 |
| Fundamentals | Questions about operating systems, networks, storage, concurrency. | Depth "below the API". | Phases 12–18 |
| System design | Design a service or an infrastructure piece; discuss failures and numbers. | Structure, trade-offs, estimates, reliability. | Phases 16–21, this lesson |
| Project deep dive | Explain one of your projects in detail; answer "why not X?". | Real understanding of your own work. | Phase 23, Lesson 24.2 |
| Behavioural | Stories from your past work. | Ownership, learning, working with people. | Section 5 of this lesson |

In all of them, the interviewer wants to see **how you think**, not only the final answer. A clear, structured
attempt with a small mistake often impresses more than a silent perfect answer.

## 2. Coding interviews: the method, out loud

Use the seven steps from Lesson 07.18 (understand, small examples, brute force, find the waste, better algorithm,
tests, measure). In an interview, add one rule: **say every step out loud.** The interviewer cannot help or give
points for thoughts they cannot hear.

**A worked example.** "Given a list of busy time intervals `(start, end)`, merge the ones that overlap and return
them sorted."

1. **Understand, and ask.** "Do intervals that only touch, like `(1, 4)` and `(4, 5)`, count as overlapping?" — "Yes."
   "Can the input be empty?" — "Yes." "Is it sorted?" — "No."
2. **Small examples.** `[(1, 3), (2, 6), (8, 10)]` gives `[(1, 6), (8, 10)]`; empty gives empty; `(1, 10)` and
   `(2, 3)` give `(1, 10)` (one inside the other).
3. **Brute force, briefly.** "I could compare every pair and merge until nothing changes; that is O(n²) or worse."
4. **Find the waste and improve.** "If I sort by start, overlapping intervals become neighbours. Then one pass is
   enough: either extend the last merged interval or start a new one. Sorting is O(n log n), the pass is O(n)."
5. **Write it, then test it** with the examples from step 2.

```rust
// Interview practice: merge overlapping time intervals.

/// Merges overlapping or touching intervals `(start, end)` and returns them sorted.
/// Example: [(1, 3), (2, 6), (8, 10)] becomes [(1, 6), (8, 10)].
fn merge(mut intervals: Vec<(u32, u32)>) -> Vec<(u32, u32)> {
    intervals.sort_unstable();
    let mut merged: Vec<(u32, u32)> = Vec::with_capacity(intervals.len());
    for (start, end) in intervals {
        match merged.last_mut() {
            Some(last) if start <= last.1 => last.1 = last.1.max(end),
            _ => merged.push((start, end)),
        }
    }
    merged
}

fn main() {
    let busy = vec![(8, 10), (1, 3), (2, 6), (15, 18), (17, 20)];
    println!("{:?}", merge(busy));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn merges_overlaps_and_keeps_gaps() {
        assert_eq!(merge(vec![(1, 3), (2, 6), (8, 10)]), vec![(1, 6), (8, 10)]);
    }

    #[test]
    fn edge_cases() {
        assert_eq!(merge(vec![]), vec![]);
        assert_eq!(merge(vec![(5, 7)]), vec![(5, 7)]);
        assert_eq!(merge(vec![(1, 4), (4, 5)]), vec![(1, 5)]); // touching
        assert_eq!(merge(vec![(1, 10), (2, 3)]), vec![(1, 10)]); // contained
    }
}
```

**What happens:**

1. `merge` takes the vector by value (`mut intervals`), so it may sort it in place without copying. Tuples sort by
   the first element, then by the second, which is exactly "by start".
2. `merged.last_mut()` gives a mutable reference to the last merged interval, or `None` when `merged` is empty.
3. The match guard `if start <= last.1` checks for overlap or touching; then the end becomes the larger of the two
   ends. `max` matters for the "contained" case: `(1, 10)` followed by `(2, 3)` must keep 10.
4. Otherwise the interval starts a new group. `Vec::with_capacity` avoids repeated growth (Lesson 07.3).

Save it as `src/bin/intervals.rs` in `code/p24-career` and run it:

```bash
cargo run -q --bin intervals
cargo test --bin intervals
```

```text
[(1, 6), (8, 10), (15, 20)]
```

```text
running 2 tests
test tests::edge_cases ... ok
test tests::merges_overlaps_and_keeps_gaps ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

6. **State the complexity at the end**: "Time O(n log n) for the sort, extra memory O(n) for the result."

> 💡 **Tip:** In Rust interviews, know the standard tools by heart: `Vec`, `VecDeque`, `HashMap`, `HashSet`,
> `BTreeMap`, `BinaryHeap` (with `std::cmp::Reverse` for a min-heap), `sort_unstable`, `sort_by_key`,
> `binary_search`, `iter().rev()`, `windows(2)`. If the borrow checker blocks you, explain the ownership problem
> out loud; a short `clone()` with the comment "I would avoid this copy in production" is better than silence.

> 🧪 **Testing:** Write the edge-case tests *before* the interviewer asks. Empty input, one element, touching and
> contained intervals: naming them shows the habit of thinking about boundaries (Lesson 05.6).

## 3. System design interviews: a method in seven steps

A system design interview is a conversation of about 45 minutes about a question like "design a URL shortener" or
"design a metrics pipeline". There is no single right answer. The interviewer watches whether you **structure** the
problem, **justify** decisions with numbers, and **see the failures**.

| Step | What you do | Time |
|---|---|---|
| 1. Requirements | Ask what the system must do (functional) and how well (non-functional). Write both lists down. | 5 min |
| 2. Estimates | Requests per second, data size, growth. Decide whether one machine is enough. | 5 min |
| 3. API | The main requests and responses. | 3 min |
| 4. Data model | What is stored, the keys, the access pattern. | 5 min |
| 5. High-level design | A diagram of the parts and the data flow. | 10 min |
| 6. Deep dives | Two or three hard parts: the bottleneck, failures, consistency. | 15 min |
| 7. Trade-offs and wrap-up | What you chose, what you gave up, what you would do next. | 2 min |

The worked example below follows the steps. It is a design you know from the inside, because you built a small
version in Lesson 16.15.

### Step 1 — Requirements

```text
Functional
- Create a short link for a long URL; return a short code such as "aZ3k9Q".
- Visiting https://<host>/<code> redirects to the long URL.
- Links never expire (simplification for the interview).

Non-functional
- Redirects are fast: p99 under 50 ms inside our data centre.
- Redirects keep working when one machine fails (high availability).
- A created link is never lost (durability).
- Codes should not be easy to guess one after another.
```

Ask about the scale. Assume the interviewer says: "one million new links per day, about 100 visits per link,
keep links for five years". These numbers are invented for the exercise.

### Step 2 — Estimates, with a program

Estimates are simple arithmetic, but it is easy to slip by a factor of a thousand. Write them as a program once, and
reuse it for every practice design. Create `src/bin/estimate.rs`:

```rust
// Back-of-the-envelope numbers for a URL shortener, as in a system design interview.

const SECONDS_PER_DAY: f64 = 86_400.0;
const DAYS_PER_YEAR: f64 = 365.0;

/// What we assume about the system.
struct Load {
    new_links_per_day: f64,
    reads_per_write: f64,
    bytes_per_link: f64,
    years: f64,
    peak_factor: f64,
}

/// What follows from the assumptions.
struct Estimate {
    writes_per_second: f64,
    reads_per_second: f64,
    peak_reads_per_second: f64,
    total_links: f64,
    storage_gb: f64,
}

fn estimate(load: &Load) -> Estimate {
    let writes_per_second = load.new_links_per_day / SECONDS_PER_DAY;
    let reads_per_second = writes_per_second * load.reads_per_write;
    let total_links = load.new_links_per_day * DAYS_PER_YEAR * load.years;
    Estimate {
        writes_per_second,
        reads_per_second,
        peak_reads_per_second: reads_per_second * load.peak_factor,
        total_links,
        storage_gb: total_links * load.bytes_per_link / 1e9,
    }
}

/// The shortest code made of 62 symbols (a-z, A-Z, 0-9) that gives at least
/// `headroom` times more possible codes than links.
fn code_length(total_links: f64, headroom: f64) -> i32 {
    let mut length = 1;
    while 62f64.powi(length) < total_links * headroom {
        length += 1;
    }
    length
}

fn main() {
    let load = Load {
        new_links_per_day: 1_000_000.0,
        reads_per_write: 100.0,
        bytes_per_link: 500.0,
        years: 5.0,
        peak_factor: 3.0,
    };
    let e = estimate(&load);
    println!("writes per second:     {:>12.1}", e.writes_per_second);
    println!("reads per second:      {:>12.1}", e.reads_per_second);
    println!("peak reads per second: {:>12.1}", e.peak_reads_per_second);
    println!(
        "links after {} years:   {:>12.2e}",
        load.years, e.total_links
    );
    println!("storage (GB):          {:>12.1}", e.storage_gb);
    println!(
        "short code length:     {:>12}",
        code_length(e.total_links, 10.0)
    );
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn one_link_per_second_on_average() {
        let load = Load {
            new_links_per_day: SECONDS_PER_DAY,
            reads_per_write: 10.0,
            bytes_per_link: 1_000.0,
            years: 1.0,
            peak_factor: 2.0,
        };
        let e = estimate(&load);
        assert_eq!(e.writes_per_second, 1.0);
        assert_eq!(e.reads_per_second, 10.0);
        assert_eq!(e.peak_reads_per_second, 20.0);
        assert_eq!(e.total_links, SECONDS_PER_DAY * 365.0);
        assert!((e.storage_gb - 31.536).abs() < 1e-9);
    }

    #[test]
    fn code_length_grows_with_the_number_of_links() {
        assert_eq!(code_length(62.0, 1.0), 1);
        assert_eq!(code_length(63.0, 1.0), 2);
        assert_eq!(code_length(1e9, 10.0), 6);
    }
}
```

**What happens:**

1. `Load` holds the assumptions and `Estimate` the results, so every number has a name. In an interview you write
   the same on the whiteboard: assumptions on the left, results on the right.
2. Average writes per second are links per day divided by 86,400 seconds; reads are writes times the read ratio.
   Traffic is not flat over the day, so `peak_factor` (here 3) estimates the busiest hour.
3. Storage is links × bytes per link. 500 bytes covers a long URL, the code and a few fields such as the creation
   time. `1e9` converts bytes to gigabytes.
4. `code_length` finds the shortest code with 62 possible symbols per position that leaves ten times more codes than
   links, so random codes rarely collide. `powi` raises a float to an integer power.
5. The first test uses numbers that are easy to check by hand: 86,400 links a day is exactly one per second.

```bash
cargo run -q --bin estimate
cargo test --bin estimate
```

```text
writes per second:             11.6
reads per second:            1157.4
peak reads per second:       3472.2
links after 5 years:         1.82e9
storage (GB):                 912.5
short code length:                6
```

```text
running 2 tests
test tests::code_length_grows_with_the_number_of_links ... ok
test tests::one_link_per_second_on_average ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Reading the numbers** is the important part. About 12 writes and 1,200 reads per second on average, 3,500 at peak:
one database with a cache in front handles this comfortably (Phase 16). Under a terabyte of data in five years: it fits
on one machine, with replicas for availability. So the honest conclusion is **"no sharding needed at this scale"**.
Saying that, with numbers, is a strong answer; adding sharding "because big companies do it" is a weak one.

> ⚡ **Performance:** The read-to-write ratio of 100 to 1 says where to spend effort: on the read path. A cache of the
> most visited codes takes most of the load off the database.

### Step 3 — API

```text
POST /links        body: {"url": "https://example.com/a/very/long/page"}
                   201 Created, body: {"code": "aZ3k9Q"}
GET  /{code}       302 Found, header Location: https://example.com/a/very/long/page
                   404 Not Found if the code does not exist
```

Mention the choice of redirect status: **301** (moved permanently) lets browsers cache the redirect, which saves
requests but hides later visits from your statistics; **302** (found, temporary) sends every visit to you.

### Step 4 — Data model

One table (or key-value collection) `links`: `code` (the primary key), `url`, `created_at`. The access pattern is a
lookup by exact key, the easiest case for any database, with an index on the key (Lesson 17.5).

### Step 5 — High-level design

```text
 clients ──> load balancer ──> app servers (stateless, 2+)
                                   │  read: cache first
                                   ├──> cache (Redis): code -> url
                                   │  miss / write
                                   └──> database primary ──replication──> replica(s)
```

The app servers keep no state, so you can add or replace them freely (Lesson 15.14). The cache answers most reads;
the database is the source of truth.

### Step 6 — Deep dives

Pick two or three, or let the interviewer choose:

- **Generating codes.** Option A: a random 6-character code, insert it, retry on the rare collision (the unique key
  catches it). Option B: a counter turned into base-62 text: no collisions, but codes are guessable one after another,
  which breaks a non-functional requirement. Choose A, and say why.
- **A just-created link is not on the replica yet.** Replication is asynchronous (Lesson 18.6), so a redirect right
  after creation may miss. Fix: on a miss, read from the primary before answering 404.
- **The database primary fails.** Redirects of cached codes keep working; creating links fails with a clear error
  until a replica is promoted. State it: availability of reads matters more than of writes here.
- **Abuse.** Rate limits per client on `POST /links`, and checking new URLs against known bad sites (Lesson 20.8).

### Step 7 — Trade-offs and wrap-up

In two sentences: "A single primary database with replicas and a cache fits the load with a large margin; random
6-character codes avoid guessing at the cost of a rare retry. Next I would add click statistics through a message
queue (Lesson 16.11), so that redirects never wait for analytics."

> 🔒 **Security:** A URL shortener can be used to hide where a link really goes. Mention it in the deep dive: check
> new URLs, allow users to report abuse, and never redirect to non-web schemes such as `javascript:`.

## 4. Practising system design

- **Use designs you know from the inside.** This course gave you many: a key-value store (17.14), a replicated store
  (18.14), a load balancer (15.15), a job queue (14.15), a service on Kubernetes (21.13), your capstone (Phase 23).
- **Practise with a timer** and a blank page. Follow the seven steps even when you know the answer.
- **Say the numbers.** Keep a short list of rough costs in your head: a memory access is nanoseconds, a network
  round trip inside a data centre is well under a millisecond, a disk `fsync` can take milliseconds, a request across
  continents takes tens of milliseconds or more (Phases 12, 15 and 17).
- **Get feedback.** Practise with a friend, a colleague or a study group; one person asks, the other designs, then
  swap. Write down the questions you could not answer and study them.

## 5. Behavioural questions: the STAR method

"Tell me about a time when you made a mistake." "…when you disagreed with someone." "…when you learned something
new quickly." Prepare five true stories and tell each with **STAR**:

| Part | Content | Example (from course work) |
|---|---|---|
| **Situation** | The context, in one or two sentences. | "My key-value store lost writes in a crash test." |
| **Task** | What you had to achieve. | "Find out why and guarantee durability." |
| **Action** | What *you* did, step by step. | "I wrote a test that kills the process, found that I replied before `fsync`, moved the reply after it." |
| **Result** | The outcome, with a number if possible, and what you learned. | "No lost writes in 1,000 crash runs; I now write the crash test first." |

Good stories come from incidents, bugs you found, a review you received, a design you changed, a contribution
(Lessons 24.4 and 24.5). Keep each to about two minutes.

## Common mistakes

**Coding in silence.** The interviewer sees only typing. Talk: what you try, why, and what you check next.

**Jumping to code before the examples.** Two minutes on examples and edge cases saves ten minutes of rewriting.

**Designing for an imaginary scale.** Without estimates, people add shards, queues and caches everywhere. Compute
first; add parts only when a number asks for them.

**Ignoring failures.** "What happens when this machine dies?" will come. Ask it yourself for each part of your diagram.

**Memorised answers.** Interviewers change the question slightly to see whether you understand. Practise the method,
not the answers.

## Try it yourself

1. Add `intervals` and `estimate` to `code/p24-career`, run them and their tests.
2. Solve one problem from Lesson 07.18 again, out loud, with a 30-minute timer. Record yourself once and listen.
3. Do the full URL shortener design on paper in 45 minutes, without looking at this lesson, then compare.
4. Write your five STAR stories in `notes/interview-stories.md`.

## Exercises

### Exercise 1 — Meeting rooms (medium)

"Given meetings `(start, end)`, how many rooms are needed so that no two meetings in one room overlap? A meeting that
ends at 10 and one that starts at 10 can share a room." Solve it out loud with the method, write
`rooms_needed(meetings: &[(u32, u32)]) -> usize` in `src/bin/rooms.rs`, and write the tests first.

<details>
<summary>Hint</summary>

Sort all start times and, separately, all end times. Walk through the starts; before taking a room for a start, free
every room whose meeting ended at or before that start. The answer is the largest number of rooms in use at once.

</details>

<details>
<summary>Answer</summary>

```rust
/// How many rooms are needed so that no two meetings in one room overlap?
/// A meeting that ends at 10 and one that starts at 10 can share a room.
fn rooms_needed(meetings: &[(u32, u32)]) -> usize {
    let mut starts: Vec<u32> = meetings.iter().map(|m| m.0).collect();
    let mut ends: Vec<u32> = meetings.iter().map(|m| m.1).collect();
    starts.sort_unstable();
    ends.sort_unstable();

    let (mut in_use, mut most, mut next_end) = (0, 0, 0);
    for start in starts {
        // Free every room whose meeting ended before or at this start.
        while next_end < ends.len() && ends[next_end] <= start {
            in_use -= 1;
            next_end += 1;
        }
        in_use += 1;
        most = most.max(in_use);
    }
    most
}

fn main() {
    let day = [(9, 10), (9, 12), (10, 11), (11, 13), (12, 13)];
    println!("rooms needed: {}", rooms_needed(&day));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_the_largest_overlap() {
        assert_eq!(rooms_needed(&[]), 0);
        assert_eq!(rooms_needed(&[(1, 5), (2, 6), (5, 7)]), 2);
        assert_eq!(rooms_needed(&[(1, 10), (2, 3), (4, 5)]), 2);
        assert_eq!(rooms_needed(&[(1, 2), (2, 3)]), 1);
        assert_eq!(rooms_needed(&[(1, 4), (1, 4), (1, 4)]), 3);
    }
}
```

```text
rooms needed: 2
```

Why it works: this is the two-pointer technique (Lesson 07.15) over two sorted lists. `<=` in the `while` makes
"ends at 10, starts at 10" share a room, as required. Time O(n log n) for the sorts, memory O(n).

</details>

### Exercise 2 — Estimate a metrics system (medium)

Design question: "2,000 servers each send 500 metric values every 10 seconds; each stored value takes 16 bytes; keep 30
days." Compute by hand the values per second and the storage, then add a function
`metrics_storage_gb(servers, metrics, interval_s, bytes, days) -> f64` to `estimate.rs` with a test that is easy to
check by hand. What does the result mean for the design?

<details>
<summary>Hint</summary>

Values per second = servers × metrics ÷ interval. Bytes per day = values per second × bytes × 86,400.

</details>

<details>
<summary>Answer</summary>

By hand: 2,000 × 500 ÷ 10 = 100,000 values per second; × 16 bytes = 1.6 MB per second; × 86,400 ≈ 138 GB per day;
× 30 ≈ 4,147 GB, about 4 TB (terabytes).

```rust
/// Storage for a metrics system: every server sends `metrics` values every `interval_s` seconds.
fn metrics_storage_gb(servers: f64, metrics: f64, interval_s: f64, bytes: f64, days: f64) -> f64 {
    let points_per_second = servers * metrics / interval_s;
    points_per_second * bytes * SECONDS_PER_DAY * days / 1e9
}

#[test]
fn metrics_storage_for_one_point_per_second() {
    // 1 server, 1 metric, every second, 1 byte, 1 day = 86,400 bytes.
    let gb = metrics_storage_gb(1.0, 1.0, 1.0, 1.0, 1.0);
    assert!((gb - 86_400.0 / 1e9).abs() < 1e-15);
}
```

With a line at the end of `main` that prints `metrics_storage_gb(2_000.0, 500.0, 10.0, 16.0, 30.0)`:

```text
metrics storage (GB):        4147.2
```

Meaning: 100,000 writes per second is far more than the URL shortener's 12, and the data grows by over 100 GB per day.
This design needs a storage engine built for many small appends (an LSM tree, Lesson 17.7) and probably compression or
downsampling of old data; the write path, not the read path, is the centre of the design.

</details>

### Exercise 3 — One STAR story (easy, no code)

Write one STAR story about a bug you found in a course project. Keep it under 150 words.

<details>
<summary>Hint</summary>

Most of the words go to **Action**. The **Result** should contain what changed in how you work.

</details>

<details>
<summary>Answer</summary>

There is no single answer. Check yours: the situation is clear to a stranger in two sentences; the actions are things
*you* did; the result has a fact (a number, a merged fix, a test that now exists) and a lesson you still apply.

</details>

## Quiz

1. Why should you talk while you solve a coding problem?

<details>
<summary>Answer</summary>

The interviewer evaluates your thinking, not only the result. Talking also lets them give hints and lets you notice
your own mistakes.

</details>

2. What are the seven steps of the system design method?

<details>
<summary>Answer</summary>

Requirements, estimates, API, data model, high-level design, deep dives, trade-offs and wrap-up.

</details>

3. The estimate gives 1,200 reads per second and under 1 TB of data. Do you shard the database?

<details>
<summary>Answer</summary>

No. One database with replicas and a cache handles this; sharding would add complexity without need. Say so, with the
numbers.

</details>

4. Why are random short codes better than a counter in this design?

<details>
<summary>Answer</summary>

Counter-based codes are easy to guess one after another, which exposes other people's links. Random codes avoid that; the
rare collision is caught by the unique key and retried.

</details>

5. What does STAR stand for, and which part gets the most words?

<details>
<summary>Answer</summary>

Situation, Task, Action, Result. The Action part, because it shows what you did.

</details>

## Summary

- Interview loops combine coding, systems coding, fundamentals, system design, a project deep dive and behavioural
  questions; all of them look at how you think.
- In coding interviews, follow the Phase 07 method out loud: examples and edge cases first, then a simple solution, then
  a better one, then tests and complexity.
- In system design, use seven steps: requirements, estimates, API, data model, diagram, deep dives, trade-offs.
- Estimate with simple arithmetic (a small program helps) and let the numbers decide the design.
- Prepare five true STAR stories, and practise everything with a timer and feedback.

## Sources and further reading

- [Tech Interview Handbook](https://www.techinterviewhandbook.org/) — free guides to coding, system design and
  behavioural interviews.
- [The System Design Primer](https://github.com/donnemartin/system-design-primer) — a large, free collection of system
  design topics and practice questions.
- [Latency numbers every programmer should know (interactive)](https://colin-scott.github.io/personal_website/research/interactive_latency.html) —
  rough costs of memory, disk and network operations for estimates.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — the standard book about the ideas behind
  system design: replication, partitioning, consistency.
- [Massachusetts Institute of Technology, Career Advising — The STAR method for behavioral interviews](https://capd.mit.edu/resources/the-star-method-for-behavioral-interviews/) —
  a short guide to STAR stories.
- [std::collections](https://doc.rust-lang.org/std/collections/index.html) — when to use which standard collection.

---

[🏠 Start](../START-HERE.md) · [← 24.6 Resume and job search](06-resume-and-job-search.md) · [📋 Phase 24](README.md) · [24.8 Keep learning: books, papers and habits →](08-keep-learning.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
