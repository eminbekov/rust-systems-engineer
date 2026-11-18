<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.5 Processes and signals](05-processes-and-signals.md) · [📋 Phase 11](README.md) · [11.7 Package managers →](07-package-managers.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/06-services-with-systemd.md) <!-- nav -->

# Lesson 11.6 — Services with systemd

> **Phase 11 — Linux, the shell and containers** · Lesson 6 of 16 · ⏱ about 120 minutes

## What you will learn

- What a service is and what `systemd` does: starting, stopping, restarting and supervising programs.
- How to write a *unit file* for your own Rust program and run it as a service under its own user.
- How to control it with `systemctl` and read its logs with `journalctl`.
- How a restart policy brings a crashed service back, and how a few lines of hardening limit the damage a bug can do.

## Before you start

- You finished [Lesson 11.5 — Processes and signals](05-processes-and-signals.md). You built `p11-heartbeat` in `~/code/p11-heartbeat` inside the VM, with `cargo build --release`. If you did not, go back: this lesson uses that program.
- You know the system user from [Lesson 11.4](04-users-groups-and-permissions.md), the folders `/etc` and `/usr/local/bin` from [Lesson 11.2](02-the-linux-file-system.md), and `sudo tee` is the idea of `tee` from [Lesson 11.3](03-text-tools-pipes-and-redirection.md).
- All commands run inside the VM (`limactl shell dev`). If you only have Docker, Section 8 shows a systemd container.

## New words

| Word | Meaning |
|---|---|
| **systemd** | The first process (PID 1; a PID, process ID, is the number of a process, Lesson 11.5) on most modern Linux systems. It starts the system, starts and supervises services, and collects their logs. |
| **service** | A program that runs in the background for a long time, started and watched by the system, such as a web server or a database. |
| **unit** | Anything systemd manages. It is described by a *unit file*. Types: `.service`, `.timer`, `.target`, `.socket` … |
| **unit file** | A small text file in INI format (named after the *initialization* files of old systems: `[Section]` and `Key=value`) that says what to run and how. |
| **target** | A group of units that stands for a state of the system, such as `multi-user.target` ("the system is up and ready for users"). |
| **enable** | Make a service start automatically at boot. (`start` only starts it now.) |
| **journal** | The central log of systemd. `journalctl` reads it. Everything a service writes to stdout and stderr goes there. |
| **restart policy** | The rule that says when systemd starts a service again after it ended: never, on failure, always. |
| **cgroup** (control group) | A Linux feature that groups processes so that the system can track them, limit their resources and stop them all together. systemd puts every service into its own cgroup. |
| **hardening** | Settings that reduce what a program is allowed to do, so that a bug or an attacker can do less damage. |

## 1. What systemd does

When the Linux kernel has started, it runs one program as **PID 1**. On Ubuntu it is `systemd`. PID 1 starts everything else, in the right order, and it
**adopts orphans** and **collects exit statuses** ([Lesson 11.5](05-processes-and-signals.md)). For long-running programs it gives you:

- start at boot, in order of dependencies;
- restart after a crash;
- one place for logs and one command to see the state;
- a separate user, limits and protection for each service;
- a clean stop: it sends `SIGTERM`, waits, and sends `SIGKILL` only when the program does not obey.

That last point is why Lesson 11.5 taught signals first: the program you wrote already shuts down cleanly on `SIGTERM`.

Look at what runs on your VM (the list in your VM is longer):

```bash
ps -o pid,cmd -p 1
systemctl list-units --type=service --state=running --no-pager
```

```text
    PID CMD
      1 /sbin/init
  UNIT                     LOAD   ACTIVE SUB     DESCRIPTION
  dbus.service             loaded active running D-Bus System Message Bus
  getty@tty1.service       loaded active running Getty on tty1
  systemd-journald.service loaded active running Journal Service
  systemd-logind.service   loaded active running User Login Management
```

**What happens:** `/sbin/init` is a link to `systemd`. `systemctl` ("system control") is the command that talks to systemd. `list-units` shows loaded units; `--type=service` keeps services; `--state=running` keeps the running ones.
`LOAD` says if the unit file was read, `ACTIVE` is the high-level state, `SUB` is the detailed state.

## 2. Prepare: a user and a place for the program

A service should live in a predictable place and run as its own user. Create the system user from Lesson 11.4 and *install* the binary you built in Lesson 11.5:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin heartbeat
sudo install -m 755 ~/code/p11-heartbeat/target/release/p11-heartbeat /usr/local/bin/p11-heartbeat
ls -l /usr/local/bin/p11-heartbeat
```

```text
-rwxr-xr-x 1 root root 571376 Nov 18 07:06 /usr/local/bin/p11-heartbeat
```

**What happens:**

1. `useradd --system ...` creates the user `heartbeat` that nobody can log in as.
2. `install -m 755 source target` copies a file and sets its mode in one step (a safe alternative to `cp` plus `chmod`).
3. The file is owned by **root**, not by `heartbeat`: the service user can run the program but cannot change it. `/usr/local/bin` is the FHS (Filesystem Hierarchy Standard) place for programs you install yourself ([Lesson 11.2](02-the-linux-file-system.md)). The size depends on your build.

> 🔒 **Security:** The program belongs to root and the process runs as `heartbeat`. If an attacker takes over the service, they cannot overwrite the program to make it permanent. Keep **code owned by an administrator, data owned by the service user**.

## 3. The unit file

Unit files that you write belong to `/etc/systemd/system/`. (Files in `/usr/lib/systemd/system/` come from packages; do not edit those.) Create `heartbeat.service`:

```bash
sudo tee /etc/systemd/system/heartbeat.service > /dev/null <<'EOF'
[Unit]
Description=Heartbeat demo service from the course
After=network.target

[Service]
User=heartbeat
Group=heartbeat
Environment=HEARTBEAT_NAME=systemd
Environment=HEARTBEAT_SECONDS=2
ExecStart=/usr/local/bin/p11-heartbeat
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=2
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
EOF
systemd-analyze verify /etc/systemd/system/heartbeat.service
echo "verify exit status: $?"
```

```text
verify exit status: 0
```

**What happens:**

1. `sudo tee file > /dev/null <<'EOF' ... EOF` writes the text into a file that only root may create. We use `tee` and not `sudo cat > file` because the redirection `>` is done by **your** shell *before* `sudo` starts, and your shell has no right to write there: the result would be `Permission denied`. With `tee`, the program that writes the file runs as root. `> /dev/null` hides the copy that `tee` prints to the screen.
2. `systemd-analyze verify` checks the unit file and prints nothing when it is fine.

The file has three sections:

| Section | Key | Meaning |
|---|---|---|
| `[Unit]` | `Description=` | text shown in `status` and in the logs |
| | `After=network.target` | start after the network is configured (ordering only, not a requirement) |
| `[Service]` | `User=`, `Group=` | run as this user and group |
| | `Environment=` | set an environment variable for the process (the program reads `HEARTBEAT_NAME`) |
| | `ExecStart=` | the command to run: an **absolute path**, no shell, no pipes |
| | `ExecReload=` | the command for `systemctl reload`: here it sends `SIGHUP` to the main process (`$MAINPID`) |
| | `Restart=on-failure`, `RestartSec=2` | restart after a failure, wait 2 seconds before it |
| | `NoNewPrivileges=`, `ProtectSystem=strict`, `ProtectHome=`, `PrivateTmp=` | hardening (Section 7) |
| `[Install]` | `WantedBy=multi-user.target` | when the service is *enabled*, start it as part of normal system start-up |

By default `Type=simple`: systemd considers the service started as soon as it has launched `ExecStart`, and the program must stay in the **foreground** (no daemonizing). Our program does exactly this.

> 💡 **Tip:** Because systemd collects stdout and stderr, a service does **not** need to write its own log files. It just prints lines with `println!` / `eprintln!`, and the journal does the rest ([Lesson 09.10](../09-testing-debugging-benchmarking/10-logging-and-tracing.md)). This is also the rule of the twelve-factor app ([Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md)).

## 4. Start, stop, status

systemd reads unit files when it starts and when you tell it. After every change of a unit file, run `daemon-reload`:

```bash
sudo systemctl daemon-reload
systemctl status heartbeat --no-pager
sudo systemctl start heartbeat
systemctl status heartbeat --no-pager
```

```text
○ heartbeat.service - Heartbeat demo service from the course
     Loaded: loaded (/etc/systemd/system/heartbeat.service; disabled; preset: enabled)
     Active: inactive (dead)
● heartbeat.service - Heartbeat demo service from the course
     Loaded: loaded (/etc/systemd/system/heartbeat.service; disabled; preset: enabled)
     Active: active (running) since Wed 2026-11-18 07:06:43 UTC; 15s ago
 Invocation: f9db13e9be7a457ba0d53b32e9a178b4
   Main PID: 148 (p11-heartbeat)
      Tasks: 2 (limit: 9563)
     Memory: 2.8M (peak: 2.8M)
        CPU: 5ms
     CGroup: /system.slice/heartbeat.service
             └─148 /usr/local/bin/p11-heartbeat

Nov 18 07:06:43 lima-dev systemd[1]: Started heartbeat.service - Heartbeat demo service from the course.
Nov 18 07:06:43 lima-dev p11-heartbeat[148]: starting: pid 148, greeting systemd, every 2 s
Nov 18 07:06:43 lima-dev p11-heartbeat[148]: heartbeat 1 from systemd
Nov 18 07:06:45 lima-dev p11-heartbeat[148]: heartbeat 2 from systemd
```

(The times, PIDs and the long ID will be different on your VM.) **How to read `status`:**

1. The first symbol: `○` inactive, `●` active, `×` failed (red/green in a colour terminal).
2. `Loaded:` shows where the unit file is, whether it is `enabled` (starts at boot) or `disabled`.
3. `Active:` is the current state and since when. `active (running)` is the healthy state of a service.
4. `Main PID`, `Tasks` (threads and processes), `Memory`, `CPU` are live numbers from the service's **cgroup**. `CGroup:` lists all processes of the service. Our program has two tasks: the main thread and the worker thread.
5. The last lines are the recent log lines from the journal.

The commands you will use daily:

| Command | What it does |
|---|---|
| `sudo systemctl start NAME` | start now |
| `sudo systemctl stop NAME` | stop now (sends `SIGTERM`, then `SIGKILL` after a timeout) |
| `sudo systemctl restart NAME` | stop and start |
| `sudo systemctl reload NAME` | run `ExecReload` (here: `SIGHUP`, without a restart) |
| `sudo systemctl enable NAME` | start at boot (`enable --now` also starts it now) |
| `sudo systemctl disable NAME` | do not start at boot |
| `systemctl status NAME` | state plus the last log lines |
| `systemctl is-active NAME` | prints `active` or `inactive`; the exit status is 0 only for `active` |
| `systemctl is-enabled NAME` | prints `enabled` or `disabled` |
| `systemctl cat NAME` | show the unit file that systemd uses |
| `systemctl --failed` | list the units that failed |

`NAME` can be written with or without `.service`. Reading (`status`, `is-active`) does not need `sudo`; changing the state does.

Try `reload`, `stop` and `enable`:

```bash
sudo systemctl reload heartbeat
sudo systemctl stop heartbeat
systemctl is-active heartbeat
sudo systemctl enable heartbeat
ls -l /etc/systemd/system/multi-user.target.wants/heartbeat.service
systemctl is-enabled heartbeat
sudo systemctl disable heartbeat
```

```text
Reloaded heartbeat.service - Heartbeat demo service from the course.
inactive
Created symlink '/etc/systemd/system/multi-user.target.wants/heartbeat.service' → '/etc/systemd/system/heartbeat.service'.
lrwxrwxrwx 1 root root 37 Nov 18 07:07 /etc/systemd/system/multi-user.target.wants/heartbeat.service -> /etc/systemd/system/heartbeat.service
enabled
Removed '/etc/systemd/system/multi-user.target.wants/heartbeat.service'.
```

**What happens:** `enable` is only a **symbolic link** ([Lesson 11.2](02-the-linux-file-system.md)) from the "wants" folder of the target named in `WantedBy=` to your unit file. At boot, systemd starts everything linked in `multi-user.target.wants/`. `disable` removes the link. `start`/`stop` and `enable`/`disable` are independent: a service can be running and disabled, or stopped and enabled.

## 5. Logs with `journalctl`

Start the service again, wait about twenty seconds, and look at its log (the numbers you see depend on the time you waited):

```bash
sudo systemctl start heartbeat
journalctl -u heartbeat -n 5 --no-pager
journalctl -u heartbeat --since "1 minute ago" --no-pager -o cat | tail -n 2
```

```text
Nov 18 07:06:49 lima-dev p11-heartbeat[148]: heartbeat 4 from systemd
Nov 18 07:06:51 lima-dev p11-heartbeat[148]: heartbeat 5 from systemd
Nov 18 07:06:53 lima-dev p11-heartbeat[148]: heartbeat 6 from systemd
Nov 18 07:06:55 lima-dev p11-heartbeat[148]: heartbeat 7 from systemd
Nov 18 07:06:57 lima-dev p11-heartbeat[148]: heartbeat 8 from systemd
heartbeat 7 from systemd
heartbeat 8 from systemd
```

**What happens:**

1. `-u heartbeat` selects the unit. Every line shows the time, the host name, the program name with its PID and the text that the program printed.
2. `-n 5` shows the last five lines; `--no-pager` prints directly instead of opening `less`.
3. `--since "1 minute ago"` filters by time (also `--until`, or dates like `"2026-11-18 07:00"`). `-o cat` prints only the message, without the time and host.

Useful forms: `journalctl -u heartbeat -f` follows new lines live (stop with `Ctrl + C`, like `tail -f`); `journalctl -b` shows the current boot only; `journalctl -p err` shows only errors and worse; `journalctl -u heartbeat -o json` prints structured fields.

> 💡 **Tip:** If `journalctl` says it cannot read the journal, your user is not in the group `adm` or `systemd-journal`; use `sudo journalctl ...`. In the Lima VM your user is allowed to read it.

## 6. Restart policy: the crashed service comes back

Now the reason to use a supervisor. Kill the service with `SIGKILL`, the way a crash looks to systemd:

```bash
PID=$(systemctl show -p MainPID --value heartbeat)
echo "main pid: $PID"
sudo kill -9 $PID
sleep 4
systemctl show -p NRestarts -p MainPID heartbeat
journalctl -u heartbeat --since "-10s" --no-pager | grep -v 'heartbeat [0-9]'
```

```text
main pid: 148
MainPID=188
NRestarts=1
Nov 18 07:06:59 lima-dev systemd[1]: heartbeat.service: Main process exited, code=killed, status=9/KILL
Nov 18 07:06:59 lima-dev systemd[1]: heartbeat.service: Failed with result 'signal'.
Nov 18 07:07:02 lima-dev systemd[1]: heartbeat.service: Scheduled restart job, restart counter is at 1.
Nov 18 07:07:02 lima-dev systemd[1]: Started heartbeat.service - Heartbeat demo service from the course.
Nov 18 07:07:02 lima-dev p11-heartbeat[188]: starting: pid 188, greeting systemd, every 2 s
```

**What happens:**

1. `systemctl show -p MainPID --value NAME` prints just the PID of the main process; `kill -9` ends it. The `grep -v` hides the ordinary heartbeat lines. The service had no chance to clean up: the log shows no `bye`.
2. systemd noticed: `code=killed, status=9/KILL`, "Failed with result 'signal'". Because of `Restart=on-failure`, it scheduled a restart after `RestartSec=2`, and started a **new process with a new PID** (188 here). `NRestarts=1` counts it.
3. A normal `systemctl stop` is not a failure, so it is not restarted. A clean exit (status 0) is not a failure either.

| `Restart=` | Restart when |
|---|---|
| `no` (default) | never |
| `on-failure` | exit status not 0, killed by an unclean signal (`SIGHUP`, `SIGINT`, `SIGTERM` and `SIGPIPE` count as clean), or a timeout |
| `always` | always, even after a clean exit, unless you used `systemctl stop` |

systemd refuses to restart a service forever if it crashes immediately again and again (defaults: five starts in ten seconds, then it gives up and shows `failed` with "start-limit-hit"). This protects the machine from a restart loop. You fix the bug; then `sudo systemctl reset-failed NAME` and start again.

## 7. Hardening: give the service less

`NoNewPrivileges=yes` stops the process and its children from gaining rights (for example through a setuid program). `ProtectSystem=strict` makes the whole file system read-only for the service (except `/dev`, `/proc`, `/sys`), `ProtectHome=yes` hides `/home`, `/root` and `/run/user`, and `PrivateTmp=yes` gives the service its own empty `/tmp`. systemd can score a unit:

```bash
systemd-analyze security heartbeat.service --no-pager | tail -n 1
```

```text
→ Overall exposure level for heartbeat.service: 8.3 EXPOSED :-(
```

**What happens:** the command checks about 80 settings; the lower the number, the less the service can do. 8.3 is "EXPOSED" even with four directives, because the many other doors are still open. Add these lines to the `[Service]` section (for example with `sudo nano /etc/systemd/system/heartbeat.service`, or by writing the file again), then reload and restart:

```text
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_UNIX
RestrictNamespaces=yes
SystemCallFilter=@system-service
UMask=0077
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart heartbeat
systemctl is-active heartbeat
systemd-analyze security heartbeat.service --no-pager | tail -n 1
```

```text
active
→ Overall exposure level for heartbeat.service: 4.2 OK :-)
```

**What happens:** the service still works and the exposure fell from 8.3 to 4.2. What each line does: no access to device files (`PrivateDevices`), the kernel's settings, modules and cgroups are read-only, only Unix-socket networking is allowed (`RestrictAddressFamilies=AF_UNIX`; our program needs no network), no new namespaces, only the system calls that normal services need (`SystemCallFilter=@system-service`), and new files are private (`UMask=0077`).
If you also add an empty `CapabilityBoundingSet=` (drop all special rights) the score falls to 2.1 and this program still works.

> 🔒 **Security:** Harden step by step: add one or two lines, restart, read the log. A too-strict setting shows as a failure at start-up (`status=...`) or in the log, and you remove that one line. For a service that opens a network port (the project in Lesson 11.15, and the services of later phases), allow `AF_INET` and `AF_INET6`.

## 8. If you only have Docker

A container normally does not run systemd. To follow this lesson without the Lima VM, build a throw-away image. The names start with `course-`; remove everything at the end. Save this as `Dockerfile` in an empty folder:

```text
FROM ubuntu:26.04
RUN apt-get update \
 && apt-get install -y --no-install-recommends systemd systemd-sysv dbus sudo procps \
 && rm -rf /var/lib/apt/lists/*
STOPSIGNAL SIGRTMIN+3
CMD ["/sbin/init"]
```

```bash
docker build -t course-systemd .
docker run -d --name course-systemd-lab --privileged course-systemd
docker exec -it course-systemd-lab bash
```

Inside the container you are `root`, so leave out `sudo`. To get a Linux binary of `p11-heartbeat`, build it with the official Rust image from the project folder on your Mac, and copy it in:

```bash
cd ~/projects/rust/code/p11-heartbeat
docker run --rm -v "$PWD":/w -w /w -e CARGO_TARGET_DIR=/w/target-linux rust:1.100-slim cargo build --release
docker cp target-linux/release/p11-heartbeat course-systemd-lab:/tmp/p11-heartbeat
```

Inside the container use `/tmp/p11-heartbeat` as the file for `install` in Section 2. When you finish: `docker rm -f course-systemd-lab` and `docker rmi course-systemd`. `--privileged` gives the container wide rights on the Docker VM, so use it only for such a lab and only with this image.

## 9. Clean up

```bash
sudo systemctl stop heartbeat
sudo rm /etc/systemd/system/heartbeat.service
sudo rm /usr/local/bin/p11-heartbeat
sudo systemctl daemon-reload
sudo userdel heartbeat
```

Keep `~/code/p11-heartbeat` — a later lesson reuses the source. (Do not forget to remove the service user after you delete the unit, or the next lesson's `useradd` fails with "already exists".)

## Common mistakes

1. **No `daemon-reload` after editing the unit.** `status` shows `Warning: The unit file, source configuration file or drop-ins of heartbeat.service changed on disk. Run 'systemctl daemon-reload' to reload units.` *Fix:* run `sudo systemctl daemon-reload`, then restart.
2. **`status=203/EXEC`.** The path in `ExecStart=` is wrong or the file is not executable: `Failed at step EXEC spawning /usr/local/bin/does-not-exist: No such file or directory`. *Fix:* use the absolute path, check with `ls -l`.
3. **`status=217/USER`.** The user in `User=` does not exist: `Failed to determine credentials for user 'heartbeat': Unknown user`. *Fix:* create the user first (Lesson 11.4).
4. **`sudo echo ... > /etc/...`.** `Permission denied`, because your shell does the redirection. *Fix:* `... | sudo tee file`.
5. **A program that daemonizes itself** (forks and exits) with `Type=simple`: systemd thinks it ended. *Fix:* keep the program in the foreground; let systemd supervise.
6. **Writing log files in the program.** You lose rotation, search by time and one place to look. *Fix:* print to stdout/stderr and use `journalctl`.
7. **Running the service as root "to make it work".** *Fix:* find the missing permission with `namei -l` (Lesson 11.4) and give exactly that.

## Try it yourself

1. Follow Sections 2 to 4: create the user, install the binary, write the unit file, start the service and read `status`.
2. Run `journalctl -u heartbeat -f` in one terminal tab (open a second tab with `limactl shell dev`), and in the other run `sudo systemctl reload heartbeat` and `sudo systemctl restart heartbeat`. What lines appear for each?
3. Change `HEARTBEAT_SECONDS=2` to `5` in the unit file. What must you run before the change takes effect?
4. Kill the service with `kill -9` and with `sudo systemctl stop`. Compare the log lines and the `Restart=` behaviour.
5. Section 9: clean up, and check that `systemctl status heartbeat` says the unit could not be found.

## Exercises

### Exercise 1 — Read a failure (easy)

Create `/etc/systemd/system/broken.service` with `ExecStart=/usr/local/bin/does-not-exist`, run `daemon-reload` and `start`, and read the status. What does `status=203/EXEC` mean? Delete the unit afterwards.

<details>
<summary>Hint</summary>

Write the file with `sudo tee`, as in Section 3. A minimal unit needs only a `[Service]` section with `ExecStart=`.

</details>

<details>
<summary>Answer</summary>

```bash
sudo tee /etc/systemd/system/broken.service > /dev/null <<'EOF'
[Service]
ExecStart=/usr/local/bin/does-not-exist
EOF
sudo systemctl daemon-reload
sudo systemctl start broken
systemctl status broken --no-pager -n 3
sudo rm /etc/systemd/system/broken.service
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

```text
× broken.service
     Loaded: loaded (/etc/systemd/system/broken.service; static)
     Active: failed (Result: exit-code) since Wed 2026-11-18 07:07:26 UTC; 1s ago
    Process: 481 ExecStart=/usr/local/bin/does-not-exist (code=exited, status=203/EXEC)
   Main PID: 481 (code=exited, status=203/EXEC)

Nov 18 07:07:26 lima-dev (does-not-exist)[481]: broken.service: Failed at step EXEC spawning /usr/local/bin/does-not-exist: No such file or directory
```

Why it works: `203/EXEC` means systemd could not execute the file in `ExecStart=`: here it does not exist. `reset-failed` clears the "failed" mark.

</details>

### Exercise 2 — Configure without recompiling (medium)

Change the greeting of the running service to `Rust` and the interval to 1 second **without** touching the Rust code, and prove it in the log.

<details>
<summary>Hint</summary>

The program reads two environment variables. Change the `Environment=` lines in the unit file, then `daemon-reload` and `restart`.

</details>

<details>
<summary>Answer</summary>

```bash
sudo sed -i 's/HEARTBEAT_NAME=systemd/HEARTBEAT_NAME=Rust/; s/HEARTBEAT_SECONDS=2/HEARTBEAT_SECONDS=1/' /etc/systemd/system/heartbeat.service
sudo systemctl daemon-reload
sudo systemctl restart heartbeat
sleep 3
journalctl -u heartbeat -n 3 --no-pager -o cat
```

```text
heartbeat 1 from Rust
heartbeat 2 from Rust
heartbeat 3 from Rust
```

(You may see other numbers: they depend on how long you waited.)

Why it works: configuration lives in the environment, outside the binary ([Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md)). `sed -i 's/old/new/'` edits the file in place ([Lesson 11.3](03-text-tools-pipes-and-redirection.md)).

</details>

### Exercise 3 — A smoke test for the service (hard)

A **smoke test** is a quick check that something basic works. Write a short shell test that exits with status 0 only if (a) the service `heartbeat` is active, and (b) the journal of the last minute contains a line `heartbeat` followed by a number. Print `PASS` or `FAIL`. Run it with the service started and stopped.

<details>
<summary>Hint</summary>

`systemctl is-active --quiet NAME` sets only the exit status. `grep -q` does the same for a pattern. Join the checks with `&&`.

</details>

<details>
<summary>Answer</summary>

```bash
sudo systemctl start heartbeat
sleep 3
systemctl is-active --quiet heartbeat \
  && journalctl -u heartbeat --since "1 minute ago" --no-pager -o cat | grep -q 'heartbeat [0-9]' \
  && echo PASS || echo FAIL
sudo systemctl stop heartbeat
systemctl is-active --quiet heartbeat \
  && journalctl -u heartbeat --since "1 minute ago" --no-pager -o cat | grep -q 'heartbeat [0-9]' \
  && echo PASS || echo FAIL
```

```text
PASS
FAIL
```

Why it works: each command succeeds or fails through its exit status. `&&` runs the next part only after a success and `||` runs `echo FAIL` after any failure. The same test can run after every deployment; [Lesson 11.8](08-bash-scripting-basics.md) puts such checks into a script, and the project in [Lesson 11.15](15-project-deploy-a-rust-service.md) uses one.

</details>

## Quiz

1. What is PID 1 on Ubuntu, and what does it do for services?

<details>
<summary>Answer</summary>

`systemd`. It starts services in order, supervises them, restarts them by policy, collects their logs and stops them cleanly.

</details>

2. What is the difference between `start` and `enable`?

<details>
<summary>Answer</summary>

`start` runs the service now. `enable` makes it start at boot (a symlink in `multi-user.target.wants/`). They are independent.

</details>

3. Why do we write unit files with `sudo tee` and not `sudo cat > file`?

<details>
<summary>Answer</summary>

The `>` redirection is done by your normal shell before `sudo` runs, so it has no right to write in `/etc`. With `tee` the writing program runs as root.

</details>

4. Which command shows the log of one service, and how do you follow it live?

<details>
<summary>Answer</summary>

`journalctl -u NAME`; add `-f` to follow.

</details>

5. The service crashed with `SIGKILL` and came back with a new PID. Which line in the unit file did that?

<details>
<summary>Answer</summary>

`Restart=on-failure` (with `RestartSec=` as the delay).

</details>

6. Why does the service binary belong to root while the process runs as `heartbeat`?

<details>
<summary>Answer</summary>

So that a compromised service cannot replace its own program. Least privilege: the service user can run the file but not change it.

</details>

## Summary

- `systemd` is PID 1. It starts, supervises, restarts and stops services, and collects their logs in the journal.
- A unit file in `/etc/systemd/system/` has `[Unit]`, `[Service]` and `[Install]` sections. `ExecStart=` needs an absolute path and a program that stays in the foreground.
- Run `sudo systemctl daemon-reload` after every change. Then `start`, `stop`, `restart`, `reload`, `enable`, `status`, `is-active`.
- `journalctl -u NAME` (with `-n`, `-f`, `--since`, `-o cat`) reads what the program printed to stdout and stderr.
- `Restart=on-failure` and `RestartSec=` bring a crashed service back; a clean `SIGTERM` stop is not restarted.
- Run the service as its own system user, keep the binary owned by root, and harden with `systemd-analyze security` as a guide.
- A service that handles `SIGTERM` (Lesson 11.5) stops cleanly when systemd asks it to.

## Sources and further reading

- [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html) — all options of the `[Service]` section.
- [systemd.unit(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html) — the general format of unit files, and `[Unit]` and `[Install]`.
- [systemctl(1)](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) — every `systemctl` command.
- [journalctl(1)](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) — filters and output formats of the journal.
- [systemd.exec(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html) — the sandboxing options (`ProtectSystem`, `PrivateTmp`, `SystemCallFilter` …).
- [systemd-analyze(1)](https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html) — `verify` and `security`.

---

[🏠 Start](../START-HERE.md) · [← 11.5 Processes and signals](05-processes-and-signals.md) · [📋 Phase 11](README.md) · [11.7 Package managers →](07-package-managers.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
