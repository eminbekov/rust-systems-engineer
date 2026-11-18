<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.12 TLS in Rust servers with rustls](12-tls-in-rust-servers.md) · [📋 Phase 15](README.md) · [15.14 Proxies and load balancers →](14-proxies-and-load-balancers.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/13-quic-and-http3.md) <!-- nav -->

# Lesson 15.13 — QUIC and HTTP/3

> **Phase 15 — Networking** · Lesson 13 of 16 · ⏱ about 135 minutes

## What you will learn

- Which problems of TCP plus TLS led to QUIC: head-of-line blocking, slow handshakes and a protocol that is hard to change.
- How QUIC works: UDP datagrams, built-in TLS 1.3, independent streams and connection IDs.
- How to open a QUIC connection and many streams with `quinn`, and what QUIC looks like in a packet capture.
- How HTTP/3 maps HTTP onto QUIC, how clients find out that a server speaks it, and how to make an HTTP/3 request with `h3`.

## Before you start

- You finished [Lesson 15.12 — TLS in Rust servers with rustls](12-tls-in-rust-servers.md): certificates, `rcgen`, ALPN and the `ring` crypto provider.
- You know TCP (Transmission Control Protocol) and UDP (User Datagram Protocol) from [Lesson 15.3 — TCP and UDP](03-tcp-and-udp.md), and HTTP (HyperText Transfer Protocol) versions from [Lesson 15.9 — HTTP with hyper and reqwest](09-http-with-hyper-and-reqwest.md).
- Create the practice package:

```bash
cd ~/projects/rust
cargo new code/p15-quic
cd code/p15-quic
mkdir -p src/bin tests && rm src/main.rs
cargo add tokio@1 --features full
cargo add quinn@0.11 rcgen@0.14
cargo add rustls@0.23 --no-default-features --features ring,std
cargo add h3@0.0.8 h3-quinn@0.0.10 http@1 bytes@1 webpki-roots@1
```

`quinn` uses `rustls` with the `ring` provider by default, so the whole program has exactly one crypto provider, as Lesson 15.12 recommends.

## New words

| Word | Meaning |
|---|---|
| **QUIC** | A transport protocol that runs over UDP and includes encryption and streams. The name is not an abbreviation any more. Defined in RFC 9000 (RFC: Request for Comments, the documents that define internet protocols). |
| **HTTP/3** | HTTP carried over QUIC (RFC 9114). Same methods, headers and status codes as HTTP/1.1 and HTTP/2; a different transport. |
| **TLS 1.3** (Transport Layer Security) | The encryption protocol of Lesson 10.11. QUIC uses its handshake inside its own packets. |
| **head-of-line blocking** | When one lost or slow piece holds back everything behind it in a queue, even pieces that belong to other requests. |
| **stream** (QUIC) | An ordered byte stream inside a QUIC connection. A connection can have thousands; a loss in one does not stop the others. |
| **bidirectional / unidirectional stream** | A stream on which both sides can send / on which only the opener sends. |
| **connection ID** | A number in every QUIC packet that identifies the connection, so it survives a change of IP (Internet Protocol) address or port. |
| **connection migration** | Keeping a QUIC connection alive when a client moves, for example from Wi-Fi to mobile data. |
| **RTT** (Round-Trip Time) | The time a packet needs to go to the other side and an answer to come back. |
| **0-RTT** (zero round-trip time) | Sending request data in the very first packet to a server you talked to before. Fast, but the data can be replayed by an attacker. |
| **endpoint** (`quinn`) | One UDP socket that can be a client, a server or both, with all its QUIC connections. |
| **Alt-Svc** (Alternative Services) | An HTTP response header that says "I am also available over another protocol", for example `h3=":443"`. |
| **QPACK** | The header compression of HTTP/3 (RFC 9204), designed so that streams do not block each other. |
| **datagram** (QUIC) | An unreliable message sent inside an encrypted QUIC connection: it may be lost and is never resent (RFC 9221). |
| **SYN, ACK, API** | The TCP handshake flags "synchronise" and "acknowledge" (Lesson 15.3); Application Programming Interface, the types and functions a library offers to your code. |

## 1. Why a new transport

HTTP/2 (Lesson 15.9) sends many requests over **one** TCP connection. That is efficient, but TCP delivers bytes strictly in order. If one packet is lost, TCP holds back every byte behind it until the lost packet is resent, even bytes that belong to completely different requests:

```text
 HTTP/2 over TCP: one ordered byte stream
   packets:   [A1][B1][C1][A2][B2]...        B1 is lost
   delivered: A1 ── wait ── wait ── wait ── (B1 resent) B1 C1 A2 B2
              requests A and C wait for B's lost packet: head-of-line blocking

 HTTP/3 over QUIC: independent streams inside UDP datagrams
   stream A:  A1 A2 ...       delivered at once
   stream B:  B1 (lost) ... B1 resent ... B2      only B waits
   stream C:  C1 ...          delivered at once
```

There are two more problems:

1. **Handshake time.** Before the first HTTP byte, TCP needs one round trip and TLS 1.3 needs another: two RTTs. On a phone network with an RTT of 100 ms, that is 0.2 seconds of waiting before anything happens. QUIC does the transport and the TLS handshake **together**, in one RTT, and can even send data in the first packet when it resumes an earlier session (0-RTT).
2. **A protocol that cannot change.** TCP lives in the operating system kernel, and many routers and firewalls on the path look into TCP headers and drop what they do not recognise. Improving TCP takes many years. QUIC runs over plain UDP, lives in a library inside the program (like `quinn`), and encrypts almost all of its headers, so the middle of the network cannot depend on them.

Why UDP and not a new IP protocol? Because almost every network on the planet already passes UDP (DNS, the Domain Name System, uses it). A brand-new protocol number would be blocked in many places.

## 2. QUIC in one picture

```text
 UDP datagram (to port 443)
 └─ QUIC packet: header [connection ID, packet number]  + encrypted payload
                                                           └─ frames:
                                                              STREAM 0: "GET /a ..."
                                                              STREAM 4: "GET /b ..."
                                                              ACK: packets 1-7 received
                                                              CRYPTO: TLS handshake bytes
```

| Feature | TCP + TLS 1.3 | QUIC |
|---|---|---|
| Runs on | IP, in the kernel | UDP, in a library |
| Handshake before data | 2 RTT (1 for TCP, 1 for TLS) | 1 RTT (0-RTT when resuming) |
| Encryption | only the payload; TCP headers are visible | payload and most of the header |
| Ordering | one ordered byte stream | many independent streams |
| Loss of one packet | blocks everything behind it | blocks only the streams in that packet |
| Address change (Wi-Fi → mobile) | connection breaks | continues, thanks to the connection ID |
| Messages without retransmission | not possible | QUIC datagrams |

QUIC still gives each stream what TCP gives a connection: retransmission of lost data, ordering inside the stream, flow control and congestion control. It just does it **per stream** inside one connection. TLS is not optional: every QUIC connection is encrypted with TLS 1.3, and ALPN (Application-Layer Protocol Negotiation, Lesson 15.12) is required, so both sides always agree on what they speak.

## 3. QUIC in Rust with quinn

`quinn` is a QUIC implementation for Tokio. An **endpoint** is one UDP socket; on it you accept or open connections, and on a connection you open streams. To keep the example in one file, the program runs the server and the client in the same process. Create `src/bin/quic_echo.rs`:

```rust
use std::net::SocketAddr;
use std::sync::Arc;
use std::time::{Duration, Instant};

use quinn::{ClientConfig, Endpoint, ServerConfig};
use rustls::RootCertStore;
use rustls::pki_types::{CertificateDer, PrivatePkcs8KeyDer};

/// The server: echoes every stream back in upper case. "slow" streams wait 500 ms first.
async fn run_server(endpoint: Endpoint) {
    while let Some(incoming) = endpoint.accept().await {
        tokio::spawn(async move {
            let conn = incoming.await.expect("handshake");
            println!("server: connection from {}", conn.remote_address());
            // Every bidirectional stream the client opens arrives here.
            while let Ok((mut send, mut recv)) = conn.accept_bi().await {
                tokio::spawn(async move {
                    let msg = recv.read_to_end(1024).await.expect("read");
                    if msg.starts_with(b"slow") {
                        tokio::time::sleep(Duration::from_millis(500)).await;
                    }
                    send.write_all(&msg.to_ascii_uppercase())
                        .await
                        .expect("write");
                    send.finish().expect("finish");
                });
            }
        });
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. A self-signed certificate for "localhost", in memory.
    let cert = rcgen::generate_simple_self_signed(vec!["localhost".to_string()])?;
    let cert_der = CertificateDer::from(cert.cert);
    let key_der = PrivatePkcs8KeyDer::from(cert.signing_key.serialize_der());

    // 2. The server endpoint: one UDP socket.
    let server_config = ServerConfig::with_single_cert(vec![cert_der.clone()], key_der.into())?;
    let server = Endpoint::server(server_config, "127.0.0.1:0".parse()?)?;
    let server_addr: SocketAddr = server.local_addr()?;
    println!("server: listening on UDP {server_addr}");
    tokio::spawn(run_server(server));

    // 3. The client endpoint trusts exactly this one certificate.
    let mut roots = RootCertStore::empty();
    roots.add(cert_der)?;
    let mut client = Endpoint::client("127.0.0.1:0".parse()?)?;
    client.set_default_client_config(ClientConfig::with_root_certificates(Arc::new(roots))?);

    let start = Instant::now();
    let conn = client.connect(server_addr, "localhost")?.await?;
    println!("client: connected after {:?}", start.elapsed());

    // 4. Three streams on ONE connection, all at the same time.
    let mut tasks = Vec::new();
    for msg in ["slow stream one", "stream two", "stream three"] {
        let conn = conn.clone();
        tasks.push(tokio::spawn(async move {
            let (mut send, mut recv) = conn.open_bi().await?;
            send.write_all(msg.as_bytes()).await?;
            send.finish()?;
            let reply = recv.read_to_end(1024).await?;
            println!(
                "client: {:>7.1?} {:?}",
                start.elapsed(),
                String::from_utf8_lossy(&reply)
            );
            Ok::<_, Box<dyn std::error::Error + Send + Sync>>(())
        }));
    }
    for task in tasks {
        task.await?.map_err(|e| e.to_string())?;
    }

    conn.close(0u32.into(), b"done");
    client.wait_idle().await;
    Ok(())
}
```

**What happens:**

1. `rcgen::generate_simple_self_signed` makes a certificate and a key in memory (Lesson 15.12 made a whole CA, certificate authority; for one local test a self-signed certificate is enough, because the client trusts exactly this certificate).
2. `ServerConfig::with_single_cert` builds the TLS part of the server for QUIC. `Endpoint::server(config, addr)` binds **one UDP socket**; there is no `listen`/`accept` per TCP connection, because UDP has no connections. QUIC builds them itself.
3. `endpoint.accept()` yields an `Incoming` connection attempt; `.await` on it runs the QUIC + TLS handshake. Then `conn.accept_bi()` yields each **stream** the client opens: a pair of a send half and a receive half.
4. `read_to_end(1024)` reads until the other side has **finished** its half of the stream, with a size limit (never read unlimited data from the network). `send.finish()` says "I have nothing more to send on this stream", like closing the write half of a TCP connection.
5. The client endpoint gets a `ClientConfig` that trusts exactly our certificate. `client.connect(addr, "localhost")` starts the handshake (the name is checked against the certificate, as in Lesson 15.12) and `.await` waits for it.
6. Three tasks open three streams on the **same** connection (`conn.clone()` is a cheap handle to it). The server makes the first one wait 500 ms.
7. `conn.close(code, reason)` closes the connection with an application error code (0 = all fine), and `wait_idle()` gives the close message time to reach the other side.

```bash
cargo run -q --bin quic_echo
```

```text
server: listening on UDP 127.0.0.1:56651
client: connected after 4.985083ms
server: connection from 127.0.0.1:52522
client:   5.7ms "STREAM TWO"
client:   5.8ms "STREAM THREE"
client: 507.2ms "SLOW STREAM ONE"
```

The two fast streams finished at once, although the slow stream was opened first and shares the same connection. On one HTTP/1.1 connection, the answers would come back strictly in order, and the fast requests would wait for the slow one.

> ⚡ **Performance:** opening a QUIC stream costs no round trip: the stream simply starts with its first frame. A connection, in contrast, costs a handshake. So open one connection per server and many streams on it, just as you reuse a `reqwest::Client` for HTTP.

## 4. What QUIC looks like on the wire

Let us capture the start of a real QUIC connection. The program in Section 5 talks HTTP/3 to a public server; in a capture it looks like this (taken in a Linux container with `tcpdump -i eth0 -n -c 8 udp port 443` while the program ran; see "Try it yourself"):

```text
07:48:59.544410 IP 172.17.0.3.57268 > 104.18.26.14.443: UDP, length 1200
07:48:59.636023 IP 104.18.26.14.443 > 172.17.0.3.57268: UDP, length 1200
07:48:59.636044 IP 104.18.26.14.443 > 172.17.0.3.57268: UDP, length 1200
07:48:59.636045 IP 104.18.26.14.443 > 172.17.0.3.57268: UDP, length 1200
07:48:59.636046 IP 104.18.26.14.443 > 172.17.0.3.57268: UDP, length 529
07:48:59.639888 IP 172.17.0.3.57268 > 104.18.26.14.443: UDP, length 1200
07:48:59.639957 IP 172.17.0.3.57268 > 104.18.26.14.443: UDP, length 1275
07:48:59.640077 IP 172.17.0.3.57268 > 104.18.26.14.443: UDP, length 66
```

**What happens:**

1. There is no SYN, SYN-ACK, ACK (the TCP handshake of Lesson 15.3): just UDP datagrams to port 443. To `tcpdump`, and to any router on the way, it is "UDP, length 1200" and nothing more.
2. The client's first datagram is exactly 1200 bytes. QUIC requires the first packet to be padded to at least 1200 bytes, so that an attacker with a forged source address cannot use a tiny packet to make a server send a big answer to a victim.
3. The server answers with its certificate and handshake data (about 4 KiB, kibibytes, in four datagrams) about 90 ms later: one RTT to this server. Only 4 ms after that, the client already sends its request; the HTTP request travelled in the client's second flight, with no extra round trip.

With `-X` you can see the first bytes of the QUIC packet, right after the 8-byte UDP header:

```text
	0x0010:  6812 1a0e b8c6 01bb 04b8 32ff c100 0000  h.........2.....
	0x0020:  0114 b03f cb18 29a6 0e8f 8957 269e f74c  ...?..)....W&..L
```

`c1` is the first byte of a QUIC **long header** (top bit 1: a handshake packet; the next bit is always 1). `00 00 00 01` is the QUIC version: 1, from RFC 9000. `14` = 20 is the length of the destination connection ID, and the 20 bytes that follow are that ID. Everything after the header is encrypted, even in this very first packet.

> 🔒 **Security:** QUIC encrypts more than TCP + TLS: the packet numbers and the acknowledgements are protected too, so the network cannot see or change them. The price is that tools such as firewalls and load balancers see less. Some companies block UDP port 443 for this reason; clients then fall back to HTTP/2 over TCP.

## 5. HTTP/3 with h3

HTTP/3 keeps the meaning of HTTP and changes the transport:

- every request/response pair uses its **own bidirectional QUIC stream**, so a slow or lost response does not block others;
- headers are compressed with QPACK and sent in binary frames (HEADERS, DATA);
- each side opens a few unidirectional *control* streams for settings.

A client does not know in advance that a server speaks HTTP/3. Usually it first connects with HTTP/2 over TCP, and the server advertises HTTP/3 with an **Alt-Svc** header:

```bash
curl -sI https://cloudflare-quic.com/ | grep -i -E '^(HTTP|alt-svc)'
```

```text
HTTP/2 200 
alt-svc: h3=":443"; ma=86400
```

**What happens:** `h3=":443"` means "HTTP/3 is available on UDP port 443 of the same host"; `ma=86400` ("max age") says the client may remember this for a day. The next request from a browser goes over QUIC. (The macOS `curl` cannot speak HTTP/3 itself, so it stays on HTTP/2.)

The Rust crate `h3` implements HTTP/3 on top of any QUIC library; `h3-quinn` connects it to `quinn`. It is still young (version 0.0.x, so its API changes between versions), but it works. Create `src/bin/h3_get.rs`:

```rust
use std::sync::Arc;

use bytes::Buf;
use quinn::crypto::rustls::QuicClientConfig;
use rustls::RootCertStore;
use tokio::net::lookup_host;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let host = "cloudflare-quic.com";

    // 1. TLS for QUIC: Mozilla's roots, and the ALPN name of HTTP/3, "h3".
    let roots = RootCertStore::from_iter(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());
    let mut tls = rustls::ClientConfig::builder()
        .with_root_certificates(roots)
        .with_no_client_auth();
    tls.alpn_protocols = vec![b"h3".to_vec()];
    let quic_tls = QuicClientConfig::try_from(tls)?;

    // 2. A QUIC connection over UDP port 443.
    let addr = lookup_host((host, 443))
        .await?
        .find(|a| a.is_ipv4())
        .ok_or("no IPv4 address")?;
    let mut endpoint = quinn::Endpoint::client("0.0.0.0:0".parse()?)?;
    endpoint.set_default_client_config(quinn::ClientConfig::new(Arc::new(quic_tls)));
    let conn = endpoint.connect(addr, host)?.await?;
    println!("QUIC connection to {addr}, RTT about {:?}", conn.rtt());

    // 3. HTTP/3 on top of the QUIC connection.
    let (mut driver, mut send_request) = h3::client::new(h3_quinn::Connection::new(conn)).await?;
    let drive = tokio::spawn(async move { driver.wait_idle().await });

    let request = http::Request::get(format!("https://{host}/")).body(())?;
    let mut stream = send_request.send_request(request).await?;
    stream.finish().await?; // no request body
    let response = stream.recv_response().await?;
    println!("{:?} {}", response.version(), response.status());
    println!("server: {:?}", response.headers().get("server"));

    let mut size = 0;
    while let Some(chunk) = stream.recv_data().await? {
        size += chunk.remaining();
    }
    println!("body: {size} bytes");

    drop(send_request);
    drive.abort();
    endpoint.wait_idle().await;
    Ok(())
}
```

**What happens:**

1. We build the `rustls` client configuration ourselves, because HTTP/3 **requires** the ALPN value `h3`. `QuicClientConfig::try_from` checks that the configuration is usable for QUIC (TLS 1.3 only) and wraps it.
2. `lookup_host` (Lesson 15.10) resolves the name; we pick an IPv4 address. `endpoint.connect(addr, host)` uses `host` for the certificate check and SNI (Server Name Indication, Lesson 15.12).
3. `h3::client::new` sets up HTTP/3 on the connection and returns two things: a **driver** that must keep running to handle the control streams (we run it in its own task), and `send_request`, a handle for making requests.
4. `send_request(request)` opens a new bidirectional stream and sends the HEADERS frame. `finish()` says "no request body". `recv_response()` waits for the response headers, and `recv_data()` returns the body in chunks until `None`.
5. At the end we drop the request handle, stop the driver and let the endpoint finish closing.

```bash
cargo run -q --bin h3_get
```

```text
QUIC connection to 104.18.26.14:443, RTT about 92.449917ms
HTTP/3.0 200 OK
server: Some("cloudflare")
body: 125959 bytes
```

The response says `HTTP/3.0`: the same `http::Response` type you know from `hyper` and `reqwest`, carried over QUIC. (The address, RTT and size depend on your network and on the day.)

> 📏 **Convention:** most services do not terminate QUIC themselves. A reverse proxy or a load balancer at the edge (Lesson 15.14) speaks HTTP/3 to the internet and HTTP/1.1 or HTTP/2 to the services behind it. Use QUIC directly in your program when you need its features: many independent streams, connection migration or datagrams.

## Common mistakes

**1. No ALPN.** Remove the line `tls.alpn_protocols = vec![b"h3".to_vec()];` and the handshake fails:

```text
Error: ConnectionClosed(ConnectionClose { error_code: Code::crypto(78), frame_type: None, reason: b"" })
```

`Code::crypto(78)` is a TLS alert inside QUIC: hexadecimal `78` is 120, the alert *no_application_protocol*. QUIC servers refuse clients that do not say which protocol they want to speak.

**2. Forgetting `finish()`.** If the client never calls `send.finish()`, the server's `read_to_end` keeps waiting for more data, and the client waits for the reply: both wait forever. The program prints the connection lines and then hangs (stopped here after 3 seconds with `timeout 3`, exit code 124). A stream ends only when its sender finishes it.

**3. Assuming UDP is always allowed.** Some networks block or throttle UDP. Real clients race QUIC against TCP or fall back to HTTP/2 when QUIC does not answer quickly. Never make a public service available **only** over HTTP/3.

**4. Treating QUIC datagrams like streams.** Datagrams (Exercise 3) are encrypted but **unreliable**: they can be lost or arrive out of order, and they must fit into one packet. Use them for data where a newer value replaces an older one (positions in a game, live measurements), and streams for everything that must arrive.

## Try it yourself

Capture QUIC yourself in a Linux container (the capture needs Linux and root; never on the Mac itself). From `code/p15-quic`:

```bash
docker run --rm -it --cap-add NET_RAW --cap-add NET_ADMIN -v "$PWD":/work -w /work rust:1.100-slim bash
```

Inside the container (`CARGO_TARGET_DIR` keeps the Linux build out of your Mac's `target/` folder):

```bash
apt-get update -qq && apt-get install -y -qq --no-install-recommends tcpdump > /dev/null
export CARGO_TARGET_DIR=/tmp/target
cargo build -q --bin h3_get
tcpdump -i eth0 -n -c 8 udp port 443 &
sleep 1
/tmp/target/debug/h3_get
```

You see the program's output and eight UDP lines like those in Section 4. Run it again with `tcpdump … -X -c 1` and find `c1`, the version `00000001` and the connection ID length in the first packet. Type `exit` to leave and delete the container.

## Exercises

### Exercise 1 — Two slow streams (easy)

Change the messages in `quic_echo` to `["slow stream one", "slow stream two", "stream three"]`. Predict the order and the times of the three replies, then run it.

<details>
<summary>Hint</summary>

The server handles each stream in its own task, and each slow stream waits 500 ms on its own.

</details>

<details>
<summary>Answer</summary>

```text
server: listening on UDP 127.0.0.1:57831
client: connected after 2.950584ms
server: connection from 127.0.0.1:64932
client:   3.6ms "STREAM THREE"
client: 505.2ms "SLOW STREAM ONE"
client: 505.2ms "SLOW STREAM TWO"
```

Why it works: the two slow streams wait **at the same time**, not one after the other, so both finish after about 500 ms, not 1000 ms. The fast stream is not delayed by either. Streams are independent both on the network and in the server.

</details>

### Exercise 2 — Test the streams (medium)

Write `tests/quic.rs` with two tests. A helper starts the echo server (without the slow path) on `127.0.0.1:0` and returns its address and certificate. Test 1: five streams on one connection, each gets exactly its own message back in upper case. Test 2: a client that trusts the certificate of a *different* server cannot connect.

<details>
<summary>Hint</summary>

Make the server loop over `accept()` so it can serve several connections. For test 2, start two servers, and connect to the first while trusting the second one's certificate. The error message contains `invalid peer certificate`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::net::SocketAddr;
use std::sync::Arc;

use quinn::{ClientConfig, Endpoint, ServerConfig};
use rustls::RootCertStore;
use rustls::pki_types::{CertificateDer, PrivatePkcs8KeyDer};

/// Starts an upper-case echo server; returns its address and its certificate.
fn start_echo_server() -> (SocketAddr, CertificateDer<'static>) {
    let cert = rcgen::generate_simple_self_signed(vec!["localhost".to_string()]).unwrap();
    let cert_der = CertificateDer::from(cert.cert);
    let key_der = PrivatePkcs8KeyDer::from(cert.signing_key.serialize_der());
    let config = ServerConfig::with_single_cert(vec![cert_der.clone()], key_der.into()).unwrap();
    let server = Endpoint::server(config, "127.0.0.1:0".parse().unwrap()).unwrap();
    let addr = server.local_addr().unwrap();
    tokio::spawn(async move {
        while let Some(incoming) = server.accept().await {
            let Ok(conn) = incoming.await else { continue };
            while let Ok((mut send, mut recv)) = conn.accept_bi().await {
                tokio::spawn(async move {
                    let msg = recv.read_to_end(1024).await.unwrap();
                    send.write_all(&msg.to_ascii_uppercase()).await.unwrap();
                    send.finish().unwrap();
                });
            }
        }
    });
    (addr, cert_der)
}

/// A client endpoint that trusts exactly `cert`.
fn client_trusting(cert: CertificateDer<'static>) -> Endpoint {
    let mut roots = RootCertStore::empty();
    roots.add(cert).unwrap();
    let mut client = Endpoint::client("127.0.0.1:0".parse().unwrap()).unwrap();
    client
        .set_default_client_config(ClientConfig::with_root_certificates(Arc::new(roots)).unwrap());
    client
}

#[tokio::test]
async fn every_stream_gets_its_own_answer() {
    let (addr, cert) = start_echo_server();
    let conn = client_trusting(cert)
        .connect(addr, "localhost")
        .unwrap()
        .await
        .unwrap();
    let mut tasks = Vec::new();
    for i in 0..5 {
        let conn = conn.clone();
        tasks.push(tokio::spawn(async move {
            let (mut send, mut recv) = conn.open_bi().await.unwrap();
            send.write_all(format!("message {i}").as_bytes())
                .await
                .unwrap();
            send.finish().unwrap();
            let reply = recv.read_to_end(1024).await.unwrap();
            assert_eq!(reply, format!("MESSAGE {i}").into_bytes());
        }));
    }
    for task in tasks {
        task.await.unwrap();
    }
}

#[tokio::test]
async fn a_client_that_trusts_another_certificate_is_refused() {
    let (addr, _cert) = start_echo_server();
    let (_, other_cert) = start_echo_server();
    let result = client_trusting(other_cert)
        .connect(addr, "localhost")
        .unwrap()
        .await;
    let err = result.unwrap_err();
    assert!(
        err.to_string().contains("invalid peer certificate"),
        "{err}"
    );
}
```

```bash
cargo test --test quic
```

```text
running 2 tests
test a_client_that_trusts_another_certificate_is_refused ... ok
test every_stream_gets_its_own_answer ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Why it works: each stream task checks its own reply, so a mix-up between streams (a reply arriving on the wrong stream) would fail the test. The negative test proves that QUIC really verifies the certificate: encryption without verification would accept the wrong server.

</details>

### Exercise 3 — Unreliable messages in a reliable connection (hard)

Write `src/bin/quic_datagrams.rs`: a server that accepts one connection and prints three QUIC datagrams it receives, and a client that prints the largest datagram it may send and then sends `position update 1` to `3` as datagrams.

<details>
<summary>Hint</summary>

`conn.send_datagram(bytes)` sends one datagram (it takes `bytes::Bytes`; a `String` converts with `.into()`), and `conn.read_datagram().await` receives one. `conn.max_datagram_size()` says how large a datagram may be on this connection (`None` if the peer does not support them).

</details>

<details>
<summary>Answer</summary>

```rust
use std::sync::Arc;

use quinn::{ClientConfig, Endpoint, ServerConfig};
use rustls::RootCertStore;
use rustls::pki_types::{CertificateDer, PrivatePkcs8KeyDer};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cert = rcgen::generate_simple_self_signed(vec!["localhost".to_string()])?;
    let cert_der = CertificateDer::from(cert.cert);
    let key_der = PrivatePkcs8KeyDer::from(cert.signing_key.serialize_der());

    let server_config = ServerConfig::with_single_cert(vec![cert_der.clone()], key_der.into())?;
    let server = Endpoint::server(server_config, "127.0.0.1:0".parse()?)?;
    let server_addr = server.local_addr()?;
    let receiver = tokio::spawn(async move {
        let conn = server.accept().await.unwrap().await.unwrap();
        for _ in 0..3 {
            let datagram = conn.read_datagram().await.unwrap();
            println!(
                "server got datagram: {:?}",
                String::from_utf8_lossy(&datagram)
            );
        }
    });

    let mut roots = RootCertStore::empty();
    roots.add(cert_der)?;
    let mut client = Endpoint::client("127.0.0.1:0".parse()?)?;
    client.set_default_client_config(ClientConfig::with_root_certificates(Arc::new(roots))?);
    let conn = client.connect(server_addr, "localhost")?.await?;

    println!("largest datagram: {:?} bytes", conn.max_datagram_size());
    for i in 1..=3 {
        conn.send_datagram(format!("position update {i}").into())?;
    }
    receiver.await?;
    conn.close(0u32.into(), b"done");
    client.wait_idle().await;
    Ok(())
}
```

```text
largest datagram: Some(1162) bytes
server got datagram: "position update 1"
server got datagram: "position update 2"
server got datagram: "position update 3"
```

Why it works: datagrams travel inside the same encrypted connection as streams, but QUIC never resends them. A datagram must fit into one packet: 1162 bytes here, the packet size minus QUIC's headers. On `127.0.0.1` nothing gets lost; on a real network the server must not wait for exactly three, because some may never arrive.

</details>

## Quiz

1. What is head-of-line blocking in HTTP/2 over TCP, and how does QUIC avoid it?

<details>
<summary>Answer</summary>

TCP delivers bytes strictly in order, so one lost packet holds back all later bytes of the connection, including those of other HTTP/2 requests. QUIC has independent streams: a lost packet holds back only the streams whose data it carried.

</details>

2. Why is QUIC built on UDP instead of being a new protocol next to TCP?

<details>
<summary>Answer</summary>

Almost every network passes UDP, while a new IP protocol would be blocked by many routers and firewalls. UDP also lets QUIC live in a library instead of the kernel, so it can be improved quickly.

</details>

3. How many round trips does a client wait before it can send an HTTP request over TCP + TLS 1.3, and over QUIC?

<details>
<summary>Answer</summary>

Two over TCP + TLS 1.3 (one for TCP, one for TLS); one over QUIC, which does both handshakes together. When it resumes an earlier session, QUIC can even send data in the first packet (0-RTT).

</details>

4. What does `alt-svc: h3=":443"; ma=86400` tell a browser?

<details>
<summary>Answer</summary>

That the same site is also available over HTTP/3 on UDP port 443, and that the browser may remember this for 86 400 seconds (one day). The browser will try QUIC for the next requests.

</details>

5. What happens if a QUIC stream's sender never calls `finish()` and the receiver uses `read_to_end`?

<details>
<summary>Answer</summary>

The receiver keeps waiting for the end of the stream, which never comes. If the sender in turn waits for a reply, both wait forever.

</details>

6. Why does a QUIC client pad its first packet to 1200 bytes?

<details>
<summary>Answer</summary>

To prevent amplification attacks: an attacker with a forged source address could otherwise send a tiny packet and make the server send a much bigger answer to the victim. With padding, the server's first answer is not much larger than what it received.

</details>

## Summary

- TCP's single ordered stream causes head-of-line blocking for HTTP/2, and TCP + TLS needs two round trips before data; QUIC fixes both.
- QUIC runs over UDP, encrypts with TLS 1.3 (ALPN required), carries many independent streams, and identifies connections by connection IDs.
- In `quinn`, an `Endpoint` is one UDP socket; `connect`/`accept` give connections; `open_bi`/`accept_bi` give streams; `finish()` ends a stream.
- On the wire, QUIC is just UDP to port 443; the first packet is padded to 1200 bytes and almost everything is encrypted.
- HTTP/3 maps each request onto its own QUIC stream; servers advertise it with `Alt-Svc`; `h3` with `h3-quinn` is the Rust implementation.
- QUIC datagrams are encrypted but unreliable; use them only for data that may be lost.

## Sources and further reading

- [RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000) — the QUIC protocol itself; section 14.1 explains the 1200-byte rule.
- [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114) — how HTTP maps onto QUIC streams.
- [quinn documentation](https://docs.rs/quinn/latest/quinn/) — `Endpoint`, `Connection`, streams and datagrams.
- [h3 on GitHub](https://github.com/hyperium/h3) — the HTTP/3 crate, with client and server examples.
- [RFC 9221 — An Unreliable Datagram Extension to QUIC](https://www.rfc-editor.org/rfc/rfc9221) — the datagrams of Exercise 3.
- [High Performance Browser Networking](https://hpbn.co/) — a free book that explains latency, TCP and HTTP/2, the background for why QUIC exists.

---

[🏠 Start](../START-HERE.md) · [← 15.12 TLS in Rust servers with rustls](12-tls-in-rust-servers.md) · [📋 Phase 15](README.md) · [15.14 Proxies and load balancers →](14-proxies-and-load-balancers.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
