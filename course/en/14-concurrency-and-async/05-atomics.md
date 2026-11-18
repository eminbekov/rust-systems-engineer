<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.4 Thread pools, scoped threads and rayon](04-thread-pools-scoped-threads-and-rayon.md) · [📋 Phase 14](README.md) · [14.6 Memory ordering →](06-memory-ordering.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/05-atomics.md) <!-- nav -->

# Lesson 14.5 — Atomics

> **Phase 14 — Concurrency and async in depth** · Lesson 5 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What an atomic operation is, and which atomic types Rust has.
- The basic operations: `load`, `store`, `swap` and the `fetch_…` family, and why `fetch_add` silently wraps around.
- How compare-and-swap (`compare_exchange`) works, how to write a CAS (compare-and-swap) loop, and when `try_update` and `update` write it for you.
- Typical uses: unique IDs, a stop flag, a limit counter, two numbers packed into one atomic.
- What the CPU (Central Processing Unit) of your Mac really does for an atomic operation.

## Before you start

- You finished [Lesson 14.4 — Thread pools, scoped threads and rayon](04-thread-pools-scoped-threads-and-rayon.md).
- You saw the *lost update* with `load` + `store` in [Lesson 14.1](01-why-concurrency-is-hard.md), and a first `AtomicUsize` counter in [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md).
- You know interior mutability from [Lesson 06.4](../06-rust-advanced/04-interior-mutability.md), bit operations from [Lesson 12.4](../12-computer-systems/04-bitwise-operations.md), and you can read a little arm64 assembly ([Lesson 12.7](../12-computer-systems/07-reading-assembly.md)).
- Keep working in `~/projects/rust/code/p14-concurrency`; each example is a new file in `src/bin/`.

## New words

| Word | Meaning |
|---|---|
| **atomic operation** | An operation on memory that other threads can see only as "not started" or "finished", never half-done. |
| **atomic type** | A type such as `AtomicU64` or `AtomicBool` whose methods are atomic operations. It can be changed through a shared reference `&`. |
| **`Ordering`** | An argument of every atomic method. It says what *other* memory operations are ordered around this one. In this lesson we always use `Relaxed`; Lesson 14.6 explains the others. |
| **`Relaxed`** | The weakest ordering: the operation on *this one variable* is atomic, and nothing more is promised. Enough for counters and IDs. |
| **read-modify-write (RMW)** | One atomic operation that reads a value, computes a new one and writes it, with no gap between: `swap`, `fetch_add`, `compare_exchange`. |
| **wrap around** | When a number goes past its largest value and starts again from the smallest one: for `u8`, 255 + 1 gives 0. |
| **CAS** (compare-and-swap) | An atomic operation: "if the value is still `current`, replace it with `new`; otherwise tell me the actual value". In Rust it is `compare_exchange`. |
| **CAS loop** | Read the value, compute a new value, try to put it in with CAS, and repeat if another thread changed the value in the meantime. |
| **spurious failure** | `compare_exchange_weak` may fail even though the value was equal. It is allowed so that it can be faster on some CPUs. In a loop it does not matter. |
| **LSE** (Large System Extensions) | A part of the arm64 instruction set (since ARMv8.1) with single instructions for atomic operations, such as `ldadd` (load and add). Apple M chips have it. |
| **false sharing** | Two threads write *different* variables that happen to lie in the same cache line, so the line still jumps between cores as if they shared one variable. |

## 1. What "atomic" means

In Lesson 14.1 four threads ran `let v = counter.load(); counter.store(v + 1);` and lost three quarters of the updates. Each step was safe, but another thread could run **between** the two steps.

An **atomic operation** has no "between". When a thread does `fetch_add(1)`, every other thread sees the counter either before the addition or after it, never in the middle, and two `fetch_add` calls can never both read the same old value. The CPU guarantees it in hardware (section 5).

The standard library has these types in `std::sync::atomic`:

| Type | Holds |
|---|---|
| `AtomicBool` | `bool` |
| `AtomicI8`, `AtomicU8`, `AtomicI16`, `AtomicU16`, `AtomicI32`, `AtomicU32`, `AtomicI64`, `AtomicU64` | integers of that size |
| `AtomicIsize`, `AtomicUsize` | pointer-sized integers |
| `AtomicPtr<T>` | a raw pointer `*mut T` (Lesson 14.8 uses it) |

Two things are special about them:

- **They change through `&self`.** You do not need `let mut` or `&mut`. Like `Cell` (Lesson 06.4), they are built on `UnsafeCell`, but unlike `Cell` they are `Sync`: many threads may hold a `&AtomicU64` at once. So you can put them in an `Arc`, borrow them in scoped threads, or keep them in a `static`.
- **Every method takes an `Ordering`.** For now we always pass `Ordering::Relaxed`: "make this one operation atomic, I need nothing else". That is correct whenever the atomic is *only a number* (a counter, an ID, a statistic). When an atomic is used to say "the *other* data is ready", `Relaxed` is not enough; that is the topic of Lesson 14.6.

## 2. The basic operations

Create `src/bin/atomic_basics.rs`:

```rust
use std::sync::atomic::{AtomicBool, AtomicU32, Ordering};

fn main() {
    let x = AtomicU32::new(10); // no `mut` needed: atomics change through `&self`

    println!("load           -> {}", x.load(Ordering::Relaxed));
    x.store(20, Ordering::Relaxed);
    println!("store(20)      -> now {}", x.load(Ordering::Relaxed));

    let old = x.swap(30, Ordering::Relaxed);
    println!("swap(30)       -> old {old}, now {}", x.load(Ordering::Relaxed));

    let old = x.fetch_add(5, Ordering::Relaxed);
    println!("fetch_add(5)   -> old {old}, now {}", x.load(Ordering::Relaxed));

    let old = x.fetch_sub(15, Ordering::Relaxed);
    println!("fetch_sub(15)  -> old {old}, now {}", x.load(Ordering::Relaxed));

    let old = x.fetch_max(100, Ordering::Relaxed);
    println!("fetch_max(100) -> old {old}, now {}", x.load(Ordering::Relaxed));

    let flag = AtomicBool::new(false);
    let old = flag.fetch_or(true, Ordering::Relaxed);
    println!("fetch_or(true) -> old {old}, now {}", flag.load(Ordering::Relaxed));
}
```

```text
load           -> 10
store(20)      -> now 20
swap(30)       -> old 20, now 30
fetch_add(5)   -> old 30, now 35
fetch_sub(15)  -> old 35, now 20
fetch_max(100) -> old 20, now 100
fetch_or(true) -> old false, now true
```

**What happens:**

1. `load` reads, `store` writes. Each one alone is atomic; together they are two steps (the lost update).
2. `swap` writes a new value and returns the old one, in **one** step.
3. The `fetch_…` methods (`fetch_add`, `fetch_sub`, `fetch_and`, `fetch_or`, `fetch_xor`, `fetch_max`, `fetch_min`) change the value and return the **old** value, in one step. They are atomic read-modify-write operations.
4. The returned old value is often the useful part. `fetch_or(true)` on a flag tells you whether *you* were the first to set it: only one thread can ever get `old false`.

The most common use is a global counter in a `static`. A `static` lives for the whole program and every thread can see it, which is safe because the atomic needs no `&mut`. Create `src/bin/next_id.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

static NEXT_ID: AtomicU64 = AtomicU64::new(1);

/// Returns a new ID that no other call ever returned.
fn next_id() -> u64 {
    NEXT_ID.fetch_add(1, Ordering::Relaxed)
}

fn main() {
    thread::scope(|s| {
        for name in ["a", "b", "c"] {
            s.spawn(move || {
                let ids: Vec<u64> = (0..3).map(|_| next_id()).collect();
                println!("thread {name} got {ids:?}");
            });
        }
    });
    println!("next free id: {}", NEXT_ID.load(Ordering::Relaxed));
}
```

```text
thread a got [1, 2, 3]
thread c got [4, 5, 6]
thread b got [7, 8, 9]
next free id: 10
```

**What happens:** `fetch_add` returns the old value and moves the counter on in the same step, so no two calls can get the same number, whatever the timing. Which thread gets which numbers changes from run to run; that every number appears exactly once never changes. Exercise 1 tests it.

### `fetch_add` wraps around

Normal integer arithmetic in Rust panics on overflow in a debug build ([Lesson 12.2](../12-computer-systems/02-integers-in-memory.md)). Atomic `fetch_add` and `fetch_sub` **never** panic: they wrap around, in debug and in release. Create `src/bin/wrap.rs`:

```rust
use std::sync::atomic::{AtomicU8, Ordering};

fn main() {
    let small = AtomicU8::new(254);
    for _ in 0..3 {
        let old = small.fetch_add(1, Ordering::Relaxed);
        println!("{old} + 1 = {}", small.load(Ordering::Relaxed));
    }
}
```

```text
254 + 1 = 255
255 + 1 = 0
0 + 1 = 1
```

With an `AtomicU64` counting one million per second, wrapping takes more than 500,000 years, so it is fine. With an `AtomicU8` or `AtomicU16`, or with `fetch_sub` on a counter that can reach zero, it is a real bug.

> 🔒 **Security:** A wrapped counter can hand out an ID that is already in use, or turn "0 free slots" into "4 billion free slots" after one `fetch_sub` too many. When a value must not wrap, do not use `fetch_add`: use the checked CAS loop of the next section (Exercise 2).

## 3. Compare-and-swap

`fetch_add` can only add. What if the new value depends on a rule, for example "add 1, but never above 3"? For that there is **compare-and-swap** (CAS), in Rust `compare_exchange`:

```text
 compare_exchange(current, new):
   in ONE atomic step:
     if value == current { value = new;  return Ok(current) }
     else                {                return Err(actual value) }
```

You say "I believe the value is still `current`; if I am right, make it `new`". If another thread changed the value in the meantime, nothing is written and you learn the actual value. Then you compute again and retry. That is a **CAS loop**.

Here is a limiter that allows at most `max` holders at the same time, for example open connections. Create `src/bin/limiter.rs`:

```rust
use std::sync::atomic::{AtomicU32, Ordering};
use std::thread;

/// Allows at most `max` holders at the same time.
struct Limiter {
    current: AtomicU32,
    max: u32,
}

impl Limiter {
    fn try_acquire(&self) -> bool {
        let mut seen = self.current.load(Ordering::Relaxed);
        loop {
            if seen == self.max {
                return false; // full: give up, do not wait
            }
            match self.current.compare_exchange_weak(
                seen,     // "I believe the value is still `seen`..."
                seen + 1, // "...then please make it `seen + 1`"
                Ordering::Relaxed,
                Ordering::Relaxed,
            ) {
                Ok(_) => return true,
                Err(actual) => seen = actual, // someone was faster: retry
            }
        }
    }

    fn release(&self) {
        self.current.fetch_sub(1, Ordering::Relaxed);
    }
}

fn main() {
    let limiter = Limiter {
        current: AtomicU32::new(0),
        max: 3,
    };
    let results: Vec<bool> = (0..5).map(|_| limiter.try_acquire()).collect();
    println!("five tries with max 3: {results:?}");
    limiter.release();
    println!("after one release: {}", limiter.try_acquire());

    // Many threads: try to take, hold for a moment, give back.
    let ok = AtomicU32::new(0);
    let limiter = Limiter {
        current: AtomicU32::new(0),
        max: 3,
    };
    thread::scope(|s| {
        for _ in 0..8 {
            s.spawn(|| {
                for _ in 0..10_000 {
                    if limiter.try_acquire() {
                        ok.fetch_add(1, Ordering::Relaxed);
                        limiter.release();
                    }
                }
            });
        }
    });
    println!(
        "successful acquires: {}, current at the end: {}",
        ok.load(Ordering::Relaxed),
        limiter.current.load(Ordering::Relaxed)
    );
}
```

```text
five tries with max 3: [true, true, true, false, false]
after one release: true
successful acquires: 8209, current at the end: 0
```

**What happens:**

1. `try_acquire` reads the current value into `seen`. If it is already `max`, it says no.
2. Otherwise it asks `compare_exchange_weak(seen, seen + 1, …)`: "if the value is still `seen`, make it `seen + 1`". If that works (`Ok`), we have a slot.
3. If another thread changed the value between our `load` and our CAS, the CAS fails with `Err(actual)`. Nothing was written. We take `actual` as the new `seen`, check the limit again and retry. The value can therefore never go above `max`, whatever the timing.
4. The two `Ordering` arguments are for success and for failure; both are `Relaxed` here (Lesson 14.6).
5. With 8 threads and a limit of 3, many tries fail (the number of successes changes from run to run). At the end, every successful `try_acquire` was followed by one `release`, so the counter is back to 0.

### `compare_exchange` or `compare_exchange_weak`?

`compare_exchange_weak` may fail **spuriously**: it can return `Err` even though the value was equal. On arm64 CPUs without LSE, a CAS is built from two instructions (load-exclusive and store-exclusive), and the pair can fail if anything disturbs it; the weak version does not hide that with an extra inner loop, so it is a little faster. Inside a retry loop a spurious failure only means one more round, so **use `compare_exchange_weak` in loops** and `compare_exchange` when you try only once.

### `try_update` and `update` write the loop for you

Every atomic integer has two methods that contain the CAS loop:

- `update(set_order, fetch_order, |old| new)` — the closure computes the new value from the old one; it is retried until the CAS succeeds. Returns the old value.
- `try_update(set_order, fetch_order, |old| Some(new) or None)` — the same, but the closure may return `None` to give up. Returns `Ok(old)` or `Err(current)`.

The closure may run **several times** (once per retry), so it must not have side effects such as printing or pushing into a vector. Here is an ID generator that refuses to wrap, using `checked_add` (which returns `None` on overflow). Create `src/bin/next_id_checked.rs`:

```rust
use std::sync::atomic::{AtomicU8, Ordering};

static NEXT_ID: AtomicU8 = AtomicU8::new(250); // tiny type, so we can see the end

/// Returns a new ID, or None when all IDs are used up. Never wraps around.
fn next_id() -> Option<u8> {
    NEXT_ID
        .try_update(Ordering::Relaxed, Ordering::Relaxed, |id| id.checked_add(1))
        .ok()
}

fn main() {
    let ids: Vec<Option<u8>> = (0..8).map(|_| next_id()).collect();
    println!("{ids:?}");
}
```

```text
[Some(250), Some(251), Some(252), Some(253), Some(254), None, None, None]
```

**What happens:** `try_update` loads the value, calls the closure, and tries a CAS with the result. When the value is 255, `checked_add(1)` returns `None`, so `try_update` gives up and returns `Err(255)`, which `.ok()` turns into `None`. The value stays at 255 forever: no ID is ever handed out twice. (ID 255 itself is never returned; that is the price of this simple design.)

> ⚠️ **Common mistake:** You may see `fetch_update` in older code and articles. In Rust 1.99 it was renamed: the compiler now warns ``use of deprecated method `…::fetch_update`: renamed to `try_update` for consistency``. Use `try_update` (or `update` when the closure never gives up).

## 4. Two more patterns

**A stop flag.** Worker threads check an `AtomicBool` and stop when it is set. Create `src/bin/stop_flag.rs`:

```rust
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use std::thread;
use std::time::Duration;

fn main() {
    let stop = AtomicBool::new(false);
    let rounds = AtomicU64::new(0);

    thread::scope(|s| {
        for _ in 0..3 {
            s.spawn(|| {
                while !stop.load(Ordering::Relaxed) {
                    rounds.fetch_add(1, Ordering::Relaxed); // "work"
                    thread::sleep(Duration::from_millis(1));
                }
            });
        }
        thread::sleep(Duration::from_millis(100));
        stop.store(true, Ordering::Relaxed); // ask all workers to stop
    });

    println!(
        "workers stopped after about {} rounds",
        rounds.load(Ordering::Relaxed)
    );
}
```

```text
workers stopped after about 243 rounds
```

**What happens:** three workers loop until they see `true`. After 100 ms the main thread stores `true`, every worker notices it at its next check, and the scope ends when all three have returned. `Relaxed` is enough here because the flag carries no other data: it only says "stop". (The `ctrlc` handler of Lesson 06.11 used the same pattern.) If the flag also had to say "and the results you need are now in this vector", it would need the orderings of Lesson 14.6.

**Packing two values into one atomic.** Two separate atomics are two separate steps. If you keep `count` and `sum` in two `AtomicU64`s, a reader can see the new `count` with the old `sum`. When two small numbers must change together, put them into **one** 64-bit atomic: for example `count` in the high 32 bits and `sum` in the low 32 bits, changed with `update`. Exercise 3 builds this.

## 5. What the CPU does

How can hardware do "read, add, write" with no gap? Let us look at the machine code. Create `src/bin/asm_demo.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

#[inline(never)]
pub fn plain_add(x: &mut u64) {
    *x += 1;
}

#[inline(never)]
pub fn atomic_add(x: &AtomicU64) -> u64 {
    x.fetch_add(1, Ordering::Relaxed)
}

fn main() {
    let mut a = 0;
    plain_add(&mut a);
    let b = AtomicU64::new(0);
    atomic_add(&b);
    println!("{a} {}", b.load(Ordering::Relaxed));
}
```

Ask the compiler for assembly instead of a program, then find the file:

```bash
cargo rustc --release --bin asm_demo -- --emit asm
find target/release -name 'asm_demo.s'
```

In that file, search for `plain_add` and `atomic_add`. The bodies are (the long mangled function names are shortened here):

```text
plain_add:
	ldr	x8, [x0]
	add	x8, x8, #1
	str	x8, [x0]
	ret

atomic_add:
	mov	w8, #1
	ldadd	x8, x8, [x0]
	ret
```

**What happens:**

1. `plain_add` is three instructions: load the value into register `x8`, add 1, store it back. Another core can write the same memory between the `ldr` and the `str`: that is the lost update of Lesson 14.1, in hardware.
2. `atomic_add` puts 1 into `w8` and then runs **one** instruction, `ldadd` ("load and add"), from the arm64 **LSE** extension. The CPU core takes exclusive ownership of the cache line, reads the old value into `x8`, adds, and writes back before any other core may touch the line. The old value in `x8` is the return value of `fetch_add`.
3. `#[inline(never)]` keeps the two functions separate so that we can find them; without it the compiler would put their code directly into `main`.

> ⚡ **Performance:** An atomic operation on a line that no other core uses costs about as much as a normal one. When many cores hammer the **same** atomic, every operation must move the cache line between cores, and it gets much slower (the same effect as the contended lock in Lesson 14.2). Two tips: let each thread count in a local variable and add to the shared atomic once at the end; and watch for **false sharing**: two atomics used by different threads but placed next to each other in memory share one cache line and slow each other down. `crossbeam::utils::CachePadded<T>` pads a value to a full cache line to prevent it.

> 💡 **Tip:** A `Mutex<u64>` is easier to get right than a CAS loop, and fast when it is not contended. Use atomics for single numbers and flags. When the invariant involves more than one value (and they do not fit into one atomic), use a lock.

## Common mistakes

1. **`load` followed by `store` to change a value.** That is two steps: the lost update. Use a `fetch_…` method, `update`, or a CAS loop.
2. **Forgetting that `fetch_add` and `fetch_sub` wrap.** They never panic. If the value must stay in a range, use `try_update` with `checked_add` / `checked_sub`.
3. **Two atomics that must change together.** Each one is atomic, the pair is not. Pack them into one atomic or use a mutex.
4. **Side effects inside the closure of `update` / `try_update`.** It can run several times. Compute only.
5. **Using `Relaxed` for a flag that publishes data.** "Set `ready = true` after filling the buffer" needs `Release` and `Acquire`. Lesson 14.6 shows the bug and the fix.

## Try it yourself

1. In `wrap.rs`, use an `AtomicI8` starting at `126`. What comes after 127? (Answer: `127 + 1 = -128`, the smallest `i8`.)
2. In `limiter.rs`, change `compare_exchange_weak` to `compare_exchange`. The program still works. Then replace the whole CAS loop in `try_acquire` with a single line: `self.current.try_update(Ordering::Relaxed, Ordering::Relaxed, |c| (c < self.max).then_some(c + 1)).is_ok()`. Run it again: the first two lines are the same, only the number of successful acquires changes from run to run.
3. In `stop_flag.rs`, remove the line `stop.store(true, Ordering::Relaxed);`. The program never ends. Why? Stop it with `Ctrl+C`.

## Exercises

### Exercise 1 — Unique IDs under pressure (easy)

Write `src/bin/id_test.rs` with the `next_id` function of section 2 and a test `ids_are_unique_across_threads`: 8 scoped threads each take 1,000 IDs; the test collects all of them and checks that there are 8,000 IDs and 8,000 *different* IDs.

<details>
<summary>Hint</summary>

Each thread returns its `Vec<u64>`; join the handles and `flat_map` them into one vector. Put the vector into a `HashSet<u64>`: a set keeps each value once, so its length is the number of different values.

</details>

<details>
<summary>Answer</summary>

```rust
use std::sync::atomic::{AtomicU64, Ordering};

static NEXT_ID: AtomicU64 = AtomicU64::new(1);

fn next_id() -> u64 {
    NEXT_ID.fetch_add(1, Ordering::Relaxed)
}

fn main() {
    println!("{} {}", next_id(), next_id());
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashSet;
    use std::thread;

    #[test]
    fn ids_are_unique_across_threads() {
        let all: Vec<u64> = thread::scope(|s| {
            let handles: Vec<_> = (0..8)
                .map(|_| s.spawn(|| (0..1000).map(|_| next_id()).collect::<Vec<_>>()))
                .collect();
            handles
                .into_iter()
                .flat_map(|h| h.join().unwrap())
                .collect()
        });
        let unique: HashSet<u64> = all.iter().copied().collect();
        assert_eq!(all.len(), 8000);
        assert_eq!(unique.len(), 8000, "some ID was given out twice");
    }
}
```

```text
running 1 test
test tests::ids_are_unique_across_threads ... ok
```

Why it works: `fetch_add` hands out each value exactly once. Try to break the test: replace `fetch_add` with the `load` + `store` pair of Lesson 14.1. The test then fails with the message `some ID was given out twice`, because two threads can read the same value.

</details>

### Exercise 2 — The limiter with `try_update` (medium)

Rewrite `try_acquire` of `limiter.rs` with `try_update` instead of the hand-written loop (see Try it yourself, step 2), and write a test `never_more_than_max_holders`: 8 threads each try 10,000 times to acquire; after a successful acquire, a thread records the current number of holders with `fetch_max` into a second atomic `highest`, then releases. At the end `highest` must be at most `max` (3) and `current` must be 0.

<details>
<summary>Hint</summary>

`(c < self.max).then_some(c + 1)` returns `Some(c + 1)` when there is room and `None` otherwise. Inside the test, after `try_acquire()` returned `true`, read `limiter.current` and call `highest.fetch_max(that_value, Ordering::Relaxed)`.

</details>

<details>
<summary>Answer</summary>

The new method:

```rust
    fn try_acquire(&self) -> bool {
        self.current
            .try_update(Ordering::Relaxed, Ordering::Relaxed, |c| {
                (c < self.max).then_some(c + 1)
            })
            .is_ok()
    }
```

The test, at the end of `src/bin/limiter.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn never_more_than_max_holders() {
        let limiter = Limiter {
            current: AtomicU32::new(0),
            max: 3,
        };
        let highest = AtomicU32::new(0);
        thread::scope(|s| {
            for _ in 0..8 {
                s.spawn(|| {
                    for _ in 0..10_000 {
                        if limiter.try_acquire() {
                            let now = limiter.current.load(Ordering::Relaxed);
                            highest.fetch_max(now, Ordering::Relaxed);
                            limiter.release();
                        }
                    }
                });
            }
        });
        assert!(highest.load(Ordering::Relaxed) <= 3);
        assert_eq!(limiter.current.load(Ordering::Relaxed), 0);
    }
}
```

```text
running 1 test
test tests::never_more_than_max_holders ... ok
```

Why it works: `try_update` is the same CAS loop: the closure decides, from the value it saw, whether there is room; the CAS writes `c + 1` only if nobody changed the value since. So `current` can never pass `max`. `fetch_max` keeps the largest value any thread observed; if the limiter were broken (for example with `fetch_add` and a check *after* it), `highest` would sometimes reach 4 or more.

</details>

### Exercise 3 — Count and sum in one atomic (hard)

Write `src/bin/stats.rs` with a `Stats` type that records values (`record(&self, value: u32)`) and returns a consistent pair `snapshot(&self) -> (count, sum)`. Store both in **one** `AtomicU64`: the count in the high 32 bits, the sum in the low 32 bits. `main` records 10, 20 and 30 and prints count, sum and average. Test: 4 threads each record the value 5 ten thousand times, while a fifth thread takes 10,000 snapshots and asserts each time that `sum == count * 5`. At the end the snapshot is `(40000, 200000)`.

<details>
<summary>Hint</summary>

Packing: `(count << 32) | sum`. Unpacking: `count = p >> 32`, `sum = p & 0xFFFF_FFFF`. Change the packed value with `update`, whose closure gets the old packed value and returns the new one. Why would two separate atomics (`count` and `sum`) fail the reader's assertion?

</details>

<details>
<summary>Answer</summary>

```rust
use std::sync::atomic::{AtomicU64, Ordering};

/// Count (high 32 bits) and sum (low 32 bits) packed into ONE atomic,
/// so a reader always sees a matching pair.
struct Stats {
    packed: AtomicU64,
}

impl Stats {
    fn new() -> Self {
        Stats {
            packed: AtomicU64::new(0),
        }
    }

    fn record(&self, value: u32) {
        self.packed
            .update(Ordering::Relaxed, Ordering::Relaxed, |p| {
                let count = (p >> 32) + 1;
                let sum = (p & 0xFFFF_FFFF) + value as u64;
                assert!(sum <= u32::MAX as u64, "sum does not fit in 32 bits");
                (count << 32) | sum
            });
    }

    /// Returns (count, sum), always from the same moment.
    fn snapshot(&self) -> (u64, u64) {
        let p = self.packed.load(Ordering::Relaxed);
        (p >> 32, p & 0xFFFF_FFFF)
    }
}

fn main() {
    let stats = Stats::new();
    for v in [10, 20, 30] {
        stats.record(v);
    }
    let (count, sum) = stats.snapshot();
    println!("count = {count}, sum = {sum}, average = {}", sum / count);
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn readers_always_see_a_matching_pair() {
        let stats = Stats::new();
        thread::scope(|s| {
            for _ in 0..4 {
                s.spawn(|| {
                    for _ in 0..10_000 {
                        stats.record(5); // every value is 5...
                    }
                });
            }
            s.spawn(|| {
                for _ in 0..10_000 {
                    let (count, sum) = stats.snapshot();
                    assert_eq!(sum, count * 5); // ...so sum is always 5 * count
                }
            });
        });
        assert_eq!(stats.snapshot(), (40_000, 200_000));
    }
}
```

```text
count = 3, sum = 60, average = 20
```

`cargo test --bin stats` shows `test tests::readers_always_see_a_matching_pair ... ok`.

Why it works: one `update` changes count and sum in one CAS, and one `load` reads both, so a reader can never see the count of one moment with the sum of another. With two atomics, a writer does `count += 1` and then `sum += 5` as two steps, and a reader that runs between them sees `sum == (count - 1) * 5`: the assertion fails. The price of packing: each half has only 32 bits, so the sum must stay below about 4.29 billion (the `assert!` checks it; a real program would use a lock or a 128-bit design instead).

</details>

## Quiz

1. Why can `AtomicU64` be changed through `&self`, while a `u64` needs `&mut`?

<details>
<summary>Answer</summary>

Atomic types are built on `UnsafeCell` (interior mutability) and every change is an atomic hardware operation, so changing them through a shared reference cannot cause a data race. That is why they are `Sync` and can be shared by many threads, in an `Arc`, by borrowing, or in a `static`.

</details>

2. What does `fetch_add` return, and why is that useful?

<details>
<summary>Answer</summary>

The value **before** the addition. Because reading and adding happen in one step, the returned value is unique to this call: that is how an ID generator gives each caller a different number.

</details>

3. What happens when an `AtomicU8` holding 255 gets `fetch_add(1)`?

<details>
<summary>Answer</summary>

It wraps around to 0 and returns 255. Atomic `fetch_add` never panics on overflow, not even in a debug build.

</details>

4. Describe a CAS loop in your own words.

<details>
<summary>Answer</summary>

Read the current value; compute the new value from it; call `compare_exchange(old, new)`. If the value is still `old`, the new value is written and you are done. If another thread changed it, you get the actual value back, compute again from that and retry.

</details>

5. Why is `compare_exchange_weak` fine inside a loop?

<details>
<summary>Answer</summary>

It may fail spuriously (return `Err` although the value was equal), but in a loop that only causes one more round. In exchange it can be a little faster on CPUs where CAS is built from a load-exclusive/store-exclusive pair.

</details>

6. On an Apple M chip, which single instruction does `fetch_add(1, Relaxed)` on an `AtomicU64` become, and how does it differ from `*x += 1`?

<details>
<summary>Answer</summary>

`ldadd` from the arm64 LSE extension: one instruction that loads, adds and stores while the core holds the cache line exclusively. `*x += 1` is three separate instructions (`ldr`, `add`, `str`), and another core can write between them.

</details>

## Summary

- An atomic operation is indivisible: other threads see it as not started or finished. Rust has `AtomicBool`, the integer atomics and `AtomicPtr`; they change through `&self` and are `Sync`.
- `load`/`store` are single steps; `swap`, `fetch_add`, `fetch_sub`, `fetch_or`, `fetch_max` and friends are atomic read-modify-write operations that return the old value.
- `fetch_add` and `fetch_sub` wrap around silently. Use `try_update` with `checked_add` when a value must stay in range.
- `compare_exchange` writes only if the value is still what you expect. A CAS loop retries until it succeeds; `update` and `try_update` (the new names of `fetch_update`) write the loop for you.
- `Relaxed` is enough when the atomic is only a number or a stop flag. Two values that must change together need one packed atomic or a lock.
- On arm64 a `fetch_add` is one `ldadd` instruction. Contended atomics are slow because the cache line moves between cores; watch out for false sharing.

## Sources and further reading

- [std::sync::atomic](https://doc.rust-lang.org/std/sync/atomic/index.html) — the module documentation: all atomic types and a short introduction to orderings.
- [AtomicU64 in the standard library](https://doc.rust-lang.org/std/sync/atomic/type.AtomicU64.html) — every method, including `compare_exchange_weak`, `update` and `try_update`. (In Rust 1.100 `AtomicU64` is defined as `Atomic<u64>`, a generic type that stable code cannot name directly yet; that is why compiler messages say `Atomic::<u64>`.)
- [Rust Atomics and Locks, chapter 2 — Atomics](https://marabos.nl/atomics/atomics.html) — Mara Bos explains load, store, fetch-and-modify and compare-and-exchange with many small examples.
- [Rust Atomics and Locks, chapter 7 — Understanding the Processor](https://marabos.nl/atomics/hardware.html) — how atomics map to arm64 and x86-64 instructions, and why cache lines matter.
- [crossbeam_utils::CachePadded on docs.rs](https://docs.rs/crossbeam-utils/latest/crossbeam_utils/struct.CachePadded.html) — the padding type that prevents false sharing.

---

[🏠 Start](../START-HERE.md) · [← 14.4 Thread pools, scoped threads and rayon](04-thread-pools-scoped-threads-and-rayon.md) · [📋 Phase 14](README.md) · [14.6 Memory ordering →](06-memory-ordering.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
