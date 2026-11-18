<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.10 DNS: how names become addresses](10-dns-how-names-become-addresses.md) · [📋 Phase 15](README.md) · [15.12 TLS in Rust servers with rustls →](12-tls-in-rust-servers.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/15-networking/11-project-dns-resolver.md) <!-- nav -->

# Lesson 15.11 — Project: a DNS resolver

> **Phase 15 — Networking** · Lesson 11 of 16 · ⏱ about 8 hours (4 sittings)

## What you will learn

- How to turn a protocol specification (RFC 1035, a Request for Comments document) into code: exact bytes out, careful parsing in.
- How to build a DNS query by hand and send it over UDP with a timeout.
- How to parse a binary message safely: bounds checks, compression pointers and pointer loops.
- How to test a network program offline with packets captured from the real network.

## Before you start

- You finished [Lesson 15.10 — DNS: how names become addresses](10-dns-how-names-become-addresses.md). Keep its Section 5 open: the byte layout of the query and the response is the map for this project.
- You know `std::net::UdpSocket` from [Lesson 15.5 — UDP sockets in Rust](05-udp-sockets-in-rust.md) and binary framing from [Lesson 15.7 — Designing a protocol: framing and parsing](07-designing-a-protocol.md).
- You can write integration tests in `tests/` ([Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)) and commit with Git ([Phase 01](../01-git-and-version-control/README.md)).
- Create the project package, as announced in Lesson 15.1. It needs **no external crates**: only the standard library.

```bash
cd ~/projects/rust
cargo new code/p15-dns
cd code/p15-dns
mkdir -p tests/data
```

## New words

| Word | Meaning |
|---|---|
| **RFC** (Request for Comments) | The numbered documents that define internet protocols. RFC 1035 defines the DNS (Domain Name System) message format. |
| **specification** | A precise description of a format or a protocol. Your code must produce and accept exactly what it says. |
| **wire format** | The exact bytes that travel over the network, as opposed to the text that people write (`rust-lang.org`). |
| **fixture** | Fixed test data stored next to the tests. Here: real DNS responses captured with `tcpdump`. |
| **bounds check** | Checking that a position is inside the buffer before reading it. In Rust, `slice.get(i)` returns `None` instead of panicking. |
| **pointer loop** | A malicious or broken compression pointer that points to itself or forward, so a naive parser follows it forever. |
| **UDP, IP, TCP, TTL** | User Datagram Protocol, Internet Protocol, Transmission Control Protocol, Time To Live: the transports and the cache lifetime from Lessons 15.3 and 15.10. |
| **CNAME, MX, NS, SOA** | Canonical NAME, Mail eXchange, Name Server, Start Of Authority: record types from Lesson 15.10. |
| **QR, RD, AA, TC, RCODE** | Query/Response, Recursion Desired, Authoritative Answer, TrunCated, Response CODE: the header flags from Lesson 15.10. |
| **AAAA, AD, NXDOMAIN** | The IPv6 address record (four times the size of an A record); the flag Authenticated Data of DNSSEC (DNS Security Extensions, signed records), which `dig` sets; the RCODE 3, "non-existent domain" (all from Lesson 15.10). |
| **QDCOUNT, ANCOUNT, NSCOUNT, ARCOUNT, RDLENGTH** | Field names from RFC 1035: the numbers of questions, answers, authority records and additional records in the header, and the resource data length in front of a record's data. |
| **I/O** (input/output) | Reading and writing data, here through the network socket. |
| **exit code** | The number a program returns to the shell when it ends: 0 for success, other values for errors (Lesson 11.5). |

## 1. How the resolver works

Your program is a small **stub resolver**: it sends one question to a recursive resolver such as `1.1.1.1` and prints the answer. It never walks the tree itself (that is a stretch goal).

```text
 resolve rust-lang.org --type A
        │
        ▼
 build_query(id, "rust-lang.org", A) ──► 31 bytes ──► UdpSocket.send_to(1.1.1.1:53)
                                                              │
 print records ◄── parse_message(bytes) ◄── 95 bytes ◄── recv_from (3 s timeout)
                         │
                         ├─ header (12 bytes): id, flags, 4 counts
                         ├─ questions:  name, type, class
                         └─ answers / authorities / additionals: records
```

The code is split in two, as in Lesson 15.9: a **library** (`src/lib.rs`) that only turns names into bytes and bytes into structs, and a thin **program** (`src/main.rs`) that parses the command line and uses the network. The library does no I/O (input/output), so you can test every byte of it without a network.

> 🔒 **Security:** a DNS response is **untrusted input** from the network. A broken or hostile packet can claim 65 535 answers, a label longer than the packet, or a pointer that points to itself. Your parser must answer every such packet with an `Err`, never with a panic, a hang or a huge allocation. Use `get()` instead of indexing, and never trust a count or a length field.

## The project

### What you will build

A command-line tool `resolve`:

```text
$ resolve rust-lang.org
rust-lang.org.	233	A	185.199.109.153
rust-lang.org.	233	A	185.199.111.153
rust-lang.org.	233	A	185.199.108.153
rust-lang.org.	233	A	185.199.110.153
$ resolve rust-lang.org --type MX
rust-lang.org.	60	MX	10 mxa.mailgun.org.
rust-lang.org.	60	MX	10 mxb.mailgun.org.
$ resolve no-such-name.rust-lang.org
error: name does not exist (NXDOMAIN)
```

(TTLs and the order of addresses change from run to run: the resolver's cache counts down, and it shuffles the records.)

### Contract

**Command line.**

```text
resolve <name> [--type A|AAAA|MX] [--server <ip>]
```

- `<name>`: a host name, with or without the final dot. Invalid names (empty labels, labels over 63 bytes, a name over 253 characters, characters other than letters, digits, `-` and `_`) are rejected **before** anything is sent.
- `--type`: `A` (default), `AAAA` or `MX`, in upper or lower case.
- `--server`: the IPv4 or IPv6 address of a recursive resolver (default `1.1.1.1`); the port is always 53.
- The binary is called `resolve`. Add this to `Cargo.toml`:

```toml
[[bin]]
name = "resolve"
path = "src/main.rs"
```

**Output and exit codes.**

| Situation | Standard output | Standard error | Exit code |
|---|---|---|---|
| Answer received | one line per answer record: `name<TAB>ttl<TAB>type<TAB>data` | — | 0 |
| The name exists but has no record of this type | `(no records of this type)` | — | 0 |
| RCODE 3 | — | `error: name does not exist (NXDOMAIN)` | 1 |
| Any other RCODE | — | `error: server answered with RCODE <n>` | 1 |
| No answer within 3 seconds | — | `error: no answer from <ip>:53 within 3 s` | 1 |
| TC flag set | — | `error: answer was truncated (TCP fallback is a stretch goal)` | 1 |
| Bad arguments | — | `usage: resolve <name> [--type A\|AAAA\|MX] [--server <ip>]` | 2 |
| Invalid name | — | `error: invalid name: <name>` | 2 |

Names are printed with the final dot. The data of `A`/`AAAA` is the address in normal text form, of `CNAME` and `NS` a name, of `MX` `<preference> <exchange>`. Records of other types are printed as `TYPE<n>` with `(<len> bytes)`.

**Network behaviour.** Use a **random** 16-bit ID for every query and set only the RD flag (`0x0100`). After sending, ignore every datagram that does not come from the server's address, that is not a response (QR = 0) or whose ID differs, and keep waiting until the timeout.

**Library API** (`src/lib.rs`, the crate is `p15_dns`):

```text
pub enum RecordType { A, Ns, Cname, Mx, Aaaa, Other(u16) }
pub enum DnsError { BadName(String), TooShort, PointerLoop, LabelTooLong, BadData(&'static str) }

pub fn build_query(id: u16, name: &str, qtype: RecordType) -> Result<Vec<u8>, DnsError>;
pub fn read_name(msg: &[u8], pos: usize) -> Result<(String, usize), DnsError>;
pub fn parse_message(msg: &[u8]) -> Result<Message, DnsError>;

pub struct Header { pub id: u16, pub flags: u16, pub qdcount: u16, pub ancount: u16, pub nscount: u16, pub arcount: u16 }
pub struct Question { pub name: String, pub qtype: RecordType, pub class: u16 }
pub struct Record { pub name: String, pub rtype: RecordType, pub class: u16, pub ttl: u32, pub data: RData }
pub enum RData { A(Ipv4Addr), Aaaa(Ipv6Addr), Cname(String), Ns(String), Mx { preference: u16, exchange: String }, Unknown(Vec<u8>) }
pub struct Message { pub header: Header, pub questions: Vec<Question>, pub answers: Vec<Record>, pub authorities: Vec<Record>, pub additionals: Vec<Record> }
```

This is the *shape*, not compilable code: add `#[derive(Debug, PartialEq)]` (and `Clone, Copy, Eq` for `RecordType`), `impl Display`, `impl std::error::Error for DnsError`, and helper methods such as `Header::rcode()` as you need them. `read_name` returns the name **with** the final dot and the position right after the name in the message.

**Quality.** `cargo test` passes, `cargo clippy --all-targets` prints no warnings, `cargo fmt --check` is clean, the project has a `README.md` with usage examples, and there is one Git commit per milestone.

### The test fixtures

These are real responses from `1.1.1.1`, captured with `tcpdump` in the container of Lesson 15.10 (`dig … +noedns`). Only the DNS message is kept: the first 28 bytes (IP and UDP headers) were removed. Save each block as a file in `tests/data/`, exactly as shown.

`tests/data/a.hex` — `rust-lang.org A`, four answers:

```text
78 97 81 80 00 01 00 04 00 00 00 00 09 72 75 73 74 2d 6c 61 6e 67 03 6f 72 67 00 00 01 00 01 c0 0c 00 01 00 01 00 00 01 0d 00 04 b9 c7 6e 99 c0 0c 00 01 00 01 00 00 01 0d 00 04 b9 c7 6f 99 c0 0c 00 01 00 01 00 00 01 0d 00 04 b9 c7 6d 99 c0 0c 00 01 00 01 00 00 01 0d 00 04 b9 c7 6c 99
```

`tests/data/mx.hex` — `rust-lang.org MX`, with pointers inside the data:

```text
ec ef 81 80 00 01 00 02 00 00 00 00 09 72 75 73 74 2d 6c 61 6e 67 03 6f 72 67 00 00 0f 00 01 c0 0c 00 0f 00 01 00 00 00 3c 00 10 00 0a 03 6d 78 61 07 6d 61 69 6c 67 75 6e c0 16 c0 0c 00 0f 00 01 00 00 00 3c 00 08 00 0a 03 6d 78 62 c0 31
```

`tests/data/cname.hex` — `www.rust-lang.org A`, a CNAME and four A records:

```text
1f c8 81 80 00 01 00 05 00 00 00 00 03 77 77 77 09 72 75 73 74 2d 6c 61 6e 67 03 6f 72 67 00 00 01 00 01 c0 0c 00 05 00 01 00 00 01 2c 00 15 09 72 75 73 74 2d 6c 61 6e 67 06 67 69 74 68 75 62 02 69 6f 00 c0 2f 00 01 00 01 00 00 0e 10 00 04 b9 c7 6f 99 c0 2f 00 01 00 01 00 00 0e 10 00 04 b9 c7 6c 99 c0 2f 00 01 00 01 00 00 0e 10 00 04 b9 c7 6d 99 c0 2f 00 01 00 01 00 00 0e 10 00 04 b9 c7 6e 99
```

`tests/data/nxdomain.hex` — `no-such-name.rust-lang.org A`: RCODE 3 and an SOA record in the authority section:

```text
ba d7 81 83 00 01 00 00 00 01 00 00 0c 6e 6f 2d 73 75 63 68 2d 6e 61 6d 65 09 72 75 73 74 2d 6c 61 6e 67 03 6f 72 67 00 00 01 00 01 c0 19 00 06 00 01 00 00 03 84 00 48 06 6e 73 2d 36 38 33 09 61 77 73 64 6e 73 2d 32 31 03 6e 65 74 00 11 61 77 73 64 6e 73 2d 68 6f 73 74 6d 61 73 74 65 72 06 61 6d 61 7a 6f 6e 03 63 6f 6d 00 00 00 00 01 00 00 1c 20 00 00 03 84 00 12 75 00 00 01 51 80
```

Tests read them with this helper, which you can copy into `tests/fixtures.rs`:

```rust
/// Reads a file of hex bytes such as "78 97 81 80 ...".
fn fixture(name: &str) -> Vec<u8> {
    let path = format!("{}/tests/data/{name}", env!("CARGO_MANIFEST_DIR"));
    let text = std::fs::read_to_string(path).unwrap();
    text.split_whitespace()
        .map(|h| u8::from_str_radix(h, 16).unwrap())
        .collect()
}
```

`CARGO_MANIFEST_DIR` is the folder of `Cargo.toml`, so the path works no matter where you start `cargo test`.

> 🧪 **Testing:** fixtures from the real network are the best parser tests you can get: they contain exactly the details (pointers, record order, extra sections) that real servers send. Because they are files, the tests run offline, fast, and give the same result every time. Add a new fixture whenever you meet a packet your parser gets wrong.

### Step-by-step plan

#### M1 — Build the query

**Goal:** `build_query(0x7897, "rust-lang.org", RecordType::A)` returns exactly the 31 query bytes from Lesson 15.10, except that the flags are `01 00` (we set only RD; `dig` also set AD).

**Guidance:**

1. Write `RecordType` with a method `code()` (A = 1, NS = 2, CNAME = 5, MX = 15, AAAA = 28) and `DnsError`.
2. Write a private `encode_name(name, &mut Vec<u8>)`: strip one final dot, check the rules of the contract (reuse your `is_valid_name` from Exercise 3 of Lesson 15.10 if you did it, and also allow `_`, which real names such as `_dmarc.example.com` use and the contract permits), then push for each label its length byte and its bytes, and finally a `0`.
3. `build_query` pushes the 12-byte header (`id.to_be_bytes()`, flags `0x0100`, QDCOUNT 1, three zero counts), the encoded name, the type and class `1`, all big-endian.
4. Write a unit test in `src/lib.rs` that compares with the expected bytes, and one for bad names.

<details>
<summary>Hint</summary>

Write the expected bytes so that you can read them: `9, b'r', b'u', b's', b't', b'-', b'l', b'a', b'n', b'g', 3, b'o', b'r', b'g', 0`. `u16::to_be_bytes()` returns `[u8; 2]`, which `extend_from_slice(&…)` appends. Use `Vec::with_capacity(32)` if you like; it is not required.

</details>

**Done when:**

```bash
cargo test --lib
git commit -am "M1: build DNS queries"
```

```text
running 2 tests
test tests::builds_the_query_from_the_capture ... ok
test tests::rejects_bad_names ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Your test names may differ; the count and `ok` matter.)

#### M2 — Send it and get bytes back

**Goal:** `src/main.rs` builds a query for `rust-lang.org`, sends it to `1.1.1.1:53` and prints how many bytes came back.

**Guidance:**

1. `UdpSocket::bind("0.0.0.0:0")` lets the OS (operating system) pick a free local port; `set_read_timeout(Some(Duration::from_secs(3)))` limits the wait.
2. `send_to(&query, server)` sends one datagram. A buffer of 512 bytes is enough: that is the classic DNS limit over UDP without EDNS (Extension Mechanisms for DNS, Lesson 15.10).
3. `recv_from(&mut buf)` returns the length and the sender. Compare the sender with the server and the first two bytes with your ID.
4. For the ID, use something unpredictable. Without external crates, `std::hash::RandomState::new().hash_one(std::time::SystemTime::now()) as u16` gives a different value every run (see the security note in "Common mistakes").
5. Print a temporary debug line. You will replace it in M5.

<details>
<summary>Hint</summary>

The ID is in the first two bytes: `u16::from_be_bytes([buf[0], buf[1]])`. If `recv_from` fails because of the timeout, the error kind is `WouldBlock` or `TimedOut` depending on the OS; for now, just pass it up with `?`.

</details>

**Done when:**

```bash
cargo run -q -- rust-lang.org
git commit -am "M2: send the query over UDP"
```

```text
sent 31 bytes, received 95 bytes from 1.1.1.1:53, id 0x1234 (sent 0x1234)
```

(Your ID differs every time; both IDs must be equal. The number of received bytes can change if the site changes its records.)

#### M3 — Parse the header and the question

**Goal:** `parse_message` reads the header and the questions; the answers can stay empty for now.

**Guidance:**

1. Write a small private `Reader { msg: &[u8], pos: usize }` with methods `u8()`, `u16()`, `u32()` and `bytes(n)`. Each one uses `msg.get(…)` and returns `Err(DnsError::TooShort)` instead of panicking. Every other part of the parser reads only through it.
2. Parse the six header fields, then `qdcount` questions: name (in this milestone you may read only plain labels, without pointers), type, class.
3. Create `tests/fixtures.rs` with the `fixture` helper and a first test on `a.hex`: ID `0x7897`, one question named `rust-lang.org.`, `ancount` 4.

<details>
<summary>Hint</summary>

`bytes(n)` must not overflow: compute the end with `self.pos.checked_add(n)`, then `self.msg.get(self.pos..end)`. Do not create a `Vec` with the capacity of a count from the packet: a hostile `ancount` of 65 535 should cost nothing until the records really exist.

</details>

**Done when:**

```bash
cargo test --test fixtures
git commit -am "M3: parse the header and the question"
```

```text
running 1 test
test parses_the_header_and_question ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

#### M4 — Names with compression

**Goal:** `read_name(msg, pos)` handles labels, pointers and the end byte, and returns an error for every broken name.

**Guidance:**

1. Read the byte at `pos`. `0` ends the name. A value with the top two bits set (`b & 0xC0 == 0xC0`) is a pointer: the offset is `((b & 0x3F) << 8) | next_byte`. A value from 1 to 63 is a label length. Anything else (64 to 191) is an error.
2. After the **first** pointer, remember `pos + 2`: that is where the caller continues, because the name in *this* place of the message ends with the pointer. Labels read after jumping do not move this position.
3. Protect against loops: a pointer must point **backwards** (`target < pos`), and allow at most a fixed number of jumps (for example 32).
4. Build the text: each label followed by a dot. An empty name (only the `0` byte) is the root, `"."`.
5. Use `read_name` for question names and test it: a pointer to itself (`c0 0c` at offset 12) gives `PointerLoop`; a cut label gives `TooShort`.

<details>
<summary>Hint</summary>

Keep two variables: `pos` (where you are reading now) and `after: Option<usize>` (where the caller continues). Return `after.unwrap_or(pos)`. A minimal message for the loop test: twelve zero bytes followed by `0xc0, 0x0c`.

</details>

**Done when:**

```bash
cargo test
git commit -am "M4: names with compression pointers"
```

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The first line is the unit tests, the second `tests/fixtures.rs`; `cargo test` also prints empty results for the binary and the doc tests.)

#### M5 — Records and the command line

**Goal:** the full contract: all sections parsed, records printed, options and exit codes.

**Guidance:**

1. A record is: name, type, class, TTL (`u32`), RDLENGTH, then RDLENGTH bytes of data. Read the data with `bytes(len)` first, so the position is always right, even for types you do not understand.
2. Decode the data by type: A needs exactly 4 bytes, AAAA exactly 16 (`Ipv4Addr::from([u8; 4])`, `Ipv6Addr::from([u8; 16])`). For CNAME and NS call `read_name(msg, start)` where `start` is the beginning of the data: names inside data may contain pointers into the *whole* message. For MX: a 2-byte preference, then a name at `start + 2`. Other types become `Unknown`.
3. Parse `ancount` answers, `nscount` authorities and `arcount` additionals.
4. `impl Display for Record` in the contract format (`{name}\t{ttl}\t{type}\t{data}`).
5. In `main`: parse the arguments by hand (`std::env::args()` and a `match` on `"--type"` and `"--server"`), check the response as the contract says, print the answers, and return `std::process::ExitCode`.

<details>
<summary>Hint</summary>

Converting a slice into an array checks the length for you: `let b: [u8; 4] = raw.try_into().map_err(|_| DnsError::BadData("A needs 4 bytes"))?;`. For the exit code, `fn main() -> ExitCode` and `ExitCode::from(2)`. To turn the timeout into the contract message: `.map_err(|_| format!("no answer from {server} within 3 s"))?`. `192.0.2.1` is a documentation address that never answers, which makes it perfect for testing the timeout.

</details>

**Done when:**

```bash
cargo run -q -- www.rust-lang.org
cargo run -q -- rust-lang.org --type AAAA --server 8.8.8.8
cargo run -q -- no-such-name.rust-lang.org; echo "exit $?"
cargo run -q -- bad..name; echo "exit $?"
cargo run -q -- rust-lang.org --type TXT; echo "exit $?"
cargo run -q -- rust-lang.org --server 192.0.2.1; echo "exit $?"
git commit -am "M5: records, options and exit codes"
```

```text
www.rust-lang.org.	300	CNAME	rust-lang.github.io.
rust-lang.github.io.	3600	A	185.199.108.153
rust-lang.github.io.	3600	A	185.199.109.153
rust-lang.github.io.	3600	A	185.199.110.153
rust-lang.github.io.	3600	A	185.199.111.153
rust-lang.org.	300	AAAA	2606:50c0:8002::153
rust-lang.org.	300	AAAA	2606:50c0:8001::153
rust-lang.org.	300	AAAA	2606:50c0:8003::153
rust-lang.org.	300	AAAA	2606:50c0:8000::153
error: name does not exist (NXDOMAIN)
exit 1
error: invalid name: bad..name
exit 2
usage: resolve <name> [--type A|AAAA|MX] [--server <ip>]
exit 2
error: no answer from 192.0.2.1:53 within 3 s
exit 1
```

Compare a few answers with `dig +short <name> <type>`: the sets of values must be equal (the order and the TTLs may differ).

#### M6 — Fixture tests, README and quality gate

**Goal:** tests for every fixture, a README and a clean `clippy`/`fmt`.

**Guidance:**

1. In `tests/fixtures.rs`, have one test per fixture (extend the M3 test for `a.hex` instead of adding a second one): the four A records (check the first one: name, TTL 269, `185.199.110.153`); the MX exchanges `10 mxa.mailgun.org.` and `10 mxb.mailgun.org.` (this tests pointers *inside* data); the CNAME `rust-lang.github.io.` followed by records for that name; the NXDOMAIN fixture with RCODE 3, no answers and one authority record of type 6 (SOA, which your parser keeps as `Other(6)`).
2. Add a test to `tests/fixtures.rs` that cuts a fixture at every length and checks that `parse_message` never panics (see the hint).
3. Write `README.md`: what the tool does, the usage line, two examples, how to run the tests.

<details>
<summary>Hint</summary>

The "cut everywhere" test, a cheap form of fuzzing (Lesson 09.8 shows real fuzzing):

```rust
#[test]
fn no_prefix_of_a_real_packet_panics() {
    let full = fixture("mx.hex");
    for len in 0..full.len() {
        let _ = parse_message(&full[..len]); // must return, Ok or Err
    }
}
```

</details>

**Done when:**

```bash
cargo fmt --check && cargo clippy --all-targets && cargo test
git add . && git commit -m "M6: fixture tests and README"
```

```text
    Checking p15-dns v0.1.0 (/Users/you/projects/rust/code/p15-dns)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.24s
...
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(`clippy` prints `Checking` and `Finished` and no warnings; your path differs. The numbers of tests are those of the plan above: four unit tests and five fixture tests.)

### Stretch goals

1. **Iterative resolution from the root** (`--trace`). Send queries with RD = 0 (flags `0x0000`), starting at `198.41.0.4` (`a.root-servers.net`). If the answer section is empty, take the NS records from the authority section and an A record for one of them from the additional section (glue), and ask that server next. Stop when you get an answer with the AA flag. Print each step, like `dig +trace`. If a referral comes without glue, resolve the NS name first (a recursive call). Limit the number of steps.
2. **TCP fallback.** When TC is set, send the same query over TCP port 53 with a 2-byte big-endian length in front, and read the answer the same way (`read_exact` for the length, then for the message).
3. **More record types**: TXT (one or more length-prefixed strings), SOA (two names and five `u32` numbers), and `--type` for them.
4. **A cache** in a long-running mode (`resolve --repl`): keep answers in a `HashMap<(String, RecordType), (Instant, Vec<Record>)>` and serve them until the TTL runs out.
5. **Compare with hickory.** Write a test (marked `#[ignore]`, because it needs the network) that resolves a list of names with your code and with `hickory-resolver` (Lesson 15.10) and compares the sets of addresses.
6. **Capture your own fixture**: capture a response for `crates.io` with `tcpdump -x` in the container, cut off the first 28 bytes, and add a test for it.

## Common mistakes

**1. Reading the next record from the wrong place after a pointer.** If `read_name` returns the position where the *jumped-to* name ended, the next field is read from the middle of the question, and you get garbage types and huge lengths. The position after a compressed name is always "right after the first pointer", 2 bytes after it started.

**2. Indexing with `msg[pos]`.** One short packet then panics your program, and in a server a panic per packet is a denial of service. Use `get()` everywhere, through one small reader, so that there is only one place to get it right.

**3. Forgetting that names inside data can be compressed.** CNAME, NS and MX data contain names, and servers compress them (the MX fixture ends with `c0 16` and `c0 31`). Parse them with `read_name` on the **whole message**, not on the data slice alone: a pointer is an offset from the start of the message.

**4. Accepting any answer.** If you do not check the source address and the ID, a late answer to an earlier query, or a forged one, is printed as the truth. This is exactly how DNS spoofing works.

> 🔒 **Security:** the ID from `RandomState` is fine for learning, but it is not a cryptographic random number. A real resolver uses a cryptographically secure generator (for example the `getrandom` crate) for the ID **and** sends each query from a random source port (which `bind("0.0.0.0:0")` already gives you). Together they make a forged answer very hard to guess.

## Quiz

1. Why does the library do no network I/O at all?

<details>
<summary>Answer</summary>

So that all of the protocol logic (building and parsing bytes) can be tested offline, quickly and repeatably with fixtures. The program around it only connects the library to a socket.

</details>

2. A name in the answer section is `c0 0c`. Where does the next field (the type) start?

<details>
<summary>Answer</summary>

Two bytes after the start of the name, right after the pointer. Where the pointer leads (offset 12) does not change the position in the answer.

</details>

3. How does your parser make sure that a pointer loop cannot hang it?

<details>
<summary>Answer</summary>

Every pointer must point to an earlier position than the one being read, and the number of jumps is limited. A pointer to itself or forward is rejected with `PointerLoop`.

</details>

4. Why does the CNAME fixture contain `c0 2f`, and what is at offset `0x2f`?

<details>
<summary>Answer</summary>

`c0 2f` is a pointer to offset 47, where the CNAME's data starts: the name `rust-lang.github.io.`. The four A records use it as their name instead of repeating the labels.

</details>

5. Why should the program read the RDLENGTH bytes before it decodes the data?

<details>
<summary>Answer</summary>

So that the position of the next record is always correct (start of data + RDLENGTH), even for record types the program does not understand, and so that a too-long RDLENGTH is caught as `TooShort` before decoding.

</details>

6. What exit code does `resolve` return for NXDOMAIN, and why is it not 0?

<details>
<summary>Answer</summary>

1. The program did not get what it was asked for, so a script that uses it (`resolve name && connect`) must be able to see the failure. Bad usage returns 2, a different code, so scripts can tell the two cases apart.

</details>

## Summary

- A stub resolver is small: build 12 header bytes and a question, send one UDP datagram, wait with a timeout, check source and ID, parse.
- Names on the wire are length-prefixed labels ending with `0`; a byte `0xC0` or above starts a 2-byte compression pointer to an offset in the whole message.
- After a compressed name, parsing continues right after the first pointer; pointers must go backwards and jumps are limited.
- Every read goes through bounds-checked helpers, so any broken packet gives an `Err`, never a panic or a hang.
- Real captured packets as fixtures make the parser testable offline; cutting them at every length is a cheap robustness test.
- The library/program split, exit codes and a contract make the tool easy to test and to use from scripts.

## Sources and further reading

- [RFC 1035 — Domain Names: Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035) — section 4.1 is the message format and 4.1.4 the compression scheme you implemented.
- [RFC 3596 — DNS Extensions to Support IP Version 6](https://www.rfc-editor.org/rfc/rfc3596) — defines the AAAA record (type 28).
- [IANA — DNS Parameters](https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml) — the official list of record types, classes and RCODEs.
- [std::net::UdpSocket](https://doc.rust-lang.org/std/net/struct.UdpSocket.html) — `bind`, `send_to`, `recv_from` and `set_read_timeout`.
- [Emil Hernvall — Building a DNS server in Rust](https://github.com/EmilHernvall/dnsguide) — a free step-by-step guide that goes on to a recursive resolver and a server.

---

[🏠 Start](../START-HERE.md) · [← 15.10 DNS: how names become addresses](10-dns-how-names-become-addresses.md) · [📋 Phase 15](README.md) · [15.12 TLS in Rust servers with rustls →](12-tls-in-rust-servers.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
