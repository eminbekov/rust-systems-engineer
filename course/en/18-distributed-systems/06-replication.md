<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.5 Logical clocks: Lamport and vector clocks](05-logical-clocks.md) · [📋 Phase 18](README.md) · [18.7 Consistency models →](07-consistency-models.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/06-replication.md) <!-- nav -->

# Lesson 18.6 — Replication

> **Phase 18 — Distributed systems** · Lesson 6 of 15 · ⏱ about 135 minutes

## What you will learn

- Why systems keep several copies of the same data, and what each copy costs.
- How single-leader replication works: one leader, a replication log and followers that apply it in order.
- What replication lag is, and why a follower can return old data.
- Why asynchronous replication can lose writes that were already confirmed, and what synchronous replication costs.
- How leaderless systems use quorums, and why `R + W > N` makes every read meet the latest write.

## Before you start

- You finished [Lesson 18.5 — Logical clocks](05-logical-clocks.md).
- You know the write-ahead log from [Lesson 17.4](../17-storage-and-databases/04-write-ahead-log.md): replication ships a log of changes in the same way.
- Create the package for this lesson. It needs no dependencies; the whole "cluster" is simulated in memory, so that
  every failure happens exactly when we want it:

```bash
cd ~/projects/rust
cargo new --lib code/p18-replication
cd code/p18-replication
mkdir src/bin tests
```

## New words

| Word | Meaning |
|---|---|
| **replica** | One copy of the data, usually on its own machine. |
| **replication** | Keeping several replicas of the same data up to date. |
| **leader** | The one replica that accepts writes and decides their order. Also called *primary*. |
| **follower** | A replica that only copies the leader's changes and may serve reads. Also called *replica*, *secondary* or *standby*. |
| **replication log** | The ordered list of changes that the leader sends to its followers. |
| **replication lag** | How far a follower is behind the leader: how many changes it has not applied yet. |
| **synchronous / asynchronous replication** | The leader confirms a write to the client only after followers have it (synchronous), or at once (asynchronous). |
| **failover** | Making a follower the new leader when the old leader has failed. |
| **split brain** | Two nodes both believe they are the leader and both accept writes. |
| **leaderless replication** | Clients write to and read from several replicas directly; no replica is special. |
| **quorum** | The minimum number of replicas that must answer for an operation to count. **N** is the number of replicas, **W** the write quorum, **R** the read quorum. |
| **read repair** | When a read finds a replica with an old value, it writes the newest value back to it. |

## 1. Why keep copies?

Replication means keeping the same data on several machines. There are four reasons:

| Reason | Example |
|---|---|
| **Availability**: keep working when a machine fails | The leader dies; a follower takes over within seconds. |
| **Durability**: do not lose data when a disk dies | Every row exists on three disks in three racks. |
| **Read scaling**: more machines answer reads | Ten followers serve the product catalogue; one leader takes the few writes. |
| **Latency**: a copy close to the user | A replica in each region answers reads locally. |

If the data never changed, replication would be trivial: copy it once. All the difficulty comes from **changes**.
Every write must reach every replica, the network can lose or delay it (Lesson 18.2), and replicas can crash in the
middle. The rest of this lesson is about the different ways to handle that.

## 2. Single-leader replication

The most common design, used for example by PostgreSQL and by the Raft algorithm that you will build later:

```text
           writes                   replication log
client ──────────────► leader  [1: x=1] [2: x=2] [3: y=5] ...
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
       follower 0                 follower 1
   applied 3 of 3 entries     applied 2 of 3 entries   ← behind
             ▲                         ▲
             └──── reads may go to followers ──┘
```

1. Only the **leader** accepts writes. It puts each change into its **replication log**, in order. The log is very
   much like the write-ahead log of Lesson 17.4.
2. **Followers** receive the log entries and apply them **in the same order**. Because they start from the same
   state and apply the same changes in the same order, they end up with the same data.
3. Reads can go to the leader (always up to date) or to followers (cheaper, but possibly behind).

Let us build this in memory. Put this into `src/lib.rs`:

```rust
//! Replication (Lesson 18.6).

use std::collections::HashMap;

/// One change in the replication log.
#[derive(Debug, Clone, PartialEq)]
pub struct Entry {
    pub key: String,
    pub value: String,
}

/// One copy of the data. It applies log entries in order and counts them.
#[derive(Debug, Default)]
pub struct Replica {
    data: HashMap<String, String>,
    applied: usize,
}

impl Replica {
    pub fn apply(&mut self, entries: &[Entry]) {
        for entry in entries {
            self.data.insert(entry.key.clone(), entry.value.clone());
        }
        self.applied += entries.len();
    }

    pub fn get(&self, key: &str) -> Option<&str> {
        self.data.get(key).map(String::as_str)
    }

    /// How many log entries this replica has applied.
    pub fn applied(&self) -> usize {
        self.applied
    }
}

/// One leader and its followers, with asynchronous replication.
#[derive(Default)]
pub struct Cluster {
    pub log: Vec<Entry>,
    pub leader: Replica,
    pub followers: Vec<Replica>,
}

impl Cluster {
    pub fn new(followers: usize) -> Self {
        let mut cluster = Cluster::default();
        cluster.followers.resize_with(followers, Replica::default);
        cluster
    }

    /// A client write. The leader appends it to the log, applies it and answers "OK"
    /// at once, without waiting for any follower. Returns the entry's position in the log.
    pub fn write(&mut self, key: &str, value: &str) -> usize {
        let entry = Entry {
            key: key.to_string(),
            value: value.to_string(),
        };
        self.leader.apply(std::slice::from_ref(&entry));
        self.log.push(entry);
        self.log.len()
    }

    /// Sends follower `i` every log entry that it has not applied yet.
    pub fn replicate(&mut self, i: usize) {
        let follower = &mut self.followers[i];
        follower.apply(&self.log[follower.applied..]);
    }

    /// The leader has crashed: follower `i` becomes the new leader with the data it has.
    /// Returns the entries that the old leader had confirmed but that are now lost.
    pub fn fail_over(&mut self, i: usize) -> Vec<Entry> {
        self.leader = self.followers.remove(i);
        self.log.split_off(self.leader.applied)
    }
}
```

**What happens:**

1. A `Replica` is a key-value map plus the number of log entries it has applied. Since entries are applied in order
   and never skipped, that one number says exactly which entries a replica has.
2. `Cluster::write` is the leader's side of a write: append to the log, apply, and return the position at once. It
   does not wait for any follower: this is **asynchronous** replication.
3. `replicate(i)` is one "network delivery" to follower `i`: it ships the entries from `applied` to the end of the
   log. In a real system this happens all the time in the background; here **we** decide when, so we can create
   any delay we like.
4. `fail_over(i)` imitates a crashed leader: follower `i` becomes the leader, and the log is cut back to what the new
   leader has. `split_off` returns the entries that were cut off: the writes that the old leader had confirmed but
   nobody else had.

## 3. Replication lag and lost writes

First, lag. `src/bin/lag.rs` writes `x = 1`, replicates it everywhere, then writes `x = 2` and replicates it to
follower 0 only:

```rust
use p18_replication::Cluster;

fn main() {
    let mut cluster = Cluster::new(2);
    cluster.write("x", "1");
    cluster.replicate(0);
    cluster.replicate(1);

    cluster.write("x", "2");
    cluster.replicate(0); // follower 1 does not get the new entry yet

    println!("leader:     x = {:?}", cluster.leader.get("x"));
    for (i, follower) in cluster.followers.iter().enumerate() {
        let lag = cluster.log.len() - follower.applied();
        println!("follower {i}: x = {:?} (lag: {lag})", follower.get("x"));
    }
}
```

`cargo run --bin lag`:

```text
leader:     x = Some("2")
follower 0: x = Some("2") (lag: 0)
follower 1: x = Some("1") (lag: 1)
```

A client that reads from follower 1 sees the **old** value. In a healthy system the lag is milliseconds, but under
load or during a network problem it can be seconds or minutes. A user who changes their name and then reloads the
page from a lagging follower sees the old name, and thinks the change was lost. Exercise 3 fixes exactly this.

Second, failover. `src/bin/failover.rs` confirms a write to a client and then crashes the leader before the entry is
shipped:

```rust
use p18_replication::Cluster;

fn main() {
    let mut cluster = Cluster::new(2);
    cluster.write("balance", "100");
    cluster.replicate(0);
    cluster.replicate(1);

    let position = cluster.write("balance", "150");
    println!("client: write number {position} (balance = 150) confirmed");

    // The leader crashes before it sends entry 2 to anybody.
    let lost = cluster.fail_over(0);
    println!("new leader: balance = {:?}", cluster.leader.get("balance"));
    println!("lost, although confirmed: {lost:?}");
}
```

`cargo run --bin failover`:

```text
client: write number 2 (balance = 150) confirmed
new leader: balance = Some("100")
lost, although confirmed: [Entry { key: "balance", value: "150" }]
```

The client was told "OK". The write is gone. This is the price of asynchronous replication, and real databases with
asynchronous followers have exactly this behaviour on failover: the last few confirmed writes may be lost.

The alternatives:

| Mode | The leader answers "OK" when… | Cost |
|---|---|---|
| asynchronous | it has written the change itself | confirmed writes can be lost on failover |
| synchronous | **every** follower has the change | one slow or dead follower stops all writes |
| semi-synchronous | **one** (or a few) followers have it | a little slower; survives the loss of the leader |

Many real setups use the middle way: one synchronous follower, the others asynchronous. Raft (Lesson 18.11) uses a
**majority**: a write is confirmed when more than half of the nodes have it, so it survives the loss of any minority.

> ⚠️ **Common mistake: two leaders.** Failover is decided by other nodes, usually because the old leader stopped
> answering heartbeats (Lesson 18.2). But the old leader may only be slow or cut off. If it comes back and keeps
> accepting writes, you have **split brain**: two leaders, two diverging histories. The cure is the one from
> Lesson 18.4: a number that grows with each new leader (a fencing token; Raft calls it the *term*), checked on
> every write.

> 🧪 **Testing:** Because the simulation is deterministic, both stories make good tests. Add to the end of
> `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_follower_that_is_behind_returns_old_data() {
        let mut cluster = Cluster::new(2);
        cluster.write("x", "1");
        cluster.replicate(0);
        cluster.replicate(1);
        cluster.write("x", "2");
        cluster.replicate(0);
        assert_eq!(cluster.followers[0].get("x"), Some("2"));
        assert_eq!(cluster.followers[1].get("x"), Some("1")); // stale
    }

    #[test]
    fn asynchronous_failover_loses_confirmed_writes() {
        let mut cluster = Cluster::new(1);
        cluster.write("x", "1");
        cluster.replicate(0);
        cluster.write("x", "2"); // confirmed to the client, never replicated
        let lost = cluster.fail_over(0);
        assert_eq!(cluster.leader.get("x"), Some("1"));
        assert_eq!(lost.len(), 1);
    }
}
```

## 4. Leaderless replication and quorums

Some databases (in the style of Amazon's Dynamo) have **no leader** at all. With N replicas, a client sends each
write to all N and treats it as successful when **W** of them confirm. It sends each read to all N and waits for
**R** answers, then takes the value with the newest version.

Why does that work? If every write reached at least W replicas and every read asks R replicas, and `R + W > N`, then
the two groups **must** share at least one replica: there are not enough replicas for them to miss each other. That
shared replica has the newest value.

```text
N = 3, W = 2, R = 2:       replica 1   replica 2   replica 3
    write reached            new         new         old
    read asked                           new         old    → sees "new" through replica 2
```

You do not have to believe this rule: we can check it for small N by trying **every** possible write group and read
group. A group of replicas is a set; we write it as a number whose bits say which replicas are in it (bit 0 =
replica 1, and so on, as in Lesson 12.4). Two groups share a replica if their bitwise AND is not zero. Add to
`src/lib.rs` (before the tests):

```rust
/// Checks every possible write set of size `w` and read set of size `r` among `n` replicas:
/// true if each pair shares at least one replica.
pub fn quorums_always_overlap(n: u32, w: u32, r: u32) -> bool {
    let sets_of = |size: u32| (0u32..1 << n).filter(move |set| set.count_ones() == size);
    sets_of(w).all(|write_set| sets_of(r).all(|read_set| write_set & read_set != 0))
}
```

**What happens:**

1. `0u32..1 << n` are all numbers from 0 to 2ⁿ − 1, that is, all subsets of n replicas.
2. `set.count_ones() == size` keeps only the subsets with the right number of members.
3. `all(... all(...))` checks every pair of a write group and a read group. `move` copies `size` into the inner
   closure, so that `sets_of` can be called twice.

And a table, `src/bin/quorums.rs`:

```rust
use p18_replication::quorums_always_overlap;

fn main() {
    let n = 3;
    for (w, r) in [(1, 1), (2, 1), (2, 2), (3, 1), (1, 3)] {
        let overlap = quorums_always_overlap(n, w, r);
        println!(
            "N={n} W={w} R={r}: R+W > N is {:5}  sets always overlap: {overlap}",
            r + w > n
        );
    }
}
```

`cargo run --bin quorums`:

```text
N=3 W=1 R=1: R+W > N is false  sets always overlap: false
N=3 W=2 R=1: R+W > N is false  sets always overlap: false
N=3 W=2 R=2: R+W > N is true   sets always overlap: true
N=3 W=3 R=1: R+W > N is true   sets always overlap: true
N=3 W=1 R=3: R+W > N is true   sets always overlap: true
```

The brute-force check agrees with the formula in every row. Turn it into a test for N = 1 to 5 and all W and R; add
it inside `mod tests`:

```rust
    #[test]
    fn overlapping_quorums_follow_r_plus_w_greater_than_n() {
        for n in 1..=5 {
            for w in 1..=n {
                for r in 1..=n {
                    assert_eq!(
                        quorums_always_overlap(n, w, r),
                        r + w > n,
                        "n={n} w={w} r={r}"
                    );
                }
            }
        }
    }
```

`cargo test --lib`:

```text
test tests::asynchronous_failover_loses_confirmed_writes ... ok
test tests::a_follower_that_is_behind_returns_old_data ... ok
test tests::overlapping_quorums_follow_r_plus_w_greater_than_n ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Common choices are N = 3 with W = 2 and R = 2 (survive one failed replica for both reads and writes), or W = N and
R = 1 (fast reads, but one dead replica blocks writes). Replicas that missed a write are fixed in two ways: **read
repair** (a read that sees an old version writes the new one back; Exercise 2) and a background process that compares
replicas.

> ⚡ **Performance:** A quorum operation waits for the W-th or R-th fastest answer, not for the slowest. With N = 3 and
> R = 2, one slow replica does not slow down reads at all. This is one of the main reasons for leaderless designs.

Quorums are not magic, though. Two concurrent writes can reach different replicas in different orders (the version
numbers need vector clocks from Lesson 18.5 to detect that), and a write that failed on some replicas may still be
present on others. Lesson 18.7 shows what guarantees such a system can and cannot give.

| | Single leader | Multi-leader | Leaderless |
|---|---|---|---|
| Who accepts writes | one leader | one leader per region or device | any replica |
| Order of writes | decided by the leader | conflicts must be merged | conflicts must be detected and merged |
| When the leader fails | failover needed | other leaders continue | nothing special |
| Typical use | most relational databases, Raft | several data centres, offline apps | very high availability stores |

## Common mistakes

> ⚠️ **Common mistake: reading your own write from a follower.** After a write, a read from a lagging follower may not
> see it. Read from the leader for a while after a write, or use the position of the write as in Exercise 3.

## Try it yourself

1. In `failover.rs`, call `cluster.replicate(0);` right after the second write, then run it. Nothing is lost,
   because the new leader had the entry.
2. Replicate to follower 1 instead and fail over to follower 0. The write is lost again: what matters is whether the
   **new leader** had it.

## Exercises

### Exercise 1 — Synchronous writes (easy)

Add `pub fn write_sync(&mut self, key: &str, value: &str) -> usize` to `Cluster`: like `write`, but it replicates to
every follower before it returns. Write an integration test `tests/sync_and_sessions.rs` that makes two synchronous
writes, fails over to follower 0, and checks that nothing is lost and the new leader has the second value.

<details>
<summary>Hint</summary>

`write_sync` can call `self.write(...)` and then `self.replicate(i)` for every `i` in `0..self.followers.len()`.

</details>

<details>
<summary>Answer</summary>

In `impl Cluster`:

```rust
    /// A write that is confirmed only after every follower has applied it.
    pub fn write_sync(&mut self, key: &str, value: &str) -> usize {
        let position = self.write(key, value);
        for i in 0..self.followers.len() {
            self.replicate(i);
        }
        position
    }
```

In `tests/sync_and_sessions.rs`:

```rust
use p18_replication::Cluster;

#[test]
fn synchronous_writes_survive_a_failover() {
    let mut cluster = Cluster::new(2);
    cluster.write_sync("balance", "100");
    cluster.write_sync("balance", "150");
    let lost = cluster.fail_over(0);
    assert!(lost.is_empty());
    assert_eq!(cluster.leader.get("balance"), Some("150"));
}
```

Why it works: every follower has every confirmed entry before the client hears "OK", so whichever follower becomes the
leader has all of them. The price: if one follower were down, `write_sync` would have to wait for it, and in a real
system writes would stop.

</details>

### Exercise 2 — Read repair (medium)

Create `src/leaderless.rs` (and `pub mod leaderless;`) with a `VersionedReplica` that stores `(version, value)` per
key and keeps only the higher version on `put`, and a function
`pub fn quorum_read(replicas: &mut [VersionedReplica], key: &str, ask: &[usize]) -> Option<String>` that reads from
the replicas listed in `ask`, returns the newest value, and writes it back to the asked replicas that had an older one.
Test: three replicas have version 1; version 2 reaches replicas 0 and 1 (W = 2); a read from replicas 1 and 2 (R = 2)
returns the new value and repairs replica 2.

<details>
<summary>Hint</summary>

`ask.iter().filter_map(|&i| replicas[i].get(key)).max_by_key(|(version, _)| *version)` finds the newest answer.
Return the stored pair by value (`cloned()`), so you can then borrow `replicas` mutably for the repair.

</details>

<details>
<summary>Answer</summary>

```rust
//! Exercise 2: a leaderless store with read repair.

use std::collections::HashMap;

/// A replica in a leaderless store: every value carries a version number.
#[derive(Debug, Default)]
pub struct VersionedReplica {
    data: HashMap<String, (u64, String)>,
}

impl VersionedReplica {
    /// Stores the value unless the replica already has the same or a newer version.
    pub fn put(&mut self, key: &str, version: u64, value: &str) {
        let newer = match self.data.get(key) {
            Some((old, _)) => version > *old,
            None => true,
        };
        if newer {
            self.data
                .insert(key.to_string(), (version, value.to_string()));
        }
    }

    pub fn get(&self, key: &str) -> Option<(u64, String)> {
        self.data.get(key).cloned()
    }
}

/// Reads `key` from the replicas listed in `ask`, returns the newest value, and writes it
/// back to the asked replicas that had an older version or none (read repair).
pub fn quorum_read(replicas: &mut [VersionedReplica], key: &str, ask: &[usize]) -> Option<String> {
    let newest = ask
        .iter()
        .filter_map(|&i| replicas[i].get(key))
        .max_by_key(|(version, _)| *version)?;
    for &i in ask {
        replicas[i].put(key, newest.0, &newest.1);
    }
    Some(newest.1)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_read_finds_the_newest_value_and_repairs_the_stale_replica() {
        let mut replicas: Vec<VersionedReplica> =
            (0..3).map(|_| VersionedReplica::default()).collect();
        for replica in replicas.iter_mut() {
            replica.put("x", 1, "old");
        }
        // A write with W = 2 reached replicas 0 and 1 only.
        replicas[0].put("x", 2, "new");
        replicas[1].put("x", 2, "new");

        // A read with R = 2 asks replicas 1 and 2.
        assert_eq!(
            quorum_read(&mut replicas, "x", &[1, 2]),
            Some("new".to_string())
        );
        assert_eq!(replicas[2].get("x"), Some((2, "new".to_string()))); // repaired
    }
}
```

`cargo test --lib`:

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: because R + W > N, at least one asked replica (here replica 1) has version 2. Writing it back to the
others costs nothing extra for the reader and makes the next read more likely to find the new value everywhere.

</details>

### Exercise 3 — Read your own writes (hard)

Write `pub fn read_your_writes(cluster: &Cluster, i: usize, key: &str, min_position: usize) -> Result<Option<String>, String>`
in `src/lib.rs`. The client passes the position that `write` returned for its last write; the function reads from
follower `i` only if that follower has applied at least that many entries, and returns an error otherwise (the client
can then try another replica or the leader). Add a test to `tests/sync_and_sessions.rs`: after a write that only
follower 0 has, follower 0 returns the new value and follower 1 returns an error.

<details>
<summary>Hint</summary>

Compare `follower.applied()` with `min_position`. Return the value as an owned `String` (`map(str::to_string)`),
because the result must not borrow from the cluster.

</details>

<details>
<summary>Answer</summary>

```rust
/// Reads `key` from follower `i`, but only if it has applied at least `min_position`
/// entries, the position of the client's own last write.
pub fn read_your_writes(
    cluster: &Cluster,
    i: usize,
    key: &str,
    min_position: usize,
) -> Result<Option<String>, String> {
    let follower = &cluster.followers[i];
    if follower.applied() < min_position {
        return Err(format!(
            "follower {i} has applied {} of {min_position} entries",
            follower.applied()
        ));
    }
    Ok(follower.get(key).map(str::to_string))
}
```

The test, added to `tests/sync_and_sessions.rs` (and change its first line to
`use p18_replication::{Cluster, read_your_writes};`):

```rust
#[test]
fn a_client_never_reads_older_data_than_its_own_write() {
    let mut cluster = Cluster::new(2);
    cluster.write("name", "Ann");
    cluster.replicate(0);
    cluster.replicate(1);
    let mine = cluster.write("name", "Anna"); // the client remembers the position: 2
    cluster.replicate(0);

    assert_eq!(
        read_your_writes(&cluster, 0, "name", mine),
        Ok(Some("Anna".to_string()))
    );
    assert!(read_your_writes(&cluster, 1, "name", mine).is_err()); // follower 1 is behind
}
```

`cargo test --test sync_and_sessions`:

```text
test synchronous_writes_survive_a_failover ... ok
test a_client_never_reads_older_data_than_its_own_write ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: a log position is a promise "everything up to here". If a follower has applied that many entries, it
has the client's write. This guarantee is called *read-your-writes*; Lesson 18.7 lists it with the other session
guarantees.

</details>

## Quiz

1. Why do followers apply the leader's log entries in exactly the same order?

<details>
<summary>Answer</summary>

Because they start from the same state, applying the same changes in the same order gives the same result. In a
different order, `x = 1` and `x = 2` could end as 1 on one replica and 2 on another.

</details>

2. A write was confirmed to the client, and then the leader crashed. Under which replication mode can the write be
   lost?

<details>
<summary>Answer</summary>

Asynchronous: the leader confirmed before any follower had the entry, and the new leader may not have it. With
synchronous replication (or a confirmed majority), the new leader has it.

</details>

3. What is split brain, and what prevents it?

<details>
<summary>Answer</summary>

Two nodes both act as leader and accept writes, so the data diverges. A leader number (term or fencing token) that
grows with each new leader, checked on every write, lets the rest of the system reject the old leader.

</details>

4. N = 5. Which of these pairs guarantee that a read meets the latest write: W = 3, R = 3; W = 2, R = 3; W = 5, R = 1?

<details>
<summary>Answer</summary>

W = 3, R = 3 (6 > 5) and W = 5, R = 1 (6 > 5). W = 2, R = 3 gives 5, not more than 5, so a read can miss the write.

</details>

## Summary

- Replication keeps copies for availability, durability, read scaling and latency; all difficulty comes from changes.
- Single-leader replication: the leader orders all writes in a log, followers apply it in the same order.
- Followers lag behind, so reads from them can return old data.
- Asynchronous replication can lose confirmed writes on failover; synchronous replication stalls on a slow follower;
  majorities (Raft) are the middle way.
- Failover risks split brain; a growing leader number (term, fencing token) prevents it.
- Leaderless systems read and write quorums; `R + W > N` makes read and write groups overlap, and read repair fixes
  stale replicas.

## Sources and further reading

- [Designing Data-Intensive Applications](https://dataintensive.net/) — the replication chapter covers leaders and followers, replication lag, multi-leader and leaderless designs in depth.
- [PostgreSQL: Log-Shipping Standby Servers](https://www.postgresql.org/docs/current/warm-standby.html) — how a real database ships its write-ahead log to followers, and what synchronous replication means there.
- [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — the paper behind leaderless replication with N, W and R quorums and read repair.
- [Redis replication](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/) — a short, practical description of asynchronous leader-follower replication and its trade-offs.

---

[🏠 Start](../START-HERE.md) · [← 18.5 Logical clocks: Lamport and vector clocks](05-logical-clocks.md) · [📋 Phase 18](README.md) · [18.7 Consistency models →](07-consistency-models.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
