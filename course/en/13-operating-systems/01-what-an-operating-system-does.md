<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 13 — Operating systems from the inside](README.md) · [📋 Phase 13](README.md) · [13.2 Processes from the inside →](02-processes-from-the-inside.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/01-what-an-operating-system-does.md) <!-- nav -->

# Lesson 13.1 — What an operating system does

> **Phase 13 — Operating systems from the inside** · Lesson 1 of 18 · ⏱ about 135 minutes

## What you will learn

- The three jobs of an operating system: hiding the hardware behind simple ideas, sharing the machine, and protecting programs from each other.
- What the kernel is, and the difference between user space and kernel space.
- The three ways the CPU enters the kernel: system calls, interrupts and exceptions.
- How to look inside a running Linux kernel through `/proc`, by hand and from a Rust program.
- How this phase organises its code, and how to run it in your VM `dev`.

## Before you start

- You have the Linux VM `dev` from [Lesson 11.1 — Linux and a virtual machine on your Mac](../11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md), and you can use the shell, files, processes and signals (Phase 11).
- You know how the CPU runs instructions ([Lesson 12.6](../12-computer-systems/06-how-the-cpu-runs-instructions.md)) and how a program is loaded into memory ([Lesson 12.12 — Linking and loading programs](../12-computer-systems/12-linking-and-loading.md)).
- You know what `unsafe` means ([Lesson 06.6 — Unsafe Rust and Miri](../06-rust-advanced/06-unsafe-rust-and-miri.md)). We use it once here, on purpose, to break a rule.

## New words

| Word | Meaning |
|---|---|
| **CPU** (Central Processing Unit) / **RAM** (Random Access Memory) | The chip that runs instructions / the main memory (Phase 12). |
| **VM** (Virtual Machine) | A computer simulated by software; your Linux VM is called `dev` (Lesson 11.1). |
| **OS** (Operating System) | The software that manages the computer's hardware and lets programs use it safely: Linux, macOS, Windows. |
| **kernel** | The central part of the OS. It runs with full power over the hardware; everything else asks it for help. |
| **user space** | The world where normal programs run (your Rust programs, the shell, `ls`). Code here has limited power. |
| **kernel space** | The memory and the CPU mode where the kernel runs. User programs cannot read or change it. |
| **privilege level** | A mode of the CPU that decides which instructions and which memory are allowed. On arm64 it is called an **EL** (Exception Level): EL0 for programs, EL1 for the kernel. On x86-64 it is called a **ring**: ring 3 for programs, ring 0 for the kernel. |
| **system call** (syscall) | A request from a program to the kernel: "open this file", "give me memory", "start a process". Lesson 13.3 explains it in detail. |
| **interrupt** | A signal from a device (a timer, a disk, a network card) that makes the CPU stop what it is doing and run kernel code. |
| **exception** (CPU) | An event caused by the instruction that is running: a bad memory address, division by zero, a forbidden instruction. The CPU jumps into the kernel to handle it. |
| **driver** | Kernel code that knows how to talk to one kind of device. |
| **abstraction** | A simple idea that hides a complicated thing: a "file" hides disk blocks, a "process" hides the CPU switching. |
| **PID** (Process IDentifier) | The number the kernel gives to each running process (Phase 11). |
| **procfs** (process file system) | The folder `/proc`. Its files are not on a disk: the kernel creates their content each time you read them. |
| **POSIX** (Portable Operating System Interface) | A standard that describes the system calls and tools a Unix-like OS should have. Linux and macOS follow most of it. |
| **KiB / MiB** (kibibyte / mebibyte) | 1 KiB = 1,024 bytes, 1 MiB = 1,024 KiB (Lesson 12.1). |

## 1. The problem an operating system solves

Imagine an office building where many companies share one power line and one set of lifts. The building has a
**manager**: companies ask the manager for a room or a key, and the manager keeps them out of each other's rooms.

A computer has the same problem: one CPU with a few cores, one RAM, one disk and one network card, but hundreds of
programs that want them at the same time. The **operating system** is the building manager. It has three jobs:

1. **Abstraction — hide the hardware behind simple ideas.** Your program does not know which disk
   model you have. It says "open the file `notes.txt`", and the OS turns that into commands for this exact disk.
   *File*, *process*, *socket* and *virtual memory* are all abstractions made by the OS.
2. **Sharing — divide the machine between programs.** The OS gives each program short turns on the CPU
   (Lesson 13.9), a part of the memory (Lesson 13.10), and its share of the disk and the network.
3. **Protection — keep programs apart.** A bug in your browser must not crash your editor, and a normal
   user's program must not read another user's files or change the OS itself.

You already met the results of these jobs in Phase 11: processes with PIDs, users and permissions, signals,
services. In this phase you look at them **from the inside**: what the kernel actually does, and how a Rust
program talks to it.

## 2. The kernel, user space and kernel space

The most important part of the OS is the **kernel**. It is one big program that the computer starts first
(after the firmware) and that runs until you switch the computer off. Everything else — the shell, `systemd`,
your Rust programs — runs **on top of** the kernel, in **user space**.

```text
 user space (EL0 on arm64, ring 3 on x86-64)
 +-------------------------------------------------------------+
 |  your Rust program     bash      systemd      sshd    ...   |
 |        |                 |                                   |
 |  Rust standard library (std)                                 |
 |        |                                                     |
 |  C library (libc): small functions that make system calls    |
 +--------|----------------------------------------------------+
          |  system call: the only door into the kernel
 +--------v----------------------------------------------------+
 |  kernel space (EL1 on arm64, ring 0 on x86-64)              |
 |  processes and scheduler | memory manager | file systems    |
 |  network stack           | drivers                          |
 +--------|----------------------------------------------------+
          v
   hardware: CPU cores, RAM, disk, network card, timer
```

The CPU itself enforces this border. It always runs in a **privilege level**:

- In the **low** level (EL0 on arm64, ring 3 on x86-64) the CPU refuses some instructions, for example
  the ones that change memory protection or talk to devices directly, and it refuses to touch memory that
  belongs to the kernel.
- In the **high** level (EL1 on arm64, ring 0 on x86-64) everything is allowed. Only the kernel runs there.

The kernel sets up the memory map of every program (Lesson 13.10 explains how) so that the kernel's own
memory is marked "kernel only". If a program tries to read it, the CPU stops the instruction and hands control
to the kernel, and the kernel punishes the program. Let us watch that happen.

### Watching protection work

This small program tries to read one byte from an address in the upper half of the address space, which on
Linux belongs to the kernel (`src/bin/poke-kernel.rs`, see Section 5 for the folder layout):

```rust
fn main() {
    // An address in the upper half of the address space: on Linux, that half belongs to the kernel.
    let kernel_address = 0xffff_8000_0000_0000_usize as *const u8;
    println!("trying to read kernel memory at {kernel_address:p} ...");
    // SAFETY: none! This is deliberately wrong code, to watch the kernel stop us.
    let value = unsafe { std::ptr::read_volatile(kernel_address) };
    println!("read {value}"); // never printed
}
```

```bash
cargo run -q --bin poke-kernel
echo "exit code: $?"
```

```text
trying to read kernel memory at 0xffff800000000000 ...
bash: line 1:   150 Segmentation fault      cargo run -q --bin poke-kernel
exit code: 139
```

**What happens:**

1. `0xffff_8000_0000_0000_usize as *const u8` turns a number into a raw pointer (Lesson 06.6). Creating it is
   allowed; reading through it is the dangerous part, so it needs `unsafe`.
2. `read_volatile` really performs the read (the compiler may not remove it). The CPU, running in the low
   privilege level, checks the memory map, sees "kernel only", and raises an **exception**.
3. The kernel's exception handler sees that a user program touched memory it may not touch. It sends the
   program the signal `SIGSEGV` (signal 11, "segmentation violation"), whose default action is to kill the process.
4. The shell reports `Segmentation fault`, and the exit code is 139 = 128 + 11, the rule for "killed by a signal"
   that you know from Phase 11. The second `println!` never runs.

The exact message depends on your shell: in an interactive bash you see `Segmentation fault (core dumped)`.

> 🔒 **Security:** This border is what makes a multi-user machine safe. Your program can crash itself, but it
> cannot read the kernel's memory, other programs' memory or files it has no permission for. Every attack on an
> OS tries to find a hole in exactly this border, which is why kernels are written very carefully.

## 3. Three doors into the kernel

The kernel does not run "next to" your program all the time. The CPU runs your program's instructions until
one of three things happens, and only then does it jump into kernel code:

| Door | Who causes it | Example |
|---|---|---|
| **system call** | your program, on purpose | "read 4 KiB from this file", "write this text to the terminal" |
| **exception** | the instruction that is running, by accident or by design | a bad address (as above), division by zero on some CPUs, a page of memory that is not loaded yet (Lesson 13.10) |
| **interrupt** | a device, at any moment | the timer ticks, a network packet arrives, a disk finished reading |

Each door works the same way. The CPU saves where the program was, switches to the high privilege level,
and jumps to a kernel address that the kernel chose at boot. The kernel does its work and then either returns
to the same program or, if it decides that another program should run now, switches to that one.

The **timer interrupt** is the reason one endless loop cannot freeze your machine. Many times per second
a hardware timer interrupts the CPU, and the kernel gets a chance to give the CPU to someone else. You will see
this in [Lesson 13.9 — Threads and scheduling](09-threads-and-scheduling.md).

## 4. Linux: a kernel you can look into

**Linux** is a kernel. A **distribution** such as Ubuntu is the Linux kernel plus thousands of user-space
programs (Lesson 11.1). The Linux kernel is *monolithic*: the scheduler, the memory manager, the file systems,
the network stack and most drivers all live in the same big program in kernel space. Extra drivers can be loaded
while it runs, as **kernel modules**.

The command `uname` (Unix name) tells you which kernel runs. Try it in your VM:

```bash
uname -r
uname -m
```

Real output from the Linux container used to check this lesson (in your VM the version ends with `-generic`):

```text
7.0.14-linuxkit
aarch64
```

`-r` prints the kernel release, `-m` the machine type: `aarch64` is the Linux name for arm64.

### `/proc`: the kernel's window

The kernel shows much of its internal state as **files** in the folder `/proc`. These files are not stored on
any disk: when you read one, the kernel writes the answer at that moment. This is the **procfs**.

```bash
cat /proc/version
cat /proc/uptime
cat /proc/loadavg
head -3 /proc/meminfo
ls /proc | head -5
```

Real output (yours will have different numbers):

```text
Linux version 7.0.14-linuxkit (root@buildkitsandbox) (gcc (Alpine 15.2.0) 15.2.0, GNU ld (GNU Binutils) 2.45.1) #1 SMP PREEMPT Fri Oct 30 13:45:20 UTC 2026
63588.65 746910.25
0.52 0.91 0.51 2/492 183
MemTotal:        8124192 kB
MemFree:          355980 kB
MemAvailable:    7084184 kB
1
185
186
buddyinfo
bus
```

**What happens:**

1. `/proc/version` — the kernel's version and the compiler that built it. `SMP` (Symmetric MultiProcessing)
   means it can use many CPU cores; `PREEMPT` means that even the kernel's own code can be paused in the middle so that
   a more urgent task runs sooner (programs in user space can always be paused).
2. `/proc/uptime` — seconds since boot, then the total idle time of all cores added together.
3. `/proc/loadavg` — the average number of runnable processes over 1, 5 and 15 minutes, then "running now / all
   tasks", then the last PID that was given out.
4. `/proc/meminfo` — memory in KiB (the kernel writes `kB` but means KiB). `MemAvailable` is how much a new
   program could get without pushing others out.
5. `ls /proc` — every folder with a number as its name is a **running process**: `/proc/1` is PID 1. The other
   entries describe the whole system. [Lesson 13.2 — Processes from the inside](02-processes-from-the-inside.md) opens these folders.

> 💡 **Tip:** `man 5 proc` describes every file in `/proc`. Section 5 of the manual is about file formats; when a
> lesson writes `proc(5)`, it means "the manual page `proc` in section 5".

## 5. How this phase organises code, and where to run it

Most programs in this phase use Linux-only kernel features. You write them on your Mac in RustRover, but you
**build and run them in the VM `dev`**, never as root on your Mac.

**Code layout.** All small examples of Lessons 13.1–13.7 and 13.9 live in one package, `code/p13-os`. Each
example is its own binary in `src/bin/`: the file `src/bin/os-info.rs` becomes the program `os-info`, which you run
with `cargo run --bin os-info`. Projects (the shell in Lesson 13.8) get their own package. Create the package on
your Mac:

```bash
cd ~/projects/rust
cargo new code/p13-os
rm code/p13-os/src/main.rs
mkdir code/p13-os/src/bin
```

We delete `src/main.rs` because a package with only `src/bin/` files has no default binary; you always name
the one you want with `--bin`.

**Rust inside the VM.** If you did not install Rust in the VM during Phase 11, do it once now. Enter the VM
with `limactl shell dev` and run:

```bash
sudo apt-get update
sudo apt-get install -y build-essential strace rsync
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustc --version
```

Example output of the last command (yours will look similar):

```text
rustc 1.100.0 (0ef1bc386 2026-11-09)
```

`build-essential` brings the C compiler and the linker that Rust needs on Linux (Lesson 12.12), `strace` is the
tool of [Lesson 13.3 — System calls](03-system-calls.md), and `rsync` copies folders.

**Copy, then run.** Your Mac folder is visible in the VM but read-only (Lesson 11.1), so copy the package into
the VM home and build it there. Run this inside the VM each time you changed code (replace `student` with your
Mac user name):

```bash
rsync -a --delete --exclude target /Users/student/projects/rust/code/p13-os/ ~/p13-os/
cd ~/p13-os
cargo run --bin os-info
```

`--exclude target` skips the build output of the Mac, which is useless on Linux; `--delete` removes files in
the copy that you deleted on the Mac. The trailing `/` after `p13-os/` means "the content of this folder".

> 💡 **Tip:** Every program in this phase was checked in a Linux container built from the official `rust:1.100`
> image (Debian, arm64). The outputs in the lessons come from there, so kernel versions, PIDs and sizes in your VM
> will be different. The behaviour is the same.

## 6. Your first look at the kernel from Rust

Reading `/proc` from Rust is ordinary file reading (Lesson 04.10). Create `src/bin/os-info.rs`:

```rust
use std::fs;

/// Returns the first number in the text of /proc/uptime: seconds since boot.
fn parse_uptime(text: &str) -> Option<f64> {
    text.split_whitespace().next()?.parse().ok()
}

/// Finds a line like "MemTotal:  8000000 kB" and returns the number (in KiB).
fn meminfo_kib(text: &str, key: &str) -> Option<u64> {
    for line in text.lines() {
        if let Some(rest) = line.strip_prefix(key).and_then(|r| r.strip_prefix(':')) {
            return rest.split_whitespace().next()?.parse().ok();
        }
    }
    None
}

fn main() -> std::io::Result<()> {
    let version = fs::read_to_string("/proc/version")?;
    let uptime = fs::read_to_string("/proc/uptime")?;
    let meminfo = fs::read_to_string("/proc/meminfo")?;

    println!("kernel:    {}", version.trim());
    if let Some(secs) = parse_uptime(&uptime) {
        println!("uptime:    {secs:.0} seconds");
    }
    let total_mib = meminfo_kib(&meminfo, "MemTotal").unwrap_or(0) / 1024;
    let avail_mib = meminfo_kib(&meminfo, "MemAvailable").unwrap_or(0) / 1024;
    println!("memory:    {total_mib} MiB total, {avail_mib} MiB available");
    println!("this PID:  {}", std::process::id());
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn uptime_takes_the_first_number() {
        assert_eq!(parse_uptime("350735.47 234388.90\n"), Some(350735.47));
        assert_eq!(parse_uptime(""), None);
    }

    #[test]
    fn meminfo_finds_the_exact_key() {
        let text = "MemTotal:        8131124 kB\nMemFree:          912000 kB\n";
        assert_eq!(meminfo_kib(text, "MemTotal"), Some(8131124));
        assert_eq!(meminfo_kib(text, "MemFree"), Some(912000));
        assert_eq!(meminfo_kib(text, "Mem"), None);
    }
}
```

Copy it to the VM (the `rsync` line above), then:

```bash
cargo run -q --bin os-info
cargo test -q --bin os-info
```

```text
kernel:    Linux version 7.0.14-linuxkit (root@buildkitsandbox) (gcc (Alpine 15.2.0) 15.2.0, GNU ld (GNU Binutils) 2.45.1) #1 SMP PREEMPT Fri Oct 30 13:45:20 UTC 2026
uptime:    63802 seconds
memory:    7933 MiB total, 6822 MiB available
this PID:  62

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. `fs::read_to_string("/proc/version")` opens and reads the file like any other. The kernel generates its
   text on the spot.
2. `parse_uptime` takes the first word and parses it as `f64`. The `?` inside a function returning `Option`
   returns `None` early if there is no first word (Lesson 04.10).
3. `meminfo_kib` looks for a line that starts with the key **followed by a colon**, so `"Mem"` does not match
   `"MemTotal"` by accident. The test `meminfo_finds_the_exact_key` checks exactly that edge case.
4. `std::process::id()` asks the kernel for this process's PID. Behind this call is a system call, `getpid`;
   Lesson 13.3 shows you how to see it.
5. The parsing functions are separate from `main` and take `&str`, so the tests do not need `/proc` at all and
   also run on your Mac.

> 🧪 **Testing:** Keep the code that *talks to the kernel* thin and the code that *understands the answer* in
> pure functions that take text or numbers. You can test the pure part anywhere, with fixed inputs copied from a real
> machine, including strange cases that your machine rarely shows.

Commit your work on the Mac when it runs: `git add code/p13-os` and `git commit -m "feat(p13): os-info reads /proc"`.

## Common mistakes

> ⚠️ **Common mistake:** Running the programs of this phase on macOS. `/proc` does not exist there, so `os-info`
> fails with:
>
> ```text
> Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
> ```
>
> Code 2 is `ENOENT`, "no such file". Run the program in the VM.

> ⚠️ **Common mistake:** Building inside the read-only Mac folder in the VM. `cargo build` in
> `/Users/student/projects/rust/code/p13-os` fails because Cargo cannot create `target/` there, with an error that
> ends in `Read-only file system (os error 30)`. Copy the package to `~/p13-os` first.

> ⚠️ **Common mistake:** Thinking that the files in `/proc` have a size. `ls -l /proc/uptime` shows size 0,
> yet `cat` prints text. Always read such files until the end, never by their reported length.

## Try it yourself

1. In the VM, run `cat /proc/uptime` twice, a few seconds apart. The first number grows: the file is new each time.
2. Run `cat /proc/self/status | head -5`. `self` always means "the process that is reading". Which program is it?
   (It is `cat` itself, so the `Name:` line says `cat`.)
3. Run `cargo run -q --bin os-info` twice. The PID changes each time, because each run is a new process.
4. Run `cargo run -q --bin poke-kernel`, then `echo $?`. Explain the number 139 in your own words.

## Exercises

### Exercise 1 — How many CPUs? (easy)

Write `src/bin/cpus.rs`. Count the lines of `/proc/cpuinfo` that start with `processor` and print the count. Also
print what `std::thread::available_parallelism()` says. Example output:

```text
/proc/cpuinfo lists 12 CPUs
available_parallelism says 12
```

<details>
<summary>Hint</summary>

`text.lines().filter(|line| line.starts_with("processor")).count()`. `available_parallelism` returns
`io::Result<NonZero<usize>>`, which you can print with `{}`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs;

fn count_processors(cpuinfo: &str) -> usize {
    cpuinfo
        .lines()
        .filter(|line| line.starts_with("processor"))
        .count()
}

fn main() -> std::io::Result<()> {
    let text = fs::read_to_string("/proc/cpuinfo")?;
    println!("/proc/cpuinfo lists {} CPUs", count_processors(&text));
    let std_count = std::thread::available_parallelism()?;
    println!("available_parallelism says {std_count}");
    Ok(())
}
```

Why it works: the kernel writes one block per CPU core into `/proc/cpuinfo`, and every block starts with a
`processor : N` line. The two numbers can differ when a container or a CPU limit restricts your program
(Lesson 13.16): `available_parallelism` respects those limits, `/proc/cpuinfo` does not.

</details>

### Exercise 2 — Parse `/proc/loadavg`, test first (medium)

The fourth field of `/proc/loadavg` looks like `2/492`: runnable tasks now, and all tasks. Write
`src/bin/loadavg.rs` with a function `parse_tasks(loadavg: &str) -> Option<(u32, u32)>`. Write the tests **first**:
one with the real line `"0.52 0.91 0.51 2/492 183\n"` (expect `Some((2, 492))`), and one that checks that `""`,
`"0.52 0.91 0.51 2-492 183"` and `"0.52 0.91 0.51 x/492 183"` give `None`. Then make `main` print a line like
`runnable now: 2 of 491 tasks`.

<details>
<summary>Hint</summary>

`split_whitespace().nth(3)` gives the fourth word (counting from 0). `split_once('/')` returns `None` when there is
no `/`, and `parse().ok()?` returns `None` when a part is not a number.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs;

/// Reads the fourth field of /proc/loadavg, "running/total", as two numbers.
fn parse_tasks(loadavg: &str) -> Option<(u32, u32)> {
    let field = loadavg.split_whitespace().nth(3)?;
    let (running, total) = field.split_once('/')?;
    Some((running.parse().ok()?, total.parse().ok()?))
}

fn main() -> std::io::Result<()> {
    let text = fs::read_to_string("/proc/loadavg")?;
    match parse_tasks(&text) {
        Some((running, total)) => println!("runnable now: {running} of {total} tasks"),
        None => eprintln!("could not parse {text:?}"),
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_a_real_line() {
        assert_eq!(parse_tasks("0.52 0.91 0.51 2/492 183\n"), Some((2, 492)));
    }

    #[test]
    fn rejects_broken_input() {
        assert_eq!(parse_tasks(""), None);
        assert_eq!(parse_tasks("0.52 0.91 0.51 2-492 183"), None);
        assert_eq!(parse_tasks("0.52 0.91 0.51 x/492 183"), None);
    }
}
```

```text
runnable now: 2 of 491 tasks
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: every `?` turns a missing word, a missing `/` or a failed parse into `None`, so broken input never
panics. The tests use fixed text, so they pass on any machine, even on your Mac.

</details>

### Exercise 3 — A tiny `ps` (hard)

Write `src/bin/mini-ps.rs` that lists all processes: read the folder `/proc`, keep the entries whose name is a number,
sort the PIDs, and for each one print the PID and the content of `/proc/<pid>/comm` (the short program name).
A process may exit while you are reading; your program must not crash then.

<details>
<summary>Hint</summary>

`fs::read_dir("/proc")?` gives entries; `entry?.file_name()` is an `OsString`, use `.to_str()` and
`.parse::<u32>().ok()`. For each PID, `if let Ok(comm) = fs::read_to_string(...)` simply skips a process that vanished.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fs;

fn main() -> std::io::Result<()> {
    let mut pids: Vec<u32> = Vec::new();
    for entry in fs::read_dir("/proc")? {
        let name = entry?.file_name();
        // Only folders whose name is a number are processes.
        if let Some(pid) = name.to_str().and_then(|s| s.parse().ok()) {
            pids.push(pid);
        }
    }
    pids.sort();
    println!("{:>6}  COMMAND", "PID");
    for pid in pids {
        // A process can exit between read_dir and read_to_string: skip it then.
        if let Ok(comm) = fs::read_to_string(format!("/proc/{pid}/comm")) {
            println!("{pid:>6}  {}", comm.trim_end());
        }
    }
    Ok(())
}
```

Output in a small container (your VM shows many more lines):

```text
   PID  COMMAND
     1  bash
   189  mini-ps
```

Why it works: this is what the real `ps` does, only with more files per process. Skipping a read error is the
correct reaction here, because the list of processes changes all the time. Two things happening in an order you
cannot control is called a **race** (Phase 14 studies races); here it is normal, not a bug.

</details>

## Quiz

1. Name the three jobs of an operating system.

<details>
<summary>Answer</summary>

Abstraction (simple ideas such as files and processes instead of raw hardware), sharing (dividing CPU time, memory,
disk and network between programs) and protection (keeping programs, users and the kernel apart).

</details>

2. What is the difference between user space and kernel space?

<details>
<summary>Answer</summary>

User space is where normal programs run, with the CPU in the low privilege level (EL0 or ring 3): some instructions and
the kernel's memory are forbidden. Kernel space is the kernel's memory and the high privilege level (EL1 or ring 0),
where everything is allowed.

</details>

3. Your program divides by zero, a network packet arrives, and your program asks to open a file. Which door into the kernel is each of these?

<details>
<summary>Answer</summary>

Division by zero is an exception (caused by the running instruction; on arm64 integer division by zero returns 0
instead, and Rust checks for it itself and panics). The packet is an interrupt (caused by a device). Opening a file
is a system call (a deliberate request).

</details>

4. Why can an endless loop in one program not freeze the whole machine?

<details>
<summary>Answer</summary>

Because of the timer interrupt: the hardware timer regularly interrupts the CPU, the kernel runs, and it can give the CPU
to another program. The looping program is paused and continues later.

</details>

5. `ls -l /proc/meminfo` shows size 0, but `cat` prints many lines. Why?

<details>
<summary>Answer</summary>

`/proc` is the procfs: its files are not stored anywhere. The kernel creates the content when you read the file, so
there is no stored size to report.

</details>

6. `poke-kernel` ended with exit code 139. Which signal killed it, and who sent it?

<details>
<summary>Answer</summary>

139 − 128 = 11, the signal `SIGSEGV`. The kernel sent it: the CPU raised an exception on the forbidden read, and the
kernel's exception handler turned it into a signal for the process.

</details>

## Summary

- The OS hides hardware behind abstractions, shares the machine between programs and protects them from each other.
- The kernel is the part of the OS that runs in the high privilege level (EL1 on arm64, ring 0 on x86-64); programs run in user space (EL0, ring 3).
- The CPU enters the kernel only through system calls, exceptions and interrupts; the timer interrupt lets the kernel take the CPU back.
- Linux is a monolithic kernel; a distribution adds user-space programs around it.
- `/proc` is a window into the kernel: its files are generated on each read, and every numbered folder is a process.
- In this phase you write code on the Mac in `code/p13-os` (one binary per file in `src/bin/`), copy it to the VM with `rsync`, and run it there.

## Sources and further reading

- [proc(5) — Linux manual page](https://man7.org/linux/man-pages/man5/proc.5.html) — every file in `/proc`, explained.
- [uname(1) — Linux manual page](https://man7.org/linux/man-pages/man1/uname.1.html) — the options of `uname`.
- [The Linux Kernel documentation](https://docs.kernel.org/) — the official documentation of the kernel itself.
- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) — a free textbook; its first chapters explain the OS's jobs in the same spirit as this phase.
- [Arm A-profile architecture: exception model](https://developer.arm.com/documentation/102412/latest/) — how exception levels EL0–EL3 work on arm64.

---

[🏠 Start](../START-HERE.md) · [← Phase 13 — Operating systems from the inside](README.md) · [📋 Phase 13](README.md) · [13.2 Processes from the inside →](02-processes-from-the-inside.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
