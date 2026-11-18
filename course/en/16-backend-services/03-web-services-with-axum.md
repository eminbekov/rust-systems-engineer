<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.2 Designing HTTP APIs: REST](02-designing-http-apis.md) · [📋 Phase 16](README.md) · [16.4 Errors, validation and middleware →](04-errors-validation-and-middleware.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/03-web-services-with-axum.md) <!-- nav -->

# Lesson 16.3 — Web services with axum

> **Phase 16 — Backend services** · Lesson 3 of 16 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to build the tasks API (Application Programming Interface) of Lesson 16.2 with `axum`: routes, handlers and the methods `get`, `post`,
  `patch`, `delete`.
- How *extractors* (`Path`, `Query`, `Json`, `State`) turn parts of a request into typed Rust values.
- How a handler builds a response: a status code, headers and a JSON (JavaScript Object Notation) body.
- How to share data between requests with `State` and `Arc<Mutex<…>>`.
- How to test the whole router in memory, without opening a port.

## Before you start

- You finished [Lesson 16.2 — Designing HTTP APIs: REST](02-designing-http-apis.md). Keep its contract
  table open: this lesson implements it.
- You know `Arc` and `Mutex` ([Lesson 05.10 — Threads, channels and shared state](../05-rust-intermediate/10-threads-channels-and-shared-state.md))
  and why a lock must not be held across `.await` ([Lesson 06.9](../06-rust-advanced/09-async-await-introduction.md), section 8).
- You know `serde` derives from [Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md).

## New words

| Word | Meaning |
|---|---|
| **extractor** | A type in a handler's parameter list that axum fills from the request: `Path` from the path, `Query` from the query string, `Json` from the body, `State` from the shared state. |
| **rejection** | The error response axum sends by itself when an extractor fails, for example `400` for broken JSON. Your handler does not run. |
| **`IntoResponse`** | The axum trait for "can be turned into an HTTP (HyperText Transfer Protocol) response". Handlers return types that implement it: text, `Json`, `StatusCode`, tuples, `Result`. |
| **application state** | Data that all requests share, such as the task store or (later) a database connection pool. In axum you attach it with `.with_state(...)` and read it with `State`. |
| **tower** | A crate that defines `Service`: "something that takes a request and asynchronously returns a response". An axum `Router` is a tower `Service`. |
| **`oneshot`** | A helper from tower that sends exactly one request to a service and returns its response. Tests use it to call the router without a network. |

## 1. The package

The tasks service lives in its own package, which you grow until Lesson 16.8:

```bash
cd ~/projects/rust
cargo new code/p16-tasks
cd code/p16-tasks
cargo add axum@0.8
cargo add tokio@1 --features full
cargo add serde@1 --features derive
cargo add --dev tower@0.5 --features util
cargo add --dev http-body-util@0.1
```

`Cargo.toml` now contains:

```toml
[dependencies]
axum = "0.8"
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
http-body-util = "0.1"
tower = { version = "0.5", features = ["util"] }
```

`tower` and `http-body-util` are only needed by the tests (Section 8), so they are **dev-dependencies**.
Move the `API.md` you wrote in Lesson 16.2 into this folder.

## 2. The whole service

Replace `src/main.rs` with the following program. Read it once from top to bottom; the next sections explain
each part.

```rust
use std::collections::BTreeMap;
use std::sync::{Arc, Mutex};

use axum::{
    Json, Router,
    extract::{Path, Query, State},
    http::{StatusCode, header},
    response::IntoResponse,
    routing::get,
};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize)]
struct Task {
    id: u64,
    title: String,
    done: bool,
}

#[derive(Debug, Deserialize)]
struct NewTask {
    title: String,
}

#[derive(Debug, Deserialize)]
struct TaskChanges {
    title: Option<String>,
    done: Option<bool>,
}

#[derive(Debug, Deserialize)]
struct ListFilter {
    done: Option<bool>,
}

#[derive(Debug, Default)]
struct Store {
    last_id: u64,
    tasks: BTreeMap<u64, Task>,
}

type AppState = Arc<Mutex<Store>>;

async fn list_tasks(
    State(state): State<AppState>,
    Query(filter): Query<ListFilter>,
) -> Json<Vec<Task>> {
    let store = state.lock().unwrap();
    let tasks = store
        .tasks
        .values()
        .filter(|task| filter.done.is_none_or(|done| task.done == done))
        .cloned()
        .collect();
    Json(tasks)
}

async fn create_task(
    State(state): State<AppState>,
    Json(new_task): Json<NewTask>,
) -> impl IntoResponse {
    let mut store = state.lock().unwrap();
    store.last_id += 1;
    let task = Task {
        id: store.last_id,
        title: new_task.title,
        done: false,
    };
    store.tasks.insert(task.id, task.clone());
    let location = format!("/tasks/{}", task.id);
    (StatusCode::CREATED, [(header::LOCATION, location)], Json(task))
}

async fn get_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<Json<Task>, StatusCode> {
    let store = state.lock().unwrap();
    match store.tasks.get(&id) {
        Some(task) => Ok(Json(task.clone())),
        None => Err(StatusCode::NOT_FOUND),
    }
}

async fn update_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(changes): Json<TaskChanges>,
) -> Result<Json<Task>, StatusCode> {
    let mut store = state.lock().unwrap();
    let task = store.tasks.get_mut(&id).ok_or(StatusCode::NOT_FOUND)?;
    if let Some(title) = changes.title {
        task.title = title;
    }
    if let Some(done) = changes.done {
        task.done = done;
    }
    Ok(Json(task.clone()))
}

async fn delete_task(State(state): State<AppState>, Path(id): Path<u64>) -> StatusCode {
    let mut store = state.lock().unwrap();
    match store.tasks.remove(&id) {
        Some(_) => StatusCode::NO_CONTENT,
        None => StatusCode::NOT_FOUND,
    }
}

async fn health() -> &'static str {
    "ok\n"
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health))
        .route("/tasks", get(list_tasks).post(create_task))
        .route(
            "/tasks/{id}",
            get(get_task).patch(update_task).delete(delete_task),
        )
        .with_state(state)
}

#[tokio::main]
async fn main() {
    let state = AppState::default();
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app(state)).await.unwrap();
}
```

**What happens:**

1. `Task` is what the API sends (`Serialize`). `NewTask` is the body of `POST /tasks`, `TaskChanges` the body
   of `PATCH` (both `Deserialize`). In `TaskChanges` both fields are `Option`: a missing field becomes `None`,
   which means "do not change it".
2. `ListFilter` is the query string of `GET /tasks`: `?done=false` becomes `ListFilter { done: Some(false) }`,
   no query string becomes `ListFilter { done: None }`.
3. `Store` keeps the tasks in a `BTreeMap`, which keeps keys sorted, so the list comes out oldest first, as the
   contract says. `last_id` gives every new task the next number.
4. `AppState` is a short name for `Arc<Mutex<Store>>`: one store, shared by all requests.
5. There is one handler per row of the contract. `app` connects them: `get(list_tasks).post(create_task)`
   puts two methods on the same path. `{id}` in `"/tasks/{id}"` is the path parameter from Lesson 16.2.
6. `app` is a separate function so that `main` and the tests (Section 8) build exactly the same router.

Run it with `cargo run`; it prints `listening on http://127.0.0.1:3000`. Section 6 talks to it.

## 3. Extractors: from request to typed values

Look at a handler's parameters:

```rust
async fn update_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(changes): Json<TaskChanges>,
) -> Result<Json<Task>, StatusCode>
```

You never parse anything yourself. Each parameter type is an **extractor**, and axum fills it before your
handler runs:

| Extractor | Takes from the request | Example |
|---|---|---|
| `Path<u64>` | the `{id}` part of the path, parsed as `u64` | `/tasks/7` → `7` |
| `Query<ListFilter>` | the query string, deserialized with serde | `?done=false` → `ListFilter { done: Some(false) }` |
| `Json<TaskChanges>` | the body, checked for `content-type: application/json`, then deserialized | `{"done":true}` → `TaskChanges { title: None, done: Some(true) }` |
| `State<AppState>` | the shared state given to `.with_state(...)` | the `Arc<Mutex<Store>>` |

`State(state): State<AppState>` is a pattern (Lesson 06.3): it unpacks the wrapper struct `State` so that the
variable `state` is the `AppState` inside.

If an extractor cannot do its job (the path says `abc` instead of a number, the JSON is broken), axum sends a
**rejection** response at once, and your handler never runs. This is a big win: inside the handler, `id` is
**always** a valid `u64`. Section 7 shows the rejections.

> ⚠️ **Common mistake:** the body can be read only once, so the extractor that reads it (`Json`, `String`,
> `Bytes`) must be the **last** parameter. All the others may come in any order. The compiler error for a
> wrong order is confusing; see "Common mistakes".

## 4. Responses: anything that implements `IntoResponse`

A handler can return any type that implements `IntoResponse`. The ones used here:

| Return value | Response |
|---|---|
| `&'static str`, `String` | `200`, `content-type: text/plain; charset=utf-8`, the text |
| `Json(value)` | `200`, `content-type: application/json`, `value` serialized with serde |
| `StatusCode::NO_CONTENT` | that status, empty body |
| `(StatusCode, [(header, value)], Json(task))` | the status, extra headers, the JSON body |
| `Result<A, B>` where both implement `IntoResponse` | `A` for `Ok`, `B` for `Err` |

`create_task` returns the tuple form: `201 Created`, a `location` header and the new task. Its return type is
`impl IntoResponse`, which means "some type that implements `IntoResponse`; the compiler knows which". That
saves you from writing the long tuple type.

`get_task` returns `Result<Json<Task>, StatusCode>`. Because `Result` implements `IntoResponse`, the `?`
operator works in handlers: in `update_task`, `.ok_or(StatusCode::NOT_FOUND)?` turns a missing task into a
`404` response and returns early. For now an error is only a bare status code; Lesson 16.4 turns errors
into the JSON error body of the contract.

## 5. Shared state

Every request runs in its own task, maybe on a different thread (Lesson 16.1). They must all see **the same**
store, so the store sits behind an `Arc` (shared ownership between tasks) and a `Mutex` (one task at a time
may change it). `.with_state(state)` gives the router a copy of the `Arc`; for every request axum clones the
`Arc` (cheap: it only increases a counter) and hands it to the `State` extractor.

```text
  request 1 ─▶ task ─▶ State(state) ─┐
  request 2 ─▶ task ─▶ State(state) ─┼─▶ Arc ─▶ Mutex ─▶ Store { last_id, tasks }
  request 3 ─▶ task ─▶ State(state) ─┘
```

This is the standard `std::sync::Mutex`, not an async one. That is fine **because no handler holds the lock
across an `.await`**: each one locks, changes the map, and drops the guard at the end of the function. A
blocking lock that is held for a few microseconds does not hurt the runtime.

> ⚡ **Performance:** keep critical sections tiny: lock, copy or change what you need, unlock. Never do slow
> work (a network call, a database query) while you hold a lock.

Remember Lesson 16.1: this store lives in the process memory. Restart the service and all tasks are gone,
and two instances would have two different stores. Lessons 16.5 to 16.7 move the tasks into PostgreSQL.

## 6. Talking to the service

With the service running, use a second terminal. Create a task:

```bash
curl -i -H 'content-type: application/json' -d '{"title":"Buy milk"}' http://127.0.0.1:3000/tasks
```

```text
HTTP/1.1 201 Created
content-type: application/json
location: /tasks/1
content-length: 40
date: Wed, 18 Nov 2026 07:08:46 GMT

{"id":1,"title":"Buy milk","done":false}
```

Now walk through the rest of the contract (`-s` hides the progress bar, `echo` adds a line break after the
JSON):

```bash
curl -s -H 'content-type: application/json' -d '{"title":"Write lesson"}' http://127.0.0.1:3000/tasks; echo
curl -s http://127.0.0.1:3000/tasks; echo
curl -s -X PATCH -H 'content-type: application/json' -d '{"done":true}' http://127.0.0.1:3000/tasks/1; echo
curl -s 'http://127.0.0.1:3000/tasks?done=false'; echo
```

```text
{"id":2,"title":"Write lesson","done":false}
[{"id":1,"title":"Buy milk","done":false},{"id":2,"title":"Write lesson","done":false}]
{"id":1,"title":"Buy milk","done":true}
[{"id":2,"title":"Write lesson","done":false}]
```

The URL (Uniform Resource Locator) with `?` is in quotes because the shell gives `?` a special meaning. Finally, delete task 1 and try
to read it:

```bash
curl -i -X DELETE http://127.0.0.1:3000/tasks/1
curl -i http://127.0.0.1:3000/tasks/1
```

```text
HTTP/1.1 204 No Content
date: Wed, 18 Nov 2026 07:08:46 GMT

HTTP/1.1 404 Not Found
content-length: 0
date: Wed, 18 Nov 2026 07:08:46 GMT
```

This is exactly the conversation from Lesson 16.2, Section 5, except that the `404` still has an empty body.

## 7. What axum does with bad input

Send some broken requests, as an attacker or a buggy client would:

```bash
curl -i -H 'content-type: application/json' -d '{"title":' http://127.0.0.1:3000/tasks
curl -i -d '{"title":"x"}' http://127.0.0.1:3000/tasks
curl -i -H 'content-type: application/json' -d '{"name":"x"}' http://127.0.0.1:3000/tasks
curl -i http://127.0.0.1:3000/tasks/abc
curl -i 'http://127.0.0.1:3000/tasks?done=maybe'
```

The answers, without the `date` and `content-length` lines:

```text
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Failed to parse the request body as JSON: title: EOF while parsing a value at line 1 column 9

HTTP/1.1 415 Unsupported Media Type
content-type: text/plain; charset=utf-8

Expected request with `Content-Type: application/json`

HTTP/1.1 422 Unprocessable Entity
content-type: text/plain; charset=utf-8

Failed to deserialize the JSON body into the target type: missing field `title` at line 1 column 12

HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Invalid URL: Cannot parse `abc` to a `u64`

HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Failed to deserialize query string: done: provided string was not `true` or `false`
```

**What happens:**

1. Broken JSON → the `Json` extractor rejects with `400`.
2. Without `-H 'content-type: application/json'`, `curl -d` sends the type `application/x-www-form-urlencoded`
   (the format of forms in HTML, the HyperText Markup Language of web pages), so `Json` rejects with `415`.
3. Valid JSON, but without the field `title` → `422`. (hyper prints the older name of `422`,
   "Unprocessable Entity"; RFC (Request for Comments) 9110 calls it "Unprocessable Content". The number is what counts.)
4. `abc` is not a `u64` → the `Path` extractor rejects with `400`.
5. `maybe` is not a `bool` → the `Query` extractor rejects with `400`.

The status codes already match the contract, and no handler ran for any of these. Two things are still
wrong: the bodies are plain text, not our JSON error format, and nothing stops an **empty** title
(`{"title":""}` creates a task with title `""`). Lesson 16.4 fixes both.

Stop the service with `Ctrl+C`.

## 8. Testing the router without a network

A `Router` is a tower `Service`: something that takes a request and returns a response. tower's
`ServiceExt::oneshot` sends one request **directly**, in memory: no port, no `curl`, very fast. Add this at
the end of `src/main.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use axum::http::Request;
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    /// Sends one request to the router and returns the status and the body as text.
    async fn send(app: Router, request: Request<Body>) -> (StatusCode, String) {
        let response = app.oneshot(request).await.unwrap();
        let status = response.status();
        let bytes = response.into_body().collect().await.unwrap().to_bytes();
        (status, String::from_utf8(bytes.to_vec()).unwrap())
    }

    fn post_json(uri: &str, json: &'static str) -> Request<Body> {
        Request::post(uri)
            .header(header::CONTENT_TYPE, "application/json")
            .body(Body::from(json))
            .unwrap()
    }

    #[tokio::test]
    async fn created_task_can_be_read_back() {
        let app = app(AppState::default());

        let (status, body) = send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;
        assert_eq!(status, StatusCode::CREATED);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);

        let request = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);
    }

    #[tokio::test]
    async fn unknown_task_is_not_found() {
        let app = app(AppState::default());
        let request = Request::get("/tasks/42").body(Body::empty()).unwrap();
        let (status, _body) = send(app, request).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
}
```

```text
running 2 tests
test tests::unknown_task_is_not_found ... ok
test tests::created_task_can_be_read_back ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. `Request::post(uri).header(...).body(...)` builds an HTTP request as a Rust value (the type comes from the
   `http` crate, which axum re-exports as `axum::http`).
2. `app.oneshot(request)` consumes the router and returns the response. That is why the first call uses
   `app.clone()`: a clone of the router shares the same `Arc`, so the second request sees the task the first one
   created.
3. The body of a response arrives as a stream of pieces. `collect()` (from `http_body_util::BodyExt`) waits for
   all of them, and `to_bytes()` joins them.
4. Each test creates its own `AppState::default()`, so tests never see each other's tasks and can run in parallel.

> 🧪 **Testing:** these tests go through routing, extractors, handlers and serialization: everything except the
> TCP (Transmission Control Protocol) socket. That makes them a good middle level of the test pyramid (Lesson 09.1). Write one for every row of
> the contract table.

## Common mistakes

**1. The body extractor is not last.** You write `Json(new_task): Json<NewTask>` before `State(...)`:

```text
error[E0277]: the trait bound `fn(Json<NewTask>, State<_>) -> _ {create_task}: Handler<_, _>` is not satisfied
   --> src/main.rs:116:47
    |
116 |         .route("/tasks", get(list_tasks).post(create_task))
    |                                          ---- ^^^^^^^^^^^ unsatisfied trait bound
    = note: Consider using `#[axum::debug_handler]` to improve the error message
```

The message only says "this function is not a handler". To get a better one, enable the `macros` feature
(`cargo add axum@0.8 --features macros`) and put `#[axum::debug_handler]` above the function. Now the
compiler says exactly what is wrong:

```text
error: `Json<_>` consumes the request body and thus must be the last argument to the handler function
  --> src/main.rs:60:21
   |
60 |     Json(new_task): Json<NewTask>,
   |                     ^^^^
```

Fix: move the `Json` parameter to the end. The same "not a handler" error also appears when a parameter type
is not an extractor, or when the future of the handler is not `Send` (for example, a `MutexGuard` kept across an
`.await`).

**2. The old path syntax.** Older axum versions (and many blog posts) write `"/tasks/:id"`. axum 0.8 refuses it
when the router is built, so the program compiles but panics at start:

```text
listening on http://127.0.0.1:3000

thread 'main' (5506020) panicked at src/main.rs:117:10:
Path segments must not start with `:`. For capture groups, use `{capture}`. If you meant to literally match a segment starting with a colon, call `without_v07_checks` on the router.
```

Fix: `"/tasks/{id}"`.

**3. Forgetting `.with_state(state)`.** A router whose handlers use `State<AppState>` is a
`Router<AppState>`: "a router that still needs an `AppState`". Only after `.with_state(...)` does it become a
plain `Router`, which `axum::serve` accepts:

```text
error[E0308]: mismatched types
   --> src/main.rs:114:5
    |
113 |   fn app(state: AppState) -> Router {
    |                              ------ expected `Router` because of return type
...
    = note: expected struct `Router<()>`
               found struct `Router<Arc<std::sync::Mutex<Store>>>`
```

## Try it yourself

1. Start the service again and create two tasks. Mark task 2 done, then send `PATCH /tasks/2` with
   `{"title":"Write lesson 16.3"}` only. Check that `done` is still `true`.
2. Restart the service and run `curl -s http://127.0.0.1:3000/tasks`. The list is empty (`[]`): the state lived in
   memory.
3. Run `cargo test` and `cargo clippy`, then commit: `git add code/p16-tasks && git commit -m "p16-tasks: in-memory tasks API"`.

## Exercises

### Exercise 1 — Test that delete works (easy)

Write a test `deleted_task_is_gone_and_delete_is_idempotent`: create a task, `DELETE /tasks/1` (expect `204`),
`GET /tasks/1` (expect `404`), and `DELETE /tasks/1` again (expect `404`). Use the `send` and `post_json` helpers.

<details>
<summary>Hint</summary>

`Request::delete("/tasks/1").body(Body::empty()).unwrap()` builds the request. You need it twice; a small
closure `let delete = || Request::delete(...)...;` saves typing.

</details>

<details>
<summary>Answer</summary>

```rust
    #[tokio::test]
    async fn deleted_task_is_gone_and_delete_is_idempotent() {
        let app = app(AppState::default());
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;

        let delete = || Request::delete("/tasks/1").body(Body::empty()).unwrap();
        let (status, _) = send(app.clone(), delete()).await;
        assert_eq!(status, StatusCode::NO_CONTENT);

        let get = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, _) = send(app.clone(), get).await;
        assert_eq!(status, StatusCode::NOT_FOUND);

        // The second DELETE finds nothing, but the end state is the same: no task 1.
        let (status, _) = send(app, delete()).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
```

Why it works: the router clones share one store, so the requests see each other's effects. The status of the
second `DELETE` is different (`404`), but the **state** is the same as after the first one. That is what
idempotent means (Lesson 16.2).

</details>

### Exercise 2 — A `limit` for the list (medium)

Add an optional query parameter `limit` to `GET /tasks`: return at most `limit` tasks, 50 when it is missing,
and never more than 100. Write a test that creates three tasks and checks that `?limit=2` returns the first two.

<details>
<summary>Hint</summary>

Add `limit: Option<usize>` to `ListFilter`. In the iterator chain, `.take(n)` stops after `n` items.
`unwrap_or(50).min(100)` gives the rule in one expression.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, Deserialize)]
struct ListFilter {
    done: Option<bool>,
    limit: Option<usize>,
}

// in list_tasks, after .filter(...): .take(filter.limit.unwrap_or(50).min(100))

    #[tokio::test]
    async fn limit_returns_only_the_first_tasks() {
        let app = app(AppState::default());
        for title in ["a", "b", "c"] {
            let body = format!(r#"{{"title":"{title}"}}"#);
            let request = Request::post("/tasks")
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(body))
                .unwrap();
            send(app.clone(), request).await;
        }
        let request = Request::get("/tasks?limit=2").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(
            body,
            r#"[{"id":1,"title":"a","done":false},{"id":2,"title":"b","done":false}]"#
        );
    }
```

Why it works: `Query` deserializes `limit=2` into `Some(2)`; a missing `limit` is `None`. The cap of 100 protects
the service from `?limit=100000000` (Lesson 10.4: limits on everything). In `format!`, `{{` and `}}` are literal
braces.

</details>

### Exercise 3 — `PUT` replaces a task (hard)

Add `PUT /tasks/{id}` with the body `{"title": "...", "done": true}`: both fields are **required**, and the
task is replaced completely. Answer `200` with the task, or `404`. Write a test that sends the same `PUT`
twice and checks that both answers are equal: `PUT` is idempotent.

<details>
<summary>Hint</summary>

Create a new struct `ReplaceTask { title: String, done: bool }` (no `Option`, so a missing field gives `422`).
Chain `.put(replace_task)` onto the method router of `"/tasks/{id}"`. Tuples of `(StatusCode, String)` can be
compared with `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, Deserialize)]
struct ReplaceTask {
    title: String,
    done: bool,
}

async fn replace_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(new): Json<ReplaceTask>,
) -> Result<Json<Task>, StatusCode> {
    let mut store = state.lock().unwrap();
    let task = store.tasks.get_mut(&id).ok_or(StatusCode::NOT_FOUND)?;
    task.title = new.title;
    task.done = new.done;
    Ok(Json(task.clone()))
}

// in app(): get(get_task).put(replace_task).patch(update_task).delete(delete_task)

    #[tokio::test]
    async fn put_twice_gives_the_same_result() {
        let app = app(AppState::default());
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;

        let put = || {
            Request::put("/tasks/1")
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(r#"{"title":"Buy oat milk","done":true}"#))
                .unwrap()
        };
        let first = send(app.clone(), put()).await;
        let second = send(app.clone(), put()).await;
        assert_eq!(first, second);
        assert_eq!(first.0, StatusCode::OK);
        assert_eq!(first.1, r#"{"id":1,"title":"Buy oat milk","done":true}"#);
    }
```

All five tests (the two from Section 8 and the three from the exercises) pass:

```text
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `PUT` sets every field to a value given by the client, so the result does not depend on how many
times it runs. A `PATCH` with `Option` fields and a `PUT` with required fields are two different contracts; the
types make the difference visible.

</details>

## Quiz

1. What is an extractor? Name four.

<details>
<summary>Answer</summary>

A handler parameter type that axum fills from the request before the handler runs: `Path`, `Query`, `Json`,
`State` (also `String`, `Bytes`, `HeaderMap` and others).

</details>

2. A client sends `GET /tasks/abc`. Which code runs, and what is the answer?

<details>
<summary>Answer</summary>

None of your code: the `Path<u64>` extractor fails and axum answers with a rejection, `400 Bad Request`,
``Invalid URL: Cannot parse `abc` to a `u64` ``. `get_task` never runs.

</details>

3. Why must `Json<T>` be the last parameter of a handler?

<details>
<summary>Answer</summary>

It reads (consumes) the request body, and the body can be read only once. Extractors that look only at the
path, query, headers or state can come before it.

</details>

4. Why is `std::sync::Mutex` acceptable in these handlers?

<details>
<summary>Answer</summary>

Each handler holds the lock for a very short time and never across an `.await`. A short blocking lock does not
stall the runtime, and the guard is never kept in a future that must be `Send`.

</details>

5. What does `.with_state(state)` change in the type of the router?

<details>
<summary>Answer</summary>

It turns a `Router<AppState>` (a router that still needs its state) into a `Router<()>`, written simply
`Router`, which can be served.

</details>

## Summary

- A `Router` maps a path and a method to a handler: `.route("/tasks", get(list_tasks).post(create_task))`;
  path parameters are written `{id}`.
- Extractors (`Path`, `Query`, `Json`, `State`) turn the request into typed values; if one fails, axum answers with
  a rejection (`400`, `415`, `422`) and the handler does not run.
- The body extractor must be the last parameter; `#[axum::debug_handler]` explains handler errors.
- Handlers return anything that implements `IntoResponse`: `Json`, `StatusCode`, tuples with headers, and
  `Result`, which makes `?` work.
- Shared state goes into `.with_state(...)`; `Arc<Mutex<…>>` with tiny critical sections is fine for in-memory data.
- `tower::ServiceExt::oneshot` tests the whole router in memory.

## Sources and further reading

- [axum — crate documentation](https://docs.rs/axum/0.8) — routing, handlers and the overview of all extractors.
- [axum::extract module](https://docs.rs/axum/0.8/axum/extract/index.html) — every extractor, the ordering rule and how rejections work.
- [axum::response module](https://docs.rs/axum/0.8/axum/response/index.html) — all the ways to build a response with `IntoResponse`.
- [axum examples](https://github.com/tokio-rs/axum/tree/main/examples) — official small programs, including a "todos" example and testing examples.
- [tower::ServiceExt](https://docs.rs/tower/0.5/tower/trait.ServiceExt.html) — `oneshot` and the other helpers used in tests.

---

[🏠 Start](../START-HERE.md) · [← 16.2 Designing HTTP APIs: REST](02-designing-http-apis.md) · [📋 Phase 16](README.md) · [16.4 Errors, validation and middleware →](04-errors-validation-and-middleware.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
