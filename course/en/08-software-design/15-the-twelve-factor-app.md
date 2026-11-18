<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.14 Layers and hexagonal architecture](14-layers-and-hexagonal-architecture.md) · [📋 Phase 08](README.md) · [08.16 Project: refactor a messy program →](16-project-refactor-a-messy-program.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/15-the-twelve-factor-app.md) <!-- nav -->

# Lesson 08.15 — The twelve-factor app

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 15 of 17 · ⏱ about 120 minutes

## What you will learn

- What the twelve-factor methodology is, who it is for, and its twelve rules in plain words.
- How to read configuration from the environment in Rust, with types, defaults, clear errors and tests.
- Why a service logs to standard output, keeps no important state in memory, binds its own port and must be ready to be stopped at any moment.
- How to keep secrets out of your code and out of your logs.
- Where the twelve factors are too strict or out of date.

## Before you start

- You finished [Lesson 08.14 — Layers and hexagonal architecture](14-layers-and-hexagonal-architecture.md).
- You know environment variables, standard output and error, and exit codes from the terminal ([Lesson 00.4](../00-getting-started/04-terminal-basics.md)), `Result` and custom errors ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)), and `Cargo.lock` ([Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
- You have `curl` in your terminal (it is part of macOS). You will use it only against a program running on your own computer.
- Create the package: `cargo new code/p08-twelve`.

## New words

| Word | Meaning |
|---|---|
| **twelve-factor app** | A set of twelve rules for building programs that run as services and are easy to deploy, scale and maintain. |
| **service** | A program that runs for a long time and answers requests from other programs, such as a web server. |
| **deploy** | One running copy of the program in one place: your laptop, a test server, production. |
| **environment variable** | A named text value that the operating system gives to a program when it starts, such as `PORT=8080`. |
| **backing service** | A service that your program uses over the network: a database, a message queue, an email server. |
| **stateless** | Keeping no important data between requests inside the process; anything that must survive is stored in a backing service. |
| **port binding** | The program opens a network port by itself and listens on it, instead of being plugged into another server. |
| **HTTP** (HyperText Transfer Protocol) | The protocol that web browsers and web services use to talk to each other. |
| **TCP** (Transmission Control Protocol) | The reliable, ordered connection on which web traffic (HTTP) runs. |
| **signal** (`SIGTERM`) | A small message from the operating system to a process. `SIGTERM` means "please stop"; `SIGKILL` stops the process at once. |
| **graceful shutdown** | Stopping in an orderly way: finish the current requests, close connections, then exit. |
| **release** | A built program together with its configuration for one deploy. |
| **parity** (dev/prod parity) | Keeping the development, test and production environments as similar as possible. |

## 1. What it is, and who it is for

The **twelve-factor app** is a list of twelve rules, written around 2011 by engineers who ran many web programs on a hosting platform. They had seen what makes a program easy or painful to run, and wrote down
what the easy ones have in common. The text is free at 12factor.net. It is not a law and not a Rust topic. It is a checklist of habits for **services**: programs that run for a long time, serve requests and are
started, stopped and copied by other tools (containers, process managers; Phase 11).

A command-line tool that you run by hand does not need all twelve rules, but half of them (configuration, logs, dependencies, exit codes) are good habits there too.

## 2. The twelve factors

| # | Name | The rule, in one sentence | In a Rust service |
|---|---|---|---|
| I | **Codebase** | One codebase in version control, many deploys. | One Git repository per app; shared code becomes a library crate (Lesson 08.12). The same commit runs on your laptop and in production. |
| II | **Dependencies** | Declare every dependency; never rely on something being installed. | `Cargo.toml` and a committed `Cargo.lock`; build with `--locked`. |
| III | **Config** | Store configuration in the environment. | `std::env::var`, read once at start-up into a typed `Config`. |
| IV | **Backing services** | Treat databases, queues and mail servers as attached resources, swapped by changing a setting. | A URL (Uniform Resource Locator, an address) in config (`DATABASE_URL`), and a port (trait) in the code (Lesson 08.14). |
| V | **Build, release, run** | Keep the stages apart: build once, combine with config into a release, run it. | `cargo build --release --locked` makes one binary; config comes from outside, not from editing the binary. |
| VI | **Processes** | Run as stateless, share-nothing processes. | No important data in `static` variables or process memory; store it in a database. |
| VII | **Port binding** | Export services by binding a port. | `TcpListener::bind` in your own program. |
| VIII | **Concurrency** | Scale out by running more processes. | Run several copies; threads and `async` work inside one process (Phase 14). |
| IX | **Disposability** | Start fast, stop gracefully; be ready to be killed at any time. | Fast start-up, handle `SIGTERM`, make work restartable. |
| X | **Dev/prod parity** | Keep development, staging and production as similar as possible. | Same kind of database everywhere; containers help (Phase 11). |
| XI | **Logs** | Treat logs as a stream of events written to standard output. | `println!`/`tracing` to stdout; the platform collects them. |
| XII | **Admin processes** | Run one-off admin tasks with the same code and config. | A second binary in `src/bin/`, run with `cargo run --bin migrate`. |

The factors that you can see and test inside a small Rust program are III, VI, VII, IX and XI. You will build a tiny web service that demonstrates them.

## 3. The demo service

The service answers every HTTP request with a greeting. It is small on purpose, so that the habits stay visible. The package is a library (`src/lib.rs`) plus a thin `main.rs` (Lesson 08.12). In `src/lib.rs`:

```rust
//! A tiny web service that follows the twelve-factor ideas.

pub mod config;
pub mod server;
```

### 3.1 Factor III: configuration from the environment

Configuration is everything that **changes between deploys**: the port, addresses of backing services, passwords. It must not live in the code (a constant) or in a file committed next to the code.
The environment is the common denominator: every operating system, container tool and hosting platform can set environment variables. `src/config.rs`:

```rust
//! Factor III: configuration comes from the environment.

use std::fmt;

/// All settings of the service.
#[derive(Clone, PartialEq)]
pub struct Config {
    /// The TCP port to listen on.
    pub port: u16,
    /// The text of the greeting.
    pub greeting: String,
    /// Stop after this many requests (useful for demos and tests).
    pub max_requests: Option<u32>,
    api_key: Option<String>,
}

/// A setting is wrong.
#[derive(Debug, PartialEq)]
pub struct ConfigError {
    /// The name of the environment variable.
    pub name: &'static str,
    /// What is wrong with its value.
    pub reason: &'static str,
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}: {}", self.name, self.reason)
    }
}

impl std::error::Error for ConfigError {}

impl Config {
    /// Reads the settings through `get`, which looks up a variable by name.
    ///
    /// Taking a function instead of calling `std::env::var` makes this testable.
    pub fn from_vars(get: impl Fn(&str) -> Option<String>) -> Result<Config, ConfigError> {
        let port = match get("PORT") {
            None => 8080,
            Some(text) => match text.parse::<u16>() {
                Ok(port) => port,
                Err(_) => {
                    return Err(ConfigError {
                        name: "PORT",
                        reason: "must be a number from 0 to 65535",
                    });
                }
            },
        };
        let max_requests = match get("APP_MAX_REQUESTS") {
            None => None,
            Some(text) => Some(text.parse::<u32>().map_err(|_| ConfigError {
                name: "APP_MAX_REQUESTS",
                reason: "must be a whole number",
            })?),
        };
        Ok(Config {
            port,
            greeting: get("APP_GREETING").unwrap_or_else(|| String::from("Hello")),
            max_requests,
            api_key: get("APP_API_KEY"),
        })
    }

    /// Reads the settings from the real environment.
    pub fn from_env() -> Result<Config, ConfigError> {
        Config::from_vars(|name| std::env::var(name).ok())
    }

    /// Is an API key configured? (The key itself is never printed.)
    pub fn has_api_key(&self) -> bool {
        self.api_key.is_some()
    }
}

impl fmt::Debug for Config {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Config")
            .field("port", &self.port)
            .field("greeting", &self.greeting)
            .field("max_requests", &self.max_requests)
            .field("api_key", &self.api_key.as_ref().map(|_| "***"))
            .finish()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashMap;

    fn config_from(pairs: &[(&str, &str)]) -> Result<Config, ConfigError> {
        let vars: HashMap<String, String> = pairs
            .iter()
            .map(|(name, value)| ((*name).to_string(), (*value).to_string()))
            .collect();
        Config::from_vars(|name| vars.get(name).cloned())
    }

    #[test]
    fn defaults_apply_when_nothing_is_set() {
        let config = config_from(&[]).unwrap();
        assert_eq!(config.port, 8080);
        assert_eq!(config.greeting, "Hello");
        assert_eq!(config.max_requests, None);
        assert!(!config.has_api_key());
    }

    #[test]
    fn values_come_from_the_environment() {
        let config = config_from(&[("PORT", "9000"), ("APP_GREETING", "Salom")]).unwrap();
        assert_eq!(config.port, 9000);
        assert_eq!(config.greeting, "Salom");
    }

    #[test]
    fn a_bad_port_names_the_variable() {
        let error = config_from(&[("PORT", "http")]).unwrap_err();
        assert_eq!(error.name, "PORT");
        assert_eq!(error.to_string(), "PORT: must be a number from 0 to 65535");
    }

    #[test]
    fn debug_output_hides_the_secret() {
        let config = config_from(&[("APP_API_KEY", "super-secret-value")]).unwrap();
        let text = format!("{config:?}");
        assert!(!text.contains("super-secret-value"));
        assert!(text.contains("***"));
    }
}
```

**What happens:**

1. `Config::from_vars` reads the settings through a function `get` that looks up a name. In production `get` is `std::env::var`; in tests it is a `HashMap`. Because the code does not call `std::env` directly, tests do not
   touch the real environment, and they can run in parallel without interfering (tests share one process, and so one environment).
2. Each setting has a **type** (`u16`, `Option<u32>`) and a **default** (port 8080, greeting "Hello"). A wrong value is a `ConfigError` that names the variable and says what is wrong, instead of a panic or a silent fallback.
3. `from_env` is the only place that touches the real environment. It runs once at start-up, so a configuration error stops the program immediately, with a clear message, instead of failing in the middle of a request.
4. `api_key` is a *secret*. The field is private and the hand-written `Debug` prints `***` instead of its value, so a log line that prints the whole `Config` cannot leak it. A test checks that.

> 🔒 **Security:** Never print secrets: not in logs, not in error messages, not in `Debug` output. Never commit them to Git (add local files such as `.env` to `.gitignore`). Environment variables are practical but not perfect for secrets:
> the process and its child programs can read them, and some tools show them in crash reports. Larger systems use a secrets manager or files with strict permissions. Lesson 10.8 teaches secrets management.

### 3.2 Factors VI, VII, IX, XI: the server

`src/server.rs`:

```rust
//! Factors VI, VII, IX and XI: a stateless process that binds a port and logs to stdout.

use std::io::{self, BufRead, BufReader, Write};
use std::net::{TcpListener, TcpStream};

use crate::config::Config;

/// Writes one event as one line on standard output. The environment decides where it goes.
pub fn log(level: &str, message: &str) {
    println!("{level} {message}");
}

fn handle(stream: TcpStream, greeting: &str) -> io::Result<()> {
    let mut reader = BufReader::new(stream.try_clone()?);
    let mut request_line = String::new();
    reader.read_line(&mut request_line)?;
    // Read and ignore the headers: they end with an empty line.
    loop {
        let mut header = String::new();
        if reader.read_line(&mut header)? == 0 || header.trim().is_empty() {
            break;
        }
    }

    let body = format!("{greeting} from the twelve-factor demo\n");
    let mut stream = stream;
    write!(
        stream,
        "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{body}",
        body.len()
    )?;
    log("info", &format!("request: {}", request_line.trim()));
    Ok(())
}

/// Listens on the configured port and answers requests until `max_requests` is reached.
pub fn serve(config: &Config) -> io::Result<()> {
    let listener = TcpListener::bind(("127.0.0.1", config.port))?;
    log("info", &format!("listening on {}", listener.local_addr()?));

    let mut served = 0;
    for stream in listener.incoming() {
        handle(stream?, &config.greeting)?;
        served += 1;
        if config.max_requests == Some(served) {
            break;
        }
    }
    log("info", "stopping");
    Ok(())
}
```

**What happens:**

1. `serve` binds the port **itself** with `TcpListener::bind` (factor VII). No other web server is needed; the program *is* the service. The address is fixed at `127.0.0.1` (only your own computer can connect; Exercise 3 makes it a setting) and `config.port` comes from the environment.
2. `log` writes one event per line to standard output with `println!` (factor XI). The program does not open log files, rotate them or send them anywhere; that is the job of the environment (a terminal, a file redirect, a container platform).
   A real service uses a logging library (Lesson 09.10) that adds timestamps and structure, but it still writes to standard output.
3. `handle` keeps nothing between requests (factor VI): the answer depends only on the request and the configuration. Because the process holds no state, you can run ten copies behind a load balancer, or restart one at any moment, and nothing is lost.
4. The loop stops after `max_requests` answers. A real service would run forever; the limit makes the demo and tests finish.

`src/main.rs`:

```rust
use std::process::ExitCode;

use p08_twelve::config::Config;
use p08_twelve::server::{log, serve};

fn main() -> ExitCode {
    let config = match Config::from_env() {
        Ok(config) => config,
        Err(error) => {
            eprintln!("error: {error}");
            return ExitCode::from(2);
        }
    };
    log("info", &format!("starting with {config:?}"));

    match serve(&config) {
        Ok(()) => ExitCode::SUCCESS,
        Err(error) => {
            eprintln!("error: {error}");
            ExitCode::FAILURE
        }
    }
}
```

**What happens:** `main` is the composition root (Lesson 08.14). It loads the configuration, reports an error on **standard error** with exit status `2`, logs the configuration (with the secret hidden), and runs the server. Anything that goes
wrong while serving gives exit status `1`.

## 4. Run it

Open two terminals. In the first one (the error in the first line is on purpose):

```bash
cd code/p08-twelve
PORT=http cargo run -q
echo "exit status: $?"
PORT=18080 APP_MAX_REQUESTS=2 cargo run -q
```

```text
error: PORT: must be a number from 0 to 65535
exit status: 2
info starting with Config { port: 18080, greeting: "Hello", max_requests: Some(2), api_key: None }
info listening on 127.0.0.1:18080
```

The server now waits. In the second terminal, call it twice:

```bash
curl -s http://127.0.0.1:18080/hello
curl -s http://127.0.0.1:18080/
```

```text
Hello from the twelve-factor demo
Hello from the twelve-factor demo
```

In the first terminal, the log shows the two requests, and the server stops:

```text
info request: GET /hello HTTP/1.1
info request: GET / HTTP/1.1
info stopping
```

(If port 18080 is busy on your computer, choose another one: that is exactly what factor III is for.)

**Change behaviour without changing code.** Set the greeting and a secret, and let the *environment* decide where the logs go, by redirecting standard output to a file:

```bash
PORT=18081 APP_GREETING=Salom APP_API_KEY=abc123 APP_MAX_REQUESTS=1 cargo run -q > app.log
```

In the second terminal, `curl -s http://127.0.0.1:18081/` prints `Salom from the twelve-factor demo`. Then `cat app.log`:

```text
info starting with Config { port: 18081, greeting: "Salom", max_requests: Some(1), api_key: Some("***") }
info listening on 127.0.0.1:18081
info request: GET / HTTP/1.1
info stopping
```

The key `abc123` is nowhere in the log. The program has no idea whether its output goes to a terminal or a file: the deploy decides.

**Disposability (factor IX).** Start the server without a request limit, in the background, and ask the system to stop it politely with `SIGTERM`. (`$!` is the identifier of the last background process.)

```bash
cargo build -q
PORT=18082 target/debug/p08-twelve &
pid=$!
sleep 1
kill -TERM $pid
wait $pid
echo "exit status: $?"
```

```text
info starting with Config { port: 18082, greeting: "Hello", max_requests: None, api_key: None }
info listening on 127.0.0.1:18082
exit status: 143
```

`143` is `128 + 15`: the process was ended by signal number 15 (`SIGTERM`). Our program has no signal handler, so the operating system's default applied: it stops at once, even in the middle of a request. A production service
should catch `SIGTERM`, stop accepting new connections, finish the requests in progress, and then exit. Doing that needs either the `signal-hook` crate or an async runtime. Lesson 11.5 explains signals, Lesson 13.7 shows how a program handles them, and
Lesson 14.14 covers graceful shutdown. The habit to take from here: **start fast and assume that you can be stopped at any moment**, so that a stop never loses data. That is only possible if the process keeps no important state in memory (factor VI).

(Send `kill` only to a process you started yourself.)

## 5. The remaining factors in practice

**II. Dependencies, and V. Build, release, run.** `Cargo.toml` lists dependencies and `Cargo.lock` pins exact versions (Lesson 05.8). Build for deployment with `--locked`, so that nobody gets a surprise upgrade:

```bash
cargo build --release --locked
```

If `Cargo.toml` and `Cargo.lock` disagree (say, someone added a dependency and forgot to commit the lock file), the command fails instead of quietly choosing versions:

```text
error: cannot update the lock file .../Cargo.lock because --locked was passed to prevent this
```

The result, `target/release/p08-twelve`, is **one build artefact**. The same binary is combined with different configuration to make the release for the test server and the release for production. You never edit the binary or recompile to change a setting.

**I. Codebase.** One repository, one app. Test server and production run the same commit with different environment variables. If two apps share code, extract a library crate (a workspace member, Lesson 08.12) and let both depend on it; do not copy the code.

**IV. Backing services.** The code should not know whether the database is on the same machine or on another continent. It receives a URL (`DATABASE_URL=...`) in its configuration, and it talks to a port (a trait, Lesson 08.14). Swapping a local database for a hosted one is a configuration change, with no change to the code.

**VIII. Concurrency.** When load grows, you start more processes. This works only because the processes share nothing (factor VI). Threads and `async` tasks inside a process are fine too; they are about using the machine, while factor VIII is about scaling out.

**X. Dev/prod parity.** The more your laptop looks like production, the fewer "it works on my machine" surprises: the same type and a similar version of the database, the same configuration variables, the same way to start the program. Containers (Phase 11) make this easier. A common failure is testing with an in-memory fake only,
and meeting the real database for the first time in production; keep the contract test of Lesson 08.14 running against the real adapter as well.

**XII. Admin processes.** One-off jobs (a database migration, a data fix, a cleanup) run with the **same code and the same configuration** as the service. In Cargo, put them in a second binary: `src/bin/migrate.rs`, started with `cargo run --bin migrate`. They reuse the library and `Config`, so they cannot drift away from the service.

## 6. Where the twelve factors are not enough

The text is more than ten years old and was written for one style of hosting. Keep these limits in mind:

- **Environment variables are flat text.** Large or nested configuration (many routes, feature lists) is awkward there; a configuration file whose *path* comes from the environment is a good compromise.
- **Secrets deserve more care** than plain variables (Lesson 10.8).
- **Not every program is a service.** A library, a command-line tool or a desktop program uses only some factors.
- **Logs as plain lines** are a minimum. Modern services emit structured events (JSON, JavaScript Object Notation, or key-value fields; Lesson 09.10) so that tools can search them. The principle (write to standard output, let the platform collect) stays.
- **"Stateless" has costs.** Moving all state to a database makes every request slower than reading memory. Caches in memory are fine if you can lose them at any moment.

## Common mistakes

1. **Reading `std::env::var` all over the code.** Read once at start-up into a typed `Config` and pass it down. Scattered reads cannot be tested or validated.
2. **Defaults for secrets.** A default password in the code is a published password. A missing secret should be an error (or `None`) and never a built-in value.
3. **Parsing configuration lazily.** If the port is wrong, the program should fail at start-up, not at the first request.
4. **Writing log files from the program.** Log to standard output and let the environment route it.
5. **Keeping important data in process memory.** A restart, a crash or a second copy loses or splits it.
6. **Printing the whole `Config` with `{:?}` while it contains a secret.** Redact in `Debug`, as the demo does.

## Try it yourself

1. Create `code/p08-twelve` with the files of Section 3. Run `cargo test`, then `cargo clippy --all-targets -- -D warnings`.
2. Follow Section 4: the bad port, the two `curl` requests, the redirect to `app.log` (check that the key is not in the file), and the `SIGTERM` experiment.
3. Start the server twice on the same port. The second copy reports an error and exits with status 1:
   `error: Address already in use (os error 48)`. (The exact number in brackets depends on the operating system.)
4. Add `app.log` and `.env` to your `.gitignore`, then commit: `git add code/p08-twelve && git commit -m "feat(p08): twelve-factor demo service"`.

## Exercises

### Exercise 1 — Which factor? (easy)

Name the factor that each of these habits breaks: (a) `let db = "postgres://admin:hunter2@prod-db/app";` in the source; (b) the program opens `app.log` and rotates it every day; (c) a `static` counter that counts
visitors, shown on the home page; (d) a deploy script that edits a text file inside the compiled binary; (e) the production database is a different kind of database from the one used on laptops.

<details>
<summary>Hint</summary>

Match each to the table in Section 2.

</details>

<details>
<summary>Answer</summary>

(a) III, Config (and a security problem: a committed password). (b) XI, Logs: the program should write to standard output only. (c) VI, Processes: the count is lost on restart and differs between copies; keep it in a backing
service (IV). (d) V, Build/release/run: configuration must be combined with the build at the release stage, never by changing the artefact. (e) X, Dev/prod parity.

</details>

### Exercise 2 — Test the error path (medium)

Add a test to `config.rs` that proves that a non-numeric `APP_MAX_REQUESTS` is rejected, and that the message names the variable.

<details>
<summary>Hint</summary>

Use the helper `config_from(&[("APP_MAX_REQUESTS", "many")])` in the `tests` module and compare `error.to_string()`.

</details>

<details>
<summary>Answer</summary>

Inside `mod tests`:

```rust
    #[test]
    fn a_bad_request_limit_names_the_variable() {
        let error = config_from(&[("APP_MAX_REQUESTS", "many")]).unwrap_err();
        assert_eq!(error.to_string(), "APP_MAX_REQUESTS: must be a whole number");
    }
```

`cargo test` shows `test config::tests::a_bad_request_limit_names_the_variable ... ok`.

</details>

### Exercise 3 — A typed setting (hard)

Add the setting `APP_BIND` (an IP address, Internet Protocol; default `127.0.0.1`) so that the server can listen on `0.0.0.0` (all network interfaces, for example inside a container). It must be a real `IpAddr` in `Config`, rejected with a clear message if it is not an address,
and used by `serve`. Write tests for the default, a valid value and an invalid value.

<details>
<summary>Hint</summary>

`std::net::IpAddr` implements `FromStr`, so `text.parse::<IpAddr>()` works. `IpAddr::from([127, 0, 0, 1])` builds the default. `TcpListener::bind` accepts a tuple `(IpAddr, u16)`.

</details>

<details>
<summary>Answer</summary>

In `config.rs`: add `use std::net::IpAddr;`, a field `pub bind: IpAddr,` before `port`, and in `from_vars`, before `max_requests` is read (and `bind` in the `Ok(Config { ... })`):

```rust
        let bind = match get("APP_BIND") {
            None => IpAddr::from([127, 0, 0, 1]),
            Some(text) => text.parse::<IpAddr>().map_err(|_| ConfigError {
                name: "APP_BIND",
                reason: "must be an IP address",
            })?,
        };
```

In `server.rs`: `TcpListener::bind((config.bind, config.port))?`. Tests:

```rust
    #[test]
    fn the_bind_address_is_typed() {
        assert_eq!(config_from(&[]).unwrap().bind.to_string(), "127.0.0.1");
        let config = config_from(&[("APP_BIND", "0.0.0.0")]).unwrap();
        assert_eq!(config.bind.to_string(), "0.0.0.0");
        let error = config_from(&[("APP_BIND", "localhost")]).unwrap_err();
        assert_eq!(error.name, "APP_BIND");
    }
```

All tests pass (`6 passed` together with the existing ones from Section 3 and Exercise 2), and Clippy is clean.

> 🔒 **Security:** The default `127.0.0.1` accepts only connections from the same computer. `0.0.0.0` accepts connections from the network. A safe default with an explicit opt-in is the right design.

</details>

## Quiz

1. Why is configuration kept in the environment and not in the code?

<details>
<summary>Answer</summary>

Configuration changes between deploys (laptop, test, production) while the code stays the same. If it is in the code, every change needs a rebuild, and secrets end up in version control. The environment is understood by every system and tool.

</details>

2. Why does the service write logs to standard output instead of to a file?

<details>
<summary>Answer</summary>

Where logs go is a decision of the environment (a terminal, a file, a collection service), not of the program. A program that manages log files also has to handle rotation, permissions and disk space, and it cannot work in places without a writable disk.

</details>

3. What does "stateless process" mean, and what does it allow?

<details>
<summary>Answer</summary>

The process keeps no important data between requests; anything that must survive is in a backing service. It allows running many copies, restarting at any moment and moving the process to another machine without losing data.

</details>

4. What exit status does a process get when it is ended by `SIGTERM`, and why?

<details>
<summary>Answer</summary>

143 in a shell: `128 + 15`, where 15 is the number of `SIGTERM`.

</details>

5. What does `cargo build --release --locked` protect you from?

<details>
<summary>Answer</summary>

From a build that silently uses different dependency versions than the committed `Cargo.lock`: if the lock file would have to change, the command fails.

</details>

## Summary

- The **twelve-factor app** is a checklist of habits for services: one codebase; declared dependencies; config in the environment; backing services as attached resources; separate build, release and run; stateless processes;
  port binding; scaling out with processes; disposability; dev/prod parity; logs as a stream; admin tasks with the same code.
- Read configuration **once**, at start-up, into a typed struct with defaults and clear errors. Accept a lookup function so that tests do not touch the real environment.
- **Never print or commit secrets.** Redact them in `Debug`.
- A service **binds its own port**, writes **events to standard output** and keeps **no important state** in memory.
- Assume you can be stopped at any moment (`SIGTERM` gives 143 when not handled). Graceful shutdown comes later in the course.
- Build with `--locked`; one binary, many configurations; admin tasks are extra binaries using the same library.
- The methodology is a good default, not a law: env vars are flat, secrets need more care, and not every program is a service.

## Sources and further reading

- [The Twelve-Factor App](https://12factor.net/) — the original text, one short page per factor.
- [The Twelve-Factor App — III. Config](https://12factor.net/config) — why configuration belongs in the environment.
- [The Twelve-Factor App — XI. Logs](https://12factor.net/logs) — logs as an event stream.
- [The Twelve-Factor App — IX. Disposability](https://12factor.net/disposability) — fast start-up and graceful shutdown.
- [Rust standard library — `std::env::var`](https://doc.rust-lang.org/std/env/fn.var.html) — reading an environment variable, and its error type.
- [Rust standard library — `TcpListener`](https://doc.rust-lang.org/std/net/struct.TcpListener.html) — listening on a TCP port.
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — why the lock file exists and when to commit it.

---

[🏠 Start](../START-HERE.md) · [← 08.14 Layers and hexagonal architecture](14-layers-and-hexagonal-architecture.md) · [📋 Phase 08](README.md) · [08.16 Project: refactor a messy program →](16-project-refactor-a-messy-program.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
