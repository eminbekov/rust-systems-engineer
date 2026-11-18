<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 23.2 Writing a design document](02-writing-a-design-document.md) · [📋 Phase 23](README.md) · [23.4 Capstone option 2: a high-performance HTTP reverse proxy →](04-capstone-http-reverse-proxy.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/03-capstone-distributed-key-value-store.md) <!-- nav -->

# Lesson 23.3 — Capstone option 1: a distributed key-value store

> **Phase 23 — Capstone projects** · Lesson 3 of 10 · ⏱ about 50 hours (25 sittings)

## What you will learn

- What a production-style distributed key-value store does, and why building one proves deep systems skills.
- A reference architecture: an HTTP API, a Raft core, a write-ahead log and a state machine on three nodes.
- The contract of the capstone: the server and client command lines, the HTTP API and the guarantees.
- Seven milestones, from a walking skeleton to snapshots, batching and a simulated chaos test, each with "Done when" checks.
- The quality bar for this option, stretch goals, and which real systems to study.

## Before you start

- You finished [Lesson 23.1](01-how-capstone-projects-work.md) and [Lesson 23.2](02-writing-a-design-document.md): you have a skeleton and a design-document template.
- This capstone grows out of two earlier projects: [Lesson 17.14 — Project: your own key-value store](../17-storage-and-databases/14-project-key-value-store.md) (`p17-kv`, the storage engine) and [Lesson 18.14 — Project: a replicated key-value store](../18-distributed-systems/14-project-replicated-key-value-store.md) (`p18-kv`, Raft on three nodes with a text protocol).
- You will use again: Raft from [Lessons 18.9–18.11](../18-distributed-systems/09-consensus-and-raft.md), linearizability from [Lesson 18.7 — Consistency models](../18-distributed-systems/07-consistency-models.md), idempotency from [Lesson 18.3](../18-distributed-systems/03-retries-backoff-and-idempotency.md), simulation with `turmoil` from [Lesson 18.12](../18-distributed-systems/12-deterministic-simulation-testing.md), `axum` from [Lesson 16.3](../16-backend-services/03-web-services-with-axum.md) and load tests with `oha` from [Lesson 19.2](../19-performance-and-profiling/02-load-testing.md).

## New words

| Word | Meaning |
|---|---|
| **state machine** (replicated) | The part that holds the data (here: your key-value engine). Every node applies the same committed log entries in the same order, so every state machine ends in the same state. |
| **contract test** | A test function written against a trait, not a type. Every implementation (in memory, one node, a cluster) must pass the same test. |
| **request ID** (ID = identifier) | A pair (client number, sequence number) that a client puts on each write, so that the store can recognise a retry and apply it only once. |
| **deduplication table** | The part of the state machine that remembers, per client, the last applied sequence number and its result. |
| **read index** | A way to serve a linearizable read without writing a log entry: the leader notes its commit index, confirms with one round of heartbeats that it is still leader, waits until that index is applied, then reads. |
| **stale read** | A read that may return slightly old data, for example from a follower. It is cheaper, but not linearizable. |
| **snapshot** | A copy of the whole state machine at one log index. With a snapshot, the log entries before that index can be deleted. |
| **log compaction** | Deleting old log entries after a snapshot, so that the log does not grow forever. |
| **`InstallSnapshot`** | The Raft message that sends a snapshot to a follower that is so far behind that the leader no longer has the entries it needs. |
| **batching** (group commit) | Writing many proposals to the log with one `fsync` instead of one `fsync` each. It raises throughput a lot. |
| **pipelining** | Sending the next `AppendEntries` to a follower before the answer to the previous one has arrived. |
| **membership change** | Adding or removing a node from a running cluster without stopping it. |
| **307 Temporary Redirect** | An HTTP (HyperText Transfer Protocol) status that says "send the same request, with the same method and body, to the address in the `Location` header". |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The usual encoding of text as bytes; Rust's `String` and `&str` are always valid UTF-8. |
| **URL** (Uniform Resource Locator) | A web address, such as `https://example.test/v1/kv/color`. |
| **RPC** (Remote Procedure Call) | A call of a function on another machine over the network, such as Raft's `AppendEntries`. |
| **ADR** (Architecture Decision Record) | A short file that records one important decision: context, decision, consequences (Lesson 23.2). |
| **SSD** (Solid-State Drive) | A storage device made of flash memory, with no moving parts. |
| **CI** (Continuous Integration) | A server that runs your checks automatically on every push (Lesson 09.12). |

## 1. What you build and why it matters

You build `kvd`, a small store in the spirit of etcd: a key-value database that runs on three machines and stays **correct** while one of them crashes, restarts or is cut off from the network. Every write goes through Raft, so a write that the store has acknowledged survives the loss of any one node. Clients talk to it over HTTP with `curl` or with a small `kv` command-line client.

You already built both halves. `p17-kv` stores data durably on one machine. `p18-kv` replicates a log with Raft and passes chaos tests in simulation. This capstone turns them into a system that a team could actually run:

- a real **HTTP API** with clear status codes and redirects;
- **idempotent writes**, so that a client can safely retry after a timeout;
- **fast linearizable reads** with read index, and cheap stale reads;
- **snapshots and log compaction**, so that the log does not grow without limit;
- **batching**, so that one `fsync` covers many writes;
- **operations**: metrics, logs, health checks, a three-node Docker Compose setup and benchmark numbers.

Why it matters: almost every large system depends on a store like this. Kubernetes keeps all its state in etcd; databases such as TiKV and CockroachDB are built from many Raft groups. Interviews for infrastructure and database roles often ask the questions this project answers: what happens when the leader dies in the middle of a write, how a read can be linearizable, how a log stays small. After this capstone you can answer them with your own code and your own test results.

## 2. Reference architecture

```text
            kv CLI, curl, contract tests, oha
                      |
                      |  HTTP/JSON: PUT, GET, DELETE /v1/kv/{key}, GET /v1/status
                      v
 +-------------------------------------------+   Raft RPC: AppendEntries,
 | kvd node 1 (leader)                       |   RequestVote, InstallSnapshot
 |                                           |<-------------------+
 |  +-------------+      +----------------+  |                    |
 |  | HTTP API    |----->| driver task    |  |          +-------------------+
 |  | (axum)      |<-----| (owns the core)|  |<-------->| kvd node 2        |
 |  +-------------+      +--+----------+--+  |          | (follower)        |
 |                          |          |     |          +-------------------+
 |              step, tick, |          | committed,     +-------------------+
 |              propose     |          | in order  <--->| kvd node 3        |
 |                          v          v     |          | (follower)        |
 |  +------------------+  +----------------+ |          +-------------------+
 |  | Raft core        |  | state machine: | |
 |  | (pure, no I/O)   |  | KV engine +    | |
 |  +--------+---------+  | dedup table    | |
 |           |            +-------+--------+ |
 |           v                    v          |
 |  +------------------+  +----------------+ |
 |  | Raft log (WAL,   |  | snapshots      | |
 |  | fsync, batches)  |  | (atomic files) | |
 |  +------------------+  +----------------+ |
 +-------------------------------------------+
```

The parts and their jobs:

1. **HTTP API** turns requests into proposals and waits for their results. It never touches Raft state directly.
2. **Driver task** (from Project 18.14) owns the Raft core. It feeds it ticks, messages and proposals, saves the hard state *before* sending messages, and hands committed entries to the state machine.
3. **Raft core** is pure logic without network or disk, so that you can test it with exact, repeatable steps.
4. **Raft log** is a write-ahead log (Lesson 17.4) with batching: many entries, one `fsync`.
5. **State machine** is your key-value engine plus the deduplication table. Both change only by applying committed entries.
6. **Snapshots** let the node delete old log entries and help a slow follower catch up.

> 📏 **Convention:** Keep the Raft core free of I/O (input/output) and of `tokio`. Every bug you find later is easier to reproduce in a pure function that you can call with exact inputs. This is the hexagonal architecture of Lesson 08.14 applied to consensus.

## The project

### What you will build

A Cargo workspace `code/p23-kv` with these crates (rename them if you like, but say so in the design document):

| Crate | Kind | Job |
|---|---|---|
| `kv-raft` | library | the Raft core (start from your `p18-raft`) |
| `kv-store` | library | the state machine: engine, deduplication table, snapshots |
| `kv-api` | library | request and response types shared by server and client |
| `kvd` | binary | the server: driver, HTTP API, Raft transport |
| `kv` | binary | the command-line client |
| `kv-sim` | tests only | `turmoil` simulation tests and the history checker |

**Non-goals** (write them into your design document): no transactions over several keys, no sharding (one Raft group holds everything), no authentication (the server listens on `127.0.0.1` by default), no range scans.

### Contract

**Server command line:**

```text
kvd --id <N> --listen <ADDR> --peers <ID=ADDR,...> --data <DIR>
    [--election-timeout-ms 300..600] [--heartbeat-ms 50] [--snapshot-every 10000]
```

Example for a local three-node cluster: `kvd --id 1 --listen 127.0.0.1:7001 --peers 2=127.0.0.1:7002,3=127.0.0.1:7003 --data data/1`. One address serves both the client API and Raft traffic (under `/raft/…`), or choose two ports and document it.

**HTTP API** (all bodies are JSON, JavaScript Object Notation):

| Method and path | Request body | Success | Errors |
|---|---|---|---|
| `PUT /v1/kv/{key}` | `{"value":"blue","client":7,"seq":42}` | `200 {"index":17}` | `307`, `400`, `413`, `503` |
| `GET /v1/kv/{key}?read=linearizable` (default) | – | `200 {"value":"blue","index":17}` | `404`, `307`, `503` |
| `GET /v1/kv/{key}?read=stale` | – | `200 {"value":"blue","index":15}` | `404` |
| `DELETE /v1/kv/{key}` | `{"client":7,"seq":43}` | `200 {"deleted":true,"index":18}` | `307`, `400`, `503` |
| `GET /v1/status` | – | `200 {"id":1,"role":"leader","term":3,"leader":1,"commit_index":18,"applied_index":18,"log_first_index":1}` | – |
| `GET /healthz`, `GET /readyz` | – | `200 ok` | `503` (ready only with a known leader) |
| `GET /metrics` | – | Prometheus text format | – |

**Rules:**

1. Keys are 1–1,024 bytes of UTF-8 text (in the URL, percent-encoded); values are at most 1 MiB. Anything else is `400` or `413` with `{"error":"…"}`.
2. A write is answered with `200` only after its entry is **committed** (stored on a majority, each copy `fsync`-ed) and **applied**. `index` is its log index.
3. A follower answers writes and linearizable reads with `307 Temporary Redirect`, a `Location` header with the leader's URL, and `{"error":"not leader","leader":1}`. Without a known leader (during an election) the answer is `503 {"error":"no leader"}`.
4. **Linearizable** reads see every write that was acknowledged before the read started. **Stale** reads may be served by any node and may be behind; they return the node's applied index so the client can see how old the answer is.
5. **Idempotency:** a write with a `(client, seq)` that was already applied returns the original result and changes nothing. The deduplication table is part of the state machine: it is replicated, included in snapshots and survives restarts.
6. A request that is not committed within 2 seconds gets `503 {"error":"timeout"}`. The client may retry it with the same `(client, seq)`.

**Client command line** (prints on standard output; errors go to standard error with exit code 1):

```text
kv [--server URL] put <KEY> <VALUE>      ->  OK (index 17)
kv [--server URL] get <KEY> [--stale]    ->  blue            (or "not found", exit 1)
kv [--server URL] delete <KEY>           ->  deleted (index 18)
kv [--server URL] status                 ->  node 1: leader, term 3, commit 18, applied 18
```

The client follows redirects, retries timeouts with backoff and jitter (Lesson 18.3), and keeps its `client` number and a growing `seq`.

**The store trait.** Make the contract executable: one trait that every implementation offers, and contract tests written against the trait. Here is the starting point, with the simplest possible implementation. Put it in a scratch package `code/p23-kv-contract` to try it (in the capstone it goes into `kv-api` and a test-utilities module):

```rust
use std::collections::HashMap;

/// Identifies one client request, so that a retry can be recognised.
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct RequestId {
    pub client: u64,
    pub seq: u64,
}

/// The errors every implementation may return.
#[derive(Debug, PartialEq)]
pub enum KvError {
    /// The key is empty or longer than 1,024 bytes.
    InvalidKey,
    /// This node is not the leader; `leader` is a hint where to go.
    NotLeader { leader: Option<String> },
}

/// What every version of the store offers, from a toy in memory to a 3-node cluster.
pub trait KvStore {
    fn put(&mut self, key: &str, value: &str, id: RequestId) -> Result<(), KvError>;
    fn get(&mut self, key: &str) -> Result<Option<String>, KvError>;
    fn delete(&mut self, key: &str, id: RequestId) -> Result<bool, KvError>;
}

fn check_key(key: &str) -> Result<(), KvError> {
    if key.is_empty() || key.len() > 1024 {
        return Err(KvError::InvalidKey);
    }
    Ok(())
}

/// The simplest store that meets the contract: one HashMap in memory.
#[derive(Default)]
pub struct MemStore {
    map: HashMap<String, String>,
}

impl KvStore for MemStore {
    fn put(&mut self, key: &str, value: &str, _id: RequestId) -> Result<(), KvError> {
        check_key(key)?;
        self.map.insert(key.to_string(), value.to_string());
        Ok(())
    }

    fn get(&mut self, key: &str) -> Result<Option<String>, KvError> {
        check_key(key)?;
        Ok(self.map.get(key).cloned())
    }

    fn delete(&mut self, key: &str, _id: RequestId) -> Result<bool, KvError> {
        check_key(key)?;
        Ok(self.map.remove(key).is_some())
    }
}

/// The basic contract. Every implementation must pass it.
pub fn check_basic_contract(store: &mut impl KvStore) {
    let id = |seq| RequestId { client: 1, seq };
    assert_eq!(store.get("color"), Ok(None));
    store.put("color", "blue", id(1)).unwrap();
    assert_eq!(store.get("color"), Ok(Some("blue".to_string())));
    store.put("color", "green", id(2)).unwrap();
    assert_eq!(store.get("color"), Ok(Some("green".to_string())));
    assert_eq!(store.delete("color", id(3)), Ok(true));
    assert_eq!(store.delete("color", id(4)), Ok(false));
    assert_eq!(store.put("", "x", id(5)), Err(KvError::InvalidKey));
    let long_key = "k".repeat(1025);
    assert_eq!(store.get(&long_key), Err(KvError::InvalidKey));
}

fn main() {
    let mut store = MemStore::default();
    check_basic_contract(&mut store);
    println!("MemStore passes the basic contract");
}
```

**What happens:**

1. `RequestId` is the pair from rule 5. It derives `Hash` and `Eq`, so it can later be a key in a set or a map.
2. `KvError` lists the errors a caller must handle. `NotLeader` carries a hint, like the `307` of rule 3.
3. `KvStore` is the contract as a trait. `get` takes `&mut self` because a real client may need to update its idea of who the leader is.
4. `MemStore` is the **reference implementation**: obviously correct and tiny. It ignores the request ID for now — Exercise 2 shows why that is not enough.
5. `check_basic_contract` takes `&mut impl KvStore`, so the same function will later test a single `kvd` node through HTTP and a three-node cluster through the `kv` client library.

```bash
cargo run
```

```text
MemStore passes the basic contract
```

In your capstone, the real store will be asynchronous (the trait methods become `async fn`, which traits support), but the idea stays: one set of contract tests, run against every layer.

### Step-by-step plan

Each milestone ends with "Done when" checks. The outputs shown are **part of the contract**: they are what your finished program must print, not output produced by the course. Turn each check into an acceptance test (Lesson 23.1) before you start the milestone, and tag the commit when CI is green.

#### M0 — Walking skeleton

**Goal.** The workspace from the table above, with CI, compiles; `kvd --version` and `kv --version` work; the contract tests from this lesson pass against `MemStore`; the acceptance tests of M1–M6 exist as `#[ignore]` stubs.

**Done when:** `cargo test` shows the contract test passing and one ignored test per later milestone; CI runs `fmt`, `clippy` and `test`.

#### M1 — One node behind HTTP

**Goal.** A one-node cluster: the Raft core with no peers commits each entry at once (a majority of one is one). The HTTP API and the `kv` client work end to end, and data is durable.

<details>
<summary>Hint</summary>

Do not build a special "single-node mode": run the normal driver with an empty peer list. Then M2 changes configuration, not code paths. For the durability test, start `kvd` as a child process from the test, write in a loop, kill it with `Child::kill` (that is `SIGKILL`, the same as `kill -9`) at a random moment, restart it, and check that every write that got `200` is readable.

</details>

**Done when:**

```bash
kvd --id 1 --listen 127.0.0.1:7001 --data data/1 &
kv --server http://127.0.0.1:7001 put color blue
kv --server http://127.0.0.1:7001 get color
```

```text
OK (index 2)
blue
```

(Index 1 is the no-op entry of the new leader.) Also: `check_basic_contract` passes against an HTTP client for one node, and the test `m1_acknowledged_writes_survive_kill` (1,000 writes, a kill at a random point, restart) passes 20 times in a row.

#### M2 — Three nodes, redirects and idempotent writes

**Goal.** Three nodes on localhost. Followers redirect. Writes carry `(client, seq)`, and the deduplication table makes retries safe, also across a change of leader.

<details>
<summary>Hint</summary>

Store in the table only the **last** applied `seq` and its result per client. A request with a smaller or equal `seq` is a retry: return the stored result (for an older `seq` than the last one, an error is fine — a well-behaved client never sends it). Because the table is changed only by applying committed entries, every node builds the same table.

</details>

**Done when:**

```bash
curl -si -X PUT http://127.0.0.1:7002/v1/kv/color -H 'content-type: application/json' -d '{"value":"blue","client":7,"seq":1}'
```

```text
HTTP/1.1 307 Temporary Redirect
location: http://127.0.0.1:7001/v1/kv/color
content-type: application/json

{"error":"not leader","leader":1}
```

(`-d` alone sends the body as a web form; the `content-type` header tells the server that it is JSON. Without it, `axum`'s `Json` extractor answers `415 Unsupported Media Type`.)

Also: `kv --server http://127.0.0.1:7002 put color blue` follows the redirect and prints `OK (index …)`; a contract test for retries (Exercise 2) passes against the cluster; a `turmoil` test sends a write, drops the answer, kills the leader, and retries the same request on the new leader — the key is written once, and a later write of the same key is not overwritten by the retry.

#### M3 — Linearizable reads with read index, and stale reads

**Goal.** Reads no longer go through the log. Linearizable reads use read index; `?read=stale` reads come from the local state machine of any node.

<details>
<summary>Hint</summary>

A leader that is cut off from the others still *thinks* it is the leader until it hears a higher term. That is why read index needs one round of heartbeats acknowledged by a majority **after** the read arrives. A new leader must also first commit its no-op entry, or its commit index may be behind.

</details>

**Done when:** after 1,000 linearizable `GET`s, `log_first_index` and the last log index in `/v1/status` have not moved; a `turmoil` test partitions the old leader away from the other two, lets them elect a new leader and write a new value, and checks that the old leader answers a linearizable read with `307` or `503`, never with the old value; the same old leader may answer `?read=stale` with the old value and its applied index.

#### M4 — Snapshots and log compaction

**Goal.** Every `--snapshot-every` applied entries, the node writes a snapshot of the state machine (engine **and** deduplication table) atomically, then deletes the log before it. A follower that is too far behind receives `InstallSnapshot`.

<details>
<summary>Hint</summary>

Write the snapshot to a temporary file, `fsync` it, rename it, `fsync` the directory (the pattern of Lesson 17.2); only then delete log entries. Store the snapshot's last index and term: the consistency check of `AppendEntries` needs the term of the entry just before the first remaining one.

</details>

**Done when:** in a test with 1 million writes over 1,000 keys, the log directory never grows above a limit you state in the design document (for example 64 MiB); a follower that was stopped during 100,000 writes catches up after restart, and its metrics show one `InstallSnapshot`; a node restarted after a snapshot has the same data as before (snapshot plus log tail).

#### M5 — Throughput: batching and pipelining

**Goal.** Measure first, then make it fast: many proposals share one log write and one `fsync`, and the leader keeps several `AppendEntries` in flight per follower.

<details>
<summary>Hint</summary>

Measure the baseline with `oha` and a fixed workload before changing anything, and add a metric for "fsync calls" and "entries per fsync". Group commit is simple in the driver: take every proposal that is waiting in the channel (up to a limit), append them all, `fsync` once.

</details>

**Done when:** `docs/results.md` contains a table with writes per second and p50/p99 latency (the median and the 99th percentile, Lesson 19.1) for three nodes, before and after batching, measured the same way (hardware, value size, number of connections, duration written down); the "entries per fsync" metric is above 1 under load; all earlier tests still pass.

#### M6 — Operations and proof

**Goal.** The store can be run, observed and trusted: metrics, structured logs, graceful shutdown, a Docker Compose cluster, and a chaos simulation with a history checker in CI.

<details>
<summary>Hint</summary>

Start from the chaos test and the simple checker of Project 18.14 and extend it to the HTTP API, stale reads and idempotent retries. Keep a list of fixed seeds in the repository so that CI runs the same seeds every time, and run many random seeds in a separate, slower job.

</details>

**Done when:** `docker compose up` starts three nodes and `kv status` against each shows one leader; `/metrics` reports at least term, role, commit index, applied index, apply lag and request latency histograms; stopping the leader container with `docker compose stop` gives a new leader within the election timeout plus one second; the chaos test passes for 200 fixed seeds in CI and 2,000 random seeds in a local release run; the README states the guarantees in one sentence: "linearizable reads and writes; tolerates the failure of 1 of 3 nodes".

### Quality bar

The general bar is in Lesson 23.1. For this option it means:

| Area | What it means here |
|---|---|
| Unit and model tests | the Raft core tested step by step; a property test that compares the state machine with a `HashMap` model over random command sequences |
| Contract tests | `check_basic_contract` and the retry contract pass against `MemStore`, one node and the cluster |
| Failure tests | `kill -9` durability (M1), leader crash during writes (M2), partition (M3), slow follower (M4), chaos with seeds (M6) |
| Fuzzing | fuzz targets for the log record decoder and the snapshot decoder: corrupt bytes must give an error, never a panic or a huge allocation |
| Benchmarks | `oha` for `PUT` and `GET`, plus a `criterion` benchmark of the log append path |
| Observability | `tracing` spans per request with the log index; the metrics of M6 |
| Robustness | timeouts on every RPC, a bounded proposal queue (`503` when full), body size limit |

### Stretch goals

- **Membership changes:** add a fourth node as a non-voting learner, let it catch up from a snapshot, then make it a voter; remove a node without stopping writes.
- **Leader transfer** on graceful shutdown, so that a planned restart does not wait for an election timeout.
- **`Watch(prefix)`**: a streaming endpoint that sends every change of keys with a prefix, as etcd does.
- **Leases** for faster reads, with your clock-drift assumption written down (Lesson 18.4).
- A full **linearizability checker** for small histories (the search from Lesson 18.7), run on every chaos seed.
- **Multi-Raft:** split the key space into ranges, each with its own Raft group.

## Real systems to study

| System | What to look at |
|---|---|
| [etcd](https://etcd.io/docs/) | the API guarantees page, how leases and watches work, how the server handles client retries |
| [TiKV](https://tikv.org/deep-dive/introduction/) | the deep-dive chapters on Raft, multi-Raft and how regions split |
| [raft-rs](https://github.com/tikv/raft-rs) | a Raft core library in Rust with no I/O, the same separation as your `kv-raft` |
| [openraft](https://github.com/databendlabs/openraft) | an async Raft library in Rust; read its documentation on snapshots and membership |
| [FoundationDB testing](https://apple.github.io/foundationdb/testing.html) | how deterministic simulation finds bugs before users do |
| [Jepsen analyses](https://jepsen.io/analyses) | real reports of stores that lost data, and how the tests found it |

Read **after** you have designed a part yourself. Then compare: what did they handle that you did not? Write the answer into an ADR.

## Common mistakes

**1. Answering before `fsync`.** The write is in the operating system's page cache, the machine loses power, the acknowledged write is gone. Rule 2 says: committed means stored on a majority, each copy `fsync`-ed.

**2. Applying an entry twice after a restart.** The node replays the log from the beginning on top of a state machine that already contains some entries. Fix: store the last applied index together with the data, atomically (in the same snapshot or the same engine write), and skip entries at or below it.

**3. A deduplication table only in the leader's memory.** After a leader change, the new leader does not know which requests were applied, and a retry is applied twice. Fix: rule 5 — the table is part of the replicated state machine.

**4. Serving "linearizable" reads from a leader's local state without checking.** A partitioned old leader returns old data. Fix: read index (M3).

**5. Real time in tests.** Election timeouts from `std::time` and `rand::rng()` make failures impossible to reproduce. Fix: time and randomness come from the driver; in tests they come from `turmoil` and a seed.

**6. Unbounded queues.** Under load, proposals pile up in memory until the process is killed. Fix: a bounded channel and `503` when it is full (Lesson 20.8).

## Exercises

These exercises prepare the design document and the first contract tests. Do them in `code/p23-kv-contract` before M0.

### Exercise 1 — The ambiguous write (easy)

The leader appends a write, replicates it to a majority, applies it, and then crashes **before** it sends `200` to the client. The client sees a timeout. What does the client know? What should it do, and which rule of the contract makes that safe? Write the answer as a row of your failure-mode table.

<details>
<summary>Hint</summary>

From the client's side, "the write failed" and "the write succeeded but the answer was lost" look the same. Look at rules 5 and 6.

</details>

<details>
<summary>Answer</summary>

The client does not know whether the write happened. It retries the **same** request with the **same** `(client, seq)` on the new leader. Rule 5 makes this safe: if the write was applied, the deduplication table returns the original result and changes nothing; if it was not, it is applied now.

| Component | Failure | How we notice | What the user sees | What we do |
|---|---|---|---|---|
| Leader | crashes after commit, before answering | client timeout; new election | one slow write (timeout + retry), then `200` | client retries with the same request ID; dedup returns the stored result |

</details>

### Exercise 2 — Write a test: a retry is applied once (medium)

Add a contract function `check_retry_contract` at the end of `src/main.rs`: write `color = blue` with `(7, 1)`, then `color = green` with `(7, 2)`, then send the first request `(7, 1)` again (its answer "was lost"). The value must still be `green`. Add a test module that runs both contract functions against `MemStore`. Run it, read the failure, and fix `MemStore`.

<details>
<summary>Hint</summary>

The fix needs a `std::collections::HashSet<RequestId>` in `MemStore`. `HashSet::insert` returns `false` if the value was already in the set.

</details>

<details>
<summary>Answer</summary>

```rust
/// A retried request must be applied at most once.
pub fn check_retry_contract(store: &mut impl KvStore) {
    let first = RequestId { client: 7, seq: 1 };
    let second = RequestId { client: 7, seq: 2 };
    store.put("color", "blue", first).unwrap();
    store.put("color", "green", second).unwrap();
    // The answer to `first` was lost, so the client sends it again.
    store.put("color", "blue", first).unwrap();
    assert_eq!(store.get("color"), Ok(Some("green".to_string())));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn mem_store_passes_the_basic_contract() {
        check_basic_contract(&mut MemStore::default());
    }

    #[test]
    fn mem_store_applies_a_retry_once() {
        check_retry_contract(&mut MemStore::default());
    }
}
```

With the `MemStore` of this lesson, `cargo test` fails:

```text
thread 'tests::mem_store_applies_a_retry_once' (6562892) panicked at src/main.rs:86:5:
assertion `left == right` failed
  left: Ok(Some("blue"))
 right: Ok(Some("green"))
```

The retry overwrote the newer value — exactly the bug that loses data in a real cluster. The fix (change the `use` line to `use std::collections::{HashMap, HashSet};`):

```rust
/// The simplest store that meets the contract: one HashMap in memory.
#[derive(Default)]
pub struct MemStore {
    map: HashMap<String, String>,
    seen: HashSet<RequestId>,
}

impl KvStore for MemStore {
    fn put(&mut self, key: &str, value: &str, id: RequestId) -> Result<(), KvError> {
        check_key(key)?;
        if !self.seen.insert(id) {
            return Ok(()); // a retry: already applied, do nothing
        }
        self.map.insert(key.to_string(), value.to_string());
        Ok(())
    }

    fn get(&mut self, key: &str) -> Result<Option<String>, KvError> {
        check_key(key)?;
        Ok(self.map.get(key).cloned())
    }

    fn delete(&mut self, key: &str, _id: RequestId) -> Result<bool, KvError> {
        check_key(key)?;
        Ok(self.map.remove(key).is_some())
    }
}
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the second `put` with `(7, 1)` finds the ID in `seen` and returns without writing. A set of *all* IDs grows forever; in the capstone you keep only the last `seq` per client (the hint of M2), which is enough because a client sends its requests in order.

</details>

### Exercise 3 — Why replicate the deduplication table? (medium)

A friend suggests: "Keep the deduplication table only on the leader. It is the only node that receives writes anyway." Describe a sequence of events in which this loses or corrupts data.

<details>
<summary>Hint</summary>

Combine Exercise 1 (an answer lost after commit) with a change of leader.

</details>

<details>
<summary>Answer</summary>

1. Client 7 writes `color = blue` with `(7, 1)`; leader A commits and applies it, then crashes before answering.
2. Node B becomes leader. Its table is empty, because the table lived only in A's memory.
3. Another client, 8, writes `color = green` with `(8, 1)` on B; it is applied and acknowledged with `200`. Then client 7, after its timeout, retries `(7, 1)` on B.
4. B does not know `(7, 1)` and applies it again: `color` is `blue` again, and client 8's newer acknowledged write is lost.

With the table inside the replicated state machine, B applied `(7, 1)` from the log too, so it recognises the retry.

</details>

### Exercise 4 — Estimate the cost of `fsync` (hard)

Your SSD needs about 2 ms for one `fsync`. (a) If every write needs its own `fsync` on the leader, how many writes per second can one leader do at most? (b) With batches of 50 writes per `fsync`? (c) Why is the real limit in a three-node cluster usually lower than (b)? Put a small function for (a) and (b) into `code/p23-estimate` from Lesson 23.2, with a test.

<details>
<summary>Hint</summary>

One `fsync` at a time: 1,000 ms / 2 ms per `fsync`. For (c), think about what else must happen before a write is committed.

</details>

<details>
<summary>Answer</summary>

(a) 1,000 / 2 = 500 writes per second. (b) 500 × 50 = 25,000 writes per second. (c) A write is committed only when a majority has `fsync`-ed it, so a network round trip to a follower and the follower's own `fsync` are added; CPU (Central Processing Unit) time for HTTP and JSON also counts. Pipelining (M5) hides part of the round trip.

```rust
/// Upper bound of writes per second when one fsync covers `batch` writes.
fn max_writes_per_second(fsync_ms: f64, batch: u64) -> f64 {
    1000.0 / fsync_ms * batch as f64
}

#[test]
fn batching_multiplies_the_fsync_limit() {
    assert_eq!(max_writes_per_second(2.0, 1), 500.0);
    assert_eq!(max_writes_per_second(2.0, 50), 25_000.0);
}
```

Exact float comparison is fine here because 1000 / 2 and the products are exact in `f64`. In the binary `p23-estimate`, also call the function from `main` (for example in a `println!`); otherwise the normal build warns that it is never used. These are upper bounds for the design document; M5 measures the real numbers.

</details>

## Quiz

1. Why is this capstone a good portfolio project for infrastructure and database roles?

<details>
<summary>Answer</summary>

Many large systems depend on a replicated store like this (Kubernetes on etcd, databases built from Raft groups), and interviews ask exactly its questions: leader failure during a write, linearizable reads, log growth, idempotent retries. The project answers them with your own code and test results.

</details>

2. When may `kvd` answer a `PUT` with `200`?

<details>
<summary>Answer</summary>

Only after the entry is committed (stored and `fsync`-ed on a majority of nodes) and applied to the state machine.

</details>

3. What does a follower answer to a write, and why `307` and not `302`?

<details>
<summary>Answer</summary>

`307 Temporary Redirect` with the leader in `Location` (or `503` if it knows no leader). `307` requires the client to repeat the request with the same method and body, so a `PUT` stays a `PUT`.

</details>

4. How does read index make a read linearizable without a log entry?

<details>
<summary>Answer</summary>

The leader notes its commit index, confirms with a heartbeat round acknowledged by a majority that it is still the leader, waits until the state machine has applied at least that index, and then reads locally.

</details>

5. Why must a snapshot contain the deduplication table, not only the key-value data?

<details>
<summary>Answer</summary>

Because the log entries before the snapshot are deleted. A node restored from the snapshot would otherwise not know which requests were already applied, and a retry could be applied twice.

</details>

6. What is the difference between batching and pipelining?

<details>
<summary>Answer</summary>

Batching puts many entries into one log write and one `fsync`. Pipelining sends the next `AppendEntries` to a follower before the answer to the previous one has come back. The first saves disk time, the second hides network round trips.

</details>

7. Why should the outputs in "Done when" be turned into acceptance tests before the milestone starts?

<details>
<summary>Answer</summary>

So that the test checks the promise, not what you happened to build; it fails first (red) and the milestone is done when it passes in CI (green).

</details>

## Summary

- Option 1 turns your `p17-kv` engine and your `p18-kv` Raft cluster into a store that a team could run: HTTP API, idempotent writes, read index, snapshots, batching and operations.
- The architecture separates a pure Raft core, a driver that owns it, a write-ahead log, a state machine with a deduplication table, and snapshots.
- The contract fixes the command lines, the HTTP API, status codes and six rules; the store trait and contract tests make it executable from M0 on.
- Milestones: M0 skeleton, M1 one node over HTTP, M2 three nodes with redirects and idempotency, M3 read index, M4 snapshots, M5 batching, M6 operations and chaos in CI.
- The quality bar adds contract tests at every layer, failure tests for each milestone, fuzzing of every decoder, and benchmark numbers before and after optimisation.
- Study etcd, TiKV, raft-rs, openraft, FoundationDB and Jepsen after designing each part yourself.

## Sources and further reading

- [The Raft website](https://raft.github.io/) and [the Raft paper](https://raft.github.io/raft.pdf) — the algorithm, visualisations and many implementations.
- [Diego Ongaro's doctoral dissertation](https://github.com/ongardie/dissertation) — chapter 5 (snapshots), chapter 4 (membership changes) and chapter 6 (client interaction, read index, sessions for idempotent requests).
- [etcd — API guarantees](https://etcd.io/docs/v3.6/learning/api_guarantees/) — how a production store states its consistency and durability promises.
- [turmoil documentation](https://docs.rs/turmoil/latest/turmoil/) — hosts, partitions, crashes and seeds for M2–M6.
- [Maelstrom](https://github.com/jepsen-io/maelstrom) — a workbench that tests distributed systems against consistency models; its `lin-kv` workload fits this capstone.
- [FoundationDB — Simulation and Testing](https://apple.github.io/foundationdb/testing.html) — why deterministic simulation is worth the effort.

---

[🏠 Start](../START-HERE.md) · [← 23.2 Writing a design document](02-writing-a-design-document.md) · [📋 Phase 23](README.md) · [23.4 Capstone option 2: a high-performance HTTP reverse proxy →](04-capstone-http-reverse-proxy.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
