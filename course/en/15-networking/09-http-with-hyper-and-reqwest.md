<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.8 HTTP from scratch](08-http-from-scratch.md) · [📋 Phase 15](README.md) · [15.10 DNS: how names become addresses →](10-dns-how-names-become-addresses.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/09-http-with-hyper-and-reqwest.md) <!-- nav -->

# Lesson 15.9 — HTTP with hyper and reqwest

> **Phase 15 — Networking** · Lesson 9 of 16 · ⏱ about 135 minutes

## What you will learn

- Why real programs use an HTTP library instead of a hand-written parser, and what `hyper` and `reqwest` each do.
- The shared `http` types: `Request`, `Response`, `Method`, `StatusCode`, headers and bodies.
- How to write an HTTP/1.1 server with `hyper`: a service function, a connection builder and the `TokioIo` adapter.
- How to read a request body safely, with a size limit.
- How to call HTTP servers with `reqwest`, with timeouts and status checks, and how to test a server end to end.

## Before you start

- You finished [Lesson 15.8 — HTTP from scratch](08-http-from-scratch.md): you know the request line, headers, the empty line, `Content-Length` and keep-alive.
- You know the Tokio accept loop from [Lesson 15.6 — Async networking with Tokio](06-async-networking-with-tokio.md) and tasks from [Lesson 14.13 — Tokio in depth: tasks and channels](../14-concurrency-and-async/13-tokio-tasks-and-channels.md).
- Create the practice package. This lesson needs a library and integration tests, so it gets its own package next to `code/p15-net`: several small programs in `src/bin/`, a library in `src/lib.rs` and tests in `tests/`:

```bash
cd ~/projects/rust
cargo new code/p15-http
cd code/p15-http
mkdir -p src/bin tests
rm src/main.rs
cargo add tokio@1 --features full
cargo add hyper@1 --features server,http1
cargo add hyper-util@0.1 --features tokio
cargo add http-body-util@0.1 bytes@1
cargo add reqwest@0.13
```

The first build compiles about 150 crates (mostly `reqwest` and its TLS code; TLS, Transport Layer Security, is the encryption layer of Lesson 10.11) and takes a few minutes. Later builds are fast.

## New words

| Word | Meaning |
|---|---|
| **HTTP** (HyperText Transfer Protocol) | The request/response protocol of the web. You wrote a small version of it in Lesson 15.8. |
| **`hyper`** | A fast, low-level HTTP library. It parses and writes HTTP/1.1 and HTTP/2 messages and manages connections. It does not route URLs for you. |
| **`reqwest`** | A high-level HTTP *client* built on `hyper`. It adds a connection pool, redirects, timeouts and HTTPS. |
| **HTTPS** (HTTP Secure) | HTTP inside a TLS-encrypted connection (Lesson 10.11). |
| **`http` crate** | A small crate with the common types `Request`, `Response`, `Method`, `StatusCode`, `HeaderMap`, `Uri`. `hyper`, `reqwest` and `axum` all use it. |
| **body** | The bytes after the headers. In `hyper` a body is a *stream* of pieces (frames), because it may be large or arrive slowly. |
| **frame** (of a body) | One piece of a body: a chunk of data, or trailers (headers sent after the body). |
| **service** | A function (or object) that turns a request into a future of a response. `hyper` calls it once per request. |
| **I/O** (input/output) | Reading and writing data: here, bytes to and from a socket. |
| **adapter** | A small wrapper type that makes one library's type fit another library's trait. `TokioIo` is one. |
| **connection pool** | A set of open connections that a client keeps and reuses, so it does not open a new TCP (Transmission Control Protocol) connection for every request. |
| **redirect** | A response with a status like `301` or `308` and a `Location` header that says "ask this other URL instead". |
| **URL** (Uniform Resource Locator) | A web address such as `http://127.0.0.1:3000/health`: scheme, host, port, path. |
| **ALPN** (Application-Layer Protocol Negotiation) | A field in the TLS (Transport Layer Security) handshake in which client and server agree to speak, for example, HTTP/2. Lesson 15.12 shows it in code. |
| **end-to-end test** | A test that starts the real server and talks to it through a real socket, like a user would. |
| **API, URI, GB** (Application Programming Interface, Uniform Resource Identifier, gigabyte) | The types and functions a library offers to your code; the general name for a URL or a path such as `/health`; about one billion bytes. |

## 1. Why use a library

In Lesson 15.8 you parsed `GET /path HTTP/1.1` yourself. That is the best way to *understand* HTTP, but a server for real users must also handle many details correctly:

- bodies sent with `Transfer-Encoding: chunked`, and bodies that arrive slowly, piece by piece;
- keep-alive and pipelining (several requests on one connection, answered in order);
- strange or hostile input: huge headers, invalid bytes, both `Content-Length` and `Transfer-Encoding`;
- HTTP/2, which is a binary protocol with many streams on one connection;
- correct `Date` and `Content-Length` headers in every response.

`hyper` does all of this and is used by thousands of services. Its job is only the **protocol**: it reads bytes from a connection, gives you a `Request`, takes your `Response` and writes bytes. Routing, JSON (JavaScript Object Notation, a text format for data) and middleware (code that runs around every request, such as logging) belong to frameworks built on top of it, such as `axum` (Lesson 16.3).

```text
 your code:   async fn route(Request) -> Response        (you write this)
                 ▲                      │
 hyper:       parse bytes ──────────────┘ write bytes     (protocol, keep-alive, limits)
                 ▲                      ▼
 tokio:       TcpStream (read)          TcpStream (write) (sockets, from Lesson 15.6)
```

The client side is the same story. `hyper` has a low-level client too, but `reqwest` wraps it in a friendly API, and that is what most programs use.

## 2. The `http` types

Every HTTP library in Rust shares the types of the `http` crate (`hyper` re-exports them), so you learn them once:

| Type | What it holds | Example |
|---|---|---|
| `Method` | the verb | `Method::GET`, `Method::POST` |
| `Uri` | the target of the request | `/upper?x=1`; `.path()` gives `/upper` |
| `StatusCode` | the result | `StatusCode::OK` (200), `StatusCode::NOT_FOUND` (404) |
| `HeaderMap` | all headers; names are case-insensitive | `headers()["content-type"]` |
| `Request<B>` | method, URI, version, headers and a body of type `B` | `Request<Incoming>` |
| `Response<B>` | status, version, headers and a body of type `B` | `Response<Full<Bytes>>` |

The body type is a **generic parameter** `B`. For a request that arrives from the network, `hyper` gives you `hyper::body::Incoming`: a stream that you read frame by frame. For a response whose bytes you already have in memory, you use `http_body_util::Full<Bytes>`: a body with exactly one data frame. `Bytes` is the cheaply clonable byte buffer from the `bytes` crate (Lesson 15.7).

## 3. A first hyper server

Create `src/bin/hello.rs`:

```rust
use std::convert::Infallible;
use std::net::SocketAddr;

use bytes::Bytes;
use http_body_util::Full;
use hyper::body::Incoming;
use hyper::server::conn::http1;
use hyper::service::service_fn;
use hyper::{Request, Response};
use hyper_util::rt::TokioIo;
use tokio::net::TcpListener;

async fn hello(req: Request<Incoming>) -> Result<Response<Full<Bytes>>, Infallible> {
    let text = format!("Hello! You asked for {}\n", req.uri().path());
    Ok(Response::new(Full::new(Bytes::from(text))))
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    let listener = TcpListener::bind(addr).await?;
    println!("listening on http://{addr}");

    loop {
        let (stream, peer) = listener.accept().await?;
        let io = TokioIo::new(stream);
        tokio::spawn(async move {
            let service = service_fn(hello);
            if let Err(err) = http1::Builder::new().serve_connection(io, service).await {
                eprintln!("connection from {peer} failed: {err}");
            }
        });
    }
}
```

**What happens:**

1. `hello` is the **service**. It receives one parsed request and returns a response. Its error type is `Infallible`, a type with no values: it says "this function never fails". `Response::new` makes a `200 OK` response with the given body.
2. `main` is the accept loop you know from Lesson 15.6: bind, accept, spawn one task per connection.
3. `TokioIo::new(stream)` is an **adapter**. `hyper` does not depend on Tokio; it has its own I/O traits `hyper::rt::Read` and `hyper::rt::Write`. `TokioIo` from `hyper-util` implements them for any Tokio stream.
4. `service_fn(hello)` turns the plain async function into something that implements `hyper`'s `Service` trait.
5. `http1::Builder::new().serve_connection(io, service)` returns a future that serves **all** requests of this connection (keep-alive) and finishes when the client closes it. Errors (for example garbage bytes) are logged, and only this one connection ends.

Run it in one terminal:

```bash
cargo run --bin hello
```

```text
listening on http://127.0.0.1:3000
```

In a second terminal, send a request. `-i` prints the status line and the headers:

```bash
curl -i http://127.0.0.1:3000/rust
```

```text
HTTP/1.1 200 OK
content-length: 27
date: Wed, 18 Nov 2026 07:05:07 GMT

Hello! You asked for /rust
```

You did not write `content-length` or `date`: `hyper` added both. Now ask for two URLs in one `curl` command and look at the connection lines of the verbose output (`-v`):

```bash
curl -sv http://127.0.0.1:3000/a http://127.0.0.1:3000/b 2>&1 | grep -E '^\* (Connected|Re-using)|^> GET|^Hello'
```

```text
* Connected to 127.0.0.1 (127.0.0.1) port 3000
> GET /a HTTP/1.1
Hello! You asked for /a
* Re-using existing connection with host 127.0.0.1
> GET /b HTTP/1.1
Hello! You asked for /b
```

Both requests went over **one** TCP connection, so `serve_connection` called `hello` twice. Keep-alive works without any code from you. Stop the server with `Ctrl+C`.

## 4. Routing and a body with a size limit

A real server looks at the method and the path and decides what to do. We put this logic into the library (`src/lib.rs`), so that a program *and* the tests can use it:

```rust
use std::convert::Infallible;

use bytes::Bytes;
use http_body_util::{BodyExt, Full, LengthLimitError, Limited};
use hyper::body::Incoming;
use hyper::server::conn::http1;
use hyper::service::service_fn;
use hyper::{Method, Request, Response, StatusCode};
use hyper_util::rt::TokioIo;
use tokio::net::TcpListener;

/// The largest request body we accept, in bytes.
pub const MAX_BODY: usize = 1024;

/// Builds a plain-text response with the given status.
fn text(status: StatusCode, body: impl Into<Bytes>) -> Response<Full<Bytes>> {
    Response::builder()
        .status(status)
        .header("content-type", "text/plain; charset=utf-8")
        .body(Full::new(body.into()))
        .expect("status and header are valid")
}

/// Chooses what to do from the method and the path of the request.
pub async fn route(req: Request<Incoming>) -> Result<Response<Full<Bytes>>, Infallible> {
    let response = match (req.method(), req.uri().path()) {
        (&Method::GET, "/health") => text(StatusCode::OK, "ok\n"),
        (&Method::POST, "/upper") => {
            let limited = Limited::new(req.into_body(), MAX_BODY);
            match limited.collect().await {
                Ok(collected) => {
                    let body = collected.to_bytes();
                    text(StatusCode::OK, body.to_ascii_uppercase())
                }
                Err(err) if err.is::<LengthLimitError>() => {
                    text(StatusCode::PAYLOAD_TOO_LARGE, "body is too large\n")
                }
                Err(_) => text(StatusCode::BAD_REQUEST, "could not read the body\n"),
            }
        }
        (_, "/health" | "/upper") => text(StatusCode::METHOD_NOT_ALLOWED, "wrong method\n"),
        _ => text(StatusCode::NOT_FOUND, "not found\n"),
    };
    Ok(response)
}

/// Accepts connections forever and serves each one in its own task.
pub async fn serve(listener: TcpListener) -> std::io::Result<()> {
    loop {
        let (stream, peer) = listener.accept().await?;
        tokio::spawn(async move {
            let io = TokioIo::new(stream);
            if let Err(err) = http1::Builder::new()
                .serve_connection(io, service_fn(route))
                .await
            {
                eprintln!("connection from {peer} failed: {err}");
            }
        });
    }
}
```

**What happens:**

1. `text` uses `Response::builder()`, which lets you set the status and headers step by step. `.body(…)` returns a `Result` because a builder could get an invalid header; ours are fixed and valid, so `expect` is fine here.
2. `route` matches on the pair `(method, path)`. A `match` on a tuple is a simple router: each arm is one endpoint.
3. For `POST /upper`, `req.into_body()` takes the body out of the request. `Limited::new(body, MAX_BODY)` wraps it: as soon as more than 1024 bytes arrive, reading fails with a `LengthLimitError`.
4. `.collect()` (from the `BodyExt` trait) reads all frames and joins them; `.to_bytes()` gives one `Bytes`. We return the text in upper case.
5. `err.is::<LengthLimitError>()` checks *which* error happened: too large → `413 Payload Too Large`; anything else (the client vanished in the middle) → `400 Bad Request`.
6. A known path with the wrong method gets `405 Method Not Allowed`; every other path gets `404 Not Found`.
7. `serve` is the accept loop from Section 3, now inside the library and with `route` as the service.

Create the small program `src/bin/server.rs` that uses the library (the package is called `p15-http`, so in Rust code the library is `p15_http`):

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:3000").await?;
    println!("listening on http://{}", listener.local_addr()?);
    p15_http::serve(listener).await
}
```

Start it with `cargo run --bin server` and try every route from the second terminal. The last command sends 2000 bytes of the letter `a`:

```bash
curl -s http://127.0.0.1:3000/health
curl -s -d 'make me loud' http://127.0.0.1:3000/upper; echo
curl -si -X DELETE http://127.0.0.1:3000/health | head -1
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3000/missing
head -c 2000 /dev/zero | tr '\0' a | curl -s -w '%{http_code}\n' --data-binary @- http://127.0.0.1:3000/upper
```

```text
ok
MAKE ME LOUD
HTTP/1.1 405 Method Not Allowed
404
body is too large
413
```

And send bytes that are not HTTP at all, with `nc` (netcat, from Lesson 11.11):

```bash
printf 'GARBAGE\r\n\r\n' | nc 127.0.0.1 3000
```

```text
HTTP/1.1 400 Bad Request
connection: close
content-length: 0
date: Wed, 18 Nov 2026 07:05:18 GMT

```

The server terminal shows `connection from 127.0.0.1:51829 failed: invalid HTTP method parsed` (your port number will differ). `hyper` answered with `400`, closed only that connection, and the server keeps running.

> 🔒 **Security:** never read a body without a limit. Without `Limited`, `collect()` keeps everything in memory, and one client that sends a body of 10 GB can make your server run out of memory. Pick a limit that fits the endpoint (a few KiB, kibibytes of 1024 bytes, for a form; more for an upload) and answer `413` when it is crossed.

## 5. A client with reqwest

Keep the server running. Create `src/bin/client.rs`:

```rust
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let client = reqwest::Client::builder()
        .timeout(Duration::from_secs(5))
        .user_agent("p15-http-client/0.1")
        .build()?;

    let resp = client.get("http://127.0.0.1:3000/health").send().await?;
    println!("GET  /health -> {} {:?}", resp.status(), resp.version());
    println!("     content-type: {:?}", resp.headers()["content-type"]);
    println!("     body: {:?}", resp.text().await?);

    let resp = client
        .post("http://127.0.0.1:3000/upper")
        .body("hello, hyper")
        .send()
        .await?;
    println!(
        "POST /upper  -> {} body: {:?}",
        resp.status(),
        resp.text().await?
    );

    let resp = client.get("http://127.0.0.1:3000/nope").send().await?;
    println!("GET  /nope   -> {}", resp.status());

    let resp = client.get("https://www.rust-lang.org/").send().await?;
    println!(
        "GET  https://www.rust-lang.org/ -> {} {:?}",
        resp.status(),
        resp.version()
    );
    Ok(())
}
```

**What happens:**

1. `Client::builder()` configures a client once: a **timeout** for the whole request (connect, send, wait, read) and a `User-Agent` header that tells servers who is calling. `build()` creates the client and its connection pool.
2. `client.get(url).send().await?` sends the request and waits for the status line and the headers. The body is not read yet.
3. `resp.status()`, `resp.version()` and `resp.headers()` look at the head of the response. `resp.text().await?` reads the whole body as a `String`. It takes `resp` by value, so it comes last.
4. `.post(url).body("…")` sends a body; `reqwest` sets `Content-Length` for you.
5. The `404` is **not** an error: the `?` passes, because the HTTP exchange itself worked. Section "Common mistakes" shows how to turn bad statuses into errors.
6. The last request uses HTTPS. `reqwest` brings its own TLS (it uses `rustls`, the library from Lesson 10.11), checks the certificate, follows the redirect from `www.rust-lang.org` to `rust-lang.org`, and agrees on HTTP/2 through ALPN.

```bash
cargo run --bin client
```

```text
GET  /health -> 200 OK HTTP/1.1
     content-type: "text/plain; charset=utf-8"
     body: "ok\n"
POST /upper  -> 200 OK body: "HELLO, HYPER"
GET  /nope   -> 404 Not Found
GET  https://www.rust-lang.org/ -> 200 OK HTTP/2.0
```

Our server speaks only HTTP/1.1 (we enabled only the `http1` feature), while the public site answered with HTTP/2.0. The code of the client did not change: `reqwest` picks the version per connection.

> ⚡ **Performance:** create **one** `reqwest::Client` and reuse it (it is cheap to clone, and the clones share one pool). The pool keeps connections open, so the second request to the same host skips the TCP handshake and the TLS handshake. Making a new `Client` for every request throws all of this away.

## 6. Testing the server end to end

Because `serve` takes a `TcpListener`, a test can bind to port `0`. The operating system then picks any free port, so tests never collide with a running server or with each other. Create `tests/api.rs`:

```rust
use tokio::net::TcpListener;

/// Starts the server on a free port and returns its base URL.
async fn start_server() -> String {
    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    tokio::spawn(p15_http::serve(listener));
    format!("http://{addr}")
}

#[tokio::test]
async fn health_says_ok() {
    let base = start_server().await;
    let resp = reqwest::get(format!("{base}/health")).await.unwrap();
    assert_eq!(resp.status(), 200);
    assert_eq!(resp.text().await.unwrap(), "ok\n");
}

#[tokio::test]
async fn upper_rejects_a_big_body() {
    let base = start_server().await;
    let big = "a".repeat(p15_http::MAX_BODY + 1);
    let resp = reqwest::Client::new()
        .post(format!("{base}/upper"))
        .body(big)
        .send()
        .await
        .unwrap();
    assert_eq!(resp.status(), 413);
}
```

**What happens:**

1. `#[tokio::test]` is like `#[test]`, but it starts a Tokio runtime for the test, so the test can be `async`.
2. `start_server` binds to `127.0.0.1:0`, asks for the real address with `local_addr()`, and starts `serve` in a background task. The server stops when the test's runtime stops.
3. `reqwest::get(url)` is a shortcut for "new client, GET, send". It is fine in tests.
4. The second test sends one byte more than the limit and expects `413`. A test for the limit protects you from a later change that removes it by accident.

```bash
cargo test --test api
```

```text
running 2 tests
test upper_rejects_a_big_body ... ok
test health_says_ok ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

> 🧪 **Testing:** you can also call `route` directly in a unit test, but you would need a `Request<Incoming>`, which only `hyper` can create from a real connection. Testing through a real socket on port `0` is simpler and tests more: parsing, keep-alive and your code together.

## Common mistakes

**1. The feature is not enabled.** `hyper` compiles almost nothing by default. With `hyper = "1"` and no features:

```text
error[E0433]: cannot find `server` in `hyper`
 --> src/lib.rs:6:12
  |
6 | use hyper::server::conn::http1;
  |            ^^^^^^ could not find `server` in `hyper`
  |
note: found an item that was configured out
  ...
157 |     #![feature = "server"]
    |        ------------------ the item is gated behind the `server` feature
```

The note says exactly which feature is missing. Fix: `cargo add hyper@1 --features server,http1`.

**2. Passing a Tokio stream directly to hyper.** If you write `serve_connection(stream, service)` without `TokioIo`:

```text
error[E0277]: the trait bound `tokio::net::TcpStream: hyper::rt::Read` is not satisfied
   --> src/bin/hello.rs:29:70
    |
 29 |             if let Err(err) = http1::Builder::new().serve_connection(io, service).await {
    |                                                     ---------------- ^^ the trait `hyper::rt::Read` is not implemented for `tokio::net::TcpStream`
    |                                                     |
    |                                                     required by a bound introduced by this call
    |
    = help: the following other types implement trait `hyper::rt::Read`:
              &mut T
              Box<T>
              Pin<P>
              TokioIo<T>
              ...
```

The help list even names `TokioIo<T>`. Fix: wrap the stream, `TokioIo::new(stream)`.

**3. Thinking that `404` is an `Err`.** `send().await?` fails only when the exchange fails (no connection, timeout, invalid response). A `404` or `500` is a *successful* exchange with a bad status, so your code continues as if all were fine. If a bad status should be an error, say so with `error_for_status()` (Exercise 2).

**4. No timeout.** A `reqwest::Client` without `.timeout(…)` waits forever for a server that accepted the connection but never answers. One stuck dependency then blocks your whole program. Always set a timeout that matches what you expect from the server.

## Try it yourself

1. Start `cargo run --bin server`.
2. Open a raw connection with `nc 127.0.0.1 3000` and type (press Enter after each line, and Enter twice at the end):

   ```text
   GET /health HTTP/1.1
   Host: x

   ```

3. You see `HTTP/1.1 200 OK`, the headers `content-type`, `content-length` and `date`, an empty line and `ok`. The connection stays open: type the same request again and you get a second answer. This is keep-alive, done by `hyper`. (Your Enter key sends only `\n`, not `\r\n`; `hyper` accepts both.)
4. Type `GET /health HTTP/1.1` with **no** `Host` line and press Enter twice. The HTTP/1.1 standard requires `Host`, but `hyper` is lenient here and still answers `200`; a strict proxy might answer `400`. Press `Ctrl+C` to close `nc`.

## Exercises

### Exercise 1 — A version endpoint (easy)

Add `GET /version` that answers `p15-http 0.1.0` and a newline, taking the version from `Cargo.toml` (not typed by hand). Write a test for it in `tests/api.rs`.

<details>
<summary>Hint</summary>

The macro `env!("CARGO_PKG_VERSION")` gives the version at compile time, and `concat!` joins string literals at compile time.

</details>

<details>
<summary>Answer</summary>

Add this arm to the `match` in `route`:

```rust
        (&Method::GET, "/version") => text(
            StatusCode::OK,
            concat!("p15-http ", env!("CARGO_PKG_VERSION"), "\n"),
        ),
```

And the test:

```rust
#[tokio::test]
async fn version_route_reports_the_package_version() {
    let base = start_server().await;
    let resp = reqwest::get(format!("{base}/version")).await.unwrap();
    assert_eq!(resp.status(), 200);
    assert_eq!(resp.text().await.unwrap(), "p15-http 0.1.0\n");
}
```

Why it works: Cargo sets `CARGO_PKG_VERSION` while it compiles your crate, so the text always matches `Cargo.toml`. `concat!` produces a `&'static str`, which `text` accepts because `&'static str` converts into `Bytes`.

</details>

### Exercise 2 — Bad statuses are errors (medium)

Write `src/bin/strict.rs`: for the paths `/health` and `/nope`, send a GET to the running server, and print either the body or the error. A status of 400 or more must become an `Err`.

<details>
<summary>Hint</summary>

`Response::error_for_status(self)` returns `Ok(response)` for statuses below 400 and `Err(reqwest::Error)` otherwise.

</details>

<details>
<summary>Answer</summary>

```rust
#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let client = reqwest::Client::new();
    for path in ["/health", "/nope"] {
        let url = format!("http://127.0.0.1:3000{path}");
        match client.get(&url).send().await?.error_for_status() {
            Ok(resp) => println!("{path}: {}", resp.text().await?.trim_end()),
            Err(err) => println!("{path}: error: {err}"),
        }
    }
    Ok(())
}
```

With the server running, `cargo run --bin strict` prints:

```text
/health: ok
/nope: error: HTTP status client error (404 Not Found) for url (http://127.0.0.1:3000/nope)
```

Why it works: `send().await?` handles transport errors; `error_for_status()` then turns the 404 into a `reqwest::Error` whose message names the status and the URL. We `match` on it instead of using `?` so that the loop goes on and prints both lines.

</details>

### Exercise 3 — A client that gives up (hard)

Add `GET /slow` that waits two seconds before it answers. Write a test that calls it with a client whose timeout is 300 milliseconds and checks that the error is a timeout.

<details>
<summary>Hint</summary>

`tokio::time::sleep(Duration::from_secs(2)).await` inside the arm. In the test, `send().await.unwrap_err()` gives the error, and `reqwest::Error` has a method `is_timeout()`.

</details>

<details>
<summary>Answer</summary>

Add `use std::time::Duration;` to `src/lib.rs` and the arm:

```rust
        (&Method::GET, "/slow") => {
            tokio::time::sleep(Duration::from_secs(2)).await;
            text(StatusCode::OK, "finally\n")
        }
```

The test:

```rust
#[tokio::test]
async fn client_gives_up_on_a_slow_route() {
    let base = start_server().await;
    let client = reqwest::Client::builder()
        .timeout(std::time::Duration::from_millis(300))
        .build()
        .unwrap();
    let err = client.get(format!("{base}/slow")).send().await.unwrap_err();
    assert!(err.is_timeout(), "expected a timeout, got {err}");
}
```

All four tests (two from Section 6 and two from the exercises) pass:

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.32s
```

Why it works: the client's timer fires after 300 ms and cancels the request future (cancellation, Lesson 14.14). The test finishes in about 0.3 seconds, not 2, which also shows that the server's slow task does not block the test.

</details>

## Quiz

1. What is the job of `hyper`, and what is *not* its job?

<details>
<summary>Answer</summary>

`hyper` implements the HTTP protocol: it parses requests, writes responses, handles keep-alive, chunked bodies and HTTP/2. It does not do routing, JSON, authentication or middleware; frameworks such as `axum` add those on top.

</details>

2. Why does `serve_connection` need `TokioIo::new(stream)` and not just `stream`?

<details>
<summary>Answer</summary>

`hyper` is independent of any async runtime and defines its own I/O traits (`hyper::rt::Read`/`Write`). A Tokio `TcpStream` implements Tokio's traits, not hyper's. `TokioIo` is the adapter from `hyper-util` that connects the two.

</details>

3. What does `Limited::new(body, 1024)` protect against, and what status should the server answer when the limit is crossed?

<details>
<summary>Answer</summary>

It protects against a client sending a huge body that would fill the server's memory. When more than 1024 bytes arrive, reading fails with `LengthLimitError`, and the server should answer `413 Payload Too Large`.

</details>

4. `client.get(url).send().await?` returned a response with status 500. Did the `?` return an error? How do you make it one?

<details>
<summary>Answer</summary>

No. The exchange worked, so `send` returned `Ok(response)` with status 500. Call `.error_for_status()` on the response to turn 4xx and 5xx statuses into an `Err`.

</details>

5. Why should a program reuse one `reqwest::Client`?

<details>
<summary>Answer</summary>

The client owns a connection pool. Reusing it lets later requests to the same host reuse open connections and skip the TCP and TLS handshakes. A new client per request opens a new connection every time.

</details>

## Summary

- `hyper` is the HTTP protocol engine; `reqwest` is a friendly client on top of it; both use the types of the `http` crate.
- A `hyper` server is the Tokio accept loop plus `http1::Builder::new().serve_connection(TokioIo::new(stream), service_fn(f))` in a task per connection.
- The service turns `Request<Incoming>` into `Response<Full<Bytes>>`; a `match` on `(method, path)` is a simple router.
- Request bodies are streams; read them with `Limited` and `collect()`, and answer `413` when they are too large.
- `reqwest::Client` gives you pooling, redirects, HTTPS and timeouts; always set a timeout and decide how to treat bad statuses (`error_for_status`).
- End-to-end tests bind the server to port `0` and call it with `reqwest`.

## Sources and further reading

- [hyper guide: Getting Started with a Server](https://hyper.rs/guides/1/server/hello-world/) — the official step-by-step server guide for hyper 1.
- [hyper documentation](https://docs.rs/hyper/latest/hyper/) — the API reference, including the list of Cargo features.
- [hyper-util documentation](https://docs.rs/hyper-util/latest/hyper_util/) — `TokioIo`, and the `server::conn::auto` builder that serves HTTP/1 and HTTP/2 on the same port.
- [http-body-util documentation](https://docs.rs/http-body-util/latest/http_body_util/) — `Full`, `Limited`, `BodyExt::collect` and other body helpers.
- [reqwest documentation](https://docs.rs/reqwest/latest/reqwest/) — the client API, timeouts, redirects and optional features such as `json`.
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) — the official meaning of methods, status codes and headers.

---

[🏠 Start](../START-HERE.md) · [← 15.8 HTTP from scratch](08-http-from-scratch.md) · [📋 Phase 15](README.md) · [15.10 DNS: how names become addresses →](10-dns-how-names-become-addresses.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
