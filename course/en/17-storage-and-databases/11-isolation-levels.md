<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 17.10 Transactions and ACID](10-transactions-and-acid.md) · [📋 Phase 17](README.md) · [17.12 MVCC: multi-version concurrency control →](12-mvcc.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/17-storage-and-databases/11-isolation-levels.md) <!-- nav -->

# Lesson 17.11 — Isolation levels

> **Phase 17 — Storage and database internals** · Lesson 11 of 15 · ⏱ about 120 minutes

## What you will learn

- The five classic anomalies: dirty read, non-repeatable read, phantom read, lost update and write skew.
- The four isolation levels of the SQL (Structured Query Language, the language of relational databases) standard, and what PostgreSQL really does at each of them.
- How to make every anomaly happen with two `psql` terminals, and how a stronger level stops it.
- Three ways to avoid lost updates: an atomic `UPDATE`, `SELECT ... FOR UPDATE`, and `SERIALIZABLE` with retries.
- How to retry a serialization failure (error code `40001`) correctly in Rust with sqlx.

## Before you start

- You finished [Lesson 17.10 — Transactions and ACID](10-transactions-and-acid.md) and saw a lost update there.
- You know race conditions from [Lesson 14.1 — Why concurrency is hard: races and deadlocks](../14-concurrency-and-async/01-why-concurrency-is-hard.md)
  and sqlx from [Lesson 16.6 — Databases with sqlx](../16-backend-services/06-databases-with-sqlx.md).
- The container `p17-postgres` from [Lesson 17.9](09-page-cache-and-buffer-pool.md) is running
  (`docker start p17-postgres` if it is stopped).

## New words

| Word | Meaning |
|---|---|
| **anomaly** | A result that could never happen if the transactions ran one after another. |
| **serial execution** | Running transactions one at a time, with no overlap. Always correct, but slow. |
| **serializable** | A way of running transactions concurrently whose result is the same as *some* serial order. |
| **isolation level** | A setting that says which anomalies a transaction accepts in exchange for speed. |
| **dirty read** | Reading a change of another transaction that has not committed (and may still roll back). |
| **non-repeatable read** | Reading the same row twice in one transaction and getting two different values, because another transaction committed in between. |
| **phantom read** | Running the same `WHERE` query twice and getting a different *set* of rows, because another transaction inserted or deleted rows. |
| **write skew** | Two transactions read the same data, each makes a decision from it and changes *different* rows; together they break a rule that each alone kept. |
| **snapshot** | A frozen picture of the committed data at one moment. A transaction that reads from a snapshot does not see later commits. |
| **serialization failure** | The database aborts a transaction because letting it commit would produce an anomaly. PostgreSQL reports it with code `40001`; the fix is to retry. |
| **SQLSTATE** | A five-character error code defined by the SQL standard, for example `40001` (serialization failure) or `23505` (unique violation). |
| **row lock** | A lock on one row: other transactions that want to change (or lock) the same row wait until the holder commits or rolls back. |
| **SSI** (Serializable Snapshot Isolation) | PostgreSQL's method for `SERIALIZABLE`: snapshots plus tracking of read/write dependencies, aborting a transaction when a dangerous pattern appears. |

## 1. Why not run transactions one at a time?

The simplest correct database runs one transaction, then the next: **serial execution**. Nothing can overlap, so no
anomaly can happen. But one slow transaction (a report that reads for a minute) would block every other client.

Real databases run many transactions at once and give you a choice, the **isolation level**: how much the transactions
may "see" of each other. The strongest level, **serializable**, promises that the result is the same as *some* serial
order. Weaker levels are faster and allow certain anomalies. To choose a level, you must know the anomalies.

## 2. The anomalies

Each anomaly is a timeline of two transactions, T1 and T2. Time goes down.

```text
dirty read                         non-repeatable read                phantom read
T1                T2               T1                T2               T1                       T2
                  set x = 0        read x -> 100                      count(on_call) -> 2
read x -> 0  (!)                                     set x = 70                                 insert on_call
                  ROLLBACK                           COMMIT                                     COMMIT
                                   read x -> 70 (!)                   count(on_call) -> 3 (!)

lost update                        write skew (rule: at least one doctor on call)
T1                T2               T1                          T2
read x -> 100                      count(on_call) -> 2
                  read x -> 100                                count(on_call) -> 2
write x = 90                       ann: on_call = false
COMMIT                                                         ben: on_call = false
                  write x = 90     COMMIT                      COMMIT
                  COMMIT  (!)      -> nobody is on call (!)
```

- **Dirty read**: T1 saw a value that never officially existed.
- **Non-repeatable read**: the same row read twice gives two answers inside one transaction.
- **Phantom read**: the same query returns a different set of rows (a "phantom" row appeared).
- **Lost update**: both read 100 and took 10; the result is 90 instead of 80. You built this in Lesson 17.10.
- **Write skew**: each transaction checked "2 doctors are on call, so I may leave". Each change alone was fine;
  together they broke the rule. No row was written twice, which makes write skew hard to notice.

## 3. Isolation levels: the standard and PostgreSQL

The SQL standard defines four levels by the anomalies they forbid (the standard's table lists only the first three):

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| `READ UNCOMMITTED` | possible | possible | possible |
| `READ COMMITTED` | no | possible | possible |
| `REPEATABLE READ` | no | no | possible |
| `SERIALIZABLE` | no | no | no |

The standard says what a level **must forbid**; a database may forbid more. PostgreSQL does:

| You ask for | PostgreSQL gives you | Every statement sees | Lost update | Write skew |
|---|---|---|---|---|
| `READ UNCOMMITTED` | the same as `READ COMMITTED` | — | — | — |
| `READ COMMITTED` (default) | a new snapshot for **each statement** | data committed before the statement began | possible (read, then write) | possible |
| `REPEATABLE READ` | one snapshot for the **whole transaction** (snapshot isolation) | data committed before the first statement | error `40001` | possible |
| `SERIALIZABLE` | snapshot isolation + SSI | as `REPEATABLE READ` | error `40001` | error `40001` |

So in PostgreSQL a dirty read is never possible, and `REPEATABLE READ` also prevents phantoms. How snapshots work inside
is the topic of Lesson 17.12; here we only use them.

## 4. See the anomalies in PostgreSQL

Open **two** terminals and run `docker exec -it p17-postgres psql -U postgres` in each. We call them T1 and T2. In
either one, create two small tables:

```sql
CREATE TABLE accounts (name text PRIMARY KEY, balance int NOT NULL);
INSERT INTO accounts VALUES ('alice', 100), ('bob', 50);
CREATE TABLE doctors (name text PRIMARY KEY, on_call boolean NOT NULL);
INSERT INTO doctors VALUES ('ann', true), ('ben', true);
```

Now type the statements below in the order of the step numbers, each in its terminal. The arrow shows what `psql`
printed. A statement typed outside `BEGIN ... COMMIT` is its own small transaction that commits at once.

**Non-repeatable read at `READ COMMITTED`, and how `REPEATABLE READ` stops it:**

```text
step  T1                                                        T2
1     BEGIN;
2     SELECT balance FROM accounts WHERE name = 'alice';  -> 100
3                                                               UPDATE accounts SET balance = 70 WHERE name = 'alice';
4     SELECT balance FROM accounts WHERE name = 'alice';  -> 70
5     COMMIT;
6                                                               UPDATE accounts SET balance = 100 WHERE name = 'alice';
7     BEGIN ISOLATION LEVEL REPEATABLE READ;
8     SELECT balance FROM accounts WHERE name = 'alice';  -> 100
9                                                               UPDATE accounts SET balance = 70 WHERE name = 'alice';
10    SELECT balance FROM accounts WHERE name = 'alice';  -> 100
11    COMMIT;
12    SELECT balance FROM accounts WHERE name = 'alice';  -> 70
13                                                              UPDATE accounts SET balance = 100 WHERE name = 'alice';
```

At `READ COMMITTED` each statement takes a fresh snapshot, so step 4 sees T2's commit. At `REPEATABLE READ` the
snapshot is taken at the first statement (step 8) and kept, so step 10 still sees 100. After `COMMIT`, step 12 sees the
new value.

**No dirty read, even when you ask for it:**

```text
step  T1                                                        T2
1                                                               BEGIN;
2                                                               UPDATE accounts SET balance = 0 WHERE name = 'alice';
3     BEGIN ISOLATION LEVEL READ UNCOMMITTED;
4     SELECT balance FROM accounts WHERE name = 'alice';  -> 100
5                                                               ROLLBACK;
6     COMMIT;
```

**Lost update at `READ COMMITTED`:** both read 100, both write 90, both commit; afterwards
`SELECT balance FROM accounts WHERE name = 'alice'` prints 90. Ten are lost, exactly as in Lesson 17.10. Reset alice to
100 and try the same at `REPEATABLE READ`:

```text
step  T1                                                        T2
1     BEGIN ISOLATION LEVEL REPEATABLE READ;
2     SELECT balance FROM accounts WHERE name = 'alice';  -> 100
3                                                               BEGIN ISOLATION LEVEL REPEATABLE READ;
4                                                               SELECT balance FROM accounts WHERE name = 'alice';  -> 100
5     UPDATE accounts SET balance = 90 WHERE name = 'alice';
6     COMMIT;
7                                                               UPDATE accounts SET balance = 90 WHERE name = 'alice';
8                                                               ROLLBACK;
```

Step 7 fails in T2:

```text
ERROR:  could not serialize access due to concurrent update
```

T2 wanted to change a row that was changed by a transaction that committed after T2's snapshot. Instead of silently
losing T1's update, PostgreSQL aborts T2. Its application must start the whole transaction again (read again, decide
again, write again).

**Write skew at `REPEATABLE READ`, stopped by `SERIALIZABLE`:** run this twice, first with `REPEATABLE READ` in steps 1
and 2, then (after `UPDATE doctors SET on_call = true;`) with `SERIALIZABLE`:

```text
step  T1                                                        T2
1     BEGIN ISOLATION LEVEL REPEATABLE READ;
2                                                               BEGIN ISOLATION LEVEL REPEATABLE READ;
3     SELECT count(*) FROM doctors WHERE on_call;  -> 2
4                                                               SELECT count(*) FROM doctors WHERE on_call;  -> 2
5     UPDATE doctors SET on_call = false WHERE name = 'ann';
6                                                               UPDATE doctors SET on_call = false WHERE name = 'ben';
7     COMMIT;
8                                                               COMMIT;
```

With `REPEATABLE READ` both commits succeed, and `SELECT * FROM doctors;` shows `ann | f` and `ben | f`: nobody is on
call. The two transactions changed different rows, so there was no conflict for snapshot isolation to detect. With
`SERIALIZABLE`, step 8 fails:

```text
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

SSI noticed the pattern: T1 read rows that T2 changed, and T2 read rows that T1 changed. No serial order can produce
"both left", so one of them must go. Ann's change committed, Ben is still on call. Read the `HINT`: retrying is the
expected reaction.

> 💡 **Tip:** `SHOW transaction_isolation;` prints the level of the current transaction. To change the default for one
> session: `SET default_transaction_isolation = 'serializable';`.

## 5. Three ways to avoid lost updates

1. **Let the database do the arithmetic.** `UPDATE accounts SET balance = balance - 10 WHERE name = 'alice'` reads and
   writes the row in **one** statement, under a **row lock**. If two transactions run it at the same time, the second
   one waits for the first to commit, then re-reads the row and subtracts again. We ran it from two terminals: alice
   went from 100 to 80, at the default `READ COMMITTED`. This is the best tool when the change is simple arithmetic.
2. **Lock what you read.** `SELECT ... FOR UPDATE` takes the row locks already while reading. A second transaction that
   runs the same `SELECT ... FOR UPDATE` waits until the first one commits, and then sees the new data. Exercise 4 fixes
   the doctors with it.
3. **Use `SERIALIZABLE` and retry.** You write plain read-then-write code, and the database aborts the transactions
   that would cause an anomaly. This protects even against mistakes you did not think of, but every transaction must be
   ready to run again.

> 🔒 **Security:** many real bugs in shops and banks are lost updates and write skews that attackers trigger on purpose:
> they send twenty requests in the same millisecond to use one coupon twenty times. Tests that send one request at a
> time never find these bugs. Section 6 is the kind of test that does.

## 6. Retrying in Rust

Now the lost update with real concurrency: 8 Tokio tasks each add 1 to a counter 50 times, by reading the value and
writing value + 1 in a transaction. First at `READ COMMITTED`, then at `SERIALIZABLE` with retries.

```bash
cd ~/projects/rust
cargo new code/p17-isolation
cd code/p17-isolation
cargo add sqlx@0.9 --features runtime-tokio,postgres
cargo add tokio@1 --features macros,rt-multi-thread
```

Replace `src/main.rs` with:

```rust
use sqlx::PgPool;
use sqlx::postgres::PgPoolOptions;

const TASKS: u32 = 8;
const STEPS: u32 = 50;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let url = std::env::var("DATABASE_URL")
        .unwrap_or("postgres://postgres:dev-password@127.0.0.1:5432/postgres".into());
    let pool = PgPoolOptions::new()
        .max_connections(8)
        .connect(&url)
        .await?;
    sqlx::query("CREATE TABLE IF NOT EXISTS counter (id int PRIMARY KEY, n bigint NOT NULL)")
        .execute(&pool)
        .await?;

    for level in ["READ COMMITTED", "SERIALIZABLE"] {
        sqlx::query("INSERT INTO counter VALUES (1, 0) ON CONFLICT (id) DO UPDATE SET n = 0")
            .execute(&pool)
            .await?;
        let mut handles = Vec::new();
        for _ in 0..TASKS {
            let pool = pool.clone();
            handles.push(tokio::spawn(async move {
                let mut retries = 0;
                for _ in 0..STEPS {
                    // try again until the transaction commits
                    while let Err(e) = increment(&pool, level).await {
                        if !is_serialization_failure(&e) {
                            return Err(e);
                        }
                        retries += 1;
                    }
                }
                Ok::<u32, sqlx::Error>(retries)
            }));
        }
        let mut retries = 0;
        for handle in handles {
            retries += handle.await.expect("task panicked")?;
        }
        let n: i64 = sqlx::query_scalar("SELECT n FROM counter WHERE id = 1")
            .fetch_one(&pool)
            .await?;
        println!(
            "{level:<15} counter = {n} (expected {}), retries = {retries}",
            TASKS * STEPS
        );
    }
    Ok(())
}

/// Reads the counter, then writes counter + 1, in one transaction.
async fn increment(pool: &PgPool, level: &str) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;
    let set_level = match level {
        "SERIALIZABLE" => "SET TRANSACTION ISOLATION LEVEL SERIALIZABLE",
        _ => "SET TRANSACTION ISOLATION LEVEL READ COMMITTED",
    };
    sqlx::query(set_level).execute(&mut *tx).await?;
    let n: i64 = sqlx::query_scalar("SELECT n FROM counter WHERE id = 1")
        .fetch_one(&mut *tx)
        .await?;
    sqlx::query("UPDATE counter SET n = $1 WHERE id = 1")
        .bind(n + 1)
        .execute(&mut *tx)
        .await?;
    tx.commit().await
}

/// PostgreSQL's code 40001 means "serialization failure: please retry".
fn is_serialization_failure(e: &sqlx::Error) -> bool {
    matches!(e, sqlx::Error::Database(db) if db.code().as_deref() == Some("40001"))
}
```

```bash
cargo run
```

```text
READ COMMITTED  counter = 86 (expected 400), retries = 0
SERIALIZABLE    counter = 400 (expected 400), retries = 2443
```

Your numbers will be different on every run (another run printed 84 and 2325), because they depend on the timing of
the tasks.

**What happens:**

1. The program connects with the same URL (Uniform Resource Locator, here the address of the database) style as Lesson 16.6 (set `DATABASE_URL` to use another database). It creates
   `counter` and resets it to 0 before each level.
2. `increment` starts a transaction, sets its isolation level (it must be the first statement of the transaction),
   reads `n`, writes `n + 1` and commits.
3. At `READ COMMITTED`, eight tasks read the same `n` again and again and overwrite each other: 314 of 400 increments
   are lost, and no error is reported. This is the dangerous part: **wrong data, no error**.
4. At `SERIALIZABLE`, PostgreSQL aborts a transaction whenever another one changed `n` after its snapshot. The error is
   `sqlx::Error::Database`, and `db.code()` returns the SQLSTATE `"40001"`. The `while let` loop runs the **whole**
   transaction again. The counter is exact.
5. The price is 2443 retries: eight tasks fighting over one row is the worst case for optimistic methods. For a hot
   counter, the atomic `UPDATE` of Section 5 is much better (Exercise 2).

> 🧪 **Testing:** a test for concurrency bugs must create real concurrency: many tasks, many repetitions, a real
> database. Exercise 3 turns this program into a test that fails if a single increment is lost.

## Common mistakes

> ⚠️ **Common mistake:** building SQL with `format!`. sqlx 0.9 refuses a dynamic string at compile time:
>
> ```text
> error[E0277]: dynamic SQL strings should be audited for possible injections
>    --> src/main.rs:62:17
>     |
>  62 |     sqlx::query(&sql).execute(&mut *tx).await?;
>     |     ----------- ^^^^ dynamic SQL string
>     |     |
>     |     required by a bound introduced by this call
>     |
>     = help: the trait `SqlSafeStr` is not implemented for `&std::string::String`
> ```
>
> Use literal SQL with `$1` parameters for values. An isolation level cannot be a parameter, so the program chooses
> between two literal strings with `match`.

> ⚠️ **Common mistake:** retrying only the failed statement instead of the whole transaction. After an error the
> transaction is aborted, and its earlier reads are stale anyway. Start again from `begin`.

> ⚠️ **Common mistake:** retrying **every** error. Retry only serialization failures (`40001`) and deadlocks (`40P01`).
> A syntax error or a broken constraint will fail again forever; return it.

## Try it yourself

1. In the `REPEATABLE READ` example of Section 4, let T2 insert a third doctor (`INSERT INTO doctors VALUES ('cat',
   true);`) between T1's two `SELECT count(*)`. T1 still counts 2: snapshot isolation also hides phantoms.
2. Change `TASKS` to 1 and run the program. Both levels now print 50 and 0 retries: without concurrency there is no
   anomaly, which is why such bugs hide during development.

## Exercises

### Exercise 1 — Name the anomaly (easy)

Which anomaly is each story, and what is the weakest PostgreSQL level that prevents it?

1. A report sums all balances twice in one transaction and gets two different totals, because a transfer committed in
   between.
2. Two users book a meeting room for 10:00: both check that `bookings` has no row for that room and hour, then each
   inserts its own booking row.
3. Two admins rename the same product at the same time; one name is lost.

<details>
<summary>Hint</summary>

Ask: did the same rows change between two reads (non-repeatable / phantom)? Did both write the same row (lost update)?
Did they write different rows after reading the same data (write skew)?

</details>

<details>
<summary>Answer</summary>

1. Non-repeatable read (or phantom read, if rows appeared): `REPEATABLE READ`.
2. Write skew (here with a phantom: the rows they checked did not exist yet): both read "no booking" and inserted
   different rows. Only `SERIALIZABLE` (or a `UNIQUE` constraint on room and hour) prevents it.
3. Lost update, if they read first and then write: `REPEATABLE READ` turns it into a `40001` error; a plain
   `UPDATE products SET name = $1 WHERE id = $2` without a read is not a lost update at all, just "last writer wins".

Why it works: the anomaly is defined by *what each transaction read and wrote*, not by the business story.

</details>

### Exercise 2 — The atomic increment (medium)

Add a third mode `"ATOMIC"` to the program: `increment` returns `increment_atomic(pool).await` for it, and
`increment_atomic` runs `UPDATE counter SET n = n + 1 WHERE id = 1` without an explicit transaction. Run all three.

<details>
<summary>Hint</summary>

Add `"ATOMIC"` to the array in `main`, and check `if level == "ATOMIC"` at the start of `increment`, before `begin`.

</details>

<details>
<summary>Answer</summary>

```rust
/// One statement: the database reads and writes the row under a row lock.
async fn increment_atomic(pool: &PgPool) -> Result<(), sqlx::Error> {
    sqlx::query("UPDATE counter SET n = n + 1 WHERE id = 1")
        .execute(pool)
        .await?;
    Ok(())
}
```

```text
READ COMMITTED  counter = 80 (expected 400), retries = 0
SERIALIZABLE    counter = 400 (expected 400), retries = 1553
ATOMIC          counter = 400 (expected 400), retries = 0
```

Why it works: the read and the write happen inside one statement while the row is locked, so no other transaction can
slip in between. Waiting for a row lock is much cheaper than throwing a transaction away and running it again.

</details>

### Exercise 3 — A test that catches lost increments (medium)

Write `#[tokio::test] async fn serializable_with_retry_loses_no_increment()` in `src/main.rs`: 4 tasks × 25 increments
at `SERIALIZABLE` with retries; assert that the counter is 100. Run it with `cargo test` while `p17-postgres` runs.

<details>
<summary>Hint</summary>

`#[tokio::test]` is like `#[test]`, but starts a Tokio runtime for an `async fn`. Inside the retry loop, `assert!` that
every error is a serialization failure, so other errors fail the test instead of looping forever.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn serializable_with_retry_loses_no_increment() {
        let url = std::env::var("DATABASE_URL")
            .unwrap_or("postgres://postgres:dev-password@127.0.0.1:5432/postgres".into());
        let pool = PgPool::connect(&url).await.unwrap();
        sqlx::query("CREATE TABLE IF NOT EXISTS counter (id int PRIMARY KEY, n bigint NOT NULL)")
            .execute(&pool)
            .await
            .unwrap();
        sqlx::query("INSERT INTO counter VALUES (1, 0) ON CONFLICT (id) DO UPDATE SET n = 0")
            .execute(&pool)
            .await
            .unwrap();

        let mut handles = Vec::new();
        for _ in 0..4 {
            let pool = pool.clone();
            handles.push(tokio::spawn(async move {
                for _ in 0..25 {
                    while let Err(e) = increment(&pool, "SERIALIZABLE").await {
                        assert!(is_serialization_failure(&e), "unexpected error: {e}");
                    }
                }
            }));
        }
        for handle in handles {
            handle.await.unwrap();
        }
        let n: i64 = sqlx::query_scalar("SELECT n FROM counter WHERE id = 1")
            .fetch_one(&pool)
            .await
            .unwrap();
        assert_eq!(n, 100);
    }
}
```

```text
running 1 test
test tests::serializable_with_retry_loses_no_increment ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.49s
```

Why it works: the test creates real contention on one row. Replace `"SERIALIZABLE"` with `"READ COMMITTED"` and it
fails (`left: 68`, `right: 100` in our run): the test can see the bug. (It shares the `counter` table with `main`, so do not run
both at the same moment.)

</details>

### Exercise 4 — Fix the doctors with `FOR UPDATE` (hard)

At the default `READ COMMITTED`, make the doctors example safe without `SERIALIZABLE`: each transaction first runs
`SELECT name FROM doctors WHERE on_call FOR UPDATE;`, counts the rows itself, and leaves only if there are at least 2.
Run it from two terminals (reset with `UPDATE doctors SET on_call = true;`). What does T2 see, and when?

<details>
<summary>Hint</summary>

T2's `SELECT ... FOR UPDATE` wants rows that T1 has locked, so T2's terminal "hangs" until T1 commits.

</details>

<details>
<summary>Answer</summary>

```text
step  T1                                                        T2
1     BEGIN;
2                                                               BEGIN;
3     SELECT name FROM doctors WHERE on_call FOR UPDATE;  -> ben, ann (2 rows)
4                                                               SELECT name FROM doctors WHERE on_call FOR UPDATE;  (waits)
5     UPDATE doctors SET on_call = false WHERE name = 'ann';
6     COMMIT;                                                   -> ben (1 row)
7                                                               COMMIT;   (1 row: Ben stays)
```

Why it works: T1 locked both rows it read. T2 waited for those locks; when T1 committed, T2 re-checked the rows,
found that Ann is no longer on call and returned only Ben. Seeing 1 doctor, T2 decides not to leave, and the rule
holds. The price: T2 had to wait. (At `REPEATABLE READ`, T2 would get the error `could not serialize access due to
concurrent update` instead of waiting and re-checking.)

</details>

## Quiz

1. What does "serializable" promise?

<details>
<summary>Answer</summary>

That the result of the concurrent transactions is the same as if they had run one after another in *some* order.

</details>

2. What is the difference between `READ COMMITTED` and `REPEATABLE READ` in PostgreSQL?

<details>
<summary>Answer</summary>

`READ COMMITTED` takes a new snapshot for every statement, so a transaction can see commits that happened during it.
`REPEATABLE READ` takes one snapshot at the first statement and uses it for the whole transaction.

</details>

3. Why is write skew not stopped by `REPEATABLE READ`?

<details>
<summary>Answer</summary>

The two transactions write **different** rows, so there is no write-write conflict to detect. The problem is that each
decision was based on data the other one changed; only `SERIALIZABLE` (SSI) tracks those read/write dependencies.

</details>

4. Your code gets SQLSTATE `40001`. What should it do, and what should it not do?

<details>
<summary>Answer</summary>

Run the whole transaction again from the beginning, maybe a few times with a short pause. It should not retry only the
failed statement, and it should not retry other kinds of errors.

</details>

5. Which is the best fix for a hot counter, and why?

<details>
<summary>Answer</summary>

An atomic `UPDATE ... SET n = n + 1`: the read and the write happen in one statement under a row lock, so nothing is
lost and nothing has to be retried.

</details>

## Summary

- Isolation is a trade-off: serial execution is correct but slow; weaker levels allow anomalies for speed.
- Anomalies: dirty read, non-repeatable read, phantom read, lost update and write skew.
- PostgreSQL: `READ UNCOMMITTED` = `READ COMMITTED` (snapshot per statement, the default); `REPEATABLE READ` = one
  snapshot per transaction, lost updates become `40001`; `SERIALIZABLE` also stops write skew with SSI.
- Avoid lost updates with atomic `UPDATE`s, `SELECT ... FOR UPDATE`, or `SERIALIZABLE` plus retries.
- Retry the **whole** transaction, and only on `40001` (and deadlocks, `40P01`).
- Concurrency bugs give wrong data without errors; only tests with real concurrency find them.

## Sources and further reading

- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) — the exact behaviour of every level, with examples.
- [PostgreSQL: Serialization Failure Handling](https://www.postgresql.org/docs/current/mvcc-serialization-failure-handling.html) — which errors to retry, and how.
- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html) — row locks and `FOR UPDATE`.
- [PostgreSQL: SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html) — how to choose a level.
- [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/) — the classic paper that showed the gaps in the standard's definitions (ANSI, the American National Standards Institute, publishes the SQL standard in the United States).
- [Jepsen: Consistency Models](https://jepsen.io/consistency) — a map of isolation and consistency models with clear definitions.
- [sqlx documentation](https://docs.rs/sqlx/latest/sqlx/) — transactions and database errors in sqlx.

---

[🏠 Start](../START-HERE.md) · [← 17.10 Transactions and ACID](10-transactions-and-acid.md) · [📋 Phase 17](README.md) · [17.12 MVCC: multi-version concurrency control →](12-mvcc.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
