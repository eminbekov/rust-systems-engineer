<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.14 io_uring](14-io-uring.md) · [📋 Phase 13](README.md) · [13.16 Control groups (cgroups) →](16-cgroups.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/15-namespaces.md) <!-- nav -->

# Lesson 13.15 — Namespaces

> **Phase 13 — Operating systems from the inside** · Lesson 15 of 18 · ⏱ about 135 minutes

## What you will learn

- What a Linux namespace is, and which kinds exist.
- How to see the namespaces of a process in `/proc/<pid>/ns`.
- How to create new UTS, PID and mount namespaces with the `unshare` command and from Rust with `nix`.
- Why the first process in a PID namespace is special.
- How namespaces are the "walls" of every container, and what they do *not* do.

## Before you start

- You finished [Lesson 13.14 — io_uring](14-io-uring.md).
- You know `fork` and `waitpid` ([Lesson 13.5](05-fork-exec-and-wait.md)), signals ([Lesson 13.7](07-signals-in-your-programs.md)), mounts and `/proc` ([Lesson 13.12](12-file-systems-inside.md)), and what a container is from the outside ([Lesson 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)).
- Create the package on the Mac and copy it to the VM as in Lesson 13.10:

```bash
cd ~/projects/rust
cargo new code/p13-ns
cd code/p13-ns
cargo add nix@0.32 --features sched,hostname,process,mount,fs
rm src/main.rs && mkdir src/bin
```

Creating most namespaces needs root, so you run the built programs with `sudo` **in the VM**, never on your
Mac. The outputs below come from a test container whose host name is `course-ns-demo`; your VM is called
`lima-dev`.

## New words

| Word | Meaning |
|---|---|
| **namespace** | A kernel feature that gives a group of processes their own private view of one global resource, such as the host name, the process list or the mount table. |
| **UTS namespace** (UNIX Time-Sharing) | Isolates the host name and domain name. The odd name comes from an old kernel structure, `utsname`. |
| **PID namespace** (Process ID) | Gives processes their own numbering. The first process inside gets PID 1. |
| **mount namespace** | Gives processes their own copy of the mount table, so mounts inside are invisible outside. |
| **network namespace** | Gives processes their own network interfaces, addresses and routes. |
| **user namespace** | Maps user IDs: a process can be "root" inside while being an ordinary user outside. |
| **`unshare`** | A system call (and a command) that moves the calling process into new namespaces. |
| **init process** | PID 1: the first process of a system or of a PID namespace. It adopts orphans and is protected from most signals. |
| **mount propagation** | The rule that decides whether a mount made in one mount namespace appears in others. `private` means "no". |
| **capability** | One piece of root's power, for example `CAP_SYS_ADMIN` for mounting. Creating namespaces needs it (except user namespaces). |

## 1. What a namespace is

Your VM has one kernel, one list of processes and one host name. A **namespace** wraps one of these global
resources so that a group of processes sees **its own private copy**. Inside, everything looks normal;
outside, nothing changes.

An analogy: a large office building (the kernel) with many companies. Each company has its own reception,
its own room numbers and its own phone list. Employees of company A see "room 1", "room 2"…; they do not see
company B's rooms at all, although all rooms are in the same building.

Linux has eight kinds:

| Namespace | Flag | What it isolates |
|---|---|---|
| mount | `CLONE_NEWNS` | the mount table: which file systems are mounted where |
| UTS | `CLONE_NEWUTS` | host name and domain name |
| IPC (Inter-Process Communication) | `CLONE_NEWIPC` | System V shared memory and message queues (old Unix ways for processes to exchange data) |
| PID | `CLONE_NEWPID` | process numbers |
| network | `CLONE_NEWNET` | network interfaces, IP (Internet Protocol) addresses, routes, ports |
| user | `CLONE_NEWUSER` | user and group IDs |
| cgroup | `CLONE_NEWCGROUP` | the view of the cgroup tree (Lesson 13.16) |
| time | `CLONE_NEWTIME` | some clocks (the time since boot) |

(The mount namespace was the first one, from 2002, so its flag got the general name `NEWNS`.)

Every process belongs to exactly one namespace of each kind. The kernel shows them as symbolic links in
`/proc/<pid>/ns`:

```bash
ls -l /proc/self/ns | awk 'NR>1 {print $9, $10, $11}'
```

```text
cgroup -> cgroup:[4026533114]
ipc -> ipc:[4026533110]
mnt -> mnt:[4026533108]
net -> net:[4026533115]
pid -> pid:[4026533113]
pid_for_children -> pid:[4026533113]
time -> time:[4026533243]
time_for_children -> time:[4026533243]
user -> user:[4026531837]
uts -> uts:[4026533109]
```

The number in brackets is an inode number that identifies the namespace. Two processes are in the same
namespace exactly when the numbers are equal. (`awk 'NR>1 {...}'` skips the first line of `ls -l` and prints
three columns.)

## 2. Try it with the `unshare` command

The command `unshare` (from util-linux, already installed) starts a program in new namespaces. In the VM:

```bash
hostname
sudo unshare --uts sh -c 'hostname box2; hostname'
hostname
```

```text
course-ns-demo
box2
course-ns-demo
```

The shell inside got its own UTS namespace. Changing the host name there did not change it outside.

Now a PID namespace. `--fork` starts the program as a child (Section 4 explains why), and `--mount-proc`
mounts a fresh `/proc` so that `ps` sees the new numbering:

```bash
sudo unshare --pid --fork --mount-proc sh -c 'echo "PID of this shell: $$"; ps -o pid,comm'
```

```text
PID of this shell: 1
    PID COMMAND
      1 sh
      2 ps
```

The shell believes it is PID 1, the first process of a system, and sees only itself and `ps`. This is the
same picture you get with `docker run ... ps`.

## 3. UTS namespace from Rust

The system call `unshare(flags)` moves the **calling process** into new namespaces of the given kinds. Create
`src/bin/uts.rs`:

```rust
use nix::sched::{CloneFlags, unshare};
use nix::unistd::{gethostname, sethostname};

fn main() -> nix::Result<()> {
    println!("before: hostname = {:?}", gethostname()?);

    unshare(CloneFlags::CLONE_NEWUTS)?; // from now on this process has its own UTS namespace
    sethostname("inside-box")?;
    println!("after : hostname = {:?}", gethostname()?);

    let ns = std::fs::read_link("/proc/self/ns/uts").expect("read_link failed");
    println!("my UTS namespace: {ns:?}");
    Ok(())
}
```

Build it in `~/p13-ns`, then run the binary with `sudo` and look at the host name afterwards:

```bash
cargo build
sudo ./target/debug/uts
hostname
readlink /proc/self/ns/uts
```

```text
before: hostname = "course-ns-demo"
after : hostname = "inside-box"
my UTS namespace: "uts:[4026533379]"
course-ns-demo
uts:[4026533109]
```

**What happens:**

1. `main` returns `nix::Result<()>`, so `?` works on every `nix` call, and an error is printed as
   `Error: EPERM` or similar.
2. `unshare(CLONE_NEWUTS)` creates a new UTS namespace, a copy of the old one, and moves this process into
   it.
3. `sethostname` changes the host name **in the new namespace only**.
4. The namespace number of the program (`4026533379`) differs from the shell's (`4026533109`). When the
   program exits, the last member of the namespace is gone, and the kernel deletes it.

We run the binary directly, not with `sudo cargo run`, because `root` usually has no Rust toolchain in its
`PATH`.

## 4. PID namespace and the special PID 1

`unshare(CLONE_NEWPID)` is different: the calling process **stays** in its old PID namespace (its PID
cannot change while it runs). Only the **children** created afterwards are born inside the new one, and
the first child becomes its PID 1. That is why the command needed `--fork`. Create `src/bin/pidns.rs`:

```rust
use nix::sched::{CloneFlags, unshare};
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, fork, getpid, getppid};

fn main() -> nix::Result<()> {
    println!("parent: my PID is {}", getpid());
    unshare(CloneFlags::CLONE_NEWPID)?; // affects the CHILDREN created from now on

    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            println!(
                "child : my PID is {}, my parent's PID is {}",
                getpid(),
                getppid()
            );
            std::process::exit(0);
        }
        ForkResult::Parent { child } => {
            println!("parent: the child's PID is {child}");
            waitpid(child, None)?;
        }
    }
    Ok(())
}
```

`sudo ./target/debug/pidns`:

```text
parent: my PID is 234
parent: the child's PID is 235
child : my PID is 1, my parent's PID is 0
```

**What happens:**

1. The parent is PID 234 in the outer namespace, and it stays so.
2. `fork` creates the child inside the new PID namespace. The **same process** has two numbers: 235 seen
   from outside, 1 seen from inside. PID namespaces are nested: the outer one sees everything, the inner
   one sees only its own processes.
3. The child's parent lives outside its namespace, so inside there is no number for it: `getppid()`
   returns 0.

**Why PID 1 is special.** In every PID namespace, PID 1 plays the role of the *init process*:

- **Orphans are adopted by it.** When a process dies, its children get PID 1 as their new parent, and PID 1
  must `wait` for them, or they stay as zombies (Lesson 13.5).
- **Signals are filtered.** Processes inside the namespace cannot kill PID 1 with a signal for which PID 1
  has not installed a handler, not even with SIGKILL (only SIGKILL or SIGSTOP sent from *outside*, from the
  parent namespace, always gets through):

  ```bash
  sudo unshare --pid --fork --mount-proc sh -c 'kill -TERM 1; echo still alive; kill -KILL 1; echo still alive after KILL'
  ```

  ```text
  still alive
  still alive after KILL
  ```

- **When PID 1 exits, the kernel kills every other process of the namespace.**

This explains a classic container problem: `docker stop` sends SIGTERM to the container's PID 1. If your
program runs as PID 1 and does not handle SIGTERM ([Lesson 13.7](07-signals-in-your-programs.md)), the
signal is ignored, and Docker kills it with SIGKILL after 10 seconds. Small init programs such as `tini`
(Docker's `--init` option) exist for exactly this reason.

## 5. Mount namespace

A mount namespace gives a process its own copy of the mount table. One detail matters: on most systems the
root mount is **shared**, which means mounts propagate between namespaces (systemd sets it up like this).
A new mount namespace starts as a copy that is still connected. So the first step inside is to make all
mounts **private**. Create `src/bin/mntns.rs`:

```rust
use std::fs;
use std::process::Command;

use nix::mount::{MsFlags, mount};
use nix::sched::{CloneFlags, unshare};

fn main() -> nix::Result<()> {
    unshare(CloneFlags::CLONE_NEWNS)?; // our own copy of the mount table

    // Stop mount events from travelling back to the parent namespace.
    mount(
        None::<&str>,
        "/",
        None::<&str>,
        MsFlags::MS_REC | MsFlags::MS_PRIVATE,
        None::<&str>,
    )?;

    fs::create_dir_all("/mnt/private").expect("mkdir failed");
    mount(
        Some("tmpfs"),
        "/mnt/private",
        Some("tmpfs"),
        MsFlags::empty(),
        Some("size=1m"),
    )?;
    fs::write("/mnt/private/note.txt", "only visible in here\n").expect("write failed");

    let out = Command::new("findmnt")
        .args(["-n", "/mnt/private"])
        .output()
        .expect("findmnt");
    print!("inside : {}", String::from_utf8_lossy(&out.stdout));
    print!(
        "inside : note.txt says {}",
        fs::read_to_string("/mnt/private/note.txt").unwrap()
    );
    Ok(())
}
```

```bash
sudo ./target/debug/mntns
findmnt -n /mnt/private || echo 'not mounted'
ls /mnt/private
```

```text
inside : /mnt/private tmpfs tmpfs rw,relatime,size=1024k
inside : note.txt says only visible in here
not mounted
```

**What happens:**

1. `unshare(CLONE_NEWNS)` copies the mount table for this process.
2. `mount(None, "/", None, MS_REC | MS_PRIVATE, None)` does not mount anything new. It changes the
   *propagation* of `/` and everything below it (`MS_REC`, recursive) to private.
3. The second `mount` mounts a 1 MiB **tmpfs** (a file system in RAM) on `/mnt/private`, and we write a file
   into it. The child process `findmnt` inherits our namespace and sees the mount.
4. After the program exits, the shell's namespace has no mount on `/mnt/private`, and `ls` shows the empty
   directory. The tmpfs and its file disappeared with the namespace.

This is how a container gets its own root file system: a new mount namespace, then a different root
(`pivot_root` or `chroot`), then a fresh `/proc`. You will do all three in the project,
[Lesson 13.17](17-project-mini-container-runtime.md).

## 6. Network and user namespaces, briefly

**Network namespace.** A new one contains only a loopback interface, switched off:

```bash
sudo unshare --net ip -brief link
```

The first line of the output:

```text
lo               DOWN           00:00:00:00:00:00 <LOOPBACK>
```

(On the test machine a few tunnel devices such as `tunl0` also appeared, created by kernel modules; your VM
may show only `lo`.) Container runtimes then create a pair of virtual interfaces, one end inside and one end
on a bridge outside, to give the container a network. You will learn in Lesson 15.2 (Phase 15)
about the interfaces, addresses and routes that such a setup is built from.

**User namespace.** Inside a user namespace, a process can have user ID 0, root, while outside it is an
ordinary user. This is the only namespace that a normal user may create without `sudo`:

```bash
unshare --user --map-root-user sh -c id
```

On the test machine, run as an ordinary user, the output was:

```text
uid=0(root) gid=0(root) groups=0(root)
```

"Root" inside a user namespace has power only over things that belong to that namespace; it cannot, for
example, read another user's files. Rootless containers (Podman, rootless Docker) are built on this. Some
distributions limit it: recent Ubuntu versions use AppArmor rules that can make this command fail or give
the new "root" fewer rights.

> 🔒 **Security:** namespaces are walls, not a full security boundary. All containers share **one kernel**,
> so a kernel bug can break through the walls. Real runtimes add more layers: capabilities are dropped,
> seccomp filters system calls (Lesson 13.14), and cgroups limit resources (Lesson 13.16). A process that
> is root in the outer namespace and runs in a container with `--privileged` is nearly root on the host.

## Common mistakes

1. **Running without root.** As an ordinary user, `./target/debug/uts` prints

   ```text
   before: hostname = "613981bb33ae"
   Error: EPERM
   ```

   because creating a UTS namespace needs the capability `CAP_SYS_ADMIN`. *Fix:* use `sudo` in the VM
   (or combine with a user namespace, which is more advanced).

2. **Expecting `unshare(CLONE_NEWPID)` to change your own PID.** `getpid()` stays the same; only children
   are inside. *Fix:* `fork` after `unshare`, and do the work in the child.

3. **Forgetting `MS_PRIVATE`.** Without it, a mount made inside a new mount namespace can appear in the
   VM's own namespace too (with shared propagation). *Fix:* make `/` private recursively right after
   `unshare(CLONE_NEWNS)`.

4. **`ps` shows the old processes inside a new PID namespace.** `ps` reads `/proc`, and the old `/proc`
   belongs to the old namespace. *Fix:* mount a new `proc` file system inside (`--mount-proc`, or
   `mount("proc", "/proc", "proc", …)` in a new mount namespace).

## Try it yourself

1. Find two processes in the same namespaces: compare `sudo ls -l /proc/1/ns` with `ls -l /proc/self/ns`.
   Which numbers differ inside a Docker container (`docker run --rm ubuntu:26.04 ls -l /proc/self/ns`,
   if Docker runs in your VM)?
2. Start `sudo unshare --uts --pid --fork --mount-proc bash`. Inside, set the host name, run `ps aux` and
   `echo $$`. Leave with `exit`. You have just built the core of a container by hand.
3. In a second terminal, while the shell of step 2 is running, find it with `ps -ef | grep unshare` and
   compare `sudo readlink /proc/<pid>/ns/pid` for `unshare` and for its child `bash`.

## Exercises

### Exercise 1 — Same namespace? (easy)

Write `src/bin/samens.rs` that takes two PIDs and a namespace kind (`uts`, `pid`, `mnt`…) and prints
`same` or `different`, by comparing the targets of `/proc/<pid>/ns/<kind>`. Try it with `1` and `$$`.

<details>
<summary>Hint</summary>

`std::fs::read_link(format!("/proc/{pid}/ns/{kind}"))` returns a path like `uts:[4026533109]`. Reading
another user's process needs `sudo`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::env;
use std::fs;

fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().collect();
    if args.len() != 4 {
        eprintln!("usage: samens <pid1> <pid2> <kind>");
        std::process::exit(2);
    }
    let kind = &args[3];
    let a = fs::read_link(format!("/proc/{}/ns/{kind}", args[1]))?;
    let b = fs::read_link(format!("/proc/{}/ns/{kind}", args[2]))?;
    let verdict = if a == b { "same" } else { "different" };
    println!("{verdict}: {} vs {}", a.display(), b.display());
    Ok(())
}
```

`sudo ./target/debug/samens 1 $$ uts` on the test machine:

```text
same: uts:[4026533109] vs uts:[4026533109]
```

Why it works: the link target is the identity of the namespace. Equal targets mean the same namespace;
this is also how tools such as `lsns` group processes.

</details>

### Exercise 2 — A tested flag parser (medium)

Write `pub fn parse_namespaces(list: &str) -> Result<CloneFlags, String>` in `src/lib.rs` that turns text
like `"uts,pid,mnt"` into the matching `CloneFlags`. Unknown names give `Err("unknown namespace: xyz")`.
Write tests for a combination, for spaces around names, and for an unknown name. (You will reuse this idea in
the project.)

<details>
<summary>Hint</summary>

Split with `split(',')`, `trim()` each part, `match` the name and combine flags with `|=`. Start from
`CloneFlags::empty()`.

</details>

<details>
<summary>Answer</summary>

```rust
use nix::sched::CloneFlags;

/// Turns "uts,pid,mnt" into the matching clone flags.
pub fn parse_namespaces(list: &str) -> Result<CloneFlags, String> {
    let mut flags = CloneFlags::empty();
    for name in list.split(',').map(str::trim).filter(|s| !s.is_empty()) {
        flags |= match name {
            "mnt" => CloneFlags::CLONE_NEWNS,
            "uts" => CloneFlags::CLONE_NEWUTS,
            "ipc" => CloneFlags::CLONE_NEWIPC,
            "pid" => CloneFlags::CLONE_NEWPID,
            "net" => CloneFlags::CLONE_NEWNET,
            "user" => CloneFlags::CLONE_NEWUSER,
            "cgroup" => CloneFlags::CLONE_NEWCGROUP,
            other => return Err(format!("unknown namespace: {other}")),
        };
    }
    Ok(flags)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn combines_several_names() {
        let flags = parse_namespaces("uts,pid,mnt").unwrap();
        assert_eq!(
            flags,
            CloneFlags::CLONE_NEWUTS | CloneFlags::CLONE_NEWPID | CloneFlags::CLONE_NEWNS
        );
    }

    #[test]
    fn ignores_spaces_and_empty_parts() {
        assert_eq!(
            parse_namespaces(" uts , ,net").unwrap(),
            CloneFlags::CLONE_NEWUTS | CloneFlags::CLONE_NEWNET
        );
    }

    #[test]
    fn rejects_unknown_names() {
        assert_eq!(
            parse_namespaces("uts,xyz"),
            Err("unknown namespace: xyz".to_string())
        );
    }
}
```

`cargo test`:

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `CloneFlags` is a set of bits, so `|=` adds a kind to the set. The parsing needs no root, so
it can be tested anywhere; only the code that calls `unshare` needs a privileged environment.

</details>

### Exercise 3 — A tiny "container" (hard)

Combine Sections 3–5 into `src/bin/box.rs`: `unshare` UTS, PID and mount namespaces; then `fork`. In the
child: make `/` private, mount a new `proc` on `/proc`, set the host name to `box`, and run `/bin/sh` with
`std::process::Command`, waiting for it. The parent waits for the child. Inside the shell, `hostname` must
print `box`, and `ps` must show only a few processes, with PID 1 being your program.

<details>
<summary>Hint</summary>

Mount `proc` with `mount(Some("proc"), "/proc", Some("proc"), MsFlags::empty(), None::<&str>)`. It must
happen in the child, because only the child is inside the new PID namespace. The child is PID 1 and the shell
it starts is PID 2.

</details>

<details>
<summary>Answer</summary>

```rust
use std::process::Command;

use nix::mount::{MsFlags, mount};
use nix::sched::{CloneFlags, unshare};
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, fork, sethostname};

fn child() -> nix::Result<()> {
    mount(
        None::<&str>,
        "/",
        None::<&str>,
        MsFlags::MS_REC | MsFlags::MS_PRIVATE,
        None::<&str>,
    )?;
    mount(
        Some("proc"),
        "/proc",
        Some("proc"),
        MsFlags::empty(),
        None::<&str>,
    )?;
    sethostname("box")?;
    let status = Command::new("/bin/sh")
        .status()
        .expect("cannot start /bin/sh");
    println!("shell exited with {status}");
    Ok(())
}

fn main() -> nix::Result<()> {
    unshare(CloneFlags::CLONE_NEWUTS | CloneFlags::CLONE_NEWPID | CloneFlags::CLONE_NEWNS)?;
    // SAFETY: single-threaded program.
    match unsafe { fork() }? {
        ForkResult::Child => {
            child()?;
            std::process::exit(0);
        }
        ForkResult::Parent { child } => {
            waitpid(child, None)?;
        }
    }
    Ok(())
}
```

A session on the test machine (typed commands after `#`):

```text
# hostname
box
# ps -o pid,comm
    PID COMMAND
      1 box
      2 sh
      4 ps
# exit
/bin/sh: 3: Cannot set tty process group (No such process)
shell exited with exit status: 2
```

PID 3 was the `hostname` command, which had already finished. The message at the end comes from `sh`: when
it exits, it tries to give the terminal back to the process group that was in front before. That group
lives outside the new PID namespace, so inside it has no number ("No such process"), and `sh` reports
exit status 2. Real runtimes avoid this by giving the container its own session and terminal; it does not
affect what you learn here.

Why it works: `unshare` prepared three new namespaces; `fork` put the child into the new PID namespace as
PID 1. The fresh `proc` mount shows that namespace's processes. The mount namespace keeps that `/proc` mount
away from the VM. What is still missing for a real container: its own root file system and resource limits.
That is the next two lessons.

</details>

## Quiz

1. What does a namespace do?

<details>
<summary>Answer</summary>

It gives a group of processes their own private view of one global kernel resource, such as the host name
(UTS), process numbers (PID), the mount table (mount) or the network (net). Inside it looks normal; outside
nothing changes.

</details>

2. How can you tell whether two processes share a namespace?

<details>
<summary>Answer</summary>

Compare the targets of `/proc/<pid>/ns/<kind>`, for example `uts:[4026533109]`. Equal numbers mean the same
namespace.

</details>

3. After `unshare(CLONE_NEWPID)`, what is the PID of the calling process, and what is the PID of its next
   child?

<details>
<summary>Answer</summary>

The caller keeps its old PID; it stays in the old PID namespace. Its next child is the first process of the
new namespace and has PID 1 inside (and a normal PID outside).

</details>

4. Why does `docker stop` sometimes take 10 seconds?

<details>
<summary>Answer</summary>

`docker stop` sends SIGTERM to the container's PID 1. The kernel does not deliver signals to PID 1 of a
namespace unless it has installed a handler. A program that does not handle SIGTERM ignores it, and Docker
uses SIGKILL after a 10-second timeout.

</details>

5. Why do you make `/` private right after creating a mount namespace?

<details>
<summary>Answer</summary>

With shared propagation, mounts made inside the new namespace could appear in the parent namespace too.
`MS_REC | MS_PRIVATE` cuts that connection, so the container's mounts stay inside.

</details>

## Summary

- A namespace gives processes a private view of one global resource; Linux has mount, UTS, IPC, PID, network, user, cgroup and time namespaces.
- `/proc/<pid>/ns/*` shows the namespaces of a process; equal numbers mean the same namespace.
- `unshare` (command and system call) moves a process into new namespaces; creating them needs root, except for user namespaces.
- A new PID namespace applies to children: the first child is PID 1, adopts orphans, ignores signals it has no handler for, and takes all others with it when it exits.
- A new mount namespace should be made private first; then mounts inside stay inside.
- Namespaces are the walls of a container, but they share one kernel; real runtimes add capabilities, seccomp and cgroups.

## Sources and further reading

- [namespaces(7) — Linux manual page](https://man7.org/linux/man-pages/man7/namespaces.7.html) — all kinds of namespaces and the `/proc/<pid>/ns` files.
- [pid_namespaces(7) — Linux manual page](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html) — PID 1, signals and nesting.
- [mount_namespaces(7) — Linux manual page](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html) — mount propagation (`shared`, `private`).
- [unshare(1) — Linux manual page](https://man7.org/linux/man-pages/man1/unshare.1.html) — the command used in this lesson.
- [nix::sched on docs.rs](https://docs.rs/nix/0.32.0/nix/sched/index.html) — `unshare`, `setns` and `CloneFlags` in Rust.
- [Namespaces in operation, part 1](https://lwn.net/Articles/531114/) — the start of a well-known article series on LWN.net (originally "Linux Weekly News", a long-running site about the Linux kernel).

---

[🏠 Start](../START-HERE.md) · [← 13.14 io_uring](14-io-uring.md) · [📋 Phase 13](README.md) · [13.16 Control groups (cgroups) →](16-cgroups.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
