<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.7 Designing a protocol: framing and parsing](07-designing-a-protocol.md) · [📋 Phase 15](README.md) · [15.9 HTTP with hyper and reqwest →](09-http-with-hyper-and-reqwest.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/08-http-from-scratch.md) <!-- nav -->

# Lesson 15.8 — HTTP from scratch

> **Phase 15 — Networking** · Lesson 8 of 16 · ⏱ about 3 hours (2 sittings)

## What you will learn

- The exact format of HTTP/1.1 requests and responses: the start line, headers, the empty line and the body.
- How a server knows where a body ends (`Content-Length` and chunked transfer coding) and why getting this wrong is a security hole.
- How persistent connections (keep-alive) and pipelining work, and how `Connection: close` ends them.
- How to write a strict, limited HTTP/1.1 parser as a pure function and a Tokio server around it.
- How to test the parser without a network and check the server with `curl -v` and `nc`.

## Before you start

- You finished [Lesson 15.7 — Designing a protocol: framing and parsing](07-designing-a-protocol.md). This lesson applies its three outcomes (complete, not yet, invalid) to a real protocol.
- You built a first HTTP server in [Lesson 06.11 — Project: multithreaded web server](../06-rust-advanced/11-project-multithreaded-web-server.md): it read one request per connection with a single `read` and answered with `Connection: close`. Now you do it properly.
- `code/p15-net` has `tokio` and `bytes` (Lessons 15.6 and 15.7). `curl` is part of macOS.

## New words

| Word | Meaning |
|---|---|
| **request line** | The first line of an HTTP request: method, path (the *request target*), version, for example `GET /index.html HTTP/1.1`. |
| **status line** | The first line of a response: version, status code and reason phrase, for example `HTTP/1.1 404 Not Found`. |
| **method** | What the client wants to do: `GET` (read), `POST` (send data), `PUT`, `DELETE`, `HEAD` and a few others. |
| **header field** | One `Name: value` line after the start line. Names are case-insensitive. |
| **message body** | The bytes after the empty line: an uploaded form, JSON, a web page. |
| **status code** | A three-digit number: `2xx` success, `3xx` redirection, `4xx` client error, `5xx` server error. |
| **persistent connection** (keep-alive) | One TCP connection used for several requests one after another. The default in HTTP/1.1. |
| **pipelining** | Sending the next request before the answer to the previous one has arrived. Answers must come back in the same order. |
| **chunked transfer coding** | Sending a body in pieces, each with its size in front, ending with a piece of size 0. Used when the total size is not known in advance. |
| **request smuggling** | An attack in which two HTTP programs (for example a proxy and a server) disagree on where a request ends, so that an attacker can hide a second request inside the first. |
| **CRLF** (Carriage Return, Line Feed) | The two bytes `\r\n` (13 and 10) that end every line in the head of an HTTP message. |
| **HTTP** (HyperText Transfer Protocol) | The request/response protocol of the web, the topic of this lesson. |
| **TCP, IP, RTT** (Transmission Control Protocol, Internet Protocol, Round-Trip Time) | The byte stream under HTTP/1.1, the protocol that carries its packets, and the time for one message to go to the other side and back (Lessons 15.1 and 15.3). |
| **HTTPS, TLS** (HTTP Secure, Transport Layer Security) | HTTP sent inside an encrypted TLS connection (Lesson 15.12). |
| **RFC, JSON, URL** (Request for Comments, JavaScript Object Notation, Uniform Resource Locator) | The documents that define internet standards; a common text format for data; a web address such as `http://127.0.0.1:7300/echo`. |
| **KB, MB** (kilobyte, megabyte) | About one thousand and about one million bytes. |

## 1. What an HTTP/1.1 message looks like

You saw the shape in Lesson 06.11. Here are the exact rules (RFC 9112). A request is:

```text
request-line    GET /missing HTTP/1.1\r\n            method SP target SP version CRLF
header fields   Host: 127.0.0.1:7300\r\n             name ":" optional spaces value CRLF
                User-Agent: curl/8.7.1\r\n
                Accept: */*\r\n
empty line      \r\n                                 end of the head
message body    (0 or more bytes)                    how many? see section 2
```

A response has the same structure with a **status line** first:

```text
HTTP/1.1 404 Not Found\r\n
Content-Type: text/plain\r\n
Content-Length: 10\r\n
\r\n
not found\n
```

The rules that matter for a parser:

1. Every line of the head ends with **CRLF** (`\r\n`), and an **empty line** ends the head. The head is
   therefore delimiter-framed, as in Lesson 15.7.
2. The request line has exactly three parts separated by single spaces.
3. Header names are **case-insensitive**: `Content-Length`, `content-length` and `CONTENT-LENGTH` are the
   same header. Values may have spaces around them, which are not part of the value.
4. HTTP/1.1 requests must contain a `Host` header, because one IP address often serves many websites.
5. The status code is what programs look at; the reason phrase ("Not Found") is only for people.

The head is text, but the body can be any bytes. That is why the next question needs care.

## 2. Where does the body end?

The head ends at the empty line. For the body, a request uses one of two rules:

- **`Content-Length: N`**: exactly `N` bytes follow. This is length-prefix framing, written as text.
- **`Transfer-Encoding: chunked`**: the body comes in chunks. Each chunk is its size in *hexadecimal*,
  CRLF, the data, CRLF. A chunk of size 0 ends the body:

```text
4\r\n          ← 4 bytes follow
Wiki\r\n
5\r\n          ← 5 bytes follow
pedia\r\n
0\r\n          ← the last chunk
\r\n           ← empty line: the body is complete ("Wikipedia")
```

A request with neither header has no body. (A *response* has a third option: with
`Connection: close` and no length, the body ends when the server closes the connection. That is what
your server in Lesson 06.11 relied on.)

> 🔒 **Security:** if a request contains both `Content-Length` and `Transfer-Encoding`, or two different
> `Content-Length` values, two HTTP programs can disagree about where it ends. A proxy may see one request
> where the server behind it sees two, and the attacker's hidden second request bypasses the proxy's
> checks. This is **request smuggling**. RFC 9112 tells servers to treat such requests as errors. Our
> parser rejects every request that has a `Transfer-Encoding` header (it supports only `Content-Length`)
> and every request with more than one `Content-Length`.

## 3. A strict parser as a pure function

As in Lesson 15.7, the parser does not touch the network. It gets the bytes received so far and answers
with one of three results: a complete request plus the number of bytes it used, "not yet", or an error.
The whole server is one file, `src/bin/http_server.rs`, shown in three parts. Part 1, the parser:

```rust
use bytes::{Buf, BytesMut};
use std::time::Duration;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio::time::timeout;

const MAX_HEAD: usize = 8 * 1024; // request line + headers
const MAX_BODY: usize = 1024 * 1024;

#[derive(Debug, PartialEq)]
struct Request {
    method: String,
    path: String,
    version: String,
    headers: Vec<(String, String)>, // names in lower case
    body: Vec<u8>,
}

impl Request {
    fn header(&self, name: &str) -> Option<&str> {
        let mut values = self.headers.iter().filter(|(n, _)| n == name);
        values.next().map(|(_, v)| v.as_str())
    }
}

#[derive(Debug, PartialEq)]
enum HttpError {
    BadRequest(&'static str),
    HeadTooLarge,
    BodyTooLarge,
}

fn find(haystack: &[u8], needle: &[u8]) -> Option<usize> {
    haystack.windows(needle.len()).position(|w| w == needle)
}

/// Ok(None): the request is not complete yet.
/// Ok(Some((request, n))): a complete request that used the first `n` bytes of `buf`.
fn parse_request(buf: &[u8]) -> Result<Option<(Request, usize)>, HttpError> {
    let Some(head_end) = find(buf, b"\r\n\r\n") else {
        return if buf.len() > MAX_HEAD {
            Err(HttpError::HeadTooLarge)
        } else {
            Ok(None)
        };
    };
    if head_end > MAX_HEAD {
        return Err(HttpError::HeadTooLarge);
    }
    let head = std::str::from_utf8(&buf[..head_end])
        .map_err(|_| HttpError::BadRequest("head is not UTF-8"))?;
    let mut lines = head.split("\r\n");

    // Request line: METHOD SP PATH SP VERSION
    let mut parts = lines.next().unwrap_or("").split(' ');
    let (Some(method), Some(path), Some(version), None) =
        (parts.next(), parts.next(), parts.next(), parts.next())
    else {
        return Err(HttpError::BadRequest("bad request line"));
    };
    if version != "HTTP/1.1" && version != "HTTP/1.0" {
        return Err(HttpError::BadRequest("unsupported HTTP version"));
    }

    // Headers: NAME ":" VALUE, names are case-insensitive
    let mut headers = Vec::new();
    for line in lines {
        let (name, value) = line
            .split_once(':')
            .ok_or(HttpError::BadRequest("header without ':'"))?;
        if name.is_empty() || name.contains([' ', '\t']) {
            return Err(HttpError::BadRequest("bad header name"));
        }
        headers.push((name.to_ascii_lowercase(), value.trim().to_string()));
    }
    let mut req = Request {
        method: method.to_string(),
        path: path.to_string(),
        version: version.to_string(),
        headers,
        body: Vec::new(),
    };

    // Body: only Content-Length is supported here.
    if req.header("transfer-encoding").is_some() {
        return Err(HttpError::BadRequest("Transfer-Encoding is not supported"));
    }
    let lengths = req
        .headers
        .iter()
        .filter(|(n, _)| n == "content-length")
        .count();
    if lengths > 1 {
        return Err(HttpError::BadRequest("more than one Content-Length"));
    }
    let body_len: usize = match req.header("content-length") {
        None => 0,
        Some(v) => v
            .parse()
            .map_err(|_| HttpError::BadRequest("bad Content-Length"))?,
    };
    if body_len > MAX_BODY {
        return Err(HttpError::BodyTooLarge);
    }
    let body_start = head_end + 4;
    if buf.len() < body_start + body_len {
        return Ok(None); // the body has not fully arrived
    }
    req.body = buf[body_start..body_start + body_len].to_vec();
    Ok(Some((req, body_start + body_len)))
}
```

**What happens:**

1. `find` looks for the empty line `\r\n\r\n`. Until it appears, the head is incomplete; but if more
   than 8 KiB arrived without it, the client gets an error instead of unlimited buffering.
2. The request line must split into exactly three parts: the pattern `(Some, Some, Some, None)` in the
   `let ... else` rejects two parts and four parts alike.
3. Header names are stored in lower case, so `header("content-length")` finds them however the client
   wrote them. `trim()` removes the optional spaces around the value.
4. The body rules come last: no `Transfer-Encoding`, at most one `Content-Length`, a valid number, at
   most 1 MiB. Then, if the body is not complete yet, the result is "not yet". (One leniency remains:
   `str::parse::<usize>` also accepts a leading `+`, so `Content-Length: +5` passes. RFC 9110 allows
   only digits; a production parser also checks `v.bytes().all(|b| b.is_ascii_digit())`.)
5. On success, the function returns how many bytes it used. Whatever follows in the buffer belongs to the
   **next** request (pipelining), and the caller must keep it.

## 4. Keep-alive: a loop over requests

HTTP/1.0 closed the connection after every response, which costs a handshake (one RTT, Lesson 15.3) per
request. HTTP/1.1 keeps the connection open by default: a **persistent connection**. The client or the
server ends it with the header `Connection: close`. The connection task is therefore a loop: parse every
complete request in the buffer, and only when there is none, read more bytes. Part 2 of the file:

```rust
fn route(req: &Request) -> (u16, &'static str, String) {
    match (req.method.as_str(), req.path.as_str()) {
        ("GET", "/") => (200, "OK", "hello\n".to_string()),
        ("POST", "/echo") => (200, "OK", String::from_utf8_lossy(&req.body).into_owned()),
        (_, "/" | "/echo") => (
            405,
            "Method Not Allowed",
            "method not allowed\n".to_string(),
        ),
        _ => (404, "Not Found", "not found\n".to_string()),
    }
}

fn response(status: u16, reason: &str, body: &str, close: bool) -> Vec<u8> {
    let connection = if close { "Connection: close\r\n" } else { "" };
    format!(
        "HTTP/1.1 {status} {reason}\r\nContent-Type: text/plain\r\nContent-Length: {}\r\n{connection}\r\n{body}",
        body.len()
    )
    .into_bytes()
}

fn wants_close(req: &Request) -> bool {
    match req.header("connection") {
        Some(v) if v.eq_ignore_ascii_case("close") => true,
        Some(v) if v.eq_ignore_ascii_case("keep-alive") => false,
        _ => req.version == "HTTP/1.0", // HTTP/1.1 keeps connections open by default
    }
}

async fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let mut buf = BytesMut::with_capacity(4096);
    loop {
        // First serve every complete request that is already in the buffer.
        match parse_request(&buf) {
            Ok(Some((req, used))) => {
                buf.advance(used);
                let close = wants_close(&req);
                let (status, reason, body) = route(&req);
                println!("{} {} -> {status}", req.method, req.path);
                stream
                    .write_all(&response(status, reason, &body, close))
                    .await?;
                if close {
                    return Ok(());
                }
                continue;
            }
            Ok(None) => {} // need more bytes
            Err(e) => {
                let (status, reason) = match e {
                    HttpError::BadRequest(_) => (400, "Bad Request"),
                    HttpError::HeadTooLarge => (431, "Request Header Fields Too Large"),
                    HttpError::BodyTooLarge => (413, "Content Too Large"),
                };
                println!("bad request: {e:?} -> {status}");
                let body = format!("{reason}\n");
                stream
                    .write_all(&response(status, reason, &body, true))
                    .await?;
                return Ok(());
            }
        }
        // Then read more, but do not wait forever for a slow or silent client.
        match timeout(Duration::from_secs(10), stream.read_buf(&mut buf)).await {
            Ok(Ok(0)) => return Ok(()), // the client closed the connection
            Ok(Ok(_)) => {}
            Ok(Err(e)) => return Err(e),
            Err(_) => return Ok(()), // idle for 10 seconds: close
        }
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7300").await?;
    println!("listening on http://127.0.0.1:7300");
    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(handle(stream));
    }
}
```

**What happens:**

1. `route` maps method and path to a status, a reason and a body. A known path with the wrong method gets
   `405`, an unknown path `404`.
2. `response` always sends `Content-Length`, so the client knows where our body ends and can reuse the
   connection.
3. In `handle`, `stream.read_buf(&mut buf)` appends whatever arrived to the end of the `BytesMut`
   (it grows as needed); `buf.advance(used)` removes a served request from the front. Leftover bytes
   of the next request stay in the buffer and are parsed in the next loop turn, *before* reading again.
4. A parse error is answered with the matching status (`400`, `431` or `413`) and `Connection: close`,
   and then the connection is closed: after a framing error we cannot know where the next request
   would start.
5. The 10-second timeout around each read closes idle keep-alive connections and slow clients.

Run it with `cargo run --bin http_server` and, in a second terminal, ask for two URLs in one `curl`
command. `-v` shows what curl sends (`>`), receives (`<`) and does (`*`):

```bash
curl -v http://127.0.0.1:7300/ http://127.0.0.1:7300/missing
```

```text
*   Trying 127.0.0.1:7300...
* Connected to 127.0.0.1 (127.0.0.1) port 7300
> GET / HTTP/1.1
> Host: 127.0.0.1:7300
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Content-Length: 6
< 
{ [6 bytes data]
* Connection #0 to host 127.0.0.1 left intact
hello
* Found bundle for host: 0x77bf040000 [serially]
* Can not multiplex, even if we wanted to
* Re-using existing connection with host 127.0.0.1
> GET /missing HTTP/1.1
> Host: 127.0.0.1:7300
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 404 Not Found
< Content-Type: text/plain
< Content-Length: 10
< 
{ [10 bytes data]
* Connection #0 to host 127.0.0.1 left intact
not found
```

`Connection #0 ... left intact` and `Re-using existing connection` show keep-alive at work: both
requests travelled over one TCP connection. ("Can not multiplex" means that HTTP/1.1 cannot send two
requests *at the same time* on one connection, unlike HTTP/2; Lesson 15.13.)

A body with `POST`:

```bash
curl -i -X POST --data 'hi there' http://127.0.0.1:7300/echo
```

```text
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 8

hi there
```

And **pipelining** by hand: `nc` sends two requests in one write, the second with `Connection: close`.
Both arrive in the same read, so the server answers both from one buffer, in order, and closes:

```bash
printf 'GET / HTTP/1.1\r\nHost: x\r\n\r\nGET /nope HTTP/1.1\r\nHost: x\r\nConnection: close\r\n\r\n' | nc 127.0.0.1 7300
```

```text
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 6

hello
HTTP/1.1 404 Not Found
Content-Type: text/plain
Content-Length: 10
Connection: close

not found
```

Garbage (`printf 'HELLO\r\n\r\n' | nc 127.0.0.1 7300`) gets `HTTP/1.1 400 Bad Request` with
`Connection: close`, and the server's terminal shows `bad request: BadRequest("bad request line") -> 400`.

## 5. Testing the parser

Part 3 of the file. Because `parse_request` is a pure function, its tests need no sockets:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn parse(text: &str) -> Result<Option<(Request, usize)>, HttpError> {
        parse_request(text.as_bytes())
    }

    #[test]
    fn parses_a_get_request() {
        let (req, used) = parse("GET /a HTTP/1.1\r\nHost: x\r\n\r\n")
            .unwrap()
            .unwrap();
        assert_eq!((req.method.as_str(), req.path.as_str()), ("GET", "/a"));
        assert_eq!(req.header("host"), Some("x"));
        assert_eq!(used, 28);
    }

    #[test]
    fn waits_for_head_and_body() {
        assert_eq!(parse("GET / HTTP/1.1\r\nHost: x\r\n"), Ok(None));
        assert_eq!(
            parse("POST /echo HTTP/1.1\r\nHost: x\r\nContent-Length: 5\r\n\r\nhel"),
            Ok(None)
        );
    }

    #[test]
    fn two_requests_in_one_buffer() {
        let text = "GET /1 HTTP/1.1\r\nHost: x\r\n\r\nGET /2 HTTP/1.1\r\nHost: x\r\n\r\n";
        let (first, used) = parse(text).unwrap().unwrap();
        let (second, _) = parse(&text[used..]).unwrap().unwrap();
        assert_eq!((first.path.as_str(), second.path.as_str()), ("/1", "/2"));
    }

    #[test]
    fn rejects_bad_input() {
        let bad = [
            "GET /\r\n\r\n",
            "GET / HTTP/2\r\n\r\n",
            "GET / HTTP/1.1\r\nHost: x\r\nno colon\r\n\r\n",
            "POST / HTTP/1.1\r\nHost: x\r\nContent-Length: 1\r\nContent-Length: 2\r\n\r\nx",
            "POST / HTTP/1.1\r\nHost: x\r\nTransfer-Encoding: chunked\r\n\r\n",
        ];
        for text in bad {
            assert!(
                matches!(parse(text), Err(HttpError::BadRequest(_))),
                "{text:?}"
            );
        }
        let huge = format!(
            "GET / HTTP/1.1\r\nHost: x\r\nX: {}\r\n\r\n",
            "a".repeat(10_000)
        );
        assert_eq!(parse(&huge).unwrap_err(), HttpError::HeadTooLarge);
    }
}
```

`cargo test --bin http_server` reports `test result: ok. 4 passed`.

> 🧪 **Testing:** the tests follow the three outcomes of Lesson 15.7: complete requests (and the exact
> number of bytes used, 28), incomplete heads and bodies, and a list of invalid inputs. The test with two
> requests in one buffer protects pipelining, which is easy to break by throwing away leftover bytes.

> ⚡ **Performance:** this parser copies strings and searches for `\r\n\r\n` from the start of the buffer
> after every read, which is fine for learning. Production parsers such as the `httparse` crate (used
> inside `hyper`, Lesson 15.9) parse without copying, directly from the buffer.

## Common mistakes

> ⚠️ **Common mistake: one `read` = one request.** The server of Lesson 06.11 did exactly that. It breaks
> when a request arrives in two pieces (big headers, slow networks) and when two requests arrive in one
> read (pipelining). Always loop: parse what you have, read more only when the parser says "not yet".

> ⚠️ **Common mistake: no `Content-Length` in responses on a keep-alive connection.** Without it (and
> without chunked coding) the client cannot know where your body ends and waits for the connection to
> close, which never happens. `curl` then seems to hang.

## Try it yourself

1. Run `curl -v --http1.0 http://127.0.0.1:7300/`. curl sends `HTTP/1.0`, and our server closes the
   connection after the answer (look for `Closing connection` in curl's output).
2. Run `curl -i -X DELETE http://127.0.0.1:7300/`. The answer is `HTTP/1.1 405 Method Not Allowed`.
3. Open a connection with `nc 127.0.0.1 7300`, type `GET / HTTP/1.1`, Enter, `Host: x`, Enter, Enter.
   (`nc` sends `\n`, not `\r\n`, so the server waits.) Wait 10 seconds: the idle timeout closes the
   connection.
4. Send a request with `curl -H 'X-Big: '"$(printf 'a%.0s' {1..9000})" http://127.0.0.1:7300/` and get
   `431 Request Header Fields Too Large`.

## Exercises

### Exercise 1 — Show the headers (easy)

Add a route `GET /headers` that answers with all request headers, one `name: value` per line, in the order
they arrived. Try it with `curl http://127.0.0.1:7300/headers`.

<details>
<summary>Hint</summary>

The route gets the whole `Request`. Turn each `(name, value)` pair into a line with `format!` and join the
lines with `concat()`.

</details>

<details>
<summary>Answer</summary>

Add this arm to `route`, after the `("GET", "/")` arm:

```rust
        ("GET", "/headers") => {
            let lines: Vec<String> = req
                .headers
                .iter()
                .map(|(n, v)| format!("{n}: {v}\n"))
                .collect();
            (200, "OK", lines.concat())
        }
```

`curl http://127.0.0.1:7300/headers` prints:

```text
host: 127.0.0.1:7300
user-agent: curl/8.7.1
accept: */*
```

The names are in lower case because the parser stores them that way. The position of the new arm does
not matter here, because no other pattern matches `/headers`; keeping all `GET` routes together is only
for readability.

</details>

### Exercise 2 — `Host` is required, with a test (medium)

RFC 9112 says that a server must answer `400 Bad Request` to an HTTP/1.1 request without a `Host` header.
HTTP/1.0 requests may omit it. Add this rule to `parse_request` and a test for both cases. Do the existing
tests still pass?

<details>
<summary>Hint</summary>

Check right after the header loop, before the `Request` is built. `headers.iter().any(...)` tells you
whether a name is present.

</details>

<details>
<summary>Answer</summary>

Insert in `parse_request`, just before `let mut req = Request {`:

```rust
    if version == "HTTP/1.1" && !headers.iter().any(|(n, _)| n == "host") {
        return Err(HttpError::BadRequest("missing Host header"));
    }
```

and add to `mod tests`:

```rust
    #[test]
    fn host_is_required_in_http_1_1_only() {
        let err = parse("GET / HTTP/1.1\r\n\r\n").unwrap_err();
        assert_eq!(err, HttpError::BadRequest("missing Host header"));
        assert!(parse("GET / HTTP/1.0\r\n\r\n").unwrap().is_some());
    }
```

`cargo test --bin http_server` reports `5 passed`. The existing tests still pass because every HTTP/1.1
request in them already has `Host: x`. That was deliberate: test inputs should be valid in every way
except the one the test is about. Otherwise a new rule makes old tests fail, or, worse, makes them pass
for the wrong reason (an "invalid" input rejected because of the missing `Host` instead of the bug it was
written for).

</details>

### Exercise 3 — Decode a chunked body, with tests (hard)

Write `src/bin/chunked.rs` with `fn decode_chunked(buf: &[u8]) -> Result<Option<(Vec<u8>, usize)>,
ChunkError>`, the same three-outcome shape as `parse_request`, for the chunked format of section 2. Accept
hexadecimal sizes in upper or lower case, ignore *chunk extensions* (`;name=value` after the size), refuse
bodies over 1 MiB, refuse a chunk that is not followed by CRLF, and do not support trailers (header lines
after the last chunk). Test the example from section 2, every incomplete prefix of it, and bad input.

<details>
<summary>Hint</summary>

Keep a position `pos` in the buffer. In a loop: find the next CRLF from `pos`, parse the size with
`usize::from_str_radix(hex, 16)`, then use `buf.get(pos..pos + size + 2)`, which returns `None` instead of
panicking when the data has not arrived yet. Check the limit *before* waiting for the data, as in Lesson
15.7: `size > MAX_BODY - body.len()` cannot overflow.

</details>

<details>
<summary>Answer</summary>

```rust
const MAX_BODY: usize = 1024 * 1024;

#[derive(Debug, PartialEq)]
enum ChunkError {
    Bad(&'static str),
    TooLarge,
}

/// Decodes a chunked body at the start of `buf`.
/// Ok(None): not complete yet. Ok(Some((body, n))): done, `n` bytes used.
fn decode_chunked(buf: &[u8]) -> Result<Option<(Vec<u8>, usize)>, ChunkError> {
    let mut body = Vec::new();
    let mut pos = 0;
    loop {
        // The size line: hex digits, maybe ";extension", then CRLF.
        let Some(line_len) = buf[pos..].windows(2).position(|w| w == b"\r\n") else {
            return Ok(None);
        };
        let line = std::str::from_utf8(&buf[pos..pos + line_len])
            .map_err(|_| ChunkError::Bad("size line is not text"))?;
        let hex = line.split(';').next().unwrap_or("").trim();
        let size = usize::from_str_radix(hex, 16).map_err(|_| ChunkError::Bad("bad chunk size"))?;
        pos += line_len + 2;
        if size > MAX_BODY - body.len() {
            return Err(ChunkError::TooLarge);
        }
        if size == 0 {
            // The last chunk is followed by an empty line (we do not support trailers).
            return match buf.get(pos..pos + 2) {
                None => Ok(None),
                Some(b"\r\n") => Ok(Some((body, pos + 2))),
                Some(_) => Err(ChunkError::Bad("trailers are not supported")),
            };
        }
        // The data, followed by CRLF.
        let Some(chunk) = buf.get(pos..pos + size + 2) else {
            return Ok(None);
        };
        if &chunk[size..] != b"\r\n" {
            return Err(ChunkError::Bad("chunk is not followed by CRLF"));
        }
        body.extend_from_slice(&chunk[..size]);
        pos += size + 2;
    }
}

fn main() {
    let wire = b"4\r\nWiki\r\n5\r\npedia\r\n0\r\n\r\nGET /next";
    let (body, used) = decode_chunked(wire).unwrap().unwrap();
    println!("body: {:?}", String::from_utf8_lossy(&body));
    println!("used {used} of {} bytes", wire.len());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn decodes_the_example() {
        let wire = b"4\r\nWiki\r\n5\r\npedia\r\n0\r\n\r\n";
        assert_eq!(
            decode_chunked(wire),
            Ok(Some((b"Wikipedia".to_vec(), wire.len())))
        );
    }

    #[test]
    fn every_prefix_is_incomplete() {
        let wire = b"4\r\nWiki\r\n5\r\npedia\r\n0\r\n\r\n";
        for end in 0..wire.len() {
            assert_eq!(
                decode_chunked(&wire[..end]),
                Ok(None),
                "prefix of {end} bytes"
            );
        }
    }

    #[test]
    fn rejects_bad_chunks() {
        assert_eq!(
            decode_chunked(b"zz\r\n"),
            Err(ChunkError::Bad("bad chunk size"))
        );
        assert_eq!(
            decode_chunked(b"2\r\nabXY"),
            Err(ChunkError::Bad("chunk is not followed by CRLF"))
        );
        assert_eq!(decode_chunked(b"FFFFFFFF\r\n"), Err(ChunkError::TooLarge));
    }
}
```

`cargo run --bin chunked` prints `body: "Wikipedia"` and `used 24 of 33 bytes`, and
`cargo test --bin chunked` reports `3 passed`. Why it works: every slice access uses `get`, so a short
buffer means "not yet" and never a panic; the test over all prefixes proves it for every possible split
point. The 9 unused bytes (`GET /next`) belong to the next request, exactly as in `parse_request`.

</details>

## Quiz

1. How does a server know where the head of an HTTP/1.1 request ends, and where its body ends?

<details>
<summary>Answer</summary>

The head ends at the first empty line (`\r\n\r\n`). The body is exactly `Content-Length` bytes long, or
it is chunked (`Transfer-Encoding: chunked`) and ends with a chunk of size 0; with neither header there
is no body.

</details>

2. Why is a request with both `Content-Length` and `Transfer-Encoding` dangerous?

<details>
<summary>Answer</summary>

Two programs on the path may use different headers to find the end, and so disagree about where the
request ends. An attacker can then hide a second request (request smuggling). Servers must reject it.

</details>

3. What does keep-alive save, and how is a persistent connection ended?

<details>
<summary>Answer</summary>

It saves a TCP handshake (and with HTTPS also a TLS handshake) for each request after the first. Either
side ends it by sending `Connection: close` (or by simply closing, for example after an idle timeout).

</details>

4. A client pipelines two requests. Why must the server never throw away the bytes left in its buffer
   after parsing the first one?

<details>
<summary>Answer</summary>

They are the beginning (or all) of the second request. Throwing them away would lose that request, and
the client would wait forever for its answer.

</details>

5. Which status codes does our server send for a malformed request, a 20 KB header, and a 5 MB body?

<details>
<summary>Answer</summary>

`400 Bad Request`, `431 Request Header Fields Too Large`, and `413 Content Too Large`.

</details>

## Summary

- An HTTP/1.1 message is a start line, header lines, an empty line and an optional body; head lines end
  with CRLF, header names are case-insensitive, and HTTP/1.1 requests need `Host`.
- The body ends after `Content-Length` bytes, or at the zero-size chunk of a chunked body. Conflicting
  length headers are rejected to prevent request smuggling.
- HTTP/1.1 connections are persistent by default; `Connection: close` ends them. Pipelined requests share
  one buffer, so a server parses in a loop and keeps leftover bytes.
- A good parser is a pure function with three outcomes and hard limits on head and body size; the
  connection loop adds timeouts and answers errors with `400`, `413` or `431` before closing.
- `curl -v` shows what really goes over the wire, including connection reuse; `nc` lets you send any
  bytes you like.

## Sources and further reading

- [RFC 9112 — HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc9112) — the message syntax: sections 2–7 (format, request line, body length, chunked coding) and 9 (connections, keep-alive, pipelining).
- [RFC 9110 — HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110) — methods, status codes and header meanings shared by all HTTP versions.
- [MDN — An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview) — a friendly introduction with diagrams.
- [PortSwigger — HTTP request smuggling](https://portswigger.net/web-security/request-smuggling) — how the attack works in practice.
- [`httparse` crate](https://docs.rs/httparse/latest/httparse/) — the fast, zero-copy HTTP/1 parser used by `hyper`.

---

[🏠 Start](../START-HERE.md) · [← 15.7 Designing a protocol: framing and parsing](07-designing-a-protocol.md) · [📋 Phase 15](README.md) · [15.9 HTTP with hyper and reqwest →](09-http-with-hyper-and-reqwest.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
