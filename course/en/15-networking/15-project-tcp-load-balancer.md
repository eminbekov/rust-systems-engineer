<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.14 Proxies and load balancers](14-proxies-and-load-balancers.md) · [📋 Phase 15](README.md) · [15.16 Checkpoint →](16-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/15-project-tcp-load-balancer.md) <!-- nav -->

# Lesson 15.15 — Project: a TCP load balancer

> **Phase 15 — Networking** · Lesson 15 of 16 · ⏱ about 8 hours (4 sittings)

## What you will learn

- How to turn the 30-line proxy of Lesson 15.14 into a load balancer you could actually run: health checks, failover, connect timeouts, a connection limit and graceful shutdown.
- How to share state between the accept loop, the connection tasks and a background health checker without locks.
- How to shut a network server down cleanly with `CancellationToken`, `TaskTracker` and Unix signals.
- How to test a whole network program end to end with in-process backends, on random ports, in a fraction of a second.

## Before you start

- You finished [Lesson 15.14 — Proxies and load balancers](14-proxies-and-load-balancers.md): `copy_bidirectional`, round robin with an atomic counter, and why health checks, timeouts and limits matter.
- You know `tokio::select!`, `Semaphore` and timeouts from [Lesson 14.13 — Tokio in depth: tasks and channels](../14-concurrency-and-async/13-tokio-tasks-and-channels.md), and cancellation and graceful shutdown from [Lesson 14.14 — Cancellation, timeouts and graceful shutdown](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md).
- You know `clap` from [Lesson 05.11 — Practical crates](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md) and signals from [Lesson 11.5 — Processes and signals](../11-linux-shell-and-containers/05-processes-and-signals.md).
- Create the project package (the name was announced in Lesson 15.1) and copy the test backend from Lesson 15.14:

```bash
cd ~/projects/rust
cargo new code/p15-lb
cd code/p15-lb
mkdir -p src/bin tests
cp ../p15-proxy/src/bin/backend.rs src/bin/
cargo add tokio@1 --features full
cargo add tokio-util@0.7 --features rt
cargo add clap@4 --features derive
```

## New words

| Word | Meaning |
|---|---|
| **failover** | Sending a client to another backend when the chosen one fails. |
| **active / passive health check** | A check the load balancer makes on its own (a test connection every interval) / a conclusion drawn from real traffic (a client connection to the backend failed). |
| **connection limit** | The maximum number of client connections the program serves at once. Extra clients are refused instead of overloading everything. |
| **draining** | During shutdown: no longer accepting new connections while the open ones are allowed to finish. |
| **drain timeout** | The longest time the program waits for open connections during shutdown before it cuts them. |
| **`CancellationToken`** | A `tokio-util` type: one side calls `cancel()`, every task that waits on `cancelled()` wakes up. Used to tell background tasks to stop. |
| **`TaskTracker`** | A `tokio-util` type that counts spawned tasks and lets you `wait()` until all of them have finished. |
| **SIGINT / SIGTERM** (signal interrupt / signal terminate) | The Unix signals sent by `Ctrl+C` and by service managers such as systemd, Docker and Kubernetes when they want a program to stop (Lesson 11.5). |
| **TCP** (Transmission Control Protocol) | The reliable byte stream your load balancer forwards (Lesson 15.3). |
| **HTTP, TLS, SNI** (HyperText Transfer Protocol, Transport Layer Security, Server Name Indication) | The protocol the test backends speak; the encryption layer of Lesson 15.12; the host name a TLS client sends in its first, unencrypted message (used in a stretch goal). |

## 1. The design

The load balancer has three kinds of tasks that share two pieces of state:

```text
                         ┌──────────────────────── Pool ─────────────────────────┐
                         │ backends: [(addr, healthy: AtomicBool)], next: AtomicUsize │
                         └───────▲───────────────────────▲──────────────────────────┘
                                 │ candidates()          │ set_healthy()
  clients ──► accept loop ───────┤                       │
              │  Semaphore       │                       │
              │  (max conns)     │                health-check task
              ▼                  │                (every interval: connect to each backend)
        connection task ─────────┘
        (try healthy backends in turn, copy_bidirectional, record failures)

  shutdown signal ──► accept loop stops ──► TaskTracker waits for connection tasks
                                            (at most the drain timeout)
```

- The **pool** knows the backends and whether each one is healthy. `AtomicBool` and `AtomicUsize` let every task read and update it without a lock.
- The **accept loop** takes a permit from a `Semaphore` for every client (or refuses the client when none is left) and spawns a **connection task**.
- The connection task asks the pool for the healthy backends, starting at the next one in turn, and tries them one after another until a connection succeeds. That is round robin plus failover.
- A **health-check task** connects to every backend at a fixed interval and updates the pool, so a backend that comes back is used again.
- On **SIGINT or SIGTERM**, the accept loop stops, the listener is closed, and the program waits for the open connections (up to the drain timeout).

The decision logic sits in a library (`src/lib.rs`) with a `run` function that takes the listener and a *shutdown future* as arguments. The binary only parses the command line and passes `Ctrl+C` as that future; the tests pass a channel they control. This is what makes the tests in M6 possible.

## The project

### What you will build

```text
$ lb --backend 127.0.0.1:9001 --backend 127.0.0.1:9002
lb listening on 127.0.0.1:8080, 2 backend(s)
127.0.0.1:61334 -> 127.0.0.1:9001: 77 bytes up, 99 bytes down
127.0.0.1:61340 -> 127.0.0.1:9002: 77 bytes up, 100 bytes down
127.0.0.1:61388: cannot reach 127.0.0.1:9002: Connection refused (os error 61)
backend 127.0.0.1:9002 is DOWN
127.0.0.1:61388 -> 127.0.0.1:9001: 77 bytes up, 99 bytes down
backend 127.0.0.1:9002 is UP
shutting down, waiting for 0 connection(s)
all connections closed, bye
```

(One backend was stopped and started again during this run; the last two lines appear after `Ctrl+C`.)

### Contract

**Command line** (with `clap`; the binary is named `lb`):

```text
lb [--listen <addr>] --backend <addr> [--backend <addr> …]
   [--max-conns <n>] [--health-interval-ms <ms>] [--connect-timeout-ms <ms>] [--drain-timeout-ms <ms>]
```

| Option | Default | Meaning |
|---|---|---|
| `--listen` | `127.0.0.1:8080` | address to accept clients on |
| `--backend` | required, repeatable | one backend address per option |
| `--max-conns` | `100` | client connections served at the same time |
| `--health-interval-ms` | `1000` | time between two health-check rounds; the first round runs at start |
| `--connect-timeout-ms` | `1000` | limit for every connection attempt to a backend |
| `--drain-timeout-ms` | `5000` | how long shutdown waits for open connections |

Add `[[bin]] name = "lb"` with `path = "src/main.rs"` to `Cargo.toml`. Missing or invalid options make `clap` print its usage and exit with code 2.

**Behaviour.**

1. Every accepted client gets the **healthy** backends in round-robin order (each new client starts one position later) and is connected to the first one that accepts within the connect timeout. Then bytes are copied in both directions until both sides close.
2. A backend whose connection attempt fails or times out (from a client or from a health check) is marked **down**; a successful health check marks it **up**. Only changes are logged: `backend <addr> is DOWN` / `backend <addr> is UP`.
3. If no healthy backend accepts, the client's connection is closed and `<peer>: no healthy backend, closing` is logged.
4. If `--max-conns` clients are already being served, a new client is closed at once and `<peer>: connection limit (<n>) reached, closing` is logged.
5. After a finished connection: `<peer> -> <backend>: <up> bytes up, <down> bytes down`. After a failed attempt: `<peer>: cannot reach <backend>: <error>`.
6. On SIGINT (`Ctrl+C`) or SIGTERM: stop accepting and close the listener, log `shutting down, waiting for <n> connection(s)`, wait for the open connections, and log `all connections closed, bye`, or after the drain timeout `drain timeout: <n> connection(s) cut`. Then exit with code 0.

**Library API** (`src/lib.rs`, crate `p15_lb`) used by the binary and the tests:

```text
pub struct Config {
    pub backends: Vec<SocketAddr>,
    pub max_conns: usize,
    pub health_interval: Duration,
    pub connect_timeout: Duration,
    pub drain_timeout: Duration,
}

pub async fn run(listener: TcpListener, config: Config, shutdown: impl Future<Output = ()>);
```

`run` returns only after shutdown has finished. You decide the rest (for example a `Pool` type with its own unit tests).

**Quality.** `cargo test` passes (unit tests for the pool and the integration tests of M6), `cargo clippy --all-targets` is clean, `cargo fmt --check` is clean, a `README.md` explains usage, and there is one Git commit per milestone.

### Step-by-step plan

For every "Done when", start backends in extra terminals first: `cargo run -q --bin backend -- 9001 blue` and `cargo run -q --bin backend -- 9002 green` (the program from Lesson 15.14). Port numbers in the outputs change from run to run.

#### M1 — Arguments and one backend

**Goal:** `lb` parses the contract's options and forwards every client to the **first** backend.

**Guidance:**

1. Write the `Args` struct with `#[derive(Parser)]`. `SocketAddr` implements `FromStr`, so `clap` parses and validates addresses for you. A repeatable option is a `Vec<SocketAddr>` with `#[arg(long = "backend", required = true)]`.
2. Convert `Args` into `Config` (milliseconds into `Duration::from_millis`).
3. In `lib.rs`, write a first `run` with the accept loop and `copy_bidirectional` from Lesson 15.14. For now, ignore `shutdown` (you can pass `std::future::pending()` from `main`).

<details>
<summary>Hint</summary>

`#[arg(long, default_value_t = 100)] max_conns: usize` gives `--max-conns` with a default. `#[command(name = "lb")]` makes the usage text say `lb`. Put the binary section into `Cargo.toml` before you run `cargo run --bin lb`.

</details>

**Done when:**

```bash
cargo run -q --bin lb -- --help | head -5
cargo run -q --bin lb; echo "exit $?"
cargo run -q --bin lb -- --backend 127.0.0.1:9001      # in its own terminal
curl -s http://127.0.0.1:8080/
git add . && git commit -m "M1: forward to one backend"
```

```text
A TCP load balancer

Usage: lb [OPTIONS] --backend <BACKENDS>

Options:
error: the following required arguments were not provided:
  --backend <BACKENDS>

Usage: lb --backend <BACKENDS>

For more information, try '--help'.
exit 2
hello from blue, you are 127.0.0.1:62699
```

#### M2 — Round robin over all backends

**Goal:** clients take turns over all backends.

**Guidance:**

1. Create a `Pool` in `lib.rs` with the backends and an `AtomicUsize` counter, as `RoundRobin` in Lesson 15.14.
2. Give it a method that returns **all** backends in turn order, starting at the next position: for three backends `[1, 2, 3]`, then `[2, 3, 1]`, then `[3, 1, 2]`. In M3 you will filter this list and try its entries one after another.
3. Write a unit test for the rotation.
4. Share the pool with the connection tasks through an `Arc`.

<details>
<summary>Hint</summary>

`let start = self.next.fetch_add(1, Ordering::Relaxed);` then `(0..n).map(|i| backends[(start + i) % n])`.

</details>

**Done when:**

```bash
cargo test --lib
cargo run -q --bin lb -- --backend 127.0.0.1:9001 --backend 127.0.0.1:9002
for i in 1 2 3 4; do curl -s http://127.0.0.1:8080/; done
git commit -am "M2: round robin"
```

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
hello from blue, you are 127.0.0.1:61337
hello from green, you are 127.0.0.1:61341
hello from blue, you are 127.0.0.1:61346
hello from green, you are 127.0.0.1:61350
```

#### M3 — Health, failover and connect timeouts

**Goal:** a dead backend costs the clients nothing: they are sent to another one at once, the dead one is skipped until it comes back, and a backend that does not answer at all is given up after the connect timeout.

**Guidance:**

1. Add an `AtomicBool` "healthy" per backend (start with `true`). The rotation method now returns only healthy backends. A `set_healthy(addr, bool)` method uses `swap`, which returns the old value, so you can log only real changes.
2. Wrap every connection attempt in `tokio::time::timeout(config.connect_timeout, TcpStream::connect(addr))`. Treat a timeout like any other failure.
3. In the connection task, try the candidates in order. On failure, log it, mark the backend down and try the next. If all fail, log `no healthy backend` and return (dropping the client closes it).
4. Spawn a health-check task in `run`: `tokio::time::interval(config.health_interval)`, and on every tick try to connect to every backend (healthy or not) and record the result. Note that the first `tick()` completes at once, so the first check runs at start.
5. Extend the unit tests: a down backend is skipped; after `set_healthy(addr, true)` it is used again.

<details>
<summary>Hint</summary>

Two layers of `Result` come out of `timeout(…, connect(…)).await`: the outer one says "timed out", the inner one is the connection error. A small helper `async fn connect(addr, limit) -> io::Result<TcpStream>` that turns the timeout into `io::Error::new(io::ErrorKind::TimedOut, "connect timed out")` keeps the rest of the code simple. To test the timeout, use `192.0.2.1:9`: a documentation address where nothing ever answers.

</details>

**Done when:** with both backends and `lb` running, stop `green` (`Ctrl+C` in its terminal), send three requests, wait two seconds, start `green` again, wait two seconds and send two requests:

```bash
for i in 1 2 3; do curl -s http://127.0.0.1:8080/; done
```

```text
hello from blue, you are 127.0.0.1:61384
hello from blue, you are 127.0.0.1:61390
hello from blue, you are 127.0.0.1:61395
```

```bash
for i in 1 2; do curl -s http://127.0.0.1:8080/; done
```

```text
hello from green, you are 127.0.0.1:62021
hello from blue, you are 127.0.0.1:62026
```

The `lb` log shows the failover on the second request (no client saw an error) and both state changes:

```text
127.0.0.1:61383 -> 127.0.0.1:9001: 77 bytes up, 99 bytes down
127.0.0.1:61388: cannot reach 127.0.0.1:9002: Connection refused (os error 61)
backend 127.0.0.1:9002 is DOWN
127.0.0.1:61388 -> 127.0.0.1:9001: 77 bytes up, 99 bytes down
127.0.0.1:61394 -> 127.0.0.1:9001: 77 bytes up, 99 bytes down
backend 127.0.0.1:9002 is UP
127.0.0.1:62019 -> 127.0.0.1:9002: 77 bytes up, 100 bytes down
127.0.0.1:62024 -> 127.0.0.1:9001: 77 bytes up, 99 bytes down
```

And a backend that never answers is marked down by the first health check, after half a second:

```bash
cargo run -q --bin lb -- --backend 192.0.2.1:9 --backend 127.0.0.1:9001 --connect-timeout-ms 500
for i in 1 2; do curl -s http://127.0.0.1:8080/; done
git commit -am "M3: health checks, failover, connect timeout"
```

```text
lb listening on 127.0.0.1:8080, 2 backend(s)
backend 192.0.2.1:9 is DOWN
hello from blue, you are 127.0.0.1:62706
hello from blue, you are 127.0.0.1:62708
```

(The first two lines come from `lb`, the last two from `curl`.)

#### M4 — A connection limit

**Goal:** at most `--max-conns` clients are served at once; extra clients are refused immediately.

**Guidance:**

1. Create `Arc<Semaphore>` with `max_conns` permits in `run`.
2. In the accept loop, use `Arc::clone(&limit).try_acquire_owned()`: it never waits. If it fails, log the refusal and drop the client.
3. Move the permit (an `OwnedSemaphorePermit`) into the connection task, so it is returned when the task ends, whatever way it ends.

<details>
<summary>Hint</summary>

The permit is an RAII (Resource Acquisition Is Initialization) guard like the lease in Exercise 3 of Lesson 15.14. Calling `drop(permit)` at the end of the task makes the intent visible, but it would be returned anyway.

</details>

**Done when:** start `lb -- --backend 127.0.0.1:9001 --max-conns 1`, hold the only slot with `nc 127.0.0.1 8080` in another terminal (type nothing), and try `curl`; then end `nc` with `Ctrl+C` and try again:

```bash
curl -sS http://127.0.0.1:8080/
curl -sS http://127.0.0.1:8080/      # after nc has ended
git commit -am "M4: connection limit"
```

```text
curl: (56) Recv failure: Connection reset by peer
hello from blue, you are 127.0.0.1:62699
```

The `lb` log:

```text
127.0.0.1:62687: connection limit (1) reached, closing
127.0.0.1:62684 -> 127.0.0.1:9001: 0 bytes up, 99 bytes down
127.0.0.1:62698 -> 127.0.0.1:9001: 77 bytes up, 99 bytes down
```

(`nc` sent nothing, so 0 bytes went up; the backend's greeting went down.)

#### M5 — Graceful shutdown

**Goal:** `Ctrl+C` and SIGTERM stop the load balancer without cutting the connections that are still open, but never waiting longer than the drain timeout.

**Guidance:**

1. In `main`, build the shutdown future from `tokio::signal::ctrl_c()` and `tokio::signal::unix::signal(SignalKind::terminate())`, combined with `tokio::select!`, and pass it to `run`.
2. In `run`, pin the future (`tokio::pin!(shutdown)`) and `select!` between it and `listener.accept()` in the loop. When it completes, leave the loop and `drop(listener)`: from now on, connection attempts are refused by the operating system.
3. Spawn connection tasks with a `TaskTracker` (`tracker.spawn(…)`). After the loop: `tracker.close()`, log `tracker.len()`, then `timeout(config.drain_timeout, tracker.wait())`.
4. Stop the health-check task with a `CancellationToken`: `select!` on `token.cancelled()` and the interval tick, and call `token.cancel()` at shutdown.

<details>
<summary>Hint</summary>

`TaskTracker::wait()` only finishes after `close()` has been called **and** all tasks have ended; `close()` does not stop the tasks, it only says "no more will be added". Tasks still running when the drain timeout expires are stopped when `run` returns and the Tokio runtime shuts down.

</details>

**Done when:** start `lb -- --backend 127.0.0.1:9001`, open a connection that lasts a few seconds (`sleep 3 | nc 127.0.0.1 8080` in another terminal), and press `Ctrl+C` in the `lb` terminal while it is open. A `curl` during shutdown is refused:

```bash
curl -sS http://127.0.0.1:8080/
```

```text
curl: (7) Failed to connect to 127.0.0.1 port 8080 after 0 ms: Couldn't connect to server
```

The `lb` log, which ends by itself when `nc` finishes:

```text
lb listening on 127.0.0.1:8080, 1 backend(s)
shutting down, waiting for 1 connection(s)
127.0.0.1:62600 -> 127.0.0.1:9001: 0 bytes up, 99 bytes down
all connections closed, bye
```

With `--drain-timeout-ms 1000` and a connection that lasts five seconds (`sleep 5 | nc …`), stopped with `pkill -TERM -f target/debug/lb` (SIGTERM, as `docker stop` would send):

```text
lb listening on 127.0.0.1:8080, 1 backend(s)
shutting down, waiting for 1 connection(s)
drain timeout: 1 connection(s) cut
```

```bash
git commit -am "M5: graceful shutdown"
```

#### M6 — End-to-end tests, README, quality gate

**Goal:** tests that start real backends and the real load balancer inside the test process, on random ports, and check the whole contract.

**Guidance:**

1. In `tests/lb.rs`, write three helpers: a **test backend** on `127.0.0.1:0` that writes a one-letter name and then waits until the client closes (so a test can hold a connection open); a **dead backend** (bind a listener on port 0, remember its address, drop it); and **`start_lb(backends, max_conns)`**, which binds `127.0.0.1:0`, creates a `tokio::sync::oneshot` channel and spawns `run(listener, config, async { let _ = rx.await; })`. Return the address, the sender (the "stop button") and the `JoinHandle`. Use short intervals (100 ms health checks, 200 ms connect timeout) so tests stay fast.
2. Write four tests: connections take turns (`a, b, a, b`); a dead backend is skipped; with `max_conns = 1` and one held connection, an extra client reads 0 bytes; after the stop signal, new connections fail, `run` is not finished while a connection is held, and finishes soon after that connection is dropped.
3. Write the `README.md` (what, usage, options table, how to test) and run the quality gate.

<details>
<summary>Hint</summary>

A test helper that reads the backend's name through the load balancer:

```rust
async fn ask(lb: SocketAddr) -> (TcpStream, String) {
    let mut s = TcpStream::connect(lb).await.unwrap();
    let mut name = [0u8; 1];
    s.read_exact(&mut name).await.unwrap();
    (s, String::from_utf8_lossy(&name).into_owned())
}
```

It returns the stream too: keep it in a variable to hold the connection open, or ignore it (`ask(lb).await.1`) to close it at once. `JoinHandle::is_finished()` tells whether `run` has returned.

</details>

**Done when:**

```bash
cargo fmt --check && cargo clippy --all-targets && cargo test
git add . && git commit -m "M6: end-to-end tests and README"
```

```text
...
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
running 4 tests
test the_connection_limit_refuses_extra_clients ... ok
test connections_take_turns ... ok
test a_dead_backend_is_skipped ... ok
test shutdown_waits_for_open_connections ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.10s
```

(The first result is the two unit tests of the pool; the order of the four integration tests changes between runs. Run `cargo test` a few times: the tests must pass every time, never only sometimes.)

### Stretch goals

1. **Rise and fall thresholds.** Mark a backend down only after 2 failed checks in a row and up only after 2 successes, so one lost packet does not flip it. Log the counts.
2. **Least connections.** Replace round robin with the `LeastConnections` lease from Exercise 3 of Lesson 15.14, choosing among healthy backends only.
3. **Idle timeout.** Close connections on which no byte moved for `--idle-timeout-ms`. `copy_bidirectional` cannot do this; copy the two directions yourself with two `tokio::io::copy` calls on split halves and a shared "last activity" time.
4. **Metrics.** Count accepted, refused and failed connections and bytes per backend in atomics, and print them every 10 seconds (or serve them over HTTP with `hyper` from Lesson 15.9; Lesson 16.12 shows real metrics).
5. **TLS passthrough by SNI.** Peek at the first bytes of a TLS client hello (`TcpStream::peek`), read the SNI name, and choose a backend group by name, without decrypting anything.
6. **Run it in a container** with a Dockerfile as in [Lesson 11.13](../11-linux-shell-and-containers/13-dockerfiles-for-rust.md), and check with `docker stop` that SIGTERM drains the connections.

## Common mistakes

**1. Holding a lock across `.await`.** If the pool lived in a `Mutex<Vec<…>>` and a task held the guard while it connected to a backend, every other task would wait for that connection attempt, up to the connect timeout. Keep the state in atomics (or copy what you need out of the lock) and never `.await` while holding a `std::sync::Mutex` guard.

**2. Forgetting to give the permit back on errors.** If the permit is returned by hand at the end of the "happy path", every connection that ends in an error leaks one permit, and after `max_conns` errors the load balancer refuses everybody. Move the owned permit into the task: dropping it is automatic on every path.

**3. Shutdown that never ends.** Waiting for `tracker.wait()` without a timeout hangs forever if one client keeps its connection open. Waiting with a timeout but without closing the listener lets new clients in during the drain. Do both, in this order: stop accepting, then wait with a deadline.

**4. Flaky end-to-end tests.** Fixed ports (`8080`) collide with running programs; `sleep` calls guess how long something takes. Bind every test server to port `0`, read the real address, and wait for *events* (a reply, a closed connection, a finished task) with a `timeout` as a safety net instead of sleeping.

> 🔒 **Security:** a load balancer faces the internet, so treat it as such: run it as a normal user (binding a port below 1024 is not needed behind a container or a firewall), keep `--max-conns` finite, and keep the connect timeout short, so that slow or malicious clients and dead backends cannot exhaust its resources.

## Quiz

1. Why does a failed connection attempt from a client mark the backend down at once, instead of waiting for the next health check?

<details>
<summary>Answer</summary>

It is a passive health check: real traffic just proved that the backend does not accept connections. Marking it down at once means the next clients skip it immediately, instead of each of them paying for a failed attempt until the next active check runs.

</details>

2. Why does the connection task try **all** healthy backends in order, and why is that retry always safe here?

<details>
<summary>Answer</summary>

So that a backend that died since the last health check costs the client nothing: the next backend in the list takes over (failover). It is safe because a failed `connect` means no byte of the client's data reached any backend; nothing has happened yet that could be repeated.

</details>

3. What is the difference between `try_acquire_owned()` and `acquire_owned().await` on the semaphore, and which behaviour does the contract ask for?

<details>
<summary>Answer</summary>

`try_acquire_owned()` returns at once, with an error when no permit is free; `acquire_owned().await` waits until a permit is free. The contract asks for immediate refusal, so `try_acquire_owned()`. (Waiting would also be a valid design: then the extra clients queue instead of being refused.)

</details>

4. In which order does graceful shutdown happen, and what would go wrong if you swapped the steps?

<details>
<summary>Answer</summary>

First stop accepting and close the listener, then wait for the open connections with a deadline. If you waited first, new clients would keep arriving and the wait might never end; without a deadline, one client that never leaves could keep the program alive forever.

</details>

5. Why does `run` take the listener and a shutdown future as parameters instead of binding a port and listening for `Ctrl+C` itself?

<details>
<summary>Answer</summary>

So that tests can give it a listener on port 0 and a shutdown signal they control (a oneshot channel), while the binary passes a real port and the Unix signals. This is dependency injection: the function does not decide where its inputs come from, which makes it testable.

</details>

## Summary

- A useful L4 load balancer is a small set of cooperating tasks: an accept loop, one task per connection, and a health checker, sharing a pool of atomics.
- Round robin over healthy backends plus "try the next one if connect fails" gives failover that clients never notice.
- Every connection attempt needs a timeout; a backend that does not answer is as bad as one that refuses.
- An owned semaphore permit per connection enforces a connection limit on every code path.
- Graceful shutdown: signal → stop accepting → `TaskTracker` waits for open connections → give up at the drain timeout.
- `run(listener, config, shutdown)` makes the whole program testable end to end with in-process backends on port 0.

## Sources and further reading

- [tokio::io::copy_bidirectional](https://docs.rs/tokio/latest/tokio/io/fn.copy_bidirectional.html) — forwarding in both directions, used in every connection task.
- [tokio_util::task::TaskTracker](https://docs.rs/tokio-util/latest/tokio_util/task/task_tracker/struct.TaskTracker.html) — waiting for spawned tasks during shutdown.
- [tokio_util::sync::CancellationToken](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html) — telling background tasks to stop.
- [Tokio — Graceful Shutdown](https://tokio.rs/tokio/topics/shutdown) — the official guide to the pattern used in M5.
- [tokio::sync::Semaphore](https://docs.rs/tokio/latest/tokio/sync/struct.Semaphore.html) — `try_acquire_owned` and owned permits.
- [HAProxy — Health checks](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/health-checks/) — how a production load balancer configures checks, rise and fall thresholds.

---

[🏠 Start](../START-HERE.md) · [← 15.14 Proxies and load balancers](14-proxies-and-load-balancers.md) · [📋 Phase 15](README.md) · [15.16 Checkpoint →](16-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
