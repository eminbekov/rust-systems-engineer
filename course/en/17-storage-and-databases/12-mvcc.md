<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.11 Isolation levels](11-isolation-levels.md) · [📋 Phase 17](README.md) · [17.13 How a SQL query runs →](13-how-a-sql-query-runs.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/12-mvcc.md) <!-- nav -->

# Lesson 17.12 — MVCC: multi-version concurrency control

> **Phase 17 — Storage and database internals** · Lesson 12 of 15 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why keeping several versions of a row lets readers and writers work without blocking each other.
- How PostgreSQL stores versions in a table page, and how to see them with `xmin`, `xmax`, `ctid` and `pageinspect`.
- What a snapshot is made of, and the rule that decides which version a transaction sees.
- Why old versions pile up, what `VACUUM` does, and why one forgotten transaction can stop it.
- How to build a tiny MVCC store in Rust with snapshot isolation and "first committer wins".

## Before you start

- You finished [Lesson 17.11 — Isolation levels](11-isolation-levels.md): snapshots, `REPEATABLE READ`, write skew.
- You know write sets and commit from [Lesson 17.10 — Transactions and ACID](10-transactions-and-acid.md), and pages of
  8 KiB (kibibytes of 1,024 bytes) from [Lesson 17.9 — The page cache and the buffer pool](09-page-cache-and-buffer-pool.md).
- The container `p17-postgres` is running, and you can open two `psql` terminals (T1 and T2) as in Lesson 17.11.

## New words

| Word | Meaning |
|---|---|
| **MVCC** (Multi-Version Concurrency Control) | Keeping several versions of each row, so that every transaction can read the version that matches its snapshot. |
| **version** (in PostgreSQL: *tuple*) | One stored copy of a row, with notes about which transaction created it and which one deleted it. |
| **transaction ID** (XID) | A number that PostgreSQL gives every transaction when it first writes. Bigger means "got its number later". |
| **`xmin` / `xmax`** | Hidden columns of every PostgreSQL row version: the XID that created it, and the XID that deleted or replaced it (0 = nobody). |
| **`ctid`** | The physical address of a row version: (page number, slot number in the page). |
| **visibility rule** | The rule that decides whether a version exists for a given snapshot. |
| **dead version** (dead tuple) | A version that no current or future snapshot can see. It only takes space. |
| **`VACUUM`** | The PostgreSQL command (and the background *autovacuum* worker) that removes dead versions and makes their space reusable. |
| **bloat** | A table or index that is much bigger than its live data, because of dead versions and empty space. |
| **first committer wins** | A rule of snapshot isolation: if two concurrent transactions write the same key, the first to commit wins and the second is aborted. |
| **garbage collection** (of versions) | Removing versions that nobody can see any more; `VACUUM` is PostgreSQL's version. |

## 1. The idea: never overwrite, add a version

In Lesson 17.11 a transaction at `REPEATABLE READ` kept seeing alice's old balance while another transaction changed
it. Where did the old value come from? The database **did not overwrite it**. An update writes a **new version** of the
row and keeps the old one:

```text
 alice, 100   created by tx 10456, deleted by tx 10457     <- old version
 alice,  90   created by tx 10457, deleted by nobody       <- new version
```

Each transaction reads from its snapshot, and the snapshot decides which version "exists" for it. This is **MVCC**
(Multi-Version Concurrency Control). It gives a famous property:

> **Readers never block writers, and writers never block readers.**

A reader does not need a lock to protect the row from changes, because the change goes into a different version. Only
two **writers** of the same row still have to deal with each other (wait for a row lock, or abort with `40001`).

## 2. Versions in PostgreSQL

Every PostgreSQL row version has hidden system columns. You can select them by name. In T1, start from a fresh table:

```sql
DROP TABLE accounts;
CREATE TABLE accounts (name text PRIMARY KEY, balance int NOT NULL);
INSERT INTO accounts VALUES ('alice', 100), ('bob', 50);
SELECT ctid, xmin, xmax, name, balance FROM accounts;
UPDATE accounts SET balance = 90 WHERE name = 'alice';
SELECT ctid, xmin, xmax, name, balance FROM accounts;
```

```text
 ctid  | xmin  | xmax | name  | balance
-------+-------+------+-------+---------
 (0,1) | 10456 |    0 | alice |     100
 (0,2) | 10456 |    0 | bob   |      50
(2 rows)

UPDATE 1
 ctid  | xmin  | xmax | name  | balance
-------+-------+------+-------+---------
 (0,2) | 10456 |    0 | bob   |      50
 (0,3) | 10457 |    0 | alice |     90
(2 rows)
```

Your transaction IDs will be different numbers; what matters is how they relate.

**What happens:**

1. The `INSERT` ran as transaction 10456, so both rows have `xmin = 10456`. `xmax = 0`: nobody deleted them.
2. `ctid` `(0,1)` means "page 0, slot 1". Both rows are in the first 8 KiB page of the table.
3. After the `UPDATE`, alice is at `(0,3)`: a **new slot**, created by transaction 10457. The old version in slot 1 is
   still on the page, but your snapshot does not show it any more.

Now watch an update that is not committed yet:

```text
step  T1                                                        T2
1     BEGIN;
2     UPDATE accounts SET balance = 80 WHERE name = 'alice';
3     SELECT pg_current_xact_id();  -> 10458
4                                                               SELECT ctid, xmin, xmax, name, balance FROM accounts;
5                                                               SELECT pg_current_snapshot();
6     COMMIT;
7                                                               SELECT ctid, xmin, xmax, name, balance FROM accounts;
```

T2's results at steps 4, 5 and 7:

```text
 ctid  | xmin  | xmax  | name  | balance
-------+-------+-------+-------+---------
 (0,2) | 10456 |     0 | bob   |      50
 (0,3) | 10457 | 10458 | alice |      90
(2 rows)

 pg_current_snapshot
---------------------
 10458:10458:
(1 row)

 ctid  | xmin  | xmax | name  | balance
-------+-------+------+-------+---------
 (0,2) | 10456 |    0 | bob   |      50
 (0,4) | 10458 |    0 | alice |      80
(2 rows)
```

**What happens:**

1. `pg_current_xact_id()` shows T1's XID, 10458.
2. At step 4, T2 sees the **old** version `(0,3)`, but with `xmax = 10458`: "transaction 10458 wants to delete this".
   Since 10458 has not committed, the version is still alive for T2. T2 did not wait: no reader blocks.
3. A snapshot is written `xmin:xmax:list`. `10458:10458:` means: every XID below 10458 has finished; every XID from
   10458 on counts as **not committed yet** (for this snapshot); the list of other running XIDs is empty.
4. After the commit, T2's next statement takes a new snapshot (`READ COMMITTED`) and sees the new version `(0,4)`.

## 3. The visibility rule

From the hidden columns and a snapshot, PostgreSQL decides for every version whether it exists. Simplified:

```text
A version is visible to a snapshot if
    its creator (xmin) committed and is "in the past" of the snapshot
AND NOT
    its deleter (xmax) committed and is "in the past" of the snapshot.

"In the past" = the XID is below the snapshot's xmax and not in its list of running XIDs.
```

Whether an XID committed or rolled back is stored in a small separate structure, the commit log (`pg_xact`). A rolled
back transaction's versions are simply never visible: this is how PostgreSQL gets **undo for free** (Lesson 17.10).

> 💡 **Tip:** the first time a reader checks the commit log for a version, PostgreSQL writes small "hint bits" into the
> version ("creator committed"), so that later readers do not have to look again. This is why a plain `SELECT` can make
> pages dirty; you may see `dirtied=` in `EXPLAIN (ANALYZE, BUFFERS)` after a big `INSERT`.

## 4. Dead versions and `VACUUM`

Let us look at the page itself. `pageinspect` is an extension shipped with PostgreSQL that decodes raw pages. In T1:

```sql
CREATE EXTENSION pageinspect;
SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('accounts', 0));
```

```text
CREATE EXTENSION
 lp | t_xmin | t_xmax | t_ctid
----+--------+--------+--------
  1 |  10456 |  10457 | (0,3)
  2 |  10456 |      0 | (0,2)
  3 |  10457 |  10458 | (0,4)
  4 |  10458 |      0 | (0,4)
(4 rows)
```

Four versions for two rows. `lp` is the slot ("line pointer"). Slots 1 and 3 are alice's old versions; their `t_ctid`
points to the newer version, so the versions of one row form a chain. Both old versions are **dead**: their deleters
committed, and no running transaction has a snapshot old enough to need them. Now clean up:

```sql
VACUUM accounts;
SELECT lp, lp_flags, t_xmin, t_xmax FROM heap_page_items(get_raw_page('accounts', 0));
```

```text
VACUUM
 lp | lp_flags | t_xmin | t_xmax
----+----------+--------+--------
  1 |        2 |        |
  2 |        1 |  10456 |      0
  3 |        0 |        |
  4 |        1 |  10458 |      0
(4 rows)
```

**What happens:**

1. `lp_flags = 1` means "normal": slots 2 and 4 hold the live rows.
2. Slot 3 is now `0`, "unused": its space can hold a future version.
3. Slot 1 is `2`, "redirect": the primary key index still points at slot 1, so the slot stays as a tiny pointer to the
   live version in slot 4. The dead data itself is gone.

You rarely run `VACUUM` by hand: the **autovacuum** worker runs it in the background when enough rows have changed.

**One open transaction can stop `VACUUM`.** A version is dead only if *no* snapshot needs it. In T2, open a snapshot and
keep it; in T1, update and vacuum:

```text
step  T1                                                        T2
1                                                               BEGIN ISOLATION LEVEL REPEATABLE READ;
2                                                               SELECT balance FROM accounts WHERE name = 'alice';  -> 80
3     UPDATE accounts SET balance = 70 WHERE name = 'alice';
4     VACUUM (VERBOSE) accounts;
5                                                               COMMIT;
6     VACUUM (VERBOSE) accounts;
```

`VERBOSE` prints a report; its `tuples:` line at step 4 and at step 6:

```text
tuples: 0 removed, 3 remain, 1 are dead but not yet removable
tuples: 1 removed, 2 remain, 0 are dead but not yet removable
```

While T2's snapshot was open, the version `alice, 80` was "dead but not yet removable": T2 might still read it. After
T2 committed, the second `VACUUM` removed it.

> ⚡ **Performance:** a session that runs `BEGIN` and then waits (a crashed client, a developer's open `psql`, a
> long report) keeps its snapshot, and from then on `VACUUM` cannot remove **any** version that became dead after it, in
> the whole database. Tables grow (**bloat**), scans read more pages, the buffer pool holds garbage. Watch for sessions
> in the state `idle in transaction` in `pg_stat_activity`, and set `idle_in_transaction_session_timeout`.

## 5. A tiny MVCC store in Rust

Now build the same idea in a few lines. Instead of XIDs and a commit log we use one counter, the **commit timestamp**:

- Every commit gets the next number: 1, 2, 3, ...
- A transaction remembers `start_ts`, the last commit number when it began. That number *is* its snapshot.
- A version is visible to a transaction if its `commit_ts <= start_ts`; of those, the newest wins.
- Writes wait in a write set (as in Lesson 17.10). At commit, if another transaction committed a newer version of one
  of our keys after `start_ts`, we abort: **first committer wins**. PostgreSQL's `REPEATABLE READ` reaches the same
  result a little earlier: the second *writer* of a row waits for the first one's row lock, and as soon as the first
  one commits, the second gets `could not serialize access due to concurrent update` ("first updater wins").

```bash
cd ~/projects/rust
cargo new --lib code/p17-mvcc
cd code/p17-mvcc
```

Replace `src/lib.rs` with:

```rust
use std::collections::BTreeMap;

/// One version of a value. `None` means "deleted in this version".
#[derive(Debug)]
struct Version {
    commit_ts: u64,
    value: Option<String>,
}

#[derive(Debug, Default)]
pub struct Mvcc {
    versions: BTreeMap<String, Vec<Version>>, // oldest version first
    clock: u64,                               // timestamp of the last commit
}

#[derive(Debug)]
pub struct Tx {
    start_ts: u64, // the snapshot: sees commits with commit_ts <= start_ts
    writes: BTreeMap<String, Option<String>>,
}

#[derive(Debug, PartialEq)]
pub struct Conflict {
    pub key: String,
}

impl Tx {
    pub fn put(&mut self, key: &str, value: &str) {
        self.writes.insert(key.to_string(), Some(value.to_string()));
    }

    pub fn delete(&mut self, key: &str) {
        self.writes.insert(key.to_string(), None);
    }
}

impl Mvcc {
    pub fn begin(&self) -> Tx {
        Tx {
            start_ts: self.clock,
            writes: BTreeMap::new(),
        }
    }

    /// Reads `key` as `tx` sees it: its own writes first, then its snapshot.
    pub fn get<'a>(&'a self, tx: &'a Tx, key: &str) -> Option<&'a str> {
        if let Some(own) = tx.writes.get(key) {
            return own.as_deref();
        }
        let versions = self.versions.get(key)?;
        let visible = versions.iter().rev().find(|v| v.commit_ts <= tx.start_ts)?;
        visible.value.as_deref()
    }

    /// Commits `tx`, unless another transaction committed a newer version of
    /// one of its keys after `tx` started ("first committer wins").
    pub fn commit(&mut self, tx: Tx) -> Result<u64, Conflict> {
        for key in tx.writes.keys() {
            let newest = self.versions.get(key).and_then(|v| v.last());
            if newest.is_some_and(|v| v.commit_ts > tx.start_ts) {
                return Err(Conflict { key: key.clone() });
            }
        }
        self.clock += 1;
        for (key, value) in tx.writes {
            let commit_ts = self.clock;
            self.versions
                .entry(key)
                .or_default()
                .push(Version { commit_ts, value });
        }
        Ok(self.clock)
    }

    /// How many versions are stored for `key` (live and dead).
    pub fn version_count(&self, key: &str) -> usize {
        self.versions.get(key).map_or(0, Vec::len)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn store_with(key: &str, value: &str) -> Mvcc {
        let mut db = Mvcc::default();
        let mut tx = db.begin();
        tx.put(key, value);
        db.commit(tx).unwrap();
        db
    }

    #[test]
    fn a_snapshot_does_not_see_later_commits() {
        let mut db = store_with("x", "1");
        let reader = db.begin();
        let mut writer = db.begin();
        writer.put("x", "2");
        db.commit(writer).unwrap();
        assert_eq!(db.get(&reader, "x"), Some("1"));
        assert_eq!(db.get(&db.begin(), "x"), Some("2"));
    }

    #[test]
    fn the_second_writer_of_a_key_is_aborted() {
        let mut db = store_with("x", "1");
        let mut t1 = db.begin();
        let mut t2 = db.begin();
        t1.put("x", "from t1");
        t2.put("x", "from t2");
        assert!(db.commit(t1).is_ok());
        assert_eq!(db.commit(t2), Err(Conflict { key: "x".into() }));
    }
}
```

**What happens:**

1. `versions` maps a key to its versions, oldest first. A delete is a version with `value: None`, like PostgreSQL's
   `xmax`: the key stops existing *from that version on*, while older snapshots still see the old value.
2. `begin` copies the clock: from now on, this transaction sees exactly the commits up to `start_ts`. Taking a snapshot
   costs nothing.
3. `get` looks at the transaction's own writes first (read your own writes), then walks the versions from newest to
   oldest and takes the first one with `commit_ts <= start_ts`. `as_deref` turns `&Option<String>` into
   `Option<&str>`.
4. `commit` first checks every written key. If the newest version of a key has a `commit_ts` greater than our
   `start_ts`, somebody committed it after our snapshot was taken, and our write was based on old data: `Conflict`.
5. Otherwise the clock moves on and every write becomes a new version with the new timestamp. Old versions stay.
6. The two tests are the two promises of snapshot isolation: snapshots are stable, and two writers of one key cannot
   both win.

```bash
cargo test
```

```text
running 2 tests
test tests::the_second_writer_of_a_key_is_aborted ... ok
test tests::a_snapshot_does_not_see_later_commits ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The lost update of Lesson 17.10 again, now on MVCC. Create `src/bin/timeline.rs`:

```rust
use p17_mvcc::Mvcc;

fn main() {
    let mut db = Mvcc::default();
    let mut setup = db.begin();
    setup.put("alice", "100");
    println!("setup committed at ts {}", db.commit(setup).unwrap());

    let reader = db.begin(); // snapshot at ts 1
    let mut t1 = db.begin();
    let mut t2 = db.begin();
    t1.put("alice", "90");
    t2.put("alice", "80");
    println!("t1 committed at ts {}", db.commit(t1).unwrap());
    match db.commit(t2) {
        Ok(ts) => println!("t2 committed at ts {ts}"),
        Err(conflict) => println!("t2 aborted: {conflict:?}"),
    }

    println!(
        "reader (started at ts 1) sees alice = {:?}",
        db.get(&reader, "alice")
    );
    println!(
        "a new transaction sees alice = {:?}",
        db.get(&db.begin(), "alice")
    );
    println!("versions of alice stored: {}", db.version_count("alice"));
}
```

```bash
cargo run --bin timeline
```

```text
setup committed at ts 1
t1 committed at ts 2
t2 aborted: Conflict { key: "alice" }
reader (started at ts 1) sees alice = Some("100")
a new transaction sees alice = Some("90")
versions of alice stored: 2
```

t1 and t2 both started at ts 1 and both wrote alice. t1 committed first; t2 found a version newer than its snapshot and
was aborted instead of silently losing t1's update. The old reader still sees 100, and two versions are stored: the
price of MVCC is space.

> 🧪 **Testing:** MVCC bugs show up as "a transaction saw something it should not". Test with explicit timelines:
> start the transactions in a fixed order, commit in a fixed order, and assert what each one sees. Exercise 2 uses this
> to prove a *weakness* of snapshot isolation.

## Common mistakes

> ⚠️ **Common mistake:** thinking that `UPDATE` changes the row in place. In PostgreSQL it writes a new version, so a
> table that is updated often grows until `VACUUM` reclaims the space. Updating every row of a big table at once can
> double its size.

> ⚠️ **Common mistake:** believing snapshot isolation is serializable. It stops lost updates (first committer wins), but
> two transactions that write **different** keys are never in conflict, so write skew passes (Exercise 2). PostgreSQL's
> `SERIALIZABLE` adds SSI (Serializable Snapshot Isolation, Lesson 17.11) on top for this reason.

> ⚠️ **Common mistake:** keeping transactions open while waiting for something slow (a user, or a call over HTTP, the Hypertext Transfer Protocol). Besides
> holding locks, the open snapshot stops cleanup everywhere (Section 4).

## Try it yourself

1. In T1, run `BEGIN; DELETE FROM accounts WHERE name = 'bob';` and do not commit. In T2, select
   `ctid, xmin, xmax, name` from `accounts`: bob is still there, with `xmax` set to T1's XID. Then `ROLLBACK` in T1 and
   select again in T2: `xmax` still shows that XID, but the commit log says it rolled back, so the version is alive.
2. Run `SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'accounts';` after a few updates and
   again after `VACUUM`. (These statistics are collected in the background and may lag a second behind.)

## Exercises

### Exercise 1 — Which version is visible? (easy)

Key `x` has three versions: `"a"` committed at ts 1, a delete committed at ts 4, and `"c"` committed at ts 6. What does
`get` return for transactions with `start_ts` 0, 1, 3, 4, 5 and 6? Answer on paper, then write a test.

<details>
<summary>Hint</summary>

Take the newest version with `commit_ts <= start_ts`. A delete version means "no value". Inside the `tests` module you
may build a `Tx { start_ts, writes: BTreeMap::new() }` directly, because private fields are visible to child modules.

</details>

<details>
<summary>Answer</summary>

0 → none (x did not exist yet), 1 → `"a"`, 3 → `"a"`, 4 → none (the delete), 5 → none, 6 → `"c"`.

```rust
    #[test]
    fn which_version_does_each_snapshot_see() {
        let mut db = Mvcc::default();
        let mut t = db.begin();
        t.put("x", "a");
        db.commit(t).unwrap(); // ts 1
        for _ in 0..2 {
            let mut t = db.begin();
            t.put("other", "-");
            db.commit(t).unwrap(); // ts 2 and 3
        }
        let mut t = db.begin();
        t.delete("x");
        db.commit(t).unwrap(); // ts 4
        let mut t = db.begin();
        t.put("other", "-");
        db.commit(t).unwrap(); // ts 5
        let mut t = db.begin();
        t.put("x", "c");
        db.commit(t).unwrap(); // ts 6

        let read_at = |start_ts| {
            let snapshot = Tx {
                start_ts,
                writes: BTreeMap::new(),
            };
            db.get(&snapshot, "x").map(String::from)
        };
        assert_eq!(read_at(0), None); // before x existed
        assert_eq!(read_at(1), Some("a".to_string()));
        assert_eq!(read_at(3), Some("a".to_string()));
        assert_eq!(read_at(4), None); // the delete is visible
        assert_eq!(read_at(5), None);
        assert_eq!(read_at(6), Some("c".to_string()));
    }
```

Why it works: the snapshot is only a number; the visibility rule "newest version at or before my timestamp" answers
every case, including deletes.

</details>

### Exercise 2 — Prove that snapshot isolation allows write skew (medium)

Write a test `snapshot_isolation_allows_write_skew` with the doctors of Lesson 17.11: ann and ben are "on call"; two
transactions each read the other doctor, see "on call", and set themselves "off". Assert that **both** commits succeed
and that nobody is on call afterwards.

<details>
<summary>Hint</summary>

The keys written are `"ann"` (by t1) and `"ben"` (by t2). `commit` only checks the keys a transaction **writes**.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn snapshot_isolation_allows_write_skew() {
        let mut db = Mvcc::default();
        let mut setup = db.begin();
        setup.put("ann", "on call");
        setup.put("ben", "on call");
        db.commit(setup).unwrap();

        let mut t1 = db.begin();
        let mut t2 = db.begin();
        // both see two doctors on call, so each one may leave
        assert_eq!(db.get(&t1, "ben"), Some("on call"));
        assert_eq!(db.get(&t2, "ann"), Some("on call"));
        t1.put("ann", "off");
        t2.put("ben", "off");
        assert!(db.commit(t1).is_ok());
        assert!(db.commit(t2).is_ok()); // different keys: no conflict detected

        let now = db.begin();
        assert_eq!(db.get(&now, "ann"), Some("off"));
        assert_eq!(db.get(&now, "ben"), Some("off")); // nobody is on call
    }
```

Why it works: first committer wins compares only **write** sets. The rule that was broken depends on what the
transactions **read**, which this store does not track. A test that documents a known limitation is valuable: if
somebody later adds read tracking (like SSI), this test tells them exactly which behaviour changed.

</details>

### Exercise 3 — Vacuum (hard)

Add `pub fn vacuum(&mut self, oldest_start_ts: u64) -> usize`: remove every version that no transaction with
`start_ts >= oldest_start_ts` can see, and return how many were removed. Keep, for each key, the newest version with
`commit_ts <= oldest_start_ts` and everything newer. If what remains is a single delete version that everyone sees,
remove the key completely. Test it with an old reader that keeps an old version alive.

<details>
<summary>Hint</summary>

`versions.iter().rposition(|v| v.commit_ts <= oldest_start_ts)` finds the index of the version the oldest snapshot
sees; everything before that index can go (`versions.drain(..i)`). At the end,
`self.versions.retain(|_, v| !v.is_empty())` drops empty keys.

</details>

<details>
<summary>Answer</summary>

```rust
    /// Removes versions that no snapshot at or after `oldest_start_ts` can see.
    /// Returns how many versions were removed.
    pub fn vacuum(&mut self, oldest_start_ts: u64) -> usize {
        let mut removed = 0;
        for versions in self.versions.values_mut() {
            // the newest version that the oldest snapshot can see
            let keep_from = versions
                .iter()
                .rposition(|v| v.commit_ts <= oldest_start_ts)
                .unwrap_or(0);
            removed += keep_from;
            versions.drain(..keep_from);
            // a delete that every snapshot sees can go, too
            if versions.len() == 1
                && versions[0].value.is_none()
                && versions[0].commit_ts <= oldest_start_ts
            {
                versions.clear();
                removed += 1;
            }
        }
        self.versions.retain(|_, versions| !versions.is_empty());
        removed
    }
```

```rust
    #[test]
    fn vacuum_keeps_what_the_oldest_snapshot_needs() {
        let mut db = store_with("x", "1"); // ts 1
        let old_reader = db.begin(); // start_ts 1
        for value in ["2", "3"] {
            let mut tx = db.begin();
            tx.put("x", value);
            db.commit(tx).unwrap(); // ts 2, 3
        }
        assert_eq!(db.vacuum(old_reader.start_ts), 0); // "1" is still needed
        assert_eq!(db.get(&old_reader, "x"), Some("1"));

        drop(old_reader);
        assert_eq!(db.vacuum(db.begin().start_ts), 2); // only "3" is needed now
        assert_eq!(db.version_count("x"), 1);

        let mut tx = db.begin();
        tx.delete("x");
        db.commit(tx).unwrap(); // ts 4
        assert_eq!(db.vacuum(db.begin().start_ts), 2); // "3" and the delete
        assert_eq!(db.version_count("x"), 0);
    }
```

Why it works: a version can be needed only by snapshots whose `start_ts` is at least `oldest_start_ts`, and all of
them see the version at index `keep_from` or a newer one. The caller must pass the `start_ts` of the oldest **running**
transaction; one forgotten transaction keeps `oldest_start_ts` small and nothing is removed, exactly like the
`idle in transaction` session in Section 4.

</details>

## Quiz

1. Why do readers not block writers under MVCC?

<details>
<summary>Answer</summary>

A writer creates a new version instead of changing the old one, so the reader can keep reading the version that its
snapshot sees. Nothing has to be locked for reading.

</details>

2. What do `xmin` and `xmax` of a PostgreSQL row version mean?

<details>
<summary>Answer</summary>

`xmin` is the ID of the transaction that created the version; `xmax` is the ID of the transaction that deleted or
replaced it (0 if none). Whether those transactions committed is looked up in the commit log.

</details>

3. A snapshot is `500:510:503,507`. Is a version with `xmin = 505` (committed) visible? And one with `xmin = 507`?

<details>
<summary>Answer</summary>

505: yes. It is below 510 and not in the list of running transactions, and it committed. 507: no. It is in the list,
so it was still running when the snapshot was taken.

</details>

4. What does `VACUUM` remove, and what can stop it?

<details>
<summary>Answer</summary>

Dead versions: versions whose deleter committed and that no snapshot can still see. A long-open transaction (or
snapshot) stops it, because versions that died after that snapshot was taken might still be needed.

</details>

5. How does PostgreSQL "undo" a rolled-back transaction without undo information?

<details>
<summary>Answer</summary>

It does not change anything back. The commit log marks the transaction as aborted, so its new versions are never
visible and the versions it "deleted" stay visible. `VACUUM` removes the leftovers later.

</details>

## Summary

- MVCC keeps several versions of each row; a transaction reads the version that its snapshot sees, so readers and
  writers do not block each other.
- PostgreSQL versions carry `xmin` (creator) and `xmax` (deleter); `ctid` is the physical slot; `pageinspect` shows
  every version on a page.
- A snapshot is `xmin:xmax:running list`; a version is visible if its creator committed in the snapshot's past and its
  deleter did not.
- Old versions become dead; `VACUUM` (and autovacuum) reclaims them, but only if no open snapshot still needs them.
- Snapshot isolation = snapshots + first committer wins: it stops lost updates but allows write skew.
- In a toy store, a snapshot can be one number: the last commit timestamp.

## Sources and further reading

- [PostgreSQL: Concurrency Control (introduction)](https://www.postgresql.org/docs/current/mvcc-intro.html) — MVCC in PostgreSQL's own words.
- [PostgreSQL: System Columns](https://www.postgresql.org/docs/current/ddl-system-columns.html) — `xmin`, `xmax`, `ctid`.
- [PostgreSQL: System Information Functions](https://www.postgresql.org/docs/current/functions-info.html) — `pg_current_xact_id` and `pg_current_snapshot`, with the snapshot format.
- [PostgreSQL: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html) — why vacuum is needed, autovacuum, transaction ID wraparound.
- [PostgreSQL: pageinspect](https://www.postgresql.org/docs/current/pageinspect.html) and [Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html) — the tools and the page format used in Section 4.
- [PostgreSQL: Heap-Only Tuples (HOT)](https://www.postgresql.org/docs/current/storage-hot.html) — why slot 1 became a redirect.

---

[🏠 Start](../START-HERE.md) · [← 17.11 Isolation levels](11-isolation-levels.md) · [📋 Phase 17](README.md) · [17.13 How a SQL query runs →](13-how-a-sql-query-runs.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
