<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.6 Memory ordering](06-memory-ordering.md) · [📋 Phase 14](README.md) · [14.8 Lock-free data structures →](08-lock-free-data-structures.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/07-building-a-spin-lock.md) <!-- nav -->

# Lesson 14.7 — Building a spin lock

> **Phase 14 — Concurrency and async in depth** · Lesson 7 of 16 · ⏱ about 135 minutes

## What you will learn

- How a lock can be built from one atomic flag and a loop.
- How to wrap the data in `UnsafeCell`, why you must write `unsafe impl Sync` yourself, and how to justify it.
- How a guard type with `Deref`, `DerefMut` and `Drop` makes the lock safe to use.
- Why taking the lock needs `Acquire` and releasing it needs `Release`, shown with lost updates on your Mac and with Miri.
- Why you should almost never use a spin lock in a normal program, measured against `std::sync::Mutex`.

## Before you start

- You finished [Lesson 14.6 — Memory ordering](06-memory-ordering.md). You know release/acquire pairs.
- You know `UnsafeCell` from [Lesson 06.4](../06-rust-advanced/04-interior-mutability.md), `unsafe` and Miri from [Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md), and `Deref` and `Drop` from [Lesson 05.9](../05-rust-intermediate/09-smart-pointers.md).
- Keep working in `~/projects/rust/code/p14-concurrency`. This lesson adds a **library** part to the package: a file `src/lib.rs` next to `src/main.rs`. Programs in `src/bin/` can then use it as the crate `p14_concurrency` (Cargo turns the dash into an underscore).

## New words

| Word | Meaning |
|---|---|
| **spin lock** | A lock where a waiting thread does not sleep: it runs in a loop ("spins") and checks the lock again and again until it is free. |
| **spinning** | Busy-waiting in a loop, using the CPU (Central Processing Unit), until a condition changes. |
| **test-and-set** | An atomic operation that sets a flag and returns its old value in one step. `swap(true)` on an `AtomicBool` is one. |
| **`UnsafeCell<T>`** | The basic building block of interior mutability: it gives a raw pointer `*mut T` from `&self`. It is not `Sync`, so you must promise thread safety yourself. |
| **`unsafe impl`** | Implementing a trait whose rules the compiler cannot check, such as `Sync`. You promise that the rules hold. |
| **`SAFETY:` comment** | A comment above `unsafe` code that explains why the code is sound: which rule must hold and why it does. |
| **guard** | A value that proves you hold the lock, gives access to the data, and releases the lock in its `Drop`. |
| **`spin_loop()`** | `std::hint::spin_loop`: a hint to the CPU that the thread is in a busy-wait loop, so it can save power or let another hardware thread run. |
| **TTAS** (test-and-test-and-set) | A spin lock that waits with plain loads and tries the expensive `swap` only when the lock looks free. |
| **preemption** | The OS (Operating System) stops a running thread to let another one run. A preempted lock holder blocks all spinners. |
| **oversubscription** | More runnable threads than CPU cores, so some threads always wait for a core. |
| **ticket lock** | A fair spin lock: each thread takes a number and waits until that number is served, like the queue at a post office. |
| **data race detector** | The part of Miri that reports two unsynchronized accesses to the same memory, at least one a write, as undefined behaviour. |

## 1. The idea

A lock needs one bit of state: free or taken. An `AtomicBool` is enough:

```text
 lock():   loop { old = swap(locked, true); if old == false { return } }   ← I flipped it: it is mine
 unlock(): store(locked, false)
```

`swap(true)` sets the flag and returns the old value in **one** atomic step (a *test-and-set*). If the old value was `false`, the lock was free and now belongs to us. If it was `true`, somebody else holds it: we try again. Because the swap is atomic, two threads can never both see `false`.

This is a **spin lock**. It never asks the OS for anything. That makes it the simplest possible lock, and, as section 6 shows, usually a bad one. Building it is still the best way to understand what every lock must do.

## 2. The lock type and its `Sync` promise

Create `src/lib.rs`:

```rust
//! Small concurrency building blocks written in Phase 14.

pub mod spin_lock;
```

and the module file `src/spin_lock.rs` (the tests at the end are explained in section 5):

```rust
//! A spin lock: the simplest lock, built from one atomic flag.

use std::cell::UnsafeCell;
use std::hint::spin_loop;
use std::ops::{Deref, DerefMut};
use std::sync::atomic::{AtomicBool, Ordering};

pub struct SpinLock<T> {
    locked: AtomicBool,
    value: UnsafeCell<T>,
}

// SAFETY: only one thread at a time can reach `value` (through a Guard),
// so sharing the lock is fine as long as T itself may move between threads.
unsafe impl<T: Send> Sync for SpinLock<T> {}

impl<T> SpinLock<T> {
    pub const fn new(value: T) -> Self {
        SpinLock {
            locked: AtomicBool::new(false),
            value: UnsafeCell::new(value),
        }
    }

    pub fn lock(&self) -> Guard<'_, T> {
        // Acquire: pairs with the Release store in Guard::drop.
        while self.locked.swap(true, Ordering::Acquire) {
            spin_loop(); // tell the CPU that we are waiting in a loop
        }
        Guard { lock: self }
    }
}

/// Proof that we hold the lock. Unlocks when dropped.
pub struct Guard<'a, T> {
    lock: &'a SpinLock<T>,
}

impl<T> Deref for Guard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: the guard exists, so this thread holds the lock.
        unsafe { &*self.lock.value.get() }
    }
}

impl<T> DerefMut for Guard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        // SAFETY: the guard exists, so this thread holds the lock.
        unsafe { &mut *self.lock.value.get() }
    }
}

impl<T> Drop for Guard<'_, T> {
    fn drop(&mut self) {
        // Release: everything we did with the value becomes visible
        // to the next thread whose Acquire swap sees `false`.
        self.lock.locked.store(false, Ordering::Release);
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn no_update_is_lost() {
        let counter = SpinLock::new(0u64);
        thread::scope(|s| {
            for _ in 0..4 {
                s.spawn(|| {
                    for _ in 0..100 {
                        *counter.lock() += 1;
                    }
                });
            }
        });
        assert_eq!(*counter.lock(), 400);
    }

    #[test]
    fn works_with_non_copy_data() {
        let names = SpinLock::new(Vec::new());
        thread::scope(|s| {
            for t in 0..4 {
                let names = &names;
                s.spawn(move || names.lock().push(format!("thread {t}")));
            }
        });
        let mut v = names.lock().clone();
        v.sort();
        assert_eq!(v, ["thread 0", "thread 1", "thread 2", "thread 3"]);
    }
}
```

**What happens** (the type and `Sync`):

1. `SpinLock<T>` holds the flag and the data. The data sits in an `UnsafeCell<T>`, because we want to change it through `&self` (many threads share a `&SpinLock`). `UnsafeCell::get()` returns a raw pointer `*mut T`; turning it into a reference is `unsafe`, and *we* must make sure that no two threads do it at the same time.
2. `new` is a `const fn`, so a `SpinLock` can even live in a `static`.
3. `UnsafeCell<T>` is **not** `Sync`: the compiler cannot know how you will use it. So `SpinLock<T>` is not `Sync` either, and you cannot share it between threads. We promise it with `unsafe impl<T: Send> Sync for SpinLock<T> {}`. The promise is true because the only way to reach the value is through a `Guard`, and only one guard can exist at a time.
4. Why `T: Send` and not `T: Sync`? Through the lock, different threads get `&mut T` one after another. That is like **moving** the value from thread to thread, which is exactly what `Send` allows. `std::sync::Mutex<T>` has the same bound.
5. The `// SAFETY:` comment states the reason for the promise. Every `unsafe` block and every `unsafe impl` should have one (a convention from Lesson 06.6).

Without the `unsafe impl` line, the compiler stops any program that shares the lock between threads. If you comment it out, `cargo build --bin spin_demo` (the program of section 4) fails like this:

```text
error[E0277]: `UnsafeCell<u64>` cannot be shared between threads safely
  --> src/bin/spin_demo.rs:8:21
   |
   ...
   = help: within `SpinLock<u64>`, the trait `Sync` is not implemented for `UnsafeCell<u64>`
note: required because it appears within the type `SpinLock<u64>`
  --> src/spin_lock.rs:8:12
   |
 8 | pub struct SpinLock<T> {
   |            ^^^^^^^^
   = note: required for `&SpinLock<u64>` to implement `Send`
```

> 🔒 **Security:** `unsafe impl Sync` switches off a compiler check for **every** user of your type. If the promise is wrong, safe code elsewhere can create a data race: undefined behaviour, which in the worst case is an exploitable memory bug. Keep such code small, comment the reason, and test it with Miri (section 5).

## 3. The guard

`lock()` does not return `()` or `bool`. It returns a `Guard`, and the guard is the only door to the data:

1. `Deref` gives `&T`, `DerefMut` gives `&mut T`. So `*counter.lock() += 1` and `names.lock().push(…)` work like with a `MutexGuard`. Each `unsafe` block is justified by one fact: a guard exists only while its thread holds the lock.
2. `Drop` releases the lock. You cannot forget to unlock, and you cannot unlock twice, because a value is dropped exactly once.
3. The guard borrows the lock (`&'a SpinLock<T>`), so the borrow checker makes sure the lock outlives every guard.

This is the same design as `std::sync::Mutex` and its `MutexGuard`: the type system turns "please remember to lock and unlock correctly" into rules the compiler checks.

> 📏 **Convention:** Make the unsafe part as small as possible and hide it behind a safe API (Application Programming Interface). Here only two lines are `unsafe`, and no user of `SpinLock` ever writes `unsafe`.

## 4. Why `Acquire` and `Release`

The lock protects the data only if the next owner **sees** what the previous owner wrote. That is the release/acquire pair of Lesson 14.6:

```text
 thread A (owner)                          thread B (next owner)
 *value += 1          (plain write)
 store(false, Release) ─ ─ ─ ─ ─ ─ ─ ─ ─▶ swap(true, Acquire) returns false
                                           *value += 1  → sees A's write
```

- **`Acquire` on the swap that takes the lock**: everything the previous owner did before its release is visible to us after we got the lock.
- **`Release` on the store that frees the lock**: everything we did while holding the lock is visible to the next owner.

What happens with `Relaxed` instead? Create `src/bin/spin_demo.rs`, which uses the library:

```rust
use p14_concurrency::spin_lock::SpinLock;
use std::thread;

fn main() {
    let counter = SpinLock::new(0u64);
    thread::scope(|s| {
        for _ in 0..4 {
            s.spawn(|| {
                for _ in 0..1_000_000 {
                    *counter.lock() += 1;
                }
            });
        }
    });
    println!("expected 4000000, got {}", *counter.lock());
}
```

With the correct orderings, `cargo run --release --bin spin_demo` prints:

```text
expected 4000000, got 4000000
```

Now, as an experiment, change both orderings in `src/spin_lock.rs` to `Ordering::Relaxed` (the `swap` in `lock` and the `store` in `drop`) and run it again, twice:

```text
expected 4000000, got 3999969
expected 4000000, got 3999986
```

**What happens:** each thread still takes the lock correctly (the swap is still atomic, so only one thread at a time is "inside"). But the `+= 1` is a normal load and store of the `u64`. Without release/acquire, the next owner may read the counter **before** the previous owner's write has reached it, on your weakly ordered Mac, so a few increments are lost. The lock looked right and was wrong. Change the orderings back to `Acquire` and `Release` now.

## 5. Testing the lock: tests and Miri

The two tests at the end of `src/spin_lock.rs` are ordinary stress tests: four threads hammer the lock, and the result must be exact. Library tests run with `cargo test --lib`:

```text
running 2 tests
test spin_lock::tests::no_update_is_lost ... ok
test spin_lock::tests::works_with_non_copy_data ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

A stress test can pass by luck, especially on a machine that reorders less. **Miri** (Lesson 06.6) runs the program in an interpreter that checks every memory access, and its **data race detector** reports unsynchronized accesses even when the result happens to be right. Run the tests under Miri (it needs the nightly toolchain with the `miri` component, as in Lesson 06.6):

```bash
cargo +nightly miri test --lib
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.70s
```

With the `Relaxed` experiment of section 4, Miri finds the bug at once:

```text
error: Undefined Behavior: Data race detected between (1) non-atomic write on thread `unnamed-2` and (2) retag write of type `u64` on thread `unnamed-3` at alloc40451
  --> src/spin_lock.rs:50:18
   |
50 |         unsafe { &mut *self.lock.value.get() }
   |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^ (2) just happened here
   |
help: and (1) occurred earlier here
  --> src/spin_lock.rs:74:25
   |
74 |                         *counter.lock() += 1;
   |                         ^^^^^^^^^^^^^^^^^^^^
```

**What happens:** Miri tracks which operations happen-before which. One thread wrote the counter (line 74, the `+= 1` in the test), and another thread created a `&mut` to the same memory (line 50) without any happens-before between the two: a data race, and so undefined behaviour. Miri does not need the bug to cause a wrong number; it sees the missing synchronization itself. (A *retag* is Miri's name for creating a reference; it counts as an access.)

> 🧪 **Testing:** For code with `unsafe` and atomics, use three tools together: stress tests (fast, real hardware), Miri (finds data races and other undefined behaviour in the runs it executes), and loom (explores all interleavings of a small model; Lesson 14.9 shows how to test a lock like this one with `loom::cell::UnsafeCell`).

## 6. Should you use a spin lock?

Almost never in a normal program. Let us measure why. Create `src/bin/spin_vs_mutex.rs`:

```rust
use p14_concurrency::spin_lock::SpinLock;
use std::sync::Mutex;
use std::thread;
use std::time::{Duration, Instant};

const TOTAL: u64 = 2_400_000;

/// `threads` threads share TOTAL increments of one counter.
fn time_it(threads: u64, add_one: impl Fn() + Sync) -> Duration {
    let start = Instant::now();
    thread::scope(|s| {
        for _ in 0..threads {
            s.spawn(|| {
                for _ in 0..TOTAL / threads {
                    add_one();
                }
            });
        }
    });
    start.elapsed()
}

fn main() {
    for threads in [1, 4, 48] {
        let spin = SpinLock::new(0u64);
        let mutex = Mutex::new(0u64);
        let s = time_it(threads, || *spin.lock() += 1);
        let m = time_it(threads, || *mutex.lock().unwrap() += 1);
        println!("{threads:>2} threads: SpinLock {s:>9.1?}   Mutex {m:>9.1?}");
    }
}
```

`cargo run --release --bin spin_vs_mutex`, two runs:

```text
 1 threads: SpinLock     5.3ms   Mutex    12.2ms
 4 threads: SpinLock   158.6ms   Mutex    33.3ms
48 threads: SpinLock      2.3s   Mutex    36.0ms
 1 threads: SpinLock     3.1ms   Mutex    18.5ms
 4 threads: SpinLock   117.0ms   Mutex    42.7ms
48 threads: SpinLock      3.2s   Mutex    35.4ms
```

**What happens:**

1. **One thread**: the spin lock is faster. Taking a free lock is one `swap`, and there is nothing else to do.
2. **Four threads**: the spin lock is 3–5 times slower. Every waiting thread executes `swap(true)` in a loop. Each swap is a *write*, so each one pulls the cache line of the flag to that core, away from the owner, which then needs it back to unlock.
3. **48 threads** on 12 cores (oversubscription): the spin lock is about 60–90 times slower. Now the owner of the lock is often **preempted**: the OS stops it to run another thread, perhaps a spinner. While the owner sleeps, all spinners burn their whole time slice for nothing. `Mutex` puts waiting threads to sleep, so the owner gets a core and finishes quickly.

So a spin lock only makes sense when **all** of these hold: the critical section is tiny, the lock is rarely contended, and the holder cannot be preempted. That is true inside an OS kernel (which can switch off preemption) or on a microcontroller without an OS, and almost never in a normal program. The standard `Mutex` is the better choice: on Linux it first spins a few times and then sleeps with futex (Lesson 14.2); on macOS the system mutex puts waiting threads to sleep.

> ⚡ **Performance:** A spin loop should at least wait with plain **loads** and try the `swap` only when the lock looks free (*test-and-test-and-set*, Exercise 2). Loads let all waiting cores keep a shared copy of the cache line, so they do not steal it from the owner. On the test machine this made the 48-thread case about 3 times faster, and still about 25 times slower than `Mutex`.

## Common mistakes

1. **`Relaxed` in a lock.** The lock still lets one thread in at a time, but the data is not published: lost updates on arm64 and a data race for Miri (sections 4 and 5).
2. **A wrong `Sync` bound.** `unsafe impl<T> Sync for SpinLock<T>` (without `T: Send`) would let you share, for example, an `Rc` between threads through the lock. Copy the bound of `std::sync::Mutex`: `T: Send`.
3. **Returning `&mut T` from `lock()` instead of a guard.** Then nothing unlocks at the right time, and the reference can outlive the lock. Always return a guard that unlocks in `Drop`.
4. **Using a spin lock where threads may wait long.** If the critical section can block (input/output, a sleep, another lock), spinners burn CPU for the whole time. Use `Mutex`.
5. **Writing `unsafe` without a `SAFETY:` comment.** The next reader (including you in a month) cannot check the promise.

## Try it yourself

1. Run `cargo test --lib` and `cargo +nightly miri test --lib`. Then do the `Relaxed` experiment of section 4 and run both again. Which one fails reliably? Put the orderings back.
2. Remove the line `spin_loop();` from `lock` (leave an empty loop body `{}`; the compiler then warns that the import of `spin_loop` is unused) and run `spin_vs_mutex` again. Does the hint change the numbers on your machine? On the test machine the run without the hint was not better (48 threads: `3.6s`); the noise between runs is larger than the difference. Put the line back.
3. In `spin_vs_mutex.rs`, add `12` to the list of thread counts (as many threads as the test machine has cores). Is the spin lock closer to the mutex there, or not? (On the test machine: `12 threads: SpinLock 864.9ms   Mutex 25.2ms`.)

## Exercises

### Exercise 1 — `try_lock` (easy)

Add `pub fn try_lock(&self) -> Option<Guard<'_, T>>` to `SpinLock`: it takes the lock only if it is free right now, otherwise it returns `None` immediately. Use `compare_exchange` (not a loop). Write a test `try_lock_fails_while_locked`: while a guard from `lock()` is alive, `try_lock()` returns `None`; after dropping the guard, it returns `Some`, and the value is still there.

<details>
<summary>Hint</summary>

`compare_exchange(false, true, success, failure)` succeeds only when the flag was `false`. Which ordering does success need, and which failure? (On failure we did not get the lock and read no data.) `Result::ok()` and `Option::map` turn the result into an `Option<Guard>`.

</details>

<details>
<summary>Answer</summary>

Inside `impl<T> SpinLock<T>`:

```rust
    /// Takes the lock only if it is free right now.
    pub fn try_lock(&self) -> Option<Guard<'_, T>> {
        // Acquire on success: pairs with the Release store in Guard::drop.
        self.locked
            .compare_exchange(false, true, Ordering::Acquire, Ordering::Relaxed)
            .ok()
            .map(|_| Guard { lock: self })
    }
```

In `mod tests`:

```rust
    #[test]
    fn try_lock_fails_while_locked() {
        let lock = SpinLock::new(5);
        let guard = lock.lock();
        assert!(lock.try_lock().is_none());
        drop(guard);
        let again = lock.try_lock().expect("lock is free now");
        assert_eq!(*again, 5);
    }
```

```text
test spin_lock::tests::try_lock_fails_while_locked ... ok
```

Why it works: `compare_exchange` writes `true` only if the flag is still `false`, in one atomic step, so two `try_lock` calls can never both succeed. Success uses `Acquire` for the same reason as `lock()`: we will read the data. Failure can be `Relaxed`, because then we return `None` and touch nothing. We use the strong `compare_exchange`, because a spurious failure of the weak version would make `try_lock` return `None` for a free lock.

</details>

### Exercise 2 — Test-and-test-and-set (medium)

Change `lock()` so that a waiting thread spins with `load(Ordering::Relaxed)` and tries the `swap` only when it saw `false`. All tests must still pass (also under Miri). Run `spin_vs_mutex` before and after and compare the numbers.

<details>
<summary>Hint</summary>

Use an outer `loop`: first try `swap(true, Acquire)`; if it returned `false`, return the guard. Otherwise run an inner `while self.locked.load(Relaxed) { spin_loop(); }` and then go round the outer loop again. Why can the inner load be `Relaxed`?

</details>

<details>
<summary>Answer</summary>

```rust
    pub fn lock(&self) -> Guard<'_, T> {
        // Acquire: pairs with the Release store in Guard::drop.
        loop {
            // Try to take the lock: one atomic write.
            if !self.locked.swap(true, Ordering::Acquire) {
                return Guard { lock: self };
            }
            // Busy: wait with plain reads, which do not take the cache line away.
            while self.locked.load(Ordering::Relaxed) {
                spin_loop();
            }
        }
    }
```

Results of `cargo run --release --bin spin_vs_mutex` with this version on the test machine:

```text
 1 threads: SpinLock     3.8ms   Mutex    11.9ms
 4 threads: SpinLock   334.8ms   Mutex    32.3ms
48 threads: SpinLock   885.9ms   Mutex    31.2ms
 1 threads: SpinLock     2.8ms   Mutex    10.2ms
 4 threads: SpinLock    87.1ms   Mutex    28.6ms
48 threads: SpinLock   973.3ms   Mutex    39.7ms
```

Why it works: the inner `load` only *reads*, so all waiting cores can keep a shared copy of the cache line and the owner keeps its own; nobody steals the line until the owner's release store changes it. The inner load can be `Relaxed` because it only decides *when to try*; the `swap` with `Acquire` is what actually takes the lock and synchronizes. With 48 threads the time dropped from 2.3–3.2 s to about 0.9 s. The 4-thread numbers jump around a lot between runs, which is typical for contended spinning. It is still far slower than `Mutex`, which is the main lesson.

</details>

### Exercise 3 — A fair ticket lock (hard)

A plain spin lock is unfair: the thread that happens to swap at the right moment wins, and an unlucky thread can wait for a long time. Write `src/ticket_lock.rs` (add `pub mod ticket_lock;` to `src/lib.rs`) with `TicketLock<T>`: two `AtomicU32` counters, `next_ticket` and `now_serving`. `lock()` takes a ticket with `fetch_add` and spins until `now_serving` equals its ticket; dropping the guard moves `now_serving` on by one. Write two tests: no update is lost (4 threads × 100 increments), and with one thread, `now_serving` goes 0, 1, 2, 3, 4 when the lock is taken and dropped five times. Then compare its speed with `SpinLock` (4 threads, 250,000 increments each).

<details>
<summary>Hint</summary>

Which operation publishes the data to the next owner, and which one receives it? The `fetch_add` on `next_ticket` only hands out numbers. Tests inside `ticket_lock.rs` can read the private fields, because a test module is a child of the module.

</details>

<details>
<summary>Answer</summary>

```rust
//! A ticket lock: a fair spin lock. Threads are served in arrival order.

use std::cell::UnsafeCell;
use std::hint::spin_loop;
use std::ops::{Deref, DerefMut};
use std::sync::atomic::{AtomicU32, Ordering};

pub struct TicketLock<T> {
    next_ticket: AtomicU32,
    now_serving: AtomicU32,
    value: UnsafeCell<T>,
}

// SAFETY: as for SpinLock, only the ticket holder reaches `value`.
unsafe impl<T: Send> Sync for TicketLock<T> {}

impl<T> TicketLock<T> {
    pub const fn new(value: T) -> Self {
        TicketLock {
            next_ticket: AtomicU32::new(0),
            now_serving: AtomicU32::new(0),
            value: UnsafeCell::new(value),
        }
    }

    pub fn lock(&self) -> TicketGuard<'_, T> {
        // Take a number. Relaxed: the number itself publishes nothing.
        let my_ticket = self.next_ticket.fetch_add(1, Ordering::Relaxed);
        // Acquire: pairs with the Release in TicketGuard::drop.
        while self.now_serving.load(Ordering::Acquire) != my_ticket {
            spin_loop();
        }
        TicketGuard { lock: self }
    }
}

pub struct TicketGuard<'a, T> {
    lock: &'a TicketLock<T>,
}

impl<T> Deref for TicketGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: we hold the current ticket.
        unsafe { &*self.lock.value.get() }
    }
}

impl<T> DerefMut for TicketGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        // SAFETY: we hold the current ticket.
        unsafe { &mut *self.lock.value.get() }
    }
}

impl<T> Drop for TicketGuard<'_, T> {
    fn drop(&mut self) {
        // Release: the next ticket holder sees everything we did.
        self.lock.now_serving.fetch_add(1, Ordering::Release);
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn no_update_is_lost() {
        let counter = TicketLock::new(0u64);
        thread::scope(|s| {
            for _ in 0..4 {
                s.spawn(|| {
                    for _ in 0..100 {
                        *counter.lock() += 1;
                    }
                });
            }
        });
        assert_eq!(*counter.lock(), 400);
    }

    #[test]
    fn tickets_are_served_in_order() {
        let lock = TicketLock::new(());
        for expected in 0..5 {
            assert_eq!(lock.now_serving.load(Ordering::Relaxed), expected);
            drop(lock.lock());
        }
        assert_eq!(lock.next_ticket.load(Ordering::Relaxed), 5);
    }
}
```

`cargo test --lib` and `cargo +nightly miri test --lib` pass all five tests of the phase library (including the three of `spin_lock`). A small comparison program (4 threads × 250,000 increments for each lock) printed on the test machine, in two runs:

```text
SpinLock  : 1000000 in 32.1ms
TicketLock: 1000000 in 1.5s
SpinLock  : 1000000 in 21.1ms
TicketLock: 1000000 in 332.9ms
```

Why it works: `fetch_add` gives every thread a different ticket, in arrival order, and only the thread whose ticket equals `now_serving` may enter. Only the owner ever changes `now_serving`, by one, with `Release`; the next ticket holder reads it with `Acquire`, so the data is handed over correctly. The lock is **fair** (first come, first served), but fairness has a price: the lock must go to exactly the next thread, even if that thread is not running at the moment, while a plain spin lock goes to whichever thread is ready. That is why it was about 15 to 50 times slower here.

</details>

## Quiz

1. Why is `swap(true)` enough to take a spin lock safely?

<details>
<summary>Answer</summary>

It sets the flag and returns the old value in one atomic step. Only one thread can get the old value `false`; all others get `true` and must try again.

</details>

2. Why does `SpinLock<T>` need `unsafe impl Sync`, and why is the bound `T: Send`?

<details>
<summary>Answer</summary>

It contains an `UnsafeCell<T>`, which is not `Sync`, so the compiler does not make `SpinLock<T>` `Sync`. We promise it ourselves, because only one guard can access the value at a time. The bound is `T: Send` because the lock hands the value (as `&mut T`) from one thread to the next, which is like moving it between threads.

</details>

3. What two jobs does the guard type do?

<details>
<summary>Answer</summary>

It gives access to the data (`Deref`/`DerefMut`) only while the lock is held, and it releases the lock in `Drop`, so you cannot forget to unlock or unlock twice.

</details>

4. What goes wrong if the lock uses `Relaxed` for both the swap and the store?

<details>
<summary>Answer</summary>

Mutual exclusion still works, but the writes of the previous owner are not guaranteed to be visible to the next owner. On arm64 some increments are lost, and Miri reports a data race.

</details>

5. Why is a spin lock so slow with 48 threads on 12 cores?

<details>
<summary>Answer</summary>

The lock owner is often preempted by the OS. While it does not run, nobody can release the lock, and all spinning threads waste their time slices. A `Mutex` puts waiters to sleep, so the owner gets CPU time and finishes.

</details>

6. What does Miri find that a stress test may not?

<details>
<summary>Answer</summary>

A data race: two accesses to the same memory without happens-before, at least one a write. Miri detects the missing synchronization itself, even in runs where the final result happens to be correct.

</details>

## Summary

- A spin lock is an `AtomicBool` plus a loop: `swap(true)` until the old value is `false`; unlock with `store(false)`.
- The data lives in an `UnsafeCell<T>`. You promise thread safety with `unsafe impl<T: Send> Sync`, justified in a `SAFETY:` comment.
- A guard with `Deref`, `DerefMut` and `Drop` is the only way to the data and always unlocks.
- Take the lock with `Acquire`, release it with `Release`. With `Relaxed`, updates were lost on the Mac and Miri reported a data race.
- Test unsafe concurrent code with stress tests, Miri and loom.
- Spin locks lose badly under contention and oversubscription. Use `std::sync::Mutex` in normal programs; spin locks belong in kernels and on bare hardware.

## Sources and further reading

- [Rust Atomics and Locks, chapter 4 — Building Our Own Spin Lock](https://marabos.nl/atomics/building-spinlock.html) — Mara Bos builds the same lock step by step, with the guard and the safety reasoning.
- [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html) — the rules for getting references from an `UnsafeCell`.
- [std::hint::spin_loop](https://doc.rust-lang.org/std/hint/fn.spin_loop.html) — what the spin hint does.
- [The Rustonomicon — Send and Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html) — when it is correct to implement `Send` and `Sync` yourself.
- [Miri on GitHub](https://github.com/rust-lang/miri) — how Miri works and what its data race detector checks.

---

[🏠 Start](../START-HERE.md) · [← 14.6 Memory ordering](06-memory-ordering.md) · [📋 Phase 14](README.md) · [14.8 Lock-free data structures →](08-lock-free-data-structures.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
