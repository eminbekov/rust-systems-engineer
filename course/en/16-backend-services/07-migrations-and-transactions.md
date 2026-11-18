<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.6 Databases with sqlx](06-databases-with-sqlx.md) · [📋 Phase 16](README.md) · [16.8 Authentication in a web service →](08-authentication-in-a-web-service.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/07-migrations-and-transactions.md) <!-- nav -->

# Lesson 16.7 — Migrations and transactions

> **Phase 16 — Backend services** · Lesson 7 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a database migration is, and how to create, run and inspect migrations with `sqlx-cli`.
- Why you must never change a migration that has already run.
- How the service runs its own migrations when it starts, and how `#[sqlx::test]` uses them.
- What a transaction is: several changes that happen completely or not at all.
- How to use a transaction in Rust with sqlx (`begin`, `commit`, and rollback when the transaction is dropped), and how to test that it really rolls back.

## Before you start

- You finished [Lesson 16.6 — Databases with sqlx](06-databases-with-sqlx.md): `code/p16-tasks` uses PostgreSQL,
  and `sqlx-cli` is installed (`cargo sqlx --version` works).
- The container `p16-postgres` is running.

## New words

| Word | Meaning |
|---|---|
| **schema** | The structure of a database: its tables, columns, types, constraints and indexes (not the data in them). |
| **migration** | One numbered SQL (Structured Query Language) file that changes the schema one step, for example "create the table `tasks`". Migrations run in order, and each runs once per database. |
| **schema drift** | When two databases that should have the same schema (your laptop, the test server, production) slowly become different. Migrations prevent it. |
| **checksum** | A short fingerprint computed from a file's bytes. If one byte changes, the checksum changes. |
| **transaction** | A group of SQL statements that the database treats as one unit: either all of their changes are kept, or none. |
| **commit / rollback** | Ending a transaction by keeping all its changes (commit) or throwing them all away (rollback). |
| **atomic** | "Cannot be split": an atomic operation is never seen half done. |
| **ACID** (Atomicity, Consistency, Isolation, Durability) | The four guarantees of database transactions. Lesson 17.10 explains each one in depth. |

## 1. Why migrations?

In Lesson 16.5 you typed `CREATE TABLE` in `psql`, and in Lesson 16.6 the tests ran `schema.sql`. That works once.
Now imagine the service is in production with a million tasks, and you need a new column. You cannot drop the
database and run `schema.sql` again. You need a **change**: `ALTER TABLE tasks ADD COLUMN ...`. And that change must
run on your laptop, on every colleague's laptop, in the tests, and in production, each exactly once, in the right
order.

A **migration** is one such change in its own file. The files are numbered (sqlx uses a timestamp), and the database
remembers which ones it has already run, in a table called `_sqlx_migrations`:

```text
 migrations/                                   database: _sqlx_migrations
 ├── 20261118073832_create_tasks.sql     ──▶   20261118073832 create tasks           ✔ applied
 ├── 20261118073943_create_archived_...  ──▶   20261118073943 create archived tasks  ✔ applied
 └── 20261120090000_add_due_date.sql     ──▶   (not yet)  → `migrate run` applies only this one
```

Running migrations is always safe: already applied files are skipped, new ones are applied in order. Every database
built from the same folder ends with the same schema. The migrations are code: they live in Git and are reviewed
like code.

## 2. Your first migrations

Start from an empty database. `sqlx-cli` reads `DATABASE_URL` from `.env`, like the macros:

```bash
cd ~/projects/rust/code/p16-tasks
cargo sqlx database drop -y
cargo sqlx database create
cargo sqlx migrate add create_tasks
```

```text
Creating migrations/20261118073832_create_tasks.sql
```

**What happens:**

1. `database drop -y` deletes the database `tasks` with everything in it (`-y` answers "yes" to the question).
   This is fine on your laptop, and it is the last time you will build this database by hand.
2. `database create` creates it again, empty.
3. `migrate add create_tasks` creates the folder `migrations` and an empty file whose name starts with the current
   date and time. Yours has a different number; the order of the numbers is the order of the migrations.

Put the tables of Lesson 16.5 into the file (it is the content of `schema.sql` plus the index):

```sql
-- The tasks and their comments (the tables of Lesson 16.5).
CREATE TABLE tasks (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title      TEXT NOT NULL CHECK (char_length(title) BETWEEN 1 AND 200),
    done       BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE comments (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    task_id BIGINT NOT NULL REFERENCES tasks (id) ON DELETE CASCADE,
    text    TEXT NOT NULL
);

CREATE INDEX comments_task_id_idx ON comments (task_id);
```

Delete `schema.sql`: the migration replaces it. Now look at the state, apply, and look again:

```bash
cargo sqlx migrate info
cargo sqlx migrate run
cargo sqlx migrate info
```

```text
20261118073832/pending create tasks
Applied 20261118073832/migrate create tasks (6.512875ms)
20261118073832/installed create tasks
```

`pending` means "in the folder, not yet in this database"; `installed` means "applied". In `psql` you can see the
bookkeeping table next to your tables:

```sql
SELECT version, description, success FROM _sqlx_migrations;
```

```text
    version     | description  | success
----------------+--------------+---------
 20261118073832 | create tasks | t
(1 row)
```

Now the second migration, a table for the archive that Section 6 needs:

```bash
cargo sqlx migrate add create_archived_tasks
```

```sql
-- Done tasks are moved here by POST /tasks/archive-done.
CREATE TABLE archived_tasks (
    id          BIGINT PRIMARY KEY,
    title       TEXT NOT NULL,
    archived_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```bash
cargo sqlx migrate run
```

```text
Applied 20261118073943/migrate create archived tasks (5.881666ms)
```

Only the new file ran. `archived_tasks.id` is a plain `BIGINT PRIMARY KEY`, not `GENERATED`: an archived task keeps
the identifier it had.

> 📏 **Convention:** one migration per change, with a name that says what it does (`add_due_date_to_tasks`,
> `create_users`). Keep the schema change and the code that needs it in the same commit.

## 3. Never edit an applied migration

What if you notice a mistake in a migration that has already run? Say you change `BETWEEN 1 AND 200` to `300` in the
first file and run again:

```bash
cargo sqlx migrate run
cargo sqlx migrate info
```

```text
error: migration 20261118073832 was previously applied but has been modified
20261118073832/installed (different checksum) create tasks
applied migration had checksum b7e8b06a9283d75509f4e7e3a47c6d36fd7e2d257d77c1b51368bed55593f42ef53a9fa365ff18ab43f7b67f7a4e180c
local migration has checksum 9ad74d7a90b957610d322c84e431900cbb89a963873d01b1bf99ef8dfe9b49cd34aaa7de419ace0c0548c22d8ee832b4
```

sqlx stored a **checksum** of every file it applied. The file changed, so the checksum is different, and sqlx refuses
to continue. That is a protection: the edited file would never run again on databases that already had the old version,
so your laptop and production would silently drift apart.

The rule: **an applied migration is history; never change it.** To change the schema, add a **new** migration
(`ALTER TABLE tasks DROP CONSTRAINT ...; ALTER TABLE tasks ADD CONSTRAINT ...`). Undo your edit now (back to `200`),
and `migrate run` is happy again.

> 💡 **Tip:** a migration you have **not** yet pushed, that ran only on your laptop, can still be fixed: undo it with
> `cargo sqlx database reset -y` (drop, create, run all migrations) after editing. Once others may have run it, write a
> new migration instead.

## 4. Migrations when the service starts

Running `cargo sqlx migrate run` by hand before every start is easy to forget. sqlx can put the migration files
**into the program**: the macro `sqlx::migrate!()` reads the folder `migrations` at compile time. Add three lines to
`main`, right after the pool is created:

```rust
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&database_url)
        .await
        .expect("cannot connect to the database");
    sqlx::migrate!()
        .run(&pool)
        .await
        .expect("cannot run the migrations");
```

At every start, the service applies the migrations that are missing and skips the rest. If something is wrong (a
modified file, a failing SQL statement), the service refuses to start, which is much better than running against the
wrong schema.

`#[sqlx::test]` uses the same folder: when `migrations` exists, every test database gets all migrations before the
test starts. So the helper in the tests becomes trivial:

```rust
    /// `#[sqlx::test]` has already run the migrations in the fresh test database.
    async fn test_app(pool: PgPool) -> Router {
        app(AppState { pool })
    }
```

Now the tests use exactly the schema that production uses, and nobody has to keep `schema.sql` in sync.

> ⚠️ **Common mistake:** with several instances, all of them run `migrate!` at start at the same moment. sqlx takes a
> lock in PostgreSQL so only one applies the migrations, and the others wait. Still, a migration that changes a huge
> table can lock it for minutes; large teams run such migrations as a separate step before deploying.

## 5. Transactions

Some changes need **several** statements. The archive is a good example: "move every done task to `archived_tasks`"
is a `DELETE` from one table **and** an `INSERT` into another. If the program crashes or a statement fails between
them, tasks are lost (deleted but not archived) or duplicated. The change must be **atomic**: all of it or nothing.

A **transaction** gives exactly that. Try it in `psql`. `BEGIN` starts a transaction, `ROLLBACK` throws its changes away:

```sql
BEGIN;
INSERT INTO tasks (title) VALUES ('Inside a transaction') RETURNING id;
SELECT count(*) FROM tasks;
ROLLBACK;
SELECT count(*) FROM tasks;
```

```text
BEGIN
 id
----
  1
(1 row)

INSERT 0 1
 count
-------
     1
(1 row)

ROLLBACK
 count
-------
     0
(1 row)
```

Inside the transaction the new row exists; after `ROLLBACK` it never existed. (Its `id`, 1, is used up anyway:
another gap.) `COMMIT` instead of `ROLLBACK` would have kept it.

**An error inside a transaction** spoils the whole transaction:

```sql
BEGIN;
INSERT INTO tasks (title) VALUES ('First');
INSERT INTO tasks (title) VALUES ('');
INSERT INTO tasks (title) VALUES ('After the error');
COMMIT;
SELECT count(*) FROM tasks;
```

```text
BEGIN
INSERT 0 1
ERROR:  new row for relation "tasks" violates check constraint "tasks_title_check"
DETAIL:  Failing row contains (3, , f, 2026-11-18 07:39:07.064407+00).
ERROR:  current transaction is aborted, commands ignored until end of transaction block
ROLLBACK
 count
-------
     0
(1 row)
```

After the failed `INSERT`, PostgreSQL ignores every command until the transaction ends, and `COMMIT` answers
`ROLLBACK`: even the successful `'First'` is gone. All or nothing.

**Other clients do not see unfinished work.** Open a second terminal with `psql` (`docker exec -it p16-postgres psql
-U postgres -d tasks`). In the **first** one, start a slow transaction:

```sql
BEGIN; INSERT INTO tasks (title) VALUES ('Slow writer'); SELECT pg_sleep(10); COMMIT;
```

Within those ten seconds, in the **second** terminal, then again after they are over:

```sql
SELECT id, title FROM tasks;
```

```text
 id | title
----+-------
(0 rows)

 id |    title
----+-------------
  4 | Slow writer
(1 row)
```

While the transaction runs, the second client sees nothing; after `COMMIT`, it sees the whole result at once. This is
*isolation*, the "I" of **ACID**:

| Letter | Guarantee | In this lesson |
|---|---|---|
| A — Atomicity | all or nothing | the failed `INSERT` undid `'First'` |
| C — Consistency | constraints hold after every transaction | the `CHECK` refused `''` |
| I — Isolation | unfinished transactions are invisible to others | the second client saw `0 rows` |
| D — Durability | committed data survives a crash | Lesson 16.5, Section 8 |

There are several **isolation levels** with different trade-offs; PostgreSQL's default ("read committed") is the one
you just saw. Lessons 17.10 to 17.12 show how databases implement all four guarantees.

## 6. A transaction in Rust

Now the archive as an endpoint. Add `post` to the `routing` import (`routing::{get, post}`), then this handler:

```rust
#[derive(Debug, Serialize)]
struct Archived {
    archived: usize,
}

/// Moves every done task into `archived_tasks`: all of them or none.
async fn archive_done(State(state): State<AppState>) -> Result<Json<Archived>, ApiError> {
    let mut tx = state.pool.begin().await?;

    let moved = sqlx::query_as!(
        Task,
        "DELETE FROM tasks WHERE done RETURNING id, title, done"
    )
    .fetch_all(&mut *tx)
    .await?;

    for task in &moved {
        sqlx::query!(
            "INSERT INTO archived_tasks (id, title) VALUES ($1, $2)",
            task.id,
            task.title
        )
        .execute(&mut *tx)
        .await?;
    }

    tx.commit().await?;
    Ok(Json(Archived {
        archived: moved.len(),
    }))
}
```

and its route in `app()`, next to `/tasks`:

```rust
        .route("/tasks/archive-done", post(archive_done))
```

**What happens:**

1. `state.pool.begin()` takes a connection from the pool and sends `BEGIN`. `tx` is a `Transaction`: all queries that
   run on it belong to the transaction.
2. `.fetch_all(&mut *tx)` runs the query **on the transaction** instead of on the pool. (`*tx` gives the connection
   inside the transaction; `&mut` because a connection can run only one query at a time.) If you wrote `&state.pool`
   here by mistake, the query would run on another connection, outside the transaction.
3. `DELETE ... RETURNING` deletes the done tasks and returns exactly those rows, so the `INSERT`s archive exactly what
   was deleted, even if another request marks a task as done in the meantime.
4. `tx.commit()` sends `COMMIT`. Only now do other clients see the change.
5. If any `?` returns early, `tx` is **dropped** without `commit`. sqlx then rolls the transaction back. You never
   write `ROLLBACK` yourself: an early return, an error or a panic all undo the work. This is RAII (Resource Acquisition Is Initialization) from Lesson 08.7.

The path `/tasks/archive-done` names an action, not a resource. Lesson 16.2 allowed exactly this for `POST` when an
operation does not fit create/read/update/delete.

Recreate the database to see the migrations run at start (`cargo sqlx database drop -y` and
`cargo sqlx database create`), start the service with `cargo run`, and in the second terminal:

```bash
curl -s -H 'content-type: application/json' -d '{"title":"Buy milk"}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"title":"Write lesson"}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"title":"Call mom"}' http://127.0.0.1:3000/tasks; echo
curl -s -X PATCH -H 'content-type: application/json' -d '{"done":true}' http://127.0.0.1:3000/tasks/1 > /dev/null
curl -s -X PATCH -H 'content-type: application/json' -d '{"done":true}' http://127.0.0.1:3000/tasks/3 > /dev/null
curl -s -X POST http://127.0.0.1:3000/tasks/archive-done; echo
curl -s http://127.0.0.1:3000/tasks; echo
```

```text
{"id":1,"title":"Buy milk","done":false}
{"id":2,"title":"Write lesson","done":false}
{"id":3,"title":"Call mom","done":false}
{"archived":2}
[{"id":2,"title":"Write lesson","done":false}]
```

The new database started with identifier 1 and has both migrations: the service applied them before it began to
listen. In `psql`, `SELECT id, title FROM archived_tasks ORDER BY id;` shows tasks 1 and 3.

> ⚡ **Performance:** the loop sends one `INSERT` per task. For a few rows that is fine; for thousands, do it in one
> statement (PostgreSQL can insert from a list with `UNNEST`, or from a `DELETE ... RETURNING` in a `WITH` query).
> Also keep transactions **short**: a transaction holds a pooled connection and locks rows until it ends, so never wait
> for a slow network call inside one.

## 7. Testing the rollback

The interesting property of the transaction is what happens when something fails **in the middle**. Make the second
statement fail on purpose: put a row with `id` 1 into `archived_tasks` before archiving task 1, so the `INSERT`
violates the primary key. Add a helper for empty `POST` requests and the test to the `tests` module:

```rust
    fn post_empty(uri: &str) -> Request<Body> {
        Request::post(uri).body(Body::empty()).unwrap()
    }

    #[sqlx::test]
    async fn archive_is_all_or_nothing(pool: PgPool) {
        let app = test_app(pool.clone()).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;
        sqlx::query!("UPDATE tasks SET done = true WHERE id = 1")
            .execute(&pool)
            .await
            .unwrap();
        // Make the INSERT into archived_tasks fail: id 1 is already there.
        sqlx::query!("INSERT INTO archived_tasks (id, title) VALUES (1, 'old')")
            .execute(&pool)
            .await
            .unwrap();

        let (status, _) = send(app.clone(), post_empty("/tasks/archive-done")).await;
        assert_eq!(status, StatusCode::INTERNAL_SERVER_ERROR);

        // The DELETE was rolled back: task 1 is still in `tasks`.
        let request = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, _) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
    }
```

```text
test tests::archive_is_all_or_nothing ... ok
```

The `DELETE` succeeded, the `INSERT` failed, `?` returned, `tx` was dropped, and PostgreSQL rolled the `DELETE` back:
task 1 is still there. Remove `tx.commit()` from the handler, or replace `&mut *tx` with `&state.pool` in the `DELETE`,
and this test fails. Run `cargo sqlx prepare` again, because the handler added new queries.

> 🧪 **Testing:** to test error handling, make the failure happen for real (here: a duplicate key) instead of only
> imagining it. A test that forces the middle step to fail is the only proof that "all or nothing" works.

## Common mistakes

**1. Editing an applied migration.** Section 3: `was previously applied but has been modified`. Write a new migration.

**2. Deleting a migration file that a database has applied.** The service then refuses to start:

```text
thread 'main' (5828436) panicked at src/main.rs:224:10:
cannot run the migrations: VersionMissing(20261118074133)
```

and `cargo sqlx migrate run` says `migration 20261118074133 was previously applied but is missing in the resolved
migrations`. Restore the file from Git. On your laptop only, `cargo sqlx database reset -y` starts from scratch.

**3. Running a query on the pool inside a transaction.** `.execute(&state.pool)` between `begin` and `commit` runs on a
different connection, outside the transaction. It is not rolled back, and it does not see the transaction's
unfinished changes. Inside a transaction, always pass `&mut *tx`.

**4. Forgetting `commit`.** Without `tx.commit().await?`, the work is rolled back when `tx` is dropped, silently. The
endpoint answers success and nothing changed. The test in Section 7 style catches this.

**5. Long transactions.** A transaction that waits for a user, a file upload or another service holds a connection and
locks. Do the slow work first, then open the transaction for the quick database part.

## Try it yourself

1. Run `cargo sqlx migrate info` and `\d archived_tasks` in `psql`.
2. Run `cargo sqlx database reset -y`, then `cargo sqlx migrate info`: everything is applied again, in order.
3. In `psql`, try `BEGIN; DELETE FROM tasks; SELECT count(*) FROM tasks; ROLLBACK; SELECT count(*) FROM tasks;` and
   watch the tasks come back.
4. Commit: `git add code/p16-tasks && git commit -m "p16-tasks: migrations and an atomic archive"`.

## Exercises

### Exercise 1 — A third migration (easy)

Most queries ask for open tasks. Add a migration `index_open_tasks` that creates a **partial index**, an index that
contains only the rows matching a condition: `CREATE INDEX tasks_open_idx ON tasks (id) WHERE NOT done;`. Run it and
show that it is installed and visible in `\d tasks`.

<details>
<summary>Hint</summary>

`cargo sqlx migrate add index_open_tasks`, write the SQL into the new file, `cargo sqlx migrate run`.

</details>

<details>
<summary>Answer</summary>

```sql
-- Most queries ask for the open tasks.
CREATE INDEX tasks_open_idx ON tasks (id) WHERE NOT done;
```

```text
Applied 20261118074133/migrate index open tasks (129.933667ms)
20261118073832/installed create tasks
20261118073943/installed create archived tasks
20261118074133/installed index open tasks
```

`\d tasks` now lists `"tasks_open_idx" btree (id) WHERE NOT done` under `Indexes`.

Why it works: the index has only the open tasks, so it stays small while done tasks pile up. Because it is a
migration, every database (and every test) gets it automatically. Remember to commit the new file together with the
code.

</details>

### Exercise 2 — Test the happy path (medium)

Write a `#[sqlx::test]` `archive_moves_only_done_tasks`: create `a`, `b`, `c`, mark tasks 1 and 3 done with `PATCH`,
call `POST /tasks/archive-done`, and check the answer `{"archived":2}` and that `GET /tasks` returns only `b`.

<details>
<summary>Hint</summary>

`Request::patch(format!("/tasks/{id}"))` accepts a `String` too. Reuse `post_empty` from Section 7.

</details>

<details>
<summary>Answer</summary>

```rust
    #[sqlx::test]
    async fn archive_moves_only_done_tasks(pool: PgPool) {
        let app = test_app(pool).await;
        for title in ["a", "b", "c"] {
            let body = format!(r#"{{"title":"{title}"}}"#);
            let request = Request::post("/tasks")
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(body))
                .unwrap();
            send(app.clone(), request).await;
        }
        for id in [1, 3] {
            let request = Request::patch(format!("/tasks/{id}"))
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(r#"{"done":true}"#))
                .unwrap();
            send(app.clone(), request).await;
        }

        let (status, body) = send(app.clone(), post_empty("/tasks/archive-done")).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"archived":2}"#);

        let request = Request::get("/tasks").body(Body::empty()).unwrap();
        let (_, body) = send(app, request).await;
        assert_eq!(body, r#"[{"id":2,"title":"b","done":false}]"#);
    }
```

Why it works: the test goes only through the public API (Application Programming Interface), as a client would, so it stays valid even if the handler is
rewritten (for example as one SQL statement). Together with `archive_is_all_or_nothing` it covers both outcomes.

</details>

### Exercise 3 — Drop means rollback (hard)

Prove the rule "a dropped transaction is rolled back" without any HTTP (HyperText Transfer Protocol): write a `#[sqlx::test]`
`dropped_transaction_is_rolled_back` that begins a transaction, inserts a task through it, lets the transaction go out
of scope **without** `commit`, and then checks with `query_scalar!` that `tasks` is empty.

<details>
<summary>Hint</summary>

Put the transaction in its own block `{ ... }`; it is dropped at the closing brace. `count(*)` may be `NULL` as far as
the macro knows, so name the column `AS "count!"` to get an `i64`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[sqlx::test]
    async fn dropped_transaction_is_rolled_back(pool: PgPool) {
        {
            let mut tx = pool.begin().await.unwrap();
            sqlx::query!("INSERT INTO tasks (title) VALUES ('never saved')")
                .execute(&mut *tx)
                .await
                .unwrap();
            // No commit: `tx` is dropped at the end of this block.
        }
        let count = sqlx::query_scalar!(r#"SELECT count(*) AS "count!" FROM tasks"#)
            .fetch_one(&pool)
            .await
            .unwrap();
        assert_eq!(count, 0);
    }
```

With the two exercises of this lesson (and without the exercise tests of Lesson 16.6), `cargo test` runs ten tests:

```text
test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.99s
```

Why it works: `Transaction` implements `Drop`; if it was not committed, it rolls back (sqlx sends the `ROLLBACK` when
the connection is next used or returned to the pool). This is why the `?` in `archive_done` is enough for safety. The
test documents the behaviour your code relies on.

</details>

## Quiz

1. What does the table `_sqlx_migrations` store, and why?

<details>
<summary>Answer</summary>

Which migrations have been applied to this database (version, description, success) and a checksum of each file.
With it, `migrate run` applies only new migrations and detects edited ones.

</details>

2. You find a mistake in a migration that is already in production. What do you do?

<details>
<summary>Answer</summary>

Leave it as it is and write a new migration that fixes the schema (for example `ALTER TABLE ...`). Editing the old
file would change its checksum and would never run again on databases that already applied it.

</details>

3. What does `sqlx::migrate!()` do?

<details>
<summary>Answer</summary>

At compile time it embeds the files of the `migrations` folder in the program; `.run(&pool)` applies the missing
ones when the service starts.

</details>

4. What happens to `'First'` in a transaction where a later statement fails, and why?

<details>
<summary>Answer</summary>

It is undone too. A transaction is atomic: after an error PostgreSQL rejects further commands, and the transaction
can only end with a rollback.

</details>

5. In sqlx, what happens if a function returns early with `?` after `begin()` but before `commit()`?

<details>
<summary>Answer</summary>

The `Transaction` value is dropped without a commit, and sqlx rolls the transaction back. None of its changes are kept.

</details>

6. Why should transactions be short?

<details>
<summary>Answer</summary>

While a transaction is open it holds a connection from the pool and locks the rows it changed, so other requests
may have to wait. Do slow work outside it.

</details>

## Summary

- A migration is a numbered SQL file that changes the schema once per database; `_sqlx_migrations` records which ran.
- `cargo sqlx migrate add`, `migrate run` and `migrate info` create, apply and show migrations; `database reset`
  rebuilds a development database.
- Never edit or delete an applied migration; add a new one. sqlx checks checksums and refuses drift.
- `sqlx::migrate!().run(&pool)` applies missing migrations at start; `#[sqlx::test]` applies them to every test database.
- A transaction is all or nothing (ACID). In sqlx: `pool.begin()`, run queries on `&mut *tx`, `tx.commit()`; dropping
  the transaction without commit rolls it back.
- Test the failure in the middle, not only the happy path.

## Sources and further reading

- [sqlx `migrate!` macro](https://docs.rs/sqlx/0.9/sqlx/macro.migrate.html) — embedding and running migrations from the program.
- [sqlx-cli README](https://github.com/launchbadge/sqlx/blob/main/sqlx-cli/README.md) — `database create/drop/reset` and `migrate add/run/info/revert`, including reversible migrations.
- [sqlx `Transaction`](https://docs.rs/sqlx/0.9/sqlx/struct.Transaction.html) — `begin`, `commit`, `rollback` and the rollback on drop.
- [PostgreSQL tutorial — Transactions](https://www.postgresql.org/docs/18/tutorial-transactions.html) — `BEGIN`, `COMMIT`, `ROLLBACK` and savepoints with an example.
- [PostgreSQL — Transaction isolation](https://www.postgresql.org/docs/18/transaction-iso.html) — the isolation levels in detail (preparation for Phase 17).

---

[🏠 Start](../START-HERE.md) · [← 16.6 Databases with sqlx](06-databases-with-sqlx.md) · [📋 Phase 16](README.md) · [16.8 Authentication in a web service →](08-authentication-in-a-web-service.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
