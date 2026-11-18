<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.14 Project: a replicated key-value store](14-project-replicated-key-value-store.md) · [📋 Phase 18](README.md) · [Phase 19 — Performance and profiling →](../19-performance-and-profiling/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/15-checkpoint-stage-4-complete.md) <!-- nav -->

# Lesson 18.15 — Checkpoint: Stage 4 complete

> **Phase 18 — Distributed systems** · Lesson 15 of 15 · ⏱ about 120 minutes

## What you will learn

- Whether you really own the material of Stage 4 (Phases 15–18): networks and protocols, backend services, storage engines and distributed systems.
- How to combine these topics in four small practical tasks, each with tests.
- Which lessons to review, with direct links.
- What Stage 5 (mastery and career, starting with performance and profiling) expects from you.

## Before you start

- You finished [Lesson 18.14 — Project: a replicated key-value store](14-project-replicated-key-value-store.md), at least up to milestone M5, and merged its branch.
- Plan about 60 minutes for the quiz and 90 minutes for the tasks. Answer the quiz **from memory**, on paper or in your notes, and only then open the hidden answers.
- The tasks go into one new package:

```bash
cd ~/projects/rust
cargo new --lib code/p18-checkpoint
```

Each task is one module. Put this into `src/lib.rs` now and create the four files as you reach each task:

```rust
pub mod cache;
pub mod frames;
pub mod quorum;
pub mod wal;
```

## New words

| Word | Meaning |
|---|---|
| **Stage 4** | "Build real systems": Phase 15 (networking), Phase 16 (backend services), Phase 17 (storage and database internals) and Phase 18 (distributed systems). |
| **torn write** | A record that was only partly written to disk when the machine crashed. A log reader must detect it and stop there. |
| **TTL** (Time To Live) | How long a cached value stays valid. After its TTL, the value counts as missing. |

## Stage 4 at a glance

| Phase | What you learned | Your main project |
|---|---|---|
| 15 Networking | Layers and packets, IP (Internet Protocol) addresses and routing, TCP (Transmission Control Protocol) and UDP (User Datagram Protocol) sockets, async networking with Tokio, framing, HTTP (HyperText Transfer Protocol) from scratch, hyper and reqwest, DNS (Domain Name System), TLS (Transport Layer Security), QUIC and HTTP/3, proxies and load balancers | A DNS resolver; a TCP load balancer |
| 16 Backend services | REST (Representational State Transfer) APIs (Application Programming Interfaces), axum, errors and middleware, SQL (Structured Query Language) and PostgreSQL, sqlx, migrations and transactions, authentication, gRPC (a remote procedure call framework), Redis caching, NATS messaging, logs, metrics and traces, configuration and health checks, testing services | A URL (Uniform Resource Locator) shortener service |
| 17 Storage and database internals | Disks and files, `fsync`, binary formats, the WAL (write-ahead log), hash indexes, B-trees, LSM (log-structured merge) trees, compaction and Bloom filters, the page cache, ACID (atomicity, consistency, isolation, durability) transactions, isolation levels, MVCC (multi-version concurrency control), how a SQL query runs | Your own key-value store |
| 18 Distributed systems | Failures and timeouts, retries and idempotency, physical and logical clocks, replication, consistency models, partitioning, Raft elections and log replication, deterministic simulation, sagas | A replicated key-value store on Raft |

## Self-test quiz

Answer from memory. Each answer is hidden; open it after you wrote yours.

**Phase 15: networking**

1. TCP gives you a *byte stream*. Why does every protocol on top of TCP need *framing*, and name two ways to do it.

<details>
<summary>Answer</summary>

TCP delivers bytes in order but does not keep message boundaries: one `read` may return half a message or two and a half messages. The receiver must find where each message ends. Two ways: a delimiter (for example a newline, as in HTTP headers and our line protocols) or a length prefix (a fixed-size length before each message, as in Task 1).

</details>

2. When would you choose UDP over TCP?

<details>
<summary>Answer</summary>

When a late message is worthless or the application handles loss itself: DNS queries, live audio and video, games, and protocols such as QUIC that build their own reliability on top of UDP. UDP has no connection set-up and no head-of-line blocking, but also no delivery or order guarantees.

</details>

3. What does a TLS certificate prove to the client, and what does it not prove?

<details>
<summary>Answer</summary>

It proves that the server holds the private key for a public key that a trusted certificate authority signed for that domain name, so the client is talking to the real owner of the name and the connection is encrypted. It does not prove that the server is honest, safe or bug-free.

</details>

4. What is the difference between a layer-4 and a layer-7 load balancer?

<details>
<summary>Answer</summary>

A layer-4 (transport) balancer forwards TCP connections or UDP packets without understanding their content, like the TCP load balancer project. A layer-7 (application) balancer understands the protocol, for example HTTP, and can route by path or header, retry requests and terminate TLS.

</details>

**Phase 16: backend services**

5. Which of the HTTP methods `GET`, `POST`, `PUT` and `DELETE` are idempotent, and why does it matter for clients?

<details>
<summary>Answer</summary>

`GET`, `PUT` and `DELETE` are idempotent: doing them twice has the same effect as once. `POST` is not. A client (or a proxy) may safely retry an idempotent request after a timeout; a `POST` needs an idempotency key to be retried safely.

</details>

6. How do you prevent SQL injection with sqlx?

<details>
<summary>Answer</summary>

Never build SQL by gluing strings with user input. Use placeholders (`$1`, `$2`) and pass the values with `.bind(...)` (or the `query!` macros); the database receives the values separately from the SQL text, so input can never become code.

</details>

7. What is the difference between core NATS publish/subscribe and JetStream, in terms of delivery?

<details>
<summary>Answer</summary>

Core NATS is *at most once*: a message goes to the subscribers connected at that moment and is lost if nobody listens. JetStream stores messages in streams and gives *at least once* delivery with acknowledgements, so consumers must handle duplicates.

</details>

8. What is the difference between a liveness check and a readiness check?

<details>
<summary>Answer</summary>

Liveness says "the process is not stuck; do not restart me". Readiness says "I can serve traffic now" (for example, the database connection works). A service can be alive but not ready; the load balancer then stops sending it requests without restarting it.

</details>

**Phase 17: storage and database internals**

9. Why must a database call `fsync` before it tells the client "committed"?

<details>
<summary>Answer</summary>

A successful `write` only puts the data into the operating system's page cache in memory. A power loss can still lose it. `fsync` returns only after the data reached the disk, so the promise "committed" survives a crash.

</details>

10. What does a write-ahead log guarantee, and what must a reader do with the last record after a crash?

<details>
<summary>Answer</summary>

Every change is first appended (and synced) to the log, then applied to the data files, so after a crash replaying the log restores every committed change. The last record may be torn (only partly written); the reader detects it with a length and a checksum and stops there (Task 3).

</details>

11. B-tree or LSM tree: which one is usually better for write-heavy workloads, and what does it pay for that?

<details>
<summary>Answer</summary>

The LSM tree: writes go to an in-memory table and are flushed as sorted files sequentially. It pays with more work on reads (several files may hold the key; Bloom filters help) and with background compaction.

</details>

12. What does MVCC let readers and writers do?

<details>
<summary>Answer</summary>

Readers see a consistent snapshot of old versions while writers create new versions, so readers do not block writers and writers do not block readers. Old versions are cleaned up later (vacuum or garbage collection).

</details>

**Phase 18: distributed systems**

13. A request timed out. What do you know about whether the server performed it?

<details>
<summary>Answer</summary>

Nothing. The request may have been lost, the server may have performed it and the answer was lost, or the server may still be working. That is why retries need idempotent operations or idempotency keys.

</details>

14. Lamport clocks give every event a number. What does "a < b" for two Lamport timestamps tell you, and what does it not tell you?

<details>
<summary>Answer</summary>

If event A happened before event B (A could have influenced B), then A's timestamp is smaller. The reverse does not hold: a smaller timestamp does not prove that A happened before B; the events may be concurrent. Vector clocks can tell the two cases apart.

</details>

15. In leaderless replication with `N` replicas, a write waits for `W` acknowledgements and a read asks `R` replicas. Which condition makes every read overlap the latest write, and why?

<details>
<summary>Answer</summary>

`R + W > N`. Then every read group and every write group share at least one replica, the same overlap argument as Raft's majorities.

</details>

16. Why does consistent hashing move only a few keys when a node is added?

<details>
<summary>Answer</summary>

Keys and nodes are placed on the same ring of hash values, and each key belongs to the next node on the ring. A new node takes over only the keys between itself and the previous node; all other keys keep their owner. With `hash(key) % n`, almost every key would move.

</details>

17. A Raft leader of term 5 sees that an entry of term 3 is stored on a majority. May it mark the entry committed?

<details>
<summary>Answer</summary>

Not by counting copies of that entry. It commits only entries of its own term by counting; once an entry of term 5 after it is on a majority, the term-3 entry is committed with it (Figure 8 of the paper; Lesson 18.11). The no-op entry at the start of a term makes this happen quickly.

</details>

18. Why should every simulation test print its seed?

<details>
<summary>Answer</summary>

Because the seed is the only input that decides the run. With it, a failure can be replayed exactly, as often as needed, on any machine; without it, a rare failure may never be seen again.

</details>

## Practical tasks

All four tasks go into `code/p18-checkpoint`. Each one has a test you must write; run them all with `cargo test`.

### Task 1 — Frames from a byte stream (Phase 15)

Write `src/frames.rs` with `pub fn take_frames(buf: &mut Vec<u8>) -> Vec<Vec<u8>>`. `buf` holds bytes received so far from a TCP stream. Each frame is a 4-byte big-endian length followed by that many payload bytes. Return every **complete** frame from the front of `buf` and remove those bytes; leave an incomplete frame in `buf` for the next call. Tests: two and a half frames arrive at once (two are returned, half stays), and the same stream fed byte by byte gives the same frames, including an empty one.

<details>
<summary>Hint</summary>

Loop: stop if fewer than 4 bytes; read the length with `u32::from_be_bytes`; stop if fewer than `4 + len` bytes; otherwise copy the payload and `drain(..4 + len)`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Take every complete frame (4-byte big-endian length + payload) from the
/// front of `buf`. An incomplete frame stays in `buf` for the next read.
pub fn take_frames(buf: &mut Vec<u8>) -> Vec<Vec<u8>> {
    let mut frames = Vec::new();
    loop {
        if buf.len() < 4 {
            break;
        }
        let len = u32::from_be_bytes([buf[0], buf[1], buf[2], buf[3]]) as usize;
        if buf.len() < 4 + len {
            break; // the rest has not arrived yet
        }
        frames.push(buf[4..4 + len].to_vec());
        buf.drain(..4 + len);
    }
    frames
}

#[cfg(test)]
mod tests {
    use super::*;

    fn frame(payload: &[u8]) -> Vec<u8> {
        let mut out = (payload.len() as u32).to_be_bytes().to_vec();
        out.extend_from_slice(payload);
        out
    }

    #[test]
    fn two_frames_and_a_half() {
        let mut buf = frame(b"hello");
        buf.extend(frame(b"world"));
        let third = frame(b"later");
        buf.extend(&third[..6]); // only part of the third frame arrived

        assert_eq!(
            take_frames(&mut buf),
            [b"hello".to_vec(), b"world".to_vec()]
        );
        assert_eq!(buf, &third[..6]);

        buf.extend(&third[6..]); // the rest arrives
        assert_eq!(take_frames(&mut buf), [b"later".to_vec()]);
        assert!(buf.is_empty());
    }

    #[test]
    fn byte_by_byte_gives_the_same_frames() {
        let mut stream = frame(b"a");
        stream.extend(frame(b""));
        stream.extend(frame(b"bcd"));
        let mut buf = Vec::new();
        let mut got = Vec::new();
        for byte in stream {
            buf.push(byte);
            got.extend(take_frames(&mut buf));
        }
        assert_eq!(got, [b"a".to_vec(), b"".to_vec(), b"bcd".to_vec()]);
    }
}
```

Why it works: the function never consumes bytes of an unfinished frame, so it does not matter how the network splits the stream. The byte-by-byte test proves it for the worst split.

</details>

### Task 2 — A cache with a TTL (Phase 16)

Write `src/cache.rs` with a `TtlCache` that has `put(key, value, ttl_ms, now_ms)` and `get(key, now_ms) -> Option<String>`. The time is passed in, so the test needs no real clock. A value is valid while `now_ms < put_time + ttl_ms`; an expired value is removed on `get`. Test the last valid millisecond and the first expired one.

<details>
<summary>Hint</summary>

Store `(value, expires_at)` in a `HashMap`. A `match` with a guard (`Some((value, expires)) if now_ms < *expires`) keeps `get` short.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;

/// A cache with a time to live. Time is passed in (milliseconds), so tests
/// do not need a real clock.
#[derive(Default)]
pub struct TtlCache {
    entries: HashMap<String, (String, u64)>, // value, expires at
}

impl TtlCache {
    pub fn put(&mut self, key: &str, value: &str, ttl_ms: u64, now_ms: u64) {
        self.entries
            .insert(key.to_string(), (value.to_string(), now_ms + ttl_ms));
    }

    pub fn get(&mut self, key: &str, now_ms: u64) -> Option<String> {
        match self.entries.get(key) {
            Some((value, expires)) if now_ms < *expires => Some(value.clone()),
            Some(_) => {
                self.entries.remove(key); // expired: forget it
                None
            }
            None => None,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn an_entry_lives_exactly_its_ttl() {
        let mut cache = TtlCache::default();
        cache.put("user:7", "Ada", 1_000, 5_000);
        assert_eq!(cache.get("user:7", 5_999), Some("Ada".to_string()));
        assert_eq!(cache.get("user:7", 6_000), None);
        assert_eq!(cache.get("user:8", 5_000), None);
    }
}
```

Why it works: passing time in as a value is the same idea as `tick()` in the Raft core and the simulated clock in `turmoil`: code that does not read the clock itself is easy to test at exact moments.

</details>

### Task 3 — Replaying a log after a crash (Phase 17)

Write `src/wal.rs` with `append(log: &mut Vec<u8>, payload: &[u8])`, which adds a record (4-byte length, 4-byte checksum, payload), and `replay(log: &[u8]) -> Vec<Vec<u8>>`, which returns the payloads from the start and stops at the first record that is cut off or whose checksum does not match. Use this toy checksum (FNV-1a, a simple hash named after its authors Fowler, Noll and Vo):

```rust
/// A toy checksum (FNV-1a, 32 bits). Real logs use CRC32C.
pub fn checksum(data: &[u8]) -> u32 {
    let mut hash: u32 = 0x811c_9dc5;
    for &b in data {
        hash ^= b as u32;
        hash = hash.wrapping_mul(0x0100_0193);
    }
    hash
}
```

Tests: a clean log of three records replays all three; cutting 1 to 12 bytes off the end (a torn last record) gives two; flipping one bit in the second record's payload gives only the first.

<details>
<summary>Hint</summary>

Keep a position `pos`. While at least 8 header bytes are left: read the length and the checksum with `u32::from_le_bytes(log[pos..pos + 4].try_into().unwrap())`; check that the whole payload is there; check the checksum; push the payload and move on.

</details>

<details>
<summary>Answer</summary>

```rust
/// Append one record: length (4 bytes), checksum (4 bytes), payload.
pub fn append(log: &mut Vec<u8>, payload: &[u8]) {
    log.extend((payload.len() as u32).to_le_bytes());
    log.extend(checksum(payload).to_le_bytes());
    log.extend(payload);
}

/// Read records from the start. Stop at the first record that is cut off
/// or whose checksum does not match: everything after it is not trusted.
pub fn replay(log: &[u8]) -> Vec<Vec<u8>> {
    let mut records = Vec::new();
    let mut pos = 0;
    while pos + 8 <= log.len() {
        let len = u32::from_le_bytes(log[pos..pos + 4].try_into().unwrap()) as usize;
        let sum = u32::from_le_bytes(log[pos + 4..pos + 8].try_into().unwrap());
        let start = pos + 8;
        if start + len > log.len() {
            break; // a torn write: the crash happened in the middle
        }
        let payload = &log[start..start + len];
        if checksum(payload) != sum {
            break; // corrupted
        }
        records.push(payload.to_vec());
        pos = start + len;
    }
    records
}

#[cfg(test)]
mod tests {
    use super::*;

    fn sample() -> Vec<u8> {
        let mut log = Vec::new();
        append(&mut log, b"SET a 1");
        append(&mut log, b"SET b 2");
        append(&mut log, b"DEL a");
        log
    }

    #[test]
    fn a_clean_log_replays_everything() {
        assert_eq!(replay(&sample()).len(), 3);
    }

    #[test]
    fn a_torn_last_record_is_ignored() {
        let log = sample();
        for cut in 1..=12 {
            let records = replay(&log[..log.len() - cut]);
            assert_eq!(records.len(), 2, "cut {cut} bytes");
        }
    }

    #[test]
    fn a_flipped_bit_stops_the_replay() {
        let mut log = sample();
        log[8 + 7 + 8 + 2] ^= 0x01; // inside the payload of the second record
        assert_eq!(replay(&log), [b"SET a 1".to_vec()]);
    }
}
```

Why it works: the length tells the reader where a record should end, so a record cut off by a crash is noticed; the checksum catches records that are complete but damaged. Stopping at the first bad record is right for a log: nothing after it can be trusted to be in order. (Real logs use CRC32C, a cyclic redundancy check, which detects more kinds of damage than this toy.)

</details>

### Task 4 — The commit index of a cluster (Phase 18)

Write `src/quorum.rs` with `pub fn quorum_index(indexes: &[u64]) -> u64`. `indexes` has one value per node: the leader's last log index and every follower's `match_index`. Return the highest index that is stored on a majority of the nodes. (A Raft leader commits this index if its entry is from the current term.) Test examples, and a property: for several inputs, at least a majority has an index `>=` the result, and fewer than a majority have an index `>=` the result plus one.

<details>
<summary>Hint</summary>

Sort the values from largest to smallest. The value at position `majority - 1` is held, or exceeded, by exactly `majority` nodes.

</details>

<details>
<summary>Answer</summary>

```rust
/// The highest log index stored on a majority of the cluster.
/// `indexes` holds one value per node (the leader's last index and each
/// follower's match index).
pub fn quorum_index(indexes: &[u64]) -> u64 {
    let mut sorted = indexes.to_vec();
    sorted.sort_unstable_by(|a, b| b.cmp(a)); // largest first
    let majority = indexes.len() / 2 + 1;
    sorted[majority - 1]
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn examples() {
        assert_eq!(quorum_index(&[7, 5, 3]), 5);
        assert_eq!(quorum_index(&[9, 9, 2, 2, 2]), 2);
        assert_eq!(quorum_index(&[10, 8, 8, 1, 0]), 8);
        assert_eq!(quorum_index(&[4]), 4);
    }

    #[test]
    fn a_majority_has_it_and_no_higher_index_is_on_a_majority() {
        let cases: [&[u64]; 4] = [
            &[7, 5, 3],
            &[1, 2, 3, 4],
            &[0, 0, 9],
            &[6, 6, 6, 1, 2, 9, 4],
        ];
        for idx in cases {
            let q = quorum_index(idx);
            let majority = idx.len() / 2 + 1;
            let with = |n: u64| idx.iter().filter(|&&i| i >= n).count();
            assert!(with(q) >= majority, "{idx:?}");
            assert!(with(q + 1) < majority, "{idx:?}");
        }
    }
}
```

Run `cargo test`: the eight tests of the four tasks pass (the order of the lines may differ, because tests run in parallel).

```text
test frames::tests::byte_by_byte_gives_the_same_frames ... ok
test cache::tests::an_entry_lives_exactly_its_ttl ... ok
test quorum::tests::examples ... ok
test wal::tests::a_clean_log_replays_everything ... ok
test frames::tests::two_frames_and_a_half ... ok
test quorum::tests::a_majority_has_it_and_no_higher_index_is_on_a_majority ... ok
test wal::tests::a_torn_last_record_is_ignored ... ok
test wal::tests::a_flipped_bit_stops_the_replay ... ok
test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: after sorting in descending order, the first `majority` values are all `>=` the value at position `majority - 1`, so that index is on a majority; the next larger index is missing on at least the node at that position, so it is on fewer nodes. Our `advance_commit` in Lesson 18.11 found the same index by counting, one candidate at a time.

</details>

## If you got something wrong

| Topic | Review this lesson |
|---|---|
| Layers, packets, IP addresses | [Lesson 15.1 — How the internet works: layers and packets](../15-networking/01-how-the-internet-works.md), [Lesson 15.2 — IP addresses, subnets and routing](../15-networking/02-ip-addresses-and-routing.md) |
| TCP, UDP and sockets | [Lesson 15.3 — TCP and UDP](../15-networking/03-tcp-and-udp.md), [Lesson 15.4 — TCP sockets in Rust](../15-networking/04-tcp-sockets-in-rust.md), [Lesson 15.6 — Async networking with Tokio](../15-networking/06-async-networking-with-tokio.md) |
| Framing and parsing | [Lesson 15.7 — Designing a protocol: framing and parsing](../15-networking/07-designing-a-protocol.md) |
| HTTP | [Lesson 15.8 — HTTP from scratch](../15-networking/08-http-from-scratch.md), [Lesson 15.9 — HTTP with hyper and reqwest](../15-networking/09-http-with-hyper-and-reqwest.md) |
| DNS | [Lesson 15.10 — DNS: how names become addresses](../15-networking/10-dns-how-names-become-addresses.md) |
| TLS and QUIC | [Lesson 15.12 — TLS in Rust servers with rustls](../15-networking/12-tls-in-rust-servers.md), [Lesson 15.13 — QUIC and HTTP/3](../15-networking/13-quic-and-http3.md) |
| Proxies and load balancers | [Lesson 15.14 — Proxies and load balancers](../15-networking/14-proxies-and-load-balancers.md) |
| REST APIs and axum | [Lesson 16.2 — Designing HTTP APIs: REST](../16-backend-services/02-designing-http-apis.md), [Lesson 16.3 — Web services with axum](../16-backend-services/03-web-services-with-axum.md) |
| SQL, sqlx, transactions | [Lesson 16.5 — SQL and PostgreSQL basics](../16-backend-services/05-sql-and-postgresql-basics.md), [Lesson 16.6 — Databases with sqlx](../16-backend-services/06-databases-with-sqlx.md), [Lesson 16.7 — Migrations and transactions](../16-backend-services/07-migrations-and-transactions.md) |
| Caching | [Lesson 16.10 — Caching with Redis](../16-backend-services/10-caching-with-redis.md) |
| NATS and JetStream | [Lesson 16.11 — Messaging with NATS](../16-backend-services/11-messaging-with-nats.md) |
| Health checks | [Lesson 16.13 — Configuration, health checks and containers](../16-backend-services/13-configuration-health-checks-and-containers.md) |
| `fsync` and durability | [Lesson 17.2 — Durability: fsync and crash safety](../17-storage-and-databases/02-durability-and-fsync.md) |
| Write-ahead log | [Lesson 17.4 — The write-ahead log (WAL)](../17-storage-and-databases/04-write-ahead-log.md) |
| B-trees, LSM trees, Bloom filters | [Lesson 17.6 — B-trees](../17-storage-and-databases/06-b-trees.md), [Lesson 17.7 — LSM trees](../17-storage-and-databases/07-lsm-trees.md), [Lesson 17.8 — Compaction and Bloom filters](../17-storage-and-databases/08-compaction-and-bloom-filters.md) |
| Transactions, isolation, MVCC | [Lesson 17.10 — Transactions and ACID](../17-storage-and-databases/10-transactions-and-acid.md), [Lesson 17.11 — Isolation levels](../17-storage-and-databases/11-isolation-levels.md), [Lesson 17.12 — MVCC: multi-version concurrency control](../17-storage-and-databases/12-mvcc.md) |
| Timeouts, retries, idempotency | [Lesson 18.2 — Failures and timeouts](02-failures-and-timeouts.md), [Lesson 18.3 — Retries, backoff and idempotency](03-retries-backoff-and-idempotency.md) |
| Clocks | [Lesson 18.4 — Time and clocks](04-time-and-clocks.md), [Lesson 18.5 — Logical clocks: Lamport and vector clocks](05-logical-clocks.md) |
| Replication, quorums, consistency | [Lesson 18.6 — Replication](06-replication.md), [Lesson 18.7 — Consistency models](07-consistency-models.md) |
| Partitioning | [Lesson 18.8 — Partitioning and consistent hashing](08-partitioning-and-consistent-hashing.md) |
| Raft | [Lesson 18.9 — Consensus and an overview of Raft](09-consensus-and-raft.md), [Lesson 18.10 — Raft: leader election](10-raft-leader-election.md), [Lesson 18.11 — Raft: log replication](11-raft-log-replication.md) |
| Simulation testing | [Lesson 18.12 — Deterministic simulation testing](12-deterministic-simulation-testing.md) |
| Sagas and 2PC | [Lesson 18.13 — Distributed transactions and sagas](13-distributed-transactions-and-sagas.md) |

If you missed more than five questions, do not rush into Stage 5. Review the lessons in the table, redo one exercise from each, and take the quiz again after a day and after a week (spaced repetition, from the Stage 1 checkpoint).

## What's next

**Stage 5 — Mastery and career** starts with [Phase 19 — Performance and profiling](../19-performance-and-profiling/README.md). Until now you made systems *correct*: they survive crashes, partitions and retries. Phase 19 makes them *fast*, and teaches you to prove it with numbers: latency percentiles, load tests, profilers and flame graphs, `perf` on Linux, allocation profiling, data layout, SIMD (Single Instruction, Multiple Data), build settings, and the performance of async and I/O (input/output) code. Your Stage 4 projects are perfect material: the URL shortener, the key-value store and the replicated store all have real hot paths to measure. After Phase 19 come reliability and hardening (Phase 20), cloud-native infrastructure with Kubernetes (Phase 21), a specialization (Phase 22), the capstone projects (Phase 23) and career and open source (Phase 24).

Before you start Phase 19:

1. **Write a short retrospective** in your learning journal: the three hardest topics of Stage 4, the project you are proudest of, and one bug that a test (or a simulation seed) found for you.
2. **Tag the milestone** in your learning repository: `git tag -a stage-4-complete -m "Stage 4 complete"` and `git push --tags`.
3. **Check your projects** with the quality gate: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test` in every `code/p15-*`, `code/p16-*`, `code/p17-*` and `code/p18-*` package.
4. **Keep the Linux VM (virtual machine) `dev` ready.** Phase 19 uses `perf`, which runs only on Linux.
5. **Know the tools of Phase 19.** Do not install them now; each lesson tells you when. Versions were checked on 2026-11-17.

| Crate or tool | Version | What it is for |
|---|---|---|
| `criterion` | 0.8 | statistics-driven benchmarks |
| `hdrhistogram` | 7.6 | recording latency percentiles |
| `oha` | 1.17 | an HTTP load generator |
| `flamegraph` (cargo subcommand) | 0.6 | flame graphs from `perf` or DTrace |
| `dhat` | 0.3 | heap (allocation) profiling |
| `mimalloc` / `tikv-jemallocator` | 0.1 / 0.7 | alternative memory allocators |

## Sources and further reading

- [Designing Data-Intensive Applications](https://dataintensive.net/) — Martin Kleppmann's book covers almost all of Phases 17 and 18; read it again now, it will read differently.
- [The Raft Consensus Algorithm](https://raft.github.io/) — the paper, the visualization and a list of real implementations to read.
- [MIT 6.5840 Distributed Systems](https://pdos.csail.mit.edu/6.824/) — lectures and labs if you want more distributed systems before moving on.
- [Jepsen analyses](https://jepsen.io/analyses) — reports on how real databases behaved under partitions; a good way to test your understanding of consistency models.
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) — the official tutorial, for a quick review of the async networking that all of Stage 4 was built on.
- [The Rust Performance Book](https://nnethercote.github.io/perf-book/) — a short book by Nicholas Nethercote; a good first look at Phase 19.

---

[🏠 Start](../START-HERE.md) · [← 18.14 Project: a replicated key-value store](14-project-replicated-key-value-store.md) · [📋 Phase 18](README.md) · [Phase 19 — Performance and profiling →](../19-performance-and-profiling/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
