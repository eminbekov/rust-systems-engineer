<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.8 Project: your own shell](08-project-your-own-shell.md) · [📋 Phase 13](README.md) · [13.10 Virtual memory →](10-virtual-memory.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/09-threads-and-scheduling.md) <!-- nav -->

# Lesson 13.9 — Threads and scheduling

> **Phase 13 — Operating systems from the inside** · Lesson 9 of 18 · ⏱ about 135 minutes

## What you will learn

- What a thread is for the kernel, and how it differs from a process: Linux makes both with `clone`, with different flags.
- How to see the threads of a process: TIDs, `/proc/<pid>/task` and `ps -L`.
- What the scheduler does: run queues, time slices, preemption and context switches, and how to count them.
- How Linux shares the CPU fairly (EEVDF, nice values and weights), and how to measure it yourself.
- How to pin a thread to a CPU with `sched_setaffinity`, and what creating a thread costs compared with a process.

## Before you start

- You finished [Lesson 13.8 — Project: your own shell](08-project-your-own-shell.md), or at least Lessons 13.1 to 13.7.
- You used threads with `std::thread::spawn`, `join`, `Arc` and `Mutex` in [Lesson 05.10 — Threads, channels and shared state](../05-rust-intermediate/10-threads-channels-and-shared-state.md). Here you see what the kernel does with them.
- You know the timer interrupt (Lesson 13.1), process states (Lesson 13.2) and `clone` from the `strace` output of Lesson 13.5.

## New words

| Word | Meaning |
|---|---|
| **thread** | One line of execution inside a process: its own registers and stack, but the same memory, files and signal handlers as the other threads of the process. |
| **task** | The Linux kernel's word for anything it schedules. Every thread is a task; a single-threaded process is one task. |
| **TID** (Thread IDentifier) | The kernel's number for one thread. The main thread's TID equals the PID. |
| **TGID** (Thread Group IDentifier) | The kernel's name for what you call the PID: the ID shared by all threads of one process. |
| **LWP** (Light-Weight Process) | The old name for a thread; `ps -L` shows TIDs in a column called `LWP`. |
| **scheduler** | The part of the kernel that decides which task runs on which CPU core, and for how long. |
| **run queue** | The list of runnable tasks (state `R`) waiting for one CPU core. Each core has its own. |
| **time slice** | How long a task may run before the scheduler checks whether someone else should run. |
| **preemption** | The kernel taking the CPU away from a running task, using the timer interrupt, without asking it. |
| **context switch** | Saving the registers of one task and loading those of another, so that the CPU continues the other one. |
| **nice value** | A number from −20 to 19 that a user sets on a process: higher means "nicer to others", so a smaller share of the CPU. |
| **EEVDF** (Earliest Eligible Virtual Deadline First) | The algorithm of Linux's normal scheduler since kernel 6.6; it replaced CFS (Completely Fair Scheduler). |
| **CPU affinity** | The set of CPU cores a thread is allowed to run on. |
| **CPU**, **VM**, **MiB**, **I/O** | Known from earlier lessons: Central Processing Unit; Virtual Machine (your VM `dev`); mebibyte (1,024 × 1,024 bytes); Input/Output. |

## 1. A thread is a task that shares

You know a process as "a running program with its own memory". A thread is a second (third, fourth…) line of execution
**inside the same process**. Think of a kitchen: the process is the kitchen with its fridge and its pots; threads are the
cooks. Every cook has their own hands and their own place in the recipe, but all use the same fridge and pots — which is why
they must agree who takes what (Phase 14).

What each thread has, and what the threads of one process share:

| Own per thread | Shared by all threads of the process |
|---|---|
| registers, including the instruction pointer | memory: code, heap, statics (one address space) |
| stack | the fd table: open files, pipes, sockets |
| TID | PID, parent, user, current directory |
| signal mask (Lesson 13.7) | signal dispositions (handlers) |
| scheduling state (running, sleeping) | resource limits |

For the Linux kernel there is no deep difference between a thread and a process: both are **tasks** with a
`task_struct`, both created by the system call `clone`. The difference is in the flags: "share the memory?", "share the fd
table?", "share the signal handlers?". `fork` (Lesson 13.5) shares none of these; a new thread shares all of them. Here is
what `std::thread::spawn` asks for (`strace -f -e trace=clone,clone3` of the program in Section 2):

```text
clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, child_tid=0xffff8fa0f1f0, parent_tid=0xffff8fa0f1f0, exit_signal=0, stack=0xffff8f800000, stack_size=0x20ea00, tls=0xffff8fa0f840}, 88) = -1 ENOSYS (Function not implemented)
clone(child_stack=0xffff8fa0ea00, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID ...
```

**What happens:**

1. `CLONE_VM` — share the memory (the "virtual memory"). `CLONE_FILES` — share the fd table. `CLONE_FS` — share the current
   directory. `CLONE_SIGHAND` — share the signal handlers.
2. `CLONE_THREAD` — put the new task into the same **thread group**, so it has the same PID and counts as part of this process.
3. `stack=…, stack_size=0x20ea00` — the C library allocated a new stack of about 2 MiB for the thread. `CLONE_SETTLS` gives it
   its own thread-local storage (where per-thread values such as `errno` live).
4. As in Lesson 13.5, `clone3` fails only because the checking container blocks it; the C library falls back to `clone`.
   In your VM `clone3` succeeds.

## 2. Seeing threads: TIDs and `/proc/<pid>/task`

Every thread has its own number, the **TID**. `getpid()` returns the same PID in all threads; `gettid()` returns a different
TID in each. Every thread also gets a folder `/proc/<pid>/task/<tid>/`, with its own `status`, `stat` and `comm` (its name).
`std::thread::Builder::name` sets that name. Create `src/bin/threads.rs`:

```rust
use nix::unistd::{getpid, gettid};
use std::{fs, thread, time::Duration};

fn main() {
    println!("main:     pid {} tid {}", getpid(), gettid());
    let mut handles = Vec::new();
    for i in 1..=3 {
        let handle = thread::Builder::new()
            .name(format!("worker-{i}"))
            .spawn(move || {
                println!("worker-{i}: pid {} tid {}", getpid(), gettid());
                thread::sleep(Duration::from_millis(300)); // stay alive while main looks
            })
            .unwrap();
        handles.push(handle);
    }
    thread::sleep(Duration::from_millis(100));

    // Every thread is a "task" of this process, with its own folder.
    let mut tasks: Vec<String> = fs::read_dir("/proc/self/task")
        .unwrap()
        .map(|e| e.unwrap().file_name().into_string().unwrap())
        .collect();
    tasks.sort();
    for tid in tasks {
        let comm = fs::read_to_string(format!("/proc/self/task/{tid}/comm")).unwrap();
        println!("  /proc/self/task/{tid}: {}", comm.trim());
    }
    for h in handles {
        h.join().unwrap();
    }
}
```

This lesson uses the `sched` feature of `nix` in Section 5; add it now with `cargo add nix@0.32 --features sched`. Then:

```bash
cargo build -q --release
./target/release/threads
```

```text
main:     pid 1174 tid 1174
worker-2: pid 1174 tid 1176
worker-1: pid 1174 tid 1175
worker-3: pid 1174 tid 1177
  /proc/self/task/1174: threads
  /proc/self/task/1175: worker-1
  /proc/self/task/1176: worker-2
  /proc/self/task/1177: worker-3
```

**What happens:**

1. All four lines show PID 1174: one process. The TIDs differ; the main thread's TID equals the PID.
2. `worker-2` printed before `worker-1`: once threads exist, the scheduler decides the order, not your code.
3. `/proc/self/task` has one folder per thread, and `comm` holds the name given with `Builder::name` (the main thread is
   named after the program).

`ps -L` lists threads too. While the program runs in the background:

```bash
./target/release/threads > /dev/null & sleep 0.15; ps -L -o pid,lwp,nlwp,stat,comm -p $!
```

```text
    PID     LWP NLWP STAT COMMAND
      6       6    4 Sl   threads
      6       8    4 Sl   worker-1
      6       9    4 Sl   worker-2
      6      10    4 Sl   worker-3
```

`LWP` is the TID, `NLWP` the number of threads, and the `l` in `Sl` means "multi-threaded". In `top`, press `H` to switch
between processes and threads.

> 💡 **Tip:** Name your threads. When a server uses 100 % CPU, `top -H` or `ps -L` then shows **which** thread does it
> ("db-writer", "http-3") instead of 20 lines with the program's name. Debuggers and panic messages show the names too.

## 3. The scheduler: who runs, and for how long

Your VM has a few CPU cores but runs hundreds of tasks. Most of them sleep (state `S`) and need no CPU. The runnable ones
(state `R`) wait in a **run queue**; each core has its own. The **scheduler** picks the next task from a queue whenever:

- the running task blocks (`read` on an empty pipe, `sleep`, waiting for a lock) — it gives the CPU away **voluntarily**;
- the running task has used up its **time slice** and another task is waiting — the timer interrupt (Lesson 13.1) lets the
  kernel take the CPU away: **preemption**, an **involuntary** switch;
- a task with a better claim wakes up.

Switching from one task to another is a **context switch**: the kernel saves the registers of the old task in its
`task_struct`, loads the registers of the new one, switches the stack, and, if the new task belongs to another process,
switches the memory map too (Lesson 13.10). A switch costs on the order of microseconds, mostly because the CPU caches
(Lesson 12.9) now hold the old task's data.

The kernel counts both kinds of switch for every task in `status`. Create `src/bin/switches.rs`:

```rust
use std::time::{Duration, Instant};
use std::{fs, thread};

/// Reads "voluntary_ctxt_switches" and "nonvoluntary_ctxt_switches" from a status text.
fn ctxt_switches(status: &str) -> Option<(u64, u64)> {
    let field = |key: &str| -> Option<u64> {
        status
            .lines()
            .find_map(|l| l.strip_prefix(key))?
            .trim()
            .parse()
            .ok()
    };
    Some((
        field("voluntary_ctxt_switches:")?,
        field("nonvoluntary_ctxt_switches:")?,
    ))
}

fn now() -> (u64, u64) {
    let status = fs::read_to_string("/proc/thread-self/status").unwrap();
    ctxt_switches(&status).unwrap()
}

fn main() {
    let before = now();
    for _ in 0..100 {
        thread::sleep(Duration::from_millis(1)); // give the CPU away 100 times
    }
    let after_sleep = now();
    let start = Instant::now();
    while start.elapsed() < Duration::from_secs(2) {} // never give the CPU away
    let after_spin = now();
    println!(
        "sleeping 100 times: +{} voluntary, +{} involuntary",
        after_sleep.0 - before.0,
        after_sleep.1 - before.1
    );
    println!(
        "spinning for 2 s:   +{} voluntary, +{} involuntary",
        after_spin.0 - after_sleep.0,
        after_spin.1 - after_sleep.1
    );
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_both_counters() {
        let text = "Name:\tx\nvoluntary_ctxt_switches:\t12\nnonvoluntary_ctxt_switches:\t3\n";
        assert_eq!(ctxt_switches(text), Some((12, 3)));
    }

    #[test]
    fn missing_counter_gives_none() {
        assert_eq!(ctxt_switches("voluntary_ctxt_switches:\t12\n"), None);
    }
}
```

```bash
cargo build -q --release
./target/release/switches
taskset -c 0 ./target/release/switches
cargo test -q --bin switches
```

```text
sleeping 100 times: +100 voluntary, +0 involuntary
spinning for 2 s:   +0 voluntary, +14 involuntary
sleeping 100 times: +100 voluntary, +0 involuntary
spinning for 2 s:   +0 voluntary, +16266 involuntary

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. `/proc/thread-self/status` is the `status` file of the **calling thread** (`/proc/self` is the process).
2. Each `sleep` blocks the thread, so it gives the CPU away: exactly 100 voluntary switches.
3. The spin loop never blocks (`Instant::now()` runs in the vDSO, Lesson 13.3, so there is not even a system call). With 12
   free cores, the scheduler rarely needs to stop it: 14 involuntary switches in 2 seconds.
4. `taskset -c 0` allows the program to run **only on CPU 0**. That core is shared with other busy tasks of the machine, so
   now the scheduler preempts our loop thousands of times to give others their turn. Your numbers depend on how busy your VM
   is, but the pattern is the same.

> ⚡ **Performance:** Many involuntary switches mean "more runnable tasks than cores": the CPU is overloaded, and every task
> waits in the queue. Many voluntary switches mean a task often blocks — typical for programs that do small reads and writes.
> Both counters, and `/proc/loadavg` from Lesson 13.1, are the first numbers to look at when "the machine is slow".

## 4. Fair sharing: EEVDF, nice values and weights

How does Linux decide **who** runs next? Normal tasks use the policy `SCHED_OTHER`, implemented since Linux 6.6 by
**EEVDF**. The idea, simplified:

1. Each task has a **weight**. Over time, each task should get a share of the CPU proportional to its weight.
2. The kernel tracks how much CPU time each task has received compared with its fair share. A task that got less than its
   share is "eligible".
3. Among the eligible tasks, the one whose next small slice of work has the **earliest deadline** runs first. Tasks with short
   slices (interactive programs) get the CPU quickly; tasks that use their whole slice (computations) keep their fair share,
   but no more.

The weight comes from the **nice value**. Nice 0 has weight 1024; each step of nice changes the weight by about 25 %, so nice 10
has weight 110 and nice −10 has weight 9548. A normal user may only make their processes **nicer** (raise the value); lowering
it needs root.

Let us measure it. `src/bin/spin.rs` counts how much work it can do in three seconds:

```rust
use std::time::{Duration, Instant};

fn main() {
    let name = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "spin".to_string());
    let start = Instant::now();
    let mut rounds: u64 = 0;
    while start.elapsed() < Duration::from_secs(3) {
        for _ in 0..10_000 {
            rounds = std::hint::black_box(rounds + 1); // real work the compiler cannot remove
        }
    }
    println!("{name}: {} million rounds in 3 s", rounds / 1_000_000);
}
```

Run it alone, then two copies on the **same** core (with `taskset -c 0`), once with equal nice values and once with nice 0
against nice 10:

```bash
cargo build -q --release
./target/release/spin alone
taskset -c 0 ./target/release/spin A & taskset -c 0 ./target/release/spin B & wait
taskset -c 0 ./target/release/spin nice0 & taskset -c 0 nice -n 10 ./target/release/spin nice10 & wait
```

```text
alone: 11344 million rounds in 3 s
B: 5751 million rounds in 3 s
A: 5696 million rounds in 3 s
nice0: 10377 million rounds in 3 s
nice10: 1152 million rounds in 3 s
```

**What happens:**

1. `std::hint::black_box` tells the compiler "pretend you do not know this value", so it cannot replace the loop with a
   multiplication; `--release` makes the loop fast (Lesson 05.12).
2. Alone, the program gets a whole core. Two equal programs on one core get half each: 5751 and 5696, very fair.
3. With nice 0 against nice 10, the shares follow the weights: 1024 : 110 is about 9.3 : 1; we measured 10377 : 1152 = 9.0 : 1.
4. The total is a bit less than "alone", because of the context switches between them.

Both programs must be started from the same terminal for this experiment: Linux groups the processes of each terminal
session ("autogroup") and first shares the CPU between groups, then inside them.

> 📏 **Convention:** Run long background jobs (backups, big builds, batch processing) with `nice -n 10` or higher. They still
> use every idle CPU cycle, but interactive programs and services get the CPU first when they need it.

There are also **real-time** policies, `SCHED_FIFO` and `SCHED_RR`: a real-time task always runs before every normal task.
They need root (`chrt -f 50 program`), and a real-time task in an endless loop can freeze a core, so they are only for special
programs such as audio or control systems. `chrt -p PID` shows the policy of a process; normal ones say `SCHED_OTHER`.

## 5. CPU affinity: pinning a thread to a core

By default a thread may run on any core, and the scheduler moves it between cores to balance the load. **CPU affinity**
restricts it to a set of cores: `taskset -c 0` did that from outside. From inside, `nix::sched::sched_setaffinity` does it.
Create `src/bin/affinity.rs`:

```rust
use nix::sched::{CpuSet, sched_getaffinity, sched_getcpu, sched_setaffinity};
use nix::unistd::Pid;

fn main() -> nix::Result<()> {
    let allowed = sched_getaffinity(Pid::from_raw(0))?; // 0 means "this thread"
    let cpus: Vec<usize> = (0..CpuSet::count())
        .filter(|&c| allowed.is_set(c).unwrap_or(false))
        .collect();
    println!("allowed CPUs: {cpus:?}, running on CPU {}", sched_getcpu()?);

    let mut only_cpu_2 = CpuSet::new();
    only_cpu_2.set(2)?;
    sched_setaffinity(Pid::from_raw(0), &only_cpu_2)?;
    println!("after pinning: running on CPU {}", sched_getcpu()?);
    println!(
        "available_parallelism now says {}",
        std::thread::available_parallelism().unwrap()
    );
    Ok(())
}
```

```bash
./target/release/affinity
taskset -c 0-3 ./target/release/affinity
```

```text
allowed CPUs: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11], running on CPU 0
after pinning: running on CPU 2
available_parallelism now says 1
allowed CPUs: [0, 1, 2, 3], running on CPU 0
after pinning: running on CPU 2
available_parallelism now says 1
```

**What happens:**

1. `sched_getaffinity` returns a `CpuSet`, a bit set with one bit per core. `Pid::from_raw(0)` means "the calling thread".
2. `taskset -c 0-3` started the program with only cores 0–3 allowed; the affinity is inherited across `fork` and `exec`.
3. After `sched_setaffinity`, the kernel moves the thread to core 2 at once; `sched_getcpu` confirms it.
4. `available_parallelism` respects the affinity: now it says 1. This is why it, and not `/proc/cpuinfo`, is the right
   number for "how many threads should I start?" (Lesson 13.1, Exercise 1).

Pinning helps in special cases: keeping a latency-critical thread away from noisy neighbours, or keeping a thread close to
its data in big multi-socket servers. In most programs, let the scheduler decide; it balances better than a fixed rule.

## Common mistakes

> ⚠️ **Common mistake:** Expecting threads to run in the order they were started. The scheduler decides, and the order
> changes from run to run (`worker-2` before `worker-1` above). If order matters, use `join`, channels or locks (Phase 14).

> ⚠️ **Common mistake:** Starting one thread per task with thousands of tasks. Each thread has its own stack (2 MiB of
> address space by default for `std` threads) and costs context switches. A fixed pool of about `available_parallelism()`
> worker threads, or async tasks (Phase 14), scales much better.

> ⚠️ **Common mistake:** Measuring speed with an unpinned program on a busy machine and trusting one run. Other tasks steal
> CPU time, and the numbers jump. Repeat measurements, and look at the involuntary switches to see whether you were preempted.

## Try it yourself

1. Run `threads` and, while it runs, `ls /proc/$(pgrep -x threads)/task` in a second terminal.
2. Start `./target/release/spin x &` four times more than you have cores (`nproc`), then watch `top`: each gets about
   `100 % × cores / copies` of CPU.
3. Run `nice -n 19 ./target/release/spin low & ./target/release/spin normal & wait` **without** `taskset`. Why are the numbers
   now almost equal? (Hint: each copy finds a free core.)
4. Run `chrt -p $$` in the VM. Which policy does your shell use?

## Exercises

### Exercise 1 — Which thread is busy? (easy)

Write `src/bin/busy-thread.rs` that starts two named threads: `idle` sleeps for 5 seconds, `burner` spins for 5 seconds. While
it runs, find the busy thread with `top -H -p <pid>` or `ps -L -o lwp,pcpu,comm -p <pid>`.

<details>
<summary>Hint</summary>

Use `thread::Builder::new().name("burner".into()).spawn(...)` and, inside, the loop `while start.elapsed() <
Duration::from_secs(5) {}`. `pcpu` is the CPU percentage.

</details>

<details>
<summary>Answer</summary>

```rust
use std::thread;
use std::time::{Duration, Instant};

fn main() {
    println!("pid {}", std::process::id());
    let idle = thread::Builder::new()
        .name("idle".into())
        .spawn(|| thread::sleep(Duration::from_secs(5)))
        .unwrap();
    let burner = thread::Builder::new()
        .name("burner".into())
        .spawn(|| {
            let start = Instant::now();
            while start.elapsed() < Duration::from_secs(5) {}
        })
        .unwrap();
    idle.join().unwrap();
    burner.join().unwrap();
}
```

```bash
./target/release/busy-thread & sleep 2; ps -L -o lwp,pcpu,comm -p $!; wait
```

```text
pid 105
    LWP %CPU COMMAND
    105  0.0 busy-thread
    107  0.0 idle
    108  100 burner
```

Why it works: the kernel accounts CPU time per task, so per-thread numbers come for free. The names make the answer obvious.

</details>

### Exercise 2 — A test for the counters (medium)

The function `ctxt_switches` in `switches.rs` already has two tests. Add a third test that checks a status text in which the
values have extra spaces and the two lines come in the opposite order, and a fourth one for a value that is not a number. Make
all four pass.

<details>
<summary>Hint</summary>

`find_map` searches every line, so the order of the lines does not matter. `trim()` removes spaces and tabs around the number.
For a value like `"abc"`, `parse()` fails, `.ok()` gives `None`, and the `?` makes the whole function return `None`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn order_and_spaces_do_not_matter() {
        let text = "nonvoluntary_ctxt_switches:   7  \nvoluntary_ctxt_switches:\t  40\n";
        assert_eq!(ctxt_switches(text), Some((40, 7)));
    }

    #[test]
    fn a_broken_value_gives_none() {
        let text = "voluntary_ctxt_switches:\tabc\nnonvoluntary_ctxt_switches:\t1\n";
        assert_eq!(ctxt_switches(text), None);
    }
```

```text
running 4 tests
....
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the existing function needed no change — the tests prove that it already handles these cases. Tests like these are
cheap insurance before you change the parser later.

</details>

### Exercise 3 — How much does a thread cost? (hard)

Write `src/bin/spawn-cost.rs` that measures, with `std::time::Instant`, the average time of `thread::spawn(|| {}).join()` and
of `fork` + `waitpid` of a child that exits at once, each 1000 times. Build with `--release`, run it twice, and explain the
difference.

<details>
<summary>Hint</summary>

`Duration` can be divided by a `u32`: `start.elapsed() / 1000`. In the child, call `libc::_exit(0)` (Lesson 13.5). Measure
threads first and join every thread before you fork, so that the program has only one thread at each `fork`.

</details>

<details>
<summary>Answer</summary>

```rust
use nix::sys::wait::waitpid;
use nix::unistd::{ForkResult, fork};
use std::thread;
use std::time::Instant;

const N: u32 = 1000;

fn main() {
    let start = Instant::now();
    for _ in 0..N {
        thread::spawn(|| {}).join().unwrap();
    }
    let per_thread = start.elapsed() / N;

    let start = Instant::now();
    for _ in 0..N {
        // SAFETY: when we fork, the only other thread has already been joined.
        match unsafe { fork() }.unwrap() {
            ForkResult::Child => unsafe { libc::_exit(0) },
            ForkResult::Parent { child } => {
                waitpid(child, None).unwrap();
            }
        }
    }
    let per_process = start.elapsed() / N;

    println!("thread  spawn + join: {per_thread:?} each");
    println!("process fork + wait:  {per_process:?} each");
}
```

Two runs:

```text
thread  spawn + join: 97.789µs each
process fork + wait:  262.615µs each
thread  spawn + join: 47.034µs each
process fork + wait:  149.053µs each
```

Why it works: a thread shares the memory map and the fd table, so the kernel only creates a `task_struct` and the C library a
stack. `fork` must also copy the page tables and the fd table of the parent (Lesson 13.10), and the child's exit must tear them
down. The numbers change between runs (the first run also paid for warming up caches), so always measure more than once.
Both are far too slow to do per request in a fast server — which is why servers reuse threads in a pool.

</details>

## Quiz

1. What do the threads of one process share, and what does each have for itself?

<details>
<summary>Answer</summary>

They share the memory (code, heap, statics), the fd table, the PID, the current directory and the signal handlers. Each has its
own registers, stack, TID and signal mask.

</details>

2. How does Linux create a thread, and how is that different from `fork`?

<details>
<summary>Answer</summary>

With `clone` (or `clone3`) and flags such as `CLONE_VM`, `CLONE_FILES`, `CLONE_SIGHAND` and `CLONE_THREAD`, which make the new task
share memory, files and handlers and join the same thread group. `fork` is `clone` without these sharing flags, so the child gets
copies.

</details>

3. What is the difference between a voluntary and an involuntary context switch?

<details>
<summary>Answer</summary>

Voluntary: the task blocked (sleep, I/O, waiting for a lock) and gave the CPU away itself. Involuntary: the scheduler preempted it,
because its time slice was used up and another task was waiting.

</details>

4. Two CPU-heavy programs share one core; one runs with nice 0, the other with nice 10. Roughly how is the CPU shared?

<details>
<summary>Answer</summary>

By weight: 1024 against 110, so about 9 : 1. The nice-0 program gets about 90 % of the core.

</details>

5. Why is `available_parallelism()` a better choice than the number of lines in `/proc/cpuinfo` for sizing a thread pool?

<details>
<summary>Answer</summary>

Because it respects the CPU affinity (and container CPU limits): if the process may only run on two cores, it says 2, while
`/proc/cpuinfo` still lists every core of the machine.

</details>

6. What does `CLONE_THREAD` change compared with `clone` with only `CLONE_VM`?

<details>
<summary>Answer</summary>

It puts the new task into the caller's thread group: it shares the PID (TGID), it is not a separate process in `ps`, and when one
thread calls `exit_group` (or the process gets a fatal signal) all threads end together.

</details>

## Summary

- A thread is a task that shares memory, files and signal handlers with the other threads of its process; each thread has its own registers, stack, TID and signal mask.
- Linux creates threads and processes with the same `clone` system call; the flags decide what is shared.
- `/proc/<pid>/task/<tid>/`, `ps -L` and `top -H` show threads; name them with `thread::Builder::name`.
- The scheduler keeps runnable tasks in per-core run queues and switches between them on blocking (voluntary) or on preemption by the timer (involuntary).
- Normal tasks are scheduled by EEVDF, fairly by weight; the nice value (−20 to 19) sets the weight.
- CPU affinity (`taskset`, `sched_setaffinity`) restricts a thread to some cores; `available_parallelism` respects it.
- Creating a thread is cheaper than creating a process, but neither is free: reuse threads.

## Sources and further reading

- [clone(2) — Linux manual page](https://man7.org/linux/man-pages/man2/clone.2.html) — every flag that decides what a new task shares.
- [sched(7) — Linux manual page](https://man7.org/linux/man-pages/man7/sched.7.html) — scheduling policies, nice values, real-time scheduling and autogroups.
- [EEVDF Scheduler — The Linux Kernel documentation](https://docs.kernel.org/scheduler/sched-eevdf.html) — the kernel's own description of EEVDF.
- [sched_setaffinity(2) — Linux manual page](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html) — CPU affinity.
- [std::thread — Rust documentation](https://doc.rust-lang.org/std/thread/index.html) — `spawn`, `Builder`, names, stack sizes and `available_parallelism`.
- [Operating Systems: Three Easy Pieces — Scheduling: Proportional Share](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-lottery.pdf) — the idea of sharing a CPU by weights.

---

[🏠 Start](../START-HERE.md) · [← 13.8 Project: your own shell](08-project-your-own-shell.md) · [📋 Phase 13](README.md) · [13.10 Virtual memory →](10-virtual-memory.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
