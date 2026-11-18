<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.4 Errors, validation and middleware](04-errors-validation-and-middleware.md) · [📋 Phase 16](README.md) · [16.6 Databases with sqlx →](06-databases-with-sqlx.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/05-sql-and-postgresql-basics.md) <!-- nav -->

# Lesson 16.5 — SQL and PostgreSQL basics

> **Phase 16 — Backend services** · Lesson 5 of 16 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a relational database is: tables, rows, columns, types and keys.
- How to run PostgreSQL in Docker and talk to it with `psql`.
- The four basic SQL (Structured Query Language) commands, `INSERT`, `SELECT`, `UPDATE` and `DELETE`, with `WHERE`, `ORDER BY`, `LIMIT` and `RETURNING`.
- How constraints (`NOT NULL`, `CHECK`, `PRIMARY KEY`, `REFERENCES`) let the database itself refuse bad data.
- How to connect two tables with a foreign key and a `JOIN`, and what an index is for.

## Before you start

- You finished [Lesson 16.4 — Errors, validation and middleware](04-errors-validation-and-middleware.md).
- You saw a little SQL with SQLite in [Lesson 10.5 — Injection attacks](../10-security-fundamentals/05-injection-attacks.md):
  `CREATE TABLE`, `INSERT`, `SELECT` and placeholders. This lesson starts from zero anyway.
- You know containers, images and volumes (Lesson 11.12 — Containers and Docker basics). Docker Desktop is running.

## New words

| Word | Meaning |
|---|---|
| **SQL** (Structured Query Language) | The language for asking a relational database to store, change and find data. |
| **relational database** | A database that keeps data in tables and lets you combine them in queries. PostgreSQL, MySQL and SQLite are relational databases. |
| **PostgreSQL** (often "Postgres") | A free, open-source relational database server, very common for backend services. |
| **table / row / column** | A table is like a spreadsheet: each **row** is one record (one task), each **column** is one field with a fixed type (`title`, `done`). |
| **query** | One SQL command sent to the database, for example `SELECT * FROM tasks`. |
| **primary key** | The column (or columns) that identifies each row; it must be unique and never empty. |
| **constraint** | A rule the database checks on every change: `NOT NULL`, `CHECK`, `UNIQUE`, `PRIMARY KEY`, `REFERENCES`. A change that breaks it fails. |
| **foreign key** | A column that points to the primary key of another table (`comments.task_id` → `tasks.id`). The database makes sure the target exists. |
| **`NULL`** | SQL's marker for "no value / unknown". It is not zero and not an empty string. |
| **join** | Combining rows of two tables that belong together in one query result. |
| **index** | An extra structure the database keeps next to a table so it can find rows by a column quickly, without reading the whole table. |
| **`psql`** | The command-line client of PostgreSQL: you type SQL, it shows the results. |
| **DDL / DML** (Data Definition / Data Manipulation Language) | The two halves of SQL: DDL changes the structure (`CREATE TABLE`, `ALTER TABLE`), DML changes the data (`INSERT`, `UPDATE`, `DELETE`). |

## 1. Why a database server?

Lesson 16.1 ended with a rule: data that must survive restarts and be shared by all instances lives outside the
service. You could write it to a file, but then you must solve many hard problems yourself: two instances writing
at the same moment, a crash in the middle of a write, finding one task among millions without reading them all,
making sure no task has an empty title. A **database server** solves these problems once, very well:

- it stores data on disk so that it survives crashes (Phase 17 shows how);
- it lets many clients read and write at the same time without breaking the data;
- it finds data fast with indexes;
- it checks rules (constraints) on every change, whoever sends it.

```text
  service instance 1 ──┐
  service instance 2 ──┼── TCP, port 5432 ──▶  PostgreSQL server ──▶ files on disk (a Docker volume)
  psql (you)         ──┘
```

PostgreSQL is a separate program, a server, exactly like your service. Clients connect to it over TCP (port `5432`
by default) and send SQL text; it answers with rows.

## 2. PostgreSQL in Docker

The official image `postgres:18` gives you a ready server in one command:

```bash
docker run -d --name p16-postgres \
  -e POSTGRES_PASSWORD=dev-password \
  -p 127.0.0.1:5432:5432 \
  -v p16-pgdata:/var/lib/postgresql \
  postgres:18
```

**What happens:**

1. `-d` runs the container in the background and prints its long identifier.
2. `--name p16-postgres` gives it a name, so later commands can say `p16-postgres` instead of the identifier.
3. `-e POSTGRES_PASSWORD=dev-password` sets the password of the superuser `postgres`. The image refuses to start
   without one. This is a **development** password for a database on your laptop, nothing more.
4. `-p 127.0.0.1:5432:5432` makes port 5432 of the container reachable on port 5432 of your Mac, but **only** from
   your Mac (`127.0.0.1`).
5. `-v p16-pgdata:/var/lib/postgresql` keeps the database files in a Docker volume named `p16-pgdata`. The
   container can be deleted and re-created; the data stays in the volume.

Check that it is ready (`docker logs` shows what the server printed):

```bash
docker logs p16-postgres 2>&1 | tail -1
```

```text
2026-11-18 07:23:07.761 UTC [1] LOG:  database system is ready to accept connections
```

> 🔒 **Security:** binding to `127.0.0.1` matters. With `-p 5432:5432` the database would listen on every network
> your Mac is connected to, for example a café's Wi-Fi, protected only by the password `dev-password`. Real passwords
> never go into commands or Git; Lesson 16.13 and Lesson 10.8 show how to handle secrets.

`psql` is already inside the container. Start it there (`-it` connects your keyboard and terminal to it):

```bash
docker exec -it p16-postgres psql -U postgres
```

```text
psql (18.7 (Debian 18.7-1.pgdg13+2))
Type "help" for help.

postgres=#
```

`postgres=#` is the prompt: you are connected to the database `postgres` as the superuser. A server holds many
**databases**; create one for the tasks service and switch to it. Every SQL command ends with `;`. Commands that
start with a backslash, like `\c` ("connect"), belong to `psql` itself and need no `;`:

```sql
CREATE DATABASE tasks;
\c tasks
```

```text
CREATE DATABASE
You are now connected to database "tasks" as user "postgres".
```

The prompt is now `tasks=#`. Useful `psql` commands: `\dt` lists tables, `\d name` describes one table, `\q` quits.
All SQL in the rest of this lesson is typed at this prompt.

> 💡 **Tip:** SQL keywords are not case-sensitive; `select` works like `SELECT`. Writing keywords in capitals and
> names in lowercase is a common convention that makes queries easier to read.

## 3. A table

```sql
CREATE TABLE tasks (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title      TEXT NOT NULL CHECK (char_length(title) BETWEEN 1 AND 200),
    done       BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**What happens:**

1. Each line inside the brackets is one column: a name, a **type** and rules.
2. `BIGINT` is a 64-bit signed integer (like `i64`). `GENERATED ALWAYS AS IDENTITY` means: the database chooses the
   number, 1, 2, 3, …, for every new row. `PRIMARY KEY` makes `id` the unique identifier of a row.
3. `TEXT` is a string of any length. `NOT NULL` forbids "no value". `CHECK (...)` is a condition every row must meet:
   the same 1–200 characters rule as `Title::parse` in Lesson 16.4.
4. `BOOLEAN` is `true`/`false`. `DEFAULT false`: if an `INSERT` does not mention `done`, it is `false`.
5. `TIMESTAMPTZ` is a point in time with a time zone. `DEFAULT now()` fills in the moment of the insert.

`\d tasks` shows what the database understood:

```text
                                    Table "public.tasks"
   Column   |           Type           | Collation | Nullable |           Default
------------+--------------------------+-----------+----------+------------------------------
 id         | bigint                   |           | not null | generated always as identity
 title      | text                     |           | not null |
 done       | boolean                  |           | not null | false
 created_at | timestamp with time zone |           | not null | now()
Indexes:
    "tasks_pkey" PRIMARY KEY, btree (id)
Check constraints:
    "tasks_title_check" CHECK (char_length(title) >= 1 AND char_length(title) <= 200)
```

The primary key got an index automatically (`tasks_pkey`, Section 7), and the `CHECK` got the name
`tasks_title_check`. `public` is the default *schema*, a folder for tables inside a database.

| Rust | PostgreSQL |
|---|---|
| `i32` / `i64` | `INTEGER` / `BIGINT` |
| `String` | `TEXT` (or `VARCHAR(n)` with a maximum length) |
| `bool` | `BOOLEAN` |
| `f64` | `DOUBLE PRECISION` (never for money: use `NUMERIC`) |
| a point in time | `TIMESTAMPTZ` |
| `Option<T>` | a column **without** `NOT NULL` |

## 4. Insert, read, change, delete

**`INSERT`** adds rows. You name the columns you give; the others get their default:

```sql
INSERT INTO tasks (title) VALUES ('Buy milk');
INSERT INTO tasks (title) VALUES ('Write lesson'), ('Call mom') RETURNING id, title, done;
```

```text
INSERT 0 1
 id |    title     | done
----+--------------+------
  2 | Write lesson | f
  3 | Call mom     | f
(2 rows)

INSERT 0 2
```

`INSERT 0 1` means "one row inserted" (the `0` is a historical leftover). `RETURNING` sends back columns of the new
rows: this is how a service learns the `id` the database chose. Text values are in **single** quotes. `f` and `t` are
how `psql` prints `false` and `true`.

**`SELECT`** reads. `*` means "all columns":

```sql
SELECT * FROM tasks;
```

```text
 id |    title     | done |          created_at
----+--------------+------+------------------------------
  1 | Buy milk     | f    | 2026-11-18 07:23:12.18793+00
  2 | Write lesson | f    | 2026-11-18 07:23:12.18828+00
  3 | Call mom     | f    | 2026-11-18 07:23:12.18828+00
(3 rows)
```

`WHERE` keeps only the rows that match, `ORDER BY` sorts (`DESC` is descending), `LIMIT` returns at most a number of rows:

```sql
SELECT id, title FROM tasks WHERE done = false ORDER BY id DESC LIMIT 2;
```

```text
 id |    title
----+--------------
  3 | Call mom
  2 | Write lesson
(2 rows)
```

> ⚠️ **Common mistake:** without `ORDER BY` the database may return rows in **any** order, and the order can change
> from one run to the next. If the order matters (and in an API, an Application Programming Interface, it always does), say it.

**`UPDATE`** changes columns of the rows that match `WHERE`. **`DELETE`** removes them:

```sql
UPDATE tasks SET done = true WHERE id = 1 RETURNING id, title, done;
SELECT count(*) FROM tasks WHERE NOT done;
DELETE FROM tasks WHERE id = 3;
SELECT id, title, done FROM tasks ORDER BY id;
```

```text
 id |  title   | done
----+----------+------
  1 | Buy milk | t
(1 row)

UPDATE 1
 count
-------
     2
(1 row)

DELETE 1
 id |    title     | done
----+--------------+------
  1 | Buy milk     | t
  2 | Write lesson | f
(2 rows)
```

`count(*)` is an **aggregate**: it turns many rows into one number. `UPDATE 1` and `DELETE 1` tell how many rows were
affected; a service uses that number to answer `404` when it is `0`.

> ⚠️ **Common mistake:** `UPDATE tasks SET done = true;` or `DELETE FROM tasks;` **without** `WHERE` changes or deletes
> **every** row. `psql` does not ask "are you sure?". Write the `WHERE` first.

## 5. Constraints: the database says no

Now break the rules on purpose. Each command fails with a clear error, and nothing is stored:

```sql
INSERT INTO tasks (title) VALUES ('');
INSERT INTO tasks (title, done) VALUES ('Read', NULL);
INSERT INTO tasks (id, title) VALUES (10, 'Pick an id');
UPDATE tasks SET done = 'maybe' WHERE id = 2;
SELECT * FROM taks;
```

```text
ERROR:  new row for relation "tasks" violates check constraint "tasks_title_check"
DETAIL:  Failing row contains (4, , f, 2026-11-18 07:23:12.189411+00).
ERROR:  null value in column "done" of relation "tasks" violates not-null constraint
DETAIL:  Failing row contains (5, Read, null, 2026-11-18 07:23:12.189521+00).
ERROR:  cannot insert a non-DEFAULT value into column "id"
DETAIL:  Column "id" is an identity column defined as GENERATED ALWAYS.
HINT:  Use OVERRIDING SYSTEM VALUE to override.
ERROR:  invalid input syntax for type boolean: "maybe"
LINE 1: UPDATE tasks SET done = 'maybe' WHERE id = 2;
                                ^
ERROR:  relation "taks" does not exist
LINE 1: SELECT * FROM taks;
                      ^
```

**What happens:**

1. The empty title breaks `tasks_title_check`.
2. `NULL` in a `NOT NULL` column is refused.
3. With `GENERATED ALWAYS`, only the database chooses identifiers; a client cannot pick `10`.
4. Every value must fit the column type: `'maybe'` is not a boolean.
5. A typo in a table name ("relation" is PostgreSQL's word for a table) is found before anything runs.

Look at the `DETAIL` lines: the failed inserts already had the identifiers `4` and `5`. The database takes the next
number **before** it checks the row, and a failed insert does not give the number back. Insert one more task:

```sql
INSERT INTO tasks (title) VALUES ('Read a book') RETURNING id;
```

```text
 id
----
  6
(1 row)

INSERT 0 1
```

Identifiers can have **gaps**. That is normal and correct: treat identifiers as names, never as "the number of tasks".

> 🔒 **Security:** the service (Lesson 16.4) already checks titles, so why also in the database? *Defence in depth*
> (Lesson 10.1): another program, a script or a future bug may write to the same table. The database is the last line,
> and it protects the data whoever writes it.

**`NULL` in one paragraph.** `NULL` means "unknown", so comparing with it gives "unknown" too: `NULL = NULL` is not
`true` but `NULL`, and a `WHERE` with `NULL` keeps nothing. To test for it, use `IS NULL` / `IS NOT NULL`:

```sql
SELECT NULL = NULL AS equal, NULL IS NULL AS is_null;
```

```text
 equal | is_null
-------+---------
       | t
(1 row)
```

`AS` gives a result column a name. The empty cell is `NULL`. Most bugs with `NULL` disappear if columns are
`NOT NULL` unless you really need "no value".

## 6. Two tables: foreign keys and joins

Lesson 16.2 designed comments on tasks. Each comment belongs to one task, so the comments table **points** to the
tasks table:

```sql
CREATE TABLE comments (
    id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    task_id BIGINT NOT NULL REFERENCES tasks (id) ON DELETE CASCADE,
    text    TEXT NOT NULL
);
INSERT INTO comments (task_id, text) VALUES (1, 'Use the oat milk'), (1, 'Two bottles');
INSERT INTO comments (task_id, text) VALUES (99, 'Lost comment');
```

```text
CREATE TABLE
INSERT 0 2
ERROR:  insert or update on table "comments" violates foreign key constraint "comments_task_id_fkey"
DETAIL:  Key (task_id)=(99) is not present in table "tasks".
```

`REFERENCES tasks (id)` makes `task_id` a **foreign key**: the database refuses a comment for a task that does not
exist. `ON DELETE CASCADE` says what happens when a task is deleted: its comments are deleted too.

To see tasks and their comments **together**, join the tables. `JOIN ... ON` pairs every task row with every comment
row whose `task_id` is the task's `id`:

```sql
SELECT tasks.title, comments.text
FROM tasks
JOIN comments ON comments.task_id = tasks.id;
```

```text
  title   |       text
----------+------------------
 Buy milk | Use the oat milk
 Buy milk | Two bottles
(2 rows)
```

Tasks without comments are missing: a plain `JOIN` keeps only pairs that exist. A `LEFT JOIN` keeps **every** row of
the left table, with `NULL` where there is no partner. Combined with `GROUP BY` (one result row per task) and
`count`, it answers "how many comments does each task have?":

```sql
SELECT tasks.id, tasks.title, count(comments.id) AS comments
FROM tasks
LEFT JOIN comments ON comments.task_id = tasks.id
GROUP BY tasks.id
ORDER BY tasks.id;
```

```text
 id |    title     | comments
----+--------------+----------
  1 | Buy milk     |        2
  2 | Write lesson |        0
  6 | Read a book  |        0
(3 rows)
```

`count(comments.id)` counts only non-`NULL` values, so tasks without comments get `0`.

```sql
DELETE FROM tasks WHERE id = 1;
SELECT count(*) FROM comments;
```

```text
DELETE 1
 count
-------
     0
(1 row)
```

`ON DELETE CASCADE` removed both comments of task 1. Without it, the `DELETE` would fail, because the comments would
point to nothing.

## 7. Indexes

To run `WHERE comments.task_id = 6`, the database must find the matching rows. Without help it reads **every** row
of the table (a *sequential scan*): fine for a hundred rows, slow for a hundred million. An **index** is a sorted
structure (usually a B-tree, Lesson 17.6) from column values to rows, like the index at the back of a book.

PostgreSQL creates an index for every primary key and `UNIQUE` constraint, but **not** for foreign keys. Create one,
because "comments of task N" is a query the service runs all the time:

```sql
CREATE INDEX comments_task_id_idx ON comments (task_id);
```

`EXPLAIN` shows the **plan**: how the database will run a query, without running it:

```sql
EXPLAIN SELECT * FROM tasks WHERE id = 2;
```

```text
                               QUERY PLAN
-------------------------------------------------------------------------
 Index Scan using tasks_pkey on tasks  (cost=0.15..8.17 rows=1 width=49)
   Index Cond: (id = 2)
(2 rows)
```

"Index Scan using tasks_pkey": the database will jump to the row through the primary key index instead of reading the
table. Lesson 17.13 explains plans in depth.

> ⚡ **Performance:** an index makes reads by that column fast, but every `INSERT`, `UPDATE` and `DELETE` must also
> update the index, and it takes disk space. Create indexes for the queries you really run (foreign keys, columns in
> frequent `WHERE` and `ORDER BY`), not for every column.

## 8. The data survives

Quit `psql` with `\q` and restart the container, like a crash or a reboot would:

```bash
docker restart p16-postgres
docker exec -it p16-postgres psql -U postgres -d tasks -c 'SELECT id, title, done FROM tasks ORDER BY id;'
```

```text
 id |    title     | done
----+--------------+------
  2 | Write lesson | f
  6 | Read a book  | f
(2 rows)
```

`-d tasks` connects directly to the database `tasks`; `-c '...'` runs one command and exits. The rows are still
there: they live in the volume `p16-pgdata`, not in the memory of a process. This is what the in-memory store of
Lesson 16.3 could not do.

Leave the container running for the next lessons, or stop it with `docker stop p16-postgres` and start it again with
`docker start p16-postgres`. When you finish the whole phase, remove it with `docker rm -f p16-postgres` and
`docker volume rm p16-pgdata` (this deletes the data).

## Common mistakes

**1. Double quotes for text.** `WHERE title = "Buy milk"` fails: in SQL, double quotes are for **names** (columns,
tables), so PostgreSQL looks for a column called `Buy milk`:

```text
ERROR:  column "Buy milk" does not exist
```

Use single quotes for text: `'Buy milk'`. A single quote inside text is written twice: `'Mom''s birthday'`.

**2. Forgetting `WHERE`** in `UPDATE` or `DELETE` (Section 4). It changes every row.

**3. Comparing with `= NULL`.** `WHERE done = NULL` never matches anything. Use `IS NULL`.

**4. Building SQL by pasting values into the text.** This is SQL injection (Lesson 10.5). In `psql` you type values by
hand; in Rust code you will always send them separately as parameters (`$1`, `$2`), as Lesson 16.6 shows.

**5. Expecting gap-free identifiers.** Failed inserts (and, in Lesson 16.7, rolled-back transactions) leave gaps.

## Try it yourself

1. Run `\dt` and `\d comments`. Find the foreign key in the output.
2. Put the two `CREATE TABLE` commands into a file `code/p16-tasks/schema.sql` (you will need them again in Lesson
   16.6), then check that the file is valid by running it against a new, empty database:

   ```bash
   docker exec -it p16-postgres psql -U postgres -c 'CREATE DATABASE scratch;'
   docker exec -i p16-postgres psql -U postgres -d scratch < code/p16-tasks/schema.sql
   ```

   (`-i` without `-t` because the input comes from a file, not from your keyboard.) You should see `CREATE TABLE`
   twice. Then remove the test database: `docker exec -it p16-postgres psql -U postgres -c 'DROP DATABASE scratch;'`.
3. In `psql`, run `SELECT now();` twice and watch the time change.

## Exercises

Run them in `psql`, connected to the database `tasks`. First add two tasks:
`INSERT INTO tasks (title) VALUES ('Buy oat milk'), ('Milk the cow');`.

### Exercise 1 — Open tasks, newest first (easy)

List `id`, `title` and `created_at` of all tasks that are not done, the newest first. Tasks inserted by one command
have the same `created_at`; for them, put the larger `id` first.

<details>
<summary>Hint</summary>

`ORDER BY` accepts several columns separated by commas; the second one decides only when the first is equal.

</details>

<details>
<summary>Answer</summary>

```sql
SELECT id, title, created_at FROM tasks WHERE NOT done ORDER BY created_at DESC, id DESC;
```

```text
 id |    title     |          created_at
----+--------------+-------------------------------
  8 | Milk the cow | 2026-11-18 07:23:26.631474+00
  7 | Buy oat milk | 2026-11-18 07:23:26.631474+00
  6 | Read a book  | 2026-11-18 07:23:12.189823+00
  2 | Write lesson | 2026-11-18 07:23:12.18828+00
(4 rows)
```

Why it works: `now()` is the time of the **transaction**, so both rows of one `INSERT` get the same value, and the
second sort key breaks the tie. Your times differ, of course.

</details>

### Exercise 2 — Everything with milk (medium)

Mark every task whose title contains the word "milk" in any letter case ("Buy oat milk", "Milk the cow") as done, in
**one** command, and show the `id` and `title` of the changed rows.

<details>
<summary>Hint</summary>

`LIKE` compares with a pattern in which `%` means "any text"; `ILIKE` is the same but ignores the letter case.

</details>

<details>
<summary>Answer</summary>

```sql
UPDATE tasks SET done = true WHERE title ILIKE '%milk%' RETURNING id, title;
```

```text
 id |    title
----+--------------
  7 | Buy oat milk
  8 | Milk the cow
(2 rows)

UPDATE 2
```

Why it works: `'%milk%'` matches any title with "milk" anywhere; `ILIKE` makes "Milk" match too. `RETURNING` shows
exactly which rows the `UPDATE` touched, so you can check a risky change immediately.

</details>

### Exercise 3 — A test inside the database (hard)

Write a **test** for the `CHECK` constraint in SQL itself: it must try to insert an empty title, print
`test passed: the empty title was rejected` if PostgreSQL refuses it, and fail with an error if PostgreSQL accepts
it. Then prove that the test can fail: drop the constraint with
`ALTER TABLE tasks DROP CONSTRAINT tasks_title_check;`, run the test again, and finally restore the constraint with
`ALTER TABLE tasks ADD CONSTRAINT tasks_title_check CHECK (char_length(title) BETWEEN 1 AND 200);`.

<details>
<summary>Hint</summary>

PostgreSQL runs small programs with `DO $$ BEGIN ... END $$;` (the language is PL/pgSQL, Procedural Language/PostgreSQL). Inside, `RAISE EXCEPTION
'...'` fails, `RAISE NOTICE '...'` prints a message, and a block can catch an error by name with
`EXCEPTION WHEN check_violation THEN ...`.

</details>

<details>
<summary>Answer</summary>

```sql
DO $$
BEGIN
    INSERT INTO tasks (title) VALUES ('');
    RAISE EXCEPTION 'test failed: an empty title was accepted';
EXCEPTION WHEN check_violation THEN
    RAISE NOTICE 'test passed: the empty title was rejected';
END
$$;
```

With the constraint:

```text
NOTICE:  test passed: the empty title was rejected
DO
```

Without the constraint:

```text
ERROR:  test failed: an empty title was accepted
CONTEXT:  PL/pgSQL function inline_code_block line 4 at RAISE
```

Why it works: if the `INSERT` fails with `check_violation`, the `EXCEPTION` part runs and prints the notice. If the
`INSERT` succeeds, the next line raises our own error, and the test fails. Watching the test fail once proves that it
really checks something (the "red" step of Lesson 05.6). Because the whole `DO` block fails, its `INSERT` is undone
too: `SELECT count(*) FROM tasks WHERE title = '';` returns `0`. Lesson 16.7 explains why (transactions).

</details>

## Quiz

1. What is the difference between a primary key and a foreign key?

<details>
<summary>Answer</summary>

A primary key identifies each row of **its own** table (unique, not `NULL`). A foreign key is a column that points to
the primary key of **another** table, and the database makes sure the target row exists.

</details>

2. Why is it good that the database checks the title even though the service already does?

<details>
<summary>Answer</summary>

Defence in depth: other programs, scripts or a future bug in the service may write to the table. The constraint
protects the data whoever writes it.

</details>

3. What does `RETURNING id` do, and why does a service need it?

<details>
<summary>Answer</summary>

It makes `INSERT` (or `UPDATE`/`DELETE`) send back columns of the affected rows. The service learns the identifier the
database chose and can answer `201` with `location: /tasks/{id}`.

</details>

4. Why does `SELECT * FROM tasks WHERE done = NULL` return nothing?

<details>
<summary>Answer</summary>

Any comparison with `NULL` is "unknown" (`NULL`), and `WHERE` keeps only rows where the condition is `true`. Use
`IS NULL`.

</details>

5. What is the difference between `JOIN` and `LEFT JOIN`?

<details>
<summary>Answer</summary>

`JOIN` returns only pairs of rows that match. `LEFT JOIN` returns every row of the left table, with `NULL` columns
when there is no matching row on the right.

</details>

6. What does an index cost?

<details>
<summary>Answer</summary>

Disk space, and extra work on every `INSERT`, `UPDATE` and `DELETE`, because the index must be updated too. In return,
reads by the indexed column become fast.

</details>

## Summary

- A relational database stores rows in tables with typed columns; PostgreSQL is a server that clients reach over TCP.
- `docker run ... postgres:18` with a volume gives you a local server; `psql` lets you type SQL.
- `INSERT ... RETURNING`, `SELECT ... WHERE ... ORDER BY ... LIMIT`, `UPDATE ... WHERE`, `DELETE ... WHERE` are the
  everyday commands. Always write `ORDER BY` when order matters and `WHERE` before changing data.
- Constraints (`NOT NULL`, `CHECK`, `PRIMARY KEY`, `REFERENCES`) make the database refuse bad data; identifiers may have gaps.
- `NULL` means unknown: test it with `IS NULL`.
- Foreign keys connect tables; `JOIN`/`LEFT JOIN` with `GROUP BY` combine them; indexes make lookups fast.

## Sources and further reading

- [PostgreSQL 18 tutorial](https://www.postgresql.org/docs/18/tutorial.html) — the official, beginner-friendly introduction to SQL with PostgreSQL.
- [psql reference](https://www.postgresql.org/docs/18/app-psql.html) — every backslash command of `psql`.
- [PostgreSQL — Constraints](https://www.postgresql.org/docs/18/ddl-constraints.html) — `CHECK`, `NOT NULL`, `UNIQUE`, primary and foreign keys.
- [PostgreSQL — Queries](https://www.postgresql.org/docs/18/queries.html) — `SELECT`, joins, `GROUP BY`, `ORDER BY` and `LIMIT` in detail.
- [PostgreSQL — Data types](https://www.postgresql.org/docs/18/datatype.html) — all column types.
- [The postgres Docker image](https://hub.docker.com/_/postgres) — the environment variables and volume paths of the official image.

---

[🏠 Start](../START-HERE.md) · [← 16.4 Errors, validation and middleware](04-errors-validation-and-middleware.md) · [📋 Phase 16](README.md) · [16.6 Databases with sqlx →](06-databases-with-sqlx.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
