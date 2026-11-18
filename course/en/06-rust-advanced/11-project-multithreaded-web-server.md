<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.10 FFI: calling C code](10-ffi-calling-c.md) · [📋 Phase 06](README.md) · [06.12 Checkpoint: Stage 1 complete →](12-checkpoint-stage-1-complete.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/11-project-multithreaded-web-server.md) <!-- nav -->

# Lesson 06.11 — Project: multithreaded web server

> **Phase 06 — Rust advanced** · Lesson 11 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- How a web server works at the lowest level: a TCP connection, an HTTP request as plain text, and a response.
- How to build a thread pool with a bounded queue, and how to shut it down cleanly.
- How to protect a server: size limits, timeouts, "queue full" answers, and safe file paths.
- How to organize a bigger project: a library plus a thin binary, unit tests, an integration test, an acceptance script.
- How to use Git for a project: a branch, one commit per milestone, a merge and a tag.

## Before you start

- You finished [Lesson 06.10 — FFI: calling C code](10-ffi-calling-c.md).
- You know threads, channels, `Arc` and `Mutex`: [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md). The thread pool is built from exactly these tools.
- You know modules and error handling: [Lesson 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md) and [Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md).
- You know `clap` (command-line arguments): [Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md). You may also parse arguments by hand.
- You know Git branches, merges and tags: [Lesson 01.6](../01-git-and-version-control/06-branches-and-merging.md) and [Lesson 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md). Your learning repository is at `~/projects/rust`.
- `curl` is installed (it is on every Mac). `curl --version` should print a version.

## New words

| Word | Meaning |
|---|---|
| **server** | A program that waits for other programs (clients) to connect and answers their requests. |
| **client** | A program that connects to a server and sends requests. `curl` and your browser are clients. |
| **IP address** (Internet Protocol address) | The number that names a computer on a network, for example `127.0.0.1`. |
| **localhost** / loopback | The address `127.0.0.1`: "this same computer". Only programs on your computer can connect to it. |
| **port** | A number (0–65535) that picks one program on a computer. Our server uses port `7878`. |
| **TCP** (Transmission Control Protocol) | The rules for a reliable, ordered stream of bytes between two programs. The operating system implements it. |
| **socket** | The handle your program uses to send and receive data over the network. |
| **HTTP** (HyperText Transfer Protocol) | The text-based language that browsers and web servers speak on top of TCP. |
| **request line** | The first line of an HTTP request: method, path and version, for example `GET /health HTTP/1.1`. |
| **header** | A `Name: value` line after the request line (or after the status line in a response). |
| **status code** | The number in a response that says what happened: `200` OK, `404` Not Found, and so on. |
| **path** | The part of a URL after the host, like `/static/note.txt`. |
| **URL** (Uniform Resource Locator) | A web address, like `http://127.0.0.1:7878/health`. |
| **thread pool** | A fixed group of worker threads that take jobs from a queue, so you do not start a new thread for every job. |
| **worker** | One thread of the pool. |
| **job** | One piece of work for the pool: here, "handle this connection". |
| **backpressure** | A way to say "slow down" when the system is full, instead of accepting unlimited work. |
| **graceful shutdown** | Stopping a program by finishing the work in progress first, instead of killing it in the middle. |
| **timeout** | A limit on how long you wait for something. |
| **path traversal** | An attack where a path like `/static/../secret` tries to leave the allowed folder. |
| **DoS** (Denial of Service) | An attack or accident that makes a service unavailable, for example by sending too much data. |
| **KiB** (kibibyte) | 1024 bytes. `8 KiB` = 8192 bytes. |
| **CLI** (Command-Line Interface) | A program used by typing commands and options in the terminal. |
| **SIGINT** | The signal the terminal sends when you press Ctrl-C. A signal is a message from the operating system to a program (Phase 13 explains signals in depth). |
| **acceptance test** | A check that shows the finished program does what the contract says. |
| **README** | A text file at the top of a project (`README.md`) that says what the project is and how to use it. |
| **API** (Application Programming Interface) | The functions and types that a library offers to the code that uses it. |
| **TLS** (Transport Layer Security) | Encryption for network connections (the "S" in HTTPS, which means HTTP Secure). Our server does not use it; Phase 15 explains it. |
| **CI** (Continuous Integration) | A server that runs your checks automatically on every push. Phase 09 teaches it. |
| **milestone** | One step of the plan with a clear "done when" check. |

## 1. How a web server works

### TCP and sockets

When you type a URL in the browser, three things happen: the browser opens a **TCP connection** to the server's IP address and port, sends an **HTTP request** as text through it, and reads an **HTTP response**. In Rust, `std::net` gives you the tools:

- `TcpListener::bind("127.0.0.1:7878")` starts listening on a port.
- `listener.accept()` (or the loop `listener.incoming()`) waits for a client and returns a `TcpStream`: the connection.
- A `TcpStream` implements `Read` and `Write`, so you use it like a file.

### HTTP is text

Here is what `curl` really sends. This is the real output of a tiny server that prints the bytes it receives (the starter program in section 3):

```text
GET / HTTP/1.1
Host: 127.0.0.1:7878
User-Agent: curl/8.7.1
Accept: */*

```

- The first line is the **request line**: method (`GET`), path (`/`), version (`HTTP/1.1`).
- Then come **headers**, one per line.
- An empty line ends the headers. (Every line ends with the two characters `\r\n`.)

The response has the same shape:

```text
HTTP/1.1 200 OK
Content-Length: 6
Connection: close

hello
```

A **status line**, headers, an empty line, and the body. `Content-Length` tells the client how many bytes of body to expect. `Connection: close` says "I will close the connection after this answer", which keeps our server simple: one request per connection.

### Why a thread pool

A server that handles clients **one at a time** is simple, but slow clients block everybody. Starting **a new thread per connection** is easy, but an attacker (or a busy day) can make you start ten thousand threads and run out of memory. A **thread pool** is the middle way:

```text
                     ┌─────────────────────┐
 accept loop         │  bounded queue      │        workers (fixed number)
 (main thread)       │  [job][job][job]    │       ┌─────────┐
   TcpListener ──►───┤  (sync_channel)     ├──►────┤ worker 0│─ handle connection
   accept()          │                     │   ┌───┤ worker 1│─ handle connection
        │            └─────────────────────┘   │   │ worker 2│─ handle connection
        │  queue full?                         │   │ worker 3│─ ...
        ▼  answer 503 right away               │   └─────────┘
                                          receiver is shared:
                                          Arc<Mutex<Receiver<Job>>>
```

- The main thread only **accepts** connections and puts a *job* ("handle this stream") into a queue.
- A fixed number of **workers** take jobs from the queue, one each, and run them.
- A channel (`mpsc`) has many senders but **one** receiver. To let many workers receive from it, the receiver is wrapped in `Arc<Mutex<...>>`: `Arc` shares it, `Mutex` lets one worker at a time take the next job.
- The queue has a **limit**. If it is full, the server answers `503 Service Unavailable` immediately instead of growing without limit. That is **backpressure**.
- To stop, you **drop the sender**. Every worker's `recv()` then returns an error, the worker loop ends, and the pool joins all threads. That is a **graceful shutdown**, and it is a natural job for `Drop`.

## 2. Plan your Git history

Treat this project like real work. In your learning repository (`~/projects/rust`):

1. Work on a **branch**, not on `main`: `git switch -c p06-webserver`.
2. Make **one commit per milestone**, with a message in the Conventional Commits style from [Lesson 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md): `feat(p06-webserver): M3 router and responses`. A small, well-named commit is easy to review and to undo.
3. At the end, **merge** the branch into `main` (with `git merge --no-ff`, or through a pull request if you use GitHub or GitLab) and **tag** the result: `git tag -a p06-webserver-v1.0 -m "..."`.
4. Make sure `target/` is ignored. Careful: when you run `cargo new` *inside* an existing Git repository, Cargo does not create a `.gitignore` for the new package, so the `.gitignore` at the root of your learning repository must contain the line `target/`.

Here is the flow, run in a throw-away repository. Your commit hashes will be different, and in this small example the milestones are numbered from M0, while the plan below starts at M1:

```sh
git switch -c p06-webserver
git add code/p06-webserver && git commit -m "feat(p06-webserver): M0 create the project"
# ... work, then after each milestone ...
git commit -am "feat(p06-webserver): M1 parse requests"
git commit -am "feat(p06-webserver): M2 router and responses"
git commit -am "feat(p06-webserver): M3 thread pool"
git log --oneline
git switch main
git merge --no-ff p06-webserver -m "Merge branch 'p06-webserver': multithreaded web server"
git tag -a p06-webserver-v1.0 -m "p06-webserver: all milestones done"
git log --oneline --graph --decorate
```

```text
ed5a1e7 feat(p06-webserver): M3 thread pool
7a22cba feat(p06-webserver): M2 router and responses
5370d74 feat(p06-webserver): M1 parse requests
1e7d04c feat(p06-webserver): M0 create the project
b675a56 chore: start learning repository
*   3363654 (HEAD -> main, tag: p06-webserver-v1.0) Merge branch 'p06-webserver': multithreaded web server
|\  
| * ed5a1e7 (p06-webserver) feat(p06-webserver): M3 thread pool
| * 7a22cba feat(p06-webserver): M2 router and responses
| * 5370d74 feat(p06-webserver): M1 parse requests
| * 1e7d04c feat(p06-webserver): M0 create the project
|/  
* b675a56 chore: start learning repository
```

> 💡 **Tip:** Run `git status` before each commit. It must not list `target/`. If a milestone broke the build, commit anyway on your branch with a message such as `wip: ...` and fix it in the next commit: history on a private branch can be messy. 

## 3. Warm-up: the smallest server

Create the project and run the smallest possible server. This is also milestone M1.

```sh
cd ~/projects/rust
git switch -c p06-webserver
cargo new code/p06-webserver
cd code/p06-webserver
```

`src/main.rs`:

```rust
use std::io::{Read, Write};
use std::net::TcpListener;

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7878")?;
    println!("Listening on http://127.0.0.1:7878");

    for stream in listener.incoming() {
        let mut stream = stream?;

        let mut buffer = [0u8; 1024];
        let n = stream.read(&mut buffer)?;
        println!("received {n} bytes");
        println!("{}", String::from_utf8_lossy(&buffer[..n]));

        let body = "hello\n";
        let response = format!(
            "HTTP/1.1 200 OK\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{}",
            body.len(),
            body
        );
        stream.write_all(response.as_bytes())?;
    }
    Ok(())
}
```

**What happens:**

1. `TcpListener::bind` asks the operating system to listen on `127.0.0.1:7878`. The `?` returns an error if the port is taken.
2. `listener.incoming()` is an iterator that waits (blocks) until a client connects, and yields a `TcpStream` for each connection.
3. `stream.read(&mut buffer)` reads up to 1024 bytes of the request into an array and returns how many it got. We print them.
4. We build the response text: status line, headers, empty line (`\r\n\r\n`), body. `write_all` sends it. When `stream` goes out of scope at the end of the loop body, the connection closes.

Run it in one terminal (`cargo run`) and use another terminal:

```sh
curl -i http://127.0.0.1:7878/
```

```text
HTTP/1.1 200 OK
Content-Length: 6
Connection: close

hello
```

The first terminal prints `received 77 bytes` and the request text you saw above. Commit: `git add -A . && git commit -m "feat(p06-webserver): M1 hello server"`.

Now an experiment that shows why the project exists. When this server (or any one-at-a-time server) needs 2 seconds for each request, the second client waits for the first. Your finished server, started with one worker and then with four, gives these real numbers for two simultaneous requests to the slow `/sleep` route:

```text
threads=1
  request 1 took 2.012805s
  request 2 took 4.012874s
threads=4
  request 1 took 2.016573s
  request 2 took 2.018657s
```

With one worker the second request waits 2 seconds in the queue. With four workers both finish together. By the end of the project you will reproduce this.

## The project

### What you will build

`p06-webserver`: a small HTTP/1.1 server that

- listens on a TCP port and handles connections on a **pool of worker threads**;
- serves a home page, a health check, a slow route, and files from a folder;
- answers errors correctly (400, 403, 404, 405, 408, 431, 503);
- protects itself with size limits, timeouts and a bounded queue;
- prints one **access log** line per request;
- stops gracefully when you press **Ctrl-C**.

It is a library (`src/lib.rs` with modules) plus a thin `main.rs`, with unit tests, an integration test, an acceptance script, a README, and a clean `cargo clippy` and `cargo fmt`.

### Contract

**Command line:**

```text
p06-webserver [--addr 127.0.0.1:7878] [--threads 4] [--queue 32] [--root ./public]
```

| Option | Meaning | Default |
|---|---|---|
| `--addr` | address and port to listen on | `127.0.0.1:7878` |
| `--threads` | number of worker threads (at least 1) | `4` |
| `--queue` | how many accepted connections may wait for a free worker | `32` |
| `--root` | folder that holds `hello.html` and the files for `/static/` | `./public` |

`--threads 0` prints `error: --threads must be at least 1` to standard error and exits with code `2`. If the address cannot be used (for example the port is taken), print `error: cannot listen on <addr>: <reason>` and exit with code `1`.

On start, print `Listening on http://<addr>` on standard output.

**Routes** (every response has `Content-Type`, `Content-Length` and `Connection: close`):

| Request | Response |
|---|---|
| `GET /` | `200`, the file `hello.html` from `--root`, as `text/html; charset=utf-8` |
| `GET /health` | `200`, body `ok` |
| `GET /sleep` | waits 2 seconds, then `200`, body `slept` + newline |
| `GET /static/<file>` | `200` and the file from `--root`; the `Content-Type` depends on the extension (`.html`, `.txt`, `.css`, `.json`; anything else `application/octet-stream`) |
| any other path | `404`, body `404 Not Found` + newline |
| a known path with a method other than `GET` | `405` with the header `Allow: GET` |

**Errors and limits:**

| Situation | Response |
|---|---|
| The request line is not `METHOD /path HTTP/x` (wrong number of parts, path without a leading `/`, version not starting with `HTTP/`) | `400` |
| Request line + headers larger than **8 KiB** | `431` |
| No complete request within **5 seconds** after connecting | `408` |
| The path of `/static/` leaves `--root` (`..`, an absolute path, a symbolic link out of the folder) | `403` |
| `/static/` file does not exist | `404` |
| All workers are busy and the queue is full | `503`, written by the accept loop itself |

**Access log** — one line on standard output per answered request:

```text
<ip> "<METHOD> <path>" <status> <body-bytes> <milliseconds>ms
```

For example `127.0.0.1 "GET /health" 200 2 0.1ms`. If the request could not be parsed, write `"-"` instead of the method and path: `127.0.0.1 "-" 431 36 0.1ms`.

**Shutdown:** on Ctrl-C (SIGINT) the server stops accepting new connections, lets the running requests finish, prints `Shutting down worker N` for each worker (as the pool is dropped), prints `Server stopped`, and exits with code `0`.

**Recommended library API** (see "API" above; so that your tests look like the hints; you can name things differently if you keep the behaviour):

```rust
// src/http.rs
pub const MAX_HEADER_BYTES: usize = 8 * 1024;
pub struct Request { pub method: String, pub path: String, pub version: String, pub headers: Vec<(String, String)> }
pub enum ParseError { Closed, BadRequestLine, TooLarge, Timeout, Io(std::io::Error) }
pub fn parse_request<R: std::io::BufRead>(reader: R) -> Result<Request, ParseError>;
pub struct Response { pub status: u16, pub reason: &'static str, pub content_type: &'static str,
                      pub extra_headers: Vec<(&'static str, String)>, pub body: Vec<u8> }
impl Response { pub fn write_to<W: std::io::Write>(&self, out: &mut W) -> std::io::Result<()>; /* + constructors */ }

// src/router.rs
pub enum Resolved { File(std::path::PathBuf), Forbidden, NotFound }
pub fn resolve_static(root: &std::path::Path, relative: &str) -> Resolved;
pub fn route(request: &Request, root: &std::path::Path) -> Response;   // no sockets: easy to test

// src/pool.rs
pub struct PoolFull;
pub struct ThreadPool { /* ... */ }
impl ThreadPool {
    pub fn new(size: usize, queue: usize) -> ThreadPool;
    pub fn execute<F: FnOnce() + Send + 'static>(&self, job: F) -> Result<(), PoolFull>;
}                                                    // impl Drop: join all workers

// src/server.rs
pub struct Server { /* ... */ }
impl Server {
    pub fn bind(addr: &str, threads: usize, queue: usize, root: std::path::PathBuf,
                shutdown: std::sync::Arc<std::sync::atomic::AtomicBool>) -> std::io::Result<Server>;
    pub fn local_addr(&self) -> std::io::Result<std::net::SocketAddr>;
    pub fn run(self);                                // returns after `shutdown` is set
}
```

**Project layout:**

```text
code/p06-webserver/
├── Cargo.toml          ← clap and ctrlc
├── README.md           ← what it is, how to run it, how to test it
├── public/
│   ├── hello.html      ← any small HTML page
│   └── note.txt        ← a line of text for /static/note.txt
├── scripts/
│   └── accept.sh       ← the acceptance script from this lesson
├── src/
│   ├── main.rs         ← arguments, Ctrl-C handler, starts the server
│   ├── lib.rs          ← declares the modules
│   ├── http.rs  router.rs  pool.rs  server.rs
└── tests/
    └── server.rs       ← integration test: starts a real server on port 0
```

`Cargo.toml` dependencies (versions from the crates registry, checked 2026-11-17):

```toml
[dependencies]
clap = { version = "4.6", features = ["derive"] }
ctrlc = "3.5"
```

**`ctrlc`** is a small crate: `ctrlc::set_handler(move || { ... })` runs your closure when the user presses Ctrl-C. In the closure you only set an `AtomicBool` flag. The rest of the program reads that flag.

### Step-by-step plan

Commit after every milestone, on the branch `p06-webserver`. Run `cargo fmt` and `cargo clippy --all-targets -- -D warnings` before each commit (`-D warnings` turns warnings into errors).

#### M1 — Hello server

**Goal:** the smallest server from section 3 runs and `curl` gets an answer.

**Guidance:** use the warm-up code. Create `public/hello.html` and `public/note.txt`.

<details>
<summary>Hint</summary>

If `cargo run` says `Address already in use`, an old copy of the server is still running. Stop it with Ctrl-C in its terminal. If you cannot find it, `lsof -i :7878` shows the process id and `kill <pid>` stops it.

</details>

**Done when:** `curl -i http://127.0.0.1:7878/` prints `HTTP/1.1 200 OK` and the body `hello`. Commit: `feat(p06-webserver): M1 hello server`.

#### M2 — Parse the request

**Goal:** a function `parse_request` that turns the bytes from a client into a `Request`, with errors as a `ParseError`. Move the code into a library: `src/lib.rs` (with `pub mod http;`) and `src/http.rs`.

**Guidance:**

- Take any `BufRead` (a `BufReader<&TcpStream>` in the server, a `&[u8]` in tests: a byte slice implements `BufRead`).
- Read the first line with `read_line`. Split it with `split_whitespace`. It must have exactly three parts; the path must start with `/`; the version must start with `HTTP/`. Otherwise return `BadRequestLine`.
- Read header lines until an empty line. Split each at the first `:` (`split_once`) and trim.
- If `read_line` returns `0` bytes at the very beginning, the client closed without sending: `Closed`.
- **Limits already here:** never read more than `MAX_HEADER_BYTES` in total. A single endless line must not fill your memory. Use `reader.take(limit)` to cap one `read_line`. If the cap is reached without a newline, return `TooLarge`.
- Map the I/O error kinds `WouldBlock` and `TimedOut` to `Timeout` (that is how a socket read timeout appears).

<details>
<summary>Hint</summary>

`BufRead::read_line` has no length limit by itself. `Read::take(n)` wraps a reader so that at most `n` bytes can be read from it; `(&mut reader).take(n)` borrows instead of moving. Remember the budget across all lines: `remaining = MAX - used_so_far`.

</details>

**Tests to write (in `http.rs`):** a simple GET with two headers is parsed (method, path, version, header list); `"HELLO\r\n\r\n"`, `"GET /x\r\n\r\n"` and `"GET x HTTP/1.1\r\n\r\n"` give `BadRequestLine`; an empty input gives `Closed`; a header line of 9,000 bytes gives `TooLarge`; 1,000 small headers give `TooLarge`.

**Done when:** `cargo test` shows your parser tests passing, for example:

```text
test http::tests::parses_a_simple_get ... ok
test http::tests::rejects_a_bad_request_line ... ok
test http::tests::empty_input_means_closed ... ok
test http::tests::huge_headers_are_too_large ... ok
test http::tests::many_small_headers_are_too_large_too ... ok
```

Commit: `feat(p06-webserver): M2 parse requests`.

#### M3 — Router, responses and safe static files

**Goal:** `route(&Request, &Path) -> Response` implements the route table, and `Response::write_to` writes a correct HTTP response. The server uses them.

**Guidance:**

- `Response` holds status, reason text, content type, extra headers and the body. `write_to` writes the status line, `Content-Type`, `Content-Length` (the **byte** length of the body), `Connection: close`, extra headers, an empty line, the body.
- `route` is a pure function: a `Request` goes in, a `Response` comes out. No sockets. That is why you can test it without a network.
- Match on `request.path.as_str()`. Check the method per route; if the path is known but the method is not `GET`, return 405 with `Allow: GET`.
- **Static files and path traversal.** `resolve_static(root, relative)` must (1) reject any path component that is not a normal name (`..`, `/`, a drive prefix): use `Path::components()` and `Component::Normal`; (2) canonicalize both `root` and `root.join(relative)` (`Path::canonicalize` resolves `.`/`..` and symbolic links, and fails if the file does not exist); (3) check that the canonical file path `starts_with` the canonical root and `is_file()`.
- Wire it into `server.rs` for now as: for each connection, parse, route, write (still one thread).

<details>
<summary>Hint</summary>

`Path::components()` yields `Component::Normal(name)` for ordinary names and `Component::ParentDir` for `..`. `std::fs::read(path)` gives the file bytes. Choose the content type from `path.extension().and_then(|e| e.to_str())`. Remember: if `canonicalize` fails because the file is missing, that is `NotFound`, not `Forbidden`.

</details>

**Tests to write (in `router.rs`):** `/health` is 200 with body `ok`; unknown path is 404; `POST /health` is 405 with `Allow: GET`; `/` returns the content of `hello.html` from a temporary folder (use `std::env::temp_dir()`); `/static/note.txt` is served; `resolve_static` returns `Forbidden` for `../Cargo.toml`, `a/../../x` and `/etc/passwd`; a missing static file is 404.

**Done when:** start the server (`cargo run`) and run:

```sh
curl -i http://127.0.0.1:7878/health
curl -i http://127.0.0.1:7878/nope
curl -i -X POST http://127.0.0.1:7878/health
curl -i --path-as-is http://127.0.0.1:7878/static/../Cargo.toml
```

```text
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: close

ok
```

```text
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
Content-Length: 14
Connection: close

404 Not Found
```

```text
HTTP/1.1 405 Method Not Allowed
Content-Type: text/plain; charset=utf-8
Content-Length: 23
Connection: close
Allow: GET

405 Method Not Allowed
```

```text
HTTP/1.1 403 Forbidden
Content-Type: text/plain; charset=utf-8
Content-Length: 14
Connection: close

403 Forbidden
```

(`--path-as-is` stops `curl` from cleaning the `..` away before sending.) Commit: `feat(p06-webserver): M3 router and responses`.

> 🔒 **Security:** Path traversal is one of the oldest web attacks: `GET /static/../../etc/passwd`. Never build a file path from user input without checking that the final path is still inside the allowed folder. Test it explicitly, as you do here. (Percent-encoded forms such as `%2e%2e` are a stretch goal; your server does not decode them, so they simply do not match a file.)

#### M4 — The thread pool

**Goal:** `ThreadPool::new(size, queue)` and `execute(job)`. The server hands each accepted connection to the pool. Two simultaneous `/sleep` requests now finish together.

**Guidance:**

- Use `mpsc::sync_channel::<Job>(queue)` where `type Job = Box<dyn FnOnce() + Send + 'static>`. A *sync* channel has a fixed capacity. Its sender has `try_send`, which fails immediately when the queue is full: `TrySendError::Full`.
- Wrap the receiver: `Arc<Mutex<Receiver<Job>>>`. Give each of the `size` workers a clone of the `Arc` and a `JoinHandle`.
- The worker loop: lock, `recv()`, **release the lock**, run the job. If `recv()` returns `Err`, the sender is gone: leave the loop.
- Keep the sender in an `Option<SyncSender<Job>>` so that `Drop` can `take()` it. In `Drop`: drop the sender, then for each worker print `Shutting down worker {id}` and `join` its thread (the worker's `JoinHandle` is also in an `Option`, so you can `take()` it out of `&mut self`).
- `execute` returns `Err(PoolFull)` if `try_send` fails.

<details>
<summary>Hint</summary>

The line that takes a job must end the lock **in the same statement**: store the result of `receiver.lock().unwrap().recv()` in a variable with `let`, then use it. The temporary `MutexGuard` is dropped at the semicolon. See "Common mistakes" at the end: a different shape keeps the lock during the whole job, and your pool would run only one job at a time.

</details>

**Tests to write (in `pool.rs`):** ten jobs on three workers all run, and when the pool is dropped, they are finished (use an `Arc<AtomicUsize>` counter and check it after the `drop`); with one worker and a queue of one, a blocked first job, a second queued job, and a third job give `Err(PoolFull)` (use a channel to keep the first job blocked, then release it).

**Done when:** `cargo test` passes, and with the server started with one worker and then four, two simultaneous requests behave like the numbers in section 3:

```sh
cargo run -- --threads 1
# in another terminal:
curl -s -o /dev/null -w "request 1 took %{time_total}s\n" http://127.0.0.1:7878/sleep &
curl -s -o /dev/null -w "request 2 took %{time_total}s\n" http://127.0.0.1:7878/sleep &
wait
```

With `--threads 1` one request takes about 2 s and the other about 4 s. With `--threads 4` both take about 2 s. Commit: `feat(p06-webserver): M4 thread pool`.

#### M5 — Limits: timeouts, big headers, a full queue

**Goal:** the server protects itself.

**Guidance:**

- **Timeouts.** On each accepted stream call `set_read_timeout(Some(Duration::from_secs(5)))` and `set_write_timeout(...)`. A read that waits too long then fails with `WouldBlock` or `TimedOut`, which your parser maps to `Timeout`; answer `408`.
- **Header size.** `ParseError::TooLarge` becomes `431`. `BadRequestLine` becomes `400`.
- **Full queue.** When `pool.execute` returns `Err(PoolFull)`, the **accept loop itself** writes a `503` response on that stream (the workers are all busy, so it cannot ask them) and moves on.
- **Do not close too early.** If you write the `431` and close the socket while the client is still sending its giant header, the operating system resets the connection and `curl` may print nothing. After writing an error response: call `stream.shutdown(Shutdown::Write)` ("I am done sending"), then read and discard what the client still sends, up to a limit (`io::copy(&mut (&stream).take(1 << 20), &mut io::sink())`).
- **Stream mode.** Next milestone you will make the listener non-blocking. On macOS an accepted socket *inherits* that mode, and reads then fail at once with `WouldBlock`. Call `stream.set_nonblocking(false)` on every accepted stream.

<details>
<summary>Hint</summary>

To make a stream available both to the job and to the accept loop (for the 503 case) use `stream.try_clone()`: it gives a second handle to the same connection. The job gets the clone; if `execute` fails, the original is still in your hand.

</details>

**Tests:** extend the parser tests (you did the size ones in M2). The queue test from M4 covers `PoolFull`.

**Done when:**

```sh
curl -i -H "X-Big: $(head -c 9000 /dev/zero | tr '\0' a)" http://127.0.0.1:7878/
```

```text
HTTP/1.1 431 Request Header Fields Too Large
Content-Type: text/plain; charset=utf-8
Content-Length: 36
Connection: close

431 Request Header Fields Too Large
```

Start the server with `--threads 1 --queue 1`, then start four slow requests quickly, and watch two of them be refused:

```sh
for i in 1 2 3 4; do
  (curl -s -o /dev/null -w "request $i -> %{http_code}\n" http://127.0.0.1:7878/sleep &)
  sleep 0.1
done
```

```text
request 3 -> 503
request 4 -> 503
request 1 -> 200
request 2 -> 200
```

(The order of the lines can differ; two `503` and two `200` is the point.) And an idle connection gets a `408` after 5 seconds:

```sh
(sleep 6 | nc 127.0.0.1 7878) | head -3
```

```text
HTTP/1.1 408 Request Timeout
Content-Type: text/plain; charset=utf-8
Content-Length: 20
```

Commit: `feat(p06-webserver): M5 limits and timeouts`.

#### M6 — Access log, command line and graceful shutdown

**Goal:** the log line, the options of the contract, and Ctrl-C.

**Guidance:**

- **Log:** measure with `Instant::now()` at the start of the handler; print the line after writing the response. Use `println!`; each call writes a whole line, so lines from different threads do not mix.
- **CLI:** use `clap` with `#[derive(Parser)]` (Lesson 05.11) and the defaults of the contract. Check `--threads 0` yourself.
- **Shutdown flag:** create `Arc<AtomicBool>`. `ctrlc::set_handler` sets it to `true`. The accept loop checks it every time around.
- **The accept problem.** `listener.accept()` *blocks*, so the flag is not checked until another client connects. Choose and **write down in your README** one solution and its trade-off: (a) `listener.set_nonblocking(true)`; on `ErrorKind::WouldBlock` sleep 20 ms and check the flag again (simple, but a tiny delay and idle wake-ups); or (b) in the Ctrl-C handler also connect to your own address once, to wake `accept` (no polling, but more moving parts).
- **Finishing work:** `run(self)` takes ownership of the server. When the loop ends, `self` is dropped, which drops the pool, which prints `Shutting down worker N` and joins every worker, so running requests complete first. Then `main` prints `Server stopped` and returns `ExitCode::SUCCESS`.

<details>
<summary>Hint</summary>

A closure given to `ctrlc::set_handler` must be `'static + Send`, so `move` an `Arc::clone` of the flag into it. Use `Ordering::SeqCst` for both the `store` and the `load`; it is the simplest ordering and fast enough here (orderings are explained in Phase 14).

</details>

**Done when:** start the server, then:

```sh
(curl -s -i http://127.0.0.1:7878/sleep > /tmp/sleep.txt &)
sleep 0.5
# now press Ctrl-C in the server's terminal
```

The server terminal shows something like this. Idle workers stop at once. The worker that is busy with your request stops only after the request is finished, so the position of the access-log line between the `Shutting down` lines can differ from run to run:

```text
Listening on http://127.0.0.1:7878
Shutting down worker 0
127.0.0.1 "GET /sleep" 200 6 2005.2ms
Shutting down worker 1
Shutting down worker 2
Shutting down worker 3
Server stopped
```

and `head -1 /tmp/sleep.txt` shows `HTTP/1.1 200 OK`: the request that was running was **not** cut. In the server's terminal `echo $?` prints `0`. The log of other requests looks like this:

```text
127.0.0.1 "GET /health" 200 2 0.1ms
127.0.0.1 "GET /nope" 404 14 0.0ms
127.0.0.1 "POST /health" 405 23 0.1ms
127.0.0.1 "GET /static/../Cargo.toml" 403 14 0.1ms
127.0.0.1 "-" 431 36 0.1ms
```

And the arguments:

```sh
cargo run -- --threads 0; echo "exit: $?"
```

```text
error: --threads must be at least 1
exit: 2
```

Commit: `feat(p06-webserver): M6 access log, cli and graceful shutdown`.

#### M7 — Quality, README, merge and tag

**Goal:** a project you would show to someone.

**Guidance:**

- **Integration test** `tests/server.rs`: bind a `Server` on `127.0.0.1:0` (port `0` means "the operating system picks a free port"; `local_addr()` tells you which), run it in a thread, send raw requests with a `TcpStream`, assert on the answers (200 for `/health`, 404, 405 with `Allow: GET`, 400 for `NONSENSE`), then set the shutdown flag and `join` the server thread.
- **Acceptance script** `scripts/accept.sh` (below). It must print only `PASS` lines.
- **Quality gates:** `cargo test`, `cargo clippy --all-targets -- -D warnings`, `cargo fmt --check` all clean.
- **README.md** in the project: what it is; how to run; the options; the routes; how to run tests and the acceptance script; the design in five lines (accept loop, bounded queue, workers, shutdown); the accept-loop trade-off you chose; known limits (one request per connection, no TLS).
- **Git:** make sure every milestone has a commit; merge with `--no-ff`; tag `p06-webserver-v1.0`.

The acceptance script. Save it as `scripts/accept.sh`, start the server (`cargo run`, default options), and run `bash scripts/accept.sh` from the project folder:

```sh
#!/usr/bin/env bash
# Acceptance checks for p06-webserver.
# Start the server first:  cargo run -- --threads 4
# Run from the project folder:  bash scripts/accept.sh [base-url]
base="${1:-http://127.0.0.1:7878}"
pass=0
fail=0

check() { # check "name" expected actual
    if [ "$2" = "$3" ]; then
        echo "PASS  $1"
        pass=$((pass + 1))
    else
        echo "FAIL  $1 (expected '$2', got '$3')"
        fail=$((fail + 1))
    fi
}

code() { curl -s -o /dev/null -w "%{http_code}" "$@"; }

check "GET /health is 200"          200  "$(code "$base/health")"
check "GET /health body is ok"      ok   "$(curl -s "$base/health")"
check "GET / is 200"                200  "$(code "$base/")"
check "GET /nope is 404"            404  "$(code "$base/nope")"
check "POST /health is 405"         405  "$(code -X POST "$base/health")"
check "405 has Allow: GET"          "Allow: GET" "$(curl -s -i -X POST "$base/health" | tr -d '\r' | grep '^Allow:')"
check "GET /static/note.txt is 200" 200  "$(code "$base/static/note.txt")"
check "missing static file is 404"  404  "$(code "$base/static/missing.txt")"
check "path traversal is refused"   403  "$(code --path-as-is "$base/static/../Cargo.toml")"
check "huge header is 431"          431  "$(code -H "X-Big: $(head -c 9000 /dev/zero | tr '\0' a)" "$base/")"

start=$(date +%s)
curl -s -o /dev/null "$base/sleep" &
curl -s -o /dev/null "$base/sleep" &
wait
elapsed=$(( $(date +%s) - start ))
if [ "$elapsed" -le 3 ]; then
    check "two /sleep requests run together (took ${elapsed}s)" yes yes
else
    check "two /sleep requests run together (took ${elapsed}s)" yes no
fi

echo "passed: $pass, failed: $fail"
[ "$fail" -eq 0 ]
```

**Done when:** with your server running with default options:

```text
PASS  GET /health is 200
PASS  GET /health body is ok
PASS  GET / is 200
PASS  GET /nope is 404
PASS  POST /health is 405
PASS  405 has Allow: GET
PASS  GET /static/note.txt is 200
PASS  missing static file is 404
PASS  path traversal is refused
PASS  huge header is 431
PASS  two /sleep requests run together (took 2s)
passed: 11, failed: 0
```

With `--threads 1` the last check fails, as it should (`FAIL  two /sleep requests run together (took 4s)`): this proves the script really tests concurrency. And:

```text
$ cargo test
test result: ok. 15 passed; ...      ← unit tests (your count can differ)
test result: ok. 1 passed; ...       ← the integration test
$ cargo clippy --all-targets -- -D warnings
    Finished `dev` profile ...
$ cargo fmt --check && echo "fmt ok"
fmt ok
```

Finally, from the repository root: `git switch main && git merge --no-ff p06-webserver` and `git tag -a p06-webserver-v1.0 -m "p06-webserver: all milestones done"`. Commit message for the last step on the branch: `docs(p06-webserver): M7 README and acceptance script`.

### Stretch goals

- **Keep-alive.** Serve several requests on one connection (HTTP/1.1 default). Needs `Connection: keep-alive`, a loop in the handler, and an idle timeout.
- **Percent-decoding** of the path (`%20`, `%2e%2e`): decode first, then check for traversal.
- **A `HEAD` method** for `/` and `/health` (same headers, no body).
- **`/metrics`** with request counts per status code (an `Arc<Mutex<HashMap<u16, u64>>>` or atomics), in plain text.
- **A load test.** Install `oha` (`cargo install oha`) and run `oha -n 2000 -c 50 http://127.0.0.1:7878/health`. Write down requests per second for `--threads 1`, `4` and `16` in your README. Load testing gets a full phase later.
- **Compare with async.** Rebuild the same server with Tokio (Lesson 06.9): `tokio::net::TcpListener`, `tokio::spawn`, and a `Semaphore` for the queue limit. Compare code size and behaviour.
- **CI.** Add a GitHub Actions workflow that runs `cargo fmt --check`, `cargo clippy` and `cargo test` on every push. Phase 09 teaches continuous integration properly.

## Common mistakes

**1. Holding the lock while running the job.** These two worker loops look almost the same. Only one of them is a pool:

```rust
// Correct: the guard is a temporary dropped at the end of the `let` statement.
loop {
    let message = rx.lock().unwrap().recv();
    match message {
        Ok(job) => job(),
        Err(_) => break,
    }
}

// Wrong: the guard lives for the whole loop body, so only one worker runs at a time.
while let Ok(job) = rx.lock().unwrap().recv() {
    job();
}
```

A real experiment, three workers and three jobs of 200 ms each:

```text
let message = ...recv();  -> about 200 ms
while let Ok(job) = ...   -> about 600 ms
```

The temporary created in the condition of `while let` lives until the end of the body. The three jobs ran one after the other.

**2. Reads fail instantly with `WouldBlock` (os error 35).** The listener is non-blocking, and on macOS accepted sockets inherit that. Call `stream.set_nonblocking(false)` after `accept`. A real check on macOS:

```text
read failed: WouldBlock - Resource temporarily unavailable (os error 35)
```

**3. `curl` prints nothing instead of `431`.** You closed the connection while the client was still sending. In one real test, without the "shutdown and drain" step `curl` got the answer in only 1 of 30 tries; with it, in 30 of 30.

**4. Trusting the request.** Reading lines without a limit, joining paths without a check, or letting a client keep a connection open forever each give a way to crash or freeze the server. Every limit in the contract exists for a reason.

**5. A clippy lint you will probably meet.** Writing a parameter as `root: &PathBuf` gives:

```text
error: writing `&PathBuf` instead of `&Path` involves a new object where a slice will do
  --> src/server.rs:93:58
   |
93 | fn handle(mut stream: TcpStream, peer: SocketAddr, root: &PathBuf) {
   |                                                          ^^^^^^^^
```

Fix: take `&Path`. (A `&PathBuf` converts to `&Path` automatically, so callers do not change.)

**6. `unwrap()` on network results.** A client that disconnects makes `write` fail with `BrokenPipe`. Ignore such errors from a single connection (`let _ = ...`) and keep the server alive. Only a failure that makes the *whole server* useless should stop it.

**7. Panics inside a worker.** A panic in a job kills that worker thread, and the pool slowly loses workers. For this project it is enough to keep job code panic-free; a stretch is `std::panic::catch_unwind` around each job.

## Quiz

1. Why is the receiver of the channel wrapped in `Arc<Mutex<...>>`?

<details>
<summary>Answer</summary>

An `mpsc` channel has one receiver, but many workers must take jobs from it. `Arc` lets all workers own the same receiver; `Mutex` makes sure only one worker at a time calls `recv`.

</details>

2. What does a bounded queue (`sync_channel(n)`) give you that an unbounded one does not?

<details>
<summary>Answer</summary>

A limit. When `n` jobs are waiting, `try_send` fails and the server can answer `503` immediately (backpressure), instead of using more and more memory.

</details>

3. How does dropping the `ThreadPool` stop the workers?

<details>
<summary>Answer</summary>

`Drop` drops the sender. When all senders are gone, `recv()` in every worker returns `Err`, the worker leaves its loop, and `Drop` then joins each thread, so running jobs finish first.

</details>

4. Why is `GET /static/../Cargo.toml` dangerous, and how do you stop it?

<details>
<summary>Answer</summary>

`..` moves up out of the served folder, so a client could read any file the server can read. Reject path components that are not normal names, and check that the canonical final path is still inside the canonical root.

</details>

5. Why does the server answer `503` from the accept loop and not from a worker?

<details>
<summary>Answer</summary>

A `503` happens exactly when all workers are busy and the queue is full, so no worker is free to write it. The accept loop is the only thread that is free.

</details>

6. What is a graceful shutdown, and what does your server do on Ctrl-C?

<details>
<summary>Answer</summary>

Stopping after the work in progress is finished. On Ctrl-C the handler sets a flag, the accept loop stops, the pool is dropped (workers finish their current job and exit), `Server stopped` is printed and the exit code is 0.

</details>

7. Why do you make one Git commit per milestone?

<details>
<summary>Answer</summary>

Each commit is a small, working, named step: easy to review, to undo (`git revert`) and to find a bug in (`git bisect`). The history tells the story of the project.

</details>

## Summary

- A web server is a loop: accept a TCP connection, read an HTTP request (text), write a response (text).
- A thread pool uses a fixed number of workers and a channel. The receiver is shared as `Arc<Mutex<Receiver<Job>>>`.
- A bounded queue plus `try_send` gives backpressure: answer `503` when full.
- Dropping the sender is a clean signal to stop; `Drop` joins the workers (graceful shutdown).
- Never trust input: cap sizes, set timeouts, and check that file paths stay inside the root.
- Keep logic that needs no sockets (`parse_request`, `route`) in pure functions, so it is easy to test.
- Close connections carefully: shut down writing and drain the rest, or the client may never see your error.
- Work on a branch, commit every milestone, merge and tag. Run tests, `clippy` and `fmt` before each commit.

## Sources and further reading

- [The Rust Programming Language — Chapter 21: Final Project: Building a Multithreaded Web Server](https://doc.rust-lang.org/book/ch21-00-final-project-a-web-server.html) — the official walk-through that this project extends.
- [std::net::TcpListener](https://doc.rust-lang.org/std/net/struct.TcpListener.html) and [std::net::TcpStream](https://doc.rust-lang.org/std/net/struct.TcpStream.html) — binding, accepting, timeouts, `set_nonblocking`, `shutdown`.
- [std::sync::mpsc::sync_channel](https://doc.rust-lang.org/std/sync/mpsc/fn.sync_channel.html) — bounded channels and `try_send`.
- [MDN (Mozilla Developer Network) — An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview) and [HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) — the language your server speaks.
- [RFC (Request for Comments) 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112) — the formal specification of the message format (long; read the "Message Format" section).
- [ctrlc crate](https://docs.rs/ctrlc/latest/ctrlc/) — the Ctrl-C handler.
- [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) — the commit message convention used in this project.
- [Pro Git — Git Branching](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell) — branches, merges and `--no-ff`.

---

[🏠 Start](../START-HERE.md) · [← 06.10 FFI: calling C code](10-ffi-calling-c.md) · [📋 Phase 06](README.md) · [06.12 Checkpoint: Stage 1 complete →](12-checkpoint-stage-1-complete.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
