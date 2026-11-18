<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 20 — Reliability and hardening](README.md) · [📋 Phase 20](README.md) · [20.2 Errors and panics in production →](02-errors-and-panics-in-production.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/01-what-reliability-means.md) <!-- nav -->

# Lesson 20.1 — What reliability means

> **Phase 20 — Reliability and hardening** · Lesson 1 of 11 · ⏱ about 120 minutes

## What you will learn

- What *reliability*, *availability* and *hardening* mean, and how a small fault becomes a failure that users see.
- The common ways in which programs and services fail, and why the silent ones are the most dangerous.
- How to measure availability with "nines", MTBF and MTTR, and how to compute the availability of a chain of parts and of redundant copies.
- How the layers *prevent, detect, contain, recover* organize every technique of this phase.

## Before you start

- You finished Phase 19 — Performance and profiling. In Phase 16 you built backend services, and in Phase 18 ([Distributed systems](../18-distributed-systems/README.md)) you saw that networks and machines fail.
- You know integer overflow from [Lesson 02.2](../02-programming-basics/02-data-types.md) and tests with `cargo test` from [Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md).
- **How this phase organizes code.** Every lesson creates its own package `~/projects/rust/code/p20-<topic>`. Small programs go into `src/bin/<name>.rs` and run with `cargo run --bin <name>`; shared code goes into `src/lib.rs`. Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p20-reliability
cd code/p20-reliability
mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **reliability** | The ability of a system to keep doing the right job, also when something goes wrong, for a long time. |
| **availability** | The share of time in which a system answers correctly. 99.9 % means it is unusable for 0.1 % of the time. |
| **fault** | Something that is wrong or goes wrong: a bug in the code, a full disk, a lost network packet, a slow database. |
| **failure** | The moment when users see wrong behaviour: an error page, a wrong number, no answer at all. |
| **fault tolerance** | Designing a system so that faults do not turn into failures. |
| **failure mode** | One typical way of failing: crash, hang, slow answer, wrong answer, data loss. |
| **silent failure** | A failure that nobody notices at once, because the program does not crash or report it: it just gives a wrong result. |
| **nines** | A short way to say availability: "three nines" is 99.9 %, "four nines" is 99.99 %. |
| **MTBF** (Mean Time Between Failures) | The average time a system works between two failures. |
| **MTTR** (Mean Time To Recovery) | The average time from a failure until the system works again. (Some teams read the R as "repair".) |
| **redundancy** | Having more than one copy of a part, so that the system keeps working when one copy fails. |
| **single point of failure** | A part without a copy: when it fails, the whole system fails. |
| **blast radius** | How much of the system, and how many users, one failure affects. |
| **hardening** | Changing a program or system so that it has fewer ways to fail or to be attacked: limits, checks, safer settings. |
| **SLO** (Service Level Objective) | A reliability goal that a team promises itself, for example "99.9 % of requests succeed each month". Lesson 21.11 teaches SLOs in depth. |

## 1. Fault, error, failure

Imagine a café. The coffee machine has a small leak (a **fault**). Water collects under the machine, but the coffee is still fine. One day the leak reaches a cable and the machine stops in the middle of the morning rush. Now the customers get no coffee: that is a **failure**.

A program works the same way. A fault is the cause: a bug, a full disk, a timeout, a server that restarts. A failure is what the *user* sees. Between the two there is often a long, quiet path: the wrong value is stored, passed on, used again, and only much later someone sees a wrong number on a page.

```text
fault ──► wrong internal state ──► failure the user sees
(a bug,     (a wrong value,          (error page, wrong total,
 a full      a stuck thread,          no answer)
 disk)       a lost message)
```

The key idea of this phase: **faults will happen. You cannot remove all of them.** Disks fill up, networks lose packets, people deploy bugs. *Reliability* is the art of keeping faults from becoming failures, and of making the failures that do happen small, visible and short. A system that does this is **fault tolerant**.

> 💡 **Tip:** When you read about an outage, ask two questions: "what was the fault?" and "why did it become a failure?". The second question usually teaches more.

## 2. How programs fail

There are only a handful of typical **failure modes**. Learn to recognize them, because each one needs a different defence.

| Failure mode | Example | How people notice |
|---|---|---|
| **Crash** | A panic on an unexpected input ends the process. | Loudly: the process exits, a supervisor restarts it, an alert fires. |
| **Hang** | A request waits forever for a database that never answers. | Slowly: requests pile up, memory grows. |
| **Slow answer** | One disk is dying, every tenth request takes 5 seconds. | Hard to see: averages look fine, some users suffer. |
| **Wrong answer** | A counter overflows and shows a wrong total. | Often never, until a customer complains. |
| **Overload** | Ten times the normal traffic arrives at once. | Everything gets slow, then everything fails. |
| **Data loss or corruption** | A crash in the middle of a write leaves a broken file. | Days later, when someone reads the data. |
| **Dependency failure** | The payment service you call is down. | Your service fails too, unless you planned for it. |

Notice the last column. A crash is unpleasant, but it is *honest*: everybody sees it, and it can be restarted. The most dangerous failures are **silent failures**: wrong answers and corrupted data. They spread before anyone notices.

Here is a silent failure you can produce yourself. Create `src/bin/silent.rs`:

```rust
use std::hint::black_box;

fn main() {
    // Total bytes sent today, stored in a u32 (at most about 4.29 billion).
    let mut total_bytes: u32 = 4_000_000_000;
    let next_response: u32 = black_box(500_000_000);
    total_bytes += next_response;
    println!("total bytes sent: {total_bytes}");
}
```

**What happens:**

1. A `u32` holds numbers up to 4 294 967 295. We start near that limit and add 500 million.
2. `black_box` (from `std::hint`) hides the value from the optimizer. Without it, the compiler would see the overflow while compiling and refuse to build the program. Real overflows come from data at run time, and `black_box` imitates that.
3. In a debug build (`cargo run`), Rust checks every addition and panics on overflow (Lesson 02.2). In a release build (`cargo run --release`), the check is off by default for speed, and the number silently *wraps around* past zero.

```bash
cargo run -q --bin silent
cargo run -q --release --bin silent
```

```text
thread 'main' (5811233) panicked at src/bin/silent.rs:7:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
total bytes sent: 205032704
```

The debug build crashed loudly (exit code 101). The release build, the one that runs in production, printed a total that is twenty times too small and exited with code 0. No log, no error, no alert.

This is where **hardening** begins: you can turn the checks on in release builds too. Add to `Cargo.toml`:

```toml
[profile.release]
overflow-checks = true
```

```bash
cargo run -q --release --bin silent
```

```text
thread 'main' (5812076) panicked at src/bin/silent.rs:7:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

You traded a silent wrong answer for a loud crash. That is usually a good trade, and the next lesson shows how to make such a crash small and visible instead of fatal. Even better is to make the overflow impossible: use a `u64` for the total, or `checked_add`, which returns `None` instead of wrapping.

> ⚡ **Performance:** `overflow-checks = true` adds one comparison per arithmetic operation. In most services the cost is too small to measure. In a hot numeric loop, measure it (Phase 19) before you decide.

## 3. Measuring availability: nines, MTBF and MTTR

Teams describe availability with **nines**. The numbers become clearer when you turn them into minutes. Put this in `src/lib.rs` (replace the generated content):

```rust
//! Small helpers for reasoning about availability.

/// Minutes in a year of 365 days.
pub const MINUTES_PER_YEAR: f64 = 365.0 * 24.0 * 60.0;

/// How many minutes per year a service may be down at the given availability.
/// `availability` is a fraction: 0.999 means 99.9 %.
pub fn downtime_minutes_per_year(availability: f64) -> f64 {
    (1.0 - availability) * MINUTES_PER_YEAR
}

/// Availability of a chain where every part is needed (A calls B calls C).
pub fn serial(parts: &[f64]) -> f64 {
    parts.iter().product()
}

/// Availability of copies where any one copy is enough (two replicas).
pub fn redundant(copies: &[f64]) -> f64 {
    let all_down: f64 = copies.iter().map(|a| 1.0 - a).product();
    1.0 - all_down
}
```

and this in `src/bin/nines.rs`:

```rust
use p20_reliability::downtime_minutes_per_year;

fn main() {
    let levels = [0.99, 0.999, 0.9999, 0.99999];
    println!("{:>12} {:>14} {:>14}", "availability", "down/year", "down/month");
    for a in levels {
        let year = downtime_minutes_per_year(a);
        let month = year / 12.0;
        println!("{:>11.3}% {:>10.1} min {:>10.1} min", a * 100.0, year, month);
    }
}
```

**What happens:**

1. `downtime_minutes_per_year` takes the share of time that is *not* available, `1.0 - availability`, and multiplies it by the minutes in a year.
2. The binary prints four levels. `{:>12}` right-aligns a value in 12 characters; `{:.3}` prints three digits after the point.

```bash
cargo run -q --bin nines
```

```text
availability      down/year     down/month
     99.000%     5256.0 min      438.0 min
     99.900%      525.6 min       43.8 min
     99.990%       52.6 min        4.4 min
     99.999%        5.3 min        0.4 min
```

Each extra nine divides the allowed downtime by ten. Three nines allow about 44 minutes per month: one bad deployment. Five nines allow 26 seconds per month: no human can even open a laptop in that time, so such systems must repair themselves automatically. Every nine costs much more work than the one before, which is why a team chooses a *goal* that fits its users (an **SLO**, Lesson 21.11) instead of "as many nines as possible".

Two more numbers describe *how* a system reaches its availability:

- **MTBF** (Mean Time Between Failures): how long it works between failures.
- **MTTR** (Mean Time To Recovery): how long a failure lasts.

```text
availability = MTBF / (MTBF + MTTR)
```

A service that fails once every 999 hours and needs 1 hour to recover has 999 / 1000 = 99.9 %. You can improve it in two ways: fail less often (bigger MTBF) or recover faster (smaller MTTR). The second is often much cheaper: automatic restarts, health checks and quick rollbacks shrink MTTR from an hour to a minute.

## 4. Chains and copies

A real service depends on other parts. If a request needs the load balancer (LB in the picture) **and** the application **and** the database (DB), all three must work. The availabilities multiply, and the result is *worse* than the weakest part. If any one of several copies is enough, the chance that **all** copies are down at once is small, and the result is *better* than one copy.

```text
serial (all needed):      LB ──► app ──► DB        A = a1 · a2 · a3

redundant (one enough):        ┌─► app copy 1 ─┐
                          LB ──┤               ├──► DB
                               └─► app copy 2 ─┘   A_app = 1 − (1 − a)(1 − a)
```

`serial` and `redundant` in `src/lib.rs` compute exactly this. Create `src/bin/chain.rs`:

```rust
use p20_reliability::{redundant, serial};

fn main() {
    // A web service that needs a load balancer, the app and a database.
    let one_path = serial(&[0.9995, 0.999, 0.999]);
    println!("one path:              {:.4} %", one_path * 100.0);

    // The same, but the app runs as two copies: either copy can answer.
    let app_pair = redundant(&[0.999, 0.999]);
    let with_pair = serial(&[0.9995, app_pair, 0.999]);
    println!("two copies of the app: {:.4} %", with_pair * 100.0);
}
```

```bash
cargo run -q --bin chain
```

```text
one path:              99.7502 %
two copies of the app: 99.8500 %
```

**What happens:**

1. Three parts with 99.95 %, 99.9 % and 99.9 % give only 99.75 % together. Every dependency you add costs availability.
2. Two copies of the application raise the result to 99.85 %. The database is now the **single point of failure**: it has no copy, so it limits the whole chain.

These formulas assume that the copies fail *independently*. Two copies on the same machine, behind the same switch, or running the same buggy version fail together, and then redundancy gives almost nothing. A bad deployment is the classic example: it reaches every copy at once.

> 🧪 **Testing:** Floating-point results are rarely exact (Lesson 12.3), so compare them with a tolerance. Add tests to `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn close(a: f64, b: f64) -> bool {
        (a - b).abs() < 1e-9
    }

    #[test]
    fn three_nines_allow_about_526_minutes() {
        let minutes = downtime_minutes_per_year(0.999);
        assert!(close(minutes, 525.6), "got {minutes}");
    }

    #[test]
    fn a_chain_is_weaker_than_its_weakest_part() {
        let chain = serial(&[0.999, 0.999, 0.999]);
        assert!(chain < 0.999);
        assert!(close(chain, 0.997002999));
    }

    #[test]
    fn two_copies_are_stronger_than_one() {
        let pair = redundant(&[0.99, 0.99]);
        assert!(close(pair, 0.9999));
    }
}
```

```bash
cargo test -q --lib
```

```text
running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 5. Prevent, detect, contain, recover

Every technique in this phase belongs to one of four layers. Thinking in layers stops you from relying on a single defence.

| Layer | Question | Techniques (lesson) |
|---|---|---|
| **Prevent** | How do we stop faults from entering the code? | Clear error handling (20.2), fuzzing (20.3), Miri, sanitizers and loom (20.4), formal verification with Kani (20.6), supply-chain checks (20.7) |
| **Detect** | How do we notice quickly that something is wrong? | Logs, metrics and traces (Lesson 16.12), health checks (Lesson 16.13), panic reports (20.2) |
| **Contain** | How do we keep one fault small? | Timeouts and limits (20.8), isolating tasks (20.2), testing failure on purpose (20.5) |
| **Recover** | How do we get back to normal? | Restarts, retries with backoff (Lesson 18.3), graceful shutdown (Lesson 14.14), the readiness checklist (20.9) |

**Hardening** cuts across all four layers. It means looking at a working program and asking "what could go wrong here, and how do I make that smaller?". The overflow check of Section 2 is a tiny example: it moved a fault from the silent column into the loud column. A limit on the size of a request, a timeout on every network call and running as a user without special rights are other examples. In Lesson 20.9 you collect these questions into a checklist, and in Project 20.10 you apply them to a whole service.

> 🔒 **Security:** Reliability and security overlap. An attacker who can make your service crash, hang or run out of memory has caused a *denial of service* (Lesson 10.12). Most hardening steps protect against accidents and attackers at the same time.

> 📏 **Convention:** Make failure *loud and local*. A fault should be reported where it happens, with enough context to understand it, and it should hurt as few users as possible. Silence and spread are the two things to avoid.

## Common mistakes

**1. Treating "it did not crash" as "it worked".** The release build of `silent.rs` exited with code 0 and a wrong number. Exit code 0 only means the program did not report a problem. Check results, not just the absence of crashes.

**2. Adding nines by adding parts.** A second database, a cache and a message queue each add a dependency. If every request now needs all of them, availability goes *down* (Section 4). Redundancy helps only when one copy is *enough*.

**3. Copies that fail together.** Two replicas on the same machine are one replica with extra steps. Independence matters more than the number of copies.

**4. Rounding the year.** Using 360 days or forgetting leap years changes the result by about 1–2 %. For goals and dashboards, always say which period you measure (a month, 30 days, a quarter).

## Try it yourself

1. Create `code/p20-reliability` and add `src/lib.rs`, `src/bin/nines.rs` and `src/bin/chain.rs` as shown.
2. Run `cargo run -q --bin nines` and `cargo run -q --bin chain`; compare with the outputs above.
3. Create `src/bin/silent.rs`. Run it in debug and release mode, then add `overflow-checks = true` and run the release build again.
4. Change `total_bytes` to `u64` (both the variable and `next_response`). Run the release build: the total is now correct.
5. Add the three tests and run `cargo test -q --lib`.

## Exercises

### Exercise 1 — Three and a half nines (easy)

How many minutes per year and per month may a service with 99.95 % availability be down? Write a small binary `src/bin/ex1.rs` that prints both numbers with one digit after the point.

<details>
<summary>Hint</summary>

Call `downtime_minutes_per_year(0.9995)` and divide by 12 for the month.

</details>

<details>
<summary>Answer</summary>

```rust
use p20_reliability::downtime_minutes_per_year;

fn main() {
    let year = downtime_minutes_per_year(0.9995);
    println!("99.95 %: {:.1} min per year, {:.1} min per month", year, year / 12.0);
}
```

```text
99.95 %: 262.8 min per year, 21.9 min per month
```

Why it works: 0.05 % of 525 600 minutes is 262.8 minutes, about four and a half hours per year.

</details>

### Exercise 2 — Availability from MTBF and MTTR (medium)

Add `pub fn availability_from(mtbf_hours: f64, mttr_hours: f64) -> f64` to `src/lib.rs`. Write two tests: one shows that MTBF 999 h and MTTR 1 h give 0.999, the other shows that halving the recovery time gives a higher availability.

<details>
<summary>Hint</summary>

The formula is `mtbf / (mtbf + mttr)`. Compare floats with a tolerance such as `1e-12`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Availability from the mean time between failures and the mean time to recover.
pub fn availability_from(mtbf_hours: f64, mttr_hours: f64) -> f64 {
    mtbf_hours / (mtbf_hours + mttr_hours)
}

#[cfg(test)]
mod exercise_tests {
    use super::*;

    #[test]
    fn one_hour_to_recover_every_999_hours_is_three_nines() {
        let a = availability_from(999.0, 1.0);
        assert!((a - 0.999).abs() < 1e-12, "got {a}");
    }

    #[test]
    fn faster_recovery_gives_more_availability() {
        assert!(availability_from(999.0, 0.5) > availability_from(999.0, 1.0));
    }
}
```

Why it works: the second test states the lesson of Section 3 as code: you can buy availability with faster recovery, without failing less often.

</details>

### Exercise 3 — Name the failure mode (medium)

For each story, name the failure mode from Section 2 and one layer (prevent, detect, contain, recover) that would have helped most.

1. After an update, the shop shows prices without the tax for three days. Nobody gets an error.
2. A report request reads a 40 GB (gigabyte, about 40 billion bytes) table into memory, and the process is killed by the operating system.
3. The login service calls an address service that has stopped answering. Login requests wait forever and the login service runs out of threads.

<details>
<summary>Hint</summary>

Look at the "How people notice" column. Ask "was it loud or silent?" and "where could it have been stopped?".

</details>

<details>
<summary>Answer</summary>

1. **Wrong answer** (a silent failure). *Prevent* (a test that checks a price with tax) and *detect* (a metric or check that compares totals).
2. **Crash** caused by **overload** of memory. *Contain*: a limit on how much one request may read, so one bad request cannot take the process down (Lesson 20.8).
3. **Hang** caused by a **dependency failure**. *Contain*: a timeout on every outgoing call (Lesson 14.14), so a dead dependency gives a quick error instead of a stuck thread.

</details>

### Exercise 4 — How many copies? (hard)

Write `pub fn copies_needed(single: f64, target: f64) -> Option<usize>`: the smallest number of independent copies, each with availability `single`, that together reach at least `target`. Return `None` if `single` is 0 or less, and give up (return `None`) after 100 copies. Write tests: 0.99 copies reach 0.99999 with 3 copies; a 0.9999 copy reaches 0.999 alone; a 0.0 copy never helps.

<details>
<summary>Hint</summary>

Start with one copy and add copies while `redundant(&vec![single; copies])` is below the target. `vec![x; n]` makes a vector of `n` copies of `x`.

</details>

<details>
<summary>Answer</summary>

```rust
/// The smallest number of copies (each with availability `single`) that reaches `target`.
/// Returns `None` if `single` is 0 or less, because then no number of copies helps.
pub fn copies_needed(single: f64, target: f64) -> Option<usize> {
    if single <= 0.0 {
        return None;
    }
    let mut copies = 1;
    while redundant(&vec![single; copies]) < target {
        copies += 1;
        if copies > 100 {
            return None;
        }
    }
    Some(copies)
}

#[cfg(test)]
mod copies_tests {
    use super::*;

    #[test]
    fn three_copies_of_two_nines_reach_five_nines() {
        assert_eq!(copies_needed(0.99, 0.99999), Some(3));
    }

    #[test]
    fn one_copy_is_enough_when_it_is_already_good() {
        assert_eq!(copies_needed(0.9999, 0.999), Some(1));
    }

    #[test]
    fn a_dead_copy_never_helps() {
        assert_eq!(copies_needed(0.0, 0.9), None);
    }
}
```

Why it works: two copies of 99 % give 1 − 0.01² = 99.99 %, which is still below 99.999 %; three copies give 99.9999 %. The limit of 100 copies guarantees that the loop ends even for a target of exactly 1.0, which no finite number of copies reaches.

</details>

## Quiz

1. What is the difference between a fault and a failure?

<details>
<summary>Answer</summary>

A fault is a cause: a bug, a full disk, a lost packet. A failure is what users see: wrong behaviour or no answer. Fault tolerance keeps faults from becoming failures.

</details>

2. Why is a wrong answer usually worse than a crash?

<details>
<summary>Answer</summary>

A crash is loud: it is noticed, logged and restarted. A wrong answer is silent: it spreads into stored data and other systems before anyone notices, and it is much harder to clean up.

</details>

3. Roughly how much downtime per month do three nines allow?

<details>
<summary>Answer</summary>

About 44 minutes (0.1 % of a month). Each extra nine divides it by ten.

</details>

4. A service fails once a month and needs two hours to recover. Name two ways to improve its availability.

<details>
<summary>Answer</summary>

Fail less often (raise MTBF: better tests, fewer risky changes) or recover faster (lower MTTR: automatic restarts, health checks, quick rollback). Faster recovery is often the cheaper one.

</details>

5. Your request path is load balancer → app → database, each 99.9 %. Is the whole path 99.9 %?

<details>
<summary>Answer</summary>

No. All three are needed, so the availabilities multiply: 0.999³ ≈ 99.7 %. A chain is weaker than its weakest part.

</details>

6. What does `overflow-checks = true` in `[profile.release]` change, and which layer is it?

<details>
<summary>Answer</summary>

Release builds then panic on integer overflow instead of silently wrapping around. It is a hardening step in the *detect* layer: it turns a silent wrong answer into a loud, visible failure.

</details>

7. When does adding a second copy of a part give almost no benefit?

<details>
<summary>Answer</summary>

When both copies fail together: same machine, same network switch, same bad deployment. Redundancy helps only if failures are independent.

</details>

## Summary

- A **fault** is a cause; a **failure** is what users see. Reliability means keeping faults from becoming failures and keeping failures small, visible and short.
- Typical failure modes: crash, hang, slow answer, wrong answer, overload, data loss, dependency failure. Silent ones (wrong answers, corruption) are the most dangerous.
- Availability is often written in **nines**; each nine divides the allowed downtime by ten. `availability = MTBF / (MTBF + MTTR)`, so faster recovery is as valuable as fewer failures.
- Parts that are all needed multiply their availabilities (worse than the weakest); independent copies where one is enough give much better availability.
- Every technique in this phase belongs to a layer: **prevent, detect, contain, recover**. Hardening asks of every part: "what can go wrong, and how do I make it smaller?".
- `overflow-checks = true` in release builds is a first, cheap hardening step: it makes overflow loud.

## Sources and further reading

- [Site Reliability Engineering — Embracing Risk](https://sre.google/sre-book/embracing-risk/) — Google's book chapter on why 100 % is the wrong goal and how to choose an availability target.
- [Site Reliability Engineering — Availability Table](https://sre.google/sre-book/availability-table/) — the downtime for each level of nines, and the effect of MTTR.
- [The Cargo Book — Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html) — `overflow-checks` and the other settings of `[profile.release]`.
- [std::hint::black_box](https://doc.rust-lang.org/std/hint/fn.black_box.html) — what `black_box` does and does not promise.
- [The Rust Reference — Integer overflow](https://doc.rust-lang.org/reference/expressions/operator-expr.html) — the rules for overflow in debug and release builds.

---

[🏠 Start](../START-HERE.md) · [← Phase 20 — Reliability and hardening](README.md) · [📋 Phase 20](README.md) · [20.2 Errors and panics in production →](02-errors-and-panics-in-production.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
