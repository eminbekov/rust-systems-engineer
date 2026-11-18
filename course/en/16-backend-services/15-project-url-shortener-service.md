<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.14 Testing a backend service](14-testing-a-backend-service.md) · [📋 Phase 16](README.md) · [16.16 Checkpoint →](16-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/15-project-url-shortener-service.md) <!-- nav -->

# Lesson 16.15 — Project: a URL shortener service

> **Phase 16 — Backend services** · Lesson 15 of 16 · ⏱ about 12 hours (6 sittings)

## What you will learn

- How to combine everything from this phase into one real service: a REST API, PostgreSQL with migrations, a Redis cache, authentication, rate limiting, traces, a container image and integration tests.
- How to work from a written contract and prove each milestone with acceptance tests.
- How to make design decisions and write them down in a README.

This lesson has no full solution. The plan gives you a contract, milestones, small hints and the exact commands that show a milestone is done. Plan for several sittings; the time above is only for reading the plan.

## Before you start

- You finished [Lesson 16.14 — Testing a backend service](14-testing-a-backend-service.md). Every lesson of this phase is used here; keep them open.
- Docker is running. Free local ports: 8080 (the service), 15432 and 16379 (development database and cache), 16686 (Jaeger).
- Create the project and a Git branch:

```bash
cd ~/projects/rust
git switch main
git switch -c project/p16-shortener
cargo new code/p16-shortener
```

## New words

| Word | Meaning |
|---|---|
| **URL shortener** | A service that turns a long URL (Uniform Resource Locator, a web address) into a short one, and sends visitors of the short one on to the long one. |
| **short code** | The short part of the link, such as `learn` in `http://127.0.0.1:8080/learn`. |
| **redirect** | An HTTP (Hypertext Transfer Protocol) response with a `3xx` status and a `Location` header: "go to this other address". |
| **`302 Found`** | A temporary redirect: browsers do not remember it, so every visit comes back to the shortener and can be counted. (`301`/`308` are permanent and cached by browsers.) |
| **bearer token** | A secret string a client sends in the header `Authorization: Bearer <token>` to prove it may use an endpoint (Lesson 16.8). |
| **constant-time comparison** | Comparing two secrets in a way that always takes the same time, so an attacker cannot guess a token byte by byte from response times. |
| **design decision record** | A short note in the README: what you decided, why, and what you gave up. |

## The project

### What you will build

```text
              ┌──────────────────── p16-shortener (axum) ─────────────────────┐
 curl / browser│  rate limit (Redis INCR) → auth (admin routes) → handlers     │
   ───────────►│                                                               │
               │  GET /{code}: cache-aside ──► Redis   link:v1:{code}          │
               │                    miss ────► PostgreSQL  links table        │
               │  spans ──► OTLP ──► Collector ──► Jaeger (UI on 16686)        │
               └───────────────────────────────────────────────────────────────┘
```

An admin creates short links with a secret token. Anyone can open a short link and is redirected. The service counts clicks, caches lookups in Redis, limits how many requests one client may send, sends traces to Jaeger, runs in Docker Compose, and has integration tests that start their own databases.

### Contract

**Configuration** (environment variables, read once at start-up, Lesson 16.13):

| Variable | Required | Default | Meaning |
|---|---|---|---|
| `DATABASE_URL` | yes | – | PostgreSQL connection URL |
| `REDIS_URL` | no | `redis://127.0.0.1:6379/` | Redis connection URL |
| `ADMIN_TOKEN` | yes | – | The admin bearer token; at least 32 characters |
| `PUBLIC_BASE_URL` | no | `http://127.0.0.1:<APP_PORT>` | Used to build `short_url` |
| `APP_HOST` / `APP_PORT` | no | `127.0.0.1` / `8080` | Listen address |
| `RATE_LIMIT_PER_MINUTE` | no | `60` | Requests per client IP (Internet Protocol) address per minute |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | no | `http://localhost:4317` | Where traces go (Lesson 16.12) |

A missing or invalid value stops the program with exit code 1 and the message `invalid configuration: …`. Secrets are never logged.

**Endpoints:**

| Method and path | Auth | Success | Errors |
|---|---|---|---|
| `GET /healthz` | – | `200`, body `ok` | – |
| `GET /readyz` | – | `200` if PostgreSQL answers | `503` |
| `POST /api/links` | admin | `201` + created link | `401`, `409`, `422`, `429` |
| `GET /api/links/{code}/stats` | admin | `200` + stats | `401`, `404`, `429` |
| `GET /{code}` | – | `302` with `Location` and `x-cache` | `404`, `429` |

- **Request** for `POST /api/links`: `{"url": "<absolute http or https URL, at most 2048 bytes>", "code": "<optional custom code>"}`. A custom code is 3–32 characters from `A–Z a–z 0–9 - _` and is not `api`. Without `code`, the service generates 7 random letters and digits and retries on a collision.
- **Created link** (`201`): `{"code":"learn","url":"https://www.rust-lang.org/learn","short_url":"http://127.0.0.1:8080/learn"}`.
- **Stats** (`200`): `{"code":"learn","url":"https://www.rust-lang.org/learn","clicks":3}`. Every successful redirect adds one click.
- **Errors** always have a JSON (JavaScript Object Notation) body `{"error":"<message>"}`. `401` also sends `WWW-Authenticate: Bearer`; `429` also sends `Retry-After: <seconds>`.
- **`x-cache`** on redirects: `miss` (read from PostgreSQL, now cached), `hit` (read from Redis), or `unavailable` (Redis did not answer; PostgreSQL was used). The redirect must work in all three cases.
- **Rate limit:** at most `RATE_LIMIT_PER_MINUTE` requests per client IP address per fixed one-minute window, for every endpoint except `/healthz` and `/readyz`. If Redis is down, requests are allowed (and a warning is logged).

**Database** (the first migration, `migrations/20261118000000_create_links.sql`):

```sql
CREATE TABLE links (
    code TEXT PRIMARY KEY,
    url TEXT NOT NULL,
    clicks BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Redis keys:** `link:v1:{code}` → the URL, TTL 3600 s; `rate:v1:{ip}` → the request counter of the current window, TTL 60 s.

**Traces:** service name `p16-shortener`. A redirect produces the spans `request` → `redirect` → `cache_get`, then (on a miss) `db_find_link` and `cache_put`, then `db_count_click`.

**Quality:** `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test` pass; one Git commit per milestone; a `README.md` with how to run, how to test, and your design decisions.

Suggested layout:

```text
code/p16-shortener/
├── Cargo.toml
├── README.md
├── Dockerfile
├── compose.yaml
├── otel-collector.yaml
├── .env                  ← ADMIN_TOKEN for Compose; listed in .gitignore
├── migrations/
│   └── 20261118000000_create_links.sql
├── src/
│   ├── main.rs           ← config, connections, serve, shutdown
│   ├── lib.rs            ← AppState, app(state) -> Router, handlers
│   ├── config.rs
│   └── telemetry.rs
└── tests/
    └── api.rs            ← testcontainers + oneshot
```

> 🔒 **Security:** The service redirects to any URL an admin stores, which is why only admins may create links. Never allow anonymous creation without more checks: open redirectors are popular with phishing. Never put `ADMIN_TOKEN` in `compose.yaml`, in Git or in logs.

### Step-by-step plan

For milestones M1–M6, run the service on your Mac with `cargo run`, and its databases in Docker. Create `compose.yaml` with only the two databases first:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev-only-password
      POSTGRES_DB: shortener
    ports:
      - "127.0.0.1:15432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d shortener"]
      interval: 5s
      retries: 10
  cache:
    image: redis:8
    ports:
      - "127.0.0.1:16379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 10
```

Start them and set the environment in the terminal where you run the service:

```bash
docker compose up -d --wait
export DATABASE_URL=postgres://app:dev-only-password@127.0.0.1:15432/shortener
export REDIS_URL=redis://127.0.0.1:16379/
export ADMIN_TOKEN=dev-token-change-me-0123456789abcdef
export TOKEN=$ADMIN_TOKEN
```

`--wait` returns when both health checks pass. The ports 15432 and 16379 avoid clashes with the containers from earlier lessons.

#### M1 — Skeleton: configuration, health, shutdown

**Goal:** the service starts, loads and checks its configuration, answers `/healthz` and `/readyz`, and stops cleanly on Ctrl+C and `SIGTERM`.

**Guidance:** reuse `Config`, `from_lookup`, `healthz`, `readyz` and `shutdown_signal` from Lesson 16.13, with the variables of the contract. Put the router in `pub fn app(state: AppState) -> Router` in `src/lib.rs`; `AppState` holds the `PgPool` for now. Run the migrations at start-up with `sqlx::migrate!()` (Lesson 16.7). Write unit tests for the configuration rules (missing `ADMIN_TOKEN`, a token shorter than 32 characters).

<details>
<summary>Hint</summary>

The token rule is one more check in `from_lookup`: after reading `ADMIN_TOKEN`, return `ConfigError::Invalid` if `token.len() < 32`. Do not put the token itself into the error value: it would be printed.

</details>

**Done when:**

```bash
env -u ADMIN_TOKEN cargo run
```

```text
invalid configuration: environment variable ADMIN_TOKEN is not set
```

```bash
cargo run
```

```text
 INFO p16_shortener: configuration loaded config=Config { listen: 127.0.0.1:8080, public_base_url: "http://127.0.0.1:8080", rate_limit_per_minute: 60, database_url: "<redacted>", redis_url: "<redacted>", admin_token: "<redacted>" }
 INFO p16_shortener: listening on 127.0.0.1:8080
```

In a second terminal (with `TOKEN` exported as above):

```bash
curl -s -w ' %{http_code}\n' http://127.0.0.1:8080/healthz
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/readyz
```

```text
ok 200
200
```

Ctrl+C prints `shutdown signal received` and `stopped cleanly`. `cargo test` passes. Commit: `M1: skeleton with configuration and health checks`.

#### M2 — Links in PostgreSQL: create and redirect

**Goal:** `POST /api/links` stores a link (no authentication yet), `GET /{code}` redirects with `302`, and every error has the JSON body from the contract.

**Guidance:** write an error type `ApiError` with one variant per error status and `impl IntoResponse` (Lesson 16.4). Validate the URL with the `url` crate (`cargo add url@2`): parse it, then check the scheme and that it has a host. Generate codes with `rand` (`cargo add rand@0.10`). Map a unique violation to `409` for custom codes, and retry with a new code for generated ones.

<details>
<summary>Hint</summary>

A random code of 7 letters and digits in `rand` 0.10:

```rust
use rand::RngExt;

fn random_code() -> String {
    rand::rng()
        .sample_iter(rand::distr::Alphanumeric)
        .take(7)
        .map(char::from)
        .collect()
}

fn main() {
    println!("{}", random_code());
}
```

For the unique violation, match `Err(sqlx::Error::Database(db)) if db.is_unique_violation()`, as you matched check violations in Lesson 16.14.

</details>

**Done when:**

```bash
curl -s -X POST http://127.0.0.1:8080/api/links -H 'content-type: application/json' \
  -d '{"url":"https://www.rust-lang.org/learn","code":"learn"}'
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' http://127.0.0.1:8080/learn
curl -s -w ' %{http_code}\n' -X POST http://127.0.0.1:8080/api/links -H 'content-type: application/json' \
  -d '{"url":"ftp://example.com"}'
curl -s -w ' %{http_code}\n' -X POST http://127.0.0.1:8080/api/links -H 'content-type: application/json' \
  -d '{"url":"https://example.com","code":"learn"}'
curl -s -w ' %{http_code}\n' http://127.0.0.1:8080/nope-nope
```

```text
{"code":"learn","url":"https://www.rust-lang.org/learn","short_url":"http://127.0.0.1:8080/learn"}
302 https://www.rust-lang.org/learn
{"error":"url must be an absolute http or https URL"} 422
{"error":"code already taken"} 409
{"error":"no such link"} 404
```

Without `"code"`, the answer has a random 7-character code, for example `{"code":"ucLz4gv","url":"https://doc.rust-lang.org/book/","short_url":"http://127.0.0.1:8080/ucLz4gv"}`. Commit: `M2: create links and redirect`.

#### M3 — Admin authentication and click statistics

**Goal:** `POST /api/links` and `GET /api/links/{code}/stats` require `Authorization: Bearer <ADMIN_TOKEN>`; every redirect increases `clicks`.

**Guidance:** a small function `require_admin(&headers, &state) -> Result<(), ApiError>` called at the start of the two admin handlers is enough (an axum extractor or middleware is a nice alternative, Lesson 16.8). Compare the token in constant time with the `subtle` crate (`cargo add subtle@2`). Count a click with `UPDATE links SET clicks = clicks + 1 WHERE code = $1` after a successful lookup.

<details>
<summary>Hint</summary>

`subtle::ConstantTimeEq` adds `ct_eq` to byte slices: `given.as_bytes().ct_eq(expected.as_bytes()).into()` gives a `bool`. A plain `==` stops at the first different byte, so its time leaks how many leading bytes were right (Lesson 10.9).

</details>

**Done when:**

```bash
curl -s -i -X POST http://127.0.0.1:8080/api/links -H 'content-type: application/json' \
  -d '{"url":"https://example.com"}'
```

```text
HTTP/1.1 401 Unauthorized
content-type: application/json
www-authenticate: Bearer
content-length: 36
date: Wed, 18 Nov 2026 08:18:31 GMT

{"error":"missing or invalid token"}
```

With the header `-H "authorization: Bearer $TOKEN"` the same request returns `201`. After opening `/learn` once since you created it:

```bash
curl -s http://127.0.0.1:8080/api/links/learn/stats -H "authorization: Bearer $TOKEN"
```

```text
{"code":"learn","url":"https://www.rust-lang.org/learn","clicks":1}
```

Commit: `M3: admin token and click statistics`.

#### M4 — A Redis cache for redirects

**Goal:** redirects use cache-aside with `link:v1:{code}` (Lesson 16.10) and report `x-cache`; when Redis is down, redirects still work.

**Guidance:** add a `ConnectionManager` to `AppState`. Create it with `ConnectionManager::new_lazy_with_config`, so the service starts even when Redis is down, and give it short timeouts so a dead Redis costs a fraction of a second, not thirty. Any Redis error means `x-cache: unavailable` and a database lookup. Mark the helper functions with `#[instrument(skip(state))]` now; M6 needs the spans.

<details>
<summary>Hint</summary>

A helper for `src/main.rs`:

```rust
use std::time::Duration;

use redis::aio::{ConnectionManager, ConnectionManagerConfig};

fn cache(url: &str) -> redis::RedisResult<ConnectionManager> {
    let config = ConnectionManagerConfig::new()
        .set_connection_timeout(Some(Duration::from_millis(500)))
        .set_response_timeout(Some(Duration::from_millis(500)))
        .set_number_of_retries(1);
    ConnectionManager::new_lazy_with_config(redis::Client::open(url)?, config)
}
```

Call it from `run` inside the Tokio runtime: the manager needs a runtime to start its background work.

</details>

**Done when:**

```bash
docker compose exec cache redis-cli FLUSHALL
for i in 1 2; do curl -s -o /dev/null -D - http://127.0.0.1:8080/learn | grep -i x-cache; done
docker compose stop cache
curl -s -o /dev/null -D - http://127.0.0.1:8080/learn | grep -iE '^HTTP|x-cache'
docker compose start cache
```

```text
OK
x-cache: miss
x-cache: hit
HTTP/1.1 302 Found
x-cache: unavailable
```

The request while Redis was stopped was still answered quickly (0.19 s on the course's machine, far from the 30-second default timeout), and the service logged two warnings: `rate limiter unavailable, allowing request: …` (from M5, once you have it) and `cache read failed: …`. A few seconds after `docker compose start cache`, redirects are `hit` or `miss` again, without restarting the service. Commit: `M4: Redis cache for redirects`.

#### M5 — Rate limiting

**Goal:** at most `RATE_LIMIT_PER_MINUTE` requests per client IP per minute; the next ones get `429` with `Retry-After`.

**Guidance:** write the limiter as middleware with `axum::middleware::from_fn_with_state` and add it with `route_layer` to a router that has every route except the health checks; then `merge` the health routes in. Get the client address with the extractor `ConnectInfo<SocketAddr>`; for that, serve with `app(state).into_make_service_with_connect_info::<SocketAddr>()`. Use the atomic pipeline from Exercise 3 of Lesson 16.10 and add a `TTL` command to learn the seconds for `Retry-After`.

<details>
<summary>Hint</summary>

In tests there is no real connection, so `ConnectInfo` is missing and the middleware would fail. axum has a helper for exactly this: add `.layer(axum::extract::connect_info::MockConnectInfo(SocketAddr::from(([10, 0, 0, 1], 5000))))` to the router in your tests.

</details>

**Done when:** wait one minute after your last request (so a new window starts), then:

```bash
for i in $(seq 65); do curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/learn; done | sort | uniq -c
curl -s -i http://127.0.0.1:8080/learn | grep -iE '^HTTP|retry-after'
```

```text
  60 302
   5 429
HTTP/1.1 429 Too Many Requests
retry-after: 59
```

(The `Retry-After` value is the number of seconds left in the window, so yours may be a little smaller.) Commit: `M5: rate limiting per client`.

#### M6 — Traces

**Goal:** every request is a trace in Jaeger with the spans from the contract.

**Guidance:** copy `telemetry.rs` from Lesson 16.12 (traces are enough; metrics are a stretch goal) and add `TraceLayer` with `INFO` spans. Add the collector and Jaeger services from Lesson 16.12 to `compose.yaml`, publish the collector's port `127.0.0.1:4317:4317` for now, and run `docker compose up -d --wait` again. Call `shutdown` on the tracer provider before `main` returns.

<details>
<summary>Hint</summary>

If the spans `cache_get` and `db_find_link` appear as separate traces instead of under `redirect`, the functions are called outside the `redirect` span, for example from a `tokio::spawn` task: a spawned task does not inherit the current span unless you attach it with `.instrument(Span::current())` (Lesson 09.10).

</details>

**Done when:** after one redirect with an empty cache, `curl -s http://127.0.0.1:16686/api/v3/services` shows your service:

```text
{"services":["p16-shortener","jaeger"]}
```

and in the Jaeger UI (<http://localhost:16686>, service `p16-shortener`) the trace of `GET /learn` looks like this (durations from one run on the course's machine):

```text
request             1.74 ms
  redirect          1.51 ms
    cache_get       0.12 ms
    db_find_link    0.74 ms
    cache_put       0.09 ms
    db_count_click  0.54 ms
```

Commit: `M6: OpenTelemetry traces`.

#### M7 — Container image and the whole stack in Compose

**Goal:** `docker compose up -d --build` starts PostgreSQL, Redis, the collector, Jaeger and the service; the service waits for healthy databases and is healthy itself.

**Guidance:** write a `Dockerfile` as in Lesson 16.13 (distroless `nonroot`, a `healthcheck` mode, dependency caching from its Exercise 3). Remember to copy `migrations/` into the build stage: `sqlx::migrate!()` reads it at compile time. In `compose.yaml`, add an `app` service with `build: .`, the environment from the contract with host names `db`, `cache` and `collector`, port `127.0.0.1:8080:8080`, and `depends_on` with `condition: service_healthy` for `db` and `cache`. Read the token from a `.env` file next to `compose.yaml` (`ADMIN_TOKEN: ${ADMIN_TOKEN:?set ADMIN_TOKEN in .env}`), and add `.env` to `.gitignore`. Stop the `cargo run` service first: both want port 8080.

<details>
<summary>Hint</summary>

Inside Compose, the client address that the rate limiter sees is the address of Docker's network gateway, so all requests from your Mac share one counter. This is fine here; behind a real load balancer you would read the client address from a trusted `X-Forwarded-For` header (Lesson 15.14) and never from an untrusted one.

</details>

**Done when:**

```bash
docker compose up -d --build
docker compose ps --format 'table {{.Service}}\t{{.Status}}'
```

```text
SERVICE     STATUS
app         Up 20 seconds (healthy)
cache       Up 25 seconds (healthy)
collector   Up 25 seconds
db          Up 25 seconds (healthy)
jaeger      Up 25 seconds
```

The `curl` checks of M2–M6 give the same results against the container. `docker compose stop app` finishes in well under a second, and `docker compose logs app` ends with `shutdown signal received` and `stopped cleanly`. Commit: `M7: Docker image and Compose stack`.

#### M8 — Self-contained integration tests and the README

**Goal:** `cargo test` starts its own PostgreSQL and Redis with testcontainers (Lesson 16.14) and covers every row of the endpoint table; the quality gate passes; the README explains the project.

**Guidance:** a helper `test_app(rate_limit)` starts `postgres:18` and `redis:8` containers, runs the migrations, builds `AppState` and returns the router (with `MockConnectInfo`) together with both containers. Write at least these tests: missing or wrong token → `401`; create → redirect with `x-cache` `miss` then `hit`; invalid input → `422`; custom code twice → `201` then `409`; three redirects → `clicks == 3`; unknown code → `404`; with limit 3, four requests → `404, 404, 404, 429`.

<details>
<summary>Hint</summary>

Redis prints `Ready to accept connections` on standard output when it has started, so wait for it with `WaitFor::message_on_stdout("Ready to accept connections")`. Return a struct with fields `_db` and `_cache` for the containers, so they live as long as the test.

</details>

**Done when:**

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test --test api
```

```text
running 7 tests
test unknown_code_is_404 ... ok
test created_link_redirects_and_is_cached ... ok
test creating_needs_the_admin_token ... ok
test too_many_requests_get_429 ... ok
test stats_count_clicks ... ok
test custom_code_can_be_taken_only_once ... ok
test bad_input_is_rejected ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 7.74s
```

The README has: what the service does; how to run it (`.env`, `docker compose up -d --build`); how to test it; the endpoint table; and at least three design decisions, for example "302 instead of 301, so that clicks can be counted", "the cache is optional: Redis errors fall back to PostgreSQL", "the rate limiter fails open". Commit: `M8: integration tests and README`. Stop everything with `docker compose down`.

### Stretch goals

1. **Expiring links:** an optional `expires_at` in the request (a new migration); expired links answer `410 Gone`, and the cache TTL is never longer than the remaining lifetime.
2. **Cheaper click counting:** count clicks with `INCR clicks:v1:{code}` in Redis and let a background task add the counts to PostgreSQL every few seconds. Write down what is lost if Redis crashes, and why that may be acceptable.
3. **Metrics:** a counter `shortener.redirects` with the attribute `cache` (`hit`, `miss`, `unavailable`) and a histogram of request durations (Lesson 16.12), exported through the collector.
4. **Negative caching:** cache "this code does not exist" for 30 seconds, so that scanning random codes does not hit PostgreSQL every time. Make sure creating that code later removes the cached "not found".
5. **A gRPC admin API:** the same create and stats operations as a tonic service on a second port (Lesson 16.9), sharing the database code with the REST handlers.
6. **CI (Continuous Integration):** a GitHub Actions workflow that runs the quality gate and the testcontainers tests (Lesson 09.12); check it with `actionlint`.

## Quiz

1. Why does the shortener answer with `302 Found` and not `301 Moved Permanently`?

<details>
<summary>Answer</summary>

Browsers remember permanent redirects and go straight to the target next time, so the shortener would never see those clicks. A `302` is temporary: every visit comes back and can be counted (and the target could even be changed).

</details>

2. Redis is down. What happens to redirects, to the rate limiter, and to the `x-cache` header?

<details>
<summary>Answer</summary>

Redirects still work: the URL comes from PostgreSQL and `x-cache` is `unavailable`. The rate limiter allows the request and logs a warning ("fails open"). The service is slower but does not fail.

</details>

3. Why must the admin token be compared in constant time?

<details>
<summary>Answer</summary>

A normal comparison stops at the first different byte. By measuring response times, an attacker could find out how many leading bytes of a guess are right and find the token byte by byte. A constant-time comparison always takes the same time.

</details>

4. Why are the health checks outside the rate limiter?

<details>
<summary>Answer</summary>

Docker (and later Kubernetes) calls them every few seconds. If they were limited, a busy minute could make the health check fail, and the orchestrator would restart a perfectly healthy service.

</details>

## Summary

- A real service combines all the pieces of this phase; a written contract keeps them consistent and testable.
- PostgreSQL is the source of truth; Redis only speeds things up and protects the database, so every Redis failure has a fallback.
- Authentication protects the endpoints that can do harm; rate limiting protects the service from too many requests; neither applies to health checks.
- Traces show the path of each request through cache and database; graceful shutdown makes sure the last spans are sent.
- Integration tests with testcontainers prove the whole contract with real databases, on any machine with Docker.
- Each milestone ends with acceptance tests and a commit; design decisions go into the README.

## Sources and further reading

- [MDN — Redirections in HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Redirections) — permanent and temporary redirects and how browsers treat them.
- [RFC 6585 — Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585) — the definition of `429 Too Many Requests` and `Retry-After`.
- [OWASP — Unvalidated Redirects and Forwards Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html) — why open redirects are dangerous.
- [axum — middleware documentation on docs.rs](https://docs.rs/axum/latest/axum/middleware/index.html) — `from_fn_with_state`, `route_layer` and layering order.
- [subtle on docs.rs](https://docs.rs/subtle) — constant-time comparison for secrets.
- [url on docs.rs](https://docs.rs/url) — parsing and checking URLs.

---

[🏠 Start](../START-HERE.md) · [← 16.14 Testing a backend service](14-testing-a-backend-service.md) · [📋 Phase 16](README.md) · [16.16 Checkpoint →](16-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
