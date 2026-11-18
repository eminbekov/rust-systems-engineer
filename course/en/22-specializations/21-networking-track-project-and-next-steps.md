<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.20 Networking track: production proxies](20-networking-track-production-proxies.md) · [📋 Phase 22](README.md) · [22.22 Compilers track: lexing and parsing →](22-compilers-track-lexing-and-parsing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/21-networking-track-project-and-next-steps.md) <!-- nav -->

# Lesson 22.21 — Networking track: project and next steps

> **Phase 22 — Specializations** · Lesson 21 of 30 · ⏱ about 20 hours (10 sittings)

## What you will learn

- How to combine the track's pieces (socket options, zero-copy parsing, limits, load balancing) into one working HTTP/1.1 reverse proxy.
- How to handle the failure cases a proxy meets every day: bad requests, oversized heads, dead backends, slow backends.
- How to test a proxy end to end, in-process, with fake backends.
- Where to go next in networking: books, documentation, open-source projects and communities.

## Before you start

- You finished [Lesson 22.18](18-networking-track-high-performance-servers.md),
  [Lesson 22.19](19-networking-track-zero-copy-parsing.md) and
  [Lesson 22.20 — Networking track: production proxies](20-networking-track-production-proxies.md).
- You have the code of those lessons: `make_listener` and `serve_limited` (22.18), `parse.rs` and
  `read.rs` (22.19), and the test backend `backend.rs` (22.20).
- The proxy uses only Tokio, so it runs on macOS and in the VM `dev`. Run the performance measurements
  (milestone M6) in the VM.

## New words

| Word | Meaning |
|---|---|
| **status code** | The three-digit number at the start of an HTTP (Hypertext Transfer Protocol) response: `200` OK, `400` bad request, `502` bad gateway, and so on. |
| **`502 Bad Gateway`** | The status a proxy returns when it could not get a valid answer from any backend. |
| **`504 Gateway Timeout`** | The status a proxy returns when a backend did not answer in time. |
| **`431 Request Header Fields Too Large`** | The status for a request head that is larger than the server accepts. |
| **failover** | Trying another backend automatically when the chosen one fails. |
| **passive health checking** | Learning that a backend is down from failed real requests, instead of from separate test connections (active checks, Lesson 22.20). |
| **access log** | One line per request: method, path, status, backend, duration. |
| **in-process test** | A test that starts the server inside the test program (on port 0) instead of as a separate process. |

**Other abbreviations in this lesson:** VM (virtual machine; the Linux VM `dev`), CLI (command-line interface: how the
program is started and which arguments it takes), TCP (Transmission Control Protocol), TLS (Transport Layer Security,
the encryption layer of HTTPS) and URL (Uniform Resource Locator, a web address).

## 1. The project in one picture

```text
  curl / load generator
          │  HTTP/1.1 (keep-alive)
          ▼
 ┌────────────────────────────── p22-relay ───────────────────────────────┐
 │ listener (TCP_NODELAY, connection cap)                                 │
 │   └─ one task per client connection                                    │
 │        1. read_head (zero-copy parser, MAX_HEAD)  ── bad → 400 / 431   │
 │        2. read the body (Content-Length)                               │
 │        3. /__relay/stats? answer it here                               │
 │        4. rewrite the head (X-Forwarded-For, drop Connection)          │
 │        5. pick a backend: round robin, failover ── none → 502          │
 │        6. send, read the response head + body  ── too slow → 504      │
 │        7. add X-Relay, write to the client, log one line               │
 └────────────────────────────────────────────────────────────────────────┘
          │                         │
          ▼                         ▼
   backend 127.0.0.1:8001     backend 127.0.0.1:8002
```

Lesson 23.4 (capstone option 2) asks for a much bigger, high-performance reverse proxy. This project is
the small, honest version: every part is yours, and every failure case is handled on purpose.

## The project

### What you will build

`p22-relay`, an HTTP/1.1 reverse proxy built on Tokio from the pieces of this track. It forwards
requests to a list of backends with round robin, fails over when a backend is down, enforces limits,
reports simple statistics, and writes an access log.

```bash
cd ~/projects/rust
cargo new code/p22-relay
cd code/p22-relay
cargo add tokio@1 --features full
cargo add bytes@1
cargo add socket2@0.6
cargo add clap@4 --features derive
git init   # skip if code/ is already one repository
```

Copy `parse.rs` from `code/p22-net/src/` into `src/parse.rs`, and `backend.rs` from
`code/p22-proxy/src/bin/` into `src/bin/backend.rs`.

### Contract

**Command line:**

```text
Usage: p22-relay [OPTIONS] --backend <BACKENDS>

Options:
      --listen <LISTEN>              Address to listen on [default: 127.0.0.1:7400]
      --backend <BACKENDS>           A backend address; repeat for several backends
      --max-conns <MAX_CONNS>        Maximum number of client connections served at the same time [default: 1000]
      --timeout-secs <TIMEOUT_SECS>  Seconds to wait for a backend's answer [default: 5]
  -h, --help                         Print help
```

**Library API** (in `src/lib.rs`, so tests can start the proxy in-process):

```rust
pub struct Config {
    pub backends: Vec<std::net::SocketAddr>,
    pub max_conns: usize,
    pub upstream_timeout: std::time::Duration,
}

pub async fn run(listener: tokio::net::TcpListener, config: Config) -> std::io::Result<()>;
```

**Behaviour:**

1. Requests are HTTP/1.1 with an optional body announced by `Content-Length`. Client connections are
   keep-alive: one connection can carry many requests, one after another.
2. The proxy sends each request to the next backend in round-robin order. It replaces any
   `X-Forwarded-For` header with the client's IP (Internet Protocol) address and removes the
   `Connection` header.
3. Every response to the client carries the header `X-Relay: p22-relay`.
4. If a backend refuses the connection (or does not accept within 1 second), the proxy tries the next
   one. If none works: `502 Bad Gateway` with the body `no backend available\n`, then close.
5. If the chosen backend does not send a complete response within `--timeout-secs`:
   `504 Gateway Timeout` with the body `backend timed out\n`, then close.
6. A request head larger than 8 KiB (8,192 bytes): `431 Request Header Fields Too Large`, body
   `request head too large\n`. Any other unparsable request: `400 Bad Request`, body `bad request\n`.
   Both close the connection.
7. `GET /__relay/stats` is answered by the proxy itself: `200` with the body
   `requests=<N> errors=<M>\n` (all requests seen, and the requests answered with 400, 431, 502 or 504).
8. At most `--max-conns` client connections are served at once; extra connections wait (backpressure,
   Lesson 22.18).
9. One access-log line per request on standard output: `<method> <path> <status> <backend or -> <ms>ms`.

**Out of scope** (stretch goals): chunked bodies (`Transfer-Encoding: chunked`), HTTP/2, TLS, upstream
connection pooling.

### Step-by-step plan

#### M1 — Skeleton: listen and refuse politely

**Goal:** the CLI from the contract, a listener, an accept loop with one task per connection, and every
request answered with `502` (there is no forwarding yet).

**Guidance:** use `clap` with `#[derive(Parser)]` (Lesson 05.11); repeated `--backend` flags collect into a
`Vec<SocketAddr>` with `#[arg(long = "backend", required = true)]`. Put `Config` and `run` in `src/lib.rs`
and keep `main.rs` to parsing arguments and calling `run`. Set `TCP_NODELAY` on accepted connections
(Lesson 22.18). Use `read_head` from Lesson 22.19 to read each request head.

<details>
<summary>Hint</summary>

Write a helper `fn simple(status: &str, body: &str) -> String` that builds a complete response with
`Content-Length`, `Connection: close` and `X-Relay: p22-relay`. You will use it for every error status.

</details>

**Done when:**

```bash
cargo run -- --backend 127.0.0.1:8001     # in terminal 1
curl -si http://127.0.0.1:7400/hello      # in terminal 2
```

```text
HTTP/1.1 502 Bad Gateway
Content-Length: 21
Connection: close
X-Relay: p22-relay

no backend available
```

Commit: `git commit -am "M1: listener and 502 skeleton"`.

#### M2 — Forward one request

**Goal:** forward the request to the first backend and copy the response back.

**Guidance:** connect to the backend with `TcpStream::connect`. Write a new request head: the request
line, all headers except `X-Forwarded-For` and `Connection`, then `X-Forwarded-For: <client ip>` and the
empty line. Then read the response: its head ends with `\r\n\r\n`; the status code is the three digits
after `HTTP/1.1 `; the body length comes from `Content-Length`. Insert `X-Relay: p22-relay` just before
the empty line of the response head.

<details>
<summary>Hint</summary>

Read the response into a `BytesMut` until you find `\r\n\r\n`, `split_to` the head off, then read until
the buffer holds `Content-Length` bytes. Build the client response in one buffer and send it with one
`write_all` (remember the 40 ms lesson).

</details>

**Done when:** with `cargo run --bin backend -- 8001` running,

```bash
cargo run -- --backend 127.0.0.1:8001
curl -si http://127.0.0.1:7400/hello
```

```text
HTTP/1.1 200 OK
Content-Length: 43
Content-Type: text/plain
X-Relay: p22-relay

hello from backend 8001 (client 127.0.0.1)
```

and the relay prints `GET /hello 200 127.0.0.1:8001 0ms`. Commit.

#### M3 — Keep-alive and bodies

**Goal:** serve many requests on one client connection, and forward request bodies.

**Guidance:** loop in the connection task: after one response, parse the next head from the same buffer
(it may already be there, Lesson 22.19). After the head, read exactly `Content-Length` body bytes (0 if
the header is missing) and send them after the rewritten head.

<details>
<summary>Hint</summary>

`curl` reuses one connection when you give it several URLs in one command. The access log then shows
two requests, and `curl` prints both bodies.

</details>

**Done when:**

```bash
cargo run -- --backend 127.0.0.1:8001
curl -s http://127.0.0.1:7400/a http://127.0.0.1:7400/b
```

```text
hello from backend 8001 (client 127.0.0.1)
hello from backend 8001 (client 127.0.0.1)
```

and the access log shows `GET /a ...` and `GET /b ...` from one `curl` command. Commit.

#### M4 — Round robin and failover

**Goal:** spread requests over all backends, and skip backends that are down.

**Guidance:** keep the next index in an `AtomicUsize` shared by all connections (`fetch_add` then
`% backends.len()`). If `connect` fails or takes longer than one second (`tokio::time::timeout`), try
the next backend, at most once per backend. If all fail, answer `502` and count an error.

<details>
<summary>Hint</summary>

Put the backend loop in its own `async fn forward(...) -> std::io::Result<(SocketAddr, u16, Bytes)>`
that returns the backend, the status and the complete response. The connection task then only decides
which response to send.

</details>

**Done when:** with backends 8001 and 8002 running and a freshly started relay,

```bash
cargo run -- --backend 127.0.0.1:8001 --backend 127.0.0.1:8002
for i in 1 2 3 4; do curl -s http://127.0.0.1:7400/; done
```

```text
hello from backend 8001 (client 127.0.0.1)
hello from backend 8002 (client 127.0.0.1)
hello from backend 8001 (client 127.0.0.1)
hello from backend 8002 (client 127.0.0.1)
```

Stop backend 8002 (Ctrl+C) and run the loop again: all four answers come from 8001, with status 200.
Stop 8001 too: `curl -si http://127.0.0.1:7400/` prints the `502` response from M1. Commit.

#### M5 — Limits, timeouts and statistics

**Goal:** items 5–8 of the contract.

**Guidance:** wrap the whole upstream exchange (`forward`) in `tokio::time::timeout(config.upstream_timeout, ...)`.
Map `ParseError::TooLarge` to `431` and other parse errors to `400`. Count requests and errors in two
`AtomicU64`s inside a shared `Arc` state. Add the `Semaphore` from exercise 3 of Lesson 22.18 to the
accept loop.

<details>
<summary>Hint</summary>

A "backend" that accepts connections and never answers: `sleep 60 | nc -l 8009` (the `sleep` keeps the
input of `nc` open, so it does not close the connection). A head that is too large:
`python3 -c "print('GET / HTTP/1.1\r\nX-Big: ' + 'a'*9000 + '\r\n\r\n', end='')" | nc 127.0.0.1 7400`.
Make sure the 431 also happens when the whole big head arrives in one read.

</details>

**Done when:** each of these gives the shown result.

```bash
printf 'GARBAGE\r\n\r\n' | nc 127.0.0.1 7400
```

```text
HTTP/1.1 400 Bad Request
Content-Length: 12
Connection: close
X-Relay: p22-relay

bad request
```

```bash
python3 -c "print('GET / HTTP/1.1\r\nX-Big: ' + 'a'*9000 + '\r\n\r\n', end='')" | nc 127.0.0.1 7400 | head -1
```

```text
HTTP/1.1 431 Request Header Fields Too Large
```

With `sleep 60 | nc -l 8009` running in another terminal:

```bash
cargo run -- --listen 127.0.0.1:7401 --backend 127.0.0.1:8009 --timeout-secs 2
curl -si http://127.0.0.1:7401/slow
```

```text
HTTP/1.1 504 Gateway Timeout
Content-Length: 18
Connection: close
X-Relay: p22-relay

backend timed out
```

The `curl` command returns after about 2 seconds, and the relay logs `GET /slow 504 - 2002ms`.
`curl -s http://127.0.0.1:7400/__relay/stats` prints a line like `requests=13 errors=3`.
Finally, start the relay with `--max-conns 1`, hold one connection open with `sleep 5 | nc 127.0.0.1 7400`,
and run `curl -s --max-time 2 http://127.0.0.1:7400/`: it gives up with exit code 28 (timeout), because
the only place is taken. After the `nc` connection closes, the same `curl` succeeds. Commit.

#### M6 — Tests, quality and measurements

**Goal:** prove the proxy works, keep it clean, and measure it.

**Guidance:**

- Unit tests: the head rewriting (a pure function: input request, output bytes) and the response-head
  parsing (status code and `Content-Length`).
- An integration test `tests/end_to_end.rs`: start two fake backends on port 0 that answer `A` and `B`,
  start `p22_relay::run` on port 0 with them, send two requests, and check that you got one `A` and one
  `B` and that both responses carry `X-Relay`.
- In the VM, measure requests per second and p99 (99th percentile) latency through the relay with the
  load-testing tool from Lesson 19.2, then the same load directly against one backend, and then through
  the Pingora proxy of Lesson 22.20. Write the numbers, the command and the machine in the README.

<details>
<summary>Hint</summary>

A fake backend for tests is a few lines: bind `127.0.0.1:0`, accept in a loop, read once, write
`HTTP/1.1 200 OK\r\nContent-Length: 1\r\n\r\nA`. Start it with `tokio::spawn` before the relay.

</details>

**Done when:**

```bash
cargo test
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

all pass with no output from clippy and `fmt`; the test output contains your end-to-end test, for
example:

```text
test forwards_and_balances ... ok
```

and the `README.md` explains how to run the relay, lists the contract, and contains your measurement
table. Commit.

### Stretch goals

- **Upstream connection pooling:** keep idle backend connections in a pool per backend and reuse them.
  Measure the difference with your load test.
- **Chunked bodies:** support `Transfer-Encoding: chunked` in both directions (RFC 9112, section 7).
- **Thread-per-core mode:** a `--workers N` flag that starts N threads with `SO_REUSEPORT` listeners and
  current-thread runtimes (Lesson 22.18). Compare it with the default runtime.
- **Active health checks:** test every backend once per second in a background task and skip dead
  backends before a request fails.
- **Graceful shutdown:** on Ctrl+C stop accepting, finish open requests, then exit (Lesson 14.14).
- **Fuzz the request path** with `cargo fuzz` (Lesson 09.8): feed random bytes to the connection handler
  through `tokio::io::duplex`.

## Common mistakes

**Reading the response until the backend closes.** Backends keep connections open (keep-alive), so
"read until end of file" waits forever. Always use `Content-Length` (or, as a stretch goal, chunked
encoding) to know where a response ends.

**Forgetting bytes that are already in the buffer.** After one request, the next request (or part of
it) may already be in the client buffer. Parse before you read again, or pipelined requests hang
(Lesson 22.19, exercise 3).

**Holding the connection permit too long, or too briefly.** The `Semaphore` permit must live exactly as
long as the client connection: move it into the connection task. Dropping it right after `accept` turns
the cap off; keeping it in the accept loop blocks the server after the first connection.

**Logging the body or secret headers.** An access log must not contain request bodies, `Authorization`
headers or cookies (Lesson 10.8). Log the method, path, status, backend and duration only.

## Next steps in networking

You now know the core of a data-plane engineer's work. These resources take you further. Read the
documentation and the source code; they are the most accurate teachers in this field.

**Books and long reads**

- *High Performance Browser Networking* by Ilya Grigorik — free online; TCP, TLS, HTTP/1.1, HTTP/2 and
  their performance, explained very clearly.
- *The Linux Programming Interface* by Michael Kerrisk — the sockets chapters are the reference for what
  the kernel does under Tokio.
- *Beej's Guide to Network Programming* — a free, short guide to the sockets API; the examples are in C,
  but the system calls are the ones `socket2` wraps.

**Documentation and standards**

- The Tokio tutorial and the docs of `tokio`, `bytes`, `hyper` and `socket2`.
- RFC 9110 (HTTP semantics) and RFC 9112 (HTTP/1.1); RFC stands for Request for Comments, the official
  series of internet standards.

**Open-source projects to read and contribute to**

| Project | What it is | A good first step |
|---|---|---|
| Pingora | Cloudflare's proxy framework (Lesson 22.20) | read the examples, then look at issues labelled for newcomers |
| River | a reverse proxy built on Pingora | run it in your VM, improve documentation or tests |
| hyper | the HTTP library under most Rust web code | read its guides; issues tagged "easy" |
| Tokio | the async runtime | documentation improvements, small fixes |
| quinn | QUIC (a name, first read as "Quick UDP Internet Connections", where UDP is the User Datagram Protocol; the transport under HTTP/3) in Rust | read the tests to learn the protocol |
| rustls | TLS (Transport Layer Security) in Rust | issues about documentation and examples |
| hickory-dns | a DNS (Domain Name System) client and server | connects to your resolver project of Lesson 15.11 |
| Sōzu | a reverse proxy that reloads its configuration without restarts | read how it handles configuration changes |
| linkerd2-proxy | the data-plane proxy of the Linkerd service mesh | read how it uses Tower and Tokio |

Contributing well is a skill (Phase 24): read `CONTRIBUTING.md`, start with a small, useful change
(a test, a documentation fix, a reproduction of a bug), and ask in the project's chat before large work.

**Communities**

- The Tokio Discord server (channels for Tokio, hyper, tonic and more).
- The Rust users forum, users.rust-lang.org, for questions of any level.
- *This Week in Rust*, a weekly newsletter with networking news and "call for participation" issues.

> **End of the networking track.** When you finish the project, continue with
> [Lesson 22.30 — Checkpoint: your specialization](30-checkpoint-your-specialization.md). If you want a second track
> later, the next one starts at [Lesson 22.22 — Compilers track: lexing and parsing](22-compilers-track-lexing-and-parsing.md).

## Quiz

1. Why must the proxy parse the next request from the buffer before it reads from the socket again?

<details>
<summary>Answer</summary>

The client may have sent several requests at once (pipelining), so the next one can already be in the
buffer. If the proxy waits for a read first, it waits for bytes that will never come, and the client
waits for an answer: both hang.

</details>

2. Which status codes does `p22-relay` return for: a request it cannot parse, a 9,000-byte head, no
   reachable backend, a backend that never answers?

<details>
<summary>Answer</summary>

`400 Bad Request`, `431 Request Header Fields Too Large`, `502 Bad Gateway`, `504 Gateway Timeout`.

</details>

3. What is the difference between failover in M4 and the health checks of Lesson 22.20?

<details>
<summary>Answer</summary>

Failover is passive: the proxy notices a dead backend only when a real request fails to connect, then
tries the next one. Active health checks test every backend in the background, so a dead backend is
skipped before any request tries it.

</details>

4. Why does `run` take a `TcpListener` instead of an address?

<details>
<summary>Answer</summary>

A test can bind the listener to port `0`, read the real port with `local_addr()`, and then start the
proxy. With an address, the test would need a fixed free port, which makes tests collide and fail
randomly.

</details>

## Summary

- A reverse proxy is a loop of small, well-defined steps: read a head, read a body, choose a backend,
  forward, read the response, answer, log.
- Most of the code of a real proxy is about failures: invalid input (400), too much input (431), no
  backend (502), slow backend (504), too many clients (backpressure).
- Keep `run` in the library so tests can start the proxy in-process on port 0 with fake backends.
- Measure before and after every performance idea; compare with a direct connection and with a
  production proxy such as Pingora.
- Next: read hyper, Pingora and Tokio, follow the RFCs, and make a first small contribution.

## Sources and further reading

- [High Performance Browser Networking](https://hpbn.co/) — Ilya Grigorik's free book on network performance.
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) — the official introduction to Tokio, including framing.
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) — methods, status codes such as 502 and 504, and header rules.
- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112) — message syntax, `Content-Length`, chunked transfer coding.
- [The Linux Programming Interface](https://man7.org/tlpi/) — Michael Kerrisk's book; chapters 56–61 cover sockets.
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) — a free guide to the sockets API.
- [hyper](https://github.com/hyperium/hyper), [Tokio](https://github.com/tokio-rs/tokio), [River](https://github.com/memorysafety/river), [Sōzu](https://github.com/sozu-proxy/sozu), [quinn](https://github.com/quinn-rs/quinn), [rustls](https://github.com/rustls/rustls), [hickory-dns](https://github.com/hickory-dns/hickory-dns), [linkerd2-proxy](https://github.com/linkerd/linkerd2-proxy) — the projects from the table above.
- [Tokio Discord](https://discord.gg/tokio), [Rust users forum](https://users.rust-lang.org/), [This Week in Rust](https://this-week-in-rust.org/) — the communities.

---

[🏠 Start](../START-HERE.md) · [← 22.20 Networking track: production proxies](20-networking-track-production-proxies.md) · [📋 Phase 22](README.md) · [22.22 Compilers track: lexing and parsing →](22-compilers-track-lexing-and-parsing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
