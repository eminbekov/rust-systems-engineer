<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.2 Failures and timeouts](02-failures-and-timeouts.md) · [📋 Phase 18](README.md) · [18.4 Time and clocks →](04-time-and-clocks.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/03-retries-backoff-and-idempotency.md) <!-- nav -->

# Lesson 18.3 — Retries, backoff and idempotency

> **Phase 18 — Distributed systems** · Lesson 3 of 15 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Which failures are worth a retry and which are not.
- How exponential backoff and jitter keep thousands of retrying clients from knocking a recovering server down again.
- How to write a small generic `retry` helper for async code and test it with a paused clock.
- What idempotency is, and why "retry after a timeout" is only safe for idempotent operations.
- How idempotency keys make a non-idempotent operation, like a deposit, safe to retry.

## Before you start

- You finished [Lesson 18.2 — Failures and timeouts](02-failures-and-timeouts.md). This lesson starts from its main result: after a timeout the operation may or may not have happened.
- You know closures and the `Fn`/`FnMut` traits from Phase 05, generic functions with `where` clauses from [Lesson 05.1](../05-rust-intermediate/01-generics.md), and property-based tests from [Lesson 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md) (for Exercise 3).
- Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p18-retries
cd code/p18-retries
cargo add tokio@1 --features full
cargo add rand@0.10
cargo add --dev tokio@1 --features test-util
cargo add --dev proptest@1
mkdir src/bin tests
```

`rand` is the standard crate for random numbers. We need it for jitter.

## New words

| Word | Meaning |
|---|---|
| **retry** | Sending the same request again after it failed or timed out. |
| **transient failure** | A failure that goes away by itself after a while: a busy server, a dropped packet. Worth a retry. |
| **permanent failure** | A failure that will happen again every time: a wrong password, a missing file. Never worth a retry. |
| **backoff** | Waiting between retries. *Exponential* backoff doubles the wait after each failure, up to a *cap* (maximum). |
| **jitter** | Randomness added to a wait, so that many clients do not all retry at the same moment. |
| **thundering herd** | Many clients that act at exactly the same moment, for example all retrying together after an outage, and overload the server again. |
| **idempotent** | An operation is idempotent if doing it twice has the same effect as doing it once. "Set the price to 5" is idempotent; "add 5 to the price" is not. |
| **idempotency key** | A unique id that the client sends with a request and with every retry of it, so the server can recognise a repeat and not do the work twice. |
| **at-least-once delivery** | A message is delivered one or more times, never zero times (Lesson 16.11). Retrying gives at-least-once. |
| **RFC** (Request for Comments) | A document that defines an internet standard, for example RFC 9110 for HTTP. |
| **UUID** (Universally Unique Identifier) | A random 128-bit id. Two UUIDs made anywhere are, in practice, never equal. |
| **HTTP** (Hypertext Transfer Protocol) | The protocol of the web (Lesson 15.8). Its answers carry a three-digit *status code*, for example 200 (OK) or 503 (Service Unavailable). |

## 1. When is a retry a good idea?

A retry helps only if the next attempt has a real chance to succeed. Sort failures into two groups first:

| Failure | Kind | Retry? |
|---|---|---|
| Timeout, connection reset, connection refused while a node restarts | transient | yes, with backoff |
| "Server busy", "too many requests" (HTTP 429, 503) | transient | yes, and wait longer |
| "Bad request", "not found", "access denied" (HTTP 400, 404, 403) | permanent | no: the same request fails again |
| A bug in the client (wrong data format) | permanent | no: fix the code |

Then ask the second question, which is the subject of this lesson: **is it safe to do the operation twice?**
Lesson 18.2 showed that after a timeout the first attempt may have succeeded. A retry then does the work a
second time.

## 2. Exponential backoff

If a server is overloaded, retrying at once makes it worse. So we wait, and we wait longer after every
failure: 100 ms, 200 ms, 400 ms, 800 ms, and so on. To avoid absurd waits there is a **cap**. Put this into
`src/lib.rs`:

```rust
//! Retries, backoff and idempotency (Lesson 18.3).

use std::time::Duration;

/// Exponential backoff before retry number `attempt` (0, 1, 2, ...): base × 2^attempt, at most `cap`.
pub fn backoff(attempt: u32, base: Duration, cap: Duration) -> Duration {
    let factor = 2u32.saturating_pow(attempt);
    base.saturating_mul(factor).min(cap)
}
```

**What happens:**

1. `2u32.saturating_pow(attempt)` is 2 to the power `attempt`. "Saturating" means that instead of overflowing
   it stops at the largest `u32` (Lesson 12.2), so attempt 40 does not panic.
2. `base.saturating_mul(factor)` multiplies the duration, again without overflow, and `.min(cap)` limits it.

Print a table with `src/bin/backoff_table.rs`:

```rust
use std::time::Duration;

use p18_retries::backoff;

fn main() {
    let base = Duration::from_millis(100);
    let cap = Duration::from_secs(5);
    for attempt in 0..8 {
        println!("retry {attempt}: wait {:?}", backoff(attempt, base, cap));
    }
}
```

`cargo run --bin backoff_table`:

```text
retry 0: wait 100ms
retry 1: wait 200ms
retry 2: wait 400ms
retry 3: wait 800ms
retry 4: wait 1.6s
retry 5: wait 3.2s
retry 6: wait 5s
retry 7: wait 5s
```

## 3. Jitter: do not retry all at once

Imagine a server that 1,000 clients use. It crashes and restarts. All 1,000 clients saw their request fail at
the same moment, and all of them use the same backoff, so all of them retry **at the same moment** again: a
**thundering herd**. The fresh server gets 1,000 requests in one millisecond and falls over again.

The fix is **jitter**: each client waits a *random* part of the backoff. The simplest and very effective
version is called **full jitter**: wait a random time between zero and the backoff. Add `use rand::RngExt;` at
the top of `src/lib.rs`, and then this function:

```rust
/// "Full jitter": a random wait between zero and the exponential backoff.
pub fn full_jitter(attempt: u32, base: Duration, cap: Duration, rng: &mut impl RngExt) -> Duration {
    let max = backoff(attempt, base, cap).as_millis() as u64;
    Duration::from_millis(rng.random_range(0..=max))
}
```

`RngExt` is the `rand` 0.10 trait with the convenient methods such as `random_range`. The function takes the
random number generator as a parameter, so a demo or a test can pass a **seeded** generator, which gives the
same numbers on every run. Let us compare 1,000 clients with and without jitter, `src/bin/jitter.rs`:

```rust
use std::time::Duration;

use p18_retries::{backoff, full_jitter};
use rand::SeedableRng;
use rand::rngs::StdRng;

/// Prints how many of the clients retry in each 100 ms window, one `#` per 20 clients.
fn histogram(title: &str, waits: &[Duration]) {
    println!("{title}");
    for window in 0..5 {
        let from = Duration::from_millis(window * 100);
        let to = from + Duration::from_millis(100);
        let count = waits.iter().filter(|w| **w >= from && **w < to).count();
        let bar = "#".repeat(count / 20);
        println!(
            "  {:>3}-{:>3} ms {:>5} {bar}",
            window * 100,
            window * 100 + 99,
            count
        );
    }
}

fn main() {
    let base = Duration::from_millis(100);
    let cap = Duration::from_secs(5);
    let attempt = 2; // the third retry: the backoff is 400 ms
    let clients = 1000;
    let mut rng = StdRng::seed_from_u64(42);

    // Without jitter every client waits exactly 400 ms: all at the same moment.
    let fixed: Vec<Duration> = (0..clients).map(|_| backoff(attempt, base, cap)).collect();
    let jittered: Vec<Duration> = (0..clients)
        .map(|_| full_jitter(attempt, base, cap, &mut rng))
        .collect();

    histogram("without jitter", &fixed);
    histogram("with full jitter", &jittered);
}
```

`cargo run --bin jitter`:

```text
without jitter
    0- 99 ms     0 
  100-199 ms     0 
  200-299 ms     0 
  300-399 ms     0 
  400-499 ms  1000 ##################################################
with full jitter
    0- 99 ms   240 ############
  100-199 ms   251 ############
  200-299 ms   266 #############
  300-399 ms   241 ############
  400-499 ms     2 
```

**What happens:**

1. `StdRng::seed_from_u64(42)` makes a generator that always produces the same sequence for the seed 42. The
   real program would use `rand::rng()`, a generator seeded from the operating system.
2. Without jitter, all 1,000 retries land in the same window. With full jitter they are spread evenly over
   the 400 ms: about 250 per 100 ms. (The two in the last window drew exactly 400 ms, because `0..=max`
   includes `max`.)
3. The server now sees a steady stream instead of a wall. The average client waits only half as long as before,
   and the server recovers instead of crashing again.

> ⚡ **Performance:** Jitter is not only for retries. Any periodic job that runs on many machines (cache
> refresh, heartbeats, cron jobs at "every full hour") should add a little randomness, or all machines will hit
> the shared database at the same second.

## 4. A retry helper

Now we combine backoff and jitter into one helper that can retry **any** async operation. Add it to
`src/lib.rs`:

```rust
/// How often and how patiently to retry.
pub struct RetryPolicy {
    pub max_attempts: u32,
    pub base: Duration,
    pub cap: Duration,
}

/// Calls `op` until it succeeds, until it fails with an error that `retryable`
/// rejects, or until `max_attempts` calls were made. Waits with full jitter in between.
pub async fn retry<T, E, F, Fut>(
    policy: &RetryPolicy,
    retryable: impl Fn(&E) -> bool,
    mut op: F,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    let mut attempt = 0;
    loop {
        match op().await {
            Ok(value) => return Ok(value),
            Err(error) if attempt + 1 < policy.max_attempts && retryable(&error) => {
                let wait = full_jitter(attempt, policy.base, policy.cap, &mut rand::rng());
                tokio::time::sleep(wait).await;
                attempt += 1;
            }
            Err(error) => return Err(error),
        }
    }
}
```

**What happens:**

1. `op` is a closure that **starts** one attempt: each call `op()` returns a new future, of type `Fut`, which
   we then `.await`. We cannot take a single future, because a future can be awaited only once, and we may need
   several attempts. `FnMut` allows the closure to change its captured variables, for example to count attempts.
2. `retryable` decides which errors are worth another try (Section 1). Every other error is returned at once.
3. The guard `attempt + 1 < policy.max_attempts` stops after the last allowed attempt and returns its error.
4. `&mut rand::rng()` creates the operating system–seeded generator just for this one line. It is dropped
   before the `.await`, which matters: that generator may not be moved between threads, and Tokio may move a
   task to another thread at an `.await`.

## 5. Idempotency

Retrying is now easy. Whether it is **correct** depends on the operation. An operation is **idempotent** if doing
it twice leaves the system in the same state as doing it once.

| Operation | Idempotent? | Why |
|---|---|---|
| Set the balance to 100 | yes | The second time, it is already 100. |
| Add 100 to the balance | **no** | Twice adds 200. |
| Delete order 17 | yes | The second time there is nothing to delete. |
| Create a new order | **no** | Twice creates two orders. |
| Read the balance | yes | Reading changes nothing. |
| HTTP `GET`, `PUT`, `DELETE` | yes, by definition | RFC 9110, the HTTP standard, requires it. |
| HTTP `POST` | **no** | It usually creates something or triggers an action. |

If an operation is idempotent, "retry after a timeout" is always safe: even if the first attempt went through, the second changes nothing. For
non-idempotent operations, such as a deposit, we need a trick.

## 6. Idempotency keys

The trick is to let the **client** give every logical operation a unique id, the **idempotency key**, and send
the same key with every retry. The server remembers the keys it has already processed, together with the answer
it gave. When a known key arrives again, it does not do the work again; it just returns the remembered answer.

Add `use std::collections::HashMap;` at the top of `src/lib.rs`, and then the bank's rules as a plain struct
without any networking:

```rust
/// The bank's data and rules, without any networking.
#[derive(Default)]
pub struct Ledger {
    balance: u64,
    /// Idempotency key → the balance we answered the first time.
    seen: HashMap<u64, u64>,
}

impl Ledger {
    /// Deposits `amount`. A request with a key that was seen before changes nothing
    /// and gets the same answer as the first time.
    pub fn deposit(&mut self, key: Option<u64>, amount: u64) -> u64 {
        if let Some(&earlier) = key.and_then(|key| self.seen.get(&key)) {
            return earlier;
        }
        self.balance += amount;
        if let Some(key) = key {
            self.seen.insert(key, self.balance);
        }
        self.balance
    }

    pub fn balance(&self) -> u64 {
        self.balance
    }
}
```

The key is an `Option` so that we can compare: `None` is an old-style request without a key. Keeping the rules
in a pure struct also makes them easy to test on their own (Exercise 3).

Now a bank "node" behind a bad network. Lesson 18.2 built the network with channels and real timeouts; here we
only simulate the result, to keep the code short: the bank **does** each deposit, but for its first few
deposits the answer is lost, and the client waits `limit` in vain and gets a timeout. Add
`use std::sync::Mutex;` at the top of `src/lib.rs`, and then:

```rust
/// The answer did not come back in time.
#[derive(Debug, PartialEq)]
pub struct Timeout;

/// A bank node behind a bad network: the answers to the first `lost_replies`
/// deposits are lost on the way back.
pub struct Bank {
    ledger: Mutex<Ledger>,
    lost_replies: Mutex<usize>,
}

impl Bank {
    pub fn new(lost_replies: usize) -> Self {
        Bank {
            ledger: Mutex::new(Ledger::default()),
            lost_replies: Mutex::new(lost_replies),
        }
    }

    /// One deposit call over the network; the client waits at most `limit`.
    pub async fn deposit(
        &self,
        key: Option<u64>,
        amount: u64,
        limit: Duration,
    ) -> Result<u64, Timeout> {
        let new_balance = self.ledger.lock().unwrap().deposit(key, amount); // the work is done...
        let answer_lost = {
            let mut lost = self.lost_replies.lock().unwrap();
            let answer_lost = *lost > 0;
            if answer_lost {
                *lost -= 1;
            }
            answer_lost
        };
        if answer_lost {
            tokio::time::sleep(limit).await; // ...but the client waits in vain
            return Err(Timeout);
        }
        Ok(new_balance)
    }

    pub fn balance(&self) -> u64 {
        self.ledger.lock().unwrap().balance()
    }
}
```

**What happens:**

1. `Bank` wraps the ledger and a counter of answers to lose. Both are behind a `Mutex`, because `deposit` takes
   `&self` and many calls could run at the same time (Lesson 14.2).
2. `deposit` changes the ledger **first**. Then it checks whether this answer is one of the lost ones. The
   inner block `{ ... }` ends before the `.await`, so the lock is released before we sleep: a `std` mutex must
   never be held across an `.await`.
3. For a lost answer the client waits the whole `limit` and gets `Err(Timeout)`, although the money is already in.

Now the experiment, `src/bin/double_charge.rs`. It deposits 100 with retries, once without a key and once with
the key 7, each time on a fresh bank that loses two answers. Every error here is a timeout, which is transient,
so the `retryable` closure is simply `|_| true`:

```rust
use std::time::Duration;

use p18_retries::{Bank, RetryPolicy, retry};

#[tokio::main]
async fn main() {
    let policy = RetryPolicy {
        max_attempts: 5,
        base: Duration::from_millis(50),
        cap: Duration::from_secs(1),
    };
    let limit = Duration::from_millis(200);

    for key in [None, Some(7)] {
        // A fresh bank that loses the answers to the first two deposits.
        let bank = Bank::new(2);
        let mut attempt = 0;
        let result = retry(
            &policy,
            |_| true,
            || {
                attempt += 1;
                println!("key {key:?}: attempt {attempt}");
                bank.deposit(key, 100, limit)
            },
        )
        .await;
        println!(
            "key {key:?}: result {result:?}, balance {}\n",
            bank.balance()
        );
    }
}
```

`cargo run --bin double_charge`:

```text
key None: attempt 1
key None: attempt 2
key None: attempt 3
key None: result Ok(300), balance 300

key Some(7): attempt 1
key Some(7): attempt 2
key Some(7): attempt 3
key Some(7): result Ok(100), balance 100
```

**What happens:**

1. Without a key, attempts 1 and 2 were **done** by the bank, but their answers were lost; the client saw two
   timeouts and retried. Attempt 3 got an answer. The user wanted to deposit 100, and the bank now holds 300.
2. With the key 7, attempt 1 did the deposit and remembered "key 7 → 100". Attempts 2 and 3 carried the same
   key; the ledger recognised it and answered 100 without adding anything.
3. The retry helper and the network are exactly the same in both runs. Only the key makes the difference.

This is the standard recipe of distributed systems: **at-least-once delivery + idempotent processing =
the effect of exactly once**. True "exactly-once delivery" over an unreliable network is impossible, but
the *effect* of exactly once is achievable.

> 🧪 **Testing:** Turn the experiment into tests that need no real time. Add to the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    const POLICY: RetryPolicy = RetryPolicy {
        max_attempts: 5,
        base: Duration::from_millis(50),
        cap: Duration::from_secs(1),
    };
    const LIMIT: Duration = Duration::from_millis(200);

    #[tokio::test(start_paused = true)]
    async fn retrying_without_a_key_counts_the_money_three_times() {
        let bank = Bank::new(2);
        let result = retry(&POLICY, |_| true, || bank.deposit(None, 100, LIMIT)).await;
        assert_eq!(result, Ok(300));
        assert_eq!(bank.balance(), 300);
    }

    #[tokio::test(start_paused = true)]
    async fn retrying_with_a_key_counts_the_money_once() {
        let bank = Bank::new(2);
        let result = retry(&POLICY, |_| true, || bank.deposit(Some(7), 100, LIMIT)).await;
        assert_eq!(result, Ok(100));
        assert_eq!(bank.balance(), 100);
    }
}
```

`cargo test --lib`:

```text
test tests::retrying_with_a_key_counts_the_money_once ... ok
test tests::retrying_without_a_key_counts_the_money_three_times ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The first test documents the bug on purpose: it would warn you if somebody "fixed" the bank in a way that hides
the problem instead of solving it.

Three rules make idempotency keys work in a real service:

- **One key per logical operation, not per attempt.** The client creates the key *before* the first attempt
  (for example a UUID, Universally Unique Identifier: a random 128-bit id) and reuses it for every retry.
- **Store the key and the change together.** If the server adds the money and then crashes before it saves
  the key, the retry adds it again. Write both in one database transaction ([Lesson 16.7](../16-backend-services/07-migrations-and-transactions.md)).
- **Decide how long keys are kept.** The table of seen keys grows forever unless old keys expire, for example
  after 24 hours; retries must happen within that time.

> 🔒 **Security:** The server must not trust a key alone. If user A's key could return user B's stored answer,
> a guessed key would leak data. Store the key together with the user and check both.

## Common mistakes

**A new key for every attempt.** This looks like idempotency but does nothing. If, in `double_charge.rs`, the
key is created *inside* the closure, every attempt gets a fresh random key:

```rust
            || {
                attempt += 1;
                println!("key {key:?}: attempt {attempt}");
                // WRONG: a new key for every attempt
                bank.deposit(Some(rand::random::<u64>()), 100, limit)
            },
```

```text
key None: result Ok(300), balance 300
key Some(7): result Ok(300), balance 300
```

The server sees three different keys, so it sees three different deposits. The fix: create the key once,
before calling `retry`, and let the closure only copy it.

> ⚠️ **Common mistake: retrying at every layer.** If a web service retries a database call 3 times, and its
> client retries the web call 3 times, and the user's app retries 3 times, one click can become 3 × 3 × 3 = 27
> database calls during an outage. Retry in **one** layer, usually the one closest to the failing call, and let
> the others fail fast.

> ⚠️ **Common mistake: retrying permanent errors.** Retrying "404 Not Found" five times only makes the user wait
> five times longer for the same answer. Exercise 2 sorts status codes.

## Try it yourself

1. In `double_charge.rs`, change `Bank::new(2)` to `Bank::new(4)`. Without a key the bank ends at 500. With
   the key it is still 100.
2. Change it to `Bank::new(10)`. All five attempts time out and `retry` returns `Err(Timeout)`. Look at the
   balance: it is **not** zero. The "failed" call still deposited the money (with the key: exactly once).

## Exercises

### Exercise 1 — Test the backoff (easy)

Write an integration test `tests/backoff.rs` that checks two things about `backoff`: with a base of 100 ms and a
cap of 1 s, the first six waits are 100, 200, 400, 800, 1000 and 1000 ms; and huge attempt numbers (40 and
`u32::MAX`) return the cap instead of panicking.

<details>
<summary>Hint</summary>

Collect the waits into a vector of milliseconds with `(0..6).map(|a| backoff(a, base, cap).as_millis())` and
compare the whole vector with one `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::Duration;

use p18_retries::backoff;

#[test]
fn doubles_until_the_cap() {
    let base = Duration::from_millis(100);
    let cap = Duration::from_secs(1);
    let waits: Vec<u128> = (0..6).map(|a| backoff(a, base, cap).as_millis()).collect();
    assert_eq!(waits, vec![100, 200, 400, 800, 1000, 1000]);
}

#[test]
fn a_huge_attempt_number_does_not_overflow() {
    let cap = Duration::from_secs(30);
    assert_eq!(backoff(40, Duration::from_millis(100), cap), cap);
    assert_eq!(backoff(u32::MAX, Duration::from_secs(1), cap), cap);
}
```

`cargo test --test backoff`:

```text
test a_huge_attempt_number_does_not_overflow ... ok
test doubles_until_the_cap ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: with `pow` instead of `saturating_pow`, the second test would panic with "attempt to
exponentiate with overflow" in a debug build. The test protects that detail.

</details>

### Exercise 2 — Which HTTP answers deserve a retry? (medium)

Create `src/http.rs` (and add `pub mod http;` to `src/lib.rs`) with `pub fn is_retryable(status: u16) -> bool`.
Retry 408 (Request Timeout), 429 (Too Many Requests) and the 5xx server errors, except 501 (Not Implemented) and
505 (HTTP Version Not Supported), which will never work. Do not retry anything else. Write a table test with at
least six retryable and eight non-retryable codes.

<details>
<summary>Hint</summary>

A `match` with ranges reads well: `500..=599 => true`. Put the two exceptions in an arm **before** the range, because
`match` takes the first arm that fits.

</details>

<details>
<summary>Answer</summary>

```rust
//! Exercise 2: which HTTP answers are worth a retry.

/// `true` if a request that got this HTTP status may succeed when it is sent again.
pub fn is_retryable(status: u16) -> bool {
    match status {
        408 | 429 => true,  // Request Timeout, Too Many Requests
        501 | 505 => false, // Not Implemented, HTTP Version Not Supported: will never work
        500..=599 => true,  // server-side trouble, often temporary
        _ => false,         // success, redirects and the other 4xx client errors
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn retries_only_temporary_problems() {
        for status in [408, 429, 500, 502, 503, 504] {
            assert!(is_retryable(status), "{status} should be retried");
        }
        for status in [200, 201, 301, 400, 401, 403, 404, 409, 422, 501] {
            assert!(!is_retryable(status), "{status} should not be retried");
        }
    }
}
```

Why it works: the message in each `assert!` names the code that failed, so a table test still tells you exactly
which case is wrong. Remember that "retryable" is only half of the decision: a `POST` that got a 503 is safe to
retry only with an idempotency key.

</details>

### Exercise 3 — Prove that repeats change nothing (hard)

Write property-based tests (Lesson 09.4) in `tests/ledger_props.rs` for `Ledger`. Generate lists of up to 20
requests `(key, amount)` with keys from `0..5` (so repeats are frequent) and amounts from `1..100`. Property 1:
applying the whole list twice gives the same balance as applying it once. Property 2: the balance equals the sum
of the **first** amount seen for each key.

<details>
<summary>Hint</summary>

`prop::collection::vec((0u64..5, 1u64..100), 0..20)` generates such lists. For property 2, build the expected
value with a `HashMap` and `entry(key).or_insert(amount)`, which keeps only the first amount for each key.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;

use p18_retries::Ledger;
use proptest::prelude::*;

proptest! {
    #[test]
    fn repeating_every_request_changes_nothing(
        requests in prop::collection::vec((0u64..5, 1u64..100), 0..20)
    ) {
        let mut once = Ledger::default();
        for &(key, amount) in &requests {
            once.deposit(Some(key), amount);
        }
        let mut twice = Ledger::default();
        for &(key, amount) in requests.iter().chain(requests.iter()) {
            twice.deposit(Some(key), amount);
        }
        prop_assert_eq!(once.balance(), twice.balance());
    }

    #[test]
    fn only_the_first_request_with_a_key_counts(
        requests in prop::collection::vec((0u64..5, 1u64..100), 0..20)
    ) {
        let mut ledger = Ledger::default();
        let mut first: HashMap<u64, u64> = HashMap::new();
        for &(key, amount) in &requests {
            ledger.deposit(Some(key), amount);
            first.entry(key).or_insert(amount);
        }
        prop_assert_eq!(ledger.balance(), first.values().sum::<u64>());
    }
}
```

`cargo test --test ledger_props`:

```text
test repeating_every_request_changes_nothing ... ok
test only_the_first_request_with_a_key_counts ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Why it works: property 1 is the definition of idempotency, checked on 256 random histories. Property 2 also says
that a repeat with a **different** amount is ignored: the server answers the first request again, it does not
mix the two. Try deposits with `None` instead of `Some(key)`: property 1 fails at once and proptest shrinks the
counterexample to a single request.

</details>

## Quiz

1. Which of these should a client retry: a timeout, "403 Forbidden", "503 Service Unavailable"?

<details>
<summary>Answer</summary>

The timeout and 503 are transient and worth a retry with backoff. 403 is permanent: the same request will be
refused again.

</details>

2. What problem does jitter solve?

<details>
<summary>Answer</summary>

The thundering herd: clients that failed at the same moment would otherwise retry at the same moment and overload
the recovering server again. Random waits spread the retries over time.

</details>

3. Is "set the email of user 5 to a@b.c" idempotent? Is "send a welcome email to user 5"?

<details>
<summary>Answer</summary>

The first is idempotent: doing it twice leaves the same email. The second is not: twice sends two emails. It needs
an idempotency key (or a "welcome email sent" flag that is checked and set together with sending).

</details>

4. Why must the client create the idempotency key before the first attempt?

<details>
<summary>Answer</summary>

The key must be the same for all attempts of one operation. If each attempt creates its own key, the server sees
every retry as a new operation, and the work is done again.

</details>

5. What does "at-least-once delivery + idempotent processing" give you?

<details>
<summary>Answer</summary>

The effect of exactly-once: a message may arrive several times, but processing it more than once changes
nothing. True exactly-once delivery over an unreliable network is impossible.

</details>

## Summary

- Retry only transient failures, and only operations that are safe to repeat.
- Wait between retries with exponential backoff and a cap, and add full jitter so that clients do not retry in
  lockstep.
- A small generic `retry` helper takes a closure that starts one attempt; test it with a paused clock.
- An operation is idempotent if doing it twice equals doing it once. Reads, "set" and "delete" usually are;
  "add", "create" and "send" are not.
- Idempotency keys make any operation safe to retry: one key per logical operation, stored together with the
  change, kept for a limited time.
- At-least-once delivery plus idempotent processing gives the effect of exactly once.

## Sources and further reading

- [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — the Amazon Builders' Library on retries in large systems, including retry amplification across layers.
- [Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — the AWS (Amazon Web Services) Architecture Blog post that compares jitter strategies with simulations; "full jitter" comes from here.
- [Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) — how idempotency keys (client request tokens) are designed in a real cloud API (Application Programming Interface).
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html) — the HTTP standard; its section on idempotent methods defines which methods are safe to retry.
- [`rand` documentation](https://docs.rs/rand/latest/rand/) — random number generators, seeding and `random_range`.

---

[🏠 Start](../START-HERE.md) · [← 18.2 Failures and timeouts](02-failures-and-timeouts.md) · [📋 Phase 18](README.md) · [18.4 Time and clocks →](04-time-and-clocks.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
