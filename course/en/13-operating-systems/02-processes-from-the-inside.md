<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.1 What an operating system does](01-what-an-operating-system-does.md) · [📋 Phase 13](README.md) · [13.3 System calls →](03-system-calls.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/02-processes-from-the-inside.md) <!-- nav -->

# Lesson 13.2 — Processes from the inside

> **Phase 13 — Operating systems from the inside** · Lesson 2 of 18 · ⏱ about 135 minutes

## What you will learn

- The difference between a program and a process, and what the kernel remembers about every process.
- How the memory of a process is laid out (code, data, heap, stack), and how to see it in `/proc/self/maps`.
- The process states R, S, D, T and Z, and how to watch a process change state.
- How processes form a tree with PID 1 at the root.
- How to read facts about any process from `/proc/<pid>/` in Rust, using `std` and the `nix` crate.

## Before you start

- You finished [Lesson 13.1 — What an operating system does](01-what-an-operating-system-does.md) and have the package `code/p13-os` copied into your VM.
- From Phase 11 you know `ps`, `kill`, background jobs with `&`, and signals such as `SIGSTOP` and `SIGCONT`.
- From [Lesson 12.12 — Linking and loading programs](../12-computer-systems/12-linking-and-loading.md) you know that a program is an ELF file with sections such as `.text` and `.data`, and that the loader maps them into memory.

## New words

| Word | Meaning |
|---|---|
| **process** | A program while it runs: its memory, its open files, its CPU registers and the kernel's notes about it. |
| **process descriptor** | The record the kernel keeps for each process. In Linux it is a C structure called `task_struct`. |
| **PPID** (Parent Process IDentifier) | The PID of the process that created this one. |
| **UID** (User IDentifier) | The number of the user a process runs as; 0 is `root` (Lesson 11.4). |
| **address space** | All the memory addresses a process can use. Each process has its own. |
| **memory region** (mapping) | A continuous range of addresses with the same permissions and the same source, for example "the code of the program" or "the stack". |
| **stack** / **heap** | Memory for local variables of running functions / memory for values whose size or lifetime is decided at run time (`Box`, `Vec`), from Lesson 03.1. |
| **ASLR** (Address Space Layout Randomization) | The kernel places code, heap and stack at random addresses on each start, so attackers cannot guess them. |
| **RSS** (Resident Set Size) | How much of a process's memory is really in RAM right now. `/proc` calls it `VmRSS`. |
| **process state** | What a process is doing: running (R), sleeping (S), waiting for a device (D), stopped (T), or finished but not collected (Z). |
| **zombie** | A process that has finished, but whose parent has not yet read its exit code. Lesson 13.5 explains it. |
| **NUL byte** | The byte with value 0. Linux uses it to separate strings in some files. |
| **CPU**, **RAM**, **ELF**, **KiB**/**MiB**/**GiB**/**TiB**, **UTF-8** | Known from earlier lessons: Central Processing Unit; Random Access Memory; Executable and Linkable Format, the file format of Linux programs (Lesson 12.12); kibibyte, mebibyte, gibibyte, tebibyte (each 1,024 times the one before); Unicode Transformation Format, 8-bit (Lesson 12.5). |

## 1. A program is a file, a process is a running program

A **program** is a file on disk, for example `/usr/bin/sleep`, an ELF file full of instructions (Lesson 12.12).
It does nothing by itself. A **process** is what you get when the kernel *runs* a program: the kernel loads the
file into memory, gives it a stack, a PID and a place in the CPU's schedule, and starts its first instruction.

Think of a recipe and the act of cooking. The recipe is the program. Two cooks using the same recipe in two kitchens
are two processes: same instructions, separate pots, separate progress. In the same way you can run `sleep 100`
three times and get three processes from one file.

For each process the kernel keeps a **process descriptor** (Linux calls it `task_struct`). Its main parts:

```text
 task_struct (one per process, inside kernel memory)
 +---------------------------------------------------------------+
 | PID 4630, parent PID 4629, state S (sleeping)                 |
 | credentials: UID 501, GID 1000                                |
 | memory map: code, data, heap, stack, libraries   -> Section 2 |
 | open files: 0, 1, 2, 3 ...                       -> Lesson 13.4|
 | current directory, root directory                             |
 | signal handlers and blocked signals              -> Lesson 13.7|
 | saved CPU registers (while it is not running)    -> Lesson 13.9|
 | scheduling priority, CPU time used                            |
 +---------------------------------------------------------------+
```

GID (Group IDentifier) is the number of the user's main group (Lesson 11.4). Almost every file in `/proc/<pid>/`
shows one part of this record.

### Asking the kernel who you are

Add the `nix` crate to the package. `nix` gives safe Rust functions for Unix system calls; its parts are
switched on with Cargo features (Lesson 05.8):

```bash
cargo add nix@0.32 --features process,user
```

Now `Cargo.toml` contains `nix = { version = "0.32", features = ["process", "user"] }`. Create
`src/bin/who-am-i.rs`:

```rust
use nix::unistd::{getpid, getppid, getuid};
use std::fs;

fn main() -> std::io::Result<()> {
    println!("PID  (std): {}", std::process::id());
    println!("PID  (nix): {}", getpid());
    println!("PPID:       {}", getppid());
    println!("UID:        {}", getuid());
    println!("directory:  {}", std::env::current_dir()?.display());

    // The kernel's own summary of this process.
    let status = fs::read_to_string("/proc/self/status")?;
    for line in status.lines() {
        let key = line.split(':').next().unwrap_or("");
        if ["Name", "State", "PPid", "Threads", "VmRSS"].contains(&key) {
            println!("  {line}");
        }
    }
    Ok(())
}
```

```text
PID  (std): 154
PID  (nix): 154
PPID:       1
UID:        0
directory:  /work
  Name:	who-am-i
  State:	R (running)
  PPid:	1
  VmRSS:	    1744 kB
  Threads:	1
```

**What happens:**

1. `std::process::id()` and `nix::unistd::getpid()` both ask the kernel the same question. `std` gives a `u32`;
   `nix` gives a `Pid` type, which cannot be mixed up with other numbers by accident.
2. `getppid()` returns the parent. In the checking container the parent is PID 1, the container's first process.
   In your VM it is the PID of your `bash`.
3. `getuid()` is 0 here because the container runs as root. In your VM you see your own UID, for example 501.
4. `/proc/self/status` is the kernel's text summary of the process descriptor. `State: R (running)` is true at the
   moment the program reads it: it is running, of course.
5. The lines do not come in the order of our list: the loop prints them in the order the kernel writes them.

> 📏 **Convention:** Prefer `std` when it has the function (`std::process::id`, `std::env::current_dir`). Use
> `nix` for Unix calls that `std` does not offer. Both call the same kernel.

## 2. The memory of a process

Every process gets its own **address space**: a huge range of addresses (on arm64 Linux, 2⁴⁸ bytes = 256 TiB) of
which only small pieces are actually used. Each used piece is a **memory region** with its own permissions. A
typical layout, with low addresses at the bottom:

```text
 high addresses  0xffff_ffff_ffff
 +----------------------------------+
 | stack  (grows down)              |  local variables, function calls
 |   ...                            |
 | shared libraries (libc ...)      |  mapped files
 |   ...                            |
 | heap   (grows up)                |  Box, Vec, String
 | .data / .bss   rw-               |  writable statics
 | read-only data r--               |  constants
 | .text code     r-x               |  your compiled functions
 +----------------------------------+
 low addresses   0x0000_0000_0000
```

`r`, `w`, `x` mean readable, writable, executable. Code is never writable, data is never executable: a simple
protection against many attacks. Let us find real addresses. Create `src/bin/memory-map.rs`:

```rust
use std::fs;
use std::sync::atomic::AtomicU64;

static GREETING: &str = "hello";
static COUNTER: AtomicU64 = AtomicU64::new(1);

fn main() -> std::io::Result<()> {
    let on_stack = 42_u64;
    let on_heap = Box::new(7_u64);

    println!("code   (fn main)   {:p}", main as *const ());
    println!("rodata (\"hello\")   {:p}", GREETING.as_ptr());
    println!("data   (COUNTER)   {:p}", &COUNTER);
    println!("heap   (Box)       {:p}", &*on_heap);
    println!("stack  (local)     {:p}", &on_stack);
    println!();

    // The kernel's list of memory regions of this process: one line per region.
    let maps = fs::read_to_string("/proc/self/maps")?;
    for line in maps.lines() {
        if line.contains("memory-map") || line.contains("[heap]") || line.contains("[stack]") {
            println!("{line}");
        }
    }
    Ok(())
}
```

```text
code   (fn main)   0xaaaae09598a0
rodata ("hello")   0xaaaae0994f37
data   (COUNTER)   0xaaaae09c0008
heap   (Box)       0xaaaaf5c06d60
stack  (local)     0xffffd68a6ee0

aaaae0950000-aaaae09a5000 r-xp 00000000 00:34 319762                     /work/target/debug/memory-map
aaaae09bd000-aaaae09c0000 r--p 0005d000 00:34 319762                     /work/target/debug/memory-map
aaaae09c0000-aaaae09c1000 rw-p 00060000 00:34 319762                     /work/target/debug/memory-map
aaaaf5c06000-aaaaf5c27000 rw-p 00000000 00:00 0                          [heap]
ffffd6887000-ffffd68a8000 rw-p 00000000 00:00 0                          [stack]
```

**What happens:**

1. `{:p}` prints an address. `main as *const ()` is the address of the function's first instruction.
   An `AtomicU64` (atomics: Lesson 05.10) is a static that may change, so it lives in writable memory without `unsafe`.
2. Each line of `/proc/self/maps` is one region: `start-end`, permissions (`p` = private to this process), the
   offset in the file, the device and inode (Lesson 13.4) of the file, and its path.
3. `main` (…598a0) is inside the first region, `r-xp`: the code of our program, read and execute only. The
   string `"hello"` is in the same region here: the linker put the read-only constants next to the code.
4. The `r--p` region holds data that the loader fixed up at start and then made read-only.
5. `COUNTER` (…c0008) is inside the `rw-p` region of the file: writable statics.
6. The `Box` is inside `[heap]`, and `on_stack` is inside `[stack]`, at a much higher address.

Run it again and the addresses change:

```text
code   (fn main)   0xaaaacb3598a0
rodata ("hello")   0xaaaacb394f37
data   (COUNTER)   0xaaaacb3c0008
heap   (Box)       0xaaaad1d9dd60
stack  (local)     0xffffd71d1db0
```

The last three hex digits of the code address stay the same (`8a0`), but the rest moves. This is **ASLR**: the
kernel picks a random start for each region at every run.

> 🔒 **Security:** Many attacks need to know where a function or a buffer is in memory. ASLR makes those addresses
> different on every start, so a working attack on one run fails on the next. Never turn it off on real machines.

### Real memory: RSS

The address space is mostly empty promises. Memory becomes real RAM only when the program **writes** to it (Lesson
13.10 explains why). `VmRSS` in `/proc/self/status` shows the real part. Exercise 1 lets you watch it grow.

## 3. Process states

A process is not always running. On a machine with 12 cores, at most 12 processes run at the same moment;
hundreds of others wait. `/proc/<pid>/status` and `ps` show the **state** as one letter:

| Letter | Name | Meaning |
|---|---|---|
| `R` | running or runnable | on a CPU now, or ready and waiting for a turn |
| `S` | sleeping | waiting for something: a timer, a key press, data from the network |
| `D` | uninterruptible sleep | waiting for a device (often a disk); even signals must wait |
| `T` | stopped | paused by `SIGSTOP` or Ctrl-Z, until `SIGCONT` |
| `Z` | zombie | finished; the parent has not yet collected the exit code (Lesson 13.5) |

Watch a process change state in your VM:

```bash
sleep 100 &
ps -o pid,ppid,stat,comm
kill -STOP $!
grep State /proc/$!/status
kill -CONT $!
grep State /proc/$!/status
kill $!
```

Real output from the checking container (in your VM the parent is your `bash`, not PID 1):

```text
    PID    PPID STAT COMMAND
      1       0 Ss   bash
      7       1 S    sleep
      9       1 R    ps
State:	T (stopped)
State:	S (sleeping)
```

**What happens:**

1. `sleep 100 &` starts a child process in the background; `$!` is its PID (Lesson 11.5).
2. `ps` shows `sleep` in state `S`: it sleeps until its timer ends. `ps` itself is `R`, because it is running
   while it prints. The `s` after `S` for `bash` marks a *session leader* (Lesson 13.7).
3. `kill -STOP` sends `SIGSTOP`; the kernel takes the process off the CPU queue: state `T`.
4. `kill -CONT` lets it continue, and it goes back to sleeping: `S`.

> ⚡ **Performance:** A program waiting for the disk or the network is in state `S` or `D` and uses **no** CPU time.
> High CPU use means many processes in state `R`. The fourth field of `/proc/loadavg` from Lesson 13.1 counts exactly
> those.

## 4. The process tree

Every process except the first one was created by another process, its **parent**. So the processes form a tree.
At the root is **PID 1**, the first user-space program that the kernel starts at boot. In your VM it is `systemd`
(Lesson 11.6); in a container it is whatever command the container runs.

```text
systemd (1)
 ├── sshd (912)
 │    └── sshd-session (4629)
 │         └── bash (4630)
 │              ├── sleep (4700)
 │              └── ps (4701)
 └── cron (870)
```

You can draw your own tree with `ps -ef --forest` or `pstree -p`. When a parent dies before its child, the child is
not left alone: the kernel gives it a new parent, normally PID 1. You will create such an *orphan* in Lesson 13.5.

Rust reaches the parent with `getppid()`, and `/proc/<pid>/status` gives the `PPid` of any process. Exercise 3 uses
that to walk from your program up to PID 1.

## 5. Looking at any process through `/proc/<pid>/`

Each process folder has many files. The most useful ones:

| File | Content |
|---|---|
| `status` | the human-readable summary you saw above |
| `cmdline` | the arguments the process was started with, separated by NUL bytes |
| `environ` | its environment variables, separated by NUL bytes |
| `cwd` | a symbolic link to its current directory |
| `exe` | a symbolic link to its program file |
| `maps` | its memory regions |
| `fd/` | its open files (Lesson 13.4) |

```bash
ls -l /proc/self/exe /proc/self/cwd
```

```text
lrwxrwxrwx 1 root root 0 Nov 18 07:02 /proc/self/cwd -> /work
lrwxrwxrwx 1 root root 0 Nov 18 07:02 /proc/self/exe -> /usr/bin/ls
```

`self` is `ls` here, because `ls` is the process reading the folder.

`cmdline` uses NUL bytes, not spaces, between the arguments. That is how the kernel keeps `echo "a b"` (two arguments:
`echo` and `a b`) apart from `echo a b` (three arguments). Exercise 2 parses it.

> 🔒 **Security:** You can read `status` and `cmdline` of other users' processes, but not their `environ`, `maps` or
> `fd/`: as a normal user, `cat /proc/1/environ` prints `cat: /proc/1/environ: Permission denied`. Environment
> variables often contain secrets, so never pass a password on the command line: everybody on the machine can read `cmdline`.

## Common mistakes

> ⚠️ **Common mistake:** Building the feature list of `nix` too small. If you call `getuid` with only the `process`
> feature, the compiler says:
>
> ```text
> error[E0432]: unresolved import `nix::unistd::getuid`
>  --> src/main.rs:1:5
>   |                  no `getuid` in `unistd`
> note: found an item that was configured out
>   |    ---------------- the item is gated behind the `user` feature
> ```
>
> "Configured out" means the function exists in `nix` but was not compiled. Add the feature with
> `cargo add nix@0.32 --features user`.

> ⚠️ **Common mistake:** Treating an address printed by `{:p}` as a fixed fact. With ASLR it is different on every run,
> so never save addresses in files or compare them between runs.

> ⚠️ **Common mistake:** Splitting `cmdline` on spaces. An argument may contain spaces; split on the NUL byte.

## Try it yourself

1. Run `memory-map` three times and write down which hex digits change and which stay.
2. Start `sleep 300 &`, then read `/proc/$!/cmdline` with `tr '\0' ' ' < /proc/$!/cmdline; echo`.
3. Press Ctrl-Z while `sleep 300` runs in the foreground. Check its state with `ps -o pid,stat,comm`, then continue
   it with `fg` and stop it with Ctrl-C.
4. Run `pstree -p $$` in the VM. `$$` is the PID of your shell.

## Exercises

### Exercise 1 — Watch RSS grow (easy)

Write `src/bin/rss.rs` with a function `vm_rss_kib() -> u64` that reads `VmRSS` from `/proc/self/status`. Print it four
times: at the start, after `Vec::with_capacity(100 * 1024 * 1024)` (100 MiB of `u8`), after filling the vector with
`resize(100 * 1024 * 1024, 1_u8)`, and after `drop`. Explain the numbers.

<details>
<summary>Hint</summary>

`status.lines().find_map(|line| line.strip_prefix("VmRSS:"))` gives the rest of the line, for example
`"    1680 kB"`. Take the first word and parse it.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs;

fn vm_rss_kib() -> u64 {
    let status = fs::read_to_string("/proc/self/status").unwrap_or_default();
    status
        .lines()
        .find_map(|line| line.strip_prefix("VmRSS:"))
        .and_then(|rest| rest.split_whitespace().next())
        .and_then(|n| n.parse().ok())
        .unwrap_or(0)
}

fn main() {
    println!("before:          {} KiB", vm_rss_kib());
    let mut big = Vec::with_capacity(100 * 1024 * 1024);
    println!("after reserving: {} KiB", vm_rss_kib());
    big.resize(100 * 1024 * 1024, 1_u8);
    println!("after writing:   {} KiB", vm_rss_kib());
    drop(big);
    println!("after drop:      {} KiB", vm_rss_kib());
}
```

```text
before:          1680 KiB
after reserving: 1688 KiB
after writing:   104088 KiB
after drop:      1684 KiB
```

Why it works: reserving 100 MiB only creates a memory region; no RAM is used until the program writes. Writing every
byte makes the kernel give real memory, page by page, so RSS grows by about 100 MiB. After `drop` the allocator returns
such a large block to the kernel at once, and RSS falls again.

</details>

### Exercise 2 — Parse `cmdline`, with tests (medium)

Write `src/bin/cmdline.rs` with `fn split_cmdline(raw: &[u8]) -> Vec<String>`. `main` takes an optional PID argument
(default `self`), reads `/proc/<pid>/cmdline` with `fs::read` (bytes, not text) and prints each argument. Write three
tests: `b"sleep\x00100\x00"` gives `["sleep", "100"]`, `b"echo\x00a b\x00"` gives `["echo", "a b"]`, and an empty
input gives no arguments.

<details>
<summary>Hint</summary>

Slices have `split(|&b| b == 0)`. The data ends with a NUL byte, which produces one empty piece at the end; filter
out empty pieces. `String::from_utf8_lossy` turns bytes into text even if they are not valid UTF-8.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs;

/// /proc/<pid>/cmdline holds the arguments separated (and ended) by zero bytes.
fn split_cmdline(raw: &[u8]) -> Vec<String> {
    raw.split(|&b| b == 0)
        .filter(|part| !part.is_empty())
        .map(|part| String::from_utf8_lossy(part).into_owned())
        .collect()
}

fn main() -> std::io::Result<()> {
    let pid = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "self".to_string());
    let raw = fs::read(format!("/proc/{pid}/cmdline"))?;
    for (i, arg) in split_cmdline(&raw).iter().enumerate() {
        println!("argv[{i}] = {arg:?}");
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn splits_on_zero_bytes() {
        assert_eq!(split_cmdline(b"sleep\x00100\x00"), vec!["sleep", "100"]);
    }

    #[test]
    fn keeps_spaces_inside_one_argument() {
        assert_eq!(split_cmdline(b"echo\x00a b\x00"), vec!["echo", "a b"]);
    }

    #[test]
    fn empty_cmdline_gives_no_arguments() {
        assert!(split_cmdline(b"").is_empty());
    }
}
```

```bash
cargo build -q
sleep 100 &
./target/debug/cmdline $!
```

```text
argv[0] = "sleep"
argv[1] = "100"
```

Why it works: the kernel stores the argument strings exactly as the program received them, each ended by a NUL byte.
Splitting on that byte gives the original arguments back. (The filter also drops empty arguments such as `""`; a perfect
version would only drop the very last piece.)

</details>

### Exercise 3 — Walk up to PID 1 (hard)

Write `src/bin/ancestors.rs` that starts with its own PID and prints `PID  name` for itself, its parent, the parent's
parent, and so on, until it reaches PID 1 (whose `PPid` is 0). Write a helper
`fn name_and_parent(pid: u32) -> Option<(String, u32)>`.

<details>
<summary>Hint</summary>

Read `/proc/<pid>/status`, take the lines that start with `Name:` and `PPid:`. Loop `while pid != 0`. Use
`let Some(...) = ... else { break };` so that a vanished process ends the loop instead of panicking.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs;

/// Returns (name, parent PID) of a process, read from /proc/<pid>/status.
fn name_and_parent(pid: u32) -> Option<(String, u32)> {
    let status = fs::read_to_string(format!("/proc/{pid}/status")).ok()?;
    let mut name = String::new();
    let mut ppid = None;
    for line in status.lines() {
        if let Some(v) = line.strip_prefix("Name:") {
            name = v.trim().to_string();
        } else if let Some(v) = line.strip_prefix("PPid:") {
            ppid = v.trim().parse().ok();
        }
    }
    Some((name, ppid?))
}

fn main() {
    let mut pid = std::process::id();
    while pid != 0 {
        let Some((name, parent)) = name_and_parent(pid) else {
            break;
        };
        println!("{pid:>6}  {name}");
        pid = parent;
    }
}
```

In the checking container the chain is short:

```text
   545  ancestors
     1  bash
```

Example output in the VM (yours will look similar):

```text
  5012  ancestors
  4630  bash
  4629  sshd-session
  4580  sshd-session
   912  sshd
     1  systemd
```

Why it works: every process descriptor stores its parent's PID, and PID 1 has the parent 0, which means "no parent".
`cargo run` replaces itself with your program, so `cargo` does not appear in the chain.

</details>

## Quiz

1. What is the difference between a program and a process?

<details>
<summary>Answer</summary>

A program is a file with instructions. A process is a running instance of a program, with its own memory, open files,
PID and state. One program can run as many processes at the same time.

</details>

2. In `/proc/self/maps` you see `r-xp` for one region and `rw-p` for another. What lives in each?

<details>
<summary>Answer</summary>

`r-xp` (read, execute, private) is code: it can be run but not changed. `rw-p` (read, write, private) is writable data
such as statics, the heap or the stack: it can be changed but not run.

</details>

3. Why does the address of `main` change between two runs?

<details>
<summary>Answer</summary>

Because of ASLR: the kernel loads the program and its regions at a random base address on every start, to make attacks
that need fixed addresses fail.

</details>

4. A program waits for a key press. Which state is it in, and does it use CPU time?

<details>
<summary>Answer</summary>

State `S` (sleeping). It uses no CPU time; the kernel wakes it when the key press arrives.

</details>

5. Your program reserved 1 GiB with `Vec::with_capacity` but `VmRSS` grew only a little. Why?

<details>
<summary>Answer</summary>

Reserving creates a memory region, but the kernel gives real RAM only when a page is first written. RSS counts only the
memory that is really in RAM.

</details>

6. Why is it dangerous to pass a password as a command-line argument?

<details>
<summary>Answer</summary>

`/proc/<pid>/cmdline` can be read by every user on the machine, and tools like `ps` show it. Secrets belong in files with
strict permissions or in environment variables, which other users cannot read.

</details>

## Summary

- A process is a running program; the kernel keeps a process descriptor (`task_struct`) with its PID, parent, state, memory map, open files and more.
- Every process has its own address space with regions for code (`r-x`), constants (`r--`), data and heap (`rw-`) and stack; `/proc/<pid>/maps` lists them.
- ASLR puts the regions at random addresses on every run.
- RSS is the part of memory that is really in RAM; memory becomes real only when written.
- States: R (running or ready), S (sleeping), D (waiting for a device), T (stopped), Z (zombie).
- Processes form a tree with PID 1 at the root; orphans are adopted by PID 1.
- `nix::unistd` gives safe wrappers such as `getpid`, `getppid` and `getuid`; switch on the parts you need with Cargo features.

## Sources and further reading

- [proc_pid_status(5) — Linux manual page](https://man7.org/linux/man-pages/man5/proc_pid_status.5.html) — every field of `/proc/<pid>/status`.
- [proc_pid_maps(5) — Linux manual page](https://man7.org/linux/man-pages/man5/proc_pid_maps.5.html) — the format of the memory map.
- [ps(1) — Linux manual page](https://man7.org/linux/man-pages/man1/ps.1.html) — process states and output columns.
- [nix::unistd — docs.rs](https://docs.rs/nix/0.32.0/nix/unistd/index.html) — the safe wrappers used in this lesson.
- [Operating Systems: Three Easy Pieces — The Abstraction: The Process](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-intro.pdf) — a free textbook chapter about processes.

---

[🏠 Start](../START-HERE.md) · [← 13.1 What an operating system does](01-what-an-operating-system-does.md) · [📋 Phase 13](README.md) · [13.3 System calls →](03-system-calls.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
