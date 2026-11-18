<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.8 Procedural macros](08-procedural-macros.md) · [📋 Phase 06](README.md) · [06.10 FFI: calling C code →](10-ffi-calling-c.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/09-async-await-introduction.md) <!-- nav -->

# Lesson 06.9 — Introduction to async/await

> **Phase 06 — Rust advanced** · Lesson 9 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- Why programs that wait (for the network, for a timer, for a disk) need a different tool than "one thread per job".
- What a `Future` is, what `async fn` and `.await` really do, and why a future does nothing until it is polled.
- How to use the Tokio runtime: `#[tokio::main]`, `tokio::spawn`, `tokio::join!`, channels, `select!` and timeouts.
- The biggest async mistake — blocking the runtime — and the other classic errors with their real compiler messages.
- How to test async code and how to choose between async and threads.

## Before you start

- You finished [Lesson 06.8 — Procedural macros](08-procedural-macros.md). `#[tokio::main]` is a procedural macro.
- You know threads, channels and `Arc<Mutex<T>>`: [Lesson 05.10 — Threads, channels and shared state](../05-rust-intermediate/10-threads-channels-and-shared-state.md).
- You know closures and `move`: [Lesson 05.4 — Closures](../05-rust-intermediate/04-closures.md).
- Create a project: `cd ~/projects/rust && cargo new code/p06-async`, then add Tokio:

  ```sh
  cd code/p06-async
  cargo add tokio@1.54 --features rt-multi-thread,macros,time,sync
  ```

## New words

| Word | Meaning |
|---|---|
| **I/O** (Input/Output) | Everything where your program talks to the outside: network, disk, keyboard, timers. I/O is slow compared with the CPU. |
| **CPU** (Central Processing Unit) | The chip that runs program instructions. |
| **concurrency** | Having several jobs *in progress* at the same time, even if only one runs at any exact moment. |
| **parallelism** | Several jobs *actually running* at the same instant, on different CPU cores. |
| **asynchronous (async)** | A way to write code that can pause while it waits, and let other code run in the meantime. |
| **future** | A value that stands for work that will finish later. In Rust it is anything that implements the `Future` trait. |
| **poll** | To ask a future: "can you make progress now?" It answers `Ready(value)` or `Pending`. |
| **`async fn`** | A function that, when called, does not run its body but returns a future. |
| **`.await`** | Inside an async function: wait for a future to finish, and let other work run while you wait. |
| **state machine** | A thing that is always in one of a few named states and moves between them. The compiler turns every async function into one. |
| **runtime** | A library that runs futures: it polls them, wakes them up, and provides timers and network I/O. Rust's standard library has no runtime. Tokio is the most used one. |
| **executor** | The part of a runtime that decides which future to poll next. |
| **waker** | A handle that a future gives to the runtime to say "poll me again, I can progress now". |
| **task** | A future that was given to the runtime to run on its own (with `tokio::spawn`). Like a very light thread. |
| **blocking** | Doing something that keeps the thread busy or sleeping without letting the runtime run other tasks. |
| **`Send`** | A marker trait: values of this type may be moved to another thread. |
| **cancellation** | Stopping a future before it finishes. In Rust you cancel by dropping the future. |
| **timeout** | A limit on how long you are willing to wait. |

## 1. The problem: programs that wait

Imagine a cook who boils water for tea. The water needs three minutes. A *blocking* cook stands next to the pot and does nothing for three minutes. If you ask him to also toast bread, he starts after the tea is ready.

You have two ways to be faster:

1. **Hire more cooks** — one per job. This is what threads give you (Lesson 05.10). It works, but each cook needs a place in the kitchen (a thread needs memory, usually a few megabytes of address space for its stack, and the operating system has to switch between them).
2. **One clever cook who never waits.** He puts the water on the stove, and while it heats he toasts bread; when the kettle whistles he comes back. This is **async** (asynchronous) programming.

A server that talks to 10,000 clients is the same story. Most of the time each client connection is *waiting* for a network message. Ten thousand threads would be heavy; ten thousand async tasks are light.

Async is for **I/O-bound** work: the program mostly waits. For **CPU-bound** work (heavy calculation) async does not help, because there is nothing to wait for. We compare both at the end of the lesson.

## 2. What a future is

A **future** is a value that means "some work that will finish later". The standard library defines it as a trait:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

The only operation is `poll`. Someone (the runtime) calls `poll` again and again. Each time, the future either:

- returns `Poll::Ready(value)` — "I am done, here is the result", or
- returns `Poll::Pending` — "not yet; I told the runtime when to call me again" (through the **waker** inside `cx`).

`Pin<&mut Self>` means: "a mutable reference to the future, which is promised not to move in memory". Futures made by the compiler may hold references to their own fields, so they must not move once polling starts. For this lesson you only need to know the *ritual*: `pin!(future)` creates a pinned future.

To see that there is no magic, let us write a future by hand and a tiny executor that polls it. (Real runtimes are much smarter. You build one in a later phase.)

```rust
use std::future::Future;
use std::pin::{Pin, pin};
use std::task::{Context, Poll, Waker};

struct CountDown(u32);

impl Future for CountDown {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.0 == 0 {
            Poll::Ready("liftoff")
        } else {
            println!("count: {}", self.0);
            self.0 -= 1;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

fn block_on<F: Future>(future: F) -> F::Output {
    let mut future = pin!(future);
    let mut cx = Context::from_waker(Waker::noop());
    loop {
        if let Poll::Ready(value) = future.as_mut().poll(&mut cx) {
            return value;
        }
    }
}

fn main() {
    println!("{}", block_on(CountDown(3)));
}
```

**What happens:**

1. `CountDown(3)` is a future that needs to be polled four times to finish.
2. `block_on` is our mini executor. It pins the future, makes a `Context` with a "do nothing" waker (`Waker::noop()`), and calls `poll` in a loop until the answer is `Ready`.
3. The first three polls print `count: 3`, `count: 2`, `count: 1`, decrease the counter, call `cx.waker().wake_by_ref()` ("please poll me again") and return `Pending`.
4. The fourth poll sees `0` and returns `Ready("liftoff")`. `block_on` returns it.

```text
count: 3
count: 2
count: 1
liftoff
```

## 3. `async fn` and `.await`

Writing `poll` by hand is hard. The compiler does it for you. An **`async fn`** does not run when you call it. It returns a future that contains the body:

```rust
use std::future::Future;
use std::pin::pin;
use std::task::{Context, Poll, Waker};

async fn add(a: i32, b: i32) -> i32 {
    println!("adding {a} and {b}");
    a + b
}

fn block_on<F: Future>(future: F) -> F::Output {
    let mut future = pin!(future);
    let mut cx = Context::from_waker(Waker::noop());
    loop {
        if let Poll::Ready(value) = future.as_mut().poll(&mut cx) {
            return value;
        }
    }
}

fn main() {
    let future = add(2, 3);
    println!("the future exists, nothing has run yet");
    let result = block_on(future);
    println!("result = {result}");
}
```

**What happens:**

1. `add(2, 3)` creates a future. The body is **not** executed, so nothing is printed yet.
2. The program prints `the future exists, nothing has run yet`.
3. `block_on` polls the future. Now the body runs: it prints `adding 2 and 3` and returns `5`.

```text
the future exists, nothing has run yet
adding 2 and 3
result = 5
```

Futures are **lazy**: they do nothing until polled. This is the most important fact in async Rust.

Inside an async function you use **`.await`** on another future. It means: "poll that future; if it is `Pending`, pause *this* function and give control back to the runtime; when the inner future is ready, continue from this exact place".

The compiler turns an async function into a **state machine**. Every `.await` is a place where the function can stop and later continue:

```text
async fn example() {
    let a = step_one().await;     // state 0: waiting for step_one
    let b = step_two(a).await;    // state 1: waiting for step_two (a is kept)
    println!("{b}");              // state 2: done
}

       poll         poll                poll
 start ───► [wait one] ───► [wait two] ───► finished
            variables that live across an .await are stored inside the future
```

Local variables that are still needed after an `.await` are stored inside the future value. Remember this for section 8.

## 4. A runtime: Tokio

Who calls `poll`? A **runtime**. The standard library has no runtime on purpose: there are different designs for different needs. The most used one is **Tokio**. It provides:

- an **executor** that runs many tasks on a few threads;
- timers (`tokio::time`), network and file I/O, channels, locks;
- the macros `#[tokio::main]` and `#[tokio::test]`.

Your first Tokio program:

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn greet(name: &str, delay_ms: u64) -> String {
    sleep(Duration::from_millis(delay_ms)).await;
    format!("Hello, {name}!")
}

#[tokio::main]
async fn main() {
    let message = greet("Aziza", 100).await;
    println!("{message}");
}
```

**What happens:**

1. `greet` is an async function. `sleep(...)` returns a future that becomes ready after the delay. We `.await` it.
2. While `greet` waits, the thread is free: the runtime could run other tasks on it. Here there are none, so it simply waits.
3. `main` cannot be `async` by itself, because someone must create the runtime. `#[tokio::main]` is an **attribute macro** (Lesson 06.8) that does it.
4. `greet("Aziza", 100).await` pauses `main` until the greeting is ready. Then `main` prints it.

```text
Hello, Aziza!
```

`cargo expand` (Lesson 06.7) shows what `#[tokio::main]` wrote. This is the real expansion, shortened:

```rust
fn main() {
    let body = async {
        let message = greet("Aziza", 100).await;
        println!("{message}");
    };
    // ...
    {
        use tokio::runtime::Builder;
        return Builder::new_multi_thread()
            .enable_all()
            .build()
            .expect("Failed building the Runtime")
            .block_on(body);
    }
}
```

Your `async fn main` body becomes an `async` block. A normal `main` builds a runtime and runs that block with `block_on`, just like our mini `block_on`, but much stronger. There is no magic.

> 💡 **Tip:** Tokio is split in *features* so you compile only what you use. This lesson needs `rt-multi-thread` (the runtime), `macros` (`#[tokio::main]`, `join!`, `select!`), `time` and `sync`. When you are unsure, `features = ["full"]` turns everything on.

## 5. Doing several things at the same time

### `join!`

If you `.await` two futures one after the other, they run **one after the other**. To run them at the same time, use `tokio::join!`:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

async fn work(name: &str, ms: u64) -> String {
    sleep(Duration::from_millis(ms)).await;
    format!("{name} done")
}

fn about(start: Instant) -> u128 {
    (start.elapsed().as_millis() + 50) / 100 * 100
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let a = work("first", 200).await;
    let b = work("second", 200).await;
    println!("{a}, {b} after about {} ms", about(start));

    let start = Instant::now();
    let (a, b) = tokio::join!(work("first", 200), work("second", 200));
    println!("{a}, {b} after about {} ms", about(start));
}
```

**What happens:**

1. The first block awaits `work("first")` fully (200 ms), then `work("second")` (200 ms): about 400 ms in total.
2. `tokio::join!` polls both futures in the same task. While the first one waits for its timer, the second one starts its own timer. Both wait together: about 200 ms.
3. `about` rounds the time to the nearest 100 ms, so your output is stable even if the real time is 203 ms.

```text
first done, second done after about 400 ms
first done, second done after about 200 ms
```

### `spawn`

`tokio::spawn` gives a future to the runtime as an independent **task**. It returns a `JoinHandle`, which you `.await` to get the result:

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let mut handles = Vec::new();

    for id in 1..=3 {
        let handle = tokio::spawn(async move {
            sleep(Duration::from_millis(100 * (4 - id))).await;
            println!("task {id} finished");
            id * 10
        });
        handles.push(handle);
    }

    let mut total = 0;
    for handle in handles {
        total += handle.await.unwrap();
    }
    println!("total = {total}");
}
```

**What happens:**

1. The loop creates three tasks. Task 1 sleeps 300 ms, task 2 sleeps 200 ms, task 3 sleeps 100 ms. They all start immediately.
2. They finish in the order of their timers, not the order of creation: 3, 2, 1.
3. `async move` moves `id` into the task. A spawned task can outlive the function that created it, so it cannot borrow local variables.
4. `handle.await` returns `Result<T, JoinError>`. The `Err` case happens if the task panicked. Here we `unwrap()`.
5. We add the results in the order of the handles: `10 + 20 + 30 = 60`.

```text
task 3 finished
task 2 finished
task 1 finished
total = 60
```

| | `join!` | `spawn` |
|---|---|---|
| Runs on | the same task (one at a time, switching at `.await`) | separate tasks (can run on different threads) |
| Can borrow local variables | yes | no: needs `'static` and `Send` |
| Result | all results together | one `JoinHandle` per task |
| Use for | a few known futures in one function | many or independent jobs |

**Why `Send` and `'static`?** The multi-thread runtime may move a task from one thread to another between two polls, so everything inside must be `Send`. The task may also run after the spawning function returned, so it cannot hold plain references to its locals: `'static`.

## 6. Do not block the runtime

A runtime has few threads and many tasks. A task gives the thread back only at an `.await`. If a task keeps the thread busy (a long calculation, `std::thread::sleep`, a blocking file read), **all other tasks on that thread are stuck**.

To see it clearly, we use a runtime with only one thread (`flavor = "current_thread"`). First the wrong way:

```rust
use std::time::{Duration, Instant};

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let start = Instant::now();

    let a = tokio::spawn(async {
        std::thread::sleep(Duration::from_millis(300)); // blocks the whole thread!
    });
    let b = tokio::spawn(async {
        std::thread::sleep(Duration::from_millis(300));
    });

    a.await.unwrap();
    b.await.unwrap();
    println!("two 300 ms tasks took about {} ms", (start.elapsed().as_millis() + 50) / 100 * 100);
}
```

```text
two 300 ms tasks took about 600 ms
```

**What happens:** `std::thread::sleep` puts the *thread* to sleep. The one thread cannot run task `b` while it sleeps in task `a`. The two tasks ran one after the other: 600 ms. The right way is to use the async sleep:

```rust
use std::time::{Duration, Instant};

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let start = Instant::now();

    let a = tokio::spawn(async {
        tokio::time::sleep(Duration::from_millis(300)).await;
    });
    let b = tokio::spawn(async {
        tokio::time::sleep(Duration::from_millis(300)).await;
    });

    a.await.unwrap();
    b.await.unwrap();
    println!("two 300 ms tasks took about {} ms", (start.elapsed().as_millis() + 50) / 100 * 100);
}
```

```text
two 300 ms tasks took about 300 ms
```

`tokio::time::sleep(...).await` pauses only the *task*. The thread runs the other task meanwhile.

What if you must do blocking or heavy CPU work? Send it to a separate thread pool with `spawn_blocking`:

```rust
fn slow_sum(n: u64) -> u64 {
    (1..=n).sum()
}

#[tokio::main]
async fn main() {
    let handle = tokio::task::spawn_blocking(|| slow_sum(10_000_000));
    println!("waiting for the heavy work...");
    let result = handle.await.unwrap();
    println!("sum = {result}");
}
```

```text
waiting for the heavy work...
sum = 50000005000000
```

> ⚡ **Performance:** A rule of thumb: code between two `.await`s should run for microseconds, not milliseconds. If a piece of code takes longer (hashing a big file, compressing, parsing a huge document), use `spawn_blocking` or a normal thread.

## 7. Channels, timeouts and `select!`

Tasks talk through channels, like threads do. Tokio's `mpsc` channel has an async `send` and `recv`:

```rust
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(8);

    let producer = tokio::spawn(async move {
        for i in 1..=3 {
            sleep(Duration::from_millis(50)).await;
            tx.send(format!("message {i}")).await.unwrap();
        }
        // tx is dropped here, so the channel closes
    });

    while let Some(message) = rx.recv().await {
        println!("got: {message}");
    }
    producer.await.unwrap();
    println!("channel closed");
}
```

**What happens:**

1. `mpsc::channel(8)` makes a channel that holds at most 8 messages. When it is full, `send(...).await` waits: this is **backpressure**, the receiver slows down the sender.
2. The producer task sends three messages, 50 ms apart, then ends. Its `tx` is dropped.
3. `rx.recv().await` returns `Some(message)` while messages come, and `None` when all senders are gone. That ends the `while let` loop.

```text
got: message 1
got: message 2
got: message 3
channel closed
```

A **timeout** limits how long you wait. `tokio::select!` waits for the **first** of several futures and drops (cancels) the others:

```rust
use std::time::Duration;
use tokio::time::{sleep, timeout};

async fn slow() -> &'static str {
    sleep(Duration::from_millis(500)).await;
    "finished"
}

#[tokio::main]
async fn main() {
    match timeout(Duration::from_millis(100), slow()).await {
        Ok(text) => println!("ok: {text}"),
        Err(error) => println!("gave up: {error}"),
    }

    tokio::select! {
        text = slow() => println!("slow won: {text}"),
        _ = sleep(Duration::from_millis(100)) => println!("the timer won"),
    }
}
```

```text
gave up: deadline has elapsed
the timer won
```

**What happens:** `timeout` returns `Err(Elapsed)` when the time is over; the `slow()` future is dropped, so its work stops. `select!` does the same with any set of futures: the first one that completes runs its arm, the rest are dropped. **Dropping a future is how you cancel it** in Rust.

### Errors and `?`

`?` works in async functions as usual:

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn load_user(id: u32) -> Result<String, String> {
    sleep(Duration::from_millis(10)).await;
    if id == 0 {
        Err("user id 0 does not exist".to_string())
    } else {
        Ok(format!("user-{id}"))
    }
}

async fn greet(id: u32) -> Result<String, String> {
    let name = load_user(id).await?;
    Ok(format!("Hello, {name}"))
}

#[tokio::main]
async fn main() {
    println!("{:?}", greet(7).await);
    println!("{:?}", greet(0).await);
}
```

```text
Ok("Hello, user-7")
Err("user id 0 does not exist")
```

Write `.await?`: first wait for the result, then let `?` return early on `Err`.

## 8. Locks and `.await`

A `std::sync::Mutex` guard must not live across an `.await` in a spawned task. Look at the error:

```rust
use std::sync::Mutex;
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let counter = std::sync::Arc::new(Mutex::new(0));
    let c = counter.clone();
    let handle = tokio::spawn(async move {
        let mut guard = c.lock().unwrap();
        sleep(Duration::from_millis(10)).await;
        *guard += 1;
    });
    handle.await.unwrap();
    println!("{}", counter.lock().unwrap());
}
```

```text
error: future cannot be sent between threads safely
   --> src/main.rs:9:18
    |
  9 |       let handle = tokio::spawn(async move {
    |  __________________^
    ...
    |
    = help: within `{async block@src/main.rs:9:31: 9:41}`, the trait `Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:11:42
    |
 10 |         let mut guard = c.lock().unwrap();
    |             --------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
 11 |         sleep(Duration::from_millis(10)).await;
    |                                          ^^^^^ await occurs here, with `mut guard` maybe used later
```

**What it means:** remember from section 3: variables alive across an `.await` are stored in the future. The guard would be stored in the future, and the task could be moved to another thread, but a standard `MutexGuard` must be released by the thread that took it. So it is not `Send`.

There are two fixes. Either release the guard before the `.await` (best when you can):

```rust
use std::sync::{Arc, Mutex};
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let c = Arc::clone(&counter);
    let handle = tokio::spawn(async move {
        {
            let mut guard = c.lock().unwrap();
            *guard += 1;
        } // the guard is dropped here, before the await
        sleep(Duration::from_millis(10)).await;
    });
    handle.await.unwrap();
    println!("{}", counter.lock().unwrap());
}
```

```text
1
```

Or use `tokio::sync::Mutex`, whose `lock()` is async and whose guard may be held across `.await`:

```rust
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::Mutex;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();
    for _ in 0..5 {
        let c = Arc::clone(&counter);
        handles.push(tokio::spawn(async move {
            let mut guard = c.lock().await;
            sleep(Duration::from_millis(10)).await;
            *guard += 1;
        }));
    }
    for handle in handles {
        handle.await.unwrap();
    }
    println!("{}", *counter.lock().await);
}
```

```text
5
```

> 📏 **Convention:** Use the plain `std::sync::Mutex` when the lock is held for a short, non-async piece of code. Use `tokio::sync::Mutex` only when you really need to hold the lock across an `.await`.

## 9. Testing async code

A normal `#[test]` function cannot be `async`. Tokio gives `#[tokio::test]`, which builds a runtime for each test:

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn greet(name: &str, delay_ms: u64) -> String {
    sleep(Duration::from_millis(delay_ms)).await;
    format!("Hello, {name}!")
}

#[tokio::main]
async fn main() {
    println!("{}", greet("Aziza", 10).await);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn greet_builds_the_message() {
        assert_eq!(greet("Aziza", 1).await, "Hello, Aziza!");
    }

    #[tokio::test]
    async fn greet_takes_at_least_the_delay() {
        let start = std::time::Instant::now();
        greet("Bobur", 50).await;
        assert!(start.elapsed() >= Duration::from_millis(50));
    }
}
```

`cargo test` shows `test result: ok. 2 passed; 0 failed`.

> 🧪 **Testing:** Tests with real sleeps are slow and can be flaky. Tokio can pause its clock in tests (feature `test-util`, `#[tokio::test(start_paused = true)]`), so a 10-second sleep takes no real time. Keep timing assertions generous, like `>=` in the test above, never exact.

## 10. Async or threads?

| | Threads (`std::thread`) | Async (Tokio) |
|---|---|---|
| Best for | CPU-heavy work, a few long jobs | Many jobs that mostly wait for I/O |
| Cost of one more job | a stack (megabytes of address space) + operating-system (OS) work | a small future in memory |
| Code style | normal code that may block | `async`/`.await`; never block |
| Hard parts | locks, sharing data | `Send`/`'static`, blocking by mistake, cancellation |
| Libraries | most of `std` works | needs async versions of I/O (Tokio, `async` crates) |

Neither is "better". Many real programs use both: async for the network, `spawn_blocking` or threads for heavy computation. The project of this phase ([Lesson 06.11](11-project-multithreaded-web-server.md)) uses **threads** on purpose: a thread pool is the clearest way to understand how a server shares work, and you can compare with async afterwards. Later phases come back to async: you will build a tiny runtime and use Tokio in depth.

## Common mistakes

**1. Forgetting `.await`.** The future is created and thrown away. Nothing runs:

```rust
async fn say(text: &str) {
    println!("{text}");
}

#[tokio::main]
async fn main() {
    say("hello");
    println!("end of main");
}
```

```text
warning: unused implementer of `Future` that must be used
  --> src/main.rs:11:5
   |
11 |     say("hello");
   |     ^^^^^^^^^^^^
   |
   = note: futures do nothing unless you `.await` or poll them
```

Fix: `say("hello").await;`. Never ignore this warning.

**2. Using `.await` outside an async function.**

```text
error[E0728]: `await` is only allowed inside `async` functions and blocks
 --> src/main.rs:6:22
  |
5 | fn main() {
  | --------- this is not `async`
6 |     let x = answer().await;
  |                      ^^^^^ only allowed inside `async` functions and blocks
```

Fix: make the function `async`, or use a runtime's `block_on` from normal code.

**3. Writing `async fn main()` without a runtime.**

```text
error[E0752]: `main` function is not allowed to be `async`
 --> src/main.rs:1:1
  |
1 | async fn main() {
  | ^^^^^^^^^^^^^^^ `main` function is not allowed to be `async`
```

Fix: add `#[tokio::main]` above it.

**4. Borrowing a local variable in a spawned task.**

```rust
#[tokio::main]
async fn main() {
    let name = String::from("Aziza");
    let handle = tokio::spawn(async {
        println!("hello, {name}");
    });
    handle.await.unwrap();
}
```

```text
error[E0373]: async block may outlive the current function, but it borrows `name`, which is owned by the current function
 --> src/main.rs:4:31
  |
4 |     let handle = tokio::spawn(async {
  |                               ^^^^^ may outlive borrowed value `name`
  |
help: to force the async block to take ownership of `name` (and any other referenced variables), use the `move` keyword
  |
4 |     let handle = tokio::spawn(async move {
  |                                     ++++
```

Fix: use `async move`, as the compiler says. If you still need the value afterwards, `clone()` it first.

**5. Holding a `std::sync::MutexGuard` across `.await`** — see section 8.

**6. Blocking in async code** (`std::thread::sleep`, heavy loops, blocking file reads) — see section 6. There is no compiler error here. The only symptom is that everything becomes slow. Search your async code for `std::thread::sleep` and for long loops.

## Try it yourself

1. Open `~/projects/rust/code/p06-async` and paste the first Tokio program (section 4) into `src/main.rs`. Run `cargo run`: you should see `Hello, Aziza!` after a short pause.
2. Paste the `join!` example from section 5 and run it. Confirm that the first line says about 400 ms and the second about 200 ms.
3. Change `tokio::join!` to three futures, each 200 ms. How long does it take now? (About 200 ms: the waiting overlaps.)
4. Replace `tokio::time::sleep` with `std::thread::sleep` in one of the `join!` futures (remove the `.await`). What happens to the time? Why? (It becomes about 400 ms: the blocking sleep stops the other future from running.)
5. Commit your work in your learning repository: `git add code/p06-async && git commit -m "feat(p06-async): add tokio join and spawn examples"`.

## Exercises

### Exercise 1 — Two doublers (easy)

Write `async fn double_after(ms: u64, n: i32) -> i32` that sleeps `ms` milliseconds and returns `n * 2`. In `main`, run `double_after(100, 4)` and `double_after(100, 5)` **at the same time** with `join!` and print `8 + 10 = 18`.

<details>
<summary>Hint</summary>

`let (a, b) = tokio::join!(first_future, second_future);` gives both results in a tuple.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn double_after(ms: u64, n: i32) -> i32 {
    sleep(Duration::from_millis(ms)).await;
    n * 2
}

#[tokio::main]
async fn main() {
    let (a, b) = tokio::join!(double_after(100, 4), double_after(100, 5));
    println!("{a} + {b} = {}", a + b);
}
```

```text
8 + 10 = 18
```

Why it works: both futures wait for their own 100 ms timer at the same time, so the whole program needs about 100 ms, not 200 ms.

</details>

### Exercise 2 — Fix the lock (medium)

The program in section 8 does not compile (`future cannot be sent between threads safely`). Fix it in a third way, different from the two shown: keep `std::sync::Mutex`, but do the `.await` first and take the lock only afterwards.

<details>
<summary>Hint</summary>

The problem is only that the guard is alive while the task waits. Move the `sleep(...).await` line **above** the `c.lock()` line.

</details>

<details>
<summary>Answer</summary>

```rust
use std::sync::{Arc, Mutex};
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let c = Arc::clone(&counter);
    let handle = tokio::spawn(async move {
        sleep(Duration::from_millis(10)).await;
        let mut guard = c.lock().unwrap();
        *guard += 1;
    });
    handle.await.unwrap();
    println!("{}", counter.lock().unwrap());
}
```

```text
1
```

Why it works: now no guard exists when the task pauses at `.await`, so the future is `Send`. The lock is held only for the short, non-async `*guard += 1`.

</details>

### Exercise 3 — `fetch_all` with tests (medium)

Write `async fn fetch(id: u32) -> String` that sleeps a little and returns `format!("item-{id}")`, and `async fn fetch_all(ids: Vec<u32>) -> Vec<String>` that fetches all ids **concurrently** with `tokio::spawn` but returns the results **in the same order as the ids**. Write two `#[tokio::test]` tests: one for the order (`vec![3, 1, 2]`) and one for an empty input.

<details>
<summary>Hint</summary>

First spawn all tasks and collect the `JoinHandle`s in a `Vec`. Only then loop over the handles in order and `.await` each. `fetch` takes `id` by value, so it is `'static` and can be spawned directly: `tokio::spawn(fetch(id))`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn fetch(id: u32) -> String {
    // pretend that bigger ids are slower
    sleep(Duration::from_millis(100 - 10 * id as u64)).await;
    format!("item-{id}")
}

async fn fetch_all(ids: Vec<u32>) -> Vec<String> {
    let mut handles = Vec::new();
    for id in ids {
        handles.push(tokio::spawn(fetch(id)));
    }
    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

#[tokio::main]
async fn main() {
    println!("{:?}", fetch_all(vec![1, 2, 3]).await);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn results_keep_the_order_of_the_ids() {
        let results = fetch_all(vec![3, 1, 2]).await;
        assert_eq!(results, vec!["item-3", "item-1", "item-2"]);
    }

    #[tokio::test]
    async fn empty_input_gives_empty_output() {
        assert!(fetch_all(vec![]).await.is_empty());
    }
}
```

```text
["item-1", "item-2", "item-3"]
```

`cargo test` shows `test result: ok. 2 passed`.

Why it works: all tasks start before the first `await`, so they run concurrently, although task 3 finishes first. Awaiting the handles in the original order gives results in that order.

</details>

### Exercise 4 — Give up after 100 ms (medium)

Write `async fn slow_square(n: u64) -> u64` that sleeps `50 * n` milliseconds and returns `n * n`. For `n` in `[1, 3]`, call it with a 100 ms `timeout`. Print `1 squared is 1` and `3: too slow, gave up`.

<details>
<summary>Hint</summary>

`tokio::time::timeout(duration, future).await` returns `Result<T, Elapsed>`. Use `match` on it.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::Duration;
use tokio::time::{sleep, timeout};

async fn slow_square(n: u64) -> u64 {
    sleep(Duration::from_millis(50 * n)).await;
    n * n
}

#[tokio::main]
async fn main() {
    for n in [1, 3] {
        match timeout(Duration::from_millis(100), slow_square(n)).await {
            Ok(value) => println!("{n} squared is {value}"),
            Err(_) => println!("{n}: too slow, gave up"),
        }
    }
}
```

```text
1 squared is 1
3: too slow, gave up
```

Why it works: for `n = 1` the work needs 50 ms, less than the limit. For `n = 3` it needs 150 ms, so `timeout` returns `Err` and drops the `slow_square` future.

</details>

### Exercise 5 — Watch futures interleave (hard)

Write a future `YieldN { name, left }` (like `CountDown` in section 2) that returns `Pending` `left` times and then `Ready(name)`. Print a line each time it is polled. Run two of them, `A` with 2 and `B` with 3, with `tokio::join!`. What do you notice about the printed lines?

<details>
<summary>Hint</summary>

Copy `CountDown`. Add a `name: &'static str` field and print `"{} polled, {} left"`. Remember `cx.waker().wake_by_ref()` before returning `Pending`; without it, Tokio would never poll the future again.

</details>

<details>
<summary>Answer</summary>

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct YieldN {
    name: &'static str,
    left: u32,
}

impl Future for YieldN {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.left == 0 {
            Poll::Ready(self.name)
        } else {
            println!("{} polled, {} left", self.name, self.left);
            self.left -= 1;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let (a, b) = tokio::join!(
        YieldN { name: "A", left: 2 },
        YieldN { name: "B", left: 3 },
    );
    println!("{a} and {b} are done");
}
```

A real run printed this (30 runs in a row gave the same order). For fairness, `join!` polls the futures in turn and starts each new round with the next future, so that no future is always last. This is called *round-robin*:

```text
A polled, 2 left
B polled, 3 left
B polled, 2 left
A polled, 1 left
B polled, 1 left
A and B are done
```

Why it works: the lines of `A` and `B` are mixed. `join!` polls each future in turn, and each one gives control back with `Pending`. This is **cooperative** scheduling: tasks take turns because they voluntarily return at `.await` points. It also shows why a future that never returns (blocking) starves all the others.

</details>

## Quiz

1. What does an `async fn` return when you call it? When does its body run?

<details>
<summary>Answer</summary>

It returns a future (the body is stored in it). The body runs only when the future is polled, for example when you `.await` it or give it to a runtime. Until then nothing happens.

</details>

2. What are the two possible results of `poll`? What does each mean?

<details>
<summary>Answer</summary>

`Poll::Ready(value)`: the work is done and here is the result. `Poll::Pending`: not ready yet; the future has arranged (through the waker) to be polled again when it can progress.

</details>

3. Why does Rust's standard library not run futures by itself?

<details>
<summary>Answer</summary>

Rust has no built-in runtime. Runtimes are libraries (Tokio and others), because different programs need different designs. The standard library defines only the `Future` trait and the basic types.

</details>

4. What is the difference between `join!` and `spawn`?

<details>
<summary>Answer</summary>

`join!` runs several futures concurrently inside the current task and gives all results together; it can borrow local data. `spawn` creates an independent task that may run on another thread, so its future must be `Send + 'static`.

</details>

5. Why is `std::thread::sleep` a bug inside an async function?

<details>
<summary>Answer</summary>

It blocks the whole thread, so the runtime cannot run any other task on that thread while it sleeps. Use `tokio::time::sleep(...).await`, which pauses only the task.

</details>

6. How do you cancel a future in Rust?

<details>
<summary>Answer</summary>

By dropping it. `timeout` and `select!` do it for the futures that did not finish.

</details>

7. When should you prefer threads over async?

<details>
<summary>Answer</summary>

For CPU-heavy work, for a small number of long-running jobs, or when you use blocking libraries. Async is better for many tasks that mostly wait for I/O.

</details>

## Summary

- A **future** is lazy work that will finish later; the runtime **polls** it until it returns `Ready`.
- `async fn` returns a future; `.await` pauses the current function and lets the runtime run other work.
- The compiler turns an async function into a state machine. Variables used after an `.await` live inside the future.
- Rust has no built-in runtime: use Tokio (`#[tokio::main]`, `#[tokio::test]`).
- `join!` runs futures at the same time in one task; `spawn` creates independent tasks that need `Send + 'static`.
- Never block inside async code. Use `tokio::time::sleep`, `spawn_blocking` or a thread.
- Do not hold a `std::sync::MutexGuard` across `.await`; release it first, or use `tokio::sync::Mutex`.
- Dropping a future cancels it. `timeout` and `select!` use this.
- Use async for many I/O-bound jobs and threads for CPU-bound ones.

## Sources and further reading

- [The Rust Programming Language — Chapter 17: Fundamentals of Asynchronous Programming](https://doc.rust-lang.org/book/ch17-00-async-await.html) — the official book chapters on futures, tasks and streams.
- [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/) — the official async book: more detail on `Pin`, wakers and runtimes.
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) — step-by-step guide from the Tokio project.
- [tokio on docs.rs](https://docs.rs/tokio/latest/tokio/) — documentation of the API (Application Programming Interface): `spawn`, `join!`, `select!`, `sync`, `time`.
- [std::future::Future](https://doc.rust-lang.org/std/future/trait.Future.html) — the trait definition in the standard library.

---

[🏠 Start](../START-HERE.md) · [← 06.8 Procedural macros](08-procedural-macros.md) · [📋 Phase 06](README.md) · [06.10 FFI: calling C code →](10-ffi-calling-c.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
