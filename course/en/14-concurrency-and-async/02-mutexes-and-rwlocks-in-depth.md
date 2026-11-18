<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.1 Why concurrency is hard: races and deadlocks](01-why-concurrency-is-hard.md) · [📋 Phase 14](README.md) · [14.3 Condition variables →](03-condition-variables.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/02-mutexes-and-rwlocks-in-depth.md) <!-- nav -->

# Lesson 14.2 — Mutexes and read-write locks in depth

> **Phase 14 — Concurrency and async in depth** · Lesson 2 of 16 · ⏱ about 135 minutes

## What you will learn

- What a mutex does inside: the fast path without waiting and the slow path where the OS (Operating System) puts a thread to sleep.
- Why a lock that many threads fight for makes a program slower, and how to measure it.
- How long a guard really lives, and the hidden deadlock in `match` and `if let`.
- How to use `try_lock`, how to recover from a poisoned lock, and when you do not need to lock at all.
- When a read-write lock (`RwLock`) helps, when it hurts, and how splitting one lock into many (sharding) helps.

## Before you start

- You finished [Lesson 14.1 — Why concurrency is hard](01-why-concurrency-is-hard.md). You know critical sections, invariants and deadlocks.
- You know `Mutex`, guards and poisoning from [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), and how to measure time from [Lesson 05.12 — Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md).
- Keep working in `~/projects/rust/code/p14-concurrency`; each example is a new file in `src/bin/`.

## New words

| Word | Meaning |
|---|---|
| **contention** | Several threads want the same lock at the same time. A lock is *contended* when there is contention and *uncontended* when only one thread uses it. |
| **fast path / slow path** | The quick case (the lock is free: take it with one atomic operation) and the expensive case (the lock is busy: ask the OS to put the thread to sleep). |
| **futex** (fast user-space mutex) | A Linux system call that lets a thread sleep until a number in memory changes, and wakes it up later. Rust's locks use it on Linux. |
| **POSIX** (Portable Operating System Interface) | A standard set of system functions that Unix-like systems (Linux, macOS) offer. |
| **pthread** (POSIX threads) | The standard thread and lock library of Unix-like systems. On macOS, Rust's `Mutex` uses the pthread mutex. |
| **ns** (nanosecond) | One billionth of a second. Taking a free lock costs a few nanoseconds. |
| **temporary** | A value that is created in the middle of an expression and has no name, for example the guard in `m.lock().unwrap().pop()`. |
| **scrutinee** | The expression that `match` or `if let` looks at: in `match x.pop() { … }` it is `x.pop()`. |
| **`try_lock`** | A method that takes the lock only if it is free right now, and otherwise returns an error immediately instead of waiting. |
| **read-write lock** (`RwLock`) | A lock that allows **many readers at once, or one writer**, but never both. |
| **writer starvation** | Readers keep the lock busy all the time, so a writer waits for a very long time. |
| **fairness** | Whether a lock gives itself to waiting threads in the order they came. Rust's standard locks do not promise fairness. |
| **lock granularity** | How much data one lock protects. *Coarse*: one lock for everything. *Fine*: many locks, each for a small part. |
| **sharding** (lock striping) | Splitting data into several parts, each with its own lock, so threads that touch different parts do not wait for each other. |

## 1. What happens inside a mutex

A mutex is a small number in memory, plus a way to wait. Taking a lock has two cases:

```text
 lock():
   is the number "unlocked"?  ── yes ──▶ set it to "locked" with ONE atomic operation   (fast path, a few ns)
              │
              no
              ▼
   ask the OS: "put me to sleep until the lock is released"                            (slow path)
   ...sleep...
   the owner calls unlock() and the OS wakes one sleeping thread, which tries again
```

The fast path never talks to the OS. It is one atomic instruction (Lesson 14.5 shows these instructions). The slow path costs much more: a **system call** ([Lesson 13.3](../13-operating-systems/03-system-calls.md)), a context switch away from the thread, and later a context switch back.

How the OS puts the thread to sleep depends on the system. On Linux, Rust's standard library uses the **futex** system call: "sleep while this number still has this value". On macOS it uses the **pthread** mutex of the system library, which does the same job with the macOS kernel. You never call these yourself; `Mutex::lock` does it.

Let us measure the difference. Create `src/bin/lock_cost.rs`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Instant;

const TOTAL: u64 = 4_000_000;

fn main() {
    // 1. One thread: nobody else wants the lock (uncontended).
    let counter = Mutex::new(0u64);
    let start = Instant::now();
    for _ in 0..TOTAL {
        *counter.lock().unwrap() += 1;
    }
    let one = start.elapsed().as_nanos() as f64 / TOTAL as f64;
    println!("1 thread : {one:>5.1} ns per lock");

    // 2. Four threads fight for the same lock (contended).
    let counter = Arc::new(Mutex::new(0u64));
    let start = Instant::now();
    let handles: Vec<_> = (0..4)
        .map(|_| {
            let counter = Arc::clone(&counter);
            thread::spawn(move || {
                for _ in 0..TOTAL / 4 {
                    *counter.lock().unwrap() += 1;
                }
            })
        })
        .collect();
    for handle in handles {
        handle.join().unwrap();
    }
    let four = start.elapsed().as_nanos() as f64 / TOTAL as f64;
    println!("4 threads: {four:>5.1} ns per lock");
    println!("total = {}", *counter.lock().unwrap());
}
```

Always measure in release mode: `cargo run --release --bin lock_cost`.

```text
1 thread :   4.2 ns per lock
4 threads:  10.7 ns per lock
total = 4000000
```

**What happens:**

1. Both parts do the same work: 4 million increments, each inside its own lock and unlock.
2. With one thread the lock is always free, so every `lock()` takes the fast path: about 4 ns.
3. With four threads the work is split, but the lock lets only one thread in at a time. So the work is still done one increment after another, *plus* the threads now wait for each other, wake each other up, and move the *cache line* of the lock (the small block of memory, 128 bytes on Apple M chips, that a core copies into its own cache, [Lesson 12.9](../12-computer-systems/09-memory-hierarchy-and-caches.md)) between CPU (Central Processing Unit) cores. The whole job took about **2.5 times longer** with four threads than with one.
4. Your numbers will be different (they depend on the machine and on what else is running), but the shape stays: a hot, shared lock does not get faster with more threads.

> ⚡ **Performance:** A free lock is cheap; a *contended* lock is expensive. When a profile shows threads waiting for a lock, do not add threads. Make the critical section shorter (prepare the data before you lock), take the lock less often (add up locally, then update once), or split the lock (section 5).

## 2. How long does a guard live?

A guard unlocks when it is **dropped**. A guard with a name (`let guard = m.lock().unwrap();`) is dropped at the end of its block, or earlier with `drop(guard)`. A guard without a name is a **temporary**, and the rules for temporaries hold a trap.

This program takes a job from a list and wants to add a follow-up job. Create `src/bin/match_lock.rs`:

```rust
use std::sync::Mutex;

fn main() {
    let jobs = Mutex::new(vec![1, 2, 3]);

    match jobs.lock().unwrap().pop() {
        Some(job) => {
            println!("got job {job}, adding a follow-up job");
            jobs.lock().unwrap().push(job * 10); // hangs here
        }
        None => println!("no jobs"),
    }
    println!("jobs: {:?}", jobs.lock().unwrap());
}
```

```text
got job 3, adding a follow-up job
```

Then nothing. The program hangs forever (stop it with `Ctrl+C`).

**What happens:**

1. `jobs.lock().unwrap()` creates a guard without a name. `.pop()` is called through it.
2. The guard is part of the **scrutinee** of `match` (the expression after `match`). Rust keeps temporaries of the scrutinee alive **until the end of the whole `match`**, because a pattern could still borrow from them.
3. So inside the `Some(job)` arm the first guard is still alive and the mutex is still locked. The second `jobs.lock()` waits for a lock that this same thread holds: a self-deadlock, like in Lesson 14.1.
4. `if let Some(job) = jobs.lock().unwrap().pop() { … }` has the same problem: in edition 2024 the temporary is dropped before the `else` block, but it is still alive inside the `if` block. Try it: it hangs in the same place.

The fix is to end the guard's life with a `let` statement. A temporary in a `let` lives only until the `;`:

```rust
use std::sync::Mutex;

fn main() {
    let jobs = Mutex::new(vec![1, 2, 3]);

    let next = jobs.lock().unwrap().pop(); // the guard is dropped at the `;`
    match next {
        Some(job) => {
            println!("got job {job}, adding a follow-up job");
            jobs.lock().unwrap().push(job * 10);
        }
        None => println!("no jobs"),
    }
    println!("jobs: {:?}", jobs.lock().unwrap());
}
```

```text
got job 3, adding a follow-up job
jobs: [1, 2, 30]
```

> ⚠️ **Common mistake:** `match m.lock().unwrap().something() { … }` and `if let … = m.lock().unwrap()…` keep the lock for the whole block. The compiler does not warn you. Put the locked call into its own `let` first.

> 📏 **Convention:** Make the lifetime of every guard obvious. Either give it a name and keep the block short, or call `drop(guard)` at the point where the critical section ends.

## 3. `try_lock`, poisoning and locking without locking

Sometimes waiting is not acceptable: a thread that draws the screen, or a request that has a deadline. `try_lock` takes the lock only if it is free **now**. When the lock is busy it returns `Err(TryLockError::WouldBlock)` at once.

You already know **poisoning** from Lesson 05.10: if a thread panics while it holds the guard, the mutex is marked poisoned, and later `lock()` calls return `Err`. That error is not a dead end. It carries the guard inside, and you can take it out when you decide the data is still usable. Create `src/bin/try_and_poison.rs`:

```rust
use std::sync::{Arc, Mutex, TryLockError};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    // try_lock: do not wait if the lock is busy.
    let guard = data.lock().unwrap();
    match data.try_lock() {
        Ok(_) => println!("try_lock: got it"),
        Err(TryLockError::WouldBlock) => println!("try_lock: busy, do something else"),
        Err(TryLockError::Poisoned(_)) => println!("try_lock: poisoned"),
    }
    drop(guard); // end the critical section early

    // A worker panics in the middle of an update.
    let worker_data = Arc::clone(&data);
    let result = thread::spawn(move || {
        let mut v = worker_data.lock().unwrap();
        v.push(4);
        panic!("worker crashed after push");
    })
    .join();
    println!("worker failed: {}", result.is_err());

    // Recover: take the data out of the PoisonError and look at it.
    let v = data.lock().unwrap_or_else(|poisoned| poisoned.into_inner());
    println!("data after the crash: {:?}", *v);
    drop(v);

    data.clear_poison();
    println!("poisoned now: {}", data.is_poisoned());
}
```

```text
try_lock: busy, do something else

thread '<unnamed>' (5404093) panicked at src/bin/try_and_poison.rs:21:9:
worker crashed after push
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
worker failed: true
data after the crash: [1, 2, 3, 4]
poisoned now: false
```

**What happens:**

1. The main thread holds `guard`, so `try_lock` on the same mutex cannot succeed. It does not hang (unlike `lock()` in Lesson 14.1): it returns `WouldBlock` immediately.
2. `drop(guard)` releases the lock before the worker starts. Without it the worker would wait forever.
3. The worker pushes `4` and panics while holding the guard. The panic message goes to standard error; the number in brackets is the thread's ID and will differ for you.
4. `lock()` now returns `Err(PoisonError)`. `unwrap_or_else(|poisoned| poisoned.into_inner())` takes the guard out of the error. We see `[1, 2, 3, 4]`: the push did happen before the panic.
5. `clear_poison()` removes the mark after you have checked or repaired the data.

When is recovery right? When every single change keeps the data valid (pushing to a list, setting a counter). When is it wrong? When the panic could have stopped a multi-step update half-way, as in the bank of Lesson 14.1. Then the data cannot be trusted, and `unwrap()` (passing the panic on) is the honest choice.

Two methods do not lock at all, because the compiler can prove that nobody else can use the mutex:

- `mutex.get_mut()` takes `&mut self` and returns `&mut T`. If you have `&mut Mutex<T>`, no other thread can have a reference to it, so no locking is needed.
- `mutex.into_inner()` consumes the mutex and returns the data, for example after all threads have been joined.

> 💡 **Tip:** `try_lock` is useful for "do it if you can, skip it if you cannot" work, such as an optional statistics update. Do not build a retry loop out of it without a random pause; that is how livelock begins (Lesson 14.1).

## 4. Read-write locks

Many programs read shared data much more often than they change it: configuration, a routing table, a cache. A mutex lets only one reader in at a time, although readers cannot harm each other. A **read-write lock** fixes that:

```text
 RwLock state         who may enter
 ─────────────        ─────────────────────────────
 free                 one writer OR any number of readers
 N readers inside     more readers; writers wait
 1 writer inside      nobody else
```

`std::sync::RwLock<T>` has two methods instead of `lock`: `read()` returns a guard that gives `&T`, and `write()` returns a guard that gives `&mut T`. Many read guards can exist at the same time; a write guard is exclusive. Poisoning works like in `Mutex` (only a panicking *writer* poisons the lock).

Is it always faster? Let us measure. Eight threads each read a vector 20,000 times and sum it, once under a `Mutex` and once under an `RwLock`; we try a short vector (10 numbers) and a long one (10,000 numbers). Create `src/bin/rw_vs_mutex.rs`:

```rust
use std::hint::black_box;
use std::sync::{Mutex, RwLock};
use std::thread;
use std::time::{Duration, Instant};

const THREADS: usize = 8;
const READS: usize = 20_000;

/// Runs `read_once` READS times in each of THREADS threads; returns the time.
fn time_readers(read_once: impl Fn() -> u64 + Sync) -> Duration {
    let start = Instant::now();
    thread::scope(|s| {
        for _ in 0..THREADS {
            s.spawn(|| {
                for _ in 0..READS {
                    black_box(read_once());
                }
            });
        }
    });
    start.elapsed()
}

fn main() {
    for len in [10, 10_000] {
        let data: Vec<u64> = (0..len).collect();
        let mutex = Mutex::new(data.clone());
        let rwlock = RwLock::new(data);

        let m = time_readers(|| mutex.lock().unwrap().iter().sum());
        let r = time_readers(|| rwlock.read().unwrap().iter().sum());
        println!("vector of {len:>6}: Mutex {m:>9.2?}   RwLock {r:>9.2?}");
    }
}
```

`cargo run --release --bin rw_vs_mutex`, two runs:

```text
vector of     10: Mutex    2.08ms   RwLock   27.00ms
vector of  10000: Mutex  234.56ms   RwLock   24.57ms
vector of     10: Mutex    2.16ms   RwLock   23.99ms
vector of  10000: Mutex  183.26ms   RwLock   21.55ms
```

**What happens:**

1. `time_readers` takes a closure that does one read, runs it in 8 scoped threads ([Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)) and measures the time. The closure must be `Sync` because all threads share it. `black_box` (from Lesson 05.12) stops the compiler from deleting a result that nobody uses.
2. **Long reads** (10,000 numbers): with a `Mutex` the 8 threads sum one after another; with an `RwLock` they sum at the same time. The `RwLock` is about **8–10 times faster**: close to the number of threads.
3. **Short reads** (10 numbers): the `RwLock` is about **10 times slower**! Each `read()` must change a shared *reader counter* (one more reader, one fewer reader) with atomic operations. When the work inside is tiny, all threads fight over that counter, and its cache line jumps between cores all the time. The plain mutex wins.
4. Lesson: an `RwLock` pays off when reads are **long** and **many** and writes are **rare**. For tiny critical sections, a `Mutex` is usually better. Measure your case.

Two more facts from the documentation of `RwLock`:

- **The policy depends on the OS.** Whether waiting writers block new readers (which prevents writer starvation) is not promised. On some systems a constant stream of readers can make a writer wait a long time.
- **Do not take a read lock twice in one thread.** If a writer starts waiting between your two `read()` calls, a writer-preferring lock blocks your second `read()`, the writer waits for your first one, and you have a deadlock.

> 🧪 **Testing:** An `RwLock` makes it easy to update **several values as one unit**: change them all under one write guard, and readers can never see half of the change. Exercise 2 tests exactly that.

## 5. One lock or many: granularity and sharding

**Lock granularity** is how much data one lock protects. One lock around a whole `HashMap` is simple and correct, but every thread waits for every other thread, even when they touch different keys.

**Sharding** (also called *lock striping*) splits the data into N parts called *shards*, each with its own lock. A key always goes to the same shard, chosen by its hash:

```text
               hash("apple") % 4 = 1
 key "apple" ───────────────────────────▶ shard 1: Mutex<HashMap>
 key "pear"  ──── hash % 4 = 3 ─────────▶ shard 3: Mutex<HashMap>
 key "plum"  ──── hash % 4 = 1 ─────────▶ shard 1: Mutex<HashMap>  (waits only for "apple")

 shard 0: Mutex<HashMap>   shard 2: Mutex<HashMap>   (free for other keys)
```

Threads that work on keys in different shards never wait for each other. With 16 shards and 8 threads, most lock calls take the fast path. You will build one in Exercise 3. The price: an operation that needs *all* data (for example "count all keys") must lock every shard, and if it needs a consistent picture, it must hold all the locks at once, in a fixed order (Lesson 14.1).

> 💡 **Tip:** The crate `parking_lot` (version `"0.13"`) offers `Mutex` and `RwLock` types that are a little smaller, have no poisoning (`lock()` returns the guard directly), and add extras such as fair unlocking. The standard locks are good enough for most programs, and this course uses them.

## Common mistakes

1. **A guard in the scrutinee of `match` or `if let`.** The lock stays taken for the whole block; locking again inside hangs. Move the call into a `let` first (section 2).
2. **Doing slow work inside the critical section.** Reading a file, sending a network request or printing a lot while holding a lock makes every other thread wait. Prepare outside, lock, change, unlock.
3. **Using `RwLock` for tiny reads.** It can be much slower than a `Mutex` (section 4). Measure before you switch.
4. **Recovering from poisoning without thinking.** `into_inner()` on a `PoisonError` gives you data that may be half-updated. Recover only when every change keeps the data valid.
5. **A `try_lock` retry loop without a pause.** Threads can keep failing at the same rhythm (livelock). Use `lock()`, or add a short random wait between retries.

## Try it yourself

1. Run `cargo run --release --bin lock_cost` several times. Then change the number of threads from 4 to 8 (and `TOTAL / 4` to `TOTAL / 8`). Is the job faster than with one thread? (On the test machine: `1 thread : 4.1 ns`, `8 threads: 10.0 ns`. No.)
2. Copy `match_lock.rs` to `iflet_lock.rs` and write the same logic with `if let Some(job) = jobs.lock().unwrap().pop() { … }`. Run it and confirm that it hangs at the same place. Stop it with `Ctrl+C`.
3. In `rw_vs_mutex.rs`, add `1_000` to the list of lengths: `for len in [10, 1_000, 10_000]`. Where does the `RwLock` start to win on your machine? On the test machine, 1,000 numbers was still a loss for the `RwLock` (`Mutex 20.15ms`, `RwLock 43.89ms`).

## Exercises

### Exercise 1 — Two guards in one line (easy)

This function should move the last element of `a` to the end of `b`. The `main` below works, but the function hangs when it is called as `move_last(&a, &a)`, and it can deadlock with another thread that runs `move_last(&b, &a)` at the same time. Explain both problems, then rewrite it so that it never holds two locks at once.

```rust
use std::sync::Mutex;

fn move_last(a: &Mutex<Vec<i32>>, b: &Mutex<Vec<i32>>) {
    if let Some(x) = a.lock().unwrap().pop() {
        b.lock().unwrap().push(x);
    }
}

fn main() {
    let a = Mutex::new(vec![1, 2, 3]);
    let b = Mutex::new(vec![]);
    move_last(&a, &b);
    println!("a = {:?}, b = {:?}", a.lock().unwrap(), b.lock().unwrap());
}
```

<details>
<summary>Hint</summary>

Inside the `if let` block, the guard of `a` is still alive (section 2). What if the function first takes the element out, ends that critical section, and only then locks `b`?

</details>

<details>
<summary>Answer</summary>

Inside the `if let` block, the temporary guard of `a` is still alive, so the function holds `a` while it locks `b`. If `a` and `b` are the same mutex, it locks the same mutex twice and hangs. If another thread runs `move_last(&b, &a)` at the same time, the two threads take the locks in opposite order: a deadlock. The element does not need both locks at once:

```rust
use std::sync::Mutex;

fn move_last(a: &Mutex<Vec<i32>>, b: &Mutex<Vec<i32>>) {
    let item = a.lock().unwrap().pop(); // guard of `a` dropped here
    if let Some(x) = item {
        b.lock().unwrap().push(x);
    }
}

fn main() {
    let a = Mutex::new(vec![1, 2, 3]);
    let b = Mutex::new(vec![]);
    move_last(&a, &b);
    println!("a = {:?}, b = {:?}", a.lock().unwrap(), b.lock().unwrap());
}
```

```text
a = [1, 2], b = [3]
```

Why it works: each lock is held alone, so no thread ever waits for a second lock while it holds a first one: condition 2 of Lesson 14.1 ("hold and wait") is broken. Note the trade-off: between the two critical sections another thread can see the element in neither vector. If that matters for your invariant, you need both locks in a fixed order instead.

</details>

### Exercise 2 — Readers never see half an update (medium)

Write `src/bin/config.rs` with a `Config` type that holds `RwLock<HashMap<String, String>>` and has `get(&self, key) -> Option<String>` and `set_many(&self, pairs: &[(&str, &str)])`, which sets all pairs under **one** write guard. Then write a test: one writer thread sets `a` and `b` to the same value (`"1"`, `"2"`, …, `"1000"`), while four reader threads check 1,000 times, under one read guard, that `a == b`. At the end `a` must be `"1000"`.

<details>
<summary>Hint</summary>

Use `thread::scope` so that the threads can borrow `config` without `Arc`. In the reader, take `let map = config.values.read().unwrap();` once and compare `map["a"]` with `map["b"]` through the same guard. Two separate `get` calls would be two critical sections.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;
use std::sync::RwLock;

struct Config {
    values: RwLock<HashMap<String, String>>,
}

impl Config {
    fn new() -> Self {
        Config {
            values: RwLock::new(HashMap::new()),
        }
    }

    fn get(&self, key: &str) -> Option<String> {
        self.values.read().unwrap().get(key).cloned()
    }

    /// Sets several keys in ONE write lock: readers see all or nothing.
    fn set_many(&self, pairs: &[(&str, &str)]) {
        let mut map = self.values.write().unwrap();
        for (k, v) in pairs {
            map.insert(k.to_string(), v.to_string());
        }
    }
}

fn main() {
    let config = Config::new();
    config.set_many(&[("host", "localhost"), ("port", "8080")]);
    println!("{:?} {:?}", config.get("host"), config.get("port"));
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn readers_never_see_half_an_update() {
        let config = Config::new();
        config.set_many(&[("a", "0"), ("b", "0")]);
        thread::scope(|s| {
            s.spawn(|| {
                for i in 1..=1000 {
                    let v = i.to_string();
                    config.set_many(&[("a", &v), ("b", &v)]);
                }
            });
            for _ in 0..4 {
                s.spawn(|| {
                    for _ in 0..1000 {
                        let map = config.values.read().unwrap(); // one read lock
                        assert_eq!(map["a"], map["b"]);
                    }
                });
            }
        });
        assert_eq!(config.get("a").as_deref(), Some("1000"));
    }
}
```

```text
running 1 test
test tests::readers_never_see_half_an_update ... ok
```

`cargo run --bin config` prints `Some("localhost") Some("8080")`.

Why it works: the writer changes `a` and `b` inside one write guard, and a reader looks at both inside one read guard. The two critical sections can never overlap, so a reader sees either the old pair or the new pair, never a mix. A panic in a reader thread would make `thread::scope` panic too, so a failed `assert_eq!` fails the test.

</details>

### Exercise 3 — A sharded counter (hard)

Write `src/bin/sharded.rs` with `ShardedCounter`, which counts how often each key was added. It holds `Vec<Mutex<HashMap<String, u64>>>`; `new(shards)`, `add(&self, key)` and `get(&self, key) -> u64` (0 for unknown keys). Pick the shard with `DefaultHasher`. Test: 8 threads each add the 10 keys `key0`…`key9` 1,000 times; then every key must have the count 8,000.

<details>
<summary>Hint</summary>

```rust
let mut hasher = DefaultHasher::new();
key.hash(&mut hasher);                    // needs `use std::hash::{Hash, Hasher};`
let index = (hasher.finish() as usize) % self.shards.len();
```

Write a private helper `shard_for(&self, key) -> &Mutex<HashMap<String, u64>>` and use it in `add` and `get`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};
use std::sync::Mutex;

struct ShardedCounter {
    shards: Vec<Mutex<HashMap<String, u64>>>,
}

impl ShardedCounter {
    fn new(shards: usize) -> Self {
        ShardedCounter {
            shards: (0..shards).map(|_| Mutex::new(HashMap::new())).collect(),
        }
    }

    fn shard_for(&self, key: &str) -> &Mutex<HashMap<String, u64>> {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        let index = (hasher.finish() as usize) % self.shards.len();
        &self.shards[index]
    }

    fn add(&self, key: &str) {
        let mut map = self.shard_for(key).lock().unwrap();
        *map.entry(key.to_string()).or_insert(0) += 1;
    }

    fn get(&self, key: &str) -> u64 {
        self.shard_for(key)
            .lock()
            .unwrap()
            .get(key)
            .copied()
            .unwrap_or(0)
    }
}

fn main() {
    let counter = ShardedCounter::new(16);
    counter.add("apple");
    counter.add("apple");
    counter.add("pear");
    println!(
        "apple = {}, pear = {}, plum = {}",
        counter.get("apple"),
        counter.get("pear"),
        counter.get("plum")
    );
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn counts_are_exact_with_many_threads() {
        let counter = ShardedCounter::new(16);
        let keys: Vec<String> = (0..10).map(|i| format!("key{i}")).collect();
        thread::scope(|s| {
            for _ in 0..8 {
                s.spawn(|| {
                    for _ in 0..1000 {
                        for key in &keys {
                            counter.add(key);
                        }
                    }
                });
            }
        });
        for key in &keys {
            assert_eq!(counter.get(key), 8000, "wrong count for {key}");
        }
    }
}
```

```text
apple = 2, pear = 1, plum = 0
```

`cargo test --bin sharded` passes: `test tests::counts_are_exact_with_many_threads ... ok`.

Why it works: the same key always gives the same hash, so it always lands in the same shard, and each shard's `HashMap` is protected by its own mutex. Every single `add` is one critical section in one shard, so no update is lost, and threads that work on keys in different shards do not wait for each other.

</details>

## Quiz

1. What is the difference between the fast path and the slow path of `lock()`?

<details>
<summary>Answer</summary>

Fast path: the lock is free, and the thread takes it with one atomic operation in user space, in a few nanoseconds. Slow path: the lock is busy, so the thread asks the OS to put it to sleep (futex on Linux, the pthread mutex on macOS) and is woken up later: system calls and context switches, much slower.

</details>

2. Why does `match m.lock().unwrap().pop() { Some(x) => m.lock().unwrap().push(x), None => {} }` hang?

<details>
<summary>Answer</summary>

The guard is a temporary in the scrutinee of `match`, and such temporaries live until the end of the whole `match`. Inside the arm the mutex is still locked, so the second `lock()` waits for the same thread forever.

</details>

3. When is it right to recover from a poisoned mutex with `into_inner()`?

<details>
<summary>Answer</summary>

When every change to the data leaves it valid, so a panic in the middle cannot leave it broken (for example a counter or a list that only gets pushes). If a panic can interrupt a multi-step update, the invariant may be broken, and it is better to stop with `unwrap()`.

</details>

4. Why can an `RwLock` be slower than a `Mutex`, even when all threads only read?

<details>
<summary>Answer</summary>

Every `read()` and every drop of a read guard changes a shared reader counter with atomic operations. When the work inside the lock is tiny, the threads spend most of their time fighting over that counter's cache line. An `RwLock` helps when the reads themselves are long.

</details>

5. What does sharding a map buy you, and what does it cost?

<details>
<summary>Answer</summary>

Threads that work on keys in different shards do not wait for each other, so contention drops. The cost: operations over all keys must lock all shards (in a fixed order if they need a consistent picture), and the code is more complex.

</details>

## Summary

- Taking a free lock is cheap (one atomic operation). A contended lock is slow: threads sleep and wake through the OS, and the work is done one thread at a time anyway.
- More threads do not make a hot lock faster. Shorten the critical section, lock less often, or split the lock.
- A guard in the scrutinee of `match` or `if let` lives for the whole block. Put the locked call in a `let` or call `drop(guard)` when you are done.
- `try_lock` never waits. A poisoned lock still holds its data: `into_inner()` gives it back when you can trust it. `get_mut` and `into_inner` need no locking at all.
- `RwLock` lets many readers in at once. It wins for long reads and rare writes and can lose badly for tiny reads. Its writer policy depends on the OS.
- Sharding gives each part of the data its own lock, so threads on different parts do not wait for each other.

## Sources and further reading

- [std::sync::Mutex](https://doc.rust-lang.org/std/sync/struct.Mutex.html) — `lock`, `try_lock`, `get_mut`, `into_inner`, `clear_poison`, and the notes on poisoning.
- [std::sync::RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html) — the documentation, including the warning about OS-dependent priority and taking a read lock twice.
- [std::sync::PoisonError](https://doc.rust-lang.org/std/sync/struct.PoisonError.html) — how to get the guard out of a poisoned lock.
- [The Rust Reference — Destructors](https://doc.rust-lang.org/reference/destructors.html) — the exact rules for when temporaries are dropped, including the scrutinee of `match` and `if let`.
- [Rust Atomics and Locks, chapter 1 — Basics of Rust Concurrency](https://marabos.nl/atomics/basics.html) — Mara Bos explains locking, guards and `RwLock` with small examples.
- [parking_lot on docs.rs](https://docs.rs/parking_lot) — the alternative lock types mentioned in this lesson.

---

[🏠 Start](../START-HERE.md) · [← 14.1 Why concurrency is hard: races and deadlocks](01-why-concurrency-is-hard.md) · [📋 Phase 14](README.md) · [14.3 Condition variables →](03-condition-variables.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
