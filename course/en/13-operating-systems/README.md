<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.15 Checkpoint](../12-computer-systems/15-checkpoint.md) · [13.1 What an operating system does →](01-what-an-operating-system-does.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/README.md) <!-- nav -->

# Phase 13 — Operating systems from the inside

> **Stage 3 — How computers really work** · ⏱ about 5–6 weeks at ~10 hours per week

## What this phase is about

In Phase 11 you *used* Linux: you started processes, sent signals, built pipelines in the shell and ran containers. In
Phase 12 you saw how the CPU runs instructions and how a program is laid out in memory. This phase connects the two. You
look at the operating system **from the inside**: what the kernel keeps about every process, how a program asks the kernel
for help with system calls, and how files, pipes, signals and threads really work.

You learn it by writing Rust programs that talk to the kernel directly, with the `nix` and `libc` crates, and by watching
them with `strace` and `/proc`. You build the shell features you use every day — `fork`, `exec`, `|`, `>` and Ctrl-C — and then
put them together into your own shell.

The second half goes deeper into memory and I/O: virtual memory and page faults, memory-mapped files, how a file system stores
data, non-blocking I/O with `epoll` and `io_uring`, and finally the two kernel features behind every container — namespaces
and control groups. The phase ends with your own small container runtime.

Everything here runs on Linux. You write the code on your Mac and build and run it in the VM `dev` from Phase 11; Lesson 13.1
shows how.

## What you will be able to do

- [ ] Explain what the kernel does, the difference between user space and kernel space, and the ways the CPU enters the kernel.
- [ ] Read any process's state from `/proc`, and trace any program's system calls with `strace`.
- [ ] Make system calls from Rust at every level: `std`, `nix`, `libc` and (once) raw assembly, and handle `errno` correctly.
- [ ] Work with file descriptors directly: open, read, write, `dup2`, pipes, and close-on-exec, without leaks.
- [ ] Start and manage processes with `fork`, `exec` and `waitpid`, and avoid zombies and the dangers of `fork` with threads.
- [ ] Handle signals safely, explain `EINTR`, process groups and Ctrl-C, and shut a program down cleanly.
- [ ] Explain threads and the Linux scheduler, measure context switches, and use nice values and CPU affinity.
- [ ] Use virtual memory, memory mapping, `epoll`, `io_uring`, namespaces and cgroups, and build a shell and a mini container runtime.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 13.1 | [What an operating system does](01-what-an-operating-system-does.md) | 135 min |
| 13.2 | [Processes from the inside](02-processes-from-the-inside.md) | 135 min |
| 13.3 | [System calls](03-system-calls.md) | 120 min |
| 13.4 | [Files and file descriptors](04-files-and-file-descriptors.md) | 120 min |
| 13.5 | [Creating processes: fork, exec and wait](05-fork-exec-and-wait.md) | 135 min |
| 13.6 | [Pipes and redirection from the inside](06-pipes-and-redirection-inside.md) | 135 min |
| 13.7 | [Signals in your programs](07-signals-in-your-programs.md) | 135 min |
| 13.8 | [Project: your own shell](08-project-your-own-shell.md) | 8 h |
| 13.9 | [Threads and scheduling](09-threads-and-scheduling.md) | 135 min |
| 13.10 | [Virtual memory](10-virtual-memory.md) | 2.5 h |
| 13.11 | [Memory-mapped files](11-memory-mapped-files.md) | 135 min |
| 13.12 | [File systems from the inside](12-file-systems-inside.md) | 2.5 h |
| 13.13 | [Non-blocking I/O and epoll](13-non-blocking-io-and-epoll.md) | 135 min |
| 13.14 | [io_uring](14-io-uring.md) | 120 min |
| 13.15 | [Namespaces](15-namespaces.md) | 135 min |
| 13.16 | [Control groups (cgroups)](16-cgroups.md) | 120 min |
| 13.17 | [Project: a mini container runtime](17-project-mini-container-runtime.md) | 8 h |
| 13.18 | [Checkpoint](18-checkpoint.md) | 135 min |

## Projects in this phase

- **Practice package** (`code/p13-os`, Lessons 13.1–13.7 and 13.9): one small program per file in `src/bin/`, from reading `/proc` to pipelines, signal handlers and thread experiments. Later lessons add their own small packages.
- **`rsh`, your own shell** (Lesson 13.8, `code/p13-rsh`): a prompt, built-ins (`cd`, `exit`, `export`), running programs, pipes, `<` and `>` redirection and correct Ctrl-C handling, tested with `assert_cmd`.
- **`minibox`, a mini container runtime** (Lesson 13.17): `minibox run --rootfs <dir> -- /bin/sh` with its own PID, UTS and mount namespaces, a new root file system, and cgroup memory and CPU limits.
- **Checkpoint** (Lesson 13.18): a self-test quiz and practical tasks that show whether you are ready for Phase 14.

## How to study this phase

- **Run everything in the VM, never as root on your Mac.** Keep the code on the Mac, copy it with the `rsync` line from Lesson 13.1, and build in the VM. A program that goes wrong there can only break itself or the VM, never your Mac.
- **Keep `strace` and `/proc` open.** When an example surprises you, trace it (`strace -f`) and look at `/proc/<pid>/status`, `fd/` and `maps`. Seeing the system calls is the fastest way to understand them.
- **Change the examples on purpose.** Remove a `drop(writer)`, forget an `O_CLOEXEC`, leave out a `waitpid` — and watch the hang, the leak or the zombie. These mistakes are the lessons that stick.
- **Do not skip Lessons 13.4 to 13.7 before the shell.** The project uses something from each of them; reread the relevant section when a milestone does not work.
- **Plan about 10 hours a week**, two lessons a week, and a full weekend for each of the two projects.

---

[🏠 Start](../START-HERE.md) · [← 12.15 Checkpoint](../12-computer-systems/15-checkpoint.md) · [13.1 What an operating system does →](01-what-an-operating-system-does.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
