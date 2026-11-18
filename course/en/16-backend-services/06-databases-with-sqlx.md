<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.5 SQL and PostgreSQL basics](05-sql-and-postgresql-basics.md) · [📋 Phase 16](README.md) · [16.7 Migrations and transactions →](07-migrations-and-transactions.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/06-databases-with-sqlx.md) <!-- nav -->

# Lesson 16.6 — Databases with sqlx

> **Phase 16 — Backend services** · Lesson 6 of 16 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How a Rust program talks to PostgreSQL with `sqlx`: a connection pool, queries with parameters, typed rows.
- The difference between queries checked when the program **runs** and queries checked when it **compiles**.
- How sqlx's *offline mode* lets you build without a database (for CI, Continuous Integration, and for Docker).
- How to move the tasks service from memory into PostgreSQL, and how to turn database errors into a safe `500`.
- How to test the service against a real, fresh database with `#[sqlx::test]`.

## Before you start

- You finished [Lesson 16.5 — SQL and PostgreSQL basics](05-sql-and-postgresql-basics.md). The container
  `p16-postgres` is running (`docker start p16-postgres` if not) and the database `tasks` has the tables `tasks` and
  `comments`. You saved both `CREATE TABLE` commands in `code/p16-tasks/schema.sql`.
- Your `code/p16-tasks` is the version from [Lesson 16.4](04-errors-validation-and-middleware.md).
- You remember SQL (Structured Query Language) injection and parameters from [Lesson 10.5](../10-security-fundamentals/05-injection-attacks.md).

## New words

| Word | Meaning |
|---|---|
| **sqlx** | An async Rust crate for SQL databases. You write plain SQL; sqlx sends it, with parameters, and turns rows into Rust values. It is not an ORM (Object-Relational Mapper). |
| **ORM** (Object-Relational Mapper) | A library that generates SQL for you from method calls on objects. sqlx deliberately does not do this. |
| **driver** | The code that speaks a database's network protocol. sqlx contains a PostgreSQL driver written in Rust. |
| **connection string** | One URL (Uniform Resource Locator) with everything needed to connect: `postgres://user:password@host:port/database`. |
| **connection pool** | A set of open database connections that requests borrow and give back, so no request has to open its own connection. |
| **query parameter** (bind parameter) | A placeholder like `$1` in the SQL text whose value is sent separately. It is the defence against SQL injection. |
| **compile-time checked query** | A query written with a sqlx macro (`query!`, `query_as!`) that the compiler checks against the real database: SQL syntax, column names and types. |
| **offline mode** | Building with saved query descriptions (the `.sqlx` folder) instead of a running database. |
| **`.env` file** | A file with `NAME=value` lines that tools and programs can load as environment variables during development. |

## 1. sqlx in one picture

```text
  handler ──▶ pool.acquire() ──▶ connection ──TCP──▶ PostgreSQL
     ▲          (borrow)              │  "SELECT ... WHERE id = $1"  +  [7]
     │                                ▼
     └────── Task { id: 7, ... } ◀── rows ◀────────────────────────────
```

sqlx gives you three things: a **pool** of connections, a way to send SQL **with parameters**, and conversion of
result rows into Rust structs. You still write every query yourself, in SQL. That is the point: you know exactly
what the database runs, and everything you learned in Lesson 16.5 applies directly.

## 2. Setup

Add sqlx (the `postgres` driver, Tokio as the runtime, the macros) and `dotenvy`, a small crate that loads a `.env`
file:

```bash
cd ~/projects/rust/code/p16-tasks
cargo add sqlx@0.9 --features runtime-tokio,postgres,macros
cargo add dotenvy@0.15
```

```toml
dotenvy = "0.15"
sqlx = { version = "0.9", features = ["runtime-tokio", "postgres", "macros"] }
```

sqlx also turns on a few default features, among them `migrate`, which Lesson 16.7 uses. Now tell the program, and
the sqlx macros, where the database is. Create `code/p16-tasks/.env` with one line:

```text
DATABASE_URL=postgres://postgres:dev-password@127.0.0.1:5432/tasks
```

The parts of the connection string: protocol `postgres`, user `postgres`, password `dev-password`, host
`127.0.0.1`, port `5432`, database `tasks`.

> 🔒 **Security:** `.env` holds a password, so it must **not** go into Git. Add the line `.env` to
> `code/p16-tasks/.gitignore` now. For real environments the variable is set by the platform, never by a file in the
> repository (twelve-factor, Lesson 08.15; Lesson 16.13).

## 3. First queries

Before rewriting the service, try sqlx in a small separate program. Create `src/bin/db_demo.rs`:

```rust
use sqlx::postgres::PgPoolOptions;

#[derive(Debug, sqlx::FromRow)]
struct Task {
    id: i64,
    title: String,
    done: bool,
}

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    dotenvy::dotenv().ok();
    let url = std::env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&url)
        .await?;

    // Checked by the database when the program runs.
    let task: Task =
        sqlx::query_as("INSERT INTO tasks (title) VALUES ($1) RETURNING id, title, done")
            .bind("Learn sqlx")
            .fetch_one(&pool)
            .await?;
    println!("inserted: {task:?}");

    // Checked by the compiler, against the real database, when the program is built.
    let open = sqlx::query_as!(
        Task,
        "SELECT id, title, done FROM tasks WHERE NOT done ORDER BY id"
    )
    .fetch_all(&pool)
    .await?;
    for task in &open {
        println!("open: {} {} (done: {})", task.id, task.title, task.done);
    }
    Ok(())
}
```

Run it with `cargo run --bin db_demo`:

```text
inserted: Task { id: 11, title: "Learn sqlx", done: false }
open: 2 Write lesson (done: false)
open: 6 Read a book (done: false)
open: 11 Learn sqlx (done: false)
```

Your identifiers depend on what you did in Lesson 16.5; the gaps are the failed inserts from there.

**What happens:**

1. `dotenvy::dotenv().ok()` reads `.env` and sets its variables for this process (`.ok()` ignores a missing file, so
   a real environment variable works too). Then `std::env::var` reads `DATABASE_URL`.
2. `PgPoolOptions::new().max_connections(5).connect(&url)` opens a **pool**: up to five connections, opened when
   needed and reused. A `PgPool` is cheap to clone; all clones share the same connections.
3. `sqlx::query_as("...")` prepares a query whose rows become `Task`. `$1` is a **parameter**; `.bind("Learn sqlx")`
   gives its value. The value travels separately from the SQL text, so no input can change the SQL (Lesson 10.5).
4. `#[derive(sqlx::FromRow)]` teaches sqlx to build a `Task` from a row by column name. `fetch_one` expects exactly
   one row; `RETURNING` provides it.
5. `sqlx::query_as!` (with `!`, a macro) does the same job, but is checked **at compile time** (Section 4).
   `fetch_all` returns a `Vec<Task>`.

The `fetch_*` methods:

| Method | Returns | Use it when |
|---|---|---|
| `fetch_one` | `T`, an error if there is no row | exactly one row must exist (`INSERT ... RETURNING`) |
| `fetch_optional` | `Option<T>` | zero or one row (`WHERE id = $1`) |
| `fetch_all` | `Vec<T>` | a list |
| `execute` | how many rows were affected | `UPDATE`/`DELETE` without `RETURNING` |

> ⚠️ **Common mistake:** PostgreSQL has no unsigned integers. `BIGINT` is `i64` in Rust, so identifiers are `i64` from
> now on, not `u64` as in Lesson 16.3.

## 4. Queries checked by the compiler

The first query (`query_as`, a function) is plain text to the compiler. A typo in a column name is found only when
that line **runs**, maybe in production. The macro version is different: while compiling, it connects to the
database in `DATABASE_URL`, asks PostgreSQL to analyse the SQL, and checks the result against your Rust types.

Make a typo on purpose, `titel` instead of `title`, in the `query_as!` and run `cargo build`:

```text
error: error returned from database: column "titel" does not exist at line 3861
  --> src/bin/db_demo.rs:28:16
   |
28 |       let open = sqlx::query_as!(
   |  ________________^
29 | |         Task,
30 | |         "SELECT id, titel, done FROM tasks WHERE NOT done ORDER BY id"
31 | |     )
   | |_____^
```

The **compiler** reports PostgreSQL's own error. (The "line 3861" is a line inside PostgreSQL's source code; ignore
it.) Fix the typo, and now change the struct field to `id: i32`:

```text
error[E0277]: the trait bound `i32: From<i64>` is not satisfied
  --> src/bin/db_demo.rs:28:16
   |
28 |       let open = sqlx::query_as!(
   |  ________________^
...
   | |_____^ the trait `From<i64>` is not implemented for `i32`
```

PostgreSQL said the column is `BIGINT` (`i64`), and an `i64` does not always fit into an `i32`. Change it back to `i64`.

How the macros map types:

| Column | Rust type from the macro |
|---|---|
| `BIGINT NOT NULL` | `i64` |
| `TEXT NOT NULL` | `String` |
| `BOOLEAN NOT NULL` | `bool` |
| any column **without** `NOT NULL` | `Option<…>` of the type |

This is one more reason for `NOT NULL` everywhere you can: the Rust types stay simple.

> 💡 **Tip:** use the macros (`query!`, `query_as!`, `query_scalar!`) for every query whose text is fixed. Use the
> functions (`query`, `query_as`) when the code must build without a database and without `.sqlx` data (Lesson 16.14
> does this), or when a part of the SQL text is chosen at run time (Common mistake 3). Values always go into
> parameters, never into the text.

## 5. Offline mode

Compile-time checks need a running database during every build. On your laptop that is fine. In CI (Lesson 09.12) or
inside `docker build` (Lesson 16.13), there is no database. Stop the container and touch a file to force the macros to
run again:

```bash
docker stop p16-postgres
touch src/bin/db_demo.rs
cargo build
```

```text
error: error communicating with database: Connection refused (os error 61)
```

The solution is **offline mode**: you save the descriptions of all queries in the repository once, and builds use
them instead of the database. The tool for this is `sqlx-cli`. Install it once (it takes a few minutes; the features
keep it small):

```bash
cargo install --locked sqlx-cli --no-default-features --features rustls,postgres
cargo sqlx --version
```

```text
sqlx-cli-sqlx 0.9.0
```

(You may get a newer `0.9.x`.) Start the database again and prepare:

```bash
docker start p16-postgres
cargo sqlx prepare
```

```text
query data written to .sqlx in the current directory; please check this into version control
```

The folder `.sqlx` now holds one JSON (JavaScript Object Notation) file per query: the SQL text, the column names and types, which parameters it
takes. A build with `SQLX_OFFLINE=true` uses only these files:

```bash
docker stop p16-postgres
touch src/bin/db_demo.rs
SQLX_OFFLINE=true cargo build
docker start p16-postgres
```

```text
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.60s
```

> 📏 **Convention:** commit `.sqlx` to Git (as the message says) and run `cargo sqlx prepare` again whenever you add
> or change a query. In CI, `cargo sqlx prepare --check` fails if the saved data is out of date.

## 6. The tasks service on PostgreSQL

Now replace the in-memory store. Four changes, file by file.

**`src/error.rs`:** the identifier becomes `i64`, and a new variant wraps every `sqlx::Error`. `#[from]` (from
`thiserror`) lets `?` convert a `sqlx::Error` into an `ApiError` automatically:

```rust
/// Every way a request to the tasks API can fail.
#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("task {0} not found")]
    NotFound(i64),
    #[error("{0}")]
    Validation(String),
    #[error("{message}")]
    Rejected { status: StatusCode, message: String },
    #[error("internal error")]
    Database(#[from] sqlx::Error),
}
```

In `status()` add `ApiError::Database(_) => StatusCode::INTERNAL_SERVER_ERROR,`, in `code()` add
`ApiError::Database(_) => "internal",`, and make `into_response` log the details before it builds the body:

```rust
impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        if let ApiError::Database(error) = &self {
            // The details go to the log, never to the client.
            tracing::error!(%error, "database error");
        }
        let body = ErrorBody {
            code: self.code(),
            message: self.to_string(),
        };
        (self.status(), Json(body)).into_response()
    }
}
```

The `Display` text of `Database` is just `internal error`, so the client never sees what went wrong inside. `%error`
records the error with its `Display` text as a field of the log event (Lesson 09.10).

**`src/main.rs`, the imports:** remove `BTreeMap`, `Arc` and `Mutex`, and add the pool types:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::Duration;

use sqlx::PgPool;
use sqlx::postgres::PgPoolOptions;
```

(The `axum`, `serde`, `tower_http`, `error` and `title` imports stay.) In `Task`, change `id: u64` to `id: i64`. Delete
`Store` and the `type AppState`, and add a state struct instead:

```rust
#[derive(Clone)]
struct AppState {
    pool: PgPool,
}
```

**The handlers:** each one becomes one SQL query:

```rust
async fn list_tasks(
    State(state): State<AppState>,
    Query(filter): Query<ListFilter>,
) -> Result<Json<Vec<Task>>, ApiError> {
    let tasks = sqlx::query_as!(
        Task,
        "SELECT id, title, done FROM tasks
         WHERE ($1::boolean IS NULL OR done = $1)
         ORDER BY id",
        filter.done
    )
    .fetch_all(&state.pool)
    .await?;
    Ok(Json(tasks))
}

async fn create_task(
    State(state): State<AppState>,
    AppJson(new_task): AppJson<NewTask>,
) -> Result<impl IntoResponse, ApiError> {
    let title = Title::parse(&new_task.title)?.into_string();
    let task = sqlx::query_as!(
        Task,
        "INSERT INTO tasks (title) VALUES ($1) RETURNING id, title, done",
        title
    )
    .fetch_one(&state.pool)
    .await?;
    let location = format!("/tasks/{}", task.id);
    Ok((
        StatusCode::CREATED,
        [(header::LOCATION, location)],
        Json(task),
    ))
}

async fn get_task(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Task>, ApiError> {
    let task = sqlx::query_as!(Task, "SELECT id, title, done FROM tasks WHERE id = $1", id)
        .fetch_optional(&state.pool)
        .await?;
    task.map(Json).ok_or(ApiError::NotFound(id))
}

async fn update_task(
    State(state): State<AppState>,
    Path(id): Path<i64>,
    AppJson(changes): AppJson<TaskChanges>,
) -> Result<Json<Task>, ApiError> {
    let title = changes
        .title
        .as_deref()
        .map(Title::parse)
        .transpose()?
        .map(Title::into_string);
    let task = sqlx::query_as!(
        Task,
        "UPDATE tasks
         SET title = COALESCE($2, title), done = COALESCE($3, done)
         WHERE id = $1
         RETURNING id, title, done",
        id,
        title,
        changes.done
    )
    .fetch_optional(&state.pool)
    .await?;
    task.map(Json).ok_or(ApiError::NotFound(id))
}

async fn delete_task(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<StatusCode, ApiError> {
    let result = sqlx::query!("DELETE FROM tasks WHERE id = $1", id)
        .execute(&state.pool)
        .await?;
    if result.rows_affected() == 0 {
        return Err(ApiError::NotFound(id));
    }
    Ok(StatusCode::NO_CONTENT)
}
```

**What happens:**

1. In `list_tasks`, `filter.done` is an `Option<bool>`; sqlx sends `None` as `NULL`. `$1::boolean IS NULL OR done = $1`
   means "no filter, or the filter matches", so one query serves both `/tasks` and `/tasks?done=false`. `::boolean`
   tells PostgreSQL the type of `$1`.
2. `create_task` validates first, exactly as before, then lets the database choose the `id` (`RETURNING`).
3. `get_task` uses `fetch_optional`: no row becomes `None`, and `ok_or` turns that into `NotFound`.
4. `update_task` uses `COALESCE(a, b)`, which returns the first value that is not `NULL`. A missing field is sent as
   `NULL`, so `COALESCE($2, title)` keeps the old title. Finding the row and changing it happen in **one** statement, and
   `RETURNING` gives the new row; if no row has this `id`, `fetch_optional` returns `None`.
5. `delete_task` uses `execute`, which reports `rows_affected()`; `0` means the task did not exist.
6. Every `.await?` turns a `sqlx::Error` into `ApiError::Database` through `#[from]`.

**`main`:** load `.env`, build the pool, give it to the router. `app()` itself does not change:

```rust
#[tokio::main]
async fn main() {
    dotenvy::dotenv().ok();
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "p16_tasks=debug,tower_http=debug".into()),
        )
        .init();

    let database_url = std::env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&database_url)
        .await
        .expect("cannot connect to the database");

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::info!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app(AppState { pool })).await.unwrap();
}
```

`acquire_timeout` limits how long a request waits for a free connection: three seconds, then an error instead of
waiting forever.

> ⚡ **Performance:** a pool is not "the more the better". Every PostgreSQL connection is a process on the database
> server with its own memory, and the server has a limit (100 by default) shared by **all** instances of your service.
> Start small (5–10 per instance) and measure.

## 7. Run it

`cargo run`, then talk to it. The tasks you created with `psql` and with `db_demo` are already there (the output
below leaves out the `date` header):

```bash
curl -s http://127.0.0.1:3000/tasks; echo
curl -i -H 'content-type: application/json' -d '{"title":"Learn PostgreSQL"}' http://127.0.0.1:3000/tasks
```

```text
[{"id":2,"title":"Write lesson","done":false},{"id":6,"title":"Read a book","done":false},{"id":7,"title":"Buy oat milk","done":true},{"id":8,"title":"Milk the cow","done":true},{"id":11,"title":"Learn sqlx","done":false}]
HTTP/1.1 201 Created
content-type: application/json
location: /tasks/12
x-request-id: 2
content-length: 49

{"id":12,"title":"Learn PostgreSQL","done":false}
```

Restart the service and ask for task 12: it is still there, because it lives in PostgreSQL now. Then see what happens
when the database disappears. Stop it **while the service runs** and send a request:

```bash
docker stop p16-postgres
curl -s http://127.0.0.1:3000/tasks/6; echo
docker start p16-postgres
```

```text
{"code":"internal","message":"internal error"}
```

The client gets the safe answer after about three seconds (`acquire_timeout`). The service log has the details:

```text
2026-11-18T07:31:54.069175Z ERROR request{method=GET uri=/tasks/6 version=HTTP/1.1}: p16_tasks::error: database error error=pool timed out while waiting for an open connection
2026-11-18T07:31:54.069283Z ERROR request{method=GET uri=/tasks/6 version=HTTP/1.1}: tower_http::trace::on_failure: response failed classification=Status code: 500 Internal Server Error latency=3002 ms
```

The service did not crash. As soon as PostgreSQL is back, the next request works again: the pool opens new
connections by itself.

## 8. Tests with a real database

The in-memory tests of Lesson 16.4 need a database now. `#[sqlx::test]` gives each test its own **fresh, empty**
database: it connects with `DATABASE_URL`, creates a new database with a generated name, passes a pool to the test,
and drops the database when the test passes. Tests never see each other's data and can run in parallel.

The fresh database has no tables. A small helper runs `schema.sql` from Lesson 16.5 first. In the `tests` module,
replace `app(AppState::default())` with this helper and turn each test into a `#[sqlx::test]` that receives a pool:

```rust
    /// Creates the tables in the empty test database that `#[sqlx::test]` gives us.
    async fn test_app(pool: PgPool) -> Router {
        sqlx::raw_sql(include_str!("../schema.sql"))
            .execute(&pool)
            .await
            .unwrap();
        app(AppState { pool })
    }

    #[sqlx::test]
    async fn created_task_can_be_read_back(pool: PgPool) {
        let app = test_app(pool).await;

        let (status, body) = send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;
        assert_eq!(status, StatusCode::CREATED);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);

        let request = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);
    }
```

Change `unknown_task_is_not_found` and `empty_title_is_rejected` the same way (`#[sqlx::test]`, a `pool: PgPool`
parameter, `let app = test_app(pool).await;`). `include_str!` puts the file's text into the program at compile time;
`raw_sql` runs several SQL commands at once. Run `cargo test`:

```text
running 7 tests
test title::tests::rejects_empty_and_blank ... ok
test title::tests::trims_spaces ... ok
test title::tests::rejects_control_characters ... ok
test title::tests::length_limit_counts_characters_not_bytes ... ok
test tests::empty_title_is_rejected ... ok
test tests::unknown_task_is_not_found ... ok
test tests::created_task_can_be_read_back ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.28s
```

(The four `title::tests` come from Exercise 1 of Lesson 16.4; the router tests of its other exercises are left out
here. If you kept them, turn them into `#[sqlx::test]` the same way.) Because every test database is new, the first task
always gets `id` 1, so the exact-JSON assertions keep working.

> 🧪 **Testing:** these tests are slower than pure unit tests (a quarter of a second here) but test the real thing:
> your SQL, your constraints, your type mapping. Lesson 16.14 goes further and starts PostgreSQL itself from the
> tests with `testcontainers`, so `cargo test` works on any machine with Docker.

## Common mistakes

**1. No `DATABASE_URL` while building.** Without `.env` (and without the variable or `.sqlx`), the macros cannot work:

```text
error: set `DATABASE_URL` to use query macros online, or run `cargo sqlx prepare` to update the query cache
```

Check that `.env` is in the package folder (next to `Cargo.toml`) and that the database is running, or build with
`SQLX_OFFLINE=true`. RustRover's own analysis needs the same, so keep the container running while you work.

**2. Unsigned identifiers.** `Path(id): Path<u64>` passed to a query:

```text
error[E0277]: the trait bound `u64: sqlx::Encode<'_, Postgres>` is not satisfied
```

PostgreSQL has no unsigned types; use `i64` for `BIGINT`. A negative `id` in the URL simply finds no row: `404`.

**3. Building SQL with `format!`.** Putting a value into the SQL text, as in
`sqlx::query(&format!("... WHERE title = '{title}'"))`, is SQL injection (Lesson 10.5). sqlx 0.9 refuses it at compile
time: the functions accept only a string literal (`&'static str`):

```text
error[E0277]: dynamic SQL strings should be audited for possible injections
    = help: the trait `SqlSafeStr` is not implemented for `&String`
```

Values go into `$1`, `$2`. Only when a part of the SQL cannot be a parameter (for example a column name chosen from a
fixed list in your code) do you build the text yourself and wrap it in `sqlx::AssertSqlSafe(text)`, which means "I have
checked that this string is safe". The macros never accept anything but a literal.

**4. Exposing database errors.** `message: error.to_string()` for a `sqlx::Error` can contain table names, SQL and
connection details. Log them; answer `internal error`.

**5. Forgetting to run `cargo sqlx prepare`.** You add a query, CI builds offline and fails with an error that the
query is not in the cache. Run `prepare` before you commit.

## Try it yourself

1. Look at one file in `.sqlx`. Find the SQL text, the column `title` with `"type_info": "Text"`, and `"nullable"`.
2. Start the service and `psql` side by side. Insert a task in `psql` and read it with `curl`; delete one with `curl`
   and check in `psql`. Two clients, one source of truth.
3. In `psql`, look for test databases while `cargo test` runs: `\l` shows names starting with `_sqlx_test`. They
   disappear when the tests pass.
4. Commit (without `.env`!): `git add code/p16-tasks && git commit -m "p16-tasks: store tasks in PostgreSQL"`.

## Exercises

### Exercise 1 — `limit` in SQL (easy)

Bring back the `limit` query parameter of Lesson 16.3 (Exercise 2), but in SQL: add `limit: Option<i64>` to
`ListFilter`, add `LIMIT $2` to the query, use 50 when it is missing and keep it between 1 and 100. Write a
`#[sqlx::test]` that creates tasks `a`, `b`, `c` and checks that `?limit=2` returns the first two.

<details>
<summary>Hint</summary>

`filter.limit.unwrap_or(50).clamp(1, 100)` gives the value; pass it as the second argument of `query_as!`. Run
`cargo sqlx prepare` again afterwards: the query text changed.

</details>

<details>
<summary>Answer</summary>

```rust
// in ListFilter:      limit: Option<i64>,
// in list_tasks:
//     "SELECT id, title, done FROM tasks
//      WHERE ($1::boolean IS NULL OR done = $1)
//      ORDER BY id
//      LIMIT $2",
//     filter.done,
//     filter.limit.unwrap_or(50).clamp(1, 100)

    #[sqlx::test]
    async fn limit_returns_only_the_first_tasks(pool: PgPool) {
        let app = test_app(pool).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"a"}"#)).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"b"}"#)).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"c"}"#)).await;

        let request = Request::get("/tasks?limit=2").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(
            body,
            r#"[{"id":1,"title":"a","done":false},{"id":2,"title":"b","done":false}]"#
        );
    }
```

Why it works: the database does the limiting, so it sends only two rows over the network instead of all of them.
`clamp(1, 100)` also refuses `limit=0` and negative values, which PostgreSQL would treat differently.

</details>

### Exercise 2 — Test `PATCH` (medium)

Write a `#[sqlx::test]` `patch_changes_only_the_given_fields`: create "Buy milk"; `PATCH` `{"done":true}` and check
that the title is unchanged; `PATCH` `{"title":"Buy oat milk"}` and check that `done` stays `true`; `PATCH` task 99 and
expect `404`.

<details>
<summary>Hint</summary>

A closure `let patch = |json: &'static str| Request::patch("/tasks/1")...` builds the requests. Each `send` needs
`app.clone()` except the last one.

</details>

<details>
<summary>Answer</summary>

```rust
    #[sqlx::test]
    async fn patch_changes_only_the_given_fields(pool: PgPool) {
        let app = test_app(pool).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;

        let patch = |json: &'static str| {
            Request::patch("/tasks/1")
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(json))
                .unwrap()
        };
        let (status, body) = send(app.clone(), patch(r#"{"done":true}"#)).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":true}"#);

        let (_, body) = send(app.clone(), patch(r#"{"title":"Buy oat milk"}"#)).await;
        assert_eq!(body, r#"{"id":1,"title":"Buy oat milk","done":true}"#);

        let missing = Request::patch("/tasks/99")
            .header(header::CONTENT_TYPE, "application/json")
            .body(Body::from(r#"{"done":true}"#))
            .unwrap();
        let (status, _) = send(app, missing).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
```

Why it works: this test checks the `COALESCE` logic, which lives in SQL, so only a test against a real database can
check it. A test with a fake store would test the fake, not the query.

</details>

### Exercise 3 — Comments of a task (hard)

Add `GET /tasks/{id}/comments`: it returns the comments of the task as
`[{"id":1,"task_id":1,"text":"Two bottles"}]`, oldest first, and `404` if the **task** does not exist (a task without
comments gives `[]`). Write a test that inserts a comment directly with `sqlx::query!` and checks both cases.

<details>
<summary>Hint</summary>

Two queries: first `SELECT EXISTS (SELECT 1 FROM tasks WHERE id = $1)`, then the comments. The macro cannot know that
`EXISTS` never returns `NULL`, so it would give `Option<bool>`; write the column name as `AS "exists!"` (the `!` tells
sqlx "not null") and use `query_scalar!`, which returns the single value of a one-column row. In the test, pass
`pool.clone()` to `test_app` so you keep a pool for the insert.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, Serialize)]
struct Comment {
    id: i64,
    task_id: i64,
    text: String,
}

async fn list_comments(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Vec<Comment>>, ApiError> {
    let exists = sqlx::query_scalar!(
        r#"SELECT EXISTS (SELECT 1 FROM tasks WHERE id = $1) AS "exists!""#,
        id
    )
    .fetch_one(&state.pool)
    .await?;
    if !exists {
        return Err(ApiError::NotFound(id));
    }
    let comments = sqlx::query_as!(
        Comment,
        "SELECT id, task_id, text FROM comments WHERE task_id = $1 ORDER BY id",
        id
    )
    .fetch_all(&state.pool)
    .await?;
    Ok(Json(comments))
}

// in app():   .route("/tasks/{id}/comments", get(list_comments))

    #[sqlx::test]
    async fn comments_of_a_task(pool: PgPool) {
        let app = test_app(pool.clone()).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;
        sqlx::query!("INSERT INTO comments (task_id, text) VALUES (1, 'Two bottles')")
            .execute(&pool)
            .await
            .unwrap();

        let request = Request::get("/tasks/1/comments").body(Body::empty()).unwrap();
        let (status, body) = send(app.clone(), request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"[{"id":1,"task_id":1,"text":"Two bottles"}]"#);

        let request = Request::get("/tasks/2/comments").body(Body::empty()).unwrap();
        let (status, _) = send(app, request).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
```

With all three exercises, `cargo test` runs ten tests:

```text
test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.35s
```

Why it works: the `404` for a missing task follows the design of Lesson 16.2 (Exercise 2): "no comments" and "no such
task" are different answers. `r#"..."#` is needed because the SQL contains double quotes. Between the two queries
another request could delete the task; Lesson 16.7 shows how a transaction removes such gaps.

</details>

## Quiz

1. Why does a service use a connection pool instead of opening a connection per request?

<details>
<summary>Answer</summary>

Opening a connection is slow (network round trips, authentication, a new server process), and the database allows
only a limited number. A pool keeps a few connections open and lends them to requests.

</details>

2. What is the difference between `sqlx::query_as(...)` and `sqlx::query_as!(...)`?

<details>
<summary>Answer</summary>

The function is checked only when it runs; mistakes appear at run time. The macro is checked at compile time against
the real database: SQL syntax, column names, parameter and result types.

</details>

3. How does `$1` with `.bind(value)` (or a macro argument) protect against SQL injection?

<details>
<summary>Answer</summary>

The SQL text and the value travel separately; the database never interprets the value as SQL. Whatever the value
contains, it stays a value.

</details>

4. What is the `.sqlx` folder for, and when must you update it?

<details>
<summary>Answer</summary>

It stores the descriptions of all compile-time checked queries, so `SQLX_OFFLINE=true` builds work without a
database (CI, Docker). Run `cargo sqlx prepare` whenever a query is added or changed, and commit the result.

</details>

5. Why does `ApiError::Database` display only `internal error`?

<details>
<summary>Answer</summary>

Database errors can contain internal details (SQL, table names, connection information). The client gets a generic
`500`; the details go to the log.

</details>

6. What does `#[sqlx::test]` give each test?

<details>
<summary>Answer</summary>

A pool connected to a new, empty database created just for that test (and dropped after it passes), so tests are
isolated and can run in parallel.

</details>

## Summary

- sqlx sends your own SQL with parameters and turns rows into Rust values; it is a driver plus helpers, not an ORM.
- A `PgPool` (from `PgPoolOptions`) shares a few connections between all requests; clone it into the app state.
- `fetch_one`, `fetch_optional`, `fetch_all` and `execute` cover the shapes of results.
- The macros `query!`, `query_as!` and `query_scalar!` check SQL and types at compile time against `DATABASE_URL`;
  `cargo sqlx prepare` and `SQLX_OFFLINE=true` make builds work without a database.
- Map `sqlx::Error` to a generic `500` and log the details; `BIGINT` is `i64`, nullable columns are `Option`.
- `#[sqlx::test]` runs each test against its own fresh database.

## Sources and further reading

- [sqlx documentation](https://docs.rs/sqlx/0.9) — the crate overview, the feature flags and every type.
- [sqlx `query!` macro](https://docs.rs/sqlx/0.9/sqlx/macro.query.html) — how compile-time checking works, type overrides like `"name!"`, and offline mode.
- [sqlx-cli README](https://github.com/launchbadge/sqlx/blob/main/sqlx-cli/README.md) — `cargo sqlx prepare`, `database` and `migrate` commands.
- [sqlx `#[sqlx::test]`](https://docs.rs/sqlx/0.9/sqlx/attr.test.html) — test databases, migrations and fixtures.
- [PostgreSQL — Connection URIs](https://www.postgresql.org/docs/18/libpq-connect.html) — every part of a connection string (section 32.1.1).
- [PostgreSQL — COALESCE](https://www.postgresql.org/docs/18/functions-conditional.html) — `COALESCE` and the other conditional expressions.

---

[🏠 Start](../START-HERE.md) · [← 16.5 SQL and PostgreSQL basics](05-sql-and-postgresql-basics.md) · [📋 Phase 16](README.md) · [16.7 Migrations and transactions →](07-migrations-and-transactions.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
