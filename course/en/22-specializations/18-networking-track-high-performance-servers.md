<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.17 Databases track: project and next steps](17-databases-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [22.19 Networking track: zero-copy protocol parsing →](19-networking-track-zero-copy-parsing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/18-networking-track-high-performance-servers.md) <!-- nav -->

# Lesson 22.18 — Networking track: high-performance servers

> **Phase 22 — Specializations** · Lesson 18 of 30 · ⏱ about 2.5 hours (2 sittings)

> **Not your track?** This lesson starts the networking track (Lessons 22.18–22.21). If you chose
> another track, skip to [Lesson 22.22 — Compilers track: lexing and parsing](22-compilers-track-lexing-and-parsing.md)
> or go straight to [Lesson 22.30 — Checkpoint: your specialization](30-checkpoint-your-specialization.md).

## What you will learn

- What networking engineers in Rust build, and what "high performance" means for a server.
- How to create a listening socket yourself with the `socket2` crate and set socket options on it.
- What `TCP_NODELAY` does, and why forgetting it can add 40 milliseconds to every request.
- How `SO_REUSEPORT` lets several threads accept on the same port, and how this gives a "thread-per-core" server.
- Which limits (backlog, file descriptors, a connection cap) protect a server under load.

## Before you start

- You finished the earlier phases. This track builds most on [Lesson 15.4 — TCP sockets in Rust](../15-networking/04-tcp-sockets-in-rust.md),
  [Lesson 15.6 — Async networking with Tokio](../15-networking/06-async-networking-with-tokio.md),
  [Lesson 13.13 — Non-blocking I/O and epoll](../13-operating-systems/13-non-blocking-io-and-epoll.md) and
  [Lesson 19.2 — Load testing services](../19-performance-and-profiling/02-load-testing.md).
- The track's code lives in one package. Create it, with every example in `src/bin/`:

```bash
cd ~/projects/rust
cargo new code/p22-net
cd code/p22-net
cargo add tokio@1 --features full
cargo add socket2@0.6
```

- Some examples behave differently on Linux and macOS. Servers in production run on Linux, so run the
  examples in your Linux VM `dev` from Phase 11 (`limactl shell dev`, then the same commands). The
  outputs below come from Linux; the macOS differences are named where they matter.

## New words

| Word | Meaning |
|---|---|
| **data plane** | The part of a network system that moves the actual bytes (proxies, load balancers, routers). The *control plane* decides where bytes should go. |
| **socket option** | A setting on a socket, read and written with the system calls `getsockopt` and `setsockopt`, for example the size of its buffers. |
| **`socket2`** | A crate that gives safe access to almost every socket option, and lets you create a socket step by step. |
| **TCP** (Transmission Control Protocol) | The protocol that gives a reliable, ordered byte stream between two programs (Lesson 15.3). |
| **ACK** (acknowledgement) | A small TCP message that says "I received your bytes up to here". |
| **Nagle's algorithm** | A TCP rule that holds back small writes while earlier data is not yet acknowledged, so that many tiny writes travel as one packet. |
| **delayed ACK** | The receiver waits a short time (on Linux up to about 40 ms) before it sends an ACK, hoping to attach the ACK to a reply. |
| **`TCP_NODELAY`** | The socket option that turns Nagle's algorithm off. |
| **`SO_REUSEPORT`** | A socket option that lets several sockets listen on the same address and port. On Linux the kernel spreads new connections over them. |
| **thread-per-core** | A server design with one thread per CPU (Central Processing Unit) core, each with its own event loop and its own listening socket, sharing as little as possible. |
| **backlog** | The length of the queue of connections that the kernel has completed but the program has not accepted yet. |
| **accept queue** | The kernel queue that the backlog limits. `accept` takes connections from it. |
| **4-tuple** | The four numbers that identify a TCP connection: source IP (Internet Protocol) address, source port, destination IP address, destination port. |

**Other abbreviations in this lesson:** VM (virtual machine; the Linux VM `dev` from Phase 11), IPv4 and IPv6 (versions
4 and 6 of the Internet Protocol), MiB (mebibyte, 1,048,576 bytes) and RFC (Request for Comments, an official internet
standard document).

## 1. What this track is about

Networking engineers who use Rust write the programs in the middle of the network: proxies, load
balancers, API (Application Programming Interface) gateways, DNS (Domain Name System) servers, VPN (Virtual Private Network) software and
packet processors. Rust fits this work because these programs must be fast *and* must never crash on
strange input from the internet. Large companies run Rust proxies in front of a big part of the web;
you will meet one of them, Pingora, in Lesson 22.20.

The track has four lessons:

| Lesson | Skill |
|---|---|
| 22.18 (this one) | The server around the sockets: socket options, how many threads, how many listeners, limits |
| 22.19 | Reading protocols from bytes without copying them |
| 22.20 | Pingora, a framework for production proxies |
| 22.21 | Project: your own HTTP (Hypertext Transfer Protocol) proxy, and where to go next |

"High performance" for a server means three numbers, which you measured in Phase 19:

- **throughput** — how many requests (or bytes) per second it handles;
- **latency** — how long one request takes, especially the slow ones (the 99th percentile, p99);
- **concurrency** — how many connections it can hold open at the same time.

You already have the basic tools from Phase 15: a Tokio accept loop with one task per connection. This
lesson adds the knobs below that loop. Most of them are *socket options*, so we start there.

## 2. Building a listener with `socket2`

`tokio::net::TcpListener::bind` does four system calls for you: `socket`, `setsockopt(SO_REUSEADDR)`,
`bind` and `listen`. It hides the moment *between* them, and some options must be set exactly there,
before `bind`. The `socket2` crate lets you make each call yourself. Save as `src/bin/listener.rs`:

```rust
use socket2::{Domain, Protocol, SockRef, Socket, Type};
use std::net::SocketAddr;
use tokio::net::TcpListener;

fn make_listener(addr: SocketAddr, backlog: i32) -> std::io::Result<TcpListener> {
    let socket = Socket::new(Domain::for_address(addr), Type::STREAM, Some(Protocol::TCP))?;
    socket.set_reuse_address(true)?;
    socket.set_nonblocking(true)?;
    socket.bind(&addr.into())?;
    socket.listen(backlog)?;
    TcpListener::from_std(socket.into())
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = make_listener("127.0.0.1:7300".parse().unwrap(), 1024)?;
    let sock = SockRef::from(&listener);
    println!("listening on {}", listener.local_addr()?);
    println!("SO_REUSEADDR = {}", sock.reuse_address()?);
    println!("SO_RCVBUF    = {} bytes", sock.recv_buffer_size()?);

    let client = tokio::net::TcpStream::connect("127.0.0.1:7300").await?;
    println!("new connection: TCP_NODELAY = {}", client.nodelay()?);
    Ok(())
}
```

**What happens:**

1. `Socket::new` is the `socket` system call. `Domain::for_address(addr)` picks IPv4 or IPv6 to match
   the address, `Type::STREAM` with `Protocol::TCP` asks for a TCP socket.
2. `set_reuse_address(true)` sets `SO_REUSEADDR`: you can restart the server at once, even while old
   connections of the previous run are still closing on that port.
3. `set_nonblocking(true)` makes the socket non-blocking (Lesson 13.13). Tokio requires this, because
   its reactor must never get stuck in a call.
4. `bind` and `listen(backlog)` are the system calls of the same names. `socket.into()` turns the
   `socket2::Socket` into a `std::net::TcpListener`, and `TcpListener::from_std` registers it with Tokio.
5. `SockRef::from(&listener)` borrows any socket as a `socket2` view, so you can read and set options on
   sockets created by Tokio too.
6. Finally the program connects to itself. The kernel completes the connection even before anyone calls
   `accept`, and `nodelay()` shows that `TCP_NODELAY` is off on a new connection.

Run it with `cargo run --bin listener`:

```text
listening on 127.0.0.1:7300
SO_REUSEADDR = true
SO_RCVBUF    = 131072 bytes
new connection: TCP_NODELAY = false
```

> 💡 **Tip:** The receive buffer (`SO_RCVBUF`) is the kernel memory that holds bytes that arrived but
> that your program has not read yet. Bigger buffers help fast transfers over long distances; for many
> small requests the default is fine. Measure before you change it.

## 3. `TCP_NODELAY` and the 40 millisecond mystery

TCP has two old and reasonable rules that together cause a famous problem:

- **Nagle's algorithm (sender):** "if I already sent a small piece that is not acknowledged yet, keep
  the next small piece until the ACK comes, then send everything together." This saves packets when a
  program writes one byte at a time.
- **Delayed ACK (receiver):** "do not send an ACK right away; wait a little, maybe I will send a reply
  and can put the ACK on it." This also saves packets.

Now imagine a server that answers a request with **two small writes**, a header and then a body:

```text
 client                                   server (Nagle on)
   │ ── "GET\n" ───────────────────────────▶ │
   │ ◀──────────────────────── "LEN 5\n" ─── │  first small write: sent at once
   │   (client waits for the body,           │  second write "hello": Nagle holds it,
   │    so it has no reply to put an ACK on; │  because "LEN 5\n" is not acknowledged
   │    delayed ACK timer: about 40 ms)      │
   │ ── ACK (after ~40 ms) ─────────────────▶ │
   │ ◀────────────────────────── "hello" ─── │  now the body goes
```

Each side waits for the other, and only a timer breaks the wait. Let us measure it. Save as
`src/bin/nagle.rs`:

```rust
use std::time::{Duration, Instant};
use tokio::io::{AsyncBufReadExt, AsyncReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};

// The server answers every request line with TWO small writes:
// first a header line, then the body.
async fn serve(listener: TcpListener, nodelay: bool) -> std::io::Result<()> {
    let (stream, _) = listener.accept().await?;
    stream.set_nodelay(nodelay)?;
    let (read_half, mut write_half) = stream.into_split();
    let mut lines = BufReader::new(read_half).lines();
    while let Some(_request) = lines.next_line().await? {
        write_half.write_all(b"LEN 5\n").await?;
        write_half.write_all(b"hello").await?;
    }
    Ok(())
}

async fn measure(nodelay: bool) -> std::io::Result<Duration> {
    let listener = TcpListener::bind("127.0.0.1:0").await?;
    let addr = listener.local_addr()?;
    tokio::spawn(serve(listener, nodelay));

    let mut client = TcpStream::connect(addr).await?;
    client.set_nodelay(true)?;
    let mut reply = [0u8; 11]; // "LEN 5\n" + "hello"
    let rounds = 20;
    let start = Instant::now();
    for _ in 0..rounds {
        client.write_all(b"GET\n").await?;
        client.read_exact(&mut reply).await?;
    }
    Ok(start.elapsed() / rounds)
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    for nodelay in [false, true] {
        let avg = measure(nodelay).await?;
        println!("server TCP_NODELAY = {nodelay:5}: {avg:?} per request");
    }
    Ok(())
}
```

**What happens:**

1. `measure` starts a server on a free port (`:0` means "kernel, choose a port") and connects to it.
2. The server sets `TCP_NODELAY` to the value we pass, then answers each request line with two
   `write_all` calls.
3. The client sends 20 requests one after another and reads exactly 11 bytes each time. The average
   time per request is printed for both settings.

Output on Linux (your numbers will be a little different):

```text
server TCP_NODELAY = false: 39.039229ms per request
server TCP_NODELAY = true : 63.277µs per request
```

The same work is about **600 times slower** with Nagle's algorithm on. On macOS the delayed-ACK timer
is shorter, so you see a few milliseconds instead of 40, but the gap is still large.

There are two fixes, and good servers use both:

1. **Set `TCP_NODELAY`** on every accepted connection of a request-response server. This is what
   almost every HTTP server and proxy does.
2. **Write once.** Put the header and the body into one buffer and call `write_all` once (or use
   `write_vectored`, which gives the kernel several buffers in one system call). Fewer system calls are
   faster anyway.

> ⚡ **Performance:** `TCP_NODELAY` does not make the network faster. It removes an *artificial* wait.
> If your server makes many tiny writes, it now sends many tiny packets, which costs CPU. The real fix is
> to batch writes in your program, where you know where a message ends.

## 4. Many listeners on one port: `SO_REUSEPORT`

Tokio's default runtime is **work-stealing**: a few worker threads share all tasks, and an idle
worker steals tasks from a busy one (Lesson 14.13). This is a great default. Some very busy servers use
another design, **thread-per-core**: one thread per CPU core, each with its *own* single-threaded
runtime and its *own* listening socket. Nothing is shared between the threads, so there are no locks
and no cache lines moving between cores (Lesson 12.10).

For that, several sockets must listen on the same port. Normally the second `bind` fails with
`AddrInUse`. `SO_REUSEPORT`, set **before** `bind`, allows it. Save as `src/bin/reuseport.rs`:

```rust
use socket2::{Domain, Protocol, Socket, Type};
use std::net::SocketAddr;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::time::Duration;

const WORKERS: usize = 4;
const CLIENTS: usize = 1000;

fn reuseport_listener(addr: SocketAddr) -> std::io::Result<std::net::TcpListener> {
    let socket = Socket::new(Domain::for_address(addr), Type::STREAM, Some(Protocol::TCP))?;
    socket.set_reuse_address(true)?;
    socket.set_reuse_port(true)?;
    socket.set_nonblocking(true)?;
    socket.bind(&addr.into())?;
    socket.listen(1024)?;
    Ok(socket.into())
}

// One worker = one thread with its own single-threaded Tokio runtime
// and its own listening socket on the SAME port.
fn worker(addr: SocketAddr, counter: Arc<AtomicUsize>) -> std::io::Result<()> {
    let std_listener = reuseport_listener(addr)?;
    let rt = tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()?;
    rt.block_on(async move {
        let listener = tokio::net::TcpListener::from_std(std_listener)?;
        loop {
            let (stream, _) = listener.accept().await?;
            counter.fetch_add(1, Ordering::Relaxed);
            drop(stream); // a real server would spawn a task here
        }
    })
}

fn main() -> std::io::Result<()> {
    let addr: SocketAddr = "127.0.0.1:7301".parse().unwrap();
    let counters: Vec<Arc<AtomicUsize>> = (0..WORKERS)
        .map(|_| Arc::new(AtomicUsize::new(0)))
        .collect();
    for (id, counter) in counters.iter().enumerate() {
        let counter = Arc::clone(counter);
        std::thread::spawn(move || {
            if let Err(e) = worker(addr, counter) {
                eprintln!("worker {id}: {e}");
            }
        });
    }
    std::thread::sleep(Duration::from_millis(200)); // let all workers bind

    for _ in 0..CLIENTS {
        let _conn = std::net::TcpStream::connect(addr)?;
    }
    std::thread::sleep(Duration::from_millis(200)); // let workers count the last ones

    for (id, counter) in counters.iter().enumerate() {
        println!(
            "worker {id}: {} connections",
            counter.load(Ordering::Relaxed)
        );
    }
    Ok(())
}
```

**What happens:**

1. `main` starts four plain threads. Each one calls `worker`, which creates its own listener with
   `SO_REUSEPORT` on port 7301 and its own `current_thread` runtime (a Tokio runtime that runs all tasks
   on the one thread that calls `block_on`).
2. All four `bind` calls succeed because every socket set `SO_REUSEPORT` before binding.
3. `main` then opens 1,000 client connections, one after another, and closes each at once.
4. Each worker counts the connections it accepted in its own `AtomicUsize` (Lesson 14.5). `Relaxed` is
   enough because we only need the final numbers.

Output on Linux (the exact numbers change every run):

```text
worker 0: 240 connections
worker 1: 243 connections
worker 2: 262 connections
worker 3: 255 connections
```

The Linux kernel computes a hash of each connection's **4-tuple** and uses it to choose one of the
listening sockets. Different client ports give different hashes, so the connections spread evenly.
Each worker has its own accept queue, so the workers never compete for one lock.

On macOS the same program prints `worker 3: 1000 connections` and zero for the others: macOS accepts
the option, but it does not spread connections, and one socket gets them all. This is one
reason the track runs in Linux.

> ⚠️ **Common mistake:** Thread-per-core is not automatically faster. If one connection is much busier
> than others, its worker is overloaded while other workers are idle, because nothing moves work
> between them. Work-stealing handles uneven load better. Measure both designs under *your* load
> (Phase 19) before you choose.

## 5. Limits that protect a server

A server under heavy load fails at its limits, so you should know where they are:

| Limit | What it is | What happens at the limit | How to change it |
|---|---|---|---|
| backlog | the length of the accept queue | new connections wait or are dropped by the kernel | `listen(backlog)`; Linux also caps it with `net.core.somaxconn` |
| file descriptors | every socket is a file descriptor | `accept` fails with "Too many open files" | `ulimit -n` (per process), `LimitNOFILE=` in a systemd unit (Lesson 11.6) |
| buffers | kernel memory per socket | the sender waits (TCP flow control) | `SO_RCVBUF` / `SO_SNDBUF`, capped by `net.core.rmem_max` / `wmem_max` |
| your own cap | the number of connections you choose to serve | you decide: wait, or refuse | a `Semaphore` in the accept loop (exercise 3) |

Your own cap is the most important one, because it is the only one *you* control in code. Without
it, a flood of connections makes every client slow; with it, the server keeps serving a fixed number
well, and the others wait in the kernel's accept queue. That waiting is **backpressure**: the slow
part of the system pushes back on the fast part instead of collapsing.

> 🔒 **Security:** Every open connection costs memory and a file descriptor. An attacker who opens
> thousands of connections and sends nothing can use them up. A connection cap plus an idle timeout
> (Lesson 15.6) is the basic defence against this.

> 🧪 **Testing:** Socket code is tested with real sockets on `127.0.0.1` and port `0`, so tests never
> fight over a fixed port and can run in parallel. Exercises 2 and 3 do exactly this.

## Common mistakes

**Turning a blocking socket into a Tokio socket.** Delete the line `socket.set_nonblocking(true)?;`
from `make_listener` in `src/bin/listener.rs` and run it again. Tokio stops the program:

```text
thread 'main' (75) panicked at src/bin/listener.rs:10:5:
Registering a blocking socket with the tokio runtime is unsupported. If you wish to do anyways, please add `--cfg tokio_allow_from_blocking_fd` to your RUSTFLAGS. See github.com/tokio-rs/tokio/issues/7172 for details.
```

Fix: call `socket.set_nonblocking(true)?` before `TcpListener::from_std` (or `TcpStream::from_std`).

**Two listeners on one port without `SO_REUSEPORT`.** Save this as `src/bin/twice.rs`:

```rust
use std::net::TcpListener;

fn main() -> std::io::Result<()> {
    let _first = TcpListener::bind("127.0.0.1:7304")?;
    let _second = TcpListener::bind("127.0.0.1:7304")?;
    Ok(())
}
```

`cargo run --bin twice` shows that the second `bind` fails:

```text
Error: Os { code: 98, kind: AddrInUse, message: "Address already in use" }
```

(On macOS the code is 48; the kind is the same.) Fix: set `SO_REUSEPORT` on *every* socket, and set it
*before* `bind`. Setting it after `bind` changes nothing.

**Measuring on macOS and deploying on Linux.** As section 4 showed, socket behaviour differs between
operating systems. Measure performance on the operating system you deploy to.

**Changing buffer sizes without measuring.** On Linux, `set_recv_buffer_size(n)` gives you `2 * n`
(the kernel doubles it for its own bookkeeping), capped by `net.core.rmem_max`. Read the value back
and measure the effect; do not assume.

## Try it yourself

1. In your VM `dev`, run `cargo run --bin nagle` three times. Is the 40 ms gap stable?
2. Change the server in `nagle.rs` so that it builds one buffer (`b"LEN 5\nhello"`) and calls
   `write_all` once. Run it again with `TCP_NODELAY = false`. The slow case disappears, because there is
   no second small write for Nagle's algorithm to hold back.
3. Change `WORKERS` in `reuseport.rs` to 8 and `CLIENTS` to 4000. The connections still spread evenly.

## Exercises

### Exercise 1 — What did the kernel give you? (easy)

Write `src/bin/bufsize.rs`: create a TCP socket with `socket2`, print its default receive and send
buffer sizes, ask for a receive buffer of 1 MiB (1,048,576 bytes), and print the value you got. Run it
in your VM.

<details>
<summary>Hint</summary>

The getters are `recv_buffer_size()` and `send_buffer_size()`; the setter is
`set_recv_buffer_size(bytes)`. You do not need Tokio for this.

</details>

<details>
<summary>Answer</summary>

```rust
use socket2::{Domain, Protocol, Socket, Type};

fn main() -> std::io::Result<()> {
    let socket = Socket::new(Domain::IPV4, Type::STREAM, Some(Protocol::TCP))?;
    println!("default SO_RCVBUF = {}", socket.recv_buffer_size()?);
    println!("default SO_SNDBUF = {}", socket.send_buffer_size()?);
    socket.set_recv_buffer_size(1024 * 1024)?;
    println!(
        "asked for 1048576, got SO_RCVBUF = {}",
        socket.recv_buffer_size()?
    );
    Ok(())
}
```

Output on Linux (yours may differ):

```text
default SO_RCVBUF = 131072
default SO_SNDBUF = 16384
asked for 1048576, got SO_RCVBUF = 2097152
```

Why it works: Linux doubles the requested value. If `net.core.rmem_max` is smaller than your request
(on many Ubuntu systems it is 212992), you get `2 * rmem_max` instead. Check yours with
`sysctl net.core.rmem_max`.

</details>

### Exercise 2 — Test the port-sharing rule (medium)

Move the listener code into `src/lib.rs` as
`pub fn listener(addr: SocketAddr, reuse_port: bool) -> std::io::Result<std::net::TcpListener>`.
Write a unit test that proves: two listeners with `reuse_port = true` can bind the same port, and a
second listener with `reuse_port = false` fails with `ErrorKind::AddrInUse`.

<details>
<summary>Hint</summary>

Bind the first listener to `127.0.0.1:0`, then read its real address with `local_addr()` and bind the
second one to that address. `unwrap_err()` gives you the error of a failed `Result`.

</details>

<details>
<summary>Answer</summary>

```rust
use socket2::{Domain, Protocol, Socket, Type};
use std::net::SocketAddr;

pub fn listener(addr: SocketAddr, reuse_port: bool) -> std::io::Result<std::net::TcpListener> {
    let socket = Socket::new(Domain::for_address(addr), Type::STREAM, Some(Protocol::TCP))?;
    socket.set_reuse_address(true)?;
    socket.set_reuse_port(reuse_port)?;
    socket.bind(&addr.into())?;
    socket.listen(1024)?;
    socket.set_nonblocking(true)?;
    Ok(socket.into())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::io::ErrorKind;

    #[test]
    fn two_listeners_share_a_port_only_with_reuse_port() {
        let first = listener("127.0.0.1:0".parse().unwrap(), true).unwrap();
        let addr = first.local_addr().unwrap();
        assert!(listener(addr, true).is_ok());

        let lonely = listener("127.0.0.1:0".parse().unwrap(), false).unwrap();
        let err = listener(lonely.local_addr().unwrap(), false).unwrap_err();
        assert_eq!(err.kind(), ErrorKind::AddrInUse);
    }
}
```

```text
test tests::two_listeners_share_a_port_only_with_reuse_port ... ok
```

Why it works: the test uses port `0`, so the kernel picks a free port and the test never collides
with another program. Both rules are checked on real sockets, which is the only honest way to test
socket options.

</details>

### Exercise 3 — A connection cap with backpressure (hard)

Add `pub async fn serve_limited(listener: tokio::net::TcpListener, max: usize) -> std::io::Result<()>`
to `src/lib.rs`: an echo server that serves at most `max` connections at the same time. Extra
connections are not refused; they wait until a place is free. Write a `#[tokio::test]` with `max = 2`
that shows that a third client gets no echo while two clients are connected, and gets it after one of
them disconnects.

<details>
<summary>Hint</summary>

Use `Arc<tokio::sync::Semaphore>` with `max` permits. Take a permit with `acquire_owned().await`
*before* `accept`, and move the permit into the connection's task: when the task ends, the permit is
dropped and the place is free again. In the test, wrap the third echo in `tokio::time::timeout`.

</details>

<details>
<summary>Answer</summary>

Add this to `src/lib.rs`, below the code of exercise 2:

```rust
use std::sync::Arc;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;
use tokio::sync::Semaphore;

/// Echo server that serves at most `max` connections at the same time.
pub async fn serve_limited(listener: TcpListener, max: usize) -> std::io::Result<()> {
    let limit = Arc::new(Semaphore::new(max));
    loop {
        // Wait for a free place BEFORE accepting the next connection.
        let permit = Arc::clone(&limit).acquire_owned().await.unwrap();
        let (mut stream, _) = listener.accept().await?;
        tokio::spawn(async move {
            let _permit = permit; // released when this task ends
            let mut buf = [0u8; 1024];
            loop {
                match stream.read(&mut buf).await {
                    Ok(0) | Err(_) => return,
                    Ok(n) => {
                        if stream.write_all(&buf[..n]).await.is_err() {
                            return;
                        }
                    }
                }
            }
        });
    }
}

#[cfg(test)]
mod limit_tests {
    use super::*;
    use std::time::Duration;
    use tokio::net::TcpStream;
    use tokio::time::timeout;

    async fn echo_once(stream: &mut TcpStream) -> std::io::Result<()> {
        stream.write_all(b"hi").await?;
        let mut buf = [0u8; 2];
        stream.read_exact(&mut buf).await?;
        Ok(())
    }

    #[tokio::test]
    async fn third_client_waits_until_a_place_is_free() {
        let l = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = l.local_addr().unwrap();
        tokio::spawn(serve_limited(l, 2));

        let mut a = TcpStream::connect(addr).await.unwrap();
        let mut b = TcpStream::connect(addr).await.unwrap();
        echo_once(&mut a).await.unwrap();
        echo_once(&mut b).await.unwrap();

        let mut c = TcpStream::connect(addr).await.unwrap();
        let waited = timeout(Duration::from_millis(200), echo_once(&mut c)).await;
        assert!(waited.is_err(), "the third client must wait");

        drop(a); // free one place
        timeout(Duration::from_secs(2), echo_once(&mut c))
            .await
            .expect("the third client is served after a place is free")
            .unwrap();
    }
}
```

```text
test limit_tests::third_client_waits_until_a_place_is_free ... ok
```

Why it works: the third `connect` succeeds, because the kernel completes the TCP handshake and puts
the connection in the accept queue. But the server does not call `accept` until a permit is free, so
nobody reads the third client's bytes. When `a` is dropped, its task sees end of file, ends, and drops
its permit; the accept loop wakes up and serves `c`. This is backpressure in about ten lines.

</details>

## Quiz

1. Why can some socket options only be set with `socket2` and not after `TcpListener::bind`?

<details>
<summary>Answer</summary>

Options such as `SO_REUSEPORT` must be set *before* `bind`. `TcpListener::bind` creates, binds and
starts listening in one call, so there is no moment to set them. `socket2` lets you make each system
call yourself.

</details>

2. Which two TCP rules together cause the 40 ms delay, and what does each one wait for?

<details>
<summary>Answer</summary>

Nagle's algorithm on the sender holds a small write until the previous data is acknowledged. Delayed
ACK on the receiver holds the acknowledgement, hoping to attach it to a reply. Each waits for the
other, until the delayed-ACK timer (about 40 ms on Linux) fires.

</details>

3. Name two ways to avoid the delay.

<details>
<summary>Answer</summary>

Set `TCP_NODELAY` on the connection, and write the whole response with one `write_all` (or one
vectored write) instead of several small writes.

</details>

4. How does Linux choose which of several `SO_REUSEPORT` listeners gets a new connection?

<details>
<summary>Answer</summary>

It hashes the connection's 4-tuple (source address and port, destination address and port) and uses
the hash to pick one of the listening sockets. Different connections spread roughly evenly.

</details>

5. When is a work-stealing runtime a better choice than thread-per-core?

<details>
<summary>Answer</summary>

When the load is uneven: some connections are much busier than others. Work-stealing moves tasks to
idle threads; thread-per-core keeps each connection on its own thread, so one hot connection can
overload one core while others are idle.

</details>

## Summary

- The networking track is about data-plane programs: proxies, load balancers, gateways. They must be
  fast and must survive any input.
- `socket2` lets you create a socket step by step and set options before `bind`; `SockRef` reads and
  sets options on any existing socket.
- Tokio needs non-blocking sockets: call `set_nonblocking(true)` before `from_std`.
- Nagle's algorithm plus delayed ACK can add about 40 ms per request; set `TCP_NODELAY` and write each
  response at once.
- `SO_REUSEPORT` lets several sockets listen on one port; on Linux the kernel spreads connections by a
  hash of the 4-tuple, which enables thread-per-core servers. macOS does not spread them.
- Know your limits: backlog, file descriptors, buffers, and your own connection cap, which gives
  backpressure instead of collapse.

## Sources and further reading

- [socket2 documentation](https://docs.rs/socket2/0.6/socket2/) — every socket option the crate offers, with the platforms that support each one.
- [tokio::net::TcpListener](https://docs.rs/tokio/1/tokio/net/struct.TcpListener.html) — `from_std`, `bind` and `accept` in Tokio.
- [tokio::runtime::Builder](https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html) — the multi-threaded and current-thread runtimes.
- [socket(7) — Linux manual page](https://man7.org/linux/man-pages/man7/socket.7.html) — `SO_REUSEADDR`, `SO_REUSEPORT`, `SO_RCVBUF` and the other generic socket options.
- [tcp(7) — Linux manual page](https://man7.org/linux/man-pages/man7/tcp.7.html) — `TCP_NODELAY`, `TCP_QUICKACK` and the TCP settings in `/proc/sys/net/ipv4/`.
- [RFC 896 — Congestion Control in IP/TCP Internetworks](https://www.rfc-editor.org/rfc/rfc896) — the original description of Nagle's algorithm (an RFC, Request for Comments, is a document in the official series of internet standards).

---

[🏠 Start](../START-HERE.md) · [← 22.17 Databases track: project and next steps](17-databases-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [22.19 Networking track: zero-copy protocol parsing →](19-networking-track-zero-copy-parsing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
