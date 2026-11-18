<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.8 Lock-free data structures](08-lock-free-data-structures.md) · [📋 Phase 14](README.md) · [14.10 How futures work: Future, poll and Waker →](10-how-futures-work.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) <!-- nav -->

# Lesson 14.9 — Testing concurrent code with loom

> **Phase 14 — Concurrency and async in depth** · Lesson 9 of 16 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Why a normal test, even one repeated thousands of times, can pass while concurrent code is wrong.
- What a model checker is, and how the `loom` crate runs one test under every possible thread schedule.
- How to prepare a crate for loom: a `cfg(loom)` flag, a `sync` module and a separate test file.
- How loom finds a lost update, a wrong memory ordering, a spin loop that never ends and a deadlock.
- How to keep loom models small, and what loom cannot check.

## Before you start

- You finished [Lesson 14.8 — Lock-free data structures](08-lock-free-data-structures.md).
- You know atomics and `fetch_add` from [Lesson 14.5](05-atomics.md), and `Relaxed`, `Release` and `Acquire` from [Lesson 14.6 — Memory ordering](06-memory-ordering.md).
- You know spin loops from [Lesson 14.7 — Building a spin lock](07-building-a-spin-lock.md) and deadlocks from [Lesson 14.1](01-why-concurrency-is-hard.md).
- You know `Arc`, `thread::spawn` and `join` from [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), and integration tests in `tests/` from [Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md).
- Your code for this lesson goes in `~/projects/rust/code/p14-loom`.

## New words

| Word | Meaning |
|---|---|
| **interleaving** | One possible order in which the steps of several threads happen. Two threads with two steps each already have six interleavings. |
| **schedule** | The decisions "which thread runs next" that produce one interleaving. The operating system's scheduler makes them, and you cannot control them. |
| **lost update** | A bug where two threads read the same old value, both write a new one, and one of the writes is lost. |
| **stress test** | A test that runs concurrent code many times, hoping that a rare bad schedule happens at least once. |
| **model checker** | A tool that does not hope: it tries every possible schedule (or every schedule inside a limit) and checks your assertions in each one. |
| **loom** | A model checker for concurrent Rust code, made by the Tokio project. Its types replace `std` atomics, locks and threads during a test. |
| **model** (`loom::model`) | The closure that loom runs again and again, once per schedule. Each run is called an *iteration*. |
| **deterministic** | Gives the same result every time for the same input. Loom tests must be deterministic: no randomness, no clock, no real I/O (input/output: files, network). |
| **preemption** | The scheduler stops a running thread in the middle of its work and runs another one. |
| **preemption bound** | A limit on how many preemptions loom tries in one schedule. A smaller bound means fewer iterations. |
| **state explosion** | The number of possible schedules grows extremely fast with each extra thread or step, so big models take too long to check. |
| **`cfg` flag** | A name that you switch on when compiling (`--cfg loom`). Code marked `#[cfg(loom)]` is compiled only when it is on. |
| **`RUSTFLAGS`** | An environment variable with extra options that Cargo passes to the Rust compiler `rustc`. |

## 1. Why normal tests are not enough

Here is a counter with a bug. `increment` reads the value and then writes the value plus one, in two separate steps. You met this bug, the *lost update*, in [Lesson 14.1](01-why-concurrency-is-hard.md): two threads can read the same number.

Create the library and replace `src/lib.rs`:

```bash
cd ~/projects/rust
cargo new --lib code/p14-loom
cd code/p14-loom
```

```rust
//! Small concurrent types, tested with loom.

use std::sync::atomic::{AtomicUsize, Ordering};

/// A counter that many threads can increment.
pub struct Counter {
    value: AtomicUsize,
}

impl Counter {
    pub fn new() -> Self {
        Counter {
            value: AtomicUsize::new(0),
        }
    }

    /// Adds one. BUG: the read and the write are two separate steps.
    pub fn increment(&self) {
        let current = self.value.load(Ordering::SeqCst);
        self.value.store(current + 1, Ordering::SeqCst);
    }

    pub fn get(&self) -> usize {
        self.value.load(Ordering::SeqCst)
    }
}

impl Default for Counter {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Arc;
    use std::thread;

    #[test]
    fn two_threads_increment_once_each() {
        for _ in 0..1000 {
            let counter = Arc::new(Counter::new());
            let other = Arc::clone(&counter);
            let handle = thread::spawn(move || other.increment());
            counter.increment();
            handle.join().unwrap();
            assert_eq!(counter.get(), 2);
        }
    }
}
```

**What happens:**

1. The test repeats the same small story 1000 times: two threads, each increments once, the result must be 2.
2. `thread::spawn` starts the second thread. While the operating system starts it, the main thread usually finishes its own `increment` already.
3. So the two increments almost never overlap, and the bug stays hidden.

Run `cargo test`:

```text
running 1 test
test tests::two_threads_increment_once_each ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.11s
```

The test is green, and the code is wrong. This is the main problem of concurrent code: the bad schedule exists, but it is rare, and it depends on timing, on the machine and on the load.

Sometimes a stress test does find a bug. The "data plus `Relaxed` flag" litmus test of [Lesson 14.6](06-memory-ordering.md) saw old data a few thousand times in 100,000 rounds on an M4 Mac, but only because `DATA` and `READY` sat in separate cache lines; without that padding it saw none, and an x86-64 machine would show none at all. A test that fails "sometimes, on some machines" is a flaky test, and it does not tell you that the code is correct when it passes.

## 2. What loom does

Look at the two steps of each thread in the counter. Here are two of the possible interleavings:

```text
Schedule A (correct)              Schedule B (lost update)
main:  load   -> 0                main:  load   -> 0
main:  store  1                   other: load   -> 0
other: load   -> 1                main:  store  1
other: store  2                   other: store  1
result: 2                         result: 1
```

A model checker runs your test once for **every** such schedule. Loom does it like this:

1. You use loom's types (`loom::sync::atomic::AtomicUsize`, `loom::sync::Mutex`, `loom::thread::spawn`, `loom::sync::Arc`) instead of the `std` ones, but only while testing.
2. Every operation on these types is a point where loom may switch to another thread. Loom's own scheduler, not the operating system, decides who runs.
3. `loom::model(|| { ... })` runs your closure, records the choices it made, then runs it again with different choices, until all schedules are done.
4. For atomics loom also simulates the memory model: a `Relaxed` or `Acquire` load may return an *older* value, when the rules of [Lesson 14.6](06-memory-ordering.md) allow it. So loom finds ordering bugs even on a machine that would never show them.
5. Loom also notices deadlocks (all threads blocked) and leaks of loom objects.

Loom is deterministic: when a test fails, it fails every time you run it, on every computer.

## 3. Preparing a crate for loom

Loom's types work only inside `loom::model`. Your normal build must keep using `std`. In Lesson 14.6 the test file itself used loom's types, so loom could be a dev-dependency. Here the code under test lives in your library, so the library must switch between loom and `std`. The usual solution has three parts.

**Part 1: `Cargo.toml`.** Add loom only when the `loom` flag is on, and tell the compiler that `loom` is a known flag name:

```toml
[package]
name = "p14-loom"
version = "0.1.0"
edition = "2024"

[dependencies]

[target.'cfg(loom)'.dependencies]
loom = "0.7"

[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ["cfg(loom)"] }
```

`[target.'cfg(loom)'.dependencies]` means "this dependency exists only when `cfg(loom)` is true". Without the `[lints.rust]` part, the compiler warns about `unexpected cfg condition name: loom`, because it checks that every `cfg` name is known.

**Part 2: a `sync` module.** One file decides where the types come from. Create `src/sync.rs`:

```rust
//! One place that decides where the sync types come from.

#[cfg(loom)]
pub(crate) use loom::sync::atomic::{AtomicUsize, Ordering};

#[cfg(not(loom))]
pub(crate) use std::sync::atomic::{AtomicUsize, Ordering};
```

In `src/lib.rs`, add `mod sync;` at the top and change the `use` line to `use sync::{AtomicUsize, Ordering};`. The rest of the code does not change. `pub(crate)` makes the names visible in the whole crate, but not outside it.

**Part 3: a test file only for loom.** Create `tests/loom.rs`:

```rust
#![cfg(loom)]

use loom::sync::Arc;
use loom::thread;
use p14_loom::Counter;

#[test]
fn two_threads_increment_once_each() {
    loom::model(|| {
        let counter = Arc::new(Counter::new());
        let other = Arc::clone(&counter);
        let handle = thread::spawn(move || other.increment());
        counter.increment();
        handle.join().unwrap();
        assert_eq!(counter.get(), 2);
    });
}
```

**What happens:**

1. `#![cfg(loom)]` at the top of the file switches off the whole file in a normal build. `cargo test` still compiles it, and reports `running 0 tests` for it.
2. The test body is the same story as before, but inside `loom::model`, and with `loom::sync::Arc` and `loom::thread`.
3. There is no loop of 1000: loom itself repeats the closure, once per schedule.

Run the loom tests with the flag, in release mode, and only the `loom` test file:

```bash
RUSTFLAGS="--cfg loom" cargo test --release --test loom
```

`--release` matters because loom may run the closure thousands of times. `--test loom` matters because the unit tests in `src/lib.rs` do not use `loom::model` (Common mistakes shows what happens otherwise).

> 💡 **Tip:** Changing `RUSTFLAGS` makes Cargo rebuild every dependency. If you switch between normal and loom runs often, give loom its own build folder: `CARGO_TARGET_DIR=target/loom RUSTFLAGS="--cfg loom" cargo test --release --test loom`. Then both builds stay cached.

## 4. Loom finds the lost update

Run the command. The test fails on the first try:

```text
     Running tests/loom.rs (target/release/build/p14-loom/8533a2a354401770/out/loom-8533a2a354401770)

running 1 test
test two_threads_increment_once_each ... FAILED

failures:

---- two_threads_increment_once_each stdout ----

thread 'two_threads_increment_once_each' (4876607) panicked at tests/loom.rs:15:9:
assertion `left == right` failed
  left: 1
 right: 2
```

`left: 1` is schedule B from Section 2: one update was lost. Fix `increment` so that reading and writing are one indivisible step:

```rust
    /// Adds one. The read and the write are one indivisible step.
    pub fn increment(&self) {
        self.value.fetch_add(1, Ordering::SeqCst);
    }
```

Run again, this time with `LOOM_LOG=info` so loom tells you how many schedules it checked. `--nocapture` shows the log, which the test harness hides otherwise:

```bash
RUSTFLAGS="--cfg loom" LOOM_LOG=info cargo test --release --test loom -- --nocapture
```

```text
running 1 test
test two_threads_increment_once_each ...  INFO loom::model: Completed in 4 iterations
ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The log also prints a line per thread switch, and colours; they are left out here.) "Completed in 4 iterations" means loom ran the closure four times and the assertion held in all four. That is not "it passed 1000 times by luck": it is "it passed in every schedule that exists".

> 🧪 **Testing:** Keep the normal test as well. It checks that the code works with the real `std` types and real threads, and `cargo test` runs it on every change. The loom test checks all schedules. They answer different questions.

## 5. Loom finds a memory-ordering bug

Now a type where every single operation is atomic, but the *order* is wrong. A `Mailbox` holds one number: one thread puts it, another takes it. Add two names to `src/sync.rs` (both `use` lines become `{AtomicBool, AtomicU64, AtomicUsize, Ordering}`), then create `src/mailbox.rs`:

```rust
use crate::sync::{AtomicBool, AtomicU64, Ordering};

/// A box for one number: one thread puts it, another thread takes it.
pub struct Mailbox {
    value: AtomicU64,
    ready: AtomicBool,
}

impl Mailbox {
    pub fn new() -> Self {
        Mailbox {
            value: AtomicU64::new(0),
            ready: AtomicBool::new(false),
        }
    }

    pub fn put(&self, value: u64) {
        self.value.store(value, Ordering::Relaxed);
        self.ready.store(true, Ordering::Relaxed); // BUG: should be Release
    }

    pub fn take(&self) -> Option<u64> {
        if self.ready.load(Ordering::Relaxed) {
            // BUG: should be Acquire
            Some(self.value.load(Ordering::Relaxed))
        } else {
            None
        }
    }
}

impl Default for Mailbox {
    fn default() -> Self {
        Self::new()
    }
}
```

In `src/lib.rs` add `mod mailbox;` and `pub use mailbox::Mailbox;`. Then add a test to `tests/loom.rs`:

```rust
#[test]
fn taker_sees_the_value_that_was_put() {
    use p14_loom::Mailbox;
    loom::model(|| {
        let mailbox = Arc::new(Mailbox::new());
        let sender = Arc::clone(&mailbox);
        let handle = thread::spawn(move || sender.put(42));
        if let Some(value) = mailbox.take() {
            assert_eq!(value, 42);
        }
        handle.join().unwrap();
    });
}
```

**What happens:**

1. The sender writes `42`, then sets `ready`.
2. The taker checks `ready`. Only if it is `true`, it reads the value, and the value must be `42`.
3. With `Relaxed`, nothing connects the two atomics. A thread may see `ready == true` and still see the old value `0`. This is exactly what the arm64 stress test showed in Section 1.

```text
---- taker_sees_the_value_that_was_put stdout ----

thread 'taker_sees_the_value_that_was_put' (4883659) panicked at tests/loom.rs:27:13:
assertion `left == right` failed
  left: 0
 right: 42
```

Loom found the stale value at once, on any machine. The fix is the release/acquire pair from [Lesson 14.6](06-memory-ordering.md): `put` stores `ready` with `Ordering::Release`, and `take` loads it with `Ordering::Acquire`. A load that sees the released `true` also sees everything the sender wrote before it. After the fix the test passes in 6 iterations.

## 6. Spin loops and deadlocks

A **spin loop** waits by checking a flag again and again. Here is a `Latch`, a gate that starts closed (`src/latch.rs`; add `mod latch;` and `pub use latch::Latch;` to `src/lib.rs`):

```rust
use crate::sync::{AtomicBool, Ordering, hint};

/// A gate that starts closed. Threads wait until someone opens it.
pub struct Latch {
    open: AtomicBool,
}

impl Latch {
    pub fn new() -> Self {
        Latch {
            open: AtomicBool::new(false),
        }
    }

    pub fn open(&self) {
        self.open.store(true, Ordering::Release);
    }

    pub fn wait(&self) {
        while !self.open.load(Ordering::Acquire) {
            hint::spin_loop();
        }
    }
}

impl Default for Latch {
    fn default() -> Self {
        Self::new()
    }
}
```

`hint` also comes from the `sync` module: add `pub(crate) use loom::hint;` under `#[cfg(loom)]` and `pub(crate) use std::hint;` under `#[cfg(not(loom))]`.

Why? Loom's scheduler is not fair *on purpose*: it tries every schedule, including "the waiting thread runs forever and the other one never runs". `loom::hint::spin_loop()` tells loom "I cannot make progress until another thread runs", so loom switches threads there. If you call `std::hint::spin_loop()` instead, loom sees an endless loop:

```text
thread 'wait_returns_after_open' (4904556) panicked at /Users/you/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/loom-0.7.3/src/rt/path.rs:175:9:
Model exceeded maximum number of branches. This is often caused by an algorithm requiring the processor to make progress, e.g. spin locks.
```

The same rule applies to the spin lock from [Lesson 14.7](07-building-a-spin-lock.md): inside the waiting loop call the `spin_loop` from your `sync` module (or `loom::thread::yield_now()` under `cfg(loom)`).

Loom also finds **deadlocks**. When every thread is waiting for a lock that another thread holds, loom stops with a message like this one (you will produce it in Exercise 3):

```text
deadlock; threads = [(Id(0), Blocked(Location(None))), (Id(1), Blocked(Location(None)))]
```

## 7. Keep models small, and know the limits

The number of schedules grows very fast. These are real counts for the fixed `Counter` test (`LOOM_LOG=info`):

| Model | Iterations |
|---|---|
| 2 threads, one increment each | 4 |
| 3 threads, one increment each | 253 |
| 3 threads, `LOOM_MAX_PREEMPTIONS=1` | 10 |
| 3 threads, `LOOM_MAX_PREEMPTIONS=2` | 40 |
| 3 threads, `LOOM_MAX_PREEMPTIONS=3` | 106 |

One extra thread turned 4 into 253. A test with four threads and loops can take hours. Rules that keep loom useful:

- **Two or three threads, a few operations each.** Test the smallest story that shows the property ("two puts, one take"), not a whole program.
- **Set a preemption bound for bigger models:** `LOOM_MAX_PREEMPTIONS=2` or `3`. The loom documentation says that 2 or 3 is enough to catch most bugs. The check is then no longer complete, but much faster.
- **Always `--release`.**

What loom cannot do:

- It sees only loom types. A `std::sync::Mutex` or `std::sync::Arc` hidden inside your code is invisible to it, and so are the schedules around it.
- It cannot model every reordering that `Relaxed` allows (for example, two operations in one thread swapping places), so a few rare bugs remain possible.
- It treats `SeqCst` loads and stores as `AcqRel`, which is weaker. This can produce false alarms for code that really needs `SeqCst` (a `SeqCst` fence is modelled correctly).
- No real time, randomness or I/O inside a model. Replace them with simple fakes (Phase 09 showed test doubles).

> 🔒 **Security:** Concurrency bugs in `unsafe` code (Lessons 14.7 and 14.8) are memory-safety bugs: a data race on an `UnsafeCell` is undefined behaviour. Loom has its own `loom::cell::UnsafeCell`, which reports two unsynchronised accesses as a "causality violation". Use it, and also run your normal tests under Miri (`cargo +nightly miri test`, [Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)), which checks the `unsafe` rules but does not try every schedule.

## Common mistakes

**1. Putting loom into `dev-dependencies`.** The library code itself (`src/sync.rs`) uses loom, so a test-only dependency is not enough:

```text
error[E0433]: cannot find module or crate `loom` in this scope
 --> src/sync.rs:4:16
  |
4 | pub(crate) use loom::sync::atomic::{AtomicUsize, Ordering};
  |                ^^^^ use of unresolved module or unlinked crate `loom`
```

Fix: use `[target.'cfg(loom)'.dependencies]`, not `[target.'cfg(loom)'.dev-dependencies]`. (In Lesson 14.6 a dev-dependency was enough, because only the test file used loom.)

**2. Running every test with the loom flag.** `RUSTFLAGS="--cfg loom" cargo test --release` without `--test loom` also runs the unit tests in `src/lib.rs`. Under the flag, `Counter` uses loom atomics, but the unit test does not call `loom::model`:

```text
thread 'tests::two_threads_increment_once_each' (5338353) panicked at /Users/you/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/loom-0.7.3/src/rt/scheduler.rs:128:13:
cannot access Loom execution state from outside a Loom model. are you accessing a Loom synchronization primitive from outside a Loom test (a call to `model` or `check`)?
```

Fix: always add `--test loom` (or put `#[cfg(not(loom))]` on the unit-test module).

**3. Importing `std` types directly.** If one file writes `use std::sync::atomic::AtomicBool;` instead of `use crate::sync::AtomicBool;`, loom does not see that atomic and cannot explore its schedules. The test may pass while the bug is still there. Fix: every sync type in the crate comes from `crate::sync`.

**4. `std::hint::spin_loop()` or an empty loop in a model.** You get "Model exceeded maximum number of branches". Fix: use the `spin_loop` from your `sync` module.

## Try it yourself

1. Follow Sections 1 to 6 in `code/p14-loom`. From Section 3 on, run both commands after each step:

   ```bash
   cargo test
   RUSTFLAGS="--cfg loom" cargo test --release --test loom
   ```

2. Put the bug back into `Counter::increment` for a moment and watch loom fail, then fix it again.
3. In `Mailbox::put`, change only `Release` back to `Relaxed` and keep `Acquire` in `take`. Predict the result, then run. (Loom fails again with `left: 0`: release and acquire work only as a pair.)
4. Run `cargo clippy --all-targets` and `cargo fmt`, then commit: `git add code/p14-loom && git commit -m "p14-loom: loom tests for counter, mailbox and latch"`.

## Exercises

### Exercise 1 — Three threads (easy)

Write a second loom test for the fixed `Counter`: the main thread and two spawned threads increment once each; the result must be 3. How many iterations does loom report? Then run it with `LOOM_MAX_PREEMPTIONS=1`.

<details>
<summary>Hint</summary>

Spawn the two threads in a `map` over `0..2`, collect the handles in a `Vec`, increment in the main thread, then join every handle.

</details>

<details>
<summary>Answer</summary>

```rust
#[test]
fn three_threads_increment_once_each() {
    loom::model(|| {
        let counter = Arc::new(Counter::new());
        let handles: Vec<_> = (0..2)
            .map(|_| {
                let other = Arc::clone(&counter);
                thread::spawn(move || other.increment())
            })
            .collect();
        counter.increment();
        for handle in handles {
            handle.join().unwrap();
        }
        assert_eq!(counter.get(), 3);
    });
}
```

```text
 INFO loom::model: Completed in 253 iterations
```

With `LOOM_MAX_PREEMPTIONS=1` it is `Completed in 10 iterations`. Why it works: each extra thread multiplies the number of places where loom can switch. The preemption bound cuts the schedules that need many forced switches.

</details>

### Exercise 2 — Exactly one winner (medium)

A `Ticket` can be claimed by only one thread:

```rust
use loom::sync::atomic::{AtomicBool, Ordering};

struct Ticket {
    taken: AtomicBool,
}

impl Ticket {
    fn try_claim(&self) -> bool {
        if !self.taken.load(Ordering::Acquire) {
            self.taken.store(true, Ordering::Release);
            true
        } else {
            false
        }
    }
}
```

Write a loom test (in a new file `tests/loom_ticket.rs` with `#![cfg(loom)]`) where two threads call `try_claim`, and assert that exactly one of them gets `true`. See it fail, then fix `try_claim` with one atomic operation.

<details>
<summary>Hint</summary>

`handle.join().unwrap()` returns the value that the thread's closure returned. "Exactly one is `true`" for two booleans is `a != b`. For the fix, look at `swap`: it writes a new value and returns the old one in one step.

</details>

<details>
<summary>Answer</summary>

```rust
#![cfg(loom)]

use loom::sync::Arc;
use loom::sync::atomic::{AtomicBool, Ordering};
use loom::thread;

struct Ticket {
    taken: AtomicBool,
}

impl Ticket {
    fn try_claim(&self) -> bool {
        !self.taken.swap(true, Ordering::AcqRel)
    }
}

#[test]
fn exactly_one_thread_wins() {
    loom::model(|| {
        let ticket = Arc::new(Ticket {
            taken: AtomicBool::new(false),
        });
        let other = Arc::clone(&ticket);
        let handle = thread::spawn(move || other.try_claim());
        let main_won = ticket.try_claim();
        let other_won = handle.join().unwrap();
        assert!(
            main_won != other_won,
            "main_won={main_won}, other_won={other_won}"
        );
    });
}
```

With the buggy version the test fails with `main_won=true, other_won=true`: both threads loaded `false` before either stored `true`. The fixed version passes (`Completed in 4 iterations`). Why it works: `swap` reads the old value and writes `true` as one indivisible step, so only one thread can see the old `false`.

</details>

### Exercise 3 — Find the deadlock (hard)

Two bank accounts, each with a `loom::sync::Mutex<i64>`. `transfer(from, to, amount)` locks `from`, then `to`. One thread transfers 10 from A to B while the main thread transfers 20 from B to A. Write the loom test, see the deadlock message, then fix `transfer` so that the test passes and checks the final balances (A = 110, B = 90).

<details>
<summary>Hint</summary>

The fix from [Lesson 14.1](01-why-concurrency-is-hard.md): all threads must take locks in the same global order. Give each account an `id` and always lock the smaller id first. Then decide which balance goes down.

</details>

<details>
<summary>Answer</summary>

```rust
#![cfg(loom)]

use loom::sync::{Arc, Mutex};
use loom::thread;

struct Account {
    id: u32,
    balance: Mutex<i64>,
}

fn transfer(from: &Account, to: &Account, amount: i64) {
    // Always lock the account with the smaller id first.
    let (first, second) = if from.id < to.id {
        (from, to)
    } else {
        (to, from)
    };
    let mut first_balance = first.balance.lock().unwrap();
    let mut second_balance = second.balance.lock().unwrap();
    if first.id == from.id {
        *first_balance -= amount;
        *second_balance += amount;
    } else {
        *second_balance -= amount;
        *first_balance += amount;
    }
}

#[test]
fn two_transfers_in_opposite_directions() {
    loom::model(|| {
        let a = Arc::new(Account {
            id: 1,
            balance: Mutex::new(100),
        });
        let b = Arc::new(Account {
            id: 2,
            balance: Mutex::new(100),
        });
        let (a2, b2) = (Arc::clone(&a), Arc::clone(&b));
        let handle = thread::spawn(move || transfer(&a2, &b2, 10));
        transfer(&b, &a, 20);
        handle.join().unwrap();
        assert_eq!(*a.balance.lock().unwrap(), 110);
        assert_eq!(*b.balance.lock().unwrap(), 90);
    });
}
```

The version that locks `from` first stops with `deadlock; threads = [(Id(0), Blocked(...)), (Id(1), Blocked(...))]`. The fixed version passes in 5 iterations. Why it works: with one global lock order, no thread can hold the "second" lock while it waits for the "first" one, so a cycle of waiting threads is impossible.

</details>

### Exercise 4 — Check the spin lock from Lesson 14.7 (hard)

Loom can only check the lock if it also sees the data inside it. Add a `SpinLock<T>` to `p14-loom` (`src/spin_lock.rs`, and `pub mod spin_lock;` in `src/lib.rs`). Use `UnsafeCell` from your `sync` module, and give the lock a closure API (API: Application Programming Interface, the methods a type offers) instead of a guard: `with_lock(&self, f: impl FnOnce(&mut T) -> R) -> R` locks, runs `f` with the value and unlocks. Write a loom test in `tests/loom_spin.rs`: two threads each add 1 under the lock; the result must be 2. Then change the `Acquire` of the lock operation to `Relaxed` and read loom's report.

<details>
<summary>Hint</summary>

`loom::cell::UnsafeCell` has no `get()`; instead `with_mut(|ptr: *mut T| …)` gives you the raw pointer for the time of the closure, and loom records the access. Give your `std` version of `UnsafeCell` in `src/sync.rs` the same method (the loom documentation shows this "shim"). If loom stops with "Model exceeded maximum number of branches" although you use the `spin_loop` from your `sync` module: the `swap` loop of Lesson 14.7 *writes* `true` on every failed try, and loom cannot finish exploring that. Wait with plain `load`s and try `compare_exchange` only when the lock looks free (the "test and test-and-set" lock).

</details>

<details>
<summary>Answer</summary>

`src/sync.rs` gets the loom `UnsafeCell` and a small `std` replacement with the same API:

```rust
//! One place that decides where the sync types come from.

#[cfg(loom)]
pub(crate) use loom::cell::UnsafeCell;
#[cfg(loom)]
pub(crate) use loom::hint;
#[cfg(loom)]
pub(crate) use loom::sync::atomic::{AtomicBool, AtomicU64, AtomicUsize, Ordering};

#[cfg(not(loom))]
pub(crate) use std::hint;
#[cfg(not(loom))]
pub(crate) use std::sync::atomic::{AtomicBool, AtomicU64, AtomicUsize, Ordering};

/// The same small API as `loom::cell::UnsafeCell`, built on the std type.
#[cfg(not(loom))]
#[derive(Debug)]
pub(crate) struct UnsafeCell<T>(std::cell::UnsafeCell<T>);

#[cfg(not(loom))]
impl<T> UnsafeCell<T> {
    pub(crate) fn new(value: T) -> Self {
        UnsafeCell(std::cell::UnsafeCell::new(value))
    }

    pub(crate) fn with_mut<R>(&self, f: impl FnOnce(*mut T) -> R) -> R {
        f(self.0.get())
    }
}
```

`src/spin_lock.rs`:

```rust
//! The spin lock from Lesson 14.7, with a closure API that loom can check.

use crate::sync::{AtomicBool, Ordering, UnsafeCell, hint};

pub struct SpinLock<T> {
    locked: AtomicBool,
    value: UnsafeCell<T>,
}

// SAFETY: only the thread that holds the lock reaches `value`.
unsafe impl<T: Send> Sync for SpinLock<T> {}

impl<T> SpinLock<T> {
    pub fn new(value: T) -> Self {
        SpinLock {
            locked: AtomicBool::new(false),
            value: UnsafeCell::new(value),
        }
    }

    /// Locks, runs `f` with the value, unlocks.
    pub fn with_lock<R>(&self, f: impl FnOnce(&mut T) -> R) -> R {
        // Wait with plain loads; try to take the lock only when it looks free.
        while self
            .locked
            .compare_exchange(false, true, Ordering::Acquire, Ordering::Relaxed)
            .is_err()
        {
            while self.locked.load(Ordering::Relaxed) {
                hint::spin_loop();
            }
        }
        // SAFETY: we hold the lock, so no other thread uses the value now.
        let result = self.value.with_mut(|ptr| f(unsafe { &mut *ptr }));
        self.locked.store(false, Ordering::Release);
        result
    }
}
```

`tests/loom_spin.rs`:

```rust
#![cfg(loom)]

use loom::sync::Arc;
use loom::thread;
use p14_loom::spin_lock::SpinLock;

#[test]
fn two_threads_increment_under_the_lock() {
    loom::model(|| {
        let lock = Arc::new(SpinLock::new(0_u32));
        let other = Arc::clone(&lock);
        let handle = thread::spawn(move || other.with_lock(|n| *n += 1));
        lock.with_lock(|n| *n += 1);
        handle.join().unwrap();
        assert_eq!(lock.with_lock(|n| *n), 2);
    });
}
```

```text
 INFO loom::model: Completed in 17 iterations
```

With `Ordering::Relaxed` as the success ordering of `compare_exchange`, loom reports:

```text
thread 'two_threads_increment_under_the_lock' (6021419) panicked at /Users/you/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/loom-0.7.3/src/rt/location.rs:115:9:
Causality violation: Concurrent write accesses to `UnsafeCell`.
```

Why it works: `Acquire` on the successful `compare_exchange` pairs with the `Release` store in the unlock, so the second thread's access to the cell happens after the first thread's access. With `Relaxed` there is no such order, and loom's `UnsafeCell` sees two writes that are not ordered: a data race, which loom calls a causality violation. The closure API keeps every access to the value inside `with_mut`, where loom can see it.

</details>

## Quiz

1. Why can a test that loops 1000 times pass while the code has a race?

<details>
<summary>Answer</summary>

The bad schedule is rare and depends on timing. Starting a thread is slow, so the threads in the test almost never overlap. The loop only repeats the most likely schedules; it does not try the rare ones.

</details>

2. What does `loom::model` do with the closure you give it?

<details>
<summary>Answer</summary>

It runs the closure many times. Each run follows a different schedule of the loom threads and, for atomics, may return different allowed values. It stops when all schedules (inside the preemption bound) are checked, or at the first panic.

</details>

3. Why does a crate that uses loom need a `sync` module and a `cfg(loom)` flag?

<details>
<summary>Answer</summary>

Loom types work only inside a loom model, so the normal build must use `std`. The `sync` module switches all sync types in one place: loom types when `--cfg loom` is on, `std` types otherwise.

</details>

4. What does the message "Model exceeded maximum number of branches" usually mean?

<details>
<summary>Answer</summary>

A loop waits for another thread but never tells loom about it, so loom explores "this thread spins forever". Use `loom::hint::spin_loop()` or `loom::thread::yield_now()` in the loop (through your `sync` module).

</details>

5. Your loom tests pass. Does that prove that the code is correct? Name one limit.

<details>
<summary>Answer</summary>

No. Loom checks only what its types can see and what your assertions check. Limits: `std` types hidden in the code are invisible; some `Relaxed` reorderings are not modelled; `SeqCst` is treated as `AcqRel`; and a model covers only the small story you wrote.

</details>

## Summary

- Concurrent bugs hide in rare schedules. Normal tests and stress tests try only some schedules, and the result depends on the machine.
- Loom is a model checker: `loom::model` runs a test once per possible schedule, simulates the memory model for atomics, and finds lost updates, ordering bugs, endless spin loops and deadlocks.
- Prepare the crate with `[target.'cfg(loom)'.dependencies]`, a `check-cfg` lint entry, a `sync` module that picks loom or `std` types, and a `tests/loom.rs` file marked `#![cfg(loom)]`.
- Run it with `RUSTFLAGS="--cfg loom" cargo test --release --test loom`; use `LOOM_LOG=info` to see the iteration count.
- Spin loops must use loom's `spin_loop` or `yield_now`, or loom never finishes.
- Keep models tiny (two or three threads), use `LOOM_MAX_PREEMPTIONS` for bigger ones, and keep normal tests and Miri as well.

## Sources and further reading

- [loom on docs.rs](https://docs.rs/loom/0.7.3/loom/) — the official documentation: setup, yielding, debugging with `LOOM_LOG`, checkpoints and limitations.
- [loom on GitHub](https://github.com/tokio-rs/loom) — the README with the list of unsupported features.
- [Rust Atomics and Locks — Chapter 3. Memory Ordering](https://marabos.nl/atomics/memory-ordering.html) — by Mara Bos, free online; the rules that loom simulates.
- [The rustc book — Checking conditional configurations](https://doc.rust-lang.org/rustc/check-cfg.html) — why `unexpected_cfgs` warns and how `check-cfg` declares a custom flag.
- [The Cargo Book — Environment variables](https://doc.rust-lang.org/cargo/reference/environment-variables.html) — `RUSTFLAGS` and `CARGO_TARGET_DIR`.
- [Miri](https://github.com/rust-lang/miri) — the interpreter that detects undefined behaviour; a good partner for loom.

---

[🏠 Start](../START-HERE.md) · [← 14.8 Lock-free data structures](08-lock-free-data-structures.md) · [📋 Phase 14](README.md) · [14.10 How futures work: Future, poll and Waker →](10-how-futures-work.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
