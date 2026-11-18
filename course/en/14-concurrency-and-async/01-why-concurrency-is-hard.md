<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 14 — Concurrency and async in depth](README.md) · [📋 Phase 14](README.md) · [14.2 Mutexes and read-write locks in depth →](02-mutexes-and-rwlocks-in-depth.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/01-why-concurrency-is-hard.md) <!-- nav -->

# Lesson 14.1 — Why concurrency is hard: races and deadlocks

> **Phase 14 — Concurrency and async in depth** · Lesson 1 of 16 · ⏱ about 135 minutes

## What you will learn

- Why the order in which threads run is not under your control, and what an *interleaving* is.
- The difference between a data race (Rust stops it at compile time) and a race condition (Rust cannot stop it).
- Two classic race conditions: the *lost update* and *check-then-act*.
- What a deadlock is, the four conditions it needs, and the simplest cure: a fixed lock order.
- Why concurrency bugs hide from normal tests, and what you can do about it.

## Before you start

- You finished [Lesson 05.10 — Threads, channels and shared state](../05-rust-intermediate/10-threads-channels-and-shared-state.md). You know `thread::spawn`, `join`, `Arc<Mutex<T>>`, `Send`/`Sync`, and you saw a first deadlock there.
- You know how the OS (Operating System) runs threads on the cores of the CPU (Central Processing Unit) and switches between them: [Lesson 13.9 — Threads and scheduling](../13-operating-systems/09-threads-and-scheduling.md).
- Create the project for the whole phase:

```bash
cd ~/projects/rust
cargo new code/p14-concurrency
cd code/p14-concurrency
mkdir -p src/bin
```

In this phase, every example is a separate file in `src/bin/`, for example `src/bin/lost_update.rs`, and you run it with `cargo run --bin lost_update`. Cargo builds every file in `src/bin/` as its own small program, so you can keep all examples side by side. Later lessons add crates to the same `Cargo.toml`.

## New words

| Word | Meaning |
|---|---|
| **interleaving** | One possible order in which the steps of several threads are mixed together in time. |
| **shared state** | Data that more than one thread can read or change. |
| **data race** | Two threads touch the same memory at the same time, at least one of them writes, and nothing orders the two accesses. In safe Rust this cannot compile. |
| **race condition** | A bug where the result depends on the timing of threads. It can happen in safe Rust. |
| **lost update** | A race condition: two threads read the same old value, both write a new value, and one of the writes is lost. |
| **read-modify-write** | An operation in three steps: read a value, compute a new one, write it back. `x += 1` is one. |
| **check-then-act** | A race condition: a thread checks a condition, and acts on it later, but the condition changed in between. |
| **critical section** | The code that runs while a lock is held. |
| **invariant** | A rule about your data that must always be true, for example "the total money in the bank does not change". |
| **deadlock** | Two or more threads wait for each other forever. |
| **lock ordering** | A rule that every thread takes several locks in the same fixed order. It prevents deadlocks. |
| **livelock** | Threads keep reacting to each other and stay busy, but none of them makes progress. |
| **starvation** | One thread waits for a very long time (or forever) because other threads always get the resource first. |
| **heisenbug** | A bug that disappears or changes when you try to observe it (for example by adding a `println!`). Concurrency bugs are often heisenbugs. |
| **watchdog** | A small piece of code that watches for progress and reports when there is none. |
| **OS** (Operating System) | The program (macOS, Linux) that runs other programs and decides which thread runs on which core. |
| **CPU** (Central Processing Unit) | The chip that runs program instructions. It has several *cores*; each core runs one thread at a time. |
| **TOCTOU** (Time Of Check To Time Of Use) | A check-then-act bug in security: something is checked, then changed by someone else before it is used. |

## 1. You do not control the order

A single-threaded program is like a recipe read by one cook: step 1, step 2, step 3, always in the same order. A program with two threads is like two cooks in one small kitchen who share the knives and the stove. Each cook follows their own recipe in order, but *how their steps mix together* depends on who is faster at that moment.

You learned in Lesson 13.9 that the OS *scheduler* (the part of the operating system that picks which thread runs next) decides when each thread runs, and that it can pause a thread between any two machine instructions. So if thread A has steps `A1 A2` and thread B has steps `B1 B2`, all of these orders are possible:

```text
A1 A2 B1 B2
A1 B1 A2 B2
A1 B1 B2 A2
B1 A1 A2 B2
B1 A1 B2 A2
B1 B2 A1 A2
```

Each line is one **interleaving**. With two threads of two steps there are 6. With two threads of 10 steps there are already 184,756. You cannot test them all by running the program a few times, and the one that breaks your program may happen once in a million runs, on a busy day, on a customer's machine.

Concurrent code is correct only if it is correct for **every** interleaving. That is the whole difficulty of this phase.

## 2. Data race versus race condition

Two words sound alike but mean different things.

A **data race** is a low-level memory problem: two threads access the same memory at the same time, at least one writes, and there is no lock or atomic operation that orders them. The result is *undefined behaviour* ([Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)): anything may happen. Safe Rust **does not compile** a data race. Here two threads try to change one local variable:

```rust
use std::thread;

fn main() {
    let mut count = 0;
    thread::scope(|s| {
        s.spawn(|| count += 1);
        s.spawn(|| count += 1);
    });
    println!("{count}");
}
```

```text
error[E0499]: cannot borrow `count` as mutable more than once at a time
 --> src/bin/data_race.rs:7:17
  |
6 |         s.spawn(|| count += 1);
  |         ----------------------
  |         |       |  |
  |         |       |  first borrow occurs due to use of `count` in closure
  |         |       first mutable borrow occurs here
  |         ...
7 |         s.spawn(|| count += 1);
  |                 ^^ ----- second borrow occurs due to use of `count` in closure
  |                 |
  |                 second mutable borrow occurs here
```

**What happens:**

1. Each closure changes `count`, so each closure needs a `&mut` borrow of it.
2. Both threads live at the same time, so there would be two `&mut` borrows at once.
3. The borrow checker forbids two `&mut` borrows at the same time, so the data race is stopped before the program exists.

A **race condition** is a *logic* problem: every single memory access is safe (each one is inside a lock or is atomic), but the **steps are combined in a wrong way**, so the result depends on timing. The compiler cannot see your intent, so it cannot stop this. The rest of this lesson shows race conditions in perfectly safe Rust.

```text
 data race       = unsafe memory access     → safe Rust: compile error
 race condition  = wrong order of safe steps → safe Rust: compiles, gives wrong results
```

## 3. The lost update

`AtomicU64` (you met atomics at the end of Lesson 05.10; Lesson 14.5 explains them fully) is a number that many threads may read and write without a lock. Each single `load` (read) and `store` (write) is safe. But look at what this program does with them. Create `src/bin/lost_update.rs`:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

fn main() {
    let counter = Arc::new(AtomicU64::new(0));
    let mut handles = Vec::new();

    for _ in 0..4 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..100_000 {
                // Two separate steps: read, then write. Not one step!
                let value = counter.load(Ordering::Relaxed);
                counter.store(value + 1, Ordering::Relaxed);
            }
        }));
    }
    for handle in handles {
        handle.join().unwrap();
    }

    println!("expected 400000, got {}", counter.load(Ordering::Relaxed));
}
```

Run it three times with `cargo run --bin lost_update`:

```text
expected 400000, got 108080
expected 400000, got 106163
expected 400000, got 104529
```

Your numbers will be different, and they change on every run. That is the sign of a race condition.

**What happens:**

1. Four threads share one counter through `Arc`. Each thread wants to add 1, 100,000 times.
2. `load` reads the current value; `store` writes `value + 1`. `Ordering::Relaxed` is explained in Lesson 14.6; here it only means "just this one number, no other guarantees".
3. Between the `load` and the `store` of one thread, another thread can also `load` the **same** old value. Both then store the same new value. Two additions happened, but the counter grew by one. One update is **lost**.
4. With four threads working all the time, most updates are lost: about three quarters here.

Here is one bad interleaving, step by step:

```text
 time   thread A                 thread B                 counter
  1     value = load()  → 7                               7
  2                              value = load()  → 7      7
  3     store(7 + 1)                                      8
  4                              store(7 + 1)             8   ← should be 9
```

`counter += 1` looks like one action, but it is a **read-modify-write**: read, compute, write. If another thread can run between the read and the write, updates get lost. There are two fixes, and both make the three steps one indivisible unit:

- An atomic read-modify-write operation: `counter.fetch_add(1, Ordering::Relaxed)` (Exercise 2).
- A lock that is held from the read until the write (Exercise 2 again).

> ⚠️ **Common mistake:** Thinking that "every operation is atomic" means "my code is atomic". Two atomic operations in a row are **two** steps, and another thread can run between them.

## 4. Check-then-act

The second classic race condition uses a `Mutex`, and every access is locked. A bank account has 100. Two people try to withdraw 80 at the same moment. Create `src/bin/withdraw.rs`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn withdraw(balance: &Mutex<i64>, amount: i64) -> bool {
    // Step 1: check (the lock is taken and released in this one line).
    let enough = *balance.lock().unwrap() >= amount;
    if !enough {
        return false;
    }
    // Some other work happens here, for example writing a log line.
    thread::sleep(Duration::from_millis(10));
    // Step 2: act (the lock is taken again).
    *balance.lock().unwrap() -= amount;
    true
}

fn main() {
    let balance = Arc::new(Mutex::new(100));
    let mut handles = Vec::new();

    for name in ["Alice", "Bob"] {
        let balance = Arc::clone(&balance);
        handles.push(thread::spawn(move || {
            let ok = withdraw(&balance, 80);
            println!("{name} withdraws 80: {ok}");
        }));
    }
    for handle in handles {
        handle.join().unwrap();
    }
    println!("final balance: {}", *balance.lock().unwrap());
}
```

```text
Alice withdraws 80: true
Bob withdraws 80: true
final balance: -60
```

(The two first lines may come in the other order.)

**What happens:**

1. In `withdraw`, the first `lock()` creates a temporary guard. The guard is dropped at the end of that statement, so the lock is released right after the check.
2. Both threads check "is the balance at least 80?" while the balance is still 100. Both get `true`.
3. Both sleep 10 ms. The sleep only makes the bad interleaving happen every time; without it, the bug would appear rarely, which is worse.
4. Both lock again and subtract 80. The balance becomes −60, which breaks the **invariant** "the balance is never negative".

The bug is not in any single line. It is in the gap between the check and the action: the lock was released, so the fact "there is enough money" was no longer true when we acted on it. The fix is to hold **one** lock for the check *and* the action. Create `src/bin/withdraw_fixed.rs`: copy `withdraw.rs` and replace only the function `withdraw`:

```rust
fn withdraw(balance: &Mutex<i64>, amount: i64) -> bool {
    let mut guard = balance.lock().unwrap(); // one lock for check AND act
    if *guard < amount {
        return false;
    }
    thread::sleep(Duration::from_millis(10));
    *guard -= amount;
    true
}
```

```text
Alice withdraws 80: true
Bob withdraws 80: false
final balance: 20
```

**What happens:** the guard lives until the end of the function. While Alice is inside, Bob waits at `lock()`. When Bob gets the lock, he sees 20 and gets `false`. Which person wins depends on timing, but the invariant always holds.

> 📏 **Convention:** The **critical section** must cover the whole invariant: everything you check and everything you change because of the check. "Lock each line separately" is not thread safety.

> 🔒 **Security:** Check-then-act bugs are a real class of security holes, called TOCTOU (Time Of Check To Time Of Use): a program checks "may this user open this file?", and the file is swapped before it is opened. The cure is the same: make the check and the use one step.

## 5. Deadlock

You saw a deadlock in Lesson 05.10: two threads, two locks, taken in opposite orders. Now let us see it in a realistic program and fix it properly. A bank moves money between accounts; each account has its own mutex, so transfers between *different* accounts can run in parallel. Create `src/bin/transfer.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

struct Bank {
    accounts: Vec<Mutex<i64>>,
}

impl Bank {
    fn transfer(&self, from: usize, to: usize, amount: i64) {
        let mut a = self.accounts[from].lock().unwrap();
        let mut b = self.accounts[to].lock().unwrap();
        *a -= amount;
        *b += amount;
    }
}

fn main() {
    let bank = Arc::new(Bank {
        accounts: vec![Mutex::new(1000), Mutex::new(1000)],
    });
    let done = Arc::new(AtomicU64::new(0)); // how many transfers finished

    for (from, to) in [(0, 1), (1, 0)] {
        let bank = Arc::clone(&bank);
        let done = Arc::clone(&done);
        thread::spawn(move || {
            for _ in 0..100_000 {
                bank.transfer(from, to, 1);
                done.fetch_add(1, Ordering::Relaxed);
            }
        });
    }

    // A watchdog: if no transfer finishes for one second, we are stuck.
    let mut last = 0;
    loop {
        thread::sleep(Duration::from_secs(1));
        let now = done.load(Ordering::Relaxed);
        if now == 200_000 {
            println!("all 200000 transfers done");
            break;
        }
        if now == last {
            println!("no progress after {now} transfers: deadlock!");
            std::process::exit(1);
        }
        last = now;
    }
}
```

Three runs:

```text
no progress after 71431 transfers: deadlock!
no progress after 0 transfers: deadlock!
no progress after 30 transfers: deadlock!
```

**What happens:**

1. Thread 1 transfers from account 0 to account 1: it locks 0, then 1. Thread 2 transfers from 1 to 0: it locks 1, then 0.
2. Sooner or later thread 1 holds lock 0 and thread 2 holds lock 1 at the same moment. Each now waits for the lock the other one holds. Neither can continue, and neither will ever release its lock.
3. The main thread is a **watchdog**. Every second it checks the `done` counter. If the number did not grow, nothing is moving, so it reports the deadlock and ends the whole process with `std::process::exit(1)`. Without the watchdog the program would just hang silently, which is what a deadlock looks like in real life.
4. *When* it hangs is random: after 71,431 transfers or right at the start. Remember that when someone says "it worked when I tested it".

### The four conditions

A deadlock can only happen when **all four** of these are true at once (computer scientists call them the Coffman conditions):

| Condition | In our bank |
|---|---|
| 1. **Mutual exclusion** — a resource can be held by one thread only | A mutex is held by one thread at a time. |
| 2. **Hold and wait** — a thread holds one resource while waiting for another | `transfer` holds `from` while it waits for `to`. |
| 3. **No preemption** — nobody can take a resource away from a thread | Nobody can force a thread to give up a `MutexGuard`. |
| 4. **Circular wait** — there is a circle of threads, each waiting for the next | Thread 1 waits for thread 2, which waits for thread 1. |

Break any one condition and deadlock becomes impossible. The easiest one to break is number 4.

### The cure: one global lock order

Give every lock a number, and make every thread take locks in increasing order. Then a circle cannot form: a thread that holds lock 5 only ever waits for locks greater than 5, so nobody can wait "backwards". Create `src/bin/transfer_fixed.rs`: copy `transfer.rs` and replace only the body of `transfer`:

```rust
    fn transfer(&self, from: usize, to: usize, amount: i64) {
        if from == to {
            return; // locking the same mutex twice would hang
        }
        // Always lock the account with the smaller index first.
        let (low, high) = if from < to { (from, to) } else { (to, from) };
        let mut low_guard = self.accounts[low].lock().unwrap();
        let mut high_guard = self.accounts[high].lock().unwrap();
        if from == low {
            *low_guard -= amount;
            *high_guard += amount;
        } else {
            *high_guard -= amount;
            *low_guard += amount;
        }
    }
```

```text
all 200000 transfers done
```

**What happens:**

1. Both threads now lock account 0 first and account 1 second, whatever the direction of the transfer.
2. A thread holding lock 0 may wait for lock 1. A thread holding lock 1 never waits for lock 0, because it already took lock 0 before. No circle, no deadlock.
3. The `from == to` check matters: a transfer from an account to itself would lock the same mutex twice in one thread. The standard `Mutex` is not *re-entrant* (it does not let the same thread lock it again); on macOS such a program simply hangs forever.

> 💡 **Tip:** In real code, the "number" of a lock can be anything that is fixed and comparable: an account ID, a position in a vector, or the memory address of the mutex.

## 6. Livelock and starvation

Two more ways that threads fail to make progress, without hanging completely:

- **Livelock.** Two polite people meet in a corridor. Both step left, then both step right, then both left… They are busy, but nobody passes. In code this happens when threads use `try_lock` (Lesson 14.2), see that the other lock is busy, release everything, and retry at the same rhythm. The usual fix is to wait a small *random* time before retrying.
- **Starvation.** One thread almost never gets the lock because other threads always grab it first. The program works, but one client waits for minutes. Fairness of locks is a topic of Lesson 14.2.

## 7. Why your tests do not find these bugs

Concurrency bugs depend on timing, and timing depends on everything: the number of cores, other programs, the debug or release build, even a `println!` (printing takes a lock inside the standard library, which changes the timing). A test that passed 1,000 times proves very little. This is why concurrency bugs are often **heisenbugs**.

What helps:

1. **Design so that races are impossible.** Keep the critical section around the whole invariant. Prefer message passing (channels) to shared state when you can. Use one lock order.
2. **Make bad interleavings likely.** Use many threads, many iterations, and short sleeps in the "gap" while you investigate (as we did in `withdraw`).
3. **Test invariants, not timing.** At the end, check something that must always be true: the total money, the number of items, the sum.
4. **Detect hangs instead of waiting forever.** Use a watchdog or a time limit.
5. **Explore interleavings systematically.** The `loom` crate runs your test under *every* possible interleaving of a small model. You will meet it in Lesson 14.6 and learn it fully in Lesson 14.9.

> 🧪 **Testing:** A good concurrency test has three parts: start many threads that hammer the shared data, `join` them all, then assert an invariant. Run it with `cargo test --release` too: the release build is faster, which changes the timing and often shows other interleavings.

## Common mistakes

1. **Locking each step separately.** `if *m.lock().unwrap() > 0 { *m.lock().unwrap() -= 1; }` takes the lock twice, so it is a check-then-act bug. Hold one guard for both steps.
2. **Two atomic operations used as one.** `load` then `store` is a lost-update bug. Use `fetch_add`, `fetch_sub`, or the compare-and-swap loop from Lesson 14.5.
3. **Taking two locks in different orders in different places.** A deadlock waiting to happen. Write the lock order down in a comment and follow it everywhere.
4. **Locking the same `Mutex` twice in one thread.** It hangs (or may panic; the standard library does not promise which). This often hides in a helper function that locks a mutex the caller already holds.
5. **"It passed on my machine."** Timing on another machine is different. Test invariants with many threads, and use a watchdog or a time limit so a hang becomes a clear failure.

## Try it yourself

1. Run `cargo run --bin lost_update` five times and write down the results. Now run `cargo run --release --bin lost_update`. Is the result closer to 400,000 or further away? (On the test machine it was `102672`: the faster code collides even more often.)
2. In `withdraw.rs`, remove the line `thread::sleep(Duration::from_millis(10));` and run the program twenty times in a row:

   ```bash
   for i in $(seq 20); do cargo run -q --bin withdraw | tail -1; done
   ```

   The compiler warns that `Duration` is now unused; that is fine for this experiment. On the test machine all 20 runs printed `final balance: 20`. The bug is still there; it is only hiding, because the gap between check and act is now very short. Put the sleep back.

## Exercises

### Exercise 1 — How many threads? (easy)

Change the number of threads in `lost_update.rs` from 4 to 1, then to 8 (also fix the expected number in the message). Run each version and explain the results.

<details>
<summary>Hint</summary>

With one thread, is there anybody who can run between the `load` and the `store`?

</details>

<details>
<summary>Answer</summary>

Results on the test machine:

```text
expected 100000, got 100000
expected 800000, got 169544
expected 800000, got 353120
```

With one thread, no other thread can run between `load` and `store`, so no update is lost and the result is exact. With eight threads, more threads collide, so a lot of updates are lost, and the number jumps around from run to run. A test with a single thread would never find this bug.

</details>

### Exercise 2 — Two ways to fix the lost update (medium)

Make two copies of `lost_update.rs`: `lu_fixed.rs`, which uses one atomic operation instead of `load` + `store`, and `lu_mutex.rs`, which uses `Arc<Mutex<u64>>` but still reads the value and writes it back in two statements. Both must always print `got 400000`.

<details>
<summary>Hint</summary>

The atomic type has a method that adds and returns the old value in one step. For the mutex version, take the guard once and keep it for both statements.

</details>

<details>
<summary>Answer</summary>

In `lu_fixed.rs`, replace the two lines inside the loop with one line (the rest of the file stays the same):

```rust
                counter.fetch_add(1, Ordering::Relaxed); // one indivisible step
```

`lu_mutex.rs`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0u64));
    let mut handles = Vec::new();

    for _ in 0..4 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..100_000 {
                let mut guard = counter.lock().unwrap();
                let value = *guard; // read...
                *guard = value + 1; // ...and write while holding the lock
            }
        }));
    }
    for handle in handles {
        handle.join().unwrap();
    }

    println!("expected 400000, got {}", *counter.lock().unwrap());
}
```

```text
expected 400000, got 400000
```

Why it works: in both versions the read and the write form one unit. `fetch_add` does it in hardware; the mutex does it by keeping other threads out while the guard is alive. Two steps are fine *inside* one critical section.

</details>

### Exercise 3 — Test the bank (medium)

Add a test to `transfer_fixed.rs` called `transfers_keep_the_total`. It creates a bank with 3 accounts of 1000 each, starts 4 threads that make 10,000 transfers of 1 between different pairs of accounts, joins them, and asserts that the sum of all balances is still 3000. Run it with `cargo test --bin transfer_fixed`.

<details>
<summary>Hint</summary>

A test module inside a file in `src/bin/` works like one in `src/main.rs`: `#[cfg(test)] mod tests { use super::*; ... }`. To get different pairs, compute `from` and `to` from the thread number and the loop counter with `% 3`. Your `transfer` already returns early when `from == to`.

</details>

<details>
<summary>Answer</summary>

Add at the end of `src/bin/transfer_fixed.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn transfers_keep_the_total() {
        let bank = Arc::new(Bank {
            accounts: (0..3).map(|_| Mutex::new(1000)).collect(),
        });
        let mut handles = Vec::new();
        for t in 0..4 {
            let bank = Arc::clone(&bank);
            handles.push(thread::spawn(move || {
                for i in 0..10_000 {
                    let from = (t + i) % 3;
                    let to = (t + 2 * i + 1) % 3;
                    bank.transfer(from, to, 1);
                }
            }));
        }
        for handle in handles {
            handle.join().unwrap();
        }
        let total: i64 = bank.accounts.iter().map(|a| *a.lock().unwrap()).sum();
        assert_eq!(total, 3000);
    }
}
```

```text
running 1 test
test tests::transfers_keep_the_total ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Why it works: the test checks the invariant, not the timing. It also checks for deadlocks indirectly: with the old `transfer` this test would sometimes hang, and `cargo test` would print `has been running for over 60 seconds`. With the lock order it always finishes.

</details>

### Exercise 4 — Five philosophers (hard)

A classic puzzle: five philosophers sit at a round table. Between each two neighbours lies one fork, so there are five forks. To eat, a philosopher needs both the fork on the left (number `p`) and the fork on the right (number `(p + 1) % 5`). If every philosopher first takes the left fork and then the right one, they can all hold one fork and wait forever.

Write `src/bin/philosophers.rs`: the forks are `Arc<Vec<Mutex<u32>>>` (the number counts how often the fork was used). Each of 5 threads "eats" 10,000 times: it locks both forks and adds 1 to each counter. The program must never deadlock and must print the use count of every fork.

<details>
<summary>Hint</summary>

Which of the four conditions can you break? Philosopher 4 has the forks 4 and 0. If everybody takes the fork with the smaller number first, what does philosopher 4 do?

</details>

<details>
<summary>Answer</summary>

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let forks: Arc<Vec<Mutex<u32>>> = Arc::new((0..5).map(|_| Mutex::new(0)).collect());
    let mut handles = Vec::new();

    for p in 0..5 {
        let forks = Arc::clone(&forks);
        handles.push(thread::spawn(move || {
            let left = p;
            let right = (p + 1) % 5;
            // Lock order rule: the smaller fork number first.
            let (first, second) = (left.min(right), left.max(right));
            for _ in 0..10_000 {
                let mut a = forks[first].lock().unwrap();
                let mut b = forks[second].lock().unwrap();
                *a += 1; // count how often each fork was used
                *b += 1;
            }
        }));
    }
    for handle in handles {
        handle.join().unwrap();
    }
    let uses: Vec<u32> = forks.iter().map(|f| *f.lock().unwrap()).collect();
    println!("all philosophers ate 10000 times; fork uses: {uses:?}");
}
```

```text
all philosophers ate 10000 times; fork uses: [20000, 20000, 20000, 20000, 20000]
```

Why it works: philosophers 0 to 3 take the left fork first, but philosopher 4 takes fork 0 (his right one) first, because 0 < 4. This breaks the circle (condition 4). Every fork is used by its two neighbours, 10,000 times each, so every count is 20,000.

</details>

## Quiz

1. What is the difference between a data race and a race condition? Which one can safe Rust have?

<details>
<summary>Answer</summary>

A data race is an unordered access to the same memory by two threads, at least one writing; it is undefined behaviour, and safe Rust refuses to compile it. A race condition is a logic bug where the result depends on timing, even though every single access is safe. Safe Rust can have race conditions.

</details>

2. Why is `x.store(x.load(Relaxed) + 1, Relaxed)` wrong when several threads run it?

<details>
<summary>Answer</summary>

It is a read-modify-write split into two separate atomic steps. Another thread can read the same old value between the `load` and the `store`, so two increments produce one change: an update is lost. Use `fetch_add`.

</details>

3. A function locks a mutex to check a value, unlocks it, then locks it again to change the value. What kind of bug is this?

<details>
<summary>Answer</summary>

Check-then-act. Between the two locks, another thread can change the value, so the check is out of date when the action happens. One guard must cover both the check and the action.

</details>

4. Name the four conditions for a deadlock. Which one does a global lock order break?

<details>
<summary>Answer</summary>

Mutual exclusion, hold and wait, no preemption, circular wait. A global lock order breaks circular wait: threads only wait for locks with higher numbers, so no circle can form.

</details>

5. A concurrency test passed 1,000 times in a row. Is the code correct?

<details>
<summary>Answer</summary>

Not necessarily. The bad interleaving may be rare, or impossible on this machine and build, but likely on another one. Passing tests increase confidence, but correctness has to come from the design (critical sections around invariants, lock order), and tools like `loom` can check all interleavings of small models.

</details>

6. What is the difference between deadlock and livelock?

<details>
<summary>Answer</summary>

In a deadlock, threads are blocked and do nothing. In a livelock, threads are active (they retry, release and take locks again) but still make no progress.

</details>

## Summary

- The scheduler can switch threads between any two steps. Concurrent code must be correct for **every** interleaving, not only for the ones you saw.
- Safe Rust stops **data races** at compile time. It cannot stop **race conditions**, which are logic bugs in how safe steps are combined.
- A **lost update** happens when a read-modify-write is split into separate steps. Use `fetch_add` or one critical section.
- **Check-then-act** breaks when the lock is released between the check and the action. One guard must cover the whole invariant.
- A **deadlock** needs four conditions at once. A **global lock order** breaks circular wait and is the simplest cure.
- Concurrency bugs hide from tests. Test invariants with many threads, detect hangs with a watchdog or a time limit, and use `loom` (Lessons 14.6 and 14.9).

## Sources and further reading

- [The Rust Programming Language — 16.3 Shared-State Concurrency](https://doc.rust-lang.org/book/ch16-03-shared-state.html) — the official chapter on `Mutex` and `Arc`.
- [The Rustonomicon — Data Races and Race Conditions](https://doc.rust-lang.org/nomicon/races.html) — the precise definition of what safe Rust prevents and what it does not.
- [std::sync::Mutex](https://doc.rust-lang.org/std/sync/struct.Mutex.html) — the documentation, including what happens when a thread locks the same mutex twice.
- [Rust Atomics and Locks, chapter 1 — Basics of Rust Concurrency](https://marabos.nl/atomics/basics.html) — a free book by Mara Bos; this chapter reviews threads, locks and shared state.
- [Operating Systems: Three Easy Pieces — Common Concurrency Problems](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-bugs.pdf) — a free textbook chapter with real deadlock and race bugs from large programs.

---

[🏠 Start](../START-HERE.md) · [← Phase 14 — Concurrency and async in depth](README.md) · [📋 Phase 14](README.md) · [14.2 Mutexes and read-write locks in depth →](02-mutexes-and-rwlocks-in-depth.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
