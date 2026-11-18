<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.15 Project: a TCP load balancer](15-project-tcp-load-balancer.md) · [📋 Phase 15](README.md) · [Phase 16 — Backend services →](../16-backend-services/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/16-checkpoint.md) <!-- nav -->

# Lesson 15.16 — Checkpoint

> **Phase 15 — Networking** · Lesson 16 of 16 · ⏱ about 105 minutes

## What you will learn

This lesson teaches nothing new. It checks that you can use what Phase 15 taught, and it shows you where to go back if something is still weak. You will:

- Answer 18 questions about layers and addresses, TCP and UDP, sockets and async networking, protocols and HTTP, DNS, TLS and QUIC, proxies and load balancers.
- Solve 4 practical tasks: three small programs with tests, and one diagnosis of real error messages.
- Find the lessons you should repeat, with direct links.

## Before you start

- You finished [Lesson 15.15 — Project: a TCP load balancer](15-project-tcp-load-balancer.md), and your `p15-dns` and `p15-lb` projects pass their tests.
- Do the quiz **closed book**: no lessons, no compiler, no search. Write your answer down first, then open the hidden answer.
- For the practical tasks you may use the compiler and the lessons, but try first without hints. Create one package for them:

```bash
cd ~/projects/rust
cargo new --lib code/p15-check
cd code/p15-check
cargo add tokio@1 --features full
cargo add bytes@1
mkdir tests
```

Put `pub mod subnet;`, `pub mod frames;` and `pub mod words;` into `src/lib.rs`; each task fills one of these modules.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A pause at the end of a phase to test yourself before you go on. |
| **closed book** | Answering from memory only, without looking anything up. |
| **diagnosis** | Finding the cause of a failure from its symptoms, here from error messages. |
| **TCP / UDP / DNS / TLS** | Transmission Control Protocol / User Datagram Protocol / Domain Name System / Transport Layer Security: the protocols of this phase. |
| **Other abbreviations of this phase** | IP (Internet Protocol), MAC (Media Access Control, the link-layer address), OS (operating system), SYN/RST (the TCP *synchronize* and *reset* flags), HTTP (HyperText Transfer Protocol), TLD (Top-Level Domain), TTL (Time To Live), CNAME (Canonical NAME), CA (Certificate Authority), SAN (Subject Alternative Name), SNI (Server Name Indication), ALPN (Application-Layer Protocol Negotiation), RFC (Request for Comments), L4/L7 (layer 4/layer 7), CIDR (Classless Inter-Domain Routing, the `address/prefix` notation), NAT (Network Address Translation), QUIC (the UDP-based transport of HTTP/3; not an abbreviation), API (Application Programming Interface). |

## Self-test quiz

Mark one point for each answer that is **correct and complete**. 15 or more points: go on. Fewer: repeat the lessons from the table at the end first.

### Layers, addresses and routing

1. Name the four layers of the internet model and, for each, what it decides.

<details>
<summary>Answer</summary>

Link: the next hop on the local network (MAC addresses). Internet: which computer (IP addresses, routing). Transport: which program (ports; TCP or UDP). Application: what the bytes mean (HTTP, DNS, TLS…). Each layer wraps the one above it in its own header ([Lesson 15.1](01-how-the-internet-works.md)).

</details>

2. What are the network address, the broadcast address and the number of usable host addresses of `192.168.5.130/26`?

<details>
<summary>Answer</summary>

`/26` leaves 6 host bits, so blocks of 64: `.0`, `.64`, `.128`, `.192`. `130` falls into the block starting at `.128`. Network `192.168.5.128`, broadcast `192.168.5.191`, usable 2⁶ − 2 = 62 addresses ([Lesson 15.2](02-ip-addresses-and-routing.md)).

</details>

3. A routing table has `0.0.0.0/0 via 10.0.0.1`, `10.0.0.0/8 dev eth0` and `10.1.2.0/24 via 10.0.0.9`. Which route is used for `10.1.2.3`, and why?

<details>
<summary>Answer</summary>

`10.1.2.0/24 via 10.0.0.9`. All three routes match, and the **longest prefix match** chooses the most specific one, `/24` ([Lesson 15.2](02-ip-addresses-and-routing.md)).

</details>

### TCP, UDP and sockets

4. What does "connection refused" mean at the TCP level, and how does it differ from a connection that times out?

<details>
<summary>Answer</summary>

"Refused": the host answered the SYN with a RST, because nothing listens on that port; you know at once. A timeout: nobody answered at all (the host is down, or a firewall drops packets), so the client waits until its limit ([Lesson 15.3](03-tcp-and-udp.md)).

</details>

5. A server has thousands of sockets in `CLOSE-WAIT`. Whose bug is it, and what is it?

<details>
<summary>Answer</summary>

The server's own. `CLOSE-WAIT` means the other side has closed, but this program has not closed its socket yet: it forgets to drop or close connections when a peer goes away ([Lesson 15.3](03-tcp-and-udp.md)).

</details>

6. A client calls `write_all` twice with 100 bytes each. The server calls `read` once. How many bytes can it get, and what follows for every protocol over TCP?

<details>
<summary>Answer</summary>

Anything from 1 to 200 bytes: TCP is a byte stream and keeps no message borders. Every protocol over TCP needs **framing** (a delimiter, a length prefix or a fixed size), and the reader must handle partial and combined messages ([Lessons 15.4](04-tcp-sockets-in-rust.md) and [15.7](07-designing-a-protocol.md)).

</details>

7. What happens with a UDP datagram of 2000 bytes if the receiver calls `recv_from` with a 512-byte buffer?

<details>
<summary>Answer</summary>

The receiver gets the first 512 bytes, and the rest of that datagram is lost silently. UDP keeps message borders, so one receive returns at most one datagram, and the buffer must be big enough ([Lesson 15.5](05-udp-sockets-in-rust.md)).

</details>

### Async networking, protocols and HTTP

8. Why can a Tokio server keep 10 000 idle connections open with a few threads, while a thread-per-connection server cannot easily do it?

<details>
<summary>Answer</summary>

Every OS thread costs a stack and kernel scheduling. A Tokio task is a small state machine; the runtime uses epoll (Linux) or kqueue (macOS) to wake only the tasks whose sockets are ready, so idle connections cost almost nothing ([Lesson 15.6](06-async-networking-with-tokio.md)).

</details>

9. A length-prefixed decoder has three possible outcomes. What are they, and what must it do with the buffer in each case?

<details>
<summary>Answer</summary>

A complete frame: remove exactly that frame from the buffer and return it. Not complete yet: consume nothing and wait for more bytes. Invalid (for example a length above the limit): return an error at once, without waiting for or allocating that many bytes ([Lesson 15.7](07-designing-a-protocol.md)).

</details>

10. A request has both `Content-Length` and `Transfer-Encoding: chunked`. What must a server do, and why?

<details>
<summary>Answer</summary>

Reject it (answer `400` and close). If a proxy and a server disagree about where the body ends, an attacker can hide a second request inside the first: request smuggling ([Lesson 15.8](08-http-from-scratch.md)).

</details>

11. In a `hyper` server, why is the body wrapped in `Limited::new(body, MAX)` before `collect()`, and which status should a too-big body get?

<details>
<summary>Answer</summary>

Without a limit, `collect()` keeps the whole body in memory, so one client can exhaust the server's memory. `Limited` fails as soon as more than `MAX` bytes arrive; the server answers `413 Payload Too Large` ([Lesson 15.9](09-http-with-hyper-and-reqwest.md)).

</details>

### DNS

12. Describe the path of a lookup for `www.rust-lang.org` from your program to the answer, naming the three kinds of servers.

<details>
<summary>Answer</summary>

The program asks the OS stub resolver (after `/etc/hosts`). The stub asks a **recursive resolver**, which asks a **root server** (referral to the `org` servers), an `org` **TLD** server (referral to the `rust-lang.org` servers) and finally an **authoritative** server, which answers with the `aa` flag: here a CNAME and A records. The recursive resolver caches every answer for its TTL ([Lesson 15.10](10-dns-how-names-become-addresses.md)).

</details>

13. In a DNS message, a name field contains `c0 0c`. What does it mean, and what must a parser check so a hostile packet cannot hang it?

<details>
<summary>Answer</summary>

A compression pointer (top two bits `11`) to offset 12, where the name of the question starts. A parser must follow pointers only backwards and limit the number of jumps, check every position against the message length, and continue reading right after the **first** pointer ([Lesson 15.11](11-project-dns-resolver.md)).

</details>

### TLS and QUIC

14. Your Rust client reports `InvalidCertificate(UnknownIssuer)` against your own test server, and `NotValidForNameContext` against another one. What is wrong in each case?

<details>
<summary>Answer</summary>

`UnknownIssuer`: the chain does not lead to a root the client trusts; the client needs your test CA in its `RootCertStore`. `NotValidForNameContext`: the chain is fine, but the name the client asked for is not in the certificate's SAN list ([Lesson 15.12](12-tls-in-rust-servers.md)).

</details>

15. Why does a lost packet slow down all requests of an HTTP/2 connection but only one stream of an HTTP/3 connection?

<details>
<summary>Answer</summary>

HTTP/2 runs over one TCP byte stream, which delivers strictly in order, so everything behind the lost packet waits (head-of-line blocking). HTTP/3 runs over QUIC, which has independent streams inside UDP datagrams; only the streams whose data was in the lost packet wait ([Lesson 15.13](13-quic-and-http3.md)).

</details>

16. What does QUIC require from TLS that a plain TLS-over-TCP server may leave out, and what happens if a client forgets it?

<details>
<summary>Answer</summary>

ALPN: a QUIC client must say which application protocol it wants (for example `h3`). Without it, the server ends the handshake with the alert *no_application_protocol* (shown by `quinn` as `Code::crypto(78)`) ([Lesson 15.13](13-quic-and-http3.md)).

</details>

### Proxies and load balancers

17. Give two things an L7 proxy can do that an L4 proxy cannot, and one advantage of the L4 proxy.

<details>
<summary>Answer</summary>

L7 can route per request by path or host, and add or change headers (for example `X-Forwarded-For`), cache or retry idempotent requests. L4 works for any protocol over TCP and costs almost nothing per byte, because it never parses the traffic ([Lesson 15.14](14-proxies-and-load-balancers.md)).

</details>

18. List the steps of a graceful shutdown of a load balancer in the right order.

<details>
<summary>Answer</summary>

Receive SIGTERM or SIGINT; stop accepting and close the listener; stop background tasks such as health checks; wait for the open connections (`TaskTracker::wait`) with a drain timeout; exit, cutting what is still open after the deadline ([Lesson 15.15](15-project-tcp-load-balancer.md)).

</details>

## Practical tasks

### Task 1 — A subnet calculator

In `src/subnet.rs`, write `pub fn subnet(cidr: &str) -> Option<(Ipv4Addr, Ipv4Addr, u32)>` that returns the network address, the broadcast address and the number of usable hosts (2ʰ − 2, where h is the number of host bits) for an input such as `"10.20.30.40/22"`. Return `None` for invalid input and for prefixes above 30. Write tests for `/22`, for the borders `/30` and `/0`, and for bad input.

<details>
<summary>Hint</summary>

`Ipv4Addr::to_bits()` gives the address as `u32`, `Ipv4Addr::from_bits` turns it back. The mask is `u32::MAX << (32 - prefix)`, but shifting a `u32` by 32 is not allowed, so treat `/0` separately. Count the hosts in `u64`: for `/0`, 2³² does not fit into `u32`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::net::Ipv4Addr;

/// Network address, broadcast address and number of usable hosts of an IPv4 subnet
/// such as "10.20.30.40/22". Prefixes /31 and /32 have no usable hosts in this rule: None.
pub fn subnet(cidr: &str) -> Option<(Ipv4Addr, Ipv4Addr, u32)> {
    let (addr, prefix) = cidr.split_once('/')?;
    let addr: Ipv4Addr = addr.parse().ok()?;
    let prefix: u32 = prefix.parse().ok()?;
    if prefix > 30 {
        return None;
    }
    let mask = if prefix == 0 {
        0
    } else {
        u32::MAX << (32 - prefix)
    };
    let network = addr.to_bits() & mask;
    let broadcast = network | !mask;
    let usable = (1u64 << (32 - prefix)) - 2;
    Some((
        Ipv4Addr::from_bits(network),
        Ipv4Addr::from_bits(broadcast),
        usable as u32,
    ))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_slash_22() {
        let (net, bc, n) = subnet("10.20.30.40/22").unwrap();
        assert_eq!(net, Ipv4Addr::new(10, 20, 28, 0));
        assert_eq!(bc, Ipv4Addr::new(10, 20, 31, 255));
        assert_eq!(n, 1022);
    }

    #[test]
    fn the_borders() {
        assert_eq!(subnet("192.168.1.77/30").unwrap().2, 2);
        let (net, bc, n) = subnet("8.8.8.8/0").unwrap();
        assert_eq!(
            (net, bc, n),
            (Ipv4Addr::new(0, 0, 0, 0), Ipv4Addr::BROADCAST, u32::MAX - 1)
        );
        assert_eq!(subnet("10.0.0.1/31"), None);
        assert_eq!(subnet("10.0.0.1/33"), None);
        assert_eq!(subnet("10.0.0/24"), None);
    }
}
```

Why it works: `/22` leaves 10 host bits; `30 = 0b0001_1110` with the last two bits cleared is `28`, so the network is `10.20.28.0` and the broadcast sets all 10 host bits: `10.20.31.255`. Every `?` turns bad input into `None` instead of a panic.

</details>

### Task 2 — A frame decoder

In `src/frames.rs`, write `pub fn decode(buf: &mut BytesMut) -> Result<Option<Bytes>, TooLong>` for frames made of a 2-byte big-endian length and that many bytes, with a maximum of 1024 bytes. Test: a frame that arrives in two pieces, two frames in one buffer, and a length of 65 535 that must be rejected at once.

<details>
<summary>Hint</summary>

Check in this order: at least 2 bytes? length above the limit? the whole frame present? Only then `buf.advance(2)` and `buf.split_to(len).freeze()`. Until then, do not change `buf`.

</details>

<details>
<summary>Answer</summary>

```rust
use bytes::{Buf, Bytes, BytesMut};

pub const MAX_FRAME: usize = 1024;

#[derive(Debug, PartialEq)]
pub struct TooLong(pub usize);

/// Takes one frame (2-byte big-endian length, then that many bytes) off the front of `buf`.
/// Ok(None) means "not complete yet"; nothing is consumed then.
pub fn decode(buf: &mut BytesMut) -> Result<Option<Bytes>, TooLong> {
    if buf.len() < 2 {
        return Ok(None);
    }
    let len = u16::from_be_bytes([buf[0], buf[1]]) as usize;
    if len > MAX_FRAME {
        return Err(TooLong(len));
    }
    if buf.len() < 2 + len {
        return Ok(None);
    }
    buf.advance(2);
    Ok(Some(buf.split_to(len).freeze()))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn waits_for_the_whole_frame() {
        let mut buf = BytesMut::from(&[0, 5, b'h', b'e'][..]);
        assert_eq!(decode(&mut buf), Ok(None));
        assert_eq!(buf.len(), 4, "nothing may be consumed");
        buf.extend_from_slice(b"llo");
        assert_eq!(decode(&mut buf), Ok(Some(Bytes::from_static(b"hello"))));
        assert!(buf.is_empty());
    }

    #[test]
    fn two_frames_in_one_read() {
        let mut buf = BytesMut::from(&[0, 1, b'a', 0, 2, b'b', b'c'][..]);
        assert_eq!(decode(&mut buf).unwrap().unwrap(), "a");
        assert_eq!(decode(&mut buf).unwrap().unwrap(), "bc");
        assert_eq!(decode(&mut buf), Ok(None));
    }

    #[test]
    fn rejects_a_huge_length_before_reading_it() {
        let mut buf = BytesMut::from(&[0xff, 0xff][..]);
        assert_eq!(decode(&mut buf), Err(TooLong(65535)));
    }
}
```

Why it works: the limit is checked as soon as the length is known, before waiting for the body, so a hostile length costs nothing. Consuming only complete frames keeps the leftover bytes of the next frame in the buffer for the next call.

</details>

### Task 3 — An async word counter with a test

In `src/words.rs`, write `pub async fn serve(listener: TcpListener) -> std::io::Result<()>`: for every line a client sends, the server answers with the number of words in that line and a newline. Many clients must be served at the same time. In `tests/words.rs`, start it on port 0 and check two clients **concurrently**, with a timeout so a hanging server fails the test instead of blocking it.

<details>
<summary>Hint</summary>

One task per connection; `stream.into_split()` and `BufReader::new(reader).lines()`; `split_whitespace().count()`. In the test, run the two clients with `tokio::join!` inside `tokio::time::timeout`.

</details>

<details>
<summary>Answer</summary>

`src/words.rs`:

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

/// Serves forever: for every line a client sends, answers with the number of words in it.
pub async fn serve(listener: TcpListener) -> std::io::Result<()> {
    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(async move {
            let (reader, mut writer) = stream.into_split();
            let mut lines = BufReader::new(reader).lines();
            while let Ok(Some(line)) = lines.next_line().await {
                let answer = format!("{}\n", line.split_whitespace().count());
                if writer.write_all(answer.as_bytes()).await.is_err() {
                    break;
                }
            }
        });
    }
}
```

`tests/words.rs`:

```rust
use std::time::Duration;

use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::time::timeout;

async fn ask(addr: std::net::SocketAddr, lines: &[&str]) -> Vec<String> {
    let stream = TcpStream::connect(addr).await.unwrap();
    let (reader, mut writer) = stream.into_split();
    let mut replies = BufReader::new(reader).lines();
    let mut out = Vec::new();
    for line in lines {
        writer
            .write_all(format!("{line}\n").as_bytes())
            .await
            .unwrap();
        out.push(replies.next_line().await.unwrap().unwrap());
    }
    out
}

#[tokio::test]
async fn two_clients_at_the_same_time() {
    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    tokio::spawn(p15_check::words::serve(listener));

    let one = ask(addr, &["hello world", "", "a b c"]);
    let two = ask(addr, &["  spaces   everywhere  "]);
    let (one, two) = timeout(Duration::from_secs(2), async { tokio::join!(one, two) })
        .await
        .expect("the server must answer within 2 s");
    assert_eq!(one, ["2", "0", "3"]);
    assert_eq!(two, ["2"]);
}
```

With all three tasks done, `cargo test` prints:

```text
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
test two_clients_at_the_same_time ... ok
```

Why it works: each client has its own task, so the two `ask` calls run at the same time without waiting for each other. The test checks the edge cases (an empty line, extra spaces), and the timeout turns "the server hangs" into a clear failure.

</details>

### Task 4 — Diagnose the errors (no code)

For each message, say at which layer the problem is, what the most likely cause is, and the first command you would run to check it.

```text
a) curl: (6) Could not resolve host: no-such-name.rust-lang.org
b) curl: (7) Failed to connect to 127.0.0.1 port 8099 after 0 ms: Couldn't connect to server
c) curl: (28) Failed to connect to 192.0.2.1 port 80 after 3003 ms: Timeout was reached
d) curl: (60) SSL certificate problem: unable to get local issuer certificate
e) curl: (56) Recv failure: Connection reset by peer   (through your load balancer, every second request)
```

<details>
<summary>Hint</summary>

Think about how far each connection got: the name, the TCP handshake, the TLS handshake, or the data after it. "After 0 ms" and "after 3003 ms" tell you whether somebody answered.

</details>

<details>
<summary>Answer</summary>

- **a)** DNS. The name does not exist (NXDOMAIN) or is misspelled, or no resolver is reachable. Check with `dig +noall +comments <name> | grep status` and look at `/etc/hosts` ([Lesson 15.10](10-dns-how-names-become-addresses.md)).
- **b)** TCP. The connection was refused at once (a RST): nothing listens on port 8099. Check with `ss -tlnp` on Linux (or `lsof -iTCP:8099 -sTCP:LISTEN` on macOS) whether the server runs and on which address ([Lessons 15.3](03-tcp-and-udp.md) and [15.4](04-tcp-sockets-in-rust.md)).
- **c)** TCP or IP. Nobody answered the SYN within 3 seconds: the host is down, the route is wrong, or a firewall drops the packets. Check the route with `ip route get 192.0.2.1` (Lesson 15.2) and capture with `tcpdump` to see whether SYNs go out and nothing comes back.
- **d)** TLS. The handshake worked up to the certificate check: the chain does not lead to a trusted root (a private or self-signed CA). Look at the chain with `openssl s_client -connect host:443 -showcerts`, and trust the right CA with `--cacert` instead of turning verification off ([Lesson 15.12](12-tls-in-rust-servers.md)).
- **e)** The load balancer. It accepted the client but could not reach one backend and dropped the connection: one backend is down and there are no health checks or failover. Check the proxy's log for `cannot reach` ([Lessons 15.14](14-proxies-and-load-balancers.md) and [15.15](15-project-tcp-load-balancer.md)).

All five messages are real; you saw most of them in this phase. Reading an error as "how far did the connection get?" is the fastest way to the cause.

</details>

## If you got something wrong

| Topic | Lesson to repeat |
|---|---|
| Layers, packets, encapsulation, `tcpdump` | [Lesson 15.1 — How the internet works: layers and packets](01-how-the-internet-works.md) |
| IPv4/IPv6, CIDR, subnets, NAT, routing | [Lesson 15.2 — IP addresses, subnets and routing](02-ip-addresses-and-routing.md) |
| Handshake, states, refused vs timed out, TCP vs UDP | [Lesson 15.3 — TCP and UDP](03-tcp-and-udp.md) |
| `TcpListener`, `TcpStream`, partial reads, timeouts | [Lesson 15.4 — TCP sockets in Rust](04-tcp-sockets-in-rust.md) |
| `UdpSocket`, datagram sizes, retries | [Lesson 15.5 — UDP sockets in Rust](05-udp-sockets-in-rust.md) |
| Tokio servers, split streams, `select!`, limits | [Lesson 15.6 — Async networking with Tokio](06-async-networking-with-tokio.md) |
| Framing, decoders, `BytesMut`, codecs | [Lesson 15.7 — Designing a protocol: framing and parsing](07-designing-a-protocol.md) |
| HTTP/1.1 messages, body length, keep-alive, smuggling | [Lesson 15.8 — HTTP from scratch](08-http-from-scratch.md) |
| `hyper` servers, body limits, `reqwest`, end-to-end tests | [Lesson 15.9 — HTTP with hyper and reqwest](09-http-with-hyper-and-reqwest.md) |
| Resolvers, record types, TTL, the wire format | [Lesson 15.10 — DNS: how names become addresses](10-dns-how-names-become-addresses.md) |
| Building and parsing DNS messages, compression | [Lesson 15.11 — Project: a DNS resolver](11-project-dns-resolver.md) |
| Certificates, `rustls` servers, SNI, ALPN, mutual TLS | [Lesson 15.12 — TLS in Rust servers with rustls](12-tls-in-rust-servers.md) |
| QUIC streams, HTTP/3, ALPN in QUIC | [Lesson 15.13 — QUIC and HTTP/3](13-quic-and-http3.md) |
| Forward/reverse proxies, L4/L7, algorithms | [Lesson 15.14 — Proxies and load balancers](14-proxies-and-load-balancers.md) |
| Health checks, limits, graceful shutdown, testing | [Lesson 15.15 — Project: a TCP load balancer](15-project-tcp-load-balancer.md) |
| Tasks, `select!`, semaphores, cancellation | [Lesson 14.13 — Tokio in depth: tasks and channels](../14-concurrency-and-async/13-tokio-tasks-and-channels.md) and [Lesson 14.14 — Cancellation, timeouts and graceful shutdown](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md) |
| TLS basics, chains of trust, `curl -v` | [Lesson 10.11 — TLS and certificates](../10-security-fundamentals/11-tls-and-certificates.md) |

When you repeat a lesson, do not only reread it: redo its exercises, and write the tests first.

## What's next

You finished Phase 15. You can now follow a request from a name to bytes on the wire, write servers and clients for TCP, UDP, HTTP, TLS and QUIC, parse binary protocols safely, and build and test a load balancer. Before you go on:

1. Commit the checkpoint package and push your projects' last state to your own repository.
2. Write a few lines in your learning notes: which layer still feels like a black box, and which stretch goal of `p15-dns` or `p15-lb` you want to try.
3. Take a break.

In **Phase 16 — Backend services** you use these foundations to build real services: HTTP APIs with `axum` (which is built on `hyper`), databases with `sqlx`, authentication, gRPC, caching, messaging, and logs, metrics and traces. Start with [Lesson 16.1 — What a backend service is](../16-backend-services/01-what-a-backend-service-is.md).

## Sources and further reading

- [Computer Networks: A Systems Approach](https://book.systemsapproach.org/) — a free textbook that covers every layer of this phase in more depth.
- [High Performance Browser Networking](https://hpbn.co/) — a free book on latency, TCP, TLS, HTTP/2 and more.
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) — the classic guide to the socket API under `std::net` and Tokio.
- [The Tokio tutorial](https://tokio.rs/tokio/tutorial) — the official guide to async networking with Tokio.
- [RFC Editor](https://www.rfc-editor.org/) — the official home of all RFCs used in this phase (TCP 9293, HTTP 9110/9112, DNS 1035, TLS 8446, QUIC 9000, HTTP/3 9114).

---

[🏠 Start](../START-HERE.md) · [← 15.15 Project: a TCP load balancer](15-project-tcp-load-balancer.md) · [📋 Phase 15](README.md) · [Phase 16 — Backend services →](../16-backend-services/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
