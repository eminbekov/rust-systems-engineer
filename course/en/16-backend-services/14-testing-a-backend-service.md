<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.13 Configuration, health checks and containers](13-configuration-health-checks-and-containers.md) · [📋 Phase 16](README.md) · [16.15 Project: a URL shortener service →](15-project-url-shortener-service.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/14-testing-a-backend-service.md) <!-- nav -->

# Lesson 16.14 — Testing a backend service

> **Phase 16 — Backend services** · Lesson 14 of 16 · ⏱ about 135 minutes

## What you will learn

- Which kinds of tests a backend service needs, and how the test pyramid applies to it.
- How to send HTTP (Hypertext Transfer Protocol) requests to an axum router inside a test, without a network, with `oneshot`.
- How a test can start its own PostgreSQL in Docker with `testcontainers`, so that `cargo test` needs no manual set-up.
- How `#[sqlx::test]` gives every test a fresh, migrated database on one shared server.
- How to keep tests independent, fast and honest.

## Before you start

- You finished [Lesson 16.13 — Configuration, health checks and containers](13-configuration-health-checks-and-containers.md).
- You know the test pyramid ([Lesson 09.1](../09-testing-debugging-benchmarking/01-the-test-pyramid.md)) and test doubles ([Lesson 09.3](../09-testing-debugging-benchmarking/03-test-doubles.md)).
- You know axum handlers and extractors ([Lessons 16.3](03-web-services-with-axum.md) and [16.4](04-errors-validation-and-middleware.md)), sqlx ([16.6](06-databases-with-sqlx.md)) and migrations ([16.7](07-migrations-and-transactions.md)).
- Docker is running.

## New words

| Word | Meaning |
|---|---|
| **in-process test** | A test that calls the application's code directly, in the same process, instead of through a real network connection. |
| **`tower::Service`** | The trait behind axum: "something that takes a request and returns a future with a response". An axum `Router` is a `Service`. |
| **`oneshot`** | A method from `tower::ServiceExt`: send exactly one request to a service and get its response. |
| **testcontainers** | A library that starts Docker containers from test code and removes them when the test ends. |
| **test fixture** | Everything a test needs prepared before it runs: a database, tables, some rows. |
| **test isolation** | Each test has its own data, so tests cannot change each other's results, even when they run in parallel. |
| **`#[sqlx::test]`** | An attribute from sqlx that creates a new database for one test, runs your migrations in it and passes a pool to the test. |
| **flaky test** | A test that sometimes passes and sometimes fails without a code change (Lesson 05.6). Shared data and timing are the usual causes. |

## 1. What to test in a service

Lesson 09.1 introduced the pyramid: many small fast tests at the bottom, few big slow ones at the top. For a backend service, the levels look like this:

| Level | What it checks | Needs | Speed |
|---|---|---|---|
| **Unit tests** | Pure logic: validation, parsing, calculations, configuration (Lesson 16.13) | Nothing | Microseconds |
| **Handler tests, in-process** | Routing, extractors, status codes, JSON (JavaScript Object Notation) shape | The `Router`, maybe a fake repository | Milliseconds |
| **Integration tests with a real database** | SQL (Structured Query Language) queries, migrations, constraints, transactions | PostgreSQL (in Docker) | Tens of milliseconds per test, plus start-up |
| **End-to-end tests** | The real binary over the network, with all its dependencies | Everything running | Seconds |

Most of this lesson is about the middle two levels, because they are where services break most often: a wrong status code, a query that fails on real data, a migration that does not run. Mocking the database would hide exactly those bugs, so we test against a **real** PostgreSQL, started automatically.

## 2. The service under test

A small notes API (Application Programming Interface): `POST /notes` creates a note, `GET /notes/{id}` reads one. Create it:

```bash
cd ~/projects/rust/code
cargo new p16-tested-notes
cd p16-tested-notes
cargo add axum@0.8 serde_json@1
cargo add serde@1 --features derive
cargo add sqlx@0.9 --features postgres,runtime-tokio
cargo add tokio@1 --features macros,rt-multi-thread
cargo add --dev tower@0.5 --features util
cargo add --dev testcontainers@0.28
mkdir migrations tests
```

The migration, `migrations/20261118000000_create_notes.sql`:

```sql
CREATE TABLE notes (
    id BIGSERIAL PRIMARY KEY,
    text TEXT NOT NULL
);
```

The application lives in `src/lib.rs`, so that tests can build it:

```rust
use axum::extract::{Path, State};
use axum::http::StatusCode;
use axum::routing::{get, post};
use axum::{Json, Router};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;

#[derive(Debug, PartialEq, Serialize, Deserialize, sqlx::FromRow)]
pub struct Note {
    pub id: i64,
    pub text: String,
}

#[derive(Deserialize)]
pub struct NewNote {
    pub text: String,
}

/// Builds the whole HTTP application. Tests call this too.
pub fn app(pool: PgPool) -> Router {
    Router::new()
        .route("/notes", post(create_note))
        .route("/notes/{id}", get(get_note))
        .with_state(pool)
}

async fn create_note(
    State(pool): State<PgPool>,
    Json(new): Json<NewNote>,
) -> Result<(StatusCode, Json<Note>), StatusCode> {
    if new.text.trim().is_empty() {
        return Err(StatusCode::UNPROCESSABLE_ENTITY);
    }
    let note = sqlx::query_as::<_, Note>("INSERT INTO notes (text) VALUES ($1) RETURNING id, text")
        .bind(&new.text)
        .fetch_one(&pool)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok((StatusCode::CREATED, Json(note)))
}

async fn get_note(
    State(pool): State<PgPool>,
    Path(id): Path<i64>,
) -> Result<Json<Note>, StatusCode> {
    let note = sqlx::query_as::<_, Note>("SELECT id, text FROM notes WHERE id = $1")
        .bind(id)
        .fetch_optional(&pool)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    note.map(Json).ok_or(StatusCode::NOT_FOUND)
}
```

And a short `src/main.rs` that connects, runs the migrations and serves:

```rust
use sqlx::PgPool;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = std::env::var("DATABASE_URL")?;
    let pool = PgPool::connect(&url).await?;
    sqlx::migrate!().run(&pool).await?;

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await?;
    println!("listening on {}", listener.local_addr()?);
    axum::serve(listener, p16_tested_notes::app(pool)).await?;
    Ok(())
}
```

**What happens:**

1. The key design choice is `pub fn app(pool: PgPool) -> Router`. `main` and the tests build the **same** router; only the pool differs. Everything that main does besides that (reading the environment, binding a port) is small and does not need tests of this kind.
2. We use runtime-checked queries (`query_as::<_, Note>`) with `#[derive(sqlx::FromRow)]`, so the project compiles without a database. The compile-time checked `query!` macros from Lesson 16.6 work the same way in tests.
3. The handlers return plain `StatusCode` errors to keep the example short; a real service uses the error type from Lesson 16.4.
4. `sqlx::migrate!()` embeds the files from `migrations/` into the binary at compile time (Lesson 16.7).

## 3. Calling the router without a network

An axum `Router` implements the trait `tower::Service`: it takes an HTTP `Request` and returns a future that gives a `Response`. The extension trait `tower::ServiceExt` adds `oneshot`, which sends **one** request and waits for the answer. No port, no TCP (Transmission Control Protocol) connection, no `reqwest`: the test calls the router the way the server would.

We need two small helpers in the test file: `send` runs a request and returns the status and the body as text; `post_json` builds a JSON request. You will see them in the complete `tests/api.rs` in Section 4; here is `send`:

```rust
/// Sends one request to the app, without any network, and returns status and body.
async fn send(app: &Router, request: Request<Body>) -> (StatusCode, String) {
    let response = app.clone().oneshot(request).await.unwrap();
    let status = response.status();
    let bytes = axum::body::to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap();
    (status, String::from_utf8(bytes.to_vec()).unwrap())
}
```

**What happens:**

1. `oneshot` takes the service **by value** (it consumes it), so we pass a clone. Cloning a `Router` is cheap: its routes are shared.
2. In `post_json` (Section 4), `Request::post(uri)` is the request builder from the `http` crate (re-exported as `axum::http`). Do not forget the `content-type` header: axum's `Json` extractor refuses a body without it (Exercise 1).
3. The response body is a stream of bytes; `axum::body::to_bytes` collects all of it. `usize::MAX` means "no size limit", which is fine in a test.

## 4. A real PostgreSQL for every test

The handlers need a database. **testcontainers** lets the test start one in Docker, like `docker run`, and removes the container when the test is done. Here is the complete `tests/api.rs`:

```rust
use std::time::Duration;

use axum::Router;
use axum::body::Body;
use axum::http::{Request, StatusCode};
use p16_tested_notes::{Note, app};
use sqlx::PgPool;
use testcontainers::core::{IntoContainerPort, WaitFor};
use testcontainers::runners::AsyncRunner;
use testcontainers::{ContainerAsync, GenericImage, ImageExt};
use tower::ServiceExt; // for `oneshot`

/// Starts a fresh PostgreSQL in Docker and returns the app connected to it.
/// Keep the container value alive: dropping it removes the container.
async fn test_app() -> (Router, ContainerAsync<GenericImage>) {
    let container = GenericImage::new("postgres", "18")
        .with_exposed_port(5432.tcp())
        .with_wait_for(WaitFor::message_on_stderr(
            "database system is ready to accept connections",
        ))
        .with_env_var("POSTGRES_PASSWORD", "test")
        .start()
        .await
        .expect("Docker must be running");
    let port = container.get_host_port_ipv4(5432).await.unwrap();
    let url = format!("postgres://postgres:test@127.0.0.1:{port}/postgres");

    // The server may restart once after its first start-up, so retry briefly.
    let mut attempts = 0;
    let pool = loop {
        match PgPool::connect(&url).await {
            Ok(pool) => break pool,
            Err(_) if attempts < 20 => {
                attempts += 1;
                tokio::time::sleep(Duration::from_millis(250)).await;
            }
            Err(err) => panic!("cannot connect to the test database: {err}"),
        }
    };
    sqlx::migrate!().run(&pool).await.unwrap();
    (app(pool), container)
}

/// Sends one request to the app, without any network, and returns status and body.
async fn send(app: &Router, request: Request<Body>) -> (StatusCode, String) {
    let response = app.clone().oneshot(request).await.unwrap();
    let status = response.status();
    let bytes = axum::body::to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap();
    (status, String::from_utf8(bytes.to_vec()).unwrap())
}

fn post_json(uri: &str, json: &str) -> Request<Body> {
    Request::post(uri)
        .header("content-type", "application/json")
        .body(Body::from(json.to_string()))
        .unwrap()
}

#[tokio::test]
async fn created_note_can_be_read_back() {
    let (app, _db) = test_app().await;

    let (status, body) = send(&app, post_json("/notes", r#"{"text":"buy milk"}"#)).await;
    assert_eq!(status, StatusCode::CREATED);
    let created: Note = serde_json::from_str(&body).unwrap();
    assert_eq!(created.text, "buy milk");

    let uri = format!("/notes/{}", created.id);
    let (status, body) = send(&app, Request::get(uri).body(Body::empty()).unwrap()).await;
    assert_eq!(status, StatusCode::OK);
    assert_eq!(serde_json::from_str::<Note>(&body).unwrap(), created);
}

#[tokio::test]
async fn unknown_note_is_404() {
    let (app, _db) = test_app().await;
    let request = Request::get("/notes/999").body(Body::empty()).unwrap();
    let (status, _) = send(&app, request).await;
    assert_eq!(status, StatusCode::NOT_FOUND);
}
```

```bash
cargo test --test api
```

```text
running 2 tests
test unknown_note_is_404 ... ok
test created_note_can_be_read_back ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.55s
```

**What happens:**

1. `GenericImage::new("postgres", "18")` describes the official image, like `postgres:18` on the command line. `with_env_var` sets the password, as `-e` would.
2. `with_exposed_port(5432.tcp())` publishes the container's port 5432 on a **random free port** of your Mac; `get_host_port_ipv4(5432)` tells us which one. So tests never collide on a port, and nothing else on your machine is disturbed.
3. `with_wait_for(...)` makes `start()` wait until PostgreSQL prints that it accepts connections. The official image starts a temporary server during its first set-up and then restarts, so a connection can still fail for a moment; the short retry loop covers that.
4. `sqlx::migrate!().run(&pool)` creates the tables. The test runs against the same schema as production, because it uses the same migration files.
5. `test_app` returns the container together with the router. The test keeps it in `_db`. When the test ends, `_db` is dropped and testcontainers removes the container. A name that starts with `_` silences the "unused variable" warning but still keeps the value alive until the end of the scope.
6. Each test gets **its own** container, so the tests are isolated and can run in parallel. The first run is slower because Docker downloads the image; after that, the two tests took 1.55 s together.

> 🧪 **Testing:** Look at what the first test checks: the status code, the JSON shape (by deserializing it into `Note`), and that reading returns exactly what was created. It does not check the exact id value: ids come from a database sequence, and a test that expects `id == 1` can break when tests share a database.

## 5. Faster: one server, a new database per test

A container per test costs about a second. If you have hundreds of database tests, use **`#[sqlx::test]`**: you start **one** PostgreSQL server yourself (or your CI, Continuous Integration, pipeline starts it), and sqlx creates a fresh database on it for every test, runs the migrations from `migrations/`, gives the test a `PgPool`, and deletes the database afterwards. Create `tests/with_sqlx_test.rs`:

```rust
use axum::body::Body;
use axum::http::{Request, StatusCode};
use sqlx::PgPool;
use tower::ServiceExt;

/// `#[sqlx::test]` creates an empty database for this test, runs
/// `migrations/` in it, passes a pool, and drops the database afterwards.
#[sqlx::test]
async fn empty_text_is_rejected(pool: PgPool) {
    let app = p16_tested_notes::app(pool);
    let request = Request::post("/notes")
        .header("content-type", "application/json")
        .body(Body::from(r#"{"text":"   "}"#))
        .unwrap();

    let response = app.oneshot(request).await.unwrap();

    assert_eq!(response.status(), StatusCode::UNPROCESSABLE_ENTITY);
}

#[sqlx::test]
async fn ids_start_at_one_in_every_test_database(pool: PgPool) {
    let id: i64 = sqlx::query_scalar("INSERT INTO notes (text) VALUES ('a') RETURNING id")
        .fetch_one(&pool)
        .await
        .unwrap();
    assert_eq!(id, 1);
}
```

Start one server and tell the tests where it is with `DATABASE_URL`:

```bash
docker run -d --name p16-test-db -e POSTGRES_PASSWORD=test -p 127.0.0.1:5433:5432 postgres:18
DATABASE_URL=postgres://postgres:test@127.0.0.1:5433/postgres cargo test --test with_sqlx_test
```

```text
running 2 tests
test empty_text_is_rejected ... ok
test ids_start_at_one_in_every_test_database ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.23s
```

**What happens:**

1. The attribute replaces `#[tokio::test]` and adds the `pool` argument. The user in `DATABASE_URL` must be allowed to create databases (the `postgres` user of the image is).
2. The second test shows the isolation: in every test database the first id is 1, however many tests ran before, because each test has its own fresh tables. That is why this test may check an exact id, while the testcontainers test above did not need to.
3. The two tests together took 0.23 s instead of 1.55 s, because no container had to start.
4. Port 5433 avoids a clash with a PostgreSQL you may already run on 5432 for the earlier lessons. Remove the server with `docker rm -f p16-test-db` when you are done.

Without `DATABASE_URL`, every such test fails at once (shortened):

```text
thread 'empty_text_is_rejected' (6019166) panicked at .../sqlx-postgres-0.9.1/src/testing/mod.rs:93:44:
DATABASE_URL must be set: EnvVar(NotPresent)
```

Which one should you use?

| Tool | Good for | Price |
|---|---|---|
| testcontainers | Self-contained tests: `cargo test` needs only Docker; any image (Redis, NATS, …) | ~1 s start-up per container |
| `#[sqlx::test]` | Many PostgreSQL tests, fast | Someone must start the server and set `DATABASE_URL` |
| A fake repository (Lesson 09.3) | Handler logic without any database | Does not test SQL at all |

Many projects combine them: `#[sqlx::test]` for the many database tests, with the server started by a testcontainers helper or by the CI pipeline (Lesson 09.12), and fakes where the database is not the point.

> 🔒 **Security:** Test credentials such as `POSTGRES_PASSWORD=test` are fine because the containers listen only on your machine and live only for the test. Never point `DATABASE_URL` for tests at a shared or production database: `#[sqlx::test]` creates and drops databases there.

## Common mistakes

> ⚠️ **Common mistake:** Writing `let (app, _) = test_app().await;`. The plain `_` is not a variable: the container is dropped **immediately**, and every request then fails. The test waited for the pool's 30-second timeout and failed:

```text
thread 'unknown_note_is_404' (6038758) panicked at tests/api.rs:81:5:
assertion `left == right` failed
  left: 500
```

Fix: bind it to a name, `_db`, so it lives until the end of the test.

> ⚠️ **Common mistake:** Docker is not running. testcontainers cannot start anything, and `expect("Docker must be running")` makes the reason clear. Here `DOCKER_HOST` pointed at a socket that does not exist:

```text
thread 'unknown_note_is_404' (6050912) panicked at tests/api.rs:24:10:
Docker must be running: Client(Init(SocketNotFoundError("/tmp/no-docker-here.sock")))
```

> ⚠️ **Common mistake:** Tests that share one database and expect exact data, such as "there are exactly 2 notes". They pass alone and fail together, or fail in a different order: **flaky tests**. Give every test its own database, or make it check only the data it created.

> ⚠️ **Common mistake:** Testing only the happy path. Most production incidents in services are on error paths: invalid input, missing rows, constraint violations. Every handler deserves at least one test for each status code it can return.

## Try it yourself

1. Run `cargo test` (all test files) with the server from Section 5 running and `DATABASE_URL` set. Count the tests: there are four.
2. While `cargo test --test api` runs, look at `docker ps` in another terminal: you see short-lived `postgres:18` containers with random names, which disappear when the tests finish.
3. Remove the line `sqlx::migrate!().run(&pool).await.unwrap();` from `test_app` and run the tests again. Both tests now fail with `left: 500`, because the table does not exist. Put the line back.

## Exercises

### Exercise 1 — What does `Json` reject? (easy)

Predict the status code for three bad requests to `POST /notes`, then write one `#[sqlx::test]` test that checks all three: (a) a correct JSON body but **no** `content-type` header; (b) `content-type: application/json` with the broken body `{"text":`; (c) valid JSON with the wrong field, `{"title":"a"}`.

<details>
<summary>Hint</summary>

These requests never reach your handler code: the `Json` extractor rejects them first (Lesson 16.4). A small helper `status_for(pool, request)` that returns only the status keeps the test short.

</details>

<details>
<summary>Answer</summary>

(a) `415 Unsupported Media Type`, (b) `400 Bad Request`, (c) `422 Unprocessable Entity`. Put the test in `tests/exercises.rs`:

```rust
use axum::body::Body;
use axum::http::{Request, StatusCode};
use p16_tested_notes::{Note, app};
use sqlx::PgPool;
use tower::ServiceExt;

async fn status_for(pool: PgPool, request: Request<Body>) -> StatusCode {
    app(pool).oneshot(request).await.unwrap().status()
}

#[sqlx::test]
async fn bad_bodies_are_rejected_by_the_json_extractor(pool: PgPool) {
    let no_type = Request::post("/notes")
        .body(Body::from(r#"{"text":"a"}"#))
        .unwrap();
    assert_eq!(
        status_for(pool.clone(), no_type).await,
        StatusCode::UNSUPPORTED_MEDIA_TYPE
    );

    let broken = Request::post("/notes")
        .header("content-type", "application/json")
        .body(Body::from(r#"{"text":"#))
        .unwrap();
    assert_eq!(
        status_for(pool.clone(), broken).await,
        StatusCode::BAD_REQUEST
    );

    let wrong_field = Request::post("/notes")
        .header("content-type", "application/json")
        .body(Body::from(r#"{"title":"a"}"#))
        .unwrap();
    assert_eq!(
        status_for(pool, wrong_field).await,
        StatusCode::UNPROCESSABLE_ENTITY
    );
}
```

(`Note` is used by the next exercise; if you do only this one, remove it from the `use` line.)

Why it works: axum's `Json` checks the header first (415), then the JSON syntax (400), then whether the data fits `NewNote` (422). Knowing these codes helps you write correct API documentation, and the test makes sure an axum upgrade does not change them silently.

</details>

### Exercise 2 — List all notes (medium)

Add `GET /notes`, which returns all notes ordered by id as a JSON array. Write a `#[sqlx::test]` test that inserts two notes directly with SQL and checks that the endpoint returns their texts in order.

<details>
<summary>Hint</summary>

A route can have several methods: `.route("/notes", post(create_note).get(list_notes))`. Use `fetch_all` and return `Json<Vec<Note>>`. In the test, deserialize the body with `serde_json::from_slice::<Vec<Note>>`.

</details>

<details>
<summary>Answer</summary>

In `src/lib.rs`:

```rust
async fn list_notes(State(pool): State<PgPool>) -> Result<Json<Vec<Note>>, StatusCode> {
    let notes = sqlx::query_as::<_, Note>("SELECT id, text FROM notes ORDER BY id")
        .fetch_all(&pool)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(notes))
}
```

and the route `.route("/notes", post(create_note).get(list_notes))`. The test:

```rust
#[sqlx::test]
async fn list_returns_all_notes_in_order(pool: PgPool) {
    for text in ["first", "second"] {
        sqlx::query("INSERT INTO notes (text) VALUES ($1)")
            .bind(text)
            .execute(&pool)
            .await
            .unwrap();
    }
    let response = app(pool)
        .oneshot(Request::get("/notes").body(Body::empty()).unwrap())
        .await
        .unwrap();
    assert_eq!(response.status(), StatusCode::OK);
    let bytes = axum::body::to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap();
    let notes: Vec<Note> = serde_json::from_slice(&bytes).unwrap();
    let texts: Vec<&str> = notes.iter().map(|n| n.text.as_str()).collect();
    assert_eq!(texts, ["first", "second"]);
}
```

Why it works: inserting the fixture rows with SQL keeps the test focused on the list endpoint (it does not depend on `POST` working). `ORDER BY id` makes the order defined; without it, PostgreSQL may return rows in any order, and the test would be flaky in principle.

</details>

### Exercise 3 — A database rule becomes a 422 (hard)

Add a second migration that limits notes to 200 characters with a `CHECK` constraint. Right now a longer note would make the `INSERT` fail and the handler return `500`. Change the handler so that a check violation becomes `422`, and write the test first (it must fail with 500 before your fix).

<details>
<summary>Hint</summary>

The migration: `ALTER TABLE notes ADD CONSTRAINT text_not_too_long CHECK (length(text) <= 200);`. In `map_err`, match `sqlx::Error::Database(db)` and ask `db.is_check_violation()`. Build a long text with `"x".repeat(201)` and `serde_json::json!`.

</details>

<details>
<summary>Answer</summary>

`migrations/20261118000100_limit_note_length.sql`:

```sql
ALTER TABLE notes ADD CONSTRAINT text_not_too_long CHECK (length(text) <= 200);
```

In `create_note`, replace the `map_err` of the `INSERT`:

```rust
        .map_err(|err| match err {
            sqlx::Error::Database(db) if db.is_check_violation() => {
                StatusCode::UNPROCESSABLE_ENTITY
            }
            _ => StatusCode::INTERNAL_SERVER_ERROR,
        })?;
```

The test:

```rust
#[sqlx::test]
async fn too_long_text_is_422_not_500(pool: PgPool) {
    let json = serde_json::json!({ "text": "x".repeat(201) }).to_string();
    let request = Request::post("/notes")
        .header("content-type", "application/json")
        .body(Body::from(json))
        .unwrap();
    assert_eq!(
        status_for(pool, request).await,
        StatusCode::UNPROCESSABLE_ENTITY
    );
}
```

Why it works: the constraint makes the database the final guard of the rule, even for data that does not come through this handler. PostgreSQL reports error class `23514` (check violation); sqlx turns it into `is_check_violation() == true`, and the handler maps it to a client error instead of "server failed". Only a test with a real database can catch this; a fake repository knows nothing about constraints.

</details>

## Quiz

1. Why does the lesson test against a real PostgreSQL instead of a mock database?

<details>
<summary>Answer</summary>

Many service bugs live in SQL, migrations and constraints. A mock would accept any query and hide them. A real database in a container is still fast and fully automatic.

</details>

2. What does `oneshot` do, and why do we call it on `app.clone()`?

<details>
<summary>Answer</summary>

It sends one request to a `tower::Service` (our `Router`) and returns the response, without any network. It consumes the service, so we pass a clone; cloning a `Router` is cheap.

</details>

3. Why does `test_app` return the container, and what happens if the test does not keep it?

<details>
<summary>Answer</summary>

Dropping the `ContainerAsync` value removes the container. If the test does not keep it (for example with `let (app, _) = ...`), the database disappears at once and every query fails.

</details>

4. How does `#[sqlx::test]` keep tests isolated, and what does it need?

<details>
<summary>Answer</summary>

It creates a new database for each test on the server from `DATABASE_URL`, runs the migrations there, and drops it afterwards. It needs a running server and a user allowed to create databases.

</details>

5. A test expects `id == 1` and passes alone, but fails when run with other tests. What is wrong?

<details>
<summary>Answer</summary>

The tests share a database, so another test already used id 1: a flaky test caused by shared data. Give each test its own database, or do not assert values that depend on other tests.

</details>

## Summary

- Test a service at several levels: pure logic in unit tests, routing and status codes in-process, SQL against a real database, and a few end-to-end tests.
- Build the router in one function, `app(pool)`, used by `main` and by the tests.
- `tower::ServiceExt::oneshot` sends a request to the router without a network; collect the body with `axum::body::to_bytes`.
- testcontainers starts real containers from the test on random ports and removes them on drop; keep the container value alive.
- `#[sqlx::test]` gives each test a fresh, migrated database on one server: fast and isolated.
- Isolated tests check only their own data; test the error paths, not only the happy path.

## Sources and further reading

- [axum — testing example](https://github.com/tokio-rs/axum/tree/main/examples/testing) — official examples of `oneshot` tests and tests over a real socket.
- [tower::ServiceExt on docs.rs](https://docs.rs/tower/latest/tower/trait.ServiceExt.html) — `oneshot` and the other helper methods.
- [testcontainers for Rust](https://rust.testcontainers.org/) — the official guide: images, wait conditions, ports, cleanup.
- [sqlx::test on docs.rs](https://docs.rs/sqlx/latest/sqlx/attr.test.html) — all options of the attribute, including fixtures and custom migrations.
- [PostgreSQL — Error codes (Appendix A)](https://www.postgresql.org/docs/current/errcodes-appendix.html) — the error classes such as `23514` check violation.

---

[🏠 Start](../START-HERE.md) · [← 16.13 Configuration, health checks and containers](13-configuration-health-checks-and-containers.md) · [📋 Phase 16](README.md) · [16.15 Project: a URL shortener service →](15-project-url-shortener-service.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
