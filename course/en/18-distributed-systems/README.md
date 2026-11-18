<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.15 Checkpoint](../17-storage-and-databases/15-checkpoint.md) · [18.1 What a distributed system is →](01-what-a-distributed-system-is.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/README.md) <!-- nav -->

# Phase 18 — Distributed systems

> **Stage 4 — Build real systems** · ⏱ about 5–6 weeks at ~10 hours per week

## What this phase is about

A distributed system is a group of programs on several computers that work together as one service and talk only by sending messages over a network. Almost every serious backend is one: a database with replicas, a message broker cluster, a key-value store spread over many machines. In Phases 15 to 17 you built the parts — network code, services and a storage engine. This phase is about what happens when you run many copies of them and the network, the clocks and the machines start to fail.

The first half teaches you to think clearly about failure. You learn why a timeout can never tell you whether the other side did the work, how to retry safely with backoff, jitter and idempotency keys, why the clocks of two machines disagree, and how logical clocks order events without them. Then you keep copies of data: replication with a leader and followers, quorums without a leader, the consistency models that say exactly which answers a replicated system may give, and partitioning with consistent hashing.

The second half is about agreement. You study consensus and build the core of Raft — leader election and log replication — and you learn the most valuable testing skill of the phase: deterministic simulation with `turmoil`, which runs a whole cluster inside one test with network partitions, delays and crashes, and replays any failure exactly. The phase ends with distributed transactions and sagas, and a project that turns your key-value store from Lesson 17.14 into a replicated, fault-tolerant service.

Every lesson uses a small package in `code/p18-<topic>` with real, tested code. Most examples simulate the network inside one program, so that each failure happens exactly when the lesson wants it to — which is also how professionals test such systems.

## What you will be able to do

- [ ] Explain partial failure, and why a timeout means "unknown", not "failed".
- [ ] Write retries with exponential backoff and full jitter, and make non-idempotent operations safe with idempotency keys.
- [ ] Choose the right clock for a task, and explain why last-write-wins and leases without fencing tokens lose data.
- [ ] Implement Lamport and vector clocks and use them to detect concurrent writes.
- [ ] Explain single-leader, multi-leader and leaderless replication, replication lag, failover and quorums (`R + W > N`).
- [ ] Define linearizability, sequential, causal and eventual consistency, check a history for linearizability, and state the CAP theorem correctly.
- [ ] Partition data with consistent hashing and virtual nodes, and implement Raft leader election and log replication.
- [ ] Test a distributed system deterministically with simulated partitions, delays and crashes.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 18.1 | [What a distributed system is](01-what-a-distributed-system-is.md) | 120 min |
| 18.2 | [Failures and timeouts](02-failures-and-timeouts.md) | 135 min |
| 18.3 | [Retries, backoff and idempotency](03-retries-backoff-and-idempotency.md) | 3 h |
| 18.4 | [Time and clocks](04-time-and-clocks.md) | 2.5 h |
| 18.5 | [Logical clocks: Lamport and vector clocks](05-logical-clocks.md) | 135 min |
| 18.6 | [Replication](06-replication.md) | 135 min |
| 18.7 | [Consistency models](07-consistency-models.md) | 135 min |
| 18.8 | [Partitioning and consistent hashing](08-partitioning-and-consistent-hashing.md) | 120 min |
| 18.9 | [Consensus and an overview of Raft](09-consensus-and-raft.md) | 135 min |
| 18.10 | [Raft: leader election](10-raft-leader-election.md) | 3 h |
| 18.11 | [Raft: log replication](11-raft-log-replication.md) | 3 h |
| 18.12 | [Deterministic simulation testing](12-deterministic-simulation-testing.md) | 135 min |
| 18.13 | [Distributed transactions and sagas](13-distributed-transactions-and-sagas.md) | 135 min |
| 18.14 | [Project: a replicated key-value store](14-project-replicated-key-value-store.md) | 20 h |
| 18.15 | [Checkpoint: Stage 4 complete](15-checkpoint-stage-4-complete.md) | 120 min |

## Projects in this phase

- **Small packages in the lessons** (`code/p18-nodes`, `code/p18-failures`, `code/p18-retries`, `code/p18-clocks`, `code/p18-logical-clocks`, `code/p18-replication`, `code/p18-consistency`, `code/p18-partitioning`, `code/p18-consensus`, `code/p18-raft` (Lessons 18.10 and 18.11), `code/p18-sim`, `code/p18-saga` and `code/p18-checkpoint`): each one has tests, real output and exercises that ask you to write a test.
- **[Project 18.14 — A replicated key-value store](14-project-replicated-key-value-store.md)**: put your Raft (from Lessons 18.10 and 18.11) in front of your key-value store from Lesson 17.14, and prove with deterministic simulation tests in `turmoil` that it keeps its promises through partitions and crashes.

## How to study this phase

1. **Draw the timeline.** For every scenario, sketch the nodes as horizontal lines and the messages as arrows, as the lessons do. Most distributed-systems bugs become obvious on paper.
2. **Break things on purpose.** Pause a node with Ctrl+Z, lose a reply, skew a clock, fail over at the worst moment. Each lesson shows one failure; invent two more and predict the result before you run it.
3. **Ask "what does the client know?"** After every timeout, retry or failover, write down what the client can be sure of and what it cannot. This one question prevents most design mistakes.
4. **Prefer pure logic and simulated time.** Keep algorithms free of real clocks and sockets (pass `now` in, return messages out), so that tests can drive them deterministically. Lessons 18.10 to 18.14 depend on this habit.
5. **Read one real source per week.** The Raft paper, a Jepsen analysis or a chapter of *Designing Data-Intensive Applications* next to the lessons makes the ideas stick.

---

[🏠 Start](../START-HERE.md) · [← 17.15 Checkpoint](../17-storage-and-databases/15-checkpoint.md) · [18.1 What a distributed system is →](01-what-a-distributed-system-is.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
