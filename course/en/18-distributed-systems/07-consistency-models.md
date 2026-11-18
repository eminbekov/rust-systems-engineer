<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.6 Replication](06-replication.md) · [📋 Phase 18](README.md) · [18.8 Partitioning and consistent hashing →](08-partitioning-and-consistent-hashing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/07-consistency-models.md) <!-- nav -->

# Lesson 18.7 — Consistency models

> **Phase 18 — Distributed systems** · Lesson 7 of 15 · ⏱ about 135 minutes

## What you will learn

- What a consistency model is: a contract that says which answers a replicated system is allowed to give.
- What linearizability means, and how to check a history of reads and writes for it with a small search program.
- The weaker models — sequential, causal and eventual consistency — and the session guarantees such as read-your-writes.
- What the CAP theorem really says about network partitions.
- How to choose the model a feature needs.

## Before you start

- You finished [Lesson 18.6 — Replication](06-replication.md). Its stale reads and lost writes are the problems this lesson gives names to.
- You know happens-before and concurrency from [Lesson 18.5](05-logical-clocks.md), and recursion with backtracking from [Lesson 07.7](../07-algorithms-and-data-structures/07-recursion.md).
- Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p18-consistency
cd code/p18-consistency
mkdir src/bin tests
```

## New words

| Word | Meaning |
|---|---|
| **consistency model** | A contract between a data store and its clients: the rules that say which values a read may return. |
| **register** | The simplest kind of data: one value that can be written and read. A key in a key-value store is a register. |
| **history** | The record of what clients did: each call with its start time, end time, kind (read or write) and value. |
| **linearizability** | The strongest common model: the system behaves as if there were **one** copy of the data and every operation took effect at a single instant between its start and its end. |
| **sequential consistency** | All clients see the operations in one order that keeps each client's own order, but that order may disagree with real time. |
| **causal consistency** | Operations that are causally related (happens-before) are seen in that order by everybody; concurrent ones may be seen in different orders. |
| **eventual consistency** | If no new writes happen, all replicas eventually return the same value. Nothing is promised about the meantime. |
| **session guarantee** | A promise for the calls of one client, for example "you always see your own writes". |
| **CAP theorem** | During a network **P**artition, a system cannot be both **C**onsistent (linearizable) and **A**vailable (answering every request). |
| **stale read** | A read that returns a value that was already overwritten. |

## 1. What a consistency model is

In Lesson 18.6 a client wrote `x = 2` and then read `x = 1` from a lagging follower. Is that a bug? It depends on
what the system **promised**. A **consistency model** is that promise: a set of rules that says which values a
read is allowed to return, given the writes that happened.

An everyday picture. You send a letter to a friend and then call them. "Strong" consistency means that the letter
is surely there when you call. "Eventual" consistency means the letter will arrive some day. Both are honest
promises; they are just different promises, with different costs.

Stronger models are easier to program against, because they behave more like a single computer. Weaker models
allow faster, more available systems, because replicas need to coordinate less. We go from strong to weak.

## 2. Linearizability: the illusion of one copy

A system is **linearizable** if every history it produces could have come from **one single copy** of the data,
where each operation takes effect at **one instant** between the moment the client sent it and the moment the
client got the answer.

The second part is the important one: real time matters. If a write **finished** before a read **started**, the read
must see that write (or a newer one). If two operations **overlap** in time, they may take effect in either order.

Here are four histories of one register that starts at 0. Each line is one client; a box `[....]` is a call from its
start to its end, one character per millisecond; `r→0` is a read that returned 0:

```text
ms    0         10        20        30
1 write, then read: linearizable
  A   [.write 1.]
  B                       [.read → 1]
2 stale read: NOT linearizable: the write had finished
  A   [.write 1.]
  B                       [.read → 0]
3 reads during a write: linearizable: the write took effect between the reads
  A   [...........write 1...........]
  B        [r→0.]
  C                  [r→1.]
4 new, then old: NOT linearizable: once 1 was seen, 0 is gone
  A   [...........write 1...........]
  B        [r→1.]
  C                  [r→0.]
```

History 4 surprises many people: both reads overlap with the write, so each one alone would be fine. But B's read
**finished** before C's read **started**, so B's read took effect first. If B already saw the new value, the write
had already happened, and C may not see the old value any more. A system that reads from replicas with different lag
produces exactly this history.

### A tiny checker

We can decide linearizability with a search: try to put the calls into an order, one by one, such that (1) a call
may go next only if no other waiting call **had to** come before it, and (2) every read returns the value of the
register at its place in the order. If some order works, the history is linearizable. This is backtracking
([Lesson 07.7](../07-algorithms-and-data-structures/07-recursion.md)): try a call, go deeper, and undo the choice if
it leads to a dead end. Put this into `src/lib.rs`:

```rust
//! Consistency models (Lesson 18.7).

/// What one call did to the register. A read carries the value it returned.
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Op {
    Write(u32),
    Read(u32),
}

/// One call in a history: which client made it, when it started and ended (in ms), and what it did.
#[derive(Debug, Clone, Copy)]
pub struct Call {
    pub client: char,
    pub start: u32,
    pub end: u32,
    pub op: Op,
}

/// A rule `must_precede(a, b)`: call `a` has to take effect before call `b`.
pub type Rule = fn(&Call, &Call) -> bool;

/// Linearizability: a call that ended before another started must take effect first.
fn real_time(a: &Call, b: &Call) -> bool {
    a.end < b.start
}

/// Searches for an order of the calls that a single copy of the register (starting at 0)
/// could have produced, where each call takes effect at one instant between its start and
/// end. Returns that order (as indexes into `history`), or `None` if there is none.
pub fn linearize(history: &[Call]) -> Option<Vec<usize>> {
    find_order(history, real_time)
}

/// Finds an order of all calls that respects `must_precede` and that a single register
/// starting at 0 could have produced.
pub fn find_order(history: &[Call], must_precede: Rule) -> Option<Vec<usize>> {
    let mut order = Vec::new();
    if search(history, must_precede, &mut order, 0) {
        Some(order)
    } else {
        None
    }
}

fn search(history: &[Call], must_precede: Rule, order: &mut Vec<usize>, value: u32) -> bool {
    if order.len() == history.len() {
        return true; // every call has found its place
    }
    for i in 0..history.len() {
        if order.contains(&i) {
            continue;
        }
        // Call i may take effect next only if no waiting call must come before it.
        let waiting = |j: &usize| !order.contains(j) && *j != i;
        if (0..history.len())
            .filter(waiting)
            .any(|j| must_precede(&history[j], &history[i]))
        {
            continue;
        }
        let next_value = match history[i].op {
            Op::Write(v) => v,
            Op::Read(v) if v == value => value,
            Op::Read(_) => continue, // this read cannot happen now
        };
        order.push(i);
        if search(history, must_precede, order, next_value) {
            return true;
        }
        order.pop(); // a dead end: try another call in this place
    }
    false
}
```

**What happens:**

1. `Call` records one operation of a history. For a read, `Op::Read(v)` stores the value that the read **returned**.
2. `Rule` is a *function pointer type*: any function with the signature `fn(&Call, &Call) -> bool` can be used as a
   value of this type. Keeping the rule separate lets Exercise 3 reuse the whole search for a different model.
3. For linearizability, the rule is real time: `a.end < b.start` means "a finished before b started, so a must come
   first".
4. `search` builds the order in the vector `order`. At each step it tries every call that is not placed yet. It skips a
   call if a waiting call must come before it, or if it is a read whose value does not match the current value of
   the register. Otherwise it places the call and goes deeper with the new value.
5. When the deeper search fails, `order.pop()` undoes the choice and the loop tries the next call. If every call is
   placed, a valid order has been found.

The search tries many orders, so it is only for small histories. Real checkers (such as the one in the Jepsen test
tool) are much smarter, but they answer the same question.

Now check our four histories, `src/bin/histories.rs`:

```rust
use p18_consistency::{Call, Op, linearize};

fn call(client: char, start: u32, end: u32, op: Op) -> Call {
    Call {
        client,
        start,
        end,
        op,
    }
}

fn main() {
    let histories = [
        (
            "1 write, then read",
            vec![
                call('A', 0, 10, Op::Write(1)),
                call('B', 20, 30, Op::Read(1)),
            ],
        ),
        (
            "2 stale read",
            vec![
                call('A', 0, 10, Op::Write(1)),
                call('B', 20, 30, Op::Read(0)),
            ],
        ),
        (
            "3 reads during a write",
            vec![
                call('A', 0, 30, Op::Write(1)),
                call('B', 5, 10, Op::Read(0)),
                call('C', 15, 20, Op::Read(1)),
            ],
        ),
        (
            "4 new, then old",
            vec![
                call('A', 0, 30, Op::Write(1)),
                call('B', 5, 10, Op::Read(1)),
                call('C', 15, 20, Op::Read(0)),
            ],
        ),
    ];
    for (name, history) in &histories {
        match linearize(history) {
            Some(order) => println!("{name:24} linearizable, order {order:?}"),
            None => println!("{name:24} NOT linearizable"),
        }
    }
}
```

`cargo run --bin histories`:

```text
1 write, then read       linearizable, order [0, 1]
2 stale read             NOT linearizable
3 reads during a write   linearizable, order [1, 0, 2]
4 new, then old          NOT linearizable
```

For history 3 the checker found the order "B reads 0, A writes 1, C reads 1": the write took effect somewhere between
10 and 15 ms. That is allowed, because the write was still running.

> 🧪 **Testing:** Recording the history of a test run and checking it is how real distributed databases are tested
> for consistency bugs. Add the two key cases as unit tests at the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn call(client: char, start: u32, end: u32, op: Op) -> Call {
        Call {
            client,
            start,
            end,
            op,
        }
    }

    #[test]
    fn a_stale_read_after_the_write_finished_is_not_linearizable() {
        let history = [
            call('A', 0, 10, Op::Write(1)),
            call('B', 20, 30, Op::Read(0)),
        ];
        assert_eq!(linearize(&history), None);
    }

    #[test]
    fn a_read_during_the_write_may_see_the_old_value() {
        let history = [
            call('A', 0, 30, Op::Write(1)),
            call('B', 5, 10, Op::Read(0)),
        ];
        assert_eq!(linearize(&history), Some(vec![1, 0]));
    }
}
```

`cargo test --lib`:

```text
test tests::a_stale_read_after_the_write_finished_is_not_linearizable ... ok
test tests::a_read_during_the_write_may_see_the_old_value ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

How do real systems become linearizable? Single-leader systems can read and write **only at the leader** (and make
sure it really still is the leader); consensus systems such as Raft (Lessons 18.9–18.11) do this. Reading from
asynchronous followers is not linearizable, as history 2 shows. Leaderless quorums with `R + W > N` are **not**
automatically linearizable either: history 4 can still happen while a write is reaching the replicas one by one.

## 3. Weaker models

Linearizability needs coordination: someone must make sure that every read sees every finished write, which costs
round trips and fails when the network is cut. Weaker models give up part of the promise:

| Model | The promise | Example of what is allowed |
|---|---|---|
| **Linearizable** | one copy, real-time order | nothing surprising |
| **Sequential** | one order for everybody that keeps each client's own order, but not real time | history 2: B's read may be placed before A's write, because they are different clients |
| **Causal** | happens-before is respected everywhere (Lesson 18.5); concurrent writes may be seen in different orders | two users see two unrelated comments in a different order, but a reply never appears before the comment it answers |
| **Eventual** | if writes stop, replicas converge | anything stale for a while, even going back and forth between old and new |

Between these, there are useful promises for **one client's session**, which can be given even by weakly
consistent systems:

| Session guarantee | Meaning | Lesson 18.6 problem it fixes |
|---|---|---|
| **Read your writes** | after you write, your reads see that write | the user who changed their name and saw the old one |
| **Monotonic reads** | once you saw a value, you never see an older one | reading from follower 0 and then from a more lagging follower 1 |
| **Monotonic writes** | your writes are applied in the order you made them | — |
| **Consistent prefix** | you see writes in an order that makes sense (an answer never before its question) | — |

A common way to get the first two: the client remembers the log position it has seen (Exercise 3 of Lesson 18.6)
and only accepts replicas that are at least that far, or it always talks to the same replica ("sticky" sessions).

## 4. The CAP theorem

The CAP theorem is often quoted as "pick two out of three: consistency, availability, partition tolerance". That is
misleading. Network partitions are not a choice; they happen (Lesson 18.2). The real statement is:

> **When the network is partitioned**, a system must choose: either it stays **linearizable** and some nodes refuse
> requests (not available), or every node keeps answering and some answers may be stale (not linearizable).

```text
           partition
  client ── node 1  ║  node 2 ── client
   writes x = 2     ║     reads x
                    ║
  CP choice: node 2 refuses to answer ("I cannot reach the others") → consistent, not available
  AP choice: node 2 answers with what it has (x = 1)                  → available, not consistent
```

The two labels come from the letters of CAP: **CP** means "stays **C**onsistent during a **P**artition", **AP** means
"stays **A**vailable during a **P**artition". A Raft cluster makes the CP choice: the side with a majority keeps working, the minority side refuses writes and
linearizable reads. A shopping cart in a leaderless store makes the AP choice: every replica accepts writes, and
conflicts are merged later (Lesson 18.5).

There is a second trade-off that matters even when there is **no** partition: stronger consistency costs latency,
because replicas must talk before they answer. The PACELC formulation (if **P**artition: **A** or **C**; **E**lse:
**L**atency or **C**onsistency) names both.

> 💡 **Tip:** Labels like "CP database" or "AP database" are rough. Most real databases offer several modes, and the
> same database can be linearizable for some operations and not for others. Ask "which guarantee does this
> particular read have?" instead.

## 5. Choosing a model

| Feature | What it needs | Why |
|---|---|---|
| A username must be unique | linearizable | two users registering "anna" at the same moment must not both succeed |
| Account balance and transfers | linearizable (and transactions) | money must not appear or disappear |
| "Who is the leader?" or a lock | linearizable (consensus) | split brain otherwise (Lesson 18.6) |
| A user's own profile page | read your writes | they must see their change |
| Comments on a post | causal | replies after the comment they answer |
| A "likes" counter, a "last seen" time | eventual | a slightly old number harms nobody |

## Common mistakes

> ⚠️ **Common mistake: "quorums make it linearizable".** `R + W > N` makes read and write groups overlap, but history 4
> can still happen while a write is half done. Linearizable quorum systems need extra steps (for example, a read
> that finds different versions must write the newest one back to a quorum **before** it answers).

> ⚠️ **Common mistake: testing consistency on one machine.** On a laptop with no lag every system looks linearizable.
> Consistency bugs appear only with delays, partitions and failovers, so tests must create them; Lesson 18.12 does
> this with a simulator.

> ⚠️ **Common mistake: forgetting real time.** "All replicas agree on the order" is sequential consistency, not
> linearizability. The difference shows up exactly when a client tells another client "I have written it, go read it".

## Try it yourself

1. Add a fifth history to `histories.rs`: A writes 1 from 0 to 10 ms, then B writes 2 from 20 to 30 ms, then C reads
   1 from 40 to 50 ms. Predict the answer, then run it (not linearizable: C read an overwritten value).
2. Move C's read in that history to 25–35 ms, during B's write. Now it is linearizable, with the order `[0, 2, 1]`:
   the read took effect before the second write.
3. In history 4, make B's read end at 16 ms instead of 10. Now the two reads overlap, and the history is linearizable.

## Exercises

### Exercise 1 — When "new, then old" is fine (easy)

Write an integration test `tests/overlap.rs` with history 4 and a variant in which C's read starts at 8 ms, so that the
two reads **overlap**. Check that history 4 is not linearizable and that the variant is, with the order `[2, 0, 1]`.

<details>
<summary>Hint</summary>

Copy the small `call` helper into the test file. When two reads overlap, neither must come before the other, so the
read of 0 can take effect first, before the write.

</details>

<details>
<summary>Answer</summary>

```rust
use p18_consistency::{Call, Op, linearize};

fn call(client: char, start: u32, end: u32, op: Op) -> Call {
    Call {
        client,
        start,
        end,
        op,
    }
}

#[test]
fn new_then_old_is_wrong_only_if_the_reads_do_not_overlap() {
    let one_after_the_other = [
        call('A', 0, 30, Op::Write(1)),
        call('B', 5, 10, Op::Read(1)),
        call('C', 15, 20, Op::Read(0)),
    ];
    assert_eq!(linearize(&one_after_the_other), None);

    let overlapping = [
        call('A', 0, 30, Op::Write(1)),
        call('B', 5, 10, Op::Read(1)),
        call('C', 8, 20, Op::Read(0)),
    ];
    assert_eq!(linearize(&overlapping), Some(vec![2, 0, 1]));
}
```

`cargo test --test overlap`:

```text
test new_then_old_is_wrong_only_if_the_reads_do_not_overlap ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: in the variant, C's read (8–20 ms) started before B's read (5–10 ms) ended, so the order "C reads 0,
A writes 1, B reads 1" respects every real-time constraint. Linearizability is about what clients could have
observed, and here they could not tell.

</details>

### Exercise 2 — Monotonic reads (medium)

A client makes several reads, possibly from different replicas. For each read we know how many log entries the
answering replica had applied. Create `src/session.rs` (and `pub mod session;`) with
`pub fn first_backwards_read(positions: &[usize]) -> Option<usize>` that returns the index of the first read that saw
**fewer** entries than the read before it, or `None` if the reads never went back in time. Test it with
`[3, 5, 5, 4, 6]` (answer 3), `[1, 2, 2, 3]` and an empty list (both `None`).

<details>
<summary>Hint</summary>

`positions.windows(2)` gives every pair of neighbours; `.position(...)` finds the first pair where the second number
is smaller. The read that went back is the **second** of the pair, so add 1.

</details>

<details>
<summary>Answer</summary>

```rust
//! Exercise 2: monotonic reads.

/// `positions[k]` is how many log entries the replica had applied when it answered the
/// client's k-th read. Returns the index of the first read that went back in time.
pub fn first_backwards_read(positions: &[usize]) -> Option<usize> {
    positions
        .windows(2)
        .position(|pair| pair[1] < pair[0])
        .map(|i| i + 1)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn switching_to_a_lagging_replica_goes_back_in_time() {
        assert_eq!(first_backwards_read(&[3, 5, 5, 4, 6]), Some(3));
        assert_eq!(first_backwards_read(&[1, 2, 2, 3]), None);
        assert_eq!(first_backwards_read(&[]), None);
    }
}
```

Why it works: monotonic reads only forbid going **back**; equal positions (the same replica twice, or two replicas
that are equally up to date) are fine. A client that passes its highest position with every read lets the system
refuse replicas that would break this rule.

</details>

### Exercise 3 — Sequential consistency (hard)

Create `src/sequential.rs` (and `pub mod sequential;`) with `pub fn sequentialize(history: &[Call]) -> Option<Vec<usize>>`
that reuses `find_order` with a different rule: only an **earlier call of the same client** must come first; real time
between different clients does not matter. Test two cases: history 2 (the stale read by another client) is not
linearizable but is sequentially consistent, with the order `[1, 0]`; and a history in which client A writes 1 and
then **itself** reads 0 is not sequentially consistent.

<details>
<summary>Hint</summary>

The rule is `a.client == b.client && a.start < b.start`. Write it as a normal `fn` with the signature of `Rule`, and pass
its name to `find_order`.

</details>

<details>
<summary>Answer</summary>

```rust
//! Exercise 3: sequential consistency.

use crate::{Call, find_order};

/// Sequential consistency: only the order of each client's own calls must be kept.
fn same_client_earlier(a: &Call, b: &Call) -> bool {
    a.client == b.client && a.start < b.start
}

pub fn sequentialize(history: &[Call]) -> Option<Vec<usize>> {
    find_order(history, same_client_earlier)
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::{Op, linearize};

    fn call(client: char, start: u32, end: u32, op: Op) -> Call {
        Call {
            client,
            start,
            end,
            op,
        }
    }

    #[test]
    fn another_clients_stale_read_is_allowed() {
        let history = [
            call('A', 0, 10, Op::Write(1)),
            call('B', 20, 30, Op::Read(0)),
        ];
        assert_eq!(linearize(&history), None);
        assert_eq!(sequentialize(&history), Some(vec![1, 0]));
    }

    #[test]
    fn a_client_must_see_its_own_write() {
        let history = [
            call('A', 0, 10, Op::Write(1)),
            call('A', 20, 30, Op::Read(0)),
        ];
        assert_eq!(sequentialize(&history), None);
    }
}
```

`cargo test --lib`:

```text
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: sequential consistency allows the order "B reads 0, then A writes 1", because B and A are different
clients and only each client's own order counts. It still does not allow a client to miss its own write. Seeing the
two checkers differ only in one small rule shows how close the definitions are.

</details>

## Quiz

1. What is a consistency model?

<details>
<summary>Answer</summary>

A contract between a data store and its clients that says which values reads may return, given the writes that
happened.

</details>

2. A write finished at 10 ms. A read by another client started at 20 ms and returned the old value. Which models does
   this history break: linearizable, sequential, eventual?

<details>
<summary>Answer</summary>

It breaks linearizability (real time). It can be sequentially consistent (the read is ordered before the write) and
eventually consistent.

</details>

3. Why is history 4 ("new, then old") not linearizable, although both reads overlap with the write?

<details>
<summary>Answer</summary>

The two reads do not overlap each other: B's read finished before C's started, so B's must take effect first. B saw the
new value, so the write had taken effect, and C can no longer see the old value.

</details>

4. State the CAP theorem correctly.

<details>
<summary>Answer</summary>

During a network partition, a system must choose between linearizability and availability: either some requests are
refused, or some answers may be stale.

</details>

5. Which consistency do you need for "a username must be unique", and which for "the number of likes"?

<details>
<summary>Answer</summary>

Uniqueness needs linearizability (or a transaction over a single leader), otherwise two users can take the same name at
the same moment. A likes counter is fine with eventual consistency.

</details>

## Summary

- A consistency model is a contract about which values reads may return. Stronger models are easier to use; weaker
  ones allow faster and more available systems.
- Linearizability: the system behaves like one copy, and every operation takes effect at one instant between its start
  and end. Real time matters.
- A small backtracking search can check a history for linearizability; with a different rule it checks sequential
  consistency.
- Sequential, causal and eventual consistency are weaker; session guarantees (read your writes, monotonic reads)
  protect one client's experience.
- CAP: during a partition, choose linearizability or availability. Without a partition, the trade-off is latency
  against consistency.
- Choose per feature: uniqueness, money and leadership need linearizability; counters and feeds do not.

## Sources and further reading

- [Consistency Models](https://jepsen.io/consistency) — Jepsen's clickable map of consistency models, with a page for each model, including [linearizability](https://jepsen.io/consistency/models/linearizable).
- [Linearizability: A Correctness Condition for Concurrent Objects](https://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf) — the 1990 paper by Herlihy and Wing that defined linearizability.
- [Please stop calling databases CP or AP](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html) — Martin Kleppmann on what the CAP theorem does and does not say.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — the chapter on consistency and consensus explains linearizability, ordering and the cost of strong guarantees.

---

[🏠 Start](../START-HERE.md) · [← 18.6 Replication](06-replication.md) · [📋 Phase 18](README.md) · [18.8 Partitioning and consistent hashing →](08-partitioning-and-consistent-hashing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
