<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.13 QUIC and HTTP/3](13-quic-and-http3.md) · [📋 Phase 15](README.md) · [15.15 Project: a TCP load balancer →](15-project-tcp-load-balancer.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/14-proxies-and-load-balancers.md) <!-- nav -->

# Lesson 15.14 — Proxies and load balancers

> **Phase 15 — Networking** · Lesson 14 of 16 · ⏱ about 120 minutes

## What you will learn

- What forward proxies, reverse proxies and load balancers are, and where you meet them.
- The difference between a layer 4 (TCP) and a layer 7 (HTTP) proxy, and between TLS termination and TLS passthrough.
- How to write a working TCP proxy in Rust with `tokio::io::copy_bidirectional`.
- The common load balancing algorithms, and how to implement round robin safely for many tasks.
- What else a production load balancer must do: health checks, timeouts, retries, limits and passing on the client's address.

## Before you start

- You finished [Lesson 15.13 — QUIC and HTTP/3](13-quic-and-http3.md).
- You know the layers from [Lesson 15.1 — How the internet works: layers and packets](01-how-the-internet-works.md), the Tokio accept loop from [Lesson 15.6](06-async-networking-with-tokio.md), and atomics from [Lesson 14.5 — Atomics](../14-concurrency-and-async/05-atomics.md).
- Create the practice package. It has a small library (the backend chooser) with tests, and two programs:

```bash
cd ~/projects/rust
cargo new --lib code/p15-proxy
cd code/p15-proxy
mkdir -p src/bin
cargo add tokio@1 --features full
```

## New words

| Word | Meaning |
|---|---|
| **proxy** | A program that sits between a client and a server and passes their traffic on, possibly looking at it or changing it. |
| **forward proxy** | A proxy that works for the **clients**: they send all their requests through it (for example a company proxy). |
| **reverse proxy** | A proxy that works for the **servers**: clients think it is the server; it passes requests to the real servers behind it. |
| **load balancer** | A reverse proxy that spreads requests or connections over several equal servers (**backends**). |
| **backend** / **upstream** | One of the real servers behind a proxy. |
| **L4 / L7** (layer 4 / layer 7) | Layer numbers from the OSI (Open Systems Interconnection) model: 4 is the transport (TCP, Transmission Control Protocol), 7 the application (HTTP, HyperText Transfer Protocol). An L4 proxy copies bytes; an L7 proxy understands requests. |
| **TLS termination** | The proxy holds the certificate, decrypts the TLS (Transport Layer Security) traffic and talks to the backends in plain text or with new TLS. |
| **TLS passthrough** | The proxy does not decrypt; it forwards the encrypted bytes, choosing a backend only from the SNI (Server Name Indication) name in the first message. |
| **health check** | A regular test that tells the proxy whether a backend can take traffic. |
| **round robin** | Taking the backends in turn: 1, 2, 3, 1, 2, 3, … |
| **least connections** | Choosing the backend with the fewest open connections. |
| **idempotent** | A request that has the same effect whether it runs once or several times (`GET`, `PUT`), so it can be retried safely. |
| **`X-Forwarded-For`** | An HTTP header in which proxies write the original client's IP (Internet Protocol) address. |
| **QUIC, RFC** | The transport of Lesson 15.13, built on UDP (User Datagram Protocol); its name is not an abbreviation; Request for Comments, the documents that define internet standards. |

## 1. Forward and reverse proxies

A proxy is a middleman. What matters is **for whom** it works:

```text
 forward proxy (works for clients)          reverse proxy (works for servers)

 laptop ─┐                                                    ┌─► app server 1
 laptop ─┼─► company proxy ──► the internet     client ──► proxy ──┼─► app server 2
 laptop ─┘   (filters, logs, caches)          (looks like       └─► app server 3
                                               "the server")
```

- A **forward proxy** is configured on the clients (`HTTPS_PROXY=…`). It decides what they may reach, and the outside world sees the proxy's address instead of theirs.
- A **reverse proxy** stands in front of servers. Clients connect to it as if it were the service. It can terminate TLS, route `/api` to one group of servers and `/static` to another, cache, limit request rates, and hide how many servers exist behind it. Common ones are nginx, HAProxy, Envoy and Traefik; in Kubernetes, a Service and an Ingress controller play these roles.
- A **load balancer** is a reverse proxy whose main job is to spread the load over equal backends, and to stop sending traffic to backends that are down.

You have already used one: in Lesson 15.13 the HTTP/3 server `cloudflare-quic.com` is a reverse proxy, which talks to its own servers behind it.

## 2. Layer 4 and layer 7

A proxy can work at two levels:

| | L4 proxy (TCP) | L7 proxy (HTTP) |
|---|---|---|
| Sees | addresses, ports, a stream of bytes | methods, paths, headers, bodies |
| Chooses the backend | once per **connection** | per **request** (one connection can carry many) |
| Can route by | IP and port (and SNI with TLS passthrough) | host, path, header, cookie |
| Can change traffic | no | yes: add headers, rewrite paths, compress, cache |
| Works for | any protocol over TCP: databases, SSH (Secure Shell), MQTT (Message Queuing Telemetry Transport, a messaging protocol for small devices), HTTP | only the protocol it understands |
| Cost per byte | very small | parsing and buffering |
| Examples | HAProxy `mode tcp`, nginx `stream`, Kubernetes Service | nginx `http`, Envoy, Traefik, an Ingress |

TLS adds one more choice. With **TLS termination**, the proxy holds the certificate and the private key (Lesson 15.12), decrypts, and can then work at L7. With **TLS passthrough**, the proxy forwards the encrypted bytes; it can still read the SNI name in the client's first message, which is not encrypted, and choose a backend by name, but it sees nothing else. (A newer TLS extension, Encrypted Client Hello, hides even that name from everybody but the server operator; with it, routing by SNI works only for a proxy that holds the matching key.) Passthrough keeps the keys only on the backends; termination lets the proxy look into requests.

> 🔒 **Security:** a terminating proxy sees every password and token in clear text, so it is part of your most sensitive infrastructure. If the network between the proxy and the backends is not fully trusted, use TLS there too (often mutual TLS, Lesson 15.12).

## 3. A TCP proxy in 30 lines

An L4 proxy does very little: accept a connection, open a connection to a backend, and copy bytes in both directions until both sides are done. Tokio has exactly that function: `tokio::io::copy_bidirectional`.

First a backend to test with. It answers every HTTP request with its name and the address it sees the client at. Create `src/bin/backend.rs`:

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

/// A tiny HTTP backend: `backend <port> <name>` answers every request with its name.
#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut args = std::env::args().skip(1);
    let port = args.next().unwrap_or_else(|| "9001".to_string());
    let name = args.next().unwrap_or_else(|| "backend".to_string());
    let listener = TcpListener::bind(format!("127.0.0.1:{port}")).await?;
    println!("{name} listening on {}", listener.local_addr()?);

    loop {
        let (mut stream, peer) = listener.accept().await?;
        let name = name.clone();
        tokio::spawn(async move {
            let mut buf = [0u8; 1024];
            let _ = stream.read(&mut buf).await; // the request; we ignore its content
            let body = format!("hello from {name}, you are {peer}\n");
            let reply = format!(
                "HTTP/1.1 200 OK\r\ncontent-length: {}\r\nconnection: close\r\n\r\n{body}",
                body.len()
            );
            let _ = stream.write_all(reply.as_bytes()).await;
        });
    }
}
```

The proxy needs to choose a backend for each connection. Put the chooser into the library, `src/lib.rs`:

```rust
use std::net::SocketAddr;
use std::sync::atomic::{AtomicUsize, Ordering};

/// A list of backends, chosen in turn: round robin.
pub struct RoundRobin {
    backends: Vec<SocketAddr>,
    next: AtomicUsize,
}

impl RoundRobin {
    pub fn new(backends: Vec<SocketAddr>) -> Self {
        assert!(!backends.is_empty(), "need at least one backend");
        RoundRobin {
            backends,
            next: AtomicUsize::new(0),
        }
    }

    /// Returns the next backend. Safe to call from many tasks at once.
    pub fn pick(&self) -> SocketAddr {
        let n = self.next.fetch_add(1, Ordering::Relaxed);
        self.backends[n % self.backends.len()]
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn takes_turns_and_wraps_around() {
        let a: SocketAddr = "127.0.0.1:9001".parse().unwrap();
        let b: SocketAddr = "127.0.0.1:9002".parse().unwrap();
        let rr = RoundRobin::new(vec![a, b]);
        let picks: Vec<_> = (0..5).map(|_| rr.pick()).collect();
        assert_eq!(picks, [a, b, a, b, a]);
    }
}
```

**What happens:**

1. `next` counts how many backends were handed out. `fetch_add(1)` increases it and returns the old value **atomically**: two tasks calling `pick` at the same moment always get different numbers, without a lock (Lesson 14.5).
2. `n % len` turns the ever-growing counter into an index: 0, 1, 0, 1, … `Relaxed` ordering is enough, because no other data depends on this counter (Lesson 14.6).
3. When the counter reaches `usize::MAX`, `fetch_add` wraps around to 0 instead of panicking, so the proxy can run forever. (On a 64-bit machine that takes many lifetimes anyway.)
4. The test pins down the exact order, including the wrap-around.

Now the proxy, `src/bin/l4proxy.rs`:

```rust
use std::net::SocketAddr;
use std::sync::Arc;

use p15_proxy::RoundRobin;
use tokio::io::copy_bidirectional;
use tokio::net::{TcpListener, TcpStream};

/// `l4proxy <backend> [<backend> …]`: listens on 127.0.0.1:8080 and forwards every
/// connection to the next backend, byte for byte.
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let backends: Vec<SocketAddr> = std::env::args()
        .skip(1)
        .map(|a| a.parse())
        .collect::<Result<_, _>>()?;
    let pool = Arc::new(RoundRobin::new(backends));
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("proxy listening on {}", listener.local_addr()?);

    loop {
        let (mut client, peer) = listener.accept().await?;
        let backend = pool.pick();
        tokio::spawn(async move {
            match TcpStream::connect(backend).await {
                Ok(mut server) => match copy_bidirectional(&mut client, &mut server).await {
                    Ok((up, down)) => {
                        println!("{peer} -> {backend}: {up} bytes up, {down} bytes down")
                    }
                    Err(err) => eprintln!("{peer} -> {backend}: {err}"),
                },
                Err(err) => eprintln!("{peer}: cannot reach {backend}: {err}"),
            }
        });
    }
}
```

**What happens:**

1. The backend addresses come from the command line; `collect::<Result<_, _>>()?` stops with an error if one of them does not parse.
2. For every accepted client, `pool.pick()` chooses a backend, and a task connects to it.
3. `copy_bidirectional(&mut client, &mut server)` copies client → server and server → client **at the same time**, until both directions have ended. When one side closes its writing half, the function passes that on (it calls `shutdown` on the other side) and keeps copying the other direction: a half-close works correctly. It returns how many bytes went each way.
4. If the backend cannot be reached, the task logs it and drops the client connection.

Start two backends and the proxy, each in its own terminal (or with `&` at the end of the line), and send four requests through the proxy and one directly to a backend:

```bash
cargo run -q --bin backend -- 9001 blue
cargo run -q --bin backend -- 9002 green
cargo run -q --bin l4proxy -- 127.0.0.1:9001 127.0.0.1:9002
```

```bash
for i in 1 2 3 4; do curl -s http://127.0.0.1:8080/; done
curl -s http://127.0.0.1:9001/
```

```text
hello from blue, you are 127.0.0.1:58455
hello from green, you are 127.0.0.1:58457
hello from blue, you are 127.0.0.1:58459
hello from green, you are 127.0.0.1:58461
hello from blue, you are 127.0.0.1:58462
```

The proxy logs one line per connection:

```text
proxy listening on 127.0.0.1:8080
127.0.0.1:58454 -> 127.0.0.1:9001: 77 bytes up, 99 bytes down
127.0.0.1:58456 -> 127.0.0.1:9002: 77 bytes up, 100 bytes down
127.0.0.1:58458 -> 127.0.0.1:9001: 77 bytes up, 99 bytes down
127.0.0.1:58460 -> 127.0.0.1:9002: 77 bytes up, 100 bytes down
```

**What happens:** the requests alternate between `blue` and `green`. Look at the ports: `curl` connected to the proxy from port 58454, but `blue` saw a client on port 58455, the proxy's own outgoing connection. **The backend no longer sees the real client.** (77 bytes is `curl`'s request; the reply of `green` is one byte longer because its name is.)

> 🧪 **Testing:** the chooser is a plain library type, so `cargo test` checks it without any network. Keep the decision logic (which backend, is it healthy, may this connection start) apart from the I/O (input/output) code, and test it on its own; then test the whole proxy with a few local backends on port 0, as you will do in Lesson 15.15.

## 4. Choosing a backend

Round robin is simple and fair when all requests cost about the same. Other algorithms fit other situations:

| Algorithm | How it chooses | Good when |
|---|---|---|
| **round robin** | in turn | backends are equal, requests are similar |
| **weighted round robin** | in turn, but a backend with weight 3 gets three turns for one | backends have different sizes (Exercise 2) |
| **least connections** | the backend with the fewest open connections | connections live for very different times (Exercise 3) |
| **random** | any backend, chosen at random | many independent proxies; no shared state needed |
| **power of two choices** | pick two at random, take the less busy one | almost as good as least connections, much cheaper at scale |
| **hashing** (by client IP, key or path) | the same input always goes to the same backend | caches and *sticky sessions*; consistent hashing (Lesson 18.8) keeps most keys in place when backends change |

> ⚡ **Performance:** the chooser runs for every connection or request, so keep it cheap and lock-free: an atomic counter, not a `Mutex<Vec<…>>` that every task must wait for. Measure before you choose a "smarter" algorithm; round robin plus good health checks is enough surprisingly often.

## 5. What a real load balancer adds

The proxy above works, but it is not yet safe to put in front of real users. Stop the `green` backend (`Ctrl+C`), restart the proxy with a backend that does not exist (`9003`), and send two requests:

```bash
cargo run -q --bin l4proxy -- 127.0.0.1:9001 127.0.0.1:9003
```

```bash
curl -s http://127.0.0.1:8080/
curl -sS http://127.0.0.1:8080/
```

```text
hello from blue, you are 127.0.0.1:58465
curl: (56) Recv failure: Connection reset by peer
```

The proxy log says `127.0.0.1:58466: cannot reach 127.0.0.1:9003: Connection refused (os error 61)` (on Linux the error number is 111). Every second user gets an error, forever. A production load balancer avoids this with:

- **Health checks.** *Active*: every few seconds, try to connect (L4) or `GET /health` (L7) on each backend, and mark it *down* after a few failures and *up* again after a few successes. *Passive*: count real failures and take a backend out when they pile up. The chooser then skips backends that are down.
- **Timeouts** everywhere: for connecting to a backend (a dead machine often does not answer at all, so `connect` would wait for minutes), for idle connections, and for whole requests at L7.
- **Retries**, carefully. If `connect` to a backend fails, nothing was sent yet, so trying the next backend is always safe. At L7, retry a request only if it is **idempotent**: retrying a `POST /payments` that may already have been processed can charge someone twice.
- **Limits.** A maximum number of connections in total and per backend, so that a traffic spike queues or is refused instead of overwhelming everything (backpressure, Lesson 14.13).
- **Graceful shutdown.** On a restart, stop accepting, let open connections finish (with a deadline), then exit (Lesson 14.14).
- **The client's address.** Because the backend sees only the proxy, an L7 proxy adds `X-Forwarded-For: <client IP>` (or the standard `Forwarded: for=…` header, RFC 7239); an L4 proxy can send a *PROXY protocol* line before the data, if the backend expects it.

You will build health checks, limits and graceful shutdown into a real L4 load balancer in Lesson 15.15 — Project: a TCP load balancer.

## Common mistakes

**1. Trusting `X-Forwarded-For` from anybody.** Any client can send this header itself, with any address in it. Only the proxy you control may set it: the proxy must **remove** incoming `X-Forwarded-For` headers from clients (or append to them knowingly), and the backend must trust the header only when the connection comes from that proxy. Otherwise rate limits and access rules based on client IP are trivially bypassed.

**2. No connect timeout.** A backend machine that is switched off does not refuse connections: it simply does not answer. `TcpStream::connect` then waits for the operating system's limit, often more than a minute, while clients pile up. Wrap it in `tokio::time::timeout` (Lesson 14.14).

**3. Copying one direction at a time.** A proxy that first reads the whole request and only then forwards it, or that copies client → server and only afterwards server → client, deadlocks with protocols in which both sides talk at once, and adds delay to all others. Copy both directions concurrently, as `copy_bidirectional` does.

**4. Retrying everything.** An L7 proxy that retries every failed request turns one slow backend into a storm of duplicate requests, and repeats non-idempotent operations. Retry only safe cases, with a limit, and preferably on a different backend.

## Try it yourself

1. Start the two backends and the proxy as in Section 3.
2. Run `for i in $(seq 1 6); do curl -s http://127.0.0.1:8080/; done`. Count how often you see `blue` and `green`.
3. Stop `green` with `Ctrl+C` and run the loop again. Every second request fails with `Connection reset by peer`, and the proxy log shows `cannot reach 127.0.0.1:9002`. This is exactly the problem that health checks solve in Lesson 15.15.
4. Start `green` again: without restarting the proxy, the next requests work again, because the proxy connects anew for every client.

## Exercises

### Exercise 1 — Not only HTTP (easy)

Show that an L4 proxy forwards any protocol. Use `nc -l 9005` as the only backend, point the proxy at it, send one line of plain text through the proxy with `nc`, and look at what the backend received and what the proxy logged.

<details>
<summary>Hint</summary>

Start `nc -l 9005` in one terminal (it prints what it receives), the proxy with `127.0.0.1:9005` in another, then `printf 'not HTTP at all\n' | nc -w 1 127.0.0.1 8080` in a third (`-w 1` closes after one second).

</details>

<details>
<summary>Answer</summary>

The backend terminal prints:

```text
not HTTP at all
```

The proxy logs (your port differs):

```text
127.0.0.1:58917 -> 127.0.0.1:9005: 16 bytes up, 0 bytes down
```

Why it works: the proxy never looks at the bytes, so it does not care what protocol they belong to. 16 bytes is the text plus the newline; nothing came back, so 0 bytes went down. An L7 HTTP proxy would have rejected this input as an invalid request.

</details>

### Exercise 2 — Weighted round robin (medium)

Write `Weighted` in a new module `src/weighted.rs` (add `pub mod weighted;` to `src/lib.rs`): `Weighted::new(&[(addr, weight)])` and `pick()`, where a backend with weight 3 gets three turns for every turn of a backend with weight 1, and weight 0 means "never". Write tests for both rules.

<details>
<summary>Hint</summary>

The simplest correct solution repeats every address `weight` times in a list (`std::iter::repeat_n(addr, weight)` inside `flat_map`) and then does plain round robin over that list.

</details>

<details>
<summary>Answer</summary>

```rust
use std::net::SocketAddr;
use std::sync::atomic::{AtomicUsize, Ordering};

/// Round robin where a backend with weight 3 gets three turns for every one turn of weight 1.
pub struct Weighted {
    slots: Vec<SocketAddr>,
    next: AtomicUsize,
}

impl Weighted {
    pub fn new(backends: &[(SocketAddr, usize)]) -> Self {
        let slots: Vec<SocketAddr> = backends
            .iter()
            .flat_map(|&(addr, weight)| std::iter::repeat_n(addr, weight))
            .collect();
        assert!(
            !slots.is_empty(),
            "need at least one backend with weight > 0"
        );
        Weighted {
            slots,
            next: AtomicUsize::new(0),
        }
    }

    pub fn pick(&self) -> SocketAddr {
        let n = self.next.fetch_add(1, Ordering::Relaxed);
        self.slots[n % self.slots.len()]
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_heavier_backend_gets_more_turns() {
        let a: SocketAddr = "127.0.0.1:9001".parse().unwrap();
        let b: SocketAddr = "127.0.0.1:9002".parse().unwrap();
        let w = Weighted::new(&[(a, 3), (b, 1)]);
        let picks: Vec<_> = (0..8).map(|_| w.pick()).collect();
        assert_eq!(picks, [a, a, a, b, a, a, a, b]);
    }

    #[test]
    fn weight_zero_means_never() {
        let a: SocketAddr = "127.0.0.1:9001".parse().unwrap();
        let b: SocketAddr = "127.0.0.1:9002".parse().unwrap();
        let w = Weighted::new(&[(a, 0), (b, 2)]);
        assert!((0..10).all(|_| w.pick() == b));
    }
}
```

Why it works: after expansion, `[(a, 3), (b, 1)]` becomes the list `a, a, a, b`, and round robin over it gives exactly the wanted ratio. The `assert!` protects against a list of only zero weights, which would make `% 0` panic. (Real load balancers spread the heavy backend's turns out, `a, b, a, a` instead of `a, a, a, b`, to avoid bursts; that is a nice extension.)

</details>

### Exercise 3 — Least connections with a lease (hard)

Write `LeastConnections` in `src/least.rs`. `LeastConnections::pick(&pool)` chooses the backend with the fewest open connections and returns a `Lease`; while the lease exists, the connection counts as open, and dropping the lease ends it. Test: with two idle backends the first pick gets the first one, the second pick the other one, and after the first lease is dropped the first backend is chosen again.

<details>
<summary>Hint</summary>

Store `Vec<(SocketAddr, AtomicUsize)>` in an `Arc`. The lease keeps an `Arc` to the pool and the index, and `impl Drop for Lease` does `fetch_sub(1)`. `iter().enumerate().min_by_key(...)` finds the least busy entry (on a tie it returns the first one).

</details>

<details>
<summary>Answer</summary>

```rust
use std::net::SocketAddr;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

/// Picks the backend with the fewest open connections.
pub struct LeastConnections {
    backends: Vec<(SocketAddr, AtomicUsize)>,
}

/// Proof that one connection to `addr` is open. Dropping it closes the count.
pub struct Lease {
    pool: Arc<LeastConnections>,
    index: usize,
    pub addr: SocketAddr,
}

impl LeastConnections {
    pub fn new(addrs: Vec<SocketAddr>) -> Arc<Self> {
        assert!(!addrs.is_empty(), "need at least one backend");
        let backends = addrs
            .into_iter()
            .map(|a| (a, AtomicUsize::new(0)))
            .collect();
        Arc::new(LeastConnections { backends })
    }

    /// Chooses the least busy backend and counts the new connection.
    pub fn pick(pool: &Arc<Self>) -> Lease {
        let (index, (addr, open)) = pool
            .backends
            .iter()
            .enumerate()
            .min_by_key(|(_, (_, open))| open.load(Ordering::Relaxed))
            .expect("not empty");
        open.fetch_add(1, Ordering::Relaxed);
        Lease {
            pool: Arc::clone(pool),
            index,
            addr: *addr,
        }
    }

    pub fn open_connections(&self, addr: SocketAddr) -> usize {
        self.backends
            .iter()
            .find(|(a, _)| *a == addr)
            .map_or(0, |(_, open)| open.load(Ordering::Relaxed))
    }
}

impl Drop for Lease {
    fn drop(&mut self) {
        self.pool.backends[self.index]
            .1
            .fetch_sub(1, Ordering::Relaxed);
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_busy_backend_is_avoided_until_its_connection_ends() {
        let a: SocketAddr = "127.0.0.1:9001".parse().unwrap();
        let b: SocketAddr = "127.0.0.1:9002".parse().unwrap();
        let pool = LeastConnections::new(vec![a, b]);

        let first = LeastConnections::pick(&pool); // both idle: the first one wins
        assert_eq!(first.addr, a);
        let second = LeastConnections::pick(&pool); // a is busy, b is idle
        assert_eq!(second.addr, b);
        assert_eq!(pool.open_connections(a), 1);

        drop(first); // the connection to a ends
        assert_eq!(pool.open_connections(a), 0);
        assert_eq!(LeastConnections::pick(&pool).addr, a);
        drop(second);
    }
}
```

With `pub mod least;` and `pub mod weighted;` in `src/lib.rs`, `cargo test` prints `test result: ok. 4 passed` for the library.

Why it works: the lease is an RAII (Resource Acquisition Is Initialization) guard, like a `MutexGuard`: the count goes up when it is created and down in `Drop`, so it cannot be forgotten, even when the connection task ends with an error or a panic. In the proxy you would keep the lease alive inside the connection task until `copy_bidirectional` returns. Reading the minimum and then adding is not one atomic step, so under heavy load two tasks may choose the same backend at the same instant; for balancing that small inaccuracy is acceptable.

</details>

## Quiz

1. What is the difference between a forward proxy and a reverse proxy?

<details>
<summary>Answer</summary>

A forward proxy works for clients: they are configured to send their traffic through it, and servers see the proxy instead of them. A reverse proxy works for servers: clients connect to it as if it were the service, and it passes requests on to the real servers behind it.

</details>

2. Why can an L4 proxy not route `/api` and `/static` to different backends, and what can it use instead?

<details>
<summary>Answer</summary>

An L4 proxy only sees a stream of bytes and chooses the backend once per connection, before any request is read; it does not parse HTTP, so it knows nothing about paths. It can route by address and port, and with TLS passthrough by the SNI name in the first TLS message.

</details>

3. In the demo, `curl` used port 58454, but the backend reported a client on port 58455. Why, and why does it matter?

<details>
<summary>Answer</summary>

The backend's TCP connection comes from the proxy, which opened a new connection from its own port 58455. The backend never sees the real client, so logging, rate limiting or access rules by client address need extra help: `X-Forwarded-For`/`Forwarded` headers at L7 or the PROXY protocol at L4.

</details>

4. Why is it always safe to try another backend when `connect` fails, but not always safe to retry a failed HTTP request?

<details>
<summary>Answer</summary>

If `connect` fails, no byte of the request has reached any backend, so nothing happened yet. A failed HTTP request may already have been processed (the answer was lost, not the request); repeating a non-idempotent request such as a payment could do it twice.

</details>

5. Why does `RoundRobin::pick` use `AtomicUsize::fetch_add` instead of a `Mutex<usize>`?

<details>
<summary>Answer</summary>

`fetch_add` increments and returns the old value in one atomic CPU (Central Processing Unit) operation, so every task gets a different number without waiting for a lock. It is correct and cheaper than a mutex for something that runs on every connection.

</details>

## Summary

- A forward proxy works for clients; a reverse proxy stands in front of servers; a load balancer is a reverse proxy that spreads traffic over equal backends.
- L4 proxies copy bytes and choose once per connection; L7 proxies understand HTTP and can route, change and retry per request.
- TLS termination puts the keys on the proxy and lets it see requests; passthrough forwards encrypted bytes, routing at most by SNI.
- `tokio::io::copy_bidirectional` is the heart of an L4 proxy: both directions at once, with correct half-close.
- Round robin with an atomic counter is cheap and safe; weighted, least connections, power of two choices and hashing fit other situations.
- Production load balancers add health checks, timeouts, careful retries, limits, graceful shutdown, and a way to pass on the client's address.

## Sources and further reading

- [tokio::io::copy_bidirectional](https://docs.rs/tokio/latest/tokio/io/fn.copy_bidirectional.html) — the function behind the proxy, and how it handles shutdown.
- [Envoy — Supported load balancers](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers) — round robin, least request, ring hash and others, explained by a widely used proxy.
- [Kubernetes — Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/) — how a Kubernetes Service works as an L4 load balancer.
- [RFC 7239 — Forwarded HTTP Extension](https://www.rfc-editor.org/rfc/rfc7239) — the standard header for passing on the client's address.
- [The PROXY protocol](https://www.haproxy.org/download/3.0/doc/proxy-protocol.txt) — how L4 proxies pass on the client's address, specified by the HAProxy authors.
- [RFC 9110 — HTTP Semantics, section 9.2.2](https://www.rfc-editor.org/rfc/rfc9110) — the definition of idempotent methods.

---

[🏠 Start](../START-HERE.md) · [← 15.13 QUIC and HTTP/3](13-quic-and-http3.md) · [📋 Phase 15](README.md) · [15.15 Project: a TCP load balancer →](15-project-tcp-load-balancer.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
