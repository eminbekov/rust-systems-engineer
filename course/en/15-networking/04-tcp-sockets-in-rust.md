<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.3 TCP and UDP](03-tcp-and-udp.md) · [📋 Phase 15](README.md) · [15.5 UDP sockets in Rust →](05-udp-sockets-in-rust.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/04-tcp-sockets-in-rust.md) <!-- nav -->

# Lesson 15.4 — TCP sockets in Rust

> **Phase 15 — Networking** · Lesson 4 of 16 · ⏱ about 135 minutes

## What you will learn

- Which system calls are behind `TcpListener` and `TcpStream`, and the difference between a listening socket and a connected socket.
- How to write a server that serves many clients at once with one thread per connection.
- Why one `write` does not mean one `read`, how to detect the end of a stream, and how to close only one direction with `shutdown`.
- How to use timeouts, port `0` and the right bind address, and what the common socket errors mean.
- How to test network code with real sockets on `127.0.0.1:0`.

## Before you start

- You finished [Lesson 15.3 — TCP and UDP](03-tcp-and-udp.md).
- You know system calls and file descriptors from [Lesson 13.3](../13-operating-systems/03-system-calls.md) and [Lesson 13.4](../13-operating-systems/04-files-and-file-descriptors.md), threads from Phase 06 and Phase 14, and atomics from [Lesson 14.5](../14-concurrency-and-async/05-atomics.md) (only for the last exercise).
- Examples go into `code/p15-net/src/bin/`. You need two terminals: one for the server, one for the client.

## New words

| Word | Meaning |
|---|---|
| **listening socket** | A socket that only accepts new connections. In Rust it is a `TcpListener`. |
| **connected socket** | A socket for one connection, through which data flows. In Rust it is a `TcpStream`. |
| **bind** | To attach a socket to a local address and port. |
| **accept** | To take the next finished connection from the backlog of a listening socket. |
| **backlog** | The kernel queue of connections that finished the handshake but were not accepted yet. |
| **EOF** (End Of File) | The end of a stream. For a socket: the other side has closed its sending direction. `read` then returns `Ok(0)`. |
| **half-close** | Closing only one direction of a connection with `shutdown(Shutdown::Write)`: "I will send nothing more, but I still read". |
| **echo server** | A server that sends back every byte it receives. The "hello world" of network programming. |
| **timeout** | A time limit after which a waiting operation gives up with an error. |
| **Nagle's algorithm** | A TCP feature that holds back small writes for a moment to send them together in fewer packets. `set_nodelay(true)` turns it off. |
| **TCP / UDP** (Transmission Control Protocol / User Datagram Protocol) | The byte-stream and datagram protocols of Lesson 15.3. This lesson uses TCP; Lesson 15.5 uses UDP. |
| **FIN / RST** (finish / reset) | The TCP flags from Lesson 15.3: FIN says "I am done sending" (a normal close), RST aborts the connection at once. |

## 1. What `std::net` does underneath

From Lesson 06.11 you know `TcpListener` and `TcpStream`. Each of their methods is a thin wrapper
around one or two system calls (Lesson 13.3). A socket is a file descriptor (Lesson 13.4), so after
it is created you use it with the usual `read`, `write` and `close`:

| Rust | System calls | What the kernel does |
|---|---|---|
| `TcpListener::bind(addr)` | `socket`, `bind`, `listen` | creates a socket, attaches it to the address, starts accepting handshakes into the backlog |
| `listener.accept()` / `incoming()` | `accept` | takes one finished connection and returns a **new** socket for it |
| `TcpStream::connect(addr)` | `socket`, `connect` | creates a socket and performs the three-way handshake |
| `stream.read(&mut buf)` | `read` (or `recv`) | copies bytes that have arrived from the kernel buffer into `buf`; waits if there are none |
| `stream.write(&buf)` | `write` (or `send`) | copies bytes into the kernel's send buffer; TCP sends them when it can |
| `stream.shutdown(Write)` | `shutdown` | sends a FIN, but keeps the socket open for reading |
| `drop(stream)` | `close` | closes the socket: FIN (or RST if unread data is left) |

The most important line is `accept`. A server has **one** listening socket, and `accept` creates a
**separate connected socket for every client**:

```text
                   ┌────────────────────┐
  clients ──SYN──► │ listening socket   │  127.0.0.1:7000, state LISTEN
                   │ backlog: [c3] [c4] │  (handshakes done, not accepted yet)
                   └─────────┬──────────┘
                     accept()│ returns a new socket each time
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
     TcpStream c1      TcpStream c2      (c3, c4 wait in the backlog)
     ESTABLISHED       ESTABLISHED
```

Every call here *blocks* (waits) by default: `accept` waits for a client, `read` waits for data, and
`write` waits when the send buffer is full. With threads, a blocked thread is fine: other threads keep
working. Lesson 15.6 shows the other way, async I/O with Tokio.

## 2. An echo server for many clients

Save as `src/bin/echo_server.rs`:

```rust
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;

/// Sends every byte it receives back to the client, until the client closes.
fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let peer = stream.peer_addr()?;
    let mut buf = [0u8; 1024];
    loop {
        let n = stream.read(&mut buf)?;
        if n == 0 {
            println!("{peer}: closed");
            return Ok(());
        }
        println!("{peer}: {n} bytes");
        stream.write_all(&buf[..n])?;
    }
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7000")?;
    println!("echo server on {}", listener.local_addr()?);
    for stream in listener.incoming() {
        let stream = stream?;
        thread::spawn(move || {
            if let Err(e) = handle(stream) {
                eprintln!("connection error: {e}");
            }
        });
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::net::Shutdown;

    #[test]
    fn echoes_bytes_back() {
        let listener = TcpListener::bind("127.0.0.1:0").unwrap();
        let addr = listener.local_addr().unwrap();
        thread::spawn(move || {
            let (stream, _) = listener.accept().unwrap();
            handle(stream).unwrap();
        });

        let mut client = TcpStream::connect(addr).unwrap();
        client.write_all(b"ping").unwrap();
        client.shutdown(Shutdown::Write).unwrap(); // "I am done sending"
        let mut reply = Vec::new();
        client.read_to_end(&mut reply).unwrap();
        assert_eq!(reply, b"ping");
    }
}
```

**What happens:**

1. `main` binds the listening socket and loops over `incoming()`, which calls `accept` again and again.
   Each accepted `TcpStream` moves into its own thread, so a slow client does not block the others.
2. `handle` reads up to 1024 bytes. `read` returns how many bytes it put into `buf`, which may be fewer
   than 1024. **`Ok(0)` means EOF**: the client has closed its sending direction, and no more data will
   ever come. That is the normal way a loop over a socket ends.
3. `write_all` sends the same bytes back. It loops internally until every byte is written, because a
   single `write` may accept only part of the buffer.
4. Errors in one connection are printed with `eprintln!` and end only that thread, not the server.

Run the server in one terminal with `cargo run --bin echo_server`. In the second terminal, connect with
`nc 127.0.0.1 7000`, type `hello` and Enter, then `how are you?` and Enter (each line comes back),
and press Ctrl+C. The server prints (your port number will differ):

```text
echo server on 127.0.0.1:7000
127.0.0.1:53912: 6 bytes
127.0.0.1:53912: 13 bytes
127.0.0.1:53912: closed
```

Open two `nc` windows at the same time: both are served, each by its own thread.

> 🧪 **Testing:** the test binds to `127.0.0.1:0`, so the operating system picks a free port and tests
> can run in parallel without fighting over port 7000. It runs the real `handle` in a thread and talks
> to it through a real socket. `shutdown(Shutdown::Write)` sends a FIN, so `handle` sees EOF and
> finishes; then `read_to_end` on the client sees EOF too. Run it with `cargo test --bin echo_server`:
> `test result: ok. 1 passed`.

## 3. A stream has no message borders

Lesson 15.3 said that TCP keeps bytes but not message borders. Here it is in code. This program is
self-contained: a client thread writes, and the main thread reads. Save as `src/bin/chunks.rs`:

```rust
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;
use std::time::Duration;

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:0")?;
    let addr = listener.local_addr()?;

    let client = thread::spawn(move || -> std::io::Result<()> {
        let mut stream = TcpStream::connect(addr)?;
        stream.write_all(b"hel")?;
        stream.write_all(b"lo\nwor")?;
        thread::sleep(Duration::from_millis(100));
        stream.write_all(b"ld\n")?;
        thread::sleep(Duration::from_millis(100));
        stream.write_all(&vec![b'x'; 300_000])?; // one big write
        Ok(()) // the stream is dropped here: the client closes
    });

    let (mut stream, _) = listener.accept()?;
    thread::sleep(Duration::from_millis(50)); // let the first two writes arrive
    let mut buf = [0u8; 65536];
    let mut total = 0;
    loop {
        let n = stream.read(&mut buf)?;
        if n == 0 {
            break;
        }
        total += n;
        if n < 16 {
            println!(
                "read {n:>5} bytes: {:?}",
                String::from_utf8_lossy(&buf[..n])
            );
        } else {
            println!("read {n:>5} bytes");
        }
    }
    println!("end of stream after {total} bytes");
    client.join().unwrap()
}
```

`cargo run --bin chunks` printed this on the Mac (the sizes of the big reads change from run to run):

```text
read     9 bytes: "hello\nwor"
read     3 bytes: "ld\n"
read 65328 bytes
read 65536 bytes
read 32456 bytes
read 65536 bytes
read 65536 bytes
read  5608 bytes
end of stream after 300012 bytes
```

**What happens:**

1. The writes `hel` and `lo\nwor` arrived before the first `read`, so **two writes came out as one
   read**, and that read ends in the middle of the second line (`wor`).
2. One `write_all` of 300,000 bytes came out as **six reads** of different sizes, depending on how fast
   the kernel buffers filled and emptied.
3. The total is exact: 3 + 6 + 3 + 300,000 = 300,012 bytes, in order. TCP keeps the bytes; your program
   must find the message borders itself. Lesson 15.7 is about exactly this.

For text protocols with one message per line, `BufReader` does the work: `read_line` and `lines()`
read as many chunks as needed and give you whole lines. Exercise 2 uses it.

## 4. Closing, timeouts and errors

**Closing.** Dropping a `TcpStream` closes both directions. Sometimes a client must say "my request is
complete" and *then* read the answer. `stream.shutdown(Shutdown::Write)` does that: it sends a FIN
(the server's `read` returns 0) but the client can still read. The test in section 2 uses it.

**Timeouts.** By default `read` waits forever. A client that never sends anything would keep a server
thread busy forever. Set a limit. Save as `src/bin/timeouts.rs`:

```rust
use std::io::Read;
use std::net::{TcpListener, TcpStream};
use std::time::{Duration, Instant};

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:0")?;
    let mut stream = TcpStream::connect(listener.local_addr()?)?;
    stream.set_read_timeout(Some(Duration::from_millis(200)))?;

    let start = Instant::now();
    let mut buf = [0u8; 16];
    match stream.read(&mut buf) {
        Ok(n) => println!("read {n} bytes"),
        Err(e) => println!("after {:?}: {e} (kind: {:?})", start.elapsed(), e.kind()),
    }

    // A second listener on the same address fails.
    match TcpListener::bind(listener.local_addr()?) {
        Ok(_) => println!("bound twice?"),
        Err(e) => println!("second bind: {e} (kind: {:?})", e.kind()),
    }
    Ok(())
}
```

```text
after 201.028042ms: Resource temporarily unavailable (os error 35) (kind: WouldBlock)
second bind: Address already in use (os error 48) (kind: AddrInUse)
```

**What happens:**

1. Nobody writes to the stream, so `read` waits, and after 200 ms returns an error. On Unix the kind is
   `WouldBlock` (on Windows it would be `TimedOut`), so check for both.
2. `set_write_timeout` does the same for writes, and `TcpStream::connect_timeout(&addr, duration)`
   limits the handshake.
3. Only one socket may listen on an address and port, so the second `bind` fails with `AddrInUse`.
   The numbers in `os error 35` and `os error 48` are macOS error codes; on Linux the same errors are
   `os error 11` and `os error 98`.

The errors you will meet most often:

| `ErrorKind` | Usual meaning |
|---|---|
| `ConnectionRefused` | nothing listens on that address and port (RST, Lesson 15.3) |
| `AddrInUse` | another program (or an old copy of yours) already listens on that port |
| `WouldBlock` / `TimedOut` | a timeout you set has expired |
| `ConnectionReset` | the other side aborted the connection with RST |
| `BrokenPipe` | you wrote to a connection that the other side has already closed |
| `UnexpectedEof` | `read_exact` needed more bytes, but the stream ended |

**Bind address.** `127.0.0.1:7000` accepts only clients on the same machine. `0.0.0.0:7000` accepts
connections on every interface, so other computers in the network can connect too. Inside a container
a server must bind to `0.0.0.0`, otherwise the published port cannot reach it.

> 🔒 **Security:** a server on `0.0.0.0` is reachable by everybody who can reach your machine. During
> development bind to `127.0.0.1`. Always set read timeouts and a limit on the number of connections:
> without them, a few clients that connect and send nothing can use up all your threads. Exercise 3
> builds such a limit.

> ⚡ **Performance:** TCP's **Nagle's algorithm** delays small writes a little to send them together.
> For request/response protocols with tiny messages this can add tens of milliseconds of delay.
> `stream.set_nodelay(true)` turns it off. Better still, collect a whole message in a buffer (or wrap
> the stream in a `BufWriter`) and write it at once.

## Common mistakes

> ⚠️ **Common mistake: ignoring the return value of `read`.** `stream.read(&mut buf)?;` followed by
> using the whole `buf` treats old or zero bytes as data. Always use `&buf[..n]`, and treat `n == 0` as
> the end of the stream. `cargo clippy` reports a `read` whose result is thrown away as an error:
>
> ```text
> error: read amount is not handled
>   = help: use `Read::read_exact` instead, or handle partial reads
> ```

> ⚠️ **Common mistake: `write` instead of `write_all`.** `write` may write only part of the buffer and
> returns how much it wrote. Code that ignores this loses data under load. Use `write_all`.

> ⚠️ **Common mistake: both sides wait for each other.** A client sends a request and then calls
> `read_to_end` to get the whole answer, while the server waits for EOF to know the request is complete.
> Both wait forever. Either the client calls `shutdown(Shutdown::Write)` after sending, or the protocol
> marks where a message ends (a newline, a length; Lesson 15.7).

> ⚠️ **Common mistake: forgetting to flush a `BufWriter`.** Data written into a `BufWriter<TcpStream>`
> stays in memory until the buffer is full, `flush()` is called, or the writer is dropped. A client
> that waits for an answer to an unflushed request waits forever.

## Try it yourself

1. Start `echo_server` and connect with two `nc` windows at once. Type in both and watch the server log
   show two different client ports.
2. Stop the server with Ctrl+C and start it again immediately. It starts without `AddrInUse`, because
   Rust's `TcpListener::bind` sets the option `SO_REUSEADDR` on Unix, which allows binding while old
   connections are still in TIME-WAIT.
3. Start the server twice in two terminals. The second one fails with
   `Error: Os { code: 48, kind: AddrInUse, message: "Address already in use" }`.
4. Run `cargo run --bin chunks` three times and compare the sizes of the big reads.

## Exercises

### Exercise 1 — Count the bytes (easy)

Change `handle` in `echo_server.rs` so that, when the client closes, it prints how many bytes the
connection carried in total, for example `127.0.0.1:53912: closed after 19 bytes`.

<details>
<summary>Hint</summary>

Keep a `let mut total = 0;` before the loop and add `n` after each read.

</details>

<details>
<summary>Answer</summary>

The rest of `echo_server.rs` stays the same:

```rust
fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let peer = stream.peer_addr()?;
    let mut buf = [0u8; 1024];
    let mut total = 0;
    loop {
        let n = stream.read(&mut buf)?;
        if n == 0 {
            println!("{peer}: closed after {total} bytes");
            return Ok(());
        }
        total += n;
        stream.write_all(&buf[..n])?;
    }
}
```

With the two lines from section 2 (`hello` plus newline is 6 bytes, `how are you?` plus newline is
13 bytes) the server prints `closed after 19 bytes`.

</details>

### Exercise 2 — A line-based command server, with tests (medium)

Write `src/bin/cmd_server.rs`, a server on `127.0.0.1:7001` that reads one command per line and answers
with one line: `PING` → `PONG`, `ADD 2 40` → `42` (use `i64` and report overflow as `ERR overflow`), a
bad number → `ERR ADD needs two integers`, anything else → `ERR unknown command`. Put the protocol into
a pure function `fn respond(line: &str) -> String` and test it directly. Add one test that talks to
`handle` through a real socket on port 0.

<details>
<summary>Hint</summary>

For reading lines, wrap the stream in `BufReader`; for writing you need a second handle to the same
socket: `stream.try_clone()`. `reader.lines()` ends when the client closes. In the socket test, send
both commands, call `shutdown(Shutdown::Write)`, and collect the answer lines.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io::{BufRead, BufReader, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;

/// The protocol: one command per line, one answer per line.
fn respond(line: &str) -> String {
    let mut parts = line.split_whitespace();
    match (parts.next(), parts.next(), parts.next()) {
        (Some("PING"), None, None) => "PONG".to_string(),
        (Some("ADD"), Some(a), Some(b)) => match (a.parse::<i64>(), b.parse::<i64>()) {
            (Ok(a), Ok(b)) => a
                .checked_add(b)
                .map_or("ERR overflow".into(), |s| s.to_string()),
            _ => "ERR ADD needs two integers".to_string(),
        },
        _ => "ERR unknown command".to_string(),
    }
}

fn handle(stream: TcpStream) -> std::io::Result<()> {
    let mut writer = stream.try_clone()?; // a second handle to the same socket
    let reader = BufReader::new(stream);
    for line in reader.lines() {
        let answer = respond(&line?);
        writer.write_all(answer.as_bytes())?;
        writer.write_all(b"\n")?;
    }
    Ok(()) // lines() ended: the client closed the connection
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7001")?;
    println!("command server on {}", listener.local_addr()?);
    for stream in listener.incoming() {
        let stream = stream?;
        thread::spawn(move || handle(stream));
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::net::Shutdown;

    #[test]
    fn answers_commands() {
        assert_eq!(respond("PING"), "PONG");
        assert_eq!(respond("ADD 2 40"), "42");
        assert_eq!(respond("ADD 2 x"), "ERR ADD needs two integers");
        assert_eq!(respond("ADD 9223372036854775807 1"), "ERR overflow");
        assert_eq!(respond("DELETE everything"), "ERR unknown command");
    }

    #[test]
    fn works_over_a_real_socket() {
        let listener = TcpListener::bind("127.0.0.1:0").unwrap();
        let addr = listener.local_addr().unwrap();
        thread::spawn(move || {
            let (stream, _) = listener.accept().unwrap();
            handle(stream).unwrap();
        });

        let mut client = TcpStream::connect(addr).unwrap();
        client.write_all(b"PING\nADD 1 2\n").unwrap();
        client.shutdown(Shutdown::Write).unwrap();
        let answers: Vec<String> = BufReader::new(client).lines().map(|l| l.unwrap()).collect();
        assert_eq!(answers, ["PONG", "3"]);
    }
}
```

`cargo test --bin cmd_server` reports `2 passed`. With the server running, typing `PING`, `ADD 2 40` and
`HELLO` into `nc 127.0.0.1 7001` gives back `PONG`, `42` and `ERR unknown command`.

Why it works: `BufReader` collects chunks until it has a whole line, so the protocol code never sees a
half command. Keeping the protocol in a pure function makes most tests fast and independent of the
network; one socket test proves that the pieces fit together.

</details>

### Exercise 3 — At most N clients (hard)

Write `src/bin/limited.rs`: an echo server that serves at most 2 clients at the same time. A third
client gets the line `BUSY\n` and is disconnected right away. When a client leaves, its place becomes
free. Write a test: two clients are served, the third reads `BUSY\n`, and after one of the first two
disconnects, a new client is served again.

<details>
<summary>Hint</summary>

Count active connections with an `Arc<AtomicUsize>`. Increase it in the accept loop (not in the thread,
or several threads could pass the check at once). To decrease it reliably, even when `handle` returns an
error, put the decrement into the `Drop` of a small guard struct that the thread owns.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io::{Read, Write};
use std::net::{TcpListener, TcpStream};
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::thread;

/// Lowers the counter when it is dropped, even if the handler returns early with an error.
struct Slot(Arc<AtomicUsize>);

impl Drop for Slot {
    fn drop(&mut self) {
        self.0.fetch_sub(1, Ordering::SeqCst);
    }
}

fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let mut buf = [0u8; 1024];
    loop {
        let n = stream.read(&mut buf)?;
        if n == 0 {
            return Ok(());
        }
        stream.write_all(&buf[..n])?;
    }
}

fn serve(listener: TcpListener, max: usize) -> std::io::Result<()> {
    let active = Arc::new(AtomicUsize::new(0));
    for stream in listener.incoming() {
        let mut stream = stream?;
        if active.fetch_add(1, Ordering::SeqCst) >= max {
            active.fetch_sub(1, Ordering::SeqCst);
            let _ = stream.write_all(b"BUSY\n");
            continue; // the stream is dropped: the connection closes
        }
        let slot = Slot(Arc::clone(&active));
        thread::spawn(move || {
            let _slot = slot; // lives until the handler is finished
            let _ = handle(stream);
        });
    }
    Ok(())
}

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7002")?;
    println!(
        "echo server (at most 2 clients) on {}",
        listener.local_addr()?
    );
    serve(listener, 2)
}

#[cfg(test)]
mod tests {
    use super::*;

    fn echo_once(stream: &mut TcpStream) -> Vec<u8> {
        stream.write_all(b"hi").unwrap();
        let mut buf = [0u8; 2];
        stream.read_exact(&mut buf).unwrap();
        buf.to_vec()
    }

    #[test]
    fn third_client_is_turned_away() {
        let listener = TcpListener::bind("127.0.0.1:0").unwrap();
        let addr = listener.local_addr().unwrap();
        thread::spawn(move || serve(listener, 2));

        let mut a = TcpStream::connect(addr).unwrap();
        let mut b = TcpStream::connect(addr).unwrap();
        assert_eq!(echo_once(&mut a), b"hi"); // both are really being served
        assert_eq!(echo_once(&mut b), b"hi");

        let mut c = TcpStream::connect(addr).unwrap();
        let mut answer = String::new();
        c.read_to_string(&mut answer).unwrap();
        assert_eq!(answer, "BUSY\n");

        drop(a); // a slot becomes free ...
        thread::sleep(std::time::Duration::from_millis(100));
        let mut d = TcpStream::connect(addr).unwrap();
        assert_eq!(echo_once(&mut d), b"hi"); // ... and a new client gets it
    }
}
```

`cargo test --bin limited` reports `1 passed` (it passed in five runs in a row).

Why it works: `fetch_add` returns the old value, so the check and the increment are one atomic step,
and only the accept loop does it. The `Slot` guard frees the place in `Drop`, which runs however the
thread ends. The short sleep in the test gives the server thread time to see EOF from `a`. The load
balancer project (Lesson 15.15) needs exactly this kind of connection limit.

</details>

## Quiz

1. Which system calls does `TcpListener::bind` make, and what does `accept` return?

<details>
<summary>Answer</summary>

`socket`, `bind` and `listen`. `accept` returns a **new** connected socket (a `TcpStream`) for one
client, plus the client's address; the listening socket stays open for the next clients.

</details>

2. What does `Ok(0)` from `stream.read` mean?

<details>
<summary>Answer</summary>

End of stream (EOF): the other side has closed its sending direction, and no more bytes will arrive.

</details>

3. A client writes 100 bytes with one `write_all`. How many `read` calls does the server need?

<details>
<summary>Answer</summary>

Any number from one upwards. TCP may deliver the bytes in one piece or several, and a single read may
also include bytes of the next message. The server must not depend on it.

</details>

4. Why do tests bind to `127.0.0.1:0` instead of a fixed port?

<details>
<summary>Answer</summary>

Port 0 makes the operating system pick a free port. Tests running in parallel (or another program)
cannot collide on the same port, and `local_addr()` tells the test where to connect.

</details>

5. What is the difference between dropping a `TcpStream` and calling `shutdown(Shutdown::Write)`?

<details>
<summary>Answer</summary>

Dropping closes the whole socket: no more reading or writing. `shutdown(Write)` only sends a FIN for
the sending direction; the program can still read the answer.

</details>

6. Your server runs in a Docker container and binds to `127.0.0.1:8080`. The port is published with
   `-p 8080:8080`, but `curl` from the Mac cannot connect. Why?

<details>
<summary>Answer</summary>

Inside the container, `127.0.0.1` is the container's own loopback interface. Connections from the
published port arrive on the container's `eth0` address, where nothing listens. The server must bind
to `0.0.0.0:8080`.

</details>

## Summary

- `TcpListener::bind` = `socket` + `bind` + `listen`; `accept` returns a new `TcpStream` per client;
  `TcpStream::connect` performs the handshake.
- One thread per connection is the simplest way to serve many clients; always handle errors per
  connection.
- `read` returns the number of bytes it got; `Ok(0)` is EOF. Use `&buf[..n]`, `write_all`, and
  `BufReader` for lines. Writes and reads do not match one to one.
- `shutdown(Shutdown::Write)` half-closes; `set_read_timeout`, `set_write_timeout` and
  `connect_timeout` stop endless waiting.
- Bind to `127.0.0.1` for local use and `0.0.0.0` for all interfaces (and inside containers); port `0`
  picks a free port.
- Test with real sockets on `127.0.0.1:0` and keep the protocol logic in pure, separately tested
  functions.

## Sources and further reading

- [`std::net::TcpListener`](https://doc.rust-lang.org/std/net/struct.TcpListener.html) and [`std::net::TcpStream`](https://doc.rust-lang.org/std/net/struct.TcpStream.html) — every method used in this lesson.
- [The Rust Programming Language — 21.1 Building a Single-Threaded Web Server](https://doc.rust-lang.org/book/ch21-01-single-threaded.html) — the book's version of the first steps.
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) — the socket system calls explained in detail (with C examples).
- [socket(7) manual page](https://man7.org/linux/man-pages/man7/socket.7.html) — the Linux socket options, including `SO_REUSEADDR`.
- [`std::io::ErrorKind`](https://doc.rust-lang.org/std/io/enum.ErrorKind.html) — the list of I/O error kinds.

---

[🏠 Start](../START-HERE.md) · [← 15.3 TCP and UDP](03-tcp-and-udp.md) · [📋 Phase 15](README.md) · [15.5 UDP sockets in Rust →](05-udp-sockets-in-rust.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
