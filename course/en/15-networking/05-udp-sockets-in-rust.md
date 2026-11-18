<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.4 TCP sockets in Rust](04-tcp-sockets-in-rust.md) · [📋 Phase 15](README.md) · [15.6 Async networking with Tokio →](06-async-networking-with-tokio.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/05-udp-sockets-in-rust.md) <!-- nav -->

# Lesson 15.5 — UDP sockets in Rust

> **Phase 15 — Networking** · Lesson 5 of 16 · ⏱ about 135 minutes

## What you will learn

- How `UdpSocket` works: `bind`, `send_to`, `recv_from`, and why one socket can talk to many peers.
- What happens to message borders, to a too-small buffer and to a too-large datagram.
- What `connect` means for UDP and how it makes errors visible.
- How to build reliability yourself: timeouts, retries and request IDs.
- How to test UDP code, including lost datagrams.

## Before you start

- You finished [Lesson 15.4 — TCP sockets in Rust](04-tcp-sockets-in-rust.md).
- You remember from [Lesson 15.3](03-tcp-and-udp.md) that UDP sends single datagrams with no
  guarantees of delivery, order or uniqueness.
- Examples go into `code/p15-net/src/bin/`. For trying the servers by hand you use `nc -u` (UDP mode
  of netcat), which is part of macOS.

## New words

| Word | Meaning |
|---|---|
| **peer** | The program on the other side of a conversation. With UDP every datagram can come from a different peer. |
| **truncation** | Cutting off the end of a datagram because the receive buffer was too small. The rest is lost. |
| **fragmentation** | Splitting one IP packet that is bigger than the MTU into several smaller packets. If one fragment is lost, the whole datagram is lost. |
| **request ID** | A number that a client puts into each request and the server copies into the answer, so that the client can match answers to requests. |
| **retry** | Sending a request again after a timeout because no answer came. |
| **idempotent** | An operation that has the same effect whether it runs once or several times. Only idempotent requests are safe to retry blindly. |
| **amplification attack** | An attack in which a small request with a forged source address makes a server send a large answer to a victim. |
| **IP spoofing** | Sending packets with a false source address. Easy with UDP, because there is no handshake. |
| **UDP / TCP / IP / MTU** (User Datagram Protocol / Transmission Control Protocol / Internet Protocol / Maximum Transmission Unit) | From Lessons 15.1 and 15.3: the datagram protocol of this lesson, the stream protocol, the protocol under both, and the biggest packet one link can carry (1500 bytes on Ethernet). |
| **DNS, ICMP, QUIC, FIN** | Domain Name System (names to addresses, a classic UDP protocol); Internet Control Message Protocol (error messages such as "port unreachable"); QUIC (a newer transport built on UDP, Lesson 15.13); the TCP "finish" flag (Lesson 15.3). |

## 1. `UdpSocket`: no listen, no accept

A UDP server has no `listen` and no `accept`, because there are no connections. It binds a socket and
receives datagrams; each one comes with the address of its sender. Save as `src/bin/udp_echo.rs`:

```rust
use std::net::UdpSocket;

fn main() -> std::io::Result<()> {
    let socket = UdpSocket::bind("127.0.0.1:7100")?;
    println!("UDP echo server on {}", socket.local_addr()?);
    let mut buf = [0u8; 1500];
    loop {
        let (n, peer) = socket.recv_from(&mut buf)?;
        println!(
            "{peer}: {n} bytes: {:?}",
            String::from_utf8_lossy(&buf[..n])
        );
        socket.send_to(&buf[..n], peer)?;
    }
}
```

**What happens:**

1. `UdpSocket::bind` makes the system calls `socket` and `bind`. From now on the kernel delivers every
   datagram for port 7100 to this socket.
2. `recv_from` waits for the next datagram, copies it into `buf`, and returns its length and the
   sender's address (`peer`).
3. `send_to` sends one datagram to an address. We answer the peer we just heard from.
4. One thread and one socket serve **all** clients: there is no per-client state in the kernel.

Run it (`cargo run --bin udp_echo`) and, in a second terminal, send two datagrams:

```bash
printf "hello\n" | nc -u -w 1 127.0.0.1 7100
printf "second\n" | nc -u -w 1 127.0.0.1 7100
```

Each `nc` prints its line back (`hello`, then `second`), and the server shows two different peers,
because each `nc` got its own ephemeral port:

```text
UDP echo server on 127.0.0.1:7100
127.0.0.1:60033: 6 bytes: "hello\n"
127.0.0.1:50304: 7 bytes: "second\n"
```

`-u` selects UDP, and `-w 1` tells `nc` to give up after one second of silence (with UDP it cannot know
that the conversation is over, since there is no FIN).

| | TCP (`TcpListener` + `TcpStream`) | UDP (`UdpSocket`) |
|---|---|---|
| Sockets on the server | one listening + one per client | one for everybody |
| Receive | `read`: some bytes of the stream | `recv_from`: one whole datagram + sender |
| Send | `write_all`: bytes into the stream | `send_to`: one datagram to an address |
| End of conversation | EOF (`Ok(0)`) | does not exist; decide yourself (timeout) |

## 2. Borders, small buffers and big datagrams

Datagrams keep their borders, but the buffer you give to `recv_from` decides how much of each one you
get. This self-contained program sends four datagrams and receives them into an 8-byte buffer. Save as
`src/bin/datagrams.rs`:

```rust
use std::net::UdpSocket;

fn main() -> std::io::Result<()> {
    let server = UdpSocket::bind("127.0.0.1:0")?;
    let client = UdpSocket::bind("127.0.0.1:0")?;
    let server_addr = server.local_addr()?;

    for msg in ["one", "two", "three", "a much longer fourth message"] {
        client.send_to(msg.as_bytes(), server_addr)?;
    }

    let mut buf = [0u8; 8]; // deliberately too small for the last message
    for _ in 0..4 {
        let (n, from) = server.recv_from(&mut buf)?;
        println!(
            "{n} bytes from {from}: {:?}",
            String::from_utf8_lossy(&buf[..n])
        );
    }

    // A datagram larger than the IPv4 limit cannot be sent at all.
    let huge = vec![0u8; 70_000];
    if let Err(e) = client.send_to(&huge, server_addr) {
        println!("70000 bytes: {e} (kind: {:?})", e.kind());
    }
    Ok(())
}
```

`cargo run --bin datagrams`:

```text
3 bytes from 127.0.0.1:51287: "one"
3 bytes from 127.0.0.1:51287: "two"
5 bytes from 127.0.0.1:51287: "three"
8 bytes from 127.0.0.1:51287: "a much l"
70000 bytes: Message too long (os error 40) (kind: Uncategorized)
```

**What happens:**

1. Four sends produce exactly four receives. Compare this with `chunks.rs` in Lesson 15.4, where two
   TCP writes came out as one read.
2. The fourth datagram had 28 bytes, but the buffer had 8. The kernel copied 8 bytes and **threw away
   the rest**, without any error. This is **truncation**. Always use a buffer as large as the largest
   datagram your protocol allows.
3. A UDP datagram over IPv4 can carry at most 65,507 bytes (65,535 minus the IP and UDP headers). Bigger
   sends fail immediately with "Message too long" (on Linux it is `os error 90`).

Even 65,507 bytes is too much in practice. A datagram bigger than the MTU (1500 bytes on Ethernet) is
split by IP into **fragments**, and if *one* fragment is lost, the whole datagram is lost. Many networks
also drop fragments on purpose. Real UDP protocols therefore keep datagrams small: classic DNS answers
are at most 512 bytes, and QUIC uses about 1,200 bytes.

## 3. `connect` for UDP

`UdpSocket::connect` sends nothing. It only remembers a **default peer** in the kernel. After that:

- `send` and `recv` work without an address;
- datagrams from any *other* address are dropped by the kernel;
- ICMP errors for this peer, such as "port unreachable" from Lesson 15.3, are reported to your
  program.

The last point is useful. Save as `src/bin/udp_connect.rs`:

```rust
use std::net::UdpSocket;
use std::time::Duration;

fn main() -> std::io::Result<()> {
    // Find a free port, then close it again, so that nobody listens there.
    let closed_port = UdpSocket::bind("127.0.0.1:0")?.local_addr()?;

    let socket = UdpSocket::bind("127.0.0.1:0")?;
    socket.set_read_timeout(Some(Duration::from_millis(500)))?;
    socket.connect(closed_port)?; // only sets the default peer: no packet is sent
    socket.send(b"anybody there?")?;

    let mut buf = [0u8; 64];
    match socket.recv(&mut buf) {
        Ok(n) => println!("got {n} bytes"),
        Err(e) => println!("recv: {e} (kind: {:?})", e.kind()),
    }
    Ok(())
}
```

```text
recv: Connection refused (os error 61) (kind: ConnectionRefused)
```

**What happens:**

1. The temporary socket in the first line is dropped at the end of the statement, so its port is free
   again and nobody listens on it.
2. `send` succeeds: the datagram leaves. The receiving kernel answers with ICMP "port unreachable".
3. Because the socket is connected, the kernel passes that ICMP message to us, and `recv` returns
   `ConnectionRefused` instead of waiting until the timeout. An unconnected socket would simply wait
   500 ms and report `WouldBlock`.

Do not rely on this message: across a real network, firewalls often drop ICMP. It is a quick hint, not
a guarantee.

## 4. Reliability is your job

With UDP, a request or its answer can be lost, and nobody tells you. The standard recipe is:

1. Set a **read timeout**.
2. If no answer comes in time, **retry**, a limited number of times.
3. Put a **request ID** into each request. The server copies it into the answer. Ignore answers whose ID
   does not match: they may be late answers to an earlier attempt, or from somebody else.

This program simulates a bad network with a server that "loses" every second datagram. Save as
`src/bin/retry.rs`:

```rust
use std::io::{Error, ErrorKind};
use std::net::{SocketAddr, UdpSocket};
use std::thread;
use std::time::{Duration, Instant};

/// A test server that "loses" every second datagram and answers the others.
fn lossy_server(socket: UdpSocket) {
    let mut buf = [0u8; 512];
    let mut count = 0;
    while let Ok((n, peer)) = socket.recv_from(&mut buf) {
        count += 1;
        if count % 2 == 1 {
            continue; // pretend this datagram was lost on the way
        }
        let mut answer = buf[..2].to_vec(); // keep the request id
        answer.extend(buf[2..n].to_ascii_uppercase());
        let _ = socket.send_to(&answer, peer);
    }
}

/// Sends `body` with a 2-byte id and waits for the answer with the same id, up to 3 times.
fn request(
    socket: &UdpSocket,
    server: SocketAddr,
    id: u16,
    body: &[u8],
) -> std::io::Result<Vec<u8>> {
    let mut packet = id.to_be_bytes().to_vec();
    packet.extend_from_slice(body);
    let mut buf = [0u8; 512];
    for attempt in 1..=3 {
        socket.send_to(&packet, server)?;
        match socket.recv_from(&mut buf) {
            Ok((n, from)) if from == server && n >= 2 && buf[..2] == id.to_be_bytes() => {
                return Ok(buf[2..n].to_vec());
            }
            Ok(_) => println!("  attempt {attempt}: unexpected datagram, ignored"),
            Err(e) if matches!(e.kind(), ErrorKind::WouldBlock | ErrorKind::TimedOut) => {
                println!("  attempt {attempt}: no answer within the timeout");
            }
            Err(e) => return Err(e),
        }
    }
    Err(Error::new(
        ErrorKind::TimedOut,
        "no answer after 3 attempts",
    ))
}

fn main() -> std::io::Result<()> {
    let server = UdpSocket::bind("127.0.0.1:0")?;
    let server_addr = server.local_addr()?;
    thread::spawn(move || lossy_server(server));

    let socket = UdpSocket::bind("127.0.0.1:0")?;
    socket.set_read_timeout(Some(Duration::from_millis(200)))?;
    for (id, word) in [(1, "hello"), (2, "world")] {
        let start = Instant::now();
        let answer = request(&socket, server_addr, id, word.as_bytes())?;
        println!(
            "id {id}: {:?} after {:?}",
            String::from_utf8_lossy(&answer),
            start.elapsed()
        );
    }
    Ok(())
}
```

`cargo run --bin retry`:

```text
  attempt 1: no answer within the timeout
id 1: "HELLO" after 201.307583ms
  attempt 1: no answer within the timeout
id 2: "WORLD" after 201.188333ms
```

**What happens:**

1. Each request is 2 bytes of ID (big-endian, Lesson 15.1) followed by the body. The server answers with
   the same ID and the body in upper case.
2. The first attempt of each request is "lost", so `recv_from` hits the 200 ms timeout. The client sends
   the same packet again, and the second attempt is answered.
3. The match guard checks three things before it accepts an answer: it comes from the server, it is long
   enough, and its ID matches. Anything else is ignored.
4. After three failed attempts `request` returns its own `TimedOut` error, so the caller does not wait
   forever.

Every retry costs one timeout of waiting. Real clients therefore start with a short timeout and make it
longer with each attempt (*exponential backoff*), and they limit the total time. Lesson 18.3 (retries, backoff and
idempotency) comes back to this.

> ⚠️ **Common mistake: retrying requests that are not idempotent.** If the *answer* was lost, the
> server has already done the work once. Retrying "what is the address of example.com?" is harmless;
> retrying "transfer 10 euros" may do it twice. Make requests idempotent, or let the server remember
> recent request IDs and answer duplicates from memory.

> 🔒 **Security:** a UDP server cannot know whether the source address of a datagram is real (**IP
> spoofing**). If a tiny request makes your server send a big answer, an attacker can forge the
> victim's address and use your server to flood the victim (an **amplification attack**). Rules:
> keep answers no bigger than requests for unknown peers, rate-limit per source address, and validate
> every datagram before you act on it.

> 🧪 **Testing:** UDP tests are easy to make deterministic. Bind every socket to `127.0.0.1:0`, simulate
> loss in a test server (like `lossy_server`), and keep timeouts short (50 ms) so that a test that waits
> for a timeout still finishes quickly. Exercise 2 does exactly this.

## Common mistakes

> ⚠️ **Common mistake: a receive buffer that is too small.** Truncated datagrams lose their end without
> an error. Size the buffer for the largest datagram your protocol allows (for example 1500 bytes, or
> 65,535 if you do not know).

> ⚠️ **Common mistake: forgetting the timeout.** Without `set_read_timeout`, a client whose request was
> lost waits in `recv_from` forever. Every UDP client needs a timeout.

> ⚠️ **Common mistake: trusting the sender address.** `recv_from` tells you where a datagram *claims* to
> come from. Use it to send the answer, not as proof of identity.

## Try it yourself

1. Start `udp_echo` and run `nc -u 127.0.0.1 7100` interactively. Type several lines. Each line is one
   datagram, and each comes back.
2. Stop the server and type another line into the same `nc`. Nothing comes back. Depending on its
   version, `nc` keeps waiting or quietly exits after the ICMP error from section 3; it never gets a
   clear message that the receiver is gone.
3. In `datagrams.rs`, change the buffer to `[0u8; 64]` and check that the fourth message arrives whole.
4. In `retry.rs`, change `count % 2 == 1` to `count % 3 != 0` (only every third datagram is answered)
   and watch requests need three attempts.

## Exercises

### Exercise 1 — A UDP time server (easy)

Write `src/bin/time_server.rs`: a server on `127.0.0.1:7101` that answers every datagram, whatever it
contains, with the current time as seconds since 1 January 1970 (the *Unix time*) and a newline. Test it
with `echo x | nc -u -w 1 127.0.0.1 7101`.

<details>
<summary>Hint</summary>

`SystemTime::now().duration_since(UNIX_EPOCH)` from `std::time` gives a `Duration`; `.as_secs()` turns
it into a number. The content of the request can be ignored with `let (_, peer) = ...`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::net::UdpSocket;
use std::time::{SystemTime, UNIX_EPOCH};

fn main() -> std::io::Result<()> {
    let socket = UdpSocket::bind("127.0.0.1:7101")?;
    let mut buf = [0u8; 64];
    loop {
        let (_, peer) = socket.recv_from(&mut buf)?; // the content does not matter
        let seconds = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .expect("the clock is after 1970")
            .as_secs();
        socket.send_to(format!("{seconds}\n").as_bytes(), peer)?;
    }
}
```

`echo x | nc -u -w 1 127.0.0.1 7101` printed `1794986824`; your number will be larger. Compare it with
`date +%s` in the terminal.

Why it works: the answer is small, and asking twice does no harm, so this protocol needs no request IDs
at all. (A public server like this would still be a small amplification risk: its answer is longer than
a one-byte request.)

</details>

### Exercise 2 — Test the retry logic (medium)

Add a `tests` module to `retry.rs` with three tests: (a) `request` succeeds when the first datagram is
lost (use `lossy_server`); (b) it returns an error of kind `TimedOut` when nobody answers; (c) it ignores
an answer with the wrong ID and accepts the next one with the right ID. Keep each test fast by using
short timeouts.

<details>
<summary>Hint</summary>

For (b), bind a socket and simply never read from it: datagrams to it are accepted by the kernel but
never answered. For (c), write a tiny server in the test thread that first answers with the bytes
`b"\x00\x09stale"` (ID 9) and then echoes the retry unchanged.

</details>

<details>
<summary>Answer</summary>

Add this at the end of `retry.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn client(timeout_ms: u64) -> UdpSocket {
        let socket = UdpSocket::bind("127.0.0.1:0").unwrap();
        socket
            .set_read_timeout(Some(Duration::from_millis(timeout_ms)))
            .unwrap();
        socket
    }

    #[test]
    fn retries_after_a_lost_datagram() {
        let server = UdpSocket::bind("127.0.0.1:0").unwrap();
        let addr = server.local_addr().unwrap();
        thread::spawn(move || lossy_server(server));

        let answer = request(&client(50), addr, 7, b"abc").unwrap();
        assert_eq!(answer, b"ABC");
    }

    #[test]
    fn gives_up_when_nobody_answers() {
        let silent = UdpSocket::bind("127.0.0.1:0").unwrap(); // bound, but never reads
        let addr = silent.local_addr().unwrap();

        let err = request(&client(50), addr, 1, b"hi").unwrap_err();
        assert_eq!(err.kind(), ErrorKind::TimedOut);
    }

    #[test]
    fn ignores_an_answer_with_another_id() {
        let server = UdpSocket::bind("127.0.0.1:0").unwrap();
        let addr = server.local_addr().unwrap();
        thread::spawn(move || {
            let mut buf = [0u8; 64];
            let (_, peer) = server.recv_from(&mut buf).unwrap();
            server.send_to(b"\x00\x09stale", peer).unwrap(); // wrong id
            let (n, peer) = server.recv_from(&mut buf).unwrap(); // the retry
            server.send_to(&buf[..n], peer).unwrap(); // right id
        });

        let answer = request(&client(200), addr, 3, b"fresh").unwrap();
        assert_eq!(answer, b"fresh");
    }
}
```

`cargo test --bin retry` reports `test result: ok. 3 passed; ... finished in 0.15s`.

Why it works: each test builds exactly the network behaviour it needs: loss, silence, a stale answer.
Nothing depends on a real network, and the slowest test waits only 3 × 50 ms.

</details>

### Exercise 3 — A chat relay (hard)

Write `src/bin/relay.rs`: a UDP server on `127.0.0.1:7102` that remembers every address it has received
a datagram from, and forwards each datagram to all *other* known peers. Write a test with three client
sockets: when `c` sends `hi from c`, both `a` and `b` receive it, and `c` itself receives nothing.

<details>
<summary>Hint</summary>

Keep a `HashSet<SocketAddr>` of peers. In the test, `connect` each client socket to the relay so that
you can use `send` and `recv`, and give each a short read timeout, so that "receives nothing" becomes a
`WouldBlock` error after the timeout.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashSet;
use std::net::{SocketAddr, UdpSocket};

/// Forwards every datagram to all other peers that have ever sent something.
fn relay(socket: UdpSocket) -> std::io::Result<()> {
    let mut peers: HashSet<SocketAddr> = HashSet::new();
    let mut buf = [0u8; 1500];
    loop {
        let (n, from) = socket.recv_from(&mut buf)?;
        peers.insert(from);
        for &peer in peers.iter().filter(|&&p| p != from) {
            socket.send_to(&buf[..n], peer)?;
        }
    }
}

fn main() -> std::io::Result<()> {
    let socket = UdpSocket::bind("127.0.0.1:7102")?;
    println!("relay on {}", socket.local_addr()?);
    relay(socket)
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;
    use std::time::Duration;

    fn peer() -> UdpSocket {
        let s = UdpSocket::bind("127.0.0.1:0").unwrap();
        s.set_read_timeout(Some(Duration::from_millis(300)))
            .unwrap();
        s
    }

    fn recv_text(s: &UdpSocket) -> Option<String> {
        let mut buf = [0u8; 100];
        let n = s.recv(&mut buf).ok()?;
        Some(String::from_utf8_lossy(&buf[..n]).into_owned())
    }

    #[test]
    fn messages_reach_everybody_else() {
        let server = UdpSocket::bind("127.0.0.1:0").unwrap();
        let addr = server.local_addr().unwrap();
        thread::spawn(move || relay(server));

        let (a, b, c) = (peer(), peer(), peer());
        for s in [&a, &b, &c] {
            s.connect(addr).unwrap();
        }
        a.send(b"a joined").unwrap(); // nobody else known yet
        b.send(b"b joined").unwrap();
        assert_eq!(recv_text(&a).as_deref(), Some("b joined"));
        c.send(b"hi from c").unwrap();
        assert_eq!(recv_text(&a).as_deref(), Some("hi from c"));
        assert_eq!(recv_text(&b).as_deref(), Some("hi from c"));
        assert_eq!(recv_text(&c), None); // the sender gets no copy
    }
}
```

`cargo test --bin relay` reports `1 passed`. You can also try it by hand: run the relay and connect two
or three `nc -u 127.0.0.1 7102` windows; each one must type a line first, so that the relay learns its
address.

Why it works: one socket and one loop serve every peer, because UDP keeps no per-peer state; the program
keeps its own state (the set of peers). In a real chat you would also remove peers that have been
silent for a while, since UDP never tells you that a peer has left.

</details>

## Quiz

1. Why does a UDP server need neither `listen` nor `accept`?

<details>
<summary>Answer</summary>

There are no connections in UDP. The socket receives every datagram sent to its port, and each
datagram carries its sender's address, so one socket can serve all peers.

</details>

2. A peer sends a 2,000-byte datagram, and you call `recv_from` with a 1,500-byte buffer. What happens?

<details>
<summary>Answer</summary>

You get the first 1,500 bytes; the remaining 500 bytes are thrown away without an error (truncation).

</details>

3. What does `UdpSocket::connect` send over the network?

<details>
<summary>Answer</summary>

Nothing. It only stores a default peer in the kernel, which filters incoming datagrams from other
addresses and lets ICMP errors for that peer reach the program.

</details>

4. Why does a UDP client put a request ID into each request?

<details>
<summary>Answer</summary>

To match answers to requests. After a retry, a late answer to an earlier attempt (or a datagram from
somebody else) may arrive; the ID lets the client recognise and ignore it.

</details>

5. Why are real UDP datagrams usually kept below about 1,200–1,400 bytes?

<details>
<summary>Answer</summary>

Bigger datagrams exceed the MTU and are fragmented; losing any fragment loses the whole datagram, and
many networks drop fragments. Small datagrams travel in one packet.

</details>

6. What is an amplification attack, and why does it work well with UDP?

<details>
<summary>Answer</summary>

The attacker sends small requests with the victim's address forged as the source; the server sends big
answers to the victim. UDP has no handshake, so the server cannot check that the source address is real.

</details>

## Summary

- `UdpSocket::bind`, `recv_from` and `send_to` are all a UDP server needs; one socket serves all peers.
- Each send is one datagram and each receive returns one datagram: borders are kept, but a too-small
  buffer truncates silently, and datagrams over 65,507 bytes cannot be sent.
- Keep datagrams small (well below the MTU) to avoid fragmentation.
- `connect` sets a default peer, filters other senders and surfaces ICMP errors; it sends nothing.
- Reliability is your job: read timeouts, a limited number of retries, request IDs, and idempotent
  requests.
- Never trust the source address; protect public UDP services against amplification.

## Sources and further reading

- [`std::net::UdpSocket`](https://doc.rust-lang.org/std/net/struct.UdpSocket.html) — all methods, including `connect`, `peek` and the multicast options.
- [RFC 768 — User Datagram Protocol](https://datatracker.ietf.org/doc/html/rfc768) — the UDP standard.
- [RFC 8085 — UDP Usage Guidelines](https://datatracker.ietf.org/doc/html/rfc8085) — official advice for protocol designers: message sizes, retries, congestion and security.
- [udp(7) manual page](https://man7.org/linux/man-pages/man7/udp.7.html) — how Linux handles UDP sockets and ICMP errors.
- [CISA — UDP-Based Amplification Attacks](https://www.cisa.gov/news-events/alerts/2014/01/17/udp-based-amplification-attacks) — an alert of CISA (the Cybersecurity and Infrastructure Security Agency of the United States): which real protocols were abused and how much they amplify.

---

[🏠 Start](../START-HERE.md) · [← 15.4 TCP sockets in Rust](04-tcp-sockets-in-rust.md) · [📋 Phase 15](README.md) · [15.6 Async networking with Tokio →](06-async-networking-with-tokio.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
