<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.15 Checkpoint](../10-security-fundamentals/15-checkpoint.md) · [11.1 Linux and a virtual machine on your Mac →](01-linux-and-a-virtual-machine.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/README.md) <!-- nav -->

# Phase 11 — Linux, the shell and containers

> **Stage 2 — Engineering fundamentals** · ⏱ about 4 weeks at ~10 hours per week

## What this phase is about

Almost every program that a systems engineer writes in Rust ends up running on **Linux**: on a server in the cloud, inside a container, on a small board. Your Mac is a fine place to write code, but it is not where the code will live. This phase teaches you the home of your programs: how a Linux machine is organised, how to find your way around it, and how to run, watch and ship a Rust program there.

You start by creating a real Linux machine inside your Mac (a virtual machine called `dev`) and use it in every lesson. Then you learn the everyday skills in a sensible order: the file system, text tools and pipes, users and permissions, processes and signals, services with `systemd`, and package managers. After that you learn to automate work with bash scripts and to make them robust, to log in to remote machines over SSH (Secure Shell), and to read the network with a few commands.

The second half of the phase is about **containers**. A container packs a program with everything it needs, so that it runs the same way on your laptop, on a colleague's computer and on a server. You learn what a container really is, how to write a Dockerfile for a Rust program, and how to run several containers together with Docker Compose. The phase ends with a project in which you deploy a Rust service end to end, and a checkpoint that closes Stage 2.

## What you will be able to do

- [ ] Create, start, stop and rebuild a Linux virtual machine on your Mac, and know which files are shared with it.
- [ ] Move around the Linux file system, read the permissions of any file, and fix a "Permission denied" without `chmod 777`.
- [ ] Answer questions about a log file with `grep`, `sort`, `uniq`, `awk` and pipes, and redirect the three standard streams.
- [ ] List processes, send signals, read exit statuses, and write a Rust program that shuts down cleanly on `SIGTERM`.
- [ ] Run a Rust program as a `systemd` service under its own user, read its logs with `journalctl`, and make it restart after a crash.
- [ ] Install, inspect and remove software with `apt`, and package a Rust program as a `.deb`.
- [ ] Write and test bash scripts that handle arguments, errors and exit statuses.
- [ ] Connect to remote servers with SSH keys, check the network, build a small Docker image for a Rust program, and deploy a service with Docker Compose.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 11.1 | [Linux and a virtual machine on your Mac](01-linux-and-a-virtual-machine.md) | 120 min |
| 11.2 | [The Linux file system](02-the-linux-file-system.md) | 120 min |
| 11.3 | [Text tools, pipes and redirection](03-text-tools-pipes-and-redirection.md) | 2.5 h |
| 11.4 | [Users, groups and permissions](04-users-groups-and-permissions.md) | 135 min |
| 11.5 | [Processes and signals](05-processes-and-signals.md) | 135 min |
| 11.6 | [Services with systemd](06-services-with-systemd.md) | 120 min |
| 11.7 | [Package managers](07-package-managers.md) | 120 min |
| 11.8 | [Bash scripting basics](08-bash-scripting-basics.md) | 2.5 h |
| 11.9 | [Robust bash scripts](09-robust-bash-scripts.md) | 2.5 h |
| 11.10 | [SSH and remote servers](10-ssh-and-remote-servers.md) | 135 min |
| 11.11 | [Networking commands](11-networking-commands.md) | 120 min |
| 11.12 | [Containers and Docker basics](12-containers-and-docker-basics.md) | 120 min |
| 11.13 | [Dockerfiles for Rust programs](13-dockerfiles-for-rust.md) | 135 min |
| 11.14 | [Docker Compose](14-docker-compose.md) | 135 min |
| 11.15 | [Project: deploy a Rust service](15-project-deploy-a-rust-service.md) | 6 h |
| 11.16 | [Checkpoint: Stage 2 complete](16-checkpoint-stage-2-complete.md) | 2.5 h |

## Projects in this phase

- **Small programs and scripts** that you write along the way: `code/p11-streams` (a filter for pipelines), `code/p11-secretfile` (a private file), `code/p11-heartbeat` (a service that handles signals), `code/p11-bind` (a server and its bind address), `code/p11-hello` (a tiny web service for Docker), `code/p11-counter` (the same service with PostgreSQL, for Compose) and the scripts in `code/p11-scripts`.
- **[Lesson 11.15 — Project: deploy a Rust service](15-project-deploy-a-rust-service.md)** in `code/p11-deploy`: you deploy one Rust service in two ways, as a systemd service in the VM with a deploy script that can roll back, and as a container run by Docker Compose, and you write the acceptance script that checks both.
- **[Lesson 11.16 — Checkpoint: Stage 2 complete](16-checkpoint-stage-2-complete.md)** tests everything you learned in Phases 07 to 11.

## How to study this phase

1. **Type every command yourself in the VM.** Do not paste blindly. Read each command before you press Enter, as [Lesson 00.4](../00-getting-started/04-terminal-basics.md) taught you. The prompt tells you where you are: `%` is your Mac, `student@lima-dev:...$` is the VM.
2. **Break things on purpose in the VM.** It is a lab: delete a file you should not, kill a service, write a wrong unit file, and read the error. You can always `limactl delete dev` and start again. Keep what matters in Git.
3. **Read the error before you search for it.** Linux error messages are short and exact (`Permission denied`, `No such file or directory`, `status=203/EXEC`). Look the message up in the lesson's "Common mistakes" first.
4. **Keep a cheat sheet.** In your learning notes, collect the commands you used this week with one line about what each does. You will use them for the rest of your career.
5. **Plan for about three to four weeks.** Two lessons a week leaves time for the exercises and for the project. Stop the VM (`limactl stop dev`) when you finish for the day.

---

[🏠 Start](../START-HERE.md) · [← 10.15 Checkpoint](../10-security-fundamentals/15-checkpoint.md) · [11.1 Linux and a virtual machine on your Mac →](01-linux-and-a-virtual-machine.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
