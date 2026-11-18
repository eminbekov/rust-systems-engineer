<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.10 How futures work: Future, poll and Waker](10-how-futures-work.md) · [📋 Phase 14](README.md) · [14.12 Pin and self-referential futures →](12-pin-and-self-referential-futures.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/11-build-a-tiny-async-runtime.md) <!-- nav -->

# Lesson 14.11 — Build a tiny async runtime

> **Phase 14 — Concurrency and async in depth** · Lesson 11 of 16 · ⏱ about 135 minutes

## What you will learn

- What the parts of an async runtime are: tasks, a run queue, wakers, timers and a loop that ties them together.
- How a `Waker` built with `std::task::Wake` puts its task back into the queue.
- How to write a `sleep` future that needs no extra thread, because the executor keeps all timers in one list.
- How the executor sleeps until the next timer or the next wake, and never spins.
- What Tokio adds on top of this design, and why.

## Before you start

- You finished [Lesson 14.10 — How futures work: Future, poll and Waker](10-how-futures-work.md). This lesson reuses its ideas: the waker contract, `Wake`, `park` and `unpark`.
- You know `Box<dyn Trait>` from [Lesson 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md), `RefCell` from [Lesson 06.4](../06-rust-advanced/04-interior-mutability.md), and a queue (`VecDeque`) from Phase 07.
- You know what epoll does from [Lesson 13.13 — Non-blocking I/O and epoll](../13-operating-systems/13-non-blocking-io-and-epoll.md). We only mention it here.
- Your code for this lesson goes in `~/projects/rust/code/p14-tiny-runtime`, a library with small programs in `src/bin/`, as in Lesson 14.10.

## New words

| Word | Meaning |
|---|---|
| **task** | A future that the runtime owns and runs on its own, started with `spawn`. Each task has its own waker. |
| **run queue** | The list of tasks that were woken and are waiting to be polled. The executor takes them from the front. |
| **executor** | The part of the runtime that takes tasks from the run queue and polls them. |
| **timer** | A note "wake this waker at this time". A `sleep` future creates one. |
| **deadline** | The moment (`Instant`) when a timer is due. |
| **reactor** | The part of a runtime that waits for I/O (input/output) events from the operating system (epoll on Linux, kqueue on macOS) and calls the wakers of the tasks that wait for them. Our tiny runtime has timers but no reactor. |
| **thread-local** | A variable with a separate copy in each thread, made with the `thread_local!` macro. |
| **cooperative scheduling** | Tasks give control back by themselves, at `.await` points. The runtime cannot stop a task that never awaits. |
| **starvation** | A task never gets to run, because other tasks keep the thread busy. |

## 1. The plan

A runtime is a loop around a queue. Here is everything our runtime will have, and how a task moves through it:

```text
            spawn(future)
                  |
                  v
   +-------------------------------+   pop_front   +-------------------+
   | run queue: VecDeque<Arc<Task>>| ------------> | poll the task     |
   +-------------------------------+               +-------------------+
          ^             ^                             |            |
          |             |                        Ready |            | Pending
   waker.wake()    waker.wake()                        v            v
          |             |                        drop the task   the future keeps
   +-------------+  +-----------------+                           its waker somewhere:
   | timers list |  | another thread  |                           a timer, a channel,
   | (deadline,  |  | (like Delay in  |                           another thread
   |  waker)     |  |  Lesson 14.10)  |
   +-------------+  +-----------------+

   When the queue is empty: park the thread until the nearest deadline or until a wake.
```

The rules:

1. `spawn` wraps a future in a `Task` and pushes it into the run queue.
2. The executor pops a task and polls it once.
3. If the task is `Ready`, it is finished and dropped. If it is `Pending`, it has given its waker to someone.
4. When that someone calls `wake`, the task is pushed into the queue again, and the executor thread is unparked.
5. When there is nothing to poll, the executor fires due timers, and then sleeps until the nearest deadline (or until a wake from another thread).

Create the library:

```bash
cd ~/projects/rust
cargo new --lib code/p14-tiny-runtime
cd code/p14-tiny-runtime
mkdir src/bin
```

No dependencies: everything comes from the standard library. All the code of Sections 2 to 4 goes into `src/lib.rs`, one part after another.

## 2. Tasks and their wakers

The first part of `src/lib.rs`:

```rust
//! A tiny single-threaded async runtime: a task queue, wakers and timers.

use std::cell::RefCell;
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Wake, Waker};
use std::thread::{self, Thread};
use std::time::{Duration, Instant};

/// A future of any type, on the heap, that can move between threads.
type BoxFuture = Pin<Box<dyn Future<Output = ()> + Send>>;

/// Everything that the executor, the tasks and the wakers share.
struct Shared {
    /// Tasks that were woken and wait to be polled.
    queue: Mutex<VecDeque<Arc<Task>>>,
    /// Sleeping tasks: when to wake them, and how.
    timers: Mutex<Vec<(Instant, Waker)>>,
    /// The thread that runs the executor, so a waker can unpark it.
    thread: Thread,
}

/// One spawned future and the way back into the queue.
struct Task {
    future: Mutex<Option<BoxFuture>>,
    shared: Arc<Shared>,
}

impl Wake for Task {
    fn wake(self: Arc<Self>) {
        let shared = Arc::clone(&self.shared);
        shared.queue.lock().unwrap().push_back(self);
        shared.thread.unpark();
    }
}

thread_local! {
    /// The runtime of this thread, while `block_on` runs.
    static CURRENT: RefCell<Option<Arc<Shared>>> = const { RefCell::new(None) };
}

fn current() -> Arc<Shared> {
    CURRENT
        .with(|current| current.borrow().clone())
        .expect("must be called inside block_on")
}

/// Starts a new task. It runs when the executor gets to it.
pub fn spawn<F>(future: F)
where
    F: Future<Output = ()> + Send + 'static,
{
    let shared = current();
    let task = Arc::new(Task {
        future: Mutex::new(Some(Box::pin(future))),
        shared: Arc::clone(&shared),
    });
    shared.queue.lock().unwrap().push_back(task);
}
```

**What happens:**

1. `BoxFuture` is one type for all tasks. Every `async` block has its own, unnamed type, so the queue stores them as `Box<dyn Future>`. `Pin<Box<…>>` (from `Box::pin`) gives the pinned future that `poll` needs; Lesson 14.12 explains pinning. `Send` is needed because a `Waker` may be sent to another thread, and the waker holds the task.
2. `Shared` is the heart of the runtime. All its fields are behind a `Mutex`, because wakers can be called from other threads.
3. `Task` holds its future in `Mutex<Option<…>>`. `Option` because the future is taken out while it is polled, and is gone after it finishes.
4. `impl Wake for Task` is the waker: "push me into the queue again and unpark the executor thread". This is the same trick as `ThreadWaker` in Lesson 14.10, plus the queue.
5. `CURRENT` is a thread-local: `spawn` and `sleep` must find the runtime without getting it as an argument. `block_on` sets it, and `current()` reads it. Tokio uses the same idea, which is why `tokio::spawn` panics outside a runtime.
6. `spawn` builds a task and pushes it into the queue. It does not poll it: the task runs when the executor reaches it.

## 3. Two futures to await: `sleep` and `yield_now`

The second part of `src/lib.rs`:

```rust
/// A future that is ready once `deadline` has passed.
pub struct Sleep {
    deadline: Instant,
}

/// Waits for `duration` without blocking the thread.
pub fn sleep(duration: Duration) -> Sleep {
    Sleep {
        deadline: Instant::now() + duration,
    }
}

impl Future for Sleep {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if Instant::now() >= self.deadline {
            return Poll::Ready(());
        }
        let shared = current();
        shared
            .timers
            .lock()
            .unwrap()
            .push((self.deadline, cx.waker().clone()));
        Poll::Pending
    }
}

/// Lets other tasks run once, then continues.
pub fn yield_now() -> YieldNow {
    YieldNow { yielded: false }
}

pub struct YieldNow {
    yielded: bool,
}

impl Future for YieldNow {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.yielded {
            Poll::Ready(())
        } else {
            self.yielded = true;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

**What happens:**

1. `sleep` only computes the deadline. Remember: futures are lazy, nothing is registered yet.
2. When `Sleep` is polled before its deadline, it adds `(deadline, waker)` to the runtime's timer list and returns `Pending`. The contract from Lesson 14.10 is kept: the executor will call this waker when the time comes.
3. Compare with `Delay` from Lesson 14.10: no helper thread. A thousand sleeping tasks cost a thousand small entries in a `Vec`, not a thousand threads.
4. If `Sleep` is polled twice before its deadline, it is added twice. A second wake only causes one extra, harmless poll. (Tokio stores each timer only once; that needs more code.)
5. `yield_now` is `YieldOnce` from Lesson 14.10: it wakes itself and returns `Pending` once. Its task goes to the **back** of the queue, so every other ready task runs first.

## 4. The executor loop

The last part of `src/lib.rs`, the loop that makes everything move:

```rust
/// Polls one task once. A finished task is dropped.
fn run_task(task: Arc<Task>) {
    let mut slot = task.future.lock().unwrap();
    if let Some(mut future) = slot.take() {
        let waker = Waker::from(Arc::clone(&task));
        let mut cx = Context::from_waker(&waker);
        if future.as_mut().poll(&mut cx).is_pending() {
            *slot = Some(future);
        }
    }
}

/// Wakes every timer whose deadline has passed.
/// Returns the nearest deadline that is still in the future.
fn fire_timers(shared: &Shared) -> Option<Instant> {
    let now = Instant::now();
    let mut due = Vec::new();
    let next = {
        let mut timers = shared.timers.lock().unwrap();
        timers.retain(|(deadline, waker)| {
            if *deadline <= now {
                due.push(waker.clone());
                false
            } else {
                true
            }
        });
        timers.iter().map(|(deadline, _)| *deadline).min()
    };
    for waker in due {
        waker.wake();
    }
    next
}

/// Runs `future` and every task it spawns, until `future` is finished.
pub fn block_on<F>(future: F) -> F::Output
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
{
    let shared = Arc::new(Shared {
        queue: Mutex::new(VecDeque::new()),
        timers: Mutex::new(Vec::new()),
        thread: thread::current(),
    });
    CURRENT.with(|current| *current.borrow_mut() = Some(Arc::clone(&shared)));

    // The main future is a task too; it puts its result into `result`.
    let result = Arc::new(Mutex::new(None));
    let slot = Arc::clone(&result);
    spawn(async move {
        let value = future.await;
        *slot.lock().unwrap() = Some(value);
    });

    loop {
        // 1. Poll every task that is ready to run.
        loop {
            let next_task = shared.queue.lock().unwrap().pop_front();
            match next_task {
                Some(task) => run_task(task),
                None => break,
            }
        }
        // 2. Stop when the main future is finished.
        if let Some(value) = result.lock().unwrap().take() {
            shared.queue.lock().unwrap().clear();
            shared.timers.lock().unwrap().clear();
            CURRENT.with(|current| *current.borrow_mut() = None);
            return value;
        }
        // 3. Wake the timers that are due.
        let next_deadline = fire_timers(&shared);
        // 4. Nothing to do? Sleep until a waker or the next timer.
        if shared.queue.lock().unwrap().is_empty() {
            match next_deadline {
                Some(deadline) => {
                    thread::park_timeout(deadline.saturating_duration_since(Instant::now()))
                }
                None => thread::park(),
            }
        }
    }
}
```

**What happens:**

1. `run_task` takes the future out of the task, makes a waker from the task itself (`Waker::from(Arc<Task>)` works because `Task` implements `Wake`), and polls once. A `Pending` future is put back; a `Ready` one is simply not put back, so it is dropped.
2. `fire_timers` removes every due timer from the list and wakes it, which pushes its task into the queue. It returns the nearest remaining deadline. The wakers are called *after* the `timers` lock is released, so a woken task can never wait for a lock that we still hold.
3. `block_on` creates `Shared`, stores it in `CURRENT`, and spawns the main future as an ordinary task. The task writes the result into `result` when it finishes.
4. Step 1 of the loop: the inner `loop` pops one task at a time. The line `let next_task = …pop_front();` ends with `;`, so the queue's lock guard is dropped **before** the task is polled. This matters: a task that wakes itself or spawns a new task locks the queue again (Common mistakes shows what happens otherwise).
5. Step 2: when the main future has finished, the runtime clears the queue and the timers, and returns the value. Like in Tokio, tasks that are still running are dropped, which cancels them. Clearing also breaks the reference cycle "`Shared` holds tasks, tasks hold `Shared`", so the memory is freed.
6. Steps 3 and 4: fire due timers. If the queue is still empty, park the thread: with `park_timeout` until the nearest deadline, or with `park` until some waker calls `unpark` (another thread, as in Exercise 2). The executor never spins.

## 5. Running it

**Two tasks take turns.** Create `src/bin/two_tasks.rs`:

```rust
use p14_tiny_runtime::{block_on, spawn, yield_now};

async fn worker(name: &'static str) {
    for step in 1..=3 {
        println!("{name}: step {step}");
        yield_now().await;
    }
}

fn main() {
    block_on(async {
        spawn(worker("A"));
        spawn(worker("B"));
        yield_now().await;
        println!("main: spawned two tasks");
        for _ in 0..3 {
            yield_now().await;
        }
        println!("main: done");
    });
}
```

`cargo run --bin two_tasks`:

```text
A: step 1
B: step 1
main: spawned two tasks
A: step 2
B: step 2
A: step 3
B: step 3
main: done
```

**What happens:**

1. The main task spawns A and B (queue: A, B) and yields, so it goes to the back (queue: A, B, main).
2. A prints step 1 and yields (to the back), B does the same, then main prints its line.
3. They keep taking turns in queue order. One thread, three tasks, and each `.await` on `yield_now` is a place where another task can run. This is **cooperative scheduling**.

**Timers.** Create `src/bin/timers.rs`:

```rust
use p14_tiny_runtime::{block_on, sleep, spawn};
use std::time::{Duration, Instant};

fn main() {
    let start = Instant::now();
    block_on(async move {
        for ms in [300, 100, 200] {
            spawn(async move {
                sleep(Duration::from_millis(ms)).await;
                println!(
                    "{:>4} ms: task that slept {ms} ms",
                    start.elapsed().as_millis()
                );
            });
        }
        sleep(Duration::from_millis(400)).await;
        println!("{:>4} ms: main is done", start.elapsed().as_millis());
    });
}
```

`cargo run --bin timers`:

```text
 105 ms: task that slept 100 ms
 204 ms: task that slept 200 ms
 304 ms: task that slept 300 ms
 405 ms: main is done
```

The tasks were spawned in the order 300, 100, 200, but they finish in deadline order, and the whole program takes 400 ms, not 300 + 100 + 200 + 400 = 1000 ms. All four waits happen at the same time on **one** thread, and between the deadlines the thread sleeps in `park_timeout`.

> 🧪 **Testing:** `block_on` makes async code testable with a plain `#[test]`. A test such as `assert_eq!(block_on(async { 6 * 7 }), 42);` checks the whole path: spawn of the main task, poll, result. Exercise 1 adds a test with a thousand tasks.

## 6. What a real runtime adds

Our runtime has about 200 lines and the same skeleton as Tokio. Here is what Tokio adds, and why:

| Our runtime | Tokio | Why |
|---|---|---|
| one thread | a pool of worker threads, each with its own queue; idle workers *steal* tasks from busy ones | use all CPU (Central Processing Unit) cores |
| `Vec` of timers, scanned on every loop | a *timer wheel*: timers sorted into buckets by time | millions of timers stay cheap |
| no I/O | a reactor: epoll (Linux) or kqueue (macOS) waits for sockets, and calls the wakers | network servers |
| a task may be queued twice | a state flag per task: woken tasks are queued only once | less useless polling |
| `spawn` returns nothing | `JoinHandle` with the output, `abort()` to cancel | results and control (Exercise 3 builds a small `JoinHandle`) |
| a task that never awaits blocks everything | a budget: after many operations, Tokio's own futures return `Pending` on purpose | fairness between tasks |

The last row is important for every async program, not only for runtimes. Cooperative scheduling means that a task that computes for a long time **without an `.await`** keeps the thread. In our runtime, every other task waits until it finishes; the runtime has no way to stop it. That is the reason for the rule from Lesson 06.9: never block the runtime; move long computations to `spawn_blocking` or a thread pool.

> ⚡ **Performance:** The design "one queue, a sleep until the next event" is why async is cheap for waiting: a task that waits costs only its future in memory and an entry in a list, and a thread that has nothing to do costs no CPU at all.

## Common mistakes

**1. Holding the queue lock while a task is polled.** It is tempting to write the inner loop like this:

```rust
        while let Some(task) = shared.queue.lock().unwrap().pop_front() {
            run_task(task);
        }
```

The program prints nothing and hangs forever. The temporary lock guard in a `while let` condition lives until the end of the loop body. When the main task calls `spawn`, `spawn` locks the queue again on the same thread; a `std::sync::Mutex` cannot be locked twice by one thread, so it waits for itself. Fix: take the task out in a separate statement that ends with `;`, as in Section 4.

**2. Forgetting `unpark` in `Task::wake`.** Wakes from the executor's own thread still work, because the executor checks the queue before it parks. But a wake from another thread (Exercise 2) pushes the task while the executor sleeps in `park()`, and nobody wakes the executor: the program hangs.

**3. Calling `spawn` or `sleep` outside `block_on`.** There is no runtime in `CURRENT`:

```text
thread 'main' (5574875) panicked at src/lib.rs:47:10:
must be called inside block_on
```

Tokio has the same rule and a similar panic message.

**4. Calling the wakers while holding the timers lock.** `waker.wake()` may run code that locks other things, or, in a multi-threaded runtime, the timers lock itself. Collect the due wakers first, release the lock, then wake them, as `fire_timers` does.

## Try it yourself

1. Write the three parts of `src/lib.rs` and the two programs, and run them.
2. In `two_tasks.rs`, remove the `yield_now().await;` lines inside `worker`. Predict the order of the lines, then run. (Each worker now prints all three steps in one poll: A 1–3, then B 1–3, then both `main` lines.)
3. Add `println!("tick")` at the start of the outer `loop` in `block_on`, run `timers`, and count the ticks. Each tick is one wake-up of the executor. Remove the line again.
4. Add the tests from the 🧪 note and from Exercise 1, run `cargo test`, `cargo clippy --all-targets` and `cargo fmt`.
5. Commit: `git add code/p14-tiny-runtime && git commit -m "p14-tiny-runtime: executor, wakers and timers"`.

## Exercises

### Exercise 1 — A thousand tasks (easy)

Write a test: the main future spawns 1000 tasks; each task yields once and then increments a shared `AtomicUsize`. After `block_on` returns, the counter must be 1000. Then write a second test: `sleep` of 50 ms takes at least 50 ms.

<details>
<summary>Hint</summary>

Share the counter with `Arc`. The main future must not finish before the tasks: let it `yield_now().await` twice, so every task gets two turns (one for its own `yield_now`, one to finish).

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::atomic::{AtomicUsize, Ordering};

    #[test]
    fn block_on_returns_the_value() {
        assert_eq!(block_on(async { 6 * 7 }), 42);
    }

    #[test]
    fn a_thousand_tasks_all_run() {
        let count = Arc::new(AtomicUsize::new(0));
        let for_tasks = Arc::clone(&count);
        block_on(async move {
            for _ in 0..1000 {
                let count = Arc::clone(&for_tasks);
                spawn(async move {
                    yield_now().await;
                    count.fetch_add(1, Ordering::SeqCst);
                });
            }
            // Give every task the chance to run twice.
            yield_now().await;
            yield_now().await;
        });
        assert_eq!(count.load(Ordering::SeqCst), 1000);
    }

    #[test]
    fn sleep_waits_at_least_its_duration() {
        let start = Instant::now();
        block_on(sleep(Duration::from_millis(50)));
        assert!(start.elapsed() >= Duration::from_millis(50));
    }
}
```

`cargo test` shows `test result: ok. 3 passed; 0 failed` for the library. Why it works: tasks run in queue order. The main task is behind all 1000 tasks after its first yield, and behind all of them again after its second, so every task has finished when main finishes. Each test runs on its own thread, and `CURRENT` is thread-local, so the tests do not share a runtime.

</details>

### Exercise 2 — A wake from another thread (medium)

Copy `Shared`, `Delay` and its `impl Future` from Lesson 14.10 into `src/bin/delay.rs` and run `block_on(Delay::new(Duration::from_millis(150)))` with *your* runtime. Print the time. Then comment out `shared.thread.unpark();` in `Task::wake` and run again. Explain the difference.

<details>
<summary>Hint</summary>

`Delay`'s helper thread calls `waker.wake()`. Which code runs then, and on which thread? Where is the executor thread at that moment?

</details>

<details>
<summary>Answer</summary>

The `main` of `src/bin/delay.rs` (the copied `Delay` code goes above it):

```rust
fn main() {
    let start = Instant::now();
    block_on(Delay::new(Duration::from_millis(150)));
    println!(
        "woken from another thread after {} ms",
        start.elapsed().as_millis()
    );
}
```

```text
woken from another thread after 150 ms
```

Without `unpark()` the program prints nothing and never ends (stop it with Ctrl+C). Why: the helper thread runs `Task::wake`, which pushes the task into the queue. But the executor thread is in `thread::park()`, because the queue was empty and there was no timer. Only `unpark` ends that sleep. With `unpark` it wakes, finds the task, polls it, and `block_on` returns.

</details>

### Exercise 3 — `JoinHandle` (hard)

Write `spawn_with_handle(future) -> JoinHandle<T>` on top of your `spawn` (in a program file, without changing the library). The `JoinHandle<T>` is a future that is ready with the task's output. Test it: spawn `slow_square(2)` and `slow_square(3)`, where `slow_square(n)` sleeps `100 * n` ms and returns `n * n`, and print the sum of the two results and the time. It must be 13 after about 300 ms.

<details>
<summary>Hint</summary>

The task and the handle share `Arc<Mutex<JoinState<T>>>` with `result: Option<T>` and `waker: Option<Waker>`. The task stores the result and wakes the stored waker. The handle's `poll` takes the result if it is there; otherwise it stores its waker and returns `Pending` — the same pattern as `Delay`.

</details>

<details>
<summary>Answer</summary>

```rust
use p14_tiny_runtime::{block_on, sleep, spawn};
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::time::{Duration, Instant};

struct JoinState<T> {
    result: Option<T>,
    waker: Option<Waker>,
}

/// A future that gives the output of a spawned task.
struct JoinHandle<T> {
    state: Arc<Mutex<JoinState<T>>>,
}

fn spawn_with_handle<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
{
    let state = Arc::new(Mutex::new(JoinState {
        result: None,
        waker: None,
    }));
    let for_task = Arc::clone(&state);
    spawn(async move {
        let value = future.await;
        let mut state = for_task.lock().unwrap();
        state.result = Some(value);
        if let Some(waker) = state.waker.take() {
            waker.wake();
        }
    });
    JoinHandle { state }
}

impl<T> Future for JoinHandle<T> {
    type Output = T;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        let mut state = self.state.lock().unwrap();
        match state.result.take() {
            Some(value) => Poll::Ready(value),
            None => {
                state.waker = Some(cx.waker().clone());
                Poll::Pending
            }
        }
    }
}

async fn slow_square(n: u64) -> u64 {
    sleep(Duration::from_millis(100 * n)).await;
    n * n
}

fn main() {
    let start = Instant::now();
    let total = block_on(async {
        let a = spawn_with_handle(slow_square(2));
        let b = spawn_with_handle(slow_square(3));
        a.await + b.await
    });
    println!("total = {total} after {} ms", start.elapsed().as_millis());
}
```

```text
total = 13 after 305 ms
```

Why it works: both tasks are spawned before main awaits anything, so their sleeps (200 ms and 300 ms) overlap. `a.await` waits about 200 ms, and `b.await` about 100 ms more. The check "is the result there?" and "store my waker" happen under one lock, so the wake cannot be lost (Lesson 14.10).

</details>

## Quiz

1. What is in the run queue, and who puts things into it?

<details>
<summary>Answer</summary>

Tasks (`Arc<Task>`) that are ready to be polled. `spawn` puts new tasks there, and every `wake()` of a task's waker puts that task there again, from any thread.

</details>

2. Why does our `sleep` need no helper thread, unlike `Delay` in Lesson 14.10?

<details>
<summary>Answer</summary>

It registers `(deadline, waker)` in the runtime's timer list. The executor itself checks the list and sleeps with `park_timeout` until the nearest deadline, then wakes the due timers.

</details>

3. What does the executor do when the run queue is empty?

<details>
<summary>Answer</summary>

It fires the timers that are due. If the queue is still empty, it parks the thread: until the nearest deadline (`park_timeout`), or, with no timers, until a waker calls `unpark`.

</details>

4. Why is the queue's lock guard dropped before a task is polled?

<details>
<summary>Answer</summary>

Polling may call `spawn` or `wake`, which lock the queue again. A `std::sync::Mutex` locked twice on one thread waits for itself forever, so the program would hang.

</details>

5. A task runs a loop of ten seconds of computation without any `.await`. What happens to the other tasks in our runtime, and in Tokio?

<details>
<summary>Answer</summary>

In our runtime they all wait ten seconds: scheduling is cooperative, and only `.await` points give the thread back. In Tokio that worker thread is also lost for ten seconds: the other worker threads keep running other tasks, but with few workers, or with the single-threaded runtime, everything slows down or stops. Move such work to `spawn_blocking` or a thread pool.

</details>

## Summary

- A runtime is a loop around a run queue: pop a task, poll it once, keep it if `Pending`, drop it if `Ready`.
- A task's waker (`impl Wake for Task`) pushes the task back into the queue and unparks the executor thread, so wakes from any thread work.
- A thread-local `CURRENT` lets `spawn` and `sleep` find the runtime; outside `block_on` they panic.
- Timers are entries `(deadline, waker)` in one list; the executor fires due ones and parks until the nearest deadline instead of spinning.
- Never hold a lock while polling a task or calling wakers.
- Tokio adds worker threads with work stealing, a timer wheel, an I/O reactor, `JoinHandle`, and a budget for fairness; scheduling is still cooperative, so never block inside a task.

## Sources and further reading

- [Tokio tutorial — Async in depth](https://tokio.rs/tokio/tutorial/async) — builds a "mini Tokio" with the same parts, and explains how Tokio's wakers work.
- [Making the Tokio scheduler 10x faster](https://tokio.rs/blog/2019-10-scheduler) — the Tokio blog post about run queues, work stealing and the task state flag.
- [std::task::Wake](https://doc.rust-lang.org/std/task/trait.Wake.html) — the trait we implemented for `Task`.
- [std::thread::park_timeout](https://doc.rust-lang.org/std/thread/fn.park_timeout.html) — parking with a time limit, and the rules about spurious wake-ups.
- [std::thread_local](https://doc.rust-lang.org/std/macro.thread_local.html) — the macro behind `CURRENT`.
- [mio on docs.rs](https://docs.rs/mio/latest/mio/) — the I/O library under Tokio's reactor, for when you want to add I/O to your runtime.

---

[🏠 Start](../START-HERE.md) · [← 14.10 How futures work: Future, poll and Waker](10-how-futures-work.md) · [📋 Phase 14](README.md) · [14.12 Pin and self-referential futures →](12-pin-and-self-referential-futures.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
