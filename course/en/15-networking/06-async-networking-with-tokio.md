<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.5 UDP sockets in Rust](05-udp-sockets-in-rust.md) · [📋 Phase 15](README.md) · [15.7 Designing a protocol: framing and parsing →](07-designing-a-protocol.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/06-async-networking-with-tokio.md) <!-- nav -->

# Lesson 15.6 — Async networking with Tokio

> **Phase 15 — Networking** · Lesson 6 of 16 · ⏱ about 135 minutes

## What you will learn

- Why one thread per connection stops working at thousands of connections, and what Tokio does instead.
- How to write TCP servers with `tokio::net`: an accept loop with one task per connection.
- How to read lines and write at the same time by splitting a stream into a read half and a write half.
- How to build a chat server that broadcasts messages to all clients with `tokio::sync::broadcast` and `tokio::select!`.
- How to add idle timeouts and test async servers with `#[tokio::test]`.

## Before you start

- You finished [Lesson 15.5 — UDP sockets in Rust](05-udp-sockets-in-rust.md).
- You know `async`/`.await` and the basics of Tokio from [Lesson 06.9](../06-rust-advanced/09-async-await-introduction.md), tasks and channels from [Lesson 14.13 — Tokio in depth: tasks and channels](../14-concurrency-and-async/13-tokio-tasks-and-channels.md), and timeouts and graceful shutdown from [Lesson 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md).
- You know how epoll and kqueue wait for many file descriptors at once ([Lesson 13.13 — Non-blocking I/O and epoll](../13-operating-systems/13-non-blocking-io-and-epoll.md)).
- Add Tokio to the practice package (the version comes from crates.io; on 2026-11-17 it was 1.54.1):

```bash
cd ~/projects/rust/code/p15-net
cargo add tokio@1 --features full
```

## New words

| Word | Meaning |
|---|---|
| **task** | A unit of async work that Tokio runs, started with `tokio::spawn`. Many thousands of tasks share a few threads. |
| **accept loop** | The loop that accepts connections and starts one task for each. |
| **read half / write half** | The two parts of a split `TcpStream`: one can only read, the other can only write. Each can be used in a different place. |
| **broadcast channel** | A channel in which every message is delivered to *every* receiver, not just one. |
| **lagging** | A broadcast receiver that is so slow that the channel had to drop messages it had not read yet. |
| **`select!`** | A Tokio macro that waits for several futures at once and runs the branch of the first one that finishes. |
| **idle timeout** | Closing a connection that has sent nothing for a certain time. |
| **file descriptor limit** | The maximum number of open files (and sockets) per process, shown by `ulimit -n`. |
| **TCP / UDP / SYN** (Transmission Control Protocol / User Datagram Protocol / synchronise) | The two transport protocols of Lesson 15.3, and the TCP flag of the first handshake segment. |
| **API** (Application Programming Interface) | The types and functions a library offers to your code. |

## 1. Why threads run out

The thread-per-connection server of Lesson 15.4 is simple and correct. Its problem is the cost per
connection. Every thread has its own stack (Rust gives new threads 2 MiB of address space by default),
and the operating system has to switch between threads. With ten connections that is nothing. With ten
thousand mostly idle connections (a chat, a game, a load balancer), you have ten thousand threads that
almost always just wait.

An async runtime turns this around:

```text
 threads (Lesson 15.4)                       Tokio (this lesson)
 ┌───────────┐ ┌───────────┐                 ┌────────────────────────────────────────┐
 │ thread 1  │ │ thread 2  │  ...  10,000    │ a few worker threads (one per CPU core)│
 │ blocked   │ │ blocked   │       threads   │  run whichever task is ready           │
 │ in read() │ │ in read() │                 ├────────────────────────────────────────┤
 └───────────┘ └───────────┘                 │ 10,000 tasks: small state machines     │
 each one: a stack, a kernel thread          │ waiting cost: a few hundred bytes each │
                                             ├────────────────────────────────────────┤
                                             │ reactor: epoll (Linux) / kqueue (macOS)│
                                             │ "which sockets have data now?"         │
                                             └────────────────────────────────────────┘
```

A Tokio socket is **non-blocking** (Lesson 13.13). When a task calls `stream.read(&mut buf).await` and
no data is there, the task does not block the thread: it returns `Pending`, the reactor remembers that
this task waits for this socket, and the thread runs another task. When the kernel reports "data on
that socket", the task is woken and continues after its `.await`. You write code that *looks* blocking,
and the runtime makes it cheap to wait.

## 2. An async echo server

The API of `tokio::net` mirrors `std::net`; the methods are `async`, and the read and write methods come
from the traits `AsyncReadExt` and `AsyncWriteExt`. Save as `src/bin/async_echo.rs`:

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};

async fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let mut buf = [0u8; 1024];
    loop {
        let n = stream.read(&mut buf).await?;
        if n == 0 {
            return Ok(()); // EOF: the client closed
        }
        stream.write_all(&buf[..n]).await?;
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7200").await?;
    println!("async echo server on {}", listener.local_addr()?);
    loop {
        let (stream, peer) = listener.accept().await?;
        println!("{peer}: connected");
        tokio::spawn(async move {
            if let Err(e) = handle(stream).await {
                eprintln!("{peer}: error: {e}");
            }
            println!("{peer}: done");
        });
    }
}
```

**What happens:**

1. `#[tokio::main]` starts the runtime and runs `main` as the first task.
2. `TcpListener::bind(...).await` and `listener.accept().await` work like their `std` versions, but
   waiting does not block a thread.
3. `tokio::spawn` starts a new **task** for each connection. The `async move` block takes ownership of
   `stream` and `peer`. The accept loop goes straight back to `accept`.
4. `handle` is the echo loop of Lesson 15.4 with `.await` added. `read` returns `Ok(0)` at EOF, exactly
   as before.

Run it and connect with `nc 127.0.0.1 7200`, type `hello`, and press Ctrl+C:

```text
async echo server on 127.0.0.1:7200
127.0.0.1:51883: connected
127.0.0.1:51883: done
```

Now the point of all this. The next program starts the same echo server and then opens 5,000 client
connections **at the same time**, each sending a message and checking the echo. Save as
`src/bin/many_clients.rs`:

```rust
use std::sync::Arc;
use std::time::Instant;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::Semaphore;
use tokio::task::JoinSet;

async fn echo(mut stream: TcpStream) -> std::io::Result<()> {
    let mut buf = [0u8; 64];
    loop {
        let n = stream.read(&mut buf).await?;
        if n == 0 {
            return Ok(());
        }
        stream.write_all(&buf[..n]).await?;
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:0").await?;
    let addr = listener.local_addr()?;
    tokio::spawn(async move {
        while let Ok((stream, _)) = listener.accept().await {
            tokio::spawn(echo(stream));
        }
    });

    let clients = 5_000;
    let connecting = Arc::new(Semaphore::new(100)); // at most 100 handshakes at once
    let start = Instant::now();
    let mut set = JoinSet::new();
    for i in 0..clients {
        let connecting = Arc::clone(&connecting);
        set.spawn(async move {
            let permit = connecting.acquire_owned().await.expect("semaphore open");
            let mut stream = TcpStream::connect(addr).await?;
            let msg = format!("client {i}");
            stream.write_all(msg.as_bytes()).await?;
            let mut buf = vec![0u8; msg.len()];
            stream.read_exact(&mut buf).await?;
            assert_eq!(buf, msg.as_bytes());
            drop(permit);
            Ok::<_, std::io::Error>(stream) // keep the connection open
        });
    }
    let mut open = Vec::new();
    while let Some(result) = set.join_next().await {
        open.push(result.expect("task panicked")?);
    }
    println!("{} connections open at the same time", open.len());
    println!("all echoed in {:?}", start.elapsed());
    Ok(())
}
```

`cargo run --bin many_clients` on the Mac:

```text
5000 connections open at the same time
all echoed in 105.960416ms
```

**What happens:**

1. 5,000 client tasks and 5,000 server tasks (10,000 sockets) live in one process, on Tokio's few worker
   threads. Every client keeps its connection open until the end (`open` holds the streams).
2. `JoinSet` collects the results of many tasks; `join_next` returns them as they finish.
3. The `Semaphore` (a counter of permits, Lesson 14.13) lets at most 100 clients do their handshake at
   the same time. Without it, all 5,000 SYNs arrive at once and overflow the listen backlog: macOS keeps
   at most 128 waiting connections (`sysctl kern.ipc.somaxconn`), and the extra clients fail with
   `ConnectionReset`. Real clients never arrive in the same microsecond, so this is a fair simulation.

If you see `Error: Os { code: 24, kind: TooManyOpenFiles, message: "Too many open files" }`, your shell
allows too few open files per process. Check with `ulimit -n`; if it says `256`, raise it for this
terminal with `ulimit -n 20000` and run again.

> ⚡ **Performance:** measure with `--release` before you draw conclusions, and remember what this
> test measures: 5,000 connections that each exchange a few bytes on loopback. Async helps most when
> you have *many connections that mostly wait*. For a few busy connections, threads are just as good.

## 3. Lines, and reading while writing

For line-based protocols, wrap the read side in `tokio::io::BufReader` and call `lines()`, from the
trait `AsyncBufReadExt`. Often a task must also *write* while it waits to read, for example a chat
client that receives messages from others while its user is silent. For that, split the stream with
`let (reader, mut writer) = stream.into_split();` and wrap only the reader:
`BufReader::new(reader).lines()`. `into_split` gives an `OwnedReadHalf` and an `OwnedWriteHalf`. They are separate values, so one part of
your code can wait in `lines.next_line().await` while another writes with `writer.write_all(...)`. (There
is also `stream.split()`, which borrows instead of moving; use it when both halves stay in one function.)

## 4. A chat server: broadcast and `select!`

A chat server must deliver each message to *every* connected client. A **broadcast channel** does
exactly that: every receiver gets every message. Each connection task now waits for **two** things at
once: a line from its own client, or a message from the channel. `tokio::select!` waits for both and
runs the branch of whichever comes first. Save as `src/bin/chat.rs`:

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::broadcast;

async fn handle(
    stream: TcpStream,
    tx: broadcast::Sender<String>,
    name: String,
) -> std::io::Result<()> {
    let mut rx = tx.subscribe();
    let (reader, mut writer) = stream.into_split();
    let mut lines = BufReader::new(reader).lines();
    let _ = tx.send(format!("* {name} joined\n"));
    loop {
        tokio::select! {
            line = lines.next_line() => match line? {
                Some(text) => { let _ = tx.send(format!("{name}: {text}\n")); }
                None => break, // EOF: the client left
            },
            msg = rx.recv() => match msg {
                Ok(text) => writer.write_all(text.as_bytes()).await?,
                Err(broadcast::error::RecvError::Lagged(n)) => {
                    writer.write_all(format!("* {n} messages skipped\n").as_bytes()).await?;
                }
                Err(broadcast::error::RecvError::Closed) => break,
            },
        }
    }
    let _ = tx.send(format!("* {name} left\n"));
    Ok(())
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7201").await?;
    println!("chat on {}", listener.local_addr()?);
    let (tx, _) = broadcast::channel::<String>(100);
    loop {
        let (stream, peer) = listener.accept().await?;
        let tx = tx.clone();
        tokio::spawn(async move {
            if let Err(e) = handle(stream, tx, peer.to_string()).await {
                eprintln!("{peer}: {e}");
            }
        });
    }
}
```

**What happens:**

1. `broadcast::channel(100)` creates a sender and a first receiver (which we drop with `_`). Every task
   gets a clone of the sender and calls `subscribe()` to get its own receiver.
2. In `select!`, both branches are futures: `lines.next_line()` finishes when the client sends a line,
   `rx.recv()` finishes when anybody's message arrives in the channel. The first one to finish runs its
   branch; the other future is dropped and created again in the next loop turn. Both of these futures
   are *cancel-safe*: dropping them loses no data (Lesson 14.14 explains why that matters).
3. A line from the client is sent into the channel, so every task, including this one, writes it to its
   client.
4. The channel keeps at most 100 unread messages per receiver. A client that reads too slowly **lags**:
   its receiver gets `Lagged(n)`, and we tell the client how many messages it missed instead of letting
   memory grow without limit. This is backpressure (Lesson 14.13) with a clear policy: slow readers lose
   messages, fast ones are not held back.

Start the server and connect two `nc 127.0.0.1 7201` windows. When client A types `hello from A` and
client B types `hi A, this is B`, A sees:

```text
* 127.0.0.1:52540 joined
* 127.0.0.1:52542 joined
127.0.0.1:52540: hello from A
127.0.0.1:52542: hi A, this is B
* 127.0.0.1:52542 left
```

(The last line appears when B presses Ctrl+C. Clients are named by their address here; Exercise 3 adds
real names.)

> 🔒 **Security:** this server trusts its clients completely. A client can send a line of a gigabyte
> (`lines()` keeps reading until it finds a newline) or stay silent forever. Before such a server meets
> the internet, it needs a maximum line length, an idle timeout (Exercise 2), and a connection limit
> (Exercise 3 of Lesson 15.4). Lesson 15.7 shows how to limit frame sizes.

## 5. UDP and shutdown in Tokio

`tokio::net::UdpSocket` has the same methods as the `std` version from Lesson 15.5, with `.await`:
`bind`, `send_to`, `recv_from`, `connect`. Timeouts are written with `tokio::time::timeout(duration,
future)` instead of `set_read_timeout`.

A real server also needs a **graceful shutdown**: stop accepting, let running connections finish, then
exit. You built that in Lesson 14.14. For a network server, the accept loop uses `select!` with two
branches: `listener.accept()` starts a task as before, and `tokio::signal::ctrl_c()` breaks out of the
loop, so no new connections are accepted. The running tasks are tracked (for example with a `JoinSet` or a `CancellationToken` from `tokio-util`) so
that `main` can wait for them before it returns. The load balancer project (Lesson 15.15) uses this
pattern.

> 🧪 **Testing:** async servers are tested like the threaded ones: bind to `127.0.0.1:0`, start the
> server with `tokio::spawn`, connect with `tokio::net::TcpStream`. Mark the test `#[tokio::test]`
> instead of `#[test]`; it creates a runtime for each test. Wrap waiting steps in
> `tokio::time::timeout`, so that a bug makes the test fail after a second instead of hanging forever.

## Common mistakes

> ⚠️ **Common mistake: blocking inside a task.** `std::thread::sleep`, `std::net::TcpStream`, or a long
> computation inside an async task blocks a whole worker thread, and every other task on it waits. Use
> `tokio::time::sleep`, `tokio::net`, and `tokio::task::spawn_blocking` for heavy work (Lesson 06.9).

> ⚠️ **Common mistake: using `std::io` traits on Tokio streams.** `tokio::net::TcpStream` has no
> `read` method until you import `tokio::io::AsyncReadExt`. The compiler says:
>
> ```text
> error[E0599]: no method named `read` found for struct `tokio::net::TcpStream` in the current scope
> ```
>
> and suggests the import. Use `AsyncReadExt`, `AsyncWriteExt` and `AsyncBufReadExt` from `tokio::io`,
> not `std::io::Read` and `Write`.

## Try it yourself

1. Start `chat` and connect three `nc` windows. Close one with Ctrl+C and watch the others get
   `* ... left`.
2. Change `broadcast::channel::<String>(100)` to `(2)`, connect two clients, and paste many lines quickly
   into one of them. Sometimes the other client sees `* N messages skipped`.
3. In `many_clients.rs`, change `Semaphore::new(100)` to `Semaphore::new(5000)` and run it a few times.
   You will probably see `ConnectionReset` from an overflowing backlog.
4. Run `many_clients` with `cargo run --release --bin many_clients` and compare the time.

## Exercises

### Exercise 1 — Who is online? (easy)

Change `async_echo.rs` so that it prints how many clients are connected each time one connects or
leaves, for example `127.0.0.1:51879: connected (1 online)` and `127.0.0.1:51879: done (0 online)`.

<details>
<summary>Hint</summary>

Share an `Arc<AtomicUsize>` between the accept loop and the tasks (Lesson 14.5). `fetch_add` and
`fetch_sub` return the *old* value.

</details>

<details>
<summary>Answer</summary>

`handle` stays as in section 2; `main` becomes:

```rust
#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7200").await?;
    let online = Arc::new(AtomicUsize::new(0));
    loop {
        let (stream, peer) = listener.accept().await?;
        let now = online.fetch_add(1, Ordering::SeqCst) + 1;
        println!("{peer}: connected ({now} online)");
        let online = Arc::clone(&online);
        tokio::spawn(async move {
            let _ = handle(stream).await;
            let now = online.fetch_sub(1, Ordering::SeqCst) - 1;
            println!("{peer}: done ({now} online)");
        });
    }
}
```

with `use std::sync::Arc;` and `use std::sync::atomic::{AtomicUsize, Ordering};` at the top. With two
`nc` clients, the first staying longer:

```text
127.0.0.1:51879: connected (1 online)
127.0.0.1:51880: connected (2 online)
127.0.0.1:51880: done (1 online)
127.0.0.1:51879: done (0 online)
```

Why it works: an atomic counter can be changed from many tasks without a lock; adding or subtracting 1
to the returned old value gives the new number.

</details>

### Exercise 2 — Hang up on silent clients, with tests (medium)

Write `src/bin/idle.rs`: an echo server on `127.0.0.1:7202` whose handler takes an `idle: Duration`. If
a client sends nothing for that long, the server writes `bye: idle\n` and closes the connection. Write
two `#[tokio::test]` tests: a silent client receives exactly `bye: idle\n` and then EOF; an active client
gets its echo.

<details>
<summary>Hint</summary>

Wrap each read: `tokio::time::timeout(idle, stream.read(&mut buf)).await` returns `Err(Elapsed)` when
the time is up, and `Ok(result_of_read)` otherwise. In the test, use a short `idle` such as 100 ms, and
`read_to_string` on the client to wait for EOF.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::Duration;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio::time::timeout;

/// An echo handler that hangs up on clients that stay silent for longer than `idle`.
async fn handle(mut stream: TcpStream, idle: Duration) -> std::io::Result<()> {
    let mut buf = [0u8; 1024];
    loop {
        let n = match timeout(idle, stream.read(&mut buf)).await {
            Ok(result) => result?,
            Err(_elapsed) => {
                stream.write_all(b"bye: idle\n").await?;
                return Ok(());
            }
        };
        if n == 0 {
            return Ok(());
        }
        stream.write_all(&buf[..n]).await?;
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7202").await?;
    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(handle(stream, Duration::from_secs(10)));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    async fn start(idle: Duration) -> std::net::SocketAddr {
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(async move {
            let (stream, _) = listener.accept().await.unwrap();
            handle(stream, idle).await.unwrap();
        });
        addr
    }

    #[tokio::test]
    async fn silent_client_is_disconnected() {
        let addr = start(Duration::from_millis(100)).await;
        let mut client = TcpStream::connect(addr).await.unwrap();
        let mut answer = String::new();
        client.read_to_string(&mut answer).await.unwrap(); // waits for EOF
        assert_eq!(answer, "bye: idle\n");
    }

    #[tokio::test]
    async fn active_client_is_served() {
        let addr = start(Duration::from_millis(500)).await;
        let mut client = TcpStream::connect(addr).await.unwrap();
        client.write_all(b"ping").await.unwrap();
        let mut buf = [0u8; 4];
        client.read_exact(&mut buf).await.unwrap();
        assert_eq!(&buf, b"ping");
    }
}
```

`cargo test --bin idle` reports `2 passed; ... finished in 0.10s`.

Why it works: `timeout` races the read against a timer. When the timer wins, the read future is dropped
(cancelled), which is safe for `read` because no bytes were taken from the socket. Returning from
`handle` drops the stream, which closes the connection, so the client's `read_to_string` sees EOF.

</details>

### Exercise 3 — A chat with names, with a test (hard)

Write `src/bin/named_chat.rs`: like `chat.rs`, but the server first writes `name? `, and the first line
the client sends becomes its name. Messages look like `ann: hello`. Move the accept loop into
`async fn serve(listener: TcpListener)`, so that a test can start it on port 0. Write a test with two
clients, `ann` and `bob`: when `bob` sends `hi everyone`, both receive `bob: hi everyone`.

<details>
<summary>Hint</summary>

Read the name with `lines.next_line().await?` *before* you subscribe. In the test, write a helper that
reads lines until one ends with an expected text, inside `tokio::time::timeout`, because other lines
(`name? `, join messages) arrive in between. Let `ann` see her own `* ann joined` before `bob` connects:
then you know that ann's task has subscribed and cannot miss bob's messages.

</details>

<details>
<summary>Answer</summary>

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::broadcast;

async fn handle(stream: TcpStream, tx: broadcast::Sender<String>) -> std::io::Result<()> {
    let (reader, mut writer) = stream.into_split();
    let mut lines = BufReader::new(reader).lines();
    writer.write_all(b"name? ").await?;
    let Some(name) = lines.next_line().await? else {
        return Ok(()); // left before giving a name
    };
    let name = name.trim().to_string();
    let mut rx = tx.subscribe();
    let _ = tx.send(format!("* {name} joined\n"));
    loop {
        tokio::select! {
            line = lines.next_line() => match line? {
                Some(text) => { let _ = tx.send(format!("{name}: {text}\n")); }
                None => break,
            },
            msg = rx.recv() => match msg {
                Ok(text) => writer.write_all(text.as_bytes()).await?,
                Err(broadcast::error::RecvError::Lagged(_)) => continue,
                Err(broadcast::error::RecvError::Closed) => break,
            },
        }
    }
    let _ = tx.send(format!("* {name} left\n"));
    Ok(())
}

async fn serve(listener: TcpListener) -> std::io::Result<()> {
    let (tx, _) = broadcast::channel::<String>(100);
    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(handle(stream, tx.clone()));
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    serve(TcpListener::bind("127.0.0.1:7203").await?).await
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::time::Duration;
    use tokio::io::Lines;
    use tokio::net::tcp::{OwnedReadHalf, OwnedWriteHalf};
    use tokio::time::timeout;

    type Incoming = Lines<BufReader<OwnedReadHalf>>;

    async fn join(addr: std::net::SocketAddr, name: &str) -> (Incoming, OwnedWriteHalf) {
        let (r, mut w) = TcpStream::connect(addr).await.unwrap().into_split();
        w.write_all(format!("{name}\n").as_bytes()).await.unwrap();
        (BufReader::new(r).lines(), w)
    }

    /// Reads lines until one ends with `expected`; fails after one second.
    async fn wait_for(lines: &mut Incoming, expected: &str) {
        let found = timeout(Duration::from_secs(1), async {
            while let Some(line) = lines.next_line().await.unwrap() {
                if line.ends_with(expected) {
                    return true;
                }
            }
            false // the connection was closed
        });
        assert!(found.await.unwrap_or(false), "never got {expected:?}");
    }

    #[tokio::test]
    async fn messages_carry_the_sender_name() {
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(serve(listener));

        let (mut ann_in, _ann_out) = join(addr, "ann").await;
        wait_for(&mut ann_in, "* ann joined").await; // ann is subscribed now
        let (mut bob_in, mut bob_out) = join(addr, "bob").await;
        wait_for(&mut ann_in, "* bob joined").await;

        bob_out.write_all(b"hi everyone\n").await.unwrap();
        wait_for(&mut ann_in, "bob: hi everyone").await;
        wait_for(&mut bob_in, "bob: hi everyone").await;
    }
}
```

`cargo test --bin named_chat` reports `1 passed` (five runs in a row passed).

Why it works: the name is read before the task subscribes, so a client without a name never receives
chat messages. The test avoids a race: a broadcast receiver only gets messages sent *after* it
subscribed, so the test first waits until ann's own join message has come back. The first line ann
receives is `name? * ann joined` (the prompt has no newline), which is why the helper checks
`ends_with`, not equality.

</details>

## Quiz

1. Why is a waiting Tokio task cheaper than a waiting thread?

<details>
<summary>Answer</summary>

A waiting task is only a small state machine in memory; it needs no stack of its own and no kernel
thread. A few worker threads run all tasks, and the reactor (epoll/kqueue) tells the runtime which
sockets are ready.

</details>

2. What happens if you call `std::thread::sleep(Duration::from_secs(5))` inside a connection task?

<details>
<summary>Answer</summary>

The worker thread is blocked for 5 seconds. All other tasks on that thread stop making progress during
that time. Use `tokio::time::sleep(...).await` instead.

</details>

3. Why do we split a `TcpStream` into a read half and a write half in the chat server?

<details>
<summary>Answer</summary>

The task must wait for a line from its client and, at the same time, be able to write messages from
others to that client. Two separate halves can be used in the two branches of `select!` without fighting
over one `&mut` borrow.

</details>

4. What is the difference between an `mpsc` channel and a `broadcast` channel?

<details>
<summary>Answer</summary>

In `mpsc`, each message is received by exactly one receiver (there is only one). In `broadcast`, every
subscribed receiver gets a copy of every message.

</details>

5. A chat client stops reading but stays connected. What happens to its broadcast receiver, and why is
   that good for the server?

<details>
<summary>Answer</summary>

Its receiver falls more than 100 messages behind, and the channel drops the oldest ones; the next
`recv` returns `Lagged(n)`. Memory stays bounded and the other clients are not slowed down.

</details>

## Summary

- Thread-per-connection pays a stack and a kernel thread for every connection; Tokio runs many tasks on
  a few threads and uses epoll/kqueue to wake only the ready ones.
- `tokio::net::TcpListener` + `accept().await` + `tokio::spawn` per connection is the basic async server;
  reads and writes come from `AsyncReadExt` and `AsyncWriteExt`.
- `into_split` gives independent read and write halves; `BufReader::lines()` reads whole lines.
- `broadcast` delivers every message to every subscriber; `select!` waits for several events and runs the
  first; `Lagged` is how a bounded broadcast channel handles slow readers.
- Add idle timeouts with `tokio::time::timeout`, limits on lines and connections, and a graceful shutdown.
- Test with `#[tokio::test]`, port `0`, and timeouts around every wait. Watch `ulimit -n` and the listen
  backlog when you open thousands of connections.

## Sources and further reading

- [Tokio tutorial — I/O](https://tokio.rs/tokio/tutorial/io) — reading, writing, splitting and the echo server, from the Tokio authors.
- [Tokio tutorial — Select](https://tokio.rs/tokio/tutorial/select) — how `select!` works and what cancel safety means.
- [`tokio::net`](https://docs.rs/tokio/latest/tokio/net/index.html) — the API of `TcpListener`, `TcpStream` and `UdpSocket`.
- [`tokio::sync::broadcast`](https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html) — the broadcast channel and its lagging rules.
- [Tokio — graceful shutdown](https://tokio.rs/tokio/topics/shutdown) — the official guide to shutting a server down cleanly.

---

[🏠 Start](../START-HERE.md) · [← 15.5 UDP sockets in Rust](05-udp-sockets-in-rust.md) · [📋 Phase 15](README.md) · [15.7 Designing a protocol: framing and parsing →](07-designing-a-protocol.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
