<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.15 Project: a URL shortener service](15-project-url-shortener-service.md) · [📋 Phase 16](README.md) · [Phase 17 — Storage and database internals →](../17-storage-and-databases/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/16-checkpoint.md) <!-- nav -->

# Lesson 16.16 — Checkpoint

> **Phase 16 — Backend services** · Lesson 16 of 16 · ⏱ about 90 minutes

## What you will learn

- Whether you are ready for Phase 17: this lesson checks the whole phase, from REST design to containers.
- Which lessons to review if an answer surprises you.

Answer each question in your own words **before** you open the answer. Do the practical tasks without looking back at the lessons first; look only when you are stuck.

## Before you start

- You finished [Lesson 16.15 — Project: a URL shortener service](15-project-url-shortener-service.md), at least up to milestone M4.
- Docker is running (tasks 2 and 3 use it).

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A lesson that checks what you learned in a phase, with questions and practical tasks instead of new material. |

## Self-test quiz

1. What is a backend service, and what are its typical parts?

<details>
<summary>Answer</summary>

A long-running program that other programs (browsers, apps, other services) call over the network. Typical parts: a network API (Application Programming Interface) such as HTTP (Hypertext Transfer Protocol) or gRPC, business logic, storage (a database), and operational parts: configuration, logs, metrics, traces, health checks.

</details>

2. In a REST API, which HTTP methods are idempotent, and why does that matter?

<details>
<summary>Answer</summary>

`GET`, `PUT` and `DELETE` (and `HEAD`, `OPTIONS`) are idempotent: sending the same request twice has the same effect as sending it once. `POST` is not. It matters for retries: a client may safely retry an idempotent request after a timeout, but a retried `POST` can create a duplicate.

</details>

3. Which status code fits each case: a new resource was created; the request body is not valid JSON (JavaScript Object Notation); the JSON is valid but a field breaks a rule; the caller sent no credentials; the caller is known but not allowed; too many requests?

<details>
<summary>Answer</summary>

`201 Created`; `400 Bad Request`; `422 Unprocessable Entity`; `401 Unauthorized`; `403 Forbidden`; `429 Too Many Requests` (with `Retry-After`).

</details>

4. In axum, how does a handler get the path parameter, the JSON body and the shared application state?

<details>
<summary>Answer</summary>

Through extractors in its arguments: `Path(id): Path<i64>`, `Json(body): Json<NewNote>` and `State(state): State<AppState>`. The state is added once with `.with_state(...)` on the router. The body extractor (`Json`) must be the last argument.

</details>

5. What is middleware in an axum service, and name two things it is good for.

<details>
<summary>Answer</summary>

A layer (a `tower` service) that wraps the handlers and sees every request and response before and after them. Good for things every route needs: logging and tracing (`TraceLayer`), timeouts, authentication, rate limiting, compression, CORS (Cross-Origin Resource Sharing) headers.

</details>

6. Why must SQL (Structured Query Language) values be passed with `.bind(...)` and never formatted into the query string?

<details>
<summary>Answer</summary>

With `.bind`, the value travels separately from the SQL text, so the database never interprets it as SQL. Formatting user input into the string allows SQL injection (Lesson 10.5): input like `'; DROP TABLE links; --` would become part of the command.

</details>

7. What does sqlx check at compile time with `query!`, and what is offline mode for?

<details>
<summary>Answer</summary>

The `query!` macros connect to a database (from `DATABASE_URL`) while compiling and check that the SQL is valid and that the Rust types match the columns and parameters. Offline mode stores that information in the `.sqlx` folder (created by `cargo sqlx prepare`), so the project builds without a database, for example in CI (Continuous Integration) or in a Docker build.

</details>

8. Why do schema changes go into migration files instead of being typed into `psql` by hand? And what does a transaction guarantee?

<details>
<summary>Answer</summary>

Migrations are versioned files in Git, applied in order and recorded in the database, so every environment (laptop, test, production) gets exactly the same schema, and tests can create it from scratch. A transaction groups several statements so that either all of them take effect (`COMMIT`) or none (`ROLLBACK` or an error), and other connections never see a half-done change.

</details>

9. How must a web service store user passwords, and why not with SHA-256 (Secure Hash Algorithm, 256-bit)?

<details>
<summary>Answer</summary>

With a slow, salted password-hashing function such as Argon2 (Lesson 10.6), and only the hash is stored. SHA-256 is designed to be fast, so an attacker who steals the database can try billions of guesses per second; Argon2 makes each guess deliberately expensive in time and memory.

</details>

10. In a `.proto` file, which changes are safe for old clients, and which are not?

<details>
<summary>Answer</summary>

Safe: adding a field with a new number, renaming a field, adding a new RPC (Remote Procedure Call). Not safe: changing a field's number or type, or reusing the number of a deleted field (mark it `reserved` instead).

</details>

11. Describe cache-aside in three steps, and say what the service does when Redis is down.

<details>
<summary>Answer</summary>

(1) Read the key from the cache; on a hit, return it. (2) On a miss, read the database. (3) Store the result in the cache with a TTL (Time To Live) and return it. When Redis is down, the service logs a warning and uses the database directly: the cache is an optimization, never the source of truth.

</details>

12. In NATS, what is the difference between two normal subscribers and two subscribers in a queue group? Which delivery promise does JetStream add?

<details>
<summary>Answer</summary>

Two normal subscribers both get every message; two queue-group members share the messages, each one going to only one of them. JetStream stores messages in a stream and gives at-least-once delivery with acknowledgements, so handlers must be idempotent.

</details>

13. Which question does each signal answer: logs, metrics, traces?

<details>
<summary>Answer</summary>

Logs: what exactly happened in one event. Metrics: how much and how often, over time. Traces: where the time went in one request, across services.

</details>

14. What is the difference between a liveness check and a readiness check? What should each one test?

<details>
<summary>Answer</summary>

Liveness asks "is the process alive and able to answer?"; failing it makes the orchestrator restart the container, so it must not depend on other services. Readiness asks "can it do real work now?"; it may check the database, and failing it only stops traffic to that copy.

</details>

15. Why does a service in a container need to handle `SIGTERM`?

<details>
<summary>Answer</summary>

`docker stop` and Kubernetes send `SIGTERM` and, after a grace period, `SIGKILL`. Handling `SIGTERM` lets the service stop accepting new requests, finish the running ones, and flush logs and telemetry before it exits.

</details>

16. You test a handler that reads from PostgreSQL. Name two ways to give the test a real, isolated database.

<details>
<summary>Answer</summary>

(1) testcontainers: the test starts its own `postgres:18` container on a random port and removes it afterwards. (2) `#[sqlx::test]`: with one running server in `DATABASE_URL`, sqlx creates a fresh database with all migrations for each test and drops it afterwards.

</details>

## Practical tasks

### Task 1 — An endpoint with tests (axum)

Create `code/p16-checkpoint-sum`. Write `GET /sum?a=<i32>&b=<i32>` that answers `{"sum": a + b}`. Text instead of a number must give `400`; a sum that does not fit into `i32` must give `422`. Write the tests **in-process** with `oneshot`, without starting a server.

<details>
<summary>Hint</summary>

`Query(pair): Query<Pair>` with `#[derive(Deserialize)] struct Pair { a: i32, b: i32 }` rejects bad numbers with `400` by itself. For the overflow, use `checked_add`, which returns `None` instead of panicking. You need `tower` with the feature `util` as a dev-dependency.

</details>

<details>
<summary>Answer</summary>

```rust
use axum::extract::Query;
use axum::http::StatusCode;
use axum::routing::get;
use axum::{Json, Router};
use serde::Deserialize;
use serde_json::{Value, json};

#[derive(Deserialize)]
struct Pair {
    a: i32,
    b: i32,
}

async fn sum(Query(pair): Query<Pair>) -> Result<Json<Value>, StatusCode> {
    let total = pair
        .a
        .checked_add(pair.b)
        .ok_or(StatusCode::UNPROCESSABLE_ENTITY)?;
    Ok(Json(json!({ "sum": total })))
}

fn app() -> Router {
    Router::new().route("/sum", get(sum))
}

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app()).await.unwrap();
}

#[cfg(test)]
mod tests {
    use super::app;
    use axum::body::Body;
    use axum::http::{Request, StatusCode};
    use tower::ServiceExt;

    async fn call(uri: &str) -> (StatusCode, String) {
        let request = Request::get(uri).body(Body::empty()).unwrap();
        let response = app().oneshot(request).await.unwrap();
        let status = response.status();
        let body = axum::body::to_bytes(response.into_body(), usize::MAX)
            .await
            .unwrap();
        (status, String::from_utf8(body.to_vec()).unwrap())
    }

    #[tokio::test]
    async fn adds_two_numbers() {
        assert_eq!(
            call("/sum?a=2&b=40").await,
            (StatusCode::OK, r#"{"sum":42}"#.to_string())
        );
    }

    #[tokio::test]
    async fn rejects_text_and_overflow() {
        assert_eq!(call("/sum?a=x&b=1").await.0, StatusCode::BAD_REQUEST);
        assert_eq!(
            call("/sum?a=2147483647&b=1").await.0,
            StatusCode::UNPROCESSABLE_ENTITY
        );
    }
}
```

Dependencies: `axum@0.8`, `serde@1` (feature `derive`), `serde_json@1`, `tokio@1` (features `macros,rt-multi-thread`), and `tower@0.5` (feature `util`) with `--dev`.

```text
test tests::adds_two_numbers ... ok
test tests::rejects_text_and_overflow ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the `Query` extractor parses the query string into `Pair` and rejects it with `400` before the handler runs. `checked_add` makes the overflow an ordinary value we can map to `422`; with `+`, a debug build would panic and the client would get no proper answer.

</details>

### Task 2 — SQL on the links table

Start a scratch PostgreSQL and create this table with data:

```bash
docker run -d --name p16-checkpoint-db -e POSTGRES_PASSWORD=test postgres:18
docker exec -it p16-checkpoint-db psql -U postgres
```

```sql
CREATE TABLE links (code TEXT PRIMARY KEY, url TEXT NOT NULL, clicks BIGINT NOT NULL DEFAULT 0);
INSERT INTO links (code, url, clicks) VALUES
  ('learn', 'https://www.rust-lang.org/learn', 42),
  ('book',  'https://doc.rust-lang.org/book/', 42),
  ('std',   'https://doc.rust-lang.org/std/', 7),
  ('tokio', 'https://tokio.rs/', 19),
  ('axum',  'https://docs.rs/axum', 0);
```

Write two queries: (a) the three most-clicked links, with ties ordered by code; (b) how many links have at least one click, and their total number of clicks.

<details>
<summary>Hint</summary>

(a) needs `ORDER BY` on two columns and `LIMIT`. (b) needs `count(*)`, `sum(...)` and a `WHERE`.

</details>

<details>
<summary>Answer</summary>

```sql
SELECT code, clicks FROM links ORDER BY clicks DESC, code LIMIT 3;
SELECT count(*) AS links, sum(clicks) AS total_clicks FROM links WHERE clicks > 0;
```

```text
 code  | clicks 
-------+--------
 book  |     42
 learn |     42
 tokio |     19
(3 rows)

 links | total_clicks 
-------+--------------
     4 |          110
(1 row)
```

Why it works: `ORDER BY clicks DESC, code` sorts by clicks and, for equal clicks, alphabetically, so the result is always the same (without the second column, `book` and `learn` could come in any order). `WHERE` removes the row with 0 clicks before `count` and `sum` run. Remove the container afterwards with `docker rm -f p16-checkpoint-db`.

</details>

### Task 3 — Find the five problems in a Compose file

This `compose.yaml` runs a service `p16-app` built like the one in Lesson 16.13 (distroless image, a `healthcheck` mode, configuration from the environment). It is valid YAML (the text format of Compose files), and `docker compose config` accepts it, but it does not work. Find five problems and fix them.

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD: dev-only-password
  app:
    image: p16-app
    environment:
      APP_HOST: 127.0.0.1
      DATABASE_URL: postgres://postgres:dev-only-password@localhost:5432/postgres
    ports:
      - "8080:8080"
    depends_on:
      - db
    healthcheck:
      test: curl -f http://localhost:8080/healthz
```

<details>
<summary>Hint</summary>

Think about: which address the service listens on inside its container; what `localhost` means inside a container; when `app` starts relative to PostgreSQL being ready; what the distroless image contains; who can reach port 8080 on your machine.

</details>

<details>
<summary>Answer</summary>

1. `APP_HOST: 127.0.0.1` makes the service unreachable from outside its container ("Empty reply from server"). Use `0.0.0.0`.
2. `localhost` in `DATABASE_URL` is the `app` container itself. Use the service name: `@db:5432`.
3. Plain `depends_on` waits only for the `db` container to start. Give `db` a `healthcheck` with `pg_isready` and use `condition: service_healthy`.
4. The shell-form health check needs `/bin/sh` and `curl`, which the distroless image does not have. Use the exec form with the program's own mode: `["CMD", "/app/p16-app", "healthcheck"]`.
5. `"8080:8080"` publishes the port on all network interfaces of your machine. Use `"127.0.0.1:8080:8080"` for local development.

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD: dev-only-password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10
  app:
    image: p16-app
    environment:
      APP_HOST: 0.0.0.0
      DATABASE_URL: postgres://postgres:dev-only-password@db:5432/postgres
    ports:
      - "127.0.0.1:8080:8080"
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "/app/p16-app", "healthcheck"]
```

Why it works: each fix matches one lesson rule: listen on all interfaces inside containers, address other containers by service name, start only after dependencies are healthy, health checks must use programs that exist in the image, and publish ports only where needed. `docker compose -f compose.yaml config -q` checks the fixed file's syntax.

</details>

### Task 4 — Choose the communication style

For each case, choose REST over HTTP, gRPC, core NATS publish/subscribe, or NATS JetStream, and give one reason:

(a) A mobile app loads the user's order history. (b) The orders service must tell billing, email and analytics that an order was created, and billing must never miss one. (c) The checkout service asks the pricing service for 200 prices per second inside your network, with a strict contract. (d) A dashboard shows the live CPU (Central Processing Unit) load of servers; old values are useless.

<details>
<summary>Hint</summary>

Ask two questions per case: does the caller need an answer now? Is losing a message acceptable?

</details>

<details>
<summary>Answer</summary>

(a) REST over HTTP: a public client, simple to call from any device, easy to cache and debug. (b) NATS JetStream: one event for many receivers, stored and delivered at least once, so billing gets it even after a restart. (c) gRPC: internal, frequent calls with a checked `.proto` contract and small binary messages. (d) Core NATS publish/subscribe: fast fan-out to whoever is listening now; a lost value is replaced by the next one a second later.

Why it works: the choice follows from whether the caller waits for a reply (REST, gRPC, request/reply) or not (events), and from whether messages may be lost (core NATS) or not (JetStream).

</details>

## If you got something wrong

| Topic | Review |
|---|---|
| What a service is, its parts | [16.1 — What a backend service is](01-what-a-backend-service-is.md) |
| REST, methods, status codes, idempotency | [16.2 — Designing HTTP APIs: REST](02-designing-http-apis.md) |
| axum routing, extractors, state | [16.3 — Web services with axum](03-web-services-with-axum.md) |
| Error types, validation, middleware | [16.4 — Errors, validation and middleware](04-errors-validation-and-middleware.md) |
| SQL queries, tables, indexes | [16.5 — SQL and PostgreSQL basics](05-sql-and-postgresql-basics.md) |
| sqlx, `query!`, offline mode | [16.6 — Databases with sqlx](06-databases-with-sqlx.md) |
| Migrations, transactions | [16.7 — Migrations and transactions](07-migrations-and-transactions.md) |
| Passwords, sessions, tokens | [16.8 — Authentication in a web service](08-authentication-in-a-web-service.md) |
| `.proto`, tonic, status codes, streaming | [16.9 — gRPC with tonic](09-grpc-with-tonic.md) |
| Cache-aside, TTL, invalidation, rate limits | [16.10 — Caching with Redis](10-caching-with-redis.md) |
| Subjects, queue groups, JetStream | [16.11 — Messaging with NATS](11-messaging-with-nats.md) |
| Logs, metrics, traces, the Collector | [16.12 — Logs, metrics and traces with OpenTelemetry](12-logs-metrics-and-traces.md) |
| Configuration, health checks, `SIGTERM`, images | [16.13 — Configuration, health checks and containers](13-configuration-health-checks-and-containers.md) |
| `oneshot`, testcontainers, `#[sqlx::test]` | [16.14 — Testing a backend service](14-testing-a-backend-service.md) |
| Putting it all together | [16.15 — Project: a URL shortener service](15-project-url-shortener-service.md) |

## What's next

In this phase you **used** PostgreSQL and Redis as black boxes: you sent queries and trusted the answers. [Phase 17 — Storage and database internals](../17-storage-and-databases/README.md) opens the box. You will learn how disks and files really work, why `fsync` matters for crash safety, and build the pieces of a database yourself: a write-ahead log, hash indexes, B-trees and LSM (log-structured merge) trees, transactions and isolation levels, and finally your own key-value store. After that, the services you build will make better use of their databases, because you will know what each query costs.

Before you start Phase 17, tidy up the containers of this phase: `docker rm -f p16-redis p16-nats` removes the cache and
the message broker (`docker ps -a --filter name=p16-` lists what is left). Keep `p16-postgres` only if you still want the
tasks data; otherwise remove it and its volume as Lesson 16.5 showed.

## Summary

- A backend service is an API plus logic plus storage plus the operational parts that keep it observable and restartable.
- REST over HTTP for public APIs, gRPC for internal calls with a strict contract, NATS for events and work queues; JetStream when messages must not be lost.
- PostgreSQL is the source of truth: parameterized queries, migrations and transactions keep it correct; Redis only makes it faster.
- Observability means logs, metrics and traces; OpenTelemetry sends them in one standard way.
- Containers need configuration from the environment, liveness and readiness checks, `SIGTERM` handling and small non-root images.
- Test services at every level, with real databases in containers for the SQL parts.

## Sources and further reading

- [The Twelve-Factor App](https://12factor.net/) — a short checklist for services that are easy to run, which this phase followed.
- [axum documentation on docs.rs](https://docs.rs/axum) — the reference for everything you built with axum.
- [PostgreSQL documentation](https://www.postgresql.org/docs/current/) — the official manual, good to read alongside Phase 17.
- [OpenTelemetry documentation](https://opentelemetry.io/docs/) — concepts and guides for all three signals.

---

[🏠 Start](../START-HERE.md) · [← 16.15 Project: a URL shortener service](15-project-url-shortener-service.md) · [📋 Phase 16](README.md) · [Phase 17 — Storage and database internals →](../17-storage-and-databases/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
