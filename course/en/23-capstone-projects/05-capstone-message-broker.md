<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 23.4 Capstone option 2: a high-performance HTTP reverse proxy](04-capstone-http-reverse-proxy.md) · [📋 Phase 23](README.md) · [23.6 Capstone option 4: an observability pipeline →](06-capstone-observability-pipeline.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/05-capstone-message-broker.md) <!-- nav -->

# Lesson 23.5 — Capstone option 3: a message broker

> **Phase 23 — Capstone projects** · Lesson 5 of 10 · ⏱ about 50 hours (25 sittings)

## What you will learn

- What a log-based message broker does, and why the append-only log is one of the most important ideas in systems.
- A reference architecture: brokers with partitioned, segmented logs, producers, consumer groups with committed offsets, and replication with a high watermark.
- The contract: the binary protocol, the command-line tools, the delivery promises and the rules for offsets.
- Seven milestones from a reference model to segment files, a network protocol, consumer groups, replication and idempotent producers.
- The quality bar for this option, stretch goals, and the real systems to study.

## Before you start

- You finished [Lesson 23.1](01-how-capstone-projects-work.md) and [Lesson 23.2](02-writing-a-design-document.md).
- You used a broker in [Lesson 16.11 — Messaging with NATS](../16-backend-services/11-messaging-with-nats.md): subjects, queue groups, at-least-once delivery and JetStream. Now you build one.
- You will use again: binary formats from [Lesson 17.3](../17-storage-and-databases/03-binary-file-formats.md), the write-ahead log from [Lesson 17.4](../17-storage-and-databases/04-write-ahead-log.md), `fsync` from [Lesson 17.2](../17-storage-and-databases/02-durability-and-fsync.md), framing from [Lesson 15.7 — Designing a protocol: framing and parsing](../15-networking/07-designing-a-protocol.md), replication from [Lesson 18.6](../18-distributed-systems/06-replication.md), idempotency from [Lesson 18.3](../18-distributed-systems/03-retries-backoff-and-idempotency.md) and model-based tests from [Lesson 17.14](../17-storage-and-databases/14-project-key-value-store.md).

## New words

| Word | Meaning |
|---|---|
| **broker** | A server that receives messages from producers, stores them, and hands them to consumers. |
| **producer** / **consumer** | A program that sends messages to the broker / that reads them from it. |
| **topic** | A named stream of messages, such as `orders`. |
| **partition** | One ordered, append-only log inside a topic. A topic with 3 partitions has 3 independent logs; order is kept only *within* a partition. |
| **offset** | The position of a record in its partition: 0, 1, 2, … It never changes and is never reused. |
| **log end offset** | The offset that the next appended record will get. |
| **segment** | One file of a partition's log. When a segment is full, a new one starts; old segments are deleted as a whole. |
| **sparse index** | An index file with an entry for only every *n*-th record (offset → byte position), so that a read can jump close to the right place and scan the rest. |
| **retention** | The rule for how long records are kept, by age or by total size. |
| **consumer group** | A set of consumers that share the partitions of a topic: each partition is read by exactly one member of the group. |
| **committed offset** | The offset a consumer group has saved as "processed up to here". After a restart, reading continues from it. |
| **rebalance** | Giving the partitions to the group members again after a member joins or leaves. |
| **consumer lag** | How far a group is behind: log end offset minus committed offset. |
| **long polling** | A fetch that waits on the broker until data arrives or a time limit passes, instead of returning empty at once. |
| **acks** | The producer's choice of how long to wait: `0` (not at all), `1` (the leader wrote it), `all` (every in-sync replica has it). |
| **ISR** (In-Sync Replicas) | The replicas of a partition that are up to date with the leader within a time limit. |
| **high watermark** | The first offset that not every in-sync replica has yet: every record below it is on all in-sync replicas. Consumers may read only below it. |
| **idempotent producer** | A producer whose retried batches are recognised by the broker (producer ID, an identifier, and sequence number) and stored only once. |
| **CRC-32C** (Cyclic Redundancy Check, 32 bits, Castagnoli) | A 32-bit checksum; a different value after reading means the bytes were damaged (Lesson 17.3). |
| **CI** (Continuous Integration) | A server that runs your checks automatically on every push (Lesson 09.12). |

## 1. What you build and why it matters

You build `brokerd`, a small log-based broker in the spirit of Kafka: producers append records to **topics**, each topic is split into **partitions**, each partition is an append-only **log** on disk, and consumers read from any offset they like, as fast or as slow as they like. Records are not deleted when they are read; they are deleted by **retention**. Groups of consumers share partitions and remember their progress as **committed offsets**. In the later milestones, each partition is **replicated** to other brokers.

The append-only log is one of the most important ideas in systems. You met it as the write-ahead log of a database (Lesson 17.4) and as the Raft log (Lesson 18.11). A broker makes the log itself the product: many services write events into it, and many others read them independently, each at its own pace. This is how many organisations connect their services, build data pipelines, and replay history after a bug.

Why it matters for you: a broker combines storage (segment files, indexes, `fsync`, crash recovery), networking (a binary protocol, batching, long polling), and distributed systems (replication, a high watermark, failover, idempotent producers). "How does Kafka guarantee ordering and durability?" is a common interview question — after this capstone you can answer it from your own code.

## 2. Reference architecture

```text
 producers                                         consumers (group "billing")
 bk produce orders --key k                         member A: partitions 0, 1
    |  Produce(topic, partition, batch, acks)      member B: partition 2
    v                                                  ^  Fetch(topic, part, offset,
 +-----------------------------------------------+     |        max_wait_ms, min_bytes)
 | brokerd 1                                     |-----+
 |  +--------------+   +-----------------------+ |     |  CommitOffset(group, part, off)
 |  | protocol     |-->| partition orders/0    | |<----+
 |  | (frames)     |   |  segments:            | |
 |  +--------------+   |  00000000.log  .index | |   group coordinator:
 |                     |  00004096.log  .index | |   members, assignment,
 |                     |  log end offset: 5120 | |   offsets in __offsets
 |                     |  high watermark: 5118 | |
 |                     +-----------^-----------+ |
 +---------------------------------|-------------+
                                   | followers fetch from the leader
                     +-------------+-------------+
                     | brokerd 2, brokerd 3      |
                     | (replicas of orders/0)    |
                     +---------------------------+
```

The parts and their jobs:

1. **The protocol layer** reads length-prefixed frames, decodes requests, and writes responses with the same correlation ID.
2. **Each partition** is a directory of segment files. A segment has a `.log` file with record batches and a `.index` file with a sparse index from offset to byte position. The name of a segment is its first offset.
3. **The group coordinator** keeps the members of each consumer group, assigns partitions to them, and stores committed offsets — itself in an internal topic, `__offsets`.
4. **Replication** (M5): each partition has a leader broker and followers; followers fetch from the leader like consumers; the leader moves the high watermark when all in-sync replicas have a record.

> 📏 **Convention:** Keep the storage of one partition (`PartitionLog`) a library with no network code. Then the model-based test (M1), the crash tests and the benchmarks all run against it directly, and the broker is only a thin layer that turns frames into calls.

## The project

### What you will build

A Cargo workspace `code/p23-broker`:

| Crate | Kind | Job |
|---|---|---|
| `broker-log` | library | one partition: segments, sparse index, recovery, retention; plus the reference model |
| `broker-proto` | library | frame and message encoding and decoding (no I/O, input/output) |
| `brokerd` | binary | the broker: TCP (Transmission Control Protocol) server, topics, groups, replication |
| `bk` | binary | the command-line tool: create topics, produce, consume, describe groups |
| `broker-tests` | tests only | end-to-end, crash and simulation tests |

**Non-goals:** no transactions across partitions, no log compaction by key (stretch goal), no authentication, no compatibility with the real Kafka protocol (your protocol is your own and documented in `docs/protocol.md`).

### Contract

**Command lines:**

```text
brokerd --id <N> --listen <ADDR> --data <DIR> [--peers <ID=ADDR,...>]
        [--segment-bytes 1048576] [--flush every-batch|interval-ms=<N>]

bk [--broker ADDR] topic create <TOPIC> --partitions <N> [--replicas <R>] [--retention-bytes <B>] [--retention-ms <MS>]
bk [--broker ADDR] produce <TOPIC> [--key <KEY>] [--acks 0|1|all] <VALUE>    ->  orders/1@42
bk [--broker ADDR] consume <TOPIC> --group <GROUP> [--from-beginning]       ->  orders/1@42 key=k value
bk [--broker ADDR] groups describe <GROUP>                                  ->  one line per partition with lag
```

`orders/1@42` means "topic `orders`, partition 1, offset 42". `consume` prints one line per record and keeps running until `Ctrl+C`.

**Wire protocol.** Every request and response is a frame. All integers are big-endian:

| Field | Size | Meaning |
|---|---|---|
| length | 4 bytes, `u32` | number of bytes **after** this field |
| api key | 2 bytes, `u16` | 1 Produce, 2 Fetch, 3 ListOffsets, 4 CommitOffset, 5 FetchOffset, 6 Metadata, 7 JoinGroup, 8 Heartbeat, 9 LeaveGroup |
| api version | 2 bytes, `u16` | starts at 0; lets you change a message later without breaking old clients |
| correlation ID | 4 bytes, `u32` | chosen by the client; the response carries the same value |
| body | length − 8 bytes | the message itself, described in `docs/protocol.md` |

Test vector (it goes into a test in Exercise 3): a Produce frame, version 0, correlation ID 7, body `hi` is exactly `00 00 00 0a 00 01 00 00 00 00 00 07 68 69`.

Records travel and are stored in **batches**: a batch header (base offset, record count, CRC-32C checksum of the batch, producer ID, base sequence number), then the records (key length, key, value length, value). The broker writes a batch to disk as it received it, without re-encoding every record.

**Rules:**

1. **Offsets:** in each partition they start at 0, grow by exactly 1 per record, and are never reused, also not after retention.
2. **Partitioning:** a record with a key goes to `hash(key) % partitions` (name your hash function in the design document, and keep it stable forever); records without a key are spread round robin by the producer.
3. **Ordering:** records of one partition are read in offset order. There is no order between partitions.
4. **Durability:** with `--flush every-batch`, a batch is `fsync`-ed before `acks=1` is answered; with `interval-ms=N`, at most N ms of acknowledged data can be lost in a power failure — the README must say so. `acks=all` (M5) answers only when every in-sync replica has the batch.
5. **Fetch:** returns records from the requested offset; at the log end offset it waits up to `max_wait_ms` until at least `min_bytes` are available (long polling); an offset below the log start or above the log end is the error `OffsetOutOfRange` with both limits.
6. **Visibility** (M5): consumers see only records below the high watermark.
7. **Delivery:** at-least-once. A consumer commits an offset **after** it has processed the records before it; after a crash, it continues from the committed offset and may see some records again.
8. **Retention:** by total size and by age per topic; only whole, closed segments are deleted.
9. **Recovery:** after a crash, the last segment is checked batch by batch; a torn or corrupt batch at the end is cut off; everything before it is kept.

**The reference model.** Before you write segment files, write the model of one partition that every later test compares with. Try it in a scratch package `code/p23-broker-model`:

```rust
/// One record as the broker stores it.
#[derive(Clone, Debug, PartialEq)]
pub struct Record {
    pub offset: u64,
    pub value: Vec<u8>,
}

#[derive(Debug, PartialEq)]
pub enum FetchError {
    /// The offset is before the oldest kept record or after the end.
    OffsetOutOfRange { log_start: u64, log_end: u64 },
}

/// The reference model of one partition: in memory, slow, obviously correct.
#[derive(Default)]
pub struct ModelLog {
    log_start: u64,
    records: Vec<Record>,
}

impl ModelLog {
    /// Appends a record and returns its offset.
    pub fn append(&mut self, value: &[u8]) -> u64 {
        let offset = self.log_end();
        self.records.push(Record {
            offset,
            value: value.to_vec(),
        });
        offset
    }

    /// The offset that the next appended record will get.
    pub fn log_end(&self) -> u64 {
        self.log_start + self.records.len() as u64
    }

    /// Up to `max` records, starting at offset `from`.
    pub fn fetch(&self, from: u64, max: usize) -> Result<Vec<Record>, FetchError> {
        if from < self.log_start || from > self.log_end() {
            return Err(FetchError::OffsetOutOfRange {
                log_start: self.log_start,
                log_end: self.log_end(),
            });
        }
        let skip = (from - self.log_start) as usize;
        Ok(self.records.iter().skip(skip).take(max).cloned().collect())
    }

    /// Retention: forget every record with an offset below `offset`.
    pub fn delete_before(&mut self, offset: u64) {
        let offset = offset.min(self.log_end());
        if offset > self.log_start {
            let n = (offset - self.log_start) as usize;
            self.records.drain(..n);
            self.log_start = offset;
        }
    }
}

fn main() {
    let mut log = ModelLog::default();
    for value in ["a", "b", "c"] {
        let offset = log.append(value.as_bytes());
        println!("appended {value:?} at offset {offset}");
    }
    let from_one: Vec<u64> = log.fetch(1, 10).unwrap().iter().map(|r| r.offset).collect();
    println!("fetch(1, 10) -> offsets {from_one:?}");
    log.delete_before(2);
    println!("after retention: fetch(0, 10) -> {:?}", log.fetch(0, 10));
    println!("next append gets offset {}", log.append(b"d"));
    println!("fetch(4, 10) -> {:?}", log.fetch(4, 10));
}
```

**What happens:**

1. `log_start` is the offset of the oldest kept record; `log_end()` is the offset the next record gets. Everything the broker promises about offsets (rules 1 and 5) is expressed with these two numbers.
2. `append` gives the new record the offset `log_end()` and returns it, so offsets are consecutive.
3. `fetch` accepts any offset from `log_start` up to and **including** `log_end()`. Fetching exactly at the end is not an error: it means "I am up to date", and in the real broker that is where long polling waits.
4. `delete_before` is retention. It moves `log_start` forward but never past the end, and never changes `log_end()`, so offsets are not reused.

```bash
cargo run
```

```text
appended "a" at offset 0
appended "b" at offset 1
appended "c" at offset 2
fetch(1, 10) -> offsets [1, 2]
after retention: fetch(0, 10) -> Err(OffsetOutOfRange { log_start: 2, log_end: 3 })
next append gets offset 3
fetch(4, 10) -> Ok([])
```

The model ignores files, batches and time. That is the point: it is so simple that it is obviously right, and in M1 a property-based test runs thousands of random operations on your real `PartitionLog` and on `ModelLog` and checks that they always agree.

### Step-by-step plan

The outputs below are **part of the contract**: what your finished broker must print. They are not output produced by the course. Write each check as an acceptance test before you start the milestone.

#### M0 — Walking skeleton

**Goal.** The workspace with CI; `ModelLog` and its tests in `broker-log`; the frame header encoder and decoder in `broker-proto` with the test vector; `#[ignore]` stubs for M1–M6.

**Done when:** `cargo test` passes the model tests and the test-vector test; `brokerd --version` and `bk --version` work.

#### M1 — One partition on disk

**Goal.** `PartitionLog` with segment files, a sparse index, rolling to a new segment at `--segment-bytes`, retention by size and age, and recovery (rule 9).

<details>
<summary>Hint</summary>

Write each batch as: length, CRC-32C, then the batch bytes. On recovery, read batches from the start of the last segment; stop at the first batch whose length goes past the end of the file or whose checksum is wrong, and truncate the file there. Rebuild the sparse index of the last segment from the batches you read; do not trust an index file that may have been written after the data.

</details>

**Done when:** a `proptest` model-based test with random appends, fetches from random offsets, retention and *reopen* (close and open the log again) always agrees with `ModelLog`; a crash test that cuts the last segment at every byte position inside the last batch recovers every complete batch and nothing else; a fetch from the middle of a 1 GiB partition reads less than one index interval of data before the first record it returns (count the bytes in a test).

#### M2 — The network protocol

**Goal.** `brokerd` with one topic and one partition: Produce, Fetch with long polling, ListOffsets. `bk produce` and `bk consume` without groups (`--from-beginning`).

<details>
<summary>Hint</summary>

`tokio_util::codec` with a length-delimited codec does the framing; you decode the body. Long polling: every partition has a `tokio::sync::Notify` (or a `watch` channel with the log end offset); a waiting fetch sleeps until it is notified or `max_wait_ms` has passed.

</details>

**Done when:**

```bash
bk produce orders hello
bk consume orders --from-beginning
```

```text
orders/0@0
orders/0@0 value=hello
```

Also: a producer sends 1 million records in batches and a consumer reads exactly the same records in the same order; a consumer waiting at the log end receives a new record within 10 ms of its production; the fuzz target for frame and message decoding runs 10 minutes without a panic or an allocation larger than the frame.

#### M3 — Topics, partitions and retention

**Goal.** `topic create`, Metadata, several partitions per topic, key-based partitioning (rule 2) and retention per topic.

<details>
<summary>Hint</summary>

One directory per partition: `<data>/orders-0/`, `<data>/orders-1/`. Store topic settings in a small file per topic, written atomically (temporary file, `fsync`, rename). Test the partitioning function alone first: same key → same partition, and keys spread roughly evenly.

</details>

**Done when:** 10,000 records with 100 different keys land so that all records of one key are in one partition, in production order; a topic with `--retention-bytes 10485760` (10 MiB) never keeps more than 10 MiB plus one segment per partition while you produce 100 MiB; `bk topic create orders --partitions 3` twice gives the error `topic orders already exists`.

#### M4 — Consumer groups and committed offsets

**Goal.** JoinGroup, Heartbeat, LeaveGroup, a simple assignment (for example by ranges), rebalancing when members join or leave or stop sending heartbeats, CommitOffset and FetchOffset stored durably in `__offsets`.

<details>
<summary>Hint</summary>

Give the group a *generation* number that grows with every rebalance, and reject commits from an older generation: a member that was too slow and lost its partitions must not overwrite the offsets of the new owner. Write the commit as a record in `__offsets`, keyed by (group, topic, partition); on start, read the topic to rebuild the latest offsets.

</details>

**Done when:**

```bash
bk groups describe billing
```

```text
orders/0  member=A  committed=1200  end=1250  lag=50
orders/1  member=A  committed=980   end=980   lag=0
orders/2  member=B  committed=1105  end=1190  lag=85
```

Also: with two consumers in one group and three partitions, every partition is read by exactly one of them; when one consumer is killed, the other takes over its partitions within the session timeout and continues from the committed offset; a checker that records every processed (partition, offset) shows **no gaps** (duplicates are allowed — rule 7).

#### M5 — Replication

**Goal.** Each partition has a leader and followers on other brokers; followers fetch from the leader; the leader keeps the ISR and the high watermark; `acks=all`; a simple controller chooses a new leader from the ISR when a leader fails.

<details>
<summary>Hint</summary>

The controller is the hard part to get right. Two good options: one broker is a fixed controller (simple, but a single point of failure — say so in the README), or the controller's metadata lives in your Raft store from option 1 or Project 18.14. A follower that rejoins must first truncate its log to the high watermark it last knew, because records above it may never have been committed.

</details>

**Done when:** a `turmoil` simulation kills the leader of a partition during a load of `acks=all` produces — every acknowledged record is readable after failover, in order, and no consumer ever received a record that later disappeared; with one of three brokers stopped, `acks=all` still succeeds (the ISR shrinks); metrics show under-replicated partitions while the broker is down.

#### M6 — Idempotent producers, performance and operations

**Goal.** Producer IDs and sequence numbers (a retried batch is stored once), benchmarks, metrics, and a three-broker Docker Compose setup.

<details>
<summary>Hint</summary>

The broker keeps, per (producer ID, partition), the last sequence number it stored, as part of the partition's state; it must survive restarts and leader changes, like the deduplication table of option 1. Benchmark with a fixed message size (for example 1 KiB) and record batch size, `acks`, number of partitions and the flush policy.

</details>

**Done when:** a test makes the broker drop the answer to a produce request, the producer retries, and the partition contains the batch exactly once; `docs/results.md` reports MB/s (megabytes per second) and p50/p99 produce latency (the median and the 99th percentile, Lesson 19.1) for `acks=1` and `acks=all` with `--flush every-batch` and with `interval-ms=10`, and explains the differences; consumer lag per group is visible in `/metrics`; `docker compose up` starts three brokers and the M4 commands work against it.

### Quality bar

| Area | What it means here |
|---|---|
| Model-based tests | `PartitionLog` against `ModelLog` (M1); the group coordinator against a simple model of assignments |
| Crash tests | a cut at every byte of the last batch (M1); `kill -9` of `brokerd` during produce; restart from disk |
| Simulation | leader failover with `acks=all` (M5) over many seeds |
| Fuzzing | frame and message decoders, batch decoder, recovery of a segment from random bytes |
| Benchmarks | MB/s and latency per `acks` and flush policy; `criterion` for the append path and the batch decoder |
| Observability | metrics for log end offset, high watermark, ISR size, consumer lag, `fsync` latency |
| Robustness | maximum frame size checked **before** allocating; limits on batch size, connections and waiting fetches |

> 🔒 **Security:** A length field from the network is outside input. Check it against the maximum frame size before you allocate a buffer of that size; otherwise one 4-byte frame header (`ff ff ff ff`) makes the broker try to allocate 4 GiB.

### Stretch goals

- **Log compaction by key:** keep only the newest record of each key in old segments (like `__offsets` needs in the long run).
- **Zero-copy fetch** with the Linux system call `sendfile`, which copies file data from the page cache to the socket inside the kernel, without passing it through your program (the idea of avoiding copies is in Lesson 22.19).
- **Exactly-once processing** for consume-transform-produce, with transactions across partitions.
- **Push consumers** in the style of NATS JetStream, next to pull-based fetch.
- **Tiered storage:** move old segments to an object store (a local MinIO container) and fetch them on demand.

## Real systems to study

| System | What to look at |
|---|---|
| [Apache Kafka — documentation](https://kafka.apache.org/documentation/) | the "Design" section: the log, efficiency, replication, the ISR and the high watermark |
| [Kafka protocol guide](https://kafka.apache.org/protocol) | how a real binary protocol versions its messages and batches records |
| [KIP-98](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging) | a KIP (Kafka Improvement Proposal) is Kafka's design document for a change; this one designs idempotent producers and transactions, with all the edge cases |
| [NATS JetStream](https://docs.nats.io/nats-concepts/jetstream) | a different design (streams and consumers on the server) for the same problem |
| [Apache Iggy](https://github.com/apache/iggy) | a message-streaming server written in Rust; compare its storage layout with yours |

## Common mistakes

**1. Exposing records above the high watermark.** A consumer reads offset 500 from the leader, the leader crashes, the new leader never had offset 500 — the consumer has seen a record that no longer exists. Rule 6: only below the high watermark.

**2. Committing before processing.** The consumer commits offset 100, crashes while processing record 99, and never sees it again: at-most-once delivery by accident. Rule 7: process, then commit.

**3. Trusting the index after a crash.** The index entry was written, the data was not. Fix: on recovery, rebuild the index of the last segment from the data.

**4. One `fsync` per record.** Throughput collapses to the number of `fsync`s per second (about 500 for 2 ms each). Fix: batches, and a documented flush policy (rule 4).

**5. A rebalance that lets two members read one partition.** The old owner is slow, does not notice the rebalance, and keeps committing. Fix: generation numbers (the hint of M4).

**6. Unbounded waiting fetches.** Ten thousand consumers in long poll, each holding a buffer. Fix: a limit on waiting fetches per connection and in total, and `max_wait_ms` always enforced.

## Exercises

Do these in `code/p23-broker-model` before M0.

### Exercise 1 — At-least-once in practice (easy)

A consumer in group `billing` reads records 40–49 of partition 0, sends an invoice for each, and commits offset 50. It crashes after sending the invoice for record 45 but before committing. What happens after the restart, and what must the invoice service do about it?

<details>
<summary>Hint</summary>

The last committed offset is from the previous round. Look at rule 7.

</details>

<details>
<summary>Answer</summary>

The committed offset is still 40, so after the restart the consumer reads records 40–49 again and sends invoices 40–45 a **second** time. That is at-least-once delivery. The invoice service must therefore be idempotent: for example, it uses (topic, partition, offset) or an invoice ID inside the record as a unique key and ignores duplicates. Committing *before* processing would avoid duplicates but could lose invoices (at-most-once).

</details>

### Exercise 2 — Write tests for the offset rules (medium)

Add a test module to `src/main.rs` with two tests: (a) after `delete_before(2)` on a log with two records, the next `append` returns 2 and `fetch(0, 10)` returns `OffsetOutOfRange` with `log_start: 2, log_end: 3`; (b) on a log with one record, `fetch(1, 10)` is `Ok` with no records and `fetch(2, 10)` is an error.

<details>
<summary>Hint</summary>

`FetchError` derives `PartialEq`, so you can compare a whole `Err(…)` with `assert_eq!`. For (b), `Ok(vec![])` works because `Record` derives `PartialEq` too.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn offsets_are_never_reused_after_retention() {
        let mut log = ModelLog::default();
        log.append(b"a");
        log.append(b"b");
        log.delete_before(2);
        assert_eq!(log.append(b"c"), 2);
        assert_eq!(
            log.fetch(0, 10),
            Err(FetchError::OffsetOutOfRange {
                log_start: 2,
                log_end: 3
            })
        );
    }

    #[test]
    fn fetching_at_the_end_is_empty_and_beyond_is_an_error() {
        let mut log = ModelLog::default();
        log.append(b"a");
        assert_eq!(log.fetch(1, 10), Ok(vec![]));
        assert!(log.fetch(2, 10).is_err());
    }
}
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: (a) pins rule 1 (no reuse) and the error of rule 5; (b) pins the difference between "up to date" and "out of range", which your long polling depends on.

</details>

### Exercise 3 — The test vector (medium)

Write `pub fn encode_frame(api_key: u16, version: u16, correlation_id: u32, body: &[u8]) -> Vec<u8>` test-first: the test compares `encode_frame(1, 0, 7, b"hi")` with the 14 bytes of the test vector in the contract.

<details>
<summary>Hint</summary>

`u32::to_be_bytes` and `u16::to_be_bytes` give the big-endian bytes (Lesson 12.1). The length counts the 8 header bytes after the length field plus the body.

</details>

<details>
<summary>Answer</summary>

```rust
/// Encodes one frame: length, api key, version, correlation ID, body (all big-endian).
pub fn encode_frame(api_key: u16, version: u16, correlation_id: u32, body: &[u8]) -> Vec<u8> {
    let length = (8 + body.len()) as u32;
    let mut frame = Vec::with_capacity(4 + length as usize);
    frame.extend_from_slice(&length.to_be_bytes());
    frame.extend_from_slice(&api_key.to_be_bytes());
    frame.extend_from_slice(&version.to_be_bytes());
    frame.extend_from_slice(&correlation_id.to_be_bytes());
    frame.extend_from_slice(body);
    frame
}
```

The test, inside `mod tests`:

```rust
#[test]
fn a_produce_frame_matches_the_protocol_document() {
    let frame = encode_frame(1, 0, 7, b"hi");
    let expected = [
        0x00, 0x00, 0x00, 0x0a, // length = 10 bytes after this field
        0x00, 0x01, // api key 1 = Produce
        0x00, 0x00, // version 0
        0x00, 0x00, 0x00, 0x07, // correlation ID 7
        b'h', b'i', // body
    ];
    assert_eq!(frame, expected);
}
```

Why it works: a test vector written in the protocol document and checked by a test keeps the document and the code in agreement. If anyone ever changes the byte order or the length rule by mistake, this test fails. (The cast `as u32` is safe here only because frames are limited far below 4 GiB; the real encoder should check the limit.)

</details>

### Exercise 4 — A property test for the model (hard)

With `proptest` (Lesson 09.4; `cargo add --dev proptest@1`), write a property test: for any sequence of up to 100 random operations, each either `Append(value)` or `DeleteBefore(offset)`, (a) the offsets returned by `append` are 0, 1, 2, … in order, and (b) after every operation, fetching from `log_start` returns exactly `log_end() - log_start` records.

<details>
<summary>Hint</summary>

Describe an operation with an `enum Op` and build a strategy with `prop_oneof!` and `prop_map`, as in the model-based test of Project 17.14. Give appends a higher weight than deletes (`3 => …, 1 => …`), otherwise the log is empty most of the time. The test module may read the private field `log_start`, because it is a child module.

</details>

<details>
<summary>Answer</summary>

```rust
use proptest::prelude::*;

#[derive(Debug, Clone)]
enum Op {
    Append(Vec<u8>),
    DeleteBefore(u64),
}

fn op() -> impl Strategy<Value = Op> {
    prop_oneof![
        3 => prop::collection::vec(any::<u8>(), 0..8).prop_map(Op::Append),
        1 => (0u64..50).prop_map(Op::DeleteBefore),
    ]
}

proptest! {
    #[test]
    fn appended_offsets_are_consecutive(ops in prop::collection::vec(op(), 0..100)) {
        let mut log = ModelLog::default();
        let mut expected_next = 0;
        for op in ops {
            match op {
                Op::Append(value) => {
                    prop_assert_eq!(log.append(&value), expected_next);
                    expected_next += 1;
                }
                Op::DeleteBefore(offset) => log.delete_before(offset),
            }
            let kept = log.fetch(log.log_start, usize::MAX).unwrap();
            prop_assert_eq!(kept.len() as u64, log.log_end() - log.log_start);
        }
    }
}
```

Put these lines inside `mod tests`. With the four tests of Exercises 2–4, `cargo test -q` prints:

```text
running 4 tests
....
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.07s
```

Why it works: proptest runs the body for 256 random sequences by default and shrinks a failing one to a minimal example. `DeleteBefore` with offsets up to 50 sometimes points beyond the end, which tests the `min` in `delete_before`. In M1, the same `Op` enum gets more variants (`Fetch`, `Reopen`) and drives your real `PartitionLog` next to the model.

</details>

## Quiz

1. Why does a topic have partitions, and what ordering does a consumer get?

<details>
<summary>Answer</summary>

Partitions let a topic be written and read in parallel (by several brokers and several consumers). Order is guaranteed only within one partition; records with the same key go to the same partition, so they stay in order.

</details>

2. What do the log start offset and the log end offset mean, and what happens on a fetch exactly at the log end?

<details>
<summary>Answer</summary>

The log start is the offset of the oldest kept record; the log end is the offset the next record will get. A fetch at the log end is not an error: the consumer is up to date, and the broker waits (long polling) until new data arrives or `max_wait_ms` passes.

</details>

3. Why is a sparse index enough?

<details>
<summary>Answer</summary>

Because records in a segment are sorted by offset. The index jumps to the nearest earlier entry, and the broker scans forward a small, bounded distance. A full index would be much larger and gain little.

</details>

4. What is the high watermark, and why may consumers read only below it?

<details>
<summary>Answer</summary>

The first offset that not every in-sync replica has yet; every record below it is on all in-sync replicas. Records at or above it may exist only on some replicas and can disappear in a failover; reading them would let a consumer see data that is later lost.

</details>

5. What problem do generation numbers solve in consumer groups?

<details>
<summary>Answer</summary>

A slow member that missed a rebalance might keep committing offsets for partitions it no longer owns. With a generation number on every commit, the coordinator rejects commits from an old generation.

</details>

6. What does `acks=all` promise, and what does it cost?

<details>
<summary>Answer</summary>

The producer is answered only when every in-sync replica has the batch, so the batch survives the loss of the leader. It costs latency (a round trip to the followers and their writes) and needs the ISR to stay healthy.

</details>

7. Why is a reference model like `ModelLog` worth writing, even though it is never used in production?

<details>
<summary>Answer</summary>

It is so simple that it is obviously correct, so it can be the judge in model-based tests: thousands of random operations on the real, complicated log are compared with the model, and every difference is a bug.

</details>

## Summary

- Option 3 is a log-based broker: topics split into partitions, each partition an append-only log of segment files with a sparse index, consumers that read from offsets, and retention instead of deletion on read.
- Consumer groups share partitions and store committed offsets; delivery is at-least-once, so consumers must handle duplicates.
- Replication adds leaders, followers, the ISR and the high watermark; `acks` lets producers choose between latency and durability; idempotent producers remove duplicates from retries.
- The contract fixes the command lines, a versioned binary frame format with a test vector, and nine rules about offsets, ordering, durability, fetch, visibility, delivery, retention and recovery.
- Milestones: M0 model and framing, M1 partition on disk, M2 protocol, M3 topics and retention, M4 consumer groups, M5 replication, M6 idempotent producers, performance and operations.
- `ModelLog` is the judge for model-based tests from M1 on; study Kafka, KIP-98, NATS JetStream and Apache Iggy after designing each part yourself.

## Sources and further reading

- [Apache Kafka documentation — Design](https://kafka.apache.org/documentation/) — the clearest description of a log-based broker: persistence, efficiency, replication and delivery semantics.
- [A Guide To The Kafka Protocol](https://kafka.apache.org/protocol) — framing, versioning and record batches in a production protocol.
- [KIP-98 — Exactly Once Delivery and Transactional Messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging) — producer IDs, sequence numbers and the reasoning behind them.
- [NATS JetStream concepts](https://docs.nats.io/nats-concepts/jetstream) — streams, consumers and acknowledgements in the broker you used in Lesson 16.11.
- [tokio-util codec](https://docs.rs/tokio-util/latest/tokio_util/codec/index.html) — `LengthDelimitedCodec` and the `Decoder`/`Encoder` traits for M2.
- [proptest documentation](https://docs.rs/proptest/latest/proptest/) — strategies, `prop_oneof!` and shrinking for the model-based tests.

---

[🏠 Start](../START-HERE.md) · [← 23.4 Capstone option 2: a high-performance HTTP reverse proxy](04-capstone-http-reverse-proxy.md) · [📋 Phase 23](README.md) · [23.6 Capstone option 4: an observability pipeline →](06-capstone-observability-pipeline.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
