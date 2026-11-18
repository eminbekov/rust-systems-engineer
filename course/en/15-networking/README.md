<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.16 Checkpoint: Stage 3 complete](../14-concurrency-and-async/16-checkpoint-stage-3-complete.md) · [15.1 How the internet works: layers and packets →](01-how-the-internet-works.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/README.md) <!-- nav -->

# Phase 15 — Networking

> **Stage 4 — Build real systems** · ⏱ about 4–5 weeks at ~10 hours per week

## What this phase is about

Almost every program a systems engineer writes talks to other programs over a network: a web service, a
database client, a load balancer, a cache. Frameworks hide most of what happens on the way. When something
goes wrong in production (a request that hangs, thousands of connections in `TIME-WAIT`, a client that is
"refused"), you need to see below the framework. This phase opens the black box.

You start at the bottom: how data travels as packets, what IP (Internet Protocol) addresses, subnets and
routes are, and what TCP (Transmission Control Protocol) and UDP (User Datagram Protocol) promise. You capture real packets with `tcpdump` in a Linux container and decode them byte by byte in
Rust. Then you write servers and clients yourself: with threads and `std::net` first, then with Tokio for
thousands of connections, and you learn how to design your own protocol on top of a byte stream.

With that foundation you build the protocols of the internet from scratch: an HTTP/1.1 (HyperText Transfer
Protocol) server with a strict parser, then real-world HTTP with `hyper` and `reqwest`, a DNS (Domain Name
System) resolver that builds and parses DNS messages by hand, TLS (Transport Layer Security) in your own
servers with `rustls`, a look at QUIC (a newer transport on top of UDP) and HTTP/3, and finally proxies and a
TCP load balancer.

Every experiment runs on your own machine: on `127.0.0.1`, in throw-away Linux containers, or against
public pages that you only read. You never scan or attack hosts that you do not own.

## What you will be able to do

- [ ] Explain the internet's layers and encapsulation, and read a `tcpdump` capture of a TCP conversation.
- [ ] Work with IP addresses, CIDR (Classless Inter-Domain Routing) subnets and routing tables, and explain the longest prefix match.
- [ ] Explain how TCP gives a reliable byte stream (handshake, acknowledgements, flow control, connection states) and when UDP is the better choice.
- [ ] Write TCP and UDP servers and clients with `std::net` and with Tokio, with timeouts, limits and tests on `127.0.0.1:0`.
- [ ] Design a binary protocol with framing, and write parsers that never panic on hostile input.
- [ ] Implement HTTP/1.1 from scratch (keep-alive, `Content-Length`, chunked bodies), and use `hyper` and `reqwest` for real services.
- [ ] Build a DNS resolver, add TLS to servers with `rustls`, and explain what QUIC and HTTP/3 change.
- [ ] Build a TCP load balancer with health checks, connection limits and graceful shutdown.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 15.1 | [How the internet works: layers and packets](01-how-the-internet-works.md) | 135 min |
| 15.2 | [IP addresses, subnets and routing](02-ip-addresses-and-routing.md) | 2.5 h |
| 15.3 | [TCP and UDP](03-tcp-and-udp.md) | 105 min |
| 15.4 | [TCP sockets in Rust](04-tcp-sockets-in-rust.md) | 135 min |
| 15.5 | [UDP sockets in Rust](05-udp-sockets-in-rust.md) | 135 min |
| 15.6 | [Async networking with Tokio](06-async-networking-with-tokio.md) | 135 min |
| 15.7 | [Designing a protocol: framing and parsing](07-designing-a-protocol.md) | 2.5 h |
| 15.8 | [HTTP from scratch](08-http-from-scratch.md) | 3 h |
| 15.9 | [HTTP with hyper and reqwest](09-http-with-hyper-and-reqwest.md) | 135 min |
| 15.10 | [DNS: how names become addresses](10-dns-how-names-become-addresses.md) | 2.5 h |
| 15.11 | [Project: a DNS resolver](11-project-dns-resolver.md) | 8 h |
| 15.12 | [TLS in Rust servers with rustls](12-tls-in-rust-servers.md) | 135 min |
| 15.13 | [QUIC and HTTP/3](13-quic-and-http3.md) | 135 min |
| 15.14 | [Proxies and load balancers](14-proxies-and-load-balancers.md) | 120 min |
| 15.15 | [Project: a TCP load balancer](15-project-tcp-load-balancer.md) | 8 h |
| 15.16 | [Checkpoint](16-checkpoint.md) | 105 min |

## Projects in this phase

- **Small programs in every lesson** in one practice package, `code/p15-net`: each example is a file in `src/bin/` and runs with `cargo run --bin <name>`.
- **[Lesson 15.11 — Project: a DNS resolver](11-project-dns-resolver.md)** in `code/p15-dns`: a command-line resolver that builds DNS queries by hand, parses the answers (including name compression), and is tested with captured packets.
- **[Lesson 15.15 — Project: a TCP load balancer](15-project-tcp-load-balancer.md)** in `code/p15-lb`: a Tokio load balancer with round-robin, health checks, connection limits and graceful shutdown.
- **[Lesson 15.16 — Checkpoint](16-checkpoint.md)**: a quiz and practical tasks before the next phase.

## How to study this phase

- **Look at the wire.** Whenever a lesson sends bytes, run `tcpdump` (in the container) or `curl -v` and compare what you see with what the lesson explains. Seeing a SYN (the TCP segment that opens a connection), a FIN (the one that closes it) or a `Content-Length` makes the rules stick.
- **Use two terminals.** Almost every example is a server and a client. Keep one terminal for each, and stop old servers before starting new ones, or you will meet `Address already in use`.
- **Test with port 0.** Tests that bind to `127.0.0.1:0` get a free port every time and can run in parallel. Write a test for every protocol rule you implement.
- **Treat every byte from the network as hostile.** Check every length before you use it, set limits and timeouts, and make bad input an error, never a panic.
- **Keep your experiments local.** Use `127.0.0.1`, your own containers and public pages that you only read; never scan or load-test hosts you do not own.

---

[🏠 Start](../START-HERE.md) · [← 14.16 Checkpoint: Stage 3 complete](../14-concurrency-and-async/16-checkpoint-stage-3-complete.md) · [15.1 How the internet works: layers and packets →](01-how-the-internet-works.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
