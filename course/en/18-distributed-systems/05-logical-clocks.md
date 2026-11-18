<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.4 Time and clocks](04-time-and-clocks.md) · [📋 Phase 18](README.md) · [18.6 Replication →](06-replication.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/05-logical-clocks.md) <!-- nav -->

# Lesson 18.5 — Logical clocks: Lamport and vector clocks

> **Phase 18 — Distributed systems** · Lesson 5 of 15 · ⏱ about 135 minutes

## What you will learn

- What "happened before" means when there is no shared clock, and what it means for two events to be *concurrent*.
- How a Lamport clock gives every event a number that respects cause and effect, with two lines of code.
- What a Lamport clock cannot tell you, and how a vector clock fixes that.
- How to compare two vector clocks: before, after, equal or concurrent.
- How replicated databases use vector clocks to detect conflicting writes instead of losing one of them.

## Before you start

- You finished [Lesson 18.4 — Time and clocks](04-time-and-clocks.md). Its conclusion is the starting point here: wall-clock timestamps from different machines cannot order events.
- You know `BTreeMap` from Phase 04, implementing `Display` from Phase 05, and `proptest` from [Lesson 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md).
- Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p18-logical-clocks
cd code/p18-logical-clocks
cargo add --dev proptest@1
mkdir src/bin tests
```

## New words

| Word | Meaning |
|---|---|
| **event** | Anything that happens on one node: a local step, sending a message or receiving one. |
| **happens-before** (→) | "a → b" means event a could have influenced event b: either a came first on the same node, or a message carried a's effect to b, possibly through other events in between. |
| **concurrent** events | Two events where neither happened before the other. No information could flow from one to the other. It says nothing about the wall-clock time. |
| **causality** | The relation of cause and effect between events; in distributed systems, the same as happens-before. |
| **logical clock** | A counter that orders events by happens-before instead of by physical time. |
| **Lamport clock** | A logical clock with one number per node, invented by Leslie Lamport in 1978. |
| **vector clock** | A logical clock with one counter **per node** in every timestamp. It can tell concurrent events apart from ordered ones. |
| **total order** | An order in which any two different items can be compared: one is always first. |
| **partial order** | An order in which some pairs cannot be compared. Concurrent events are such pairs. |
| **sibling** | One of several concurrent versions of the same value that a database keeps because it cannot know which is newer. |

## 1. Ordering without a clock

Lesson 18.4 showed that wall clocks cannot decide which of two events on different machines came first. But
often we do not need the real time at all. We need something weaker and much more useful: **could event a have
influenced event b?** If node b read a value that node a wrote, b's next write must count as "after" a's
write, whatever the clocks say.

This is the **happens-before** relation, written a → b. It is defined by three rules:

1. If a and b happen on the **same node** and a comes first, then a → b.
2. If a is **sending** a message and b is **receiving** that message, then a → b.
3. If a → b and b → c, then a → c (it is *transitive*).

If neither a → b nor b → a, the events are **concurrent**: no information could flow between them. This is the
example we use for the whole lesson. Time flows to the right; arrows between lines are messages:

```text
node a:  a1 ──── a2 ─────────────────────────────── a3
                  \ m1
node b:      b1 ── b2 ── b3
                          \ m2
node c:         c1 ─────── c2
```

- a1 → a2 → b2 → b3 → c2: a chain of steps and messages. So a1 → c2: what node a knew at a1 could reach c2.
- a3 and c2 are concurrent: no message went from a3 to node c, or from c2 to node a, so neither could
  influence the other — even if a3 happened "later" by the wall clock.
- b1 and a1 are concurrent too.

## 2. Lamport clocks

In 1978 Leslie Lamport proposed a very small rule that gives every event a number, so that **if a → b, then
L(a) < L(b)**:

- Every node keeps a counter, starting at 0.
- Before each local event or send, the node adds 1. A message carries the sender's counter.
- When a node receives a message with counter `remote`, it sets its counter to `max(own, remote) + 1`.

Put this into `src/lib.rs`:

```rust
//! Logical clocks (Lesson 18.5).

/// A Lamport clock: one counter that every node keeps for itself.
#[derive(Debug, Default)]
pub struct LamportClock {
    time: u64,
}

impl LamportClock {
    /// A local event or sending a message: count one up.
    pub fn tick(&mut self) -> u64 {
        self.time += 1;
        self.time
    }

    /// Receiving a message that carries the sender's time `remote`: jump past it.
    pub fn receive(&mut self, remote: u64) -> u64 {
        self.time = self.time.max(remote) + 1;
        self.time
    }
}
```

Now replay the example from the diagram, `src/bin/lamport.rs`:

```rust
use p18_logical_clocks::LamportClock;

fn main() {
    let mut a = LamportClock::default();
    let mut b = LamportClock::default();
    let mut c = LamportClock::default();

    println!("a: local event           L={}", a.tick());
    let m1 = a.tick();
    println!("a: sends m1 to b         L={m1}");
    println!("b: local event           L={}", b.tick());
    println!("b: receives m1           L={}", b.receive(m1));
    let m2 = b.tick();
    println!("b: sends m2 to c         L={m2}");
    println!("c: local event           L={}", c.tick());
    println!("c: receives m2           L={}", c.receive(m2));
    println!("a: local event           L={}", a.tick());
}
```

`cargo run --bin lamport`:

```text
a: local event           L=1
a: sends m1 to b         L=2
b: local event           L=1
b: receives m1           L=3
b: sends m2 to c         L=4
c: local event           L=1
c: receives m2           L=5
a: local event           L=3
```

**What happens:**

1. Node b's own counter was 1 when m1 arrived with 2. `max(1, 2) + 1 = 3`: b jumps **past** everything a knew when
   it sent the message. Node c does the same with m2: `max(1, 4) + 1 = 5`.
2. Follow the chain a1 → a2 → b2 → b3 → c2: the numbers are 1, 2, 3, 4, 5. They grow along every chain of cause
   and effect. That is the guarantee.
3. Now look at a3 (L=3) and c2 (L=5). 3 < 5, but we saw in Section 1 that the two events are **concurrent**. The
   rule only works one way: a → b means L(a) < L(b), but L(a) < L(b) does **not** mean a → b.

A Lamport clock is still very useful. Pairs `(L, node id)` give a **total order** of all events that every node
computes the same way and that never contradicts causality: compare L first, and use the node id to break ties. Many
algorithms need exactly that, for example to decide which of two requests for a lock came "first" in a fair way.
Raft's term numbers (Lesson 18.10) work in a similar way to Lamport time.

## 3. Vector clocks

To tell concurrent events apart from ordered ones, each timestamp must remember **what the event knew about every
node**. A **vector clock** is a map `node → counter`:

- Before each local event or send, node n adds 1 to **its own** entry.
- A message carries the sender's whole vector.
- On receive, the node takes the maximum of each entry (it learns everything the sender knew), then adds 1 to
  its own entry.

Two vector clocks V and W compare like this:

| Condition | Result |
|---|---|
| every entry of V ≤ the entry of W, and at least one is < | V **before** W |
| every entry of V ≥ the entry of W, and at least one is > | V **after** W |
| all entries equal | **equal** |
| some entries smaller, others larger | **concurrent** |

A missing entry counts as 0. Add `use std::collections::BTreeMap;` and `use std::fmt;` at the top of `src/lib.rs`,
then:

```rust
/// How two events are related.
#[derive(Debug, PartialEq)]
pub enum Order {
    Before,
    After,
    Equal,
    Concurrent,
}

/// A vector clock: one counter for every node, by node name.
#[derive(Debug, Clone, Default, PartialEq)]
pub struct VectorClock {
    counts: BTreeMap<String, u64>,
}

impl VectorClock {
    /// A local event on `node` (also used before sending a message).
    pub fn tick(&mut self, node: &str) {
        *self.counts.entry(node.to_string()).or_insert(0) += 1;
    }

    /// Takes in everything the other clock knows: the maximum of each counter.
    pub fn merge(&mut self, other: &VectorClock) {
        for (node, &count) in &other.counts {
            let mine = self.counts.entry(node.clone()).or_insert(0);
            *mine = (*mine).max(count);
        }
    }

    /// Receiving a message on `node` that carries the sender's clock.
    pub fn receive(&mut self, node: &str, message: &VectorClock) {
        self.merge(message);
        self.tick(node);
    }

    fn get(&self, node: &str) -> u64 {
        self.counts.get(node).copied().unwrap_or(0)
    }

    pub fn compare(&self, other: &VectorClock) -> Order {
        let mut smaller = false; // some counter of self is smaller than in other
        let mut larger = false; // some counter of self is larger than in other
        for node in self.counts.keys().chain(other.counts.keys()) {
            let (a, b) = (self.get(node), other.get(node));
            smaller |= a < b;
            larger |= a > b;
        }
        match (smaller, larger) {
            (false, false) => Order::Equal,
            (true, false) => Order::Before,
            (false, true) => Order::After,
            (true, true) => Order::Concurrent,
        }
    }
}

impl fmt::Display for VectorClock {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let parts: Vec<String> = self
            .counts
            .iter()
            .map(|(n, c)| format!("{n}:{c}"))
            .collect();
        write!(f, "[{}]", parts.join(" "))
    }
}
```

**What happens:**

1. `tick` adds 1 to the node's own entry, creating it with 0 first if needed (`entry(...).or_insert(0)`).
2. `merge` takes the maximum of every entry; `receive` is merge plus tick, exactly the rule above.
3. `compare` walks over the nodes of both clocks and remembers whether **any** entry was smaller and whether **any**
   was larger. The `match` turns the two flags into the four results of the table. (A node that appears in both
   maps is checked twice; that does no harm.)
4. `Display` prints a clock as `[a:2 b:3]`. A `BTreeMap` keeps its keys sorted, so the output is stable.

The example again, `src/bin/vector.rs`:

```rust
use p18_logical_clocks::VectorClock;

fn main() {
    let mut a = VectorClock::default();
    let mut b = VectorClock::default();
    let mut c = VectorClock::default();

    a.tick("a");
    let a1 = a.clone();
    println!("a1 a: local event    {a}");
    a.tick("a");
    let m1 = a.clone();
    println!("a2 a: sends m1       {a}");
    b.tick("b");
    println!("b1 b: local event    {b}");
    b.receive("b", &m1);
    println!("b2 b: receives m1    {b}");
    b.tick("b");
    let m2 = b.clone();
    println!("b3 b: sends m2       {b}");
    c.tick("c");
    let c1 = c.clone();
    println!("c1 c: local event    {c}");
    c.receive("c", &m2);
    let c2 = c.clone();
    println!("c2 c: receives m2    {c}");
    a.tick("a");
    let a3 = a.clone();
    println!("a3 a: local event    {a}");

    println!();
    println!("a1 vs c2: {:?}", a1.compare(&c2));
    println!("c2 vs a1: {:?}", c2.compare(&a1));
    println!("a3 vs c1: {:?}", a3.compare(&c1));
    println!("a3 vs c2: {:?}", a3.compare(&c2));
}
```

`cargo run --bin vector`:

```text
a1 a: local event    [a:1]
a2 a: sends m1       [a:2]
b1 b: local event    [b:1]
b2 b: receives m1    [a:2 b:2]
b3 b: sends m2       [a:2 b:3]
c1 c: local event    [c:1]
c2 c: receives m2    [a:2 b:3 c:2]
a3 a: local event    [a:3]

a1 vs c2: Before
c2 vs a1: After
a3 vs c1: Concurrent
a3 vs c2: Concurrent
```

Read the vector of c2, `[a:2 b:3 c:2]`, as "c2 has seen 2 events of node a, 3 of node b and 2 of its own". a3 is
`[a:3]`: it is ahead of c2 in a's entry (3 > 2) but behind in b's and c's. Neither knows everything the other
knows, so they are concurrent — the case that Lamport clocks could not see.

> 🧪 **Testing:** The rules of vector clocks are perfect for property-based tests: "merging in any order gives the
> same clock", "a clock is never after the merge of itself with another". Exercise 2 writes them. For now add one
> example test to the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_message_orders_the_events() {
        let mut a = VectorClock::default();
        let mut b = VectorClock::default();
        a.tick("a"); // a sends a message
        let message = a.clone();
        b.tick("b"); // b does something before the message arrives
        let b_before = b.clone();
        b.receive("b", &message);
        assert_eq!(message.compare(&b), Order::Before);
        assert_eq!(b.compare(&message), Order::After);
        assert_eq!(message.compare(&b_before), Order::Concurrent);
    }
```

`cargo test --lib`:

```text
test tests::a_message_orders_the_events ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 4. What vector clocks are used for

**Detecting conflicts.** Some replicated databases let clients write to any replica (Lesson 18.6 calls this
*leaderless* replication). Two clients may change the same shopping cart on two replicas at the same time. If the
database used last write wins (Lesson 18.4), one change would silently disappear. With a vector clock on every
version, the database can see that the two versions are **concurrent**, keep **both** as *siblings*, and let the
application merge them (for a cart: the union of the items). If one version is *after* the other, the older one
can safely be thrown away. Exercise 3 builds this decision.

**Causal delivery.** A chat system can hold back a reply until the message it answers has been shown, by comparing
vector clocks.

**The cost.** A vector clock has one entry per node that ever wrote, so it grows with the size of the system, and it
travels with every message and every stored version. Real systems limit it, for example by counting per replica
(not per client) and by removing old entries. Another popular compromise is the **hybrid logical clock**: a
wall-clock time plus a small logical counter. It stays close to real time but never goes backwards and respects
causality like a Lamport clock. Several modern databases use it.

| Clock | Size of a timestamp | If a → b | Can detect concurrency |
|---|---|---|---|
| Wall clock | one number | **not** guaranteed: skew can reverse the order | no |
| Lamport clock | one number | L(a) < L(b) | no |
| Vector clock | one number per node | V(a) before V(b) | **yes** |

> 💡 **Tip:** Rust's standard library has a trait made for partial orders: `PartialOrd`, whose `partial_cmp`
> returns `Option<Ordering>`. A vector clock could implement it and return `None` for concurrent clocks. We use an
> explicit `compare` with an `Order` enum instead, because "concurrent" is an important answer here, not a missing one.

## Common mistakes

**Comparing vector clocks with `<`.** Because `VectorClock` does not implement `PartialOrd`, `a1 < c2` does not
compile:

```text
error[E0369]: binary operation `<` cannot be applied to type `VectorClock`
  --> src/bin/vector.rs:32:37
   |
32 |     println!("a1 before c2: {}", a1 < c2);
   |                                  -- ^ -- VectorClock
   |                                  |
   |                                  VectorClock
   |
note: `VectorClock` does not implement `PartialOrd`
```

That is a good thing: a plain "less than" would hide the fourth answer. Use `a1.compare(&c2)` and handle all four
variants of `Order` in a `match`.

> ⚠️ **Common mistake: reading Lamport order as cause and effect.** L(a) < L(b) does not mean that a caused b or
> even that a came first. It only means that b did not happen before a. To know whether two events are
> concurrent, you need vector clocks.

> ⚠️ **Common mistake: forgetting to tick on receive.** If `receive` only merges, the receive event gets the same
> vector as the message, and the two events look equal. Every event, including a receive, adds 1 to the node's own
> entry.

## Try it yourself

1. In `vector.rs`, add the comparisons `b1 vs a1` and `b1 vs c2` (save `b1` with `let b1 = b.clone();`). Predict
   the results from the diagram before you run it.
2. Let node c send a message m3 back to node a after c2, and let a receive it as a new event a4. Print a4's vector:
   it now dominates every other event, and `a3 vs a4` is `Before`.
3. Do the same with the Lamport clock in `lamport.rs`. What is L(a4)?

## Exercises

### Exercise 1 — Lamport jumps (easy)

Write an integration test `tests/lamport.rs`: a fresh clock ticks to 1; receiving a message stamped 10 gives 11; the
next tick gives 12; receiving an **old** message stamped 3 gives 13, not 4.

<details>
<summary>Hint</summary>

Every call returns the new time, so each step is one `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
use p18_logical_clocks::LamportClock;

#[test]
fn receiving_jumps_past_the_sender() {
    let mut b = LamportClock::default();
    assert_eq!(b.tick(), 1);
    assert_eq!(b.receive(10), 11); // the sender was far ahead
    assert_eq!(b.tick(), 12);
    assert_eq!(b.receive(3), 13); // an old message never moves the clock back
}
```

`cargo test --test lamport`:

```text
test receiving_jumps_past_the_sender ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `max(own, remote) + 1` never moves a clock backwards, so an old or delayed message cannot make a node
"forget" what it already knew.

</details>

### Exercise 2 — Properties of vector clocks (medium)

Write property-based tests in `tests/vector_props.rs`. Generate clocks by ticking the nodes "a", "b" and "c" in a
random order (a list of up to 10 indices from `0..3`). Property 1: merging x into y gives the same clock as merging
y into x, and both x and y are `Before` or `Equal` to the merged clock. Property 2: if x compared with y is `Before`,
then y compared with x is `After` (and the same for the other three answers).

<details>
<summary>Hint</summary>

Write a helper `fn clock(ticks: &[usize]) -> VectorClock` that ticks `["a", "b", "c"][i]` for every index, and a
strategy `prop::collection::vec(0usize..3, 0..10)`. `matches!(value, Order::Before | Order::Equal)` checks for one
of two variants.

</details>

<details>
<summary>Answer</summary>

```rust
use p18_logical_clocks::{Order, VectorClock};
use proptest::prelude::*;

/// Builds a clock by ticking the nodes "a", "b" and "c" in the given order.
fn clock(ticks: &[usize]) -> VectorClock {
    let mut clock = VectorClock::default();
    for &i in ticks {
        clock.tick(["a", "b", "c"][i]);
    }
    clock
}

fn ticks() -> impl Strategy<Value = Vec<usize>> {
    prop::collection::vec(0usize..3, 0..10)
}

proptest! {
    #[test]
    fn merging_gives_a_clock_that_knows_both(x in ticks(), y in ticks()) {
        let (x, y) = (clock(&x), clock(&y));
        let mut xy = x.clone();
        xy.merge(&y);
        let mut yx = y.clone();
        yx.merge(&x);
        prop_assert_eq!(&xy, &yx); // the order of merging does not matter
        prop_assert!(matches!(x.compare(&xy), Order::Before | Order::Equal));
        prop_assert!(matches!(y.compare(&xy), Order::Before | Order::Equal));
    }

    #[test]
    fn before_and_after_are_mirror_images(x in ticks(), y in ticks()) {
        let (x, y) = (clock(&x), clock(&y));
        let expected = match x.compare(&y) {
            Order::Before => Order::After,
            Order::After => Order::Before,
            Order::Equal => Order::Equal,
            Order::Concurrent => Order::Concurrent,
        };
        prop_assert_eq!(y.compare(&x), expected);
    }
}
```

`cargo test --test vector_props`:

```text
test before_and_after_are_mirror_images ... ok
test merging_gives_a_clock_that_knows_both ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Why it works: the maximum is commutative, so the order of merging cannot matter, and a maximum is never smaller than
its inputs. Property 2 checks that `compare` is consistent with itself; a typo in one arm of its `match` would break
it at once.

</details>

### Exercise 3 — Keep both versions when in doubt (hard)

Create `src/siblings.rs` (and `pub mod siblings;`) with a `Version { clock: VectorClock, value: String }` and a
function `pub fn resolve(a: Version, b: Version) -> Vec<Version>` that returns only the newer version when the two
are ordered (only one when they are equal) and **both** when they are concurrent. Test it with a shopping list:
`[r1:1] "milk"` is replaced by `[r1:2] "milk, eggs"`, but `[r1:2] "milk, eggs"` and `[r1:1 r2:1] "milk, bread"`
(written on two replicas after both saw "milk") are both kept.

<details>
<summary>Hint</summary>

Implement `resolve` as a `match` on `a.clock.compare(&b.clock)`. In the test, a small helper that builds a `Version`
from a list of node names to tick keeps the cases readable.

</details>

<details>
<summary>Answer</summary>

```rust
//! Exercise 3: keep both values when two writes were concurrent.

use crate::{Order, VectorClock};

/// One stored version of a value, with the vector clock of the write that made it.
#[derive(Debug, Clone, PartialEq)]
pub struct Version {
    pub clock: VectorClock,
    pub value: String,
}

/// The versions that must be kept after seeing `a` and `b`: one if they are ordered, both if not.
pub fn resolve(a: Version, b: Version) -> Vec<Version> {
    match a.clock.compare(&b.clock) {
        Order::Before => vec![b],
        Order::After | Order::Equal => vec![a],
        Order::Concurrent => vec![a, b],
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn version(ticks: &[&str], value: &str) -> Version {
        let mut clock = VectorClock::default();
        for node in ticks {
            clock.tick(node);
        }
        Version {
            clock,
            value: value.to_string(),
        }
    }

    #[test]
    fn a_newer_version_replaces_an_older_one() {
        let old = version(&["r1"], "milk");
        let new = version(&["r1", "r1"], "milk, eggs");
        assert_eq!(resolve(old.clone(), new.clone()), vec![new.clone()]);
        assert_eq!(resolve(new.clone(), old), vec![new]);
    }

    #[test]
    fn concurrent_versions_are_both_kept() {
        let on_r1 = version(&["r1", "r1"], "milk, eggs"); // [r1:2]
        let on_r2 = version(&["r1", "r2"], "milk, bread"); // [r1:1 r2:1]
        let kept = resolve(on_r1.clone(), on_r2.clone());
        assert_eq!(kept, vec![on_r1, on_r2]);
    }
}
```

`cargo test --lib`:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: only a version that is *after* another one can safely replace it, because it has seen everything
the other one had. For concurrent versions nobody can know which change the user meant to keep, so the database
keeps both and the application merges them — here it would show "milk, eggs, bread".

</details>

## Quiz

1. Event a is a send, and event b is the receive of that message on another node. What do you know about their
   Lamport times?

<details>
<summary>Answer</summary>

L(a) < L(b), because the receiver sets its clock to at least L(a) + 1. In general: a → b implies L(a) < L(b).

</details>

2. L(x) = 4 and L(y) = 9. Did x happen before y?

<details>
<summary>Answer</summary>

Not necessarily. We only know that y did not happen before x. x and y may be concurrent.

</details>

3. Compare `[a:2 b:1]` with `[a:1 b:3]`, and `[a:1 b:3]` with `[a:2 b:3]`.

<details>
<summary>Answer</summary>

The first pair is concurrent (a: 2 > 1, but b: 1 < 3). In the second pair `[a:1 b:3]` is before `[a:2 b:3]` (one entry
smaller, none larger).

</details>

4. Why does a replicated database keep two siblings instead of choosing one?

<details>
<summary>Answer</summary>

Because their vector clocks are concurrent: neither write knew about the other, so there is no "newer" one. Choosing
one would silently lose the other user's change; keeping both lets the application merge them.

</details>

5. What is the main cost of vector clocks compared with Lamport clocks?

<details>
<summary>Answer</summary>

Size: a vector clock has one entry per node, and it must be sent with every message and stored with every version. A
Lamport clock is a single number.

</details>

## Summary

- Happens-before (a → b) means a could have influenced b: same node in order, or connected by messages. Events
  without such a path are concurrent.
- A Lamport clock is one counter per node: tick on every event, `max(own, remote) + 1` on receive. a → b implies
  L(a) < L(b), but not the other way round.
- `(L, node id)` pairs give a total order that respects causality.
- A vector clock keeps one counter per node and can tell before, after, equal and concurrent apart.
- Leaderless databases use vector clocks to keep concurrent versions as siblings instead of losing one.
- Vector clocks grow with the number of nodes; hybrid logical clocks are a common compromise.

## Sources and further reading

- [Time, Clocks, and the Ordering of Events in a Distributed System](https://lamport.azurewebsites.net/pubs/time-clocks.pdf) — Leslie Lamport's original 1978 paper; short and readable, with the happens-before relation and the clock rule.
- [Vector clock](https://en.wikipedia.org/wiki/Vector_clock) — an overview of vector clocks with the update and comparison rules and a space-time diagram.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — the chapters on replication and consistency show how version vectors detect concurrent writes in leaderless databases.
- [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — the paper that made vector clocks and siblings famous in real databases (the shopping cart example comes from here).
- [`proptest` documentation](https://docs.rs/proptest/latest/proptest/) — strategies and collection generators used in Exercise 2.

---

[🏠 Start](../START-HERE.md) · [← 18.4 Time and clocks](04-time-and-clocks.md) · [📋 Phase 18](README.md) · [18.6 Replication →](06-replication.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
