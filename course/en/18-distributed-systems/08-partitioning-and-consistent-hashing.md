<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.7 Consistency models](07-consistency-models.md) · [📋 Phase 18](README.md) · [18.9 Consensus and an overview of Raft →](09-consensus-and-raft.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/08-partitioning-and-consistent-hashing.md) <!-- nav -->

# Lesson 18.8 — Partitioning and consistent hashing

> **Phase 18 — Distributed systems** · Lesson 8 of 15 · ⏱ about 120 minutes

## What you will learn

- Why big systems split their data into partitions, and how partitioning works together with replication.
- The two basic ways to split data, by key range and by hash, and what a hot spot is.
- Why every node must use the same, stable hash function, and why `HashMap`'s hasher is the wrong one.
- Why "hash modulo the number of nodes" moves almost all data when a node is added, measured on 100,000 keys.
- How consistent hashing with virtual nodes moves only about 1/N of the keys, built as a ring on a `BTreeMap`.

## Before you start

- You finished [Lesson 18.7 — Consistency models](07-consistency-models.md) and [Lesson 18.6 — Replication](06-replication.md).
- You know how hash functions and hash tables work from [Lesson 07.6](../07-algorithms-and-data-structures/06-hash-tables.md), bitwise operations from [Lesson 12.4](../12-computer-systems/04-bitwise-operations.md), and `BTreeMap` from Phase 04.
- Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p18-partitioning
cd code/p18-partitioning
mkdir src/bin tests
```

## New words

| Word | Meaning |
|---|---|
| **partition** | One part of a data set, stored on one node (and its replicas). Also called a **shard**. |
| **partitioning** | Splitting the data into partitions so that each node stores and serves only part of it. Also called **sharding**. |
| **partition key** | The value that decides which partition a record belongs to, for example the user id. |
| **key-range partitioning** | Each partition owns a continuous range of keys, like the volumes of an encyclopedia (A–C, D–F, …). |
| **hash partitioning** | The partition is chosen by a hash of the key, which spreads keys evenly but destroys their order. |
| **hot spot** | A partition that gets much more load than the others. |
| **rebalancing** | Moving data between nodes when nodes are added or removed, or when the load is uneven. |
| **consistent hashing** | A way to assign keys to nodes on a ring of hash values, so that adding or removing a node moves only a small share of the keys. |
| **virtual node** | One of many points that one real node owns on the ring. More points give a more even share. |
| **FNV** (Fowler–Noll–Vo) | A simple, fast, non-cryptographic hash function named after its three inventors. |

## 1. Why split the data?

Replication (Lesson 18.6) gives every replica **all** the data. That does not help when the data itself is too big
for one machine, or when there are more writes than one leader can handle. Then we **partition**: split the data
so that each node is responsible for only a part.

Partitioning and replication are used together. Each partition is replicated, so a cluster of six machines may hold
three partitions with two replicas each:

```text
             partition 1 (users a–h)   partition 2 (users i–p)   partition 3 (users q–z)
machine 1        leader
machine 2        follower                  leader
machine 3                                  follower                  leader
machine 4                                                            follower
...
```

The goal of partitioning is to spread **data and load evenly**. If one partition gets most of the requests, the
other machines sit idle while one is overloaded. Two questions decide everything: which partition does a key belong
to, and what happens when we add a machine?

## 2. Two ways to split: by range or by hash

| | Key-range partitioning | Hash partitioning |
|---|---|---|
| Rule | partition 1 owns keys from "a" to "h", partition 2 from "i" to "p", … | partition = function of hash(key) |
| Range queries ("all orders of May") | fast: the keys sit next to each other in one partition | slow: the keys are spread over all partitions |
| Even spread | only if the boundaries are chosen well, and they must move as data grows | automatic, if the hash is good |
| Typical danger | **hot spots**: if the key is a timestamp, all new writes go to the "today" partition | a single very popular key still lands on one partition |

A hot spot can appear with any method. If a celebrity with millions of followers is one key, every request for that
key goes to one partition whatever the partitioning scheme. Applications then split such keys by hand, for example by
adding a small random number to the key and reading all variants.

This lesson builds hash partitioning, because it is what most key-value stores use, including the one you will
replicate in Lesson 18.14.

## 3. A hash that every node agrees on

All nodes, clients and routers must compute the **same** partition for the same key, today and after an upgrade.
So the hash function must be fully defined by its algorithm. We use **FNV-1a**, which takes three lines, and add a
**mixing** step, because FNV alone gives similar values for similar keys such as `user:1` and `user:2`. Put this into
`src/lib.rs`:

```rust
//! Partitioning and consistent hashing (Lesson 18.8).

/// The 64-bit FNV-1a hash. Its result is fixed by its definition, so every node and every
/// version of the program computes the same number for the same key.
pub fn fnv1a(bytes: &[u8]) -> u64 {
    let mut hash: u64 = 0xcbf2_9ce4_8422_2325;
    for &byte in bytes {
        hash ^= u64::from(byte);
        hash = hash.wrapping_mul(0x0100_0000_01b3);
    }
    hash
}

/// Mixes the bits of `x` (the "finalizer" of MurmurHash3), so that similar keys such as
/// "user:1" and "user:2" end up far apart.
pub fn mix(mut x: u64) -> u64 {
    x ^= x >> 33;
    x = x.wrapping_mul(0xff51_afd7_ed55_8ccd);
    x ^= x >> 33;
    x = x.wrapping_mul(0xc4ce_b9fe_1a85_ec53);
    x ^= x >> 33;
    x
}

/// The hash used for partitioning in this lesson.
pub fn key_hash(key: &str) -> u64 {
    mix(fnv1a(key.as_bytes()))
}

/// The simple way: node number = hash modulo the number of nodes.
pub fn mod_n(key: &str, nodes: u64) -> u64 {
    key_hash(key) % nodes
}
```

**What happens:**

1. FNV-1a starts from a fixed number (the "offset basis"), and for each byte it XORs (exclusive-or, `^`) the byte in and
   multiplies by a fixed prime. `wrapping_mul` lets the multiplication overflow on purpose, keeping the lowest 64 bits
   (Lesson 12.2).
2. `mix` shifts and multiplies again, so that a change in one input bit changes about half of the output bits. The
   constants come from the well-known MurmurHash3 hash; you do not need to understand why exactly these numbers work,
   only that they are fixed.
3. `mod_n` is the obvious way to pick one of `nodes` partitions: the remainder of the hash divided by the number of
   nodes.

> ⚠️ **Common mistake: using `HashMap`'s hasher for partitioning.** `HashMap` uses `RandomState`, which picks new random
> keys in **every process**, to protect against *hash flooding* (an attacker sending many keys with the same
> hash to slow a table down). Two nodes would disagree about every key.
> Here is the proof, `src/bin/random_state.rs`:

```rust
use std::collections::hash_map::RandomState;
use std::hash::BuildHasher;

fn main() {
    // The hasher that HashMap uses by default: its keys are chosen at random in every process.
    let hasher = RandomState::new();
    let partition = hasher.hash_one("user:1") % 4;
    println!("user:1 belongs to partition {partition}");
}
```

Five runs of `cargo run --bin random_state` (yours will differ):

```text
user:1 belongs to partition 3
user:1 belongs to partition 2
user:1 belongs to partition 2
user:1 belongs to partition 3
user:1 belongs to partition 2
```

`std::hash::DefaultHasher::new()` is better (its keys are fixed), but its documentation says that the algorithm may
change between Rust releases, so two nodes built with different compiler versions could still disagree.

## 4. Hash modulo N, and what happens when N changes

`hash % nodes` spreads keys evenly. The trouble starts when the number of nodes changes. Let us count how many of
100,000 keys move to another node when a fifth node joins four, `src/bin/mod_n.rs`:

```rust
use p18_partitioning::mod_n;

fn main() {
    let keys: Vec<String> = (0..100_000).map(|i| format!("user:{i}")).collect();
    let moved = keys
        .iter()
        .filter(|key| mod_n(key, 4) != mod_n(key, 5))
        .count();
    println!(
        "hash mod N, from 4 to 5 nodes: {moved} of {} keys move",
        keys.len()
    );
}
```

`cargo run --bin mod_n`:

```text
hash mod N, from 4 to 5 nodes: 80091 of 100000 keys move
```

**80 %** of the data must be copied over the network, although the new node should only take 20 % (its fair share
of five). The reason: `h % 4` and `h % 5` agree only when the remainders happen to match, which is the case for about
one key in five. During the move, a cluster that wanted **more** capacity is busy copying almost everything.

## 5. Consistent hashing: a ring

Consistent hashing puts both **nodes** and **keys** on the same circle of hash values, from 0 to 2⁶⁴ − 1, which wraps
around. A key belongs to the **first node clockwise** from its position:

```text
                 0 / 2^64
                    │
          n3 ●──────┼──────● n1        key k1 → walk clockwise → n1
            ╱       │       ╲          key k2 → walk clockwise → n2
           │    k1 ×│        │
  k3 ×     │        │        │         when n4 is added between n1 and n2,
           │        │   × k2 │         only keys between n1 and n4 change owner
            ╲       │       ╱          (they move from n2 to n4); nobody else moves
          n2 ●──────┼──────●
                    │     n4 (new)
```

When a node joins, it takes over only the keys between its position and the previous node: the other keys keep
their owner. With one point per node, though, the arcs between points have very different lengths, so some nodes own
much more than others. The fix is **virtual nodes**: every real node is placed on the ring many times, at the
positions `hash("n1#0")`, `hash("n1#1")`, and so on. Many small arcs average out.

A `BTreeMap<u64, String>` from ring position to node name is a perfect ring: it keeps the positions sorted, and
`range(hash..)` finds the first position at or after the key. Add `use std::collections::BTreeMap;` at the top of
`src/lib.rs`, and then:

```rust
/// A consistent-hashing ring. Each node owns several points on a circle of u64 values.
#[derive(Default)]
pub struct Ring {
    points: BTreeMap<u64, String>,
    vnodes: u32,
}

impl Ring {
    pub fn new(vnodes: u32) -> Self {
        Ring {
            points: BTreeMap::new(),
            vnodes,
        }
    }

    pub fn add(&mut self, node: &str) {
        for i in 0..self.vnodes {
            let point = key_hash(&format!("{node}#{i}"));
            self.points.insert(point, node.to_string());
        }
    }

    pub fn remove(&mut self, node: &str) {
        self.points.retain(|_, owner| owner != node);
    }

    /// The owner of `key`: the first point at or after the key's hash, going around the circle.
    pub fn node_for(&self, key: &str) -> Option<&str> {
        let hash = key_hash(key);
        self.points
            .range(hash..)
            .next()
            .or_else(|| self.points.iter().next())
            .map(|(_, node)| node.as_str())
    }
}
```

**What happens:**

1. `add` inserts `vnodes` points for the node. `format!("{node}#{i}")` makes a different name for each point, so their
   hashes land all over the ring.
2. `remove` deletes all points of a node with `retain`, which keeps only the entries for which the closure says `true`.
3. `node_for` hashes the key and asks the map for the first point at or after it. If there is none (the key is after
   the last point), it wraps around to the first point of the circle. That is the "clockwise" walk in two lines.

Now compare one point per node with 100 points per node, `src/bin/ring.rs`:

```rust
use std::collections::BTreeMap;

use p18_partitioning::Ring;

/// How many of the keys each node owns.
fn load(ring: &Ring, keys: &[String]) -> BTreeMap<String, usize> {
    let mut load = BTreeMap::new();
    for key in keys {
        *load
            .entry(ring.node_for(key).unwrap().to_string())
            .or_insert(0) += 1;
    }
    load
}

fn main() {
    let keys: Vec<String> = (0..100_000).map(|i| format!("user:{i}")).collect();
    for vnodes in [1, 100] {
        let mut ring = Ring::new(vnodes);
        for node in ["n1", "n2", "n3", "n4"] {
            ring.add(node);
        }
        let owners: Vec<String> = keys
            .iter()
            .map(|k| ring.node_for(k).unwrap().to_string())
            .collect();
        println!(
            "{vnodes} point(s) per node, 4 nodes: {:?}",
            load(&ring, &keys)
        );

        ring.add("n5");
        let moved = keys
            .iter()
            .zip(&owners)
            .filter(|(key, old)| ring.node_for(key).unwrap() != old.as_str())
            .count();
        println!(
            "{vnodes} point(s) per node, 5 nodes: {:?}",
            load(&ring, &keys)
        );
        println!("keys moved: {moved}\n");
    }
}
```

`cargo run --bin ring`:

```text
1 point(s) per node, 4 nodes: {"n1": 62520, "n2": 4163, "n3": 9375, "n4": 23942}
1 point(s) per node, 5 nodes: {"n1": 3185, "n2": 4163, "n3": 9375, "n4": 23942, "n5": 59335}
keys moved: 59335

100 point(s) per node, 4 nodes: {"n1": 22608, "n2": 22804, "n3": 26474, "n4": 28114}
100 point(s) per node, 5 nodes: {"n1": 18901, "n2": 17019, "n3": 20875, "n4": 23017, "n5": 20188}
keys moved: 20188
```

**What happens:**

1. With one point per node, the ring is very uneven: `n1` owns 62 % of the keys and `n2` only 4 %. When `n5` joins,
   its single point happens to land inside `n1`'s huge arc, so it takes 59 % of all keys from `n1` alone. Correct, but
   badly balanced.
2. With 100 points per node, each node owns between 22 % and 28 %, and the new node takes **20,188** keys, almost
   exactly its fair fifth, a little from every old node.
3. Compare with Section 4: hash mod N moved 80,091 keys for the same change.

> 🧪 **Testing:** The most important property of consistent hashing is easy to test: when a node is added, every key
> either stays where it was or moves to the **new** node, never between two old nodes. Exercise 2 writes this test.

> ⚡ **Performance:** `node_for` is one hash and one `BTreeMap` lookup, O(log P) for P points on the ring: with
> 5 nodes × 100 points that is about 9 comparisons. Routing is cheap even for thousands of virtual nodes.

## 6. Routing and rebalancing

Two practical questions remain.

**Who knows the mapping?** A client that wants key `user:42` must find its node. Three common designs:

| Design | How it works |
|---|---|
| Any node can answer | the client asks any node; that node forwards the request or tells the client where to go |
| A routing layer | a proxy (Lesson 15.14) knows the mapping and forwards every request |
| A smart client | the client library knows the mapping itself and talks to the right node directly |

In all three, the mapping itself must be stored somewhere reliable, and all parties must agree on it. Many systems
keep it in a small coordination service built on consensus (Lessons 18.9–18.11), such as etcd.

**How is data moved?** When the ring changes, the new owner copies the affected keys from the old owner in the
background while the old owner keeps serving them, and the mapping switches only when the copy is complete. Moving
data costs network bandwidth and disk time, which is exactly why moving 20 % instead of 80 % matters.

> 🔒 **Security:** If clients can choose keys freely, an attacker can try to create many keys that land on one
> partition and overload it. A keyed hash (with a secret, like `RandomState`) prevents that — but then every node
> needs the same secret. Systems that face untrusted keys usually hash with a fixed, secret, cluster-wide key.

## Common mistakes

> ⚠️ **Common mistake: partitioning by a timestamp.** With key-range partitioning on "created at", all new writes go
> to the newest partition: a permanent hot spot. Put another value first in the key (for example, the user id) or use
> hash partitioning.

> ⚠️ **Common mistake: too few virtual nodes.** With one point per node, Section 5 showed a 62 % / 4 % split. Use
> dozens to hundreds of points per node.

## Try it yourself

1. In `ring.rs`, add `10` to the list of point counts. Where between 1 and 100 does the balance become acceptable?
2. After adding `n5`, call `ring.remove("n1")` and print the load again. Where did `n1`'s keys go?
3. In `mod_n.rs`, count the moved keys when going from 4 to 8 nodes instead of 5. Why is it about half?

## Exercises

### Exercise 1 — Check the hash against its definition (easy)

FNV-1a has published test values: the empty input gives `0xcbf29ce484222325`, `"a"` gives `0xaf63dc4c8601ec8c`,
and `"foobar"` gives `0x85944171f73967e8`. Write a unit test in `src/lib.rs` that checks all three. Why does such a
test matter more in a distributed system than in a single program?

<details>
<summary>Hint</summary>

Byte string literals such as `b"foobar"` have the type `&[u8; 6]`, which turns into `&[u8]` automatically. Underscores
in number literals (`0xcbf2_9ce4_...`) are only for readability.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fnv1a_matches_the_published_test_values() {
        assert_eq!(fnv1a(b""), 0xcbf2_9ce4_8422_2325);
        assert_eq!(fnv1a(b"a"), 0xaf63_dc4c_8601_ec8c);
        assert_eq!(fnv1a(b"foobar"), 0x8594_4171_f739_67e8);
    }
}
```

`cargo test --lib`:

```text
test tests::fnv1a_matches_the_published_test_values ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the test pins the hash to its definition. If somebody "improves" the function, the test fails before
the change reaches production, where nodes running the old and the new version would send the same key to different
partitions.

</details>

### Exercise 2 — Nothing moves between old nodes (medium)

Write an integration test `tests/ring_moves.rs`: build a ring of four nodes with 100 points each, record the owner of
10,000 keys, add `n5`, and check that every key either kept its owner or moved to `n5`. Then remove `n5` and check that
every key is back with its original owner.

<details>
<summary>Hint</summary>

Store the owners as `Vec<String>` (owned strings), because the ring is changed afterwards and the borrowed `&str`
results would block that. `keys.iter().zip(&before)` walks over keys and old owners together.

</details>

<details>
<summary>Answer</summary>

```rust
use p18_partitioning::Ring;

#[test]
fn a_new_node_only_takes_keys_and_never_shuffles_the_others() {
    let mut ring = Ring::new(100);
    for node in ["n1", "n2", "n3", "n4"] {
        ring.add(node);
    }
    let keys: Vec<String> = (0..10_000).map(|i| format!("user:{i}")).collect();
    let before: Vec<String> = keys
        .iter()
        .map(|k| ring.node_for(k).unwrap().to_string())
        .collect();

    ring.add("n5");
    for (key, old) in keys.iter().zip(&before) {
        let new = ring.node_for(key).unwrap();
        assert!(new == old || new == "n5", "{key} moved from {old} to {new}");
    }

    ring.remove("n5");
    let back: Vec<String> = keys
        .iter()
        .map(|k| ring.node_for(k).unwrap().to_string())
        .collect();
    assert_eq!(back, before); // removing the node gives every key back to its old owner
}
```

`cargo test --test ring_moves`:

```text
test a_new_node_only_takes_keys_and_never_shuffles_the_others ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

Why it works: adding points to the ring can only shorten the arcs of existing points; a key whose walk now stops
earlier stops at a point of the new node. Removing those points restores the old arcs exactly.

</details>

### Exercise 3 — Rendezvous hashing (hard)

Rendezvous hashing (also called *highest random weight*) needs no ring at all: for a key, compute a score
`hash(node + key)` for every node, and the node with the highest score owns the key. Create `src/rendezvous.rs` (and
`pub mod rendezvous;`) with `pub fn owner<'a>(key: &str, nodes: &[&'a str]) -> Option<&'a str>`. Test that when a fifth
node is added to four, every key that moves goes to the new node, and between 15 % and 25 % of 10,000 keys move.

<details>
<summary>Hint</summary>

`nodes.iter().copied().max_by_key(|node| key_hash(&format!("{node}/{key}")))`. The lifetime `'a` says that the
returned name lives as long as the node names in the slice, not as long as the key.

</details>

<details>
<summary>Answer</summary>

```rust
//! Exercise 3: rendezvous hashing.

use crate::key_hash;

/// Rendezvous ("highest random weight") hashing: every node gets a score for the key,
/// and the node with the highest score owns it.
pub fn owner<'a>(key: &str, nodes: &[&'a str]) -> Option<&'a str> {
    nodes
        .iter()
        .copied()
        .max_by_key(|node| key_hash(&format!("{node}/{key}")))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_new_node_takes_about_a_fifth_and_only_for_itself() {
        let keys: Vec<String> = (0..10_000).map(|i| format!("user:{i}")).collect();
        let four = ["n1", "n2", "n3", "n4"];
        let five = ["n1", "n2", "n3", "n4", "n5"];
        let mut moved = 0;
        for key in &keys {
            let before = owner(key, &four).unwrap();
            let after = owner(key, &five).unwrap();
            if before != after {
                assert_eq!(after, "n5", "{key} moved between old nodes");
                moved += 1;
            }
        }
        assert!((1500..2500).contains(&moved), "moved {moved}");
    }
}
```

`cargo test --lib`:

```text
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Why it works: adding a node changes nothing in the scores of the old nodes, so a key changes its owner only if the new
node's score beats the old maximum, which happens for about one key in five. The price: finding the owner costs one
hash per node, O(N), instead of a ring lookup — fine for tens of nodes, slow for thousands.

</details>

## Quiz

1. Why are partitioning and replication usually used together?

<details>
<summary>Answer</summary>

Partitioning spreads data and load over machines; replication keeps each partition available and durable when a
machine fails. Each partition has its own replicas (and often its own leader).

</details>

2. You partition orders by key range on the order date. What problem do you expect?

<details>
<summary>Answer</summary>

A hot spot: all new orders have today's date, so all writes go to one partition while the others are idle.

</details>

3. Why must the partitioning hash not use `HashMap`'s default hasher?

<details>
<summary>Answer</summary>

`RandomState` chooses random keys in every process, so every node would compute a different partition for the same key.
The hash must be fixed by its algorithm and the same on every node and every version.

</details>

4. A cluster grows from 10 to 11 nodes. Roughly how many keys move with hash mod N, and how many with consistent
   hashing?

<details>
<summary>Answer</summary>

With mod N almost all of them: only about 1 key in 11 keeps its node, so about 91 % move. With consistent hashing (and
enough virtual nodes) about 1/11, roughly 9 %, all of them to the new node.

</details>

5. What do virtual nodes fix?

<details>
<summary>Answer</summary>

The uneven arcs of a ring with one point per node. With many points per node, each node's share averages out close to
1/N, and a new node takes a little from every old node instead of a lot from one.

</details>

## Summary

- Partitioning (sharding) splits data so that each node stores and serves only part of it; each partition is also
  replicated.
- Key-range partitioning keeps ranges together but risks hot spots; hash partitioning spreads keys evenly but loses
  their order.
- The partitioning hash must be stable and identical everywhere: a fixed algorithm such as FNV-1a with a mixing step,
  never `RandomState`.
- `hash % N` moves most keys when N changes: 80 % from 4 to 5 nodes.
- Consistent hashing on a ring moves only about 1/N of the keys, all to the new node; virtual nodes make the shares even.
- Somebody must store and agree on the mapping (often a consensus-based service), and data moves in the background.

## Sources and further reading

- [Consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing) — the history of the technique and how virtual nodes work.
- [Rendezvous hashing](https://en.wikipedia.org/wiki/Rendezvous_hashing) — the "highest random weight" alternative from Exercise 3.
- [Fowler–Noll–Vo hash function](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function) — the definition of FNV-1a and its constants.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — the chapter on partitioning (sharding) covers key ranges, hashing, hot spots, rebalancing and request routing.
- [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — a real system that combines a consistent-hashing ring, virtual nodes and replication.
- [`std::hash::DefaultHasher`](https://doc.rust-lang.org/std/hash/struct.DefaultHasher.html) — the documentation's warning that its algorithm may change between releases.

---

[🏠 Start](../START-HERE.md) · [← 18.7 Consistency models](07-consistency-models.md) · [📋 Phase 18](README.md) · [18.9 Consensus and an overview of Raft →](09-consensus-and-raft.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
