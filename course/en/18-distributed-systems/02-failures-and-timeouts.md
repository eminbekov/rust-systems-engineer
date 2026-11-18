<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.1 What a distributed system is](01-what-a-distributed-system-is.md) · [📋 Phase 18](README.md) · [18.3 Retries, backoff and idempotency →](03-retries-backoff-and-idempotency.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/02-failures-and-timeouts.md) <!-- nav -->

# Lesson 18.2 — Failures and timeouts

> **Phase 18 — Distributed systems** · Lesson 2 of 15 · ⏱ about 135 minutes

## What you will learn

- The common ways in which nodes and networks fail, from a clean crash to a node that is only slow.
- Why a timeout can never tell you what happened on the other side, shown with a small bank "node".
- How to test timeouts instantly with Tokio's paused clock.
- How to choose a timeout from measured latencies, and how one deadline protects a chain of calls.
- How a failure detector with heartbeats works, and why it can only *suspect* a node, never be sure.

## Before you start

- You finished [Lesson 18.1 — What a distributed system is](01-what-a-distributed-system-is.md).
- You know Tokio tasks, `mpsc` and `oneshot` channels and the actor pattern from [Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md), and `timeout`, deadlines and the paused clock from [Lesson 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md).
- Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p18-failures
cd code/p18-failures
cargo add tokio@1 --features full
cargo add --dev tokio@1 --features test-util
mkdir src/bin
```

In this lesson the "nodes" are Tokio tasks inside one program, and the "network" is a channel. That keeps the
examples small and the tests fast, and the ideas are exactly the same as with real machines.

## New words

| Word | Meaning |
|---|---|
| **failure model** | A description of the ways in which a part of the system is allowed to fail. Algorithms are designed for one failure model. |
| **crash-stop** | A node stops and never comes back. |
| **crash-recovery** | A node stops and later restarts, with only what it had saved to disk (Lesson 17.2). |
| **omission** | A message is lost: it is sent but never delivered. |
| **gray failure** | A node is not dead but works badly: very slow, or failing only some requests. Hard to detect. |
| **Byzantine failure** | A node sends wrong or even malicious messages. Most systems in this course assume nodes do not lie. |
| **timeout** | A limit on how long we wait for an answer. When it expires, we stop waiting; it does not stop the other side. |
| **deadline** | One fixed moment by which a whole operation must finish, shared by all its steps. |
| **heartbeat** | A small message "I am alive" that a node sends regularly. |
| **failure detector** | A component that watches heartbeats and *suspects* nodes that went silent. |
| **OOM** (Out-Of-Memory) killer | The part of the Linux kernel that kills a process when the machine runs out of memory. |
| **UDP** (User Datagram Protocol) | The protocol that sends single messages without connections or delivery guarantees (Lesson 15.3). |
| **CI** (Continuous Integration) | A server that runs your build and tests on every change (Lesson 09.12). |
| **percentile** | The value below which a given share of measurements falls. The 99th percentile (**p99**) of latencies is the time that 99 % of requests did not exceed. Lesson 19.1 covers it in depth. |

## 1. How nodes fail

Engineers describe failures with a few standard names. When you read about a system ("it tolerates the
crash of one node out of three"), the name tells you exactly what is promised.

| Failure | What happens | Example |
|---|---|---|
| **Crash-stop** | The node stops forever. | A machine whose disk burned out is thrown away. |
| **Crash-recovery** | The node stops and restarts. Memory is lost; the disk survives, but only data that was `fsync`ed. | A process is killed by the OOM (Out-Of-Memory) killer and restarted by systemd (Lesson 11.6). |
| **Omission** | Some messages are lost. | A UDP (User Datagram Protocol) packet is dropped by a busy router; a TCP (Transmission Control Protocol) connection is reset. |
| **Network partition** | Groups of nodes cannot reach each other, but all are running. | A switch between two server racks fails. |
| **Gray failure** | The node is alive but slow or partly broken. | A disk that takes 2 s per write; a node paused by Ctrl+Z, as in Lesson 18.1. |
| **Byzantine** | The node sends wrong or malicious messages. | A hacked machine, or a bit flip in memory. Out of scope for this phase. |

Most real systems, and Raft later in this phase, assume **crash-recovery nodes on a network that can lose,
delay, duplicate and reorder messages**. That is the model we use from now on.

> 💡 **Tip:** A slow node is often worse than a dead one. A dead node is noticed and replaced. A slow node
> keeps accepting work, makes every request that touches it slow, and passes every simple health check.

## 2. A timeout cannot see the other side

Lesson 18.1 showed that a node may do the work after the client gave up. Let us make this precise with a
tiny bank. The bank is an actor task (Lesson 14.13). It applies every deposit at once, but its reply needs some
time to travel back, like an answer on a slow network. Put this into `src/lib.rs`:

```rust
//! Failures and timeouts (Lesson 18.2).

use std::time::Duration;

use tokio::sync::{mpsc, oneshot};

/// A request to the bank "node".
pub enum Command {
    Deposit {
        amount: u64,
        reply: oneshot::Sender<u64>,
    },
    Balance {
        reply: oneshot::Sender<u64>,
    },
}

/// Why a call to the bank did not return a result.
#[derive(Debug, PartialEq)]
pub enum CallError {
    /// No answer within the limit: the deposit may or may not have happened.
    Timeout,
    /// The bank is not running any more.
    Gone,
}

/// Starts the bank. It does every deposit at once, but each reply needs `reply_delay`
/// to travel back, like an answer on a slow network.
pub fn spawn_bank(reply_delay: Duration) -> mpsc::Sender<Command> {
    let (tx, mut rx) = mpsc::channel(16);
    tokio::spawn(async move {
        let mut balance = 0;
        while let Some(command) = rx.recv().await {
            match command {
                Command::Deposit { amount, reply } => {
                    balance += amount; // the work is done here...
                    tokio::spawn(async move {
                        tokio::time::sleep(reply_delay).await; // ...the answer is slow
                        let _ = reply.send(balance);
                    });
                }
                Command::Balance { reply } => {
                    let _ = reply.send(balance);
                }
            }
        }
    });
    tx
}

/// Asks the bank to deposit `amount` and waits at most `limit` for the new balance.
pub async fn deposit(
    bank: &mpsc::Sender<Command>,
    amount: u64,
    limit: Duration,
) -> Result<u64, CallError> {
    let (reply, answer) = oneshot::channel();
    let command = Command::Deposit { amount, reply };
    bank.send(command).await.map_err(|_| CallError::Gone)?;
    match tokio::time::timeout(limit, answer).await {
        Ok(Ok(balance)) => Ok(balance),
        Ok(Err(_)) => Err(CallError::Gone),
        Err(_) => Err(CallError::Timeout),
    }
}

/// Asks the bank for the current balance.
pub async fn balance(bank: &mpsc::Sender<Command>) -> Result<u64, CallError> {
    let (reply, answer) = oneshot::channel();
    bank.send(Command::Balance { reply })
        .await
        .map_err(|_| CallError::Gone)?;
    answer.await.map_err(|_| CallError::Gone)
}
```

**What happens:**

1. `spawn_bank` starts a task that owns `balance` and handles one `Command` at a time. Nobody else can touch the
   balance: like a real remote node, it can only be reached by messages.
2. For a deposit, the bank adds the amount **first**. Then it spawns a small task that waits `reply_delay`
   and only then sends the answer. This imitates a reply that is slow on the network.
3. `deposit` is the client. It sends the command and waits for the answer with `tokio::time::timeout`. The
   result has two layers: the outer `Err` means "time is up"; the inner `Err` means the bank dropped the
   reply sender, for example because it stopped.
4. `CallError::Timeout` is documented honestly: "may or may not have happened". There is no way to write a
   better comment.

Now the experiment, `src/bin/timeout_lies.rs`:

```rust
use std::time::Duration;

use p18_failures::{balance, deposit, spawn_bank};

#[tokio::main]
async fn main() {
    // The bank does the work at once, but its answer needs 300 ms to come back.
    let bank = spawn_bank(Duration::from_millis(300));

    match deposit(&bank, 100, Duration::from_millis(100)).await {
        Ok(new_balance) => println!("deposit done, balance {new_balance}"),
        Err(error) => println!("deposit failed: {error:?}"),
    }
    println!(
        "balance after the 'failed' deposit: {:?}",
        balance(&bank).await
    );

    match deposit(&bank, 100, Duration::from_millis(500)).await {
        Ok(new_balance) => println!("deposit done, balance {new_balance}"),
        Err(error) => println!("deposit failed: {error:?}"),
    }
}
```

`cargo run --bin timeout_lies`:

```text
deposit failed: Timeout
balance after the 'failed' deposit: Ok(100)
deposit done, balance 200
```

**What happens:**

1. The first deposit waits only 100 ms, but the reply needs 300 ms. The client reports `Timeout`.
2. Yet the balance is 100. The "failed" deposit happened. If the client had shown "Payment failed, please try
   again" to a user, and the user had clicked again, the money would be counted twice.
3. The second deposit waits 500 ms, long enough, and sees the balance 200.

There are three possible outcomes of any remote call, not two:

```text
            ┌── Ok(answer)         → it happened, here is the result
 a call ────┼── definite error     → it surely did NOT happen (e.g. Connection refused)
            └── timeout / unknown  → it may have happened, may happen later, or never
```

Every design in the next lessons has to handle the third branch. Lesson 18.3 shows the standard answer:
make operations safe to repeat, then repeat them.

## 3. Testing it without waiting

A test with real 300 ms sleeps is slow, and on a busy machine it can fail by chance: a **flaky** test. Tokio's
paused clock (Lesson 14.14) solves both problems. Add a test module at the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test(start_paused = true)]
    async fn a_timed_out_deposit_can_still_happen() {
        let bank = spawn_bank(Duration::from_secs(10));
        let result = deposit(&bank, 100, Duration::from_secs(1)).await;
        assert_eq!(result, Err(CallError::Timeout));
        // The client gave up, but the money is there.
        assert_eq!(balance(&bank).await, Ok(100));
    }
}
```

`cargo test --lib`:

```text
test tests::a_timed_out_deposit_can_still_happen ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

A 10-second reply and a 1-second timeout, tested in 0.00 seconds. With the clock paused, Tokio jumps straight to
the next timer whenever all tasks are waiting, so the order of events is always the same.

> 🧪 **Testing:** Write the "timeout, but it happened" case as a test for every remote operation that changes
> data. It documents the danger, and Lesson 18.3 will turn it into a test that proves the retry is safe.

## 4. How long should you wait?

There is no perfect timeout. Each choice has a cost:

| Timeout | What goes wrong |
|---|---|
| Too short | Healthy but slightly slow nodes are treated as failed. Clients retry, which adds load, which makes nodes slower: a vicious circle. |
| Too long | Users wait for a dead node. Threads, connections and memory stay busy, and the client cannot do anything useful in the meantime. |

A practical rule is to **measure** the latencies of the real service and pick a timeout clearly above the slow
end, for example a few times the 99th percentile. If 99 % of calls finish within 40 ms, a timeout of 100 to
200 ms rarely cuts a healthy call but still notices a stuck one quickly. Exercises 1 and 3 build this.

When one request makes several calls, give the whole chain **one deadline**, and let each step use only the
time that is left (Lesson 14.14). If every step had its own fixed timeout, a chain of five steps could take
five times longer than the user is willing to wait. Exercise 2 builds such a chain and tests it.

> 🔒 **Security:** Timeouts also protect you from attacks. A client that connects and then sends one byte per
> minute can hold a connection, a buffer and maybe a thread forever. Every server needs read and write
> timeouts on its connections, not only every client.

## 5. Failure detectors: heartbeats

Instead of discovering a dead node only when a request times out, nodes often send **heartbeats**: small
"I am alive" messages, for example every 100 ms. A **failure detector** remembers when it last heard from
each node, and **suspects** every node that has been silent for longer than a limit.

Add these imports at the top of `src/lib.rs`, next to the others:

```rust
use std::collections::HashMap;
use std::time::Instant;
```

Then add the detector (before the test module):

```rust
/// Suspects a node when no heartbeat came from it for longer than `timeout`.
pub struct FailureDetector {
    timeout: Duration,
    last_seen: HashMap<String, Instant>,
}

impl FailureDetector {
    pub fn new(timeout: Duration) -> Self {
        FailureDetector {
            timeout,
            last_seen: HashMap::new(),
        }
    }

    /// Records that `node` said "I am alive" at the moment `now`.
    pub fn heartbeat(&mut self, node: &str, now: Instant) {
        self.last_seen.insert(node.to_string(), now);
    }

    /// The nodes that have been silent for too long at the moment `now`, sorted by name.
    pub fn suspects(&self, now: Instant) -> Vec<String> {
        let mut silent: Vec<String> = self
            .last_seen
            .iter()
            .filter(|(_, seen)| now.duration_since(**seen) > self.timeout)
            .map(|(node, _)| node.clone())
            .collect();
        silent.sort();
        silent
    }
}
```

The detector never reads the clock itself: the caller passes `now`. This small design choice makes it a
**pure** piece of logic that a test can drive with any times it likes. Add this test inside `mod tests`:

```rust
    #[test]
    fn a_silent_node_becomes_a_suspect() {
        let start = Instant::now();
        let ms = Duration::from_millis;
        let mut detector = FailureDetector::new(ms(250));
        detector.heartbeat("a", start);
        detector.heartbeat("b", start);
        detector.heartbeat("a", start + ms(200));

        assert!(detector.suspects(start + ms(250)).is_empty());
        assert_eq!(detector.suspects(start + ms(300)), vec!["b"]);
        assert_eq!(detector.suspects(start + ms(500)), vec!["a", "b"]);
    }
```

`let ms = Duration::from_millis;` stores the function itself in a variable, so `ms(250)` is a short way to write
`Duration::from_millis(250)`. At 250 ms nobody is suspected (the rule is "longer than", and `b` was silent for
exactly 250 ms). At 300 ms `b` is suspected; at 500 ms `a` too.

Now a live demonstration with three nodes. Node `a` is healthy, node `b` **freezes** for 400 ms and then
continues, node `c` **crashes** after 200 ms. Create `src/bin/heartbeats.rs`:

```rust
use std::time::Duration;

use p18_failures::FailureDetector;
use tokio::sync::mpsc;
use tokio::time::{Instant, sleep_until};

/// A node that sends a heartbeat at each of the given moments (in ms after `start`).
async fn node(name: &'static str, beats: Vec<u64>, tx: mpsc::Sender<&'static str>, start: Instant) {
    for ms in beats {
        sleep_until(start + Duration::from_millis(ms)).await;
        let _ = tx.send(name).await;
    }
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let (tx, mut rx) = mpsc::channel(64);
    let schedules = [
        ("a", (0..10).map(|i| i * 100).collect()), // always healthy
        ("b", vec![0, 100, 200, 300, 700, 800, 900]), // frozen from 300 to 700 ms
        ("c", vec![0, 100, 200]),                  // crashes after 200 ms
    ];
    for (name, beats) in schedules {
        tokio::spawn(node(name, beats, tx.clone(), start));
    }

    let mut detector = FailureDetector::new(Duration::from_millis(220));
    let mut suspected: Vec<String> = Vec::new();
    // The detector looks at the heartbeats every 100 ms, at 50, 150, 250, ... ms.
    let first_check = start + Duration::from_millis(50);
    let mut check = tokio::time::interval_at(first_check, Duration::from_millis(100));
    while start.elapsed() < Duration::from_millis(1000) {
        tokio::select! {
            Some(name) = rx.recv() => detector.heartbeat(name, Instant::now().into_std()),
            _ = check.tick() => {
                let now_suspected = detector.suspects(Instant::now().into_std());
                if now_suspected != suspected {
                    let t = start.elapsed().as_millis() / 10 * 10;
                    println!("t={t}ms: suspects {now_suspected:?}");
                    suspected = now_suspected;
                }
            }
        }
    }
}
```

`cargo run --bin heartbeats`:

```text
t=450ms: suspects ["c"]
t=550ms: suspects ["b", "c"]
t=750ms: suspects ["c"]
```

**What happens:**

1. Each node task sends its name into the channel at the moments in its schedule. `sleep_until` waits until a
   moment, not for a duration, so small delays do not add up.
2. The main loop uses `select!` (Lesson 14.14) to record every heartbeat that arrives and to look for suspects on
   every tick. `into_std()` turns Tokio's `Instant` into the `std` one that the detector uses. The time is
   rounded down to 10 ms, so tiny scheduling delays do not change the printed numbers.
3. Node `c` last spoke at 200 ms. At the check at 450 ms it had been silent for 250 ms, more than 220 ms:
   suspected. It never comes back.
4. Node `b` last spoke at 300 ms and is suspected at 550 ms. At 700 ms it speaks again, and at the next check it
   is no longer a suspect. The detector was **wrong** about `b`: it was only frozen.

This is the fundamental limit of failure detection. From the outside, a crashed node and a very slow node look
**the same** until the slow node speaks again. A detector with a short timeout notices crashes fast but makes
more false suspicions; a long timeout makes fewer mistakes but notices crashes later. Real systems pick a
balance, and the algorithms built on top of them (like Raft in Lesson 18.10) are designed to stay correct
even when the detector is wrong.

## Common mistakes

> ⚠️ **Common mistake: treating a timeout as "it did not happen".** As Section 2 showed, a timed-out operation
> may be completed. Never repeat a non-repeatable operation just because it timed out. Lesson 18.3 shows how to
> make repetition safe.

> ⚠️ **Common mistake: real sleeps in tests.** A test that sleeps 300 ms and expects a 100 ms timeout to fire
> will sometimes fail on a busy CI (Continuous Integration) server. Use `#[tokio::test(start_paused = true)]`,
> and keep the logic pure (pass `now` in) where you can, as the failure detector does.

## Try it yourself

1. In `timeout_lies.rs`, change the first limit from 100 ms to 250 ms and then to 350 ms. Find the border
   where the deposit changes from `Timeout` to success.
2. In `heartbeats.rs`, change the detector timeout from 220 ms to 450 ms. Run it: `c` is now suspected much
   later, and `b` is never suspected at all. You traded speed of detection for fewer mistakes.

## Exercises

### Exercise 1 — The 99th percentile, with tests (easy)

Create `src/latency.rs` (and add `pub mod latency;` to `src/lib.rs`) with
`pub fn percentile(samples: &[Duration], p: f64) -> Duration` using the **nearest-rank** method: sort the
samples, take rank `ceil(p / 100 × n)` (counting from 1). Write tests: for the samples 1 ms … 100 ms in any order,
p50 is 50 ms, p99 is 99 ms and p100 is 100 ms; for nine samples of 10 ms and one of 400 ms, p50 is 10 ms but
p99 is 400 ms.

<details>
<summary>Hint</summary>

`let mut sorted = samples.to_vec(); sorted.sort();` and then `sorted[rank - 1]`, because Rust indexes from 0.
Convert with `as f64` and back with `as usize` after `.ceil()`.

</details>

<details>
<summary>Answer</summary>

```rust
//! Exercises 1 and 3: percentiles and an adaptive timeout.

use std::time::Duration;

/// The `p`-th percentile (0 < p <= 100) of `samples`, by the nearest-rank method.
pub fn percentile(samples: &[Duration], p: f64) -> Duration {
    assert!(!samples.is_empty(), "no samples");
    assert!(p > 0.0 && p <= 100.0, "p must be in (0, 100]");
    let mut sorted = samples.to_vec();
    sorted.sort();
    let rank = (p / 100.0 * sorted.len() as f64).ceil() as usize;
    sorted[rank - 1]
}

#[cfg(test)]
mod tests {
    use super::*;

    fn ms(n: u64) -> Duration {
        Duration::from_millis(n)
    }

    #[test]
    fn percentiles_of_one_to_a_hundred() {
        let samples: Vec<Duration> = (1..=100).rev().map(ms).collect();
        assert_eq!(percentile(&samples, 50.0), ms(50));
        assert_eq!(percentile(&samples, 99.0), ms(99));
        assert_eq!(percentile(&samples, 100.0), ms(100));
    }

    #[test]
    fn one_slow_sample_out_of_ten_shows_in_p99() {
        let mut samples = vec![ms(10); 9];
        samples.push(ms(400));
        assert_eq!(percentile(&samples, 50.0), ms(10));
        assert_eq!(percentile(&samples, 99.0), ms(400));
    }
}
```

Why it works: the second test shows why engineers look at p99 and not at the average or the median: one slow
call in ten is invisible in p50 but decides the timeout.

</details>

### Exercise 2 — One deadline for two calls (medium)

Write an integration test `tests/deadline.rs` with a helper
`async fn deposit_twice(bank, amount, total: Duration) -> Vec<Result<u64, CallError>>` that makes two deposits
and gives **both together** at most `total`. Test it with a paused clock: with a reply delay of 600 ms and a total
of 1 s, the first deposit succeeds (balance 10) and the second times out — and the balance afterwards is 20.

<details>
<summary>Hint</summary>

Compute `let deadline = Instant::now() + total;` once (use `tokio::time::Instant`, which follows the paused
clock). Before each call, `deadline.saturating_duration_since(Instant::now())` is the time that is left; pass it
as the `limit`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::Duration;

use p18_failures::{CallError, Command, balance, deposit, spawn_bank};
use tokio::sync::mpsc;
use tokio::time::Instant;

/// Deposits `amount` twice, with one deadline for both calls together.
async fn deposit_twice(
    bank: &mpsc::Sender<Command>,
    amount: u64,
    total: Duration,
) -> Vec<Result<u64, CallError>> {
    let deadline = Instant::now() + total;
    let mut results = Vec::new();
    for _ in 0..2 {
        // Each call gets only the time that is left before the deadline.
        let left = deadline.saturating_duration_since(Instant::now());
        results.push(deposit(bank, amount, left).await);
    }
    results
}

#[tokio::test(start_paused = true)]
async fn the_second_call_runs_out_of_time() {
    let bank = spawn_bank(Duration::from_millis(600));
    let results = deposit_twice(&bank, 10, Duration::from_secs(1)).await;
    assert_eq!(results, vec![Ok(10), Err(CallError::Timeout)]);
    // The second deposit timed out, but it was still applied.
    assert_eq!(balance(&bank).await, Ok(20));
}
```

`cargo test --test deadline`:

```text
test the_second_call_runs_out_of_time ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the first call uses 600 ms, so only 400 ms are left for the second, which needs 600 ms. With a
separate 1-second timeout per call, both would have succeeded but the chain would have taken 1.2 s.
`saturating_duration_since` returns zero instead of panicking when the deadline has already passed.

</details>

### Exercise 3 — A timeout that learns (hard)

Add to `src/latency.rs` a function `pub fn adaptive_timeout(recent: &[Duration], minimum: Duration) -> Duration`:
twice the p99 of the recent latencies, but never less than `minimum`, and exactly `minimum` when there are no
measurements yet. Write a test with three cases: no samples; samples of 10 and 12 ms (the result is the minimum
of 50 ms); samples of 10 and 100 ms (the result is 200 ms). Then think: where in a real client would the
`recent` samples come from?

<details>
<summary>Hint</summary>

Reuse `percentile` from Exercise 1. `Duration` can be multiplied by an integer (`d * 2`) and compared with
`.max(other)`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Twice the p99 of the recent latencies, but never below `minimum`.
pub fn adaptive_timeout(recent: &[Duration], minimum: Duration) -> Duration {
    if recent.is_empty() {
        return minimum;
    }
    (percentile(recent, 99.0) * 2).max(minimum)
}
```

And inside `mod tests`:

```rust
    #[test]
    fn the_timeout_follows_the_latencies() {
        let minimum = ms(50);
        assert_eq!(adaptive_timeout(&[], minimum), ms(50));
        assert_eq!(adaptive_timeout(&[ms(10), ms(12)], minimum), ms(50));
        assert_eq!(adaptive_timeout(&[ms(10), ms(100)], minimum), ms(200));
    }
```

`cargo test --lib`:

```text
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the timeout grows when the service becomes slow and shrinks when it recovers; the minimum protects
against a timeout so short that normal jitter would trigger it. In a real client, `recent` would be a window of
the last few hundred measured call durations, for example kept in a `VecDeque` (a double-ended queue,
[Lesson 07.5](../07-algorithms-and-data-structures/05-stacks-and-queues.md)) that drops the oldest sample when
it is full.

</details>

## Quiz

1. What is the difference between crash-stop and crash-recovery?

<details>
<summary>Answer</summary>

A crash-stop node never comes back. A crash-recovery node restarts later; it loses its memory but keeps what it
had durably saved on disk.

</details>

2. The client's deposit timed out. Name the three groups of possible outcomes of any remote call.

<details>
<summary>Answer</summary>

Success with an answer; a definite error (it surely did not happen, e.g. the connection was refused); and an
unknown outcome (timeout): it may have happened, may happen later, or may never happen.

</details>

3. Why did the test with a 10-second delay finish in 0.00 seconds?

<details>
<summary>Answer</summary>

`start_paused = true` freezes Tokio's clock. When every task is waiting for a timer, Tokio jumps the clock
forward to the next timer at once, so no real time passes.

</details>

4. A request calls three services in a row, each with a 1-second timeout. What is the worst-case wait, and how
   does a deadline help?

<details>
<summary>Answer</summary>

Up to 3 seconds (plus processing). With one deadline of, say, 1 second for the whole request, each call gets only
the time that is left, so the whole request never takes longer than 1 second.

</details>

5. The failure detector suspected node `b`, but `b` was alive. Is the detector broken?

<details>
<summary>Answer</summary>

No. A failure detector can only suspect: from outside, a silent node and a slow node look the same. Every
detector trades fast detection against false suspicions, and the algorithms on top must stay correct when it is
wrong.

</details>

## Summary

- Name failures precisely: crash-stop, crash-recovery, omission, partition, gray failure and Byzantine. This
  phase assumes crash-recovery nodes on a network that loses, delays and reorders messages.
- A remote call has three outcomes: success, definite failure and **unknown**. A timeout always means unknown.
- A timed-out operation can still complete; handling that case is the job of the next lesson.
- Test time-based code with Tokio's paused clock, and keep time-based logic pure by passing `now` in.
- Choose timeouts from measured latencies (a few times p99), and give a chain of calls one deadline.
- Heartbeats and a failure detector find silent nodes, but a detector can only suspect: slow and dead look alike.

## Sources and further reading

- [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — the Amazon Builders' Library on how to choose timeouts from latency percentiles in real services.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — the chapter "The Trouble with Distributed Systems" discusses unreliable networks, timeouts and failure detection.
- [`tokio::time::timeout`](https://docs.rs/tokio/latest/tokio/time/fn.timeout.html) — the API (Application Programming Interface) used in this lesson.
- [`tokio::time::pause`](https://docs.rs/tokio/latest/tokio/time/fn.pause.html) — how the paused clock in tests works.
- [Implementing health checks](https://aws.amazon.com/builders-library/implementing-health-checks/) — the Amazon Builders' Library on heartbeats, health checks and gray failures.

---

[🏠 Start](../START-HERE.md) · [← 18.1 What a distributed system is](01-what-a-distributed-system-is.md) · [📋 Phase 18](README.md) · [18.3 Retries, backoff and idempotency →](03-retries-backoff-and-idempotency.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
