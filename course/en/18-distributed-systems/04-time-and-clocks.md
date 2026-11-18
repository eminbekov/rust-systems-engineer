<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.3 Retries, backoff and idempotency](03-retries-backoff-and-idempotency.md) · [📋 Phase 18](README.md) · [18.5 Logical clocks: Lamport and vector clocks →](05-logical-clocks.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/04-time-and-clocks.md) <!-- nav -->

# Lesson 18.4 — Time and clocks

> **Phase 18 — Distributed systems** · Lesson 4 of 15 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The two clocks in every computer, the wall clock and the monotonic clock, and which one to use for what.
- Why the wall clock can jump backwards, and how Rust's `SystemTime` and `Instant` make that visible.
- Why the clocks of two machines never agree exactly, and how much they drift.
- How "last write wins" with timestamps silently loses data when clocks disagree.
- Why a lock with a timeout (a lease) is not safe on its own, and how fencing tokens fix it.

## Before you start

- You finished [Lesson 18.3 — Retries, backoff and idempotency](03-retries-backoff-and-idempotency.md).
- You know `Duration` and `Instant` from earlier phases and the frozen node from [Lesson 18.1](01-what-a-distributed-system-is.md).
- Create the package for this lesson. It needs no dependencies:

```bash
cd ~/projects/rust
cargo new --lib code/p18-clocks
cd code/p18-clocks
mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **wall clock** | The clock that tells the time of day and the date, like the clock on a wall. In Rust: `SystemTime`. It can be changed and can jump. |
| **monotonic clock** | A clock that only moves forward, at a steady rate, from some arbitrary starting point. In Rust: `Instant`. Good for measuring durations; meaningless on another machine. |
| **UTC** (Coordinated Universal Time) | The world's reference time, without time zones or summer time. |
| **Unix epoch** | The moment 1970-01-01 00:00:00 UTC. Wall-clock times are often stored as "seconds since the epoch". |
| **NTP** (Network Time Protocol) | The protocol that computers use to set their wall clocks from time servers over the network. |
| **clock drift** | A clock running slightly too fast or too slow. Measured in **ppm** (parts per million): 1 ppm is 1 µs of error per second. |
| **clock skew** | The difference between the clocks of two machines at the same moment. |
| **LWW** (Last Write Wins) | A rule for conflicts: keep the value with the latest timestamp, throw the others away. |
| **lease** | A lock that is valid only for a limited time, so that it frees itself if its holder dies. |
| **fencing token** | A number that grows with every new lease. The storage refuses writes that carry an older number than one it has already seen. |

## 1. Two clocks in every computer

Every computer offers two very different clocks:

| | Wall clock (`SystemTime`) | Monotonic clock (`Instant`) |
|---|---|---|
| Answers | "What time and date is it?" | "How much time has passed since that moment?" |
| Set by | NTP, the administrator, the user | nobody: it just counts |
| Can jump | yes, forwards and **backwards** | no, never backwards |
| Comparable between machines | roughly (if both use NTP) | **no**: each machine starts it at a different point |
| Use it for | timestamps for humans, logs, expiry dates | timeouts, measuring durations, rate limits |

Let us read both, `src/bin/two_clocks.rs`:

```rust
use std::thread::sleep;
use std::time::{Duration, Instant, SystemTime, UNIX_EPOCH};

fn main() {
    let wall_start = SystemTime::now();
    let mono_start = Instant::now();
    sleep(Duration::from_millis(250));
    println!("wall clock:      {:?}", wall_start.elapsed());
    println!("monotonic clock: {:?}", mono_start.elapsed());

    let since_epoch = wall_start.duration_since(UNIX_EPOCH).unwrap();
    println!("seconds since 1970-01-01 00:00 UTC: {}", since_epoch.as_secs());
}
```

`cargo run --bin two_clocks` (your numbers will differ):

```text
wall clock:      Ok(255.015ms)
monotonic clock: 255.109292ms
seconds since 1970-01-01 00:00 UTC: 1794990293
```

**What happens:**

1. Both clocks measured about 250 ms (plus a few milliseconds for the program to wake up). On a quiet machine they
   agree.
2. But look at the types. `SystemTime::elapsed()` returns a `Result`: Rust reminds you that the wall clock may
   have moved backwards in the meantime, and then there is no positive duration to return. `Instant::elapsed()`
   returns a plain `Duration`, because the monotonic clock cannot go backwards.

## 2. When the wall clock jumps

The wall clock jumps more often than you might think:

- NTP finds that the clock is 2 seconds fast and **steps** it back. (For small errors NTP prefers to **slew**:
  to run the clock a little slower until it is right. For large errors it steps.)
- An administrator or the user changes the time by hand.
- A virtual machine is paused, moved to another host and resumed; its clock is then suddenly far behind.
- A **leap second**: once in a while, UTC gets an extra second to stay in line with the Earth's rotation, and
  systems handle it in different ways, some by repeating a second.

Rust lets you see such a jump without changing your Mac's clock. We build two wall-clock readings by hand,
`src/bin/backwards.rs`:

```rust
use std::time::{Duration, Instant, UNIX_EPOCH};

fn main() {
    // Pretend the time service moved the wall clock 2 seconds back between two readings.
    let first = UNIX_EPOCH + Duration::from_secs(1_800_000_000);
    let second = first - Duration::from_secs(2);
    println!("wall: second - first = {:?}", second.duration_since(first));

    // An Instant can never be earlier than one read before it. If you subtract the
    // wrong way round, you get zero instead of a negative duration.
    let a = Instant::now();
    let b = Instant::now();
    println!("monotonic: b - a = {:?}", b.duration_since(a));
    println!("monotonic: a - b = {:?}", a.duration_since(b));
}
```

`cargo run --bin backwards`:

```text
wall: second - first = Err(SystemTimeError(2s))
monotonic: b - a = 42ns
monotonic: a - b = 0ns
```

**What happens:**

1. `UNIX_EPOCH + Duration::from_secs(1_800_000_000)` is a `SystemTime` in the year 2027; `second` is 2 seconds
   *earlier*, as if the clock had been stepped back between the two readings.
2. `second.duration_since(first)` cannot give a negative `Duration` (a `Duration` is never negative), so it returns
   `Err(SystemTimeError(2s))`: "the other time is 2 s later than this one".
3. For `Instant`, `duration_since` with the arguments the wrong way round returns zero instead of an error. Two
   readings of the monotonic clock one after another are a few dozen nanoseconds apart.

The rule that follows: **measure durations, timeouts and deadlines with `Instant`, never with `SystemTime`.**
Tokio's timers (Lessons 18.2 and 18.3) already use a monotonic clock for this reason.

## 3. Clocks drift, and machines disagree

The clock in a computer is a small quartz crystal that vibrates at a known frequency. No two crystals are exactly
alike, and their speed changes with temperature. A typical computer clock is off by 10 to 100 ppm. At 50 ppm a
clock gains or loses about 4.3 seconds per day (Exercise 1 calculates this).

NTP corrects this by asking time servers and adjusting the clock regularly. But NTP itself works over the network
(Lesson 18.2: messages are delayed by unknown amounts), so the result is never perfect. Typical numbers:

| Setup | Typical skew between two machines |
|---|---|
| NTP over the internet | a few to tens of milliseconds |
| NTP inside one data centre | about a millisecond or less |
| NTP broken or blocked by a firewall | grows without limit, seconds per day |
| Special hardware (GPS, Global Positioning System, receivers and atomic clocks) | microseconds |

The important consequence: **a timestamp from machine A and a timestamp from machine B cannot be compared
exactly.** If A says 12:00:00.005 and B says 12:00:00.003, B's event may well have happened *after* A's.

## 4. "Last write wins" loses writes

Many replicated databases resolve conflicts with **LWW** (Last Write Wins): every write carries the writer's
wall-clock timestamp, and when two replicas disagree, the value with the larger timestamp wins. It is simple,
and it silently loses data when clocks disagree. Let us see it. Put this into `src/lib.rs`:

```rust
//! Time and clocks (Lesson 18.4).

/// A register that keeps the value with the highest timestamp: "last write wins".
#[derive(Debug, Default)]
pub struct LwwRegister {
    value: Option<String>,
    timestamp_ms: u64,
}

impl LwwRegister {
    /// Stores `value` if its timestamp is newer than the stored one. Returns whether it was stored.
    pub fn write(&mut self, value: &str, timestamp_ms: u64) -> bool {
        if self.value.is_some() && timestamp_ms <= self.timestamp_ms {
            return false;
        }
        self.value = Some(value.to_string());
        self.timestamp_ms = timestamp_ms;
        true
    }

    pub fn read(&self) -> Option<&str> {
        self.value.as_deref()
    }
}

/// The clock of one machine: the real time plus this machine's error.
pub struct NodeClock {
    pub offset_ms: i64,
}

impl NodeClock {
    pub fn now_ms(&self, real_ms: u64) -> u64 {
        real_ms.saturating_add_signed(self.offset_ms)
    }
}
```

`NodeClock` simulates a machine whose clock is `offset_ms` ahead (positive) or behind (negative).
`saturating_add_signed` adds an `i64` to a `u64` without overflowing. Now two nodes write to the same key,
`src/bin/lost_update.rs`:

```rust
use p18_clocks::{LwwRegister, NodeClock};

fn main() {
    let a = NodeClock { offset_ms: 100 }; // node a's clock is 100 ms fast
    let b = NodeClock { offset_ms: 0 }; // node b's clock is right
    let mut x = LwwRegister::default();

    let ts = a.now_ms(1000);
    println!(
        "real 1000 ms: a writes \"1\" with timestamp {ts}: stored {}",
        x.write("1", ts)
    );
    let ts = b.now_ms(1050);
    println!(
        "real 1050 ms: b writes \"2\" with timestamp {ts}: stored {}",
        x.write("2", ts)
    );
    println!("x = {:?}", x.read());
}
```

`cargo run --bin lost_update`:

```text
real 1000 ms: a writes "1" with timestamp 1100: stored true
real 1050 ms: b writes "2" with timestamp 1050: stored false
x = Some("1")
```

Node `b` wrote 50 ms **after** node `a` in real time, perhaps after reading `a`'s value and deciding to change it.
But `a`'s clock is 100 ms fast, so its old write carries the larger timestamp, and `b`'s newer write is thrown away.
No error, no warning: the data is simply gone. With 100 ms of skew, every write that comes less than 100 ms after
a write from the fast node can be lost.

> 🧪 **Testing:** Bugs like this never show up on one machine, where all writes use the same clock. Simulate the
> clocks, as `NodeClock` does, and test with skew. Add to `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_fast_clock_hides_a_later_write() {
        let fast = NodeClock { offset_ms: 100 };
        let correct = NodeClock { offset_ms: 0 };
        let mut x = LwwRegister::default();
        assert!(x.write("1", fast.now_ms(1000))); // real time 1000 ms
        assert!(!x.write("2", correct.now_ms(1050))); // real time 1050 ms: later!
        assert_eq!(x.read(), Some("1"));
    }
}
```

LWW is still used where losing a concurrent write is acceptable, for example for a "last seen online" field. For
anything important, systems use **logical clocks** (next lesson) or **consensus** (Lessons 18.9 to 18.11) to decide
the order of events, not wall clocks.

## 5. Leases and fencing tokens

A distributed lock is often built as a **lease**: "client 1 holds the lock for the next 10 seconds". If client 1
crashes, the lease simply runs out, and nobody waits forever. The danger is a client that is **paused**, not
crashed (Lesson 18.1): it does not know that time passed.

```text
time   lock service             client 1                      client 2
 0s    lease to 1 until 10s     "I have the lock"
 1s                             ...frozen (overloaded machine)...
12s    lease ran out;
       lease to 2 until 22s                                   "I have the lock", writes
15s                             wakes up: "I still have the lock", writes  ← two writers!
```

Client 1 cannot protect itself: from its own point of view, no time passed between "I have the lock" and "I write".
Checking the clock just before writing does not help either, because the pause can happen between the check and
the write.

The fix must be in the **storage**: every lease comes with a **fencing token**, a number that grows with every
new lease, and the storage refuses any write with a token older than the newest one it has seen. Add to
`src/lib.rs` (before the test module):

```rust
/// A lock service that gives out leases. Every new lease gets a larger fencing token.
#[derive(Default)]
pub struct LockService {
    last_token: u64,
    lease_ends_at: u64,
}

impl LockService {
    /// Gives a lease of `lease` seconds if nobody holds a valid one at time `now`.
    pub fn acquire(&mut self, now: u64, lease: u64) -> Option<u64> {
        if now < self.lease_ends_at {
            return None; // somebody else still holds the lock
        }
        self.last_token += 1;
        self.lease_ends_at = now + lease;
        Some(self.last_token)
    }
}

/// Storage that refuses writes with an older token than the newest it has seen.
#[derive(Default)]
pub struct FencedStorage {
    highest_token: u64,
    pub log: Vec<String>,
}

impl FencedStorage {
    pub fn write(&mut self, token: u64, text: &str) -> Result<(), String> {
        if token < self.highest_token {
            return Err(format!(
                "token {token} is older than {}",
                self.highest_token
            ));
        }
        self.highest_token = token;
        self.log.push(text.to_string());
        Ok(())
    }
}
```

And the story from the diagram, `src/bin/fencing.rs`:

```rust
use p18_clocks::{FencedStorage, LockService};

fn main() {
    let mut locks = LockService::default();
    let mut storage = FencedStorage::default();

    let token1 = locks.acquire(0, 10).unwrap();
    println!("t=0s:  client 1 gets the lock for 10 s, token {token1}");
    println!("t=1s:  client 1 freezes (its machine is overloaded)");
    let token2 = locks.acquire(12, 10).unwrap();
    println!("t=12s: the lease has run out; client 2 gets the lock, token {token2}");
    println!(
        "t=13s: client 2 writes: {:?}",
        storage.write(token2, "from client 2")
    );
    println!("t=15s: client 1 wakes up, still thinks it holds the lock, and writes:");
    println!("       {:?}", storage.write(token1, "from client 1"));
    println!("stored: {:?}", storage.log);
}
```

`cargo run --bin fencing`:

```text
t=0s:  client 1 gets the lock for 10 s, token 1
t=1s:  client 1 freezes (its machine is overloaded)
t=12s: the lease has run out; client 2 gets the lock, token 2
t=13s: client 2 writes: Ok(())
t=15s: client 1 wakes up, still thinks it holds the lock, and writes:
       Err("token 1 is older than 2")
stored: ["from client 2"]
```

**What happens:**

1. The lock service and the storage use only the token, never a clock, to decide who may write. The lease time is
   used only to decide when a **new** lease may be given out.
2. Client 2's write raised the storage's `highest_token` to 2. When the woken-up client 1 arrives with token 1, the
   storage knows that a newer lock holder exists and refuses.

The same idea appears again in Raft: its **term** number (Lesson 18.10) is a fencing token for leaders.

Add this test to `mod tests`:

```rust
    #[test]
    fn a_paused_client_cannot_overwrite_newer_data() {
        let mut locks = LockService::default();
        let mut storage = FencedStorage::default();
        let token1 = locks.acquire(0, 10).unwrap();
        assert_eq!(locks.acquire(5, 10), None); // the lease of client 1 is still valid
        let token2 = locks.acquire(12, 10).unwrap(); // client 1's lease ran out
        assert_eq!(storage.write(token2, "from client 2"), Ok(()));
        assert!(storage.write(token1, "from client 1").is_err());
        assert_eq!(storage.log, vec!["from client 2"]);
    }
```

`cargo test --lib`:

```text
test tests::a_fast_clock_hides_a_later_write ... ok
test tests::a_paused_client_cannot_overwrite_newer_data ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 6. Rules for time in distributed systems

| Task | Use |
|---|---|
| Timeouts, deadlines, measuring how long something took | the monotonic clock (`Instant`, `tokio::time::Instant`) |
| A timestamp that a human reads (logs, "created at") | the wall clock in UTC (`SystemTime`) |
| Deciding which of two events on **different** machines came first | not the wall clock: logical clocks (Lesson 18.5) or consensus (Lessons 18.9–18.11) |
| Making sure only one node acts as the owner of something | a lease **plus** fencing tokens checked by the storage |
| Expiry that may be a little late or early (cache entries, sessions) | the wall clock, with a safety margin larger than the expected skew |

## Common mistakes

**Unwrapping a wall-clock difference.** `later.duration_since(earlier).unwrap()` looks harmless and works for months,
until NTP steps the clock back (`src/bin/unwrap_jump.rs`):

```rust
use std::time::{Duration, UNIX_EPOCH};

fn main() {
    let first = UNIX_EPOCH + Duration::from_secs(1_800_000_000);
    let second = first - Duration::from_secs(2); // the wall clock jumped back
    let took = second.duration_since(first).unwrap();
    println!("took {took:?}");
}
```

```text
thread 'main' (6295495) panicked at src/bin/unwrap_jump.rs:6:45:
called `Result::unwrap()` on an `Err` value: SystemTimeError(2s)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

(The number after `'main'` identifies the thread and is different on every run.) The fix: measure durations with
`Instant`. If you really must subtract two wall-clock times, handle the error, for example with
`.unwrap_or(Duration::ZERO)`.

## Try it yourself

1. In `lost_update.rs`, change node `a`'s offset to `-100` (100 ms slow). Now `b`'s write wins, as it should.
   Then also change `b`'s real time from `1050` to `950` in both places, so that `b` writes 50 ms **before** `a`
   (its message just arrives later). Run it: `x` is still `"2"`. This time the slow clock made the **older** write
   win, and `a`'s newer write is the one that is lost.

## Exercises

### Exercise 1 — How far does a clock drift? (easy)

Create `src/drift.rs` (and add `pub mod drift;` to `src/lib.rs`) with
`pub fn drift_per_day(ppm: f64) -> Duration` and `pub fn max_sync_interval(ppm: f64, allowed: Duration) -> Duration`:
how often a clock must be corrected so that its error never grows beyond `allowed`. Write tests: 50 ppm drifts
4,320 ms per day, 1 ppm drifts 86,400 µs per day, and with 50 ppm and 10 ms allowed, the clock must be corrected at
least every 200 s.

<details>
<summary>Hint</summary>

A day has 24 × 60 × 60 seconds; the error is `seconds × ppm / 1,000,000`. For the interval, turn the formula
around: `allowed_seconds × 1,000,000 / ppm`. `Duration::from_secs_f64` builds a `Duration` from an `f64`.

</details>

<details>
<summary>Answer</summary>

```rust
//! Exercises 1 and 3.

use std::time::Duration;

/// How far a clock that is wrong by `ppm` parts per million drifts in one day.
pub fn drift_per_day(ppm: f64) -> Duration {
    Duration::from_secs_f64(24.0 * 60.0 * 60.0 * ppm / 1_000_000.0)
}

/// How often a clock must be corrected so that it never drifts more than `allowed`.
pub fn max_sync_interval(ppm: f64, allowed: Duration) -> Duration {
    Duration::from_secs_f64(allowed.as_secs_f64() * 1_000_000.0 / ppm)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fifty_ppm_is_more_than_four_seconds_a_day() {
        assert_eq!(drift_per_day(50.0).as_millis(), 4320);
        assert_eq!(drift_per_day(1.0).as_micros(), 86_400);
    }

    #[test]
    fn sync_often_enough_for_ten_milliseconds() {
        let interval = max_sync_interval(50.0, Duration::from_millis(10));
        assert_eq!(interval.as_secs(), 200);
    }
}
```

Why it works: 86,400 s × 50 / 1,000,000 = 4.32 s. The second function explains why NTP clients check the time every
few minutes, not once a day: a cheap clock needs it to stay within milliseconds.

</details>

### Exercise 2 — Last write wins, without ties (medium)

Two writes can carry the **same** millisecond. Then `LwwRegister` keeps whichever arrived first, so two replicas
that receive the writes in different orders end up with different values. Create `src/stamp.rs` (and
`pub mod stamp;`) with a `Stamp { ms: u64, node: u32 }` that is compared by `ms` first and by `node` on a tie, and a
`Register` that keeps the value with the largest stamp. Write a test: two writes with the same `ms` from nodes 1 and
2, applied in both orders, give equal registers holding the value from node 2.

<details>
<summary>Hint</summary>

`#[derive(PartialEq, Eq, PartialOrd, Ord)]` on a struct compares the fields in the order they are written, like a
dictionary compares words letter by letter. So put `ms` first.

</details>

<details>
<summary>Answer</summary>

```rust
//! Exercise 2: last write wins, with a tie-breaker.

/// A timestamp with the writer's node id as a tie-breaker.
/// Deriving `Ord` compares `ms` first and `node` only when `ms` is equal.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct Stamp {
    pub ms: u64,
    pub node: u32,
}

#[derive(Debug, Default, PartialEq)]
pub struct Register {
    current: Option<(Stamp, String)>,
}

impl Register {
    pub fn write(&mut self, value: &str, stamp: Stamp) {
        let newer = match &self.current {
            None => true,
            Some((old, _)) => stamp > *old,
        };
        if newer {
            self.current = Some((stamp, value.to_string()));
        }
    }

    pub fn read(&self) -> Option<&str> {
        self.current.as_ref().map(|(_, value)| value.as_str())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn the_order_of_arrival_does_not_matter() {
        let w1 = ("from node 1", Stamp { ms: 500, node: 1 });
        let w2 = ("from node 2", Stamp { ms: 500, node: 2 }); // the same millisecond

        let mut one_way = Register::default();
        one_way.write(w1.0, w1.1);
        one_way.write(w2.0, w2.1);

        let mut other_way = Register::default();
        other_way.write(w2.0, w2.1);
        other_way.write(w1.0, w1.1);

        assert_eq!(one_way, other_way);
        assert_eq!(one_way.read(), Some("from node 2"));
    }
}
```

Why it works: with the tie-breaker every two stamps from different nodes are different, so all replicas pick the
same winner whatever the order of arrival. Replicas now **agree** — but the clock-skew problem from Section 4 is
still there: they may agree on the wrong value.

</details>

### Exercise 3 — Time with error bars (hard)

If you know that a clock is wrong by at most `max_error` ms, a reading of `t` really means "somewhere between
`t - max_error` and `t + max_error`". Add to `src/drift.rs` a type `Interval { earliest_ms, latest_ms }` with
`Interval::around(reading_ms, max_error_ms)` and `surely_before(&self, other: &Interval) -> bool`, which is true
only when `self` happened before `other` **whatever** the real errors were. Test three readings with an error of
5 ms: 1000, 1008 and 1020. Which pairs have a certain order?

<details>
<summary>Hint</summary>

Event A is surely before B only if even the latest possible time of A is earlier than the earliest possible time of
B. If the two intervals overlap, the order is unknown in both directions.

</details>

<details>
<summary>Answer</summary>

```rust
/// A reading of a clock with a known maximum error: the real time is somewhere in between.
#[derive(Debug, Clone, Copy)]
pub struct Interval {
    pub earliest_ms: u64,
    pub latest_ms: u64,
}

impl Interval {
    /// A reading `reading_ms` from a clock that may be wrong by up to `max_error_ms`.
    pub fn around(reading_ms: u64, max_error_ms: u64) -> Self {
        Interval {
            earliest_ms: reading_ms.saturating_sub(max_error_ms),
            latest_ms: reading_ms + max_error_ms,
        }
    }

    /// True only if `self` certainly happened before `other`, whatever the real errors were.
    pub fn surely_before(&self, other: &Interval) -> bool {
        self.latest_ms < other.earliest_ms
    }
}
```

And in the test module of `src/drift.rs`:

```rust
    #[test]
    fn overlapping_readings_have_no_known_order() {
        let a = Interval::around(1000, 5); // 995..=1005
        let b = Interval::around(1008, 5); // 1003..=1013: overlaps with a
        let c = Interval::around(1020, 5); // 1015..=1025
        assert!(!a.surely_before(&b));
        assert!(!b.surely_before(&a));
        assert!(a.surely_before(&c));
        assert!(!c.surely_before(&a));
    }
```

`cargo test --lib`:

```text
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: 1000 and 1008 are only 8 ms apart, less than twice the error, so either could be first. 1000 and 1020
are 20 ms apart, more than twice the error, so the order is certain. Google's Spanner database builds on this idea:
it waits out the error interval before it reports a commit.

</details>

## Quiz

1. You need to stop an operation after 5 seconds. Which clock do you use, and why?

<details>
<summary>Answer</summary>

The monotonic clock (`Instant`, or Tokio's timers). The wall clock can be stepped forwards or backwards while you
wait, which would make the timeout fire too early or much too late.

</details>

2. Node A's clock is 30 ms fast. Node A writes at real time 0 ms, node B (correct clock) writes at real time 20 ms,
   with last write wins. Which value survives?

<details>
<summary>Answer</summary>

A's: its timestamp is 30 ms, B's is 20 ms. B's later write is lost.

</details>

3. Why can a client not protect itself from writing with an expired lease by checking the time just before it
   writes?

<details>
<summary>Answer</summary>

The pause can happen right after the check and before the write reaches the storage (or the write can be delayed in
the network). The client cannot see its own pause. Only the storage, which compares fencing tokens, can refuse the
stale write.

</details>

4. What is a fencing token, and who checks it?

<details>
<summary>Answer</summary>

A number that grows with every new lease. The lock holder sends it with every write, and the **storage** rejects
writes whose token is older than the newest token it has already seen.

</details>

## Summary

- Every computer has a wall clock (`SystemTime`: time of day, can jump) and a monotonic clock (`Instant`: only
  forward, local to one machine).
- Use `Instant` for timeouts and durations, `SystemTime` in UTC for human-readable timestamps.
- Clocks drift by tens of ppm, and NTP keeps machines within milliseconds of each other at best. Timestamps from
  different machines cannot be compared exactly.
- Last write wins with wall-clock timestamps silently loses writes when clocks are skewed.
- A lease is a promise about time that a paused process can break without noticing. Fencing tokens, checked by the
  storage, make leases safe.
- To order events across machines, use logical clocks (next lesson) or consensus, not wall clocks.

## Sources and further reading

- [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — Martin Kleppmann's essay on leases, process pauses and fencing tokens; the timeline in Section 5 follows its argument.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — the chapter "The Trouble with Distributed Systems" covers unreliable clocks, last write wins and process pauses.
- [`std::time::Instant`](https://doc.rust-lang.org/std/time/struct.Instant.html) and [`std::time::SystemTime`](https://doc.rust-lang.org/std/time/struct.SystemTime.html) — the documentation explains exactly what each clock guarantees on each operating system.
- [RFC 5905 — Network Time Protocol Version 4](https://datatracker.ietf.org/doc/html/rfc5905) — the NTP standard (an RFC, Request for Comments, is a document that defines an internet standard), for readers who want to see how clock offset and network delay are estimated.

---

[🏠 Start](../START-HERE.md) · [← 18.3 Retries, backoff and idempotency](03-retries-backoff-and-idempotency.md) · [📋 Phase 18](README.md) · [18.5 Logical clocks: Lamport and vector clocks →](05-logical-clocks.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
