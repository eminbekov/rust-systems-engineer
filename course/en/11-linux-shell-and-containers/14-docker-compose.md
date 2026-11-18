<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.13 Dockerfiles for Rust programs](13-dockerfiles-for-rust.md) · [📋 Phase 11](README.md) · [11.15 Project: deploy a Rust service →](15-project-deploy-a-rust-service.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/14-docker-compose.md) <!-- nav -->

# Lesson 11.14 — Docker Compose

> **Phase 11 — Linux, the shell and containers** · Lesson 14 of 16 · ⏱ about 135 minutes

## What you will learn

- What Docker Compose is, and how one `compose.yaml` file describes a whole application: a Rust service and a PostgreSQL database.
- How services find each other by name, and how `depends_on` with a health check controls the start order.
- How to keep data in a volume and a password in a secret file, not in the image or in Git.
- How a restart policy brings a crashed service back, and what happens when the database goes away.
- The daily commands: `up`, `ps`, `logs`, `exec`, `down`.

## Before you start

- You finished [Lesson 11.13 — Dockerfiles for Rust programs](13-dockerfiles-for-rust.md). The folder `code/p11-hello` with its `Dockerfile` and `.dockerignore` exists, and Docker Desktop is running.
- You know containers, volumes, health checks and secrets ([Lesson 11.12](12-containers-and-docker-basics.md), [Lesson 10.8](../10-security-fundamentals/08-secrets-management.md)).
- All commands run on the **Mac**. Docker Compose is part of Docker Desktop: `docker compose version` must print a version.
- We use the official image `postgres:18`. You need no database knowledge beyond one `SELECT`; databases are taught in Phase 17.

## New words

| Word | Meaning |
|---|---|
| **Docker Compose** | A tool (`docker compose …`) that starts several containers together from one description file. |
| **`compose.yaml`** | The description file. It lists **services**, and may list volumes, networks and secrets. |
| **YAML** (YAML Ain't Markup Language) | A text format for data: `key: value` pairs, lists with `-`, and **indentation with spaces** for nesting (never tabs). |
| **service** | One part of the application, run as one or more containers from one image, for example `app` or `db`. |
| **project** | One running instance of a Compose file. Its name (here `course-counter`) is the prefix of all containers, networks and volumes it creates. |
| **default network** | Compose creates a private network for the project. Every service can reach the others by the **service name** (for example `db`) as a host name. |
| **`depends_on`** | Says that a service needs another one first. With `condition: service_healthy` Compose waits until the health check of the other service passes. |
| **restart policy** | What Docker does when a container ends: `no`, `on-failure`, `always`, `unless-stopped`. |
| **secret** | A piece of sensitive data (a password) that Compose gives to a container as a file under `/run/secrets/`, not as an environment variable. |
| **PostgreSQL** | A popular open-source database server. The client program is `psql`. |
| **SQL** (Structured Query Language) | The language of relational databases: `INSERT`, `SELECT`, … |

## 1. What Compose does

A real application is rarely one program. Our example has two: a Rust web service and a database. With plain `docker run` you would type two long commands in the right order and create the network and the volume by hand. **Compose** writes all of it down in a file that you keep in Git, and starts everything with one command:

```text
   your Mac
   127.0.0.1:18080 ---> +------------------ project "course-counter" -------------------+
                        |  app  (our Rust service)  --- network "default" ---  db      |
                        |       listens on 8080          (names resolve)   (postgres)  |
                        +-----------------------------------------------------|--------+
                                                                       volume "pgdata"
```

The file uses **YAML**. Three rules are enough: `key: value` makes a map entry; `- item` makes a list entry; and **indentation** (spaces, usually two per level) shows what belongs to what. A wrong indent changes the meaning or breaks the file, and a tab is an error.

## 2. The Rust service with a database

The service `p11-counter` is `p11-hello` plus one route: `GET /hits` stores a visit in PostgreSQL and answers with the total number of visits. Start from a copy:

```bash
cd ~/projects/rust/code
cp -r p11-hello p11-counter
cd p11-counter
rm -rf target Dockerfile.naive Dockerfile.distroless
sed -i '' 's/p11-hello/p11-counter/g' Cargo.toml Dockerfile src/main.rs
cargo add postgres@0.19
```

(`sed -i '' 's/a/b/g' files` replaces text inside the files; on the Mac the empty `''` after `-i` is required. Do not run it on files you have not copied.) The crate `postgres` is the synchronous PostgreSQL client. The service reads the **password from a file**, the way real deployments do. Edit `src/main.rs` in these steps.

**Step 1.** At the top add two `use` lines: `use std::error::Error;` before the other `std` lines, and `use postgres::{Client, NoTls};` before `use signal_hook…`.

**Step 2.** Replace `Config` and its `impl` with:

```rust
/// Settings of the service, read from environment variables.
#[derive(Debug, PartialEq)]
struct Config {
    bind: String,
    greeting: String,
    db_host: String,
    db_user: String,
    db_name: String,
    db_password_file: String,
}

impl Config {
    /// `get` looks up a variable by name (so tests need no real environment).
    fn from_vars(get: impl Fn(&str) -> Option<String>) -> Config {
        let var = |name: &str, default: &str| get(name).unwrap_or_else(|| default.to_string());
        Config {
            bind: var("BIND", "127.0.0.1:8080"),
            greeting: var("GREETING", "Hello"),
            db_host: var("DB_HOST", "localhost"),
            db_user: var("DB_USER", "app"),
            db_name: var("DB_NAME", "app"),
            db_password_file: var("DB_PASSWORD_FILE", "/run/secrets/db_password"),
        }
    }
}
```

**Step 3.** In `reason`, add the arm `503 => "Service Unavailable",` before the `_` arm.

**Step 4.** After `reason`, add the database code:

```rust
/// Opens a connection and makes sure the table exists. The password is read from a file.
fn connect(config: &Config) -> Result<Client, Box<dyn Error>> {
    let password = std::fs::read_to_string(&config.db_password_file)?;
    let mut client = postgres::Config::new()
        .host(&config.db_host)
        .user(&config.db_user)
        .password(password.trim())
        .dbname(&config.db_name)
        .connect_timeout(Duration::from_secs(3))
        .connect(NoTls)?;
    client.batch_execute(
        "CREATE TABLE IF NOT EXISTS hits (id BIGSERIAL PRIMARY KEY, at TIMESTAMPTZ NOT NULL DEFAULT now())",
    )?;
    Ok(client)
}

/// Stores one visit and returns how many visits there are in total.
fn record_hit(client: &mut Client) -> Result<i64, postgres::Error> {
    client.execute("INSERT INTO hits DEFAULT VALUES", &[])?;
    let row = client.query_one("SELECT count(*) FROM hits", &[])?;
    Ok(row.get(0))
}

/// Turns an error and its causes into one line, for example `db error: FATAL: ...`.
fn describe(error: &dyn Error) -> String {
    let mut text = error.to_string();
    let mut source = error.source();
    while let Some(cause) = source {
        text = format!("{text}: {cause}");
        source = cause.source();
    }
    text
}

/// A database connection that is opened when needed and opened again after a failure.
struct Database<'a> {
    config: &'a Config,
    client: Option<Client>,
}

impl Database<'_> {
    fn hit(&mut self) -> Result<i64, Box<dyn Error>> {
        // Try the open connection first; if it is broken, forget it and connect again.
        if let Some(client) = self.client.as_mut() {
            match record_hit(client) {
                Ok(total) => return Ok(total),
                Err(_) => self.client = None,
            }
        }
        let client = self.client.insert(connect(self.config)?);
        let result = record_hit(client);
        if result.is_err() {
            self.client = None;
        }
        Ok(result?)
    }
}
```

**Step 5.** Replace `handle` with a version that knows `/hits`:

```rust
fn handle(mut stream: TcpStream, greeting: &str, database: &mut Database) -> std::io::Result<()> {
    stream.set_read_timeout(Some(Duration::from_secs(2)))?;
    let mut buffer = [0u8; 2048];
    let size = stream.read(&mut buffer)?;
    let request = String::from_utf8_lossy(&buffer[..size]);
    let request_line = request.lines().next().unwrap_or("");

    let (status, body) = if request_line.starts_with("GET /hits ") {
        match database.hit() {
            Ok(total) => (200, format!("hits: {total}\n")),
            Err(error) => {
                eprintln!("database error: {}", describe(error.as_ref()));
                (503, "database unavailable\n".to_string())
            }
        }
    } else {
        route(request_line, greeting)
    };
    println!("{request_line} -> {status}");
    write!(
        stream,
        "HTTP/1.1 {status} {}\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{body}",
        reason(status),
        body.len()
    )
}
```

**Step 6.** In `main`, just before the `while` loop add `let mut database = Database { config: &config, client: None };` (formatted over four lines by `cargo fmt`), and change the call to `.and_then(|()| handle(stream, &config.greeting, &mut database))`. In the test `config_has_defaults` add `assert_eq!(config.db_host, "localhost");`.

Format, check and test:

```bash
cargo fmt && cargo clippy --all-targets && cargo test
```

```text
running 3 tests
test tests::config_reads_variables ... ok
test tests::config_has_defaults ... ok
test tests::routes_requests ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. The connection is **not** opened at start-up but at the first `/hits`. If the database is not ready or goes away, the service stays alive, answers `503 database unavailable` for `/hits` and `200` for `/health`, and connects again at the next request. A web service should survive the loss of a dependency.
2. `hit()` tries the open connection first. If that fails (the database was restarted), it forgets the connection, opens a new one and tries once more.
3. The password comes from the file named by `DB_PASSWORD_FILE`, not from an environment variable. Compose will put the secret there ([Lesson 10.8](../10-security-fundamentals/08-secrets-management.md)).
4. `describe` walks the chain of causes of an error (`Error::source`), because the `postgres` error alone prints just `db error`.
5. The unit tests cover only what needs no database. The real test is the running stack (Exercise 2).

The `Dockerfile` of Lesson 11.13 needs no change (`sed` already renamed the program in it). Create the password file. It is a secret, so it must never be committed to Git; `.gitignore` takes care of it:

```bash
mkdir -p secrets
head -c 18 /dev/urandom | base64 > secrets/db_password.txt
chmod 600 secrets/db_password.txt
printf 'secrets/\n.env\ntarget\n' > .gitignore
```

## 3. The Compose file

Create `compose.yaml` in `code/p11-counter`:

```yaml
name: course-counter

services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: app
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - pgdata:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

  app:
    build: .
    image: course-counter:1
    environment:
      BIND: 0.0.0.0:8080
      DB_HOST: db
      DB_USER: app
      DB_NAME: app
    secrets:
      - db_password
    ports:
      - "127.0.0.1:${HOST_PORT:-18080}:8080"
    depends_on:
      db:
        condition: service_healthy
    restart: on-failure

volumes:
  pgdata:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

| Key | Meaning |
|---|---|
| `name:` | The project name. Every object gets this prefix (`course-counter-db-1`, network `course-counter_default`, volume `course-counter_pgdata`), so cleaning up is easy. |
| `services:` | One entry per service. |
| `image: postgres:18` | Use this official image. A pinned major version, not `latest`. |
| `environment:` | Environment variables of the container. For the database, the official image reads the user and database name, and the password **from a file** when the name ends with `_FILE`. |
| `secrets:` (inside a service) | Mount the secret `db_password` at `/run/secrets/db_password` in this container. |
| `volumes: - pgdata:/var/lib/postgresql` | Mount the named volume `pgdata` where PostgreSQL 18 keeps its data. |
| `healthcheck:` | The command to test health (`pg_isready` is part of the image) and how often. |
| `build: .` and `image:` | Build the image of this service from the `Dockerfile` of the current folder and name it `course-counter:1`. |
| `DB_HOST: db` | The **service name** is the host name inside the project network. |
| `ports: - "127.0.0.1:…:8080"` | Publish the service on the Mac, only on loopback. `${HOST_PORT:-18080}` means "the variable `HOST_PORT`, or 18080 if it is not set". |
| `depends_on: … service_healthy` | Start `app` only after the health check of `db` passes. |
| `restart:` | The policy (Section 5). |
| top-level `volumes:` and `secrets:` | Declare the volume and the secret: here the secret is the file `secrets/db_password.txt`. |

The database service has **no** `ports:` entry. Nobody outside the project can reach PostgreSQL; only `app` can, over the project network. Check the file before you start anything:

```bash
docker compose config --quiet && echo "config ok"
```

```text
config ok
```

(`docker compose config` without `--quiet` prints the file as Compose understands it, with all variables replaced. It is the first thing to run when something looks wrong.)

## 4. Up, look, down

```bash
docker compose up -d --build --wait
docker compose ps
```

`up` creates the network, the volume and the containers in order, `-d` runs in the background, `--build` builds the image of `app`, and `--wait` returns when all services are healthy. The last lines of the output:

```text
 Volume course-counter_pgdata Created
 Network course-counter_default Created
 Container course-counter-db-1 Created
 Container course-counter-app-1 Created
 Container course-counter-db-1 Started
 Container course-counter-db-1 Healthy
 Container course-counter-app-1 Started
 Container course-counter-app-1 Healthy
```

```text
NAME                   IMAGE              COMMAND                  SERVICE   CREATED          STATUS                    PORTS
course-counter-app-1   course-counter:1   "/usr/local/bin/p11-…"   app       16 seconds ago   Up 10 seconds (healthy)   127.0.0.1:18080->8080/tcp
course-counter-db-1    postgres:18        "docker-entrypoint.s…"   db        16 seconds ago   Up 16 seconds (healthy)   5432/tcp
```

(The first build downloads `postgres:18` and compiles the crates: about a minute.) `app` started **after** `db` was healthy, and `db` shows `5432/tcp` without an arrow: the port exists inside the network only. Use the service:

```bash
curl -s http://localhost:18080/hits
curl -s http://localhost:18080/hits
curl -s http://localhost:18080/
docker compose logs --no-log-prefix app
docker compose exec db psql -U app -d app -c 'SELECT count(*) FROM hits;'
```

```text
hits: 1
hits: 2
Hello from p11-counter
listening on 0.0.0.0:8080
GET /hits HTTP/1.1 -> 200
GET /hits HTTP/1.1 -> 200
GET / HTTP/1.1 -> 200
 count 
-------
     2
(1 row)
```

The service counted two visits and the database table holds the same two rows. `docker compose exec db …` runs a command in the running `db` container (`psql` is the PostgreSQL client). From inside the project network a service name is an address:

```bash
docker compose exec db getent hosts app
```

```text
172.31.0.3      app
```

| Command | What it does |
|---|---|
| `docker compose up -d [--build] [--wait]` | create and start (rebuild, wait until healthy) |
| `docker compose ps` | list the containers of the project |
| `docker compose logs [-f] [service]` | show the logs of all services or one (`-f` follows) |
| `docker compose exec service command` | run a command in a running service |
| `docker compose stop` / `start` / `restart [service]` | stop, start, restart without deleting |
| `docker compose down` | stop and delete containers and network, **keep volumes** |
| `docker compose down -v` | the same and delete the volumes too (the data is gone) |

**Data survives `down`, not `down -v`.** Run `docker compose down`, then `docker compose up -d --wait` and `curl -s http://localhost:18080/hits`: the answer is `hits: 3`, because the volume `course-counter_pgdata` was kept. After `docker compose down -v` and a new `up`, the first answer is `hits: 1` again.

> ⚠️ **Common mistake:** Running `docker compose down -v` out of habit on a project whose volume holds real data. The `-v` deletes it without asking. In production, back up first (Exercise 3).

## 5. When things fail

**The database goes away.** The service survives:

```bash
docker compose stop db
curl -s -w "[%{http_code}]\n" http://localhost:18080/hits
docker compose logs --no-log-prefix app | tail -1
docker compose start db
sleep 6
curl -s -w "[%{http_code}]\n" http://localhost:18080/hits
```

```text
database unavailable
[503]
database error: error connecting to server: failed to lookup address information: Name or service not known
hits: 4
[200]
```

While `db` is stopped, `/hits` answers `503` and the log says why; `/health` would still answer `200`. When the database is back, the next request reconnects by itself, without a restart of `app`.

**The service crashes.** The restart policy decides what Docker does:

| `restart:` | Meaning |
|---|---|
| `no` (default) | never restart |
| `on-failure` | restart when the process ends with a non-zero status |
| `always` | restart whatever the exit status, also after a Docker restart |
| `unless-stopped` | like `always`, but not if you stopped the container yourself |

To see it, add a deliberately bad route at the start of `handle`, right after `let request_line = …;`. It is **training only**; delete it afterwards, because a service must never have a button that kills it:

```rust
    if request_line.starts_with("GET /crash ") {
        eprintln!("crashing on purpose");
        std::process::exit(1);
    }
```

Rebuild with `docker compose up -d --build --wait`, and then:

```bash
curl -sS -m 3 http://localhost:18080/crash
sleep 4
docker compose ps --format 'table {{.Service}}\t{{.Status}}'
docker inspect course-counter-app-1 --format 'restarts={{.RestartCount}}'
docker compose logs --no-log-prefix app | tail -2
```

```text
curl: (52) Empty reply from server
SERVICE   STATUS
app       Up 3 seconds (health: starting)
db        Up 20 seconds (healthy)
restarts=1
crashing on purpose
listening on 0.0.0.0:8080
```

The process ended with status 1, Docker started it again (`restarts=1`), and the service is back in seconds. With `on-failure` a normal stop (`docker compose stop`, status 0 after `SIGTERM`) is **not** restarted. Also note: `docker kill` and `docker compose kill` count as a *manual* stop, so the policy does **not** apply; this is why we used a real crash.

> 💡 **Tip:** A restart policy is a safety net, not a fix. Read `docker compose logs` to find out why it crashed. Add a limit (`on-failure:5`) if a crash loop would burn resources.

**A wrong or changed password.** `POSTGRES_PASSWORD_FILE` is read only when PostgreSQL creates a **new** data directory. If you change `secrets/db_password.txt` while the volume already exists, the database keeps the old password:

```text
database unavailable
app-1  | database error: db error: FATAL: password authentication failed for user "app"
```

Either put the old password back, change the password inside the database with `ALTER USER`, or (only if the data is disposable) run `docker compose down -v` to start from scratch. The `describe` function paid off here: without it the log would only say `db error`.

> 🔒 **Security:** Secrets in Compose are files, not environment variables, because variables are visible in `docker inspect` and in process listings. Keep the file out of Git (`.gitignore`), and out of the image (it is mounted at run time, not copied). On Docker Desktop for Mac the container user can read the file as it is. On a Linux server the file keeps the permissions of the host, so a file with mode `600` owned by your user may be unreadable for the container user; check with `docker compose exec`, and give the file the right owner or group instead of making it world-readable. Real production systems use a secret store of the platform ([Lesson 10.8](../10-security-fundamentals/08-secrets-management.md)).

Clean up when you are done: `docker compose down -v`, and `docker image rm course-counter:1`.

## Common mistakes

1. **Bad indentation or a tab in YAML.** Compose says `yaml: line 12: …` or silently ignores a key. *Fix:* spaces only, two per level; `docker compose config` shows what was understood.
2. **`depends_on` without a health check** only waits until the container *starts*, not until the database is ready. *Fix:* `condition: service_healthy` with a `healthcheck`.
3. **`localhost` as the database host.** Inside `app`, `localhost` is the app container. *Fix:* use the service name: `DB_HOST: db`.
4. **Publishing the database port** (`5432:5432`) on all interfaces. *Fix:* publish only what users need; `db` needs no `ports:`.
5. **A password in `compose.yaml` or in Git.** *Fix:* secret file, `.gitignore`; rotate a password that has ever been committed.
6. **`docker compose down -v` that deletes the data.** *Fix:* use `down`; back up before `-v`.
7. **Expecting `restart:` to work after `docker kill`.** *Fix:* test with a real crash.
8. **Changing the secret on an existing volume** (Section 5). *Fix:* know that the password is set once.

## Try it yourself

1. Build the stack, call `/hits` three times, and look at the table with `psql`.
2. Run `docker compose down`, start again, and check that the counter continues. Then use `down -v` and see it restart at 1.
3. Stop only `db`, call `/hits`, start `db`, and call again.
4. Start the stack with another host port without editing the file: `HOST_PORT=18090 docker compose up -d --wait`.
5. Remove everything: `docker compose down -v` and the image.

## Exercises

### Exercise 1 — Change a setting without editing the file (easy)

Run the stack on host port 18090 using the variable, check with `docker compose ps` which port is published, and call `/hits` on the new port.

<details>
<summary>Hint</summary>

The file has `${HOST_PORT:-18080}`. Set the variable in front of the command (`HOST_PORT=… docker compose …`). The container is re-created because the port setting changed.

</details>

<details>
<summary>Answer</summary>

```bash
HOST_PORT=18090 docker compose up -d --wait
docker compose ps --format 'table {{.Service}}\t{{.Ports}}'
curl -s http://localhost:18090/hits
```

```text
SERVICE   PORTS
app       127.0.0.1:18090->8080/tcp
db        5432/tcp
hits: 1
```

(The counter shows `1` after a fresh `down -v`; otherwise it continues from the stored value.)

Why it works: Compose replaces `${HOST_PORT:-18080}` with the variable of the shell, or with the default after `:-`. The same file therefore works on machines where port 18080 is busy.

</details>

### Exercise 2 — A test for the whole stack (medium)

Write `test-compose.sh` that starts the stack, checks that the first two visits give `hits: 1` and `hits: 2`, restarts the database and checks that the data survived (`hits: 3`), and always removes the stack with a `trap`.

<details>
<summary>Hint</summary>

`docker compose up -d --build --wait` returns when the services are healthy. `trap '…down -v…' EXIT` cleans up on every exit path. After `docker compose restart db` wait a few seconds for the database to accept connections.

</details>

<details>
<summary>Answer</summary>

```bash
#!/usr/bin/env bash
# test-compose.sh - starts the stack, checks the counter, and always cleans up.
set -u
cd "$(dirname "$0")" || exit 1

failures=0
check() {   # check "description" expected actual
    if [[ $2 == "$3" ]]; then
        echo "ok   - $1"
    else
        echo "FAIL - $1 (expected [$2], got [$3])"
        failures=$((failures + 1))
    fi
}

trap 'docker compose down -v > /dev/null 2>&1' EXIT

docker compose up -d --build --wait > /dev/null 2>&1
check "stack starts and is healthy" 0 "$?"

first=$(curl -s http://localhost:18080/hits)
second=$(curl -s http://localhost:18080/hits)
check "first visit" "hits: 1" "$first"
check "second visit" "hits: 2" "$second"

docker compose restart db > /dev/null 2>&1
sleep 5
check "data survives a database restart" "hits: 3" "$(curl -s http://localhost:18080/hits)"

(( failures == 0 ))
```

```text
ok   - stack starts and is healthy
ok   - first visit
ok   - second visit
ok   - data survives a database restart
```

Why it works: the trap removes containers, network and volume even when a check fails, so the next run starts from `hits: 1`. The test relies on the retry in `Database::hit`: the first request after the restart finds the old connection broken, opens a new one and still answers correctly.

</details>

### Exercise 3 — Back up and restore the database (hard)

With the stack running and a few visits recorded, write the content of the database to a file `backup.sql` with `pg_dump` (run inside the `db` container, output to your Mac). Then delete everything with `down -v`, start again, restore the file with `psql`, and show that `/hits` continues from the old number.

<details>
<summary>Hint</summary>

`docker compose exec -T db pg_dump -U app app > backup.sql` (`-T` turns off the terminal, needed for redirection). To restore: `docker compose exec -T db psql -U app -d app < backup.sql`. The new database is empty and the app creates the table only at the first `/hits`, so restore **before** the first visit, or drop the table first.

</details>

<details>
<summary>Answer</summary>

```bash
curl -s http://localhost:18080/hits               # hits: 4, for example
docker compose exec -T db pg_dump -U app app > backup.sql
docker compose down -v
docker compose up -d --wait
docker compose exec -T db psql -q -U app -d app < backup.sql > /dev/null
docker compose exec db psql -U app -d app -t -c 'SELECT count(*) FROM hits;'
curl -s http://localhost:18080/hits
```

```text
     4
hits: 5
```

Why it works: `pg_dump` writes SQL statements that rebuild the table and its rows (`CREATE TABLE`, `COPY`, and the sequence value). The restore runs them in the new, empty database, so the counter continues. Keep `backup.sql` out of Git if the data is private. A backup that you have never restored is not a backup: this exercise is the test.

</details>

## Quiz

1. What is the advantage of a Compose file compared with several `docker run` commands?

<details>
<summary>Answer</summary>

It describes the whole application in one file that can be kept in Git, starts and stops everything with one command, creates the network and volumes, and applies the same settings every time.

</details>

2. How does `app` find the database, and why does it not use `localhost`?

<details>
<summary>Answer</summary>

Compose creates a project network where each service name is a host name, so `DB_HOST=db` works. `localhost` inside the `app` container is the `app` container itself, not the database.

</details>

3. What does `depends_on` with `condition: service_healthy` add compared with plain `depends_on`?

<details>
<summary>Answer</summary>

Plain `depends_on` only orders the start of the containers. With `service_healthy` Compose waits until the health check of the other service passes, so the database accepts connections before `app` starts.

</details>

4. What is the difference between `docker compose down` and `docker compose down -v`?

<details>
<summary>Answer</summary>

`down` removes containers and the network but keeps named volumes (the data survives). `down -v` removes the volumes too, so the data is lost.

</details>

5. Why is the password passed as a file and not as an environment variable?

<details>
<summary>Answer</summary>

Environment variables can be seen with `docker inspect` and in process information, and are easily copied into logs. A file under `/run/secrets/` is visible only inside the container, and the image and the Git repository never contain it.

</details>

6. A container was killed with `docker kill` and has `restart: on-failure`. Why is it not restarted?

<details>
<summary>Answer</summary>

Docker treats `docker kill` (and `stop`) as a manual stop and does not apply the restart policy. A real crash, where the process ends by itself with a non-zero status, is restarted.

</details>

## Summary

- Compose describes services, networks, volumes and secrets in `compose.yaml` and starts them with `docker compose up -d`; the `name:` prefix keeps all objects of the project together and easy to remove.
- Services reach each other by service name on the project network. Publish only the ports that users need, and bind them to `127.0.0.1` when they are for you alone.
- `depends_on` with `condition: service_healthy` and a `healthcheck` waits for readiness, not just for start.
- Data lives in named volumes: `down` keeps them, `down -v` deletes them. Back up before you delete, and test the restore.
- Passwords belong in secret files and are read by the program from a path (`*_FILE`), not committed and not baked into images.
- A restart policy (`on-failure`, `unless-stopped`) restarts crashed containers, but not manually stopped ones. A good service survives a missing dependency and reconnects.

## Sources and further reading

- [Docker Compose overview](https://docs.docker.com/compose/) — what Compose is and how to use it.
- [Compose file reference](https://docs.docker.com/reference/compose-file/) — every key of `compose.yaml`.
- [Control startup and shutdown order in Compose](https://docs.docker.com/compose/how-tos/startup-order/) — `depends_on` conditions and health checks.
- [Use secrets in Compose](https://docs.docker.com/compose/how-tos/use-secrets/) — how secrets are mounted.
- [Docker Official Image: postgres](https://hub.docker.com/_/postgres) — environment variables such as `POSTGRES_PASSWORD_FILE`, and where the data is stored.
- [Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/) — restart policies in detail.
- [`pg_dump` and `pg_isready`](https://www.postgresql.org/docs/18/app-pgdump.html) — the PostgreSQL 18 manual pages for the backup tool (the `pg_isready` page is [here](https://www.postgresql.org/docs/18/app-pg-isready.html)).

---

[🏠 Start](../START-HERE.md) · [← 11.13 Dockerfiles for Rust programs](13-dockerfiles-for-rust.md) · [📋 Phase 11](README.md) · [11.15 Project: deploy a Rust service →](15-project-deploy-a-rust-service.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
