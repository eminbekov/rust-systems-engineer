<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.8 Overload, limits and backpressure](08-overload-limits-and-backpressure.md) · [📋 Phase 20](README.md) · [20.10 Project: harden a service →](10-project-harden-a-service.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/09-production-readiness-checklist.md) <!-- nav -->

# Lesson 20.9 — The production-readiness checklist

> **Phase 20 — Reliability and hardening** · Lesson 9 of 11 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a production-readiness review is, and why every item on the checklist needs **evidence**.
- A complete checklist for a Rust service, with a pointer to the lesson that teaches each item.
- How to check the whole configuration at start and refuse to run with a clear message.
- The difference between liveness and readiness, and how a service drains its traffic before it stops.
- How to let a script check the items that a machine can check.

## Before you start

- You finished [Lesson 20.8 — Overload, limits and backpressure](08-overload-limits-and-backpressure.md).
- You know graceful shutdown and signals ([Lesson 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md)), health checks and configuration ([Lesson 16.13](../16-backend-services/13-configuration-health-checks-and-containers.md)), the Clippy lints against panics ([Lesson 20.2](02-errors-and-panics-in-production.md)) and testing a router with `oneshot` ([Lesson 16.3](../16-backend-services/03-web-services-with-axum.md)).
- Create the package for this lesson. It is a library (the parts we test) and a program (`src/main.rs`) in one package:

```bash
cd ~/projects/rust
cargo new --lib code/p20-ready
cd code/p20-ready
cargo add tokio@1 --features full
cargo add axum@0.8
cargo add --dev tower@0.5 --features util
```

## New words

| Word | Meaning |
|---|---|
| **production** | The real environment where real users use the service. |
| **PRR** (Production Readiness Review) | A review before a service goes to production (or after a big change): the team goes through a checklist and shows evidence for each item. |
| **evidence** | Something that proves an item is done: a test, a command and its output, a dashboard, a document. "We think so" is not evidence. |
| **fail fast** | Stop at once with a clear error when something is wrong, instead of running in a broken state. |
| **liveness** | "Is the process alive and able to answer?" If not, the platform restarts it. |
| **readiness** | "Should this copy get traffic right now?" If not, the load balancer stops sending requests to it, but does not restart it. |
| **draining** | Before stopping: report "not ready", wait until the load balancer stops sending new requests, then finish the open ones. |
| **SIGTERM** | The "please terminate" signal that service managers (systemd, Kubernetes) send before they stop a process. |
| **runbook** | A short document for the person on call: how to recognise a problem, what to check, what to do. |
| **rollback** | Going back to the previous working version after a bad deployment. |

## 1. What "ready for production" means

Code that passes its tests on your computer is ready for **review**. Code that is ready for **production** can also survive bad input, slow dependencies, overload, restarts, attacks and the night when somebody else has to fix it. Nobody remembers all of that by heart, so teams use a **checklist** in a **production-readiness review** (PRR): before launch, the team goes through every item and shows the evidence.

Three rules make a checklist useful:

1. **Every item has evidence.** "Timeouts: yes" is not enough. "Timeouts: every outgoing call goes through `call_with_deadline`, test `slow_dependency_gives_503`" is.
2. **"Not needed" is allowed, with a reason.** A command-line tool does not need a readiness endpoint. Write *why*, so the next reviewer does not have to guess.
3. **The checklist is repeated.** Not only before the first launch, but after every big change: a new dependency, a new data store, a new kind of traffic.

## 2. The checklist

Copy these tables into a file `READINESS.md` in your project and fill in the evidence column. The last column shows where the course taught the item.

**A. Correctness**

| Item | Evidence (example) | Lesson |
|---|---|---|
| Unit and integration tests for the main paths and the error paths | `cargo test`, coverage report | 09.1, 09.7 |
| Every parser of untrusted input has a fuzz target, run regularly | `fuzz/fuzz_targets/`, scheduled CI (Continuous Integration) job | 20.3 |
| Every `unsafe` block has a `SAFETY` comment and is tested under Miri | `cargo +nightly miri test` | 06.6, 10.12, 20.4 |
| Concurrent code with atomics or custom locks is tested with loom | loom tests | 14.9, 20.4 |
| Formatting and Clippy are clean in CI | CI log | 09.12 |

**B. Failure handling**

| Item | Evidence (example) | Lesson |
|---|---|---|
| Expected errors are returned, not panicked; panic lints are on | `[lints.clippy]` in `Cargo.toml` | 20.2 |
| A panic in one request does not kill the process, and is logged | panic hook, test | 20.2 |
| Every outgoing call (database, HTTP: HyperText Transfer Protocol, cache) has a timeout | code review, test with a slow fake | 14.14, 18.2 |
| Retries only for idempotent operations, with backoff and jitter | code, test | 18.3 |
| Behaviour when a dependency is down was tested | chaos test with toxiproxy | 20.5 |

**C. Limits**

| Item | Evidence (example) | Lesson |
|---|---|---|
| Maximum request body size | `RequestBodyLimitLayer`, test with a big body | 16.4 |
| Concurrency limit with load shedding, or a bounded queue | layer stack, load test | 20.8 |
| Rate limit per client where clients can be abusive | test | 20.8 |
| Bounded channels and pools; sizes written down and explained | code | 20.8, 16.6 |
| The capacity is known (requests per second at the target latency) | load-test report | 19.2 |

**D. Observability**

| Item | Evidence (example) | Lesson |
|---|---|---|
| Structured logs with a request ID (identifier); no secrets or personal data in logs | sample log lines | 09.10, 16.4 |
| Metrics for rate, errors and duration of every endpoint | dashboard | 16.12 |
| Traces across service boundaries | a trace in Jaeger | 16.12 |
| An SLO (Service Level Objective: a reliability goal) and an alert that fires when it is in danger | alert rule | 20.1, 21.11 |

**E. Health and lifecycle**

| Item | Evidence (example) | Lesson |
|---|---|---|
| Liveness and readiness endpoints with different meanings | `/livez`, `/readyz`, test | 16.13, this lesson |
| On SIGTERM: not ready, drain, finish open requests, exit `0` | shutdown script and its output | 14.14, this lesson |
| A hard time limit for the shutdown | code | 14.14 |
| The service starts without manual steps (migrations, warm-up) | start-up log | 16.7 |

**F. Configuration and secrets**

| Item | Evidence (example) | Lesson |
|---|---|---|
| The whole configuration is checked at start; bad values stop the start with a clear message | test, start with bad values | 16.13, this lesson |
| No secrets in Git, images or logs; they come from a secret store or the environment | review, secret scanner | 10.8 |
| Defaults are safe (for example: listen on `127.0.0.1` unless told otherwise) | config tests | 10.1 |

**G. Security and supply chain**

| Item | Evidence (example) | Lesson |
|---|---|---|
| Input validated at the border; authentication and authorization on every endpoint | tests | 10.4, 10.7, 16.8 |
| `cargo audit` and `cargo deny` in CI, on every push and weekly | CI config | 10.13 |
| New dependencies are reviewed (`cargo vet`); releases have an SBOM (Software Bill of Materials), checksums and a signature | `supply-chain/`, release files | 20.7 |
| The container runs as a non-root user, from a small image | Dockerfile | 11.13 |

**H. Deployment and recovery**

| Item | Evidence (example) | Lesson |
|---|---|---|
| A rollback to the previous version was **tried**, not only planned | notes from the test | 21.7 |
| Database migrations are backwards compatible with the running version | migration review | 16.7 |
| Backups exist **and a restore was tested** | restore log | — |
| More than one copy runs, so one crash is not an outage | deployment config | 20.1 |

**I. Operations**

| Item | Evidence (example) | Lesson |
|---|---|---|
| An owner team and an on-call person are named | team page | 21.12 |
| A runbook exists for the most likely alerts | `RUNBOOK.md` | 21.12 |
| Dashboards show the SLO and the main dependencies | dashboard link | 21.9, 21.11 |

Some items point to Phase 21, which comes next, and a few (marked —) are not taught in this course; learn them from your team's tools. Write them down now as "not done yet": a checklist also shows what is still missing.

> 📏 **Convention:** keep the checklist **short enough to be used**. Thirty precise items with evidence beat two hundred vague ones that nobody reads. When an incident shows a gap, add one item for it.

The rest of the lesson builds two items from groups E and F in code, and then a script for the items a machine can check.

## 3. Fail fast on bad configuration

A service that starts with a wrong setting is worse than one that does not start at all: it may run for hours, then fail at night when the bad value is first used. So read and check **everything** at start, report **every** problem at once, and stop with a non-zero exit code.

Replace `src/lib.rs`:

```rust
//! Two production-readiness items in code: a checked configuration and health endpoints.

use std::time::Duration;

/// Everything the service needs to start. Built once, checked once.
#[derive(Debug, Clone, PartialEq)]
pub struct Config {
    pub port: u16,
    pub drain: Duration,
}

impl Config {
    /// Reads the settings through `lookup` (environment variables in `main`, a list in tests)
    /// and reports **every** problem, not only the first one.
    pub fn from_lookup(lookup: impl Fn(&str) -> Option<String>) -> Result<Config, Vec<String>> {
        let mut errors = Vec::new();

        let port = match lookup("PORT") {
            None => 3000,
            Some(text) => match text.parse::<u16>() {
                Ok(port) if port > 0 => port,
                _ => {
                    errors.push(format!(
                        "PORT must be a number from 1 to 65535, got {text:?}"
                    ));
                    0
                }
            },
        };

        let drain_seconds = match lookup("DRAIN_SECONDS") {
            None => 2,
            Some(text) => match text.parse::<u64>() {
                Ok(seconds) if seconds <= 60 => seconds,
                _ => {
                    errors.push(format!("DRAIN_SECONDS must be 0 to 60, got {text:?}"));
                    0
                }
            },
        };

        if errors.is_empty() {
            Ok(Config {
                port,
                drain: Duration::from_secs(drain_seconds),
            })
        } else {
            Err(errors)
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    /// A fake environment made from a list of (name, value) pairs.
    fn env(pairs: &[(&str, &str)]) -> impl Fn(&str) -> Option<String> {
        let pairs: Vec<(String, String)> = pairs
            .iter()
            .map(|(k, v)| (k.to_string(), v.to_string()))
            .collect();
        move |name| {
            pairs
                .iter()
                .find(|(k, _)| k == name)
                .map(|(_, v)| v.clone())
        }
    }

    #[test]
    fn reports_every_bad_value() {
        let errors =
            Config::from_lookup(env(&[("PORT", "eighty"), ("DRAIN_SECONDS", "600")])).unwrap_err();
        assert_eq!(errors.len(), 2, "{errors:?}");
        assert!(errors[0].contains("PORT"));
        assert!(errors[1].contains("DRAIN_SECONDS"));
    }
}
```

`cargo test --lib`:

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. `from_lookup` does not call `std::env::var` itself. It gets a **function** that looks a name up. `main` passes `|name| std::env::var(name).ok()`; the tests pass a fake environment. Tests that change real environment variables would disturb each other, because all tests of a binary share one process.
2. Each setting has a safe default (`None => 3000`) and a range check. `Ok(port) if port > 0` is a match guard (Lesson 04.4): port 0 parses as a `u16`, but is not a valid port to listen on.
3. A bad value does not stop the function. The message goes into `errors`, a placeholder value keeps the code going, and at the end the function returns **all** messages. The person who fixes the configuration sees every problem in one run, not one per restart.
4. Every message says which setting, what is allowed, and what was given (`{text:?}` adds quotes, so an empty or space-only value is visible).

## 4. Liveness, readiness and draining

A platform such as Kubernetes asks every copy of a service two different questions (you will configure them in Lesson 21.5):

| Question | Endpoint | If the answer is "no" |
|---|---|---|
| **Liveness:** are you alive? | `/livez` | The process is **restarted**. |
| **Readiness:** may I send you traffic? | `/readyz` | The load balancer **stops sending** requests to this copy; it is not restarted. |

(Lesson 16.13 called the liveness endpoint `/healthz`, and Phase 21 keeps that name for the Kubernetes liveness probe. The name is only a convention: the Kubernetes API server itself uses `/livez` and `/readyz`, and so does this phase. What matters is that the two endpoints answer different questions; a probe can be pointed at any path.)

The difference matters. Readiness is "no" while the service starts (caches are still empty) and while it shuts down. Liveness must stay "yes" in both cases, or the platform kills a copy that is only busy starting or finishing its work.

Replace the line `use std::time::Duration;` at the top of `src/lib.rs` with these lines:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::time::Duration;

use axum::Router;
use axum::extract::State;
use axum::http::StatusCode;
use axum::routing::get;
```

and append the health part:

```rust
/// Shared state: is this copy of the service ready to receive traffic?
#[derive(Clone, Default)]
pub struct Health {
    ready: Arc<AtomicBool>,
}

impl Health {
    pub fn set_ready(&self, ready: bool) {
        self.ready.store(ready, Ordering::SeqCst);
    }

    pub fn is_ready(&self) -> bool {
        self.ready.load(Ordering::SeqCst)
    }
}

/// Liveness: "the process is alive and can answer". Never checks other services.
async fn livez() -> &'static str {
    "alive\n"
}

/// Readiness: "send me traffic". False while starting and while shutting down.
async fn readyz(State(health): State<Health>) -> (StatusCode, &'static str) {
    if health.is_ready() {
        (StatusCode::OK, "ready\n")
    } else {
        (StatusCode::SERVICE_UNAVAILABLE, "not ready\n")
    }
}

/// A slow request, to watch what happens to work that is in flight during shutdown.
async fn slow() -> &'static str {
    tokio::time::sleep(Duration::from_secs(3)).await;
    "slow request finished\n"
}

pub fn app(health: Health) -> Router {
    Router::new()
        .route("/livez", get(livez))
        .route("/readyz", get(readyz))
        .route("/slow", get(slow))
        .with_state(health)
}
```

Create `src/main.rs`. It uses Unix signals, so it runs on macOS and Linux:

```rust
use std::process::ExitCode;

use p20_ready::{Config, Health, app};
use tokio::signal::unix::{SignalKind, signal};

/// Waits for Ctrl+C (SIGINT) or SIGTERM, the signal that service managers send.
async fn stop_signal() {
    let mut term = signal(SignalKind::terminate()).expect("cannot listen for SIGTERM");
    tokio::select! {
        _ = tokio::signal::ctrl_c() => {}
        _ = term.recv() => {}
    }
}

#[tokio::main]
async fn main() -> ExitCode {
    // 1. Check the whole configuration before doing anything else.
    let config = match Config::from_lookup(|name| std::env::var(name).ok()) {
        Ok(config) => config,
        Err(errors) => {
            for error in errors {
                eprintln!("config error: {error}");
            }
            return ExitCode::from(2);
        }
    };

    let health = Health::default(); // not ready yet
    let listener = tokio::net::TcpListener::bind(("127.0.0.1", config.port))
        .await
        .unwrap();
    // A real service would connect to its database and warm its caches here.
    health.set_ready(true);
    println!("listening on 127.0.0.1:{}, ready", config.port);

    // 2. On a signal: report "not ready", wait for the load balancer to notice, then stop.
    let drain = config.drain;
    let draining = health.clone();
    let shutdown = async move {
        stop_signal().await;
        draining.set_ready(false);
        println!("signal: not ready any more, draining for {drain:?}");
        tokio::time::sleep(drain).await;
        println!("drain over: no new connections, finishing open requests");
    };

    axum::serve(listener, app(health))
        .with_graceful_shutdown(shutdown)
        .await
        .unwrap();
    println!("all requests finished, exiting");
    ExitCode::SUCCESS
}
```

First, the fail-fast part. Start it with two bad settings:

```bash
PORT=eighty DRAIN_SECONDS=600 cargo run -q
echo "exit code: $?"
```

```text
config error: PORT must be a number from 1 to 65535, got "eighty"
config error: DRAIN_SECONDS must be 0 to 60, got "600"
exit code: 2
```

Now the shutdown. This script starts the service, begins a slow request, sends SIGTERM and watches the endpoints. Save it as `shutdown-demo.sh` and make it executable:

```bash
#!/usr/bin/env bash
# Starts the service, stops it with SIGTERM, and watches the health endpoints.
set -u
status() { curl -s -o /dev/null -w '%{http_code}' "http://127.0.0.1:3000$1"; }

./target/release/p20-ready &
pid=$!
sleep 0.5
echo "readyz: $(status /readyz)"
curl -s http://127.0.0.1:3000/slow > slow.txt &   # a request that needs 3 seconds
sleep 0.2
kill -TERM "$pid"
sleep 0.3
echo "readyz: $(status /readyz)   livez: $(status /livez)"
wait "$pid"
echo "exit code: $?"
echo "the slow request got: $(cat slow.txt)"
echo "readyz: $(status /readyz)"
```

`cargo build --release`, then `./shutdown-demo.sh`:

```text
listening on 127.0.0.1:3000, ready
readyz: 200
signal: not ready any more, draining for 2s
readyz: 503   livez: 200
drain over: no new connections, finishing open requests
all requests finished, exiting
exit code: 0
the slow request got: slow request finished
readyz: 000
```

**What happens:**

1. Before the signal, `/readyz` answers `200`. The service set itself ready only **after** the listener was bound (and, in a real service, after the database connection worked).
2. `kill -TERM` sends SIGTERM. The shutdown future sets readiness to false at once. During the drain, `/readyz` answers `503` while `/livez` still answers `200`: "do not send me new work, but do not kill me".
3. Why wait at all? A load balancer asks `/readyz` only every few seconds. Until it notices the `503`, it keeps sending requests. If the service stopped accepting connections at once, those requests would fail. The drain time (here 2 s) should be a little longer than the time between two readiness checks.
4. After the drain, `with_graceful_shutdown` stops accepting new connections and waits for the open requests. The slow request, started before the signal, still gets its full answer.
5. The process ends with exit code `0`, so the service manager knows the stop was clean. The last `000` means curl could not connect at all: the process is gone.

> ⚠️ **Common mistake:** checking the database in **liveness**. When the database is down, every copy fails its liveness check and is restarted again and again. That does not repair the database; it only adds restarts on top of the outage. Liveness checks only the process itself.

> 💡 **Tip:** the drain has no upper limit for the open requests here. A real service also needs a **hard limit** for the whole shutdown (Lesson 14.14), and the platform's own grace period (for example 30 s in Kubernetes) must be longer than drain plus that limit.

## 5. Let a machine check what it can

Many items in groups A, B and G can be checked by commands. A small script turns them into a list of PASS and FAIL lines. Save it as `readiness-check.sh`:

```bash
#!/usr/bin/env bash
# Runs the checklist items that a machine can check. Exit code 1 if any of them fails.
set -u
failed=0

check() {
    local name="$1"
    shift
    if "$@" > /dev/null 2>&1; then
        echo "PASS  $name"
    else
        echo "FAIL  $name   (run: $*)"
        failed=1
    fi
}

check "code is formatted"         cargo fmt --check
check "no Clippy warnings"        cargo clippy --all-targets -- -D warnings
check "no unwrap outside tests"   cargo clippy --lib --bins -- -D clippy::unwrap_used
check "all tests pass"            cargo test
check "no known vulnerabilities"  cargo audit
check "lock file exists"          test -f Cargo.lock
exit "$failed"
```

```bash
chmod +x readiness-check.sh
./readiness-check.sh
echo "exit code: $?"
```

```text
PASS  code is formatted
PASS  no Clippy warnings
FAIL  no unwrap outside tests   (run: cargo clippy --lib --bins -- -D clippy::unwrap_used)
PASS  all tests pass
PASS  no known vulnerabilities
PASS  lock file exists
exit code: 1
```

**What happens:**

1. `check` takes a name and a command (`"$@"` is all remaining arguments), runs the command with its output hidden and prints PASS or FAIL (with the command, so you can run it and read the details). The exit code is `1` if anything failed, so CI can run the script as one step.
2. `cargo clippy --lib --bins -- -D clippy::unwrap_used` turns the `unwrap_used` lint from Lesson 20.2 into an error for the library and the program, but not for tests. It found the two `.unwrap()` calls in `main.rs`. You remove them in Try it yourself.

The script does not replace the review. It cannot see whether a timeout is sensible, whether the runbook is useful or whether a restore was tested. It removes the boring part, so the review can spend its time on those questions.

## Common mistakes

**1. A checklist without evidence.** Ticking boxes from memory. Ask for the test, the command or the link.

**2. Readiness that is always `200`.** Then the load balancer sends traffic to copies that are starting or stopping, and every deployment causes a short burst of errors.

**3. Exiting at once on SIGTERM.** Open requests are cut off and requests that are already on the way fail. Drain first, then finish.

**4. Reading configuration lazily.** A typo in a setting that is first used at 3 a.m. Check everything at start.

## Try it yourself

1. Make `./readiness-check.sh` pass: in `main.rs`, replace the two `.unwrap()` calls by a `match` (or `if let Err(error) = …`) that prints the error with `eprintln!` and returns `ExitCode::FAILURE`. Then start the service twice; the second copy should print `cannot listen on port 3000: Address already in use (os error 48)` (on Linux the error number is 98) and exit with code `1`.
2. Run `DRAIN_SECONDS=0 ./shutdown-demo.sh`. What changes in the output, and why is `0` a risky value in production?
3. Fill in groups E and F of the checklist for `p20-ready`, with evidence. Which items are still missing?

## Exercises

### Exercise 1 — Which item was missing? (easy)

For each incident, name the checklist item (group and item) that would most likely have prevented it.

1. Every deployment causes about 30 seconds of `502` errors.
2. A payment provider hangs; after two minutes the service has 10,000 open requests and runs out of memory.
3. After an update the service runs for six hours, then crashes when the nightly job reads a setting with a typo.
4. A database disk fails. The backups exist, but nobody knows how to restore them, and the restore takes two days.

<details>
<summary>Hint</summary>

Look at groups B, C, E, F and H.

</details>

<details>
<summary>Answer</summary>

1. **E:** readiness and draining on SIGTERM. Copies stop accepting work before the load balancer knows it.
2. **B:** a timeout on every outgoing call, and **C:** a concurrency limit with load shedding. Either one would have kept the number of open requests small.
3. **F:** the whole configuration is checked at start. The typo would have stopped the deployment at once.
4. **H:** backups exist **and a restore was tested**. A tested restore also tells you how long it takes.

Why it works: most incidents are not exotic. They fall into a few groups, and a checklist is the memory of those groups.

</details>

### Exercise 2 — Test the health endpoints (medium)

Write an integration test `tests/health.rs` that checks, without a network: `/readyz` is `503` before `set_ready(true)`, `200` after it, and `503` again after `set_ready(false)`; `/livez` is `200` the whole time.

<details>
<summary>Hint</summary>

Write a small helper `async fn status(health: &Health, path: &str) -> StatusCode` that builds `app(health.clone())` and sends one request with `oneshot`. All clones of `Health` share the same `Arc<AtomicBool>`.

</details>

<details>
<summary>Answer</summary>

```rust
use axum::body::Body;
use axum::http::{Request, StatusCode};
use p20_ready::{Health, app};
use tower::ServiceExt;

async fn status(health: &Health, path: &str) -> StatusCode {
    let request = Request::get(path).body(Body::empty()).unwrap();
    app(health.clone()).oneshot(request).await.unwrap().status()
}

#[tokio::test]
async fn readiness_follows_the_flag_and_liveness_stays_ok() {
    let health = Health::default();
    assert_eq!(
        status(&health, "/readyz").await,
        StatusCode::SERVICE_UNAVAILABLE
    );
    assert_eq!(status(&health, "/livez").await, StatusCode::OK);

    health.set_ready(true);
    assert_eq!(status(&health, "/readyz").await, StatusCode::OK);

    health.set_ready(false); // shutting down
    assert_eq!(
        status(&health, "/readyz").await,
        StatusCode::SERVICE_UNAVAILABLE
    );
    assert_eq!(status(&health, "/livez").await, StatusCode::OK);
}
```

`cargo test --test health`:

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the test builds a new router for each request, but all routers share the same flag, because `Health` holds an `Arc`. That is exactly how the real service works: the shutdown future and the handlers hold clones of one `Health`. This test is the evidence for the item "liveness and readiness endpoints with different meanings".

</details>

### Exercise 3 — Can we reach the database? (hard)

A readiness check sometimes also asks "can I reach my database?". Write a public function `can_connect(address: &str) -> bool` in `src/lib.rs` that returns `true` only if a TCP (Transmission Control Protocol) connection to `address` can be opened within 200 ms. Write a test with an open and a closed port, without a real database. Then explain how you would use it in `readyz`, and what the danger is.

<details>
<summary>Hint</summary>

`tokio::time::timeout(Duration::from_millis(200), tokio::net::TcpStream::connect(address))` gives `Ok(Ok(stream))` only on success. For a test port, bind `TcpListener` to `127.0.0.1:0` (the system picks a free port), read `local_addr()`, and `drop` the listener to close the port again.

</details>

<details>
<summary>Answer</summary>

In `src/lib.rs`:

```rust
/// Can we open a TCP connection to `address` within 200 ms?
pub async fn can_connect(address: &str) -> bool {
    let attempt = tokio::net::TcpStream::connect(address);
    matches!(
        tokio::time::timeout(Duration::from_millis(200), attempt).await,
        Ok(Ok(_))
    )
}
```

`tests/dependency.rs`:

```rust
use p20_ready::can_connect;

#[tokio::test]
async fn sees_an_open_port_and_a_closed_one() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
    let address = listener.local_addr().unwrap().to_string();
    assert!(can_connect(&address).await);

    drop(listener); // nobody listens there any more
    assert!(!can_connect(&address).await);
}
```

`cargo test --test dependency` passes.

Why it works: the operating system answers a connection to a closed local port at once with "connection refused", and an open port accepts it, so the test is fast and needs no database. The 200 ms timeout keeps `/readyz` fast even when the database hangs. To use it, give `Health` a field `database: Option<String>` and let `readyz` answer `200` only if `is_ready()` **and** `can_connect(address).await` (or there is no address). The danger: if **every** copy depends on one database, they all become "not ready" together when it fails, and the load balancer has nowhere to send traffic, not even for requests that do not need the database. Check only dependencies without which the service can do nothing useful, and never check them in liveness.

</details>

## Quiz

1. What makes a checklist item "done" in a production-readiness review?

<details>
<summary>Answer</summary>

Evidence: a test, a command with its output, a dashboard or a document that proves it. Or "not needed" with a written reason.

</details>

2. What is the difference between liveness and readiness, and what does the platform do when each one fails?

<details>
<summary>Answer</summary>

Liveness asks "is the process alive?"; if it fails, the process is restarted. Readiness asks "should it get traffic now?"; if it fails, the load balancer stops sending requests, but the process keeps running.

</details>

3. Why does the service wait (drain) after SIGTERM before it stops accepting connections?

<details>
<summary>Answer</summary>

The load balancer only checks readiness every few seconds. Until it sees the `503`, it keeps sending new requests. Waiting a little longer than that interval means no request arrives at a copy that already refuses connections.

</details>

4. Why should a configuration check report all problems at once?

<details>
<summary>Answer</summary>

So the person fixing the configuration sees everything in one run. With one error per start, five typos need five deployments.

</details>

5. Why should liveness not check the database?

<details>
<summary>Answer</summary>

When the database is down, every copy would fail liveness and be restarted again and again. Restarts do not fix the database; they only add more failure and load.

</details>

## Summary

- A production-readiness review goes through a checklist before launch and after big changes; every item needs evidence or a written "not needed".
- The checklist groups: correctness, failure handling, limits, observability, health and lifecycle, configuration and secrets, security and supply chain, deployment and recovery, operations.
- Check the whole configuration at start, report every problem, and exit with a non-zero code.
- Liveness means "alive", readiness means "send me traffic"; they must be different endpoints with different checks.
- On SIGTERM: become not ready, drain, stop accepting, finish open requests, exit `0`, with a hard limit for the whole shutdown.
- Let a script check what a machine can check, and spend the review on the questions that need judgement.

## Sources and further reading

- SRE (Site Reliability Engineering): [Google SRE book: Launch Coordination Checklist](https://sre.google/sre-book/launch-checklist/) — the checklist that Google's launch reviews grew from.
- [Kubernetes: liveness, readiness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) — how a platform uses the two endpoints.
- [Kubernetes: pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) — SIGTERM, the grace period and what happens to traffic during shutdown.
- [axum: `serve` and graceful shutdown](https://docs.rs/axum/latest/axum/serve/struct.Serve.html) — `with_graceful_shutdown`.
- [tokio: Unix signals](https://docs.rs/tokio/latest/tokio/signal/unix/index.html) — listening for SIGTERM.
- [The Twelve-Factor App: Config](https://12factor.net/config) — configuration in the environment, separate from code.

---

[🏠 Start](../START-HERE.md) · [← 20.8 Overload, limits and backpressure](08-overload-limits-and-backpressure.md) · [📋 Phase 20](README.md) · [20.10 Project: harden a service →](10-project-harden-a-service.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
