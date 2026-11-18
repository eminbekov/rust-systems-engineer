<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.9 The production-readiness checklist](09-production-readiness-checklist.md) · [📋 Phase 20](README.md) · [20.11 Checkpoint →](11-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/10-project-harden-a-service.md) <!-- nav -->

# Lesson 20.10 — Project: harden a service

> **Phase 20 — Reliability and hardening** · Lesson 10 of 11 · ⏱ about 10 hours (5 sittings)

## What you will learn

- How to review a working service against the production-readiness checklist and find its weak points.
- How to turn panics into answers, and add a deadline, size limits and load shedding that are covered by tests.
- How to use Miri and fuzzing on the parts of a service where they apply, and turn every finding into a regression test.
- How to add supply-chain checks and a chaos test with toxiproxy, and prove with numbers that the service now fails fast.

This is a **project lesson**. It gives you a small service that works but is not ready for production, the target (the contract), a plan with milestones, hints, and commands that prove each milestone is done. It does not give you the hardened code.

## Before you start

- You finished [Lesson 20.9 — The production-readiness checklist](09-production-readiness-checklist.md) and the earlier lessons of this phase. You will use panics and Clippy lints ([20.2](02-errors-and-panics-in-production.md)), fuzzing ([20.3](03-fuzzing-in-depth.md)), Miri ([20.4](04-miri-sanitizers-and-loom-in-ci.md)), toxiproxy ([20.5](05-fault-injection-and-chaos-testing.md)), supply-chain tools ([20.7](07-supply-chain-security-in-depth.md)), limits ([20.8](08-overload-limits-and-backpressure.md)) and the checklist with `readiness-check.sh` ([20.9](09-production-readiness-checklist.md)).
- You know axum and Redis from Phase 16 ([Lesson 16.3](../16-backend-services/03-web-services-with-axum.md), [Lesson 16.10](../16-backend-services/10-caching-with-redis.md)). If you built the URL (Uniform Resource Locator) shortener of [Lesson 16.15](../16-backend-services/15-project-url-shortener-service.md), you may harden it instead of the starter below; the milestones work the same way.
- Tools: the nightly toolchain with Miri, `cargo-fuzz`, `cargo-audit`, `cargo-deny`, `cargo-vet`, `cargo-auditable`, `cargo-cyclonedx`, Docker, `curl` and `jq`.
- Git: the project goes to `~/projects/rust/code/p20-shortener`. Work on a branch (`git switch -c project/20-hardening`) and commit after every milestone.

## New words

| Word | Meaning |
|---|---|
| **starter code** | Code that the lesson gives you as a starting point. Here it works, but it has weak points on purpose. |
| **weak point** | A place where the service can crash, hang, be abused or fail without anybody noticing. |
| **toxic** | In toxiproxy, a fault that is added to a connection: extra latency, a cut connection, a slow trickle of bytes. |
| **deadline** | The latest moment by which a whole request must be answered, here 300 ms after it arrived. |
| **regression test** | A test that repeats an input that once found a bug, so the bug cannot come back unnoticed. |

## The project

### What you will build

You get **p20-shortener**: a URL shortener with three endpoints. `POST /links` stores a URL in Redis and returns a short code; `GET /{code}` redirects to the URL; `GET /health` says `ok`. It passes its tests. But under bad input, a slow Redis or a stopped Redis it panics, and nothing protects it from overload.

You will harden it step by step, measure every change, and finish with a filled-in `READINESS.md` in which every item has evidence. The service reaches Redis through **toxiproxy**, so you can make Redis slow or "down" whenever you want:

```text
 curl / oha ──▶ p20-shortener ──▶ toxiproxy :26379 ──────▶ course-p20-redis :6379
 (your Mac)     127.0.0.1:3000    (Docker, API on :18474)    (Docker)
```

### Starter code

Create the package and add the crates:

```bash
cd ~/projects/rust
cargo new --lib code/p20-shortener
cd code/p20-shortener
cargo add axum@0.8
cargo add tokio@1 --features full
cargo add serde@1 --features derive
cargo add redis@1 --features tokio-comp,connection-manager
cargo add --dev tower@0.5 --features util
mkdir tests
```

`src/code.rs` turns numbers into short codes and back:

```rust
//! Short codes: a number written in base 62 (digits, then a-z, then A-Z).

const ALPHABET: &[u8; 62] = b"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

/// The value of every ASCII character, or 255 for characters outside the alphabet.
const VALUES: [u8; 128] = {
    let mut table = [255u8; 128];
    let mut i = 0;
    while i < 62 {
        table[ALPHABET[i] as usize] = i as u8;
        i += 1;
    }
    table
};

/// Writes `n` in base 62: 0 → "0", 61 → "Z", 62 → "10".
pub fn encode(mut n: u64) -> String {
    if n == 0 {
        return "0".to_string();
    }
    let mut digits = Vec::new();
    while n > 0 {
        digits.push(ALPHABET[(n % 62) as usize]);
        n /= 62;
    }
    digits.reverse();
    String::from_utf8(digits).expect("the alphabet is ASCII")
}

/// Reads a base-62 code back into a number. `None` if it is not a valid code.
pub fn decode(code: &str) -> Option<u64> {
    if code.is_empty() {
        return None;
    }
    let mut n: u64 = 0;
    for byte in code.bytes() {
        // SAFETY: a byte is at most 255, and the table covers every byte. (Fast: no bounds check.)
        let value = unsafe { *VALUES.as_ptr().add(usize::from(byte)) };
        if value == 255 {
            return None;
        }
        n = n * 62 + u64::from(value);
    }
    Some(n)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_values_and_round_trip() {
        assert_eq!(encode(0), "0");
        assert_eq!(encode(62), "10");
        assert_eq!(decode("10"), Some(62));
        assert_eq!(decode("ab-c"), None);
        for n in [1, 7, 1_000, 123_456_789] {
            assert_eq!(decode(&encode(n)), Some(n));
        }
    }
}
```

`src/store.rs` hides the storage behind a trait, so tests can use a store in memory:

```rust
//! Where links are kept: Redis in production, memory in tests.

use std::collections::HashMap;
use std::sync::{Arc, Mutex};

use redis::AsyncCommands;
use redis::aio::ConnectionManager;

#[derive(Debug)]
pub struct StoreError(pub String);

impl From<redis::RedisError> for StoreError {
    fn from(err: redis::RedisError) -> Self {
        StoreError(err.to_string())
    }
}

/// Everything the handlers need from a storage.
pub trait Store: Clone + Send + Sync + 'static {
    /// Saves a URL and returns its new number.
    fn save(&self, url: &str) -> impl Future<Output = Result<u64, StoreError>> + Send;
    /// Finds the URL with this number.
    fn load(&self, id: u64) -> impl Future<Output = Result<Option<String>, StoreError>> + Send;
}

#[derive(Clone)]
pub struct RedisStore {
    conn: ConnectionManager,
}

impl RedisStore {
    pub async fn connect(url: &str) -> Result<RedisStore, StoreError> {
        let client = redis::Client::open(url)?;
        let conn = client.get_connection_manager().await?;
        Ok(RedisStore { conn })
    }
}

impl Store for RedisStore {
    fn save(&self, url: &str) -> impl Future<Output = Result<u64, StoreError>> + Send {
        let mut conn = self.conn.clone();
        let url = url.to_string();
        async move {
            let id: u64 = conn.incr("links:next_id", 1).await?;
            let () = conn.set(format!("link:{id}"), url).await?;
            Ok(id)
        }
    }

    fn load(&self, id: u64) -> impl Future<Output = Result<Option<String>, StoreError>> + Send {
        let mut conn = self.conn.clone();
        async move { Ok(conn.get(format!("link:{id}")).await?) }
    }
}

/// A store in memory, for tests.
#[derive(Clone, Default)]
pub struct MemoryStore {
    links: Arc<Mutex<HashMap<u64, String>>>,
}

impl Store for MemoryStore {
    fn save(&self, url: &str) -> impl Future<Output = Result<u64, StoreError>> + Send {
        let mut links = self.links.lock().unwrap();
        let id = links.len() as u64 + 1;
        links.insert(id, url.to_string());
        async move { Ok(id) }
    }

    fn load(&self, id: u64) -> impl Future<Output = Result<Option<String>, StoreError>> + Send {
        let url = self.links.lock().unwrap().get(&id).cloned();
        async move { Ok(url) }
    }
}
```

`src/lib.rs` has the handlers:

```rust
//! A small URL shortener. It works, but it is not ready for production yet.

pub mod code;
pub mod store;

use axum::extract::{Path, State};
use axum::http::StatusCode;
use axum::response::{IntoResponse, Redirect, Response};
use axum::routing::{get, post};
use axum::{Json, Router};
use serde::{Deserialize, Serialize};

use crate::code::{decode, encode};
use crate::store::Store;

/// Accepts only absolute http and https addresses of at most 2,000 bytes.
pub fn validate_url(input: &str) -> Result<&str, &'static str> {
    let input = input.trim();
    if input.len() > 2000 {
        return Err("url is too long");
    }
    if &input[..7] == "http://" || &input[..8] == "https://" {
        Ok(input)
    } else {
        Err("url must start with http:// or https://")
    }
}

#[derive(Deserialize)]
struct NewLink {
    url: String,
}

#[derive(Serialize)]
struct Created {
    code: String,
}

async fn create<S: Store>(
    State(store): State<S>,
    Json(body): Json<NewLink>,
) -> Result<(StatusCode, Json<Created>), (StatusCode, &'static str)> {
    let url = validate_url(&body.url).map_err(|e| (StatusCode::BAD_REQUEST, e))?;
    let id = store.save(url).await.unwrap();
    Ok((StatusCode::CREATED, Json(Created { code: encode(id) })))
}

async fn follow<S: Store>(State(store): State<S>, Path(code): Path<String>) -> Response {
    let Some(id) = decode(&code) else {
        return (StatusCode::NOT_FOUND, "no such link\n").into_response();
    };
    match store.load(id).await.unwrap() {
        Some(url) => Redirect::temporary(&url).into_response(),
        None => (StatusCode::NOT_FOUND, "no such link\n").into_response(),
    }
}

async fn health() -> &'static str {
    "ok\n"
}

pub fn app<S: Store>(store: S) -> Router {
    Router::new()
        .route("/links", post(create::<S>))
        .route("/health", get(health))
        .route("/{code}", get(follow::<S>))
        .with_state(store)
}
```

`src/main.rs`:

```rust
use p20_shortener::app;
use p20_shortener::store::RedisStore;

#[tokio::main]
async fn main() {
    let redis_url = std::env::var("REDIS_URL").unwrap_or("redis://127.0.0.1:26379/".to_string());
    let store = RedisStore::connect(&redis_url).await.unwrap();
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app(store)).await.unwrap();
}
```

and `tests/api.rs`:

```rust
use axum::body::Body;
use axum::http::{Request, StatusCode, header};
use p20_shortener::app;
use p20_shortener::store::MemoryStore;
use tower::ServiceExt;

#[tokio::test]
async fn create_then_follow() {
    let app = app(MemoryStore::default());

    let create = Request::post("/links")
        .header(header::CONTENT_TYPE, "application/json")
        .body(Body::from(r#"{"url":"https://www.rust-lang.org/"}"#))
        .unwrap();
    let response = app.clone().oneshot(create).await.unwrap();
    assert_eq!(response.status(), StatusCode::CREATED);

    // The first link gets number 1, which is the code "1".
    let follow = Request::get("/1").body(Body::empty()).unwrap();
    let response = app.oneshot(follow).await.unwrap();
    assert_eq!(response.status(), StatusCode::TEMPORARY_REDIRECT);
    assert_eq!(
        response.headers()[header::LOCATION],
        "https://www.rust-lang.org/"
    );
}

#[tokio::test]
async fn unknown_code_is_404() {
    let response = app(MemoryStore::default())
        .oneshot(Request::get("/zz").body(Body::empty()).unwrap())
        .await
        .unwrap();
    assert_eq!(response.status(), StatusCode::NOT_FOUND);
}
```

`cargo test` passes: 1 unit test and 2 API (Application Programming Interface) tests that call the router directly.

Start Redis and toxiproxy (the image is published by the toxiproxy authors) in one Docker network, and create the proxy:

```bash
docker network create course-p20-net
docker run -d --name course-p20-redis --network course-p20-net redis:8
docker run -d --name course-p20-toxiproxy --network course-p20-net \
  -p 127.0.0.1:18474:8474 -p 127.0.0.1:26379:26379 ghcr.io/shopify/toxiproxy:2.13.0
curl -s -X POST http://127.0.0.1:18474/proxies \
  -d '{"name":"redis","listen":"0.0.0.0:26379","upstream":"course-p20-redis:6379"}'
```

```text
{"name":"redis","listen":"[::]:26379","upstream":"course-p20-redis:6379","enabled":true,"Logger":{},"toxics":[]}
```

The API port is `18474` instead of toxiproxy's usual `8474`, so it does not clash with the container from Lesson 20.5 if that one still runs. Only toxiproxy is published on your Mac; Redis is reachable only through it.

### Contract

When you are done, the service behaves like this:

| Situation | Answer |
|---|---|
| `POST /links` with a valid `http://` or `https://` URL | `201` and `{"code":"…"}` |
| `POST /links` with any other text in `url` (also `""`, `"https:/"`, text that is not ASCII, the American Standard Code for Information Interchange) | `400`, never a panic |
| A request body larger than 4 KiB (4,096 bytes) | `413` |
| `GET /{code}` for a stored code | `307` with `location: <url>` |
| `GET /{code}` for any other text | `404`, never a panic |
| Redis returns an error or the connection is broken | `503` with `retry-after: 1`, never a panic |
| A request is not finished after 300 ms | `503` at 300 ms |
| More than 64 requests at the same time | the extra ones get `503` at once |
| `GET /livez` | `200` while the process runs |
| `GET /readyz` | `200` only if the service is ready **and** Redis answers `PING` within 200 ms; otherwise `503` |
| SIGTERM (the stop signal) | `/readyz` becomes `503`, a drain of 2 s, open requests finish, exit code `0` |
| `REDIS_URL` not set | `config error: REDIS_URL is not set` on standard error, exit code `2` |

And these checks pass:

| Check | Command |
|---|---|
| Tests, formatting, Clippy, no `unwrap` outside tests, no advisories | `./readiness-check.sh` (Lesson 20.9) |
| Miri on the library tests | `cargo +nightly miri test --lib` |
| Two fuzz targets, 60 seconds each without a crash | `cargo +nightly fuzz run decode` and `… validate_url` |
| Licenses, bans, sources, advisories | `cargo deny check` |
| Every dependency vetted or exempted | `cargo vet` |
| The release binary carries its dependency list; an SBOM (Software Bill of Materials) exists | `cargo auditable build --release`, `cargo cyclonedx` |
| Redis slow, Redis down, Redis back | `./chaos.sh` |

### Step-by-step plan

Work through the milestones in order. Run the "Done when" commands, then commit. Try before you open a hint.

---

#### M1 — Run it, break it, write down what you found

**Goal:** the starter runs against Redis, and you have a list of its weak points.

**Guidance:**

1. `cargo build --release` and `./target/release/p20-shortener`. In a second terminal, create a link and follow it.
2. Send the URL `https:/` and watch both terminals.
3. Copy `readiness-check.sh` from Lesson 20.9 into the project and run it.
4. Create `READINESS.md` with the tables of Lesson 20.9. Read the starter code with the checklist in your hand and write every gap into the evidence column ("missing: …"). Also read every `unsafe` block and every slice index.
5. Commit the starter and `READINESS.md` (`chore: starter code and readiness baseline`).

<details>
<summary>Hint — what to look for</summary>

You should find at least eight weak points: `unwrap()` on every storage result; no deadline of its own (only the Redis client's internal default of 500 ms); no explicit body limit; no concurrency limit; `/health` instead of separate liveness and readiness; no graceful shutdown; a silent default for `REDIS_URL`; slicing in `validate_url`; the `SAFETY` comment in `decode` (is the table really 256 entries long?); `n * 62` in `decode`; no `deny.toml`, no `supply-chain/`.

</details>

**Done when:** creating and following a link works:

```bash
curl -s -X POST http://127.0.0.1:3000/links -H 'content-type: application/json' \
  -d '{"url":"https://www.rust-lang.org/"}'
curl -si http://127.0.0.1:3000/1 | head -2
```

```text
{"code":"1"}
HTTP/1.1 307 Temporary Redirect
location: https://www.rust-lang.org/
```

the short URL crashes the request (curl exit code 52 means "empty reply from server"):

```bash
curl -s -X POST http://127.0.0.1:3000/links -H 'content-type: application/json' -d '{"url":"https:/"}'
echo "curl exit code: $?"
```

```text
curl exit code: 52
```

while the service terminal shows the panic:

```text
thread 'tokio-rt-worker' (6318885) panicked at src/lib.rs:22:42:
end byte index 8 is out of bounds for string of length 7
```

and `./readiness-check.sh` shows exactly one `FAIL`: `no unwrap outside tests`. `READINESS.md` lists your weak points.

---

#### M2 — Answers instead of panics, a deadline and limits

**Goal:** no storage problem can crash a request or make it slow, and the limits of the contract are in place, each with a test.

**Guidance:**

1. Give `StoreError` an `IntoResponse` implementation that logs the error with `eprintln!` and answers `503` with `retry-after: 1`. Let the handlers return `Result<Response, StoreError>` and use `?` instead of `unwrap()`. `MemoryStore` may not use `unwrap()` either: `lock().unwrap_or_else(PoisonError::into_inner)` takes the data even after a panic.
2. Add the layer stack of Lesson 20.8 to the `/links` and `/{code}` routes: `HandleErrorLayer`, load shedding, a concurrency limit of 64 and a timeout of 300 ms. Put `.with_state(...)` at the very end of `app`, so the layers are built once.
3. Add `DefaultBodyLimit::max(4 * 1024)` (from `axum::extract`) to the same routes.
4. Write tests with two **fake stores** in `tests/api.rs`: `SlowStore` (every method sleeps 10 s) and `BrokenStore` (every method returns `Err`). Test: `GET /1` with `SlowStore` answers `503` in at most 300 ms of paused time; with `BrokenStore` it answers `503` with `retry-after: 1`; a body of 10,000 bytes answers `413`.
5. Commit (`feat: errors become 503, deadline, limits`).

<details>
<summary>Hint 1 — a fake store in a few lines</summary>

In an `impl` block, an `async fn` may implement a trait method that is declared as `fn … -> impl Future<Output = …> + Send`:

```rust
#[derive(Clone)]
struct BrokenStore;

impl Store for BrokenStore {
    async fn save(&self, _url: &str) -> Result<u64, StoreError> {
        Err(StoreError("broken pipe".to_string()))
    }
    async fn load(&self, _id: u64) -> Result<Option<String>, StoreError> {
        Err(StoreError("broken pipe".to_string()))
    }
}
```

</details>

<details>
<summary>Hint 2 — measuring paused time</summary>

Use `#[tokio::test(start_paused = true)]` and `let start = tokio::time::Instant::now();` before the request, then `assert!(start.elapsed() <= Duration::from_millis(300));`. Add `tokio` with the feature `test-util` as a dev-dependency.

</details>

**Done when:** `cargo test --test api` shows your new tests (the names are yours) and passes:

```text
test broken_store_gives_503_not_a_panic ... ok
test big_body_is_413 ... ok
test unknown_code_is_404 ... ok
test create_then_follow ... ok
test slow_store_gives_503_within_the_deadline ... ok
```

`./readiness-check.sh` may still show `FAIL` for `no unwrap outside tests`, but only because of `main.rs`, which you fix in M3.

---

#### M3 — Liveness, readiness, configuration and shutdown

**Goal:** the lifecycle rows of the contract.

**Guidance:**

1. Add `fn ping(&self)` to the `Store` trait (Redis: the `PING` command; memory: always `Ok`).
2. Replace `/health` by `/livez` and `/readyz` as in Lesson 20.9. Keep the ready flag and the store together in one `AppState<S>`. `/readyz` gives `ping()` at most 200 ms.
3. In `main`: no default for `REDIS_URL`, no `unwrap()`, and the shutdown of Lesson 20.9 (SIGTERM or Ctrl+C, not ready, drain 2 s, finish).
4. Extend the slow-store test: `/readyz` answers `503`.
5. Commit (`feat: liveness, readiness, drain on SIGTERM`).

<details>
<summary>Hint — PING with the redis crate</summary>

`let _: String = redis::cmd("PING").query_async(&mut conn).await?;` sends any command and converts the answer (`PONG`) into the type you ask for.

</details>

**Done when:**

```bash
env -u REDIS_URL cargo run -q
echo "exit code: $?"
```

```text
config error: REDIS_URL is not set
exit code: 2
```

and, with the service running (`REDIS_URL=redis://127.0.0.1:26379/ ./target/release/p20-shortener`), about 0.3 s after `kill -TERM <pid>` the endpoints answer like this, and the process exits within 3 s:

```text
readyz 503  livez 200
```

`./readiness-check.sh` now shows only `PASS` lines.

---

#### M4 — Miri on the unsafe code

**Goal:** find out whether the `unsafe` block in `decode` is really safe, and make it safe.

**Guidance:**

1. Run `cargo +nightly miri test --lib`. It passes. Why? Which inputs do the tests give to `decode`?
2. Add a test with a character that is not ASCII, for example `decode("é")`, and run Miri again.
3. Fix `decode` without `unsafe`, and correct the comment. Then Miri must pass.
4. Commit (`fix(code): no out-of-bounds read in decode`).

<details>
<summary>Hint</summary>

`"é"` is two bytes in UTF-8 (the usual encoding of Rust strings), both above 127, and the table has 128 entries. `VALUES.get(usize::from(byte))` returns `None` for such a byte; with `?` the function returns `None` too. A plain `cargo test` may even pass with the bug: reading outside the table is undefined behaviour, and the program may read whatever lies next to it.

</details>

**Done when:** before the fix, Miri reports the bug (shortened; the test name is yours):

```text
test code::more_tests::rejects_non_ascii ... error: Undefined Behavior: in-bounds pointer arithmetic failed: attempting to offset pointer by 195 bytes, but got alloc43 which is only 128 bytes from the end of the allocation
  --> src/code.rs:38:31
   |
38 |         let value = unsafe { *VALUES.as_ptr().add(usize::from(byte)) };
   |                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
```

and after the fix `cargo +nightly miri test --lib` ends with `test result: ok.` and no error (Miri runs only the library's unit tests here; the API tests use networking parts of Tokio that Miri is not needed for).

---

#### M5 — Fuzz the two parsers

**Goal:** `decode` and `validate_url` can take **any** text without panicking, and every crash the fuzzer found is a regression test.

**Guidance:**

1. `cargo fuzz init -t decode` and `cargo fuzz add validate_url`. Each target takes `text: &str`.
2. `decode`: call it, and if it returns `Some(n)`, check the property `decode(&encode(n)) == Some(n)`.
3. `validate_url`: call it, and if it returns `Ok(url)`, check that `url` starts with `http://` or `https://` and has no spaces at either end.
4. Run each target with `-- -max_total_time=60`. For every crash: read it (`cargo +nightly fuzz run <target> <artifact>`), fix the code, add a regression test with that input, run again.
5. Commit the targets and the fixes (`test: fuzz targets for decode and validate_url`).

<details>
<summary>Hint 1 — what the fuzzer finds</summary>

In `validate_url`: `end byte index 7 is out of bounds for string of length 0`. In `decode` (once Miri's bug is fixed): `attempt to multiply with overflow`, for a long code such as `"OOHttztOHtt"`. If you skipped M4, `decode` first reports `AddressSanitizer: global-buffer-overflow`: the same bug Miri found.

</details>

<details>
<summary>Hint 2 — the fixes</summary>

`str::starts_with` never panics. For the overflow, `n.checked_mul(62)?.checked_add(u64::from(value))?` returns `None` instead of a wrong number. Add `assert_eq!(decode(&encode(u64::MAX)), Some(u64::MAX));` so you know the largest valid code still works.

</details>

**Done when:** both targets run for 60 seconds without a crash and end with a line like:

```text
Done 1102458 runs in 61 second(s)
```

(your number of runs will differ), and `cargo test` passes with the new regression tests.

---

#### M6 — The supply chain

**Goal:** the dependencies are checked, recorded and shipped with the binary.

**Guidance:**

1. Write `deny.toml` as in Lesson 10.13 and add `publish = false` to `Cargo.toml`. Run `cargo deny check licenses` and decide which licenses to allow: read them, do not just copy the list.
2. `cargo audit`, then `cargo vet init` and `cargo vet`. Commit `supply-chain/`.
3. Build the release with `cargo auditable build --release`, check it with `cargo audit bin`, and create the SBOM with `cargo cyclonedx --format json`.
4. Add the commands to `READINESS.md` as evidence. Commit (`chore: supply-chain checks`).

<details>
<summary>Hint — the license list</summary>

With only `MIT` and `Apache-2.0`, `redis` (BSD-3-Clause, a Berkeley Software Distribution license), `xxhash-rust` (BSL-1.0, the Boost Software License) and the `icu_*` crates (Unicode-3.0) are rejected. All three are permissive licenses; if your team accepts them, add `"BSD-3-Clause"`, `"BSL-1.0"` and `"Unicode-3.0"` to `allow`.

</details>

**Done when** (your counts will differ):

```text
advisories ok, bans ok, licenses ok, sources ok
Vetting Succeeded (106 exempted)
       Found 'cargo auditable' data in target/release/p20-shortener (102 dependencies)
```

`cargo audit` and `cargo audit bin` find nothing, and `jq '.components | length' p20-shortener.cdx.json` prints a number (here `101`).

---

#### M7 — The chaos test

**Goal:** prove with numbers that the service fails **fast** when Redis is slow or down, and recovers by itself.

**Guidance:** save this script as `chaos.sh` and make it executable. For each state of the proxy it asks for `/1` and `/readyz` and prints the status codes and the time of the `/1` request.

```bash
#!/usr/bin/env bash
# Breaks the connection to Redis in three ways and checks that the service answers fast every time.
set -u
API=http://127.0.0.1:18474
probe() {
    local label="$1"
    local link ready
    link=$(curl -s -o /dev/null -w '%{http_code} in %{time_total}s' http://127.0.0.1:3000/1)
    ready=$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:3000/readyz)
    echo "$label  GET /1: $link   readyz: $ready"
}

probe "normal          "
curl -s -o /dev/null -X POST "$API/proxies/redis/toxics" \
    -d '{"name":"slow","type":"latency","stream":"downstream","attributes":{"latency":1000}}'
probe "Redis 1 s slow  "
curl -s -o /dev/null -X DELETE "$API/proxies/redis/toxics/slow"
curl -s -o /dev/null -X POST "$API/proxies/redis" -d '{"enabled":false}'
probe "Redis down      "
curl -s -o /dev/null -X POST "$API/proxies/redis" -d '{"enabled":true}'
sleep 2
probe "Redis back      "
```

This is what it printed for the **starter**, the "before" picture (you can see it yourself if you run the script right after M1; reading ahead is allowed):

```text
normal            GET /1: 307 in 0.002004s   readyz: 404
Redis 1 s slow    GET /1: 000 in 0.506946s   readyz: 000
Redis down        GET /1: 000 in 0.002065s   readyz: 000
Redis back        GET /1: 307 in 0.002589s   readyz: 404
```

`000` means "no HTTP (HyperText Transfer Protocol) answer at all": the request panicked. (`/readyz` does not exist in the starter, so it is treated as a code.) The slow case takes 0.5 s because only the Redis client's internal timeout stopped it.

<details>
<summary>Hint — when the result is not what you expect</summary>

If the slow case takes 1 s, your deadline does not cover the store call: check that the timeout layer is on the route and that `.with_state(...)` comes last. If "Redis back" still shows `503`, wait a little longer: the `ConnectionManager` reconnects with a short backoff.

</details>

**Done when:** against your hardened service (with the link `1` created first):

```text
normal            GET /1: 307 in 0.002501s   readyz: 200
Redis 1 s slow    GET /1: 503 in 0.302653s   readyz: 503
Redis down        GET /1: 503 in 0.000754s   readyz: 503
Redis back        GET /1: 307 in 0.001027s   readyz: 200
```

No line shows `000`, no answer takes longer than about 0.31 s, and the service recovered without a restart. Commit `chaos.sh` (`test: chaos test with toxiproxy`).

---

#### M8 — The readiness review

**Goal:** a finished `READINESS.md` and a project that someone else can run.

**Guidance:**

1. Go through `READINESS.md` once more. Every item has evidence (a test name, a command, a file) or "not needed" with a reason, or "missing" with a plan (Phase 21 items such as rollbacks and dashboards stay "missing" for now).
2. Write `README.md`: what the service does, how to start Redis and toxiproxy, how to run the tests, the fuzzers, Miri and `chaos.sh`.
3. `./readiness-check.sh` must show only `PASS`.
4. Clean up Docker when you are done: `docker rm -f course-p20-toxiproxy course-p20-redis` and `docker network rm course-p20-net`.
5. Commit (`docs: readiness review and README`) and merge the branch.

**Done when:** `./readiness-check.sh` prints six `PASS` lines and exits with `0`, and a classmate (or you, tomorrow) can follow `README.md` from a fresh clone.

### Stretch goals

- **A rate limit per client** on `POST /links` with the token bucket of Lesson 20.8: `429` with `retry-after`, and a test.
- **Before and after under load:** run `oha -z 10s -c 200 http://127.0.0.1:3000/1` (Lesson 19.2) against the starter and the hardened service while Redis is slow. Compare the latency percentiles and the status codes.
- **More toxics:** `timeout` (the connection stops answering) and `slicer` (the answer arrives in tiny pieces). Does the service still answer within 300 ms?
- **A CI (Continuous Integration) workflow** with three jobs: the readiness check on every push, the fuzzers and Miri on a schedule (Lessons 20.3 and 20.4), and the release steps of Lesson 20.7.

## Common mistakes

**1. Hardening without measuring.** A timeout that is not covered by a test or by the chaos script may be in the wrong place (inside the limit, or not on the route). Every change gets a test or a number.

**2. Silencing Miri or the fuzzer.** Marking the failing test `#[cfg_attr(miri, ignore)]` or shortening the fuzz run hides the bug. Fix the code; ignore only tests that Miri cannot run at all (network, files), with a comment.

**3. Checking Redis in liveness.** Then a Redis outage restarts every copy of the service in a loop. Redis belongs in readiness only.

## Quiz

1. In M1 the `https:/` request ended with curl exit code 52, but the service kept running. Why, and why is that still a serious problem?

<details>
<summary>Answer</summary>

axum runs each connection in its own Tokio task; the panic unwound that task only, so the server survived. But the client got no answer at all, the panic message is the only trace, and any user can repeat it as often as they like: a cheap way to waste server resources and fill the logs (a denial of service).

</details>

2. Why did `cargo +nightly miri test --lib` pass on the starter code, although `decode` had undefined behaviour?

<details>
<summary>Answer</summary>

Miri checks only the code that runs. The starter's tests called `decode` with ASCII text only, so the out-of-bounds read never happened. Miri needs tests (or fuzz inputs) that reach the dangerous path.

</details>

3. With a 1 s latency toxic, the starter answered after 0.5 s and the hardened service after 0.3 s. Where do the two numbers come from?

<details>
<summary>Answer</summary>

0.5 s is the default response timeout inside the Redis `ConnectionManager`; the starter then panicked on the error. 0.3 s is the service's own deadline (the timeout layer), which turns the slow request into a `503` before the client's timeout expires.

</details>

## Summary

- Start with a baseline: run the service, break it on purpose, and write every gap into the checklist before you change code.
- Turn every storage error into a `503` with `retry-after`, give each request one deadline, and add body, concurrency and load-shedding limits, each covered by a test with a fake store.
- Separate liveness from readiness, check the configuration at start, and drain on SIGTERM.
- Miri finds undefined behaviour only on paths that tests reach; the fuzzer finds the inputs nobody thought of. Every finding becomes a regression test.
- `cargo deny`, `cargo audit`, `cargo vet`, `cargo auditable` and an SBOM make the dependencies visible and checked.
- A chaos test with toxiproxy turns "it should fail fast" into numbers: no answer slower than the deadline, no panic, recovery without a restart.

## Sources and further reading

- [toxiproxy on GitHub](https://github.com/Shopify/toxiproxy) — the HTTP API, all toxics and their attributes.
- [axum: `DefaultBodyLimit`](https://docs.rs/axum/latest/axum/extract/struct.DefaultBodyLimit.html) — the body size limit of the extractors.
- [redis: `ConnectionManager`](https://docs.rs/redis/latest/redis/aio/struct.ConnectionManager.html) — reconnection and the default timeouts.
- [Rust Fuzz Book](https://rust-fuzz.github.io/book/) — `cargo fuzz` targets, corpus and crash files.
- [Miri on GitHub](https://github.com/rust-lang/miri) — what Miri checks and its limits.
- [cargo-deny book](https://embarkstudios.github.io/cargo-deny/) and [cargo-vet book](https://mozilla.github.io/cargo-vet/) — the supply-chain policies.

---

[🏠 Start](../START-HERE.md) · [← 20.9 The production-readiness checklist](09-production-readiness-checklist.md) · [📋 Phase 20](README.md) · [20.11 Checkpoint →](11-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
