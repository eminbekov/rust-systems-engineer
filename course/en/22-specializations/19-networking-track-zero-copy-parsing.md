<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.18 Networking track: high-performance servers](18-networking-track-high-performance-servers.md) · [📋 Phase 22](README.md) · [22.20 Networking track: production proxies →](20-networking-track-production-proxies.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/19-networking-track-zero-copy-parsing.md) <!-- nav -->

# Lesson 22.19 — Networking track: zero-copy protocol parsing

> **Phase 22 — Specializations** · Lesson 19 of 30 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What "zero-copy" parsing means, and why borrowed slices with lifetimes make it safe in Rust.
- How to write a parser for the head of an HTTP/1.1 request that returns slices of the input.
- How to handle input that arrives in pieces: the difference between "invalid" and "not complete yet".
- How the `bytes` crate (`BytesMut`, `Bytes`) lets you keep parsed pieces without copying them.
- How the parser-combinator crate `nom` expresses the same parser, and how to test a parser with properties.

## Before you start

- You finished [Lesson 22.18 — Networking track: high-performance servers](18-networking-track-high-performance-servers.md).
- You know framing from [Lesson 15.7 — Designing a protocol: framing and parsing](../15-networking/07-designing-a-protocol.md),
  HTTP (Hypertext Transfer Protocol) messages from [Lesson 15.8 — HTTP from scratch](../15-networking/08-http-from-scratch.md),
  lifetimes from [Lesson 06.5 — Lifetimes in depth](../06-rust-advanced/05-lifetimes-in-depth.md) and property tests from
  [Lesson 09.4 — Property-based testing](../09-testing-debugging-benchmarking/04-property-based-testing.md).
- Add the crates to `code/p22-net`, and create the module file `src/parse.rs` with the line
  `pub mod parse;` at the top of `src/lib.rs`:

```bash
cd ~/projects/rust/code/p22-net
cargo add bytes@1
cargo add nom@8
cargo add --dev proptest@1
```

## New words

| Word | Meaning |
|---|---|
| **zero-copy parsing** | Parsing that does not copy the input: the results are *views* (borrowed slices) into the bytes that are already in memory. |
| **request head** | The first part of an HTTP/1.1 request: the request line and the headers, ending with an empty line. The body (if any) comes after it. |
| **CRLF** (Carriage Return, Line Feed) | The two bytes `\r\n` that end every line in HTTP/1.1. |
| **partial input** | Bytes that are a correct *beginning* of a message, but not the whole message yet. The right reaction is "read more", not "error". |
| **pipelining** | A client sends several requests without waiting for answers, so one read can contain more than one request. |
| **`BytesMut`** | A growable byte buffer from the `bytes` crate, used for reading from sockets. |
| **`Bytes`** | A cheap, shareable, read-only piece of a byte buffer. Cloning or slicing it copies no bytes, only a pointer and a reference count. |
| **reference count** | A number stored next to shared data that counts its owners; the data is freed when the count drops to zero (like `Arc`, Lesson 05.10). |
| **parser combinator** | A small parser that you combine with others into bigger parsers, like Lego bricks. `nom` is a library of them. |

## 1. What copying costs, and what "zero-copy" means

A proxy reads every request from the network into a buffer. A simple parser then builds a struct with
`String` fields: the method, the path, every header name and value. Each `String` is a new heap
allocation plus a copy of the bytes (Lesson 12.13). For one request this is nothing. For 100,000
requests per second with 15 headers each, it is millions of allocations per second, and the allocator
and the copies can take more time than the actual work.

A **zero-copy** parser does not create new strings. It returns *slices that point into the buffer*:

```text
 buf (bytes from the socket, already in memory)
 ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬────┬────┬───┬───┬───┬───┬───┬
 │ G │ E │ T │   │ / │ h │ e │ l │ l │ o │   │ H  │ T  │ T │ P │ / │ 1 │ . │ ...
 └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴────┴────┴───┴───┴───┴───┴───┴
   ▲           ▲
   │           └─ path:   &str { pointer to byte 4, length 6 }
   └─ method:  &str { pointer to byte 0, length 3 }
```

A slice is just a pointer and a length (16 bytes on a 64-bit machine), whatever the size of the data.
The trick has one danger: if the buffer is reused while someone still holds a slice, the slice would
silently show wrong bytes. In Rust, the **lifetime** in the result type ties every slice to
the buffer, and the borrow checker rejects any program that changes the buffer while a slice is alive.
You get the speed with no risk.

## 2. A hand-written parser for the request head

An HTTP/1.1 request head looks like this (each line ends with CRLF, and an empty line ends the head):

```text
GET /hello HTTP/1.1\r\n
Host: example.com\r\n
Accept: */*\r\n
\r\n
```

The parser must answer one of three things: "here is a complete head, it is N bytes long", "not complete
yet, read more", or "this is not a valid request". Put this in `src/parse.rs`:

```rust
//! A zero-copy parser for the head of an HTTP/1.1 request.

/// The largest request head we accept, in bytes.
pub const MAX_HEAD: usize = 8 * 1024;

#[derive(Debug, PartialEq)]
pub struct Request<'a> {
    pub method: &'a str,
    pub path: &'a str,
    pub headers: Vec<(&'a str, &'a [u8])>,
}

#[derive(Debug, PartialEq)]
pub enum Parsed<'a> {
    /// A whole head was found; it is `len` bytes long (with the empty line).
    Complete { request: Request<'a>, len: usize },
    /// Not enough bytes yet: read more and call again.
    Partial,
}

#[derive(Debug, PartialEq)]
pub enum ParseError {
    BadRequestLine,
    BadHeader,
    TooLarge,
}

pub fn parse_request(buf: &[u8]) -> Result<Parsed<'_>, ParseError> {
    let Some(end) = find(buf, b"\r\n\r\n") else {
        return if buf.len() > MAX_HEAD {
            Err(ParseError::TooLarge)
        } else {
            Ok(Parsed::Partial)
        };
    };
    if end + 4 > MAX_HEAD {
        return Err(ParseError::TooLarge);
    }
    let mut lines = buf[..end].split(|&b| b == b'\n');

    let request_line = lines.next().ok_or(ParseError::BadRequestLine)?;
    let request_line = request_line.strip_suffix(b"\r").unwrap_or(request_line);
    let mut parts = request_line.split(|&b| b == b' ');
    let (Some(method), Some(path), Some(b"HTTP/1.1"), None) =
        (parts.next(), parts.next(), parts.next(), parts.next())
    else {
        return Err(ParseError::BadRequestLine);
    };
    let method = text(method, ParseError::BadRequestLine)?;
    let path = text(path, ParseError::BadRequestLine)?;

    let mut headers = Vec::new();
    for line in lines {
        let line = line.strip_suffix(b"\r").unwrap_or(line);
        let colon = line
            .iter()
            .position(|&b| b == b':')
            .ok_or(ParseError::BadHeader)?;
        let name = text(&line[..colon], ParseError::BadHeader)?;
        headers.push((name, line[colon + 1..].trim_ascii()));
    }
    let request = Request {
        method,
        path,
        headers,
    };
    Ok(Parsed::Complete {
        request,
        len: end + 4,
    })
}

/// Position of the first `needle` in `haystack`.
fn find(haystack: &[u8], needle: &[u8]) -> Option<usize> {
    haystack.windows(needle.len()).position(|w| w == needle)
}

/// Borrow `bytes` as `&str` if they are non-empty, visible ASCII characters.
fn text(bytes: &[u8], err: ParseError) -> Result<&str, ParseError> {
    if bytes.is_empty() || !bytes.iter().all(|b| b.is_ascii_graphic()) {
        return Err(err);
    }
    // All bytes are ASCII, so this is valid UTF-8 and cannot fail.
    std::str::from_utf8(bytes).map_err(|_| err)
}
```

**What happens:**

1. `Request<'a>` holds only borrowed data: `&'a str` and `&'a [u8]`. The lifetime `'a` says "these
   slices live inside some buffer that lives at least as long as `'a`". `parse_request(buf: &[u8])`
   returns `Parsed<'_>`, so `'a` is the lifetime of `buf` (lifetime elision, Lesson 05.3).
2. First the parser looks for the empty line `\r\n\r\n`. If it is not there, the head is not complete:
   the answer is `Partial`, unless the buffer is already larger than `MAX_HEAD`, then it is `TooLarge`.
   A complete head that is longer than `MAX_HEAD` is `TooLarge` too, so the limit holds however the
   bytes arrive.
3. `split(|&b| b == b'\n')` cuts the head into lines without copying; `strip_suffix(b"\r")` removes the
   `\r` from each line, again only by making a shorter slice.
4. The request line must have exactly three parts separated by single spaces, and the third must be
   `HTTP/1.1`. The `let ... else` with a tuple pattern checks all of this in one place (Lesson 06.3).
5. `text` checks that a slice contains only visible ASCII (American Standard Code for Information
   Interchange) characters and borrows it as `&str`. `from_utf8` does not copy; it only checks.
6. Each header is split at the first `:`. The value is a `&[u8]`, not a `&str`, because HTTP allows
   bytes in values that are not valid UTF-8 (Unicode Transformation Format, 8-bit, the usual text encoding, Lesson 12.5). `trim_ascii` drops the spaces around it.
7. `len: end + 4` is the length of the whole head including the final `\r\n\r\n`, so the caller knows
   where the next request (or the body) starts.

Now feed it bytes that arrive in two pieces, as they would from a socket. Save as
`src/bin/parse_demo.rs`:

```rust
use p22_net::parse::{Parsed, parse_request};

fn main() {
    let mut buf: Vec<u8> = Vec::new();

    // The bytes arrive from the network in two pieces.
    buf.extend_from_slice(b"GET /hello HTTP/1.1\r\nHost: exa");
    println!("after piece 1: {:?}", parse_request(&buf));

    buf.extend_from_slice(b"mple.com\r\nAccept: */*\r\n\r\n");
    match parse_request(&buf) {
        Ok(Parsed::Complete { request, len }) => {
            println!("complete, head is {len} of {} bytes", buf.len());
            println!("method = {}, path = {}", request.method, request.path);
            for (name, value) in &request.headers {
                println!("header {name}: {}", String::from_utf8_lossy(value));
            }
            let offset = request.path.as_ptr() as usize - buf.as_ptr() as usize;
            println!("path lives inside buf at offset {offset}");
        }
        other => println!("unexpected: {other:?}"),
    }
}
```

`cargo run --bin parse_demo`:

```text
after piece 1: Ok(Partial)
complete, head is 55 of 55 bytes
method = GET, path = /hello
header Host: example.com
header Accept: */*
path lives inside buf at offset 4
```

The last line is the proof of zero-copy: the address of `request.path` is the address of `buf` plus 4,
so the path *is* bytes 4 to 9 of the buffer, not a copy of them.

> 🔒 **Security:** `MAX_HEAD` is not optional. Without it, a client that sends bytes forever without an
> empty line makes your buffer grow until the process runs out of memory. Every real HTTP server has
> such a limit (8 KiB to 64 KiB is common). Never trust input sizes from the network.

> ⚡ **Performance:** `find` searches from the start of the buffer on every call. If a slow client sends
> one byte at a time, that is O(n²) work (Lesson 07.1). Production parsers remember where they stopped,
> or search with fast routines such as the `memchr` crate. The parser inside hyper is the `httparse`
> crate, which is worth reading after this lesson.

## 3. Keeping parsed pieces: `BytesMut` and `Bytes`

Borrowed slices have one limit: they cannot outlive the buffer. But a proxy wants to *keep* parts of a
request: send the path to a logging task, keep the head until the upstream server answers, and at the
same time reuse the buffer for the next request. Copying would bring back the cost we just removed.

The `bytes` crate solves this with two types:

- `BytesMut` is the buffer you read into. Its method `split_to(n)` cuts the first `n` bytes off as a
  separate value, *without copying*: both parts still point into the same memory, and the original
  buffer now starts at byte `n`.
- `freeze()` turns that cut-off `BytesMut` into `Bytes`, a read-only handle with a reference count.
  `clone()` and `slice(range)` on `Bytes` create new handles to the same memory. The memory is freed
  when the last handle is dropped. `Bytes` is `Send` and `'static`, so it can move to other tasks and
  threads.

Save as `src/bin/bytes_demo.rs`:

```rust
use bytes::{Bytes, BytesMut};
use p22_net::parse::{Parsed, parse_request};

fn main() {
    let mut buf = BytesMut::with_capacity(4096);
    // Two requests arrived in one read (this is called pipelining).
    buf.extend_from_slice(b"GET /a HTTP/1.1\r\nHost: x\r\n\r\nGET /b HTTP/1.1\r\nHost: y\r\n\r\n");
    let start_of_buffer = buf.as_ptr() as usize;

    let mut heads: Vec<(Bytes, Bytes)> = Vec::new();
    while let Ok(Parsed::Complete { request, len }) = parse_request(&buf) {
        // 1. Remember where the path is. `request` borrows `buf` until here.
        let path_start = request.path.as_ptr() as usize - buf.as_ptr() as usize;
        let path_end = path_start + request.path.len();
        // 2. Cut the head off the front of the buffer. No bytes are copied.
        let head: Bytes = buf.split_to(len).freeze();
        // 3. A Bytes for just the path: shares the same memory as `head`.
        let path = head.slice(path_start..path_end);
        heads.push((head, path));
    }

    for (head, path) in &heads {
        println!(
            "path {:?}: head at offset {}, path at offset {}",
            path,
            head.as_ptr() as usize - start_of_buffer,
            path.as_ptr() as usize - start_of_buffer
        );
    }
    println!("bytes left in buf: {}", buf.len());

    // Bytes is 'static and Send: it can move to another thread.
    let (_, path) = heads.remove(1);
    let handle = std::thread::spawn(move || format!("worker got {path:?}"));
    println!("{}", handle.join().unwrap());
}
```

**What happens:**

1. The buffer holds two pipelined requests. The `while let` loop parses one head per round.
2. Inside the loop, `request` borrows `buf`. We only read two numbers from it (where the path starts
   and ends). After that line `request` is not used any more, so the borrow ends, and the next line may
   change `buf` (non-lexical lifetimes: a borrow lasts until its last use).
3. `buf.split_to(len)` cuts off the first head. `buf` now starts at the second request.
4. `head.slice(...)` makes a `Bytes` for the path that points into the same memory as `head`.
5. The printed offsets are measured from the start of the original buffer: the second head starts at
   byte 28, exactly where it was in the buffer. Nothing moved, nothing was copied.
6. Finally a path moves into another thread. A `&[u8]` could not do this, because the thread may live
   longer than the buffer; `Bytes` owns its share of the memory.

`cargo run --bin bytes_demo`:

```text
path b"/a": head at offset 0, path at offset 4
path b"/b": head at offset 28, path at offset 32
bytes left in buf: 0
worker got b"/b"
```

This is the standard pattern of Rust network code: read into `BytesMut`, parse with borrowed slices,
then `split_to(...).freeze()` the finished message into `Bytes` and pass it on. Tokio's
`AsyncReadExt::read_buf` reads straight into a `BytesMut` (exercise 3), and hyper, Pingora and most
Tokio codecs pass `Bytes` around.

## 4. The same idea with `nom`

Writing parsers by hand is fine for a small format. For bigger formats, a **parser combinator** library
helps: you write small parsers ("one or more uppercase letters", "exactly a space") and combine them.
In `nom`, a parser is a function that takes the input and returns `Ok((rest, value))` or an error, and
the values are slices of the input, so `nom` is zero-copy too. Save as `src/bin/nom_demo.rs`:

```rust
use nom::bytes::streaming::{tag, take_while1};
use nom::{IResult, Parser};

/// `METHOD SP PATH SP HTTP/1.1 CRLF`, returns (method, path).
fn request_line(input: &[u8]) -> IResult<&[u8], (&[u8], &[u8])> {
    let method = take_while1(|b: u8| b.is_ascii_uppercase());
    let path = take_while1(|b: u8| b.is_ascii_graphic());
    let (rest, (method, _, path, _, _)) =
        (method, tag(" "), path, tag(" HTTP/1.1"), tag("\r\n")).parse(input)?;
    Ok((rest, (method, path)))
}

fn main() {
    for input in [
        &b"GET /index.html HTTP/1.1\r\nHost: x\r\n"[..],
        &b"GET /index.ht"[..],
        &b"get / HTTP/1.1\r\n"[..],
    ] {
        match request_line(input) {
            Ok((rest, (method, path))) => println!(
                "ok: method={:?} path={:?}, {} bytes left",
                String::from_utf8_lossy(method),
                String::from_utf8_lossy(path),
                rest.len()
            ),
            Err(nom::Err::Incomplete(needed)) => println!("incomplete: {needed:?}"),
            Err(e) => println!("error: {e:?}"),
        }
    }
}
```

**What happens:**

1. `take_while1(f)` is a parser that takes one or more bytes for which `f` is true. `tag(" ")` is a
   parser that expects exactly that text.
2. A tuple of parsers is itself a parser that runs them in order; `.parse(input)` (from the `Parser`
   trait, which `nom` 8 requires you to import) runs it. The result is the rest of the input and a tuple
   of the five results; we keep two of them.
3. We import from `nom::bytes::streaming`. The **streaming** versions return `Err::Incomplete` when
   the input ends too early, which is our `Partial`. The `complete` versions would return an error
   instead, which is right for a whole file, but wrong for a socket.
4. The `?` passes errors up; `IResult` is `nom`'s `Result` type.

`cargo run --bin nom_demo`:

```text
ok: method="GET" path="/index.html", 9 bytes left
incomplete: Size(1)
error: Error(Error { input: [103, 101, 116, 32, 47, 32, 72, 84, 84, 80, 47, 49, 46, 49, 13, 10], code: TakeWhile1 })
```

The third input fails because `get` is lowercase, and the error shows where (`input` is the rest of the
bytes, `103` is `g`) and which parser failed. Which style to choose? A hand-written parser gives the
best control over speed and error messages; `nom` gives shorter code for bigger grammars. Both are used
in production.

> 🧪 **Testing:** A parser receives input from strangers, so test it with inputs you did not think of.
> Two properties are worth testing for every network parser: "it never panics, whatever the bytes" and
> "every strict prefix of a valid message is `Partial`". Exercise 2 writes both with `proptest`, and
> `cargo fuzz` (Lesson 09.8) is the next step.

## Common mistakes

**Keeping a parsed request while reusing the buffer.** Save as `src/bin/dangling.rs`:

```rust
use p22_net::parse::{Parsed, parse_request};

fn main() {
    let mut buf = b"GET / HTTP/1.1\r\n\r\n".to_vec();
    let Ok(Parsed::Complete { request, .. }) = parse_request(&buf) else {
        return;
    };
    buf.clear(); // reuse the buffer for the next request
    println!("{}", request.path);
}
```

```text
error[E0502]: cannot borrow `buf` as mutable because it is also borrowed as immutable
 --> src/bin/dangling.rs:8:5
  |
5 |     let Ok(Parsed::Complete { request, .. }) = parse_request(&buf) else {
  |                                                              ---- immutable borrow occurs here
...
8 |     buf.clear(); // reuse the buffer for the next request
  |     ^^^^^^^^^^^ mutable borrow occurs here
9 |     println!("{}", request.path);
  |                    ------------ immutable borrow later used here
```

This is the borrow checker protecting the zero-copy design: `request.path` points into `buf`, so `buf`
cannot be cleared while the path is still used. Fix: finish using the request before you change the
buffer, or turn the parts you need into `Bytes` first (section 3).

**Treating partial input as an error.** TCP (Transmission Control Protocol) gives you bytes, not messages (Lesson 15.4). A parser that
returns an error for `GET /hel` will fail randomly, depending on how the network split the bytes. Always
have three outcomes: complete, partial, invalid.

**Forgetting how many bytes were used.** After a complete head, the caller must remove exactly `len`
bytes from the buffer. If it removes the whole buffer, a second pipelined request is lost; if it removes
too few, the next parse sees garbage.

**Converting every header to `String` "to be safe".** It works, but it brings back all the allocations.
Keep slices (or `Bytes`) and convert only the few values you really need to own.

## Try it yourself

1. In `parse_demo.rs`, cut the input in three pieces instead of two, with the cut inside `\r\n\r\n`.
   The answer stays `Partial` until the last piece.
2. Send a request line with two spaces between the method and the path (`GET  / HTTP/1.1`). Which error
   do you get, and why?
3. In `nom_demo.rs`, change `streaming` to `complete` in the import and run it again. The second input
   now prints an `error`, not `incomplete`.

## Exercises

### Exercise 1 — Find a header (easy)

Add a method `pub fn header(&self, name: &str) -> Option<&'a [u8]>` to `Request<'a>` that returns the
value of the first header with that name. Header names in HTTP are case-insensitive, so `host` must find
`Host`. Write a unit test.

<details>
<summary>Hint</summary>

`str::eq_ignore_ascii_case` compares two strings ignoring ASCII case. Note the return type: the value
lives as long as the *buffer* (`'a`), not only as long as `&self`.

</details>

<details>
<summary>Answer</summary>

Put this in `src/parse.rs`:

```rust
impl<'a> Request<'a> {
    /// The value of the first header called `name`, ignoring ASCII case.
    pub fn header(&self, name: &str) -> Option<&'a [u8]> {
        self.headers
            .iter()
            .find(|(n, _)| n.eq_ignore_ascii_case(name))
            .map(|&(_, value)| value)
    }
}

#[cfg(test)]
mod header_tests {
    use super::*;

    #[test]
    fn header_lookup_ignores_case() {
        let buf = b"GET / HTTP/1.1\r\nHost: a.example\r\nX-Id: 7\r\n\r\n";
        let Ok(Parsed::Complete { request, .. }) = parse_request(buf) else {
            panic!("expected a complete request");
        };
        assert_eq!(request.header("host"), Some(&b"a.example"[..]));
        assert_eq!(request.header("X-ID"), Some(&b"7"[..]));
        assert_eq!(request.header("Accept"), None);
    }
}
```

```text
test parse::header_tests::header_lookup_ignores_case ... ok
```

Why it works: `iter()` gives references to the `(name, value)` pairs; `.map(|&(_, value)| value)`
copies out the slice (a slice reference is `Copy`), which still points into the buffer, so it can be
returned with lifetime `'a`.

</details>

### Exercise 2 — Properties of a network parser (medium)

Write an integration test `tests/never_panics.rs` with `proptest` that checks two properties:
`parse_request` never panics for any byte vector up to 2,000 bytes, and every strict prefix of a valid
request is `Partial`. Add a normal test that the whole valid request is `Complete` with the right length.

<details>
<summary>Hint</summary>

`proptest::collection::vec(any::<u8>(), 0..2000)` generates byte vectors. For the prefixes, generate
a cut position `cut in 0..VALID.len()` and parse `&VALID[..cut]`. Inside `proptest!`, use
`prop_assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
use p22_net::parse::{Parsed, parse_request};
use proptest::prelude::*;

const VALID: &[u8] = b"POST /api/items HTTP/1.1\r\nHost: shop\r\nContent-Length: 0\r\n\r\n";

proptest! {
    #[test]
    fn never_panics_on_any_bytes(input in proptest::collection::vec(any::<u8>(), 0..2000)) {
        let _ = parse_request(&input);
    }

    #[test]
    fn every_strict_prefix_is_partial(cut in 0..VALID.len()) {
        prop_assert_eq!(parse_request(&VALID[..cut]), Ok(Parsed::Partial));
    }
}

#[test]
fn the_whole_request_is_complete() {
    let parsed = parse_request(VALID).unwrap();
    assert!(matches!(parsed, Parsed::Complete { len, .. } if len == VALID.len()));
}
```

```text
test the_whole_request_is_complete ... ok
test every_strict_prefix_is_partial ... ok
test never_panics_on_any_bytes ... ok
```

Why it works: proptest runs each property with 256 random inputs by default. A panic (for example an
index out of range in a slice) would fail the first test and be shrunk to a small example. The second
property catches parsers that answer "error" or "complete" too early.

</details>

### Exercise 3 — Read heads from a stream (hard)

Write `src/read.rs` with
`pub async fn read_head<R: AsyncRead + Unpin>(reader: &mut R, buf: &mut BytesMut) -> Result<Option<Bytes>, ReadError>`.
It reads until `buf` contains a complete head, then cuts the head off with `split_to` and returns it as
`Bytes`. It returns `Ok(None)` if the peer closes the connection before sending anything, and an error
if it closes in the middle of a head. Test it with `tokio::io::duplex`, sending two pipelined requests
one byte at a time.

<details>
<summary>Hint</summary>

Loop: first try to parse what is already in `buf` (a previous read may have brought two requests);
only if the result is `Partial`, call `reader.read_buf(buf).await` from `AsyncReadExt`. `read_buf`
returns `Ok(0)` at end of file. `tokio::io::duplex(64)` gives two connected in-memory streams.

</details>

<details>
<summary>Answer</summary>

Add `pub mod read;` to `src/lib.rs`, then:

```rust
use crate::parse::{ParseError, Parsed, parse_request};
use bytes::{Bytes, BytesMut};
use tokio::io::{AsyncRead, AsyncReadExt};

#[derive(Debug)]
pub enum ReadError {
    Io(std::io::Error),
    Parse(ParseError),
    /// The peer closed the connection in the middle of a head.
    UnexpectedEof,
}

/// Reads until `buf` holds one whole request head and returns that head.
/// Returns `Ok(None)` if the peer closed the connection before sending anything.
pub async fn read_head<R: AsyncRead + Unpin>(
    reader: &mut R,
    buf: &mut BytesMut,
) -> Result<Option<Bytes>, ReadError> {
    loop {
        match parse_request(buf).map_err(ReadError::Parse)? {
            Parsed::Complete { len, .. } => return Ok(Some(buf.split_to(len).freeze())),
            Parsed::Partial => {}
        }
        let n = reader.read_buf(buf).await.map_err(ReadError::Io)?;
        if n == 0 {
            return if buf.is_empty() {
                Ok(None)
            } else {
                Err(ReadError::UnexpectedEof)
            };
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use tokio::io::AsyncWriteExt;

    #[tokio::test]
    async fn reads_heads_sent_one_byte_at_a_time() {
        let (mut client, mut server) = tokio::io::duplex(64);
        tokio::spawn(async move {
            for byte in b"GET /1 HTTP/1.1\r\n\r\nGET /2 HTTP/1.1\r\n\r\n" {
                client.write_all(&[*byte]).await.unwrap();
            }
            // dropping `client` closes the connection
        });

        let mut buf = BytesMut::new();
        let first = read_head(&mut server, &mut buf).await.unwrap().unwrap();
        let second = read_head(&mut server, &mut buf).await.unwrap().unwrap();
        assert_eq!(&first[..], b"GET /1 HTTP/1.1\r\n\r\n");
        assert_eq!(&second[..], b"GET /2 HTTP/1.1\r\n\r\n");
        assert!(read_head(&mut server, &mut buf).await.unwrap().is_none());
    }

    #[tokio::test]
    async fn a_cut_head_is_an_error() {
        let mut input: &[u8] = b"GET / HTTP/1.1\r\nHost:";
        let mut buf = BytesMut::new();
        let result = read_head(&mut input, &mut buf).await;
        assert!(matches!(result, Err(ReadError::UnexpectedEof)));
    }
}
```

```text
test read::tests::a_cut_head_is_an_error ... ok
test read::tests::reads_heads_sent_one_byte_at_a_time ... ok
```

Why it works: parsing *before* reading matters for pipelining: after the first head is cut off, the
second may already be in `buf`, and waiting for a read would hang. `read_buf` appends to `BytesMut` and
grows it as needed. `&[u8]` implements `AsyncRead`, which makes the second test a one-liner input.

</details>

## Quiz

1. What does a zero-copy parser return instead of `String` values?

<details>
<summary>Answer</summary>

Borrowed slices (`&str`, `&[u8]`) that point into the input buffer, or `Bytes` handles that share the
buffer's memory.

</details>

2. Why can zero-copy parsing not cause "slices that show wrong bytes" in safe Rust?

<details>
<summary>Answer</summary>

The result type has a lifetime tied to the buffer. The borrow checker rejects any code that changes or
frees the buffer while a slice from it is still used (error E0502 in the common mistakes).

</details>

3. What are the three possible results of parsing bytes from a socket, and what should the caller do
   for each?

<details>
<summary>Answer</summary>

Complete (use the message and remove its `len` bytes from the buffer), partial (read more bytes and try
again), invalid (answer with an error such as `400 Bad Request` and close the connection).

</details>

4. What does `BytesMut::split_to(n)` do, and does it copy?

<details>
<summary>Answer</summary>

It cuts the first `n` bytes off into a separate `BytesMut` and leaves the rest in the original. It
does not copy bytes; both parts point into the same memory.

</details>

5. Why do we import parsers from `nom::bytes::streaming` and not `nom::bytes::complete` for network
   input?

<details>
<summary>Answer</summary>

Streaming parsers report `Incomplete` when the input ends early, which means "read more". Complete
parsers treat the end of input as final and return an error, which is wrong when more bytes may still
arrive.

</details>

6. Why does the parser need a `MAX_HEAD` limit?

<details>
<summary>Answer</summary>

Without a limit, a client that never sends the empty line makes the buffer grow without end, and the
server runs out of memory. The limit turns that into a clean `TooLarge` error.

</details>

## Summary

- Zero-copy parsing returns slices into the input instead of new strings, which removes allocations
  and copies from the hot path.
- Lifetimes make it safe: the borrow checker does not let the buffer change while a slice is alive.
- A network parser has three outcomes: complete (with its length), partial, invalid. Partial is not an
  error.
- Limit the size of what you buffer (`MAX_HEAD`); never trust sizes from the network.
- `BytesMut` is the read buffer; `split_to(..).freeze()` turns a finished message into `Bytes`, which
  can be sliced, cloned and sent to other tasks without copying.
- `nom` builds the same kind of zero-copy parser from small combinators; use its `streaming` parsers
  for sockets.
- Test parsers with properties ("never panics", "prefixes are partial") and then with fuzzing.

## Sources and further reading

- [bytes documentation](https://docs.rs/bytes/1/bytes/) — `Bytes`, `BytesMut`, `split_to`, `freeze` and how the reference counting works.
- [nom documentation](https://docs.rs/nom/8/nom/) — the list of combinators and the difference between streaming and complete parsers.
- [httparse documentation](https://docs.rs/httparse/latest/httparse/) — the zero-copy HTTP/1.x parser used by hyper; compare it with your own.
- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112) — the exact grammar of the request line and the header fields (an RFC, Request for Comments, is an official internet standard document).
- [tokio::io::AsyncReadExt::read_buf](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html) — reading from a socket straight into a `BytesMut`.
- [The Rust Programming Language — 10.3 Validating References with Lifetimes](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) — a refresher on the lifetime rules that make zero-copy safe.

---

[🏠 Start](../START-HERE.md) · [← 22.18 Networking track: high-performance servers](18-networking-track-high-performance-servers.md) · [📋 Phase 22](README.md) · [22.20 Networking track: production proxies →](20-networking-track-production-proxies.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
