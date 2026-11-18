<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.12 Logs, metrics and traces with OpenTelemetry](12-logs-metrics-and-traces.md) · [📋 Phase 16](README.md) · [16.14 Testing a backend service →](14-testing-a-backend-service.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/13-configuration-health-checks-and-containers.md) <!-- nav -->

# Lesson 16.13 — Configuration, health checks and containers

> **Phase 16 — Backend services** · Lesson 13 of 16 · ⏱ about 135 minutes

## What you will learn

- How to read a service's configuration from environment variables into a typed struct, check it at start-up, and keep secrets out of logs.
- The difference between a liveness check and a readiness check, and how to write both in axum.
- How to stop cleanly when Docker sends `SIGTERM`.
- How to package the service in a small, non-root container image with a health check that works without a shell.
- How to start the service and its database together with Docker Compose, in the right order.

## Before you start

- You finished [Lesson 16.12 — Logs, metrics and traces with OpenTelemetry](12-logs-metrics-and-traces.md).
- You know the twelve-factor rules, especially "store config in the environment" ([Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md)).
- You know multi-stage Dockerfiles for Rust ([Lesson 11.13](../11-linux-shell-and-containers/13-dockerfiles-for-rust.md)) and Compose files ([Lesson 11.14](../11-linux-shell-and-containers/14-docker-compose.md)), signals ([Lesson 11.5](../11-linux-shell-and-containers/05-processes-and-signals.md)) and graceful shutdown ([Lesson 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md)).
- You can connect to PostgreSQL with sqlx ([Lesson 16.6](06-databases-with-sqlx.md)).

## New words

| Word | Meaning |
|---|---|
| **configuration** | Settings that change between environments (your laptop, test, production) but not between runs: addresses, ports, passwords. |
| **fail fast** | Stop at start-up with a clear message when something is wrong, instead of failing later in the middle of a request. |
| **redact** | Replace secret text with a placeholder such as `<redacted>` before printing it. |
| **health check** | A small request that a tool sends regularly to ask a service "are you OK?". |
| **liveness** | "The process is alive and can answer." If this fails, the service should be restarted. |
| **readiness** | "The service can do real work now" (its database answers). If this fails, send it no traffic, but do not restart it. |
| **orchestrator** | A program that starts, watches and restarts containers: Docker Compose on one machine, Kubernetes on many (Phase 21). |
| **`SIGTERM`** | The signal "please stop" that `docker stop` and Kubernetes send first. After a grace period (10 s in Docker) they send `SIGKILL`, which cannot be caught. |
| **distroless image** | A base image with only what a program needs to run (C libraries, certificates), and no shell or package manager. |
| **non-root user** | A normal user without administrator rights. Running a container as non-root limits the damage of a break-in. |
| **exit code** | The number a program returns when it ends: `0` means success, anything else means failure. Health-check tools read it. |

## 1. Configuration from the environment

A service needs settings: on which address to listen, where the database is, which password to use. The twelve-factor rule (Lesson 08.15) says: read them from **environment variables**, so the same image runs on your laptop and in production with different values.

Three rules make configuration safe:

1. **Read everything once, at start-up, into one typed struct.** The rest of the code uses `config.port`, never `std::env::var("APP_PORT")` in the middle of a handler.
2. **Fail fast.** A missing or broken value stops the program at once, with a message that names the variable.
3. **Never print secrets.** The database URL (Uniform Resource Locator, the address) contains a password.

Create the project:

```bash
cd ~/projects/rust/code
cargo new p16-config-health
cd p16-config-health
cargo add axum@0.8 tracing@0.1 tracing-subscriber@0.3
cargo add sqlx@0.9 --features postgres,runtime-tokio
cargo add tokio@1 --features macros,rt-multi-thread,signal,time,net
```

Create `src/config.rs`:

```rust
use std::fmt;
use std::net::{IpAddr, SocketAddr};

/// All settings of the service, read once at start-up.
#[derive(Clone)]
pub struct Config {
    pub host: IpAddr,
    pub port: u16,
    pub database_url: String,
}

#[derive(Debug, PartialEq)]
pub enum ConfigError {
    Missing(&'static str),
    Invalid { key: &'static str, value: String },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Missing(key) => write!(f, "environment variable {key} is not set"),
            ConfigError::Invalid { key, value } => write!(f, "{key} has an invalid value: {value:?}"),
        }
    }
}

impl std::error::Error for ConfigError {}

impl Config {
    /// Reads the real environment.
    pub fn from_env() -> Result<Self, ConfigError> {
        Self::from_lookup(|key| std::env::var(key).ok())
    }

    /// Reads settings through `get`, so tests can pass their own values.
    pub fn from_lookup(get: impl Fn(&str) -> Option<String>) -> Result<Self, ConfigError> {
        let host = parse_or(&get, "APP_HOST", IpAddr::from([127, 0, 0, 1]))?;
        let port = parse_or(&get, "APP_PORT", 8080)?;
        let database_url = get("DATABASE_URL").ok_or(ConfigError::Missing("DATABASE_URL"))?;
        Ok(Config {
            host,
            port,
            database_url,
        })
    }

    pub fn listen_addr(&self) -> SocketAddr {
        SocketAddr::new(self.host, self.port)
    }
}

/// Parses an optional variable, or returns the default when it is not set.
fn parse_or<T: std::str::FromStr>(
    get: &impl Fn(&str) -> Option<String>,
    key: &'static str,
    default: T,
) -> Result<T, ConfigError> {
    match get(key) {
        None => Ok(default),
        Some(value) => value.parse().map_err(|_| ConfigError::Invalid { key, value }),
    }
}

/// Never print the database URL: it contains the password.
impl fmt::Debug for Config {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Config")
            .field("host", &self.host)
            .field("port", &self.port)
            .field("database_url", &"<redacted>")
            .finish()
    }
}
```

**What happens:**

1. `Config` holds typed values: an `IpAddr`, a `u16`, a `String`. Once it exists, every value is valid; no other code has to parse or check them again (type-driven design, Lesson 08.5).
2. `from_lookup` takes a **function** that returns the value of a variable. `from_env` passes one that reads the real environment. Tests pass their own (Section "Try it yourself"). This is dependency injection with a closure (Lesson 08.9).
3. `parse_or` uses the default when a variable is missing, and returns `ConfigError::Invalid` when it is present but cannot be parsed. Optional settings get sensible defaults for development; the database URL has no default, because a wrong default database is worse than an error.
4. We write `Debug` by hand instead of deriving it, so `{config:?}` prints `<redacted>` instead of the password.

> 🔒 **Security:** In production, secrets usually come from a secret manager or from files that the orchestrator mounts (Docker and Kubernetes "secrets", Lesson 10.8), not from plain variables in a file in Git. Exercise 2 adds reading the URL from a file.

## 2. Liveness and readiness

An orchestrator (Docker, Kubernetes) watches your service with **health checks**. In Lesson 16.1 the service had one `GET /health`; here we split it in two, because there are two different questions, and mixing them up causes outages:

| Check | Question | When it fails, the orchestrator… | Our endpoint |
|---|---|---|---|
| **Liveness** | Is the process alive and able to answer? | restarts the container | `GET /healthz` |
| **Readiness** | Can it do real work right now? | stops sending it traffic, waits | `GET /readyz` |

Why not one check? Imagine the database is down for a minute. If liveness checked the database, the orchestrator would restart **every** copy of your service, again and again, which does not help the database at all. The right reaction is readiness: "not ready, send traffic elsewhere", while the process stays up and recovers by itself when the database is back.

Now `src/main.rs`:

```rust
mod config;

use std::io::{Read, Write};
use std::net::{SocketAddr, TcpStream};
use std::process::ExitCode;
use std::time::Duration;

use axum::{Router, extract::State, http::StatusCode, routing::get};
use sqlx::PgPool;
use sqlx::postgres::PgPoolOptions;
use tokio::signal::unix::{SignalKind, signal};
use tracing::{error, info, warn};

use config::Config;

#[derive(Clone)]
struct AppState {
    db: PgPool,
}

#[tokio::main]
async fn main() -> ExitCode {
    tracing_subscriber::fmt().with_ansi(false).without_time().init();

    let config = match Config::from_env() {
        Ok(config) => config,
        Err(err) => {
            error!("invalid configuration: {err}");
            return ExitCode::FAILURE;
        }
    };

    if std::env::args().nth(1).as_deref() == Some("healthcheck") {
        return healthcheck(config.port);
    }

    info!(?config, "configuration loaded");
    match run(config).await {
        Ok(()) => ExitCode::SUCCESS,
        Err(err) => {
            error!("server failed: {err}");
            ExitCode::FAILURE
        }
    }
}

async fn run(config: Config) -> Result<(), Box<dyn std::error::Error>> {
    // `connect_lazy` does not connect yet: the service starts even if the database is down.
    let db = PgPoolOptions::new()
        .acquire_timeout(Duration::from_secs(2))
        .connect_lazy(&config.database_url)?;

    let app = Router::new()
        .route("/healthz", get(healthz))
        .route("/readyz", get(readyz))
        .with_state(AppState { db });

    let listener = tokio::net::TcpListener::bind(config.listen_addr()).await?;
    info!("listening on {}", listener.local_addr()?);
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;
    info!("stopped cleanly");
    Ok(())
}

/// Liveness: the process is running and can answer HTTP.
async fn healthz() -> &'static str {
    "ok"
}

/// Readiness: the service can do real work, so its database must answer.
async fn readyz(State(state): State<AppState>) -> (StatusCode, &'static str) {
    match sqlx::query("SELECT 1").execute(&state.db).await {
        Ok(_) => (StatusCode::OK, "ready"),
        Err(err) => {
            warn!("not ready: {err}");
            (StatusCode::SERVICE_UNAVAILABLE, "database unavailable")
        }
    }
}

/// Waits for Ctrl+C (SIGINT) or SIGTERM, which `docker stop` sends.
async fn shutdown_signal() {
    let mut terminate = signal(SignalKind::terminate()).expect("cannot listen for SIGTERM");
    tokio::select! {
        _ = tokio::signal::ctrl_c() => {}
        _ = terminate.recv() => {}
    }
    info!("shutdown signal received");
}

/// `p16-config-health healthcheck`: exit code 0 if /healthz answers 200.
fn healthcheck(port: u16) -> ExitCode {
    let check = || -> std::io::Result<bool> {
        let addr = SocketAddr::from(([127, 0, 0, 1], port));
        let mut stream = TcpStream::connect_timeout(&addr, Duration::from_secs(2))?;
        stream.set_read_timeout(Some(Duration::from_secs(2)))?;
        stream.write_all(b"GET /healthz HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n")?;
        let mut response = String::new();
        stream.read_to_string(&mut response)?;
        Ok(response.starts_with("HTTP/1.1 200"))
    };
    match check() {
        Ok(true) => ExitCode::SUCCESS,
        _ => ExitCode::FAILURE,
    }
}
```

**What happens:**

1. `main` returns `ExitCode`, so the program can end with code 1 on a configuration error. Configuration is loaded **first**: nothing else starts when it is wrong.
2. `info!(?config, ...)` logs the configuration with its `Debug` form, which hides the URL.
3. `connect_lazy` creates the pool **without connecting**. Connections are opened on first use. So the process starts and answers `/healthz` even while the database is still starting, and `/readyz` tells the truth. `acquire_timeout(2 s)` makes a readiness check fail after 2 seconds instead of hanging for the default 30.
4. `/readyz` runs `SELECT 1`, the cheapest possible query, and answers `503 Service Unavailable` if the database does not answer.
5. `shutdown_signal` waits for either Ctrl+C or `SIGTERM`. `with_graceful_shutdown` then stops accepting new connections, lets running requests finish, and `run` returns.
6. `healthcheck` is a second mode of the same program, explained in Section 4: a tiny HTTP (Hypertext Transfer Protocol) client written with the standard library (you wrote one in Lesson 15.8), which turns the answer into an exit code.

Try it on your Mac. Without `DATABASE_URL`:

```bash
cargo run
```

```text
ERROR p16_config_health: invalid configuration: environment variable DATABASE_URL is not set
```

With a URL to a port where no database runs (yet):

```bash
DATABASE_URL=postgres://app:secret@127.0.0.1:5999/app cargo run
```

```text
 INFO p16_config_health: configuration loaded config=Config { host: 127.0.0.1, port: 8080, database_url: "<redacted>" }
 INFO p16_config_health: listening on 127.0.0.1:8080
```

In a second terminal:

```bash
curl -i http://127.0.0.1:8080/healthz
curl -i http://127.0.0.1:8080/readyz
```

```text
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 2
date: Wed, 18 Nov 2026 07:35:41 GMT

ok
HTTP/1.1 503 Service Unavailable
content-type: text/plain; charset=utf-8
content-length: 20
date: Wed, 18 Nov 2026 07:35:43 GMT

database unavailable
```

Alive, but not ready: exactly right. The service logged `WARN p16_config_health: not ready: pool timed out while waiting for an open connection`. Press Ctrl+C; the log ends with:

```text
 INFO p16_config_health: shutdown signal received
 INFO p16_config_health: stopped cleanly
```

## 3. Why `SIGTERM` matters

`docker stop` (and Kubernetes, when it moves a container) sends `SIGTERM` first. A process that ignores it keeps running until the grace period ends, 10 seconds in Docker, and then gets `SIGKILL`: it dies immediately, in the middle of requests, without flushing logs or telemetry (Lesson 16.12). Ctrl+C sends a different signal, `SIGINT`, so a program that only waits for `ctrl_c()` works on your laptop and dies hard in production.

Our `shutdown_signal` listens for both. With it, `docker compose stop app` in Section 5 finished in 0.16 seconds, and the last two log lines above were written.

> ⚠️ **Common mistake:** Holding work in memory that is lost on shutdown, for example messages that were received from NATS but not yet processed. A graceful shutdown should first stop taking new work, then finish or hand back the work it has (for JetStream: simply do not ack, Lesson 16.11).

## 4. A small, safe container image

Create `Dockerfile` next to `Cargo.toml`:

```dockerfile
# ---- stage 1: build the program ----
FROM rust:1.100 AS build
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release --locked

# ---- stage 2: a small image that only runs it ----
# "nonroot" images run as an unprivileged user (uid 65532), not as root.
FROM gcr.io/distroless/cc-debian13:nonroot
COPY --from=build /src/target/release/p16-config-health /app/p16-config-health
ENV APP_HOST=0.0.0.0 APP_PORT=8080
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/app/p16-config-health", "healthcheck"]
ENTRYPOINT ["/app/p16-config-health"]
```

And `.dockerignore`, so the big `target/` folder is not sent to Docker:

```text
target
```

**What happens:**

1. **Stage 1** uses the official Rust image to compile. `--locked` makes Cargo use exactly the versions in `Cargo.lock` and fail if the file is out of date: the image is built from the same dependencies you tested.
2. **Stage 2** starts from `gcr.io/distroless/cc-debian13:nonroot`, published by Google's distroless project. It contains the C libraries a Rust program needs (`cc`), certificates and time-zone data, and **no shell, no package manager, no `curl`**. Fewer programs in the image means fewer things an attacker can use. The `:nonroot` tag runs the program as user 65532 instead of root.
3. Only the compiled binary is copied from stage 1; the compiler and the source code stay behind. The final image here was 59.2 MB (megabytes).
4. `APP_HOST=0.0.0.0` makes the service listen on all interfaces **inside** the container, so Docker can forward the port to it. On your Mac the default `127.0.0.1` is the safer choice; in a container it would make the service unreachable (see Common mistakes).
5. `HEALTHCHECK` tells Docker to run `/app/p16-config-health healthcheck` every 10 seconds. There is no `curl` in the image, so the program checks itself: the exit code `0` means healthy. After 3 failures in a row, the container is marked `unhealthy`. The *exec form* `["...", "..."]` runs the program directly; a plain string would need `/bin/sh`, which this image does not have.

> ⚡ **Performance:** Every change to `src/` makes this Dockerfile compile all dependencies again. Exercise 3 shows how to cache them in their own layer; one test rebuild went from several minutes to 19 seconds.

## 5. Service and database with Compose

Create `compose.yaml`:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev-only-password
      POSTGRES_DB: app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10

  app:
    build: .
    environment:
      DATABASE_URL: postgres://app:dev-only-password@db:5432/app
    ports:
      - "127.0.0.1:8080:8080"
    depends_on:
      db:
        condition: service_healthy
```

**What happens:**

1. `db` runs the official PostgreSQL 18 image. Its own `healthcheck` uses `pg_isready`, a tool inside that image that asks the server if it accepts connections. (This image has a shell, so `CMD-SHELL` works.)
2. `app` is built from our `Dockerfile`. Inside the Compose network, the database's host name is simply `db`.
3. `depends_on` with `condition: service_healthy` makes Compose start `app` only **after** `db` reports healthy. Plain `depends_on` only waits for the container to *start*, not for PostgreSQL to be ready.
4. Only port 8080 of `app` is published, and only on `127.0.0.1`. The database is reachable only from inside the network.

Build and start everything:

```bash
docker compose up -d --build
docker compose ps --format 'table {{.Service}}\t{{.Status}}'
```

```text
SERVICE   STATUS
app       Up Less than a second (health: starting)
db        Up 5 seconds (healthy)
```

`db` was healthy before `app` even started. A few seconds later:

```text
SERVICE   STATUS
app       Up 15 seconds (healthy)
db        Up 20 seconds (healthy)
```

Now test readiness by stopping and starting the database:

```bash
curl -w ' %{http_code}\n' http://127.0.0.1:8080/readyz
docker compose stop db
curl -w ' %{http_code}\n' http://127.0.0.1:8080/healthz
curl -w ' %{http_code}\n' http://127.0.0.1:8080/readyz
docker compose start db
```

```text
ready 200
ok 200
database unavailable 503
```

The service stayed alive (and Docker kept it `healthy`, because the health check uses `/healthz`), only readiness dropped. A few seconds after `docker compose start db`, `/readyz` answers `ready 200` again, with no restart. Stop everything with `docker compose down`.

> 🔒 **Security:** `dev-only-password` in `compose.yaml` is acceptable for a local development database that is not reachable from outside. Never commit real passwords. For shared environments, put them in an `.env` file that is listed in `.gitignore`, or use Compose secrets (Lesson 10.8).

## Common mistakes

> ⚠️ **Common mistake:** Listening on `127.0.0.1` inside a container. The port is published, but nothing answers from outside: `127.0.0.1` inside the container is the container itself. Running the image with `-e APP_HOST=127.0.0.1` gives:

```text
curl: (52) Empty reply from server
```

Fix: listen on `0.0.0.0` in containers (our `Dockerfile` sets it).

> ⚠️ **Common mistake:** A shell-form health check in a distroless image, such as `--health-cmd 'curl -f http://localhost:8080/healthz'`. Docker reports:

```text
exec: "/bin/sh": stat /bin/sh: no such file or directory
```

Fix: use the exec form with a program that exists in the image, like our `healthcheck` mode.

> ⚠️ **Common mistake:** Checking the database in the liveness endpoint. A short database outage then restarts all your service containers in a loop. Keep `/healthz` about the process only.

> ⚠️ **Common mistake:** The container exits at once with code 1. Run `docker compose logs app` and read the first line: a configuration error says exactly which variable is wrong, for example `invalid configuration: environment variable DATABASE_URL is not set`.

## Try it yourself

1. Add the unit tests below at the end of `src/config.rs` and run `cargo test`. They never touch the real environment: `std::env::set_var` is `unsafe` in edition 2024, because changing the environment while other threads read it is a data race, and tests run in parallel threads.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashMap;

    fn lookup(pairs: &[(&str, &str)]) -> impl Fn(&str) -> Option<String> {
        let map: HashMap<String, String> = pairs
            .iter()
            .map(|(k, v)| (k.to_string(), v.to_string()))
            .collect();
        move |key| map.get(key).cloned()
    }

    #[test]
    fn defaults_are_used_when_optional_values_are_missing() {
        let config = Config::from_lookup(lookup(&[("DATABASE_URL", "postgres://db/app")])).unwrap();
        assert_eq!(config.listen_addr().to_string(), "127.0.0.1:8080");
    }

    #[test]
    fn missing_database_url_is_an_error() {
        let err = Config::from_lookup(lookup(&[])).unwrap_err();
        assert_eq!(err, ConfigError::Missing("DATABASE_URL"));
    }

    #[test]
    fn debug_output_hides_the_database_url() {
        let config =
            Config::from_lookup(lookup(&[("DATABASE_URL", "postgres://u:secret@db/app")])).unwrap();
        assert!(!format!("{config:?}").contains("secret"));
    }
}
```

```text
test config::tests::defaults_are_used_when_optional_values_are_missing ... ok
test config::tests::missing_database_url_is_an_error ... ok
test config::tests::debug_output_hides_the_database_url ... ok
```

2. Run `APP_PORT=80a DATABASE_URL=x cargo run`. You get `ERROR p16_config_health: invalid configuration: APP_PORT has an invalid value: "80a"`.
3. While the service runs on your Mac, run `DATABASE_URL=x cargo run -- healthcheck; echo $?`. It prints `0`. Stop the service and run it again: `1`.

## Exercises

### Exercise 1 — A test for a bad port (easy)

Add a test `port_that_is_not_a_number_is_invalid`: with `APP_PORT=80a` the result must be `ConfigError::Invalid { key: "APP_PORT", value: "80a" }`.

<details>
<summary>Hint</summary>

Use the `lookup` helper with two pairs: a database URL (so that only the port can fail) and the bad port.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn port_that_is_not_a_number_is_invalid() {
        let err = Config::from_lookup(lookup(&[
            ("DATABASE_URL", "postgres://db/app"),
            ("APP_PORT", "80a"),
        ]))
        .unwrap_err();
        assert_eq!(
            err,
            ConfigError::Invalid {
                key: "APP_PORT",
                value: "80a".to_string()
            }
        );
    }
```

Why it works: `parse_or` finds the variable, `"80a".parse::<u16>()` fails, and `map_err` builds the `Invalid` variant with the key and the original text. Because `ConfigError` derives `PartialEq`, the test compares whole values.

</details>

### Exercise 2 — Read the secret from a file (medium)

Orchestrators often give secrets as files, for example `/run/secrets/database_url`. Support `DATABASE_URL_FILE`: if it is set, read the URL from that file (without the trailing newline); otherwise use `DATABASE_URL`. Write a test that writes a temporary file and checks the result.

<details>
<summary>Hint</summary>

`std::fs::read_to_string(&path)` reads the file; `.trim()` removes the newline. If the file cannot be read, return `ConfigError::Invalid` with the key `DATABASE_URL_FILE`. In the test, build a path with `std::env::temp_dir().join(...)` and delete the file at the end.

</details>

<details>
<summary>Answer</summary>

In `from_lookup`, replace the `database_url` line:

```rust
        let database_url = match get("DATABASE_URL_FILE") {
            Some(path) => std::fs::read_to_string(&path)
                .map(|text| text.trim().to_string())
                .map_err(|_| ConfigError::Invalid {
                    key: "DATABASE_URL_FILE",
                    value: path,
                })?,
            None => get("DATABASE_URL").ok_or(ConfigError::Missing("DATABASE_URL"))?,
        };
```

The test:

```rust
    #[test]
    fn database_url_can_come_from_a_file() {
        let path = std::env::temp_dir().join("p16-config-test-database-url");
        std::fs::write(&path, "postgres://u:p@db/app\n").unwrap();
        let config =
            Config::from_lookup(lookup(&[("DATABASE_URL_FILE", path.to_str().unwrap())])).unwrap();
        assert_eq!(config.database_url, "postgres://u:p@db/app");
        std::fs::remove_file(path).unwrap();
    }
```

Why it works: the file wins when both are set, so the secret never has to appear in the process environment, where tools like `docker inspect` can show it. Editors and `echo` add a newline at the end of a file; `trim` removes it, otherwise the connection would fail with a confusing error.

</details>

### Exercise 3 — Cache the dependencies in Docker (hard)

Change the build stage so that the dependencies are compiled in their own layer, which Docker reuses as long as `Cargo.toml` and `Cargo.lock` do not change. Build twice, with a small change in `src/main.rs` between the builds, and compare the times.

<details>
<summary>Hint</summary>

Copy only `Cargo.toml` and `Cargo.lock`, create a fake `src/main.rs` with `fn main() {}`, build, and delete it. Then copy the real `src`. Cargo decides what to rebuild by file times, so `touch` the real `main.rs` before the second build.

</details>

<details>
<summary>Answer</summary>

```dockerfile
# ---- stage 1: build the program ----
FROM rust:1.100 AS build
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
# Build only the dependencies, with an empty main, so this layer is cached.
RUN mkdir src && echo 'fn main() {}' > src/main.rs \
    && cargo build --release --locked \
    && rm -rf src
COPY src ./src
# `touch` makes Cargo notice that main.rs is newer than the cached build.
RUN touch src/main.rs && cargo build --release --locked
```

The second stage stays the same. In one test on this course's machine, the first build took 120 seconds and the build after a change in `src/main.rs` took 19 seconds; with the original Dockerfile the same rebuild compiled everything again and took several minutes.

Why it works: Docker caches each instruction's result and reuses it while the instruction and its input files are unchanged. The expensive `cargo build` of all dependencies now depends only on `Cargo.toml` and `Cargo.lock`. The tool `cargo-chef` automates this pattern for workspaces with many crates.

</details>

## Quiz

1. Why does the database URL have no default value, while the port does?

<details>
<summary>Answer</summary>

A default port is harmless for development. A default database could silently connect the service to the wrong database; a clear start-up error is safer.

</details>

2. The database is down for two minutes. What should `/healthz` and `/readyz` answer, and why?

<details>
<summary>Answer</summary>

`/healthz`: 200, the process is alive, so it must not be restarted. `/readyz`: 503, so no traffic is sent to it until the database answers again.

</details>

3. Why does the service listen for `SIGTERM` and not only for Ctrl+C?

<details>
<summary>Answer</summary>

`docker stop` and Kubernetes send `SIGTERM`; Ctrl+C sends `SIGINT`. Without handling `SIGTERM`, the service is killed with `SIGKILL` after the grace period, in the middle of requests and without flushing logs and telemetry.

</details>

4. Why can't the `HEALTHCHECK` in our image use `curl`?

<details>
<summary>Answer</summary>

The distroless image has no `curl` and no shell. The check must run a program that exists in the image; here the service binary itself, in `healthcheck` mode.

</details>

5. What is the difference between `depends_on: [db]` and `depends_on` with `condition: service_healthy`?

<details>
<summary>Answer</summary>

The first waits only until the `db` container has started. The second waits until its health check passes, that is, until PostgreSQL really accepts connections.

</details>

## Summary

- Read configuration once at start-up into a typed struct; fail fast with a message that names the variable; give defaults only where a default is safe.
- Hide secrets in `Debug` output and logs; prefer secret files over plain variables in production.
- Liveness (`/healthz`) asks "is the process alive?", readiness (`/readyz`) asks "can it work now?". Never check dependencies in liveness.
- `connect_lazy` lets the service start before its database; a short `acquire_timeout` keeps readiness checks fast.
- Handle `SIGTERM` for a graceful shutdown in containers.
- A multi-stage build into a distroless `nonroot` image gives a small image without a shell; the program can be its own health check.
- Compose's `condition: service_healthy` starts services in the right order.

## Sources and further reading

- [The Twelve-Factor App — III. Config](https://12factor.net/config) — why configuration belongs in the environment.
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) — the section HEALTHCHECK lists every option of health checks; exec form versus shell form is explained there too.
- [Docker Compose — Control startup order](https://docs.docker.com/compose/how-tos/startup-order/) — `depends_on` with `service_healthy`.
- [Kubernetes — Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) — the same two ideas in Kubernetes (Phase 21).
- [GoogleContainerTools/distroless on GitHub](https://github.com/GoogleContainerTools/distroless) — what the distroless images contain and which tags exist.
- [sqlx — PgPoolOptions on docs.rs](https://docs.rs/sqlx/latest/sqlx/postgres/type.PgPoolOptions.html) — `connect_lazy`, `acquire_timeout` and other pool settings.

---

[🏠 Start](../START-HERE.md) · [← 16.12 Logs, metrics and traces with OpenTelemetry](12-logs-metrics-and-traces.md) · [📋 Phase 16](README.md) · [16.14 Testing a backend service →](14-testing-a-backend-service.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
