<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.6 Async networking with Tokio](06-async-networking-with-tokio.md) · [📋 Phase 15](README.md) · [15.8 HTTP from scratch →](08-http-from-scratch.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/07-designing-a-protocol.md) <!-- nav -->

# Lesson 15.7 — Designing a protocol: framing and parsing

> **Phase 15 — Networking** · Lesson 7 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The three ways to cut a byte stream into messages (delimiters, length prefixes, fixed sizes) and when to use each.
- How to write a frame decoder that handles the three possible outcomes: a complete frame, not enough bytes yet, and an invalid frame.
- How to design a small binary message format and parse it without ever panicking on bad input.
- How to use `bytes::BytesMut` and the `tokio_util::codec` framework (`Framed`, `Decoder`, `LengthDelimitedCodec`).
- How to test a protocol: round trips, bytes that arrive one at a time, and garbage.

## Before you start

- You finished [Lesson 15.6 — Async networking with Tokio](06-async-networking-with-tokio.md).
- You saw in [Lesson 15.4](04-tcp-sockets-in-rust.md) that TCP writes and reads do not match one to one.
- You know byte order from Lesson 15.1, enums and `match` from Phase 04, and input validation from [Lesson 10.4 — Input validation: never trust input](../10-security-fundamentals/04-input-validation.md).
- Add three crates to `code/p15-net` (versions from crates.io, checked 2026-11-17):

```bash
cargo add bytes@1
cargo add tokio-util@0.7 --features codec
cargo add futures@0.3
```

## New words

| Word | Meaning |
|---|---|
| **framing** | Deciding where one message ends and the next begins in a byte stream. |
| **frame** | One complete message as it travels on the wire, including its framing (for example its length prefix). Not the same as an Ethernet frame from Lesson 15.1. |
| **delimiter** | A special byte or byte sequence that marks the end of a message, for example `\n`. |
| **length prefix** | A number in front of a message that says how many bytes follow. |
| **payload** | The content of a frame, without the framing bytes. |
| **wire format** | The exact bytes a protocol sends: which field comes first, how big it is, which byte order. |
| **decoder / encoder** | Code that turns bytes into messages / messages into bytes. Together they are a **codec**. |
| **round trip** | Encoding a value and decoding it again. A correct codec gives back exactly the original value. |
| **`BytesMut` / `Bytes`** | Byte buffers from the `bytes` crate: `BytesMut` can grow and be cut into pieces cheaply; `Bytes` is a read-only, cheaply shareable piece. |
| **`Framed`** | A `tokio_util` type that wraps a socket and a codec and turns the byte stream into a *stream of messages* and a *sink* for messages. |
| **sink** | The opposite of a stream: something you can *send* values into, one after another. |
| **TCP, HTTP, TLS** (Transmission Control Protocol, HyperText Transfer Protocol, Transport Layer Security) | The byte-stream transport of Lesson 15.3, the web protocol (Lesson 15.8) and the encryption layer (Lesson 15.12). |
| **SMTP** (Simple Mail Transfer Protocol) | The text protocol that mail servers use to pass e-mail to each other. |
| **KV** (key-value) | The name of our example protocol: it stores values under keys, like a `HashMap` on a server. |
| **UTF-8, ASCII, GB** | The text encoding of Rust strings (Unicode Transformation Format, 8-bit); the older 128-character encoding that forms its first part (American Standard Code for Information Interchange, Lesson 12.5); a gigabyte, about one billion bytes. |

## 1. Three ways to frame a stream

TCP delivers bytes, not messages (Lesson 15.3). Every protocol on TCP must therefore answer one
question: **where does a message end?** There are three common answers:

```text
 1. delimiter            G E T ␠ k e y \n S E T ␠ k ␠ v \n          "the message ends at \n"
 2. length prefix        00 00 00 05 h e l l o 00 00 00 02 h i      "the next 5 bytes are the message"
 3. fixed size           [16 bytes][16 bytes][16 bytes]             "every message is 16 bytes"
```

| Method | Good for | Watch out for |
|---|---|---|
| **Delimiter** | text protocols that people type and read (HTTP headers, SMTP, Redis commands, our chat) | the delimiter must not appear inside a message, or must be escaped; you must scan every byte to find it |
| **Length prefix** | binary data, files, anything that may contain any byte (TLS records, HTTP/2 frames, Kafka, PostgreSQL) | the length comes from the other side: limit it, or one frame can claim 4 GB |
| **Fixed size** | very simple sensor or hardware protocols | wastes space; cannot carry messages of different sizes |

Real protocols often combine them: an HTTP/1.1 request has a delimiter-based header (lines that end with
`\r\n`, and an empty line at the end) followed by a body whose size is given by a length header
(`Content-Length`). You will parse exactly that in Lesson 15.8.

## 2. A frame decoder by hand

Let us build length-prefixed framing ourselves first, so that no library hides the idea. The receiver
collects incoming bytes in a buffer. After every read it asks: "is there a complete frame at the front?"
There are exactly **three** possible answers:

1. **Not yet**: fewer than 4 bytes, or the length says 100 bytes and only 60 are there. Keep the bytes
   and read more. *Do not consume anything.*
2. **Yes**: cut the frame off the front of the buffer and return it. More frames may follow in the same
   buffer.
3. **Invalid**: the frame breaks a rule (here: it is too large). Return an error and close the
   connection.

The `bytes` crate gives us a good buffer for this. `BytesMut` is a growable byte buffer; `split_to(n)`
cuts the first `n` bytes off as their own buffer without copying them, and `freeze()` turns that into a
read-only `Bytes`. The traits `Buf` (reading: `advance`, `get_u16`, …) and `BufMut` (writing:
`put_u32`, `put_slice`, …) read and write numbers in big-endian order. Save as `src/bin/frames.rs`:

```rust
use bytes::{Buf, BufMut, Bytes, BytesMut};

const MAX_FRAME: usize = 64 * 1024;

#[derive(Debug, PartialEq)]
enum FrameError {
    TooLarge(usize),
}

/// Adds a 4-byte big-endian length in front of the payload.
fn encode_frame(payload: &[u8], out: &mut BytesMut) {
    out.put_u32(payload.len() as u32);
    out.put_slice(payload);
}

/// Takes one complete frame from the front of `buf`, if there is one.
fn decode_frame(buf: &mut BytesMut) -> Result<Option<Bytes>, FrameError> {
    if buf.len() < 4 {
        return Ok(None); // not even the length yet
    }
    let len = u32::from_be_bytes([buf[0], buf[1], buf[2], buf[3]]) as usize;
    if len > MAX_FRAME {
        return Err(FrameError::TooLarge(len)); // check BEFORE waiting for the data
    }
    if buf.len() < 4 + len {
        buf.reserve(4 + len - buf.len()); // make room for the rest
        return Ok(None);
    }
    buf.advance(4); // drop the length
    Ok(Some(buf.split_to(len).freeze())) // cut the payload off the front
}

fn main() {
    let mut wire = BytesMut::new();
    encode_frame(b"hello", &mut wire);
    encode_frame(b"world!", &mut wire);
    println!("on the wire: {:02x?}", &wire[..]);

    // The receiver gets the bytes in pieces of 3, as TCP might deliver them.
    let mut inbox = BytesMut::new();
    for piece in wire.chunks(3) {
        inbox.extend_from_slice(piece);
        while let Some(frame) = decode_frame(&mut inbox).unwrap() {
            println!("frame: {:?}", String::from_utf8_lossy(&frame));
        }
    }

    let mut evil = BytesMut::from(&[0xff, 0xff, 0xff, 0xff][..]);
    println!("{:?}", decode_frame(&mut evil));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn waits_for_a_complete_frame() {
        let mut buf = BytesMut::from(&[0, 0, 0, 5, b'h', b'e'][..]);
        assert_eq!(decode_frame(&mut buf), Ok(None));
        assert_eq!(buf.len(), 6, "nothing may be consumed yet");
        buf.extend_from_slice(b"llo");
        assert_eq!(
            decode_frame(&mut buf),
            Ok(Some(Bytes::from_static(b"hello")))
        );
        assert!(buf.is_empty());
    }

    #[test]
    fn two_frames_in_one_read() {
        let mut buf = BytesMut::new();
        encode_frame(b"a", &mut buf);
        encode_frame(b"bc", &mut buf);
        assert_eq!(decode_frame(&mut buf).unwrap().unwrap(), "a");
        assert_eq!(decode_frame(&mut buf).unwrap().unwrap(), "bc");
        assert_eq!(decode_frame(&mut buf), Ok(None));
    }

    #[test]
    fn rejects_huge_lengths() {
        let mut buf = BytesMut::from(&[0x7f, 0, 0, 0][..]);
        assert_eq!(
            decode_frame(&mut buf),
            Err(FrameError::TooLarge(0x7f00_0000))
        );
    }
}
```

`cargo run --bin frames`:

```text
on the wire: [00, 00, 00, 05, 68, 65, 6c, 6c, 6f, 00, 00, 00, 06, 77, 6f, 72, 6c, 64, 21]
frame: "hello"
frame: "world!"
Err(TooLarge(4294967295))
```

**What happens:**

1. `encode_frame` writes the length with `put_u32` (big-endian) and then the payload: `00 00 00 05`
   and `hello`.
2. In `main`, the receiver gets the 19 bytes in pieces of 3, which is the worst case of Lesson 15.4.
   After each piece, the inner `while let` loop takes out every complete frame. Most pieces produce
   nothing; two of them complete a frame.
3. The order of the checks in `decode_frame` matters. The size limit is checked **as soon as the length
   is known**, before waiting for the data. Otherwise a client could send `ff ff ff ff` and make the
   server wait for (and buffer) 4 GB.
4. The tests cover the three outcomes. The first test also checks that "not yet" consumed nothing: a
   decoder that eats part of a frame and then waits loses data.

`cargo test --bin frames` reports `3 passed`.

## 3. Design a message format

Framing tells us where a message ends. The **payload** still needs a format. We design a tiny key-value
protocol, "KV". A client sends `Get` and `Set` requests; the server answers `Value`, `NotFound` or `Ok`.
Each payload starts with one **type byte**; strings are written as a 2-byte length plus UTF-8 bytes:

```text
 frame:    [ length: u32 ][ payload ......................................... ]
 payload:  [ type: u8 ][ fields ... ]

 type 1  Get       [01][key len: u16][key bytes]
 type 2  Set       [02][key len: u16][key bytes][value len: u16][value bytes]
 type 3  Value     [03][value len: u16][value bytes]
 type 4  NotFound  [04]
 type 5  Ok        [05]
```

Questions to answer for every protocol you design:

- **Byte order** of every number: here big-endian everywhere.
- **Limits**: frames at most 64 KiB, strings at most 65,535 bytes (the largest `u16`).
- **What is an error**: an unknown type byte, a string that is shorter than its length, bytes that are not
  UTF-8, extra bytes after the last field. Every one of these must be an error, never a panic.
- **Versioning**: how will a newer client talk to an older server? A simple way is to send a version
  number first and to keep unknown type bytes an error. (Our protocol is version 1 and skips this.)

> 📏 **Convention:** write the wire format down as a table or diagram like the one above *before* you
> write code. It is the contract between client and server, and other people (or another language) must
> be able to implement it from the description alone.

## 4. Parse it, and let `tokio_util` do the framing

You now know how framing works. In real code you do not rewrite it: `tokio_util::codec` has a ready
length-prefix codec, `LengthDelimitedCodec`, with exactly our format (4-byte big-endian length) and a
size limit. `Framed` wraps a socket together with a codec. The result is a **stream** of frames (read
them with `.next().await` from `futures::StreamExt`) and a **sink** (send frames with `.send(...).await`
from `futures::SinkExt`). We only write the payload part. Save as `src/bin/kv.rs` (it is one file; it is
shown here in two parts). Part 1, the messages:

```rust
use bytes::{Buf, BufMut, Bytes, BytesMut};
use futures::{SinkExt, StreamExt};
use std::collections::HashMap;
use std::io;
use tokio::net::{TcpListener, TcpStream};
use tokio_util::codec::{Framed, LengthDelimitedCodec};

#[derive(Debug, Clone, PartialEq)]
enum Message {
    Get { key: String },
    Set { key: String, value: String },
    Value(String),
    NotFound,
    Ok,
}

fn bad(why: &str) -> io::Error {
    io::Error::new(io::ErrorKind::InvalidData, why.to_string())
}

fn put_str(out: &mut BytesMut, s: &str) {
    out.put_u16(s.len() as u16);
    out.put_slice(s.as_bytes());
}

fn get_str(p: &mut &[u8]) -> io::Result<String> {
    if p.remaining() < 2 {
        return Err(bad("string length missing"));
    }
    let len = p.get_u16() as usize;
    if p.remaining() < len {
        return Err(bad("string shorter than its length"));
    }
    let text = String::from_utf8(p[..len].to_vec()).map_err(|_| bad("string is not UTF-8"))?;
    p.advance(len);
    Ok(text)
}

impl Message {
    /// Payload layout: one type byte, then the fields as length-prefixed strings.
    fn encode(&self) -> Bytes {
        let mut out = BytesMut::new();
        match self {
            Message::Get { key } => {
                out.put_u8(1);
                put_str(&mut out, key);
            }
            Message::Set { key, value } => {
                out.put_u8(2);
                put_str(&mut out, key);
                put_str(&mut out, value);
            }
            Message::Value(v) => {
                out.put_u8(3);
                put_str(&mut out, v);
            }
            Message::NotFound => out.put_u8(4),
            Message::Ok => out.put_u8(5),
        }
        out.freeze()
    }

    fn parse(mut p: &[u8]) -> io::Result<Message> {
        if p.is_empty() {
            return Err(bad("empty payload"));
        }
        let msg = match p.get_u8() {
            1 => Message::Get {
                key: get_str(&mut p)?,
            },
            2 => Message::Set {
                key: get_str(&mut p)?,
                value: get_str(&mut p)?,
            },
            3 => Message::Value(get_str(&mut p)?),
            4 => Message::NotFound,
            5 => Message::Ok,
            other => return Err(bad(&format!("unknown message type {other}"))),
        };
        if p.has_remaining() {
            return Err(bad("extra bytes after the message"));
        }
        Ok(msg)
    }
}
```

**What happens:**

1. `Message` is the protocol as a Rust enum: one variant per type byte. `encode` and `parse` are the only
   places that know the wire format.
2. `parse` reads from `p: &[u8]` using the `Buf` trait, which is implemented for byte slices:
   `get_u8` and `get_u16` read a number **and move the slice forward**. That is why `get_str` takes
   `&mut &[u8]`, a mutable reference to a slice: it shortens the caller's slice.
3. Before every read, the code checks `remaining()`. `get_u16` on fewer than 2 bytes would panic, so the
   check turns a short, malicious payload into an error.
4. `String::from_utf8` rejects bytes that are not valid UTF-8 (Lesson 12.5). The last check rejects extra
   bytes: a strict parser leaves less room for confusion between implementations.

Part 2, the server, the client and the tests (same file, below part 1):

```rust
/// Frames: a 4-byte big-endian length, then the payload; at most 64 KiB.
fn framed(stream: TcpStream) -> Framed<TcpStream, LengthDelimitedCodec> {
    let codec = LengthDelimitedCodec::builder()
        .max_frame_length(64 * 1024)
        .new_codec();
    Framed::new(stream, codec)
}

async fn serve_one(stream: TcpStream) -> io::Result<()> {
    let mut store: HashMap<String, String> = HashMap::new();
    let mut frames = framed(stream);
    while let Some(frame) = frames.next().await {
        let answer = match Message::parse(&frame?)? {
            Message::Get { key } => match store.get(&key) {
                Some(v) => Message::Value(v.clone()),
                None => Message::NotFound,
            },
            Message::Set { key, value } => {
                store.insert(key, value);
                Message::Ok
            }
            _ => return Err(bad("clients may only send Get and Set")),
        };
        frames.send(answer.encode()).await?;
    }
    Ok(())
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:0").await?;
    let addr = listener.local_addr()?;
    tokio::spawn(async move {
        let (stream, _) = listener.accept().await?;
        serve_one(stream).await
    });

    let mut client = framed(TcpStream::connect(addr).await?);
    let lang = || "lang".to_string();
    for request in [
        Message::Get { key: lang() },
        Message::Set {
            key: lang(),
            value: "Rust".into(),
        },
        Message::Get { key: lang() },
    ] {
        println!("-> {request:?}  {:02x?}", &request.encode()[..]);
        client.send(request.encode()).await?;
        let frame = client.next().await.expect("server closed")?;
        println!("<- {:?}", Message::parse(&frame)?);
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn every_message_survives_a_round_trip() {
        let messages = [
            Message::Get { key: "k".into() },
            Message::Set {
                key: "".into(),
                value: "ünïcödé".into(),
            },
            Message::Value("v".into()),
            Message::NotFound,
            Message::Ok,
        ];
        for m in messages {
            assert_eq!(Message::parse(&m.encode()).unwrap(), m);
        }
    }

    #[test]
    fn garbage_is_an_error_not_a_panic() {
        for payload in [&[][..], &[9], &[1, 0], &[1, 0, 5, b'a'], &[5, 0]] {
            assert!(Message::parse(payload).is_err(), "{payload:?} was accepted");
        }
    }
}
```

`cargo run --bin kv` (the hex bytes are the payloads; the codec adds the 4-byte length in front):

```text
-> Get { key: "lang" }  [01, 00, 04, 6c, 61, 6e, 67]
<- NotFound
-> Set { key: "lang", value: "Rust" }  [02, 00, 04, 6c, 61, 6e, 67, 00, 04, 52, 75, 73, 74]
<- Ok
-> Get { key: "lang" }  [01, 00, 04, 6c, 61, 6e, 67]
<- Value("Rust")
```

**What happens:**

1. `framed` builds the codec with a 64 KiB limit. A peer that announces a bigger frame gets the error
   `frame size too big` from the codec, and the connection ends.
2. In `serve_one`, `frames.next().await` returns `Some(Ok(frame))` for each complete frame, `Some(Err(..))`
   for a framing error, and `None` at EOF. All the buffering and "not yet" logic of section 2 happens
   inside the codec.
3. `Message::parse(&frame?)?` uses two `?`: the first for framing errors, the second for payload errors.
   Both end the connection with an error. For a protocol bug, closing is safer than guessing.
4. `send` takes a `Bytes`, adds the length prefix, and writes it.
5. `cargo test --bin kv` runs the round-trip test (every variant, an empty key, non-ASCII text) and the
   garbage test (empty, unknown type, a too-short length, a string shorter than its length, extra bytes).
   It reports `2 passed`.

If you want your own codec instead of `LengthDelimitedCodec`, implement the trait `Decoder` (one method,
`decode(&mut self, buf: &mut BytesMut) -> Result<Option<Item>, Error>`, with exactly the three outcomes of
section 2) and `Encoder`. Exercise 3 does this for lines.

> 🔒 **Security:** a parser is the first code that touches attacker-controlled bytes. Check every length
> against the remaining input *and* against a fixed limit, never index without checking, and treat
> anything unexpected as an error that closes the connection. Fuzzing (Lesson 09.8) is the best tool to
> find the cases you forgot: `Message::parse(&[u8])` is a perfect fuzz target.

> 🧪 **Testing:** the round-trip test is a natural fit for property-based testing (Lesson 09.4):
> "for every `Message` m, `parse(encode(m)) == m`". Add one more test that feeds a valid frame to the
> decoder one byte at a time; it catches decoders that consume bytes too early.

## Common mistakes

> ⚠️ **Common mistake: checking the length after reading the data.** A decoder that first waits for
> `len` bytes and then checks `len <= MAX` lets a client make it buffer gigabytes. Check the limit the
> moment the length is known.

> ⚠️ **Common mistake: `as` casts that cut numbers.** `s.len() as u16` silently keeps only the lowest 16
> bits: a 70,000-byte string is announced as 4,464 bytes, and the receiver misreads everything after it.
> Here the 64 KiB frame limit catches it by luck. Prefer `u16::try_from(s.len())` and return an error.

> ⚠️ **Common mistake: indexing before checking.** `p[0]` or `get_u16()` on an empty slice panics, and a
> panic in a connection task is a remote way to kill it (or, with `panic = "abort"`, the whole server).
> Every read of the input needs a length check first.

## Try it yourself

1. In `frames.rs`, change `wire.chunks(3)` to `wire.chunks(1)` and then to `wire.chunks(100)`. The
   frames printed stay the same; only how often the decoder is called changes.
2. Remove the `if len > MAX_FRAME` check and run the tests: `rejects_huge_lengths` fails. Put it back.
3. In `kv.rs`, send `Message::Ok` from the client. The server answers by closing the connection, and the
   client's `next()` returns `None`, so it panics with `server closed`.

## Exercises

### Exercise 1 — Encode by hand (easy)

Write down, byte by byte in hexadecimal, the complete frame (length prefix included) that the client
sends for `Message::Set { key: "a".into(), value: "hi".into() }`.

<details>
<summary>Hint</summary>

First write the payload: the type byte, then for each string its length as 2 bytes and its bytes (`a` is
`61`, `h` is `68`, `i` is `69`). Count the payload bytes; the frame starts with that count as 4 bytes.

</details>

<details>
<summary>Answer</summary>

```text
00 00 00 08   02   00 01 61   00 02 68 69
 length=8    Set   key "a"    value "hi"
```

The payload is 1 + (2 + 1) + (2 + 2) = 8 bytes. Encoding the message with `encode()` and a
`LengthDelimitedCodec` and printing the bytes gives exactly
`[00, 00, 00, 08, 02, 00, 01, 61, 00, 02, 68, 69]`.

</details>

### Exercise 2 — A `Delete` message, with tests (medium)

Extend the protocol with `Delete { key }` (type byte 6). The server removes the key and answers `Ok`, or
`NotFound` if it did not exist. Update `encode`, `parse` and `serve_one`, and write a test: a round
trip for `Delete`, plus the check that a type byte 6 without a key is an error. Then change `main` to
send `Set`, `Delete`, `Delete` and `Get`, and check that the answers are `Ok`, `Ok`, `NotFound`,
`NotFound`.

<details>
<summary>Hint</summary>

The compiler helps you: after you add the variant, `encode` no longer compiles until you handle it
(`match` must be exhaustive). `parse` and `serve_one` use `_` arms, so there you must remember yourself.
`HashMap::remove` returns `Option<V>`.

</details>

<details>
<summary>Answer</summary>

Add the variant `Delete { key: String },` to `Message`. In `encode`, add the arm:

```rust
            Message::Delete { key } => {
                out.put_u8(6);
                put_str(&mut out, key);
            }
```

In `parse`, add the arm `6 => Message::Delete { key: get_str(&mut p)? },` (rustfmt spreads it over three
lines). In `serve_one`, add before the `_` arm:

```rust
            Message::Delete { key } => match store.remove(&key) {
                Some(_) => Message::Ok,
                None => Message::NotFound,
            },
```

Add this test to `mod tests`:

```rust
    #[test]
    fn delete_round_trip_and_bad_delete() {
        let msg = Message::Delete { key: "lang".into() };
        assert_eq!(Message::parse(&msg.encode()).unwrap(), msg);
        assert!(Message::parse(&[6]).is_err(), "Delete without a key");
    }
```

`cargo test --bin kv` reports `3 passed`, and the changed `main` prints the four expected answers.

Why it works: the protocol lives in one enum with one `encode` and one `parse`, so adding a message is a
local change, and the round-trip test immediately checks that both sides agree on the new bytes.

</details>

### Exercise 3 — A line codec with a limit (hard)

The chat server of Lesson 15.6 accepts lines of any length. Write `src/bin/lines_codec.rs` with a
`struct MaxLineCodec { max: usize }` that implements `tokio_util::codec::Decoder` with
`Item = String`. It returns lines without the `\n`, waits when there is no `\n` yet, and returns an error
as soon as a line is longer than `max` bytes, **even before its newline arrives**. Test it without any
network, by calling `decode` on a `BytesMut` directly.

<details>
<summary>Hint</summary>

`buf.iter().position(|&b| b == b'\n')` finds the newline. If there is none and `buf.len() > max`, the line
can only get longer: that is the early error. `split_to(i)` takes the line, `advance(1)` drops the `\n`.

</details>

<details>
<summary>Answer</summary>

```rust
use bytes::{Buf, BytesMut};
use std::io;
use tokio_util::codec::Decoder;

/// Splits a byte stream into lines, and refuses lines longer than `max` bytes.
struct MaxLineCodec {
    max: usize,
}

impl Decoder for MaxLineCodec {
    type Item = String;
    type Error = io::Error;

    fn decode(&mut self, buf: &mut BytesMut) -> io::Result<Option<String>> {
        match buf.iter().position(|&b| b == b'\n') {
            Some(i) if i > self.max => Err(too_long(self.max)),
            Some(i) => {
                let line = buf.split_to(i); // the line without '\n'
                buf.advance(1); // drop the '\n'
                String::from_utf8(line.to_vec())
                    .map(Some)
                    .map_err(|_| io::Error::new(io::ErrorKind::InvalidData, "not UTF-8"))
            }
            None if buf.len() > self.max => Err(too_long(self.max)),
            None => Ok(None), // no newline yet: wait for more bytes
        }
    }
}

fn too_long(max: usize) -> io::Error {
    io::Error::new(
        io::ErrorKind::InvalidData,
        format!("line longer than {max} bytes"),
    )
}

fn main() {
    let mut codec = MaxLineCodec { max: 10 };
    let mut buf = BytesMut::from("hello\nworld\nthis line is far too long\n");
    loop {
        match codec.decode(&mut buf) {
            Ok(Some(line)) => println!("line: {line:?}"),
            Ok(None) => break,
            Err(e) => {
                println!("error: {e}");
                break;
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn lines_split_across_reads() {
        let mut codec = MaxLineCodec { max: 100 };
        let mut buf = BytesMut::from("he");
        assert_eq!(codec.decode(&mut buf).unwrap(), None);
        buf.extend_from_slice(b"llo\nwor");
        assert_eq!(codec.decode(&mut buf).unwrap().as_deref(), Some("hello"));
        assert_eq!(codec.decode(&mut buf).unwrap(), None);
        assert_eq!(&buf[..], b"wor");
    }

    #[test]
    fn a_long_line_is_refused_before_its_newline_arrives() {
        let mut codec = MaxLineCodec { max: 4 };
        let mut buf = BytesMut::from("12345"); // no '\n' yet, but already too long
        assert!(codec.decode(&mut buf).is_err());
    }

    #[test]
    fn exactly_max_is_fine() {
        let mut codec = MaxLineCodec { max: 4 };
        let mut buf = BytesMut::from("1234\n");
        assert_eq!(codec.decode(&mut buf).unwrap().as_deref(), Some("1234"));
    }
}
```

```text
line: "hello"
line: "world"
error: line longer than 10 bytes
```

`cargo test --bin lines_codec` reports `3 passed`. With `Framed::new(stream, MaxLineCodec { max: 1024 })`
the chat server would be protected. (`tokio_util` also has a ready `LinesCodec::new_with_max_length`; now
you know what it does inside.)

Why it works: the decoder keeps the "not yet" case free of side effects and fails as soon as the rule is
broken, so memory per connection is bounded by `max` plus one read.

</details>

## Quiz

1. Name the three ways to frame messages in a byte stream and one protocol that uses each.

<details>
<summary>Answer</summary>

Delimiter (HTTP header lines and Redis commands end with `\r\n`), length prefix (TLS records, HTTP/2
frames, our KV protocol), fixed size (simple hardware protocols).

</details>

2. What are the three possible results of a `decode` call, and what must the decoder do with the buffer
   in each case?

<details>
<summary>Answer</summary>

A complete frame: remove exactly that frame from the buffer and return it. Not enough bytes yet: change
nothing (at most reserve space) and return `Ok(None)`. Invalid: return an error.

</details>

3. Why must the frame size limit be checked as soon as the length prefix has arrived?

<details>
<summary>Answer</summary>

Otherwise the decoder waits for and buffers as many bytes as the peer claims, up to 4 GB for a `u32`
length, which lets one client exhaust the server's memory.

</details>

4. Why does `get_str` take `p: &mut &[u8]` instead of `p: &[u8]`?

<details>
<summary>Answer</summary>

It must move the caller's slice forward past the string it read, so that the next field is read from the
right place. A `&mut` to the slice lets it replace the caller's slice with a shorter one.

</details>

## Summary

- TCP carries bytes; **framing** decides where messages end: delimiter, length prefix or fixed size.
- A decoder has three outcomes: complete frame (consume it), not yet (consume nothing), invalid (error).
- Check size limits the moment a length is known; check every length against the remaining input.
- Write the **wire format** down first: field order, sizes, byte order, limits, errors, versioning.
- `bytes::BytesMut` with `Buf`/`BufMut` reads and writes big-endian fields and splits buffers cheaply;
  `tokio_util::codec` gives `Framed`, `Decoder`/`Encoder`, `LengthDelimitedCodec` and `LinesCodec`.
- Test with round trips, byte-by-byte input and garbage; parsers are ideal targets for fuzzing.

## Sources and further reading

- [Tokio tutorial — Framing](https://tokio.rs/tokio/tutorial/framing) — the official tutorial on buffers and frames, with a Redis-style protocol.
- [`tokio_util::codec`](https://docs.rs/tokio-util/latest/tokio_util/codec/index.html) — `Framed`, `Decoder`, `Encoder` and the ready codecs.
- [`LengthDelimitedCodec`](https://docs.rs/tokio-util/latest/tokio_util/codec/length_delimited/index.html) — all options of the length-prefix codec (prefix size, byte order, limits).
- [`bytes` crate](https://docs.rs/bytes/latest/bytes/) — `Bytes`, `BytesMut`, `Buf` and `BufMut`.
- [Redis serialization protocol (RESP)](https://redis.io/docs/latest/develop/reference/protocol-spec/) — a real, well-documented protocol that mixes delimiters and length prefixes.

---

[🏠 Start](../START-HERE.md) · [← 15.6 Async networking with Tokio](06-async-networking-with-tokio.md) · [📋 Phase 15](README.md) · [15.8 HTTP from scratch →](08-http-from-scratch.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
