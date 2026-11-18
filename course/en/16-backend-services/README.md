<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 15.16 Checkpoint](../15-networking/16-checkpoint.md) · [16.1 What a backend service is →](01-what-a-backend-service-is.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/README.md) <!-- nav -->

# Phase 16 — Backend services

> **Stage 4 — Build real systems** · ⏱ about 4–5 weeks at ~10 hours per week

## What this phase is about

A backend service is a program that runs for a long time on a server, waits for requests over the network, and
answers many clients at the same time: the part behind every app and web page. In this phase you build such services
in Rust, step by step, from a "hello" endpoint to a service with a database, logins, a cache, messages, logs, metrics
and traces, packed into containers.

You start with HTTP APIs: how to design a clean REST API, and how to build it with `axum`, with proper errors,
validation and middleware. Then you learn SQL and PostgreSQL from zero and connect the service to the database with
`sqlx`, whose queries the compiler checks against the real database. Migrations keep the schema under control,
transactions keep the data correct, and authentication lets users log in safely.

The second half adds what real systems need around that core: gRPC with `tonic` for service-to-service calls, Redis
as a cache, NATS for messages, OpenTelemetry for observability, configuration and health checks for containers, and
tests against real services. At the end you put everything together in your own URL shortener service.

## What you will be able to do

- [ ] Explain what a backend service is, why it should be stateless, and which parts a backend system has.
- [ ] Design a REST API with correct methods, status codes and a consistent JSON error format.
- [ ] Build an HTTP service with `axum`: routes, extractors, shared state, validation, middleware and router tests.
- [ ] Write SQL for PostgreSQL and use it from Rust with `sqlx`, including compile-time checked queries and offline mode.
- [ ] Evolve a database schema with migrations and keep multi-step changes correct with transactions.
- [ ] Add registration, login and protected endpoints with Argon2 and session tokens.
- [ ] Use gRPC, Redis and NATS, and make a service observable with logs, metrics and traces.
- [ ] Configure, containerize and test a complete service with its dependencies.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 16.1 | [What a backend service is](01-what-a-backend-service-is.md) | 2.5 h |
| 16.2 | [Designing HTTP APIs: REST](02-designing-http-apis.md) | 105 min |
| 16.3 | [Web services with axum](03-web-services-with-axum.md) | 3 h |
| 16.4 | [Errors, validation and middleware](04-errors-validation-and-middleware.md) | 3 h |
| 16.5 | [SQL and PostgreSQL basics](05-sql-and-postgresql-basics.md) | 2.5 h |
| 16.6 | [Databases with sqlx](06-databases-with-sqlx.md) | 3 h |
| 16.7 | [Migrations and transactions](07-migrations-and-transactions.md) | 2.5 h |
| 16.8 | [Authentication in a web service](08-authentication-in-a-web-service.md) | 3 h |
| 16.9 | [gRPC with tonic](09-grpc-with-tonic.md) | 2.5 h |
| 16.10 | [Caching with Redis](10-caching-with-redis.md) | 120 min |
| 16.11 | [Messaging with NATS](11-messaging-with-nats.md) | 120 min |
| 16.12 | [Logs, metrics and traces with OpenTelemetry](12-logs-metrics-and-traces.md) | 135 min |
| 16.13 | [Configuration, health checks and containers](13-configuration-health-checks-and-containers.md) | 135 min |
| 16.14 | [Testing a backend service](14-testing-a-backend-service.md) | 135 min |
| 16.15 | [Project: a URL shortener service](15-project-url-shortener-service.md) | 12 h |
| 16.16 | [Checkpoint](16-checkpoint.md) | 90 min |

## Projects in this phase

- `code/p16-hello`: your first `axum` service (Lessons 16.1 and 16.2).
- `code/p16-tasks`: a tasks API that grows lesson by lesson: in memory, then PostgreSQL with migrations, transactions
  and logins (Lessons 16.3 to 16.8).
- **Project 16.15 — a URL shortener service:** a REST API with PostgreSQL, a Redis cache, authentication for admin
  endpoints, rate limiting, OpenTelemetry traces, Docker Compose and integration tests.

## How to study this phase

- Keep two terminals open: one runs the service, the other runs `curl`. Read every response, including the headers
  (`curl -i`).
- Docker Desktop must be running from Lesson 16.5 on. Keep the PostgreSQL container running while you work, because
  the `sqlx` macros check your queries against it while you build.
- Follow the lessons in order: `p16-tasks` changes in every lesson, and each lesson starts from the state the previous
  one left. Commit after every lesson, so you can always go back.
- Run `cargo test` and `cargo clippy` before each commit. Most tests in this phase talk to a real database; that is
  slower, and it is the point.
- Never put real passwords or tokens into code, commands or Git. The `dev-password` of this phase is for your laptop
  only, and `.env` belongs in `.gitignore`.

---

[🏠 Start](../START-HERE.md) · [← 15.16 Checkpoint](../15-networking/16-checkpoint.md) · [16.1 What a backend service is →](01-what-a-backend-service-is.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
