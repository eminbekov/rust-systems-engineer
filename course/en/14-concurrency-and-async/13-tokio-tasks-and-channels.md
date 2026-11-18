<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.12 Pin and self-referential futures](12-pin-and-self-referential-futures.md) · [📋 Phase 14](README.md) · [14.14 Cancellation, timeouts and graceful shutdown →](14-cancellation-timeouts-and-graceful-shutdown.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/13-tokio-tasks-and-channels.md) <!-- nav -->

# Lesson 14.13 — Tokio in depth: tasks and channels

> **Phase 14 — Concurrency and async in depth** · Lesson 13 of 16 · ⏱ about 3 hours (2 sittings)

## What you will learn

- The two Tokio runtimes (multi-thread and current-thread) and how to build one with `Builder`.
- Everything about a task's result: `JoinHandle`, `JoinError` for panics and aborts, and `JoinSet` for many tasks.
- How to run blocking or heavy work with `spawn_blocking` without freezing other tasks.
- The four Tokio channels, `mpsc`, `oneshot`, `broadcast` and `watch`, which one to choose, and how a `Semaphore` limits how many tasks work at once.
- The *actor* pattern: one task owns the state, the others send it messages.

## Before you start

- You finished [Lesson 14.12 — Pin and self-referential futures](12-pin-and-self-referential-futures.md).
- You know `tokio::spawn`, `join!`, `select!`, `mpsc`, timeouts and `#[tokio::test]` from [Lesson 06.9 — Introduction to async/await](../06-rust-advanced/09-async-await-introduction.md). This lesson goes deeper.
- You know how a runtime works inside from [Lesson 14.11](11-build-a-tiny-async-runtime.md): run queue, wakers, cooperative scheduling.
- Your code for this lesson goes in `~/projects/rust/code/p14-tokio`, a library with programs in `src/bin/`:

```bash
cd ~/projects/rust
cargo new --lib code/p14-tokio
cd code/p14-tokio
cargo add tokio@1 --features full
mkdir src/bin
```

The `full` feature turns on every part of Tokio (runtime, timers, channels, signals, I/O: input/output). That is convenient for learning; a real service lists only the features it needs.

## New words

| Word | Meaning |
|---|---|
| **multi-thread runtime** | Tokio's default runtime: a pool of worker threads (by default one per CPU (Central Processing Unit) core) that share the tasks and steal work from each other. |
| **current-thread runtime** | A Tokio runtime that runs all tasks on the one thread that called `block_on`, like your runtime in Lesson 14.11. |
| **`JoinHandle<T>`** | What `tokio::spawn` returns. Awaiting it gives `Result<T, JoinError>`. |
| **`JoinError`** | The error you get when a task panicked or was aborted (cancelled). |
| **abort** | Cancel a task from outside, with `handle.abort()`. The task is dropped at its next `.await`. |
| **detach** | Let a task run without keeping its handle. Dropping a `JoinHandle` detaches the task; it does **not** stop it. |
| **`JoinSet`** | A collection of tasks whose results you can collect in the order they finish. |
| **blocking pool** | A separate set of threads that Tokio keeps for `spawn_blocking`, so blocking work does not stop the worker threads. |
| **actor** | A task that owns some state and changes it only in reaction to messages from a channel. |
| **lagging** | A `broadcast` receiver that is so slow that old messages were overwritten before it read them. |
| **semaphore** (`Semaphore`) | A counter of permits that limits how many tasks may do something at the same time. |
| **API** (Application Programming Interface) | The functions and types a library offers to your code. |

## 1. Two kinds of runtime

`#[tokio::main]` from Lesson 06.9 builds a **multi-thread** runtime. You can build both kinds by hand with `tokio::runtime::Builder`. Create `src/bin/flavors.rs`:

```rust
use std::time::Duration;

async fn show_threads(label: &str) {
    let mut handles = Vec::new();
    for id in 0..4 {
        handles.push(tokio::spawn(async move {
            tokio::time::sleep(Duration::from_millis(10)).await;
            let thread = std::thread::current();
            format!(
                "task {id} ran on {:?} {:?}",
                thread.name().unwrap_or("?"),
                thread.id()
            )
        }));
    }
    println!("{label}:");
    for handle in handles {
        println!("  {}", handle.await.unwrap());
    }
}

fn main() {
    let multi = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(2)
        .enable_all()
        .build()
        .unwrap();
    multi.block_on(show_threads("multi-thread runtime, 2 workers"));

    let single = tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap();
    single.block_on(show_threads("current-thread runtime"));
}
```

`cargo run --bin flavors` (which task lands on which worker changes between runs):

```text
multi-thread runtime, 2 workers:
  task 0 ran on "tokio-rt-worker" ThreadId(3)
  task 1 ran on "tokio-rt-worker" ThreadId(2)
  task 2 ran on "tokio-rt-worker" ThreadId(3)
  task 3 ran on "tokio-rt-worker" ThreadId(3)
current-thread runtime:
  task 0 ran on "main" ThreadId(1)
  task 1 ran on "main" ThreadId(1)
  task 2 ran on "main" ThreadId(1)
  task 3 ran on "main" ThreadId(1)
```

**What happens:**

1. `new_multi_thread().worker_threads(2)` starts two worker threads, named `tokio-rt-worker`. The four tasks are spread over both (`ThreadId(2)` and `ThreadId(3)`).
2. `enable_all()` turns on the timer and the I/O reactor. Without it, `sleep` would panic, because there is no timer.
3. `new_current_thread()` creates no threads at all: every task runs on the thread that called `block_on`, here `main`. This is your runtime from Lesson 14.11, grown up.
4. `block_on` runs a future to the end, like `#[tokio::main]` does for you.

The attribute has the same choices: `#[tokio::main(flavor = "current_thread")]`, or `#[tokio::main(worker_threads = 2)]`. Which one to use?

| | multi-thread | current-thread |
|---|---|---|
| uses | all CPU cores | one thread |
| tasks must be `Send` | yes (they move between threads) | `tokio::spawn` still requires `Send`; `LocalSet` allows non-`Send` tasks |
| good for | servers, most programs | tests, small tools, embedding into a program that has its own threads |

`#[tokio::test]` uses the **current-thread** runtime by default. Keep that in mind: a test with blocking code may hang where the real program only slows down.

## 2. Tasks and their results

A `JoinHandle<T>` is a future. Awaiting it gives `Result<T, JoinError>`: `Ok` with the task's value, or `Err` if the task panicked or was aborted. Create `src/bin/join_errors.rs`:

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    let ok = tokio::spawn(async { 6 * 7 });
    let panics = tokio::spawn(async {
        let numbers: Vec<u32> = Vec::new();
        numbers[0] // index out of bounds: the task panics
    });
    let slow = tokio::spawn(async {
        tokio::time::sleep(Duration::from_secs(10)).await;
        "never printed"
    });
    slow.abort();

    println!("ok:     {:?}", ok.await);
    match panics.await {
        Ok(value) => println!("value {value}"),
        Err(error) => println!("panics: is_panic = {}", error.is_panic()),
    }
    match slow.await {
        Ok(text) => println!("{text}"),
        Err(error) => println!("slow:   is_cancelled = {}", error.is_cancelled()),
    }
}
```

```text

thread 'tokio-rt-worker' (5675703) panicked at src/bin/join_errors.rs:8:16:
index out of bounds: the len is 0 but the index is 0
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
ok:     Ok(42)
panics: is_panic = true
slow:   is_cancelled = true
```

**What happens:**

1. The panic message is printed by the worker thread, but the program does **not** crash. Tokio catches the panic and stores it in the task's result, so one bad task cannot take down a whole server.
2. `panics.await` gives `Err(JoinError)` with `is_panic() == true`. (`error.into_panic()` would give the panic value, if you want to re-raise it.)
3. `slow.abort()` asks Tokio to cancel the task. At its next `.await` (here the `sleep`), the task's future is dropped. Its handle then gives `Err` with `is_cancelled() == true`. Dropping a future is how all cancellation works in Rust; Lesson 14.14 builds on it.

When you start many tasks, collecting their handles in a `Vec` and awaiting them in order is slow if the first one is the slowest. A **`JoinSet`** gives results in the order the tasks **finish**. Create `src/bin/join_set.rs`:

```rust
use std::time::Duration;
use tokio::task::JoinSet;

async fn fetch(id: u64) -> (u64, u64) {
    // Pretend to load something; later ids are faster.
    tokio::time::sleep(Duration::from_millis(100 - id * 20)).await;
    (id, id * 10)
}

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();
    for id in 1..=4 {
        set.spawn(fetch(id));
    }
    let mut total = 0;
    while let Some(result) = set.join_next().await {
        let (id, value) = result.unwrap();
        println!("finished: id {id} -> {value}");
        total += value;
    }
    println!("total = {total}");
}
```

```text
finished: id 4 -> 40
finished: id 3 -> 30
finished: id 2 -> 20
finished: id 1 -> 10
total = 100
```

`join_next()` returns `Some(Result<T, JoinError>)` for each finished task and `None` when the set is empty. One more property makes `JoinSet` safer than loose handles: **when a `JoinSet` is dropped, all its tasks are aborted**. A loose `JoinHandle` that is dropped only detaches its task, which keeps running in the background.

## 3. Blocking work: `spawn_blocking`

From Lesson 14.11 you know that scheduling is cooperative: a task that does not reach an `.await` keeps its thread. Here is what that looks like in Tokio, on the current-thread runtime where it is easiest to see. Create `src/bin/blocking.rs`:

```rust
use std::time::{Duration, Instant};

/// Stands for heavy, blocking work: a long computation or a blocking library call.
fn heavy_work() -> u64 {
    std::thread::sleep(Duration::from_millis(300));
    (1..=1_000_u64).sum()
}

async fn ticker(start: Instant) {
    for i in 1..=3 {
        tokio::time::sleep(Duration::from_millis(100)).await;
        println!("  tick {i} at {} ms", start.elapsed().as_millis());
    }
}

#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("blocking inside a task:");
    let start = Instant::now();
    let ticks = tokio::spawn(ticker(start));
    tokio::task::yield_now().await; // let the ticker start
    let sum = heavy_work(); // BAD: blocks the only runtime thread
    println!("  sum = {sum} at {} ms", start.elapsed().as_millis());
    ticks.await.unwrap();

    println!("with spawn_blocking:");
    let start = Instant::now();
    let ticks = tokio::spawn(ticker(start));
    let sum = tokio::task::spawn_blocking(heavy_work).await.unwrap();
    println!("  sum = {sum} at {} ms", start.elapsed().as_millis());
    ticks.await.unwrap();
}
```

```text
blocking inside a task:
  sum = 500500 at 303 ms
  tick 1 at 303 ms
  tick 2 at 405 ms
  tick 3 at 508 ms
with spawn_blocking:
  tick 1 at 100 ms
  tick 2 at 202 ms
  sum = 500500 at 303 ms
  tick 3 at 305 ms
```

**What happens:**

1. In the first part, `heavy_work()` runs directly inside `main`'s task. `std::thread::sleep` blocks the only runtime thread for 300 ms. The ticker should print at 100 ms, but it cannot run: its first tick appears at 303 ms, and every later tick is late too.
2. In the second part, `spawn_blocking(heavy_work)` runs the function on a thread of the **blocking pool**, and returns a `JoinHandle`. The runtime thread is free, so the ticker prints on time while the work is going on.
3. `spawn_blocking` takes a normal closure or function, not a future: the code inside may block as much as it wants.

Use `spawn_blocking` for: blocking libraries (some database or file APIs), `std::fs` calls in a hot loop, and computations longer than about 100 microseconds. For heavy parallel computations, a `rayon` thread pool ([Lesson 14.4](04-thread-pools-scoped-threads-and-rayon.md)) is often better; it is built for CPU work.

## 4. Four kinds of channels

You know `mpsc` from Lesson 06.9: many senders, one receiver, bounded. Tokio has three more, each for a different shape of communication:

| Channel | Senders → receivers | Each message reaches | Typical use |
|---|---|---|---|
| `mpsc` | many → one | the one receiver | a queue of work or commands; bounded capacity gives backpressure |
| `oneshot` | one → one | the receiver, exactly once | the answer to one request |
| `broadcast` | many → many | **every** receiver | events that all subscribers must see (chat messages, notifications) |
| `watch` | many → many | receivers see only the **latest** value | configuration, status, a "shutdown now" flag |

Create `src/bin/channels.rs`:

```rust
use std::time::Duration;
use tokio::sync::{broadcast, oneshot, watch};

#[tokio::main]
async fn main() {
    // oneshot: exactly one value, from one sender to one receiver.
    let (reply_tx, reply_rx) = oneshot::channel::<u32>();
    tokio::spawn(async move {
        reply_tx.send(42).unwrap();
    });
    println!("oneshot got {}", reply_rx.await.unwrap());

    // broadcast: every receiver gets every message.
    let (news_tx, mut first) = broadcast::channel::<String>(16);
    let mut second = news_tx.subscribe();
    news_tx.send("hello".to_string()).unwrap();
    news_tx.send("world".to_string()).unwrap();
    for _ in 0..2 {
        let a = first.recv().await.unwrap();
        let b = second.recv().await.unwrap();
        println!("broadcast: first got {a}, second got {b}");
    }

    // watch: only the latest value; receivers see changes.
    let (config_tx, mut config_rx) = watch::channel(1_u32);
    let watcher = tokio::spawn(async move {
        while config_rx.changed().await.is_ok() {
            println!("watch: config is now {}", *config_rx.borrow_and_update());
        }
        println!("watch: sender is gone");
    });
    for level in 2..=3 {
        tokio::time::sleep(Duration::from_millis(10)).await;
        config_tx.send(level).unwrap();
    }
    tokio::time::sleep(Duration::from_millis(10)).await;
    drop(config_tx);
    watcher.await.unwrap();
}
```

```text
oneshot got 42
broadcast: first got hello, second got hello
broadcast: first got world, second got world
watch: config is now 2
watch: config is now 3
watch: sender is gone
```

**What happens:**

1. `oneshot::channel` gives a sender whose `send` takes `self`: it can be used once. The receiver is itself a future; `.await` gives `Ok(value)`, or `Err` if the sender was dropped without sending.
2. `broadcast::channel(16)` keeps up to 16 messages. Every receiver (the first one, and each one made with `subscribe()`) gets its own copy of every message, so the message type must be `Clone`.
3. `watch::channel(1)` stores one value. `changed().await` waits until a new value is sent, and `borrow_and_update()` reads it and marks it as seen. If the sender sends 2 and 3 very quickly, a slow receiver may see only 3: `watch` is for "what is the current value", not for "every value".
4. When the `watch` sender is dropped, `changed()` returns `Err`, and the loop ends. Every Tokio channel works this way: when the other side is gone, you get an error or `None`, never a hang.

What happens when a `broadcast` receiver is too slow? The channel does not wait for it (that would let one slow subscriber stop everybody); it overwrites the oldest messages. With capacity 2 and five messages sent before the receiver reads anything, its next `recv()` returns `Err(RecvError::Lagged(3))`: "you missed 3 messages". The following calls return 4 and 5, the oldest messages that are still there, and then `Err(RecvError::Closed)` once all senders are gone.

> ⚡ **Performance:** `mpsc::unbounded_channel()` never makes the sender wait, which feels convenient. But if the receiver is slower than the senders, the queue grows until the process runs out of memory. Prefer a bounded `mpsc::channel(n)`: when it is full, `send().await` waits, and the slow consumer automatically slows down the producers. That is **backpressure**, and Lesson 14.15 is built on it.

**Limiting work, not messages: `Semaphore`.** A bounded channel limits how many *messages* wait. Sometimes you must limit how many *tasks* do something at the same time: at most 2 downloads, at most 100 open connections. `tokio::sync::Semaphore::new(n)` holds `n` **permits**. `acquire().await` waits until a permit is free and returns it as a guard; dropping the guard gives the permit back. On an `Arc<Semaphore>`, `acquire_owned()` returns a permit that owns its `Arc`, so it can move into a spawned task. `try_acquire()` never waits: it fails at once when no permit is left. Create `src/bin/semaphore.rs`:

```rust
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::Semaphore;
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let limit = Arc::new(Semaphore::new(2)); // at most 2 downloads at the same time
    let start = Instant::now();
    let mut set = JoinSet::new();
    for id in 1..=5 {
        let limit = Arc::clone(&limit);
        set.spawn(async move {
            // Waits until one of the 2 permits is free; the permit moves into the task.
            let _permit = limit.acquire_owned().await.expect("semaphore is open");
            tokio::time::sleep(Duration::from_millis(100)).await; // the "download"
            println!("download {id} done at {} ms", start.elapsed().as_millis());
        }); // `_permit` is dropped here, and the next waiting task gets it
    }
    while set.join_next().await.is_some() {}

    // try_acquire never waits: it fails at once when no permit is left.
    let _first = limit.try_acquire().expect("2 permits are free again");
    let _second = limit.try_acquire().expect("one permit is still free");
    println!("third try_acquire: {:?}", limit.try_acquire().err());
}
```

```text
download 2 done at 102 ms
download 1 done at 102 ms
download 5 done at 205 ms
download 4 done at 206 ms
download 3 done at 308 ms
third try_acquire: Some(NoPermits)
```

Five tasks start at once, but only two hold a permit at any moment, so they finish in groups: two, two, one, about 100 ms apart. Which ids come first changes from run to run; the pattern does not. A permit is returned however the task ends, even by a panic or an abort, because it is returned in `Drop`. Lessons 15.6 and 15.15 use a semaphore to limit connections.

## 5. The actor pattern

Many tasks need to change one shared thing, for example counters. You could use `Arc<Mutex<HashMap<…>>>`. The **actor** pattern is another way: one task owns the map, and everybody else sends it *commands* through an `mpsc` channel. Each command carries a `oneshot` sender for the answer. No lock is needed, because only one task ever touches the map.

Put this into `src/lib.rs`:

```rust
//! An actor: one task owns the state, others send it commands.

use std::collections::HashMap;
use tokio::sync::{mpsc, oneshot};

/// The messages the actor understands.
enum Command {
    Increment {
        key: String,
        reply: oneshot::Sender<u64>,
    },
    Get {
        key: String,
        reply: oneshot::Sender<Option<u64>>,
    },
}

/// The actor itself: it owns the map, nobody else can touch it.
async fn run_counter(mut commands: mpsc::Receiver<Command>) {
    let mut counts: HashMap<String, u64> = HashMap::new();
    while let Some(command) = commands.recv().await {
        match command {
            Command::Increment { key, reply } => {
                let count = counts.entry(key).or_insert(0);
                *count += 1;
                let _ = reply.send(*count); // the caller may be gone; that is fine
            }
            Command::Get { key, reply } => {
                let _ = reply.send(counts.get(&key).copied());
            }
        }
    }
    // All handles were dropped: the channel is closed, the actor ends.
}

/// A cheap, cloneable handle that talks to the actor.
#[derive(Clone)]
pub struct CounterHandle {
    commands: mpsc::Sender<Command>,
}

impl CounterHandle {
    /// Starts the actor task. Must be called inside a Tokio runtime.
    pub fn spawn() -> Self {
        let (commands, receiver) = mpsc::channel(32);
        tokio::spawn(run_counter(receiver));
        CounterHandle { commands }
    }

    pub async fn increment(&self, key: &str) -> u64 {
        let (reply, response) = oneshot::channel();
        let command = Command::Increment {
            key: key.to_string(),
            reply,
        };
        self.commands.send(command).await.expect("actor is gone");
        response.await.expect("actor dropped the reply")
    }

    pub async fn get(&self, key: &str) -> Option<u64> {
        let (reply, response) = oneshot::channel();
        let command = Command::Get {
            key: key.to_string(),
            reply,
        };
        self.commands.send(command).await.expect("actor is gone");
        response.await.expect("actor dropped the reply")
    }
}
```

**What happens:**

1. `Command` is an `enum` with one variant per operation. Each variant that needs an answer carries a `oneshot::Sender` for it.
2. `run_counter` is the actor: a loop over incoming commands. It is the only code that can see `counts`.
3. `CounterHandle` hides the channels. `increment` creates a `oneshot`, sends the command, then awaits the answer. For the caller it looks like a normal async method.
4. `#[derive(Clone)]` clones only the `mpsc::Sender`, so every task can have its own handle. When the last handle is dropped, `recv()` returns `None`, and the actor ends by itself.
5. `let _ = reply.send(…)` ignores the error that happens when the caller stopped waiting (for example after a timeout). The actor must not crash because one caller left.

Use it from `src/bin/actor.rs`:

```rust
use p14_tokio::CounterHandle;

#[tokio::main]
async fn main() {
    let counter = CounterHandle::spawn();
    let mut handles = Vec::new();
    for worker in 0..3 {
        let counter = counter.clone();
        handles.push(tokio::spawn(async move {
            for _ in 0..100 {
                counter.increment("visits").await;
            }
            println!("worker {worker} finished");
        }));
    }
    for handle in handles {
        handle.await.unwrap();
    }
    println!("visits = {:?}", counter.get("visits").await);
}
```

```text
worker 2 finished
worker 0 finished
worker 1 finished
visits = Some(300)
```

The order of the "finished" lines changes from run to run; the total is always 300. Three tasks on several threads, no lock, no lost update: the channel puts the commands in a line, and the actor handles them one at a time.

> 🧪 **Testing:** An actor is easy to test: create a handle in a `#[tokio::test]`, call its methods, and check the answers. A good concurrency test starts many tasks with clones of the handle and checks that the final count is exact. Exercise 2 writes both kinds of test.

When to choose which: a `Mutex` is simpler for small, quick changes to shared data. An actor is better when the state lives long, the operations are many or slow, or the state includes things like a connection that should have one owner.

## Common mistakes

**1. Thinking that dropping a `JoinHandle` stops the task.** It does not: the task is detached and keeps running. To stop it, call `abort()`, or keep tasks in a `JoinSet`, which aborts them when it is dropped.

**2. Blocking inside a task.** `std::thread::sleep`, a long loop or a blocking library call keeps a worker thread busy (Section 3). On the current-thread runtime, and in `#[tokio::test]`, it stops everything. Use `spawn_blocking`.

**3. Using `watch` when every value matters.** A `watch` receiver sees the latest value, and can skip values in between. For "every event" use `mpsc` (one consumer) or `broadcast` (many consumers).

**4. Ignoring `Lagged` from `broadcast`.** `recv()` can return `Err(Lagged(n))`. If your loop treats every `Err` as "closed" and breaks, a short burst of messages ends your subscriber. Match `broadcast::error::RecvError::Lagged(missed)` and `RecvError::Closed` separately.

**5. `unwrap()` on a reply that may be gone.** In an actor, `reply.send(x).unwrap()` panics the actor when one caller gave up waiting. Ignore that error (`let _ =`), because it is not the actor's problem.

## Try it yourself

1. Create `code/p14-tokio` and run `flavors`, `join_errors` and `join_set`.
2. In `flavors.rs`, change `worker_threads(2)` to `worker_threads(1)` and predict the thread ids.
3. In `blocking.rs`, change the main attribute to `#[tokio::main(worker_threads = 2)]` and run again. The ticker in the first part is now on time, because the second worker thread runs it. Blocking still costs you a whole worker.
4. Add the actor to `src/lib.rs`, run `actor`, and add the tests from Exercise 2.
5. Run `cargo clippy --all-targets` and `cargo fmt`, then commit: `git add code/p14-tokio && git commit -m "p14-tokio: tasks, channels and an actor"`.

## Exercises

### Exercise 1 — Squares with a `JoinSet` (easy)

Spawn one task per number 1 to 5 into a `JoinSet`; each task returns the square of its number. Collect the results with `join_next` and print their sum.

<details>
<summary>Hint</summary>

`set.spawn(async move { n * n })` needs `move`, so each task owns its own `n`. The results come in finishing order, but addition does not care about order.

</details>

<details>
<summary>Answer</summary>

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();
    for n in 1..=5_u64 {
        set.spawn(async move { n * n });
    }
    let mut sum = 0;
    while let Some(result) = set.join_next().await {
        sum += result.unwrap();
    }
    println!("sum of squares = {sum}");
}
```

```text
sum of squares = 55
```

Why it works: `join_next` returns each task's `Result` as it finishes and `None` when no tasks are left, so the loop ends by itself. `unwrap` turns a `JoinError` (panic or abort) into a panic here, which is fine for a small program.

</details>

### Exercise 2 — Teach the actor `Reset` (medium)

Add a command `Reset { key }` that removes a key from the map, and a method `reset(&self, key)`. It needs no answer. Write two `#[tokio::test]`s: ten tasks that each call `increment("hits")` 100 times give `Some(1000)`; after `reset("a")`, `get("a")` is `None` and the next `increment("a")` returns 1.

<details>
<summary>Hint</summary>

A command without an answer has no `oneshot`. Why can the test call `get` right after `reset` and be sure that the reset already happened? Think about the order of messages in one `mpsc` channel. For ten tasks, `JoinSet` has `join_all().await`, which waits for all of them.

</details>

<details>
<summary>Answer</summary>

The new variant, the new `match` arm and the new method:

```rust
    Reset {
        key: String,
    },
```

```rust
            Command::Reset { key } => {
                counts.remove(&key);
            }
```

```rust
    pub async fn reset(&self, key: &str) {
        let command = Command::Reset {
            key: key.to_string(),
        };
        self.commands.send(command).await.expect("actor is gone");
    }
```

The tests, at the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn many_tasks_never_lose_an_update() {
        let counter = CounterHandle::spawn();
        let mut tasks = tokio::task::JoinSet::new();
        for _ in 0..10 {
            let counter = counter.clone();
            tasks.spawn(async move {
                for _ in 0..100 {
                    counter.increment("hits").await;
                }
            });
        }
        tasks.join_all().await;
        assert_eq!(counter.get("hits").await, Some(1000));
    }

    #[tokio::test]
    async fn reset_forgets_a_key() {
        let counter = CounterHandle::spawn();
        counter.increment("a").await;
        counter.reset("a").await;
        assert_eq!(counter.get("a").await, None);
        assert_eq!(counter.increment("a").await, 1);
    }
}
```

`cargo test` shows `test result: ok. 2 passed; 0 failed` for the library. Why it works: one `mpsc` channel delivers messages from one sender in the order they were sent. The test sends `Reset` and then `Get` through the same handle, so the actor handles them in that order, even though `reset` does not wait for an answer.

</details>

### Exercise 3 — A three-stage pipeline (hard)

Write `async fn pipeline(lines: Vec<String>) -> (u64, usize)`: stage 1 (a task) sends the lines into a bounded `mpsc` channel of capacity 4; stage 2 (a task) parses each trimmed line as `u64`, sends good numbers into a second bounded channel and counts bad lines; stage 3 (the function itself) sums the numbers. The function returns the sum and the number of bad lines. Run it with the lines `"1"` to `"100"` plus `"oops"` and `""`. Write two tests: `["1", "x", " 2 "]` gives `(3, 1)`; and a test that shows that a full channel of capacity 1 rejects `try_send` with `TrySendError::Full`.

<details>
<summary>Hint</summary>

Stage 2 knows the number of bad lines only at the end: send it through a `oneshot`. Each stage ends when its input channel returns `None`, which happens when the previous stage drops its sender. That is how "the end of the data" flows through the pipeline without any special message.

</details>

<details>
<summary>Answer</summary>

```rust
use tokio::sync::{mpsc, oneshot};

/// Three stages connected by bounded channels:
/// lines -> parse -> sum. Returns (sum, number of bad lines).
async fn pipeline(lines: Vec<String>) -> (u64, usize) {
    let (line_tx, mut line_rx) = mpsc::channel::<String>(4);
    let (number_tx, mut number_rx) = mpsc::channel::<u64>(4);
    let (bad_tx, bad_rx) = oneshot::channel::<usize>();

    // Stage 1: produce lines.
    tokio::spawn(async move {
        for line in lines {
            if line_tx.send(line).await.is_err() {
                break; // the next stage is gone
            }
        }
    });

    // Stage 2: parse; count what cannot be parsed.
    tokio::spawn(async move {
        let mut bad = 0;
        while let Some(line) = line_rx.recv().await {
            match line.trim().parse::<u64>() {
                Ok(number) => {
                    if number_tx.send(number).await.is_err() {
                        break;
                    }
                }
                Err(_) => bad += 1,
            }
        }
        let _ = bad_tx.send(bad);
    });

    // Stage 3: sum, right here.
    let mut sum = 0;
    while let Some(number) = number_rx.recv().await {
        sum += number;
    }
    (sum, bad_rx.await.unwrap())
}

#[tokio::main]
async fn main() {
    let mut lines: Vec<String> = (1..=100).map(|n| n.to_string()).collect();
    lines.push("oops".to_string());
    lines.push("".to_string());
    let (sum, bad) = pipeline(lines).await;
    println!("sum = {sum}, bad lines = {bad}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn sums_good_lines_and_counts_bad_ones() {
        let lines = vec!["1".to_string(), "x".to_string(), " 2 ".to_string()];
        assert_eq!(pipeline(lines).await, (3, 1));
    }

    #[tokio::test]
    async fn a_full_channel_rejects_try_send() {
        let (tx, _rx) = mpsc::channel::<u32>(1);
        assert!(tx.try_send(1).is_ok());
        assert!(matches!(
            tx.try_send(2),
            Err(mpsc::error::TrySendError::Full(2))
        ));
    }
}
```

```text
sum = 5050, bad lines = 2
```

Run the tests of a program in `src/bin/` with `cargo test --bin pipeline`. Why it works: each stage owns the sender of the next channel, so when a stage finishes, its sender is dropped and the next stage's `recv()` returns `None`. With capacity 4, stage 1 can never be more than a few lines ahead of stage 2: the channels give backpressure. `try_send` never waits, so on a full channel it returns `Full` with the value inside, which makes backpressure visible in a test.

</details>

## Quiz

1. What is the difference between the multi-thread and the current-thread runtime?

<details>
<summary>Answer</summary>

The multi-thread runtime has a pool of worker threads that share and steal tasks, so it uses all cores. The current-thread runtime runs every task on the thread that called `block_on`. `#[tokio::main]` uses multi-thread; `#[tokio::test]` uses current-thread.

</details>

2. A spawned task panics. What happens to the program, and how do you find out?

<details>
<summary>Answer</summary>

The program keeps running: Tokio catches the panic (the message is still printed). Awaiting the task's `JoinHandle` gives `Err(JoinError)` with `is_panic() == true`.

</details>

3. You drop a `JoinHandle`. Does the task stop? What about dropping a `JoinSet`?

<details>
<summary>Answer</summary>

Dropping a `JoinHandle` only detaches the task; it keeps running. Dropping a `JoinSet` aborts all tasks in it.

</details>

4. When do you use `spawn_blocking`?

<details>
<summary>Answer</summary>

For code that blocks or computes for a long time without `.await`: blocking libraries, blocking file or database calls, long computations. It runs on the blocking pool, so the worker threads stay free for other tasks.

</details>

5. Which channel would you use for: a queue of jobs; the answer to one request; a "config changed" notification; chat messages for all connected users?

<details>
<summary>Answer</summary>

A bounded `mpsc` for the job queue, a `oneshot` for one answer, `watch` for the latest configuration, and `broadcast` for messages that every user must receive.

</details>

## Summary

- Tokio has a multi-thread runtime (default for `#[tokio::main]`) and a current-thread runtime (default for `#[tokio::test]`); `Builder` creates either one.
- Awaiting a `JoinHandle` gives `Result<T, JoinError>`; panics are caught and reported, `abort()` cancels a task at its next `.await`.
- Dropping a `JoinHandle` detaches the task; dropping a `JoinSet` aborts its tasks; `join_next` returns results in finishing order.
- Blocking code inside a task stops other tasks; run it with `spawn_blocking`.
- `mpsc` is a queue (bounded = backpressure), `oneshot` one answer, `broadcast` every message to every receiver (slow receivers get `Lagged`), `watch` the latest value.
- An actor owns its state and handles commands from a channel, with `oneshot` replies; it needs no locks and is easy to test.

## Sources and further reading

- [Tokio tutorial — Spawning](https://tokio.rs/tokio/tutorial/spawning) — tasks, `'static` and `Send` bounds.
- [Tokio tutorial — Channels](https://tokio.rs/tokio/tutorial/channels) — the four channels and a request/response example with `oneshot`.
- [tokio::task module](https://docs.rs/tokio/1.54.1/tokio/task/index.html) — `spawn`, `JoinHandle`, `JoinSet`, `spawn_blocking`, `yield_now`, cooperative scheduling.
- [tokio::sync module](https://docs.rs/tokio/1.54.1/tokio/sync/index.html) — all channels and synchronisation types, with a guide to choosing.
- [tokio::runtime::Builder](https://docs.rs/tokio/1.54.1/tokio/runtime/struct.Builder.html) — every runtime option.
- [Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/) — by Alice Ryhl, a Tokio maintainer; the actor pattern in more depth.

---

[🏠 Start](../START-HERE.md) · [← 14.12 Pin and self-referential futures](12-pin-and-self-referential-futures.md) · [📋 Phase 14](README.md) · [14.14 Cancellation, timeouts and graceful shutdown →](14-cancellation-timeouts-and-graceful-shutdown.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
