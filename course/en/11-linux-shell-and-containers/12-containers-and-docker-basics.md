<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.11 Networking commands](11-networking-commands.md) · [📋 Phase 11](README.md) · [11.13 Dockerfiles for Rust programs →](13-dockerfiles-for-rust.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/12-containers-and-docker-basics.md) <!-- nav -->

# Lesson 11.12 — Containers and Docker basics

> **Phase 11 — Linux, the shell and containers** · Lesson 12 of 16 · ⏱ about 120 minutes

## What you will learn

- What a container is, how it differs from a virtual machine, and the difference between an **image** and a **container**.
- The everyday commands: `docker run`, `ps`, `logs`, `exec`, `stop`, `rm`, and how to read an image name.
- How to publish a port, pass environment variables, and keep data with volumes and bind mounts.
- Why a container must handle `SIGTERM` as its first process, and how to stop containers cleanly.
- How to clean up only your own containers, images and volumes, and the main safety rules.

## Before you start

- You finished [Lesson 11.11 — Networking commands](11-networking-commands.md). The bind address (`127.0.0.1` or `0.0.0.0`) matters a lot in this lesson.
- You know processes, signals and exit statuses ([Lesson 11.5](05-processes-and-signals.md)).
- **Docker Desktop must be installed and running on your Mac.** Install it from the official page ([Docker Desktop for Mac](https://docs.docker.com/desktop/setup/install/mac-install/), choose "Apple silicon"), open the app once and wait until it says it is running. The installer asks for your Mac password; check the licence terms for your use. Then in a **Mac terminal tab** (not in `limactl shell`) the command `docker version` must print a client and a server version.
- **Where the commands run.** Docker commands in this lesson are typed on the **Mac**. Docker Desktop starts its own small hidden Linux VM (virtual machine) and runs the containers there ([Lesson 11.1](01-linux-and-a-virtual-machine.md)). It does not use your VM `dev`. The VM `dev` stays the place for Linux practice and, in the project, a deployment target.
- All names we create start with `course-`, so you can find and remove them without touching anything else on your machine.

## New words

| Word | Meaning |
|---|---|
| **container** | A normal Linux process (or a few) that runs with its own isolated view of files, processes, network and host name, and with limits on CPU and memory. It shares the kernel with the host. |
| **image** | A read-only package with everything a container needs to start: files of a small Linux system, your program, and the default command. A container is a running copy made from an image. |
| **layer** | One step of an image: a set of file changes. An image is a stack of layers, and layers are shared between images, which saves disk space and download time. |
| **registry** | A server that stores images. **Docker Hub** (`hub.docker.com`) is the default public registry. |
| **tag** | A label on an image version, written after a colon: `ubuntu:26.04`. Tags can move to a newer image; `latest` is just a tag called "latest". |
| **digest** | The exact fingerprint of one image, `sha256:…`. It never changes, unlike a tag. |
| **Docker Engine / daemon** | The background program (`dockerd`) that builds images and runs containers. The `docker` command is a **client** that talks to it. |
| **Docker Desktop** | The app for Mac and Windows that contains the Docker Engine inside a hidden Linux VM, plus tools. |
| **namespace** (Linux) | A kernel feature that gives a process its own view of one kind of resource: process numbers, network, mounts, host name. This is the isolation part of a container. |
| **cgroup** (control group) | A kernel feature that limits and counts the resources of a group of processes (memory, CPU). First met in [Lesson 11.6](06-services-with-systemd.md). |
| **publish a port** | Connect a port of the host to a port of a container (`-p 8080:80`), so programs outside the container can reach it. |
| **volume** | Storage managed by Docker that lives outside the container, so data survives when the container is removed. |
| **bind mount** | A folder of the host made visible inside the container (`-v /host/folder:/container/folder`). |
| **PID 1** (process ID 1) | The first process of a container. When it ends, the container ends. It also receives the stop signal. |

## 1. What a container is

You know a virtual machine from Lesson 11.1: a whole simulated computer with its own kernel. A **container** is lighter. It is an ordinary process of the host kernel that the kernel has *fenced in*:

- **Namespaces** give it its own view: its own list of processes (it sees itself as process number 1), its own network, its own host name and its own file system.
- **cgroups** limit how much memory and CPU it may use.
- The **image** supplies the file system: the program, its libraries and a minimal Linux userland.

```text
 Virtual machine                       Container
 +-------------------------+          +-----------+ +-----------+
 | your program            |          | program A | | program B |
 | libraries, tools        |          | + files   | | + files   |
 | guest operating system  |          +-----------+ +-----------+
 | (own kernel)            |          |  one shared Linux kernel |
 +-------------------------+          +--------------------------+
 | hypervisor              |          | container engine         |
 +-------------------------+          +--------------------------+
```

| | Virtual machine | Container |
|---|---|---|
| Kernel | its own | shared with the host |
| Start time | tens of seconds | a fraction of a second |
| Size | gigabytes | megabytes (or even less) |
| Isolation | strong (separate kernel) | weaker (one kernel) |
| Typical use | a whole system | one program and what it needs |

A container is not a security wall like a VM. It is a convenient, repeatable way to **package and run one program** with its dependencies, the same on every machine. That is why servers and developers use it.

Containers need a Linux kernel. macOS does not have one, so Docker Desktop runs a hidden Linux VM and the containers live inside it. You can see it: the container below reports the Linux kernel of that VM, not macOS.

```bash
docker run --rm --name course-hello ubuntu:26.04 bash -c 'echo "hello from a container"; hostname; uname -sr; ps aux; id'
```

Example output (the host name and the kernel version differ on your machine):

```text
hello from a container
f65599d60f60
Linux 7.0.14-linuxkit
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1 13.3  0.0   6192  5192 ?        Ss   07:43   0:00 bash -c echo "hello from a container"; hostname; uname -sr; ps aux; id
root           8  0.0  0.0   6192  3204 ?        R    07:43   0:00 ps aux
uid=0(root) gid=0(root) groups=0(root)
```

**What happens:**

1. `docker run` created a container from the image `ubuntu:26.04`, started the command after the image name, and waited for it. If the image is not on your computer, Docker first downloads it (Section 2).
2. `--rm` removes the container when it ends. `--name course-hello` gives it a name.
3. The container has its own host name (a random hexadecimal id). `uname -sr` shows `Linux` and a kernel version: the shared kernel of Docker Desktop's VM.
4. `ps aux` shows **only two processes**. The `bash` we started is **PID 1** (the first process); the container sees nothing of the host. When PID 1 ends, the container stops.
5. `id` says `root`: by default the program in a container runs as the container's root user. Lesson 11.13 shows how to avoid that.

## 2. Images: names, tags, layers

The full name of an image is `registry/repository:tag`. Short names are completed: `ubuntu:26.04` means `docker.io/library/ubuntu:26.04`, "the Docker Hub, the `library` of **official images**, repository `ubuntu`, tag `26.04`". Download an image yourself with `docker pull`:

```bash
docker pull debian:trixie-slim
```

```text
trixie-slim: Pulling from library/debian
acd43e9bebc3: Pulling fs layer
acd43e9bebc3: Already exists
84c77405c2d7: Download complete
Digest: sha256:a72b4433de1cf7d6bae0c4cee74d0547f58601a643b3375f070c51600d63b6b2
Status: Downloaded newer image for debian:trixie-slim
docker.io/library/debian:trixie-slim
```

**What happens:** each line with an id is one **layer**; Docker downloads only the layers it does not have. The `Digest` is the unchanging fingerprint of this exact image. List and inspect what you have:

```bash
docker image ls debian
docker history debian:trixie-slim
```

```text
REPOSITORY   TAG           IMAGE ID       CREATED       SIZE
debian       trixie-slim   a72b4433de1c   13 days ago   142MB
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
a72b4433de1c   13 days ago   # debian.sh --arch 'arm64' out/ 'trixie' '@1…   110MB     debuerreotype 0.17
```

`docker history` lists the layers and the instruction that made each one. `debian:trixie-slim` has one big layer (a minimal Debian system). A bigger image such as `nginx:stable` has several small layers on top of its base, one per step of its build file (Lesson 11.13 writes such a file):

```bash
docker history --format '{{.Size}}\t{{.CreatedBy}}' nginx:stable
```

```text
0B	CMD ["nginx" "-g" "daemon off;"]
0B	STOPSIGNAL SIGQUIT
0B	EXPOSE map[80/tcp:{}]
0B	ENTRYPOINT ["/docker-entrypoint.sh"]
16.4kB	COPY 30-tune-worker-processes.sh /docker-ent…
```

(This is the top of the list, newest layer first. Instructions that only set information, like `CMD` and `EXPOSE`, make layers of size 0.) Remember three facts:

1. **Images are read-only.** A container adds a thin *writable layer* on top. Everything the program writes goes there and is **deleted with the container**.
2. **Tags move, digests do not.** `ubuntu:26.04` may point to a newer build next month. For repeatable builds use a precise tag (`rust:1.100`, `postgres:18`) and, when it matters, the digest (`ubuntu@sha256:…`). Avoid `latest`.
3. **Use trusted images.** The *Docker Official Images* (`ubuntu`, `debian`, `nginx`, `postgres`, `rust` …) are reviewed and kept up to date; see [Docker Hub trusted content](https://docs.docker.com/docker-hub/image-library/trusted-content/). An unknown image from the internet is a program that you run with your files in reach: treat it like a script from a stranger (Lesson 10.13).

## 3. The life of a container

Start a web server in the background. `nginx` is a popular web server; its official image listens on port 80 inside the container:

```bash
docker run -d --name course-web -p 18080:80 nginx:stable
```

```text
1a774906f03ed0fbe661279c51e0565c3512d7a131d308a77df5c1acee3c0f3f
```

**What happens:**

1. `-d` ("detached") runs the container in the background and prints its id.
2. `-p 18080:80` **publishes** a port: port 18080 of your Mac is connected to port 80 inside the container (`host:container`). Without it the web server would run but nobody outside could reach it.
3. `nginx:stable` is the image. We give no command, so Docker runs the image's default one (here: `nginx` in the foreground).

Now look at it, talk to it and read its log:

```bash
docker ps --filter name=course-web
curl -s -o /dev/null -w "code=%{http_code}\n" http://localhost:18080/
docker logs course-web
```

```text
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                       NAMES
a8e73fa11819   nginx:stable   "/docker-entrypoint.…"   3 seconds ago   Up 2 seconds   0.0.0.0:18080->80/tcp, [::]:18080->80/tcp   course-web
code=200
192.168.65.1 - - [18/Nov/2026:07:43:06 +0000] "GET / HTTP/1.1" 200 896 "-" "curl/8.7.1" "-"
```

(The log has more lines from the start of nginx; the last line is our `curl` request.) The commands you will use every day:

| Command | What it does |
|---|---|
| `docker run [options] IMAGE [command]` | create **and** start a container |
| `docker ps` / `docker ps -a` | list running containers / all containers, also the stopped ones |
| `docker logs [-f] NAME` | print what the main process wrote to standard output and error (`-f` follows) |
| `docker exec [-it] NAME command` | run one more command inside a running container |
| `docker stop NAME` / `docker kill NAME` | send `SIGTERM`, then `SIGKILL` after a grace period / send `SIGKILL` at once |
| `docker rm NAME` | delete a stopped container (`-f` stops and deletes) |
| `docker inspect NAME` | all details as JSON (`--format` picks parts) |

**Logs come free.** Docker keeps what the main process prints to standard output and standard error. This is why services print to the console and let the platform collect it, exactly as in systemd ([Lesson 11.6](06-services-with-systemd.md)) and in the twelve-factor ideas ([Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md)).

Run a command inside the running container, then stop and remove it:

```bash
docker exec course-web nginx -v
docker exec course-web sh -c 'ls /usr/share/nginx/html'
docker stop course-web
docker ps -a --filter name=course-web --format 'table {{.Names}}\t{{.Status}}'
docker rm course-web
```

```text
nginx version: nginx/1.30.6
50x.html
index.html
course-web
NAMES        STATUS
course-web   Exited (0) Less than a second ago
course-web
```

A stopped container still exists (with its exit code and its writable layer) until you `docker rm` it. The exit code is the one of PID 1: `docker run --rm ubuntu:26.04 bash -c 'exit 7'` makes `docker run` itself exit with status 7, so containers fit into scripts ([Lesson 11.9](09-robust-bash-scripts.md)). A command that does not exist gives status 127, as in a normal shell.

**An interactive shell** is the best way to explore an image. `-i` keeps standard input open, `-t` gives a terminal:

```bash
docker run --rm -it --name course-shell ubuntu:26.04
```

You get a prompt like `root@a1b2c3d4e5f6:/#`. Try `ls /`, `cat /etc/os-release`, `ps aux`, then `exit`. Everything you installed or changed is gone afterwards (because of `--rm` and the throw-away layer).

> ⚠️ **Common mistake:** Looking for a tool such as `curl` or `ps` inside a small image and not finding it. Images contain only what their makers put in; `ubuntu:26.04` has `ps` but not `curl`. You do not fix this by installing things into a running container (they vanish), but in the build file (Lesson 11.13).

## 4. Environment variables, ports and the bind address

**Environment variables** configure the program without changing the image ([Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md)). Use `-e NAME=value`, or `--env-file` for many:

```bash
docker run --rm -e GREETING=hello -e PORT=3000 ubuntu:26.04 bash -c 'echo "$GREETING on $PORT"'
printf 'A=1\nB=two\n' > env.list
docker run --rm --env-file env.list ubuntu:26.04 bash -c 'echo "$A $B"'
```

```text
hello on 3000
1 two
```

> 🔒 **Security:** `docker inspect` shows every `-e` value, and so do process listings on the host. Do not build secrets into an image and do not put them on the command line; use a file that is not in Git and readable only by you (`--env-file` with mode `600`), or the secret mechanisms of Compose (Lesson 11.14) and the platform ([Lesson 10.8](../10-security-fundamentals/08-secrets-management.md)).

**Published ports** have the form `-p HOST_PORT:CONTAINER_PORT`. By default Docker listens on *all* addresses of your Mac, so other computers on your network could reach the container. To keep it private, put the address in front:

```bash
docker run -d --name course-web -p 127.0.0.1:18080:80 nginx:stable
docker ps --filter name=course-web --format '{{.Ports}}'
```

```text
127.0.0.1:18080->80/tcp
```

Now only your Mac can open `http://localhost:18080/`. Remove it again: `docker rm -f course-web`.

Remember Lesson 11.11: **inside** the container the server must listen on `0.0.0.0` (or at least the container's own address). Docker delivers published traffic to the container's network interface, not to its loopback. nginx does this correctly (`listen 80` means all addresses). A program of yours that binds to `127.0.0.1` cannot be reached through `-p`; you will reproduce and fix this with a Rust program in Lesson 11.13.

## 5. Data: the writable layer, volumes and bind mounts

Prove that the container's own files disappear, and that a **volume** keeps them:

```bash
docker run --name course-t1 ubuntu:26.04 bash -c 'echo data > /root/note.txt; cat /root/note.txt'
docker rm course-t1
docker run --rm ubuntu:26.04 ls /root/note.txt
```

```text
data
ls: cannot access '/root/note.txt': No such file or directory
```

The file was in the writable layer of the first container; `docker rm` deleted it. Now with a named volume:

```bash
docker volume create course-data
docker run --rm -v course-data:/data ubuntu:26.04 bash -c 'echo kept > /data/note.txt'
docker run --rm -v course-data:/data ubuntu:26.04 cat /data/note.txt
docker volume ls --filter name=course-
```

```text
course-data
kept
DRIVER    VOLUME NAME
local     course-data
```

`-v course-data:/data` mounts the volume at `/data` of the container. It belongs to Docker, lives outside any container, and survives `docker rm`. This is where a database keeps its files. A **bind mount** instead exposes a folder of your Mac; the path before the colon is absolute:

```bash
mkdir -p hostdir && echo "from the host" > hostdir/hello.txt
docker run --rm -v "$PWD/hostdir":/work:ro ubuntu:26.04 bash -c 'cat /work/hello.txt; touch /work/x'
```

```text
from the host
touch: cannot touch '/work/x': Read-only file system
```

`:ro` makes the mount read-only. Use bind mounts to share source code or configuration with a container, and volumes for data that the container owns. Remove the volume when done: `docker volume rm course-data`.

## 6. Stopping containers properly

`docker stop` sends `SIGTERM` to **PID 1** of the container and, if it does not exit within a grace period, `SIGKILL` ([Lesson 11.5](05-processes-and-signals.md)). Here is a trap, and it matters for your Rust services. A program that runs as PID 1 does not get the usual default action for signals: if it has no handler for `SIGTERM`, the kernel **ignores** the signal. Compare:

```bash
docker run -d --name course-sleep ubuntu:26.04 sleep infinity
SECONDS=0; docker stop course-sleep; echo "stop took ${SECONDS}s"
docker inspect course-sleep --format 'exit={{.State.ExitCode}}'
docker rm course-sleep
```

```text
course-sleep
stop took 3s
exit=137
```

`sleep` has no `SIGTERM` handler and runs as PID 1, so the signal was ignored. After the grace period Docker killed it: exit code **137** = 128 + 9 (`SIGKILL`). A service killed this way never runs its cleanup. Two cures:

```bash
docker run -d --init --name course-sleep2 ubuntu:26.04 sleep infinity
SECONDS=0; docker stop course-sleep2; echo "stop took ${SECONDS}s"
docker inspect course-sleep2 --format 'exit={{.State.ExitCode}}'
docker rm course-sleep2
```

```text
course-sleep2
stop took 0s
exit=143
```

1. **`--init`** starts a tiny init program as PID 1 that forwards signals to your program. Now `SIGTERM` works at once; exit 143 = 128 + 15.
2. **Handle the signal in the program**, as `p11-heartbeat` did in Lesson 11.5. Then PID 1 itself shuts down cleanly. A shell script as PID 1 can do it with `trap` ([Lesson 11.9](09-robust-bash-scripts.md)):

```bash
docker run -d --name course-trap ubuntu:26.04 bash -c 'trap "echo got TERM; exit 0" TERM; echo started; while true; do sleep 1; done'
sleep 1.5
docker stop course-trap
docker logs course-trap
docker inspect course-trap --format 'exit={{.State.ExitCode}}'
docker rm course-trap
```

```text
course-trap
started
got TERM
exit=0
```

Exit 0: a clean stop. In Lesson 11.13 your Rust service gets a `SIGTERM` handler for exactly this reason. (For your own service use the form `CMD ["/path/program"]` that you will meet there, so that the program itself, and not a shell, is PID 1.)

## 7. Limits, clean-up and safety

You can limit a container: `--memory 64m` and `--cpus 0.5` configure cgroups. Inside, the kernel shows the limits:

```bash
docker run --rm --memory 64m --cpus 0.5 ubuntu:26.04 bash -c 'cat /sys/fs/cgroup/memory.max; cat /sys/fs/cgroup/cpu.max'
```

```text
67108864
50000 100000
```

64 MiB (mebibytes; 1 MiB = 1024 × 1024 bytes) in bytes, and 50,000 µs of CPU time per 100,000 µs period, which is half a CPU.

**Clean up only your own objects.** Docker keeps containers, images and volumes until you remove them, and one machine usually holds objects of many projects. List them with filters and delete by name:

```bash
docker ps -a --filter name=course-
docker volume ls --filter name=course-
docker image ls 'course-*'
docker rm -f course-web
docker volume rm course-data
docker image rm debian:trixie-slim
```

> ⚠️ **Common mistake:** `docker system prune` and `docker container prune` delete *everything* that is stopped or unused, including things you may want later or that belong to other projects. Prefer removing by name or filter. If you ever use a prune command, read its question carefully first.

> 🔒 **Security:** Containers add convenience and some isolation, but:
> - The program runs as **root** inside by default. A flaw plus a wrong option can reach the host. Run services as a normal user (Lesson 11.13).
> - **Never** use `--privileged` or mount `/var/run/docker.sock` into a container unless you understand it: that gives the container control of the Docker Engine, which is control of the machine.
> - Never mount your whole home folder (`-v ~:/home`). Mount only what is needed, read-only when possible.
> - Take images from official or trusted sources; pin the version tag.
> - Anyone who can run `docker` on a Linux server is effectively root there.

## Common mistakes

1. **`docker: Cannot connect to the Docker daemon`.** Docker Desktop is not running. *Fix:* start the app and wait for it to be ready, then try again.
2. **`port is already allocated` / `address already in use`.** Another program or container uses the host port. *Fix:* `docker ps` (stop the old container) or choose another host port (`-p 18081:80`).
3. **Data disappeared.** It was in the writable layer. *Fix:* use a volume or a bind mount for data.
4. **Forgetting `-d`.** The terminal is blocked by the container's output. *Fix:* `Ctrl + C` stops it (for a container started without `-d`), or start with `-d` and use `docker logs -f`.
5. **A server in the container is unreachable although `-p` is set.** It listens on `127.0.0.1` inside. *Fix:* listen on `0.0.0.0` (Lesson 11.13).
6. **Using `latest`** and getting a different version tomorrow. *Fix:* pin the tag.
7. **Leaving containers around.** `docker ps -a` fills up with exited containers. *Fix:* `--rm` for short runs; remove by name.

## Try it yourself

1. Run `docker version` and `docker run --rm ubuntu:26.04 cat /etc/os-release`. What is `PRETTY_NAME`? Which kernel does `uname -r` show inside, and is it the kernel of macOS?
2. Start `nginx:stable` with `-p 127.0.0.1:18080:80` under the name `course-web`. Open `http://localhost:18080/` in a browser, read `docker logs course-web`, and remove the container.
3. Make a folder `site` with an `index.html` of your own and serve it: `-v "$PWD/site":/usr/share/nginx/html:ro`. Change the file and reload the page without restarting the container.
4. Run `docker run --rm ubuntu:26.04 bash -c 'exit 5'; echo $?` and explain the number.
5. Remove everything you created, by name.

## Exercises

### Exercise 1 — Find the facts (easy)

Without installing anything, find out with one `docker run` command each: (a) which Linux distribution and version `debian:trixie-slim` contains; (b) how many processes `ps aux` shows in a fresh `ubuntu:26.04` container (note: the `ps` command itself counts); (c) the user id inside the container.

<details>
<summary>Hint</summary>

`cat /etc/os-release`, `ps aux | wc -l` (one line is the header), `id -u`. Remember `--rm`.

</details>

<details>
<summary>Answer</summary>

```bash
docker run --rm debian:trixie-slim cat /etc/os-release
docker run --rm ubuntu:26.04 bash -c 'ps aux | wc -l'
docker run --rm ubuntu:26.04 id -u
```

(a) Debian GNU/Linux 13 (trixie). (b) The output is `4`: the header line plus `bash`, `ps` and `wc`; the pipeline starts three processes next to the header. (c) `0`, the root user.

Why it works: each `docker run` makes a fresh, isolated container, so the process list is tiny and the numbers are the same every time.

</details>

### Exercise 2 — Serve your own page (medium)

Serve a folder `site` with your own `index.html` by `nginx:stable` on `http://localhost:18082/`, reachable only from your Mac. Then change the HTML (HyperText Markup Language, the language of web pages) file and show that `curl` returns the new text **without** restarting. Remove the container.

<details>
<summary>Hint</summary>

You need `-p 127.0.0.1:18082:80` and a bind mount to `/usr/share/nginx/html`. Use `:ro` so the container cannot change your files.

</details>

<details>
<summary>Answer</summary>

```bash
mkdir -p site && echo "<h1>My own page</h1>" > site/index.html
docker run -d --name course-site -p 127.0.0.1:18082:80 -v "$PWD/site":/usr/share/nginx/html:ro nginx:stable
curl -s http://localhost:18082/
echo "<h1>Changed</h1>" > site/index.html
curl -s http://localhost:18082/
docker rm -f course-site
```

```text
<h1>My own page</h1>
<h1>Changed</h1>
```

Why it works: a bind mount shows the host folder itself inside the container, not a copy, so edits are visible at once.

</details>

### Exercise 3 — A test for container facts (medium)

Write `test-docker.sh` in `code/p11-scripts` that checks three facts and cleans up after itself even when a check fails: (1) the exit status of the program inside becomes the status of `docker run` (use `exit 3`); (2) `nginx:stable` with `-p 18081:80` answers `200` on `http://localhost:18081/`; (3) a file written into a named volume can be read by a second container. Use names that start with `course-`.

<details>
<summary>Hint</summary>

Use the `check` function from Lesson 11.9, a `trap cleanup EXIT` that runs `docker stop` and `docker volume rm`, and a short loop that waits until `curl -sf` succeeds, because the web server needs a moment to start.

</details>

<details>
<summary>Answer</summary>

```bash
#!/usr/bin/env bash
# test-docker.sh - checks three facts about containers. Needs Docker running.
set -u

failures=0
check() {   # check "description" expected actual
    if [[ $2 == "$3" ]]; then
        echo "ok   - $1"
    else
        echo "FAIL - $1 (expected [$2], got [$3])"
        failures=$((failures + 1))
    fi
}

web=course-test-web
volume=course-test-data
cleanup() {
    docker stop "$web" > /dev/null 2>&1
    docker volume rm "$volume" > /dev/null 2>&1
}
trap cleanup EXIT

# 1. The exit status of the program inside becomes the status of "docker run".
docker run --rm ubuntu:26.04 bash -c 'exit 3'
check "exit status is passed on" 3 "$?"

# 2. A published port reaches the web server.
docker run -d --rm --name "$web" -p 18081:80 nginx:stable > /dev/null
for _ in 1 2 3 4 5 6 7 8 9 10; do
    curl -sf -o /dev/null http://localhost:18081/ && break
    sleep 0.5
done
check "web answers 200" 200 "$(curl -s -o /dev/null -w '%{http_code}' http://localhost:18081/)"

# 3. Data in a named volume survives the container.
docker volume create "$volume" > /dev/null
docker run --rm -v "$volume":/data ubuntu:26.04 bash -c 'echo kept > /data/note.txt'
check "volume keeps the file" kept "$(docker run --rm -v "$volume":/data ubuntu:26.04 cat /data/note.txt)"

(( failures == 0 ))
```

```text
ok   - exit status is passed on
ok   - web answers 200
ok   - volume keeps the file
```

Why it works: `--rm` removes the short-lived containers by itself; the web container was started with `--rm` too, so `docker stop` in the trap also deletes it. The trap runs on every exit path, so a failed check leaves no volume or container behind.

</details>

## Quiz

1. What is the difference between an image and a container?

<details>
<summary>Answer</summary>

An image is a read-only package (layers of files plus a default command). A container is a running (or stopped) instance made from an image, with a thin writable layer on top. Many containers can start from one image.

</details>

2. Why can you not run Linux containers directly on macOS?

<details>
<summary>Answer</summary>

Containers use the Linux kernel of the host. macOS has another kernel, so Docker Desktop runs a small hidden Linux VM and starts the containers in it.

</details>

3. What does `-p 8080:80` mean?

<details>
<summary>Answer</summary>

Publish a port: connections to port 8080 on the host are forwarded to port 80 inside the container (`host:container`).

</details>

4. Where do the files go that a program writes inside a container, and what happens to them on `docker rm`?

<details>
<summary>Answer</summary>

Into the container's writable layer. They are deleted with the container. To keep data use a volume or a bind mount.

</details>

5. Why did `docker stop` take the whole grace period for `sleep infinity`, and how do you avoid it?

<details>
<summary>Answer</summary>

`sleep` ran as PID 1 without a `SIGTERM` handler, and the kernel ignores unhandled signals for PID 1, so Docker had to send `SIGKILL` (exit 137). Use `--init`, or let the program handle `SIGTERM` itself.

</details>

6. Name two dangerous options and say why.

<details>
<summary>Answer</summary>

`--privileged` removes most of the isolation, and mounting `/var/run/docker.sock` gives the container control of the Docker Engine. Both are close to full control of the host. Mounting the whole home folder is also risky.

</details>

## Summary

- A container is an ordinary Linux process with its own view of files, processes and network (namespaces) and limits (cgroups). It shares the kernel; on a Mac the kernel is in Docker Desktop's hidden VM.
- An image is a read-only stack of layers with a default command. A container is a running copy with a throw-away writable layer. Names look like `registry/repository:tag`; tags move, digests do not.
- Everyday commands: `docker run` (`-d`, `--rm`, `--name`, `-it`), `ps`, `logs`, `exec`, `stop`, `rm`, `inspect`.
- `-p host:container` publishes ports (add `127.0.0.1:` to keep it private), `-e` and `--env-file` set environment variables, `-v` mounts volumes or host folders.
- The process in the container is PID 1: its exit ends the container, and `docker stop` sends it `SIGTERM`. Handle the signal or use `--init`.
- Name your objects `course-…`, remove them by name, avoid prune commands, run as a normal user, and never mount the Docker socket or use `--privileged` carelessly.

## Sources and further reading

- [Docker overview](https://docs.docker.com/get-started/docker-overview/) — the official introduction: Engine, images, containers, registries.
- [Docker Desktop for Mac: install](https://docs.docker.com/desktop/setup/install/mac-install/) — installation and system requirements.
- [`docker run` reference](https://docs.docker.com/engine/containers/run/) — every option of `docker run`.
- [Docker CLI reference](https://docs.docker.com/reference/cli/docker/) — all commands, with `docker container stop` explaining signals and the grace period.
- [Volumes](https://docs.docker.com/engine/storage/volumes/) and [Port publishing and mapping](https://docs.docker.com/engine/network/port-publishing/) — storage and networking in detail.
- [Docker Engine security](https://docs.docker.com/engine/security/) — what containers do and do not isolate.

---

[🏠 Start](../START-HERE.md) · [← 11.11 Networking commands](11-networking-commands.md) · [📋 Phase 11](README.md) · [11.13 Dockerfiles for Rust programs →](13-dockerfiles-for-rust.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
