<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.10 Raft: leader election](10-raft-leader-election.md) · [📋 Phase 18](README.md) · [18.12 Deterministic simulation testing →](12-deterministic-simulation-testing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/18-distributed-systems/11-raft-log-replication.md) <!-- nav -->

# Lesson 18.11 — Raft: log replication

> **Phase 18 — Distributed systems** · Lesson 11 of 15 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How a Raft leader copies its log to the followers with `AppendEntries`, and how a follower checks that its log matches before it accepts new entries.
- How the leader repairs a follower whose log is behind or contains entries that must be thrown away.
- When an entry becomes **committed**, and why a leader may count copies only for entries of its own term.
- Why a new leader writes a *no-op* entry first.
- How to test "every replica applies the same commands in the same order" under partitions and message loss.

## Before you start

- You finished [Lesson 18.10 — Raft: leader election](10-raft-leader-election.md) and have its `code/p18-raft` package with all tests passing. This lesson changes that package.
- Remember the log picture of [Lesson 18.9](09-consensus-and-raft.md): every entry has an index (from 1) and a term; an entry is committed once it is stored on a majority.
- Keep the Raft paper's Figure 2 open if you like (see the sources). Every rule in this lesson is one line of that figure.

## New words

| Word | Meaning |
|---|---|
| **`AppendEntries`** | The leader's message to a follower: "here are entries after index `prev_log_index`, whose term is `prev_log_term`; commit up to `leader_commit`". With no entries it is a heartbeat. |
| **consistency check** | The follower's test before it appends: do I have an entry at `prev_log_index` with term `prev_log_term`? If not, it refuses. |
| **`next_index`** | For each follower, the index of the next entry the leader will send it. It moves back after a refusal and forward after a success. |
| **`match_index`** | For each follower, the highest index the leader *knows* is the same on that follower. The leader counts copies with it. |
| **commit index** | The highest index known to be committed. A node applies entries up to it. |
| **apply** | Give a committed command to the state machine (Lesson 18.9). Here the state machine is just a list, `applied`. |
| **no-op entry** | An entry with no command. A new leader appends one so that it can commit (and learn about) the entries of earlier terms. |
| **log matching** | Raft's promise: if two logs have an entry with the same index and term, they are identical up to that index. |
| **conflict** | A follower has an entry at some index with a different term than the leader's entry at that index. The follower's entry (and everything after it) was never committed and is deleted. |

## 1. The leader sends, the follower checks

When a client wants to change the data, it sends the command to the leader. The leader appends it to its own log (with its current term) and sends it to every follower in an `AppendEntries` message. The message also says **what comes right before** the new entries:

```text
leader's log:     index  1    2    3    4
                  term  [1]  [1]  [2]  [2]
                                   ^    ^
                    prev_log_index=3    new entry
                    prev_log_term=2

AppendEntries { term: 2, prev_log_index: 3, prev_log_term: 2,
                entries: [ (2, "c=7") ], leader_commit: 3 }
```

The follower accepts only if its own log has an entry at index 3 with term 2. This is the **consistency check**. Why is one entry enough? By induction: the same check was made when entry 3 was appended, and when entry 2 was appended, and so on. If the entries at index 3 agree, everything before them agrees too. That is the **log matching** promise, and it costs one comparison per message.

If the check fails, the follower answers "no". The leader then moves back by one (`next_index - 1`) and tries again with an earlier `prev_log_index`, until it finds the point where both logs agree. From there it sends everything the follower is missing. On the way, the follower deletes any of its own entries that **conflict** with the leader's:

```text
before:   leader   [1 a] [1 b] [2 c]
          follower [1 a] [1 b] [1 x] [1 y]     (x, y from an old leader, never committed)

try prev=3 (term 2): follower has term 1 at index 3 -> refuse
try prev=2 (term 1): match -> entries [2 c] replace [1 x] [1 y]

after:    follower [1 a] [1 b] [2 c]
```

Deleting entries sounds dangerous, but it is safe: an entry that conflicts with the current leader's log can **never** have been committed, because the election rule of Lesson 18.10 guarantees that the leader has every committed entry.

## 2. New messages and new state

Open `src/node.rs`. Replace the `Heartbeat` and `HeartbeatReply` variants with the two replication messages, and update `Message::term`:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum Message {
    RequestVote {
        term: u64,
        last_log_index: u64,
        last_log_term: u64,
    },
    VoteReply {
        term: u64,
        granted: bool,
    },
    AppendEntries {
        term: u64,
        prev_log_index: u64,
        prev_log_term: u64,
        entries: Vec<Entry>,
        leader_commit: u64,
    },
    AppendReply {
        term: u64,
        success: bool,
        match_index: u64,
    },
}

impl Message {
    pub fn term(&self) -> u64 {
        match self {
            Message::RequestVote { term, .. }
            | Message::VoteReply { term, .. }
            | Message::AppendEntries { term, .. }
            | Message::AppendReply { term, .. } => *term,
        }
    }
}
```

**What happens:**

1. `AppendEntries` carries the consistency check (`prev_log_index`, `prev_log_term`), the new entries (maybe none: then it is a heartbeat), and the leader's commit index.
2. `AppendReply` says whether the check passed. On success, `match_index` is the index of the last entry the follower now shares with the leader, so the leader learns exactly how far this follower is.

Add `BTreeMap` to the imports (`use std::collections::{BTreeMap, BTreeSet};`) and five fields at the end of `struct Node`:

```rust
    pub commit_index: u64,
    last_applied: u64,
    pub applied: Vec<String>,
    next_index: BTreeMap<NodeId, u64>,
    match_index: BTreeMap<NodeId, u64>,
```

Initialise them in `Node::new`, after `outbox: Vec::new(),`:

```rust
            commit_index: 0,
            last_applied: 0,
            applied: Vec::new(),
            next_index: BTreeMap::new(),
            match_index: BTreeMap::new(),
```

`commit_index` and `last_applied` start at 0 (nothing committed or applied). `applied` is our state machine: the list of commands applied so far, without no-ops. `next_index` and `match_index` are used only by a leader.

Replace `last_log` with these two helpers; `term_at` is used everywhere from now on:

```rust
    /// Index and term of the last log entry, (0, 0) for an empty log.
    fn last_log(&self) -> (u64, u64) {
        let index = self.log.len() as u64;
        (index, self.term_at(index))
    }

    /// Term of the entry at `index`; 0 for index 0 (before the first entry).
    fn term_at(&self, index: u64) -> u64 {
        if index == 0 {
            0
        } else {
            self.log[index as usize - 1].term
        }
    }
```

Index 0 means "before the first entry"; its term is 0. Every log matches every other log at index 0, so the leader can always fall back to sending everything.

## 3. Becoming leader, and the no-op entry

In `tick`, the leader no longer sends `Heartbeat`: replace `self.send_heartbeats();` with `self.replicate();` (written in Section 5). Then change `check_votes` and delete `send_heartbeats`:

```rust
    fn check_votes(&mut self) {
        if self.role == Role::Candidate && self.votes.len() >= self.majority() {
            self.role = Role::Leader;
            self.elapsed = 0;
            let next = self.log.len() as u64 + 1;
            for &peer in &self.peers {
                self.next_index.insert(peer, next);
                self.match_index.insert(peer, 0);
            }
            // A no-op entry of our own term lets us commit older entries.
            self.log.push(Entry {
                term: self.term,
                command: String::new(),
            });
            self.replicate();
        }
    }
```

**What happens:**

1. A new leader does not know how far each follower is. It starts optimistic: `next_index` is "right after my last entry", `match_index` is 0 ("I know nothing yet"). Refusals will move `next_index` back where needed.
2. It appends a **no-op entry**, an entry with an empty command, in its own term, and replicates at once.

Why the no-op? Section 6 explains the commit rule: a leader may only count copies of entries **from its own term**. A new leader may hold entries from earlier terms that are already committed, but it cannot know that. Until one entry of its own term is committed, it cannot advance its commit index at all, even if no client ever writes again. The no-op solves this right after the election. Section 7 shows a test that fails without it.

## 4. Handling the messages

In `step`, replace the two heartbeat arms with these two:

```rust
            Message::AppendEntries {
                term,
                prev_log_index,
                prev_log_term,
                entries,
                leader_commit,
            } => {
                let (success, match_index) = if term < self.term {
                    (false, 0) // an old leader: refuse; the reply tells it our term
                } else {
                    self.role = Role::Follower;
                    self.reset_election_timer();
                    self.accept_entries(prev_log_index, prev_log_term, entries, leader_commit)
                };
                let reply = Message::AppendReply {
                    term: self.term,
                    success,
                    match_index,
                };
                self.send(from, reply);
            }
            Message::AppendReply {
                term,
                success,
                match_index,
            } => {
                if self.role == Role::Leader && term == self.term {
                    self.handle_append_reply(from, success, match_index);
                }
            }
```

**What happens:**

1. An `AppendEntries` from an older term comes from an old leader: refuse. Our reply carries our newer term, so the old leader steps down (the general rule at the top of `step`).
2. Otherwise the sender is the leader of our term. A candidate in this term gives up; everyone restarts the election timer. Then `accept_entries` (next section) does the real work and returns the reply values.
3. An `AppendReply` is used only by a leader, and only if it is from the current term. An old reply from an earlier term is ignored.

## 5. The replication code

Add a fourth `impl` block at the end of `src/node.rs`:

```rust
// Log replication.
impl Node {
    /// A client asks the leader to add a command. Returns its log index.
    pub fn propose(&mut self, command: &str) -> Option<u64> {
        if self.role != Role::Leader {
            return None;
        }
        self.log.push(Entry {
            term: self.term,
            command: command.to_string(),
        });
        self.replicate();
        self.advance_commit(); // a 1-node cluster commits at once
        Some(self.log.len() as u64)
    }

    /// Leader: send every follower the entries it is missing (maybe none).
    fn replicate(&mut self) {
        for peer in self.peers.clone() {
            self.send_append(peer);
        }
    }

    fn send_append(&mut self, peer: NodeId) {
        let prev_log_index = self.next_index[&peer] - 1;
        let msg = Message::AppendEntries {
            term: self.term,
            prev_log_index,
            prev_log_term: self.term_at(prev_log_index),
            entries: self.log[prev_log_index as usize..].to_vec(),
            leader_commit: self.commit_index,
        };
        self.send(peer, msg);
    }

    /// Follower: check that our log matches the leader's at `prev_log_index`,
    /// then append. Returns (success, index of the last entry we now share).
    fn accept_entries(
        &mut self,
        prev_log_index: u64,
        prev_log_term: u64,
        entries: Vec<Entry>,
        leader_commit: u64,
    ) -> (bool, u64) {
        if prev_log_index > self.log.len() as u64 || self.term_at(prev_log_index) != prev_log_term {
            return (false, 0);
        }
        let last_new = prev_log_index + entries.len() as u64;
        for (i, entry) in entries.into_iter().enumerate() {
            let index = prev_log_index + 1 + i as u64;
            if index <= self.log.len() as u64 {
                if self.term_at(index) == entry.term {
                    continue; // we already have this entry
                }
                // Conflict: drop this entry and everything after it.
                self.log.truncate(index as usize - 1);
            }
            self.log.push(entry);
        }
        if leader_commit > self.commit_index {
            self.commit_index = leader_commit.min(last_new);
            self.apply();
        }
        (true, last_new)
    }

    fn handle_append_reply(&mut self, from: NodeId, success: bool, match_index: u64) {
        if success {
            let known = self.match_index[&from];
            self.match_index.insert(from, known.max(match_index));
            self.next_index.insert(from, known.max(match_index) + 1);
            self.advance_commit();
        } else {
            // The follower's log does not match yet: step back and retry.
            let next = self.next_index[&from];
            self.next_index.insert(from, (next - 1).max(1));
            self.send_append(from);
        }
    }

    /// Leader: commit the highest index stored on a majority,
    /// counting only entries from the current term.
    fn advance_commit(&mut self) {
        let last = self.log.len() as u64;
        for n in (self.commit_index + 1..=last).rev() {
            if self.term_at(n) != self.term {
                break;
            }
            let copies = 1 + self.match_index.values().filter(|&&m| m >= n).count();
            if copies >= self.majority() {
                self.commit_index = n;
                self.apply();
                return;
            }
        }
    }

    /// Give every newly committed command to the state machine, in order.
    fn apply(&mut self) {
        while self.last_applied < self.commit_index {
            self.last_applied += 1;
            let entry = &self.log[self.last_applied as usize - 1];
            if !entry.command.is_empty() {
                self.applied.push(entry.command.clone()); // skip no-op entries
            }
        }
    }
}
```

**What happens:**

1. `propose` is how a client's command enters the system. Only a leader accepts it; the others return `None` (a real server would tell the client who the leader is). The leader appends the entry with its own term and replicates it.
2. `replicate` sends every follower an `AppendEntries`. It runs on every heartbeat tick too, so a follower that missed a message gets the entries again a few ticks later. Message loss is repaired by simply sending again.
3. `send_append` builds the message from `next_index`: everything from `next_index` to the end of the log, plus the index and term just before. For an up-to-date follower `entries` is empty: a heartbeat.
4. `accept_entries` is the follower side. First the consistency check: if our log is too short or has a different term at `prev_log_index`, refuse. Then each entry: if we already have it (same index, same term), skip it; if we have a different entry there, it is a conflict: `truncate` drops it and everything after it; then append. Skipping matters, because the network may deliver an old, shorter `AppendEntries` late, and it must not cut off entries we already have.
5. The follower's new commit index is the leader's, but never beyond the last entry this message confirmed (`last_new`). Entries after that might still be replaced.
6. `handle_append_reply` is the leader side. On success it moves `match_index` and `next_index` forward (with `max`, so a late, old reply cannot move them back) and tries to commit. On a refusal it steps `next_index` back by one, never below 1, and sends again immediately.
7. `advance_commit` looks for the highest index `n` stored on a majority (the leader itself plus every follower with `match_index >= n`). It looks only at entries of the **current term** and stops at the first older one. Section 6 explains why.
8. `apply` hands every newly committed entry to the state machine, in index order, exactly once. `last_applied` remembers how far it got. No-op entries are skipped.

> ⚡ **Performance:** stepping `next_index` back one entry per round trip is slow for a follower that is thousands of entries behind. Real implementations let the follower answer with a hint ("my log ends at index 120" or "my entry at 300 has term 4, which starts at index 280"), so the leader jumps back in one step. They also limit how many entries go into one message.

## 6. When is an entry committed?

"Stored on a majority" is almost the whole rule. The missing part is subtle. Imagine this situation in a 5-node cluster:

```text
          index:  1    2
node 1 (leader) [1]  [2]      leader of term 2 wrote entry 2 to itself and node 2, then crashed
node 2          [1]  [2]
node 3          [1]
node 4          [1]
node 5          [1]  [3]      leader of term 3 wrote a different entry 2, crashed
```

Now node 1 becomes leader again, in term 4, and copies its entry 2 (from term 2) to node 3. Entry 2 is on nodes 1, 2 and 3: a majority. Is it committed? **No.** If node 1 crashes now, node 5 can still win an election (its last term, 3, beats the others' 2; nodes 2, 3 and 4 vote for it) and replace entry 2 everywhere with its own. An entry that was "on a majority" disappeared.

This is the famous Figure 8 of the Raft paper, and the fix is the rule in `advance_commit`: **a leader commits only by counting copies of an entry from its own term.** Once an entry of term 4 is on a majority, no node with an older last term can win an election, so that entry, and everything before it, is safe. The earlier entries become committed *indirectly*. This is also why the no-op of Section 3 matters: it is the term-4 entry that lets the older entries commit right away.

## 7. Watch it, then test it

Update `src/cluster.rs`. Add message loss to the network: two new fields, a setter, and one check in `tick`. Add `use rand::rngs::StdRng;` and `use rand::{RngExt, SeedableRng};` at the top, then:

```rust
pub struct Cluster {
    pub nodes: BTreeMap<NodeId, Node>,
    in_flight: VecDeque<Envelope>,
    cut_off: BTreeSet<NodeId>,
    loss_percent: u32,
    net_rng: StdRng,
}
```

In `Cluster::new`, add `loss_percent: 0,` and `net_rng: StdRng::seed_from_u64(seed),` to the struct literal. Then add the setter after `heal`, and the loss check in `tick`:

```rust
    /// Lose this percentage of all messages, chosen by the seeded RNG.
    pub fn set_message_loss(&mut self, percent: u32) {
        self.loss_percent = percent;
    }

    /// One tick for every node, then deliver messages until none are left.
    pub fn tick(&mut self) {
        for node in self.nodes.values_mut() {
            node.tick();
        }
        self.collect();
        while let Some(env) = self.in_flight.pop_front() {
            if self.cut_off.contains(&env.from) || self.cut_off.contains(&env.to) {
                continue; // the network loses it
            }
            if self.net_rng.random_range(0..100) < self.loss_percent {
                continue; // an unlucky message
            }
            let node = self.nodes.get_mut(&env.to).expect("unknown node");
            node.step(env.from, env.msg);
            self.collect();
        }
    }
```

Finally, two helpers at the end of `impl Cluster`, to find the leader and send it a command:

```rust
    /// The leader with the highest term, if there is one.
    pub fn leader(&self) -> Option<NodeId> {
        self.leaders()
            .into_iter()
            .max_by_key(|&(_, term)| term)
            .map(|(id, _)| id)
    }

    /// Give a command to the current leader. False if there is none.
    pub fn propose(&mut self, command: &str) -> bool {
        match self.leader() {
            Some(id) => self.nodes.get_mut(&id).unwrap().propose(command).is_some(),
            None => false,
        }
    }
```

`leader` picks the node with the highest term among those that believe they lead, because an isolated old leader may still believe it too.

Now watch a partition repair a log. Create `src/bin/replicate.rs`:

```rust
use p18_raft::Config;
use p18_raft::cluster::Cluster;

fn show(cluster: &Cluster, title: &str) {
    println!("{title}");
    for node in cluster.nodes.values() {
        let log: Vec<String> = node
            .log
            .iter()
            .map(|e| {
                let cmd = if e.command.is_empty() {
                    "-"
                } else {
                    &e.command
                };
                format!("{}:{cmd}", e.term)
            })
            .collect();
        println!(
            "  node {} {:?} term {} commit {} log [{}]",
            node.id,
            node.role,
            node.term,
            node.commit_index,
            log.join(" ")
        );
    }
}

fn run(cluster: &mut Cluster, ticks: u32) {
    for _ in 0..ticks {
        cluster.tick();
    }
}

fn main() {
    let mut cluster = Cluster::new(3, Config::default(), 7);
    run(&mut cluster, 15);
    cluster.propose("a");
    cluster.propose("b");
    run(&mut cluster, 5);
    show(&cluster, "1. two writes, replicated and committed:");

    let old = cluster.leader().unwrap();
    cluster.isolate(old);
    cluster.nodes.get_mut(&old).unwrap().propose("lost");
    run(&mut cluster, 40);
    cluster.propose("c");
    run(&mut cluster, 5);
    show(&cluster, "2. old leader cut off, a new leader wrote c:");

    cluster.heal(old);
    run(&mut cluster, 5);
    show(
        &cluster,
        "3. after healing, the old leader's log is repaired:",
    );
}
```

**What happens:**

1. `show` prints each node's log as `term:command`, with `-` for a no-op.
2. Step 1: two writes on a healthy cluster. Step 2: the leader is cut off, but it still believes it leads, so we give it a write (`lost`) directly. The majority elects a new leader, which writes `c`. Step 3: the old leader is back.

Run `cargo run --bin replicate`:

```text
1. two writes, replicated and committed:
  node 1 Leader term 1 commit 3 log [1:- 1:a 1:b]
  node 2 Follower term 1 commit 3 log [1:- 1:a 1:b]
  node 3 Follower term 1 commit 3 log [1:- 1:a 1:b]
2. old leader cut off, a new leader wrote c:
  node 1 Leader term 1 commit 3 log [1:- 1:a 1:b 1:lost]
  node 2 Leader term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
  node 3 Follower term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
3. after healing, the old leader's log is repaired:
  node 1 Follower term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
  node 2 Leader term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
  node 3 Follower term 2 commit 5 log [1:- 1:a 1:b 2:- 2:c]
```

In step 1 every log starts with the no-op of term 1, then `a` and `b`; all three are committed. In step 2 node 1 has `1:lost` at index 4, but its commit index stays 3: it can never reach a majority. Node 2 won term 2, wrote its own no-op and `c`, and committed them with node 3. In step 3, node 1 receives `AppendEntries` with term 2, steps down, fails the consistency check at index 4, and after one step back its `1:lost` is replaced by `2:- 2:c`. The write `lost` was never committed, so no client was ever told it succeeded. Losing it is correct.

Now the tests. Create `tests/replication.rs`:

```rust
use p18_raft::Config;
use p18_raft::cluster::Cluster;
use rand::rngs::StdRng;
use rand::{RngExt, SeedableRng};

fn run(cluster: &mut Cluster, ticks: u32) {
    for _ in 0..ticks {
        cluster.tick();
    }
}

#[test]
fn every_node_applies_the_same_commands() {
    let mut cluster = Cluster::new(3, Config::default(), 1);
    run(&mut cluster, 50);
    for cmd in ["a=1", "b=2", "a=3"] {
        assert!(cluster.propose(cmd));
    }
    run(&mut cluster, 10);
    for node in cluster.nodes.values() {
        assert_eq!(node.applied, ["a=1", "b=2", "a=3"], "node {}", node.id);
    }
}

#[test]
fn a_lagging_follower_catches_up() {
    let mut cluster = Cluster::new(3, Config::default(), 2);
    run(&mut cluster, 50);
    let leader = cluster.leader().unwrap();
    let lagging = if leader == 1 { 2 } else { 1 };

    cluster.isolate(lagging);
    for i in 0..5 {
        cluster.propose(&format!("x={i}"));
    }
    run(&mut cluster, 10);
    // Two of three nodes are a majority: the leader commits without it.
    assert_eq!(cluster.nodes[&leader].applied.len(), 5);
    assert_eq!(cluster.nodes[&lagging].applied.len(), 0);

    cluster.heal(lagging);
    run(&mut cluster, 10);
    assert_eq!(
        cluster.nodes[&lagging].applied,
        cluster.nodes[&leader].applied
    );
}

#[test]
fn an_uncommitted_entry_of_a_cut_off_leader_is_replaced() {
    let mut cluster = Cluster::new(5, Config::default(), 3);
    run(&mut cluster, 50);
    assert!(cluster.propose("first"));
    run(&mut cluster, 10);

    // The leader is cut off, but still believes it leads: it accepts a write.
    let old = cluster.leader().unwrap();
    cluster.isolate(old);
    assert!(
        cluster
            .nodes
            .get_mut(&old)
            .unwrap()
            .propose("lost")
            .is_some()
    );
    run(&mut cluster, 60);
    assert_eq!(cluster.nodes[&old].applied, ["first"], "never committed");

    // The majority elects a new leader and commits a different write.
    assert!(cluster.propose("second"));
    run(&mut cluster, 10);

    cluster.heal(old);
    run(&mut cluster, 20);
    for node in cluster.nodes.values() {
        assert_eq!(node.applied, ["first", "second"], "node {}", node.id);
    }
}

/// One of two lists must be the beginning of the other.
fn is_prefix(a: &[String], b: &[String]) -> bool {
    let n = a.len().min(b.len());
    a[..n] == b[..n]
}

#[test]
fn state_machine_safety_under_chaos() {
    for seed in 0..200 {
        let mut cluster = Cluster::new(5, Config::default(), seed);
        cluster.set_message_loss(20);
        let mut chaos = StdRng::seed_from_u64(seed + 10_000);
        let mut next_cmd = 0;

        for _ in 0..400 {
            match chaos.random_range(0..20) {
                0 => cluster.isolate(chaos.random_range(1..=5)),
                1 => cluster.heal(chaos.random_range(1..=5)),
                2..=5 => {
                    cluster.propose(&format!("cmd{next_cmd}"));
                    next_cmd += 1;
                }
                _ => {}
            }
            cluster.tick();

            let applied: Vec<&Vec<String>> = cluster.nodes.values().map(|n| &n.applied).collect();
            for a in &applied {
                for b in &applied {
                    assert!(is_prefix(a, b), "seed {seed}: replicas disagree");
                }
            }
        }

        // Liveness: after the network is repaired, all nodes catch up...
        for id in 1..=5 {
            cluster.heal(id);
        }
        cluster.set_message_loss(0);
        run(&mut cluster, 100);
        let applied = &cluster.nodes[&1].applied;
        for node in cluster.nodes.values() {
            assert_eq!(&node.applied, applied, "seed {seed}: node {}", node.id);
        }
        // ...and a new write still commits everywhere.
        assert!(cluster.propose("final"), "seed {seed}: no leader");
        run(&mut cluster, 20);
        for node in cluster.nodes.values() {
            assert_eq!(node.applied.last().unwrap(), "final", "seed {seed}");
        }
    }
}
```

**What happens:**

1. `every_node_applies_the_same_commands`: three writes on a healthy cluster reach every state machine in order.
2. `a_lagging_follower_catches_up`: with one of three nodes cut off, the other two are a majority and commit; when it is back, it catches up.
3. `an_uncommitted_entry_of_a_cut_off_leader_is_replaced`: the same story as the demo, as a test, on 5 nodes.
4. `state_machine_safety_under_chaos`: for 200 seeds, 400 ticks each, with 20 % of all messages lost, random nodes cut off and brought back, and a write proposed on about one tick in five. After **every** tick it checks the main promise: for any two nodes, one `applied` list is the beginning (a prefix) of the other. Nodes may be behind, but never different. At the end it repairs the network and checks liveness: everyone catches up, and a final write is applied everywhere.

```text
test every_node_applies_the_same_commands ... ok
test a_lagging_follower_catches_up ... ok
test an_uncommitted_entry_of_a_cut_off_leader_is_replaced ... ok
test state_machine_safety_under_chaos ... ok
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.65s
```

Run the whole suite with `cargo test`: the election tests of Lesson 18.10 still pass too.

Now see what the no-op is worth. Delete the `self.log.push(Entry { … })` lines (the no-op) from `check_votes` and run `cargo test --test replication` again:

```text
test state_machine_safety_under_chaos ... FAILED

thread 'state_machine_safety_under_chaos' (6268561) panicked at tests/replication.rs:123:13:
assertion `left == right` failed: seed 16: node 4
  left: ["cmd9", "cmd10", "cmd11", "cmd12", "cmd13", "cmd18", "cmd19", "cmd20", "cmd21", "cmd22", "cmd23", "cmd24", "cmd25", "cmd26", "cmd27", "cmd28", "cmd29", "cmd38", "cmd39", "cmd40", "cmd41", "cmd42", "cmd43", "cmd44", "cmd45", "cmd46", "cmd47", "cmd48", "cmd59", "cmd60", "cmd61"]
 right: ["cmd9", "cmd10", "cmd11", "cmd12", "cmd13", "cmd18", "cmd19", "cmd20", "cmd21", "cmd22", "cmd23", "cmd24", "cmd25", "cmd26", "cmd27", "cmd28", "cmd29", "cmd38", "cmd39", "cmd40", "cmd41", "cmd42", "cmd43", "cmd44", "cmd45"]
```

Safety still holds (one list is a prefix of the other), but after the repair node 4 has applied 31 commands and node 1 only 25. An old leader had committed `cmd46` … `cmd61`, and the new leader, without an entry of its own term, has no way to commit them again. It would wait for the next client write. With the no-op, it commits them right after its election. Put the no-op back.

Remember Lesson 18.10's promise: break the election's up-to-date rule (`let log_ok = true;`) and run the tests. Now that logs are not empty, the chaos test fails on its very first seed (`seed 0: node 2`): a leader without some committed entries was elected and overwrote them. Put the rule back.

> 🧪 **Testing:** the chaos test checks properties ("one list is a prefix of the other", "everyone catches up"), not exact outputs. This is property-based testing from [Lesson 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md), applied to a whole cluster. The seed plays the role of the generated input, and every failure is reproducible.

## Common mistakes

> ⚠️ **Common mistake:** truncating the follower's log on every `AppendEntries`. A late, older message with fewer entries would then delete entries that are already stored, and maybe already counted by the leader. Truncate only on a real conflict (same index, different term).

> ⚠️ **Common mistake:** setting the follower's commit index to `leader_commit` without the `min(last_new)`. The follower might mark entries as committed that it has not yet received from this leader, and apply stale entries from an old leader.

> ⚠️ **Common mistake:** committing old-term entries by counting copies (Section 6). The code looks reasonable and passes most tests; then, in a rare sequence of crashes, a committed entry disappears. Our chaos test did not catch this mistake (try it: remove the `if self.term_at(n) != self.term { break; }` lines): the Figure 8 situation needs leaders that crash at very specific moments.

> ⚠️ **Common mistake:** applying an entry before it is committed, for example as soon as it arrives on a follower. Uncommitted entries can still be replaced, but an applied command cannot be "un-applied".

## Try it yourself

1. Run `cargo run --bin replicate`. Then give the old leader three writes instead of one and check that all three disappear after the repair.
2. In `tests/replication.rs`, raise the message loss to 50 %. All four tests still pass. Why can Raft live with losing every second message? (Every heartbeat sends again whatever a follower is missing.)
3. Break the consistency check: change the condition in `accept_entries` to `if prev_log_index > self.log.len() as u64 {` and run the tests. Read which seed and which node fail, then restore the line.
4. Add a `println!` in `handle_append_reply` when a follower refuses, and run the `replicate` example to watch the leader step back.

## Exercises

### Exercise 1 — Why does `min(last_new)` matter? (easy)

In `accept_entries`, the follower sets `commit_index = leader_commit.min(last_new)`. Describe in two or three sentences a situation where using just `leader_commit` would apply a wrong command.

<details>
<summary>Hint</summary>

Think of a follower that still has entries from an old leader **after** the entries this message confirmed.

</details>

<details>
<summary>Answer</summary>

In our code every `AppendEntries` carries all entries up to the end of the leader's log, so `last_new` is never smaller than `leader_commit`. Real implementations limit the number of entries per message (see the ⚡ note in Section 5), and then it matters. A follower has the stale entries `[1 a] [1 b] [1 x]`. The leader of term 2 has `[1 a] [1 b] [2 c]` and has committed index 3. Because of the limit, its message carries only one entry: `prev_log_index: 1`, entries `[1 b]`, `leader_commit: 3`. With plain `leader_commit`, the follower would commit index 3 and apply its own stale `x` instead of `c`. With `min(last_new)` it commits only up to index 2, the last entry this message proved to match the leader.

</details>

### Exercise 2 — A unit test for log repair (medium)

Write `tests/log.rs` with one test. Create a follower (`Node::new(1, vec![2, 3], …)`), set its log to `[(1, "a"), (1, "b"), (2, "stale")]`, and deliver two `AppendEntries` from node 2 in term 3, each with one new entry `(3, "c")`:

1. with `prev_log_index: 3, prev_log_term: 3`: the follower must refuse (`success: false, match_index: 0`);
2. with `prev_log_index: 2, prev_log_term: 1`: it must accept with `match_index: 3`, and its log must become `[(1, "a"), (1, "b"), (3, "c")]`.

<details>
<summary>Hint</summary>

`node.log` is public, so you can assign a whole `Vec<Entry>`. Small helpers `entry(term, command)` and `append(node, prev_index, prev_term, entries) -> Message` keep the test readable.

</details>

<details>
<summary>Answer</summary>

```rust
use p18_raft::{Config, Entry, Message, Node};

fn entry(term: u64, command: &str) -> Entry {
    Entry {
        term,
        command: command.to_string(),
    }
}

/// Deliver an AppendEntries from leader 2 in term 3 and return the reply.
fn append(
    node: &mut Node,
    prev_log_index: u64,
    prev_log_term: u64,
    entries: Vec<Entry>,
) -> Message {
    let msg = Message::AppendEntries {
        term: 3,
        prev_log_index,
        prev_log_term,
        entries,
        leader_commit: 0,
    };
    node.step(2, msg);
    node.take_outbox().remove(0).msg
}

#[test]
fn a_follower_repairs_a_conflicting_log() {
    let mut node = Node::new(1, vec![2, 3], Config::default(), 1);
    node.log = vec![entry(1, "a"), entry(1, "b"), entry(2, "stale")];

    // The leader's entry before the new one does not match: refuse.
    let reply = append(&mut node, 3, 3, vec![entry(3, "c")]);
    assert_eq!(
        reply,
        Message::AppendReply {
            term: 3,
            success: false,
            match_index: 0
        }
    );

    // One step back, index 2 matches: drop "stale", append "c".
    let reply = append(&mut node, 2, 1, vec![entry(3, "c")]);
    assert_eq!(
        reply,
        Message::AppendReply {
            term: 3,
            success: true,
            match_index: 3
        }
    );
    assert_eq!(node.log, vec![entry(1, "a"), entry(1, "b"), entry(3, "c")]);
}
```

```text
test a_follower_repairs_a_conflicting_log ... ok
```

Why it works: in the first message, the follower's entry at index 3 has term 2, not 3: the check fails. In the second, index 2 has term 1 as expected; the new entry at index 3 has term 3, the follower's has term 2: a conflict, so `stale` is truncated and `c` appended.

</details>

### Exercise 3 — The bug the chaos test misses (hard)

Remove the three lines `if self.term_at(n) != self.term { break; }` from `advance_commit` and run `cargo test`. All tests pass. Explain (a) why the code is still wrong, using the situation of Section 6, and (b) two changes to the simulated `Cluster` that would give the chaos test a better chance to reach that situation.

<details>
<summary>Hint</summary>

In Section 6, entry 2 of term 2 reached a majority only when node 1 was leader **again**, in a later term, and then node 1 crashed. What does our network never do? Look at how `tick` delivers messages.

</details>

<details>
<summary>Answer</summary>

(a) Without the rule, a leader of term 4 counts copies of an entry from term 2 and commits it once it is on a majority. A node whose last entry has term 3 can still win the next election (its log is "more up to date" than the majority's last term 2) and overwrite that entry. A committed and maybe applied command is replaced: state machine safety is broken.

(b) Our network delivers every message instantly and in order within a tick, and nodes never crash and restart. Two useful changes: **delay and reorder messages** (keep envelopes in flight for a random number of ticks, so a leader can write to itself, lose leadership and win again while old messages are still travelling), and **crash nodes**: drop a leader's outbox and pause it for some ticks right after it appended an entry. Lesson 18.12 does both with `turmoil`, and model checkers such as `stateright` explore every order of events systematically.

</details>

## Quiz

1. What does the consistency check in `AppendEntries` compare, and why is one entry enough?

<details>
<summary>Answer</summary>

It checks that the follower has an entry at `prev_log_index` with term `prev_log_term`. One entry is enough because the same check was made for every earlier entry when it was appended, so matching at one index means matching everywhere before it (log matching).

</details>

2. A follower refuses an `AppendEntries`. What does the leader do?

<details>
<summary>Answer</summary>

It decrements that follower's `next_index` by one (never below 1) and sends again, with an earlier `prev_log_index`, until the logs match. Then it sends everything from that point.

</details>

3. Why is it safe for a follower to delete entries that conflict with the leader's?

<details>
<summary>Answer</summary>

A conflicting entry cannot be committed: the election rule guarantees that the current leader has every committed entry, so any entry that differs from the leader's log was never on a majority in a way that counts.

</details>

4. When may a leader advance its commit index to `n`?

<details>
<summary>Answer</summary>

When the entry at `n` is from the leader's **current term** and is stored on a majority (the leader plus followers with `match_index >= n`). Everything before `n` is then committed too.

</details>

5. Why does a new leader append a no-op entry?

<details>
<summary>Answer</summary>

Because it may commit only by counting entries of its own term. The no-op is such an entry; once it is on a majority, all older entries before it are committed as well, without waiting for a client write.

</details>

6. What does the chaos test check after every tick, and what at the end?

<details>
<summary>Answer</summary>

After every tick: for any two nodes, one `applied` list is a prefix of the other (safety). At the end, after repairing the network: all nodes have the same `applied` list, and a new write is applied everywhere (liveness).

</details>

## Summary

- The leader replicates with `AppendEntries`, which carries the entries plus `prev_log_index` and `prev_log_term`. The follower appends only if its log matches at that point: one check guarantees the whole prefix matches.
- On a refusal the leader moves `next_index` back and retries; the follower deletes conflicting, never-committed entries and takes the leader's.
- `match_index` tells the leader how far each follower is. An entry is committed when an entry **of the current term** is on a majority; older entries commit with it.
- A new leader appends a no-op so it can commit older entries at once.
- Followers commit up to `min(leader_commit, last_new)` and apply committed entries in order, exactly once.
- A seeded chaos test with partitions and message loss checks the prefix property after every tick, and finds broken rules quickly, but not every rare bug: Lesson 18.12 makes the simulation harsher.

## Sources and further reading

- [In Search of an Understandable Consensus Algorithm (the Raft paper)](https://raft.github.io/raft.pdf) — sections 5.3 (log replication) and 5.4 (safety, with Figure 8); section 8 explains the no-op entry.
- [The Raft Consensus Algorithm](https://raft.github.io/) — the visualization lets you write values and cut nodes off, as in this lesson's demo.
- [The Secret Lives of Data: Raft](https://thesecretlivesofdata.com/raft/) — the second half animates log replication and a network partition.
- [Consensus: Bridging Theory and Practice (Ongaro's PhD thesis)](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf) — chapter 3 is the full Raft description, including how followers can help the leader find the matching index faster.
- [MIT 6.5840 Distributed Systems](https://pdos.csail.mit.edu/6.824/) — a university course whose labs ask you to implement Raft and test it under failures, like this lesson.

---

[🏠 Start](../START-HERE.md) · [← 18.10 Raft: leader election](10-raft-leader-election.md) · [📋 Phase 18](README.md) · [18.12 Deterministic simulation testing →](12-deterministic-simulation-testing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
