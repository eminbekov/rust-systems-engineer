<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 15 — Networking](README.md) · [📋 Phase 15](README.md) · [15.2 IP addresses, subnets and routing →](02-ip-addresses-and-routing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/01-how-the-internet-works.md) <!-- nav -->

# Lesson 15.1 — How the internet works: layers and packets

> **Phase 15 — Networking** · Lesson 1 of 16 · ⏱ about 135 minutes

## What you will learn

- Why data travels through a network as many small, independent **packets**.
- The four layers of the internet (link, internet, transport, application) and what each one is responsible for.
- How **encapsulation** works: every layer wraps the data from the layer above in its own header.
- How to capture real packets with `tcpdump` in a Linux container and read them byte by byte.
- How to decode an IPv4 header with a small Rust program.

## Before you start

- You finished [Phase 14 — Concurrency and async in depth](../14-concurrency-and-async/README.md).
- You built a web server on `TcpListener` and `TcpStream` in [Lesson 06.11](../06-rust-advanced/11-project-multithreaded-web-server.md). That lesson used TCP and HTTP as black boxes; this phase opens them.
- You know bits, bytes, hexadecimal numbers and byte order from Phase 12 ([Lesson 12.1 — Bits, bytes and number systems](../12-computer-systems/01-bits-bytes-and-number-systems.md)), and bitwise operations such as `>>` and `&` ([Lesson 12.4](../12-computer-systems/04-bitwise-operations.md)).
- You know Docker from [Lesson 11.12 — Containers and Docker basics](../11-linux-shell-and-containers/12-containers-and-docker-basics.md) and the network commands from [Lesson 11.11](../11-linux-shell-and-containers/11-networking-commands.md).

**How the code of this phase is organised.** All small examples live in one practice package,
`code/p15-net`. Each example is its own program in `src/bin/<name>.rs` and runs with
`cargo run --bin <name>`. The two projects of the phase get their own packages (`code/p15-dns` and
`code/p15-lb`). Create the practice package now:

```bash
cd ~/projects/rust
cargo new code/p15-net
cd code/p15-net
mkdir -p src/bin docker
```

**Linux tools in a container.** Packet capture needs Linux tools and special permissions. You never
run them on your Mac itself. Use the VM `dev` from Phase 11 (with `sudo`), or this small image. Save
it as `code/p15-net/docker/Dockerfile`:

```dockerfile
FROM ubuntu:26.04
RUN apt-get update && apt-get install -y --no-install-recommends \
    tcpdump iproute2 netcat-openbsd curl iputils-ping iputils-tracepath ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

Build it once (it takes a minute and prints many lines):

```bash
docker build -t p15-nettools docker
```

The whole phase uses this image, called `p15-nettools`.

## New words

| Word | Meaning |
|---|---|
| **network** | Computers connected so that they can send data to each other. The **internet** is a network of networks. |
| **protocol** | A set of rules that both sides follow: which bytes to send, in which order, and what they mean. |
| **packet** | A small piece of data (usually at most about 1500 bytes) with a header that says where it goes. |
| **header** | Bytes at the front of a packet with information for one layer: addresses, lengths, flags. |
| **payload** | The data that a header carries: everything after the header. |
| **layer** | One level of the network design. Each layer solves one problem and uses the layer below it. |
| **encapsulation** | Wrapping: each layer puts the data from the layer above inside its own header, like a letter inside an envelope. |
| **IP** (Internet Protocol) | The protocol that moves packets from one computer to another across many networks. Versions 4 (IPv4) and 6 (IPv6) are in use. |
| **IP address** | The number that identifies a computer's network interface, for example `127.0.0.1`. |
| **router** | A device that receives packets and forwards each one towards its destination. |
| **TCP** (Transmission Control Protocol) | A protocol on top of IP that gives programs a reliable stream of bytes. Lesson 15.3 explains it. |
| **UDP** (User Datagram Protocol) | A simpler protocol on top of IP that sends single messages with no guarantees. Lesson 15.3. |
| **port** | A number from 0 to 65535 that says which program on a computer a packet is for, for example 80 for a web server. |
| **HTTP** (HyperText Transfer Protocol) | The protocol of the web. You met it in Lesson 06.11. |
| **DNS** (Domain Name System) | The system that turns names like `example.com` into IP addresses. Lesson 15.10. |
| **Ethernet** | The most common link-layer technology for cables. Wi-Fi works almost the same way over radio. |
| **MAC address** (Media Access Control address) | A 6-byte hardware address of a network interface, written like `42:9a:88:e7:5c:d9`. Used only inside one local network. |
| **frame / packet / segment / datagram** | The names of one unit of data at different layers: Ethernet *frame*, IP *packet*, TCP *segment*, UDP *datagram*. |
| **MTU** (Maximum Transmission Unit) | The biggest packet one link can carry. For Ethernet it is 1500 bytes. |
| **network interface** | A connection point between a computer and a network: a network card, Wi-Fi, or the virtual `lo` interface. |
| **socket** | The object a program gets from the operating system to send and receive data over the network. Lesson 15.4 explains it in detail. |
| **loopback** | A virtual interface (`lo`) with the address `127.0.0.1`. Packets sent to it never leave the computer. |
| **TTL** (Time To Live) | A counter in the IP header. Every router lowers it by one; at zero the packet is thrown away, so a lost packet cannot circle forever. |
| **checksum** | A small number computed from data, sent with it, so that the receiver can detect damaged bytes. |
| **capability** (Linux) | One piece of root's power that a process can get on its own. `NET_RAW` allows reading and sending raw packets; `NET_ADMIN` allows configuring network interfaces. |
| **packet capture** | Recording the packets that pass through an interface. `tcpdump` is the classic command-line tool for it. |
| **OSI model** (Open Systems Interconnection model) | An older, seven-layer description of networks. People still use its layer numbers: "L4" means transport, "L7" means application. |
| **SSH** (Secure Shell) | The protocol for logging in to another computer over the network and running commands there, with encryption. |
| **TLS** (Transport Layer Security) and **HTTPS** (HTTP Secure) | TLS encrypts a connection so that nobody on the way can read or change it (Lessons 10.11 and 15.12); HTTPS is HTTP sent inside TLS. |
| **ICMP** (Internet Control Message Protocol) | Small control and error messages of the internet layer, for example "this host is unreachable"; `ping` uses them. |
| **QUIC** | A newer transport protocol built on UDP and used by HTTP/3 (Lesson 15.13). The name is not an abbreviation. |
| **VM** (virtual machine) | A whole computer simulated by software; here the Linux VM `dev` from Phase 11. |
| **ASCII** (American Standard Code for Information Interchange) | The classic table of 128 characters in which, for example, `h` is the byte `0x68` (Lesson 12.5). |
| **MB** (megabyte) | About one million bytes. |

## 1. Data travels as many small letters

Imagine you want to send a whole book to a friend in another city, but the post office accepts only
postcards. You cut the book into small pieces, write each piece on a postcard, number the postcards,
and write the address on each one. The postcards travel separately. Some go through one sorting
office, some through another. Your friend collects them and puts the pages back in order.

The internet works the same way:

- Data is cut into **packets**, each at most about 1500 bytes.
- Every packet has a **header** with the destination address, like the address on a postcard.
- **Routers** are the sorting offices. A router looks only at the destination address and sends the
  packet one step closer. No router knows the whole path.
- Packets can be lost, duplicated, or arrive in a different order. Something at the ends must fix
  that, if the program needs it.

Why not send one big piece? Because a network is shared. If one big transfer took the cable for a
minute, everybody else would wait. Small packets let millions of conversations share the same
cables, and a lost packet costs only a small resend.

## 2. Four layers, each with one job

Moving data across the world is a big problem, so it is split into smaller problems. Each **layer**
solves one problem and trusts the layer below it to solve its own.

| Layer | Its job | Addresses it uses | Examples | OSI number |
|---|---|---|---|---|
| **Application** | What the data *means* | names, paths (`/index.html`) | HTTP, DNS, SSH, TLS | L7 |
| **Transport** | Which *program*; reliability if wanted | ports (`80`, `9000`) | TCP, UDP, QUIC | L4 |
| **Internet** | Which *computer*, across many networks | IP addresses (`104.20.23.154`) | IPv4, IPv6, ICMP | L3 |
| **Link** | The *next hop* on one local network | MAC addresses (`42:9a:88:e7:5c:d9`) | Ethernet, Wi-Fi | L2 (and L1, the cable itself) |

This four-layer picture is called the **TCP/IP model**. The OSI model has seven layers; you only
need its numbers, because engineers say "an L4 load balancer" (it looks at IP addresses and ports)
or "an L7 proxy" (it understands HTTP). You will build an L4 load balancer in Lesson 15.15.

The key idea: **each layer talks only to the same layer on the other side**. Your HTTP code talks to
the server's HTTP code. It does not care whether the bytes went over Wi-Fi, a cable, or the
loopback interface. And a router in the middle reads only the link and internet headers; it never
looks at your HTTP request.

```text
 your computer                                                  server
┌──────────────┐                                           ┌──────────────┐
│ application  │ ◄─────────── HTTP request/response ─────► │ application  │
│ transport    │ ◄─────────── TCP segments ──────────────► │ transport    │
│ internet     │ ◄── IP ──► ┌────────┐ ◄── IP ──► ┌──────┐ │ internet     │
│ link         │ ◄─ Wi-Fi ─►│ router │ ◄─ fibre ─►│router│►│ link         │
└──────────────┘            └────────┘            └──────┘ └──────────────┘
                        routers use only the two lowest layers
```

## 3. Encapsulation: envelopes inside envelopes

When your program sends `hello\n` over TCP, every layer adds its header in front of the data:

```text
                                          ┌──────────┐
 application data                         │ hello\n  │   6 bytes
                                          └──────────┘
                              ┌──────────┐┌──────────┐
 TCP segment                  │TCP header││ hello\n  │   32 + 6 bytes
                              └──────────┘└──────────┘
                  ┌──────────┐┌──────────┐┌──────────┐
 IP packet        │IP header ││TCP header││ hello\n  │   20 + 32 + 6 = 58 bytes
                  └──────────┘└──────────┘└──────────┘
      ┌──────────┐┌──────────┐┌──────────┐┌──────────┐
 frame│ Ethernet ││IP header ││TCP header││ hello\n  │   14 + 58 = 72 bytes
      └──────────┘└──────────┘└──────────┘└──────────┘
```

On the receiving side the layers work in reverse. The network card checks the Ethernet header and
removes it, the operating system checks the IP header and removes it, then the TCP header, and your
program receives just `hello\n`.

Notice the cost: 6 bytes of data needed 66 bytes of headers. That is why sending one byte at a time
is wasteful, and why the **MTU** matters. With an Ethernet MTU of 1500 bytes, one IP packet can
carry at most 1500 − 20 − 20 = 1460 bytes of TCP data (with the smallest TCP header). A program
that sends 1 MB therefore produces about 720 packets.

## 4. See real packets with tcpdump

`tcpdump` records the packets on an interface and prints them. It needs the right to read *all*
traffic, so it runs as root inside a container with two Linux capabilities: `NET_RAW` (open raw
sockets) and `NET_ADMIN` (configure interfaces). Start a container from your image:

```bash
docker run --rm -it --cap-add NET_RAW --cap-add NET_ADMIN p15-nettools bash
```

Inside the container, start a tiny server with `nc` (netcat), start `tcpdump` on the loopback
interface, and send `hello` to the server:

```bash
nc -l 127.0.0.1 9000 > /dev/null &
tcpdump -i lo -n -X -c 8 tcp port 9000 &
sleep 1
printf "hello\n" | nc -q 1 127.0.0.1 9000
```

**What happens:**

1. `nc -l 127.0.0.1 9000` listens on port 9000 and throws away what it receives (`> /dev/null`).
   The `&` runs it in the background.
2. `tcpdump -i lo` captures on the loopback interface. `-n` prints numbers instead of names. `-X`
   prints each packet in hexadecimal and as text. `-c 8` stops after 8 packets. `tcp port 9000`
   is a *filter*: only TCP packets to or from port 9000.
3. The second `nc` connects to the server, sends `hello\n`, waits one second (`-q 1`) and closes.

The conversation produces 8 packets. Seven of them set up and close the connection (you will read
them in Lesson 15.3). Here is the one that carries our data (the output is shortened to this
packet):

```text
06:51:54.219945 IP 127.0.0.1.36932 > 127.0.0.1.9000: Flags [P.], seq 1:7, ack 1, win 64, options [nop,nop,TS val 2778587979 ecr 1523372706], length 6
	0x0000:  4500 003a 9332 4000 4006 a989 7f00 0001  E..:.2@.@.......
	0x0010:  7f00 0001 9044 2328 f1e7 d90f a4e3 8284  .....D#(........
	0x0020:  8018 0040 fe2e 0000 0101 080a a59d e34b  ...@...........K
	0x0030:  5acc d2a2 6865 6c6c 6f0a                 Z...hello.
```

How to read it:

1. The first line is tcpdump's summary: time, `IP` (an IPv4 packet), from `127.0.0.1` port `36932`
   to `127.0.0.1` port `9000`, and `length 6`: six bytes of TCP data. The client port `36932` was
   picked by the operating system; yours will differ.
2. The other lines are the packet bytes. `0x0000` is the offset of the first byte in the line. Each
   group of four hex digits is two bytes. On the right, printable bytes are shown as text.
3. The bytes start at the IP header (`45 00 ...`). On the loopback interface tcpdump does not show
   the Ethernet header. Our `hello.` is at the very end: `68 65 6c 6c 6f 0a` is `h e l l o \n` in
   ASCII.

Now the same idea on the container's real interface, `eth0`, while `curl` loads a public web page.
The option `-e` adds the link-layer (Ethernet) header to each line:

```bash
tcpdump -i eth0 -n -e -c 3 tcp port 80 &
sleep 1
curl -s -o /dev/null http://example.com/
```

```text
06:52:07.171307 42:9a:88:e7:5c:d9 > 3a:71:10:41:e3:23, ethertype IPv4 (0x0800), length 74: 172.17.0.5.39170 > 104.20.23.154.80: Flags [S], seq 3088817596, win 65495, options [mss 65495,sackOK,TS val 1288958365 ecr 0,nop,wscale 10], length 0
06:52:07.268295 3a:71:10:41:e3:23 > 42:9a:88:e7:5c:d9, ethertype IPv4 (0x0800), length 74: 104.20.23.154.80 > 172.17.0.5.39170: Flags [S.], seq 943654894, ack 3088817597, win 65408, options [mss 65495,sackOK,TS val 183731092 ecr 1288958365,nop,wscale 7], length 0
06:52:07.268316 42:9a:88:e7:5c:d9 > 3a:71:10:41:e3:23, ethertype IPv4 (0x0800), length 66: 172.17.0.5.39170 > 104.20.23.154.80: Flags [.], ack 1, win 64, options [nop,nop,TS val 1288958462 ecr 183731092], length 0
```

You can see all the layers in one line:

| Part of the line | Layer | Meaning |
|---|---|---|
| `42:9a:88:e7:5c:d9 > 3a:71:10:41:e3:23` | link | from the container's MAC address to the MAC address of the next hop (Docker's virtual router) |
| `ethertype IPv4 (0x0800)` | link | "the payload of this frame is an IPv4 packet" |
| `172.17.0.5 > 104.20.23.154` | internet | from the container's IP address to the web server's IP address |
| `.39170 > .80` | transport | from a random client port to port 80, the HTTP port |
| `Flags [S]` | transport | a TCP connection request (Lesson 15.3) |

Look at the time stamps: the answer came back about 97 ms after the request. That is the time the
packet needed to travel to the server and back. Your numbers and addresses will be different. Leave
the container with `exit`.

If `tcpdump` prints nothing, you are probably capturing on the wrong interface: traffic to
`127.0.0.1` goes through `lo`, traffic to the internet through `eth0`. `ip -brief addr` lists the
interfaces, and `-i any` captures on all of them.

> 🔒 **Security:** a packet capture can contain passwords, cookies and private messages that were sent
> without encryption. Capture only your own traffic, on your own machines, and delete capture files
> you no longer need. This is also why the web moved to HTTPS (Lesson 10.11): with TLS, a capture
> shows only encrypted bytes.

## 5. Read the IPv4 header with Rust

The first 20 bytes of the packet above are the IPv4 header. Its layout is fixed by RFC 791 (RFC
means Request for Comments: the documents that define internet standards):

```text
 byte:  0         1         2         3
      ┌────┬────┬─────────┬───────────────────┐
  0   │ver │IHL │  TOS    │   total length    │   45 00 | 00 3a
      ├────┴────┴─────────┼───┬───────────────┤
  4   │  identification   │flg│ fragment off. │   93 32 | 40 00
      ├─────────┬─────────┼───┴───────────────┤
  8   │   TTL   │protocol │  header checksum  │   40 06 | a9 89
      ├─────────┴─────────┴───────────────────┤
 12   │          source IP address            │   7f 00 00 01
      ├───────────────────────────────────────┤
 16   │        destination IP address         │   7f 00 00 01
      └───────────────────────────────────────┘
```

- `ver` (4 bits) is the version, 4. `IHL` (Internet Header Length, 4 bits) is the header length in
  32-bit words: 5 words = 20 bytes.
- `total length` is the size of the whole packet: header plus payload.
- `TOS` (Type of Service) can mark a packet as urgent; `identification`, `flg` (flags) and
  `fragment off.` (offset) are used when a packet must be split into smaller ones. We skip them.
- `protocol` says what the payload is: 6 is TCP, 17 is UDP, 1 is ICMP (Internet Control Message
  Protocol, used by `ping`).
- Multi-byte numbers are sent **big-endian** (the most significant byte first). Networks call this
  **network byte order**. That is why `00 3a` means 0x003a = 58.

Save this program as `src/bin/ipv4_header.rs`. It contains the bytes of the captured packet:

```rust
use std::net::Ipv4Addr;

/// The bytes of one real packet, copied from the tcpdump output above.
const PACKET: [u8; 58] = [
    0x45, 0x00, 0x00, 0x3a, 0x93, 0x32, 0x40, 0x00, 0x40, 0x06, 0xa9, 0x89, 0x7f, 0x00, 0x00, 0x01,
    0x7f, 0x00, 0x00, 0x01, 0x90, 0x44, 0x23, 0x28, 0xf1, 0xe7, 0xd9, 0x0f, 0xa4, 0xe3, 0x82, 0x84,
    0x80, 0x18, 0x00, 0x40, 0xfe, 0x2e, 0x00, 0x00, 0x01, 0x01, 0x08, 0x0a, 0xa5, 0x9d, 0xe3, 0x4b,
    0x5a, 0xcc, 0xd2, 0xa2, 0x68, 0x65, 0x6c, 0x6c, 0x6f, 0x0a,
];

fn main() {
    let p = &PACKET;
    let version = p[0] >> 4;
    let header_len = (p[0] & 0x0f) as usize * 4;
    let total_len = u16::from_be_bytes([p[2], p[3]]);
    let ttl = p[8];
    let protocol = p[9];
    let src = Ipv4Addr::new(p[12], p[13], p[14], p[15]);
    let dst = Ipv4Addr::new(p[16], p[17], p[18], p[19]);

    println!("version:       {version}");
    println!("header length: {header_len} bytes");
    println!("total length:  {total_len} bytes");
    println!("TTL:           {ttl}");
    println!("protocol:      {protocol} (6 means TCP)");
    println!("source:        {src}");
    println!("destination:   {dst}");
    println!("IP payload:    {} bytes", p.len() - header_len);
}
```

**What happens:**

1. `PACKET` is a constant array with the 58 bytes from the hex dump, in the same order.
2. `p[0] >> 4` shifts the first byte right by four bits and keeps the upper half: `0x45 >> 4 = 4`.
3. `p[0] & 0x0f` keeps the lower four bits, `5`. The field counts 32-bit words, so we multiply by 4.
4. `u16::from_be_bytes([p[2], p[3]])` builds a 16-bit number from two bytes in big-endian order.
5. `Ipv4Addr::new` from `std::net` builds an address from four bytes. Its `Display` prints the
   usual dotted form.
6. The payload is everything after the header: 58 − 20 = 38 bytes (the TCP header and our data).

Run it with `cargo run --bin ipv4_header`:

```text
version:       4
header length: 20 bytes
total length:  58 bytes
TTL:           64
protocol:      6 (6 means TCP)
source:        127.0.0.1
destination:   127.0.0.1
IP payload:    38 bytes
```

The TTL is 64, the usual starting value on Linux. The packet never passed a router, so nobody
lowered it.

> 🧪 **Testing:** captured packets make excellent test data. Copy the bytes of a real packet into a test
> once, and your parser is checked against reality forever, without a network. Exercise 3 does this.

## 6. Where each layer lives

Who does all this work? Not your program.

```text
 your program         │  application layer: you decide what the bytes mean
 ─────────────────────┼─────────────── socket API (Lesson 15.4) ──────────────
 operating system     │  transport (TCP, UDP) and internet (IP) layers
 kernel               │  routing table, checksums, resending lost packets
 ─────────────────────┼──────────────────────────────────────────────────────
 network card, driver │  link layer: Ethernet / Wi-Fi frames
```

The kernel implements TCP, UDP and IP. Your program sees only the top: it asks the kernel for a
**socket** and reads and writes bytes. The `TcpStream` from Lesson 06.11 is exactly that. In this
phase you learn what the kernel does below the socket (Lessons 15.2 and 15.3), how to use sockets
well (15.4 to 15.6), and how to design the application layer yourself (15.7 and later).

## Common mistakes

> ⚠️ **Common mistake: reading a network number as little-endian.** Your Mac's processor is
> little-endian, but network headers are big-endian. With `u16::from_le_bytes([p[2], p[3]])` the
> total length becomes 14848 instead of 58. There is no error message, just a wrong number. Always
> use `from_be_bytes` for header fields.

> ⚠️ **Common mistake: using the header-length field as bytes.** `IHL` counts 32-bit words. If you
> skip the `* 4`, you cut the packet at byte 5 and read the TCP header from the wrong place. Also
> remember that the header may be longer than 20 bytes when it has options: always use `IHL`, never
> a fixed `20`.

> ⚠️ **Common mistake: running tcpdump without the permission.** In a container without the
> `NET_RAW` capability (or without `sudo` in the VM) you get:
>
> ```text
> tcpdump: lo: You don't have permission to perform this capture on that device
> (Attempt to create packet socket failed - CAP_NET_RAW may be required)
> ```
>
> Start the container with `--cap-add NET_RAW --cap-add NET_ADMIN` as shown above.

## Try it yourself

1. Start the container again and repeat the capture from section 4, but send `hello, network\n`
   instead of `hello\n`.
2. Find the data packet. Its summary line now ends with `length 15`.
3. Read bytes 2 and 3 of the hex dump (the total length). They must be `00 43`: 0x43 = 67 = 20 (IP)
   \+ 32 (TCP) + 15 (data).
4. Change `PACKET` in `ipv4_header.rs` to the bytes of your new packet (the array length becomes
   67) and check that the program prints `total length: 67 bytes`.

## Exercises

### Exercise 1 — Which layer? (easy)

For each item, name the layer (link, internet, transport, application) that uses it:
(a) the port `443`; (b) the path `/login`; (c) the address `3a:71:10:41:e3:23`;
(d) the address `104.20.23.154`; (e) the TTL; (f) the `Host:` header of an HTTP request.

<details>
<summary>Hint</summary>

Ask "who needs this to do its job?" A router needs to know the computer; the kernel needs to know
the program; only the web server needs to know the page.

</details>

<details>
<summary>Answer</summary>

(a) transport; (b) application; (c) link (a MAC address); (d) internet (an IP address);
(e) internet (it is a field of the IP header); (f) application (it is part of the HTTP message).

</details>

### Exercise 2 — Ports and data from the TCP header (medium)

Write `src/bin/tcp_ports.rs` that uses the same `PACKET` and prints the source port, the
destination port and the data. The TCP header starts right after the IP header. Its first two bytes
are the source port, the next two the destination port, and the upper four bits of byte 12 are the
TCP header length in 32-bit words. Expected output:

```text
source port:      36932
destination port: 9000
TCP header:       32 bytes
data:             "hello\n"
```

<details>
<summary>Hint</summary>

Make a slice `let tcp = &PACKET[ip_header_len..];` and index into it. The header length is
`(tcp[12] >> 4) as usize * 4`. `String::from_utf8_lossy` turns bytes into text.

</details>

<details>
<summary>Answer</summary>

```rust
const PACKET: [u8; 58] = [
    0x45, 0x00, 0x00, 0x3a, 0x93, 0x32, 0x40, 0x00, 0x40, 0x06, 0xa9, 0x89, 0x7f, 0x00, 0x00, 0x01,
    0x7f, 0x00, 0x00, 0x01, 0x90, 0x44, 0x23, 0x28, 0xf1, 0xe7, 0xd9, 0x0f, 0xa4, 0xe3, 0x82, 0x84,
    0x80, 0x18, 0x00, 0x40, 0xfe, 0x2e, 0x00, 0x00, 0x01, 0x01, 0x08, 0x0a, 0xa5, 0x9d, 0xe3, 0x4b,
    0x5a, 0xcc, 0xd2, 0xa2, 0x68, 0x65, 0x6c, 0x6c, 0x6f, 0x0a,
];

fn main() {
    let ip_header_len = (PACKET[0] & 0x0f) as usize * 4;
    let tcp = &PACKET[ip_header_len..];
    let src_port = u16::from_be_bytes([tcp[0], tcp[1]]);
    let dst_port = u16::from_be_bytes([tcp[2], tcp[3]]);
    let tcp_header_len = (tcp[12] >> 4) as usize * 4;
    let data = &tcp[tcp_header_len..];
    println!("source port:      {src_port}");
    println!("destination port: {dst_port}");
    println!("TCP header:       {tcp_header_len} bytes");
    println!("data:             {:?}", String::from_utf8_lossy(data));
}
```

Why it works: encapsulation means each header simply follows the previous one. Once you know the
length of the IP header, the TCP header starts there; once you know the TCP header length, the data
starts there. The TCP header is 32 bytes here because it carries 12 bytes of options (the time
stamps `TS val ... ecr ...` in the tcpdump line).

</details>

### Exercise 3 — A tested IPv4 parser (medium)

Write `src/bin/parse_ipv4.rs` with a struct `Ipv4Header` (header length, total length, TTL,
protocol, source, destination) and a function
`fn parse_ipv4(bytes: &[u8]) -> Result<Ipv4Header, &'static str>`. It must reject input shorter
than 20 bytes, a version other than 4, and a header length below 20 or longer than the input.
Write a test with the real header bytes, and tests for at least two of the errors. Run them with
`cargo test --bin parse_ipv4`.

<details>
<summary>Hint</summary>

Check the length **before** you index, otherwise a short input panics. To build a broken packet in
a test, copy the good array (`let mut bytes = HEADER;`) and change one byte.

</details>

<details>
<summary>Answer</summary>

```rust
use std::net::Ipv4Addr;

#[derive(Debug, PartialEq)]
struct Ipv4Header {
    header_len: usize,
    total_len: u16,
    ttl: u8,
    protocol: u8,
    src: Ipv4Addr,
    dst: Ipv4Addr,
}

fn parse_ipv4(bytes: &[u8]) -> Result<Ipv4Header, &'static str> {
    if bytes.len() < 20 {
        return Err("too short for an IPv4 header");
    }
    if bytes[0] >> 4 != 4 {
        return Err("not IPv4");
    }
    let header_len = (bytes[0] & 0x0f) as usize * 4;
    if header_len < 20 || bytes.len() < header_len {
        return Err("bad header length");
    }
    Ok(Ipv4Header {
        header_len,
        total_len: u16::from_be_bytes([bytes[2], bytes[3]]),
        ttl: bytes[8],
        protocol: bytes[9],
        src: Ipv4Addr::new(bytes[12], bytes[13], bytes[14], bytes[15]),
        dst: Ipv4Addr::new(bytes[16], bytes[17], bytes[18], bytes[19]),
    })
}

fn main() {
    let header = [
        0x45, 0x00, 0x00, 0x3a, 0x93, 0x32, 0x40, 0x00, 0x40, 0x06, 0xa9, 0x89, 0x7f, 0x00, 0x00,
        0x01, 0x7f, 0x00, 0x00, 0x01,
    ];
    println!("{:?}", parse_ipv4(&header));
    println!("{:?}", parse_ipv4(&header[..10]));
}

#[cfg(test)]
mod tests {
    use super::*;

    const HEADER: [u8; 20] = [
        0x45, 0x00, 0x00, 0x3a, 0x93, 0x32, 0x40, 0x00, 0x40, 0x06, 0xa9, 0x89, 0x7f, 0x00, 0x00,
        0x01, 0x7f, 0x00, 0x00, 0x01,
    ];

    #[test]
    fn parses_a_real_header() {
        let h = parse_ipv4(&HEADER).unwrap();
        assert_eq!(h.header_len, 20);
        assert_eq!(h.total_len, 58);
        assert_eq!(h.ttl, 64);
        assert_eq!(h.protocol, 6);
        assert_eq!(h.src, Ipv4Addr::LOCALHOST);
    }

    #[test]
    fn rejects_short_input() {
        assert_eq!(
            parse_ipv4(&HEADER[..19]),
            Err("too short for an IPv4 header")
        );
    }

    #[test]
    fn rejects_a_header_length_below_20() {
        let mut bytes = HEADER;
        bytes[0] = 0x44; // version 4, header length 4 * 4 = 16 bytes
        assert_eq!(parse_ipv4(&bytes), Err("bad header length"));
    }
}
```

`cargo run --bin parse_ipv4` prints:

```text
Ok(Ipv4Header { header_len: 20, total_len: 58, ttl: 64, protocol: 6, src: 127.0.0.1, dst: 127.0.0.1 })
Err("too short for an IPv4 header")
```

and `cargo test --bin parse_ipv4` ends with
`test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out`.

Why it works: every index is checked against the length before it is used, so no input can make the
parser panic. Bytes from the network are untrusted input (Lesson 10.4): a parser must return an
error for any garbage, never crash.

</details>

## Quiz

1. Why is data cut into packets instead of being sent as one long piece?

<details>
<summary>Answer</summary>

So that many conversations can share the same cables fairly, and so that a damaged or lost piece
costs only a small resend. Each packet is routed on its own.

</details>

2. A router forwards your packet. Which headers does it read, and which does it ignore?

<details>
<summary>Answer</summary>

It reads the link header (to receive the frame) and the IP header (to choose the next hop and lower
the TTL). It ignores the transport and application data.

</details>

3. In a tcpdump line you see `10.0.0.5.51000 > 10.0.0.9.443`. What are `51000` and `443`?

<details>
<summary>Answer</summary>

Ports. `51000` is the client's port (chosen by its operating system), `443` is the server's port
(the usual port for HTTPS).

</details>

4. What does "L4" mean, and what does "L7" mean?

<details>
<summary>Answer</summary>

Layer numbers from the OSI model: L4 is the transport layer (TCP, UDP, ports), L7 is the
application layer (HTTP, DNS).

</details>

5. The bytes `00 3a` are a 16-bit length field. What number is it, and why?

<details>
<summary>Answer</summary>

58. Network headers use big-endian (network byte order): the first byte is the most significant, so
the value is 0x003a.

</details>

## Summary

- Networks move data in small, independent **packets**; **routers** forward each one a step closer
  to its destination.
- The internet has four layers: **link** (next hop, MAC addresses), **internet** (which computer, IP
  addresses), **transport** (which program, ports; TCP or UDP), **application** (what the bytes mean).
- **Encapsulation**: each layer wraps the layer above in its own header, and the receiver unwraps in
  reverse order.
- Headers are small, but they cost bytes on every packet; the **MTU** (1500 bytes on Ethernet) limits
  the packet size.
- `tcpdump -i <interface> -n -X` shows real packets; run it in a container with
  `--cap-add NET_RAW --cap-add NET_ADMIN`.
- Header fields are **big-endian**; parse them with `from_be_bytes`, check every length before
  indexing, and use captured packets as test data.

## Sources and further reading

- [RFC 791 — Internet Protocol](https://datatracker.ietf.org/doc/html/rfc791) — the definition of IPv4, including the header layout in section 3.1.
- [RFC 1122 — Requirements for Internet Hosts](https://datatracker.ietf.org/doc/html/rfc1122) — section 1.1.3 describes the internet's layers.
- [Computer Networks: A Systems Approach](https://book.systemsapproach.org/) — a free online textbook; chapter 1 explains layering in more depth.
- [tcpdump manual page](https://www.tcpdump.org/manpages/tcpdump.1.html) — every option and the filter syntax.
- [`std::net::Ipv4Addr`](https://doc.rust-lang.org/std/net/struct.Ipv4Addr.html) — the standard library type for IPv4 addresses.
- [Docker — runtime privilege and Linux capabilities](https://docs.docker.com/engine/containers/run/) — what `--cap-add` does.

---

[🏠 Start](../START-HERE.md) · [← Phase 15 — Networking](README.md) · [📋 Phase 15](README.md) · [15.2 IP addresses, subnets and routing →](02-ip-addresses-and-routing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
