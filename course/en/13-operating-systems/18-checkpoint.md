<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.17 Project: a mini container runtime](17-project-mini-container-runtime.md) · [📋 Phase 13](README.md) · [Phase 14 — Concurrency and async in depth →](../14-concurrency-and-async/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/18-checkpoint.md) <!-- nav -->

# Lesson 13.18 — Checkpoint

> **Phase 13 — Operating systems from the inside** · Lesson 18 of 18 · ⏱ about 135 minutes

## What you will learn

This checkpoint closes Phase 13 and teaches nothing new. It checks that you can use what the phase taught, and it shows you
where to go back if something is still weak. You will:

- Answer 20 questions about the kernel, processes, system calls, file descriptors, `fork` and `exec`, pipes, signals,
  threads, virtual memory, memory mapping, file systems, `epoll`, `io_uring`, namespaces, cgroups and your container runtime.
- Solve 5 practical tasks; each one has a test or an output that tells you whether you are right.
- Find the lessons to read again in the table at the end.

## Before you start

- You finished [Lesson 13.17 — Project: a mini container runtime](17-project-mini-container-runtime.md), or at least its
  milestones M1 to M4. Task 5 uses your `minibox`.
- Answer the questions without looking back first, then open the answers.
- Create a package for the practical tasks on the Mac. Tasks 1 and 4 go into `src/lib.rs`, tasks 2 and 3 into `src/bin/`.
  Copy it into the VM (virtual machine) `dev` with `rsync` and run everything there, as in Lesson 13.1:

```bash
cd ~/projects/rust
cargo new --lib code/p13-checkpoint
mkdir code/p13-checkpoint/src/bin
cd code/p13-checkpoint
cargo add signal-hook@0.4
```

The outputs below come from a Linux container set up like the VM; PIDs (process IDs), sizes and namespace numbers on your
VM will differ.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A pause at the end of a phase to test yourself before you go on. |
| **lock file** | A file whose existence means "one copy of this program is already running". Creating it must be one atomic step, and the program must remove it when it ends. |
| **`nsenter`** | A command (from util-linux) that runs a program inside the namespaces of another process, with the `setns` system call. `docker exec` works the same way. |

## Self-test quiz

1. Name the three ways the CPU (Central Processing Unit) enters the kernel. Which one lets the kernel take the CPU back from
   a program that never makes a system call?

<details>
<summary>Answer</summary>

System calls (the program asks), exceptions (the instruction itself fails, for example a page fault or a division by zero)
and interrupts (a device or the timer signals the CPU). The **timer interrupt** arrives regularly whatever the program does,
so the kernel gets control back and can schedule another task. (Lesson 13.1)

</details>

2. A program reserves 1 GiB (gibibyte) with `vec![0u8; 1 << 30]` and then sleeps. Its RSS (Resident Set Size) stays small.
   Why?

<details>
<summary>Answer</summary>

Because of demand paging: the kernel gives a page of RAM (Random Access Memory) only when it is touched for the first time.
Zeroed memory from the allocator comes from fresh anonymous mappings that are already zero, so nothing has been written and
almost no pages are in RAM. RSS counts only pages that really are in RAM. (Lessons 13.2 and 13.10)

</details>

3. `open` fails because the file does not exist. How does that error reach you through `libc`, through `nix` and through
   `std`?

<details>
<summary>Answer</summary>

The kernel returns a negative number (−2). `libc::open` returns −1 and stores 2 (`ENOENT`) in `errno`. `nix` returns
`Err(Errno::ENOENT)`. `std::fs::File::open` returns an `io::Error` whose `kind()` is `NotFound` and whose
`raw_os_error()` is `Some(2)`. (Lesson 13.3)

</details>

4. Why is writing a million short lines through a `BufWriter` much faster than a million `println!` calls?

<details>
<summary>Answer</summary>

Standard output in Rust is flushed at every newline, so each `println!` becomes one `write` system call, and every system
call costs a trip into the kernel. `BufWriter` collects the lines in a buffer (8 KiB, kibibytes, by default) and writes it in large
pieces: a few hundred system calls instead of a million. `strace -c` shows the difference. (Lesson 13.3)

</details>

5. A process opens one file twice with `open` (descriptors 3 and 4) and then calls `dup(3)`, which returns 5. Which of the
   three descriptors share a file offset?

<details>
<summary>Answer</summary>

3 and 5. `dup` makes a new descriptor that points to the **same** open file description, which holds the offset. Each `open`
creates its own open file description, so 4 has its own offset. (Lesson 13.4)

</details>

6. What does `FD_CLOEXEC` do, and what goes wrong in a program that starts children if it is missing?

<details>
<summary>Answer</summary>

The descriptor is closed automatically when the process calls `exec`. Without it, every child program inherits the
descriptor: files stay open longer than intended, and a pipe's write end held by a child means the reader never sees end of
file. `std` sets `FD_CLOEXEC` on everything it opens. (Lessons 13.4 and 13.6)

</details>

7. Why is `nix::unistd::fork` an `unsafe` function?

<details>
<summary>Answer</summary>

`fork` copies only the calling thread. If another thread held a lock at that moment (for example the memory allocator's
lock), the lock stays locked forever in the child. So in a program with threads, the child may only call
async-signal-safe functions until it calls `exec`. The compiler cannot check that, so you must promise it. (Lesson 13.5)

</details>

8. What is a zombie, what is an orphan, and who cleans up after each?

<details>
<summary>Answer</summary>

A zombie is a child that has ended but whose exit status its parent has not collected yet; the parent must call `waitpid`.
An orphan is a process whose parent ended first; it is adopted by PID 1 (Process ID 1), which waits for it when it ends.
(Lesson 13.5)

</details>

9. A program starts `wc -l` with a pipe on its standard input, writes all its lines into the pipe, then waits for `wc`. It
   hangs forever. What is the most likely cause?

<details>
<summary>Answer</summary>

A write end of the pipe is still open somewhere: in the program itself (it never closed or dropped its copy) or in another
child that inherited it. `wc` sees end of file only when **all** write ends are closed. Drop the write end before waiting,
and create pipes with `O_CLOEXEC`. (Lesson 13.6)

</details>

10. What status does a shell show for a program killed by `SIGPIPE`? And why does a Rust program that writes into a closed
    pipe usually get an error instead of dying?

<details>
<summary>Answer</summary>

141 = 128 + 13 (`SIGPIPE` is signal 13). The Rust runtime sets `SIGPIPE` to "ignore" before `main`, so `write` returns
`EPIPE`, which `std` shows as `ErrorKind::BrokenPipe`. (Lessons 13.6 and 13.7)

</details>

11. What may a signal handler safely do, and why so little?

<details>
<summary>Answer</summary>

Only async-signal-safe work; in practice: set an atomic flag (or write one byte into a pipe) and return. The handler can
interrupt the program at any instruction, for example in the middle of `malloc` while it holds a lock; calling `println!` or
allocating there can deadlock or corrupt data. (Lesson 13.7)

</details>

12. A blocking `read` returns the error `EINTR`. What happened, and what should your code do?

<details>
<summary>Answer</summary>

A signal arrived while the call was waiting, and its handler ran; the kernel interrupted the call. Retry it
(`ErrorKind::Interrupted` in `std`; `read_exact` and `write_all` already retry), or install the handler with `SA_RESTART`.
(Lesson 13.7)

</details>

13. You press Ctrl-C while your shell `rsh` runs `sleep 30`. Which processes receive `SIGINT`, and why does only `sleep`
    die?

<details>
<summary>Answer</summary>

The terminal driver sends `SIGINT` to every process in the foreground process group: the shell and `sleep`. The shell
ignores `SIGINT`; the child reset it to the default action before `exec`, so it dies (status 130). (Lessons 13.7 and 13.8)

</details>

14. Linux creates processes and threads with the same system call. Which one, and what do threads of one process share and
    not share?

<details>
<summary>Answer</summary>

`clone`; its flags decide what is shared. Threads share the memory, the open files and the signal handlers. Each thread
has its own registers, stack, TID (Thread IDentifier) and signal mask. (Lesson 13.9)

</details>

15. After `fork`, parent and child both "have" the same 100 MiB (mebibytes) of heap. How much new RAM did `fork` use, and when does
    that change?

<details>
<summary>Answer</summary>

Almost none. Both page tables point to the same frames, marked read-only (copy-on-write). The first write by either process
to a page causes a page fault, and the kernel copies just that page. (Lesson 13.10)

</details>

16. Your program maps a file with `memmap2::Mmap`, and another program truncates the file to zero bytes. What can happen,
    and how does Rust make you acknowledge the danger?

<details>
<summary>Answer</summary>

Touching a mapped page beyond the new end of the file raises `SIGBUS`, which kills the program. Also, the "immutable"
`&[u8]` can change under your feet. That is why `Mmap::map` is `unsafe`: you promise that nobody changes the file while it is
mapped. (Lesson 13.11)

</details>

17. You delete a 2 GiB log file with `rm`, but `df` shows no space gained. Why, and how do you find the culprit?

<details>
<summary>Answer</summary>

`rm` only removes the name (`unlink`). The data is freed when the last name **and** the last open descriptor are gone, and a
running process still has the file open. `ls -l /proc/*/fd 2>/dev/null | grep deleted` (with `sudo` for other users'
processes) shows which one; restart it or make it reopen its log. (Lesson 13.12)

</details>

18. What must you do differently when you register a descriptor with `epoll` as edge-triggered instead of level-triggered?

<details>
<summary>Answer</summary>

Edge-triggered reports a change only once. After a notification you must read (or accept, or write) until the call returns
`WouldBlock`; if you leave data behind, no new event comes and the connection stalls. Level-triggered keeps reporting while
data is there. (Lesson 13.13)

</details>

19. `epoll` and `io_uring` both serve many descriptors. What is the main difference, and why must a buffer given to
    `io_uring` live until its completion arrives?

<details>
<summary>Answer</summary>

`epoll` reports **readiness** ("you can read now"), and your program then makes the call itself. `io_uring` reports
**completion**: the kernel does the read or write and puts the result into the completion queue. The kernel writes into your
buffer later, at a moment you do not control; freeing it earlier would let the kernel write into memory that now belongs to
something else. (Lesson 13.14)

</details>

20. In `minibox`, why does the parent unshare only `CLONE_NEWPID`, and what do the steps before `pivot_root` do? And why
    can namespaces alone not stop a container from eating all memory?

<details>
<summary>Answer</summary>

`CLONE_NEWPID` affects only the children created later; UTS (UNIX Time-Sharing) and mount namespaces move the caller at once, and a parent in
the child's mount namespace would lose its root at `pivot_root`. Before `pivot_root`: make all mounts private (so nothing
leaks to the VM and `pivot_root` accepts them), bind-mount the rootfs onto itself (so it is a mount point), and `chdir` into
it for `pivot_root(".", ".")`. Namespaces only change what a process **sees**; how much it may **use** is the job of
cgroups (`memory.max`, `cpu.max`, `pids.max`). (Lessons 13.15, 13.16 and 13.17)

</details>

## Practical tasks

### Task 1 — Read a process from `/proc` (easy)

In `src/lib.rs`, write `pub fn status_field<'a>(status: &'a str, key: &str) -> Option<&'a str>` that returns the value of
one field of the text of `/proc/<pid>/status`, without spaces around it: for the line `VmRSS:\t    3840 kB` and the key
`VmRSS` it returns `"3840 kB"`. The key must match exactly (`Vm` finds nothing). Add a test. Then write `src/bin/pinfo.rs`:
`pinfo [pid]` prints `Name`, `State`, `PPid`, `Threads` and `VmRSS` of that process (default: itself).

<details>
<summary>Hint</summary>

`line.split_once(':')` gives the name and the rest; compare the name with `==`. `bool::then` turns a condition into an
`Option`. In `pinfo`, `/proc/self/status` describes the reading process itself.

</details>

<details>
<summary>Answer</summary>

```rust
/// The value of one field of /proc/<pid>/status, for example "State" → "S (sleeping)".
pub fn status_field<'a>(status: &'a str, key: &str) -> Option<&'a str> {
    status.lines().find_map(|line| {
        let (name, value) = line.split_once(':')?;
        (name == key).then(|| value.trim())
    })
}

#[cfg(test)]
mod tests {
    use super::*;

    const STATUS: &str = "Name:\tbash\nState:\tS (sleeping)\nThreads:\t1\nVmRSS:\t    3840 kB\n";

    #[test]
    fn finds_fields_by_their_exact_name() {
        assert_eq!(status_field(STATUS, "State"), Some("S (sleeping)"));
        assert_eq!(status_field(STATUS, "VmRSS"), Some("3840 kB"));
        assert_eq!(status_field(STATUS, "Vm"), None);
    }
}
```

```rust
use std::{env, fs};

use p13_checkpoint::status_field;

fn main() -> std::io::Result<()> {
    let pid = env::args().nth(1).unwrap_or_else(|| "self".to_string());
    let status = fs::read_to_string(format!("/proc/{pid}/status"))?;
    for key in ["Name", "State", "PPid", "Threads", "VmRSS"] {
        println!("{key:<8} {}", status_field(&status, key).unwrap_or("-"));
    }
    Ok(())
}
```

`./target/debug/pinfo` and `./target/debug/pinfo 1` on the test machine:

```text
Name     pinfo
State    R (running)
PPid     12
Threads  1
VmRSS    1680 kB
Name     bash
State    S (sleeping)
PPid     0
Threads  1
VmRSS    2976 kB
```

Why it works: `/proc` files are text generated by the kernel on every read, so reading a process is ordinary string parsing,
and the parsing is a pure function you can test without any process. `pinfo` itself is `R` (running) while it reads its own
status. (Lessons 13.1 and 13.2)

</details>

### Task 2 — A pipeline without a shell (medium)

Write `src/bin/pipe2.rs` that runs `ls /etc | wc -l` with `std::process::Command`, without starting a shell. It prints the
number of entries and the exit status of both programs, and leaves no zombie. Compare its number with the shell's
`ls /etc | wc -l`.

<details>
<summary>Hint</summary>

Start `ls` with `stdout(Stdio::piped())`, **take** its `stdout` out of the `Child` and give it to `wc` with
`Stdio::from`. Then wait for **both**.

</details>

<details>
<summary>Answer</summary>

```rust
use std::process::{Command, Stdio};

fn main() -> std::io::Result<()> {
    let mut ls = Command::new("ls")
        .arg("/etc")
        .stdout(Stdio::piped())
        .spawn()?;
    // Take the read end out of `ls`, so that only `wc` holds it.
    let pipe = ls.stdout.take().expect("stdout was piped");
    let wc = Command::new("wc")
        .arg("-l")
        .stdin(Stdio::from(pipe))
        .output()?; // waits for wc and collects its output
    let ls_status = ls.wait()?; // reap ls too: no zombie
    print!("entries in /etc: {}", String::from_utf8_lossy(&wc.stdout));
    println!("ls: {ls_status}; wc: {}", wc.status);
    Ok(())
}
```

```bash
./target/debug/pipe2
ls /etc | wc -l
```

```text
entries in /etc: 101
ls: exit status: 0; wc: exit status: 0
101
```

Why it works: `std` creates the pipe with `O_CLOEXEC` and moves the read end into `wc`'s standard input. After `take`, the
parent's `Child` no longer holds that end, so when `ls` finishes and closes its write end, `wc` sees end of file. `output()`
waits for `wc`, and `ls.wait()` collects the status of `ls`, so neither stays a zombie. (Lessons 13.5 and 13.6)

</details>

### Task 3 — A lock file that is always removed (medium)

Write `src/bin/lockwork.rs`. At start it creates `/tmp/course-lockwork.lock`; if the file already exists, it prints
`lockwork: already running (/tmp/course-lockwork.lock exists)` and exits with 1. Then it works (sleeps in steps of 100 ms)
until `SIGTERM` or `SIGINT` arrives, removes the lock file, prints how many rounds it did and exits with 0. Check it with:

```bash
./target/debug/lockwork & sleep 0.5
./target/debug/lockwork; echo "second: exit=$?"
kill -TERM $!; wait $!; echo "first: exit=$?"
ls /tmp/course-lockwork.lock
```

Then try `kill -KILL` instead of `kill -TERM`. What is different, and why?

<details>
<summary>Hint</summary>

`OpenOptions::new().write(true).create_new(true)` checks and creates in one step and fails with `ErrorKind::AlreadyExists`.
Put the removal into `Drop` of a small `Lock` type, and stop the loop with `signal_hook::flag::register` (Lesson 13.7).

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs::{self, OpenOptions};
use std::io::ErrorKind;
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::thread;
use std::time::Duration;

use signal_hook::consts::{SIGINT, SIGTERM};

const LOCK: &str = "/tmp/course-lockwork.lock";

/// Owning a `Lock` means the lock file exists; dropping it removes the file.
struct Lock;

impl Lock {
    fn take() -> std::io::Result<Lock> {
        // create_new fails with AlreadyExists if the file is there: one atomic check-and-create.
        OpenOptions::new().write(true).create_new(true).open(LOCK)?;
        Ok(Lock)
    }
}

impl Drop for Lock {
    fn drop(&mut self) {
        let _ = fs::remove_file(LOCK);
    }
}

fn main() -> std::io::Result<()> {
    let lock = match Lock::take() {
        Ok(lock) => lock,
        Err(e) if e.kind() == ErrorKind::AlreadyExists => {
            eprintln!("lockwork: already running ({LOCK} exists)");
            std::process::exit(1);
        }
        Err(e) => return Err(e),
    };
    let stop = Arc::new(AtomicBool::new(false));
    for signal in [SIGTERM, SIGINT] {
        signal_hook::flag::register(signal, Arc::clone(&stop))?;
    }
    let mut rounds = 0;
    while !stop.load(Ordering::Relaxed) {
        thread::sleep(Duration::from_millis(100));
        rounds += 1;
    }
    drop(lock);
    println!("lockwork: stopped after {rounds} rounds, lock removed");
    Ok(())
}
```

```text
lockwork: already running (/tmp/course-lockwork.lock exists)
second: exit=1
lockwork: stopped after 5 rounds, lock removed
first: exit=0
ls: cannot access '/tmp/course-lockwork.lock': No such file or directory
```

With `kill -KILL`, the shell reports `Killed` and exit status 137, the file stays behind, and the next start says "already
running": `SIGKILL` cannot be caught, so no Rust code runs, not even `Drop`. That is why real programs also write their PID
into the lock file and check whether that process still lives. Remove the leftover with `rm /tmp/course-lockwork.lock`.
(Lessons 13.4, 13.7 and 13.17)

</details>

### Task 4 — Replace a file safely (medium)

In `src/lib.rs`, write `pub fn write_atomically(path: &Path, data: &[u8]) -> io::Result<()>`: after a crash at any moment,
the file holds either its old content or the new one, never a mix or nothing. Write a test that writes twice into a file in
a fresh folder under `std::env::temp_dir()`, checks the content and checks that no temporary file is left.

<details>
<summary>Hint</summary>

The four steps of Lesson 13.12: write a temporary file next to the target, `sync_all` it, `rename` it over the target, then
open the **directory** and `sync_all` it too. A relative path such as `config.txt` has the parent `""`, which you must
treat as `"."`.

</details>

<details>
<summary>Answer</summary>

Add to `src/lib.rs` (the test goes into the same `mod tests`):

```rust
use std::fs::{self, File};
use std::io::{self, Write};
use std::path::Path;

/// Replaces the file at `path` with `data`; after a crash it holds either the old or the new data.
pub fn write_atomically(path: &Path, data: &[u8]) -> io::Result<()> {
    let dir = match path.parent() {
        Some(p) if !p.as_os_str().is_empty() => p,
        _ => Path::new("."),
    };
    let mut tmp = path.as_os_str().to_owned();
    tmp.push(".tmp");

    let mut file = File::create(&tmp)?;
    file.write_all(data)?;
    file.sync_all()?; // 1. the new data is on disk
    fs::rename(&tmp, path)?; // 2. one atomic step: the name points to the new file
    File::open(dir)?.sync_all() // 3. the rename itself is on disk
}
```

```rust
    #[test]
    fn replaces_the_whole_file_and_leaves_no_temp_file() {
        let dir = std::env::temp_dir().join(format!("course-ckpt-{}", std::process::id()));
        fs::create_dir_all(&dir).unwrap();
        let path = dir.join("config.txt");
        write_atomically(&path, b"old and long").unwrap();
        write_atomically(&path, b"new").unwrap();
        assert_eq!(fs::read_to_string(&path).unwrap(), "new");
        assert_eq!(fs::read_dir(&dir).unwrap().count(), 1);
        fs::remove_dir_all(&dir).unwrap();
    }
```

`cargo test -q` (with the test from Task 1):

```text
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `rename` within one file system replaces the directory entry in one atomic step, so a reader sees either the
old inode or the new one. `sync_all` on the file makes the new data durable **before** the name points to it; `sync_all` on
the directory makes the new name itself durable. The test uses its own folder (with the PID in its name), so parallel tests
and earlier runs cannot disturb it. (Lesson 13.12)

</details>

### Task 5 — Prove the walls of your container (hard)

Start `minibox` in the background with `sleep 60` inside. From the VM, find the PID of that `sleep` and show, with shell
commands only: which of its namespaces `pid`, `uts`, `mnt` and `net` differ from your shell's; its PID inside and outside;
its cgroup; that its root is Alpine; and its host name, read from inside its UTS namespace. Then kill it from outside and
check that `minibox` cleaned up.

<details>
<summary>Hint</summary>

`pgrep -n -x sleep` finds the newest process called exactly `sleep`. `/proc/<pid>/ns/*`, `/proc/<pid>/cgroup`, the
`NSpid` line of `/proc/<pid>/status` and the link `/proc/<pid>/root` tell you everything; reading another root process's
links needs `sudo`. `sudo nsenter --target <pid> --uts hostname` runs `hostname` in that process's UTS namespace.

</details>

<details>
<summary>Answer</summary>

In `~/p13-minibox` in the VM:

```bash
sudo ./target/debug/minibox run --rootfs ~/minibox/alpine --hostname walls -- sleep 60 &
sleep 1
P=$(pgrep -n -x sleep)
for ns in pid uts mnt net; do echo "$ns: $(sudo readlink /proc/$P/ns/$ns) vs $(readlink /proc/self/ns/$ns)"; done
grep NSpid /proc/$P/status
cat /proc/$P/cgroup
sudo cat /proc/$P/root/etc/alpine-release
sudo nsenter --target $P --uts hostname
sudo kill -KILL $P; wait $!; echo "minibox exit=$?"
ls /sys/fs/cgroup | grep minibox || echo "no minibox groups left"
```

```text
pid: pid:[4026533108] vs pid:[4026532975]
uts: uts:[4026533111] vs uts:[4026532973]
mnt: mnt:[4026533109] vs mnt:[4026532972]
net: net:[4026532977] vs net:[4026532977]
NSpid:	536	1
0::/minibox-535
3.24.4
walls
minibox exit=137
no minibox groups left
```

Why it works: the container's process is an ordinary process of the VM, so the VM's `/proc` shows everything about it. Its
PID, UTS and mount namespaces are new, the network namespace is shared (`minibox` does not create one). `NSpid` shows the
same process as 536 outside and 1 inside. Its cgroup is the group named after `minibox`'s own PID, 535. `/proc/<pid>/root`
leads to its root, Alpine. `nsenter` joins its UTS namespace, as `docker exec` would. `SIGKILL` from the outer namespace
reaches even PID 1, `minibox` reports 137 and its `Drop` removes the group. (Lessons 13.15, 13.16 and 13.17)

</details>

## If you got something wrong

| Topic | Lesson to review |
|---|---|
| Kernel and user space, system calls, exceptions and interrupts, `/proc` | [13.1 — What an operating system does](01-what-an-operating-system-does.md) |
| Process descriptor, address space, RSS, states, the process tree | [13.2 — Processes from the inside](02-processes-from-the-inside.md) |
| System calls at four levels, `errno`, `strace`, buffering | [13.3 — System calls](03-system-calls.md) |
| Descriptor tables, `dup`, `OwnedFd`, close-on-exec | [13.4 — Files and file descriptors](04-files-and-file-descriptors.md) |
| `fork`, `exec`, `waitpid`, zombies and orphans, exit codes | [13.5 — Creating processes: fork, exec and wait](05-fork-exec-and-wait.md) |
| Pipes, end of file, `dup2`, redirection, `SIGPIPE` | [13.6 — Pipes and redirection from the inside](06-pipes-and-redirection-inside.md) |
| Signal handlers, `signal-hook`, `EINTR`, process groups and Ctrl-C | [13.7 — Signals in your programs](07-signals-in-your-programs.md) |
| Putting processes, pipes and signals together; `assert_cmd` tests | [13.8 — Project: your own shell](08-project-your-own-shell.md) |
| Threads, `clone`, the scheduler, nice values, CPU affinity | [13.9 — Threads and scheduling](09-threads-and-scheduling.md) |
| Pages, page tables, demand paging, copy-on-write, the OOM (Out Of Memory) killer | [13.10 — Virtual memory](10-virtual-memory.md) |
| `mmap`, shared and private mappings, `SIGBUS` | [13.11 — Memory-mapped files](11-memory-mapped-files.md) |
| Inodes, links, `unlink`, durability and atomic replace | [13.12 — File systems from the inside](12-file-systems-inside.md) |
| Non-blocking mode, `epoll`, level- and edge-triggered | [13.13 — Non-blocking I/O and epoll](13-non-blocking-io-and-epoll.md) |
| Submission and completion rings, buffers in flight | [13.14 — io_uring](14-io-uring.md) |
| Namespaces, PID 1 inside a namespace, mount propagation | [13.15 — Namespaces](15-namespaces.md) |
| cgroup v2, `memory.max`, `cpu.max`, `pids.max`, `cgroup.procs` | [13.16 — Control groups (cgroups)](16-cgroups.md) |
| `pivot_root`, the order of the set-up steps, clean-up with `Drop`, root-only tests | [13.17 — Project: a mini container runtime](17-project-mini-container-runtime.md) |

## What's next

You now know what is under every program you run: the kernel's process table, the system calls, the descriptor table, the
page tables, the scheduler, and the namespaces and cgroups that turn a process into a container. You built a shell and a
container runtime on top of them.

[Phase 14 — Concurrency and async in depth](../14-concurrency-and-async/README.md) builds directly on three parts of this
phase. Threads and scheduling (Lesson 13.9) become races, deadlocks, mutexes, condition variables, atomics and memory
ordering, up to your own spin lock and lock-free structures tested with `loom`. `epoll` and `io_uring` (Lessons 13.13 and
13.14) become futures, wakers and a tiny async runtime of your own, then Tokio. And signals (Lesson 13.7) return as
cancellation and graceful shutdown. Its project is a concurrent job queue.

Before you continue, commit your work in `code/p13-checkpoint` and `code/p13-minibox`, check for leftover course containers
on your Mac with `docker ps -a --filter name=course-` (remove them with `docker rm`), and add a line to your learning journal: which kernel behaviour in this
phase surprised you most?

## Sources and further reading

- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — a free textbook on processes, virtual memory, concurrency and file systems, the theory behind the whole phase.
- [The Linux man-pages project](https://man7.org/linux/man-pages/) — the reference for every system call and `/proc` file used in this phase.
- [The Linux Programming Interface](https://man7.org/tlpi/) — the classic book on Linux system programming, by the maintainer of the man pages.
- [The Linux kernel documentation](https://docs.kernel.org/) — the official documentation, including the cgroup v2 and `/proc` references.
- [nix on docs.rs](https://docs.rs/nix/0.32.0/nix/) — the Rust bindings you used in almost every lesson.

---

[🏠 Start](../START-HERE.md) · [← 13.17 Project: a mini container runtime](17-project-mini-container-runtime.md) · [📋 Phase 13](README.md) · [Phase 14 — Concurrency and async in depth →](../14-concurrency-and-async/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
