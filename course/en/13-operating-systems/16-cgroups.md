<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.15 Namespaces](15-namespaces.md) · [📋 Phase 13](README.md) · [13.17 Project: a mini container runtime →](17-project-mini-container-runtime.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/16-cgroups.md) <!-- nav -->

# Lesson 13.16 — Control groups (cgroups)

> **Phase 13 — Operating systems from the inside** · Lesson 16 of 18 · ⏱ about 120 minutes

## What you will learn

- What control groups are and how they differ from namespaces.
- How the cgroup v2 tree under `/sys/fs/cgroup` is organised: groups, controllers and interface files.
- How to limit memory (`memory.max`), CPU time (`cpu.max`) and the number of processes (`pids.max`), and how to see when a limit was hit.
- How to put a program into a cgroup from Rust, which is the last building block for the container project.

## Before you start

- You finished [Lesson 13.15 — Namespaces](15-namespaces.md).
- You remember the OOM (Out Of Memory) killer and RSS (Resident Set Size, the memory of a process that is really in RAM) ([Lesson 13.10](10-virtual-memory.md)), the scheduler ([Lesson 13.9](09-threads-and-scheduling.md)) and systemd units ([Lesson 11.6](../11-linux-shell-and-containers/06-services-with-systemd.md)).
- Create the package on the Mac and copy it to the VM as in Lesson 13.10:

```bash
cd ~/projects/rust
cargo new --lib code/p13-cgroup
mkdir code/p13-cgroup/src/bin
```

Writing to `/sys/fs/cgroup` needs root, so commands use `sudo`, **in the VM only**. Build the examples in
release mode (`cargo build --release`) so the CPU measurements are meaningful. The test machine was a
privileged container, so some paths differ from your VM; outputs that depend on your VM are marked as
examples.

## New words

| Word | Meaning |
|---|---|
| **cgroup** (control group) | A group of processes whose use of resources (memory, CPU (Central Processing Unit) time, I/O (input/output: disk reads and writes), number of processes) the kernel counts and can limit. |
| **cgroup v2** | The current version: one single tree for all resources, mounted at `/sys/fs/cgroup`. (Version 1 had a separate tree per resource.) |
| **controller** | The part of the kernel that manages one resource for cgroups: `memory`, `cpu`, `io`, `pids`, … |
| **interface file** | A file in a cgroup directory that shows a value or sets a limit, for example `memory.max`. |
| **`cgroup.procs`** | The interface file that lists the processes of a group; writing a PID into it moves that process into the group. |
| **throttling** | Making a group wait because it used up its CPU time for the current period. |
| **OOM kill** (Out Of Memory) | Killing a process because its group reached its memory limit and the kernel could not free enough. |
| **slice / scope** | systemd's names for the cgroups it creates: a *slice* groups services or users, a *scope* holds processes that were started outside systemd. |

## 1. Namespaces say *what you see*, cgroups say *how much you use*

Namespaces (Lesson 13.15) changed what a process can **see**. They do not stop it from using all the memory
or all the CPU of the machine. A container that leaks memory would still push the whole machine into the
OOM killer.

**Control groups** fill that gap. A cgroup is a group of processes with a budget:

- The kernel **counts** what the group uses (memory, CPU time, disk I/O, number of processes).
- It **enforces** limits you set: the group cannot get more memory than `memory.max`, more CPU time than
  `cpu.max`, more processes than `pids.max`.
- Children of a process start in the same cgroup, so a limit covers a whole program and everything it starts.

Every container runtime uses both: namespaces for the walls, cgroups for the budget. When Docker shows a
container as `OOMKilled`, or Kubernetes enforces `resources.limits`, it is the cgroup memory controller at
work.

The interface is a file system. There is no special system call: you create a group with `mkdir`, set a
limit with `echo`, and move a process with another `echo`.

## 2. Look around the tree

Every process belongs to exactly one cgroup. In the VM:

```bash
cat /proc/self/cgroup
```

Example output (yours will look similar):

```text
0::/user.slice/user-501.slice/session-1.scope
```

`0::` means cgroup v2, and the rest is the path inside the tree. systemd created it: your login session is
a *scope* inside your user's *slice*. The tree itself is mounted at `/sys/fs/cgroup`:

```bash
ls /sys/fs/cgroup
cat /sys/fs/cgroup/cgroup.controllers
cat /sys/fs/cgroup/cgroup.subtree_control
```

Example output of the last two commands (yours will look similar):

```text
cpuset cpu io memory hugetlb pids rdma misc
cpuset cpu io memory pids
```

- `cgroup.controllers` lists the controllers available in this group.
- `cgroup.subtree_control` lists the controllers **switched on for the children** of this group. systemd
  has already switched on `cpu`, `memory` and `pids` at the root, so new groups directly under
  `/sys/fs/cgroup` get those interface files.

> 💡 **Tip:** `systemd-cgls` prints the whole tree with the processes in it, and `systemd-cgtop` shows live
> usage per group, like `top`.

## 3. Your first cgroup: a memory limit

Create a group. The kernel fills the new directory with interface files at once:

```bash
sudo mkdir /sys/fs/cgroup/course-demo
ls /sys/fs/cgroup/course-demo
```

The output on the test machine, shortened:

```text
cgroup.controllers cgroup.events cgroup.freeze cgroup.kill ... cgroup.procs ... cpu.max cpu.stat ...
memory.current memory.events ... memory.max memory.peak ... memory.swap.max ... pids.current pids.events pids.max ...
```

All limits start at "no limit":

```bash
cat /sys/fs/cgroup/course-demo/memory.max /sys/fs/cgroup/course-demo/cpu.max /sys/fs/cgroup/course-demo/pids.max
```

```text
max
max 100000
max
```

Now a program that eats memory. Create `src/bin/hog.rs`:

```rust
use std::thread;
use std::time::Duration;

fn main() {
    const STEP: usize = 10 * 1024 * 1024; // 10 MiB
    let mut chunks: Vec<Vec<u8>> = Vec::new();
    for i in 1..=20 {
        chunks.push(vec![1u8; STEP]); // vec![1; …] really writes every byte, so it uses RAM
        println!("allocated {:>3} MiB", i * 10);
        thread::sleep(Duration::from_millis(100));
    }
    println!("done, total {} MiB", chunks.len() * 10);
}
```

It allocates 200 MiB in steps of 10 MiB. Remember demand paging (Lesson 13.10): `vec![0u8; …]` would not use
RAM until touched, so we fill with ones. Set a limit of 50 MiB, forbid swap for this group, and start the
program inside the group:

```bash
echo 50M | sudo tee /sys/fs/cgroup/course-demo/memory.max
echo 0 | sudo tee /sys/fs/cgroup/course-demo/memory.swap.max
sudo sh -c 'echo $$ > /sys/fs/cgroup/course-demo/cgroup.procs; exec ./target/release/hog'
echo "exit=$?"
```

```text
50M
0
allocated  10 MiB
allocated  20 MiB
allocated  30 MiB
allocated  40 MiB
Killed
exit=137
```

**What happens:**

1. `tee` writes its input into the file; `sudo tee` is the usual way to write a root-owned file from a
   normal shell (`sudo echo 50M > file` would not work, because the redirection is done by your own shell).
   The kernel accepts `50M` and stores it as bytes (`cat memory.max` now shows `52428800`).
2. `memory.swap.max = 0` stops the group from moving pages to swap, so the limit is reached quickly.
3. `sh -c 'echo $$ > …/cgroup.procs; exec …'` starts a small shell, moves **that shell** into the group
   (`$$` is its PID), and then replaces it with `hog` using `exec` (Lesson 13.5). `hog` therefore runs
   inside the group from its first instruction.
4. At about 50 MiB, the group cannot get more memory. The kernel tries to free memory inside the group,
   fails, and the **OOM killer kills a process of this group only**. The rest of the VM is not touched.
5. Exit status 137 = 128 + 9: SIGKILL.

The group keeps a record of what happened:

```bash
cat /sys/fs/cgroup/course-demo/memory.events
```

```text
low 0
high 0
max 25
oom 1
oom_kill 1
oom_group_kill 0
sock_throttled 0
```

`max 25` counts how often the group hit its limit (the kernel then tried to reclaim memory), and
`oom_kill 1` shows that one process was killed. This file is what tools read to report "OOMKilled".

## 4. A CPU limit

`cpu.max` holds two numbers: a **quota** and a **period**, both in microseconds. `20000 100000` means "in
every period of 100 ms, the group may run for 20 ms in total": 20% of one CPU. When the quota is used up, the
group's threads wait for the next period. That waiting is called **throttling**. Create `src/bin/spin.rs`:

```rust
use std::time::{Duration, Instant};

fn main() {
    for second in 1..=3 {
        let start = Instant::now();
        let mut loops: u64 = 0;
        while start.elapsed() < Duration::from_secs(1) {
            loops += 1; // pure CPU work
        }
        println!("second {second}: {:>5} million loops", loops / 1_000_000);
    }
}
```

Run it once without a limit, then inside the group with a 20% limit:

```bash
./target/release/spin
echo "20000 100000" | sudo tee /sys/fs/cgroup/course-demo/cpu.max
sudo sh -c 'echo $$ > /sys/fs/cgroup/course-demo/cgroup.procs; exec ./target/release/spin'
cat /sys/fs/cgroup/course-demo/cpu.stat
```

```text
second 1:    50 million loops
second 2:    51 million loops
second 3:    51 million loops
20000 100000
second 1:     9 million loops
second 2:     6 million loops
second 3:     7 million loops
usage_usec 623400
user_usec 613629
system_usec 9771
nice_usec 0
nr_periods 30
nr_throttled 30
throttled_usec 2388853
nr_bursts 0
burst_usec 0
```

**What happens:**

1. Without a limit, the loop runs about 50 million times per second (on the test machine).
2. With 20%, it gets far less: the program still runs for three seconds of *wall-clock* time, but it is
   allowed to use the CPU for only a fifth of it. The numbers are not exactly 10 million, because the program
   also measures time, and the throttling happens in 100 ms slices.
3. `cpu.stat` confirms it: in 30 periods the group was throttled 30 times, waiting about 2.4 seconds in total
   (`throttled_usec`), and it used about 0.6 seconds of CPU (`usage_usec`).

A quota bigger than the period gives more than one CPU: `200000 100000` allows two full CPUs if the
program has two busy threads. This is what `docker run --cpus=2` and a Kubernetes CPU limit of `2` write.

> ⚡ **Performance:** a CPU limit does not make a program slower per instruction; it makes it *wait*. A web
> service with a tight CPU limit can show long response times (tail latency) even when the machine is idle.
> Check `nr_throttled` before you blame your code.

## 5. Limiting the number of processes

`pids.max` stops a group from creating too many processes or threads, for example a *fork bomb* (a program
that copies itself again and again until the machine cannot start anything):

```bash
echo 5 | sudo tee /sys/fs/cgroup/course-demo/pids.max
sudo sh -c 'echo $$ > /sys/fs/cgroup/course-demo/cgroup.procs; for i in 1 2 3 4 5 6; do sleep 1 & done; wait'
echo "exit=$?"
cat /sys/fs/cgroup/course-demo/pids.events
```

```text
5
sh: 0: Cannot fork
exit=2
max 1
```

The shell itself and four `sleep` processes make five; the fifth `sleep` could not be created, and `fork`
failed with an error instead of the machine being flooded. `pids.events` counts how often the limit was hit.

When you are done, wait until the group is empty, then remove it. A cgroup can only be removed with `rmdir`
when no process is in it:

```bash
sleep 2
cat /sys/fs/cgroup/course-demo/pids.current
sudo rmdir /sys/fs/cgroup/course-demo
```

```text
0
```

## 6. Doing it from Rust

There is nothing special to do: cgroups are files, so `std::fs` is enough. A program that wants to limit
a command:

1. creates a directory under `/sys/fs/cgroup`,
2. writes the limits into `memory.max`, `cpu.max`, …,
3. writes a PID into `cgroup.procs`: either its own (then every child it starts is inside too) or a child's,
4. after the work, moves itself back out and removes the directory.

You will write exactly this as Exercise 3, and use it in the next lesson's project.

**Who else does this?** `systemd-run --scope -p MemoryMax=50M -p CPUQuota=20% <command>` asks systemd to
create such a scope for you. On a system run by systemd, that is the polite way, because systemd considers
the tree its own. Docker's `--memory` and `--cpus` options and Kubernetes `resources.limits` end up as the same
`memory.max` and `cpu.max` files.

> 🔒 **Security:** without limits, one buggy or hostile program can starve all others (a *denial of
> service*). Setting `memory.max`, `cpu.max` and `pids.max` for every service and container is basic
> hygiene, and systemd units can do it with `MemoryMax=`, `CPUQuota=` and `TasksMax=`.

## Common mistakes

1. **`sudo echo 50M > memory.max` says "Permission denied".** The redirection is done by your normal shell
   before `sudo` runs. *Fix:* `echo 50M | sudo tee …/memory.max`, or `sudo sh -c 'echo 50M > …'`.

2. **No `memory.max` file in your new group.** The `memory` controller is not switched on in the parent's
   `cgroup.subtree_control`. *Fix:* check the parent; at the root of a systemd machine it is usually on.

3. **`rmdir` fails with "Device or resource busy".** There are still processes in the group (perhaps your
   own shell, if you wrote `$$` of your login shell into `cgroup.procs`). *Fix:* move them out (write their
   PIDs into another group's `cgroup.procs`) or wait for them to end.

4. **A memory limit that "does not work".** The program only reserved memory (demand paging) and never
   touched it, or the group swaps instead of failing. *Fix:* touch the memory, and set `memory.swap.max` to 0
   in experiments.

## Try it yourself

1. Look at what your login session uses: `cat /sys/fs/cgroup$(cut -d: -f3 /proc/self/cgroup)/memory.current`.
   (`cut -d: -f3` takes the third `:`-separated field of `0::/…`, the path.)
2. Run `systemd-cgls --no-pager | head -30` and find your shell in the tree.
3. Run `hog` with `systemd-run --scope -p MemoryMax=50M -p MemorySwapMax=0 ./target/release/hog` (with
   `sudo`). Is the result the same as in Section 3?

## Exercises

### Exercise 1 — How much does my shell use? (easy)

Print the current and the peak memory use of the cgroup of your shell, using only `/proc/self/cgroup` and
`/sys/fs/cgroup`. Why is the number much larger than the RSS of the shell?

<details>
<summary>Hint</summary>

Build the path from the third field of `/proc/self/cgroup`, then read `memory.current` and `memory.peak`.

</details>

<details>
<summary>Answer</summary>

```bash
cat /sys/fs/cgroup$(cut -d: -f3 /proc/self/cgroup)/memory.current
cat /sys/fs/cgroup$(cut -d: -f3 /proc/self/cgroup)/memory.peak
```

On the test machine (a small container) the output was:

```text
856064
1425408
```

Why it works: a cgroup counts the memory of **all** its processes, and not only their RSS but also kernel
memory used for them and file pages in the page cache that they brought in. Your login session's group
contains every program you started from it.

</details>

### Exercise 2 — Tested helpers (medium)

Write two functions in `src/lib.rs` with tests:

- `pub fn cpu_max(percent: u32) -> String`: the `cpu.max` text for a share of one CPU with a 100 ms period;
  `20` → `"20000 100000"`, `250` → `"250000 100000"`.
- `pub fn cgroup_path(proc_cgroup: &str) -> Option<&str>`: the path from the text of `/proc/<pid>/cgroup`
  (the line starting with `0::`), or `None` if there is no v2 line.

<details>
<summary>Hint</summary>

`line.strip_prefix("0::")` returns `Some(rest)` only for the right line; `find_map` stops at the first
`Some`.

</details>

<details>
<summary>Answer</summary>

```rust
/// The value for `cpu.max` that allows `percent` % of one CPU per 100 ms period.
/// 100 means one whole CPU, 250 means two and a half CPUs.
pub fn cpu_max(percent: u32) -> String {
    const PERIOD_US: u32 = 100_000;
    format!("{} {PERIOD_US}", percent * PERIOD_US / 100)
}

/// Finds the cgroup v2 path in the text of /proc/<pid>/cgroup, for example "/user.slice".
pub fn cgroup_path(proc_cgroup: &str) -> Option<&str> {
    proc_cgroup
        .lines()
        .find_map(|line| line.strip_prefix("0::"))
        .map(str::trim)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn cpu_max_for_common_values() {
        assert_eq!(cpu_max(20), "20000 100000");
        assert_eq!(cpu_max(100), "100000 100000");
        assert_eq!(cpu_max(250), "250000 100000");
    }

    #[test]
    fn finds_the_v2_line() {
        let text = "0::/user.slice/user-1000.slice/session-3.scope\n";
        assert_eq!(
            cgroup_path(text),
            Some("/user.slice/user-1000.slice/session-3.scope")
        );
        assert_eq!(cgroup_path("12:pids:/old-v1-line\n"), None);
    }
}
```

`cargo test`:

```text
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: both functions only work on text, so they can be tested without root. All code that touches
`/sys/fs/cgroup` stays small and thin around them.

</details>

### Exercise 3 — `limit`: run a command with limits (hard)

Write `src/bin/limit.rs`: `limit <memory> <cpu-percent> <command> [args...]`. It creates
`/sys/fs/cgroup/course-limit-<its PID>`, writes `memory.max`, `memory.swap.max = 0` and `cpu.max` (use your
helpers), moves itself in, runs the command and waits for it, then prints the command's status and the
`oom_kill` line of `memory.events`, moves itself back to its original group and removes the group. Check:
`sudo ./target/release/limit 50M 100 ./target/release/hog` reports an OOM kill, `300M` does not, and no
`course-limit-*` group is left afterwards.

<details>
<summary>Hint</summary>

Read `/proc/self/cgroup` **before** you move, so you know where to go back. `Command::new(&args[2]).args(&args[3..]).status()`
runs the command and waits.

</details>

<details>
<summary>Answer</summary>

```rust
use std::env;
use std::fs;
use std::path::PathBuf;
use std::process::{self, Command};

use p13_cgroup::{cgroup_path, cpu_max};

fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().skip(1).collect();
    if args.len() < 3 {
        eprintln!("usage: limit <memory, e.g. 50M> <cpu percent> <command> [args...]");
        process::exit(2);
    }
    let root = PathBuf::from("/sys/fs/cgroup");
    let me = process::id().to_string();

    // Remember where we are now, so we can go back.
    let original = fs::read_to_string("/proc/self/cgroup")?;
    let original = root.join(
        cgroup_path(&original)
            .unwrap_or("/")
            .trim_start_matches('/'),
    );

    // 1. Create the group and set the limits.
    let group = root.join(format!("course-limit-{me}"));
    fs::create_dir(&group)?;
    fs::write(group.join("memory.max"), &args[0])?;
    fs::write(group.join("memory.swap.max"), "0")?;
    fs::write(
        group.join("cpu.max"),
        cpu_max(args[1].parse().expect("cpu percent")),
    )?;

    // 2. Move ourselves in; the child we start inherits the group.
    fs::write(group.join("cgroup.procs"), &me)?;
    let status = Command::new(&args[2]).args(&args[3..]).status()?;

    // 3. Report, move back out, remove the group.
    let events = fs::read_to_string(group.join("memory.events"))?;
    let oom_kills = events
        .lines()
        .find(|l| l.starts_with("oom_kill "))
        .unwrap_or("?");
    fs::write(original.join("cgroup.procs"), &me)?;
    fs::remove_dir(&group)?;
    println!("limit: command finished with {status}; {oom_kills}");
    Ok(())
}
```

```bash
sudo ./target/release/limit 50M 100 ./target/release/hog
sudo ./target/release/limit 300M 100 ./target/release/hog | tail -2
ls /sys/fs/cgroup | grep course || echo "no course groups left"
```

```text
allocated  10 MiB
allocated  20 MiB
allocated  30 MiB
allocated  40 MiB
limit: command finished with signal: 9 (SIGKILL); oom_kill 1
done, total 200 MiB
limit: command finished with exit status: 0; oom_kill 0
no course groups left
```

Why it works: a child process starts in its parent's cgroup, so `hog` is limited from its first
instruction. The OOM killer chose `hog`, the biggest process in the group, and `limit` survived to report
it. Moving back out first is necessary, because a group with a process in it cannot be removed. A small
weakness remains: if `limit` itself crashes in the middle, the group stays behind; the project in the next
lesson should clean up such leftovers.

</details>

## Quiz

1. What is the difference between a namespace and a cgroup?

<details>
<summary>Answer</summary>

A namespace changes what processes can *see* (their own host name, process list, mounts…). A cgroup
controls how much they can *use* (memory, CPU time, number of processes) and counts their usage.

</details>

2. How do you move a running process into a cgroup?

<details>
<summary>Answer</summary>

Write its PID into the group's `cgroup.procs` file. Its future children will start in the same group.

</details>

3. What does `cpu.max` = `50000 100000` mean?

<details>
<summary>Answer</summary>

In every 100 ms period, the group may use 50 ms of CPU time in total: half of one CPU. When the quota is
used, its threads are throttled until the next period.

</details>

4. A process in a group with `memory.max = 100M` touches 150 MiB of memory and cannot swap. What happens,
   and where can you see it?

<details>
<summary>Answer</summary>

The kernel tries to reclaim memory in the group; when it cannot, the OOM killer kills a process of that
group (exit status 137, SIGKILL). `memory.events` shows `oom_kill 1`. Processes outside the group are not
affected.

</details>

5. Why can `rmdir` of a cgroup fail with "Device or resource busy"?

<details>
<summary>Answer</summary>

There are still processes in the group. It must be empty before it can be removed.

</details>

## Summary

- A cgroup is a group of processes whose resource use the kernel counts and limits; children inherit their parent's group.
- cgroup v2 is one tree at `/sys/fs/cgroup`; groups are directories, settings are interface files, and controllers must be switched on in the parent's `cgroup.subtree_control`.
- `memory.max` caps memory (the OOM killer acts inside the group; see `memory.events`), `cpu.max` sets quota and period (see `cpu.stat`), `pids.max` caps the number of processes.
- `cgroup.procs` moves processes; a group can be removed only when it is empty.
- From Rust, cgroups are plain file operations; systemd, Docker and Kubernetes write the same files.

## Sources and further reading

- [The Linux kernel documentation — Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) — the full reference for every interface file used here.
- [cgroups(7) — Linux manual page](https://man7.org/linux/man-pages/man7/cgroups.7.html) — an overview of cgroups v1 and v2.
- [systemd.resource-control(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html) — `MemoryMax=`, `CPUQuota=`, `TasksMax=` and how systemd maps them to cgroups.
- [Docker documentation — Resource constraints](https://docs.docker.com/engine/containers/resource_constraints/) — how `--memory` and `--cpus` relate to cgroups.
- [Kubernetes documentation — Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) — requests and limits, for later phases.

---

[🏠 Start](../START-HERE.md) · [← 13.15 Namespaces](15-namespaces.md) · [📋 Phase 13](README.md) · [13.17 Project: a mini container runtime →](17-project-mini-container-runtime.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
