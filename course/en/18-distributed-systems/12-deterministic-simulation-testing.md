<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.11 Raft: log replication](11-raft-log-replication.md) · [📋 Phase 18](README.md) · [18.13 Distributed transactions and sagas →](13-distributed-transactions-and-sagas.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/12-deterministic-simulation-testing.md) <!-- nav -->

# Lesson 18.12 — Deterministic simulation testing

> **Phase 18 — Distributed systems** · Lesson 12 of 15 · ⏱ about 135 minutes

## What you will learn

- What *deterministic simulation testing* is, and why distributed systems need it: real networks make tests slow, flaky and impossible to replay.
- How the `turmoil` crate runs a whole cluster of Tokio programs in one thread, with a simulated clock and a simulated network.
- How to inject hardship: network partitions, slow and uneven message delays, and crashes followed by restarts.
- How a seed makes a run repeatable, and how a loop over many seeds finds a real retry bug.
- What simulation cannot find, and where model checking fits in.

## Before you start

- You finished [Lesson 18.11 — Raft: log replication](11-raft-log-replication.md). There you already did a small deterministic simulation by hand: the `Cluster` decided when time passed and which messages arrived.
- You know Tokio tasks, `timeout` and `sleep` ([Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md), [Lesson 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md)), TCP (Transmission Control Protocol) with Tokio ([Lesson 15.6](../15-networking/06-async-networking-with-tokio.md)), and retries with backoff and idempotency ([Lesson 18.3](03-retries-backoff-and-idempotency.md)).
- Create the package:

```bash
cd ~/projects/rust
cargo new --lib code/p18-sim
cd code/p18-sim
cargo add tokio@1 --features full
cargo add --dev turmoil@0.7
```

`--dev` puts `turmoil` under `[dev-dependencies]`: it is compiled only for tests, never into your real program.

## New words

| Word | Meaning |
|---|---|
| **DST** (Deterministic Simulation Testing) | Testing a distributed program inside a simulation where time, the network and randomness are all controlled by the test, so that every run with the same seed is exactly the same. |
| **non-determinism** | Anything that can differ between two runs of the same program: thread scheduling, real time, random numbers, the order in which network packets arrive. |
| **flaky test** | A test that sometimes passes and sometimes fails without any code change (Lesson 05.6). Real-network tests of distributed code are often flaky. |
| **seed** | The number that starts a random number generator. Same seed, same "random" choices. |
| **simulated time** | A clock that the simulation moves forward itself. Sleeping 5 seconds of simulated time takes almost no real time. |
| **host** | In `turmoil`, a simulated machine that runs a server program. It can be crashed and restarted. |
| **client** | In `turmoil`, a simulated machine that runs the test code. The simulation ends when all clients finish. |
| **partition** | In `turmoil`, cutting the link between two hosts, in both directions (or in one direction with `partition_oneway`). |
| **bounce** | Restart a crashed host: its program starts again from the beginning, with empty memory. |
| **hardship** | The word `turmoil` uses for injected problems: delays, lost messages, partitions, crashes. |
| **model checking** | A different technique: a tool explores *every* possible order of events of a small model of the system, instead of random runs. |

## 1. Why simulate?

Imagine testing the Raft cluster of the last two lessons with real processes on real sockets. To check "a partition in the middle of a write", you need to stop network traffic at just the right millisecond. The test needs real timeouts, so it takes seconds. And when it fails one time in a thousand, you cannot reproduce that one run: thread scheduling, timer precision and packet order were different.

Deterministic simulation removes every source of non-determinism:

| Source of non-determinism | In a simulation |
|---|---|
| Threads running in parallel | Everything runs on **one** thread, in an order chosen by the simulator. |
| The real clock | A **simulated clock** that only moves when the simulator moves it. |
| The network | A **simulated network** that delivers, delays, drops or blocks messages as the test decides. |
| Random numbers | Every random choice comes from a generator started from **one seed**. |

The result: a test that runs a cluster for minutes of simulated time in milliseconds of real time, can break the network in any way, and, when it fails, prints a seed that replays **the same failure** as often as you want. The FoundationDB database was famously built this way, and TigerBeetle runs a simulator around its database day and night (see the sources).

You have already done this. The `Cluster` of Lessons 18.10–18.11 was a deterministic simulation: no threads, no clock, a queue of messages, one seed. That works when you write the protocol sans-IO. The `turmoil` crate brings the same idea to **ordinary Tokio code** that uses sockets, `sleep` and `timeout`.

## 2. A server and a client under `turmoil`

`turmoil` gives you a `Sim`, a simulated world. In it you register **hosts** (server programs, written as async closures) and **clients** (the test code). Each one gets its own name, a fake IP (Internet Protocol) address, and its own single-threaded Tokio runtime whose clock is the simulated clock. `sim.run()` runs everything until all clients have finished.

The code under test must use `turmoil::net::{TcpListener, TcpStream}` instead of `tokio::net`. They have the same methods, so one `#[cfg]` switch is enough. Here is our system, a tiny key-value server with a client that retries. Replace `src/lib.rs`:

```rust
//! A tiny key-value server and a client with retries.

// Real network in the program, simulated network in the tests.
#[cfg(not(test))]
use tokio::net::{TcpListener, TcpStream};
#[cfg(test)]
use turmoil::net::{TcpListener, TcpStream};

#[cfg(test)]
mod sim;

use std::collections::HashMap;
use std::io;
use std::time::Duration;
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::time::{sleep, timeout};

pub const PORT: u16 = 7000;

/// The server's data.
#[derive(Default)]
pub struct Store {
    values: HashMap<String, i64>,
}

impl Store {
    /// Apply one request line and return the answer line.
    pub fn handle(&mut self, line: &str) -> String {
        let parts: Vec<&str> = line.split_whitespace().collect();
        match parts.as_slice() {
            ["SET", key, value] => match value.parse() {
                Ok(v) => {
                    self.values.insert(key.to_string(), v);
                    "OK".to_string()
                }
                Err(_) => "ERR not a number".to_string(),
            },
            ["GET", key] => match self.values.get(*key) {
                Some(v) => v.to_string(),
                None => "NONE".to_string(),
            },
            ["INCR", key] => self.incr(key),
            _ => "ERR unknown request".to_string(),
        }
    }

    fn incr(&mut self, key: &str) -> String {
        let v = self.values.entry(key.to_string()).or_insert(0);
        *v += 1;
        v.to_string()
    }
}

/// Serve forever: one request line and one answer line per connection.
pub async fn serve() -> io::Result<()> {
    let listener = TcpListener::bind(("0.0.0.0", PORT)).await?;
    let mut store = Store::default();
    loop {
        let (stream, _) = listener.accept().await?;
        let mut stream = BufReader::new(stream);
        let mut line = String::new();
        // A client that never finishes its line must not block us forever.
        match timeout(Duration::from_secs(1), stream.read_line(&mut line)).await {
            Ok(Ok(n)) if n > 0 => {
                let answer = store.handle(line.trim());
                let _ = stream.write_all(format!("{answer}\n").as_bytes()).await;
            }
            _ => continue, // broken or slow connection: drop it
        }
    }
}

/// Send one request and read the answer, with a time limit.
pub async fn request(host: &str, line: &str) -> io::Result<String> {
    let exchange = async {
        let stream = TcpStream::connect((host, PORT)).await?;
        let mut stream = BufReader::new(stream);
        stream.write_all(format!("{line}\n").as_bytes()).await?;
        let mut answer = String::new();
        if stream.read_line(&mut answer).await? == 0 {
            return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "no answer"));
        }
        Ok(answer.trim().to_string())
    };
    match timeout(Duration::from_millis(500), exchange).await {
        Ok(result) => result,
        Err(_) => Err(io::Error::new(io::ErrorKind::TimedOut, "timed out")),
    }
}

/// Retry with exponential backoff (Lesson 18.3): 100 ms, 200 ms, 400 ms, ...
pub async fn request_with_retry(host: &str, line: &str, attempts: u32) -> io::Result<String> {
    let mut delay = Duration::from_millis(100);
    let mut last_error = io::Error::other("no attempt made");
    for _ in 0..attempts {
        match request(host, line).await {
            Ok(answer) => return Ok(answer),
            Err(e) => last_error = e,
        }
        sleep(delay).await;
        delay *= 2;
    }
    Err(last_error)
}
```

**What happens:**

1. The two `use` lines with `#[cfg(...)]` pick the network types: `tokio::net` when the crate is built normally, `turmoil::net` when it is compiled for its own unit tests. `#[cfg(test)] mod sim;` adds the test module (written next) only in test builds.
2. `Store::handle` is the state machine: `SET key number`, `GET key` and `INCR key`. It is a plain function, easy to unit-test on its own.
3. `serve` accepts connections one by one. Each connection carries one request line and gets one answer line. The 1-second `timeout` around `read_line` protects the server from a client that connects and then never sends a full line (for example because the network broke).
4. `request` connects, writes, reads one line, all inside a 500 ms `timeout`. A closed connection without an answer (`read_line` returned 0) is an error too.
5. `request_with_retry` retries with exponential backoff, like Lesson 18.3. (Without jitter, to keep the example short. In a simulation, jitter must come from a seeded generator too.)

Now the first simulation. Create `src/sim.rs`:

```rust
use super::*;
use std::cell::RefCell;
use std::rc::Rc;
use turmoil::Builder;

/// Register the server program on the host called "server".
fn add_server(sim: &mut turmoil::Sim) {
    sim.host("server", || async {
        serve().await?;
        Ok(())
    });
}

#[test]
fn set_then_get() -> turmoil::Result {
    let mut sim = Builder::new().rng_seed(1).build();
    add_server(&mut sim);
    sim.client("client", async {
        assert_eq!(request("server", "SET apples 10").await?, "OK");
        assert_eq!(request("server", "GET apples").await?, "10");
        assert_eq!(request("server", "GET pears").await?, "NONE");
        Ok(())
    });
    sim.run()
}
```

**What happens:**

1. `add_server` registers a host called `server`. The closure returns a new future each time it is called; `turmoil` calls it again when the host is restarted. `serve().await?` converts the `io::Error` into `turmoil::Result`'s error type with `?`.
2. `Builder::new().rng_seed(1).build()` creates the world. **Always set a seed**; without one, `turmoil` picks a random seed from the operating system, and runs differ (Section 4 shows it).
3. The client is test code. `"server"` works as a host name: `turmoil` has a small built-in name lookup.
4. The test returns `turmoil::Result`, so `?` works inside the client and in the test itself.

Run `cargo test`:

```text
test sim::set_then_get ... ok
```

Nothing is real here: no port on your Mac was opened, and the whole exchange took microseconds of real time.

## 3. Partitions

Hardship can be injected from inside a client (functions such as `turmoil::partition("a", "b")`) or from outside, between simulation **steps**. `sim.step()` advances the world by one tick (1 ms of simulated time by default) and returns `true` once all clients are done. A loop around it lets the test act at exact moments. Add to `src/sim.rs`:

```rust
#[test]
fn a_retry_survives_a_partition() -> turmoil::Result {
    let mut sim = Builder::new().rng_seed(1).build();
    add_server(&mut sim);
    sim.client("client", async {
        let answer = request_with_retry("server", "SET apples 10", 6).await?;
        assert_eq!(answer, "OK");
        Ok(())
    });

    sim.partition("client", "server");
    let mut repaired = false;
    while !sim.step()? {
        if !repaired && sim.elapsed() >= Duration::from_secs(1) {
            sim.repair("client", "server");
            repaired = true;
        }
    }
    println!("done after {:?} of simulated time", sim.elapsed());
    Ok(())
}
```

**What happens:**

1. The client sends one `SET` with up to 6 attempts.
2. Before the first step, the test cuts the link between `client` and `server`. Every attempt fails: connecting through a partition is refused at once, then the client sleeps 100, 200, 400 ms…
3. In the step loop, once 1 second of simulated time has passed, the test repairs the link. The next attempt succeeds.
4. `sim.elapsed()` reports the simulated time.

Run it with output visible: `cargo test partition -- --nocapture`:

```text
running 1 test
done after 1.551s of simulated time
test sim::a_retry_survives_a_partition ... ok
```

The attempts were at about 0, 0.1, 0.3, 0.7 and 1.5 seconds (the backoff sums), so the fifth one, just after the repair, succeeded. The test still finished in a few milliseconds of real time.

> 💡 **Tip:** by default a simulation may run for 10 seconds of simulated time; after that `run()` and `step()` return an error. For longer scenarios, raise it with `Builder::simulation_duration`.

## 4. Delays, and the seed

Real networks are slow and uneven. `min_message_latency` and `max_message_latency` make every message take a random time between the two values. Now the 500 ms `timeout` of `request` will sometimes fire, and the retries will be used. Add a helper that counts 20 visits, and a test that runs it twice per seed:

```rust
/// 20 increments over a slow network. Returns the final count and the time taken.
fn count_visits(seed: u64) -> (String, Duration) {
    let mut sim = Builder::new()
        .rng_seed(seed)
        .min_message_latency(Duration::from_millis(20))
        .max_message_latency(Duration::from_millis(300))
        .build();
    add_server(&mut sim);

    let result = Rc::new(RefCell::new(String::new()));
    let out = result.clone();
    sim.client("client", async move {
        for _ in 0..20 {
            request_with_retry("server", "INCR visits", 8).await?;
        }
        *out.borrow_mut() = request_with_retry("server", "GET visits", 8).await?;
        Ok(())
    });
    sim.run().expect("the simulation failed");
    let total = result.borrow().clone();
    (total, sim.elapsed())
}

#[test]
fn the_same_seed_gives_the_same_run() {
    for seed in [1, 2, 3] {
        let first = count_visits(seed);
        let second = count_visits(seed);
        println!("seed {seed}: {first:?}");
        assert_eq!(first, second);
    }
}
```

**What happens:**

1. Every message takes between 20 and 300 ms. A request needs several messages (connect, the request, the answer), so some requests take longer than 500 ms and time out.
2. The client future must be `'static`, so it cannot borrow a local variable. It gets a clone of an `Rc<RefCell<String>>` and writes the final count into it; the test reads it after `run()`. (`Rc` is fine because everything runs on one thread.)
3. `the_same_seed_gives_the_same_run` runs every seed twice and compares the results, including the exact simulated time.

```text
seed 1: ("20", 4.421s)
seed 2: ("20", 4.041s)
seed 3: ("21", 5.13s)
```

Two things to see. First, two runs with the same seed are identical, down to the millisecond. Second, **seed 3 counted 21 visits**, although the client sent 20. You have found a bug without looking for it.

Determinism depends on the seed. Remove `.rng_seed(seed)` from `count_visits` and run the test twice. Each time it fails, with a different simulated time:

```text
seed 1: ("20", 4.85s)

thread 'sim::the_same_seed_gives_the_same_run' (6353819) panicked at src/sim.rs:78:9:
assertion `left == right` failed
```

Put the seed back.

## 5. Many seeds find the retry bug

Turn the observation into a property and test it on 50 seeds. Add:

```rust
#[test]
fn every_increment_is_counted_once() {
    for seed in 0..50 {
        let (total, _) = count_visits(seed);
        assert_eq!(total, "20", "seed {seed}");
    }
}
```

```text
---- sim::every_increment_is_counted_once stdout ----

thread 'sim::every_increment_is_counted_once' (6362447) panicked at src/sim.rs:86:9:
assertion `left == right` failed: seed 3
  left: "21"
 right: "20"
```

11 of the 50 seeds count 21. What happened in seed 3? A request reached the server, which incremented the counter and sent the answer, but the answer was slow, and the client's 500 ms timeout fired first. The client could not know that the increment had happened, so it retried, and the server counted again. This is exactly the problem of Lesson 18.3: **`INCR` is not idempotent, so retrying it is not safe**. A timeout means "I do not know", never "it did not happen".

The fix from Lesson 18.3 is an **idempotency key**: the client gives each logical operation a unique id and uses the **same** id for every retry. The server remembers the answer for each id and returns it again instead of repeating the work. In `src/lib.rs`, add a field to `Store`:

```rust
pub struct Store {
    values: HashMap<String, i64>,
    /// Request id -> the answer we gave. Used to recognise retries.
    answered: HashMap<String, String>,
}
```

and a second `INCR` arm in `handle`, right after the first one:

```rust
            ["INCR", key, id] => {
                if let Some(answer) = self.answered.get(*id) {
                    return answer.clone(); // a retry: do not count it again
                }
                let answer = self.incr(key);
                self.answered.insert(id.to_string(), answer.clone());
                answer
            }
```

In `count_visits`, send an id with every increment:

```rust
        for i in 0..20 {
            let line = format!("INCR visits client-{i}");
            request_with_retry("server", &line, 8).await?;
        }
```

Run `cargo test` again: `every_increment_is_counted_once` passes on all 50 seeds, and `the_same_seed_gives_the_same_run` now prints `("20", 5.13s)` for seed 3.

> 🔒 **Security:** the `answered` map grows with every request and is filled by clients. A client that sends millions of different ids can exhaust the server's memory. Real systems keep ids only for a limited time or a limited number per client, and authenticate clients so that one cannot use another's ids.

> ⚡ **Performance:** the 50 seeds simulate about 4 minutes of network time, and the test takes about 4 seconds of real time in a debug build. Run such tests with `--release` in CI (continuous integration) to try thousands of seeds.

## 6. Crashes

The third kind of hardship: a machine dies. `sim.crash("server")` stops the host's program and drops its connections. `sim.bounce("server")` starts the program again **from the beginning**: the closure in `add_server` is called again, and a new `serve()` starts with a new, empty `Store`. Add:

```rust
#[test]
fn a_crash_loses_an_acknowledged_write() -> turmoil::Result {
    let mut sim = Builder::new().rng_seed(1).build();
    add_server(&mut sim);
    sim.client("client", async {
        assert_eq!(request("server", "SET apples 10").await?, "OK");
        tokio::time::sleep(Duration::from_secs(2)).await;
        let answer = request_with_retry("server", "GET apples", 5).await?;
        assert_eq!(
            answer, "10",
            "the server said OK, so the write must survive"
        );
        Ok(())
    });

    let (mut crashed, mut restarted) = (false, false);
    while !sim.step()? {
        if !crashed && sim.elapsed() >= Duration::from_secs(1) {
            sim.crash("server");
            crashed = true;
        }
        if !restarted && sim.elapsed() >= Duration::from_millis(1500) {
            sim.bounce("server");
            restarted = true;
        }
    }
    Ok(())
}
```

**What happens:**

1. The client writes `apples = 10` and gets `OK`. Then it waits 2 seconds.
2. At 1 second the test crashes the server; at 1.5 seconds it restarts it.
3. At 2 seconds the client reads `apples` with retries and expects 10, because the server **acknowledged** the write.

```text
---- sim::a_crash_loses_an_acknowledged_write stdout ----

thread 'sim::a_crash_loses_an_acknowledged_write' (6362445) panicked at src/sim.rs:98:9:
assertion `left == right` failed: the server said OK, so the write must survive
  left: "NONE"
 right: "10"
```

Of course: the data lived only in memory. This is the most serious bug a storage system can have: the client was told "your write is safe", and it was lost. Two things fix it, and you know both: write the data durably to disk before answering, with a write-ahead log and `fsync` ([Lesson 17.2](../17-storage-and-databases/02-durability-and-fsync.md), [Lesson 17.4](../17-storage-and-databases/04-write-ahead-log.md)), and keep copies on other machines, so that a dead disk does not lose it either: Raft. Combining both is the project of Lesson 18.14.

For now, keep the test as **documentation of a known bug**, so the suite stays green and the bug stays visible. Replace the assertion:

```rust
        // A known bug, kept as a test: the write is gone after the restart.
        assert_eq!(answer, "NONE", "memory does not survive a crash");
```

When you fix the bug later, this test fails, and you change the assertion back.

> 🧪 **Testing:** `turmoil` can also simulate a disk, with the `unstable-fs` feature: writes stay in a pending buffer until `sync_all()`, and `crash` throws away whatever was not synced. It is marked unstable (its API, Application Programming Interface, may change), so this course does not use it, but it is the natural next step for testing a write-ahead log.

## 7. What simulation does not do

Simulation explores many **random** runs. It is very good at finding bugs that happen in one run of a hundred or a thousand. It does not prove that a bug cannot happen. Recall the Figure 8 bug of Lesson 18.11: it needs a specific sequence of elections and crashes, and random runs may never hit it.

**Model checking** is the complement. You describe a small model of the protocol (states, messages, the rules for each step), and a model checker visits **every** reachable state of that model, for example all orders in which 3 nodes can send and receive 6 messages. If a property can be broken, it finds the shortest sequence of events that breaks it. The Rust crate `stateright` is such a checker, and the TLA+ (Temporal Logic of Actions) language is widely used in industry, including for Raft itself. The price: the model is small and separate from the real code, and the number of states grows very fast. Use simulation for the real code and model checking for the core rules of a protocol.

## Common mistakes

> ⚠️ **Common mistake:** forgetting `rng_seed`. Without it, `turmoil` takes a random seed from the operating system; two runs differ, and a failure cannot be replayed (Section 4). Always pass a seed, and print it in every assertion message.

> ⚠️ **Common mistake:** using `tokio::net` (or `std::net`) in code under test. Those types open real sockets, outside the simulation; the test then fails with a confusing error or, worse, talks to a real port on your machine. Every socket in simulated code must come from `turmoil::net`.

> ⚠️ **Common mistake:** real non-determinism inside a host: `std::time::SystemTime::now()`, `rand::rng()`, `std::thread::spawn`, or iterating a `HashMap` to decide what to send. The run then depends on more than the seed. Use `tokio::time` (simulated), seeded generators, and sorted collections where order matters.

> ⚠️ **Common mistake:** expecting `cfg(test)` to switch types for integration tests in `tests/`. Those tests compile the library *without* `cfg(test)`, so the library would use `tokio::net`. That is why this lesson's simulation tests are a module inside the crate. Bigger projects use a Cargo feature (for example `--features simulation`) or make the network a trait.

## Try it yourself

1. Run `cargo test -- --nocapture` and find every line printed by the tests.
2. In `count_visits`, change the latencies to 0..10 ms. Do any seeds still count 21 without the idempotency key? (Temporarily change the client line back to `"INCR visits"` to check.) Why not?
3. Add `.enable_random_order()` to the builder in `count_visits`. It makes the simulator run hosts in a random order at each step, which explores more interleavings. Do the tests still pass?
4. In `a_retry_survives_a_partition`, lower the attempts from 6 to 4. Why does the test fail now? (Add up the backoff delays.)

## Exercises

### Exercise 1 — A timeout is not a failure (easy)

Write a test in a new module `src/exercises.rs` (add `#[cfg(test)] mod exercises;` to `src/lib.rs`, and start the file with the same `use` lines as `src/sim.rs`). Inside the client, block only the **answers**: `turmoil::partition_oneway("server", "client")`. Send `SET apples 10` with `request` and check that the error kind is `io::ErrorKind::TimedOut`. Then repair the link with `turmoil::repair_oneway("server", "client")` and check that `GET apples` returns `"10"`.

<details>
<summary>Hint</summary>

`request(...).await.unwrap_err()` gives the `io::Error`; `.kind()` gives its kind. A one-way partition from server to client lets the request through but blocks the answer, so the write happens although the client sees a timeout.

</details>

<details>
<summary>Answer</summary>

```rust
use super::*;
use std::cell::RefCell;
use std::rc::Rc;
use turmoil::Builder;

#[test]
fn a_timeout_does_not_mean_the_request_failed() -> turmoil::Result {
    let mut sim = Builder::new().rng_seed(1).build();
    sim.host("server", || async {
        serve().await?;
        Ok(())
    });
    sim.client("client", async {
        // Requests reach the server, but no answer comes back.
        turmoil::partition_oneway("server", "client");
        let error = request("server", "SET apples 10").await.unwrap_err();
        assert_eq!(error.kind(), io::ErrorKind::TimedOut);

        turmoil::repair_oneway("server", "client");
        // The write happened anyway.
        assert_eq!(request("server", "GET apples").await?, "10");
        Ok(())
    });
    sim.run()
}
```

Why it works: the server received and applied the `SET`; only its answer was lost. The client's timeout is honest ("I got no answer") but not a statement about the server. This is why Lesson 18.3 insists on idempotent retries.

</details>

### Exercise 2 — Same code, same seed, same bug (medium)

Without the idempotency key, seed 3 counted 21 visits. Explain in your own words why you can be **sure** that seed 3 will count 21 again tomorrow on another computer, while a test of the same code on a real network might pass a hundred times in a row.

<details>
<summary>Hint</summary>

List everything that decided the timing of messages in seed 3. Which of these things come from outside the simulation?

</details>

<details>
<summary>Answer</summary>

In the simulation, the message delays, the order in which hosts run and every timer come from the simulated clock and from one generator started with seed 3; nothing comes from the real machine. The same code and the same seed therefore make the same choices in the same order, and the slow answer that triggered the double count happens again. On a real network the timing depends on the operating system's scheduler, the load of the machine and real packet delays; the unlucky timing may occur once in a thousand runs and never again when you try to debug it.

</details>

### Exercise 3 — Idempotency keys do not survive a crash (hard)

In `src/exercises.rs`, write a test that runs the 20 increments **with** ids (and a 100 ms pause after each one), crashes the server at 1 second and restarts it at 1.2 seconds, then prints the final count. What number do you get with seed 1, and why can request ids alone not fix it?

<details>
<summary>Hint</summary>

Copy `count_visits` and the step loop of `a_crash_loses_an_acknowledged_write`. Use `while !sim.step().unwrap()` in a test that returns nothing.

</details>

<details>
<summary>Answer</summary>

```rust
#[test]
fn a_crash_in_the_middle_of_the_increments() {
    let mut sim = Builder::new().rng_seed(1).build();
    sim.host("server", || async {
        serve().await?;
        Ok(())
    });
    let result = Rc::new(RefCell::new(String::new()));
    let out = result.clone();
    sim.client("client", async move {
        for i in 0..20 {
            let line = format!("INCR visits client-{i}");
            request_with_retry("server", &line, 8).await?;
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
        *out.borrow_mut() = request_with_retry("server", "GET visits", 8).await?;
        Ok(())
    });
    let mut crashed = false;
    let mut restarted = false;
    while !sim.step().unwrap() {
        if !crashed && sim.elapsed() >= Duration::from_secs(1) {
            sim.crash("server");
            crashed = true;
        }
        if !restarted && sim.elapsed() >= Duration::from_millis(1200) {
            sim.bounce("server");
            restarted = true;
        }
    }
    println!("the server counted {}", result.borrow());
}
```

Run it with `cargo test crash_in_the_middle -- --nocapture`:

```text
the server counted 13
```

Why it works: about 7 increments happened before the crash. The restarted server starts with an empty `values` map **and** an empty `answered` map, so the old count is gone, and so is the memory of which ids were already handled. Idempotency keys protect against duplicates only as long as the server remembers them. To survive crashes, both the data and the `answered` map must be in durable, replicated storage: the state machine of a Raft cluster, as in the project.

</details>

## Quiz

1. Name four sources of non-determinism that a deterministic simulation controls.

<details>
<summary>Answer</summary>

Thread scheduling (one thread, simulator's order), the clock (simulated time), the network (simulated delivery, delays, partitions) and random numbers (one seed).

</details>

2. What is the difference between a `host` and a `client` in `turmoil`?

<details>
<summary>Answer</summary>

A host runs a server program; it is given as a closure that creates the future, so it can be crashed and restarted (`bounce`). A client runs the test code once; the simulation finishes when all clients are done.

</details>

3. What happens if you build a `Sim` without `rng_seed`?

<details>
<summary>Answer</summary>

`turmoil` seeds its generator from the operating system, so every run makes different random choices (for example message delays). The test is no longer reproducible.

</details>

4. Why did seed 3 count 21 visits, and what fixed it?

<details>
<summary>Answer</summary>

An answer arrived after the client's timeout, so the client retried an increment that had already happened. Giving each increment an idempotency key that is reused for retries, and letting the server return the stored answer for a known key, fixed it.

</details>

5. After `sim.crash("server")` and `sim.bounce("server")`, what is left of the server's in-memory data?

<details>
<summary>Answer</summary>

Nothing. `bounce` starts the host's program again from the beginning, so all memory (here the `Store`) starts empty. Only durable storage could survive.

</details>

6. What can model checking find that random simulation may miss, and what does it cost?

<details>
<summary>Answer</summary>

It explores every possible order of events of a small model, so it finds rare sequences (like Raft's Figure 8) for certain within the model. The cost is that the model is small and separate from the real code, and the number of states grows very fast.

</details>

## Summary

- Deterministic simulation testing runs a distributed system with one thread, a simulated clock, a simulated network and one seed, so every failure can be replayed exactly.
- `turmoil` does this for normal Tokio code: register hosts and clients in a `Sim`, use `turmoil::net` instead of `tokio::net`, and call `run()` or drive `step()` yourself.
- Hardship: `partition` / `repair` (also one-way), message latency ranges, `crash` / `bounce`.
- Always set `rng_seed`, print the seed in assertions, and loop over many seeds to test a property.
- Many seeds found a real bug: retrying a non-idempotent `INCR` after a timeout counts twice. Idempotency keys fix it, but only while the server remembers them.
- A crash showed that acknowledged writes in memory are lost: durability and replication are needed.
- Simulation finds likely bugs in real code; model checking (`stateright`, TLA+) proves small models correct.

## Sources and further reading

- [turmoil on GitHub](https://github.com/tokio-rs/turmoil) — the crate's repository, with tests that show TCP, UDP (User Datagram Protocol) and partitions, and examples for `axum` and gRPC (the remote procedure call framework of Lesson 16.9).
- [turmoil documentation](https://docs.rs/turmoil) — the API: `Builder`, `Sim`, `net`, and the hardship functions.
- [Simulation and Testing (FoundationDB)](https://apple.github.io/foundationdb/testing.html) — how the FoundationDB team built a database on deterministic simulation from day one.
- [We Put a Distributed Database in the Browser (TigerBeetle)](https://tigerbeetle.com/blog/2023-07-11-we-put-a-distributed-database-in-the-browser/) — TigerBeetle's simulator, with a visual demo of crashes and partitions.
- [Stateright](https://www.stateright.rs/) — a model checker for distributed systems written in Rust, with a book that model-checks small protocols.
- [Jepsen](https://jepsen.io/) — tests of real databases under partitions; its reports show which bugs real systems had.

---

[🏠 Start](../START-HERE.md) · [← 18.11 Raft: log replication](11-raft-log-replication.md) · [📋 Phase 18](README.md) · [18.13 Distributed transactions and sagas →](13-distributed-transactions-and-sagas.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
