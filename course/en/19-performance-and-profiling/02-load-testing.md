<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.1 Latency, throughput and percentiles](01-latency-throughput-and-percentiles.md) · [📋 Phase 19](README.md) · [19.3 Profilers and flame graphs →](03-profilers-and-flame-graphs.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/02-load-testing.md) <!-- nav -->

# Lesson 19.2 — Load testing services

> **Phase 19 — Performance and profiling** · Lesson 2 of 11 · ⏱ about 120 minutes

## What you will learn

- What a **load test** is, and the difference between a **closed model** and an **open model** load generator.
- How to put load on a web service with `oha` and read every part of its report.
- How to find the **saturation point** of a service: the load at which latency starts to explode.
- Why one blocking call inside an `async` handler can slow down *every* endpoint, and how a load test shows it.

## Before you start

- You finished [Lesson 19.1 — Latency, throughput and percentiles](01-latency-throughput-and-percentiles.md): percentiles, Little's law and coordinated omission are used everywhere here.
- You can write an axum service and test its router with `oneshot` ([Lesson 16.3 — Web services with axum](../16-backend-services/03-web-services-with-axum.md)).
- You know Tokio tasks and `spawn_blocking` ([Lesson 14.13 — Tokio in depth: tasks and channels](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)).
- You need two terminal windows: one runs the service, the other runs the load generator.

## New words

| Word | Meaning |
|---|---|
| **load test** | Running a service under many requests to see how its latency and throughput behave. |
| **load generator** | The program that sends the requests in a load test. Here: `oha`. |
| **connection** | One open TCP (Transmission Control Protocol) link between client and server. With **keep-alive**, many requests reuse one connection. |
| **concurrency** | How many requests are in progress at the same time. In `oha`, the `-c` option sets the number of connections working at once. |
| **closed model** | A generator with a fixed number of users: each user sends a request, waits for the answer, then sends the next one. Load drops when the server gets slow. |
| **open model** | A generator that sends requests at a fixed **rate** (for example 300 per second), whether or not earlier answers have arrived, like real users on the internet. |
| **QPS** (Queries Per Second) | The rate of requests a generator sends. Same idea as req/s. |
| **saturation** | The state in which a resource (CPU (Central Processing Unit) cores, threads, connections) is busy all the time. More load then only makes the queue longer. |
| **knee** | The point on a "load → latency" curve where latency stops being flat and starts to grow fast. |
| **capacity** | The highest load a service can carry while still meeting its latency goal (for example p99 below 50 ms). |
| **TUI** (Text User Interface) | A screen inside the terminal that redraws itself, with bars and numbers. `oha` shows one unless you pass `--no-tui`. |
| **DNS** (Domain Name System) | The system that turns a host name such as `example.com` into an IP (Internet Protocol) address. |
| **DoS** (Denial of Service) | An attack (or an accident) that makes a service unusable by giving it more work than it can do. |

## 1. What a load test is

A benchmark (Lesson 09.11) measures one piece of code alone. A **load test** measures a whole running service from the outside: the network stack, the HTTP (HyperText Transfer Protocol) parser, the async runtime, your handlers, all at once, under many requests at the same time. It answers questions like:

- How many requests per second can this service handle on this machine?
- What are p50 and p99 at 1000 req/s? At 10 000 req/s?
- At what load does it **saturate**, and what happens then: does it slow down gently or fall over?

There are a few kinds of load tests. A **smoke test** sends a little load just to see that everything works. A **load test** runs the expected traffic. A **stress test** pushes beyond it to find the limit. A **soak test** runs normal load for hours to find slow leaks. In this lesson we do the first three on your Mac.

The load generator itself can work in two ways, and the difference matters (remember coordinated omission from Lesson 19.1):

```text
closed model (oha -c 10)                 open model (oha -q 300)
10 "users", each one:                    a clock sends a request every 1/300 s:
  send -> wait for answer -> send ...      send, send, send, send ...  (does not wait)
server slow  => fewer requests sent      server slow  => requests pile up, latency grows
good for: "how much can it do?"          good for: "what do users feel at this traffic?"
```

> 🔒 **Security:** load tests are a form of DoS. Run them only against services you own, on your own machine, in your VM (virtual machine) `dev` (Phase 11) or in a test environment that you are allowed to use. Never point a load generator at someone else's website or API (Application Programming Interface: the addresses through which programs talk to a service).

## 2. A service to test

We build a small service with four endpoints that behave in very different ways. Create the package:

```bash
cd ~/projects/rust/code
cargo new p19-load
cd p19-load
cargo add axum@0.8
cargo add tokio@1 --features full
cargo add serde@1 --features derive
cargo add --dev tower@0.5 --features util
cargo add --dev http-body-util@0.1
```

`src/main.rs`:

```rust
use std::time::Duration;

use axum::{Router, extract::Query, routing::get};
use serde::Deserialize;

/// Pretend CPU work: `n` thousand rounds of arithmetic. n = 100 takes about 0.1 ms.
fn checksum(n: u64) -> u64 {
    let mut x = n;
    for _ in 0..n * 1000 {
        x = x.wrapping_mul(6364136223846793005).wrapping_add(1);
    }
    x
}

#[derive(Deserialize)]
struct WorkParams {
    n: Option<u64>,
}

async fn hello() -> &'static str {
    "hello\n"
}

async fn work(Query(params): Query<WorkParams>) -> String {
    let n = params.n.unwrap_or(100).min(10_000);
    format!("{}\n", checksum(n))
}

/// Waits 20 ms without using a thread, like a call to a database.
async fn wait() -> &'static str {
    tokio::time::sleep(Duration::from_millis(20)).await;
    "waited\n"
}

/// WRONG on purpose: blocks a whole runtime thread for 20 ms.
async fn blocking() -> &'static str {
    std::thread::sleep(Duration::from_millis(20));
    "blocked\n"
}

fn app() -> Router {
    Router::new()
        .route("/hello", get(hello))
        .route("/work", get(work))
        .route("/wait", get(wait))
        .route("/blocking", get(blocking))
}

#[tokio::main(worker_threads = 4)]
async fn main() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3019")
        .await
        .unwrap();
    println!("listening on http://127.0.0.1:3019 with 4 worker threads");
    axum::serve(listener, app()).await.unwrap();
}
```

**What happens:**

1. `/hello` does almost nothing. It shows the cost of HTTP and the runtime alone.
2. `/work` burns CPU (Central Processing Unit) time: `checksum(100)` is about 0.1 ms of arithmetic. The query parameter `n` changes the amount.
3. `/wait` waits 20 ms with `tokio::time::sleep`. While it waits, the worker thread is free to serve other requests, exactly like waiting for a database answer.
4. `/blocking` waits 20 ms with `std::thread::sleep`. That stops the **whole worker thread**. This is a bug on purpose; Section 5 shows what it does.
5. `#[tokio::main(worker_threads = 4)]` fixes the runtime at 4 worker threads. The default is one per CPU core; a fixed number makes the results easier to understand and leaves cores free for the load generator, which runs on the same Mac.
6. The port is 3019, so it does not collide with the Phase 16 service on port 3000.

> 🔒 **Security:** `.min(10_000)` limits how much work one request may ask for. Without it, a single request `/work?n=99999999999` would keep a core busy for hours. Never let a client choose an unbounded amount of work.

In the first terminal, start the service **in release mode**:

```bash
cargo run --release
```

```text
listening on http://127.0.0.1:3019 with 4 worker threads
```

In the second terminal, check every endpoint once by hand. A load test of a broken endpoint measures nothing useful:

```bash
curl -s http://127.0.0.1:3019/hello
curl -s 'http://127.0.0.1:3019/work?n=100'
curl -s http://127.0.0.1:3019/wait
```

```text
hello
13404730972402714436
waited
```

## 3. The first load test with oha

`oha` is a load generator written in Rust. Install it (it takes a few minutes to compile):

```bash
cargo install --locked oha
oha --version
```

```text
oha 1.17.0
```

Send 2000 requests over 10 connections to `/hello`:

```bash
oha -n 2000 -c 10 --no-tui http://127.0.0.1:3019/hello
```

```text
Summary:
  Success rate:	100.00%
  Total:	32.3137 ms
  Slowest:	1.5878 ms
  Fastest:	0.0978 ms
  Average:	0.1469 ms
  Requests/sec:	61893.3159

  Total data:	11.72 KiB
  Size/request:	6 B
  Size/sec:	362.66 KiB

Response time histogram:
  0.098 ms [1]    |
  0.247 ms [1988] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.396 ms [1]    |
  0.545 ms [0]    |
  0.694 ms [0]    |
  0.843 ms [0]    |
  0.992 ms [0]    |
  1.141 ms [0]    |
  1.290 ms [0]    |
  1.439 ms [1]    |
  1.588 ms [9]    |

Response time distribution:
  10.00% in 0.1253 ms
  25.00% in 0.1310 ms
  50.00% in 0.1373 ms
  75.00% in 0.1453 ms
  90.00% in 0.1584 ms
  95.00% in 0.1760 ms
  99.00% in 0.1928 ms
  99.90% in 1.5784 ms
  99.99% in 1.5878 ms


Details (average, fastest, slowest):
  DNS+dialup:	0.7054 ms, 0.5873 ms, 0.7671 ms
  DNS-lookup:	0.0758 ms, 0.0095 ms, 0.1441 ms
  First byte:	0.1468 ms, 0.0978 ms, 1.5877 ms

Status code distribution:
  [200] 2000 responses
```

**What happens** (read the report from top to bottom):

1. `-n 2000` sends 2000 requests in total; `-c 10` keeps 10 connections busy at the same time (a closed model with 10 users). `--no-tui` prints a plain report instead of the live screen. Without it, `oha` draws a live view in the terminal with a progress bar, the current requests per second and a growing histogram, and prints the same report at the end.
2. **Summary.** *Success rate* is the share of requests that got any HTTP answer (check it first!). *Total* is the wall time of the whole test. *Requests/sec* is the throughput: about 62 000 req/s. Little's law agrees: 10 in progress ÷ 0.147 ms ≈ 68 000.
3. **Response time histogram.** The same kind of bucket picture as in Lesson 19.1: 1988 requests were around 0.25 ms or faster, and 10 were near 1.5 ms.
4. **Response time distribution.** The percentiles: p50 = 0.137 ms, p99 = 0.193 ms, p99.9 = 1.58 ms.
5. **Details.** *DNS+dialup* is the time to open a connection (name lookup plus TCP handshake), measured once per connection: 10 connections, 0.7 ms each. That explains the 10 slow requests: the first request on each connection also waited for the connection to open.
6. **Status code distribution.** All 2000 answers were `200 OK`. If you see `500` or `429` here, your throughput number counts errors, which are often very fast. A fast error is not a success.

> 💡 **Tip:** `oha` can also run for a fixed time with `-z 10s` instead of `-n`. At the end, requests that are still running are cancelled and reported as `aborted due to deadline`; that is normal and their number equals about the number of connections.

## 4. Finding the limit

A single load test tells you little. What you really want is the **curve**: how latency changes as load grows. Run `/work` with more and more connections, 5 seconds each:

```bash
for c in 1 2 4 8 16 64 256; do
  echo "--- $c connections"
  oha -z 5s -c "$c" --no-tui 'http://127.0.0.1:3019/work?n=100' | grep -E 'Requests/sec|50.00%|99.00%'
done
```

The output, collected into a table (one run; the full output has the same three lines for every step):

```text
connections   req/s     p50        p99
     1         7 020    0.137 ms   0.209 ms
     2         9 001    0.216 ms   0.295 ms
     4        24 869    0.153 ms   0.248 ms
     8        34 491    0.228 ms   0.382 ms
    16        34 199    0.459 ms   0.888 ms
    64        35 505    1.798 ms   3.275 ms
   256        34 683    7.195 ms  15.774 ms
```

Read it as a story:

- **1 to 8 connections:** throughput grows quickly, latency stays below half a millisecond. The four worker threads still have idle moments. (Very small loads are uneven: here 2 connections gained less than expected. Do not draw conclusions from a single row.)
- **8 connections and more:** throughput stops at about 35 000 req/s. The 4 worker threads are **saturated**: each request needs about 0.11 ms of CPU, so 4 threads can do at most 4 ÷ 0.11 ms ≈ 36 000 req/s.
- **After saturation**, more connections only add waiting. Little's law predicts it: 64 in progress ÷ 35 505 req/s = 1.8 ms, exactly the p50 we measured. With 256 connections, p99 is 16 ms: 80 times worse than at low load, for *no* gain in throughput.

That point where the curve bends, around 8 connections here, is the **knee**. A healthy service runs *below* its knee, with some room for traffic peaks.

```text
p99 latency
   |                                          *   256
   |
   |                             *  64
   |                *  16
   |  *  *  *  * 8          <- knee
   +--------------------------------------------- connections (load)
```

> ⚡ **Performance:** a service is usually "fast enough" long before it is saturated. Choose a latency goal first (for example, p99 below 1 ms), then find the highest load that meets it. That load is your **capacity**, and it is lower than the maximum throughput.

## 5. Waiting versus blocking

Now compare `/wait` and `/blocking`. Both take 20 ms. Only the way they wait is different. 5 seconds each, with 10 and 100 connections:

```bash
oha -z 5s -c 10 --no-tui http://127.0.0.1:3019/wait | grep -E 'Requests/sec|50.00%|99.00%'
oha -z 5s -c 100 --no-tui http://127.0.0.1:3019/wait | grep -E 'Requests/sec|50.00%|99.00%'
oha -z 5s -c 10 --no-tui http://127.0.0.1:3019/blocking | grep -E 'Requests/sec|50.00%|99.00%'
oha -z 5s -c 100 --no-tui http://127.0.0.1:3019/blocking | grep -E 'Requests/sec|50.00%|99.00%'
```

```text
  Requests/sec:	441.7495
  50.00% in 23.1151 ms
  99.00% in 23.8267 ms
  Requests/sec:	4395.1061
  50.00% in 22.8651 ms
  99.00% in 24.3705 ms
  Requests/sec:	153.8951
  50.00% in 67.2152 ms
  99.00% in 106.6218 ms
  Requests/sec:	176.0175
  50.00% in 0.6080 sec
  99.00% in 1.0819 sec
```

**What happens:**

1. `/wait`, 10 connections: 442 req/s, p50 23 ms (20 ms of sleep plus timer and scheduling delay). Little's law: 10 ÷ 0.023 s ≈ 435.
2. `/wait`, 100 connections: ten times more throughput, *same* latency. A sleeping task does not hold a thread, so 4 threads can keep thousands of waiting requests.
3. `/blocking`, 10 connections: only 154 req/s and p50 67 ms. Each request holds one of the 4 threads for 20 ms, so the service can never do more than 4 ÷ 0.020 s = 200 req/s. The other requests queue.
4. `/blocking`, 100 connections: still under 200 req/s, and now p50 is 0.6 **seconds**. Pure queueing.

It gets worse: a blocked thread cannot serve *any* endpoint. Start a load on `/blocking` in one terminal and measure `/hello` from another while it runs:

```bash
oha -z 8s -c 100 --no-tui http://127.0.0.1:3019/blocking > /dev/null &
sleep 2
oha -z 3s -c 1 --no-tui http://127.0.0.1:3019/hello | grep -E 'Requests/sec|50.00%|99.00%'
```

```text
  Requests/sec:	2.3311
  50.00% in 395.6002 ms
  99.00% in 673.4340 ms
```

`/hello` normally answers in 0.14 ms. Under load on `/blocking` it took 400 ms, and the throughput of a single connection fell from about 7000 req/s to 2. One wrong `sleep` (or a slow file read, or a heavy computation, or a blocking database driver) inside an async handler hurts the whole service. Unit tests never show this; a load test shows it at once. The fix is `tokio::task::spawn_blocking` (Exercise 2) or an async version of the call. Lesson 19.9 looks at async performance more closely.

## 6. The open model: a fixed rate

The closed model asks "how much can it do?". Real users do not wait for each other, though: they arrive at some rate. `-q 300` makes `oha` send 300 requests per second. `/blocking` can do fewer than 200 per second, so the queue must grow. Compare the report without and with `--latency-correction`:

```bash
oha -z 10s -c 50 -q 300 --no-tui http://127.0.0.1:3019/blocking | grep -E 'Requests/sec|50.00%|90.00%|99.00%'
oha -z 10s -c 50 -q 300 --latency-correction --no-tui http://127.0.0.1:3019/blocking | grep -E 'Requests/sec|50.00%|90.00%|99.00%'
```

```text
  Requests/sec:	169.0926
  50.00% in 298.8903 ms
  90.00% in 426.0512 ms
  99.00% in 510.9163 ms
  Requests/sec:	172.3733
  50.00% in 2.2162 sec
  90.00% in 3.9832 sec
  99.00% in 4.3683 sec
```

**What happens:**

1. Both runs reach only about 170 req/s although 300 were planned: the service is saturated.
2. Without correction, `oha` measures each request from the moment it was *really* sent. With 50 connections all busy, many planned requests were sent late, so their waiting before sending is not counted: p50 = 0.3 s. This is coordinated omission (Lesson 19.1).
3. `--latency-correction` measures from the moment each request was *planned*. Now p50 is 2.2 s and p99 4.4 s: the queue grew for the whole 10 seconds, and that is what users arriving at 300 per second would feel.

Use the open model with correction when the question is "what latency do users get at *this* traffic?". The answer to "what is the capacity?" is the highest rate whose corrected p99 still meets your goal (Exercise 3).

> 🧪 **Testing:** a load test is not a correctness test. Before you load a service, have ordinary tests that prove its answers are right (Exercise 1), and check *Success rate* and *Status code distribution* in every report.

## Common mistakes

1. **Load-testing a debug build.** With `cargo run` (debug), the same `/work` test with 16 connections gave 6 895 req/s and p50 2.3 ms instead of 34 199 req/s and 0.46 ms. Always `cargo run --release`.
2. **Ignoring errors.** A service that answers `500 Internal Server Error` in 50 µs looks very fast. Read the success rate and the status codes before the speed.
3. **One run, one number.** Results change from run to run and with everything else your Mac is doing. Repeat each step, close heavy programs, and keep the setup the same when you compare.
4. **The generator is the bottleneck.** `oha` needs CPU too. If the generator and the service share cores, you may be measuring the generator. Leave free cores (here: 4 worker threads on a 12-core Mac), watch both processes in Activity Monitor, and for serious numbers run the generator on another machine.
5. **Too short.** A 1-second test hides slow effects: caches filling up, memory growing, connection pools running out. Use at least 10–30 seconds for numbers you want to keep, and much longer for soak tests.
6. **Only the closed model.** A closed-model test can never overload the service, so it never shows how bad the queue gets. Add an open-model run with `-q` and `--latency-correction`.

## Try it yourself

1. Run `/work?n=1000` (about 1 ms of CPU per request) with 1, 4 and 16 connections. Where is the new maximum throughput? (Expect about 4 ÷ 1.1 ms, roughly 3600 req/s; one run gave 3767 req/s with 16 connections.)
2. Restart the service with `worker_threads = 8` and repeat the sweep from Section 4. Does the knee move?
3. Run `oha` without `--no-tui` against `/wait` for 10 seconds and watch the live screen. Press `q` to stop early.

## Exercises

### Exercise 1 — Test before you load (easy)

Add tests to `src/main.rs` that use `oneshot` (Lesson 16.3): `/work?n=0` answers `200` with the body `"0\n"`; `/work` without `n` answers `200` with `checksum(100)` followed by a newline; `/work?n=abc` answers `400 Bad Request`.

<details>
<summary>Hint</summary>

Write one helper `async fn get_text(uri: &str) -> (StatusCode, String)` that builds `Request::get(uri)`, calls `app().oneshot(request)`, and collects the body with `BodyExt::collect`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use axum::http::{Request, StatusCode};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    async fn get_text(uri: &str) -> (StatusCode, String) {
        let request = Request::get(uri).body(Body::empty()).unwrap();
        let response = app().oneshot(request).await.unwrap();
        let status = response.status();
        let bytes = response.into_body().collect().await.unwrap().to_bytes();
        (status, String::from_utf8(bytes.to_vec()).unwrap())
    }

    #[tokio::test]
    async fn work_returns_the_checksum() {
        assert_eq!(
            get_text("/work?n=0").await,
            (StatusCode::OK, "0\n".to_string())
        );
        let expected = format!("{}\n", checksum(100));
        assert_eq!(get_text("/work").await, (StatusCode::OK, expected));
    }

    #[tokio::test]
    async fn work_rejects_a_bad_number() {
        let (status, _) = get_text("/work?n=abc").await;
        assert_eq!(status, StatusCode::BAD_REQUEST);
    }
}
```

```text
running 2 tests
test tests::work_rejects_a_bad_number ... ok
test tests::work_returns_the_checksum ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: with `n = 0` the loop never runs and `x` stays 0. The `Query` extractor cannot turn `"abc"` into a `u64` and rejects the request with `400` before the handler runs.

</details>

### Exercise 2 — Fix the blocking handler (medium)

Change `blocking` so that the 20 ms `std::thread::sleep` runs inside `tokio::task::spawn_blocking`. Restart the service and run `oha -z 5s -c 100 --no-tui http://127.0.0.1:3019/blocking`. What happens to throughput and p50?

<details>
<summary>Hint</summary>

`spawn_blocking` takes a closure and returns a handle that you `.await`. The result is a `Result` (it is an error if the closure panicked).

</details>

<details>
<summary>Answer</summary>

```rust
/// Fixed: the blocking call runs on Tokio's separate pool for blocking work.
async fn blocking() -> &'static str {
    tokio::task::spawn_blocking(|| std::thread::sleep(Duration::from_millis(20)))
        .await
        .unwrap();
    "blocked\n"
}
```

```text
  Requests/sec:	4071.0477
  50.00% in 24.0089 ms
  99.00% in 31.3822 ms
```

Why it works: `spawn_blocking` moves the blocking call to a separate pool of threads (up to 512 by default), so the 4 worker threads stay free. Throughput rose from 176 to about 4 000 req/s, and p50 fell from 0.6 s to 24 ms, close to the `/wait` numbers. Extra threads are not free, though: for real waiting (network, database), an async API is still the better choice.

</details>

### Exercise 3 — Find the capacity (hard)

Goal: p99 below 60 ms. With the *unfixed* `/blocking` handler, find the highest rate (try `-q` 100, 150 and 200, 10 seconds each, 50 connections, with `--latency-correction`) that meets the goal. Explain the result with the numbers from Section 5.

<details>
<summary>Hint</summary>

The theoretical limit is 4 threads ÷ 20 ms = 200 req/s. Near a limit, queues grow without end, so expect a sudden jump.

</details>

<details>
<summary>Answer</summary>

```bash
for q in 100 150 200; do
  echo "--- $q per second"
  oha -z 10s -c 50 -q "$q" --latency-correction --no-tui http://127.0.0.1:3019/blocking | grep -E 'Requests/sec|50.00%|99.00%'
done
```

One run:

```text
--- 100 per second
  Requests/sec:	100.1663
  50.00% in 28.3066 ms
  99.00% in 45.2070 ms
--- 150 per second
  Requests/sec:	150.1711
  50.00% in 29.3505 ms
  99.00% in 52.1046 ms
--- 200 per second
  Requests/sec:	164.1685
  50.00% in 1.0870 sec
  99.00% in 2.0776 sec
```

Why it works: at 100 and 150 req/s the service keeps up, and p99 is about 45–52 ms, so the capacity for this goal is about 150 req/s. At 200 req/s the planned rate is at (in practice above) the real limit of about 170 req/s; the queue grows for the whole run and p99 jumps to 2 seconds. Capacity is always lower than the maximum throughput.

</details>

## Quiz

1. What is the difference between `oha -c 50` and `oha -q 500`?

<details>
<summary>Answer</summary>

`-c 50` is a closed model: 50 connections, each sends the next request only after the previous answer. `-q 500` sets an open-model rate: 500 requests per second are planned regardless of how fast answers come back (still sent over the connections from `-c`).

</details>

2. After the knee, what happens to throughput and to latency when you add more connections?

<details>
<summary>Answer</summary>

Throughput stays about the same (the service is saturated), and latency grows roughly in proportion to the number of connections, as Little's law says: latency = in progress ÷ throughput.

</details>

3. Why does `std::thread::sleep` in one handler slow down a different endpoint?

<details>
<summary>Answer</summary>

It blocks a runtime worker thread. While all worker threads are blocked, no task can run, including the tasks for other endpoints, so their requests wait in the queue.

</details>

4. A report says 80 000 req/s, success rate 100%, and the status code distribution shows 79 000 `[503]` responses. Is the service fast?

<details>
<summary>Answer</summary>

No. *Success rate* only means an HTTP answer arrived. Almost all answers were errors (`503 Service Unavailable`), which are cheap to produce. The useful throughput is about 1000 req/s.

</details>

5. Why should you not load-test with `cargo run` without `--release`?

<details>
<summary>Answer</summary>

A debug build is unoptimized; in this lesson it was about 5 times slower. You would find the limits of the debug build, not of the program you deploy.

</details>

6. What does `--latency-correction` change, and when does it matter?

<details>
<summary>Answer</summary>

It measures each request from the moment it was planned by the `-q` rate, not from the moment it was really sent. It matters when the service cannot keep up: then requests are sent late, and without the correction their waiting time is hidden (coordinated omission).

</details>

## Summary

- A **load test** measures a running service from the outside: throughput, percentiles and errors under concurrent requests.
- `oha -n N -c C` (closed model) shows what the service can do; `oha -q R --latency-correction` (open model) shows what users feel at a given rate.
- Always read the success rate and status codes first, then throughput, then percentiles.
- Sweep the load and find the **knee**: past saturation, throughput is flat and latency grows as Little's law predicts.
- **Capacity** is the highest load that meets your latency goal, and it is below the maximum throughput.
- A blocking call in an async handler blocks a worker thread and slows *every* endpoint; use `spawn_blocking` or async APIs.
- Load-test only your own services, in release mode, for long enough, and with free cores for the generator.

## Sources and further reading

- [oha on GitHub](https://github.com/hatoo/oha) — the load generator: all options, including `-q`, `--latency-correction` and the JSON (JavaScript Object Notation, a text data format) output.
- [`tokio::task::spawn_blocking`](https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html) — how Tokio runs blocking code on a separate pool.
- [axum documentation](https://docs.rs/axum/latest/axum/) — the web framework used for the service.
- [Grafana k6 — Open and closed models](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/) — a clear description of the two load models, from another load-testing tool.
- [Brendan Gregg — The USE Method](https://www.brendangregg.com/usemethod.html) — a checklist for saturation: utilization, saturation and errors of every resource.

---

[🏠 Start](../START-HERE.md) · [← 19.1 Latency, throughput and percentiles](01-latency-throughput-and-percentiles.md) · [📋 Phase 19](README.md) · [19.3 Profilers and flame graphs →](03-profilers-and-flame-graphs.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
