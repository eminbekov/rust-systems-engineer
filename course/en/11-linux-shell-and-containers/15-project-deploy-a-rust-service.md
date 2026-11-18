<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.14 Docker Compose](14-docker-compose.md) · [📋 Phase 11](README.md) · [11.16 Checkpoint: Stage 2 complete →](16-checkpoint-stage-2-complete.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/15-project-deploy-a-rust-service.md) <!-- nav -->

# Lesson 11.15 — Project: deploy a Rust service

> **Phase 11 — Linux, the shell and containers** · Lesson 15 of 16 · ⏱ about 6 hours (3 sittings)

## What you will learn

- How to put one small Rust service into production in two ways: as a **systemd service in the VM (virtual machine) `dev`**, and as a **Docker image run by Compose**.
- How to write a deploy script that builds, copies, restarts, checks the health and rolls back when the check fails.
- How to compare the two ways: logs, restart after a crash, clean stop, which user runs the program, who can reach it.
- How to check a deployment with one acceptance script instead of by eye.

## Before you start

- You finished [Lesson 11.14 — Docker Compose](14-docker-compose.md) and everything before it in this phase. The project uses the VM `dev` ([Lesson 11.1](01-linux-and-a-virtual-machine.md)) with Rust installed in it ([Lesson 11.5](05-processes-and-signals.md)), services ([Lesson 11.6](06-services-with-systemd.md)), bash scripts ([Lessons 11.8](08-bash-scripting-basics.md) and [11.9](09-robust-bash-scripts.md)), SSH (Secure Shell) with the alias `dev` ([Lesson 11.10](10-ssh-and-remote-servers.md)), and Docker ([Lessons 11.12 to 11.14](12-containers-and-docker-basics.md)).
- You have the service `code/p11-hello` from [Lesson 11.13](13-dockerfiles-for-rust.md). You may use your own web server from [Lesson 06.11](../06-rust-advanced/11-project-multithreaded-web-server.md) instead, if it meets the service contract below.
- The VM `dev` is running, `ssh dev hostname` works, and `rsync` is installed in it (`ssh dev 'sudo apt-get install -y rsync build-essential'`).
- Docker Desktop is running. Plan a few hours for the practice itself. There is **no full solution** here; you write everything, and the "Done when" commands tell you whether it works.

## New words

| Word | Meaning |
|---|---|
| **deploy** | To put a program on a machine so that it runs for users: build or copy it, configure it, start it and check it. |
| **deployment target** | The place where the program runs: here the VM `dev` and a Docker container. |
| **release** | One version of the program that you deploy. |
| **rollback** | Going back to the previous working release after a failed deploy. |
| **smoke test** | A very quick check that the most important thing works (for example `GET /health` answers `200`) right after a deploy. |
| **idempotent** | An action that gives the same result when you repeat it. Running the deploy script twice must be safe. |
| **environment file** | A file of `NAME=value` lines that systemd reads (`EnvironmentFile=`) to set the environment of a service. |
| **log rotation** | Limiting how much log is kept: when a log file reaches a size, the oldest part is dropped. |
| **acceptance script** | A script that checks, with commands, that a system does what the contract says. |

## 1. Two ways to run the same program

You now know two ways to run a service. Before you build both, see what each one gives:

| Question | systemd in the VM | Docker with Compose |
|---|---|---|
| What is deployed? | a binary in `/usr/local/bin`, built on the VM | an image with the binary and its base system |
| Who starts and watches it? | `systemd` (unit file) | the Docker Engine (restart policy) |
| Where are the logs? | the journal: `journalctl -u p11-hello` | `docker compose logs`, kept by the log driver |
| How does it get its settings? | `/etc/p11-hello.env` | `environment:` in `compose.yaml` |
| Which user runs it? | system user `p11hello` | user `app` (uid 10001) in the image |
| Restart after a crash? | `Restart=on-failure` | `restart: unless-stopped` |
| How do you update it? | `deploy.sh`: sync, build, install, restart | build a new image, `docker compose up -d` |
| What isolates it? | service hardening options | the container: own file system, `--read-only`, no capabilities |

Neither is "better". Servers with a few long-lived programs use systemd; teams that ship many services use containers. A systems engineer knows both, and knows how to check that either one is healthy. The project makes you write both and compare them.

**Git for this lesson.** Make **one commit per milestone** (a small step from Lesson 01.3): `git add code/p11-deploy` and `git commit -m "p11-deploy: M3 deploy script"`. Create a branch first: `git switch -c lesson/11-15-deploy`.

## The project

### What you will build

A folder `code/p11-deploy` with:

```text
code/p11-deploy/
  Cargo.toml  Cargo.lock  src/main.rs   <- the service (a copy of p11-hello)
  Dockerfile  .dockerignore             <- deployment 2: the image
  compose.yaml                          <- deployment 2: the stack
  deploy/p11-hello.service              <- deployment 1: the systemd unit
  deploy/deploy.sh                      <- deployment 1: the deploy script
  deploy/check.sh                       <- the acceptance script for both
  README.md                             <- how to run, and the comparison table
```

### Contract

**The service** (your copy of `p11-hello`) must:

- read `BIND` (default `127.0.0.1:8080`) and `GREETING` from the environment;
- answer `GET /` with `<GREETING> from p11-hello` and `GET /health` with `200` and `ok`;
- write one log line per request to standard output, never to a file;
- stop cleanly on `SIGTERM` (log `got a stop signal, bye`, exit status `0`);
- have `cargo test`, `cargo clippy` and `cargo fmt --check` pass.

**Deployment 1: the VM `dev`.**

- The binary is `/usr/local/bin/p11-hello`, owned by root. The service runs as the system user `p11hello`, with no login shell and no home folder.
- Settings are in `/etc/p11-hello.env` (`BIND=127.0.0.1:8081` and `GREETING=Hello (VM)`), readable by root and the group `p11hello`, not by everybody.
- The unit `p11-hello.service` restarts the service after a failure, is enabled at boot, writes to the journal only, and has at least three hardening options (Lesson 11.6).
- Lima forwards loopback ports of the VM to your Mac, so `http://localhost:8081/` on the Mac reaches it (see the Lima documentation on [port forwarding](https://lima-vm.io/docs/config/port/)).

**`deploy/deploy.sh [-n] [HOST]`** runs on the **Mac**; `HOST` is an SSH alias (default `dev`).

| Behaviour | Rule |
|---|---|
| steps | print one line per step: `[1/5] sync …`, `[2/5] build …`, `[3/5] install …`, `[4/5] restart …`, `[5/5] health check` |
| sync | `rsync` the source to `~/deploy/p11-hello` on `HOST`, without `target/`, `.git` and `deploy/` |
| build | `cargo build --release` **on `HOST`** |
| install | keep the old binary as `/usr/local/bin/p11-hello.prev`, then install the new one with `sudo install -m 755` |
| restart | `sudo systemctl restart p11-hello` |
| health check | up to 10 tries, one second apart: `curl -fsS -m 2 http://127.0.0.1:8081/health` **on `HOST`**; success prints `health check ok …` and exits `0` |
| rollback | if no try succeeds: print `health check failed, rolling back` to standard error, put `.prev` back, restart, exit `1` |
| `-n` | dry run: show what `rsync` would send, change nothing else, exit `0` |
| usage | `-h` prints help (exit `0`); an unknown option prints a message to standard error (exit `2`) |
| quality | strict mode (`set -euo pipefail`), quoted variables, ShellCheck clean |

**Deployment 2: Docker with Compose.**

- `docker build` produces an image smaller than 150 MB (megabytes), built in stages, running as a non-root user, with a `HEALTHCHECK`.
- `compose.yaml` (project name `course-deploy`) publishes the service as `127.0.0.1:8082`, sets `BIND=0.0.0.0:8080` and `GREETING` (default `Hello (Docker)`), restarts the container unless you stopped it, has a read-only file system, drops all capabilities, forbids new privileges, and limits the log to 3 files of 1 MB.

**`deploy/check.sh [HOST]`** checks both deployments, prints `ok   - …` or `FAIL - …` for each check, and exits `0` only if all pass.

### Step-by-step plan

#### M0 — Prepare the project

**Goal:** a new folder with a working, tested copy of the service.

Copy `code/p11-hello` to `code/p11-deploy` (without `target/`), create the branch, run the quality gates, and commit. Keep the program name `p11-hello` in `Cargo.toml`.

<details>
<summary>Hint</summary>

`cp -r code/p11-hello code/p11-deploy && rm -rf code/p11-deploy/target code/p11-deploy/Dockerfile.*`. You may keep the `Dockerfile` of Lesson 11.13 and delete the others.

</details>

**Done when:**

```bash
cd ~/projects/rust/code/p11-deploy
cargo fmt --check && cargo clippy --all-targets && cargo test
git log --oneline | head -2
```

shows `test result: ok. 3 passed` and a commit for M0.

#### M1 — Run the service by hand in the VM

**Goal:** prove that the program builds and runs on Linux, before any automation. Every automatic step in M3 is something you did by hand here.

Copy the source to the VM with `rsync` (exclude `target`), build it there, run it with `BIND=127.0.0.1:8081` in the background, call it, and stop it with `SIGTERM`.

<details>
<summary>Hint</summary>

`rsync -a --exclude target ./ dev:~/manual/`, then `ssh dev 'cd ~/manual && PATH=$HOME/.cargo/bin:$PATH cargo build --release'`. A non-interactive `ssh` shell does not load `~/.cargo/env`, so `cargo` is not on the `PATH` unless you add it. Start the program with `BIND=127.0.0.1:8081 ./target/release/p11-hello &` inside the same `ssh` command, and stop it with `pkill p11-hello`.

</details>

**Done when:** one `ssh dev '…'` command that starts the program, runs `curl -s http://127.0.0.1:8081/` and `curl -s http://127.0.0.1:8081/health`, and stops it, prints:

```text
Hello from p11-hello
ok
```

and the log of the program (put it in a file and `cat` it) ends with `got a stop signal, bye`. Remove `~/manual` from the VM afterwards.

#### M2 — Run it as a systemd service

**Goal:** the program runs under its own user, starts at boot, and comes back after a crash.

Do it by hand once: create the user, install the binary from your M1 build, write `/etc/p11-hello.env`, write `deploy/p11-hello.service` in the project, copy it to `/etc/systemd/system/`, reload, enable and start it.

<details>
<summary>Hint</summary>

The commands of Lesson 11.6 apply: `useradd --system --no-create-home --shell /usr/sbin/nologin`, `install -m 755`, `systemd-analyze verify`, `daemon-reload`, `enable --now`. Use `EnvironmentFile=/etc/p11-hello.env` and set the file to mode `640`, owner `root:p11hello`. The unit needs `User=`, `Group=`, `ExecStart=`, `Restart=on-failure`, `RestartSec=2`, and options such as `NoNewPrivileges=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, `PrivateTmp=yes`. A service that listens on a port does not need write access to anything. Check the unit with `systemd-analyze verify` **after** the binary is installed, or it complains that the program is missing.

</details>

**Done when:**

```bash
ssh dev 'systemctl is-active p11-hello; systemctl is-enabled p11-hello'
ssh dev 'ps -o user=,comm= -C p11-hello'
ssh dev 'curl -s http://127.0.0.1:8081/'
ssh dev 'ls -l /etc/p11-hello.env'
```

```text
active
enabled
p11hello p11-hello
Hello (VM) from p11-hello
-rw-r----- 1 root p11hello 40 Nov 18 08:13 /etc/p11-hello.env
```

(The size and the date differ.) Then prove the restart policy and the clean stop:

```bash
ssh dev 'systemctl show -p MainPID --value p11-hello'
ssh dev 'sudo systemctl kill -s SIGKILL p11-hello; sleep 3.5; systemctl show -p MainPID --value p11-hello; systemctl is-active p11-hello'
ssh dev 'sudo systemctl stop p11-hello; sudo journalctl -u p11-hello -n 3 -o cat; sudo systemctl start p11-hello'
```

The first command prints a process number, for example `676`. The second prints a **different** number and `active`: systemd restarted the process that you killed. The third shows `got a stop signal, bye` and `Stopped p11-hello.service - …` in the journal: a clean stop, not a failure.

#### M3 — The deploy script

**Goal:** one command updates the VM, and a broken release does not stay.

Write `deploy/deploy.sh` according to the contract. Develop it in this order: options and usage; the `-n` dry run; sync and build; install; restart; the health check loop; the rollback. Run `shellcheck` often (Lesson 11.9).

<details>
<summary>Hint 1 — structure</summary>

```text
getopts for -n and -h  ->  host=${1:-dev}
src_dir=$(cd "$(dirname "$0")/.." && pwd)        # the project folder, wherever you run the script from
step() { printf '[%s] %s\n' "$1" "$2"; }
rsync flags in an array, plus -n -v when dry-running
ssh "$host" '…' for build, install, restart
for attempt in 1 … 10: if answer=$(ssh "$host" "curl …"); then success
rollback: only reached when the loop ended without success
```

</details>

<details>
<summary>Hint 2 — the traps</summary>

`rsync dir/ host:path/` (with slashes) copies the **contents**; `rsync` does not create missing parent folders, so run `ssh "$host" mkdir -p …` first (skip it for `-n` if you want a dry run that changes nothing at all). The old binary must be copied to `.prev` **before** the new one is installed. `ssh` runs the command in the user's home folder, so a relative remote path such as `deploy/p11-hello` is fine. Variables inside a double-quoted `ssh "…"` command expand **on the Mac** (ShellCheck says SC2029; that is what you want here, so silence it for the file with a comment and a reason). A `$` for the remote side needs a backslash: `\$HOME`. With `set -e`, a failing `ssh` for the health check would stop the script, so put it in an `if` or `answer=$(…)` condition. The rollback must also work when `.prev` does not exist (first deploy).

</details>

**Done when:**

```bash
deploy/deploy.sh -n dev; echo "exit=$?"
```

prints the `[1/5]` line, the list of files that `rsync` would send (`Cargo.toml`, `src/main.rs`, … but **not** `target/` or `deploy/`), ends with a line saying that nothing was built or changed, and `exit=0`. The service has the same `MainPID` before and after.

```bash
deploy/deploy.sh dev; echo "exit=$?"
```

```text
[1/5] sync source to dev:deploy/p11-hello
[2/5] build on dev
[3/5] install the binary (the old one is kept as p11-hello.prev)
[4/5] restart the service
[5/5] health check
health check ok after 1 attempt(s): ok
exit=0
```

(Your wording may differ, but the five numbered lines and `health check ok` must be there.) Run it a **second** time: it must succeed again, with a new `MainPID`. Then break a release on purpose: make `/health` answer status `500`, run the script, and restore the source afterwards:

```text
[5/5] health check
deploy.sh: health check failed, rolling back
exit=1
```

After that, `ssh dev 'curl -s -w "[%{http_code}]\n" http://127.0.0.1:8081/health'` still prints `ok` and `[200]`: the old release is running again. Also run `docker run --rm -v "$PWD/deploy":/mnt koalaman/shellcheck:stable deploy.sh`: no output.

#### M4 — The Docker image

**Goal:** a small, non-root image that checks its own health.

Use the multi-stage `Dockerfile` of Lesson 11.13 (`debian:trixie-slim`, or distroless if you prefer), with the dependency layer and `.dockerignore`.

<details>
<summary>Hint</summary>

`COPY Cargo.toml Cargo.lock ./`, a dummy `main.rs` build, `COPY src ./src`, `touch src/main.rs`, then a runtime stage with `useradd`, `USER`, `ENV BIND=0.0.0.0:8080`, `HEALTHCHECK … ["/usr/local/bin/p11-hello", "--check"]` and `CMD` in **exec form**. The `--check` option exists in `p11-hello` already; if you use your own server, give it a way to check itself or install a tool for the health check.

</details>

**Done when:**

```bash
docker build -t course-deploy:1 .
docker image ls course-deploy
docker run -d --name course-deploy-test -p 127.0.0.1:8082:8080 course-deploy:1
sleep 13; docker ps --filter name=course-deploy-test --format '{{.Status}}'
docker inspect course-deploy-test --format 'user={{.Config.User}}'
curl -s http://localhost:8082/
SECONDS=0; docker stop course-deploy-test; echo "stop took ${SECONDS}s"
docker inspect course-deploy-test --format 'exit={{.State.ExitCode}}'
docker rm course-deploy-test
```

```text
IMAGE             ID             DISK USAGE   CONTENT SIZE   EXTRA
course-deploy:1   c14d811a7bf7        141MB         30.4MB
Up 13 seconds (healthy)
user=app
Hello from p11-hello
stop took 0s
exit=0
```

(The `IMAGE`/`ID` line is the real output of `docker image ls`; the ID differs. The size must be under 150 MB, the status `healthy`, the user not empty or `root`, the stop under 3 seconds with exit `0`. Note that the service answers `Hello from p11-hello`, because without `GREETING` the default is `Hello`.)

#### M5 — Compose, restart and logs

**Goal:** the second deployment as a file, with a restart policy, hardening and limited logs.

Write `compose.yaml` according to the contract and bring it up. To test the restart policy you need a **real crash**, because `docker kill` is a manual stop. Add the training-only route from Lesson 11.14 to a **temporary** copy of the service, or to your working tree, and remove it before you commit.

<details>
<summary>Hint</summary>

Keys you need: `name: course-deploy`, `services: app: build: .`, `image: course-deploy:1`, `environment:` (use `${GREETING:-Hello (Docker)}`), `ports: - "127.0.0.1:8082:8080"`, `restart: unless-stopped`, `read_only: true`, `cap_drop: [ALL]`, `security_opt: ["no-new-privileges:true"]`, and `logging: driver: json-file` with `options: max-size: "1m"` and `max-file: "3"`. Check the file with `docker compose config --quiet` first. The crash route: `if request_line.starts_with("GET /crash ") { std::process::exit(1); }` before the routing.

</details>

**Done when:**

```bash
docker compose up -d --build --wait
curl -s http://localhost:8082/
docker compose ps --format 'table {{.Service}}\t{{.Status}}\t{{.Ports}}'
docker inspect course-deploy-app-1 --format '{{.HostConfig.LogConfig}} restart={{.HostConfig.RestartPolicy.Name}} ro={{.HostConfig.ReadonlyRootfs}} user={{.Config.User}}'
```

```text
Hello (Docker) from p11-hello
SERVICE   STATUS                    PORTS
app       Up 10 seconds (healthy)   127.0.0.1:8082->8080/tcp
{json-file map[max-file:3 max-size:1m]} restart=unless-stopped ro=true user=app
```

Then, with the temporary crash route in the image:

```bash
curl -sS -m 3 http://localhost:8082/crash
sleep 3
docker compose ps --format 'table {{.Service}}\t{{.Status}}'
docker inspect course-deploy-app-1 --format 'restarts={{.RestartCount}}'
docker compose logs --no-log-prefix app | tail -3
SECONDS=0; docker compose stop app; echo "stop took ${SECONDS}s"
docker compose logs --no-log-prefix app | tail -1
```

```text
curl: (52) Empty reply from server
SERVICE   STATUS
app       Up 2 seconds (health: starting)
restarts=1
GET / HTTP/1.1 -> 200
crashing on purpose
listening on 0.0.0.0:8080
stop took 0s
got a stop signal, bye
```

Remove the crash route, rebuild, and commit.

#### M6 — The acceptance script and the comparison

**Goal:** one script that checks both deployments, and a README that explains them.

Write `deploy/check.sh`. It runs on the Mac, asks the VM through `ssh`, and asks Docker locally. Check at least: the unit is `active` and `enabled`; `/health` answers `200` in the VM; the process runs as `p11hello`; the container is `healthy`; `/health` answers `200` on `localhost:8082`; the container user is not root; the container file system is read-only. Then write `README.md` with the commands to run each deployment and the comparison table of Section 1, filled in with **what you observed** (for example how long the restart took in each case).

<details>
<summary>Hint</summary>

Reuse the `check` function of `backup.sh`'s tests (Lesson 11.9). Useful building blocks: `ssh "$host" systemctl is-active p11-hello`, `ssh "$host" curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8081/health`, `ssh "$host" ps -o user= -C p11-hello | tr -d ' '`, `docker inspect NAME --format '{{.State.Health.Status}}'`, `docker inspect NAME --format '{{.Config.User}}'`, `docker inspect NAME --format '{{.HostConfig.ReadonlyRootfs}}'`. Note that `docker top` needs the `pid` column in its format, so `-eo user` alone fails.

</details>

**Done when:** with both deployments running, `deploy/check.sh dev; echo "exit=$?"` prints (the four lines of the VM part and the four lines of the Docker part, in your order and words):

```text
ok   - VM: service is active
ok   - VM: service is enabled
ok   - VM: /health answers 200
ok   - VM: runs as p11hello
ok   - Docker: container is healthy
ok   - Docker: /health answers 200
ok   - Docker: runs as user app
ok   - Docker: root file system is read-only
exit=0
```

With `sudo systemctl stop p11-hello` in the VM (or `docker compose stop`) the matching lines become `FAIL` and the exit status is `1`. Restart them afterwards. ShellCheck must be clean on `check.sh` too.

### Stretch goals

- **Version in the answer.** Add `GET /version` with `env!("CARGO_PKG_VERSION")` (Lesson 11.13) and let `deploy.sh` print the version it deployed.
- **No build on the VM.** Build the release inside the `rust:1.100` image on the Mac, copy only the binary with `scp`, and compare the time with M3. Check that the glibc of the VM is at least as new as that of the build image.
- **`systemd-analyze security p11-hello`** in the VM shows an "exposure level". Add hardening options until the level is below 5, and note which ones broke the service.
- **Deploy the image to the VM** (needs Docker in the VM, `limactl start template:docker` in a second instance): `docker save course-deploy:1 | ssh dev docker load`.
- **A timer for a check.** A systemd `.timer` unit in the VM that runs the health check every minute and writes the result to the journal (a timer is one of the unit types of Lesson 11.6; read `systemd.timer(5)`).
- **Zero downtime.** Start the new release on port 8083, health-check it, then switch; think about what a reverse proxy would give you.

## Common mistakes

1. **`cargo: command not found` in a script that works in your terminal.** Non-interactive `ssh` does not load `~/.cargo/env`. *Fix:* add `$HOME/.cargo/bin` to `PATH` in the remote command.
2. **The deploy script succeeds but the old version still runs.** The build copied to the wrong place, or the service was not restarted. *Fix:* print the version or a build id in the answer and check it as part of the health check.
3. **A deploy script that cannot be run twice.** `mkdir` without `-p`, a `.prev` copy that fails when the file does not exist. *Fix:* make every step idempotent.
4. **`rsync --delete` with the wrong target** deletes real files on the VM. *Fix:* the target is always inside `~/deploy/p11-hello`; use `-n` first.
5. **The health check passes against the old process.** The check ran before the restart finished. *Fix:* retry with a pause, and restart **before** checking; compare the `MainPID`.
6. **Secrets in `/etc/p11-hello.env` readable by everybody** or committed to Git. *Fix:* mode `640`, group of the service; never commit real values (use an `.example` file).
7. **Running the container as root, or with the loopback bind.** *Fix:* `USER app`, `BIND=0.0.0.0:8080` inside, `127.0.0.1:8082` on the host.
8. **Testing the restart policy with `docker kill`.** It counts as a manual stop. *Fix:* use a real crash.
9. **Leaving the `/crash` route in.** *Fix:* remove it before the commit, and check `git diff`.
10. **Forgetting to clean up.** *Fix:* `docker compose down`, `docker image rm course-deploy:1`, and in the VM `sudo systemctl disable --now p11-hello`, remove the unit, the env file, the binaries and the user (`sudo userdel p11hello`) when you finish the lesson, unless you keep it as a playground.

## Quiz

1. Why does `deploy.sh` keep the old binary as `.prev`?

<details>
<summary>Answer</summary>

So that a failed health check can put the last working release back at once, without rebuilding, and the service keeps running the old version.

</details>

2. What does the health check prove, and what can it not prove?

<details>
<summary>Answer</summary>

It proves that the process answers on its port after the restart. It does not prove that everything works (databases, other services), and a check from inside the VM or container does not prove that users can reach the published port.

</details>

3. Why must the deploy script be safe to run twice?

<details>
<summary>Answer</summary>

Deploys are repeated, retried after errors, and run by tired people. A script that fails or breaks something the second time (a missing folder, a copy without `-p`) makes the next incident worse. Idempotent steps (`mkdir -p`, `rsync`, `install`) give the same result every time.

</details>

4. Which restart policy settings do the two deployments use, and why does `docker kill` not test the Docker one?

<details>
<summary>Answer</summary>

systemd uses `Restart=on-failure`; Compose uses `restart: unless-stopped`. `docker kill` and `docker stop` count as manual stops, which the policy respects by design, so a real crash (the process exits by itself) is the correct test. (For systemd, `systemctl kill -s SIGKILL` works, because systemd sees an unexpected exit.)

</details>

5. Name three differences in how the two deployments are isolated.

<details>
<summary>Answer</summary>

The systemd service shares the VM's file system and kernel and is restricted by user and unit options (`ProtectSystem`, `NoNewPrivileges`, …). The container has its own file system from the image (read-only here), its own process and network view, dropped capabilities, and resource limits. Both run as a non-root user.

</details>

## Summary

- A deployment has five jobs: get the code or image there, configure it, start it, check it, and be able to go back.
- The systemd way: binary in `/usr/local/bin`, user and env file, a unit with `Restart=` and hardening, logs in the journal, a deploy script over SSH with sync, build, install, restart, health check and rollback.
- The Docker way: a multi-stage non-root image with a health check, a Compose file with a restart policy, a read-only file system, dropped capabilities and rotated logs.
- Repeatable steps (`rsync`, `install`, `mkdir -p`) make a deploy script idempotent. A `.prev` copy gives a cheap rollback.
- Test restart policies with a real crash, test stops with `SIGTERM`, and check both with a script, not by eye.
- Comparing two ways of doing the same job shows what each one costs and what it gives.

## Sources and further reading

- [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) — all settings of a service unit, including `Restart=` and `EnvironmentFile=`.
- [systemd.exec(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html) — the hardening options (`ProtectSystem=`, `NoNewPrivileges=`, …).
- [rsync manual page](https://man7.org/linux/man-pages/man1/rsync.1.html) — `-a`, `--delete`, `--exclude`, `-n`.
- [Compose file reference: services](https://docs.docker.com/reference/compose-file/services/) — `restart`, `read_only`, `cap_drop`, `logging`, `healthcheck`.
- [Docker logging drivers: json-file](https://docs.docker.com/engine/logging/drivers/json-file/) — log rotation options.
- [Lima: port forwarding](https://lima-vm.io/docs/config/port/) — how VM ports reach your Mac.
- [The Twelve-Factor App](https://12factor.net/) — configuration, logs and disposability as deployment principles.

---

[🏠 Start](../START-HERE.md) · [← 11.14 Docker Compose](14-docker-compose.md) · [📋 Phase 11](README.md) · [11.16 Checkpoint: Stage 2 complete →](16-checkpoint-stage-2-complete.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
