<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.4 Miri, sanitizers and loom in CI](04-miri-sanitizers-and-loom-in-ci.md) · [📋 Phase 20](README.md) · [20.6 Formal verification basics with Kani →](06-formal-verification-with-kani.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) <!-- nav -->

# Lesson 20.5 — Fault injection and chaos testing

> **Phase 20 — Reliability and hardening** · Lesson 5 of 11 · ⏱ about 135 minutes

## What you will learn

- What fault injection and chaos testing are, and the method behind them: steady state, hypothesis, inject, observe.
- How to inject faults *inside* your code with small wrapper types, and find a classic bug: assuming that one `read` fills the whole buffer.
- How to inject network faults (delay, reset, a dead dependency, data in tiny pieces) with toxiproxy in Docker, without changing your program.
- How to turn a chaos experiment into a repeatable script with a "stop button".

## Before you start

- You finished [Lesson 20.4 — Miri, sanitizers and loom in CI](04-miri-sanitizers-and-loom-in-ci.md).
- You know the `Read` trait and files from [Lesson 13.4 — Files and file descriptors](../13-operating-systems/04-files-and-file-descriptors.md), async TCP (Transmission Control Protocol) from [Lesson 15.6 — Async networking with Tokio](../15-networking/06-async-networking-with-tokio.md), and timeouts from [Lesson 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md).
- You know Docker and `curl` from Phase 11, and robust bash scripts from [Lesson 11.9](../11-linux-shell-and-containers/09-robust-bash-scripts.md). Docker Desktop must be running.
- Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p20-chaos
cd code/p20-chaos
cargo add tokio@1 --features full
mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **fault injection** | Causing faults on purpose (errors, delays, broken connections) to check that the program handles them. |
| **chaos testing** (chaos engineering) | Running experiments that inject faults into a whole running system, to learn whether it stays healthy. |
| **steady state** | The normal, measurable behaviour of a system: "all pings succeed in under 50 ms". An experiment compares against it. |
| **hypothesis** | The expected result of an experiment, written before you run it: "with 1 s of extra latency, every ping fails within 600 ms". |
| **short read** | A `read` call that returns fewer bytes than the buffer could hold. It is normal for sockets, pipes and slow devices. |
| **proxy** | A program that sits between a client and a server and passes the traffic on. A fault-injecting proxy can also change it. |
| **toxiproxy** | A TCP proxy for testing, made by Shopify. You control it with an HTTP (HyperText Transfer Protocol) API and add *toxics* to its connections. |
| **toxic** | One kind of fault in toxiproxy: `latency`, `reset_peer`, `slicer`, `timeout`, `bandwidth` and others. |
| **upstream / downstream** | Directions of traffic through a proxy: upstream goes from the client to the server, downstream from the server back to the client. |
| **API** (Application Programming Interface) | Here: the HTTP addresses with which a program (toxiproxy) lets other programs control it. |
| **game day** | A planned session in which a team injects faults into a system on purpose and practises the reaction. |

## 1. Testing failure on purpose

Lesson 20.1 said: faults will happen. Timeouts, retries and error handling are code that runs *only* when something goes wrong, so in normal tests it hardly runs at all. The idea of fault injection is simple: **make the fault happen now, while you are watching**, instead of at three o'clock at night.

The method comes from science and has four steps:

1. **Steady state.** Describe normal behaviour with a number you can measure: "5 of 5 pings succeed, each in under 50 ms".
2. **Hypothesis.** Write down what *should* happen under a fault: "with 1 s of latency, all pings fail, and each one gives up within 600 ms".
3. **Inject.** Cause exactly that fault, and nothing else.
4. **Observe and compare.** Did the system behave as you predicted? If not, you found a weakness before your users did.

You can inject faults at three levels:

| Level | How | Example |
|---|---|---|
| Inside the code | A wrapper type or a test double that fails on purpose | A `Read` that returns one byte per call (Section 2) |
| At the network boundary | A proxy between your program and its dependency | toxiproxy adds 1 s of delay (Section 3) |
| The whole system | Stop processes, machines or network links | Kill one copy of a service in a test cluster (Section 5) |

Start at the lowest level that can show the problem: it is the fastest and the most precise.

## 2. Fault injection inside the code: short reads

A network message format often starts with its length. Put this in `src/lib.rs`:

```rust
//! Reading length-prefixed messages, and a reader that injects faults for tests.

use std::io::{self, Read};

/// Reads one message: a 4-byte big-endian length, then that many bytes.
pub fn read_message(source: &mut impl Read) -> io::Result<Vec<u8>> {
    let mut header = [0u8; 4];
    source.read(&mut header)?; // BUG: `read` may fill only part of the buffer
    let len = u32::from_be_bytes(header) as usize;
    let mut body = vec![0u8; len];
    source.read(&mut body)?; // BUG: the same here
    Ok(body)
}

/// A test helper that wraps a reader and returns at most `max_chunk` bytes per call,
/// like a slow network that delivers data in small pieces.
pub struct Choppy<R> {
    pub inner: R,
    pub max_chunk: usize,
}

impl<R: Read> Read for Choppy<R> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let n = buf.len().min(self.max_chunk);
        self.inner.read(&mut buf[..n])
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    const MESSAGE: &[u8] = &[0, 0, 0, 5, b'h', b'e', b'l', b'l', b'o'];

    #[test]
    fn reads_a_message_from_memory() {
        let mut source = MESSAGE;
        assert_eq!(read_message(&mut source).unwrap(), b"hello");
    }

    #[test]
    fn reads_a_message_that_arrives_in_small_pieces() {
        let mut source = Choppy {
            inner: MESSAGE,
            max_chunk: 2,
        };
        assert_eq!(read_message(&mut source).unwrap(), b"hello");
    }
}
```

**What happens:**

1. `read_message` reads a 4-byte header, turns it into a length with `u32::from_be_bytes` (big-endian, Lesson 12.2), and reads that many bytes.
2. `Read::read(buf)` promises only to read **at least one byte** (or report the end or an error), *not* to fill `buf`. It returns how many bytes it read, and our code throws that number away. On a file or an in-memory slice, `read` usually fills the buffer. On a TCP socket, it returns whatever has arrived so far: a **short read**.
3. `Choppy` is a fault-injecting wrapper. It implements `Read` by passing every call to the inner reader, but with a buffer of at most `max_chunk` bytes. It is a few lines, and it makes any reader behave like a slow network.
4. `&[u8]` itself implements `Read`, so `MESSAGE` can be the inner reader.

```bash
cargo test -q --lib
```

```text
running 2 tests
. 1/2
tests::reads_a_message_that_arrives_in_small_pieces --- FAILED

failures:

---- tests::reads_a_message_that_arrives_in_small_pieces stdout ----

thread 'tests::reads_a_message_that_arrives_in_small_pieces' (6269709) panicked at src/lib.rs:47:9:
assertion `left == right` failed
  left: []
 right: [104, 101, 108, 108, 111]
...
test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The in-memory test passes; the choppy one fails. The first `read` got only 2 bytes, `[0, 0]`, the other two header bytes stayed `0`, so the length was 0 and the body empty. Against a real slow network this function would return wrong messages from time to time: a silent failure.

Clippy knew it all along. Its lint `unused_io_amount` is an *error* by default:

```bash
cargo clippy -q --lib
```

```text
error: read amount is not handled
 --> src/lib.rs:8:5
  |
8 |     source.read(&mut header)?; // BUG: `read` may fill only part of the buffer
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = help: use `Read::read_exact` instead, or handle partial reads
...
```

The fix is `read_exact`, which calls `read` in a loop until the buffer is full and returns the error `UnexpectedEof` if the data ends too early:

```rust
    source.read_exact(&mut header)?;
```

```rust
    source.read_exact(&mut body)?;
```

After both changes, `cargo test -q --lib` reports `2 passed`.

> 🔒 **Security:** `vec![0u8; len]` with a length taken from the network lets a sender make you allocate up to 4 GB (gigabytes) with four bytes. A real reader checks `len` against a maximum first (Lesson 10.12, and the limits of Lesson 20.8).

> 🧪 **Testing:** Wrapper types like `Choppy` are cheap and fast. Build a small set for every I/O boundary: one that delivers tiny pieces, one that is interrupted, one that breaks after N bytes (Exercise 2 builds the second one). They work with every function that takes `impl Read` or `impl Write`, which is one more reason to write functions against traits instead of concrete types (Lesson 09.3).

## 3. Network faults with toxiproxy

Wrappers need code changes. Sometimes you want to test the real, unchanged program, including the operating system's sockets. Then you put a **proxy** between the program and its dependency and let the proxy misbehave.

```text
 client ──► toxiproxy (port 7171) ──► echo server (port 7070)
                 ▲
                 │ HTTP API (port 8474): "add 1 s of latency"
               curl
```

First the two programs. Create `src/bin/echo_server.rs`, a server that answers every line with the same line:

```rust
//! A tiny TCP server: it answers every line with the same line.

use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7070").await?;
    println!("echo server listening on 127.0.0.1:7070");
    loop {
        let (socket, peer) = listener.accept().await?;
        tokio::spawn(async move {
            let (reader, mut writer) = socket.into_split();
            let mut lines = BufReader::new(reader).lines();
            while let Ok(Some(line)) = lines.next_line().await {
                if writer.write_all(format!("{line}\n").as_bytes()).await.is_err() {
                    break;
                }
            }
            println!("connection from {peer} closed");
        });
    }
}
```

and `src/bin/client.rs`, which sends five pings, each with a 500 ms limit, and exits with code 1 if any failed:

```rust
//! Sends five pings through an address and reports what happened to each one.

use std::time::{Duration, Instant};

use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpStream;
use tokio::time::timeout;

const LIMIT: Duration = Duration::from_millis(500);

async fn ping(address: &str, n: u32) -> Result<String, String> {
    let stream = timeout(LIMIT, TcpStream::connect(address))
        .await
        .map_err(|_| "connect timed out".to_string())?
        .map_err(|e| format!("connect failed: {e}"))?;
    let (reader, mut writer) = stream.into_split();
    writer
        .write_all(format!("ping {n}\n").as_bytes())
        .await
        .map_err(|e| format!("write failed: {e}"))?;
    let mut reply = String::new();
    let mut reader = BufReader::new(reader);
    match timeout(LIMIT, reader.read_line(&mut reply)).await {
        Err(_) => Err("no answer within 500 ms".to_string()),
        Ok(Err(e)) => Err(format!("read failed: {e}")),
        Ok(Ok(0)) => Err("connection closed without an answer".to_string()),
        Ok(Ok(_)) => Ok(reply.trim_end().to_string()),
    }
}

#[tokio::main]
async fn main() {
    let address = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "127.0.0.1:7171".to_string());
    let mut failures = 0;
    for n in 1..=5 {
        let started = Instant::now();
        let result = ping(&address, n).await;
        let ms = started.elapsed().as_millis();
        match result {
            Ok(reply) => println!("ping {n}: ok   {ms:>4} ms  ({reply})"),
            Err(reason) => {
                failures += 1;
                println!("ping {n}: FAIL {ms:>4} ms  ({reason})");
            }
        }
    }
    println!("{failures} of 5 failed");
    std::process::exit(if failures == 0 { 0 } else { 1 });
}
```

**What happens:**

1. Both the connect and the read have a `timeout` (Lesson 14.14). Without them, a dead dependency would make the client wait forever: a hang.
2. `read_line` on a `BufReader` keeps reading until it sees `\n`, so short reads are handled for us here.
3. `Ok(Ok(0))` means "the connection ended before any byte arrived".
4. The default address is the proxy's port 7171; a command-line argument can override it.

Build both and start the server in a second terminal tab (it keeps running):

```bash
cargo build -q
cargo run -q --bin echo_server
```

```text
echo server listening on 127.0.0.1:7070
```

Now start toxiproxy from the image published by its authors, with its API port and one proxy port, and create a proxy called `echo` with `curl`:

```bash
docker run -d --name course-toxiproxy -p 8474:8474 -p 7171:7171 ghcr.io/shopify/toxiproxy:2.13.0
curl -s localhost:8474/version
curl -s -X POST localhost:8474/proxies -d '{"name":"echo","listen":"0.0.0.0:7171","upstream":"host.docker.internal:7070"}'
```

```text
{"version": "2.13.0"}
{"name":"echo","listen":"[::]:7171","upstream":"host.docker.internal:7070","enabled":true,"Logger":{},"toxics":[]}
```

(`docker run -d` first prints the long ID, the identifier, of the new container.)

**What happens:**

1. The container is named with the `course-` prefix, so you can find and remove it later. `-p 8474:8474` publishes the control API, `-p 7171:7171` the port on which the proxy will listen.
2. `POST /proxies` creates a proxy: listen on port 7171 inside the container, and forward to `host.docker.internal:7070`. `host.docker.internal` is a name that Docker Desktop provides inside containers; it means "the Mac that runs Docker", where our echo server is.

Measure the **steady state** through the proxy:

```bash
./target/debug/client
```

```text
ping 1: ok      7 ms  (ping 1)
ping 2: ok      4 ms  (ping 2)
ping 3: ok      4 ms  (ping 3)
ping 4: ok      3 ms  (ping 4)
ping 5: ok      5 ms  (ping 5)
0 of 5 failed
```

**Inject latency.** A toxic is added with `POST /proxies/<name>/toxics`. This one delays all data from the server to the client (`downstream`) by 1000 ms:

```bash
curl -s -X POST localhost:8474/proxies/echo/toxics -d '{"name":"slow","type":"latency","stream":"downstream","attributes":{"latency":1000,"jitter":0}}'
./target/debug/client
```

```text
{"attributes":{"latency":1000,"jitter":0},"name":"slow","type":"latency","stream":"downstream","toxicity":1}
ping 1: FAIL  502 ms  (no answer within 500 ms)
ping 2: FAIL  501 ms  (no answer within 500 ms)
ping 3: FAIL  502 ms  (no answer within 500 ms)
ping 4: FAIL  501 ms  (no answer within 500 ms)
ping 5: FAIL  502 ms  (no answer within 500 ms)
5 of 5 failed
```

The hypothesis holds: every ping fails, and each one gives up after about 500 ms instead of waiting. A slow dependency costs this client at most half a second per request. Remove the toxic with `DELETE`:

```bash
curl -s -X DELETE localhost:8474/proxies/echo/toxics/slow
```

**Inject a partial fault.** Real problems often hit only *some* requests. `toxicity` is the probability that a new connection gets the toxic:

```bash
curl -s -X POST localhost:8474/proxies/echo/toxics -d '{"name":"some-slow","type":"latency","stream":"downstream","toxicity":0.4,"attributes":{"latency":800,"jitter":0}}'
./target/debug/client
curl -s -X DELETE localhost:8474/proxies/echo/toxics/some-slow
```

```text
{"attributes":{"latency":800,"jitter":0},"name":"some-slow","type":"latency","stream":"downstream","toxicity":0.4}
ping 1: ok     26 ms  (ping 1)
ping 2: FAIL  502 ms  (no answer within 500 ms)
ping 3: ok      5 ms  (ping 3)
ping 4: ok      7 ms  (ping 4)
ping 5: ok      5 ms  (ping 5)
1 of 5 failed
```

(Which pings fail is random; run it a few times.)

**Make the dependency disappear.** `POST /proxies/echo` with `{"enabled":false}` closes the proxy, as if the server were down:

```bash
curl -s -X POST localhost:8474/proxies/echo -d '{"enabled":false}'
./target/debug/client
curl -s -X POST localhost:8474/proxies/echo -d '{"enabled":true}' > /dev/null
```

```text
{"name":"echo","listen":"[::]:7171","upstream":"host.docker.internal:7070","enabled":false,"Logger":{},"toxics":[]}
ping 1: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
ping 2: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
ping 3: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
ping 4: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
ping 5: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
5 of 5 failed
```

You might expect "connection refused". But Docker Desktop itself keeps port 7171 open on the Mac and forwards connections into the container; when nothing listens there any more, it accepts and then resets the connection. Experiments like this teach you how your *real* environment fails, which is often different from the textbook.

The most useful toxics:

| Toxic | What it does | Fault it imitates |
|---|---|---|
| `latency` | Delays data by `latency` ms (± `jitter`) | A slow network or an overloaded server |
| `timeout` | Stops all data; closes after `timeout` ms (0 = never) | A server that hangs |
| `reset_peer` | Resets the connection after `timeout` ms | A crashed server, a broken connection |
| `slicer` | Cuts data into small pieces with small delays | Short reads (Section 2) on a real socket |
| `bandwidth` | Limits the speed to `rate` KB/s (kilobytes per second) | A slow mobile network |
| `limit_data` | Closes the connection after `bytes` bytes | A transfer that breaks in the middle |

## 4. A chaos experiment as a script

Typing `curl` commands is good for exploring. For a check you can repeat (after every change, or in CI, continuous integration), write the experiment down. Create `chaos.sh` in the package folder:

```bash
#!/usr/bin/env bash
# A small chaos experiment: inject faults with toxiproxy and check how the client reacts.
set -euo pipefail

API=http://localhost:8474
CLIENT=./target/debug/client

add_toxic() { curl -sf -X POST "$API/proxies/echo/toxics" -d "$1" > /dev/null; }
remove_toxic() { curl -sf -X DELETE "$API/proxies/echo/toxics/$1" > /dev/null; }

# The stop button: whatever happens, remove our toxics when the script ends.
TOXICS=(slow)
cleanup() {
  for toxic in "${TOXICS[@]}"; do
    curl -s -X DELETE "$API/proxies/echo/toxics/$toxic" > /dev/null || true
  done
}
trap cleanup EXIT

# expect <description> <expected exit code> <max seconds>
expect() {
  local description=$1 want=$2 max_seconds=$3 got=0
  local started=$SECONDS
  "$CLIENT" > /dev/null || got=$?
  local took=$((SECONDS - started))
  if [[ $got -ne $want ]]; then
    echo "FAIL $description: exit code $got, expected $want"
    return 1
  fi
  if ((took > max_seconds)); then
    echo "FAIL $description: took ${took}s, expected at most ${max_seconds}s"
    return 1
  fi
  echo "PASS $description (exit $got, ${took}s)"
}

expect "steady state: all pings succeed" 0 2
add_toxic '{"name":"slow","type":"latency","stream":"downstream","attributes":{"latency":1000}}'
expect "1 s of latency: every ping fails fast" 1 4
remove_toxic slow
expect "after the fault: back to steady state" 0 2
```

```bash
chmod +x chaos.sh
./chaos.sh
```

```text
PASS steady state: all pings succeed (exit 0, 0s)
PASS 1 s of latency: every ping fails fast (exit 1, 3s)
PASS after the fault: back to steady state (exit 0, 0s)
```

**What happens:**

1. `set -euo pipefail` (Lesson 11.9) stops the script at the first unexpected error. `curl -sf` makes `curl` fail on HTTP errors, so a typo in a toxic stops the experiment instead of silently testing nothing.
2. `trap cleanup EXIT` is the **stop button**: whether the script passes, fails or is interrupted with Ctrl+C, the toxics are removed. An experiment must never leave a fault behind.
3. `expect` runs the client and checks two things: the exit code and the time. `SECONDS` is a bash variable that counts seconds since the script started.
4. The three calls are steady state, hypothesis under fault, and recovery. The last one matters: a system that does not come back after the fault is gone has a bug too.

Does the experiment catch a real weakness? Change `LIMIT` in `client.rs` to `Duration::from_secs(10)`, rebuild with `cargo build -q`, and run the script again:

```text
PASS steady state: all pings succeed (exit 0, 0s)
FAIL 1 s of latency: every ping fails fast: exit code 0, expected 1
```

With a 10-second limit, the client waited a full second for every answer and called it a success. For a user, a request that takes a second may be as bad as a failure, and for a server it means one second of a busy connection per request (the overload of Lesson 20.8). The script caught a timeout that is too generous. Change `LIMIT` back to 500 ms and rebuild.

## 5. Chaos engineering on whole systems

The same method scales up to whole systems. Teams kill random copies of a service, cut the network between two data centres, or fill a disk, and watch dashboards to see whether users notice. Netflix made this famous with a tool called Chaos Monkey; in Kubernetes (Phase 21) tools such as Chaos Mesh do it for pods. A **game day** is a planned session in which a team does this together and practises the reaction.

The rules that keep it safe are the same as in your script:

- **Start small and local.** First in tests (Sections 2–4), then in a test environment, and only much later, with care, in production.
- **Limit the blast radius.** One copy, one dependency, a small share of traffic (`toxicity`).
- **Always have a stop button**, and decide in advance when to press it.
- **Write the hypothesis first.** An experiment without an expected result is just breaking things.

> ⚠️ **Common mistake:** Run fault injection only against systems you own and run yourself. The toxiproxy in this lesson sits on your own Mac between your own two programs.

When you are done, stop the echo server with Ctrl+C and remove the container:

```bash
docker rm -f course-toxiproxy
```

## Common mistakes

**1. Testing error handling only with "happy" readers.** A `&[u8]` or a file usually fills the whole buffer, so code that ignores short reads passes every test. Add a choppy wrapper to at least one test of every reader.

**2. Ignoring the number that `read` returns.** Use `read_exact` (or `read_to_end`, `BufReader::read_line`), and keep Clippy's `unused_io_amount` error on.

**3. No timeout on a network call.** Without a timeout, the latency experiment would not fail fast; it would hang until the toxic is removed. Every outgoing call needs a deadline.

**4. Leaving faults behind.** A toxic that stays after an experiment makes the next person's tests fail mysteriously. Use a `trap … EXIT` cleanup, and check `curl -s localhost:8474/proxies` when in doubt.

## Try it yourself

1. Create `code/p20-chaos`, write `read_message` with the bug and both tests. Watch the choppy test fail and Clippy report `unused_io_amount`. Fix it with `read_exact`.
2. Write the echo server and the client. Run the client directly against the server: `./target/debug/client 127.0.0.1:7070`.
3. Start toxiproxy, create the `echo` proxy, and repeat the four experiments of Section 3: steady state, latency, partial latency, disabled proxy.
4. Write `chaos.sh` and run it. Then make the client's limit 10 s and see the script fail.
5. Remove the container with `docker rm -f course-toxiproxy`.

## Exercises

### Exercise 1 — Predict, then measure (easy)

With a `latency` toxic of 300 ms on the downstream, what will the client print? Write your prediction, then run it.

<details>
<summary>Hint</summary>

Compare 300 ms with the client's limit of 500 ms.

</details>

<details>
<summary>Answer</summary>

```bash
curl -s -X POST localhost:8474/proxies/echo/toxics -d '{"name":"slow","type":"latency","stream":"downstream","attributes":{"latency":300}}'
./target/debug/client
curl -s -X DELETE localhost:8474/proxies/echo/toxics/slow
```

```text
ping 1: ok    304 ms  (ping 1)
ping 2: ok    304 ms  (ping 2)
ping 3: ok    304 ms  (ping 3)
ping 4: ok    310 ms  (ping 4)
ping 5: ok    309 ms  (ping 5)
0 of 5 failed
```

All pings succeed, each about 300 ms slower. 300 ms is below the 500 ms limit, so the client waits and gets the answer. (The first line of output, the JSON (JavaScript Object Notation) answer of the API, is left out here.)

</details>

### Exercise 2 — An interrupted reader (medium)

System calls can return the error `Interrupted` when a signal arrives (Lesson 13.7); the right reaction is to simply try again. Write a wrapper `Interrupting<R>` with the fields `inner: R` and `calls: u32` that returns `Err(io::ErrorKind::Interrupted)` on every odd call and reads normally on every even call. Write a test that `read_message` (fixed with `read_exact`) still returns `b"hello"`.

<details>
<summary>Hint</summary>

`io::Error::from(io::ErrorKind::Interrupted)` creates the error. Increase `calls` at the start of `read` and check `self.calls % 2 == 1`.

</details>

<details>
<summary>Answer</summary>

```rust
/// A test helper that fails with `Interrupted` on every second call.
pub struct Interrupting<R> {
    pub inner: R,
    pub calls: u32,
}

impl<R: Read> Read for Interrupting<R> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        self.calls += 1;
        if self.calls % 2 == 1 {
            return Err(io::Error::from(io::ErrorKind::Interrupted));
        }
        self.inner.read(buf)
    }
}

#[cfg(test)]
mod fault_tests {
    use super::*;

    const MESSAGE: &[u8] = &[0, 0, 0, 5, b'h', b'e', b'l', b'l', b'o'];

    #[test]
    fn interrupted_reads_are_retried() {
        let mut source = Interrupting {
            inner: MESSAGE,
            calls: 0,
        };
        assert_eq!(read_message(&mut source).unwrap(), b"hello");
    }
}
```

Why it works: `read_exact` treats `Interrupted` as "try again" and continues its loop; every other error is returned to the caller. Your own read loops must do the same.

</details>

### Exercise 3 — Data in tiny pieces (hard)

Add a fourth step to `chaos.sh`: a `slicer` toxic that sends the server's answer one byte at a time (`"average_size":1,"size_variation":0,"delay":10000`, where `delay` is in microseconds). Predict first: does the client still pass? Make sure the stop button also removes this toxic.

<details>
<summary>Hint</summary>

Add the toxic's name to the `TOXICS` array. Think about how `read_line` on a `BufReader` collects bytes.

</details>

<details>
<summary>Answer</summary>

Change the array to `TOXICS=(slow pieces)` and add at the end:

```bash
add_toxic '{"name":"pieces","type":"slicer","stream":"downstream","attributes":{"average_size":1,"size_variation":0,"delay":10000}}'
expect "answers arrive one byte at a time: still correct" 0 2
remove_toxic pieces
```

```text
PASS steady state: all pings succeed (exit 0, 0s)
PASS 1 s of latency: every ping fails fast (exit 1, 3s)
PASS after the fault: back to steady state (exit 0, 0s)
PASS answers arrive one byte at a time: still correct (exit 0, 0s)
```

Why it works: `read_line` keeps reading until it finds `\n`, however small the pieces are, so the client handles short reads correctly. Each answer takes about 70 ms (7 bytes, 10 ms each), well within the limit. This is Section 2's `Choppy` test, done on a real socket.

</details>

## Quiz

1. What are the four steps of a chaos experiment?

<details>
<summary>Answer</summary>

Define the steady state, write a hypothesis, inject one fault, observe and compare with the hypothesis (and then check that the system returns to the steady state).

</details>

2. Why can code that ignores the return value of `read` pass all tests that read from a `&[u8]`?

<details>
<summary>Answer</summary>

Reading from memory usually fills the whole buffer in one call, so the bug never shows. On sockets and pipes, `read` often returns fewer bytes: a short read.

</details>

3. What does `"toxicity": 0.4` do?

<details>
<summary>Answer</summary>

Each new connection gets the toxic with a probability of 40 %, so only some requests are affected, like many real-world faults.

</details>

4. Why does `chaos.sh` use `trap cleanup EXIT`?

<details>
<summary>Answer</summary>

It is the stop button: the toxics are removed whenever the script ends, whether it passes, fails or is interrupted. An experiment must not leave faults behind.

</details>

5. The experiment with a 10-second client limit "failed" although all pings succeeded. Why is that the right result?

<details>
<summary>Answer</summary>

The hypothesis was "under a slow dependency, the client fails fast". With 10 s, it waited a full second for each answer. That is a weakness: slow answers keep resources busy and can overload a service. The experiment found a timeout that is too generous.

</details>

## Summary

- Fault injection makes rare faults happen now, under observation. The method: steady state, hypothesis, inject, observe, then check recovery.
- Inside the code, small wrappers that implement `Read` or `Write` inject short reads, interruptions and broken connections. They found the classic bug of ignoring `read`'s return value; `read_exact` and Clippy's `unused_io_amount` prevent it.
- At the network boundary, toxiproxy adds latency, resets, hangs and slicing to real TCP connections without changing your program. `toxicity` hits only some connections.
- A chaos experiment becomes repeatable as a script with clear expectations (exit code and time) and a stop button (`trap … EXIT`).
- On whole systems the same rules apply: start small, limit the blast radius, keep a stop button, and only test systems you own.

## Sources and further reading

- [Toxiproxy](https://github.com/Shopify/toxiproxy) — the README lists every toxic, its attributes and the full HTTP API.
- [std::io::Read](https://doc.rust-lang.org/std/io/trait.Read.html) — the exact promises of `read`, `read_exact` and how `Interrupted` is handled.
- [Clippy — unused_io_amount](https://rust-lang.github.io/rust-clippy/master/index.html) — search for `unused_io_amount`, the lint that catches ignored read and write amounts.
- [Principles of Chaos Engineering](https://principlesofchaos.org/) — the short, original description of the method: steady state, hypotheses, small blast radius.
- [Docker Desktop networking](https://docs.docker.com/desktop/features/networking/) — how `host.docker.internal` and published ports work on a Mac.

---

[🏠 Start](../START-HERE.md) · [← 20.4 Miri, sanitizers and loom in CI](04-miri-sanitizers-and-loom-in-ci.md) · [📋 Phase 20](README.md) · [20.6 Formal verification basics with Kani →](06-formal-verification-with-kani.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
