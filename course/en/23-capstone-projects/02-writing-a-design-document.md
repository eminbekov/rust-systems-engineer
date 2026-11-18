<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 23.1 How capstone projects work](01-how-capstone-projects-work.md) · [📋 Phase 23](README.md) · [23.3 Capstone option 1: a distributed key-value store →](03-capstone-distributed-key-value-store.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/02-writing-a-design-document.md) <!-- nav -->

# Lesson 23.2 — Writing a design document

> **Phase 23 — Capstone projects** · Lesson 2 of 10 · ⏱ about 120 minutes

## What you will learn

- What a design document is, who reads it, and why you write it before the code.
- A template for `docs/design.md`, section by section, and the question each section answers.
- How to write goals, non-goals and requirements that can be checked, with numbers and units.
- How to make back-of-the-envelope estimates, with a small Rust program that you can test.
- How to list failure modes, record decisions in ADRs, and review a design.

## Before you start

- You finished [Lesson 23.1 — How capstone projects work](01-how-capstone-projects-work.md) and have the skeleton `code/p23-skeleton`.
- The examples use the URL shortener from [Lesson 16.15 — Project: a URL shortener service](../16-backend-services/15-project-url-shortener-service.md), because you already know how it works. Here we only write *about* it.
- Useful background: percentiles from [Lesson 19.1 — Latency, throughput and percentiles](../19-performance-and-profiling/01-latency-throughput-and-percentiles.md), threat modeling from [Lesson 10.2 — Threat modeling with STRIDE](../10-security-fundamentals/02-threat-modeling-with-stride.md), and fault, failure and availability from [Lesson 20.1 — What reliability means](../20-reliability-and-hardening/01-what-reliability-means.md).

## New words

| Word | Meaning |
|---|---|
| **design document** (design doc) | A short text that describes a problem, the chosen design, the alternatives and the risks, written before the code so that others can review it. |
| **requirement** | Something the system must do (functional requirement) or a property it must have, such as speed or durability (non-functional requirement). |
| **back-of-the-envelope estimate** | A quick, rough calculation of load, storage or time, good to within about a factor of ten. |
| **order of magnitude** | A rough size given as a power of ten: 100 ns and 300 ns have the same order of magnitude; 100 ns and 100 µs do not. |
| **capacity** | How much load a system can handle: requests per second, bytes stored, connections. |
| **failure mode** | One specific way in which a part of the system can fail, for example "the disk is full". |
| **trade-off** | A choice where getting more of one good thing means getting less of another. |
| **ADR** (Architecture Decision Record) | A short file that records one important decision: the context, the decision and its consequences. |
| **RFC** (Request for Comments) | A document that proposes a change and asks others for feedback. Rust itself changes through RFCs. |
| **SLO** (Service Level Objective) | A target for a measured property of a service, such as "99% of redirects within 50 ms" (Lesson 21.11). |
| **p99** (99th percentile) | The value below which 99% of measurements fall. "p99 latency 50 ms" means 1 request in 100 is slower than 50 ms. |
| **QPS** (Queries Per Second) | How many requests a system receives or serves per second. |
| **open question** | A part of the design that you have not decided yet. Writing it down is better than pretending it is solved. |
| **URL** (Uniform Resource Locator) | A web address, such as `https://example.test/v1/kv/color`. |
| **HTTP** (HyperText Transfer Protocol) | The request-and-response protocol of the web; HTTPS is HTTP over an encrypted connection. |
| **STRIDE** | A checklist of six threat types — Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege (Lesson 10.2). |
| **MB / MiB** (megabyte / mebibyte) | 1 MB is 1,000,000 bytes; 1 MiB is 1,048,576 bytes (2²⁰). KiB, GiB and TiB are the binary units around it. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The usual encoding of text as bytes; Rust's `String` and `&str` are always valid UTF-8. |

## 1. Why write before you code

Changing a sentence costs a minute. Changing a data format after three weeks of code costs days. A design document moves the hard decisions to the moment when they are cheapest to change.

A design document does three jobs:

1. **It makes you think clearly.** Many "I know what to build" plans fall apart when you try to write the failure modes or the numbers. That is the point: you find the gap on paper.
2. **It lets others review the design.** A reviewer can read three pages in fifteen minutes. Nobody can review three weeks of code that fast.
3. **It explains the project later.** Months from now, you (or an interviewer reading your repository) will ask "why is it like this?". The document answers.

Real teams write design documents for most non-trivial changes. Some call them RFCs; the Rust project itself decides language changes through public RFCs. The form differs; the idea is the same.

A good design document is **short** (2–5 pages for a capstone), **specific** (numbers, formats, names) and **honest** (it lists risks and open questions). It describes *decisions* and *reasons*, not every function.

## 2. The template

Create `docs/design.md` in your capstone with this skeleton:

```markdown
# <Project name> — design

Status: draft | in review | accepted | superseded · Last updated: 2026-12-01

## 1. Summary
## 2. Context and problem
## 3. Goals and non-goals
## 4. Requirements
## 5. Estimates
## 6. Design
### 6.1 Architecture
### 6.2 API and data formats
### 6.3 Key algorithms and data flow
## 7. Failure modes
## 8. Alternatives considered
## 9. Test plan
## 10. Observability and operations
## 11. Security
## 12. Milestones
## 13. Risks and open questions
```

Each section answers one question:

| Section | The question it answers |
|---|---|
| Summary | In three sentences: what is this, and what is the main idea? |
| Context and problem | Why does this need to exist? What is wrong or missing today? |
| Goals and non-goals | What will it do, and what will it explicitly not do? |
| Requirements | Which behaviour and which numbers must it reach (latency, throughput, durability)? |
| Estimates | How much load, storage and memory, roughly? Does one machine suffice? |
| Design | What are the parts, how do they talk, which formats and algorithms? |
| Failure modes | What breaks, how do we notice, what does the user see, what do we do? |
| Alternatives | What else could we do, and why did we not? |
| Test plan | How will we prove each requirement? Which test for which risk? |
| Observability and operations | Which logs, metrics and health checks? How is it started, stopped and upgraded? |
| Security | What could an attacker do (STRIDE, Lesson 10.2), and what stops them? |
| Milestones | In which order will it be built, and what is "done" for each step? |
| Risks and open questions | What might go wrong with the plan itself? What is still undecided? |

The `Status` line matters. A document is a *draft* while you write it, *in review* while someone reads it, *accepted* when you start building, and *superseded* if a later document replaces it. When the design changes during the project, update the document; an outdated design document is worse than none.

## 3. Goals, non-goals and requirements

A **goal** says what the system achieves. A **non-goal** says what it does not try to do, even though someone might expect it. Non-goals are the cheapest way to stop scope creep, because you decide them once, in writing, instead of fighting the same temptation every week.

Example for the URL shortener:

```markdown
## 3. Goals and non-goals
Goals
- Turn a long URL into a short code, and redirect visitors from the code to the URL.
- Count clicks per code.
Non-goals
- No user accounts: one admin token creates links.
- No analytics beyond a click counter (no countries, no browsers).
- No multi-region deployment: one region, one database.
```

A **requirement** must be checkable. If you cannot write a test or a measurement for it, it is not a requirement yet. The usual trick is to add a number, a unit and a condition:

| Vague | Checkable |
|---|---|
| "Redirects are fast." | "p99 latency of `GET /{code}` is below 50 ms at 1,000 QPS on one node, measured with `oha` for 60 s." |
| "Links are not lost." | "A link is stored durably before `201` is returned; after `kill -9` of the service and a restart, every acknowledged link redirects." |
| "It handles bad input." | "A URL longer than 2,048 bytes or not `http`/`https` gets `422`; the fuzz target for the request parser runs 10 minutes without a panic." |
| "It is highly available." | "Redirects keep working (with `x-cache: unavailable`) while Redis is down." |

Each checkable requirement later becomes an acceptance test or a benchmark in your milestones, and a row in `docs/results.md`.

> 💡 **Tip:** Write the requirement and the test that proves it in the same line of the test plan (Section 9 of the document): "R3 durability → test `m2_acknowledged_link_survives_kill`". Then nothing is promised without a check.

## 4. Back-of-the-envelope estimates

Before you design, you need a feeling for the size of the problem. Does everything fit in the memory of one machine? Is the disk the limit, or the network? A back-of-the-envelope estimate answers such questions in minutes. It is allowed to be wrong by a factor of two; it is not allowed to be wrong by a factor of a thousand.

Two facts help with most estimates. A day has 86,400 seconds, so **1 million events per day is about 12 per second**. And traffic is not flat: the busiest minute is often several times the average, so plan for a **peak factor** (5 is a common guess).

You also need rough costs of basic operations. These are orders of magnitude on modern hardware, not exact values; your own measurements (Phase 19) always win:

| Operation | Rough time |
|---|---|
| Read a value from the CPU (Central Processing Unit) cache | about 1 ns |
| Read a value from main memory | about 100 ns |
| Random 4 KiB read from an SSD (Solid-State Drive) | about 20–100 µs |
| `fsync` of a small write on an SSD (Lesson 17.2) | about 0.1–10 ms |
| Network round trip inside one data center | about 0.1–0.5 ms |
| Network round trip between continents | about 100–200 ms |

A spreadsheet works for estimates, but a tiny Rust program is better for a capstone: the assumptions are named, the arithmetic is tested, and you can rerun it when an assumption changes. Create it next to your other practice code:

```bash
cd ~/projects/rust
cargo new code/p23-estimate
```

`src/main.rs`:

```rust
/// The numbers we assume about the load. Every field has its unit in the name.
struct Load {
    writes_per_day: u64,
    reads_per_write: u64,
    bytes_per_record: u64,
    days_kept: u64,
    replicas: u64,
    peak_factor: u64,
}

const SECONDS_PER_DAY: f64 = 86_400.0;

/// Average events per second for a number of events per day.
fn per_second(per_day: u64) -> f64 {
    per_day as f64 / SECONDS_PER_DAY
}

/// Bytes on disk for all kept records, all replicas included.
fn storage_bytes(load: &Load) -> u64 {
    load.writes_per_day * load.days_kept * load.bytes_per_record * load.replicas
}

/// Formats bytes with binary units: KiB = 1024 bytes, MiB = 1024 KiB, …
fn human_bytes(bytes: u64) -> String {
    let units = ["B", "KiB", "MiB", "GiB", "TiB", "PiB"];
    let mut value = bytes as f64;
    let mut unit = 0;
    while value >= 1024.0 && unit < units.len() - 1 {
        value /= 1024.0;
        unit += 1;
    }
    format!("{value:.1} {}", units[unit])
}

fn main() {
    let load = Load {
        writes_per_day: 1_000_000,
        reads_per_write: 100,
        bytes_per_record: 500,
        days_kept: 5 * 365,
        replicas: 3,
        peak_factor: 5,
    };
    let peak = load.peak_factor as f64;
    let writes = per_second(load.writes_per_day);
    let reads = writes * load.reads_per_write as f64;
    let (writes_peak, reads_peak) = (writes * peak, reads * peak);
    println!("writes: {writes:>7.1}/s average {writes_peak:>7.1}/s peak");
    println!("reads:  {reads:>7.1}/s average {reads_peak:>7.1}/s peak");
    let storage = human_bytes(storage_bytes(&load));
    println!("storage after {} days: {storage}", load.days_kept);
}
```

**What happens:**

1. `Load` collects every assumption in one place, with the unit in each name (`bytes_per_record`, `days_kept`). A reviewer can argue with a single number instead of with hidden arithmetic.
2. The assumptions describe a busy URL shortener: 1 million new links per day, each read 100 times, 500 bytes per row, kept for 5 years, 3 copies of the data.
3. `per_second` turns a daily count into a rate. It uses `f64`, because 1,000,000 / 86,400 is not a whole number.
4. `storage_bytes` multiplies everything out in `u64`. With these values the result is about 2.7 × 10¹² bytes, far below the `u64` limit of about 1.8 × 10¹⁹.
5. `human_bytes` divides by 1024 until the number is small, and prints one decimal with the matching binary unit.

```bash
cargo run
```

```text
writes:    11.6/s average    57.9/s peak
reads:   1157.4/s average  5787.0/s peak
storage after 1825 days: 2.5 TiB
```

What the numbers say: writes are tiny (under 60 per second at peak), reads are moderate (under 6,000 per second at peak, which one well-written Rust service can serve), and storage is the largest number. So the design question is not "how do we scale writes?" but "where do 2.5 TiB live, and how do we read them fast?" — a cache for popular codes and one database with enough disk. This is the kind of conclusion that goes into Section 5 of the document, together with the program's output.

> ⚡ **Performance:** An estimate tells you *where to look*, not how fast your code is. After you build, measure (Lesson 23.8) and compare the measurement with the estimate. A big difference means either the estimate or the code has a surprise in it — both are worth finding.

## 5. Failure modes

Every part of a system fails sometimes. The design document says, for each important part, **how** it fails, **how you notice**, **what the user sees**, and **what the system does**. A table keeps this short:

| Component | Failure | How we notice | What the user sees | What we do |
|---|---|---|---|---|
| PostgreSQL | down or unreachable | `/readyz` fails; error metric rises | `503` on create; redirects of cached codes still work | retry with backoff; alert after 1 minute |
| PostgreSQL | slow (p99 > 200 ms) | latency histogram | slow redirects on cache misses | query timeout of 300 ms, then `503` |
| Redis | down | connection error | nothing, except slower redirects | fall back to PostgreSQL, header `x-cache: unavailable` |
| Disk of the database | full | disk usage metric above 90% | creates fail with `503` | alert at 80%; documented clean-up |
| Input | URL of 1 MB, invalid UTF-8 | request validation | `422` with an error message | body limit of 4 KiB before parsing |

Writing this table often changes the design. In the example, the line "Redis down" forced the decision that the cache is optional, which is why the shortener has the `x-cache: unavailable` header. Every row also gives you a test for the test plan: "kill Redis during a load test and check that redirects still answer".

> 🔒 **Security:** Treat an attacker as one more failure mode. Add the rows that STRIDE suggests (Lesson 10.2): who can create links, what stops someone from creating a million links per minute, what happens to a link that points to a malicious site.

## 6. Alternatives and decision records

For each big decision, the document says what else you considered and why you did not choose it. "We use PostgreSQL" is a statement; "We use PostgreSQL rather than an embedded store such as `redb`, because we need two service instances to share the data" is a decision a reviewer can check.

Decisions keep coming during the project, long after the design document is accepted. Record each important one in a short **ADR** in `docs/adr/`, numbered in order: `0001-use-postgresql.md`, `0002-cache-is-optional.md`. An ADR is never edited later; if a decision changes, a new ADR *supersedes* the old one.

```markdown
# ADR 0002 — The cache is optional

Status: accepted · Date: 2026-12-04

## Context
Redis improves redirect latency, but it is one more thing that can fail.
Our requirement R4 says redirects must keep working while Redis is down.

## Decision
Every redirect tries Redis with a 20 ms timeout. On any Redis error the
service reads PostgreSQL directly and sets the header `x-cache: unavailable`.

## Consequences
- Good: a Redis outage only makes redirects slower, never failing.
- Bad: during an outage all reads hit PostgreSQL; at the peak estimate
  (5,800 reads/s) this needs an index-only query of about 1 ms.
- We must test it: `m4_redirect_works_without_redis`.
```

ADRs are short on purpose. Context, decision, consequences: half a page. Together they tell the story of the project, and in Lesson 23.9 they become material for your README and your talk.

## 7. Reviewing a design

A design document is finished when someone other than you has read it and you have answered their questions. If there is nobody to ask, review it yourself after a night of sleep, with these questions:

1. Can a stranger understand the summary without the rest of the document?
2. Does every requirement have a number, a unit and a test in the test plan?
3. Does every component in the diagram appear in the failure-mode table?
4. For each failure: is it detected, and does the user see something sensible?
5. Is there at least one alternative for each big decision, with a reason?
6. Is anything in the milestones not covered by the goals (scope creep)? Is any goal missing from the milestones?
7. Are the open questions really open, and does each have a date or a milestone by which it must be decided?

> 🧪 **Testing:** The test plan is part of the design, not an afterthought. For each requirement and each failure-mode row, name the test that proves it and the milestone in which it is written. If a requirement cannot be tested, rewrite it until it can.

## Common mistakes

**1. A design document that is a code tour.** Ten pages of structs and function names, no reasons. Fix: write decisions and trade-offs; leave the details to the code and its documentation.

**2. Requirements without numbers.** "Fast", "reliable", "scalable". Fix: the table in Section 3 — number, unit, condition, and how it is measured.

**3. Integers that are too small for an estimate.** Storage numbers grow fast. If `Load` used `u32`, the multiplication in `storage_bytes` would not fit. With constant values the compiler even catches it:

```rust
fn main() {
    let writes_per_day: u32 = 1_000_000;
    let days_kept: u32 = 5 * 365;
    let bytes_per_record: u32 = 500;
    let total = writes_per_day * days_kept * bytes_per_record;
    println!("{total} bytes");
}
```

```text
error: this arithmetic operation will overflow
 --> src/main.rs:5:17
  |
5 |     let total = writes_per_day * days_kept * bytes_per_record;
  |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ attempt to compute `1825000000_u32 * 500_u32`, which would overflow
  |
  = note: `#[deny(arithmetic_overflow)]` on by default
```

With values read at run time there is no compiler error; a debug build panics and a release build silently wraps around (Lesson 12.2). Fix: use `u64` for byte counts and rates.

**4. No non-goals.** Then every idea feels like part of the project. Fix: write at least three non-goals, including one that you would *like* to build.

**5. Writing the document and never updating it.** The code changes, the document does not, and the document now lies. Fix: update the document when an ADR changes the design, and keep the `Status` and date current.

**6. Hiding uncertainty.** Pretending every question is answered. Fix: an honest "Risks and open questions" section. Reviewers trust a document more when it admits what is unknown.

## Try it yourself

1. Create `docs/design.md` in your first capstone (or in `code/p23-skeleton` for practice) with the template from Section 2.
2. Fill in sections 1–3: the summary in three sentences, the problem, three goals and three non-goals.
3. Create `code/p23-estimate` from Section 4 and run it. Then replace the numbers in `Load` with numbers for your capstone (for a proxy: requests per second and bytes per request; for a broker: messages per day and bytes per message) and paste the output into section 5 of your document.
4. Write the failure-mode table for the two most important components of your design.
5. Write `docs/adr/0001-<your-first-decision>.md` with Context, Decision and Consequences.

## Exercises

### Exercise 1 — Make requirements checkable (easy)

Rewrite these three requirements for a key-value store so that each can be tested: (a) "Reads are fast." (b) "Data is safe." (c) "It survives failures."

<details>
<summary>Hint</summary>

For each one, add: what is measured, a number with a unit, under which load or condition, and how you would test it.

</details>

<details>
<summary>Answer</summary>

(a) "p99 latency of `GET` for 1 KiB values is below 5 ms at 10,000 QPS on one node, measured with a 60-second load test." (b) "A `PUT` returns success only after the value is `fsync`-ed; after `kill -9` and a restart, every acknowledged value is readable (crash test with 10,000 writes)." (c) "With 3 nodes, the cluster accepts writes while any 1 node is stopped, and no acknowledged write is lost when the leader is killed (simulation test with 1,000 seeds)." Your numbers may differ; what matters is number, unit, condition and test.

</details>

### Exercise 2 — Failure-mode rows (medium)

Your capstone is an HTTP reverse proxy in front of three backend servers. Write two rows of the failure-mode table: one for "a backend crashes" and one for "a backend becomes very slow".

<details>
<summary>Hint</summary>

Use the five columns of Section 5. A crash is noticed quickly (connection refused); slowness is harder to notice and needs a timeout.

</details>

<details>
<summary>Answer</summary>

| Component | Failure | How we notice | What the user sees | What we do |
|---|---|---|---|---|
| Backend | crashes | connect error; active health check fails | at most one failed request per in-flight connection; then nothing | mark it down, send traffic to the others, check it again every 2 s |
| Backend | very slow (p99 > 2 s) | per-backend latency histogram; request timeout | `504` after the timeout instead of waiting forever | timeout of 1 s; stop sending traffic after 5 timeouts in 10 s (outlier detection) |

The second row is the more important one: a slow backend is worse than a dead one, because without a timeout it ties up connections in the proxy.

</details>

### Exercise 3 — Write tests for the estimate (medium)

Add a `#[cfg(test)] mod tests` to `code/p23-estimate/src/main.rs` with three tests: `storage_bytes` counts every replica (choose small numbers you can check by hand), `human_bytes` uses binary units (check 512, 1024, 1536 and 3 GiB), and 1 million per day is about 11.574 per second.

<details>
<summary>Hint</summary>

A test module inside `main.rs` can use the private items with `use super::*;`. Floating-point results are compared with a tolerance: `assert!((rate - 11.574).abs() < 0.001)` (Lesson 05.6).

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn small_load() -> Load {
        Load {
            writes_per_day: 10,
            reads_per_write: 1,
            bytes_per_record: 100,
            days_kept: 2,
            replicas: 3,
            peak_factor: 1,
        }
    }

    #[test]
    fn storage_counts_every_replica() {
        // 10 records/day * 2 days * 100 bytes * 3 copies
        assert_eq!(storage_bytes(&small_load()), 6_000);
    }

    #[test]
    fn human_bytes_uses_binary_units() {
        assert_eq!(human_bytes(512), "512.0 B");
        assert_eq!(human_bytes(1024), "1.0 KiB");
        assert_eq!(human_bytes(1536), "1.5 KiB");
        assert_eq!(human_bytes(3 * 1024 * 1024 * 1024), "3.0 GiB");
    }

    #[test]
    fn one_million_per_day_is_about_twelve_per_second() {
        let rate = per_second(1_000_000);
        assert!((rate - 11.574).abs() < 0.001, "rate was {rate}");
    }
}
```

`cargo test -q` prints one dot per passing test:

```text
running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: small inputs give results you can compute in your head (10 × 2 × 100 × 3 = 6,000), so the test checks the formula, not a number copied from the program's output.

</details>

### Exercise 4 — How many machines? (hard)

Add `fn nodes_needed(total_bytes: u64, bytes_per_node: u64) -> u64` that returns how many machines are needed when each one can hold `bytes_per_node`. It must round **up**: 11 bytes on machines of 5 bytes need 3 machines. Write the test first, then the function, then print the number of 1 TiB machines in `main`.

<details>
<summary>Hint</summary>

Integer division `a / b` rounds down. The integer types have a method `div_ceil` that rounds up. Test the cases "exact" (10 / 5), "with a remainder" (11 / 5) and "nothing to store" (0 / 5).

</details>

<details>
<summary>Answer</summary>

```rust
/// How many machines hold `total_bytes` if each machine can hold `bytes_per_node`.
fn nodes_needed(total_bytes: u64, bytes_per_node: u64) -> u64 {
    total_bytes.div_ceil(bytes_per_node)
}
```

The test, inside `mod tests`:

```rust
#[test]
fn nodes_needed_rounds_up() {
    assert_eq!(nodes_needed(10, 5), 2);
    assert_eq!(nodes_needed(11, 5), 3);
    assert_eq!(nodes_needed(0, 5), 0);
}
```

At the end of `main`:

```rust
let tib = 1024 * 1024 * 1024 * 1024;
let nodes = nodes_needed(storage_bytes(&load), tib);
println!("machines with 1 TiB of disk each: {nodes}");
```

```text
writes:    11.6/s average    57.9/s peak
reads:   1157.4/s average  5787.0/s peak
storage after 1825 days: 2.5 TiB
machines with 1 TiB of disk each: 3
```

Why it works: `div_ceil` computes ⌈a / b⌉ without the overflow risk of the old trick `(a + b - 1) / b`. 2.5 TiB need 3 machines, because 2 would hold only 2 TiB. Note that `bytes_per_node` must not be 0: `div_ceil` panics on division by zero, so in a real tool you would check it.

</details>

## Quiz

1. Why is a design document written before the code, not after?

<details>
<summary>Answer</summary>

Because changing a design on paper is cheap and changing it in code is expensive. Writing it first finds gaps (missing failure handling, impossible numbers) while they are still cheap to fix, and lets others review the design quickly.

</details>

2. What is a non-goal, and why is it useful?

<details>
<summary>Answer</summary>

Something you decide, in writing, not to build, although someone might expect it. It protects the scope: you make the decision once instead of being tempted every week.

</details>

3. Turn "the service is fast" into a checkable requirement.

<details>
<summary>Answer</summary>

For example: "p99 latency of `GET /{code}` is below 50 ms at 1,000 QPS on one node, measured with a 60-second load test." It has a measure (p99 latency), a number with a unit, a condition (load, hardware) and a method.

</details>

4. About how many events per second are 1 million events per day? And why plan for more than that?

<details>
<summary>Answer</summary>

About 12 per second (1,000,000 / 86,400 ≈ 11.6). Traffic is not flat, so you plan for a peak several times the average (for example 5×).

</details>

5. Which five questions does a row of the failure-mode table answer?

<details>
<summary>Answer</summary>

Which component, what failure, how we notice it, what the user sees, and what the system does about it.

</details>

6. What is an ADR, and what do you do when a decision in an old ADR changes?

<details>
<summary>Answer</summary>

An Architecture Decision Record: a short file with the context, the decision and its consequences. You do not edit the old one; you write a new ADR that supersedes it and mark the old one as superseded.

</details>

7. Why is the estimate written as a small tested program instead of numbers typed into the document?

<details>
<summary>Answer</summary>

The assumptions are named in one place, the arithmetic is checked by tests (no silent mistakes such as overflow or a wrong unit), and you can rerun it in seconds when an assumption changes.

</details>

## Summary

- A design document describes the problem, the chosen design, the alternatives and the risks. It is short, specific and honest, and it is written before the code.
- Use the template: summary, context, goals and non-goals, requirements, estimates, design, failure modes, alternatives, test plan, observability, security, milestones, risks and open questions.
- Requirements need a number, a unit, a condition and a test; non-goals protect the scope.
- Back-of-the-envelope estimates (1 million per day ≈ 12 per second, a peak factor, rough operation costs) show where the real difficulty is; a small tested program keeps them honest.
- The failure-mode table lists, per component, failure, detection, user impact and reaction, and gives the test plan its failure tests.
- Decisions during the project go into short, numbered ADRs that are never edited, only superseded.

## Sources and further reading

- [Design Docs at Google](https://www.industrialempathy.com/posts/design-docs-at-google/) — how one large company writes and reviews design documents, and when not to write one.
- [Architectural Decision Records](https://adr.github.io/) — templates and tools for ADRs.
- [Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) — the short article by Michael Nygard that introduced the ADR format.
- [The Rust RFC Book](https://rust-lang.github.io/rfcs/) and the [RFC template](https://github.com/rust-lang/rfcs/blob/master/0000-template.md) — how the Rust project itself proposes and reviews designs.
- [Site Reliability Engineering — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) — how to choose measurable targets for a service.
- [Rust standard library — `u64`](https://doc.rust-lang.org/std/primitive.u64.html) — `div_ceil`, checked and saturating arithmetic for estimates.

---

[🏠 Start](../START-HERE.md) · [← 23.1 How capstone projects work](01-how-capstone-projects-work.md) · [📋 Phase 23](README.md) · [23.3 Capstone option 1: a distributed key-value store →](03-capstone-distributed-key-value-store.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
