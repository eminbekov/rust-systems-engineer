<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.5 Atomics](05-atomics.md) · [📋 Phase 14](README.md) · [14.7 Building a spin lock →](07-building-a-spin-lock.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/06-memory-ordering.md) <!-- nav -->

# Lesson 14.6 — Memory ordering

> **Phase 14 — Concurrency and async in depth** · Lesson 6 of 16 · ⏱ about 135 minutes

## What you will learn

- Why other threads may see your writes in a different order than you wrote them, and why your arm64 Mac does this more often than many other machines.
- What *happens-before* means, and how a `Release` store and an `Acquire` load create it.
- What `SeqCst` adds, and the one classic case where you really need it.
- What `Relaxed`, `Release`, `Acquire` and `SeqCst` turn into on the CPU (Central Processing Unit) of your Mac.
- How the `loom` crate finds an ordering bug that a normal test misses, and how to choose an ordering.

## Before you start

- You finished [Lesson 14.5 — Atomics](05-atomics.md). You know `load`, `store`, `fetch_add`, `compare_exchange`, and that we used `Ordering::Relaxed` everywhere.
- You know caches and cache lines from [Lesson 12.9](../12-computer-systems/09-memory-hierarchy-and-caches.md) and can read a few arm64 instructions ([Lesson 12.7](../12-computer-systems/07-reading-assembly.md)).
- Keep working in `~/projects/rust/code/p14-concurrency`. For section 6 add `loom` as a *dev-dependency* (a crate used only by tests):

```bash
cargo add --dev loom@0.7
```

## New words

| Word | Meaning |
|---|---|
| **reordering** | Memory operations become visible to other threads in a different order than the program wrote them. The compiler and the CPU are both allowed to do it, within rules. |
| **memory model** | The rules of a language that say which values a load may return when threads share memory. Rust uses the same model as C++. |
| **strongly / weakly ordered** | A *strongly ordered* CPU reorders very little (x86-64). A *weakly ordered* CPU (arm64, the CPU of your Mac) reorders much more. |
| **x86-64** | The 64-bit CPU family of most PCs (personal computers) and many servers, made by Intel and AMD. |
| **store buffer** | A small queue inside each CPU core where writes wait before they reach the cache that other cores see. |
| **happens-before** | A guarantee between two operations: everything the first one did is visible to the second one. |
| **`Release`** | An ordering for a store: every memory operation before it in this thread is made visible no later than this store. |
| **`Acquire`** | An ordering for a load: if it reads a value written by a `Release` store, everything before that store is visible after this load. |
| **synchronizes-with** | The link that an `Acquire` load creates when it reads the value of a `Release` store. It gives a happens-before between the two threads. |
| **`AcqRel`** | `Acquire` and `Release` together, for read-modify-write operations such as `fetch_add` or `compare_exchange`. |
| **`SeqCst`** (sequentially consistent) | The strongest ordering: all `SeqCst` operations of all threads appear in **one** order that every thread agrees on. |
| **fence** | `std::sync::atomic::fence(ordering)`: an ordering rule without a memory access of its own. |
| **litmus test** | A tiny program of two or three threads that checks whether one particular reordering can happen. |
| **model checker** | A tool that runs a small concurrent program under every allowed interleaving and reordering. `loom` is one; Lesson 14.9 teaches it fully. |

## 1. The surprise: your writes travel separately

Here is a pattern you will write many times: one thread prepares **data**, then sets a **flag** that says "the data is ready"; another thread waits for the flag and then reads the data.

```text
 writer                      reader
 DATA = 42                   if READY == true:
 READY = true                    use DATA      ← expects 42
```

It looks obviously right. It is wrong unless you choose the orderings carefully, because **nothing promises that other threads see the two writes in program order**:

- The **compiler** may swap two writes to different variables, or two reads, if this thread alone would not notice. It does it to make code faster.
- The **CPU** does it too. Each core has a **store buffer**: writes wait there before they reach the cache that other cores read. A weakly ordered CPU may let a later write (`READY`) leave the buffer before an earlier one (`DATA`), for example because the cache line of `READY` was already in the core and the one of `DATA` was not. Loads can also run early.

Think of two letters posted one after another from two different post offices. You posted the first letter first, but nobody promises that it arrives first.

Within **one** thread everything still looks in order; that is a promise. The question is only what **other** threads see. The answer for `Relaxed` is: no promise at all about *other* variables.

## 2. See it happen on your Mac

A **litmus test** runs this pattern again and again and counts the bad outcome. The two threads must start each round at the same moment, so the program uses two extra counters to start and finish rounds; the interesting part is marked with the comments 1 to 4. Create `src/bin/message_passing.rs`:

```rust
use std::hint::spin_loop;
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use std::thread;

/// Keeps a value alone in its own 128-byte block (a cache line on Apple M chips).
#[repr(align(128))]
struct Padded<T>(T);

static DATA: Padded<AtomicU64> = Padded(AtomicU64::new(0));
static READY: Padded<AtomicBool> = Padded(AtomicBool::new(false));

const ROUNDS: u64 = 100_000;

/// Waits until `counter` has reached `r`.
fn wait_for(counter: &AtomicU64, r: u64) {
    while counter.load(Ordering::SeqCst) < r {
        spin_loop();
    }
}

/// In every round the writer stores DATA = r and then READY = true, while the
/// reader looks at READY and, if it is true, at DATA. Returns how often the
/// reader saw READY == true but an OLD value in DATA.
fn run(flag_store: Ordering, flag_load: Ordering) -> u64 {
    let start = AtomicU64::new(0); // the round that may start now
    let writer_done = AtomicU64::new(0);
    let reader_done = AtomicU64::new(0);
    thread::scope(|s| {
        s.spawn(|| {
            for r in 1..=ROUNDS {
                wait_for(&start, r);
                DATA.0.store(r, Ordering::Relaxed); // 1. write the data
                READY.0.store(true, flag_store); // 2. say "it is ready"
                writer_done.store(r, Ordering::SeqCst);
            }
        });
        let reader = s.spawn(|| {
            let mut stale = 0;
            for r in 1..=ROUNDS {
                wait_for(&start, r);
                let ready = READY.0.load(flag_load); // 3. is it ready?
                let data = DATA.0.load(Ordering::Relaxed); // 4. read the data
                if ready && data != r {
                    stale += 1; // "ready", but the data is old!
                }
                reader_done.store(r, Ordering::SeqCst);
            }
            stale
        });
        for r in 1..=ROUNDS {
            READY.0.store(false, Ordering::SeqCst); // reset for the new round
            start.store(r, Ordering::SeqCst);
            wait_for(&writer_done, r);
            wait_for(&reader_done, r);
        }
        reader.join().unwrap()
    })
}

fn main() {
    let relaxed = run(Ordering::Relaxed, Ordering::Relaxed);
    println!("Relaxed flag        : old data seen {relaxed} times in {ROUNDS} rounds");
    let rel_acq = run(Ordering::Release, Ordering::Acquire);
    println!("Release/Acquire flag: old data seen {rel_acq} times in {ROUNDS} rounds");
}
```

`cargo run --release --bin message_passing`, three runs:

```text
Relaxed flag        : old data seen 3685 times in 100000 rounds
Release/Acquire flag: old data seen 0 times in 100000 rounds
Relaxed flag        : old data seen 8935 times in 100000 rounds
Release/Acquire flag: old data seen 0 times in 100000 rounds
Relaxed flag        : old data seen 3093 times in 100000 rounds
Release/Acquire flag: old data seen 0 times in 100000 rounds
```

**What happens:**

1. `#[repr(align(128))]` puts `DATA` and `READY` into different cache lines, so the CPU can move them independently, as it would for two unrelated variables in a real program.
2. The main thread starts round `r` by resetting `READY` and setting `start = r`; it waits until both threads report that they finished the round. `wait_for` spins with `spin_loop()`, a hint to the CPU that we are busy-waiting (Lesson 14.7 explains it). The helper counters use `SeqCst`, the strongest ordering, so they cannot disturb the experiment.
3. In each round the writer stores `DATA = r` and then `READY = true`; at the same moment the reader reads `READY` and then `DATA`.
4. With `Relaxed` on the flag, the reader saw `READY == true` together with the **previous** round's `DATA` thousands of times: in a few percent of the rounds. Your numbers will differ, but on an Apple M chip they are rarely zero. The same happens in a debug build (`cargo run --bin message_passing`), which does not optimize, so it is the **CPU** that reorders here.
5. With `Release` on the flag store and `Acquire` on the flag load, the bad outcome **never** happens. Section 3 explains why.

> ⚠️ **Common mistake:** "It works on my machine, so the ordering is fine." On an x86-64 PC this exact `Relaxed` program would almost never show old data, because x86-64 hardware keeps stores in order and loads in order. The program is still wrong there: the *compiler* may reorder it, and the next CPU may not be so kind. Write code that is correct by Rust's rules, not by your current hardware. Your weakly ordered Mac is a good place to find such bugs.

## 3. Happens-before: `Release` and `Acquire`

The rules of the memory model are built on one idea: **happens-before**. If operation A happens-before operation B, then B sees everything A did (and everything before A).

Inside one thread, every operation happens-before the next one in program order. Between threads, you create happens-before with a pair:

- a **`Release` store**: "make everything I did before this store visible no later than the store itself";
- an **`Acquire` load** that **reads the value of that store**: "after this load, I see everything the other thread did before its release".

When the acquire load reads the released value, the two operations **synchronize-with** each other, and everything before the release happens-before everything after the acquire:

```text
 writer thread                              reader thread
 ─────────────                              ─────────────
 DATA.store(42, Relaxed)                    
        │  (program order)                  
 READY.store(true, Release) ─ ─ ─ ─ ─ ─ ─▶ READY.load(Acquire) reads true
                         synchronizes-with          │  (program order)
                                            DATA.load(Relaxed) → must see 42
```

That is why the release/acquire version never saw old data. Notice three details:

1. `DATA` itself can stay `Relaxed`. The flag carries the ordering for everything before it.
2. The guarantee only exists **if the acquire load reads the released value**. If the reader sees `READY == false`, it learns nothing about `DATA`; it must not use it. So: always check the flag, then read the data.
3. It is a pair. A `Release` store with a `Relaxed` load, or a `Relaxed` store with an `Acquire` load, gives no guarantee.

This is exactly how a mutex works inside: `lock()` ends with an **acquire** operation, `unlock()` (dropping the guard) is a **release** operation. So everything one thread did while holding the lock happens-before everything the next owner does. That is why data inside a `Mutex` is always seen correctly, and why the latch of Lesson 14.3 (Exercise 3) made the main thread see the counter updates. In Lesson 14.7 you will write these two orderings yourself.

For read-modify-write operations (`swap`, `fetch_add`, `compare_exchange`) there is **`AcqRel`**: the read part is `Acquire` and the write part is `Release`. Use it when one operation must both receive and pass on data, for example a counter where the last thread to arrive reads results that the others published.

> 📏 **Convention:** Next to every non-`Relaxed` ordering, write a short comment that names its partner: `// Release: pairs with the Acquire load in take()`. Orderings without a partner are a sign of a bug.

## 4. `SeqCst`: one order for everyone

Release/acquire is about **pairs** of threads. Some algorithms need more. Here is the classic one, called *store buffering*: thread A sets `X = 1` and then reads `Y`; thread B sets `Y = 1` and then reads `X`. Can **both** read 0? In program order it looks impossible: whoever runs second must see the first one's write. Create `src/bin/store_buffer.rs`:

```rust
use std::hint::spin_loop;
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

#[repr(align(128))]
struct Padded<T>(T);

static X: Padded<AtomicU64> = Padded(AtomicU64::new(0));
static Y: Padded<AtomicU64> = Padded(AtomicU64::new(0));

const ROUNDS: u64 = 100_000;

fn wait_for(counter: &AtomicU64, r: u64) {
    while counter.load(Ordering::SeqCst) < r {
        spin_loop();
    }
}

/// Thread A: X = 1, then read Y.   Thread B: Y = 1, then read X.
/// Returns how often BOTH threads read 0.
fn run(store: Ordering, load: Ordering) -> u64 {
    let start = AtomicU64::new(0);
    let a_done = AtomicU64::new(0);
    let b_done = AtomicU64::new(0);
    let (seen_a, seen_b) = (AtomicU64::new(0), AtomicU64::new(0));
    let mut both_zero = 0;
    thread::scope(|s| {
        s.spawn(|| {
            for r in 1..=ROUNDS {
                wait_for(&start, r);
                X.0.store(1, store);
                seen_a.store(Y.0.load(load), Ordering::SeqCst);
                a_done.store(r, Ordering::SeqCst);
            }
        });
        s.spawn(|| {
            for r in 1..=ROUNDS {
                wait_for(&start, r);
                Y.0.store(1, store);
                seen_b.store(X.0.load(load), Ordering::SeqCst);
                b_done.store(r, Ordering::SeqCst);
            }
        });
        for r in 1..=ROUNDS {
            X.0.store(0, Ordering::SeqCst);
            Y.0.store(0, Ordering::SeqCst);
            start.store(r, Ordering::SeqCst);
            wait_for(&a_done, r);
            wait_for(&b_done, r);
            if seen_a.load(Ordering::SeqCst) == 0 && seen_b.load(Ordering::SeqCst) == 0 {
                both_zero += 1;
            }
        }
    });
    both_zero
}

fn main() {
    let relaxed = run(Ordering::Relaxed, Ordering::Relaxed);
    println!("Relaxed        : both read 0 in {relaxed} of {ROUNDS} rounds");
    let rel_acq = run(Ordering::Release, Ordering::Acquire);
    println!("Release/Acquire: both read 0 in {rel_acq} of {ROUNDS} rounds");
    let seq_cst = run(Ordering::SeqCst, Ordering::SeqCst);
    println!("SeqCst         : both read 0 in {seq_cst} of {ROUNDS} rounds");
}
```

`cargo run --release --bin store_buffer`, two runs:

```text
Relaxed        : both read 0 in 31199 of 100000 rounds
Release/Acquire: both read 0 in 8 of 100000 rounds
SeqCst         : both read 0 in 0 of 100000 rounds
Relaxed        : both read 0 in 31251 of 100000 rounds
Release/Acquire: both read 0 in 53 of 100000 rounds
SeqCst         : both read 0 in 0 of 100000 rounds
```

**What happens:**

1. With `Relaxed`, both threads read 0 in about 30% of the rounds. Each thread's store waits in its core's store buffer while its load already reads the old value from the cache.
2. With `Release`/`Acquire` it still happens (rarely, but it does). That is allowed: a release store only orders what came **before** it, and an acquire load only orders what comes **after** it. Nothing forbids a later acquire load from moving before an earlier release store. And neither thread here reads a value the other one released, so no synchronizes-with link exists at all.
3. With `SeqCst` it never happens. `SeqCst` adds one rule: all `SeqCst` operations of all threads form **one single order** that every thread agrees on. In that order one of the two stores comes first, and the other thread's load comes after it, so at least one thread must see a 1.

When do you need `SeqCst`? When correctness depends on several threads agreeing on the order of operations on **different** variables, as here ("I raised my flag, so I can check yours"). That is rare in practice: flags with data, locks, reference counts and channels all work with release/acquire. Many people use `SeqCst` "to be safe"; it is always correct, but it hides the reasoning, and it can be slower.

`std::sync::atomic::fence(ordering)` is an ordering without its own memory access. `fence(SeqCst)` between the store and the load in each thread above also forbids the "both zero" outcome; you will use it in Exercise 3.

## 5. What it means on your Mac

What does each ordering become on arm64? Here is what the compiler produced for small functions that do one `store` or one `load` on an `AtomicU64` with each ordering (made with `--emit asm` as in Lesson 14.5; the full program is in Try it yourself, step 3):

| Rust | arm64 instruction | Meaning |
|---|---|---|
| `store(…, Relaxed)` | `str` | a normal store; may become visible later than later stores |
| `store(…, Release)` | `stlr` | "store-release": all earlier loads and stores are visible before it |
| `store(…, SeqCst)` | `stlr` | the same instruction |
| `load(Relaxed)` | `ldr` | a normal load; may run earlier than earlier loads |
| `load(Acquire)` | `ldapr` | "load-acquire": no later load or store may run before it |
| `load(SeqCst)` | `ldar` | like `ldapr`, and also may not move before an earlier `stlr` |

So the orderings are **real instructions** with real costs, not just notes for the compiler. The difference between `ldapr` and `ldar` is exactly the store-buffering case of section 4: an `ldapr` may run before an earlier `stlr` of the same thread, an `ldar` may not.

> ⚡ **Performance:** On arm64, `Relaxed` is the cheapest, release/acquire costs a little more, and `SeqCst` can cost more again, because it forbids more reordering. Do not optimize orderings by guessing: first make the code correct with clear release/acquire pairs, then measure (Lesson 05.12) before weakening anything.

## 6. Testing orderings with loom

The litmus programs needed 100,000 rounds and luck to show the bug. A normal test of the flag pattern would usually pass. **loom** does not rely on luck: it runs a small test again and again and makes each run take a different **interleaving**, and, for atomics, also lets loads return any older value that the memory model allows. You use loom's own versions of `Arc`, `thread` and the atomic types inside `loom::model(|| { … })`.

Here is a first look (Lesson 14.9 shows how to set loom up for a whole crate). Create the integration-test file `tests/loom_flag.rs`:

```rust
use loom::sync::Arc;
use loom::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use loom::thread;

/// The writer stores the data and then the flag; the reader checks both.
fn publish(flag_store: Ordering, flag_load: Ordering) {
    loom::model(move || {
        let data = Arc::new(AtomicU64::new(0));
        let ready = Arc::new(AtomicBool::new(false));

        let (d, r) = (Arc::clone(&data), Arc::clone(&ready));
        let writer = thread::spawn(move || {
            d.store(42, Ordering::Relaxed);
            r.store(true, flag_store);
        });

        if ready.load(flag_load) {
            assert_eq!(data.load(Ordering::Relaxed), 42, "ready, but old data");
        }
        writer.join().unwrap();
    });
}

#[test]
fn relaxed_flag() {
    publish(Ordering::Relaxed, Ordering::Relaxed);
}

#[test]
fn release_acquire_flag() {
    publish(Ordering::Release, Ordering::Acquire);
}
```

Run it in release mode (loom runs the model many times, so it likes optimized code): `cargo test --release --test loom_flag`.

```text
running 2 tests
test relaxed_flag ... FAILED
test release_acquire_flag ... ok

failures:

---- relaxed_flag stdout ----

thread 'relaxed_flag' (5774265) panicked at tests/loom_flag.rs:18:13:
assertion `left == right` failed: ready, but old data
  left: 0
 right: 42
```

**What happens:**

1. `loom::model` runs the closure many times. In each run loom decides which thread runs when, and which allowed value each atomic load returns.
2. With `Relaxed`, loom found a run in which the reader saw `ready == true` but `data == 0`, at once and on any machine, even one that would never show it in hardware.
3. With `Release`/`Acquire` no allowed run breaks the assertion, so the test passes.

> 🧪 **Testing:** Use loom for small, critical pieces of concurrent code: a flag, a lock, a queue with two threads. Keep the models tiny (two or three threads, a few operations), because the number of runs grows very fast. One limitation to know: loom treats `SeqCst` loads and stores like `AcqRel`, so it can report a false alarm for code that relies on `SeqCst` accesses (like section 4). `fence(SeqCst)` is modelled correctly.

## 7. Choosing an ordering

| Situation | Ordering |
|---|---|
| A counter, an ID, a statistic, a stop flag that carries no data | `Relaxed` |
| A flag or pointer that says "the data is ready" | store `Release`, load `Acquire` |
| Taking a lock / releasing a lock | `Acquire` when taking, `Release` when releasing (Lesson 14.7) |
| A read-modify-write that both receives and publishes data | `AcqRel` |
| Several threads must agree on the order of operations on different variables | `SeqCst` (or `fence(SeqCst)`) |
| You are not sure | use a `Mutex`, or `SeqCst` with a comment "TODO: check" and a loom test |

The best ordering is often none at all: use a `Mutex`, a channel or `OnceLock`, which contain correct orderings already. Write your own atomics code only when a measurement says the simple tools are too slow, and then follow *Rust Atomics and Locks* by Mara Bos (free online; see the sources).

## Common mistakes

1. **A `Relaxed` flag that publishes data.** The reader can see the flag but old data (section 2). Use a `Release` store and an `Acquire` load.
2. **Only one side of the pair.** `Release` on the store but `Relaxed` on the load (or the other way round) gives no guarantee.
3. **Reading the data without checking the flag.** The guarantee holds only if the acquire load *saw* the released value.
4. **Trusting the hardware.** A missing ordering may never show on x86-64 and only rarely on arm64. The code is still wrong. Test with loom.
5. **`SeqCst` everywhere without thinking.** It is correct but hides the design; you cannot tell which operations really need to be ordered.

## Try it yourself

1. Run `message_passing` several times in debug and in release mode. Then remove `#[repr(align(128))]` (so both statics may share one cache line) and run it again. On the test machine the old data then did not appear at all (0 in three runs). Why does the cache line matter? (Hint: what travels between cores is a whole cache line.)
2. In `message_passing.rs`, change only the **load** of the flag to `Acquire` and keep the store `Relaxed`. Does old data still appear? Then do the opposite (`Release` store, `Relaxed` load). On the test machine the first variant showed old data 0 or 1 times per 100,000 rounds, the second one 1,300 to 3,500 times. Both are wrong: half a pair gives no guarantee, even when the bug is rare.
3. Write `src/bin/orderings_asm.rs` with six `#[inline(never)] pub fn`s, each doing one `store(1, …)` or one `load(…)` on a `&AtomicU64` with `Relaxed`, `Release`/`Acquire` and `SeqCst`; call all six from `main`. Emit assembly as in Lesson 14.5 and compare with the table in section 5.

## Exercises

### Exercise 1 — Pick the ordering (easy)

For each case, choose the weakest correct ordering(s) and say why:

1. A web server counts requests with `REQUESTS.fetch_add(1, ?)`; a statistics thread prints the number every second.
2. A worker fills a `Vec` inside a `Mutex`, then sets `DONE.store(true, ?)`; the main thread spins on `DONE.load(?)` and then locks the mutex to read the `Vec`.
3. A worker writes results into three `AtomicU64` values and then sets `READY.store(true, ?)`; the main thread checks `READY.load(?)` and then reads the three values.
4. A `stop` flag that tells workers to finish; it carries no data.

<details>
<summary>Hint</summary>

Ask in each case: does the flag *publish* other data that the reader will use, and is that data already protected by something else?

</details>

<details>
<summary>Answer</summary>

1. `Relaxed`. The counter is just a number; nobody uses it to read other data.
2. `Relaxed` is enough for the flag here, because the `Vec` is read under the `Mutex`, and the mutex's own acquire/release pair orders it. (Using `Release`/`Acquire` on the flag would not hurt.)
3. `Release` for the store and `Acquire` for the load. The flag publishes the three values; with `Relaxed` the main thread could see `READY == true` and old values.
4. `Relaxed`. It only says "stop"; there is no data to publish.

</details>

### Exercise 2 — A loom test for a published batch (medium)

Write `tests/loom_batch.rs` with a `Batch` type: three `AtomicU64` values and an `AtomicBool` `ready` (all loom types). `fill(&self)` stores 1, 2 and 3 with `Relaxed`, then sets `ready`. `sum(&self) -> Option<u64>` returns `None` if `ready` is false, otherwise the sum. In a loom test, one thread calls `fill` while the main thread calls `sum`; if it returns `Some(sum)`, the sum must be 6; after `join`, `sum()` must be `Some(6)`. Make the test pass, and check that it fails when you use `Relaxed` for `ready`.

<details>
<summary>Hint</summary>

The types come from `loom::sync::atomic` and `loom::sync::Arc`; the thread from `loom::thread`. Which store and which load form the release/acquire pair?

</details>

<details>
<summary>Answer</summary>

```rust
use loom::sync::Arc;
use loom::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use loom::thread;

/// A batch of three numbers that one thread fills and another reads.
struct Batch {
    values: [AtomicU64; 3],
    ready: AtomicBool,
}

impl Batch {
    fn new() -> Self {
        Batch {
            values: [AtomicU64::new(0), AtomicU64::new(0), AtomicU64::new(0)],
            ready: AtomicBool::new(false),
        }
    }

    fn fill(&self) {
        for (i, v) in self.values.iter().enumerate() {
            v.store(i as u64 + 1, Ordering::Relaxed); // 1, 2, 3
        }
        self.ready.store(true, Ordering::Release);
    }

    /// The sum of the batch, or None if it is not ready yet.
    fn sum(&self) -> Option<u64> {
        if !self.ready.load(Ordering::Acquire) {
            return None;
        }
        Some(self.values.iter().map(|v| v.load(Ordering::Relaxed)).sum())
    }
}

#[test]
fn a_ready_batch_is_always_complete() {
    loom::model(|| {
        let batch = Arc::new(Batch::new());
        let writer = {
            let batch = Arc::clone(&batch);
            thread::spawn(move || batch.fill())
        };
        if let Some(sum) = batch.sum() {
            assert_eq!(sum, 6);
        }
        writer.join().unwrap();
        assert_eq!(batch.sum(), Some(6));
    });
}
```

`cargo test --release --test loom_batch`:

```text
test a_ready_batch_is_always_complete ... ok
```

With `Relaxed` on both sides of `ready`, the same test fails:

```text
test a_ready_batch_is_always_complete ... FAILED
assertion `left == right` failed
  left: 0
 right: 6
```

Why it works: the `Release` store of `ready` comes after the three value stores in program order, so they all happen-before any `Acquire` load that sees `ready == true`. With `Relaxed`, loom finds a run where the reader sees `true` but none of the values yet (sum 0). After `join`, the main thread sees everything anyway, because `join` itself synchronizes.

</details>

### Exercise 3 — Two flags need a full fence (hard)

Two threads each want to know whether the other one is "in": each raises its own flag and then checks the other one's flag. At least one of them must see the other flag raised (otherwise both could enter a critical section at once). Write `tests/loom_sb.rs` with a function `raise_then_check(flag, other, use_fence) -> bool` that stores `true` into `flag` with `Release`, optionally calls `fence(Ordering::SeqCst)`, and returns `other.load(Ordering::Acquire)`. Write two loom tests: without the fence (it must fail) and with the fence (it must pass). Explain the difference with section 4.

<details>
<summary>Hint</summary>

`fence` is in `loom::sync::atomic` too. The assertion is `assert!(main_saw_a || thread_saw_b, …)`. Remember from section 6 why we use a fence here and not `SeqCst` loads and stores.

</details>

<details>
<summary>Answer</summary>

```rust
use loom::sync::Arc;
use loom::sync::atomic::{AtomicBool, Ordering, fence};
use loom::thread;

/// Each side raises its own flag, then looks at the other one.
/// At least one side must see the other's flag raised.
fn raise_then_check(flag: &AtomicBool, other: &AtomicBool, use_fence: bool) -> bool {
    flag.store(true, Ordering::Release);
    if use_fence {
        fence(Ordering::SeqCst); // a full barrier between the store and the load
    }
    other.load(Ordering::Acquire)
}

fn model(use_fence: bool) {
    loom::model(move || {
        let a = Arc::new(AtomicBool::new(false));
        let b = Arc::new(AtomicBool::new(false));
        let (a2, b2) = (Arc::clone(&a), Arc::clone(&b));
        let t = thread::spawn(move || raise_then_check(&a2, &b2, use_fence));
        let main_saw_a = raise_then_check(&b, &a, use_fence);
        let thread_saw_b = t.join().unwrap();
        assert!(main_saw_a || thread_saw_b, "both sides saw the other flag down");
    });
}

#[test]
fn without_fence() {
    model(false);
}

#[test]
fn with_seqcst_fence() {
    model(true);
}
```

`cargo test --release --test loom_sb`:

```text
test without_fence ... FAILED
test with_seqcst_fence ... ok
thread 'without_fence' (5777257) panicked at tests/loom_sb.rs:23:9:
both sides saw the other flag down
```

Why it works: this is the store-buffering pattern of section 4. Release/acquire only orders operations *before* a release and *after* an acquire; nothing stops each thread's acquire load from running before its own release store, so both can read `false`. `fence(SeqCst)` puts all SeqCst fences into one global order: whichever fence comes first, the other thread's load runs after it and must see that thread's store. On the real hardware, section 4 showed the same effect with `SeqCst` stores and loads. In loom we use the fence because loom treats `SeqCst` accesses like `AcqRel` and would report a false alarm.

</details>

## Quiz

1. Who may reorder memory operations, and what stays guaranteed?

<details>
<summary>Answer</summary>

Both the compiler and the CPU, within the rules of the memory model. Each thread always sees its own operations in program order. What other threads see, and in which order, is guaranteed only by the orderings you choose.

</details>

2. What does it mean that arm64 is "weakly ordered", and why is that useful for you as a learner?

<details>
<summary>Answer</summary>

The CPU may make stores visible to other cores in a different order than the program wrote them, and run loads early. Bugs from missing orderings therefore appear on a Mac (as in the litmus test) that would stay hidden on a strongly ordered x86-64 machine.

</details>

3. Explain the release/acquire guarantee in one or two sentences.

<details>
<summary>Answer</summary>

If an `Acquire` load reads the value written by a `Release` store, then everything the writing thread did before the store happens-before everything the reading thread does after the load, so the reader sees all of it.

</details>

4. A thread stores data and then a flag with `Release`. The reader loads the flag with `Relaxed` and then reads the data. Is that correct?

<details>
<summary>Answer</summary>

No. The guarantee needs a pair: the load must be `Acquire`. With a `Relaxed` load, the reader may see the flag set and still see old data.

</details>

5. Which outcome does `SeqCst` forbid that release/acquire allows?

<details>
<summary>Answer</summary>

The store-buffering outcome: two threads each store to their own variable and then load the other one's variable, and both read the old value. `SeqCst` puts all its operations into one global order, so at least one thread must see the other's store.

</details>

6. What does loom do that a stress test cannot, and what is one of its limits?

<details>
<summary>Answer</summary>

It systematically explores every interleaving of a small model and lets atomic loads return any value the memory model allows, so it finds ordering bugs on any machine, every time. One limit: it treats `SeqCst` accesses as `AcqRel` (false alarms are possible); `fence(SeqCst)` is modelled correctly.

</details>

## Summary

- Other threads may see your writes in a different order. The compiler and the CPU both reorder; arm64 (your Mac) reorders much more than x86-64.
- Happens-before is the guarantee you need. A `Release` store and an `Acquire` load that reads its value create it: everything before the release is visible after the acquire.
- Use `Relaxed` for plain numbers and stop flags, release/acquire for "data is ready" flags and locks, `AcqRel` for read-modify-writes that pass data on, and `SeqCst` only when threads must agree on the order of operations on different variables.
- On arm64 the orderings are different instructions: `str`/`ldr`, `stlr`, `ldapr`, `ldar`.
- Litmus tests show reorderings on real hardware but need luck; loom finds them systematically in small models. Remember that loom models `SeqCst` accesses like `AcqRel`.
- When in doubt, use a `Mutex`, a channel or `OnceLock`, which contain correct orderings already.

## Sources and further reading

- [std::sync::atomic::Ordering](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html) — the official description of each ordering.
- [The Rustonomicon — Atomics](https://doc.rust-lang.org/nomicon/atomics.html) — the memory model, happens-before and why hardware reorders.
- [Rust Atomics and Locks, chapter 3 — Memory Ordering](https://marabos.nl/atomics/memory-ordering.html) — Mara Bos explains happens-before, release/acquire and `SeqCst` with clear diagrams.
- [Rust Atomics and Locks, chapter 7 — Understanding the Processor](https://marabos.nl/atomics/hardware.html) — what the orderings become on arm64 and x86-64, and why arm64 is weakly ordered.
- [loom on docs.rs](https://docs.rs/loom) — the model checker used in section 6, with the list of what it does not model.
- [cppreference — std::memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order) — the reference description of the C++ memory model, which Rust follows.

---

[🏠 Start](../START-HERE.md) · [← 14.5 Atomics](05-atomics.md) · [📋 Phase 14](README.md) · [14.7 Building a spin lock →](07-building-a-spin-lock.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
