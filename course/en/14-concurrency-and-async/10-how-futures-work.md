<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.9 Testing concurrent code with loom](09-testing-concurrent-code-with-loom.md) · [📋 Phase 14](README.md) · [14.11 Build a tiny async runtime →](11-build-a-tiny-async-runtime.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/10-how-futures-work.md) <!-- nav -->

# Lesson 14.10 — How futures work: Future, poll and Waker

> **Phase 14 — Concurrency and async in depth** · Lesson 10 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why a future does nothing until somebody polls it, and what the compiler turns an `async fn` into.
- The contract between a future and the executor: "return `Pending` only after you arranged a wake-up".
- How to write a *leaf future* by hand that a helper thread completes, and why it works under any runtime.
- How to build a `Waker` with the `std::task::Wake` trait and write a `block_on` that sleeps instead of spinning.
- The two classic bugs: a future that never wakes, and a wake-up that is lost.

## Before you start

- You finished [Lesson 14.9 — Testing concurrent code with loom](09-testing-concurrent-code-with-loom.md).
- You know the `Future` trait, `Poll::Ready` / `Poll::Pending`, `pin!` and the small `block_on` with `Waker::noop()` from [Lesson 06.9 — Introduction to async/await](../06-rust-advanced/09-async-await-introduction.md). This lesson starts where that one stopped.
- You know `Arc<Mutex<T>>` and threads from [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), and `enum` with `match` from Phase 04.
- Your code for this lesson goes in `~/projects/rust/code/p14-futures`, a library with several small programs in `src/bin/` (each file there is one program; run it with `cargo run --bin <file name>`).

## New words

| Word | Meaning |
|---|---|
| **lazy** | Does nothing until asked. A Rust future does no work until it is polled. |
| **leaf future** | A future at the bottom: it does not wait for other futures, it waits for something outside (a timer, a socket, another thread). Runtimes and libraries write these by hand. |
| **`Context`** | The second argument of `poll`. Today it carries one thing: the `Waker` of the task that is polling. |
| **`Waker`** | A handle that says "poll this task again". Anyone who has it (another thread, a timer) can call `wake()`. |
| **`Wake` trait** | A standard trait (`std::task::Wake`): implement `wake` for your type in an `Arc`, and `Waker::from` turns it into a `Waker`. |
| **lost wake-up** | A bug: the event happens, but nobody calls the waker (or the waker is called before it is stored), so the future is never polled again. |
| **busy polling** | Calling `poll` again and again in a loop without sleeping. It works, but burns a whole CPU (Central Processing Unit) core. |
| **park / unpark** | `std::thread::park()` puts the current thread to sleep; `thread.unpark()` from another thread wakes it up. |
| **spurious wake-up** | A wake-up without a reason. `park` may return early, so code around it must check again. |

## 1. The setup, and futures are lazy

Create the package and add two dependencies:

```bash
cd ~/projects/rust
cargo new --lib code/p14-futures
cd code/p14-futures
cargo add futures@0.3
cargo add tokio@1 --features rt-multi-thread,macros,time
mkdir src/bin
```

The `futures` crate has a small, simple executor, `futures::executor::block_on`. We use it until we write our own in Section 5.

Create `src/bin/lazy.rs`:

```rust
async fn hello() -> u32 {
    println!("hello() is running");
    7
}

fn main() {
    let future = hello();
    println!("the future was created, nothing printed yet");
    let value = futures::executor::block_on(future);
    println!("value = {value}");
}
```

Run `cargo run --bin lazy`:

```text
the future was created, nothing printed yet
hello() is running
value = 7
```

**What happens:**

1. `hello()` does not run the body. It only creates a value, the future, and stores the arguments in it.
2. The first line printed comes from `main`, so the body of `hello` has not started yet.
3. `block_on` calls `poll` on the future. Only now does the body run, print, and return `Ready(7)`.

So a future is a *description of work*. Somebody must poll it. If nobody does, nothing happens; the compiler even warns you when you create a future and drop it without `.await` ("futures do nothing unless you `.await` or poll them").

## 2. An `async fn` is a state machine

What exactly does the compiler build from an `async fn`? Here is a small leaf future, `YieldOnce`, that says "not yet" the first time and "ready" the second time, and an `async fn` that awaits it twice. Next to it is a hand-written future that does the same thing in the way the compiler does it. Create `src/bin/state_machine.rs`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

/// A future that is not ready the first time it is polled.
struct YieldOnce {
    yielded: bool,
}

impl Future for YieldOnce {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.yielded {
            Poll::Ready(())
        } else {
            self.yielded = true;
            cx.waker().wake_by_ref(); // "poll me again soon"
            Poll::Pending
        }
    }
}

fn yield_once() -> YieldOnce {
    YieldOnce { yielded: false }
}

// What we write:
async fn two_steps() -> u32 {
    println!("step 1");
    yield_once().await;
    println!("step 2");
    yield_once().await;
    println!("done");
    42
}

// Roughly what the compiler makes of it:
enum TwoSteps {
    Start,
    WaitingFirst(YieldOnce),
    WaitingSecond(YieldOnce),
    Finished,
}

impl Future for TwoSteps {
    type Output = u32;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<u32> {
        loop {
            match &mut *self {
                TwoSteps::Start => {
                    println!("step 1");
                    *self = TwoSteps::WaitingFirst(yield_once());
                }
                TwoSteps::WaitingFirst(inner) => match Pin::new(inner).poll(cx) {
                    Poll::Pending => return Poll::Pending,
                    Poll::Ready(()) => {
                        println!("step 2");
                        *self = TwoSteps::WaitingSecond(yield_once());
                    }
                },
                TwoSteps::WaitingSecond(inner) => match Pin::new(inner).poll(cx) {
                    Poll::Pending => return Poll::Pending,
                    Poll::Ready(()) => {
                        println!("done");
                        *self = TwoSteps::Finished;
                        return Poll::Ready(42);
                    }
                },
                TwoSteps::Finished => panic!("polled after completion"),
            }
        }
    }
}

fn main() {
    println!("async fn:");
    println!("result = {}", futures::executor::block_on(two_steps()));
    println!("hand-written:");
    println!("result = {}", futures::executor::block_on(TwoSteps::Start));
}
```

```text
async fn:
step 1
step 2
done
result = 42
hand-written:
step 1
step 2
done
result = 42
```

**What happens:**

1. `YieldOnce` is a leaf future. On the first poll it remembers that it yielded, calls `cx.waker().wake_by_ref()` ("poll me again") and returns `Pending`.
2. `TwoSteps` is an `enum` with one variant per *place where the code can stop*: before the start, at the first `.await`, at the second `.await`, and after the end. The inner future that is being awaited lives inside the variant.
3. `poll` runs the code up to the next `.await`, then polls the inner future. If the inner one says `Pending`, the outer one returns `Pending` too: the whole chain stops and gives the thread back.
4. When polled again, `poll` continues from the stored state, not from the beginning. The `loop` lets it run on into the next step without returning.
5. `Pin::new(inner)` makes a pinned reference to the inner future. This is allowed here because `YieldOnce` is a simple type that may be moved freely; [Lesson 14.12](12-pin-and-self-referential-futures.md) explains when it is not.

This is the main idea of async Rust: **`.await` does not block the thread, it returns `Pending` up the chain**. The local variables that must survive an `.await` are stored in the state, so the future remembers where it was.

The real compiler output is a similar state machine with numbered states. Like our `Finished` arm, it panics if it is polled after it finished. `src/bin/twice.rs` polls an `async` block twice:

```rust
use std::future::Future;
use std::pin::pin;
use std::task::{Context, Waker};

fn main() {
    let mut future = pin!(async { 1 + 1 });
    let mut cx = Context::from_waker(Waker::noop());
    println!("{:?}", future.as_mut().poll(&mut cx));
    println!("{:?}", future.as_mut().poll(&mut cx));
}
```

```text
Ready(2)

thread 'main' (5437152) panicked at src/bin/twice.rs:6:27:
`async fn` resumed after completion
```

An executor must never poll a finished future.

## 3. The contract: who polls again, and when?

In Lesson 06.9 the mini `block_on` polled in an endless loop with `Waker::noop()`. That works, but it is expensive: while the future waits, the loop calls `poll` millions of times and keeps one CPU core at 100 % (you will measure it at the end of Section 5). A real runtime instead follows a contract with every future:

> When `poll` returns `Pending`, the future must make sure that `cx.waker()` (or a clone of it) will be woken when progress is possible. The executor does not poll the task again until it is woken.

The executor promises "I will poll you again after a wake". The future promises "I will arrange a wake". If the future breaks its promise, the task sleeps forever. Here is such a buggy future in `src/bin/lost_wake.rs`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll};
use std::thread;
use std::time::{Duration, Instant};

/// Like `Delay`, but it forgets to keep the waker. BUG.
struct ForgetfulDelay {
    done: Arc<Mutex<bool>>,
}

impl ForgetfulDelay {
    fn new(duration: Duration) -> Self {
        let done = Arc::new(Mutex::new(false));
        let for_thread = Arc::clone(&done);
        thread::spawn(move || {
            thread::sleep(duration);
            *for_thread.lock().unwrap() = true; // nobody is told
        });
        ForgetfulDelay { done }
    }
}

impl Future for ForgetfulDelay {
    type Output = ();

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<()> {
        if *self.done.lock().unwrap() {
            Poll::Ready(())
        } else {
            Poll::Pending // BUG: no waker was stored
        }
    }
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let delay = ForgetfulDelay::new(Duration::from_millis(100));
    let result = tokio::time::timeout(Duration::from_secs(1), delay).await;
    println!("{result:?} after {} ms", start.elapsed().as_millis());
}
```

```text
Ok(()) after 1007 ms
```

**What happens:**

1. Tokio polls `ForgetfulDelay` once. It is not done, it returns `Pending`, and it does not keep the waker.
2. After 100 ms the helper thread sets `done = true`, but it has no way to tell anybody. Tokio does not poll the task again: it is waiting for a wake that never comes.
3. At 1000 ms the `timeout` timer fires. Tokio's timer *does* wake the task. `timeout` first polls the inner future, which now sees `done == true` and returns `Ready`.
4. The result is `Ok(())`, but 900 ms too late, and only by luck. Without the `timeout`, `delay.await` hangs forever (try it and stop the program with Ctrl+C).

> ⚠️ **Common mistake:** A future that returns `Pending` must have saved the waker somewhere, or must have passed it to someone who will call it. "I will be ready soon" is not enough: the executor does not guess.

## 4. A correct leaf future: `Delay`

Now the right way. The future and a helper thread share a small state: "are we done?" and "whom do I wake?". Put this into `src/lib.rs` (the full file, with `block_on` from Section 5 at the end, is what you will have after this lesson):

```rust
//! Hand-written futures and a tiny `block_on`.

use std::future::Future;
use std::pin::{Pin, pin};
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Wake, Waker};
use std::thread::{self, Thread};
use std::time::Duration;

/// The state that the future and the helper thread share.
struct Shared {
    done: bool,
    waker: Option<Waker>,
}

/// A future that becomes ready after `duration`.
/// A helper thread sleeps and then wakes the task.
pub struct Delay {
    shared: Arc<Mutex<Shared>>,
}

impl Delay {
    pub fn new(duration: Duration) -> Self {
        let shared = Arc::new(Mutex::new(Shared {
            done: false,
            waker: None,
        }));
        let for_thread = Arc::clone(&shared);
        thread::spawn(move || {
            thread::sleep(duration);
            let mut state = for_thread.lock().unwrap();
            state.done = true;
            if let Some(waker) = state.waker.take() {
                waker.wake();
            }
        });
        Delay { shared }
    }
}

impl Future for Delay {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let mut state = self.shared.lock().unwrap();
        if state.done {
            Poll::Ready(())
        } else {
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

**What happens:**

1. `Delay::new` creates the shared state and starts a helper thread. The thread sleeps, then sets `done = true` and, if a waker is stored, takes it out and calls `wake()`.
2. `poll` locks the state. If `done`, it returns `Ready`. Otherwise it stores a **clone** of the current waker (the `Context` only lends it) and returns `Pending`. The contract is kept: whoever sets `done` will wake us.
3. Each poll overwrites the stored waker. This matters: a task may be moved to another thread or another executor between polls, and then only the newest waker is correct. (`Waker::will_wake` can tell you whether the old one is still the same, to avoid a clone.)
4. The check of `done` and the store of the waker happen **under the same lock**. If they did not, this order would be possible: `poll` sees `done == false`; the helper thread sets `done = true` and finds no waker; `poll` stores the waker and returns `Pending`. The wake would be lost forever. The lock makes "check and store" one step from the helper thread's point of view.

`Delay` knows nothing about Tokio. It works with any executor that gives it a waker. `src/bin/delay.rs` runs it under two different ones:

```rust
use p14_futures::Delay;
use std::time::{Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now();
    Delay::new(Duration::from_millis(200)).await;
    println!("tokio: ready after {} ms", start.elapsed().as_millis());

    let start = Instant::now();
    futures::executor::block_on(Delay::new(Duration::from_millis(200)));
    println!("futures: ready after {} ms", start.elapsed().as_millis());
}
```

```text
tokio: ready after 204 ms
futures: ready after 201 ms
```

(Calling `futures::executor::block_on` inside a Tokio task blocks a runtime thread, which Lesson 06.9 told you not to do; it is acceptable only in a tiny experiment like this.)

> ⚡ **Performance:** One thread per `Delay` is fine for learning but expensive for real programs: a thread costs memory and operating-system work. Real runtimes keep *all* timers in one data structure and use one sleep for the nearest deadline. You build that in [Lesson 14.11](11-build-a-tiny-async-runtime.md).

## 5. A `Waker` from the `Wake` trait, and a `block_on` that sleeps

The executor's side of the contract: create a waker, poll, and when the answer is `Pending`, sleep until the waker is called. For one future on one thread, "sleep" is `thread::park()` and "wake" is `unpark()`.

Building a `Waker` by hand is low-level (`RawWaker` with a table of four `unsafe` functions). The standard library gives a safe way: implement the `Wake` trait for a type, put it in an `Arc`, and convert. Add this to the end of `src/lib.rs`:

```rust
/// Wakes a thread that sleeps in `block_on`.
struct ThreadWaker {
    thread: Thread,
}

impl Wake for ThreadWaker {
    fn wake(self: Arc<Self>) {
        self.thread.unpark();
    }
}

/// How many times `block_on` called `poll` (for the experiments in this lesson).
pub static POLLS: AtomicUsize = AtomicUsize::new(0);

/// Runs one future to completion on the current thread.
pub fn block_on<F: Future>(future: F) -> F::Output {
    let mut future = pin!(future);
    let waker = Waker::from(Arc::new(ThreadWaker {
        thread: thread::current(),
    }));
    let mut cx = Context::from_waker(&waker);
    loop {
        POLLS.fetch_add(1, Ordering::Relaxed);
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(value) => return value,
            Poll::Pending => thread::park(),
        }
    }
}
```

**What happens:**

1. `ThreadWaker` remembers the thread that runs `block_on` (`thread::current()`).
2. `impl Wake` says what "wake" means for us: unpark that thread. The method takes `self: Arc<Self>`, because a waker is shared and can be cloned many times. `Waker::from(Arc::new(...))` builds the low-level `RawWaker` for us, safely.
3. The loop polls once. On `Ready` it returns the value. On `Pending` it parks the thread: the thread sleeps and uses no CPU.
4. When the helper thread of a `Delay` calls `wake()`, our thread wakes up and polls again.
5. If `unpark` happens *before* `park`, nothing is lost: `unpark` leaves a "token" and the next `park` returns at once. And if `park` returns early without a reason (a spurious wake-up), the loop just polls again and gets `Pending` again. Polling too often is allowed; never polling again is the bug.

Try it with an `async fn` that awaits two delays, in `src/bin/my_block_on.rs`:

```rust
use p14_futures::{Delay, POLLS, block_on};
use std::sync::atomic::Ordering;
use std::time::{Duration, Instant};

async fn add_slowly(a: u32, b: u32) -> u32 {
    Delay::new(Duration::from_millis(100)).await;
    Delay::new(Duration::from_millis(100)).await;
    a + b
}

fn main() {
    let start = Instant::now();
    let sum = block_on(add_slowly(2, 3));
    println!("sum = {sum} after {} ms", start.elapsed().as_millis());
    println!("poll was called {} times", POLLS.load(Ordering::Relaxed));
}
```

```text
sum = 5 after 205 ms
poll was called 3 times
```

Three polls: one at the start, one after each delay. Between them the thread sleeps. This is exactly what Tokio does too, only with many tasks, many threads and one shared timer.

For comparison, here is the busy loop from Lesson 06.9 running a 200 ms `Delay`, in `src/bin/busy.rs`:

```rust
use p14_futures::Delay;
use std::future::Future;
use std::pin::pin;
use std::task::{Context, Poll, Waker};
use std::time::Duration;

/// The `block_on` from Lesson 06.9: it never sleeps.
fn busy_block_on<F: Future>(future: F) -> (F::Output, u64) {
    let mut future = pin!(future);
    let mut cx = Context::from_waker(Waker::noop());
    let mut polls = 0;
    loop {
        polls += 1;
        if let Poll::Ready(value) = future.as_mut().poll(&mut cx) {
            return (value, polls);
        }
    }
}

fn main() {
    let ((), polls) = busy_block_on(Delay::new(Duration::from_millis(200)));
    println!("busy loop: poll was called {polls} times");
}
```

`cargo run --release --bin busy`:

```text
busy loop: poll was called 39946637 times
```

Almost 40 million calls to do nothing (the exact number changes with every run), against 2 with a waker that works (one poll that returns `Pending`, one after the wake).

> 🧪 **Testing:** A hand-written future is easy to test with a hand-written executor: `block_on` turns async code back into a normal function, so a plain `#[test]` works. Exercise 2 does this.

## Common mistakes

**1. Returning `Pending` without arranging a wake.** The task is never polled again (Section 3). Every `Pending` needs a stored waker or a waker handed to someone who will call it.

**2. Checking the condition and storing the waker in two separate steps.** The event can happen in between, and its wake is lost. Keep "check" and "store waker" under one lock (or use an atomic protocol and check again after storing).

**3. Polling a future after it returned `Ready`.** Compiler-made futures panic with "`async fn` resumed after completion". Hand-written ones may return nonsense. Executors drop a future as soon as it is `Ready`.

## Try it yourself

1. Create `code/p14-futures` as in Section 1 and run `lazy`, `state_machine` and `delay`.
2. Add `ForgetfulDelay` (`src/bin/lost_wake.rs`) and run it. Then remove the `timeout` (write `delay.await;`) and see that the program hangs. Stop it with Ctrl+C.
3. Fix `ForgetfulDelay` by giving it the same `Shared` state as `Delay`. It must finish after about 100 ms.
4. Add `block_on` and run `my_block_on`. Change it to await three delays and predict the number of polls before you run it.
5. Run `cargo clippy --all-targets` and `cargo fmt`, then commit: `git add code/p14-futures && git commit -m "p14-futures: Delay and block_on"`.

## Exercises

### Exercise 1 — Count the polls (easy)

Write a future `YieldTimes { left: u32 }` that returns `Pending` `left` times (waking itself each time) and then `Ready(())`. Run `YieldTimes { left: 3 }` with your `block_on` and print `POLLS`. How many polls do you expect?

<details>
<summary>Hint</summary>

Copy `YieldOnce` from Section 2 and replace the `bool` with a counter that goes down. `wake_by_ref` unparks our own thread, so the next `park` returns at once.

</details>

<details>
<summary>Answer</summary>

```rust
use p14_futures::{POLLS, block_on};
use std::future::Future;
use std::pin::Pin;
use std::sync::atomic::Ordering;
use std::task::{Context, Poll};

/// Returns `Pending` `times` times, then `Ready`.
struct YieldTimes {
    left: u32,
}

impl Future for YieldTimes {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.left == 0 {
            Poll::Ready(())
        } else {
            self.left -= 1;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

fn main() {
    block_on(YieldTimes { left: 3 });
    println!("poll was called {} times", POLLS.load(Ordering::Relaxed));
}
```

```text
poll was called 4 times
```

Why it works: three `Pending` answers and one `Ready` make four polls. Each `wake_by_ref` calls `unpark` before `park`, so `park` does not sleep: the token is already there.

</details>

### Exercise 2 — Test the executor (medium)

Add a `#[cfg(test)] mod tests` to `src/lib.rs` with three tests: `block_on` returns the value of an `async` block; `Delay` of 50 ms takes at least 50 ms under your `block_on`; `Delay` also completes under `futures::executor::block_on`.

<details>
<summary>Hint</summary>

`block_on(async { 40 + 2 })` is a normal function call, so `assert_eq!` works. For the time, use `std::time::Instant` and compare with `>=`, never with `==`: sleeping can always take a little longer.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::time::Instant;

    #[test]
    fn block_on_returns_the_value_of_an_async_block() {
        assert_eq!(block_on(async { 40 + 2 }), 42);
    }

    #[test]
    fn delay_waits_at_least_its_duration() {
        let start = Instant::now();
        block_on(Delay::new(Duration::from_millis(50)));
        assert!(start.elapsed() >= Duration::from_millis(50));
    }

    #[test]
    fn delay_works_under_another_executor() {
        futures::executor::block_on(Delay::new(Duration::from_millis(10)));
    }
}
```

`cargo test` prints `test result: ok. 3 passed; 0 failed` for the library (and `0 tests` for each program in `src/bin/`). Why it works: `block_on` runs the future to the end on the test thread. The third test proves that `Delay` follows the contract and does not depend on one executor.

</details>

### Exercise 3 — Your own `join` (hard)

Write `Join2<A, B>`, a future that polls two futures and is ready when **both** are ready, with output `(A::Output, B::Output)`. Run two `Delay`s of 300 ms with it under your `block_on`. It must take about 300 ms, not 600 ms. You may require `A: Future + Unpin` and `B: Future + Unpin`.

<details>
<summary>Hint</summary>

Store both futures and two `Option`s for their outputs. In `poll`, poll each future that has no output yet, using `Pin::new(&mut this.a)`. `self.get_mut()` turns `Pin<&mut Self>` into `&mut Self`, which is allowed only when the type is `Unpin` (short version: `Unpin` means "safe to move even after it was pinned"; Lesson 14.12 explains it). `Join2` is `Unpin` only if all its fields are, so the outputs must be `Unpin` too. Both futures share one waker: whichever wakes, the whole `Join2` is polled again.

</details>

<details>
<summary>Answer</summary>

```rust
use p14_futures::{Delay, block_on};
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

/// Runs two futures at the same time and finishes when both are done.
struct Join2<A: Future, B: Future> {
    a: A,
    b: B,
    a_out: Option<A::Output>,
    b_out: Option<B::Output>,
}

impl<A, B> Future for Join2<A, B>
where
    A: Future + Unpin,
    B: Future + Unpin,
    A::Output: Unpin,
    B::Output: Unpin,
{
    type Output = (A::Output, B::Output);

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.get_mut();
        if this.a_out.is_none()
            && let Poll::Ready(value) = Pin::new(&mut this.a).poll(cx)
        {
            this.a_out = Some(value);
        }
        if this.b_out.is_none()
            && let Poll::Ready(value) = Pin::new(&mut this.b).poll(cx)
        {
            this.b_out = Some(value);
        }
        if this.a_out.is_some() && this.b_out.is_some() {
            Poll::Ready((this.a_out.take().unwrap(), this.b_out.take().unwrap()))
        } else {
            Poll::Pending
        }
    }
}

fn join2<A: Future, B: Future>(a: A, b: B) -> Join2<A, B> {
    Join2 {
        a,
        b,
        a_out: None,
        b_out: None,
    }
}

fn main() {
    let start = Instant::now();
    let both = join2(
        Delay::new(Duration::from_millis(300)),
        Delay::new(Duration::from_millis(300)),
    );
    block_on(both);
    println!("both done after {} ms", start.elapsed().as_millis());
}
```

```text
both done after 300 ms
```

`if cond && let Pattern = value { … }` is a *let chain* (edition 2024): the body runs only if both parts succeed. Why it works: both delays start their helper threads at creation, so they wait at the same time. Each poll of `Join2` gives the same waker to both inner futures, and a finished output is kept in its `Option`, so a finished future is never polled again.

</details>

## Quiz

1. You call an `async fn` but never `.await` it. What runs?

<details>
<summary>Answer</summary>

Nothing from its body. Calling it only creates the future (a state machine holding the arguments). The body runs when someone polls it.

</details>

2. What does one variant of the compiler's state machine correspond to?

<details>
<summary>Answer</summary>

One place where the function can stop: the start, each `.await`, and the end. The variant holds the future being awaited and the local variables that must live across that `.await`.

</details>

3. What exactly must a future do before it returns `Pending`?

<details>
<summary>Answer</summary>

Make sure the current waker (`cx.waker()`) will be woken when progress is possible: store a clone where the event source will find it, or pass it to that source. Otherwise the executor never polls it again.

</details>

4. Why does `Delay::poll` check `done` and store the waker under one lock?

<details>
<summary>Answer</summary>

To avoid a lost wake-up. If the helper thread set `done` and looked for a waker between our check and our store, it would find none, and we would then return `Pending` with nobody left to wake us.

</details>

5. What does `impl Wake for ThreadWaker` give you, and why does `wake` take `self: Arc<Self>`?

<details>
<summary>Answer</summary>

It lets `Waker::from(Arc::new(ThreadWaker { … }))` build a real `Waker` without `unsafe` code. `wake` takes an `Arc` because a waker is shared: it can be cloned and given to many places, and the data must live as long as any clone.

</details>

## Summary

- Futures are lazy: an `async fn` call creates a state machine, and nothing runs until it is polled.
- The compiler turns each `async fn` into an `enum`-like state machine with one state per `.await`; `.await` returns `Pending` up the chain instead of blocking the thread.
- The contract: return `Pending` only after arranging that `cx.waker()` will be woken; the executor polls again only after a wake.
- A leaf future such as `Delay` keeps the newest waker in shared state, checks and stores under one lock, and works with any executor.
- `std::task::Wake` builds a `Waker` safely; with `thread::park`/`unpark` it gives a `block_on` that sleeps instead of busy polling.
- Lost wake-ups (no waker stored, or a race between check and store) make tasks hang; extra polls are harmless.

## Sources and further reading

- [std::future::Future](https://doc.rust-lang.org/std/future/trait.Future.html) — the trait and the exact rules for `poll` and `Pending`.
- [std::task::Wake](https://doc.rust-lang.org/std/task/trait.Wake.html) — the trait, with a `block_on` example of its own.
- [std::task::Waker](https://doc.rust-lang.org/std/task/struct.Waker.html) — `wake`, `wake_by_ref`, `will_wake` and `noop`.
- [The Rust Programming Language — 17.5 A Closer Look at the Traits for Async](https://doc.rust-lang.org/book/ch17-05-traits-for-async.html) — `Future`, `Pin` and state machines in the official book.
- [Tokio tutorial — Async in depth](https://tokio.rs/tokio/tutorial/async) — how Tokio uses wakers, with a hand-written `Delay`.
- [Asynchronous Programming in Rust (the async book)](https://rust-lang.github.io/async-book/) — the official async book.

---

[🏠 Start](../START-HERE.md) · [← 14.9 Testing concurrent code with loom](09-testing-concurrent-code-with-loom.md) · [📋 Phase 14](README.md) · [14.11 Build a tiny async runtime →](11-build-a-tiny-async-runtime.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
