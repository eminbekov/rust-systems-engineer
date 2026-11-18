<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.7 Building a spin lock](07-building-a-spin-lock.md) · [📋 Phase 14](README.md) · [14.9 Testing concurrent code with loom →](09-testing-concurrent-code-with-loom.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/08-lock-free-data-structures.md) <!-- nav -->

# Lesson 14.8 — Lock-free data structures

> **Phase 14 — Concurrency and async in depth** · Lesson 8 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What "lock-free" and "wait-free" really promise, and how they differ from "uses no `Mutex`".
- How to build a single-producer, single-consumer ring buffer with two atomic indexes and release/acquire.
- How the type system can enforce "only one producer and one consumer".
- The two hard problems of lock-free structures with pointers: memory reclamation and ABA (a value changes from A to B and back to A).
- Why lock-free is not automatically fast, and when to use `crossbeam` instead of your own code.

## Before you start

- You finished [Lesson 14.7 — Building a spin lock](07-building-a-spin-lock.md). You know `UnsafeCell`, `unsafe impl Sync`, `SAFETY:` comments, Miri and the `src/lib.rs` of the phase project.
- You know compare-and-swap loops and false sharing from [Lesson 14.5](05-atomics.md), release/acquire from [Lesson 14.6](06-memory-ordering.md), and raw pointers and `Box::into_raw` from [Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md).
- You know linked lists from [Lesson 07.4](../07-algorithms-and-data-structures/04-linked-lists.md) and queues from [Lesson 07.5](../07-algorithms-and-data-structures/05-stacks-and-queues.md).
- In `~/projects/rust/code/p14-concurrency`, `crossbeam = "0.8"` from Lesson 14.4 is already in `Cargo.toml`.

## New words

| Word | Meaning |
|---|---|
| **blocking** (algorithm) | If one thread is stopped in the middle (for example while holding a lock), other threads can be stopped forever too. |
| **lock-free** | At any moment, **some** thread makes progress, even if other threads are stopped at any point. A CAS (compare-and-swap) loop is lock-free: if your CAS fails, another thread's CAS succeeded. |
| **wait-free** | **Every** thread finishes its operation in a bounded number of its own steps, whatever the others do. |
| **SPSC** (single-producer, single-consumer) | A queue with exactly one thread that adds and exactly one thread that removes. |
| **ring buffer** | A fixed-size array used as a queue: the indexes go round and round, `index % capacity`. |
| **memory reclamation** | Freeing memory that other threads may still be reading. The hard part of lock-free structures with pointers. |
| **use-after-free** | Reading or writing memory that was already freed. Undefined behaviour. |
| **ABA problem** | A CAS sees the value A, does not notice that it changed to B and back to A in between, and succeeds when it should fail. |
| **Treiber stack** | The classic lock-free stack: a linked list whose `head` pointer is changed with CAS. |
| **`AtomicPtr<T>`** | An atomic raw pointer `*mut T`, with `load`, `store`, `swap` and `compare_exchange`. |
| **epoch-based reclamation** | A way to free memory safely in lock-free structures: memory is freed only after every thread has left the "epoch" in which it could have seen it. `crossbeam-epoch` implements it. |

## 1. What lock-free means

"Lock-free" sounds like "uses no lock", but it is a promise about **progress**. Imagine that the OS (Operating System) stops a thread at the worst moment and does not run it again for a second:

| Kind | If one thread is stopped at a bad moment… | Example |
|---|---|---|
| **blocking** | others may wait for it forever | `Mutex`: the stopped thread holds the lock |
| **lock-free** | others still finish their operations | a CAS loop: a failed CAS means *someone else* succeeded |
| **wait-free** | everyone finishes in a bounded number of steps | `fetch_add` (one `ldadd` on your Mac); the ring buffer of section 2 |

The spin lock of Lesson 14.7 uses no `Mutex`, but it is **not** lock-free: if the owner is stopped, every spinner waits. On the other hand, a CAS loop can retry many times, so it is lock-free but not wait-free.

Why care? Lock-free code cannot deadlock, it is safe to use where a thread may be interrupted (for example inside a signal handler, Lesson 13.7), and it avoids the bad case where a preempted lock holder stops everybody. The price: it is much harder to write correctly, and, as section 4 shows, it is not automatically faster.

## 2. A lock-free SPSC ring buffer

The simplest useful lock-free structure is a queue with **one** producer and **one** consumer. Think of a round table with a fixed number of plates. The cook puts food on the next plate; the guest eats from the oldest full plate. Two numbers are enough:

```text
 capacity = 4            head = 5 (next to read)      tail = 7 (next to write)
 slot:      0      1      2      3
          [ 4 ]  [ 5 ]  [ 6 ]  [ 3 ]
                   ▲             ▲
                head%4=1      tail%4=3
 items 5 and 6 are in the ring (tail - head = 2); slots 3 and 0 are free.
```

- `tail` is changed **only by the producer**; `head` is changed **only by the consumer**. The indexes only grow; the slot is `index % capacity`.
- Empty: `head == tail`. Full: `tail - head == capacity`.
- The producer writes the slot and **then** publishes it by moving `tail` with `Release`. The consumer reads `tail` with `Acquire` and only then reads the slot. That is the "data plus flag" pattern of Lesson 14.6, with `tail` as the flag.
- In the other direction, the consumer publishes "I am done with this slot" by moving `head` with `Release`, and the producer reads `head` with `Acquire` before it reuses a slot.

Because each index has a single writer, no CAS is needed: plain loads and stores do the job. Every `push` and `pop` finishes in a few steps, so this ring buffer is even **wait-free**.

To keep the `unsafe` part small, this version stores `T: Copy + Default` values: every slot always holds a valid value (filled with `T::default()` at the start), and copying a value out leaves nothing to drop. (Real ring buffers for any `T` use `MaybeUninit<T>` and must drop the remaining items carefully; `crossbeam`'s `ArrayQueue` does that.)

Create `src/spsc.rs` and add `pub mod spsc;` to `src/lib.rs`:

```rust
//! A bounded single-producer, single-consumer (SPSC) ring buffer without locks.

use std::cell::UnsafeCell;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

struct Ring<T> {
    slots: Box<[UnsafeCell<T>]>,
    head: AtomicUsize, // the next slot to read; only the consumer changes it
    tail: AtomicUsize, // the next slot to write; only the producer changes it
}

// SAFETY: the producer writes only slots that the consumer cannot read yet
// (between head and tail they are "free"), and publishes them with a Release
// store of `tail`; the consumer reads only slots it acquired from `tail`.
unsafe impl<T: Send> Sync for Ring<T> {}

pub struct Producer<T> {
    ring: Arc<Ring<T>>,
}

pub struct Consumer<T> {
    ring: Arc<Ring<T>>,
}

/// Creates a ring buffer that holds up to `capacity` items.
pub fn channel<T: Copy + Default>(capacity: usize) -> (Producer<T>, Consumer<T>) {
    assert!(capacity > 0, "capacity must be at least 1");
    let slots = (0..capacity).map(|_| UnsafeCell::new(T::default())).collect();
    let ring = Arc::new(Ring {
        slots,
        head: AtomicUsize::new(0),
        tail: AtomicUsize::new(0),
    });
    (
        Producer {
            ring: Arc::clone(&ring),
        },
        Consumer { ring },
    )
}

impl<T: Copy> Producer<T> {
    /// Adds `item`, or gives it back when the ring is full.
    pub fn push(&mut self, item: T) -> Result<(), T> {
        let ring = &*self.ring;
        let tail = ring.tail.load(Ordering::Relaxed); // only we write it
        // Acquire: pairs with the Release store of `head` in pop, so the
        // consumer has finished reading the slot we are about to reuse.
        let head = ring.head.load(Ordering::Acquire);
        if tail - head == ring.slots.len() {
            return Err(item); // full
        }
        let slot = &ring.slots[tail % ring.slots.len()];
        // SAFETY: this slot is free: the consumer reads only below `tail`.
        unsafe { *slot.get() = item };
        // Release: publishes the slot we just wrote to the consumer.
        ring.tail.store(tail + 1, Ordering::Release);
        Ok(())
    }
}

impl<T: Copy> Consumer<T> {
    /// Takes the oldest item, or `None` when the ring is empty.
    pub fn pop(&mut self) -> Option<T> {
        let ring = &*self.ring;
        let head = ring.head.load(Ordering::Relaxed); // only we write it
        // Acquire: pairs with the Release store of `tail` in push.
        let tail = ring.tail.load(Ordering::Acquire);
        if head == tail {
            return None; // empty
        }
        let slot = &ring.slots[head % ring.slots.len()];
        // SAFETY: the producer published this slot and will not touch it
        // until we move `head` past it.
        let item = unsafe { *slot.get() };
        // Release: tells the producer that the slot may be reused.
        ring.head.store(head + 1, Ordering::Release);
        Some(item)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn full_and_empty() {
        let (mut tx, mut rx) = channel::<u32>(2);
        assert_eq!(rx.pop(), None);
        assert_eq!(tx.push(1), Ok(()));
        assert_eq!(tx.push(2), Ok(()));
        assert_eq!(tx.push(3), Err(3)); // full
        assert_eq!(rx.pop(), Some(1));
        assert_eq!(tx.push(3), Ok(()));
        assert_eq!(rx.pop(), Some(2));
        assert_eq!(rx.pop(), Some(3));
        assert_eq!(rx.pop(), None);
    }

    #[test]
    fn keeps_order_across_threads() {
        const N: u64 = 1000;
        let (mut tx, mut rx) = channel::<u64>(8);
        let producer = thread::spawn(move || {
            for i in 0..N {
                while tx.push(i).is_err() {
                    std::hint::spin_loop(); // full: wait for the consumer
                }
            }
        });
        let mut expected = 0;
        while expected < N {
            if let Some(x) = rx.pop() {
                assert_eq!(x, expected, "items must come out in order");
                expected += 1;
            }
        }
        producer.join().unwrap();
    }
}
```

**What happens:**

1. `Ring<T>` holds the slots (`UnsafeCell` because both threads touch them through a shared `Arc`) and the two indexes. The `unsafe impl Sync` promise is explained in its `SAFETY:` comment: the two threads never touch the same slot at the same time, and every hand-over goes through a release/acquire pair.
2. `channel` creates **one** `Producer` and **one** `Consumer` that share the ring. Neither type implements `Clone`, and `push`/`pop` take `&mut self`. So the compiler itself guarantees "single producer, single consumer": you cannot get a second producer, and you cannot call `push` from two threads on the same producer. That is what makes the plain stores of `tail` and `head` correct.
3. `push` reads its own `tail` with `Relaxed` (nobody else writes it), and `head` with `Acquire`. If the ring is full, it gives the item back in `Err` instead of waiting: the caller decides what to do (spin, yield, drop the item). This is backpressure again.
4. `push` writes the slot and then stores `tail + 1` with `Release`; `pop` loads `tail` with `Acquire`, reads the slot, and stores `head + 1` with `Release`. Each side waits for nothing and never retries: wait-free.
5. Both handles are `Send` (an `Arc<Ring<T>>` is `Send` because `Ring<T>` is `Send + Sync`), so you can move the producer into another thread, as the second test does.

Run the tests with `cargo test --lib`, and under Miri with `cargo +nightly miri test --lib` (Lesson 14.7):

```text
test spsc::tests::full_and_empty ... ok
test spsc::tests::keeps_order_across_threads ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Miri reports `test result: ok. 2 passed` as well: no data race and no undefined behaviour in those runs. A small program that uses it, `src/bin/spsc_demo.rs`:

```rust
use p14_concurrency::spsc;
use std::thread;

fn main() {
    let (mut tx, mut rx) = spsc::channel::<u64>(4);
    let producer = thread::spawn(move || {
        for i in 1..=10 {
            while tx.push(i).is_err() {
                thread::yield_now(); // full: let the consumer run
            }
        }
    });
    let mut got = Vec::new();
    while got.len() < 10 {
        match rx.pop() {
            Some(x) => got.push(x),
            None => thread::yield_now(), // empty: let the producer run
        }
    }
    producer.join().unwrap();
    println!("received {got:?}");
}
```

```text
received [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

`thread::yield_now()` tells the OS "I have nothing to do right now, run someone else". It is a little kinder than pure spinning when the other side may not be running.

## 3. Pointers make it hard: the Treiber stack

The ring buffer had a fixed array and two indexes with one writer each. Most lock-free structures are more ambitious: **many** threads push and pop, and the structure grows with heap nodes. The classic example is the **Treiber stack**: a linked list whose `head` is an `AtomicPtr<Node<T>>`.

**Push** is easy and safe: make a node, point its `next` to the current head, and CAS the head from "old head" to "my node". If another thread pushed in between, the CAS fails and we retry (a CAS loop, Lesson 14.5). Exercise 3 writes it.

**Pop** looks just as easy: read `head`, read `head.next`, CAS the head from `head` to `next`, and return the node's value. But two problems hide in it:

```text
 1. Use-after-free (memory reclamation)
    thread A: reads head = N1, is about to read N1.next ...      (A is paused)
    thread B: pops N1 and FREES it
    thread A: reads N1.next  ← reads freed memory: undefined behaviour

 2. ABA
    thread A: reads head = N1, next = N2, is about to CAS(N1 → N2)   (A is paused)
    thread B: pops N1, pops N2 (frees N2), pushes N1 again (same address!)
    stack is now: N1 → N3
    thread A: CAS(N1 → N2) SUCCEEDS, because head is N1 again
    head is now N2, a freed node. The stack is corrupted.
```

The ABA problem is called that because the value went from A to B and back to A, and the CAS cannot tell. Both problems come from the same question: **when is it safe to free a node that other threads might still be looking at?** A garbage collector would answer it for you; Rust has none, so lock-free structures need a **memory reclamation** scheme:

- **Epoch-based reclamation** (`crossbeam-epoch`): threads "pin" the current epoch while they work; a removed node is freed only after every thread has moved past the epoch in which it was removed.
- **Hazard pointers**: each thread publishes the pointers it is reading right now; nobody frees a node that appears in that list.
- **Avoid the question**: design the API (Application Programming Interface) so that a node is freed only by a thread that owns it exclusively. Exercise 3 does this with a stack that has `push` and `take_all` (which swaps the whole list out at once) but no single `pop`.

> 🔒 **Security:** Use-after-free bugs are among the most dangerous memory bugs, because an attacker can sometimes control what gets written into the freed memory. In lock-free code they hide behind rare timings. If you write a lock-free structure with `unsafe` pointers, run every test under Miri, which reports use-after-free, data races and memory leaks.

## 4. Is lock-free faster? Measure

Let us compare three queues with one producer and one consumer moving 5 million numbers: a `Mutex<VecDeque>`, `crossbeam::queue::ArrayQueue` (a lock-free bounded queue that allows many producers and many consumers), and our ring buffer. Create `src/bin/queue_race.rs`:

```rust
use crossbeam::queue::ArrayQueue;
use p14_concurrency::spsc;
use std::collections::VecDeque;
use std::hint::spin_loop;
use std::sync::Mutex;
use std::thread;
use std::time::Instant;

const N: u64 = 5_000_000;

fn main() {
    // 1. A VecDeque behind a Mutex.
    let queue = Mutex::new(VecDeque::with_capacity(1024));
    let start = Instant::now();
    thread::scope(|s| {
        s.spawn(|| {
            for i in 0..N {
                loop {
                    let mut q = queue.lock().unwrap();
                    if q.len() < 1024 {
                        q.push_back(i);
                        break;
                    }
                }
            }
        });
        let mut sum = 0;
        let mut got = 0;
        while got < N {
            if let Some(x) = queue.lock().unwrap().pop_front() {
                sum += x;
                got += 1;
            }
        }
        assert_eq!(sum, N * (N - 1) / 2);
    });
    println!("Mutex<VecDeque> : {:.1?}", start.elapsed());

    // 2. crossbeam's lock-free ArrayQueue (many producers and consumers allowed).
    let queue = ArrayQueue::new(1024);
    let start = Instant::now();
    thread::scope(|s| {
        s.spawn(|| {
            for i in 0..N {
                while queue.push(i).is_err() {
                    spin_loop();
                }
            }
        });
        let mut sum = 0;
        let mut got = 0;
        while got < N {
            if let Some(x) = queue.pop() {
                sum += x;
                got += 1;
            }
        }
        assert_eq!(sum, N * (N - 1) / 2);
    });
    println!("ArrayQueue      : {:.1?}", start.elapsed());

    // 3. Our SPSC ring buffer.
    let (mut tx, mut rx) = spsc::channel::<u64>(1024);
    let start = Instant::now();
    thread::scope(|s| {
        s.spawn(move || {
            for i in 0..N {
                while tx.push(i).is_err() {
                    spin_loop();
                }
            }
        });
        let mut sum = 0;
        let mut got = 0;
        while got < N {
            if let Some(x) = rx.pop() {
                sum += x;
                got += 1;
            }
        }
        assert_eq!(sum, N * (N - 1) / 2);
    });
    println!("our SPSC ring   : {:.1?}", start.elapsed());
}
```

`cargo run --release --bin queue_race`, two runs:

```text
Mutex<VecDeque> : 254.3ms
ArrayQueue      : 41.8ms
our SPSC ring   : 254.6ms
Mutex<VecDeque> : 416.2ms
ArrayQueue      : 33.0ms
our SPSC ring   : 345.7ms
```

**What happens:**

1. Every version checks that the sum of everything received is correct (`N * (N - 1) / 2`), so they all really move all 5 million numbers.
2. `ArrayQueue` is about 6–12 times faster than the mutex.
3. Our ring buffer, simple and wait-free, is **as slow as the mutex**. Lock-free does not mean fast!
4. Why? Two cache effects from Lesson 14.5. First, `head` and `tail` sit next to each other in memory, in **one cache line**: every store of `tail` by the producer takes the line away from the consumer, and every store of `head` takes it back (false sharing). Second, every `push` loads `head` and every `pop` loads `tail`, even when there is obviously room or data, so each operation needs the other core's freshest cache line.
5. `ArrayQueue` avoids both: it pads its indexes to separate cache lines (`CachePadded`) and touches the other side's data far less often. In Exercise 2 you apply the same two tricks; on the test machine that made our ring buffer about 10 times faster, a little faster than `ArrayQueue`.

> ⚡ **Performance:** In concurrent code, the cost is mostly **cache-line traffic** between cores, not the number of instructions. Count how often each operation writes a line that another core reads. A lock-free design that writes shared lines on every operation can lose to a simple lock.

> 💡 **Tip:** In real programs, use proven crates: `crossbeam::queue::ArrayQueue` (bounded) and `SegQueue` (unbounded) for MPMC (multi-producer, multi-consumer) queues, `crossbeam::channel` for channels with blocking `recv`, and `crossbeam-epoch` if you really must build a structure with reclaimed nodes. Write your own lock-free code to learn, or when a measurement proves you need it, and then test it with Miri and loom.

## Common mistakes

1. **"No `Mutex`, so it is lock-free."** A spin lock uses no `Mutex` and is still blocking. Lock-free is a progress guarantee.
2. **Two producers on an SPSC structure.** The plain `store` of `tail` is only correct with one writer. Enforce it with types (no `Clone`, `&mut self`), not with a comment.
3. **Freeing a node that another thread may still read.** Use-after-free. Use a reclamation scheme or an API where only an exclusive owner frees nodes.
4. **Trusting a CAS on a pointer to detect "nothing changed".** ABA: the same address can come back with different contents.
5. **Assuming lock-free is faster.** Measure. Watch for false sharing and for loading the other side's index on every operation.

## Try it yourself

1. Run `cargo test --lib` and `cargo +nightly miri test --lib`. In `spsc.rs`, change the `Release` store of `tail` in `push` to `Relaxed` and run Miri again. It reports `Undefined Behavior: Data race detected` between the write of the slot in `push` and the read of the slot in `pop`: without the release, the consumer may read a slot before the producer's write is visible. Put it back.
2. Try to break the "single producer" rule: in `spsc_demo.rs`, write `let tx2 = tx.clone();`. The compiler answers ``error[E0599]: no method named `clone` found for struct `Producer<T>` in the current scope``. That error is the single-producer guarantee at work. Remove the line.
3. Run `queue_race` with `spsc::channel::<u64>(16)` instead of `1024` (a tiny ring). How does the time change? (On the test machine our ring took about 214–235 ms, slightly faster than with 1024 slots; with this first version the cost is in the shared cache line, not in the size.) Then try it again after Exercise 2.

## Exercises

### Exercise 1 — Classify the progress guarantee (easy)

Is each operation blocking, lock-free or wait-free? (1) `AtomicU64::fetch_add` on your Mac; (2) a `compare_exchange_weak` loop that adds 1 with an upper limit (Lesson 14.5); (3) `SpinLock::lock` (Lesson 14.7); (4) `Producer::push` of the ring buffer; (5) `Mutex::lock`.

<details>
<summary>Hint</summary>

For each one, ask: if another thread is stopped at the worst moment, can this operation still finish? And is the number of steps bounded?

</details>

<details>
<summary>Answer</summary>

1. Wait-free: one `ldadd` instruction, always done in one step.
2. Lock-free: it may retry, but each failed CAS means another thread's CAS succeeded, so some thread always progresses. Not wait-free, because one unlucky thread could retry many times.
3. Blocking: if the lock holder is stopped, the spinners never get the lock.
4. Wait-free: a fixed number of steps and no retries; when the ring is full it returns `Err` instead of waiting.
5. Blocking: a stopped lock holder blocks everyone who waits for the lock.

</details>

### Exercise 2 — Make the ring buffer fast (medium)

Apply two changes to `src/spsc.rs` and measure with `queue_race` before and after: (1) wrap `head` and `tail` in `crossbeam::utils::CachePadded` so that they live in different cache lines; (2) let the producer remember the last `head` it loaded (a field `head_seen`) and load `head` again only when the ring *looks* full; let the consumer do the same with `tail` (`tail_seen`) when the ring *looks* empty. All tests must still pass, also under Miri.

<details>
<summary>Hint</summary>

`CachePadded::new(AtomicUsize::new(0))`; a `CachePadded<T>` derefs to `T`, so `ring.head.load(…)` stays the same. The remembered value can only be *too old*, never too new: if the producer thinks the ring is full, it may really have free space, so it must check again before giving up. Can a too-old `head_seen` ever make the producer overwrite a slot that the consumer still needs?

</details>

<details>
<summary>Answer</summary>

The changed parts of `src/spsc.rs`:

```rust
use crossbeam::utils::CachePadded;
use std::cell::UnsafeCell;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

struct Ring<T> {
    slots: Box<[UnsafeCell<T>]>,
    head: CachePadded<AtomicUsize>, // the next slot to read; only the consumer changes it
    tail: CachePadded<AtomicUsize>, // the next slot to write; only the producer changes it
}

pub struct Producer<T> {
    ring: Arc<Ring<T>>,
    head_seen: usize, // the last value of `head` we loaded
}

pub struct Consumer<T> {
    ring: Arc<Ring<T>>,
    tail_seen: usize, // the last value of `tail` we loaded
}
```

In `channel`, create the indexes with `CachePadded::new(AtomicUsize::new(0))` and the handles with `head_seen: 0` and `tail_seen: 0`. The new `push`:

```rust
    pub fn push(&mut self, item: T) -> Result<(), T> {
        let ring = &*self.ring;
        let tail = ring.tail.load(Ordering::Relaxed); // only we write it
        if tail - self.head_seen == ring.slots.len() {
            // Looks full. Acquire: pairs with the Release store of `head` in
            // pop, so the consumer has finished reading the slot we reuse.
            self.head_seen = ring.head.load(Ordering::Acquire);
            if tail - self.head_seen == ring.slots.len() {
                return Err(item); // really full
            }
        }
        let slot = &ring.slots[tail % ring.slots.len()];
        // SAFETY: this slot is free: the consumer reads only below `tail`.
        unsafe { *slot.get() = item };
        // Release: publishes the slot we just wrote to the consumer.
        ring.tail.store(tail + 1, Ordering::Release);
        Ok(())
    }
```

The new start of `pop`:

```rust
        let head = ring.head.load(Ordering::Relaxed); // only we write it
        if head == self.tail_seen {
            // Looks empty. Acquire: pairs with the Release store of `tail` in push.
            self.tail_seen = ring.tail.load(Ordering::Acquire);
            if head == self.tail_seen {
                return None; // really empty
            }
        }
```

The rest of `pop` stays as it was. Results on the test machine (the first two lines do not change):

```text
Mutex<VecDeque> : 266.6ms
ArrayQueue      : 39.0ms
our SPSC ring   : 13.1ms
Mutex<VecDeque> : 283.6ms
ArrayQueue      : 43.6ms
our SPSC ring   : 28.0ms
```

With only the second change (no padding) the ring needed about 40 ms; both changes together brought it to 13–30 ms, from about 250–350 ms before. `cargo test --lib` and `cargo +nightly miri test --lib` still pass.

Why it works: padding puts each index in its own cache line, so the producer's stores of `tail` no longer steal the line that holds `head`. Remembering the other index means that most operations touch only the core's own line; the other side's line is fetched only when the ring looks full or empty. A remembered value is always an *older* value of the same index, and indexes only grow, so the producer may think there is *less* space than there really is, never more: it can never overwrite an unread slot. When it does use a fresh `head` value, that value came from an `Acquire` load, so the consumer's reads of those slots happened-before.

</details>

### Exercise 3 — A push-only stack with `take_all` (hard)

Write `src/push_stack.rs` (add `pub mod push_stack;` to `src/lib.rs`): a lock-free stack `PushStack<T>` built on `AtomicPtr<Node<T>>` with three operations: `new()`, `push(&self, value)` (a Treiber push with a CAS loop) and `take_all(&self) -> Vec<T>`, which removes **all** values at once (newest first) by swapping the head with a null pointer. Implement `Drop` so that remaining nodes are freed. Write tests: one thread (push 1, 2, 3 → `take_all` gives `[3, 2, 1]`); four threads push 100 values each while a fifth calls `take_all` 50 times, and in the end every value 0..400 was taken exactly once; and a test that leaves a `String` on the stack when it is dropped. Run everything under Miri.

<details>
<summary>Hint</summary>

`Box::into_raw(Box::new(node))` gives a `*mut Node<T>`; `Box::from_raw(ptr)` takes ownership back (and frees it when dropped). In `push`, set `(*node).next = head` inside the loop before each CAS. Why does `take_all` avoid both use-after-free and ABA? (Who can still reach the nodes after the `swap`?)

</details>

<details>
<summary>Answer</summary>

```rust
//! A lock-free stack that supports `push` and `take_all`, but no single `pop`.

use std::ptr;
use std::sync::atomic::{AtomicPtr, Ordering};

struct Node<T> {
    value: T,
    next: *mut Node<T>,
}

pub struct PushStack<T> {
    head: AtomicPtr<Node<T>>,
}

// SAFETY: nodes are owned by the stack; values are moved in by `push` and
// out by `take_all`, so the stack may be sent or shared when T may be sent.
unsafe impl<T: Send> Send for PushStack<T> {}
unsafe impl<T: Send> Sync for PushStack<T> {}

impl<T> PushStack<T> {
    pub const fn new() -> Self {
        PushStack {
            head: AtomicPtr::new(ptr::null_mut()),
        }
    }

    pub fn push(&self, value: T) {
        let node = Box::into_raw(Box::new(Node {
            value,
            next: ptr::null_mut(),
        }));
        let mut head = self.head.load(Ordering::Relaxed);
        loop {
            // SAFETY: `node` is ours alone until the CAS below publishes it.
            unsafe { (*node).next = head };
            // Release: publishes the node's contents to `take_all`.
            match self
                .head
                .compare_exchange_weak(head, node, Ordering::Release, Ordering::Relaxed)
            {
                Ok(_) => return,
                Err(actual) => head = actual, // another push won: retry
            }
        }
    }

    /// Takes every value pushed so far, newest first.
    pub fn take_all(&self) -> Vec<T> {
        // Acquire: pairs with the Release CAS in push.
        let mut node = self.head.swap(ptr::null_mut(), Ordering::Acquire);
        let mut values = Vec::new();
        while !node.is_null() {
            // SAFETY: the swap made this list ours alone; no other thread can
            // reach these nodes any more, so we may free them.
            let boxed = unsafe { Box::from_raw(node) };
            node = boxed.next;
            values.push(boxed.value);
        }
        values
    }
}

impl<T> Default for PushStack<T> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T> Drop for PushStack<T> {
    fn drop(&mut self) {
        self.take_all(); // free the nodes that are still on the stack
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn newest_first() {
        let stack = PushStack::new();
        stack.push(1);
        stack.push(2);
        stack.push(3);
        assert_eq!(stack.take_all(), vec![3, 2, 1]);
        assert!(stack.take_all().is_empty());
    }

    #[test]
    fn many_threads_lose_nothing() {
        let stack = PushStack::new();
        let mut taken = Vec::new();
        thread::scope(|s| {
            for t in 0..4 {
                let stack = &stack;
                s.spawn(move || {
                    for i in 0..100 {
                        stack.push(t * 100 + i);
                    }
                });
            }
            s.spawn(|| {
                for _ in 0..50 {
                    taken.extend(stack.take_all()); // take while others push
                }
            });
        });
        taken.extend(stack.take_all());
        taken.sort();
        assert_eq!(taken, (0..400).collect::<Vec<_>>());
    }

    #[test]
    fn drop_frees_remaining_nodes() {
        let stack = PushStack::new();
        stack.push(String::from("left on the stack"));
        // dropping `stack` must free the node and the String (Miri checks it)
    }
}
```

`cargo test --lib` and `cargo +nightly miri test --lib` pass all five tests of the phase library (two for `spsc`, three for `push_stack`). Without the `Drop` implementation, Miri catches the leak:

```text
error: memory leaked: alloc42350 (Rust heap, size: 32, align: 8), allocated here:
  --> src/push_stack.rs:28:34
   |
28 |           let node = Box::into_raw(Box::new(Node {
```

Why it works: `push` is the Treiber push: the new node points to the head we saw, and the CAS installs it only if the head is still that value; otherwise we retry with the new head. The `Release` CAS publishes the node's contents. `take_all` replaces the head with null in **one** `swap`, so the whole list becomes ours at once: no other thread can reach those nodes any more, and only pushes of *new* nodes can happen. Therefore nobody can read a node that we free (no use-after-free), and no CAS can ever compare against a node that was removed and reused (no ABA). The `Acquire` swap makes every pushed node's contents visible to us. `Drop` frees what is left, and Miri confirms that nothing leaks.

Why both `unsafe impl` lines? `AtomicPtr<T>` is `Send` and `Sync` for **every** `T`, so without them the compiler would make `PushStack<Rc<i32>>` `Send` automatically: you could push `Rc` values in one thread and drop them in another, a data race on the reference count. Writing the two impls with the bound `T: Send` replaces those automatic impls; with them, sending a `PushStack<Rc<i32>>` to another thread fails with ``error[E0277]: `Rc<{integer}>` cannot be sent between threads safely``.

</details>

## Quiz

1. What does "lock-free" promise, and why is a spin lock not lock-free?

<details>
<summary>Answer</summary>

That some thread always makes progress, even if other threads are stopped at any point. A spin lock is not lock-free: if the thread holding it is stopped, all other threads spin forever without progress.

</details>

2. Why does the SPSC ring buffer need no compare-and-swap?

<details>
<summary>Answer</summary>

Each index has exactly one writer: only the producer changes `tail`, only the consumer changes `head`. A single writer can use plain atomic stores; release/acquire on the indexes hands the slots over safely.

</details>

3. How does the code make sure there is only one producer and one consumer?

<details>
<summary>Answer</summary>

`channel` creates exactly one `Producer` and one `Consumer`, neither implements `Clone`, and `push`/`pop` take `&mut self`, so even one handle cannot be used by two threads at once. The compiler enforces it.

</details>

4. Describe the ABA problem in your own words.

<details>
<summary>Answer</summary>

A thread reads a value A and later does a CAS expecting A. In between, other threads changed it to B and back to A (for example, a freed node's address was reused). The CAS succeeds because it only compares the value, but the situation is different from what the thread saw, so the structure gets corrupted.

</details>

5. Why was the first version of the ring buffer as slow as a mutex?

<details>
<summary>Answer</summary>

Because of cache-line traffic: `head` and `tail` shared one cache line (false sharing), and every operation loaded the other side's index. Each push and pop therefore moved a cache line between the two cores.

</details>

6. Why does `take_all` avoid the reclamation problem that a single `pop` has?

<details>
<summary>Answer</summary>

It removes the whole list with one atomic `swap`. After that, no other thread can reach any of those nodes, so the thread that took them owns them exclusively and can free them safely. A single `pop` must read `head.next` while other threads may pop and free the same node.

</details>

## Summary

- Lock-free means some thread always progresses; wait-free means every thread finishes in bounded steps. "No `Mutex`" alone means neither.
- An SPSC ring buffer needs only two indexes with one writer each and release/acquire hand-overs. The type system (no `Clone`, `&mut self`) enforces one producer and one consumer.
- Lock-free structures with heap nodes face memory reclamation (use-after-free) and ABA. Solutions: epoch-based reclamation, hazard pointers, or an API where only an exclusive owner frees nodes.
- Lock-free is not automatically fast. False sharing and touching the other side's cache line on every operation made our first ring as slow as a mutex; padding and remembered indexes made it about 10 times faster.
- Use `crossbeam` (`ArrayQueue`, `SegQueue`, `channel`, `epoch`) in real programs, and test any `unsafe` lock-free code with Miri and loom.

## Sources and further reading

- [Rust Atomics and Locks, chapter 5 — Building Our Own Channels](https://marabos.nl/atomics/building-channels.html) — Mara Bos builds channels step by step, with the same safety reasoning as our ring buffer.
- [Rust Atomics and Locks, chapter 6 — Building Our Own "Arc"](https://marabos.nl/atomics/building-arc.html) — a lock-free reference count, and when memory may be freed.
- [crossbeam::queue on docs.rs](https://docs.rs/crossbeam/latest/crossbeam/queue/index.html) — `ArrayQueue` and `SegQueue`, the lock-free queues used in practice.
- [crossbeam-epoch on docs.rs](https://docs.rs/crossbeam-epoch) — epoch-based memory reclamation for lock-free data structures.
- [std::sync::atomic::AtomicPtr](https://doc.rust-lang.org/std/sync/atomic/type.AtomicPtr.html) — the atomic pointer type used by the Treiber stack.

---

[🏠 Start](../START-HERE.md) · [← 14.7 Building a spin lock](07-building-a-spin-lock.md) · [📋 Phase 14](README.md) · [14.9 Testing concurrent code with loom →](09-testing-concurrent-code-with-loom.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
