<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.29 Cloud-native track: project and next steps](29-cloud-native-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [Phase 23 — Capstone projects →](../23-capstone-projects/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/30-checkpoint-your-specialization.md) <!-- nav -->

# Lesson 22.30 — Checkpoint: your specialization

> **Phase 22 — Specializations** · Lesson 30 of 30 · ⏱ about 3.5 hours (2 sittings)

## What you will learn

This lesson teaches nothing new. It checks that you can use what your track taught, and it shows you where to go
back if something is still weak. You will:

- Answer the questions for everyone and the questions of **your** track (seven questions in total).
- Solve one practical coding task from your track, with tests, plus two short tasks that prepare the next phase.
- Find the lessons you should repeat, with direct links.

## Before you start

- You finished your track, including its project lesson: [22.5](05-os-track-project-and-next-steps.md) (OS:
  operating system), [22.9](09-ebpf-track-project-and-next-steps.md) (eBPF: extended Berkeley Packet Filter), [22.13](13-embedded-track-project-and-next-steps.md)
  (embedded), [22.17](17-databases-track-project-and-next-steps.md) (databases),
  [22.21](21-networking-track-project-and-next-steps.md) (networking),
  [22.25](25-compilers-track-project-and-next-steps.md) (compilers) or
  [22.29](29-cloud-native-track-project-and-next-steps.md) (cloud-native).
- Do the quiz **closed book**: no lessons, no compiler, no search. Write your answer first, then open the hidden one.
- Answer only your own track's questions. Reading the other tracks' questions is allowed and is a cheap way to see
  what they are about, but you are not expected to know the answers.
- For the practical tasks you may use the compiler and the lessons. Create one package for them:
  `cargo new code/p22-checkpoint`.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A pause at the end of a phase to test yourself before you go on. |
| **closed book** | Answering from memory only, without looking anything up. |
| **track project** | The project of your track's last lesson: a kernel, an eBPF tool, a firmware, a query engine, a proxy, an interpreter or an operator. |

The quiz uses the abbreviations of all tracks. Each is explained in its track's lessons; here is a short reminder:

| Abbreviation | Stands for | Track |
|---|---|---|
| **QEMU** / **UART** / **MMIO** | Quick Emulator / Universal Asynchronous Receiver-Transmitter (a serial port) / memory-mapped input and output | OS |
| **BSS** / **MMU** / **GIC** | Block Started by Symbol (zeroed data) / Memory Management Unit / Generic Interrupt Controller | OS |
| **XDP** | eXpress Data Path, the earliest hook for received packets | eBPF |
| **HAL** / **SPI** / **I2C** | Hardware Abstraction Layer / Serial Peripheral Interface / Inter-Integrated Circuit (two common chip buses) | embedded |
| **SWD** / **RTT** | Serial Wire Debug (the probe's connection to the chip) / Real-Time Transfer (logs through the probe) | embedded |
| **SQL** / **CSV** / **SIMD** | Structured Query Language / comma-separated values / Single Instruction, Multiple Data | databases |
| **TCP** / **HTTP** | Transmission Control Protocol / HyperText Transfer Protocol | networking |
| **AST** | Abstract Syntax Tree, the tree a parser builds | compilers |
| **CRD** / **RBAC** | CustomResourceDefinition / Role-Based Access Control | cloud-native |

## Self-test quiz

### Everyone

1. Three tracks write code without the standard library. What does `#![no_std]` remove, and what is still there?

<details>
<summary>Answer</summary>

It removes `std`: everything that needs an operating system, such as files, threads, the network, and the
allocating collections by default. `core` stays: `Option`, `Result`, slices, iterators, `fmt` and the integer types.
`alloc` (`Vec`, `String`, `Box`) is available only if you provide a global allocator.

</details>

2. Every track kept some code "pure" (no hardware, kernel, network or cluster). Why?

<details>
<summary>Answer</summary>

Pure code can be unit-tested on your Mac in milliseconds. The part that touches the real world is then small and is
covered by fewer, slower tests (QEMU, a privileged container, a board, a kind cluster).

</details>

3. What is a target triple, and name one your track used that is not your Mac's.

<details>
<summary>Answer</summary>

The name of the platform the compiler builds for: CPU (Central Processing Unit) architecture, vendor, operating system (and sometimes the
binary interface). Your Mac is `aarch64-apple-darwin`. Examples: `aarch64-unknown-none-softfloat` (OS track),
`thumbv8m.main-none-eabihf` (Pico 2), `bpfel-unknown-none` (eBPF), a `wasm32` target (compilers), and Linux
`aarch64-unknown-linux-gnu` in the VM `dev` (eBPF, networking) and in the operator's container image (cloud-native).

</details>

### OS track

1. Which three things must a freestanding binary provide that `std` provided before?

<details>
<summary>Answer</summary>

A panic handler (`#[panic_handler]`), its own entry point (with `#![no_main]`, because there is no runtime to call
`main`), and a panic strategy without unwinding (`panic = "abort"`).

</details>

2. Why are the UART registers read and written with volatile accesses?

<details>
<summary>Answer</summary>

They are MMIO: device registers at memory addresses. A write has an effect outside the program, so
the compiler must not remove, merge or reorder those accesses as it may for normal memory. Volatile accesses forbid that.

</details>

3. On arm64, what happens between "a timer interrupt arrives" and "your handler runs"?

<details>
<summary>Answer</summary>

The CPU stops the current code, saves the return address and the processor state in system
registers, and jumps to the entry of the vector table that matches the exception kind and level. Your handler saves
the other registers, asks the GIC for the interrupt number, handles and acknowledges it, restores the registers and
returns with `eret`.

</details>

4. What is an identity mapping, and why is it used when the MMU is switched on?

<details>
<summary>Answer</summary>

A mapping where the virtual address equals the physical address. The code that turns on the MMU keeps running at the
same addresses after the switch, so the next instruction is still found.

</details>

### eBPF track

1. What does the eBPF verifier check, and why did your programs need explicit bounds checks?

<details>
<summary>Answer</summary>

Before the kernel loads a program, the verifier proves it is safe: every memory access is in bounds, the program
terminates, it only calls allowed helpers. If the verifier cannot prove that a packet read is in bounds, it rejects the
program, so you check `start + size <= end` before every read.

</details>

2. What is the difference between a kprobe and a tracepoint?

<details>
<summary>Answer</summary>

A kprobe attaches to (almost) any kernel function; it is flexible but depends on kernel internals, which change between
versions. A tracepoint is a fixed hook that kernel developers placed on purpose, with a documented, stable format.

</details>

3. What are eBPF maps for?

<details>
<summary>Answer</summary>

They are the shared memory of eBPF: key-value tables, arrays and ring buffers that both the program in the kernel and
your user-space program (through Aya) can read and write, for example counters per process or events to print.

</details>

4. When does an XDP program run, and which decisions can it return?

<details>
<summary>Answer</summary>

In the network driver, for each received packet, before the kernel's network stack does any work on it. It returns an
action: `XDP_PASS` (continue normally), `XDP_DROP`, `XDP_TX` (send back out of the same interface), `XDP_REDIRECT`
or `XDP_ABORTED` (an error).

</details>

### Embedded track

1. What does the microcontroller not have that your Mac has, and what does that mean for the Rust program?

<details>
<summary>Answer</summary>

No operating system, very little RAM (random-access memory, the working memory), usually no MMU and no file
system. The program is `no_std` and `no_main`, a
runtime crate provides the reset entry point and the vector table, and you supply a panic handler.

</details>

2. What do the `embedded-hal` traits give you?

<details>
<summary>Answer</summary>

Common interfaces such as `OutputPin`, `InputPin`, `DelayNs`, and buses such as SPI and I2C. A driver written against the traits works
with any chip's HAL, and you can test it on your Mac with fake pins.

</details>

3. Why does a button need debouncing, and what does "active low" mean?

<details>
<summary>Answer</summary>

The metal contacts bounce for a few milliseconds, so one press looks like several quick changes. Active low means the
pin reads *low* when the button is pressed, because a pull-up resistor holds it high otherwise.

</details>

4. What does Embassy give you compared with a super loop?

<details>
<summary>Answer</summary>

Several async tasks can wait at the same time (for a timer, a pin edge, a channel) without busy-waiting. The executor
lets the CPU sleep until an interrupt wakes the task that can make progress.

</details>

### Databases track

1. What is the difference between a logical plan and a physical plan?

<details>
<summary>Answer</summary>

The logical plan says *what* to compute, as a tree of relational operators (scan, filter, projection, aggregation). The
physical plan says *how*: which algorithms (for example a hash aggregation), in what order, on which batches.

</details>

2. Volcano model or vectorized execution: what is the difference?

<details>
<summary>Answer</summary>

In the Volcano model every operator has `next()` and pulls one row at a time from its child. In vectorized execution
operators pass batches of columns (Arrow record batches), so there are far fewer calls and the CPU works on long, tight
arrays.

</details>

3. Why does an analytics engine store data in columns (Apache Arrow)?

<details>
<summary>Answer</summary>

Analytic queries read a few columns over many rows. With columns stored together, the engine reads only the columns it
needs, the values sit next to each other in memory (good for caches and SIMD), and they compress well.

</details>

4. How does hash aggregation compute `GROUP BY city` with `COUNT(*)`?

<details>
<summary>Answer</summary>

It keeps a hash map from the group key (`city`) to an accumulator (a counter). One pass over the input updates the
accumulator of each row's key; at the end every map entry becomes one output row.

</details>

### Networking track

1. What does `SO_REUSEPORT` allow, and how does it lead to a thread-per-core server?

<details>
<summary>Answer</summary>

Several sockets can listen on the same address and port, and Linux spreads new connections over them. Each thread then
owns one listener and its own runtime, so threads share almost nothing.

</details>

2. What does "zero-copy" mean when you parse with `Bytes` and `BytesMut`?

<details>
<summary>Answer</summary>

The parsed parts (method, path, header values) are views into the buffer that was read from the socket, shared through
a reference count. No bytes are copied into new strings.

</details>

3. Why must a protocol parser handle partial input?

<details>
<summary>Answer</summary>

TCP delivers a stream of bytes, not messages. One read can end in the middle of a request, so the parser must say
"incomplete, read more" instead of failing.

</details>

4. Your proxy answers `502` for one request and `504` for another. What happened in each case?

<details>
<summary>Answer</summary>

`502 Bad Gateway`: the upstream could not be used or gave an invalid answer (for example the connection was refused).
`504 Gateway Timeout`: the upstream did not answer in time.

</details>

### Compilers track

1. What is the job of the lexer, and what is the job of the parser?

<details>
<summary>Answer</summary>

The lexer turns characters into tokens (numbers, names, operators, keywords). The parser turns tokens into a syntax tree
(AST) according to the grammar.

</details>

2. What do precedence and associativity decide, and how does Pratt parsing handle both?

<details>
<summary>Answer</summary>

Precedence decides that `1 + 2 * 3` is `1 + (2 * 3)`; associativity decides that `10 - 4 - 3` is `(10 - 4) - 3`. Pratt
parsing gives each operator a binding power and keeps reading operators only while their power is higher than the
current minimum; equal powers stop for left-associative operators.

</details>

3. What is an environment in your interpreter, and how do nested scopes find a variable?

<details>
<summary>Answer</summary>

The place where variables live: a stack of maps from names to values, one map per scope. A new block pushes a map and
pops it at the end; a lookup tries the innermost map first and then walks outward, so an inner `let` shadows an outer
one until the block ends.

</details>

4. What does WebAssembly give you, and how did you run a module outside the browser?

<details>
<summary>Answer</summary>

A portable, sandboxed binary format that runs at near-native speed on any host with a runtime. You added the targets
with `rustup target add wasm32-wasip1 wasm32-unknown-unknown`, built with `cargo build --target ...`, and ran the
module with the `wasmtime` command-line tool (or from your own program with the `wasmtime` crate).

</details>

### Cloud-native track

1. Why must a reconcile function be level-triggered and idempotent?

<details>
<summary>Answer</summary>

Events can be missed (restarts, network breaks), so reconcile must work from the current state alone. It runs many
times for the same object, including after its own writes, so a run with nothing to fix must change nothing.

</details>

2. What is the difference between a CRD, a custom resource and an operator?

<details>
<summary>Answer</summary>

A CRD adds a type to the API with its schema; a custom resource is one object of that type; an operator is a CRD plus
a controller that reconciles its objects.

</details>

3. When do you use an owner reference, and when a finalizer?

<details>
<summary>Answer</summary>

An owner reference for children in the same namespace: the garbage collector deletes them with the owner. A finalizer
for anything an owner reference cannot reach (other namespaces, outside systems): it delays the deletion until your
cleanup has run.

</details>

4. Your operator works with `cargo run` but logs `forbidden` in the cluster. Why, and how do you find the fix?

<details>
<summary>Answer</summary>

On the Mac it used your admin kubeconfig; in the cluster it runs as its ServiceAccount with only its RBAC role. The
message names the missing verb and resource; check with `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<name>`
and add exactly that verb.

</details>

## Practical tasks

### Task 1 — Your track in one small function

Solve the task of **your** track in `code/p22-checkpoint`. Each one is plain Rust that runs and tests on your Mac:
the small, pure core of something your track does for real. Put it in `src/bin/<name>.rs`, run it with
`cargo run --bin <name>` and test it with `cargo test --bin <name>`. Write at least one test.

**OS track — `os_hex`.** A kernel cannot use `format!` before it has a heap. Write
`fn write_hex(n: u64, buf: &mut [u8; 18]) -> &str` that writes `0x` and exactly 16 lowercase hex digits into `buf`
and returns them as `&str`, using only `core`. `main` prints `0xdead_beef` and `u64::MAX`.

**eBPF track — `ebpf_xdp`.** Write the decision of an XDP firewall as a plain function:
`fn decide(packet: &[u8], blocked: &[[u8; 4]]) -> Action` (`Pass` or `Drop`). Drop an IPv4 packet (Ethernet type
`0x0800` at bytes 12–13) whose source address (bytes 26–29: 14 bytes of Ethernet header plus offset 12 in the IPv4
header) is blocked. Every other packet, including one too short to hold both headers, passes.

**Embedded track — `embedded_debounce`.** Write a `Debouncer` with `fn sample(&mut self, pressed: bool) -> Option<bool>`
that reports a new button state only after `needed` equal samples in a row, and `None` otherwise.

**Databases track — `db_count_by`.** Write `fn count_by(csv: &str, column: &str) -> Result<BTreeMap<String, u64>, String>`,
the engine of `SELECT <column>, COUNT(*) … GROUP BY <column>` over a CSV (comma-separated values) text whose first
line is the header. Unknown columns and short rows are errors.

**Networking track — `net_request_line`.** Write a zero-copy parser for an HTTP request line
(`GET /index.html HTTP/1.1\r\n`) that returns `Complete { method, path, consumed }` with slices into the input,
`Incomplete` when there is no `\r\n` yet, or `Invalid`.

**Compilers track — `compiler_pratt`.** Write `fn eval(src: &str) -> Result<i64, String>` for integers with `+ - * /`
and parentheses, using Pratt parsing (binding powers). `10 - 4 - 3` must be `3`; division by zero is an error.

**Cloud-native track — `cloud_status`.** Write `fn status_for(desired: usize, pods: &[Pod]) -> Status` where a `Pod`
has `ready` and `terminating` flags. Only ready pods that are not terminating count; the phase is `Ready` when that
count reaches `desired`, otherwise `Pending`.

<details>
<summary>Hint</summary>

- OS: a nibble is 4 bits; the digit for position `i` (0 = leftmost) is `(n >> ((15 - i) * 4)) & 0xf`. Index into
  `b"0123456789abcdef"`. `core::str::from_utf8` turns the buffer back into `&str`.
- eBPF: check the length **first**, as the verifier forces you to; `u16::from_be_bytes` reads the network-order type.
- Embedded: keep the stable state, a candidate state and a counter. A sample equal to the stable state resets the counter.
- Databases: `header.split(',').position(...)` finds the column index; `entry(...).or_insert(0)` counts.
- Networking: `buf.windows(2).position(|w| w == b"\r\n")` finds the end; `split(|&b| b == b' ')` on a slice returns
  sub-slices, not copies.
- Compilers: `expr(min_bp)` parses an atom, then loops while the next operator's power is **greater** than `min_bp`,
  calling `expr(power)` for the right side.
- Cloud-native: `pods.iter().filter(...).count()`, and remember that zero desired replicas is already `Ready`.

</details>

<details>
<summary>Answer — OS track</summary>

```rust
/// Writes `n` as `0x` and 16 hexadecimal digits into `buf`, without any allocation.
fn write_hex(n: u64, buf: &mut [u8; 18]) -> &str {
    const DIGITS: &[u8; 16] = b"0123456789abcdef";
    buf[0] = b'0';
    buf[1] = b'x';
    for i in 0..16 {
        let shift = (15 - i) * 4;
        let nibble = ((n >> shift) & 0xf) as usize;
        buf[2 + i] = DIGITS[nibble];
    }
    core::str::from_utf8(buf).expect("only ASCII digits were written")
}

fn main() {
    let mut buf = [0u8; 18];
    println!("{}", write_hex(0xdead_beef, &mut buf));
    println!("{}", write_hex(u64::MAX, &mut buf));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn digits_are_in_the_right_order() {
        let mut buf = [0u8; 18];
        assert_eq!(write_hex(0x1234_abcd, &mut buf), "0x000000001234abcd");
    }
}
```

```text
0x00000000deadbeef
0xffffffffffffffff
```

The function uses only `core`, so it works unchanged inside your kernel, where `buf` can live on the stack.

</details>

<details>
<summary>Answer — eBPF track</summary>

```rust
#[derive(Debug, PartialEq)]
enum Action {
    Pass,
    Drop,
}

const ETH_HEADER: usize = 14;
const ETHERTYPE_IPV4: u16 = 0x0800;

/// Decides like an XDP program: drop IPv4 packets whose source address is blocked.
fn decide(packet: &[u8], blocked: &[[u8; 4]]) -> Action {
    // Every read is checked first, exactly what the eBPF verifier demands.
    if packet.len() < ETH_HEADER + 20 {
        return Action::Pass;
    }
    let ethertype = u16::from_be_bytes([packet[12], packet[13]]);
    if ethertype != ETHERTYPE_IPV4 {
        return Action::Pass;
    }
    let start = ETH_HEADER + 12;
    let source = [
        packet[start],
        packet[start + 1],
        packet[start + 2],
        packet[start + 3],
    ];
    if blocked.contains(&source) {
        Action::Drop
    } else {
        Action::Pass
    }
}

/// Builds a minimal Ethernet + IPv4 header with the given source address.
fn packet_from(source: [u8; 4]) -> Vec<u8> {
    let mut p = vec![0u8; ETH_HEADER + 20];
    p[12] = 0x08; // ethertype 0x0800 = IPv4
    p[13] = 0x00;
    p[ETH_HEADER + 12..ETH_HEADER + 16].copy_from_slice(&source);
    p
}

fn main() {
    let blocked = [[10, 0, 0, 66]];
    println!("{:?}", decide(&packet_from([10, 0, 0, 66]), &blocked));
    println!("{:?}", decide(&packet_from([10, 0, 0, 7]), &blocked));
    println!("{:?}", decide(&[0u8; 10], &blocked));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn short_or_non_ipv4_packets_pass() {
        assert_eq!(decide(&[0u8; 20], &[[0, 0, 0, 0]]), Action::Pass);
        let mut ipv6 = packet_from([1, 2, 3, 4]);
        ipv6[12] = 0x86;
        ipv6[13] = 0xdd;
        assert_eq!(decide(&ipv6, &[[1, 2, 3, 4]]), Action::Pass);
    }
}
```

```text
Drop
Pass
Pass
```

In Aya the same logic reads through `ctx.data()` and `ctx.data_end()` with the same bounds check, and returns
`xdp_action::XDP_DROP` or `XDP_PASS`.

</details>

<details>
<summary>Answer — Embedded track</summary>

```rust
/// Reports a new button state only after `needed` equal samples in a row.
struct Debouncer {
    stable: bool,
    candidate: bool,
    count: u8,
    needed: u8,
}

impl Debouncer {
    fn new(needed: u8) -> Self {
        Debouncer {
            stable: false,
            candidate: false,
            count: 0,
            needed,
        }
    }

    /// Feeds one sample (`true` = pressed). Returns the new state when it changes.
    fn sample(&mut self, pressed: bool) -> Option<bool> {
        if pressed == self.stable {
            self.count = 0;
            return None;
        }
        if pressed != self.candidate {
            self.candidate = pressed;
            self.count = 0;
        }
        self.count += 1;
        if self.count >= self.needed {
            self.stable = pressed;
            self.count = 0;
            Some(pressed)
        } else {
            None
        }
    }
}

fn main() {
    let mut d = Debouncer::new(3);
    let samples = [true, false, true, true, true, true, false, false, false];
    let events: Vec<Option<bool>> = samples.iter().map(|&s| d.sample(s)).collect();
    println!("{events:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn steady_press_and_release_are_reported_once() {
        let mut d = Debouncer::new(2);
        assert_eq!(d.sample(true), None);
        assert_eq!(d.sample(true), Some(true));
        assert_eq!(d.sample(true), None);
        assert_eq!(d.sample(false), None);
        assert_eq!(d.sample(false), Some(false));
    }
}
```

```text
[None, None, None, None, Some(true), None, None, None, Some(false)]
```

On the board, an Embassy task calls `sample` every few milliseconds with the pin level (inverted for an active-low
button) and acts on `Some(...)`.

</details>

<details>
<summary>Answer — Databases track</summary>

```rust
use std::collections::BTreeMap;

/// `SELECT <column>, COUNT(*) FROM csv GROUP BY <column>`: scan, project, aggregate.
fn count_by(csv: &str, column: &str) -> Result<BTreeMap<String, u64>, String> {
    let mut lines = csv.lines();
    let header = lines.next().ok_or("empty input")?;
    let index = header
        .split(',')
        .position(|name| name == column)
        .ok_or(format!("no column {column}"))?;
    let mut counts = BTreeMap::new();
    for (number, line) in lines.enumerate() {
        let value = line
            .split(',')
            .nth(index)
            .ok_or(format!("row {} is too short", number + 1))?;
        *counts.entry(value.to_string()).or_insert(0) += 1;
    }
    Ok(counts)
}

fn main() {
    let csv = "name,city\nAna,Lisbon\nBen,Oslo\nCai,Lisbon\n";
    println!("{:?}", count_by(csv, "city"));
    println!("{:?}", count_by(csv, "country"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_each_group() {
        let csv = "a,b\n1,x\n2,y\n3,x\n";
        let counts = count_by(csv, "b").unwrap();
        assert_eq!(counts["x"], 2);
        assert_eq!(counts["y"], 1);
    }
}
```

```text
Ok({"Lisbon": 2, "Oslo": 1})
Err("no column country")
```

This is a scan, a projection (one column) and a hash aggregation with a count accumulator, all in one pass. A
`BTreeMap` keeps the output sorted, so tests are stable.

</details>

<details>
<summary>Answer — Networking track</summary>

```rust
#[derive(Debug, PartialEq)]
enum Parse<'a> {
    /// A whole request line; `consumed` bytes (including CRLF) can be dropped from the buffer.
    Complete {
        method: &'a [u8],
        path: &'a [u8],
        consumed: usize,
    },
    /// No CRLF yet: read more bytes and try again.
    Incomplete,
    Invalid,
}

/// Parses `METHOD SP PATH SP VERSION CRLF` without copying any bytes.
fn parse_request_line(buf: &[u8]) -> Parse<'_> {
    let Some(end) = buf.windows(2).position(|w| w == b"\r\n") else {
        return Parse::Incomplete;
    };
    let line = &buf[..end];
    let mut parts = line.split(|&b| b == b' ');
    match (parts.next(), parts.next(), parts.next(), parts.next()) {
        (Some(method), Some(path), Some(version), None)
            if !method.is_empty() && path.starts_with(b"/") && version.starts_with(b"HTTP/") =>
        {
            Parse::Complete {
                method,
                path,
                consumed: end + 2,
            }
        }
        _ => Parse::Invalid,
    }
}

fn main() {
    let buf = b"GET /index.html HTTP/1.1\r\nHost: a\r\n";
    if let Parse::Complete {
        method,
        path,
        consumed,
    } = parse_request_line(buf)
    {
        let method = String::from_utf8_lossy(method);
        let path = String::from_utf8_lossy(path);
        println!("method={method} path={path} consumed={consumed}");
    }
    println!("{:?}", parse_request_line(b"GET /index.ht"));
    println!("{:?}", parse_request_line(b"HELLO\r\n"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn slices_point_into_the_buffer() {
        let buf = b"POST /api HTTP/1.1\r\n";
        let Parse::Complete {
            method,
            path,
            consumed,
        } = parse_request_line(buf)
        else {
            panic!("expected a complete line");
        };
        assert_eq!(method, b"POST");
        assert_eq!(path, b"/api");
        assert_eq!(consumed, buf.len());
        assert_eq!(path.as_ptr(), buf[5..].as_ptr()); // no copy was made
    }
}
```

```text
method=GET path=/index.html consumed=26
Incomplete
Invalid
```

The test `slices_point_into_the_buffer` proves zero-copy: the `path` slice starts at the very address of byte 5 of the
input. With `BytesMut` you would call `split_to(consumed)` to cut the line off the buffer.

</details>

<details>
<summary>Answer — Compilers track</summary>

```rust
/// Evaluates integer expressions with + - * / and parentheses, using Pratt parsing.
fn eval(src: &str) -> Result<i64, String> {
    let tokens: Vec<char> = src.chars().filter(|c| !c.is_whitespace()).collect();
    let mut pos = 0;
    let value = expr(&tokens, &mut pos, 0)?;
    if pos != tokens.len() {
        return Err(format!("unexpected '{}'", tokens[pos]));
    }
    Ok(value)
}

/// Binding power of an infix operator: higher binds tighter.
fn binding_power(op: char) -> Option<u8> {
    match op {
        '+' | '-' => Some(1),
        '*' | '/' => Some(2),
        _ => None,
    }
}

fn expr(t: &[char], pos: &mut usize, min_bp: u8) -> Result<i64, String> {
    let mut left = atom(t, pos)?;
    while let Some(&op) = t.get(*pos) {
        let Some(bp) = binding_power(op) else { break };
        if bp <= min_bp {
            break; // left-associative: equal power stops here
        }
        *pos += 1;
        let right = expr(t, pos, bp)?;
        left = match op {
            '+' => left + right,
            '-' => left - right,
            '*' => left * right,
            _ if right == 0 => return Err("division by zero".to_string()),
            _ => left / right,
        };
    }
    Ok(left)
}

fn atom(t: &[char], pos: &mut usize) -> Result<i64, String> {
    match t.get(*pos) {
        Some('(') => {
            *pos += 1;
            let v = expr(t, pos, 0)?;
            if t.get(*pos) != Some(&')') {
                return Err("expected ')'".to_string());
            }
            *pos += 1;
            Ok(v)
        }
        Some(c) if c.is_ascii_digit() => {
            let mut n: i64 = 0;
            while let Some(d) = t.get(*pos).and_then(|c| c.to_digit(10)) {
                n = n * 10 + i64::from(d);
                *pos += 1;
            }
            Ok(n)
        }
        Some(c) => Err(format!("unexpected '{c}'")),
        None => Err("unexpected end".to_string()),
    }
}

fn main() {
    for src in ["1 + 2 * 3", "(1 + 2) * 3", "10 - 4 - 3", "8 / (2 - 2)"] {
        println!("{src} => {:?}", eval(src));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn precedence_and_associativity() {
        assert_eq!(eval("1 + 2 * 3"), Ok(7));
        assert_eq!(eval("10 - 4 - 3"), Ok(3));
        assert_eq!(eval("2 * (3 + 4)"), Ok(14));
    }
}
```

```text
1 + 2 * 3 => Ok(7)
(1 + 2) * 3 => Ok(9)
10 - 4 - 3 => Ok(3)
8 / (2 - 2) => Err("division by zero")
```

`bp <= min_bp` is what makes `-` left-associative: after `10 - 4`, the second `-` has the same power and stops the
inner call, so the outer loop subtracts `3` from `6`.

</details>

<details>
<summary>Answer — Cloud-native track</summary>

```rust
#[derive(Debug, Clone, Copy)]
struct Pod {
    ready: bool,
    terminating: bool,
}

#[derive(Debug, PartialEq)]
struct Status {
    ready_replicas: usize,
    phase: &'static str,
}

/// Computes the status an operator reports for `desired` replicas and the pods it sees.
fn status_for(desired: usize, pods: &[Pod]) -> Status {
    let ready = pods.iter().filter(|p| p.ready && !p.terminating).count();
    let phase = if ready >= desired { "Ready" } else { "Pending" };
    Status {
        ready_replicas: ready,
        phase,
    }
}

fn main() {
    let ok = Pod {
        ready: true,
        terminating: false,
    };
    let starting = Pod {
        ready: false,
        terminating: false,
    };
    let leaving = Pod {
        ready: true,
        terminating: true,
    };
    println!("{:?}", status_for(2, &[ok, starting]));
    println!("{:?}", status_for(2, &[ok, ok, leaving]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn terminating_pods_do_not_count() {
        let leaving = Pod {
            ready: true,
            terminating: true,
        };
        assert_eq!(status_for(1, &[leaving]).ready_replicas, 0);
        assert_eq!(status_for(1, &[leaving]).phase, "Pending");
    }
}
```

```text
Status { ready_replicas: 1, phase: "Pending" }
Status { ready_replicas: 2, phase: "Ready" }
```

This is the pure part of an operator's status logic; in the StaticSite project the count came from the Deployment's
`readyReplicas` instead of a pod list.

</details>

### Task 2 — Explain your project in five minutes (no new code)

Explain your track project out loud, as if to a colleague who did another track: what it does, the two most
important design decisions, one bug you hit and how you found it, and how it is tested. Record yourself or write it
down as the first section of the project's README.

<details>
<summary>Answer</summary>

A good explanation starts with one sentence about the problem, shows the main data flow (a small diagram helps),
names decisions with their trade-off ("we copy the HTML hash into the pod template because…"), and ends with how a
reviewer can run the tests. If you needed more than five minutes, cut details, not the decisions.

</details>

### Task 3 — Plan your next month on the track (no new code)

Pick one open-source project from the "next steps" of your track's last lesson. Read its README and its contribution
guide, build it, run its tests, and find three issues labelled for newcomers. Do not comment or open anything yet;
write a plan with the issue you would start with and why.

<details>
<summary>Answer</summary>

A useful plan names the project and the exact build and test commands that worked for you, explains one issue in your
own words, lists the files you would change, and says how you would test the change. Lessons 24.4 and 24.5 show how to
turn such a plan into your first contribution.

</details>

## If you got something wrong

| Topic | Lesson to review |
|---|---|
| Choosing and working through a track | [22.1 — Choosing a specialization](01-choosing-a-specialization.md) |
| `no_std`, panic handler, entry point | [22.2 — OS track: a freestanding Rust binary](02-os-track-a-freestanding-binary.md) |
| QEMU, linker script, UART, MMIO | [22.3 — OS track: booting a kernel in QEMU](03-os-track-booting-a-kernel-in-qemu.md) |
| Exceptions, vector table, GIC, MMU | [22.4 — OS track: interrupts and paging](04-os-track-interrupts-and-paging.md) |
| What eBPF is, the verifier | [22.6 — eBPF track: what eBPF is](06-ebpf-track-what-ebpf-is.md) |
| Aya, bpf-linker, a first tracepoint program | [22.7 — eBPF track: a first program with Aya](07-ebpf-track-a-first-program-with-aya.md) |
| Maps, kprobes, tracepoint fields | [22.8 — eBPF track: maps and tracing](08-ebpf-track-maps-and-tracing.md) |
| XDP, packet bounds checks | [22.9 — eBPF track: project and next steps](09-ebpf-track-project-and-next-steps.md) |
| Microcontrollers, `no_std`, flashing | [22.10 — Embedded track: microcontrollers and no_std](10-embedded-track-microcontrollers-and-no-std.md) |
| GPIO, `embedded-hal`, debouncing | [22.11 — Embedded track: GPIO and embedded-hal](11-embedded-track-gpio-and-embedded-hal.md) |
| Embassy, tasks, `defmt` | [22.12 — Embedded track: async with Embassy](12-embedded-track-async-with-embassy.md) |
| Plans, Volcano and vectorized execution, Arrow | [22.14 — Databases track: how query engines work](14-databases-track-how-query-engines-work.md) |
| Tokens, SQL syntax trees, sqlparser | [22.15 — Databases track: parsing SQL](15-databases-track-parsing-sql.md) |
| Filter, projection, hash aggregation | [22.16 — Databases track: executing queries](16-databases-track-executing-queries.md) |
| Socket options, `SO_REUSEPORT`, thread-per-core | [22.18 — Networking track: high-performance servers](18-networking-track-high-performance-servers.md) |
| `Bytes`, partial input, zero-copy | [22.19 — Networking track: zero-copy protocol parsing](19-networking-track-zero-copy-parsing.md) |
| Pingora, upstreams, status codes | [22.20 — Networking track: production proxies](20-networking-track-production-proxies.md) |
| Lexer, parser, Pratt parsing | [22.22 — Compilers track: lexing and parsing](22-compilers-track-lexing-and-parsing.md) |
| Evaluation, environments, scopes | [22.23 — Compilers track: a tree-walking interpreter](23-compilers-track-an-interpreter.md) |
| `wasm32` targets, wasmtime | [22.24 — Compilers track: WebAssembly with Rust](24-compilers-track-webassembly.md) |
| Control loops, reconcile, CRDs | [22.26 — Cloud-native track: Kubernetes operators](26-cloud-native-track-kubernetes-operators.md) |
| kube-rs, server-side apply, owner references | [22.27 — Cloud-native track: an operator with kube-rs](27-cloud-native-track-an-operator-with-kube-rs.md) |
| Finalizers, e2e tests, RBAC | [22.28 — Cloud-native track: testing and running operators](28-cloud-native-track-testing-and-running-operators.md) |
| Your track project | the project lesson of your track (see "Before you start") |

## What's next

Phase 23 is the capstone: one larger project of your choice, with a design document, tests, benchmarks and hardening.
Your track is a strong hint for the choice: networking leads naturally to the HTTP reverse proxy, databases and
distributed systems to the key-value store, cloud-native to the observability pipeline (deployed with your own
operator), and the OS, eBPF, embedded and compilers tracks to "your own idea", for example a small kernel with a shell,
an eBPF-based monitoring tool, a sensor firmware or a language that compiles to WebAssembly. Start with
[Lesson 23.1 — How capstone projects work](../23-capstone-projects/01-how-capstone-projects-work.md).

## Sources and further reading

- [Writing an OS in Rust](https://os.phil-opp.com/) and [Rust for Linux](https://rust-for-linux.com/) — OS track.
- [The Aya book](https://aya-rs.dev/book/) and [ebpf.io — What is eBPF?](https://ebpf.io/what-is-ebpf/) — eBPF track.
- [The Embedded Rust Book](https://docs.rust-embedded.org/book/) and [the Embassy book](https://embassy.dev/book/) — embedded track.
- [Apache DataFusion](https://datafusion.apache.org/) — databases track.
- [Pingora](https://github.com/cloudflare/pingora) — networking track.
- [Crafting Interpreters](https://craftinginterpreters.com/) and [Wasmtime documentation](https://docs.wasmtime.dev/) — compilers track.
- [kube.rs](https://kube.rs/) — cloud-native track.
- [rustc — Platform Support](https://doc.rust-lang.org/rustc/platform-support.html) — every target triple Rust supports.

---

[🏠 Start](../START-HERE.md) · [← 22.29 Cloud-native track: project and next steps](29-cloud-native-track-project-and-next-steps.md) · [📋 Phase 22](README.md) · [Phase 23 — Capstone projects →](../23-capstone-projects/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
