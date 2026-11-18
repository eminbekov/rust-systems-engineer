<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.16 Control groups (cgroups)](16-cgroups.md) · [📋 Phase 13](README.md) · [13.18 Checkpoint →](18-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/17-project-mini-container-runtime.md) <!-- nav -->

# Lesson 13.17 — Project: a mini container runtime

> **Phase 13 — Operating systems from the inside** · Lesson 17 of 18 · ⏱ about 8 hours (4 sittings)

## What you will learn

- How namespaces, a new root file system and cgroups come together into a container.
- Why `pivot_root` is the safe way to change the root, and what each step before it is for.
- Which namespace flags affect the calling process and which affect only its children, and why that decides who calls `unshare`.
- How to clean up kernel objects reliably with `Drop`, and how to test a program that only works as root.

## Before you start

- You finished [Lesson 13.15 — Namespaces](15-namespaces.md) and [Lesson 13.16 — Control groups (cgroups)](16-cgroups.md), including their exercises: this project grows out of `parse_namespaces` and `box` from 13.15 and `limit` from 13.16. Keep those files open.
- You know `fork`, `execvp` and `waitpid` ([Lesson 13.5](05-fork-exec-and-wait.md)), the project format and `assert_cmd` from [Lesson 13.8 — Project: your own shell](08-project-your-own-shell.md), and `clap` and `anyhow` from [Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md).
- Docker Desktop runs on your Mac ([Lesson 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)). You only use it once, to get the files of a small Linux system.
- You write the code on the Mac in `~/projects/rust/code/p13-minibox` and build and run it in the VM `dev`, as in Lesson 13.1 (`rsync` the folder to `~/p13-minibox`). Everything that starts a container runs with `sudo` **in the VM only**, never on your Mac.

## New words

| Word | Meaning |
|---|---|
| **container runtime** | The program that actually creates a container: it sets up namespaces, the root file system and cgroups, then starts the program inside. Docker uses `runc` for this step. |
| **root file system** (rootfs) | The directory tree a container sees as `/`: its own `/bin`, `/etc`, `/lib` and so on, usually taken from an image. |
| **`docker export`** | A Docker command that writes all files of a container into one tar archive. |
| **tar archive** | One file that contains many files and folders with their owners and permissions; `tar -xf` unpacks it. |
| **`chroot`** | A system call that changes the root directory of one process. The old file system stays mounted and reachable. |
| **`pivot_root`** | A system call that swaps the root mount of the whole mount namespace: a new directory becomes `/`, and the old root can then be unmounted. |
| **bind mount** | Making a directory visible at a second place (`MS_BIND`). Here you bind a directory onto itself to turn it into a mount point. |
| **lazy unmount** (`MNT_DETACH`) | Removing a mount from the tree at once; the kernel frees it when nobody uses it any more. |
| **OCI** (Open Container Initiative) | The group that writes the open standards for container images and runtimes; `runc` follows its runtime specification. |
| **test runner** (Cargo `runner`) | A program that Cargo puts in front of every test or binary it starts, for example `sudo`. |
| **PID**, **UTS**, **OOM**, **CPU**, **VM** | Known from earlier lessons: Process ID; UNIX Time-Sharing (the namespace of the host name); Out Of Memory; Central Processing Unit; virtual machine (your VM `dev`). |

## The project

### What you will build

`minibox` — a container runtime in about 300 lines of Rust, tests not counted. It starts a command in new PID, UTS and mount
namespaces, inside a root file system of its own, with `/proc` mounted, inside a cgroup with optional memory and CPU limits,
and it returns the command's exit code and removes the cgroup afterwards. A session in the VM:

```text
$ sudo ./target/debug/minibox run --rootfs ~/minibox/alpine --hostname box1 -- /bin/sh
/ # hostname
box1
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    3 root      0:00 ps
/ # exit 3
$ echo $?
3
```

The shell inside is Alpine Linux's, not your VM's; it is PID 1; it sees only its own processes. (This output and all
outputs below come from a reference solution checked in a privileged Linux container set up like the VM, with the host name
`lima-dev`. PIDs and times on your VM will differ.) You get no complete solution: the contract, the milestones, small hints
and the "Done when" checks are your guide.

The work is split between two processes, like in the `box` exercise, but with a new root and a cgroup:

```text
 minibox (parent, stays outside)                  child (becomes PID 1 of the container)
 ---------------------------------                ---------------------------------------
 parse arguments, check root and rootfs
 create /sys/fs/cgroup/minibox-<pid>, limits
 move itself into the group
 unshare(CLONE_NEWPID)
 fork  ------------------------------------------> unshare(CLONE_NEWUTS | CLONE_NEWNS)
 waitpid(child)                                    sethostname
    |                                              make all mounts private
    |                                              bind-mount rootfs onto itself
    |                                              chdir(rootfs), pivot_root(".", ".")
    |                                              umount2(".", MNT_DETACH), chdir("/")
    |                                              mount proc on /proc
    |                                              execvp(command)
    v  <------------------ exit status -----------
 exit code, OOM report, move out, remove group
```

### Contract

**Package and files.** A binary package `code/p13-minibox` named `minibox`, with a small library inside:

```text
code/p13-minibox/
├── Cargo.toml
├── README.md          ← what minibox does, how to get a rootfs, build, run and test it
├── src/
│   ├── main.rs        ← command line (clap) → Config, error messages, exit code
│   ├── lib.rs         ← pub mod parse; pub mod cgroup; pub mod container;
│   ├── parse.rs       ← parse_memory, parse_cpus, cpu_max, parse_namespaces, cgroup_path + unit tests
│   ├── cgroup.rs      ← struct Cgroup: create, enter, oom_kills; Drop moves out and removes it
│   └── container.rs   ← run(): unshare, fork, set-up in the child, exec, wait
└── tests/
    └── cli.rs         ← integration tests; the ones that need root are #[ignore]
```

Dependencies (crates.io versions checked on 2026-11-18):

```toml
[dependencies]
anyhow = "1.0"
clap = { version = "4.6", features = ["derive"] }
nix = { version = "0.32", features = ["sched", "hostname", "process", "mount", "fs", "user"] }

[dev-dependencies]
assert_cmd = "2.2"
predicates = "3.1"
```

**Usage.**

```text
minibox run --rootfs <dir> [--memory <bytes>] [--cpus <n>] [--hostname <name>] -- <command> [args...]
```

| Option | Meaning | Default |
|---|---|---|
| `--rootfs <dir>` | the directory that becomes `/` inside; it must exist and contain `bin/` | required |
| `--memory <bytes>` | written to `memory.max`; plain bytes or a number with `K`, `M`, `G` (powers of 1024); also sets `memory.swap.max` to 0 | no limit |
| `--cpus <n>` | written to `cpu.max` as `n × 100000` microseconds per period of 100000; at least `0.01` | no limit |
| `--hostname <name>` | host name in the new UTS namespace | `minibox` |
| `-- <command> [args...]` | the program to run as PID 1 inside, found through `PATH` | required |

**Behaviour.**

- `minibox` must run as root; otherwise it prints `minibox: must be run as root (try sudo)` and exits with 125.
- The command runs in new PID, UTS and mount namespaces, as PID 1, with the given host name.
- Inside, `/` is the rootfs (through `pivot_root`), the old root is unmounted, and a fresh `proc` is mounted on `/proc`. After
  `minibox` ends, none of these mounts is visible in the VM.
- `minibox` always creates the cgroup `/sys/fs/cgroup/minibox-<PID of minibox>`, even without limits, and removes it at the
  end. If the OOM killer killed something in it, `minibox` prints `minibox: the container was killed by the OOM killer`.

**Exit statuses and errors** (the numbers 125–127 follow the conventions of `docker run` and of shells):

| Situation | Message on standard error | Status |
|---|---|---|
| the command exits with N | — | N |
| the command is killed by signal S | — | 128 + S (137 for SIGKILL) |
| the command is not found | `minibox: NAME: command not found` | 127 |
| the command is found but cannot be started | `minibox: NAME: <error text>` | 126 |
| an error of `minibox` itself (not root, bad rootfs, a failed system call) | `minibox: <what failed>: <error>` | 125 |
| wrong usage (no `--rootfs`, no command, bad `--cpus`) | clap's error message | 2 |

**Quality.** `cargo test` passes as an ordinary user, the root-only integration tests pass when run through `sudo`, `cargo
clippy --all-targets` gives no warnings, `cargo fmt --check` is clean, and there is one Git commit per milestone.

### Step-by-step plan

**Get a root file system.** On the **Mac**, let Docker create (but not start) a container from the official Alpine Linux
image and export its files:

```bash
docker create --name course-alpine alpine:3.24
docker export -o ~/projects/rust/code/alpine-3.24-rootfs.tar course-alpine
docker rm course-alpine
```

Example output (your ID differs; the first run also prints pull messages):

```text
4511a81975d6132c06bd6f2a878d472c46a52de2e2684dec99f5350608f8541c
course-alpine
```

In the **VM**, unpack it with `sudo`, so that the files belong to root as they did in the image (only root may give files
to another owner), then delete the archive on the Mac:

```bash
mkdir -p ~/minibox/alpine
sudo tar -xf /Users/student/projects/rust/code/alpine-3.24-rootfs.tar -C ~/minibox/alpine
cat ~/minibox/alpine/etc/alpine-release
ls ~/minibox/alpine
```

```text
3.24.4
bin  etc   lib    mnt  proc  run   srv  tmp  var
dev  home  media  opt  root  sbin  sys  usr
```

(Replace `student` with your Mac user name, as in Lesson 13.1. The Alpine version is whatever `3.24` points to on the day you
pull it.) Now create the package on the Mac:

```bash
cd ~/projects/rust
cargo new code/p13-minibox --name minibox
cd code/p13-minibox
cargo add nix@0.32 --features sched,hostname,process,mount,fs,user
cargo add clap@4.6 --features derive
cargo add anyhow@1.0
cargo add --dev assert_cmd@2.2 predicates@3.1
```

#### M1 — The command line, test first

**Goal.** `minibox run ...` is parsed into a `Config`, and every rule of the options table is checked by a unit test. For now
`main` only prints the `Config`.

**Guidance.**

1. In `parse.rs`, write the pure functions and their tests **before** the `clap` code. Suggested signatures:

   ```rust
   pub fn parse_memory(text: &str) -> Result<u64, String> {
       todo!()
   }

   pub fn parse_cpus(text: &str) -> Result<f64, String> {
       todo!()
   }

   pub fn cpu_max(cpus: f64) -> String {
       todo!()
   }
   ```

   `parse_memory("50M")` is `Ok(52428800)`; `""`, `"M"`, `"-5M"`, `"0"` and a number that does not fit in `u64` are errors.
   `parse_cpus` refuses anything below `0.01`, because the kernel does not accept a quota under 1000 microseconds;
   `cpu_max(0.5)` is `"50000 100000"`.
2. Copy `parse_namespaces` (Lesson 13.15, Exercise 2) and `cgroup_path` (Lesson 13.16, Exercise 2) into `parse.rs` with
   their tests. You will need both later.
3. In `main.rs`, describe the command line with `clap`'s derive style: a `Cli` struct with a subcommand enum that has one
   variant, `Run(RunArgs)`. In `RunArgs`, `#[arg(long, value_parser = parse_memory)]` lets `clap` call your function, and
   `#[arg(last = true, required = true)] command: Vec<String>` collects everything after `--`.
4. Put `Config` (rootfs, memory, cpus, hostname, command) into `container.rs` with `#[derive(Debug)]`.

<details>
<summary>Hint</summary>

When a `value_parser` function returns `Err(text)`, `clap` prints `error: invalid value '...' for '--cpus <CPUS>': text` and
exits with status 2, which is exactly the contract. For the suffix, look at the last character with `text.chars().last()`
and cut it off; multiply with `checked_mul`, which returns `None` on overflow instead of wrapping around (Lesson 12.2).

</details>

**Done when:**

```bash
cargo test -q
cargo build -q
./target/debug/minibox run --rootfs /tmp --memory 50M --cpus 0.5 -- /bin/echo hi
./target/debug/minibox run --rootfs /tmp --cpus 0 -- /bin/echo hi; echo "exit=$?"
```

shows `test result: ok` for your unit tests (the reference solution has 6), and then:

```text
Config { rootfs: "/tmp", memory: Some(52428800), cpus: Some(0.5), hostname: "minibox", command: ["/bin/echo", "hi"] }
error: invalid value '0' for '--cpus <CPUS>': --cpus must be at least 0.01

For more information, try '--help'.
exit=2
```

Commit: `git commit -m "feat(p13-minibox): M1 command line and parsers"`.

#### M2 — New namespaces and the exit code

**Goal.** `run` starts the command in new PID, UTS and mount namespaces and returns its exit code. The rootfs is only checked
for now; the command still sees the VM's files.

**Guidance.**

1. Check `nix::unistd::geteuid().is_root()` first. Then `fs::canonicalize` the rootfs (an absolute path, and a clear error
   if it does not exist) and check that it has a `bin` directory.
2. Build the `CString`s for `execvp` **before** `fork`, as in Lesson 13.5.
3. Get the flags with `parse_namespaces("pid,uts,mnt")`. The parent unshares **only** `CLONE_NEWPID`
   (`flags & CloneFlags::CLONE_NEWPID`); the child unshares the rest (`flags - CloneFlags::CLONE_NEWPID`) right after
   `fork`, then calls `sethostname` and `execvp`.
4. If `execvp` fails, the child prints the message from the contract and exits with 127 for `ENOENT` or 126 for any other
   error. The child must leave with `std::process::exit`, never by returning from `run`.
5. The parent turns the `WaitStatus` into a number: `Exited(_, code)` gives `code`, `Signaled(_, signal, _)` gives
   `128 + signal as i32`.
6. `main` returns `std::process::ExitCode`: `ExitCode::from(code as u8)` on success; on an error it prints
   `minibox: {e:#}` (the `#` makes `anyhow` print the whole chain of context on one line) and returns 125.

<details>
<summary>Hint</summary>

Why split the flags? `CLONE_NEWPID` changes only the namespace of the **children** created afterwards (Lesson 13.15, Section
4), but `CLONE_NEWUTS` and `CLONE_NEWNS` move the **caller** at once. If the parent took the mount namespace too, parent and
child would share it, and in M3 `pivot_root` would change the parent's root as well: the parent could then no longer find
`/sys/fs/cgroup` to clean up. Why must the child never return? It is a copy of the parent, including the values on its stack;
returning would run their `Drop` code in the child (in M4, that would remove the cgroup).

</details>

**Done when** (in the VM):

```bash
cargo build -q
sudo ./target/debug/minibox run --rootfs ~/minibox/alpine --hostname box1 -- /bin/sh -c 'hostname; echo "my PID is $$"; exit 7'
echo "exit=$?"
hostname
sudo ./target/debug/minibox run --rootfs ~/minibox/alpine -- no-such-cmd; echo "exit=$?"
./target/debug/minibox run --rootfs ~/minibox/alpine -- true; echo "exit=$?"
```

```text
box1
my PID is 1
exit=7
lima-dev
minibox: no-such-cmd: command not found
exit=127
minibox: must be run as root (try sudo)
exit=125
```

The host name changed only inside, the shell was PID 1, and its exit code came out of `minibox`. Commit:
`feat(p13-minibox): M2 namespaces and exit code`.

#### M3 — A new root file system and `/proc`

**Goal.** Inside, `/` is the Alpine rootfs, the VM's files cannot be reached any more, and `ps` sees only the container.

**Guidance.** In the child, after `sethostname` and before `execvp`, in this order:

1. **Make all mounts private:** `mount(None, "/", None, MS_REC | MS_PRIVATE, None)`, as in Lesson 13.15. On a systemd
   machine `/` is *shared*, and without this step your mounts would appear in the VM.
2. **Bind-mount the rootfs onto itself:** `mount(Some(rootfs), rootfs, None, MS_BIND | MS_REC, None)`. `pivot_root` only
   accepts a new root that is a mount point; a plain directory is not one, a directory bound onto itself is.
3. **Swap the roots:** `chdir(rootfs)`, then `pivot_root(".", ".")`, then `umount2(".", MntFlags::MNT_DETACH)`, then
   `chdir("/")`. With the same path twice, the old root is put *on top of* the new one at the same place, so the lazy
   unmount of `.` removes exactly the old root. No helper directory is needed; the man page `pivot_root(2)` shows this trick.
4. **Mount a fresh `/proc`:** `mount(Some("proc"), "/proc", Some("proc"), MS_NOSUID | MS_NODEV | MS_NOEXEC, None)`. The
   three flags forbid set-user-ID programs, device files and running programs from it, as Docker does.

Why not `chroot(rootfs)`? It is one line, and the program would see Alpine's files. But `chroot` only changes where paths
starting with `/` begin, for **this process**. The whole VM file system stays mounted in the namespace, only out of sight,
and a root process inside can get back to it with a well-known trick that the man page `chroot(2)` describes (a second
`chroot` into a subfolder, then `cd ..` past the new root). `pivot_root` plus the unmount really removes the old root from
the container's mount table.

<details>
<summary>Hint</summary>

Give every step its own `.context("...")` (for example `.context("pivot_root")`): when something fails, the message tells you
which step it was. `EINVAL` from `pivot_root` usually means a forgotten step 1, `EBUSY` a forgotten step 2. To see what the
container sees, run `cat /proc/self/mountinfo` inside.

</details>

**Done when:**

```bash
sudo ./target/debug/minibox run --rootfs ~/minibox/alpine -- /bin/sh -c 'cat /etc/alpine-release; echo /*; ps; wc -l < /proc/self/mountinfo'
findmnt | grep alpine || echo "no container mounts in the VM"
```

```text
3.24.4
/bin /dev /etc /home /lib /media /mnt /opt /proc /root /run /sbin /srv /sys /tmp /usr /var
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh -c cat /etc/alpine-release; echo /*; ps; wc -l < /proc/self/mountinfo
    3 root      0:00 ps
2
no container mounts in the VM
```

The container has exactly two mounts: its root and its `/proc`. PID 2 was `cat`, which had already finished. Commit: `feat(p13-minibox): M3 root file system and /proc`.

#### M4 — cgroup limits and clean-up

**Goal.** The `limit` program from Lesson 13.16 becomes a type: `Cgroup` creates the group with the limits, and its `Drop`
moves `minibox` back out and removes the group, on every path out of `run`.

**Guidance.**

1. `Cgroup::create(memory, cpus)`: read `/proc/self/cgroup` **first** and keep the original group's path (with your
   `cgroup_path`); create `/sys/fs/cgroup/minibox-<process::id()>`; write `memory.max` and `memory.swap.max` = `0` if
   there is a memory limit, `cpu.max` (with your `cpu_max`) if there is a CPU limit.
2. `enter()` writes `minibox`'s own PID into `cgroup.procs`. Call it before `unshare` and `fork`: the child is then born in the
   group, and every process it starts stays there.
3. After `waitpid`, read `memory.events`; if `oom_kill` is above 0, print the message from the contract.
4. `impl Drop for Cgroup`: write the PID into the **original** group's `cgroup.procs`, then `fs::remove_dir` the group. A
   `Drop` cannot return an error and should not panic, so print a message if the removal fails.

<details>
<summary>Hint</summary>

Create the `Cgroup` as a local variable in `run` and drop it explicitly after the report (`drop(cgroup)`). Because of `Drop`,
an early `?` return between `create` and `fork` also cleans up. The group is empty after `waitpid`: when PID 1 of a
namespace exits, the kernel kills all other processes in it and waits until they are gone (Lesson 13.15).

</details>

**Done when:**

```bash
R=~/minibox/alpine
sudo ./target/debug/minibox run --rootfs $R --memory 20M -- awk 'BEGIN { s = "x"; for (i = 0; i < 27; i++) s = s s; print length(s) }'; echo "exit=$?"
sudo ./target/debug/minibox run --rootfs $R -- awk 'BEGIN { s = "x"; for (i = 0; i < 27; i++) s = s s; print length(s) }'; echo "exit=$?"
sudo ./target/debug/minibox run --rootfs $R -- time awk 'BEGIN { for (i = 0; i < 5000000; i++) x++ }'
sudo ./target/debug/minibox run --rootfs $R --cpus 0.5 -- time awk 'BEGIN { for (i = 0; i < 5000000; i++) x++ }'
sudo ./target/debug/minibox run --rootfs $R -- cat /proc/self/cgroup
ls /sys/fs/cgroup | grep minibox || echo "no minibox groups left"
```

```text
minibox: the container was killed by the OOM killer
exit=137
134217728
exit=0
real	0m 1.07s
user	0m 1.07s
sys	0m 0.00s
real	0m 2.24s
user	0m 1.13s
sys	0m 0.00s
0::/minibox-3077
no minibox groups left
```

**What happens:**

1. The first `awk` program (Alpine's, from the rootfs) doubles a string 27 times, up to 128 MiB (mebibytes; 1 MiB =
   1024 × 1024 bytes). With a 20 MiB limit, the OOM killer kills it inside the group: status 137, and `minibox` reports it.
   Without a limit it prints the length.
2. `time` (also from Alpine) runs a busy loop. Without a limit, `real` (wall-clock time) equals `user` (CPU time). With
   `--cpus 0.5`, the same CPU work takes twice as long, because the group is throttled for half of every 100 ms period.
3. `/proc/self/cgroup` inside shows the group `minibox` created. The group is gone afterwards.

Commit: `feat(p13-minibox): M4 cgroup limits and clean-up`.

#### M5 — Integration tests, lint and README

**Goal.** A project you can show: tests for every row of the exit-status table, no lint warnings, a README.

**Guidance.** Most tests need root and the rootfs. Mark them with `#[ignore = "..."]`, so a plain `cargo test` skips them, and
read the rootfs path from an environment variable. The model for `tests/cli.rs`:

```rust
use assert_cmd::cargo::cargo_bin_cmd;

/// The extracted Alpine root file system, from the environment variable MINIBOX_ROOTFS.
fn rootfs() -> String {
    std::env::var("MINIBOX_ROOTFS").expect("set MINIBOX_ROOTFS to an extracted root file system")
}

#[test]
fn a_command_is_required() {
    cargo_bin_cmd!("minibox")
        .args(["run", "--rootfs", "/tmp"])
        .assert()
        .code(2);
}

#[test]
#[ignore = "needs root and MINIBOX_ROOTFS"]
fn passes_on_the_exit_code() {
    cargo_bin_cmd!("minibox")
        .args(["run", "--rootfs", &rootfs(), "--"])
        .args(["/bin/sh", "-c", "exit 7"])
        .assert()
        .code(7);
}
```

Add at least: `--cpus 0` gives 2; not root gives 125 with the exact message (return early from this test when
`geteuid().is_root()`); host name, PID 1 and `/etc/alpine-release` inside; an unknown command gives 127; a memory limit gives
137 **and** the group is removed afterwards.

How do the root tests get root? `cargo test` must not run as root (root has no Rust toolchain, and the build files would
belong to root). Instead, Cargo can put a **runner** in front of every test binary it starts. The setting is
`target.<target triple>.runner`, also available as an environment variable; your VM's triple is `aarch64-unknown-linux-gnu`
(`rustc -vV | grep host` shows it). `sudo --preserve-env=MINIBOX_ROOTFS` keeps that one variable, which `sudo` would
otherwise drop. Write both commands into your README.

<details>
<summary>Hint</summary>

For the clean-up test, let the container print its own group (`cat /proc/self/cgroup` before the memory hog, with `exec` in
front of `awk`), cut off `0::/`, and check that `/sys/fs/cgroup/<that name>` does not exist after `minibox` returns. The tests
run in parallel threads with one shared rootfs: that is a feature, because it finds mistakes that only show up when two
containers start at the same moment (see Common mistakes).

</details>

**Done when:**

```bash
cargo fmt --check && cargo clippy -q --all-targets && cargo test -q
export MINIBOX_ROOTFS=~/minibox/alpine
CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_RUNNER='sudo --preserve-env=MINIBOX_ROOTFS' cargo test -q -- --include-ignored
ls /sys/fs/cgroup | grep minibox || echo "no minibox groups left"
```

prints only test results, all `ok`, and no group is left. The reference solution has 6 unit tests and 7 integration tests;
the empty blocks for `main.rs` and the doc-tests are left out here:

```text
running 6 tests
......
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 7 tests
iiii...
test result: ok. 3 passed; 0 failed; 4 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 6 tests
......
test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 7 tests
.......
test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
no minibox groups left
```

Each `i` is an ignored test. Commit: `feat(p13-minibox): M5 tests, lint and README`.

### Stretch goals

- **A minimal `/dev`.** The exported rootfs has an almost empty `/dev`, so `sleep 30 &` inside prints
  `can't open '/dev/null'`. Before `pivot_root`, mount a `tmpfs` on `rootfs/dev` and bind-mount `null`, `zero`, `random`,
  `urandom` and `tty` from the VM's `/dev` onto empty files there.
- **`--pids <n>`** for `pids.max` (Lesson 13.16), and check that a fork bomb inside stops at the limit.
- **`--namespaces <list>`** with your `parse_namespaces`, for example adding `net` (then `ip link` inside shows only `lo`) or
  `cgroup` (then `/proc/self/cgroup` inside shows `0::/`).
- **Signals and leftovers.** Pass `SIGINT` and `SIGTERM` that reach `minibox` on to the container, and at start remove
  `minibox-*` groups whose `minibox` process no longer exists.
- **A user namespace** (`CLONE_NEWUSER`): the parent writes `/proc/<child>/uid_map` and `gid_map` (for example
  `0 1000 1`, "root inside is user 1000 outside") while the child waits on a pipe; afterwards root inside has no power over
  the VM's files. This is the base of rootless containers.
- **A clean environment** with `execvpe` (only `PATH`, `HOME`, `TERM`), and an `overlay` mount so that the rootfs itself is
  never changed by a container.

## Common mistakes

> ⚠️ **Common mistake:** The parent unshares all three namespaces. The container runs, but at the end:
>
> ```text
> minibox: cannot remove /sys/fs/cgroup/minibox-3919: No such file or directory (os error 2)
> ```
>
> Parent and child shared the mount namespace, so `pivot_root` swapped the parent's root too, and the parent looked for
> `/sys/fs/cgroup` inside Alpine. The group stays behind (remove it with `sudo rmdir`). *Fix:* the parent unshares only
> `CLONE_NEWPID`.

> ⚠️ **Common mistake:** `minibox: pivot_root: EBUSY: Device or resource busy`. The rootfs is a plain directory, not a mount
> point. *Fix:* bind-mount it onto itself first.

> ⚠️ **Common mistake:** `minibox: pivot_root: EINVAL: Invalid argument`, and afterwards `findmnt | grep alpine` in the VM
> shows a mount that nobody made there. Without `MS_PRIVATE`, the bind mount travelled to the VM's namespace through shared
> propagation, and `pivot_root` refuses shared mounts. *Fix:* make `/` private first; remove the leftover with
> `sudo umount ~/minibox/alpine`.

> ⚠️ **Common mistake:** Using a fixed helper directory such as `rootfs/.oldroot` for the old root, and removing it at the end.
> It works alone, but the parallel integration tests sometimes fail with `minibox: unmount old root: EINVAL: Invalid argument`:
> all containers share one rootfs, so one container removes the directory while another is still using it. *Fix:*
> `pivot_root(".", ".")` needs no directory.

> ⚠️ **Common mistake:** `sudo cargo run ...` prints `sudo: cargo: command not found`, because root's `PATH` has no Rust
> toolchain. *Fix:* build as yourself and run `sudo ./target/debug/minibox ...`, or use the runner from M5.

## Quiz

1. Why does the parent unshare only `CLONE_NEWPID`, while the child unshares UTS and mount?

<details>
<summary>Answer</summary>

`CLONE_NEWPID` affects only children created afterwards, so it must be done before `fork` by the parent. UTS and mount
namespaces move the caller at once; if the parent did it, it would share the child's mount namespace and lose its own root at
`pivot_root`.

</details>

2. What are the three things `pivot_root` needs, and which step of `minibox` provides each?

<details>
<summary>Answer</summary>

The new root must be a mount point (bind-mount the rootfs onto itself), the mounts involved must not be shared (make `/`
private with `MS_REC | MS_PRIVATE`), and the old root must be put somewhere under the new root (with `"."` twice it is
stacked on top of the new root, then unmounted lazily).

</details>

3. Why is `chroot` not enough for a container?

<details>
<summary>Answer</summary>

It only changes where one process starts paths beginning with `/`. The old file system stays mounted in the namespace, and
a root process can escape with a second `chroot` and `cd ..`. `pivot_root` with an unmount removes the old root from the
container's mount namespace.

</details>

4. A container ends with status 137. What happened, and how can `minibox` know it was the memory limit?

<details>
<summary>Answer</summary>

137 = 128 + 9: the command was killed by SIGKILL. If the group's `memory.events` shows `oom_kill` above 0, the OOM killer did
it because of `memory.max`.

</details>

5. Why is the clean-up in `Drop`, and why must the forked child never return from `run`?

<details>
<summary>Answer</summary>

`Drop` runs on every way out of the function, including an early `?` return, so the group is always removed. The child is a
copy of the parent with the same values on its stack; returning would run their `Drop` in the child too. It ends with
`process::exit` or `exec`, which run no destructors.

</details>

6. Why are most integration tests `#[ignore]`, and how do you run them?

<details>
<summary>Answer</summary>

They need root and an extracted rootfs, which an ordinary `cargo test` does not have. Run them with a Cargo runner:
`CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_RUNNER='sudo --preserve-env=MINIBOX_ROOTFS' cargo test -- --include-ignored`. Cargo
builds as you and starts each test binary through `sudo`.

</details>

## Summary

- A container is an ordinary process: new namespaces for the walls, a new root file system for the files, a cgroup for the budget.
- `CLONE_NEWPID` affects the children; UTS and mount namespaces affect the caller. So the parent unshares PID, and the child does the rest.
- The safe way to a new root: make mounts private, bind-mount the rootfs onto itself, `pivot_root(".", ".")`, lazy-unmount the old root, mount `/proc`.
- `chroot` changes one process's view only; the old file system stays reachable.
- A cgroup guard type with `Drop` removes the group on every path; a forked child leaves with `exec` or `process::exit`, never by returning.
- Exit codes tell the story: N from the command, 128 + signal, 127 not found, 126 not runnable, 125 the runtime's own error.
- Root-only tests are `#[ignore]` and run through a Cargo runner with `sudo`; running them in parallel finds races.

## Sources and further reading

- [pivot_root(2) — Linux manual page](https://man7.org/linux/man-pages/man2/pivot_root.2.html) — the rules for the new root and the `pivot_root(".", ".")` sequence.
- [chroot(2) — Linux manual page](https://man7.org/linux/man-pages/man2/chroot.2.html) — what `chroot` does and does not change.
- [mount_namespaces(7) — Linux manual page](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html) — shared and private propagation, the reason for `MS_PRIVATE`.
- [The Linux kernel documentation — Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) — `memory.max`, `cpu.max`, `memory.events` and `cgroup.procs`.
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec) — what a real runtime such as `runc` must do; compare it with `minibox`.
- [The Cargo Book — Configuration](https://doc.rust-lang.org/cargo/reference/config.html) — see `target.<triple>.runner`, the setting used for the root-only tests.
- [docker container export](https://docs.docker.com/reference/cli/docker/container/export/) — the command that made the rootfs.

---

[🏠 Start](../START-HERE.md) · [← 13.16 Control groups (cgroups)](16-cgroups.md) · [📋 Phase 13](README.md) · [13.18 Checkpoint →](18-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
