<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.12 Containers and Docker basics](12-containers-and-docker-basics.md) · [📋 Phase 11](README.md) · [11.14 Docker Compose →](14-docker-compose.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/13-dockerfiles-for-rust.md) <!-- nav -->

# Lesson 11.13 — Dockerfiles for Rust programs

> **Phase 11 — Linux, the shell and containers** · Lesson 13 of 16 · ⏱ about 135 minutes

## What you will learn

- What a Dockerfile is, and how `docker build` turns it into an image.
- Why a naive image of a Rust program is more than two gigabytes, and how a **multi-stage build** makes it about 140 MB (megabytes), or about 55 MB with a distroless image.
- How the build cache works, and how to keep it useful for Rust (the dependency layer, `.dockerignore`).
- How to run the service as a non-root user, with a health check and a clean stop.
- How to find the classic mistake "published port, but the server answers nothing".

## Before you start

- You finished [Lesson 11.12 — Containers and Docker basics](12-containers-and-docker-basics.md). Docker Desktop is running, and you know `docker run`, `-p`, `-e`, and why PID 1 (the process with process ID 1, the first one in the container) must handle `SIGTERM`.
- You know the bind address ([Lesson 11.11](11-networking-commands.md)) and how a Rust program handles signals ([Lesson 11.5](05-processes-and-signals.md)).
- All commands run on the **Mac**, in the folder `~/projects/rust/code/p11-hello`. Docker builds Linux images for the architecture of your Mac (arm64).
- All images and containers we make are named `course-hello…`.

## New words

| Word | Meaning |
|---|---|
| **Dockerfile** | A text file with the steps to build an image: start from a base image, copy files, run commands, set the start command. |
| **base image** | The image named in `FROM`; the new image is built on top of it. |
| **build context** | The folder whose files `docker build` sends to the builder. `COPY` can use only files from it. |
| **`.dockerignore`** | A file that lists names that must not be sent in the build context (like `.gitignore` for Docker). |
| **multi-stage build** | A Dockerfile with several `FROM` lines. Early stages build the program; the last stage is the small image that is shipped. |
| **build cache** | Docker remembers the result of each step. If a step and everything before it are unchanged, the old result is reused (`CACHED`). |
| **`CMD` / `ENTRYPOINT`** | The command a container runs when it starts. `ENTRYPOINT` is the fixed program, `CMD` the default command or arguments. |
| **exec form / shell form** | `CMD ["/app"]` (a list: runs the program directly) versus `CMD /app` (a string: runs it through `/bin/sh -c`). |
| **health check** | A command that Docker runs regularly inside the container; its exit status says whether the service is healthy. |
| **distroless image** | A minimal image without a shell or package manager: only the runtime files (such as the C library) that programs need. |
| **glibc** (GNU C Library; GNU is the free-software project that also wrote many standard Linux tools) | The C library of most Linux distributions. A normal Rust program on Linux uses it, so the runtime image needs a compatible version. |
| **UID** (User Identifier) | The number of a user. `0` is root. A non-root service uses another number, for example `10001`. |

## 1. The service

We need something small to package. `p11-hello` is a tiny web service: `GET /` answers with a greeting, `GET /health` with `ok`. It reads `BIND` and `GREETING` from the environment (twelve-factor, [Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md)), stops cleanly on `SIGTERM`, and can check itself with `p11-hello --check`. Create it on the Mac:

```bash
cd ~/projects/rust
cargo new code/p11-hello
cd code/p11-hello
cargo add signal-hook@0.4
```

Replace `src/main.rs`:

```rust
use std::io::{ErrorKind, Read, Write};
use std::net::{TcpListener, TcpStream};
use std::process::ExitCode;
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::thread;
use std::time::Duration;

use signal_hook::consts::{SIGINT, SIGTERM};

/// Settings of the service, read from environment variables.
#[derive(Debug, PartialEq)]
struct Config {
    bind: String,
    greeting: String,
}

impl Config {
    /// `get` looks up a variable by name (so tests need no real environment).
    fn from_vars(get: impl Fn(&str) -> Option<String>) -> Config {
        Config {
            bind: get("BIND").unwrap_or_else(|| "127.0.0.1:8080".to_string()),
            greeting: get("GREETING").unwrap_or_else(|| "Hello".to_string()),
        }
    }
}

/// Decides the answer for the first line of an HTTP request.
fn route(request_line: &str, greeting: &str) -> (u16, String) {
    let mut parts = request_line.split_whitespace();
    match (parts.next(), parts.next()) {
        (Some("GET"), Some("/")) => (200, format!("{greeting} from p11-hello\n")),
        (Some("GET"), Some("/health")) => (200, "ok\n".to_string()),
        (Some("GET"), Some(_)) => (404, "not found\n".to_string()),
        _ => (400, "bad request\n".to_string()),
    }
}

/// The short text that follows the status number in the first line of a response.
fn reason(status: u16) -> &'static str {
    match status {
        200 => "OK",
        404 => "Not Found",
        _ => "Bad Request",
    }
}

fn handle(mut stream: TcpStream, greeting: &str) -> std::io::Result<()> {
    stream.set_read_timeout(Some(Duration::from_secs(2)))?;
    let mut buffer = [0u8; 2048];
    let size = stream.read(&mut buffer)?;
    let request = String::from_utf8_lossy(&buffer[..size]);
    let request_line = request.lines().next().unwrap_or("");

    let (status, body) = route(request_line, greeting);
    println!("{request_line} -> {status}");
    write!(
        stream,
        "HTTP/1.1 {status} {}\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{body}",
        reason(status),
        body.len()
    )
}

/// `p11-hello --check`: ask our own /health and report by exit code.
/// (A tiny image has no `curl`, so the program checks itself.)
fn check(bind: &str) -> bool {
    let port = bind.rsplit(':').next().unwrap_or("8080");
    let Ok(mut stream) = TcpStream::connect(format!("127.0.0.1:{port}")) else {
        return false;
    };
    let _ = stream.set_read_timeout(Some(Duration::from_secs(2)));
    let request = "GET /health HTTP/1.1\r\nHost: check\r\nConnection: close\r\n\r\n";
    let mut reply = String::new();
    stream.write_all(request.as_bytes()).is_ok()
        && stream.read_to_string(&mut reply).is_ok()
        && reply.starts_with("HTTP/1.1 200")
}

fn main() -> std::io::Result<ExitCode> {
    let config = Config::from_vars(|name| std::env::var(name).ok());

    if std::env::args().nth(1).as_deref() == Some("--check") {
        return Ok(if check(&config.bind) {
            ExitCode::SUCCESS
        } else {
            ExitCode::FAILURE
        });
    }

    // SIGTERM and SIGINT only set a flag; the loop below notices it.
    let stop = Arc::new(AtomicBool::new(false));
    signal_hook::flag::register(SIGTERM, Arc::clone(&stop))?;
    signal_hook::flag::register(SIGINT, Arc::clone(&stop))?;

    let listener = TcpListener::bind(&config.bind)?;
    listener.set_nonblocking(true)?;
    println!("listening on {}", listener.local_addr()?);

    while !stop.load(Ordering::Relaxed) {
        match listener.accept() {
            Ok((stream, _)) => {
                let result = stream
                    .set_nonblocking(false)
                    .and_then(|()| handle(stream, &config.greeting));
                if let Err(error) = result {
                    eprintln!("connection error: {error}");
                }
            }
            Err(error) if error.kind() == ErrorKind::WouldBlock => {
                thread::sleep(Duration::from_millis(50));
            }
            Err(error) => eprintln!("accept error: {error}"),
        }
    }
    println!("got a stop signal, bye");
    Ok(ExitCode::SUCCESS)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn config_has_defaults() {
        let config = Config::from_vars(|_| None);
        assert_eq!(config.bind, "127.0.0.1:8080");
        assert_eq!(config.greeting, "Hello");
    }

    #[test]
    fn config_reads_variables() {
        let config = Config::from_vars(|name| match name {
            "BIND" => Some("0.0.0.0:3000".to_string()),
            _ => None,
        });
        assert_eq!(config.bind, "0.0.0.0:3000");
    }

    #[test]
    fn routes_requests() {
        assert_eq!(
            route("GET / HTTP/1.1", "Hi"),
            (200, "Hi from p11-hello\n".into())
        );
        assert_eq!(route("GET /health HTTP/1.1", "Hi").0, 200);
        assert_eq!(route("GET /nope HTTP/1.1", "Hi").0, 404);
        assert_eq!(route("", "Hi").0, 400);
    }
}
```

**What happens:**

1. `Config::from_vars` takes a function that looks up variables, so the tests need no real environment ([Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md)). The default bind is the safe loopback.
2. `route` decides the answer from the request line; it has no I/O, so the test checks it directly. `handle` reads the request, prints one log line and writes the response (the idea of Lesson 11.11).
3. `signal_hook::flag::register` makes `SIGTERM` and `SIGINT` set a flag instead of killing the process. The listener is **non-blocking**: `accept` returns `WouldBlock` when nobody is connecting, and the loop sleeps 50 ms, so it notices the flag within 50 ms, finishes the current request and returns. The exit status is 0.
4. `check` is the program's own health check: it connects to its port on `127.0.0.1` and wants `HTTP/1.1 200`. A small image has no `curl`, so the program checks itself (Section 5).
5. `main` returns `ExitCode` so that `--check` can report success or failure to Docker.
6. The server serves one connection at a time: enough for this lesson. Lesson 06.11 and Phase 14 show how to serve many connections at once.

```bash
cargo test
cargo run
```

```text
running 3 tests
test tests::config_has_defaults ... ok
test tests::config_reads_variables ... ok
test tests::routes_requests ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`cargo run` prints `listening on 127.0.0.1:8080` and waits; `Ctrl + C` stops it, now with the line `got a stop signal, bye`. Also run `cargo clippy` and `cargo fmt` as always. The command `cargo test` created `Cargo.lock`; the Dockerfile needs it.

## 2. The first Dockerfile

A **Dockerfile** is a recipe. Create `Dockerfile.naive` in `code/p11-hello`:

```dockerfile
FROM rust:1.100
WORKDIR /app
COPY . .
RUN cargo build --release
CMD ["./target/release/p11-hello"]
```

| Instruction | Meaning |
|---|---|
| `FROM rust:1.100` | Start from the official Rust image (Debian Linux with `rustc`, `cargo` and a C toolchain). Every Dockerfile starts with `FROM`. |
| `WORKDIR /app` | Create `/app` and make it the current folder for the following steps and for the container. |
| `COPY . .` | Copy the build context (the current folder of the Mac) into `/app`. |
| `RUN cargo build --release` | Run a command **while building**; its file changes become a new layer. |
| `CMD [...]` | The command the container runs when it starts. Not executed during the build. |

Build it. `-f` chooses the file, `-t` names the image (`name:tag`), and the final `.` is the build context:

```bash
docker build -f Dockerfile.naive -t course-hello:naive . --progress=plain
```

```text
#6 [2/4] WORKDIR /app
#7 [3/4] COPY . .
#8 [4/4] RUN cargo build --release
#8 6.015    Compiling libc v0.2.191
#8 6.015    Compiling signal-hook v0.4.6
#8 6.903    Compiling errno v0.3.15
#8 6.954    Compiling signal-hook-registry v1.4.8
#8 7.226    Compiling p11-hello v0.1.0 (/app)
#8 7.507     Finished `release` profile [optimized] target(s) in 7.10s
#9 naming to docker.io/library/course-hello:naive done
```

(Shortened. `--progress=plain` prints the whole log instead of the live display. The first build also downloads `rust:1.100`, about one gigabyte, and takes longer.) Each numbered step is one layer. It works, but look at the size:

```bash
docker image ls course-hello
```

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
course-hello:naive   bef36fef6a39       2.24GB          565MB
```

**2.24 GB** (gigabytes) for a 600 KB (kilobyte: a thousand bytes) program. The image still contains the whole Rust toolchain, the source, the `target/` folder with all intermediate files, and a compiler that a server never needs. That costs download time, disk space, and **attack surface**: every extra program in an image is something an attacker could use.

## 3. A multi-stage build

The cure: build in one image, ship another. A **multi-stage** Dockerfile has several `FROM` lines; `COPY --from=` takes files from an earlier stage. Create `Dockerfile` (the default name) in `code/p11-hello`:

```dockerfile
# ---- build stage: the full Rust toolchain ----
FROM rust:1.100 AS build
WORKDIR /app

# 1. Build only the dependencies. This layer is reused until Cargo.toml or Cargo.lock change.
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo 'fn main() {}' > src/main.rs \
    && cargo build --release \
    && rm -r src

# 2. Build our own code.
COPY src ./src
RUN touch src/main.rs && cargo build --release

# ---- runtime stage: a small image with only what the program needs ----
FROM debian:trixie-slim
RUN useradd --system --uid 10001 --no-create-home --shell /usr/sbin/nologin app
COPY --from=build /app/target/release/p11-hello /usr/local/bin/p11-hello
USER app
ENV BIND=0.0.0.0:8080
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD ["/usr/local/bin/p11-hello", "--check"]
CMD ["/usr/local/bin/p11-hello"]
```

Also create `.dockerignore` (Section 4 explains it):

```text
target
.git
Dockerfile*
```

Build and compare:

```bash
docker build -t course-hello:1 .
docker image ls course-hello
```

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
course-hello:1       76c341ce8a35        141MB         30.4MB
course-hello:naive   bef36fef6a39       2.24GB          565MB
```

From 2.24 GB to **141 MB**. (`DISK USAGE` is the size unpacked on disk; `CONTENT SIZE` is the compressed size that travels over the network.)

**What happens:**

1. `FROM rust:1.100 AS build` starts the first stage and names it `build`. All Rust tools stay in this stage.
2. The final `FROM debian:trixie-slim` starts a **new image** from a small Debian. Only this stage becomes the result. The build stage is thrown away (it stays in the build cache but not in the image).
3. `COPY --from=build …` takes **one file**, the compiled program, from the first stage.
4. `useradd` creates a system user `app` with the fixed number 10001, no home folder and no login shell; `USER app` makes the container run as that user, **not as root**. Check it later with `docker top`.
5. `ENV BIND=0.0.0.0:8080` sets the default for the container: listen on all addresses (Lesson 11.11), on a port above 1024. `EXPOSE 8080` is documentation; it does not publish anything.
6. `HEALTHCHECK` runs `p11-hello --check` every 10 seconds inside the container (Section 5).
7. `CMD ["/usr/local/bin/p11-hello"]` is in **exec form**: the program itself becomes PID 1 (Section 6).

The layers of the result are few and small:

```text
0B	CMD ["/usr/local/bin/p11-hello"]
0B	HEALTHCHECK {Test:[CMD /usr/local/bin/p11-he…
0B	EXPOSE [8080/tcp]
0B	ENV BIND=0.0.0.0:8080
0B	USER app
582kB	COPY /app/target/release/p11-hello /usr/loca…
41kB	RUN /bin/sh -c useradd --system --uid 10001 …
110MB	# debian.sh --arch 'arm64' out/ 'trixie' '@1…
```

(`docker history --format '{{.Size}}\t{{.CreatedBy}}' course-hello:1`; the newest layer first.) Of 141 MB, 110 MB are the Debian base and 0.58 MB is our program.

> 💡 **Tip:** Why `debian:trixie-slim`? A normal Rust program on Linux links to the C library **glibc**. The program must run with a glibc that is at least as new as the one it was built with. `rust:1.100` is built on Debian 13 "trixie", so the runtime image is Debian 13 too. If you ship a program built on a newer system to an older one, you get errors such as ``version `GLIBC_2.xx' not found``. An `exec format error` means the program was built for another processor architecture (for example amd64 on an arm64 machine).

## 4. The build cache

Docker saves the result of every step. When you build again, it reuses a step if **the instruction and everything before it are unchanged** (and, for `COPY`, the copied files are unchanged). Rust makes this matter, because compiling dependencies is the slowest part. That is why the Dockerfile builds in two steps:

1. Copy only `Cargo.toml` and `Cargo.lock`, create a dummy `main.rs`, and build. This compiles all **dependencies** and puts them in one layer.
2. Copy the real `src` and build again. Only our own crate is compiled.

Change the default greeting in `src/main.rs` (for example `"Hello again"`) and build again:

```bash
docker build -t course-hello:1 . --progress=plain
```

```text
#8 [build 2/6] WORKDIR /app
#8 CACHED
#9 [build 3/6] COPY Cargo.toml Cargo.lock ./
#9 CACHED
#10 [build 4/6] RUN mkdir src && echo 'fn main() {}' > src/main.rs     && cargo build --release     && rm -r src
#10 CACHED
#11 [build 5/6] COPY src ./src
#12 [build 6/6] RUN touch src/main.rs && cargo build --release
#12 0.195    Compiling p11-hello v0.1.0 (/app)
#12 0.453     Finished `release` profile [optimized] target(s) in 0.28s
```

Steps 2 to 4, with the dependencies, are `CACHED`. Only our own crate was compiled again. (With one small dependency the saving is a few seconds; with real services that use dozens of crates it is minutes.) Three details matter:

- `touch src/main.rs` makes sure Cargo notices the real source: the dummy file from step 4 is newer than the copied file otherwise, and Cargo would think nothing changed and ship the empty program.
- **`.dockerignore`** keeps `target/` (gigabytes on a real project, and built for macOS!), `.git` and the Dockerfiles out of the build context. Without it, `COPY` would send them to the builder, make the context huge, and break the cache whenever a build file changes. Check the context size in the build log: `transferring context: …`.
- The order of the instructions is the order of the cache: put things that change rarely (base image, dependencies) first and things that change often (your source) last.

> ⚡ **Performance:** The tool `cargo-chef` automates the dependency layer for big workspaces, and BuildKit can keep Cargo's download cache between builds with `RUN --mount=type=cache,target=/usr/local/cargo/registry …`. Both are described in the Docker documentation on [build cache](https://docs.docker.com/build/cache/). Measure before you add them.

## 5. Run it: ports, the bind address and the health check

```bash
docker run -d --name course-hello -p 18080:8080 course-hello:1
curl -s http://localhost:18080/
docker top course-hello -eo user,pid,comm
docker exec course-hello id
```

```text
Hello from p11-hello
USER                PID                 COMMAND
10001               46281               p11-hello
uid=10001(app) gid=999(app) groups=999(app)
```

The service answers, and it runs as user 10001, not root. The health check shows in `docker ps`:

```bash
docker ps --filter name=course-hello --format '{{.Names}}  {{.Status}}'
```

```text
course-hello  Up 1 second (health: starting)
course-hello  Up 13 seconds (healthy)
```

(The same command, once right after the start and once 12 seconds later.) Docker ran `p11-hello --check` inside the container, got exit status 0, and marked it **healthy**. Orchestrators and Compose (Lesson 11.14) use this to decide when a service is ready and whether to restart it.

Now the classic mistake from Lesson 11.11. Remove the container and start it again with the **loopback** address:

```bash
docker rm -f course-hello
docker run -d --name course-hello -p 18081:8080 -e BIND=127.0.0.1:8080 course-hello:1
curl -sS -m 3 http://localhost:18081/; echo "curl status $?"
docker exec course-hello /usr/local/bin/p11-hello --check; echo "check inside: $?"
```

```text
curl: (56) Recv failure: Connection reset by peer
curl status 56
check inside: 0
```

The port is published, but the server only listens on the container's own loopback, so the traffic that Docker brings to the container's network interface finds nobody: the connection is reset. The check **inside** still passes, because it also uses the loopback; after 13 seconds `docker ps` reports `healthy` for a service that nobody can reach. Remember: **a health check from inside does not prove the published port works**. Always test from outside with `curl`. The fix is the environment variable:

```bash
docker rm -f course-hello
docker run -d --name course-hello -p 18081:8080 -e BIND=0.0.0.0:8080 -e GREETING=Salom course-hello:1
curl -sS -m 3 http://localhost:18081/
docker rm -f course-hello
```

```text
Salom from p11-hello
```

> 🔒 **Security:** In a container, `0.0.0.0` is right, because the container has its own network. Control who may reach the service when you **publish** the port: `-p 127.0.0.1:18081:8080` keeps it on your Mac only.

## 6. Stopping: exec form against shell form

Stop the container and read its log:

```bash
docker run -d --name course-hello -p 18080:8080 course-hello:1
curl -s http://localhost:18080/ > /dev/null
SECONDS=0; docker stop course-hello; echo "took ${SECONDS}s"
docker logs course-hello
docker inspect course-hello --format 'exit={{.State.ExitCode}}'
docker rm course-hello
```

```text
course-hello
took 0s
listening on 0.0.0.0:8080
GET / HTTP/1.1 -> 200
got a stop signal, bye
exit=0
```

`SIGTERM` reached our program at once, it finished and exited with 0 in under a second. This works because the `CMD` is in **exec form** (a JSON list; JSON, JavaScript Object Notation, is the text format with `["…", "…"]` lists): Docker starts the program directly, and it is PID 1. Compare the **shell form**, `CMD /usr/local/bin/p11-hello` (no brackets). Docker then runs `/bin/sh -c "/usr/local/bin/p11-hello"`. With such an image, `docker top` shows:

```text
PID                 COMMAND             COMMAND
61500               sh                  /bin/sh -c /usr/local/bin/p11-hello
61514               p11-hello           /usr/local/bin/p11-hello
```

PID 1 is the shell, and our program is its child. `SIGTERM` goes to the shell, which does not forward it. `docker stop` waited the whole grace period (the documented default is 10 seconds, and `docker stop -t SECONDS` changes it; on the test machine it took about 3 seconds), then killed everything: the log ends with `listening on 0.0.0.0:8080` and has no `got a stop signal`, and the exit code is `137` (killed). **Always write `CMD` and `ENTRYPOINT` in exec form.**

## 7. Smaller and safer: distroless

`debian:trixie-slim` still has a shell, `apt`, and dozens of tools. A **distroless** image keeps only the files a program needs to run: the C library, certificates, a non-root user. Google publishes them; `gcr.io/distroless/cc-debian13` matches our glibc. Create `Dockerfile.distroless`:

```dockerfile
FROM rust:1.100 AS build
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo 'fn main() {}' > src/main.rs \
    && cargo build --release \
    && rm -r src
COPY src ./src
RUN touch src/main.rs && cargo build --release

FROM gcr.io/distroless/cc-debian13:nonroot
COPY --from=build /app/target/release/p11-hello /p11-hello
ENV BIND=0.0.0.0:8080
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD ["/p11-hello", "--check"]
ENTRYPOINT ["/p11-hello"]
```

```bash
docker build -f Dockerfile.distroless -t course-hello:2 .
docker image ls course-hello
```

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
course-hello:1       3590ce5b3761        141MB         30.4MB
course-hello:2       7ce130e19ae3       54.6MB         11.2MB
course-hello:naive   bef36fef6a39       2.24GB          565MB
```

| Image | Size on disk | Has shell | User |
|---|---|---|---|
| `rust:1.100` + everything (naive) | 2.24 GB | yes | root |
| multi-stage on `debian:trixie-slim` | 141 MB | yes | `app` (10001) |
| multi-stage on distroless `cc-debian13:nonroot` | 54.6 MB | no | `nonroot` (65532) |

The tag `:nonroot` makes the default user number 65532. There is no shell, so `docker exec` has nothing to run:

```text
OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH
```

That is a feature (an attacker finds no shell), but it makes debugging harder. Use `docker logs`, build-time tests, or temporarily the image `gcr.io/distroless/cc-debian13:debug-nonroot`, which adds a small shell. Remember that the **health check cannot use `curl`** here; this is why `p11-hello --check` exists.

**More hardening at run time.** Docker can take away more rights. The service needs none of them:

```bash
docker run -d --name course-hello --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 64m --pids-limit 50 -p 18080:8080 course-hello:2
curl -s http://localhost:18080/
docker stats --no-stream --format '{{.Name}} {{.MemUsage}}' course-hello
docker rm -f course-hello
```

```text
Hello from p11-hello
course-hello 1.113MiB / 64MiB
```

`--read-only` makes the container's file system read-only (the program writes nothing), `--cap-drop ALL` removes all special Linux capabilities, `no-new-privileges` forbids gaining rights, and the limits cap memory and the number of processes. The service uses 1.1 MiB (mebibytes, units of 1024 × 1024 bytes). You can read the active options back with `docker inspect course-hello --format '{{.HostConfig.ReadonlyRootfs}} {{.HostConfig.CapDrop}} {{.HostConfig.Memory}}'`, which prints `true [ALL] 67108864`.

> 🔒 **Security:** The rule for production images: small base, non-root user, no secrets inside, exact version tags, only the files the program needs. In Phase 10 you learned to check dependencies with `cargo audit` ([Lesson 10.13](../10-security-fundamentals/13-supply-chain-security.md)); run it before building the image too.

Clean up the images you built:

```bash
docker image rm course-hello:naive course-hello:1 course-hello:2
```

## Common mistakes

1. **`COPY` fails or the image is huge** because `target/` is in the context. *Fix:* `.dockerignore` with `target`.
2. **The cache never helps** because `COPY . .` comes before the dependency build. *Fix:* copy `Cargo.toml` and `Cargo.lock` first, build, then copy `src`.
3. **The container ships an empty program** (it prints nothing) because the dummy `main.rs` build is cached. *Fix:* `touch src/main.rs` before the real build.
4. **`CMD /app` in shell form** hides `SIGTERM` from the program. *Fix:* `CMD ["/app"]`.
5. **Server bound to `127.0.0.1` in the container.** Connection reset or empty reply from outside. *Fix:* `BIND=0.0.0.0:…`, and test with `curl` from the host.
6. **Running as root** (no `USER`). *Fix:* create a user and switch to it; or use a `:nonroot` image.
7. **Unpinned tags and secrets in the build** (`FROM rust:latest`, `ENV API_KEY=…`, `COPY .env`). Tags move and secrets stay in the layers forever. *Fix:* pin `rust:1.100`, and pass secrets when the container **starts**.

## Try it yourself

1. Build `Dockerfile.naive` and the multi-stage `Dockerfile`; compare `docker image ls course-hello`.
2. Run the multi-stage image with `-p 127.0.0.1:18080:8080`, call `/` and `/health`, and read `docker logs`.
3. Run it with `-e BIND=127.0.0.1:8080` and see the failure. Then fix it.
4. Stop it with `docker stop` and read the exit code. Then change `CMD` to shell form in a copy of the Dockerfile and compare the stop time.
5. Remove the images and containers you made.

## Exercises

### Exercise 1 — Read a build log (easy)

Change one character of the greeting in `src/main.rs` and rebuild the multi-stage image with `--progress=plain`. Which steps say `CACHED`, which ones run again, and why? What happens if you change `Cargo.toml` (add a comment line)?

<details>
<summary>Hint</summary>

Steps that come before the first changed input stay cached; after it, all steps run again.

</details>

<details>
<summary>Answer</summary>

After a change in `src/main.rs`: the base images, `WORKDIR`, `COPY Cargo.toml Cargo.lock` and the dependency build are `CACHED`. `COPY src ./src` and `RUN touch … && cargo build --release` run again, and so does the final `COPY --from=build`. If you add a comment to `Cargo.toml`, the `COPY Cargo.toml Cargo.lock ./` step sees a changed file, so **every step after it** runs again, including the dependency build. That is why only `Cargo.toml` and `Cargo.lock` are copied before the dependency step: they change much more rarely than the source.

</details>

### Exercise 2 — A new route, a test and a rebuild (medium)

Add the route `GET /version` that answers with the package version (`env!("CARGO_PKG_VERSION")`). Write the test first (`cargo test` must fail), then make it pass, rebuild the image and check `curl http://localhost:18080/version` in the container.

<details>
<summary>Hint</summary>

`env!("CARGO_PKG_VERSION")` is replaced by the version from `Cargo.toml` at compile time. Add one more `match` arm in `route`, before the `404` arm, and one more `assert_eq!` in `routes_requests`.

</details>

<details>
<summary>Answer</summary>

In `route`, add after the `/health` arm:

```rust
        (Some("GET"), Some("/version")) => (200, format!("{}\n", env!("CARGO_PKG_VERSION"))),
```

and in the test:

```rust
        assert_eq!(
            route("GET /version HTTP/1.1", "Hi"),
            (200, "0.1.0\n".into())
        );
```

Before you add the arm, the test fails with `left: (404, "not found\n")` because the route falls into the `404` arm. After it, `cargo test` prints `test result: ok. 3 passed`. Then:

```bash
docker build -t course-hello:1 .
docker run -d --name course-hello -p 18080:8080 course-hello:1
curl -s http://localhost:18080/version
docker rm -f course-hello
```

The last `curl` prints `0.1.0`.

Why it works: `env!` reads the variable `CARGO_PKG_VERSION`, which Cargo sets during the build, so the binary carries its version. Because dependencies are cached, the rebuild compiles only our crate.

</details>

## Quiz

1. Why is the naive image 2.24 GB, and what does a multi-stage build change?

<details>
<summary>Answer</summary>

It contains the whole Rust toolchain, the sources and the `target/` folder. A multi-stage build compiles in one stage and copies only the finished program into a small final image.

</details>

2. In what order should a Rust Dockerfile copy files so that the cache helps?

<details>
<summary>Answer</summary>

First `Cargo.toml` and `Cargo.lock`, build the dependencies; then copy `src` and build the program. Dependencies change rarely, the source often, so the expensive dependency layer stays cached.

</details>

3. Why is `CMD ["/app"]` better than `CMD /app`?

<details>
<summary>Answer</summary>

The exec form starts the program directly as PID 1, so it receives `SIGTERM` from `docker stop`. The shell form starts `/bin/sh -c`, the shell is PID 1 and does not forward the signal, so the stop waits for the grace period and then kills.

</details>

4. A service in a container listens on `127.0.0.1:8080`. Why can't you reach it through `-p 8080:8080`, and why does the internal health check still pass?

<details>
<summary>Answer</summary>

Published traffic arrives on the container's network interface, not on its loopback. The health check runs inside the container and uses the loopback, so it succeeds. The service must listen on `0.0.0.0`.

</details>

5. Why does the health check use `p11-hello --check` instead of `curl`?

<details>
<summary>Answer</summary>

A small or distroless image has no `curl` and no shell. The program contains its own check, so the image needs nothing extra.

</details>

## Summary

- A Dockerfile is a list of instructions; every step is a cached layer. `docker build -t name:tag .` builds it, and the `.` is the build context.
- A naive Rust image is over two gigabytes. A multi-stage build compiles in a `rust:1.100` stage and ships only the binary in `debian:trixie-slim` (141 MB) or distroless `cc-debian13` (55 MB).
- Cache the dependencies: copy `Cargo.toml` and `Cargo.lock` first, build with a dummy `main.rs`, then copy `src` and `touch` it. Use `.dockerignore`.
- Run as a non-root user, use exec form for `CMD`/`ENTRYPOINT` so that the program is PID 1 and handles `SIGTERM`, and read configuration from environment variables.
- Inside the container listen on `0.0.0.0`; decide who may reach the service by how you publish the port. A health check from inside does not prove that the port works from outside.
- The runtime base must match the glibc of the build image; pin versions of all images.

## Sources and further reading

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) — every instruction, including `HEALTHCHECK`, `USER`, `CMD` and `ENTRYPOINT` forms.
- [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/) — the official guide to stages and `COPY --from`.
- [Build cache](https://docs.docker.com/build/cache/) — how invalidation works, and cache mounts.
- [Building best practices](https://docs.docker.com/build/building/best-practices/) — pinning, `.dockerignore`, small images.
- [Distroless images](https://github.com/GoogleContainerTools/distroless) — the images, their tags (`nonroot`, `debug`) and what they contain.
- [Docker Official Image: rust](https://hub.docker.com/_/rust) and [debian](https://hub.docker.com/_/debian) — the available tags and their base systems.

---

[🏠 Start](../START-HERE.md) · [← 11.12 Containers and Docker basics](12-containers-and-docker-basics.md) · [📋 Phase 11](README.md) · [11.14 Docker Compose →](14-docker-compose.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
