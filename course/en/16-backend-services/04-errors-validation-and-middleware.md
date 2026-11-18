<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.3 Web services with axum](03-web-services-with-axum.md) · [📋 Phase 16](README.md) · [16.5 SQL and PostgreSQL basics →](05-sql-and-postgresql-basics.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/04-errors-validation-and-middleware.md) <!-- nav -->

# Lesson 16.4 — Errors, validation and middleware

> **Phase 16 — Backend services** · Lesson 4 of 16 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to collect every failure of an API (Application Programming Interface) in one error type that turns itself into the JSON (JavaScript Object Notation) error body of the contract.
- How to validate input once, at the border, with a type that can only hold valid values.
- How to make axum's own rejections use your error format.
- What middleware is, how layers wrap a service like the skins of an onion, and how to use the ready-made layers
  of `tower-http` (request logs, timeouts, body size limits).
- How to write your own small middleware with `axum::middleware::from_fn`.

## Before you start

- You finished [Lesson 16.3 — Web services with axum](03-web-services-with-axum.md) and have its `code/p16-tasks`.
- You know `thiserror` ([Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)),
  "parse, don't validate" ([Lesson 10.4 — Input validation](../10-security-fundamentals/04-input-validation.md))
  and `tracing` ([Lesson 09.10 — Logging and tracing](../09-testing-debugging-benchmarking/10-logging-and-tracing.md)).

## New words

| Word | Meaning |
|---|---|
| **error type** | One enum that lists every way an operation can fail. Here: `ApiError`. |
| **validation** | Checking that input follows the rules (not empty, not too long) before using it. |
| **newtype** | A struct with one field that wraps another type to give it a meaning and rules, like `Title(String)`. |
| **middleware** | Code that runs around **every** request (or a group of them), before and after the handler: logging, timeouts, limits, authentication. |
| **layer** | In tower, a value that wraps a service in a middleware. `.layer(x)` on a `Router` adds one. |
| **timeout** | A time limit: if the work is not finished in time, it is stopped and an error is returned. |
| **request ID** | A unique number or text given to each request, so that its log lines and its response can be matched later. |

## 1. One error type for the whole API

In Lesson 16.3 errors were bare status codes, and axum's rejections were plain text. The contract of Lesson
16.2 wants every error to look like `{"code": "...", "message": "..."}`. The clean way to get there: one
enum for all failures, and one place that turns it into a response.

Add `thiserror` and the `macros` feature of axum (needed in Section 3), then the other crates of this lesson:

```bash
cd ~/projects/rust/code/p16-tasks
cargo add axum@0.8 --features macros
cargo add thiserror@2
cargo add tower-http@0.7 --features trace,timeout,limit
cargo add tracing@0.1
cargo add tracing-subscriber@0.3 --features env-filter
```

Create `src/error.rs`:

```rust
use axum::{
    Json,
    extract::{FromRequest, rejection::JsonRejection},
    http::StatusCode,
    response::{IntoResponse, Response},
};
use serde::Serialize;

/// Every way a request to the tasks API can fail.
#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("task {0} not found")]
    NotFound(u64),
    #[error("{0}")]
    Validation(String),
    #[error("{message}")]
    Rejected { status: StatusCode, message: String },
}

impl ApiError {
    fn status(&self) -> StatusCode {
        match self {
            ApiError::NotFound(_) => StatusCode::NOT_FOUND,
            ApiError::Validation(_) => StatusCode::UNPROCESSABLE_ENTITY,
            ApiError::Rejected { status, .. } => *status,
        }
    }

    fn code(&self) -> &'static str {
        match self {
            ApiError::NotFound(_) => "not_found",
            ApiError::Validation(_) => "validation_failed",
            ApiError::Rejected { status, .. } => match *status {
                StatusCode::UNPROCESSABLE_ENTITY => "validation_failed",
                StatusCode::UNSUPPORTED_MEDIA_TYPE => "unsupported_media_type",
                _ => "bad_request",
            },
        }
    }
}

#[derive(Serialize)]
struct ErrorBody {
    code: &'static str,
    message: String,
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let body = ErrorBody {
            code: self.code(),
            message: self.to_string(),
        };
        (self.status(), Json(body)).into_response()
    }
}

impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        ApiError::Rejected {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}

/// Like `axum::Json`, but a bad body becomes an `ApiError` (our JSON error format).
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct AppJson<T>(pub T);
```

**What happens:**

1. `ApiError` has three variants: a missing task, a broken rule (`Validation`), and a request that axum
   itself could not read (`Rejected`, which keeps axum's status code and message).
2. `#[error("...")]` from `thiserror` writes the `Display` text of each variant, so `self.to_string()` is the
   human message. `{0}` is the first field, `{message}` a named field.
3. `status()` and `code()` map each variant to the HTTP (HyperText Transfer Protocol) status and the stable code of the contract.
4. `impl IntoResponse for ApiError` is the important part: from now on, a handler can return
   `Result<_, ApiError>`, and every `Err` becomes the JSON error body with the right status. The mapping lives in
   **one** place.
5. The last part (`From<JsonRejection>` and `AppJson`) is Section 3.

## 2. Validation at the border: a `Title` type

The contract says: a title is trimmed and has 1 to 200 characters. Where should this check live? Not
"somewhere in each handler": you would forget it in one of them. Lesson 10.4 taught the better way, *parse,
don't validate*: make a type whose only constructor checks the rules. If you hold a `Title`, it is valid.

Create `src/title.rs`:

```rust
use crate::error::ApiError;

pub const MAX_TITLE_CHARS: usize = 200;

/// A task title that follows the rules of the contract. The only way to get one is `parse`.
#[derive(Debug, Clone, PartialEq)]
pub struct Title(String);

impl Title {
    pub fn parse(text: &str) -> Result<Title, ApiError> {
        let text = text.trim();
        if text.is_empty() {
            return Err(ApiError::Validation("title must not be empty".to_string()));
        }
        let chars = text.chars().count();
        if chars > MAX_TITLE_CHARS {
            return Err(ApiError::Validation(format!(
                "title must have at most {MAX_TITLE_CHARS} characters, got {chars}"
            )));
        }
        if text.chars().any(char::is_control) {
            return Err(ApiError::Validation(
                "title must not contain control characters".to_string(),
            ));
        }
        Ok(Title(text.to_string()))
    }

    pub fn into_string(self) -> String {
        self.0
    }
}
```

**What happens:**

1. The field of `Title` is private, so code outside this module cannot write `Title(String::new())`. The only
   door is `parse`.
2. `trim()` removes spaces at both ends, so `"   "` becomes empty and is refused.
3. `chars().count()` counts **characters**, not bytes: `"я"` is one character but two bytes in UTF-8 (Unicode Transformation Format, 8-bit)
   (Lesson 12.5). A limit in bytes would surprise users of other alphabets.
4. Control characters (line breaks, tabs, the terminal bell `\u0007`) are refused: a title is one line of text,
   and such characters can break logs and screens.
5. `into_string` gives the inner `String` back when we store the task.

> 🔒 **Security:** every limit is also a defence. Without a maximum length, one client could store a
> 100-megabyte "title". Validate size, characters and format of **every** field that comes from outside.

## 3. Our own rejections: `AppJson`

`Title::parse` runs only after the JSON was read. When the JSON itself is broken, the `Json` extractor rejects
the request with plain text before our code runs. To get our format there too, we wrap `Json` in our own
extractor `AppJson`, the last three lines of `error.rs`.

**What happens:**

1. `#[derive(FromRequest)]` (from the `macros` feature) makes `AppJson<T>` an extractor.
2. `via(axum::Json)` says: "do the work with the normal `Json` extractor".
3. `rejection(ApiError)` says: "if it fails, convert its rejection into an `ApiError`". The conversion is the
   `impl From<JsonRejection> for ApiError` above, which keeps axum's status (`400`, `415`, `422`) and its
   message.

Now handlers write `AppJson(new_task): AppJson<NewTask>` instead of `Json(...)`. For responses we keep using
`axum::Json`.

## 4. The handlers with errors and validation

Change the top of `src/main.rs` to declare the two modules and import what this lesson needs:

```rust
mod error;
mod title;

use std::collections::BTreeMap;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::{Arc, Mutex};
use std::time::Duration;

use axum::{
    Json, Router,
    extract::{Path, Query, Request, State},
    http::{HeaderValue, StatusCode, header},
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::get,
};
use serde::{Deserialize, Serialize};
use tower_http::{limit::RequestBodyLimitLayer, timeout::TimeoutLayer, trace::TraceLayer};

use error::{ApiError, AppJson};
use title::Title;
```

The structs (`Task`, `NewTask`, `TaskChanges`, `ListFilter`, `Store`), `AppState`, `list_tasks` and
`health` stay as in Lesson 16.3. Replace the other four handlers:

```rust
async fn create_task(
    State(state): State<AppState>,
    AppJson(new_task): AppJson<NewTask>,
) -> Result<impl IntoResponse, ApiError> {
    let title = Title::parse(&new_task.title)?;
    let mut store = state.lock().unwrap();
    store.last_id += 1;
    let task = Task {
        id: store.last_id,
        title: title.into_string(),
        done: false,
    };
    store.tasks.insert(task.id, task.clone());
    let location = format!("/tasks/{}", task.id);
    Ok((
        StatusCode::CREATED,
        [(header::LOCATION, location)],
        Json(task),
    ))
}

async fn get_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<Json<Task>, ApiError> {
    let store = state.lock().unwrap();
    match store.tasks.get(&id) {
        Some(task) => Ok(Json(task.clone())),
        None => Err(ApiError::NotFound(id)),
    }
}

async fn update_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    AppJson(changes): AppJson<TaskChanges>,
) -> Result<Json<Task>, ApiError> {
    // Check the input before taking the lock.
    let title = changes.title.as_deref().map(Title::parse).transpose()?;
    let mut store = state.lock().unwrap();
    let task = store.tasks.get_mut(&id).ok_or(ApiError::NotFound(id))?;
    if let Some(title) = title {
        task.title = title.into_string();
    }
    if let Some(done) = changes.done {
        task.done = done;
    }
    Ok(Json(task.clone()))
}

async fn delete_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<StatusCode, ApiError> {
    let mut store = state.lock().unwrap();
    match store.tasks.remove(&id) {
        Some(_) => Ok(StatusCode::NO_CONTENT),
        None => Err(ApiError::NotFound(id)),
    }
}
```

**What happens:**

1. Every handler that can fail returns `Result<_, ApiError>`. The `?` operator turns any `ApiError` into the
   JSON error response.
2. `create_task` parses the title **first**. If it is invalid, the function returns before it touches the store:
   no task, no used-up identifier.
3. In `update_task`, the title is optional. `changes.title.as_deref()` turns `Option<String>` into
   `Option<&str>`; `.map(Title::parse)` gives `Option<Result<Title, ApiError>>`; `.transpose()` swaps it into
   `Result<Option<Title>, ApiError>`, so `?` can return the error. A missing title stays `None`.
4. `create_task` returns `Result<impl IntoResponse, ApiError>`: the success type is still the tuple from Lesson 16.3.

> 💡 **Tip:** check all input before you change any state. Then a request either fully succeeds or changes
> nothing, which is much easier to reason about. Lesson 16.7 gives the same guarantee for the database with
> transactions.

## 5. Middleware: code around every request

Some jobs belong to **every** request, not to one handler: write a log line, stop requests that take too long,
refuse huge bodies, check a login. Copying this into every handler would be repetitive and easy to forget.
**Middleware** does it once, around the whole router.

In tower, a middleware is added as a **layer**. Each layer wraps everything inside it, like the skins of an onion.
A request travels inwards through every layer to the handler; the response travels back outwards:

```text
 request ─▶ ┌─ request_id ───────────────────────────────────────┐
            │  ┌─ TraceLayer (logs) ──────────────────────────┐  │
            │  │  ┌─ TimeoutLayer (10 s) ──────────────────┐  │  │
            │  │  │  ┌─ RequestBodyLimitLayer (16 KiB) ──┐ │  │  │
            │  │  │  │   router ─▶ handler               │ │  │  │
            │  │  │  └───────────────────────────────────┘ │  │  │
            │  │  └────────────────────────────────────────┘  │  │
            │  └──────────────────────────────────────────────┘  │
 response ◀─└────────────────────────────────────────────────────┘
```

Three ready-made layers come from `tower-http` (one Cargo feature each):

- **`TraceLayer`** creates a `tracing` span for each request and logs when it starts and when the response is
  ready, with the status and the latency.
- **`TimeoutLayer`** gives each request a time limit. If the handler is not done after 10 seconds, the layer
  drops it (Lesson 14.14: dropping a future cancels it) and answers with the status you choose. We choose
  `503 Service Unavailable`: "the server could not answer in time".
- **`RequestBodyLimitLayer`** refuses bodies larger than the limit with `413 Payload Too Large`, so nobody can
  make the service read a gigabyte into memory.

The fourth one is our own. `axum::middleware::from_fn` turns an ordinary async function into a middleware.
It gets the request and `next` (everything inside it); it can do work before calling `next.run(request)`
and after it. Add it to `src/main.rs` together with the new `app` and `main`:

```rust
static NEXT_REQUEST_ID: AtomicU64 = AtomicU64::new(1);

/// Our own middleware: gives every response an `x-request-id` header.
async fn request_id(request: Request, next: Next) -> Response {
    let id = NEXT_REQUEST_ID.fetch_add(1, Ordering::Relaxed);
    let mut response = next.run(request).await;
    response
        .headers_mut()
        .insert("x-request-id", HeaderValue::from(id));
    response
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
        // The last .layer() is the outermost: requests pass through it first.
        .layer(RequestBodyLimitLayer::new(16 * 1024))
        .layer(TimeoutLayer::with_status_code(
            StatusCode::SERVICE_UNAVAILABLE,
            Duration::from_secs(10),
        ))
        .layer(TraceLayer::new_for_http())
        .layer(middleware::from_fn(request_id))
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "p16_tasks=debug,tower_http=debug".into()),
        )
        .init();

    let state = AppState::default();
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::info!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app(state)).await.unwrap();
}
```

**What happens:**

1. `request_id` takes the next number, lets the rest of the service produce the response (`next.run(request).await`),
   then adds the header. Code before `next.run` sees the request; code after it sees the response.
2. In `app`, each `.layer(...)` wraps everything written **before** it. So the last one, `request_id`, is the
   outermost skin of the onion, and the body limit is the innermost. Read the chain from the bottom up to follow a
   request.
3. `main` sets up `tracing` as in Lesson 09.10. Without a `RUST_LOG` variable it shows `debug` events of our crate
   and of `tower_http` (where `TraceLayer` writes). `tracing::info!` replaces the `println!`.

> ⚡ **Performance:** middleware runs for every request, so keep it cheap. An atomic counter and one header cost
> almost nothing; a database query in a middleware would cost a query per request.

> 📏 **Convention:** `x-request-id` is a common name for this header; tower-http even has ready-made layers for
> it (`SetRequestIdLayer`, `PropagateRequestIdLayer`).

## 6. Run it

`cargo run`, then, in the second terminal, create a task with spaces around the title:

```bash
curl -i -H 'content-type: application/json' -d '{"title":"  Buy milk  "}' http://127.0.0.1:3000/tasks
```

```text
HTTP/1.1 201 Created
content-type: application/json
location: /tasks/1
x-request-id: 1
content-length: 40
date: Wed, 18 Nov 2026 07:16:05 GMT

{"id":1,"title":"Buy milk","done":false}
```

The title was trimmed, and the response carries `x-request-id`. The service terminal shows the log of
`TraceLayer` (your terminal shows it in colour):

```text
2026-11-18T07:16:05.006327Z  INFO p16_tasks: listening on http://127.0.0.1:3000
2026-11-18T07:16:05.174757Z DEBUG request{method=POST uri=/tasks version=HTTP/1.1}: tower_http::trace::on_request: started processing request
2026-11-18T07:16:05.174895Z DEBUG request{method=POST uri=/tasks version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=201
2026-11-18T07:16:05.174936Z DEBUG request{method=POST uri=/tasks version=HTTP/1.1}: tower_http::trace::on_eos: end of stream stream_duration=0 ms
```

Now the bad requests from Lesson 16.3, Section 7, and a few new ones:

```bash
curl -s -H 'content-type: application/json' -d '{"title":"   "}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"title":"a\u0007b"}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"title":' http://127.0.0.1:3000/tasks; echo
curl -s -d '{"title":"x"}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"name":"x"}' http://127.0.0.1:3000/tasks; echo
curl -s http://127.0.0.1:3000/tasks/7; echo
curl -s -X PATCH -H 'content-type: application/json' -d '{"title":""}' http://127.0.0.1:3000/tasks/1; echo
```

```text
{"code":"validation_failed","message":"title must not be empty"}
{"code":"validation_failed","message":"title must not contain control characters"}
{"code":"bad_request","message":"Failed to parse the request body as JSON: title: EOF while parsing a value at line 1 column 9"}
{"code":"unsupported_media_type","message":"Expected request with `Content-Type: application/json`"}
{"code":"validation_failed","message":"Failed to deserialize the JSON body into the target type: missing field `title` at line 1 column 12"}
{"code":"not_found","message":"task 7 not found"}
{"code":"validation_failed","message":"title must not be empty"}
```

Every error now has the contract's shape. (`\u0007` inside JSON is the "bell" control character.) Finally,
send a body of about 20 000 bytes, larger than the 16 KiB (kibibyte, 1024 bytes) limit:

```bash
python3 -c "print('{\"title\":\"' + 'x'*20000 + '\"}')" > big.json
curl -i -H 'content-type: application/json' --data @big.json http://127.0.0.1:3000/tasks
```

```text
HTTP/1.1 413 Payload Too Large
content-type: text/plain; charset=utf-8
x-request-id: 10
content-length: 21
date: Wed, 18 Nov 2026 07:16:05 GMT

length limit exceeded
```

**What happens:**

1. `python3` (installed on every Mac) writes a file with a 20 000-character title. `--data @big.json` sends the
   file as the body.
2. `RequestBodyLimitLayer` sees `content-length` larger than 16 KiB and answers `413` **before** the request
   reaches the router; the body is never read. Because this answer comes from a layer and not from our code, it is
   plain text. That is acceptable for this rare case; `x-request-id` is still there, because `request_id` is the
   outermost layer.

Stop the service and delete `big.json`.

## 7. Tests for errors

The tests of Lesson 16.3 still pass. Make `unknown_task_is_not_found` also check the body, and add a test for the
validation error:

```rust
    #[tokio::test]
    async fn unknown_task_is_not_found() {
        let app = app(AppState::default());
        let request = Request::get("/tasks/42").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
        assert_eq!(body, r#"{"code":"not_found","message":"task 42 not found"}"#);
    }

    #[tokio::test]
    async fn empty_title_is_rejected() {
        let app = app(AppState::default());
        let (status, body) = send(app, post_json("/tasks", r#"{"title":"   "}"#)).await;
        assert_eq!(status, StatusCode::UNPROCESSABLE_ENTITY);
        assert_eq!(
            body,
            r#"{"code":"validation_failed","message":"title must not be empty"}"#
        );
    }
```

```text
running 3 tests
test tests::unknown_task_is_not_found ... ok
test tests::empty_title_is_rejected ... ok
test tests::created_task_can_be_read_back ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The layers are part of `app()`, so `oneshot` tests run through them too.

The test counts in this phase include only the tests of the lesson text, unless a lesson says otherwise. If you
kept the tests from earlier exercises (here: the three of Lesson 16.3), your numbers are higher; that is fine.

> 🧪 **Testing:** test the **unhappy paths** as carefully as the happy ones. Clients depend on the exact error
> codes; an error test protects them like the contract test of Lesson 16.2.

## Common mistakes

**1. `AppJson` without the `From` conversion.** If `impl From<JsonRejection> for ApiError` is missing, `AppJson`
is not an extractor at all. With `#[axum::debug_handler]` on the handler, the compiler says so:

```text
error[E0277]: the trait bound `AppJson<NewTask>: FromRequest<Arc<std::sync::Mutex<Store>>, _>` is not satisfied
  --> src/main.rs:71:24
   |
71 |     AppJson(new_task): AppJson<NewTask>,
   |                        ^^^^^^^ unsatisfied trait bound
   = note: Function argument is not a valid axum extractor.
```

Add the `From` implementation: the `rejection(ApiError)` attribute needs it.

**2. Changing state before validating.** If `create_task` increased `last_id` and only then parsed the title, a
rejected request would still use up an identifier. Validate first, then lock and change.

**3. Putting internal details into error messages.** When Lesson 16.6 adds a database, it is tempting to write
`ApiError::Validation(db_error.to_string())`. That sends SQL (Structured Query Language) details to strangers. Unexpected failures get their own
variant that answers only `{"code":"internal","message":"internal error"}` and writes the details to the log.

**4. Layers in the wrong order.** If `TraceLayer` were inside `TimeoutLayer`, a request stopped by the timeout would
leave no "finished" log line. Outer layers see everything that inner layers do; put logging and request IDs outside.

## Try it yourself

1. Run the service with `RUST_LOG=tower_http=info cargo run`. The request logs disappear: `TraceLayer` writes at
   `debug` level.
2. Send a title of exactly 200 characters (`'x'*200` in the Python command) and one of 201. The first is created,
   the second gets `title must have at most 200 characters, got 201`.
3. Commit: `git add code/p16-tasks && git commit -m "p16-tasks: JSON errors, validation, middleware"`.

## Exercises

### Exercise 1 — Unit tests for `Title` (easy)

Add a `#[cfg(test)] mod tests` at the end of `src/title.rs` that checks: spaces are trimmed; `""` and `" \t "`
are refused; 200 Cyrillic letters `я` (400 bytes) are accepted but 201 `x` are refused; a title with `\n` is refused.

<details>
<summary>Hint</summary>

`"я".repeat(MAX_TITLE_CHARS)` builds the long string. `assert!(Title::parse(...).is_err())` is enough for the
refusals.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn trims_spaces() {
        assert_eq!(Title::parse("  Buy milk ").unwrap().into_string(), "Buy milk");
    }

    #[test]
    fn rejects_empty_and_blank() {
        assert!(Title::parse("").is_err());
        assert!(Title::parse(" \t ").is_err());
    }

    #[test]
    fn length_limit_counts_characters_not_bytes() {
        let ok = "я".repeat(MAX_TITLE_CHARS); // 200 characters, 400 bytes
        assert!(Title::parse(&ok).is_ok());
        let too_long = "x".repeat(MAX_TITLE_CHARS + 1);
        assert!(Title::parse(&too_long).is_err());
    }

    #[test]
    fn rejects_control_characters() {
        assert!(Title::parse("a\nb").is_err());
    }
}
```

```text
test title::tests::rejects_empty_and_blank ... ok
test title::tests::rejects_control_characters ... ok
test title::tests::trims_spaces ... ok
test title::tests::length_limit_counts_characters_not_bytes ... ok
```

Why it works: `Title` is a small pure type, so unit tests are fast and need no router. The Cyrillic test proves
that the limit counts characters: a version with `text.len()` (bytes) would refuse it.

</details>

### Exercise 2 — A `server` header on every response (medium)

Write a second middleware `server_header` that adds `server: p16-tasks` to every response, and add it to `app` as
the outermost layer. Write a test that sends `GET /no-such-path` with `oneshot` and checks that even this `404`
response has the header.

<details>
<summary>Hint</summary>

`header::SERVER` is the header name and `HeaderValue::from_static("p16-tasks")` the value. In the test, keep the
whole response: `let response = app.oneshot(request).await.unwrap();` and read `response.headers()["server"]`.

</details>

<details>
<summary>Answer</summary>

```rust
async fn server_header(request: Request, next: Next) -> Response {
    let mut response = next.run(request).await;
    response
        .headers_mut()
        .insert(header::SERVER, HeaderValue::from_static("p16-tasks"));
    response
}

// in app(), after the request_id layer:
//         .layer(middleware::from_fn(server_header))

    #[tokio::test]
    async fn every_response_names_the_server() {
        let app = app(AppState::default());
        let request = Request::get("/no-such-path").body(Body::empty()).unwrap();
        let response = app.oneshot(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::NOT_FOUND);
        assert_eq!(response.headers()["server"], "p16-tasks");
    }
```

Why it works: layers wrap the **whole** router, including the fallback that answers `404` for unknown paths, so
the header appears on every response. A handler-level solution would miss exactly these cases.

</details>

### Exercise 3 — JSON errors for bad paths (hard)

`GET /tasks/abc` still answers with plain text, because `Path` has its own rejection. Create `AppPath<T>` in
`error.rs` the same way as `AppJson`, use it in the three handlers with `{id}`, and write a test that
`GET /tasks/abc` returns `400` with
``{"code":"bad_request","message":"Invalid URL: Cannot parse `abc` to a `u64`"}``.

<details>
<summary>Hint</summary>

`Path` reads only the request *parts* (not the body), so the derive is `FromRequestParts`, and the rejection type
is `axum::extract::rejection::PathRejection`. It also has `status()` and `body_text()`.

</details>

<details>
<summary>Answer</summary>

```rust
// in src/error.rs: the import becomes
//     extract::{
//         FromRequest, FromRequestParts,
//         rejection::{JsonRejection, PathRejection},
//     },

impl From<PathRejection> for ApiError {
    fn from(rejection: PathRejection) -> Self {
        ApiError::Rejected {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}

/// Like `axum::extract::Path`, but a bad path becomes an `ApiError`.
#[derive(FromRequestParts)]
#[from_request(via(axum::extract::Path), rejection(ApiError))]
pub struct AppPath<T>(pub T);

// in src/main.rs: `use error::{ApiError, AppJson, AppPath};`, remove `Path` from the
// `extract::{...}` import, and write `AppPath(id): AppPath<u64>` in get_task,
// update_task and delete_task.

    #[tokio::test]
    async fn bad_id_gives_json_error() {
        let app = app(AppState::default());
        let request = Request::get("/tasks/abc").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::BAD_REQUEST);
        assert_eq!(
            body,
            r#"{"code":"bad_request","message":"Invalid URL: Cannot parse `abc` to a `u64`"}"#
        );
    }
```

With all exercises done, `cargo test` runs nine tests:

```text
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: extractors that do not read the body implement `FromRequestParts`; the derive works the same way.
Now every error that our code or axum's extractors produce has one shape. (Do the same for `Query` if you like:
`QueryRejection`.)

</details>

## Quiz

1. Why does `ApiError` implement `IntoResponse`?

<details>
<summary>Answer</summary>

So handlers can return `Result<_, ApiError>` and use `?`. The conversion from error to status code and JSON body
then lives in one place, and all errors have the same shape.

</details>

2. What does "parse, don't validate" mean for `Title`?

<details>
<summary>Answer</summary>

Instead of checking a `String` in many places, `Title::parse` checks the rules once and returns a `Title`. Because
the field is private, every `Title` that exists has passed the checks.

</details>

3. What do `via(axum::Json)` and `rejection(ApiError)` mean on `AppJson`?

<details>
<summary>Answer</summary>

Extract with the normal `Json` extractor, and if it fails, convert its rejection into `ApiError` (through
`From<JsonRejection>`), so the client gets our JSON error format.

</details>

4. In `.layer(A).layer(B).layer(C)`, which layer sees the request first?

<details>
<summary>Answer</summary>

`C`, the last one: it is the outermost. The request goes C → B → A → handler, and the response goes back
A → B → C.

</details>

5. What happens when a handler runs longer than the `TimeoutLayer` allows?

<details>
<summary>Answer</summary>

The layer stops waiting, drops the handler's future (which cancels it) and returns a response with the configured
status, `503` here.

</details>

## Summary

- One error enum (`ApiError`) with `impl IntoResponse` gives every failure the same status mapping and JSON body;
  handlers return `Result<_, ApiError>` and use `?`.
- Validate at the border with a newtype whose only constructor checks the rules (`Title::parse`); count
  characters, not bytes; limit everything.
- `#[derive(FromRequest)]` with `via(...)` and `rejection(...)` makes axum's own rejections use your format
  (`FromRequestParts` for `Path` and `Query`).
- Middleware runs around every request. Layers wrap like an onion; the last `.layer()` is the outermost.
- `tower-http` provides `TraceLayer`, `TimeoutLayer` and `RequestBodyLimitLayer`; `middleware::from_fn` turns an
  async function into your own middleware.
- Never send internal details in an error response; log them instead.

## Sources and further reading

- [axum::error_handling](https://docs.rs/axum/0.8/axum/error_handling/index.html) — how axum thinks about errors and why handlers cannot fail without a response.
- [axum::middleware](https://docs.rs/axum/0.8/axum/middleware/index.html) — `from_fn`, the order of layers and the onion picture.
- [axum `FromRequest` derive](https://docs.rs/axum/0.8/axum/extract/derive.FromRequest.html) — `via` and `rejection` in detail.
- [tower-http](https://docs.rs/tower-http/0.7/tower_http/) — all ready-made HTTP middleware: trace, timeout, limit, CORS (Cross-Origin Resource Sharing), compression, request IDs.
- [thiserror](https://docs.rs/thiserror/2) — the `#[error(...)]` syntax.
- [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — the classic article behind the `Title` idea (examples in Haskell, the idea is general).

---

[🏠 Start](../START-HERE.md) · [← 16.3 Web services with axum](03-web-services-with-axum.md) · [📋 Phase 16](README.md) · [16.5 SQL and PostgreSQL basics →](05-sql-and-postgresql-basics.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
