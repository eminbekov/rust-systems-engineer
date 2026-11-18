<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 16 — Backend services](README.md) · [📋 Phase 16](README.md) · [16.2 Designing HTTP APIs: REST →](02-designing-http-apis.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/01-what-a-backend-service-is.md) <!-- nav -->

# Lesson 16.1 — What a backend service is

> **Phase 16 — Backend services** · Lesson 1 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a backend service is, and how it differs from the command-line programs you wrote so far.
- The parts of a typical backend system (clients, load balancer, service, database, cache, message broker) and which lesson of this phase teaches each part.
- How to run a tiny Rust service with the `axum` crate and call it with `curl`.
- Why one service process can answer many clients at the same time, and how one blocking call breaks that.
- Why a service should not keep important data in its own memory, and where the data goes instead.

## Before you start

- You built a web server by hand on top of `TcpListener` in [Lesson 06.11 — Project: multithreaded web server](../06-rust-advanced/11-project-multithreaded-web-server.md).
- You know async Rust and Tokio tasks ([Lesson 06.9 — Introduction to async/await](../06-rust-advanced/09-async-await-introduction.md), Lesson 14.13 — Tokio in depth: tasks and channels).
- You know how HTTP (HyperText Transfer Protocol) looks on the wire (Lesson 15.8 — HTTP from scratch) and how to use HTTP from Rust (Lesson 15.9 — HTTP with hyper and reqwest).
- You met hexagonal architecture (Lesson 08.14) and the twelve-factor app (Lesson 08.15). This phase uses both ideas in real services.
- Docker Desktop (or the VM (virtual machine) `dev` from Phase 11) is installed. You need it from Lesson 16.5 on.

## New words

| Word | Meaning |
|---|---|
| **backend** | The part of a software system that runs on servers: it keeps the data, applies the rules and answers requests. Users do not see it directly. |
| **frontend** | The part the user sees and touches: a web page, a phone app, a desktop program. It sends requests to the backend. |
| **service** | A program that runs for a long time, waits for requests over the network and answers them. A *backend service* is a service that is part of a backend. |
| **client** | A program that sends requests to a service: a browser, a phone app, `curl`, or another service. |
| **API** (Application Programming Interface) | The agreed list of requests a service understands and the answers it gives. It is the contract between the service and its clients. |
| **endpoint** | One kind of request a service answers, written as an HTTP method plus a path, for example `GET /health`. |
| **handler** | The function in the service that runs for one endpoint and builds the response. |
| **router** | The part of a web framework that looks at the method and path of a request and picks the handler. |
| **web framework** | A library that does the common work of an HTTP server (parsing, routing, building responses) so you only write handlers. `axum` is one. |
| **instance** | One running copy of a service. A service can run as many instances at once. |
| **stateless service** | A service that keeps no important data in its own memory between requests. Any instance can answer any request. |
| **horizontal scaling** | Handling more load by running more instances, instead of buying a bigger machine. |
| **load balancer** | A program in front of the instances that spreads incoming requests over them (Lesson 15.14). |
| **database** | A separate program that stores data safely on disk and answers questions about it. This phase uses PostgreSQL. |
| **cache** | A fast store that keeps copies of data that is read often, so the database is asked less. This phase uses Redis. |
| **message broker** | A program that receives messages from one service and delivers them to others, so they do not have to call each other directly. This phase uses NATS. |
| **health check** | A request (often `GET /health`) that only answers "I am alive". Load balancers and container platforms use it. |
| **blocking call** | A call that keeps a thread busy while it waits (for time, a file, the network), so the thread can do nothing else. |

## 1. From a program to a service

The programs you wrote so far start, do one job and stop. `minigrep` reads a file, prints the matching
lines and exits. A **service** is different: it starts, opens a network port, and then **waits**. Every
time a request arrives, it does a small piece of work, sends an answer and goes back to waiting. It stops
only when someone tells it to stop.

A good everyday picture is the kitchen of a restaurant. The kitchen does not cook one meal and close. It
opens in the morning, takes orders from many waiters at the same time, keeps the food in a fridge (not in
the cook's hands), and closes in the evening after the last orders are finished.

| | Command-line program | Service |
|---|---|---|
| How long it runs | Seconds, one job | Days or months, until stopped |
| Who starts the work | The user, with arguments | Clients, with requests over the network |
| How many users at once | One | Many at the same time |
| Where the input comes from | Arguments, files, standard input | HTTP requests (or messages) |
| Where the output goes | Standard output, files | HTTP responses (or messages) |
| What happens on an error | Print it and exit with a code | Answer this one request with an error and keep running |

The last row is important. A program may exit when something goes wrong. A service must not: one bad
request must never stop the service for all the other clients.

## 2. Frontend, backend and the API between them

Think about a banking app on a phone. The app (the **frontend**) shows your balance. It does not know your
balance itself. It sends a request over the internet to the bank's servers (the **backend**), and the
backend answers with the number.

```text
 phone app (frontend)                         bank servers (backend)
 ┌──────────────────┐   GET /accounts/42      ┌──────────────────────┐
 │ shows "1 250.00" │ ──────────────────────▶ │ checks who you are,  │
 │                  │                          │ reads the database,  │
 │                  │ ◀────────────────────── │ builds the answer    │
 └──────────────────┘   200 OK {"balance":…}  └──────────────────────┘
```

The list of requests the backend understands, and the answers it gives, is the **API**. It is a contract:
the frontend team and the backend team can work separately as long as both keep the contract. Lesson 16.2
shows how to design a good one.

The backend owns two things:

1. **The data.** Accounts, orders, messages. The frontend only shows copies.
2. **The rules.** "You can only see your own account." "A transfer cannot be negative." The backend must
   check every rule itself, because anybody can send any request to it, not only your app. You learned
   this as "never trust input" in [Lesson 10.4](../10-security-fundamentals/04-input-validation.md).

> 🔒 **Security:** a check that exists only in the frontend is not a check. An attacker skips your app and
> sends requests with `curl`. Every permission and every validation must also happen in the backend.

## 3. The parts of a backend system

A real backend is usually several programs that talk to each other over the network:

```text
                ┌──────────────┐
  clients ────▶ │ load balancer│ ───┬─────────────┬───────────────┐
  (apps, curl,  └──────────────┘    ▼             ▼               ▼
   other                       ┌─────────┐   ┌─────────┐     ┌─────────┐
   services)                   │service  │   │service  │ ... │service  │   instances of
                               │instance1│   │instance2│     │instanceN│   YOUR program
                               └────┬────┘   └────┬────┘     └────┬────┘
                                    │             │               │
                  ┌─────────────────┼─────────────┼───────────────┤
                  ▼                 ▼             ▼               ▼
            ┌──────────┐      ┌──────────┐   ┌───────────┐  ┌─────────────────┐
            │PostgreSQL│      │  Redis   │   │   NATS    │  │ logs, metrics,  │
            │(database)│      │ (cache)  │   │ (messages)│  │ traces collector│
            └──────────┘      └──────────┘   └───────────┘  └─────────────────┘
```

| Part | Its job | Where you learn it |
|---|---|---|
| Load balancer | Spreads requests over the instances | Lesson 15.14 |
| Service instances | Your Rust code: handlers, rules, validation | Lessons 16.3, 16.4 |
| API design | What requests look like | Lesson 16.2 (REST, REpresentational State Transfer), Lesson 16.9 (gRPC) |
| Database | Stores the data safely | Lessons 16.5 to 16.7 |
| Login and permissions | Who is calling, what they may do | Lesson 16.8 |
| Cache | Fast copies of hot data | Lesson 16.10 |
| Message broker | Events between services | Lesson 16.11 |
| Observability | Logs, metrics and traces | Lesson 16.12 |
| Configuration and containers | How the service is started and run | Lesson 16.13 |
| Tests | Proving it all works | Lesson 16.14 |

At the end of the phase you build all of this yourself in Lesson 16.15, a URL (Uniform Resource Locator) shortener service.

## 4. Your first service

In Lesson 06.11 you read bytes from a `TcpStream`, split the text into lines, found the path and wrote the
response by hand. That was a good way to learn, but every real service needs the same parts again and
again: parsing HTTP correctly, choosing a handler, building headers, handling many connections. A **web
framework** does that work for you. This course uses **`axum`**, a web framework built on Tokio and on
`hyper` (the HTTP library from Lesson 15.9).

**How the code of this phase is organised.** Each lesson tells you which folder to use. This lesson and the
next one use a small package `code/p16-hello`. From Lesson 16.3 to Lesson 16.8 you grow one service,
`code/p16-tasks`, step by step.

Create the package and add the two crates:

```bash
cd ~/projects/rust
cargo new code/p16-hello
cd code/p16-hello
cargo add axum@0.8
cargo add tokio@1 --features full
```

`Cargo.toml` now has:

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
```

Replace `src/main.rs` with:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::Duration;

use axum::{Router, routing::get};

// Lives in the memory of this one process. It is lost when the process stops.
static VISITS: AtomicU64 = AtomicU64::new(0);

async fn health() -> &'static str {
    "ok\n"
}

async fn slow() -> &'static str {
    tokio::time::sleep(Duration::from_secs(1)).await;
    "done after 1 second\n"
}

async fn visits() -> String {
    let n = VISITS.fetch_add(1, Ordering::Relaxed) + 1;
    format!("visit number {n}\n")
}

#[tokio::main]
async fn main() {
    let port = std::env::args().nth(1).unwrap_or_else(|| "3000".to_string());
    let addr = format!("127.0.0.1:{port}");

    let app = Router::new()
        .route("/health", get(health))
        .route("/slow", get(slow))
        .route("/visits", get(visits));

    let listener = tokio::net::TcpListener::bind(&addr).await.unwrap();
    println!("listening on http://{addr}");
    axum::serve(listener, app).await.unwrap();
}
```

**What happens:**

1. `health`, `slow` and `visits` are **handlers**: ordinary `async fn`s. axum calls one of them for each
   request. Whatever a handler returns becomes the response. A `&'static str` or a `String` becomes a
   response with status `200 OK` and the text as the body.
2. `slow` waits one second with `tokio::time::sleep(...).await`. While it waits, the thread is free to work
   on other requests (Section 5 shows this).
3. `VISITS` is a global atomic counter (Lesson 14.5). `fetch_add(1, …)` adds one and returns the old value,
   safely, even if many requests run at the same time. `Relaxed` ordering is enough for a simple counter.
4. In `main`, the first command-line argument is the port; without one we use `3000`.
5. `Router::new().route("/health", get(health))` means: "for a `GET` request to the path `/health`, call
   `health`". Each `.route` adds one endpoint. This is the **router**.
6. `TcpListener::bind` opens the port, exactly as in Lesson 06.11, but the async version from Tokio.
   `127.0.0.1` means only programs on this computer can connect.
7. `axum::serve(listener, app)` runs forever: it accepts connections, parses HTTP, asks the router for a
   handler, runs it and writes the response. It only returns if something goes very wrong.

Start the service in one terminal:

```bash
cargo run
```

```text
listening on http://127.0.0.1:3000
```

The program does not end. It waits. Open a **second** terminal and be its first client. `-i` asks `curl`
to also print the status line and the headers:

```bash
curl -i http://127.0.0.1:3000/health
```

```text
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 3
date: Wed, 18 Nov 2026 06:55:24 GMT

ok
```

You did not write a single header, but axum produced a correct HTTP response: the status, the
`content-type` (because the handler returned text), the length and the date. Now try a path that does not
exist and a method the endpoint does not accept:

```bash
curl -i http://127.0.0.1:3000/nothing-here
curl -i -X POST http://127.0.0.1:3000/health
```

```text
HTTP/1.1 404 Not Found
content-length: 0
date: Wed, 18 Nov 2026 06:55:24 GMT

HTTP/1.1 405 Method Not Allowed
allow: GET,HEAD
content-length: 0
date: Wed, 18 Nov 2026 06:55:24 GMT
```

**What happens:**

1. No route matches `/nothing-here`, so the router answers `404 Not Found` by itself.
2. `/health` exists but only for `GET`. `-X POST` sends a `POST`, so the router answers
   `405 Method Not Allowed` and even lists the allowed methods in the `allow` header. (`HEAD` is the same
   as `GET` without a body; axum adds it for free.)

Stop the service with `Ctrl+C` in the first terminal when you are done.

> 💡 **Tip:** keep two terminals open for the whole phase: one runs the service, the other runs `curl`.
> In RustRover you can split the terminal tool window, or open a second tab in it.

## 5. Many clients at the same time

A service has many clients, and they do not wait politely for each other. Start the service again
(`cargo run`) and, in the second terminal, send **three** requests to `/slow` at the same time. The `&`
at the end of a command runs it in the background, `wait` waits for all of them, and `time` measures the
whole group:

```bash
time (curl -s http://127.0.0.1:3000/slow & curl -s http://127.0.0.1:3000/slow & curl -s http://127.0.0.1:3000/slow & wait)
```

```text
done after 1 second
done after 1 second
done after 1 second
( curl -s http://127.0.0.1:3000/slow & curl -s http://127.0.0.1:3000/slow &  )  0.01s user 0.01s system 1% cpu 1.013 total
```

Three requests, each needing one second, finished in about **1 second in total**, not three. (`-s` means
"silent": no progress bar.)

**What happens:**

1. `axum::serve` starts a new Tokio **task** for every connection (you know tasks from Lesson 14.13). A task
   is cheap: a service can have many thousands of them.
2. Each request reaches `slow`, which calls `sleep(...).await`. `.await` tells the runtime: "I have nothing to
   do for one second; run other tasks". So all three tasks wait at the same time.
3. After one second, the runtime wakes all three, and each one sends its answer.

This is why backend services in Rust are almost always async: most of the time a request **waits**, for the
database, for another service, for the network. While it waits, the thread serves other requests.

Now see what happens when a handler blocks. Create `src/bin/blocking.rs`. It uses `std::thread::sleep`,
which blocks the thread, and a runtime with only **one** thread (`flavor = "current_thread"`) so the effect
is easy to see:

```rust
use std::time::Duration;

use axum::{Router, routing::get};

async fn slow() -> &'static str {
    // WRONG: this blocks the whole thread, not only this request.
    std::thread::sleep(Duration::from_secs(1));
    "done after 1 second\n"
}

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let app = Router::new().route("/slow", get(slow));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    println!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app).await.unwrap();
}
```

Stop the first service, run this one with `cargo run --bin blocking`, and send the same three requests:

```text
done after 1 second
done after 1 second
done after 1 second
( curl -s http://127.0.0.1:3000/slow & curl -s http://127.0.0.1:3000/slow &  )  0.01s user 0.01s system 0% cpu 3.025 total
```

**3 seconds.** `std::thread::sleep` never returns control to the runtime, so the only thread could not start
the second request until the first one was completely finished. The requests ran one after another.

> ⚡ **Performance:** never call blocking functions inside an async handler: `std::thread::sleep`, big
> file reads with `std::fs`, heavy computations that take many milliseconds. With the default
> multi-threaded runtime the damage is hidden at first (other threads keep working), but under load every
> blocked thread is one thread less for all clients. Use the async version (`tokio::time::sleep`,
> `tokio::fs`) or move the work to `tokio::task::spawn_blocking` (Lesson 06.9, "Do not block the runtime").

## 6. Where does the data live?

Stop the blocking service and start the normal one again with `cargo run`. Call `/visits` a few times:

```bash
curl -s http://127.0.0.1:3000/visits
curl -s http://127.0.0.1:3000/visits
```

```text
visit number 1
visit number 2
```

It counts. Now stop the service with `Ctrl+C`, start it again, and call `/visits` once more: it says
`visit number 1` again. The counter lived in the memory of the process, and that memory is gone when the
process stops. Services are stopped all the time: for a new version, when a machine is replaced, when they
crash.

There is a second problem. Real services run as **several instances** behind a load balancer. Start a
second instance on port 3001 in a third terminal (`cargo run -- 3001`). Then call both, as a load balancer
would, alternating between them:

```bash
curl -s http://127.0.0.1:3000/visits
curl -s http://127.0.0.1:3000/visits
curl -s http://127.0.0.1:3001/visits
curl -s http://127.0.0.1:3000/visits
```

```text
visit number 1
visit number 2
visit number 1
visit number 3
```

Each instance has its own counter, so the clients see numbers that make no sense. The fix is the main rule
of backend design, and factor VI of the twelve-factor app (Lesson 08.15):

> **Keep the service stateless.** Every piece of data that must survive a restart, or must be the same for
> all instances, lives **outside** the process: in a database (Lessons 16.5–16.7), a cache (Lesson 16.10)
> or a message broker (Lesson 16.11).

```text
   stateful (bad)                         stateless (good)
 ┌───────────┐ ┌───────────┐          ┌───────────┐ ┌───────────┐
 │instance 1 │ │instance 2 │          │instance 1 │ │instance 2 │
 │ visits=3  │ │ visits=1  │          │ no data   │ │ no data   │
 └───────────┘ └───────────┘          └─────┬─────┘ └─────┬─────┘
   two different answers                    └──────┬──────┘
                                                   ▼
                                            ┌────────────┐
                                            │ database   │
                                            │ visits = 4 │
                                            └────────────┘
```

A stateless service has big advantages: you can restart any instance at any time, and you can handle more
load by simply starting more instances (**horizontal scaling**). Memory inside the process is still fine
for things that may be lost: a connection pool, a small cache of settings, a counter for metrics.

Stop both instances when you are done.

## 7. What else a real service must do

Answering `GET /health` is the easy part. A service that real people depend on must also:

- **Validate every input** and answer with clear errors instead of crashing (Lesson 16.4).
- **Store data safely** and change several things at once without leaving half-done work (Lessons 16.5–16.7).
- **Know who is calling** and what they are allowed to do (Lesson 16.8).
- **Be observable:** write structured logs, expose metrics and traces, so you can see what it is doing in
  production (Lesson 16.12, building on [Lesson 09.10 — Logging and tracing](../09-testing-debugging-benchmarking/10-logging-and-tracing.md)).
- **Read its configuration from the environment** and report its health (Lesson 16.13).
- **Shut down gracefully:** stop accepting new requests, finish the running ones, then exit (Lesson 14.14).
- **Be tested** at every level, including against a real database (Lesson 16.14).

> 📏 **Convention:** almost every service has a cheap `GET /health` endpoint that does no real work and
> answers `200 OK` while the process is alive. Keep it fast and never put secrets in its answer.

## Common mistakes

**1. The port is already in use.** You start the service while an old copy still runs (maybe in another
terminal, or in RustRover):

```text
thread 'main' (4882234) panicked at src/main.rs:33:63:
called `Result::unwrap()` on an `Err` value: Os { code: 48, kind: AddrInUse, message: "Address already in use" }
```

Only one program can listen on a port. Find the old one and stop it (`Ctrl+C`), or start this one on another
port: `cargo run -- 3002`. The number in brackets after `main` is the thread's identifier; yours differs.

**2. Waiting for `cargo run` to finish.** A service never "finishes". If your terminal seems stuck after
`listening on …`, that is correct: it is waiting for requests. Use a second terminal for `curl`.

**3. Blocking inside a handler.** `std::thread::sleep`, slow loops or `std::fs::read` of a big file inside an
`async fn` handler stop one runtime thread for everybody (Section 5). Use the async version or
`spawn_blocking`.

**4. Keeping important data in a global variable or a `Vec` in memory.** It works on your laptop with one
instance and breaks on restart or with two instances (Section 6). Use it only for data you may lose.

## Try it yourself

1. Start the service and call `curl -i http://127.0.0.1:3000/visits`. Which `content-type` does axum choose
   for a `String`? (It is the same as for `&'static str`.)
2. Call `curl -I http://127.0.0.1:3000/health` (capital `-I` sends a `HEAD` request). You get the headers
   but no body. Check that `content-length` is still `3`.
3. Send ten slow requests at once and measure:

   ```bash
   time (for i in 1 2 3 4 5 6 7 8 9 10; do curl -s http://127.0.0.1:3000/slow & done; wait)
   ```

   It still takes about one second. Try the same with `cargo run --bin blocking`: about ten seconds.
4. Commit your work: `git add code/p16-hello && git commit -m "p16-hello: first axum service"`.

## Exercises

### Exercise 1 — Test a handler (easy)

A handler is just an `async fn`, so you can call it from a test. Write a test `health_says_ok` in
`src/main.rs` that checks that `health()` returns `"ok\n"`. Tests for async functions use
`#[tokio::test]` instead of `#[test]`.

<details>
<summary>Hint</summary>

Inside an `async` test you can write `health().await`. Put the test in a `#[cfg(test)] mod tests` with
`use super::*;`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn health_says_ok() {
        assert_eq!(health().await, "ok\n");
    }
}
```

`cargo test` prints `test tests::health_says_ok ... ok`.

Why it works: `#[tokio::test]` starts a small Tokio runtime for the test, so you can `.await` inside it. The
handler does not need a running server: it is a normal function that returns a value.

</details>

### Exercise 2 — A version endpoint (medium)

Add the endpoint `GET /version` that answers with the version from `Cargo.toml` and a newline (`0.1.0\n`).
Do not type the number in the code: Cargo gives it to you at compile time with
`env!("CARGO_PKG_VERSION")`. Add a test for the new handler.

<details>
<summary>Hint</summary>

`concat!(env!("CARGO_PKG_VERSION"), "\n")` builds a `&'static str` at compile time. Add one more
`.route(...)` line.

</details>

<details>
<summary>Answer</summary>

```rust
async fn version() -> &'static str {
    concat!(env!("CARGO_PKG_VERSION"), "\n")
}

// in main:
//     .route("/version", get(version))

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn version_matches_cargo_toml() {
        assert_eq!(version().await, "0.1.0\n");
    }
}
```

`curl -s http://127.0.0.1:3000/version` prints `0.1.0`.

Why it works: `env!` reads an environment variable **while compiling**, and Cargo sets
`CARGO_PKG_VERSION` for every build. When you change the version in `Cargo.toml`, the answer changes
without editing code. Many real services have such an endpoint, so you can see which version is deployed.

</details>

### Exercise 3 — Test the real server (hard)

Exercise 1 tested the handler alone. Now test the **whole** service: router included. Move the router into
a function `fn app() -> Router`, so that `main` and the test can both use it. In the test, bind a listener
to `127.0.0.1:0` (port 0 means "any free port"), start `axum::serve` in a spawned task, connect with a
`tokio::net::TcpStream`, write a raw HTTP request (you did this by hand in Lesson 06.11) and check that the
response starts with `HTTP/1.1 200 OK` and ends with `ok\n`.

<details>
<summary>Hint</summary>

`listener.local_addr()` tells you which port the system chose. Send
`b"GET /health HTTP/1.1\r\nHost: test\r\nConnection: close\r\n\r\n"`; with `Connection: close` the
server closes the connection after the answer, so `read_to_string` knows where the end is. You need
`use tokio::io::{AsyncReadExt, AsyncWriteExt};`.

</details>

<details>
<summary>Answer</summary>

```rust
// The `use` lines, VISITS, health, slow and visits stay as before.

fn app() -> Router {
    Router::new()
        .route("/health", get(health))
        .route("/slow", get(slow))
        .route("/visits", get(visits))
}

#[tokio::main]
async fn main() {
    let port = std::env::args().nth(1).unwrap_or_else(|| "3000".to_string());
    let addr = format!("127.0.0.1:{port}");
    let listener = tokio::net::TcpListener::bind(&addr).await.unwrap();
    println!("listening on http://{addr}");
    axum::serve(listener, app()).await.unwrap();
}

#[cfg(test)]
mod tests {
    use super::*;
    use tokio::io::{AsyncReadExt, AsyncWriteExt};

    #[tokio::test]
    async fn real_server_answers_health() {
        // Port 0 asks the operating system for any free port.
        let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(async move {
            axum::serve(listener, app()).await.unwrap();
        });

        let mut stream = tokio::net::TcpStream::connect(addr).await.unwrap();
        stream
            .write_all(b"GET /health HTTP/1.1\r\nHost: test\r\nConnection: close\r\n\r\n")
            .await
            .unwrap();
        let mut response = String::new();
        stream.read_to_string(&mut response).await.unwrap();

        assert!(response.starts_with("HTTP/1.1 200 OK"), "got: {response}");
        assert!(response.ends_with("ok\n"), "got: {response}");
    }
}
```

```text
running 1 test
test tests::real_server_answers_health ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: port 0 lets every test get its own free port, so tests never fight over port 3000 and can
run in parallel. The spawned task serves while the test acts as a client. This is a small **integration
test** of the whole HTTP path. Lesson 16.14 shows easier tools for this, but the idea stays the same.

</details>

## Quiz

1. Name two differences between a command-line program and a service.

<details>
<summary>Answer</summary>

A service runs for a long time and waits for requests; a CLI (command-line interface) program does one job and exits. A service
serves many clients at the same time; a CLI program has one user. A service answers an error to one
request and keeps running; a CLI program may exit on an error.

</details>

2. Why must the backend check rules that the frontend app already checks?

<details>
<summary>Answer</summary>

Because anybody can send requests to the backend directly (with `curl`, a script, a changed app). The
frontend check is only for the user's comfort; the backend check is the real protection.

</details>

3. Three requests to `/slow` (one second each) finished in one second. Why?

<details>
<summary>Answer</summary>

axum runs each connection in its own Tokio task. `tokio::time::sleep(...).await` gives control back to the
runtime, so the three tasks waited at the same time instead of one after another.

</details>

4. What changes if the handler uses `std::thread::sleep` instead?

<details>
<summary>Answer</summary>

It blocks the thread. With one runtime thread, the requests run one after another (3 seconds for three
requests). With more threads the problem is smaller but still real: each blocked thread cannot serve anyone.

</details>

5. What does "stateless service" mean, and why is it useful?

<details>
<summary>Answer</summary>

The service keeps no important data in its own memory; such data lives in a database, cache or broker. You
can restart any instance at any time, and you can run more instances to handle more load (horizontal
scaling), and every instance gives the same answers.

</details>

6. What do `404` and `405` mean, and who produced them in this lesson?

<details>
<summary>Answer</summary>

`404 Not Found`: no endpoint has this path. `405 Method Not Allowed`: the path exists, but not for this
method. The axum router produced both automatically; no handler ran.

</details>

## Summary

- A backend service is a long-running program that waits for requests over the network, answers many
  clients at the same time and keeps running when one request fails.
- The frontend shows data; the backend owns the data and the rules, and must check every rule itself.
- A backend system has parts: load balancer, service instances, database, cache, message broker and
  observability tools. This phase teaches each of them.
- `axum` turns async handler functions into an HTTP service: `Router::new().route(path, get(handler))`,
  then `axum::serve(listener, app)`.
- Every connection runs in its own Tokio task; `.await` lets one thread serve many waiting requests. A
  blocking call inside a handler destroys this.
- Keep services stateless: data that must survive a restart or be shared by instances lives outside the
  process.

## Sources and further reading

- [axum documentation](https://docs.rs/axum/0.8) — the official API reference with a short guide on the first page.
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) — the official Tokio guide; the chapters on spawning and on async in depth explain the tasks behind each connection.
- [The Twelve-Factor App — VI. Processes](https://12factor.net/processes) — the original text of the "stateless processes" rule.
- [MDN — An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview) — a clear reference for requests, responses and status codes.
- [curl manual](https://curl.se/docs/manpage.html) — every `curl` option, including `-i`, `-I`, `-s` and `-X`.

---

[🏠 Start](../START-HERE.md) · [← Phase 16 — Backend services](README.md) · [📋 Phase 16](README.md) · [16.2 Designing HTTP APIs: REST →](02-designing-http-apis.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
