<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.2 Mutexes and read-write locks in depth](02-mutexes-and-rwlocks-in-depth.md) · [📋 Phase 14](README.md) · [14.4 Thread pools, scoped threads and rayon →](04-thread-pools-scoped-threads-and-rayon.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/03-condition-variables.md) <!-- nav -->

# Lesson 14.3 — Condition variables

> **Phase 14 — Concurrency and async in depth** · Lesson 3 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why a thread should not wait for something by checking it again and again.
- How a condition variable (`Condvar`) lets a thread sleep until another thread changes the state.
- The three rules of condition variables: a mutex, a condition, and a loop.
- What spurious wake-ups and lost wake-ups are, and why the rules protect you from both.
- How to build a bounded blocking queue with one mutex and two condition variables, and how to wait with a timeout.

## Before you start

- You finished [Lesson 14.2 — Mutexes and read-write locks in depth](02-mutexes-and-rwlocks-in-depth.md). You know guards, critical sections and `drop(guard)`.
- You know `VecDeque` from [Lesson 07.5 — Stacks and queues](../07-algorithms-and-data-structures/05-stacks-and-queues.md) and the bounded queue of the thread-pool server from [Lesson 06.11](../06-rust-advanced/11-project-multithreaded-web-server.md).
- Keep working in `~/projects/rust/code/p14-concurrency`; each example is a new file in `src/bin/`.

## New words

| Word | Meaning |
|---|---|
| **busy waiting** | Waiting by checking a condition in a loop, again and again, without sleeping. It wastes a whole CPU (Central Processing Unit) core. |
| **condition variable** (`Condvar`) | A place where threads can sleep until another thread says "the state has changed, look again". It always works together with a mutex. |
| **`wait`** | A `Condvar` method that, in one step, unlocks the mutex and puts the thread to sleep. When the thread wakes up, it locks the mutex again before `wait` returns. |
| **notify** (`notify_one`, `notify_all`) | Wake up one waiting thread, or all of them. |
| **predicate** | The yes/no question a waiting thread asks about the state, for example "is the queue not empty?". |
| **spurious wake-up** | A thread returns from `wait` although nobody notified it, or the condition is already false again. It is allowed to happen. |
| **lost wake-up** | A notification is sent when nobody is waiting yet, so it disappears, and the thread that starts waiting later sleeps forever. |
| **producer / consumer** | A thread that creates work items and puts them into a queue / a thread that takes them out and handles them. |
| **bounded queue** | A queue with a maximum size. A producer has to wait when it is full. |
| **blocking** | An operation *blocks* when the calling thread stops and waits inside it (for example `lock()`, `recv()`, `pop()` on an empty blocking queue). |
| **latch** | A one-time gate: threads wait until a counter reaches zero, then all of them may go on. |

## 1. The problem: waiting for something

A worker thread loads a configuration file; the main thread must wait until it is ready. With only a mutex, the main thread can only ask "ready yet?" again and again. Create `src/bin/busy_wait.rs`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let ready = Arc::new(Mutex::new(false));

    let worker_ready = Arc::clone(&ready);
    let worker = thread::spawn(move || {
        thread::sleep(Duration::from_millis(100)); // pretend to load a config
        *worker_ready.lock().unwrap() = true;
    });

    // Busy waiting: ask again and again "are we there yet?"
    let mut checks: u64 = 0;
    while !*ready.lock().unwrap() {
        checks += 1;
    }
    println!("ready after {checks} checks");
    worker.join().unwrap();
}
```

`cargo run --release --bin busy_wait`:

```text
ready after 20770610 checks
```

**What happens:**

1. The worker sleeps 100 ms, then sets the flag to `true` under the lock.
2. The main thread locks, reads the flag, unlocks, and repeats. In 100 ms it checked the flag about 20 million times.
3. During all that time one CPU core was busy at 100% doing nothing useful, and the main thread also competed with the worker for the lock.

You could add `thread::sleep(Duration::from_millis(10))` inside the loop. That saves the CPU, but now the main thread notices the change up to 10 ms late, and you have to guess a good sleep time. What we really want is: **sleep until somebody tells me that the state changed**. That is exactly what a condition variable does.

## 2. `Condvar`: sleep until notified

A condition variable is like the "please ring" bell at a shop counter. You do not stand at the counter shouting "is anybody there?". You ring once and sit down; the shopkeeper comes when the order is ready and calls you.

`std::sync::Condvar` has three important methods:

| Method | What it does |
|---|---|
| `cvar.wait(guard)` | Takes the `MutexGuard`, **unlocks the mutex and sleeps** in one step. When woken, it **locks the mutex again** and returns the guard. |
| `cvar.notify_one()` | Wakes up one thread that is sleeping in `wait` on this condvar (if there is one). |
| `cvar.notify_all()` | Wakes up all of them. |

Here is the same program with a condition variable. The mutex and the condvar always go together, so we keep them in one tuple. Create `src/bin/ready_flag.rs`:

```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));

    let worker_pair = Arc::clone(&pair);
    let worker = thread::spawn(move || {
        thread::sleep(Duration::from_millis(100)); // pretend to load a config
        let (lock, cvar) = &*worker_pair;
        *lock.lock().unwrap() = true; // 1. change the state under the lock
        cvar.notify_one(); // 2. wake up a waiting thread
        println!("worker: config loaded");
    });

    let (lock, cvar) = &*pair;
    let mut ready = lock.lock().unwrap();
    let mut wakeups = 0;
    while !*ready {
        ready = cvar.wait(ready).unwrap(); // unlock, sleep, lock again
        wakeups += 1;
    }
    println!("main: ready after {wakeups} wake-up(s)");
    drop(ready);
    worker.join().unwrap();
}
```

```text
worker: config loaded
main: ready after 1 wake-up(s)
```

**What happens:**

1. `&*worker_pair` turns the `Arc<(Mutex, Condvar)>` into a reference to the tuple, and `let (lock, cvar) = …` gives names to both parts.
2. The main thread locks the mutex and sees `false`. It calls `cvar.wait(ready)`: the guard is handed over, the mutex is unlocked, and the thread sleeps. It uses no CPU while it sleeps.
3. The worker can now take the lock (it is free), sets the flag to `true`, and calls `notify_one`.
4. The main thread wakes up. Before `wait` returns, it locks the mutex again, so `ready` is a normal guard again. The loop checks the flag: `true`, so the loop ends. Only one wake-up was needed, instead of 20 million checks.

```text
 main                                   worker
 lock ─ flag is false
 wait(guard): unlock + sleep  ─────┐
                                   │    lock ─ flag = true ─ unlock
                                   │    notify_one() ──────────┐
 (woken) lock again ◀──────────────┴───────────────────────────┘
 flag is true → leave the loop
```

The important part is step 2: unlocking and starting to sleep happen as **one** step. If they were two steps, the worker could set the flag and notify in the gap between them, and the main thread would sleep forever. That gap is the **lost wake-up** of section 3.

## 3. The three rules

Every correct use of a condition variable follows three rules.

**Rule 1: the condition lives in a mutex-protected variable.** The condvar itself remembers nothing. It is only a waiting room. The truth ("is it ready?", "is the queue empty?") must be in normal data behind the mutex.

**Rule 2: change the state under the lock, then notify.** If you change the flag without the lock, the waiter may check it, see `false`, and go to sleep just after your notification.

**Rule 3: always wait in a loop that checks the condition.** There are two reasons:

- A **spurious wake-up** is allowed: `wait` may return even though nobody called notify. Operating systems allow this because it makes condition variables simpler and faster to implement.
- Even after a real notification, another thread may have run first and changed the state back (for example, it took the only item from the queue).

The loop `while !condition { guard = cvar.wait(guard).unwrap(); }` handles both: after every wake-up, check again.

What goes wrong without rule 1? A notification that nobody is waiting for is simply lost. In this program the worker notifies before the main thread starts waiting. Create `src/bin/lost_wakeup.rs`:

```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    // WRONG: a condition variable without a condition to check.
    let pair = Arc::new((Mutex::new(()), Condvar::new()));

    let worker_pair = Arc::clone(&pair);
    thread::spawn(move || {
        let (_lock, cvar) = &*worker_pair;
        cvar.notify_one(); // "I am done!" ... but nobody is waiting yet
    });

    thread::sleep(Duration::from_millis(100)); // main is a bit late
    let (lock, cvar) = &*pair;
    let guard = lock.lock().unwrap();
    let (_guard, result) = cvar.wait_timeout(guard, Duration::from_secs(1)).unwrap();
    println!("timed out: {}", result.timed_out());
}
```

```text
timed out: true
```

**What happens:** the notification came first and was lost, because no thread was sleeping in `wait` at that moment. With plain `wait` the main thread would sleep forever; `wait_timeout` (section 5) gives up after one second, so we can see the problem. In `ready_flag.rs` the same timing is harmless: the main thread checks the flag *before* it waits, sees `true`, and never sleeps at all.

`Condvar::wait_while` writes the loop for you. It takes the guard and a closure that answers "should I keep waiting?":

```rust
    let (lock, cvar) = &*pair;
    let ready = cvar.wait_while(lock.lock().unwrap(), |ready| !*ready).unwrap();
```

These two lines do the same job as the `let mut ready = …` line and the `while` loop in `ready_flag.rs` (only without counting the wake-ups). `wait_while` is also what the queue in the next section uses.

> ⚠️ **Common mistake:** `if !ready { guard = cvar.wait(guard).unwrap(); }` with `if` instead of `while`. It works in tests and fails in production after a spurious wake-up or when two consumers race for one item. Always use `while` or `wait_while`.

## 4. A bounded blocking queue

Now the classic use: a queue between **producers** (threads that create work) and **consumers** (threads that do the work). It has a maximum size, so a fast producer cannot fill all memory; it has to wait. Lesson 06.11 used `mpsc::sync_channel` for this. Here we build it ourselves, because it shows how such a channel works inside.

There are two different reasons to wait, so there are two condition variables:

| Who waits | For what | Condvar | Who notifies |
|---|---|---|---|
| a consumer in `pop` | "the queue is not empty" | `not_empty` | `push`, after adding an item |
| a producer in `push` | "the queue is not full" | `not_full` | `pop`, after removing an item |

There is also a `closed` flag, so consumers know when no more items will come. Create `src/bin/blocking_queue.rs`:

```rust
use std::collections::VecDeque;
use std::sync::{Condvar, Mutex};
use std::thread;
use std::time::Duration;

struct State<T> {
    items: VecDeque<T>,
    closed: bool,
}

/// A queue with a fixed capacity. `push` waits while it is full,
/// `pop` waits while it is empty.
pub struct BlockingQueue<T> {
    state: Mutex<State<T>>,
    not_empty: Condvar,
    not_full: Condvar,
    capacity: usize,
}

impl<T> BlockingQueue<T> {
    pub fn new(capacity: usize) -> Self {
        assert!(capacity > 0, "capacity must be at least 1");
        BlockingQueue {
            state: Mutex::new(State {
                items: VecDeque::new(),
                closed: false,
            }),
            not_empty: Condvar::new(),
            not_full: Condvar::new(),
            capacity,
        }
    }

    /// Waits for free space, then adds `item`. Returns `Err(item)` if the queue is closed.
    pub fn push(&self, item: T) -> Result<(), T> {
        let state = self.state.lock().unwrap();
        let mut state = self
            .not_full
            .wait_while(state, |s| s.items.len() == self.capacity && !s.closed)
            .unwrap();
        if state.closed {
            return Err(item);
        }
        state.items.push_back(item);
        self.not_empty.notify_one();
        Ok(())
    }

    /// Waits for an item. Returns `None` when the queue is closed and empty.
    pub fn pop(&self) -> Option<T> {
        let state = self.state.lock().unwrap();
        let mut state = self
            .not_empty
            .wait_while(state, |s| s.items.is_empty() && !s.closed)
            .unwrap();
        let item = state.items.pop_front();
        if item.is_some() {
            self.not_full.notify_one();
        }
        item
    }

    /// No more pushes. Waiting threads wake up; `pop` drains what is left.
    pub fn close(&self) {
        self.state.lock().unwrap().closed = true;
        self.not_empty.notify_all();
        self.not_full.notify_all();
    }
}

fn main() {
    let queue = BlockingQueue::new(2);
    thread::scope(|s| {
        s.spawn(|| {
            for i in 1..=5 {
                queue.push(i).unwrap();
                println!("produced {i}");
            }
            queue.close();
        });
        s.spawn(|| {
            while let Some(i) = queue.pop() {
                println!("    consumed {i}");
                thread::sleep(Duration::from_millis(50)); // a slow consumer
            }
            println!("    queue closed, consumer stops");
        });
    });
}
```

```text
produced 1
produced 2
produced 3
    consumed 1
    consumed 2
produced 4
    consumed 3
produced 5
    consumed 4
    consumed 5
    queue closed, consumer stops
```

**What happens:**

1. All the data (`items` and `closed`) is inside **one** mutex (rule 1). Both condvars are used with that same mutex.
2. `push` waits on `not_full` while the queue is full and not closed. When it may go on, it adds the item and calls `not_empty.notify_one()` to wake one sleeping consumer (rule 2: the state changed under the lock).
3. `pop` waits on `not_empty` while the queue is empty and not closed. After it took an item, there is free space, so it wakes one producer with `not_full.notify_one()`.
4. The capacity is 2 and the consumer is slow, so the producer must wait for free space before items 4 and 5. That waiting is **backpressure** (Lesson 06.11): the fast side is slowed down to the speed of the slow side, and memory use stays limited.
5. `close` sets the flag and wakes **everybody** with `notify_all`: every waiting consumer must see that the queue is closed, not just one. `pop` still returns the remaining items and gives `None` only when the queue is closed **and** empty, so nothing is lost.
6. The lines are printed *after* `push` and `pop` return, so the order of the printed lines can lag a little behind the real order (here the consumer had already taken item 1 when "produced 3" was printed). Your output may look slightly different.

> 💡 **Tip:** `notify_one` is enough when any single waiter can make progress (one new item, one waiting consumer). Use `notify_all` when the change matters to everybody (closing, shutting down) or when different waiters wait for different conditions on the same condvar.

> ⚡ **Performance:** You can call `notify_one` while holding the lock (as here) or right after dropping the guard. Notifying after the unlock can save the woken thread a moment of waiting for the lock. Both are correct; keep the simpler form until a measurement says otherwise.

## 5. Waiting with a time limit

`wait_timeout(guard, duration)` returns `(guard, WaitTimeoutResult)`. `result.timed_out()` says whether the time ran out. As with `wait`, the thread may wake early (spurious wake-up), so a loop is needed — and the loop must also remember how much time is left. `wait_timeout_while(guard, duration, condition)` does all of that for you:

```rust
    // A fragment: inside a method of the queue from section 4.
    let state = self.state.lock().unwrap();
    let (state, result) = self
        .not_empty
        .wait_timeout_while(state, Duration::from_millis(50), |s| s.items.is_empty())
        .unwrap();
    if result.timed_out() { /* nothing arrived in 50 ms */ }
```

Timeouts are how you avoid waiting forever for a thread that crashed or a producer that never comes. In Exercise 2 you will add `pop_timeout` to the queue.

> 🧪 **Testing:** Do not test a blocking structure with a single thread and `sleep`s. Start several producers and consumers, close the queue when the producers are done, collect everything the consumers received, sort it, and compare it with what was sent: every item must arrive **exactly once**. Exercise 1 is that test.

> 📏 **Convention:** Before you write your own condvar code, check whether the standard library already has the tool: `mpsc::sync_channel` is a bounded blocking queue, `std::sync::Barrier` lets N threads wait for each other, `OnceLock` waits for a value to be set once. Write condvar code when none of them fits.

## Common mistakes

1. **`if` instead of `while` around `wait`.** A spurious wake-up or another consumer can make the condition false again. Use `while` or `wait_while`.
2. **A condvar without a condition.** Notifications are not stored. If you notify before the other thread waits, it is lost (section 3). The state must live in the mutex.
3. **Changing the state without the lock.** Rule 2. The waiter can check, see the old value, and sleep just after your notification.
4. **Using two different mutexes with one condvar.** All threads that wait on one condvar must use the same mutex. The documentation says that `wait` may panic if a condvar is used with more than one mutex.
5. **`let _ = m.lock().unwrap();` to "hold" a lock.** The pattern `_` does not bind the guard, so it is dropped at once. Rust refuses to compile it. In `src/bin/let_underscore.rs`:

   ```rust
   use std::sync::Mutex;

   fn main() {
       let m = Mutex::new(0);
       let _ = m.lock().unwrap(); // meant: "hold the lock for this block"
       println!("is the lock held here?");
   }
   ```

   ```text
   error: non-binding let on a synchronization lock
    --> src/bin/let_underscore.rs:5:9
     |
   5 |     let _ = m.lock().unwrap(); // meant: "hold the lock for this block"
     |         ^ this lock is not assigned to a binding and is immediately dropped
   ```

   Give it a name that starts with an underscore, `let _guard = …`, which keeps it until the end of the block.

## Try it yourself

1. In `busy_wait.rs`, add `thread::sleep(Duration::from_millis(10));` inside the `while` loop. How many checks are left now? (On the test machine: `ready after 9 checks`.) What is the cost? Think about how late the main thread can notice the change.
2. In `blocking_queue.rs`, change the capacity from 2 to 1, then to 10. How does the order of "produced" and "consumed" lines change? With capacity 10, the producer never has to wait: all five "produced" lines come first.
3. In `ready_flag.rs`, remove the line `cvar.notify_one();` (the compiler now warns that `cvar` is unused in the worker). Run the program: it prints `worker: config loaded` and then hangs. Why? Stop it with `Ctrl+C`, then put the line back.

## Exercises

### Exercise 1 — Every item exactly once (medium)

Add a test `every_item_arrives_exactly_once` to `blocking_queue.rs`. Use a queue of capacity 8, 4 producers (producer `p` pushes the numbers `p * 1000 + i` for `i` in `0..1000`) and 4 consumers that pop until `None` and return what they got. After all producers have finished, close the queue. The sorted list of received numbers must be exactly `0..4000`. Add a second, small test: after `close`, `push(7)` returns `Err(7)` and `pop()` returns `None`.

<details>
<summary>Hint</summary>

Use `thread::scope`. Keep the `ScopedJoinHandle`s of the producers and the consumers in two vectors. Join the producers, call `queue.close()`, then join the consumers and `extend` one big vector with their results. Inside the producer closure you need `move` (for `p`), so first make a reference `let queue = &queue;` and move that.

</details>

<details>
<summary>Answer</summary>

Add at the end of `src/bin/blocking_queue.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn every_item_arrives_exactly_once() {
        let queue = BlockingQueue::new(8);
        let mut received = Vec::new();
        thread::scope(|s| {
            let producers: Vec<_> = (0..4)
                .map(|p| {
                    let queue = &queue;
                    s.spawn(move || {
                        for i in 0..1000 {
                            queue.push(p * 1000 + i).unwrap();
                        }
                    })
                })
                .collect();
            let consumers: Vec<_> = (0..4)
                .map(|_| {
                    s.spawn(|| {
                        let mut got = Vec::new();
                        while let Some(x) = queue.pop() {
                            got.push(x);
                        }
                        got
                    })
                })
                .collect();
            for p in producers {
                p.join().unwrap();
            }
            queue.close(); // all producers are done
            for c in consumers {
                received.extend(c.join().unwrap());
            }
        });
        received.sort();
        let expected: Vec<i32> = (0..4000).collect();
        assert_eq!(received, expected);
    }

    #[test]
    fn push_after_close_gives_the_item_back() {
        let queue = BlockingQueue::new(1);
        queue.close();
        assert_eq!(queue.push(7), Err(7));
        assert_eq!(queue.pop(), None);
    }
}
```

```text
running 2 tests
test tests::push_after_close_gives_the_item_back ... ok
test tests::every_item_arrives_exactly_once ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Why it works: if an item were lost, a number would be missing; if one were delivered twice, a number would appear twice. Both would make the sorted list differ from `0..4000`. Closing only after all producers joined guarantees that consumers stop only when everything was sent. If `close` did not wake all consumers, the test would hang instead of passing.

</details>

### Exercise 2 — `pop_timeout` (medium)

Add `pub fn pop_timeout(&self, timeout: Duration) -> Option<T>` to `BlockingQueue`: like `pop`, but it returns `None` if no item arrives within `timeout`. Write two tests: on an empty queue, `pop_timeout(50 ms)` returns `None` and takes at least 50 ms; and an item pushed by another thread after 10 ms is returned by `pop_timeout(1 s)`.

<details>
<summary>Hint</summary>

Copy `pop` and replace `wait_while(state, …)` with `wait_timeout_while(state, timeout, …)`. It returns a tuple `(guard, result)`. You do not even need `result`: after the wait, if the queue is still empty, `pop_front()` returns `None`. Measure time in the test with `std::time::Instant`.

</details>

<details>
<summary>Answer</summary>

The method, inside `impl<T> BlockingQueue<T>`:

```rust
    /// Like `pop`, but gives up after `timeout`.
    pub fn pop_timeout(&self, timeout: Duration) -> Option<T> {
        let state = self.state.lock().unwrap();
        let (mut state, _result) = self
            .not_empty
            .wait_timeout_while(state, timeout, |s| s.items.is_empty() && !s.closed)
            .unwrap();
        let item = state.items.pop_front();
        if item.is_some() {
            self.not_full.notify_one();
        }
        item
    }
```

The tests, inside `mod tests` (add `use std::time::Instant;` there):

```rust
    #[test]
    fn pop_timeout_gives_up_on_an_empty_queue() {
        let queue: BlockingQueue<i32> = BlockingQueue::new(1);
        let start = Instant::now();
        assert_eq!(queue.pop_timeout(Duration::from_millis(50)), None);
        assert!(start.elapsed() >= Duration::from_millis(50));
    }

    #[test]
    fn pop_timeout_returns_an_item_that_arrives_in_time() {
        let queue = BlockingQueue::new(1);
        thread::scope(|s| {
            s.spawn(|| {
                thread::sleep(Duration::from_millis(10));
                queue.push(42).unwrap();
            });
            assert_eq!(queue.pop_timeout(Duration::from_secs(1)), Some(42));
        });
    }
```

```text
running 4 tests
test tests::push_after_close_gives_the_item_back ... ok
test tests::every_item_arrives_exactly_once ... ok
test tests::pop_timeout_returns_an_item_that_arrives_in_time ... ok
test tests::pop_timeout_gives_up_on_an_empty_queue ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.06s
```

Why it works: `wait_timeout_while` keeps waiting while the queue is empty, but never longer than `timeout` in total, even after spurious wake-ups. When it returns, `pop_front` tells the truth: an item or `None`. The second test uses a long timeout (1 s) on purpose, so a slow machine does not make the test flaky; it still finishes after about 10 ms.

</details>

### Exercise 3 — A countdown latch (hard)

Write `src/bin/latch.rs` with a `Latch` type: `new(n)`, `count_down(&self)` (decreases the counter, never below zero) and `wait(&self)` (blocks until the counter is zero). In `main`, three workers print "worker N is ready" and count down; the main thread waits and then prints "all workers are ready, main goes on". Write a test that 4 threads increment an `AtomicUsize` and count down, and after `wait()` the atomic is 4; and a test that `Latch::new(0).wait()` returns at once.

<details>
<summary>Hint</summary>

The state is a `Mutex<usize>` plus one `Condvar`. Who must be woken when the counter reaches zero: one thread or all? In `wait`, use `wait_while` and keep the returned guard in a variable named `_guard` (see common mistake 5).

</details>

<details>
<summary>Answer</summary>

```rust
use std::sync::{Condvar, Mutex};
use std::thread;

/// Lets threads wait until `count_down` was called `n` times.
struct Latch {
    remaining: Mutex<usize>,
    zero: Condvar,
}

impl Latch {
    fn new(n: usize) -> Self {
        Latch {
            remaining: Mutex::new(n),
            zero: Condvar::new(),
        }
    }

    fn count_down(&self) {
        let mut remaining = self.remaining.lock().unwrap();
        if *remaining > 0 {
            *remaining -= 1;
            if *remaining == 0 {
                self.zero.notify_all();
            }
        }
    }

    fn wait(&self) {
        let remaining = self.remaining.lock().unwrap();
        let _guard = self.zero.wait_while(remaining, |r| *r > 0).unwrap();
    }
}

fn main() {
    let latch = Latch::new(3);
    thread::scope(|s| {
        for id in 0..3 {
            let latch = &latch;
            s.spawn(move || {
                println!("worker {id} is ready");
                latch.count_down();
            });
        }
        latch.wait();
        println!("all workers are ready, main goes on");
    });
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::atomic::{AtomicUsize, Ordering};

    #[test]
    fn wait_returns_only_after_all_count_downs() {
        let latch = Latch::new(4);
        let done = AtomicUsize::new(0);
        thread::scope(|s| {
            for _ in 0..4 {
                s.spawn(|| {
                    done.fetch_add(1, Ordering::Relaxed);
                    latch.count_down();
                });
            }
            latch.wait();
            assert_eq!(done.load(Ordering::Relaxed), 4);
        });
    }

    #[test]
    fn a_latch_of_zero_does_not_block() {
        Latch::new(0).wait();
    }
}
```

```text
worker 0 is ready
worker 2 is ready
worker 1 is ready
all workers are ready, main goes on
```

(The order of the worker lines changes from run to run; the last line is always last.) `cargo test --bin latch` shows both tests `ok`.

Why it works: the counter is the condition, and it lives in the mutex (rule 1). `count_down` changes it under the lock and notifies (rule 2). It uses `notify_all`, because *every* waiting thread may go on once the counter is zero. `wait_while` checks the condition first, so `Latch::new(0).wait()` never sleeps, and a late `wait()` after the last `count_down` does not miss the notification (rule 3). The atomic counter in the test is read after `wait()`; the mutex inside the latch makes sure the main thread sees all four increments (Lesson 14.6 explains why).

</details>

## Quiz

1. What is wrong with waiting for a flag by locking and checking it in a loop?

<details>
<summary>Answer</summary>

It is busy waiting: one core runs at 100% doing nothing useful, millions of times per second, and the loop also competes for the lock with the thread that should set the flag. Adding a sleep saves CPU but makes the reaction late.

</details>

2. What does `cvar.wait(guard)` do, step by step?

<details>
<summary>Answer</summary>

It takes the guard, unlocks the mutex and puts the thread to sleep, all as one step. When the thread is woken (by a notify or spuriously), it locks the mutex again and returns the guard.

</details>

3. Why must the waiting thread check the condition in a loop?

<details>
<summary>Answer</summary>

Because of spurious wake-ups (`wait` may return without a notify) and because another thread may have changed the state again between the notify and the moment this thread got the lock. After every wake-up the condition must be checked again.

</details>

4. A thread calls `notify_one()` before any thread has called `wait()`. What happens to the notification?

<details>
<summary>Answer</summary>

It is lost: a condition variable does not remember notifications. This is harmless only if the state is stored in the mutex and the waiter checks it before waiting, so it sees that the condition is already true and does not sleep.

</details>

5. Why does the blocking queue use two condition variables?

<details>
<summary>Answer</summary>

There are two different reasons to wait: consumers wait for "not empty" and producers wait for "not full". With separate condvars, `push` can wake exactly a consumer and `pop` exactly a producer. With one condvar, `notify_one` might wake a thread of the wrong kind, which then goes back to sleep while the right thread keeps sleeping.

</details>

## Summary

- Do not wait by checking in a loop. A condition variable lets a thread sleep until another thread changes the state.
- `wait` unlocks the mutex and sleeps in one step, and locks again before it returns. `notify_one` wakes one waiter; `notify_all` wakes all of them.
- Three rules: the condition lives in the mutex; change it under the lock, then notify; always wait in a `while` loop or with `wait_while`.
- Notifications are not stored (lost wake-up), and `wait` may return without one (spurious wake-up). The rules handle both.
- A bounded blocking queue uses one mutex and two condvars (`not_empty`, `not_full`). Its capacity creates backpressure; a `closed` flag with `notify_all` stops everybody cleanly.
- `wait_timeout_while` waits with a time limit and handles spurious wake-ups and the remaining time for you.

## Sources and further reading

- [std::sync::Condvar](https://doc.rust-lang.org/std/sync/struct.Condvar.html) — `wait`, `wait_while`, `wait_timeout_while`, `notify_one`, `notify_all`, with the official example.
- [std::sync::Barrier](https://doc.rust-lang.org/std/sync/struct.Barrier.html) — the ready-made tool when N threads must wait for each other.
- [Rust Atomics and Locks, chapter 1 — Basics of Rust Concurrency](https://marabos.nl/atomics/basics.html) — Mara Bos explains thread parking and condition variables with a queue example.
- [Operating Systems: Three Easy Pieces — Condition Variables](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-cv.pdf) — a free textbook chapter on producers, consumers, and why you need a `while` loop.

---

[🏠 Start](../START-HERE.md) · [← 14.2 Mutexes and read-write locks in depth](02-mutexes-and-rwlocks-in-depth.md) · [📋 Phase 14](README.md) · [14.4 Thread pools, scoped threads and rayon →](04-thread-pools-scoped-threads-and-rayon.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
