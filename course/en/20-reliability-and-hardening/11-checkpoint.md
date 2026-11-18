<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.10 Project: harden a service](10-project-harden-a-service.md) · [📋 Phase 20](README.md) · [Phase 21 — Cloud-native infrastructure →](../21-cloud-native-infrastructure/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/11-checkpoint.md) <!-- nav -->

# Lesson 20.11 — Checkpoint

> **Phase 20 — Reliability and hardening** · Lesson 11 of 11 · ⏱ about 105 minutes

## What you will learn

This lesson teaches nothing new. It checks that you can use what Phase 20 taught, and it shows you where to go back if something is still weak. You will:

- Answer 16 questions about reliability, panics, fuzzing, Miri, sanitizers, loom, Kani, chaos testing, the supply chain, overload and production readiness.
- Solve 4 practical tasks that mix several lessons. Three of them have tests, one has a fuzz target.
- Find the lessons you should repeat, with direct links.

## Before you start

- You finished [Lesson 20.10 — Project: harden a service](10-project-harden-a-service.md). Your `p20-shortener` passes `./readiness-check.sh`, and `READINESS.md` is committed.
- Do the quiz **closed book**: no lessons, no compiler, no search. Write your answer down first, then open the hidden answer.
- For the practical tasks you may use the compiler and the lessons, but try first without hints.
- Create one package for the three coding tasks:

```bash
cd ~/projects/rust
cargo new --lib code/p20-checkpoint
cd code/p20-checkpoint
cargo add tokio@1 --features sync
```

Task 1 goes into `src/header.rs`, Task 2 into `src/limiter.rs`, Task 3 into `src/budget.rs`, and `src/lib.rs` contains:

```rust
pub mod budget;
pub mod header;
pub mod limiter;
```

- Work on a branch: `git switch -c lesson/20-11-checkpoint`.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A pause at the end of a phase to test yourself before you go on. |
| **closed book** | Answering from memory only, without looking anything up. |
| **retry budget** | A rule on the calling side: retries may be at most a fixed share of all requests, for example 10 %. |
| **admission control** | Deciding at the door whether a new request may come in, instead of letting it wait inside. |

## Self-test quiz

Mark one point for each answer that is **correct and complete**. 13 or more points: go on. Fewer: repeat the lessons from the table at the end.

### Reliability and panics (Lessons 20.1 and 20.2)

1. What is the difference between a fault and a failure? Give one example of each for a web service.

<details>
<summary>Answer</summary>

A **fault** is a cause: a bug, a full disk, a slow database, a lost packet. A **failure** is what users see: an error page, a wrong answer, no answer. Example: the database becomes slow (fault), and the checkout page times out (failure). Reliability work keeps faults from becoming failures. (Lesson 20.1.)

</details>

2. A service fails once every 999 hours and needs 1 hour to recover. What is its availability? Name the two ways to improve it, and the one that is often cheaper.

<details>
<summary>Answer</summary>

`MTBF / (MTBF + MTTR)` = 999 / 1000 = **99.9 %** (MTBF: mean time between failures, MTTR: mean time to recovery). Fail less often (a bigger MTBF) or recover faster (a smaller MTTR). Faster recovery is often cheaper: automatic restarts, health checks and quick rollbacks. (Lesson 20.1.)

</details>

3. A Tokio task panics while it holds a `std::sync::Mutex`. What does the code that awaits the task see, and what do later `lock()` calls return?

<details>
<summary>Answer</summary>

Awaiting the `JoinHandle` gives `Err(JoinError)` with `is_panic() == true`; the other tasks keep running. The mutex is **poisoned**: `lock()` returns `Err(PoisonError)`, and `into_inner()` still gives the data, which may be half-changed. (Lesson 20.2.)

</details>

4. When would you choose `panic = "abort"`, and what must then exist outside the program?

<details>
<summary>Answer</summary>

When a bug means the state in memory cannot be trusted any more, for example in a storage engine in the middle of a write (*crash-only* design). The first panic ends the whole process at once (exit code 134), without `Drop`, so a **supervisor** (systemd, Kubernetes) must restart it, and the program must rebuild its state from durable storage. (Lesson 20.2.)

</details>

### Finding bugs before users do (Lessons 20.3, 20.4 and 20.6)

5. What does a stateful, model-based fuzz target compare, and which kind of bug can it find that a "does it crash?" target cannot?

<details>
<summary>Answer</summary>

It runs a generated sequence of operations on the real code **and** on a simple model (for example a `Vec` or a `HashMap`), and fails at the first difference. It finds **wrong answers**, not only crashes. (Lesson 20.3.)

</details>

6. A fuzz target with a derived `Arbitrary` keeps failing with OOM (out of memory) in `vec![0; capacity]`. What is wrong, and how do you fix it?

<details>
<summary>Answer</summary>

The fuzzer picks huge capacities, so the target, not the code under test, runs out of memory. Shape the input: a hand-written `impl Arbitrary` (or `#[arbitrary(with = …)]`) with `u.int_in_range(0..=7)` keeps sizes small. (Lesson 20.3.)

</details>

7. Which tool answers each question: (a) "Did my `unsafe` code break a rule of Rust on the paths the tests ran?" (b) "Do two threads of my compiled program race on memory?" (c) "Can any interleaving of these three threads break my lock-free stack?" (d) "Did C code that I call write outside its buffer?"

<details>
<summary>Answer</summary>

(a) Miri. (b) ThreadSanitizer. (c) loom. (d) AddressSanitizer. (Lesson 20.4.)

</details>

8. Kani proved a function for all inputs. What does "bounded" mean for that proof, and why is a strong `kani::assume` dangerous?

<details>
<summary>Answer</summary>

Loops are unwound only a fixed number of times (`#[kani::unwind(n)]`) and arrays have a fixed size, so the proof holds only up to those bounds. `kani::assume` removes inputs from the check: an assumption that is stronger than the function's real contract can hide exactly the input that breaks it. (Lesson 20.6.)

</details>

### Faults and chaos (Lesson 20.5)

9. Name the steps of a chaos experiment, in order.

<details>
<summary>Answer</summary>

Define the **steady state** (normal, measurable behaviour), write a **hypothesis**, **inject** the fault, **observe**, and check that the system **recovers** to the steady state after the fault is removed. Start small and keep a stop button. (Lesson 20.5.)

</details>

10. What does `"toxicity": 0.4` mean in toxiproxy, and why does a chaos script use `trap cleanup EXIT`?

<details>
<summary>Answer</summary>

Only about 40 % of new connections get the toxic, like a fault that hits some requests. `trap … EXIT` runs the clean-up function however the script ends (success, error, Ctrl+C), so no fault is left behind. (Lesson 20.5.)

</details>

### The supply chain (Lesson 20.7)

11. Which tool or practice answers: (a) "Has a person read this dependency?" (b) "Which crate versions are inside the binary that runs on the server?" (c) "Give the customer a standard list of components." (d) "Was this binary really built from this commit?"

<details>
<summary>Answer</summary>

(a) `cargo vet`. (b) `cargo auditable` with `cargo audit bin`. (c) An SBOM (Software Bill of Materials), for example from `cargo cyclonedx`. (d) A reproducible build that a second machine rebuilds and compares, plus signed provenance. (Lesson 20.7.)

</details>

12. Why do you sign `SHA256SUMS` and not only publish it?

<details>
<summary>Answer</summary>

Whoever can replace the binary on the download page can also replace the checksum file. Only the owner of the private key can make a valid signature, so a changed checksum file is detected. Users check the signature first, then the checksums. (Lesson 20.7.)

</details>

### Overload and readiness (Lessons 20.8 to 20.10)

13. A service gets 400 requests per second, and each takes 250 ms. How many are in flight? With a concurrency limit of 50, what is the capacity?

<details>
<summary>Answer</summary>

400 × 0.25 = **100** in flight (Little's law). Capacity = 50 ÷ 0.25 s = **200 requests per second**, so about 200 per second must wait or be shed. (Lesson 20.8.)

</details>

14. When does a service answer `429`, when `503`, and which header belongs to both?

<details>
<summary>Answer</summary>

`429 Too Many Requests` when one client is over its own rate limit; `503 Service Unavailable` when the service is overloaded as a whole or a dependency is down. Both carry `Retry-After`. (Lesson 20.8.)

</details>

15. What happens, step by step, when a well-built service gets SIGTERM (the stop signal that a service manager sends)?

<details>
<summary>Answer</summary>

Readiness becomes "not ready" (`503`) while liveness stays `200`; the service waits a short drain time so the load balancer stops sending requests; then it stops accepting new connections, finishes the open requests within a hard time limit, and exits with code `0`. (Lesson 20.9.)

</details>

16. In axum, why does `app` end with `.with_state(...)` after the tower layers, even when the state is `()`?

<details>
<summary>Answer</summary>

Otherwise axum builds the handler's layers again for every request, so a `ConcurrencyLimitLayer` gives every request its own fresh limit and limits nothing. `.with_state(...)` turns the handlers into routes once, so all requests share one layer stack. A test with two concurrent requests and a limit of one catches the mistake. (Lessons 20.8 and 20.10.)

</details>

## Practical tasks

Solve each task in the checkpoint package. Run `cargo test` (and the fuzzer for Task 1) before you open the answer.

### Task 1 — A header parser that never panics (Lessons 20.2 and 20.3)

Write `split_header(line: &str) -> Option<(&str, &str)>` in `src/header.rs`. It splits `Name: value` into the name and the trimmed value. Rules: the line is at most `MAX_LINE = 1024` bytes; the name is 1 to 64 ASCII (American Standard Code for Information Interchange) letters, digits or `-`; anything else gives `None`. It must **never panic**, for any text. Write unit tests, then a fuzz target `header` (`cargo fuzz init -t header`) with two properties: it never panics, and a parsed line written again as `format!("{name}: {value}")` parses to the same two parts. Run it for 30 seconds.

<details>
<summary>Hint</summary>

`str::split_once(':')` returns `None` when there is no colon and never panics, unlike slicing with an index you computed yourself. Check the name with `bytes().all(…)`. For the second property, think about a line that is exactly 1,024 bytes long and has no space after the colon.

</details>

<details>
<summary>Answer</summary>

```rust
//! Task 1: a header line parser that never panics.

/// The longest line we accept, in bytes.
pub const MAX_LINE: usize = 1024;

/// Splits `Name: value` into the name and the trimmed value.
/// The name is 1 to 64 ASCII letters, digits or `-`.
pub fn split_header(line: &str) -> Option<(&str, &str)> {
    if line.len() > MAX_LINE {
        return None;
    }
    let (name, value) = line.split_once(':')?;
    let name_ok = (1..=64).contains(&name.len())
        && name.bytes().all(|b| b.is_ascii_alphanumeric() || b == b'-');
    if !name_ok {
        return None;
    }
    Some((name, value.trim()))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn splits_a_normal_line() {
        assert_eq!(split_header("Retry-After: 1"), Some(("Retry-After", "1")));
        assert_eq!(split_header("x:"), Some(("x", "")));
    }

    #[test]
    fn rejects_bad_lines() {
        assert_eq!(split_header("no colon"), None);
        assert_eq!(split_header(": empty name"), None);
        assert_eq!(split_header("bad name: x"), None);
        assert_eq!(split_header("é: x"), None);
        assert_eq!(split_header(&"a".repeat(2000)), None);
    }
}

#[cfg(test)]
mod edge_tests {
    use super::*;

    #[test]
    fn writing_a_full_line_again_can_make_it_too_long() {
        let line = format!("x:{}", "a".repeat(MAX_LINE - 2));
        let (name, value) = split_header(&line).unwrap();
        assert_eq!(split_header(&format!("{name}: {value}")), None);
    }
}
```

The fuzz target `fuzz/fuzz_targets/header.rs`:

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;
use p20_checkpoint::header::{MAX_LINE, split_header};

fuzz_target!(|line: &str| {
    // Property 1: never panics. Property 2: a parsed line, written again, parses to the same parts
    // (only if the new line still fits: "x:a" becomes the longer "x: a").
    if let Some((name, value)) = split_header(line) {
        let again = format!("{name}: {value}");
        if again.len() <= MAX_LINE {
            assert_eq!(split_header(&again), Some((name, value)));
        }
    }
});
```

```bash
cargo +nightly fuzz run header -- -max_total_time=30
```

```text
Done 14460559 runs in 31 second(s)
```

Why it works: `split_once` and `bytes()` cannot panic, and every length is checked before it is used. The second property is only true while the rewritten line still fits: `"x:" + 1,022 × "a"` is exactly 1,024 bytes, but `"x: " + …` is 1,025 and is refused. The target therefore checks the round trip only when `again.len() <= MAX_LINE`. A wrong property makes the fuzzer report a "bug" that is really a mistake in the test; the fuzzer may also never reach such a long, special input in 30 seconds, which is why the answer keeps a unit test for that edge case too.

</details>

### Task 2 — Admission control that sheds (Lesson 20.8)

Write a `Limiter` in `src/limiter.rs` with `new(max)`, `try_enter(&self) -> Option<…>` and `in_flight(&self) -> usize`. `try_enter` must **never wait**: it returns a permit if a slot is free and `None` at once if not. Dropping the permit frees the slot. Clones of a `Limiter` share the same slots. Write tests for: a third request is refused at limit 2; after one permit is dropped, a request gets in again; clones share the slots.

<details>
<summary>Hint</summary>

`tokio::sync::Semaphore` counts free slots. `Arc<Semaphore>::try_acquire_owned()` returns `Ok(OwnedSemaphorePermit)` or an error immediately; the owned permit gives its slot back in `Drop`. `available_permits()` tells you how many slots are free.

</details>

<details>
<summary>Answer</summary>

```rust
//! Task 2: admission control that refuses instead of waiting.

use std::sync::Arc;

use tokio::sync::{OwnedSemaphorePermit, Semaphore};

#[derive(Clone)]
pub struct Limiter {
    slots: Arc<Semaphore>,
    max: usize,
}

impl Limiter {
    pub fn new(max: usize) -> Limiter {
        Limiter {
            slots: Arc::new(Semaphore::new(max)),
            max,
        }
    }

    /// A permit if a slot is free, `None` at once if not. Dropping the permit frees the slot.
    pub fn try_enter(&self) -> Option<OwnedSemaphorePermit> {
        Arc::clone(&self.slots).try_acquire_owned().ok()
    }

    /// How many slots are taken right now.
    pub fn in_flight(&self) -> usize {
        self.max - self.slots.available_permits()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn refuses_when_full_and_recovers() {
        let limiter = Limiter::new(2);
        let first = limiter.try_enter();
        let second = limiter.try_enter();
        assert!(first.is_some() && second.is_some());
        assert!(
            limiter.try_enter().is_none(),
            "a third request must be shed"
        );
        assert_eq!(limiter.in_flight(), 2);

        drop(first);
        assert_eq!(limiter.in_flight(), 1);
        assert!(limiter.try_enter().is_some());
    }

    #[test]
    fn clones_share_the_slots() {
        let limiter = Limiter::new(1);
        let copy = limiter.clone();
        let _permit = limiter.try_enter().unwrap();
        assert!(copy.try_enter().is_none());
    }
}
```

Why it works: the semaphore is the counter of free slots, and the permit is a guard object, like a `MutexGuard`: holding it means "I am inside", dropping it means "I left", even if the request panics or is cancelled. Because the semaphore sits in an `Arc`, clones share it; without the `Arc`, every clone would have its own slots and the limit would not work, which is the same mistake as building a layer per request.

</details>

### Task 3 — A retry budget (Lessons 20.8 and 18.3)

Write a `RetryBudget` in `src/budget.rs`: `new(ratio)`, `record_request(&mut self)` for every first attempt, and `try_retry(&mut self) -> bool`, which allows a retry only while the retries stay at most `ratio` × the requests (rounded down), and counts it. Tests: with `0.1` and 100 requests, exactly 10 of 20 retry attempts are allowed; with no requests, no retry is allowed.

<details>
<summary>Hint</summary>

Keep two counters. `(requests as f64 * ratio).floor() as u64` is the number of allowed retries.

</details>

<details>
<summary>Answer</summary>

```rust
//! Task 3: a retry budget.

/// Allows retries only while they are at most `ratio` of all requests.
pub struct RetryBudget {
    ratio: f64,
    requests: u64,
    retries: u64,
}

impl RetryBudget {
    pub fn new(ratio: f64) -> RetryBudget {
        RetryBudget {
            ratio,
            requests: 0,
            retries: 0,
        }
    }

    /// Call once for every first attempt.
    pub fn record_request(&mut self) {
        self.requests += 1;
    }

    /// Asks for one retry. `true` means: retry, and it is counted.
    pub fn try_retry(&mut self) -> bool {
        let allowed = (self.requests as f64 * self.ratio).floor() as u64;
        if self.retries < allowed {
            self.retries += 1;
            true
        } else {
            false
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ten_percent_of_one_hundred_is_ten_retries() {
        let mut budget = RetryBudget::new(0.1);
        for _ in 0..100 {
            budget.record_request();
        }
        let granted = (0..20).filter(|_| budget.try_retry()).count();
        assert_eq!(granted, 10);
    }

    #[test]
    fn no_requests_no_retries() {
        let mut budget = RetryBudget::new(0.1);
        assert!(!budget.try_retry());
    }
}
```

Why it works: when only a few requests fail (normal, random faults), the budget is large enough for all their retries. When many requests fail at once (the other service is overloaded), the budget runs out quickly, and the client stops adding load: a retry storm cannot start. Real budgets also forget old requests (for example, they count only the last ten seconds); that is a good extension.

</details>

### Task 4 — Review your hardened service (no new code; Lessons 20.5, 20.9 and 20.10)

Open `READINESS.md` of `p20-shortener` and answer in writing:

1. Which three items are still "missing", and in which lesson of Phase 21 do you expect to close each?
2. Pick one failure mode from Lesson 20.1 (crash, hang, slow answer, wrong answer, overload, dependency failure) and list the evidence in your project that covers it.
3. Write the steady state and the hypothesis for one **new** chaos experiment on the shortener that `chaos.sh` does not cover yet.

<details>
<summary>Answer (an example)</summary>

1. A tested rollback (Lesson 21.7), dashboards with the SLO, the Service Level Objective (Lessons 21.9 and 21.11), and a runbook with an on-call owner (Lesson 21.12).
2. **Dependency failure:** the `BrokenStore` and `SlowStore` tests, the "Redis down" and "Redis 1 s slow" lines of `chaos.sh`, and `/readyz` returning `503` when `PING` fails.
3. **Steady state:** `GET /1` answers `307` in under 10 ms, `/readyz` answers `200`. **Hypothesis:** with a `timeout` toxic (Redis accepts the connection but never answers), `GET /1` answers `503` within 310 ms, `/readyz` answers `503` within 210 ms, and both recover within 3 s after the toxic is removed.

Why it works: a review is not a test run. It asks which evidence exists, which is missing, and which faults have not been tried yet. Your answers will differ; what matters is that each one points to something you can show.

</details>

## If you got something wrong

| Topic | Go back to |
|---|---|
| Faults, failures, nines, MTBF and MTTR, chains and copies | [20.1 What reliability means](01-what-reliability-means.md) |
| Errors vs bugs, panics in tasks, poisoning, panic hook, unwind vs abort | [20.2 Errors and panics in production](02-errors-and-panics-in-production.md) |
| Model-based fuzzing, shaping `Arbitrary`, corpus, continuous fuzzing | [20.3 Fuzzing in depth](03-fuzzing-in-depth.md) |
| Miri, AddressSanitizer, ThreadSanitizer, loom in CI (Continuous Integration) | [20.4 Miri, sanitizers and loom in CI](04-miri-sanitizers-and-loom-in-ci.md) |
| Fault injection, toxiproxy, chaos experiments | [20.5 Fault injection and chaos testing](05-fault-injection-and-chaos-testing.md) |
| Proof harnesses, assumptions, bounds | [20.6 Formal verification basics with Kani](06-formal-verification-with-kani.md) |
| `cargo vet`, `cargo auditable`, SBOM, reproducible builds, signatures | [20.7 Supply-chain security in depth](07-supply-chain-security-in-depth.md) |
| Little's law, bounded queues, load shedding, rate limits, retry storms | [20.8 Overload, limits and backpressure](08-overload-limits-and-backpressure.md) |
| The checklist, fail-fast configuration, liveness, readiness, draining | [20.9 The production-readiness checklist](09-production-readiness-checklist.md) |
| Putting it all together on a real service | [20.10 Project: harden a service](10-project-harden-a-service.md) |

## What's next

You finished Phase 20. You can now measure and reason about reliability, keep panics contained and visible, find bugs with fuzzers, Miri, sanitizers, loom and Kani, test failure on purpose, secure the supply chain, protect a service from overload, and review it with a checklist. Before you go on:

1. Merge your checkpoint branch (`git switch main && git merge lesson/20-11-checkpoint`).
2. Remove what you no longer need: `cargo clean` in the phase's packages, and any `course-…` containers and networks (`docker ps -a --filter name=course-`).
3. Keep `READINESS.md` of the shortener: Phase 21 fills in many of its "missing" items.

In **[Phase 21 — Cloud-native infrastructure](../21-cloud-native-infrastructure/README.md)** you will run your services on Kubernetes: the readiness and liveness endpoints become probes, SIGTERM and the drain time become the pod's termination grace period, the release workflow becomes a CI/CD (continuous integration and delivery) pipeline, and the SLO and the alerts of the checklist become real dashboards.

## Sources and further reading

- [Google SRE book](https://sre.google/sre-book/table-of-contents/) (SRE: Site Reliability Engineering) — the chapters on overload, cascading failures and launch checklists behind Lessons 20.8 and 20.9.
- [Rust Fuzz Book](https://rust-fuzz.github.io/book/) — `cargo fuzz` and structure-aware fuzzing.
- [Miri on GitHub](https://github.com/rust-lang/miri) — what Miri checks.
- [Kani Rust Verifier](https://model-checking.github.io/kani/) — the documentation of Kani.
- [toxiproxy on GitHub](https://github.com/Shopify/toxiproxy) — all toxics.
- [cargo-vet book](https://mozilla.github.io/cargo-vet/) — recording audits.

---

[🏠 Start](../START-HERE.md) · [← 20.10 Project: harden a service](10-project-harden-a-service.md) · [📋 Phase 20](README.md) · [Phase 21 — Cloud-native infrastructure →](../21-cloud-native-infrastructure/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
