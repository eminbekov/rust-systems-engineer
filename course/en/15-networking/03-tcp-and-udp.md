<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.2 IP addresses, subnets and routing](02-ip-addresses-and-routing.md) · [📋 Phase 15](README.md) · [15.4 TCP sockets in Rust →](04-tcp-sockets-in-rust.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/03-tcp-and-udp.md) <!-- nav -->

# Lesson 15.3 — TCP and UDP

> **Phase 15 — Networking** · Lesson 3 of 16 · ⏱ about 105 minutes

## What you will learn

- How ports and the "four-tuple" identify one conversation between two programs.
- What TCP promises (a reliable, ordered byte stream) and how it keeps the promise: sequence numbers, acknowledgements, retransmission, flow control and congestion control.
- How a TCP connection opens (the three-way handshake) and closes, read from a real capture, and what the states `LISTEN`, `ESTABLISHED` and `TIME-WAIT` mean.
- What UDP gives you instead (single messages, no guarantees) and when that is the better choice.

## Before you start

- You finished [Lesson 15.2 — IP addresses, subnets and routing](02-ip-addresses-and-routing.md).
- You have the `p15-nettools` image from [Lesson 15.1](01-how-the-internet-works.md) and the capture of the `hello` conversation from there.
- Examples go into `code/p15-net/src/bin/`.

## New words

| Word | Meaning |
|---|---|
| **connection** | In TCP, an agreement between two programs to exchange a stream of bytes, with state kept on both sides. |
| **four-tuple** | The four values that identify a TCP connection: source IP, source port, destination IP, destination port. |
| **ephemeral port** | A temporary port that the operating system picks for the client side of a connection. |
| **byte stream** | Data seen as one long row of bytes with no message borders, like a file being read. |
| **sequence number** | The number of the first byte in a TCP segment. It lets the receiver put bytes in order and find gaps. |
| **acknowledgement** (ACK) | A message "I have received everything before byte N". In TCP it is a number in the header plus the ACK flag. |
| **retransmission** | Sending data again because no acknowledgement came back in time. |
| **RTT** (Round-Trip Time) | The time a packet needs to reach the other side plus the time the answer needs to come back. |
| **flow control** | Stopping the sender from sending more than the *receiver* can store. TCP uses the *receive window* for it. |
| **congestion control** | Stopping the sender from sending more than the *network* can carry. |
| **handshake** | The first messages of a protocol, in which both sides agree to talk. TCP's handshake has three segments. |
| **SYN, ACK, FIN, RST, PSH** | TCP flags: synchronise (open), acknowledge, finish (I am done sending), reset (abort now), push (deliver now). |
| **MSS** (Maximum Segment Size) | The largest amount of data one TCP segment may carry, announced in the handshake. |
| **TIME-WAIT** | The state the side that closed first stays in for a while (60 seconds on Linux) after a connection ends. |
| **datagram** | One UDP message. It is delivered whole or not at all. |
| **ss** (socket statistics) | The Linux command that lists sockets and their states. |
| **TCP / UDP / IP** (Transmission Control Protocol / User Datagram Protocol / Internet Protocol) | The two transport protocols of this lesson, and the internet protocol under them (Lesson 15.1). |
| **HTTP, HTTPS, DNS, SSH, TLS** | Protocols on top of TCP or UDP that you met in Lesson 15.1: HyperText Transfer Protocol (the web), HTTP Secure (HTTP inside TLS), Domain Name System (names to addresses), Secure Shell (remote login), Transport Layer Security (encryption). |
| **ICMP** (Internet Control Message Protocol) and **QUIC** | ICMP carries control and error messages such as "port unreachable"; QUIC is a newer transport built on UDP (Lesson 15.13). |
| **URG** (urgent) | A TCP flag for "urgent data"; it exists in the header but is almost never used. |

## 1. Ports and conversations

The IP layer brings a packet to the right **computer**. But a computer runs many programs that use the
network at the same time. The transport layer adds **ports**: 16-bit numbers that say which program a
packet belongs to. A server *listens* on a well-known port (80 for HTTP, 443 for HTTPS, 53 for DNS, 22
for SSH). A client gets an **ephemeral port** from its operating system. On Linux the range is:

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

```text
32768	60999
```

(macOS uses 49152 to 65535.) One TCP connection is identified by the **four-tuple**:

```text
 (source IP,  source port,  destination IP,  destination port)
 (127.0.0.1,  36932,        127.0.0.1,       9000)            ← the "hello" connection of 15.1
```

Because the client port differs, one server port can serve thousands of clients at the same time. Add
the protocol (TCP or UDP) and you have the "five-tuple" that firewalls and load balancers use.

## 2. What TCP promises

IP only *tries* to deliver packets. They can be lost, duplicated or reordered. TCP builds on top of IP
and gives your program something much nicer: a **reliable, ordered byte stream**. What you write on one
side comes out on the other side, every byte, in the same order, or you get an error. To keep this
promise TCP uses five ideas:

1. **Sequence numbers.** Every byte has a number. A segment carries the number of its first byte. The
   receiver can therefore sort segments and see gaps.
2. **Acknowledgements.** The receiver sends back "ACK N": "I have everything before byte N, send N
   next".
3. **Retransmission.** If the sender gets no acknowledgement within a timeout (a bit more than one
   RTT), it sends the data again. Duplicates are recognised by their sequence numbers and dropped.
4. **Flow control.** Every segment carries a **window**: how many more bytes the receiver can store.
   If the receiving program reads slowly, its window shrinks, and the sender must wait. At window 0 the
   sender stops completely.
5. **Congestion control.** The sender also guesses how much the *network* can carry. It starts slowly,
   sends more while everything is acknowledged, and backs off sharply when packets get lost, because
   loss usually means a router queue is full.

There is one more property, which surprises many beginners and matters in Lesson 15.7: TCP is a
**stream of bytes, not of messages**. If you send `hello` and then `world` with two writes, the other
side may read `helloworld` in one read, or `hel` and then `loworld`. TCP keeps the bytes and their order;
it does not keep your message borders.

## 3. Open, talk, close: reading a real conversation

Here are the 8 packets of the `hello` capture from Lesson 15.1, without the hex dump. The client is port
`36932`, the server port `9000`:

```text
1  127.0.0.1.36932 > 127.0.0.1.9000: Flags [S], seq 4058503438, win 65495, options [mss 65495,sackOK,TS val 2778587979 ecr 0,nop,wscale 10], length 0
2  127.0.0.1.9000 > 127.0.0.1.36932: Flags [S.], seq 2766373507, ack 4058503439, win 65483, options [mss 65495,sackOK,TS val 1523372706 ecr 2778587979,nop,wscale 10], length 0
3  127.0.0.1.36932 > 127.0.0.1.9000: Flags [.], ack 1, win 64, options [nop,nop,TS val 2778587979 ecr 1523372706], length 0
4  127.0.0.1.36932 > 127.0.0.1.9000: Flags [P.], seq 1:7, ack 1, win 64, options [nop,nop,TS val 2778587979 ecr 1523372706], length 6
5  127.0.0.1.9000 > 127.0.0.1.36932: Flags [.], ack 7, win 64, options [nop,nop,TS val 1523372706 ecr 2778587979], length 0
6  127.0.0.1.36932 > 127.0.0.1.9000: Flags [F.], seq 7, ack 1, win 64, options [nop,nop,TS val 2778587979 ecr 1523372706], length 0
7  127.0.0.1.9000 > 127.0.0.1.36932: Flags [F.], seq 1, ack 8, win 64, options [nop,nop,TS val 1523372706 ecr 2778587979], length 0
8  127.0.0.1.36932 > 127.0.0.1.9000: Flags [.], ack 2, win 64, options [nop,nop,TS val 2778587979 ecr 1523372706], length 0
```

(The line numbers on the left were added, and the time stamps removed, to make it readable.) In tcpdump,
`S` is SYN, `.` is ACK, `P` is PSH, `F` is FIN, and `R` would be RST. The same conversation as a picture:

```text
 client :36932                                         server :9000
      │ ── 1  SYN      seq=4058503438 ───────────────────► │   "let's talk, my bytes start here"
      │ ◄─ 2  SYN+ACK  seq=2766373507 ack=4058503439 ───── │   "OK, mine start here; got yours"
      │ ── 3  ACK      ack=1 ────────────────────────────► │   handshake done: ESTABLISHED
      │ ── 4  PSH+ACK  seq=1:7  "hello\n" ───────────────► │   6 bytes of data
      │ ◄─ 5  ACK      ack=7 ──────────────────────────── │   "I have bytes 1..6, next is 7"
      │ ── 6  FIN+ACK  seq=7 ────────────────────────────► │   client: "I will send no more"
      │ ◄─ 7  FIN+ACK  seq=1 ack=8 ────────────────────── │   server: "me neither"
      │ ── 8  ACK      ack=2 ────────────────────────────► │   client enters TIME-WAIT
```

**What happens:**

1. **Packets 1–3, the three-way handshake.** Each side picks a random starting sequence number (the
   *initial sequence number*), sends it with SYN, and acknowledges the other's number plus one. A SYN
   counts as one byte, so the server answers `ack 4058503439` (= 4058503438 + 1). Random starting
   numbers make it hard for an attacker to inject fake segments into a connection.
2. The `options` agree on details: `mss 65495` is the largest segment each side accepts (large because
   this is the loopback interface; on Ethernet it is 1460), and `wscale` lets the window be larger
   than 65,535 bytes.
3. **From packet 3 on, tcpdump shows relative numbers**: `ack 1` means "starting number + 1". This is
   only how tcpdump prints them; the real header still holds the big numbers.
4. **Packets 4–5, data.** `seq 1:7` carries bytes 1 to 6 (`hello\n`, `length 6`). The server answers
   `ack 7`: everything before byte 7 has arrived. `win 64` (multiplied by 2^10 because of `wscale 10`)
   is the receive window.
5. **Packets 6–8, closing.** Each direction is closed separately with a FIN. The client sends FIN
   (it had no more data), the server answers with its own FIN, and the client acknowledges it. A FIN
   also counts as one byte, so the acknowledgements become `ack 8` and `ack 2`.

The kernel does all of this. In Rust, the handshake happens inside `TcpStream::connect`, the data
segments inside `write`, and the FIN when the stream is dropped (or when you call `shutdown`). You can
even see that the handshake does not need your server code. Save as `src/bin/handshake.rs`:

```rust
use std::net::{TcpListener, TcpStream};
use std::time::Instant;

fn main() -> std::io::Result<()> {
    // Port 0 asks the operating system for any free port.
    let listener = TcpListener::bind("127.0.0.1:0")?;
    let addr = listener.local_addr()?;

    let start = Instant::now();
    let stream = TcpStream::connect(addr)?; // we never call accept()!
    println!("connected to {addr} in {:?}", start.elapsed());
    println!("our side of the connection: {}", stream.local_addr()?);

    drop(listener); // close the listening socket
    match TcpStream::connect(addr) {
        Ok(_) => println!("connected again?"),
        Err(e) => println!("second connect: {e} (kind: {:?})", e.kind()),
    }
    Ok(())
}
```

**What happens:**

1. Binding to port `0` lets the operating system choose a free port; `local_addr()` tells us which one.
2. `connect` returns after the three-way handshake. It succeeds although the program never calls
   `accept()`: the kernel completes handshakes for a listening socket on its own and keeps the finished
   connections in a queue (the *backlog*) until the program accepts them (Lesson 15.4).
3. After `drop(listener)` nobody listens on that port. The kernel answers the next SYN with RST, and
   `connect` returns an error of kind `ConnectionRefused`.

`cargo run --bin handshake` on the Mac prints (your ports and times will differ):

```text
connected to 127.0.0.1:53416 in 63.083µs
our side of the connection: 127.0.0.1:53417
second connect: Connection refused (os error 61) (kind: ConnectionRefused)
```

The handshake took 63 microseconds because the "network" was the loopback interface. Across the world it
takes one RTT, often 50–150 ms, before the first byte of data can even be sent. This is why opening a new
connection for every request is slow, and why HTTP keeps connections open (Lesson 15.8).

The refused connection looks like this in tcpdump (captured in the container with `nc -v 127.0.0.1 9001`
while nothing listened on port 9001):

```text
nc: connect to 127.0.0.1 port 9001 (tcp) failed: Connection refused
07:15:11.419945 IP 127.0.0.1.56060 > 127.0.0.1.9001: Flags [S], seq 283570947, win 65495, options [mss 65495,sackOK,TS val 1777619782 ecr 0,nop,wscale 10], length 0
07:15:11.419954 IP 127.0.0.1.9001 > 127.0.0.1.56060: Flags [R.], seq 0, ack 283570948, win 0, length 0
```

A SYN, answered immediately by RST. Remember this pattern: "connection refused" means the computer was
reachable, but no program listened on that port. If instead nothing at all comes back, the connect hangs
until a timeout: then a firewall or a routing problem is the more likely cause.

## 4. Connection states with `ss`

Both sides keep state for each connection, and `ss` (socket statistics) shows it. In the container,
start a server, connect a client that stays two seconds, and look twice:

```bash
docker run --rm -it p15-nettools bash
nc -l 127.0.0.1 9000 > /dev/null &
(sleep 2) | nc -N 127.0.0.1 9000 &
sleep 1
ss -tan
sleep 3
ss -tan
```

(`-t` TCP only, `-a` all sockets including listening ones, `-n` numbers. `nc -N` closes the connection
when its input ends, after the `sleep 2`.)

```text
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port 
LISTEN 0      1          127.0.0.1:9000       0.0.0.0:*    
ESTAB  0      0          127.0.0.1:9000     127.0.0.1:47088
ESTAB  0      0          127.0.0.1:47088    127.0.0.1:9000 
State     Recv-Q Send-Q Local Address:Port  Peer Address:Port
TIME-WAIT 0      0          127.0.0.1:47088    127.0.0.1:9000
```

1. `LISTEN` is the listening socket: it accepts new connections and never carries data itself.
2. Two `ESTAB` (established) lines: the *same* connection seen from both ends, because client and
   server run on one machine. Note the swapped local and peer addresses.
3. After the close, the server side is gone, but the client side is in **TIME-WAIT**. The side that
   sends the first FIN waits there for 60 seconds on Linux, so that late, delayed segments of the old
   connection cannot be mistaken for a new connection with the same four-tuple.
4. `Recv-Q` and `Send-Q` show bytes waiting in the kernel buffers. A growing `Recv-Q` on an established
   socket means the program does not read fast enough.

> 💡 **Tip:** two states tell you a lot in production. Thousands of `TIME-WAIT` sockets on a client
> mean it opens a new connection for every request: reuse connections. Many sockets in `CLOSE-WAIT`
> mean the *other* side has closed, but your program never closed its end: a bug in your code, usually
> a stream that is never dropped.

## 5. UDP: just send a message

UDP adds almost nothing to IP: ports, a length and a checksum. Its header is only 8 bytes:

```text
 ┌──────────────────┬──────────────────┐
 │ source port      │ destination port │
 ├──────────────────┼──────────────────┤
 │ length           │ checksum         │
 └──────────────────┴──────────────────┘
   then the data of one datagram
```

There is no connection, no handshake, no acknowledgement, no retransmission and no ordering. Each
`send` produces one **datagram**, and each receive returns exactly one whole datagram: UDP **keeps
message borders**. Two messages sent with `nc -u` in the container:

```bash
tcpdump -i lo -n -c 2 udp port 9002 &
printf "first\n" | nc -u -w 0 127.0.0.1 9002
printf "second\n" | nc -u -w 0 127.0.0.1 9002
```

```text
07:15:28.417843 IP 127.0.0.1.47723 > 127.0.0.1.9002: UDP, length 6
07:15:28.419811 IP 127.0.0.1.46760 > 127.0.0.1.9002: UDP, length 7
```

Two packets, nothing else. Nobody listened on port 9002, and the sender never found out. When a
datagram reaches a port without a listener, the receiving kernel may answer with an ICMP message
(captured here for port 9003):

```text
07:15:15.438910 IP 127.0.0.1.39002 > 127.0.0.1.9003: UDP, length 7
07:15:15.438949 IP 127.0.0.1 > 127.0.0.1: ICMP 127.0.0.1 udp port 9003 unreachable, length 43
```

But that message is a courtesy: firewalls often drop it, and lost datagrams produce no message at all.
With UDP, *your program* must decide what to do about loss, order and duplicates (Lesson 15.5).

Why would anyone want that? Because sometimes the guarantees of TCP get in the way:

| | TCP | UDP |
|---|---|---|
| Setup before data | handshake, one RTT | none |
| Delivery | reliable, ordered, no duplicates | maybe, any order, maybe twice |
| Unit | byte stream (no message borders) | datagram (borders kept) |
| A lost packet | everything after it waits until it is resent | only that message is lost |
| Used by | HTTP/1.1 and HTTP/2, SSH, databases, TLS | DNS, video and voice calls, games, QUIC and HTTP/3 |

The row "a lost packet" is the key. TCP delivers in order, so one lost segment holds back all later
bytes, even if they already arrived. This is called **head-of-line blocking**. For a voice call, a
sound that arrives late is useless; it is better to skip it. DNS uses UDP because a question and an
answer fit into one datagram each, and a lost one is simply asked again. QUIC (Lesson 15.13) builds its
own reliable streams on top of UDP to avoid head-of-line blocking between streams.

> ⚡ **Performance:** the costs of TCP are the handshake (one RTT before any data), head-of-line
> blocking, and the slow start of congestion control on every new connection. Most of them disappear
> when a connection is reused for many requests. Measure before you decide that you "need UDP".

## Common mistakes

> ⚠️ **Common mistake: expecting one read per write.** A server that does one `read` and assumes it
> got exactly one message works on loopback in tests and fails on a real network, where segments are
> split and merged. TCP does not keep message borders. Lesson 15.7 shows how to frame messages.

> ⚠️ **Common mistake: treating "refused" and "timed out" the same.** `ConnectionRefused` means the host
> answered with RST: the program is not running or listens on another port or address. A timeout means
> no answer at all: check firewalls, routing and the address itself.

> ⚠️ **Common mistake: assuming UDP reports errors.** `send_to` succeeds even if nobody receives the
> datagram. Only a reply (or its absence after a timeout) tells you anything.

## Try it yourself

1. In the container, start `tcpdump -i lo -n tcp port 9000 &`, then `nc -l 127.0.0.1 9000 &`, then
   connect with `nc 127.0.0.1 9000` and type two lines, pressing Enter after each.
2. Find the two data segments. Each has the length of one line plus 1 (the newline), and the sequence
   numbers continue. If you typed `abc` and `de`, you see `seq 1:5` with `length 4`, then `seq 5:8`
   with `length 3`.
3. Press Ctrl+C in the client and find the FIN segments. Which side sent the first FIN?
4. Run `ss -tan` and find the `TIME-WAIT` socket. Is it on the side that closed first?

## Exercises

### Exercise 1 — Who closed first? (easy)

On a server you see this line with `ss -tan`:

```text
TIME-WAIT 0      0          10.0.0.5:8080     10.0.0.9:51312
```

Which program sent the first FIN: the one on `10.0.0.5:8080` or the one on `10.0.0.9:51312`? Which of
them is probably the server?

<details>
<summary>Hint</summary>

TIME-WAIT is kept by the side that closes first. Which port looks like a server port, and which like an
ephemeral one?

</details>

<details>
<summary>Answer</summary>

The socket in TIME-WAIT belongs to `10.0.0.5:8080`, so that side sent the first FIN. Port 8080 is a
typical server port and 51312 an ephemeral client port: here the **server** closed the connection
first, which is normal for HTTP servers that answer with `Connection: close`.

</details>

### Exercise 2 — Decode TCP flags, with tests (medium)

The flags of a TCP segment are the bits of byte 13 of the TCP header: FIN = `0x01`, SYN = `0x02`,
RST = `0x04`, PSH = `0x08`, ACK = `0x10`, URG = `0x20`. Write `src/bin/tcp_flags.rs` with
`fn tcp_flags(flags: u8) -> String` that writes them like tcpdump, in the order F, S, R, P, `.` (ACK),
U, and `"none"` when no flag is set. Test at least SYN (`"S"`), SYN+ACK (`"S."`), PSH+ACK (`"P."`) and
FIN+PSH+ACK (`"FP."`). Byte 13 of the `hello` packet from Lesson 15.1 is `0x18`; what does your function
print for it?

<details>
<summary>Hint</summary>

Keep a list of `(bit, character)` pairs in the right order and check each bit with `flags & bit != 0`
(Lesson 12.4).

</details>

<details>
<summary>Answer</summary>

```rust
/// Writes TCP flags the way tcpdump does: "S" for SYN, "." for ACK, and so on.
fn tcp_flags(flags: u8) -> String {
    const NAMES: [(u8, char); 6] = [
        (0x01, 'F'), // FIN: no more data from me
        (0x02, 'S'), // SYN: synchronise sequence numbers (open)
        (0x04, 'R'), // RST: reset, abort the connection
        (0x08, 'P'), // PSH: push the data to the program now
        (0x10, '.'), // ACK: the acknowledgement number is valid
        (0x20, 'U'), // URG: urgent data (almost never used)
    ];
    let mut out = String::new();
    for (bit, name) in NAMES {
        if flags & bit != 0 {
            out.push(name);
        }
    }
    if out.is_empty() {
        out.push_str("none");
    }
    out
}

fn main() {
    // Byte 13 of the TCP header in the "hello" packet of Lesson 15.1 is 0x18.
    for byte in [0x02, 0x12, 0x10, 0x18, 0x11, 0x14] {
        println!("{byte:#04x} -> [{}]", tcp_flags(byte));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn handshake_flags() {
        assert_eq!(tcp_flags(0x02), "S");
        assert_eq!(tcp_flags(0x12), "S.");
        assert_eq!(tcp_flags(0x10), ".");
    }

    #[test]
    fn data_and_closing_flags() {
        assert_eq!(tcp_flags(0x18), "P.");
        assert_eq!(tcp_flags(0x11), "F.");
        assert_eq!(tcp_flags(0x19), "FP.");
        assert_eq!(tcp_flags(0x14), "R.");
    }

    #[test]
    fn no_flags() {
        assert_eq!(tcp_flags(0), "none");
    }
}
```

```text
0x02 -> [S]
0x12 -> [S.]
0x10 -> [.]
0x18 -> [P.]
0x11 -> [F.]
0x14 -> [R.]
```

`cargo test --bin tcp_flags` reports `3 passed`. Why it works: each flag is one bit, so the byte is a
small set of yes/no answers; testing each bit in a fixed order reproduces tcpdump's notation. `0x18` is
`[P.]`, exactly what tcpdump printed for the data packet.

</details>

### Exercise 3 — Follow the numbers (medium)

In the capture of section 3 the client's initial sequence number is 4058503438. Using real (absolute)
numbers, answer: (a) What is the sequence number of the byte `h` of `hello`? (b) Which acknowledgement
number does the server send in packet 5? (c) What is the sequence number of the client's FIN, and which
number acknowledges it?

<details>
<summary>Hint</summary>

The SYN uses up one sequence number. The data then starts at the next number. A FIN uses up one number
too. Relative number 1 in tcpdump means "initial number + 1".

</details>

<details>
<summary>Answer</summary>

(a) 4058503439 (initial number + 1, because the SYN used 4058503438). (b) The data covers 4058503439 to
4058503444 (six bytes), so the server acknowledges 4058503445 (relative `ack 7`). (c) The FIN has
sequence number 4058503445 (relative `seq 7`) and is acknowledged with 4058503446 (relative `ack 8`).

</details>

## Quiz

1. Why can one server port (say 443) serve thousands of clients at the same time?

<details>
<summary>Answer</summary>

A connection is identified by the whole four-tuple. Each client has a different IP address or a
different ephemeral port, so every connection is distinct even though the server side is the same.

</details>

2. What does an acknowledgement number of 1000 mean?

<details>
<summary>Answer</summary>

"I have received every byte before byte 1000; the next byte I expect is 1000."

</details>

3. What is the difference between flow control and congestion control?

<details>
<summary>Answer</summary>

Flow control protects the receiver: the sender may not exceed the receive window, which the receiver
announces. Congestion control protects the network: the sender limits itself based on loss and delay,
because full router queues drop packets.

</details>

4. A client calls `connect` and gets "connection refused". What happened on the wire?

<details>
<summary>Answer</summary>

The client sent a SYN, and the server's computer answered with RST because no socket listened on that
address and port.

</details>

5. Which side of a TCP connection ends up in TIME-WAIT, and why does the state exist?

<details>
<summary>Answer</summary>

The side that sent the first FIN. It waits so that delayed segments of the old connection die out
before the same four-tuple can be used again, and so that it can resend the last ACK if it was lost.

</details>

6. Why does DNS normally use UDP and not TCP?

<details>
<summary>Answer</summary>

A question and its answer each fit in one datagram. UDP needs no handshake, so one round trip is
enough, and a lost datagram is simply asked again after a short timeout.

</details>

## Summary

- Ports pick the program; a TCP connection is identified by the four-tuple (source IP, source port,
  destination IP, destination port). Clients use ephemeral ports.
- TCP gives a **reliable, ordered byte stream** using sequence numbers, acknowledgements,
  retransmission, flow control (receive window) and congestion control.
- A connection opens with the three-way handshake (SYN, SYN+ACK, ACK) and closes with a FIN in each
  direction; RST aborts, and "connection refused" is a SYN answered by RST.
- `ss -tan` shows states: `LISTEN`, `ESTAB`, `TIME-WAIT` (side that closed first), `CLOSE-WAIT` (your
  program has not closed yet).
- TCP does **not** keep message borders; UDP does, but gives no delivery, order or duplicate guarantees.
- UDP fits small request/answer protocols and real-time data; TCP's costs are the handshake RTT and
  head-of-line blocking.

## Sources and further reading

- [RFC 9293 — Transmission Control Protocol (TCP)](https://datatracker.ietf.org/doc/html/rfc9293) — the current TCP standard; section 3.5 covers opening and closing connections.
- [RFC 768 — User Datagram Protocol](https://datatracker.ietf.org/doc/html/rfc768) — the whole UDP standard, three pages long.
- [High Performance Browser Networking — Building Blocks of TCP](https://hpbn.co/building-blocks-of-tcp/) — a free book chapter on the handshake, flow control and congestion control and what they cost.
- [tcp(7) manual page](https://man7.org/linux/man-pages/man7/tcp.7.html) — the Linux TCP settings and socket options.
- [ss(8) manual page](https://man7.org/linux/man-pages/man8/ss.8.html) — all options of `ss`.

---

[🏠 Start](../START-HERE.md) · [← 15.2 IP addresses, subnets and routing](02-ip-addresses-and-routing.md) · [📋 Phase 15](README.md) · [15.4 TCP sockets in Rust →](04-tcp-sockets-in-rust.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
