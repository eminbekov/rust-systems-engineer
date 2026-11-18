<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.4 Users, groups and permissions](04-users-groups-and-permissions.md) · [📋 Phase 11](README.md) · [11.6 Services with systemd →](06-services-with-systemd.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/11-linux-shell-and-containers/05-processes-and-signals.md) <!-- nav -->

# Lesson 11.5 — Processes and signals

> **Phase 11 — Linux, the shell and containers** · Lesson 5 of 16 · ⏱ about 135 minutes

## What you will learn

- What a process is, how processes form a tree, and how to list and inspect them with `ps`, `pgrep` and `/proc`.
- How to run commands in the background and bring them back with `&`, `jobs`, `fg` and `bg`.
- What signals are, how `kill` sends them, and why `SIGTERM` comes before `SIGKILL`.
- What an exit status is and how to read it with `$?`.
- How a Rust program catches `SIGTERM` and `SIGHUP`, and how to install Rust inside the VM to build it.

## Before you start

- You finished [Lesson 11.4 — Users, groups and permissions](04-users-groups-and-permissions.md).
- You know from [Lesson 00.3](../00-getting-started/03-how-a-computer-runs-a-program.md) that a running program is a *process*. Here we look at processes from the outside, as an administrator does.
- Sections 1 to 4 are typed in the VM (`limactl shell dev`). Section 5 starts on your Mac and moves into the VM.

## New words

| Word | Meaning |
|---|---|
| **process** | A running instance of a program, with its own memory, its own user and its own number. |
| **PID** (Process ID) | The number that identifies a process. The kernel gives the next free number to every new process. |
| **PPID** (Parent PID) | The PID of the process that started this one. |
| **process tree** | Every process (except the first) has a parent, so processes form a tree. |
| **PID 1** | The first process that the kernel starts. It is the ancestor of all others. On our VM it is `systemd` (Lesson 11.6). |
| **foreground / background** | A foreground job holds the terminal until it ends. A background job runs while you keep typing. |
| **job** | A command (or pipeline) that your shell started and keeps track of. |
| **signal** | A small message that the kernel or another process sends to a process, such as "please stop". |
| **SIGTERM / SIGKILL** | Signal 15, "please terminate" (the program may react), and signal 9, "terminate now" (the kernel kills it, no reaction possible). |
| **exit status** (exit code) | A number from 0 to 255 that a process leaves when it ends. 0 means success; anything else means failure. |
| **zombie** | A process that has ended but whose parent has not yet read its exit status. It uses no memory, only a slot in the process table. |
| **orphan** | A process whose parent has ended. PID 1 adopts it. |
| **daemon** | A process that runs in the background for a long time, usually a service (Lesson 11.6). |

## 1. Processes: list and inspect

When you start `ls`, the shell asks the kernel for a new process, the program runs, and the process ends. A long-running program is the same thing, only longer. Start one
process in the background and look at it:

```bash
sleep 300 &
ps -o pid,ppid,user,stat,etime,cmd -p $!
```

```text
    PID    PPID USER     STAT     ELAPSED CMD
   4643    4640 student  S          00:00 sleep 300
```

(Your PIDs will be different.) **What happens:**

1. `sleep 300` waits 300 seconds and does nothing. The `&` at the end starts it in the background (Section 2) and the shell goes on.
2. `$!` is a shell variable that holds the PID of the last background job. `ps -p` selects that one process.
3. `-o` chooses the columns: `PID`, `PPID` (the parent: your shell), `USER`, `STAT` (state), `ELAPSED` (time since start) and `CMD` (the command line).
4. `STAT` `S` means *sleeping*: the process waits for something (here for the timer) and uses no CPU (processor) time. Other letters: `R` running, `T` stopped, `Z` zombie, `D` waiting for the disk. A `+` after the letter marks a foreground process.

Normally you list many processes at once. Two classic forms (the output is long, so we filter to our `sleep`):

```bash
ps aux | grep '[s]leep 300'
```

```text
student     4643  0.0  0.0  17096  6344 ?        S    07:00   0:00 sleep 300
```

**What happens:** `ps aux` lists all processes of all users (`a`), with the user column (`u`), including those without a terminal (`x`). The columns are `USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND`: `%CPU` and `%MEM` are the shares of the processor and of memory, `VSZ` is the virtual memory size and `RSS` the real memory in kilobytes, `TTY` the terminal (`?` means none), `TIME` the CPU time used.
The pattern `'[s]leep 300'` finds the process but not the `grep` itself, whose own command line contains `sleep 300`: the brackets make the pattern not match its own text.

More ways to look:

| Command | What it shows |
|---|---|
| `ps -ef` | all processes in the "full format" including PPID |
| `ps -o pid,ppid,stat,cmd --forest` | processes as an ASCII tree (drawn with plain ASCII, American Standard Code for Information Interchange, characters; parents above children) |
| `pstree -p` | the tree drawn compactly with PIDs |
| `top` | a live table, sorted by CPU; press `q` to quit (`top -b -n 1` prints one snapshot) |
| `pgrep -a sleep` | PIDs (and command lines, with `-a`) of processes whose name contains `sleep` |
| `ps aux --sort=-%mem \| head -n 5` | the five processes that use the most memory |

**Everything about a process is in `/proc/<PID>/`** ([Lesson 11.2](02-the-linux-file-system.md)):

```bash
P=$(pgrep -n -x sleep)
grep -E '^(Name|State|Pid|PPid|Uid)' /proc/$P/status
tr '\0' ' ' < /proc/$P/cmdline; echo
ls -l /proc/$P/cwd | cut -d' ' -f9-
```

```text
Name:	sleep
State:	S (sleeping)
Pid:	4643
PPid:	4640
Uid:	1000	1000	1000	1000
sleep 300 
/proc/4643/cwd -> /home/student
```

**What happens:** `pgrep -n -x sleep` returns the newest process named exactly `sleep`. `status` has the name, state, the PIDs and the user IDs; `cmdline` has the arguments separated by zero bytes (which `tr` turns into spaces); `cwd` is a symlink to the process's current directory. `ps` and `top` just read these files and format them.

**Parents and children.** The shell started `sleep`, so the shell is its parent. If the parent ends first, the child becomes an **orphan** and PID 1 adopts it. If a child ends and the parent does not collect its exit status, the child stays as a **zombie** (`Z` in `STAT`, `<defunct>` in `CMD`). A zombie costs almost nothing, but a program that creates many of them has a bug: it never calls `wait`.

## 2. Foreground and background jobs

A normal command is a **foreground** job: the shell waits for it and the terminal belongs to it. Keyboard shortcuts talk to the foreground job:

- `Ctrl + C` sends the signal **SIGINT** ("interrupt"). Most programs stop.
- `Ctrl + Z` sends **SIGTSTP** ("suspend"). The job is paused and you get the prompt back.
- `Ctrl + D` is not a signal: it means *end of input* (stdin closed). A shell exits on it.

A real session looks like this (the `$ ` is the prompt, `^Z` and `^C` show what the shortcut echoes):

```text
$ sleep 300
^Z
[1]+  Stopped                    sleep 300
$ jobs
[1]+  Stopped                    sleep 300
$ bg
[1]+ sleep 300 &
$ jobs
[1]+  Running                    sleep 300 &
$ fg
sleep 300
^C
$ jobs
```

**What happens:**

1. `Ctrl + Z` pauses `sleep`. The shell prints `[1]+  Stopped`: job number 1, state Stopped.
2. `jobs` lists the jobs of **this** shell. `+` marks the current job, `-` the previous one.
3. `bg` ("background") resumes the job in the background. Now it runs while you work.
4. `fg` ("foreground") brings it back to hold the terminal. `Ctrl + C` ended it. The final `jobs` prints nothing: no jobs left.

You can start a job in the background right away with `&`: `sleep 300 &` prints `[1] 4643` (job number and PID). The shell names jobs `%1`, `%2`, ...: `kill %1` stops job 1, `fg %2` brings job 2 forward. `wait` waits for all background jobs to end.

> ⚠️ **Common mistake:** a background job is still tied to your terminal. When you close the terminal, the shell sends `SIGHUP` ("hang up") to its jobs, and most of them stop. To keep a command alive after you leave, use `nohup command &` (it ignores SIGHUP and writes output to `nohup.out`), or better a **service** (Lesson 11.6), or a terminal multiplexer such as `tmux`. Do not rely on `&` alone for something that must survive.

## 3. Signals

A **signal** is the kernel's way to tap a process on the shoulder. Each has a number and a name. List them:

```bash
kill -l | head -n 4
```

```text
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
```

The ones you need every day:

| Signal | No. | Meaning | Can the program catch it? |
|---|---|---|---|
| `SIGHUP` | 1 | terminal closed; by convention, many services use it for "reload your configuration" | yes |
| `SIGINT` | 2 | interrupt (`Ctrl + C`) | yes |
| `SIGKILL` | 9 | kill at once | **no** |
| `SIGSEGV` | 11 | the program touched memory it must not (a crash) | yes (rarely useful) |
| `SIGTERM` | 15 | polite request to terminate; the default of `kill` | yes |
| `SIGSTOP` | 19 | pause the process | **no** |
| `SIGCONT` | 18 | continue a paused process | yes |

"Catch" means: the program installs a **handler**, a function that runs when the signal arrives, instead of the default action (for most signals the default is to terminate). A program can clean up in its handler: finish a write, close connections, delete a temporary file.
`SIGKILL` and `SIGSTOP` cannot be caught, ignored or blocked: the kernel handles them itself.

The command `kill` sends a signal by PID or job (the name is historical: it does not always kill):

```bash
sleep 300 &
kill $!
wait $!
echo "status: $?"
sleep 300 &
kill -9 $!
wait $!
echo "status: $?"
```

```text
status: 143
status: 137
```

**What happens:** `kill PID` sends `SIGTERM` (15). `kill -9 PID` (the same as `kill -KILL PID`) sends `SIGKILL`. `wait $!` waits for that job and returns its exit status, which we print (Section 4 explains 143 and 137). The shell may also print a line like `Killed  sleep 300` for the second job.

To pause and resume a process without ending it:

```bash
sleep 300 &
kill -STOP $!
ps -o pid,stat,cmd -p $!
kill -CONT $!
ps -o pid,stat,cmd -p $!
kill $!
```

```text
    PID STAT CMD
   4713 T    sleep 300
    PID STAT CMD
   4713 S    sleep 300
```

**What happens:** `STAT` `T` means stopped; after `SIGCONT` it is `S` again, and it continues where it paused.

By name instead of PID:

```bash
sleep 301 & sleep 302 &
pkill -f 'sleep 30[12]'
pgrep -c -f 'sleep 30[12]'
```

```text
0
```

**What happens:** `pkill` sends `SIGTERM` to every process that matches (`-f` matches the whole command line, not only the name). `pgrep -c` counts the matches: 0 are left. `pkill` is powerful; run `pgrep -a` with the same pattern first to see what it would hit.

> 🔒 **Security:** You can only signal your own processes; to signal another user's process you need `sudo`. The order of good manners: first `SIGTERM` (wait a few seconds), then `SIGKILL`.
> `kill -9` gives the program no chance to clean up: files can stay half-written, locks remain, children can be orphaned. Use it as the last resort, not as a habit.

## 4. Exit status

When a process ends, it leaves a number, the **exit status**, for its parent. The shell stores the last one in `$?`:

```bash
true; echo "true -> $?"
false; echo "false -> $?"
ls /nope 2>/dev/null; echo "ls /nope -> $?"
bash -c 'exit 3'; echo "exit 3 -> $?"
nosuchcommand 2>/dev/null; echo "nosuchcommand -> $?"
```

```text
true -> 0
false -> 1
ls /nope -> 2
exit 3 -> 3
nosuchcommand -> 127
```

**What happens:** the convention is **0 = success, anything else = failure**. The commands `true` and `false` do nothing except return 0 and 1. `ls` returns 2 for a missing file. `127` is the shell's code for "command not found". Each program documents its own codes (see the manual: `man grep` says 0 = match found, 1 = none, 2 = error).

When a process is ended by a signal, the shell reports **128 + signal number**:

| Status | Meaning |
|---|---|
| 130 | ended by SIGINT (128 + 2): you pressed `Ctrl + C` |
| 137 | ended by SIGKILL (128 + 9) |
| 143 | ended by SIGTERM (128 + 15) |

In Rust, a program ends with status 0 when `main` returns normally (or returns `Ok`), 1 when `main` returns an `Err`, 101 on a panic, and any number you choose with `std::process::exit(code)`; Lesson 11.3 used `exit(2)` for a usage error. Tools like `systemd` and scripts decide what to do next from these numbers.

## 5. A Rust program that handles signals

Rust programs get signals like any other process. By default `SIGTERM` simply kills the program. A service that holds data should instead **shut down cleanly**. The crate `signal-hook` (version 0.4) delivers signals in a safe way. We write a small program, `p11-heartbeat`, that you will also run as a service in the next lesson.

Create the project on your **Mac** (open a Mac terminal tab):

```bash
cd ~/projects/rust
cargo new code/p11-heartbeat
cd code/p11-heartbeat
cargo add signal-hook@0.4
```

Replace `src/main.rs`:

```rust
use std::error::Error;
use std::thread;
use std::time::Duration;

use signal_hook::consts::{SIGHUP, SIGINT, SIGTERM};
use signal_hook::iterator::Signals;

/// What the program does when a signal arrives.
#[derive(Debug, PartialEq)]
enum Action {
    Reload,
    Stop,
}

/// Maps a signal number to an action. Other signals are ignored.
fn action_for(signal: i32) -> Option<Action> {
    match signal {
        SIGHUP => Some(Action::Reload),
        SIGTERM | SIGINT => Some(Action::Stop),
        _ => None,
    }
}

fn main() -> Result<(), Box<dyn Error>> {
    let name = std::env::var("HEARTBEAT_NAME").unwrap_or_else(|_| "world".to_string());
    let seconds: u64 = std::env::var("HEARTBEAT_SECONDS")
        .ok()
        .and_then(|text| text.parse().ok())
        .unwrap_or(2);

    println!(
        "starting: pid {}, greeting {name}, every {seconds} s",
        std::process::id()
    );

    // The worker thread prints one line per interval, forever.
    thread::spawn(move || {
        let mut count = 0;
        loop {
            count += 1;
            println!("heartbeat {count} from {name}");
            thread::sleep(Duration::from_secs(seconds));
        }
    });

    // The main thread waits for signals.
    let mut signals = Signals::new([SIGTERM, SIGINT, SIGHUP])?;
    for signal in signals.forever() {
        match action_for(signal) {
            Some(Action::Reload) => println!("got SIGHUP: pretend to reload the configuration"),
            Some(Action::Stop) => {
                println!("got signal {signal}: shutting down");
                break;
            }
            None => {}
        }
    }
    println!("bye");
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use signal_hook::consts::SIGUSR1;

    #[test]
    fn maps_signals_to_actions() {
        assert_eq!(action_for(SIGHUP), Some(Action::Reload));
        assert_eq!(action_for(SIGTERM), Some(Action::Stop));
        assert_eq!(action_for(SIGINT), Some(Action::Stop));
        assert_eq!(action_for(SIGUSR1), None);
    }
}
```

**What happens:**

1. `Signals::new([...])` asks the kernel to deliver these three signals to the program as events instead of using the default action. `signals.forever()` waits (blocks) and yields each signal number as it arrives.
2. The worker **thread** (Lesson 06.11 and Phase 14 teach threads) prints a "heartbeat" every `seconds` seconds. The main thread only waits for signals.
3. `action_for` is a pure function: a number goes in, an action comes out. It has no side effects, so the test can check it without sending real signals. This is the habit from [Phase 08](../08-software-design/README.md): keep the decision separate from the I/O.
4. `SIGHUP` only prints (a real service would re-read its configuration). `SIGTERM` and `SIGINT` print and `break` out of the loop, so `main` returns `Ok(())` and the exit status is 0.
5. The settings come from **environment variables** (`HEARTBEAT_NAME`, `HEARTBEAT_SECONDS`) with defaults. A service can then be configured without changing code ([Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md)).
6. `println!` is line-buffered, so every line appears at once; also when the output goes to a file or to the journal.

🧪 **Testing:** run `cargo test` on the Mac. Expect `test tests::maps_signals_to_actions ... ok` and `test result: ok. 1 passed`.

Now build and run it **in the VM**. The VM does not have Rust yet, and the Mac files are read-only there, so we do three steps.

**Step 1 — tools for building.** Rust needs a *linker*, which comes with the C compiler package `build-essential`. (`apt` is the package manager; Lesson 11.7 explains it.)

```bash
sudo apt-get update
sudo apt-get install -y build-essential curl
```

**Step 2 — install Rust in the VM.** Use the same installer as in [Lesson 00.5](../00-getting-started/05-install-rust-and-rustrover.md). It installs into `~/.cargo`, of the VM user:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --profile minimal
. "$HOME/.cargo/env"
cargo --version
```

Example output of the last command (yours will show the current version):

```text
cargo 1.100.0 (3acc79c0c 2026-11-05)
```

(`-s -- -y` passes `-y` to the installer to accept the defaults; `--profile minimal` skips the documentation. `. "$HOME/.cargo/env"` adds `~/.cargo/bin` to the `PATH` of this shell; new shells do it themselves.)

**Step 3 — copy the project and build.** Replace `student` by the name that `ls /Users` shows:

```bash
mkdir -p ~/code/p11-heartbeat
cd /Users/student/projects/rust/code/p11-heartbeat
cp -r Cargo.toml Cargo.lock src ~/code/p11-heartbeat/
cd ~/code/p11-heartbeat
cargo build --release
```

We copy only `Cargo.toml`, `Cargo.lock` and `src`, not the folder `target/` that your Mac build created: it contains macOS programs that Linux cannot run. Example output (yours will look similar; the first build downloads the crates and takes a few seconds to a minute):

```text
   Compiling libc v0.2.191
   Compiling signal-hook v0.4.6
   Compiling errno v0.3.15
   Compiling signal-hook-registry v1.4.8
   Compiling p11-heartbeat v0.1.0 (/home/student/code/p11-heartbeat)
    Finished `release` profile [optimized] target(s) in 3.64s
```

The program is `target/release/p11-heartbeat`. Run it in the background, send signals, and watch the exit status (the program prints every two seconds):

```bash
./target/release/p11-heartbeat &
sleep 4.5
kill -HUP $!
sleep 0.5
kill -TERM $!
wait $!
echo "exit status: $?"
```

```text
starting: pid 4750, greeting world, every 2 s
heartbeat 1 from world
heartbeat 2 from world
heartbeat 3 from world
got SIGHUP: pretend to reload the configuration
got signal 15: shutting down
bye
exit status: 0
```

**What happens:** `SIGHUP` did not stop the program, it only printed. `SIGTERM` (signal 15) ended the loop, `main` printed `bye` and returned: **exit status 0, a clean shutdown**. Try `SIGKILL` and compare:

```bash
./target/release/p11-heartbeat &
sleep 1
kill -KILL $!
wait $!
echo "exit status: $?"
```

```text
starting: pid 4754, greeting world, every 2 s
heartbeat 1 from world
exit status: 137
```

**What happens:** no `got signal`, no `bye`: the program never ran any code after the kill. The status is 137 = 128 + 9. Also see the environment variables at work:

```bash
HEARTBEAT_NAME=Alice HEARTBEAT_SECONDS=1 ./target/release/p11-heartbeat > hb.log &
sleep 2.5
kill $!
wait $!
cat hb.log
rm hb.log
```

```text
starting: pid 4763, greeting Alice, every 1 s
heartbeat 1 from Alice
heartbeat 2 from Alice
heartbeat 3 from Alice
got signal 15: shutting down
bye
```

**What happens:** the settings come from the two variables written before the command (they exist only for that one process). `> hb.log` sends stdout to a file, as in [Lesson 11.3](03-text-tools-pipes-and-redirection.md). We keep the binary for Lesson 11.6.

## Common mistakes

1. **Starting with `kill -9`.** The program cannot clean up. *Fix:* `kill PID` first, wait, then `kill -9` only if it does not exit.
2. **`pkill` with a short pattern.** `pkill ssh` can stop more than you meant. *Fix:* check with `pgrep -a pattern` first, use `-x` for the exact name.
3. **Counting on `&` to survive logout.** The job gets `SIGHUP` when the terminal closes. *Fix:* `nohup`, `tmux`, or a service (Lesson 11.6).
4. **Reading `$?` too late.** `echo $?` shows the status of the *previous* command; any command in between replaces it. *Fix:* save it at once: `status=$?`.
5. **Ignoring the exit status in your own programs.** A Rust program that fails but exits 0 hides the failure from scripts and from systemd. *Fix:* return an `Err` from `main` or call `std::process::exit` with a non-zero code.
6. **Forgetting `. "$HOME/.cargo/env"`** after installing Rust: `cargo: command not found`. *Fix:* run it, or open a new shell with `exit` and `limactl shell dev`.

## Try it yourself

1. Start `sleep 500 &` twice. List them with `jobs` and with `pgrep -a sleep`. Stop job 1 with `kill %1` and check `jobs`.
2. Run `sleep 500`, press `Ctrl + Z`, then `bg`, then `fg`, then `Ctrl + C`. Print `$?`. Expected: `130`.
3. In `/proc/$$` (your shell's PID is in `$$`) find the file `status` and print its `Name` and `PPid` lines. Who is your shell's parent? Check it with `ps -o pid,cmd -p <that PPid>`.
4. Run the heartbeat program in the background. Send `SIGHUP`, then `SIGTERM`, and print the exit status. Repeat with `SIGKILL`.
5. Clean up: `kill` any `sleep` you left (`pkill -x sleep`), and check `pgrep -a sleep`.

## Exercises

### Exercise 1 — Find and stop (easy)

Start three `sleep 600 &` jobs. With one command, stop them all by name, then check that none is left.

<details>
<summary>Hint</summary>

`pkill` takes a name. `-x` means an exact name match. `pgrep -c` counts the matches.

</details>

<details>
<summary>Answer</summary>

```bash
sleep 600 & sleep 600 & sleep 600 &
pkill -x sleep
pgrep -c -x sleep
```

```text
0
```

Why it works: `pkill -x sleep` sends SIGTERM to every process whose name is exactly `sleep`; the default action of SIGTERM is to terminate. `pgrep -c` prints 0 matches.

</details>

### Exercise 2 — Decode the status (medium)

A script ended and `echo $?` printed `143`. Another printed `130`, another `127`. What happened to each? Check your answer with real commands for 143 and 127.

<details>
<summary>Hint</summary>

Numbers above 128 mean "128 + signal number". `kill -l 15` prints the name of signal 15.

</details>

<details>
<summary>Answer</summary>

- `143` = 128 + 15: the process was ended by `SIGTERM`.
- `130` = 128 + 2: ended by `SIGINT` (`Ctrl + C`).
- `127`: the shell could not find the command.

```bash
sleep 300 &
kill $!
wait $!; echo $?
nosuchcommand; echo $?
```

```text
143
bash: nosuchcommand: command not found
127
```

Why it works: the shell converts "killed by signal N" into the status `128 + N`, and uses 127 for "command not found" (126 for "found but not executable").

</details>

### Exercise 3 — Add SIGUSR1 and a test (hard)

Teach `p11-heartbeat` a new signal: on `SIGUSR1` print `status: running`. Write the **test first**: extend the test with `action_for(SIGUSR1)` returning a new `Action::Status`, watch it fail, then make it pass.

<details>
<summary>Hint</summary>

Add the variant `Status` to the enum, add `SIGUSR1` to `Signals::new([...])` and to the `match` in `action_for`. Import `SIGUSR1` from `signal_hook::consts` in the main code; the test module sees it through `use super::*;`.

</details>

<details>
<summary>Answer</summary>

First change the test: replace the last assertion by `assert_eq!(action_for(SIGUSR1), Some(Action::Status));` (the program does not compile yet: red). Then make it pass:

```rust
use signal_hook::consts::{SIGHUP, SIGINT, SIGTERM, SIGUSR1};

#[derive(Debug, PartialEq)]
enum Action {
    Reload,
    Status,
    Stop,
}

fn action_for(signal: i32) -> Option<Action> {
    match signal {
        SIGHUP => Some(Action::Reload),
        SIGUSR1 => Some(Action::Status),
        SIGTERM | SIGINT => Some(Action::Stop),
        _ => None,
    }
}
```

In `main`, register the signal with `Signals::new([SIGTERM, SIGINT, SIGHUP, SIGUSR1])?` and add the arm `Some(Action::Status) => println!("status: running"),` to the `match`. The test module now looks like this (`use super::*;` already brings `SIGUSR1` in; `SIGUSR2` is a signal that stays unknown):

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use signal_hook::consts::SIGUSR2;

    #[test]
    fn maps_signals_to_actions() {
        assert_eq!(action_for(SIGHUP), Some(Action::Reload));
        assert_eq!(action_for(SIGTERM), Some(Action::Stop));
        assert_eq!(action_for(SIGINT), Some(Action::Stop));
        assert_eq!(action_for(SIGUSR1), Some(Action::Status));
        assert_eq!(action_for(SIGUSR2), None);
    }
}
```

`cargo test` reports `1 passed`. Check by hand in the VM: `kill -USR1 $!` prints `status: running` and the program keeps running.

Why it works: the pure function `action_for` is easy to test, and the new behaviour is added in exactly one place per layer: the enum, the mapping, the registration and the handler.

</details>

## Quiz

1. What is the difference between a program and a process?

<details>
<summary>Answer</summary>

A program is a file on disk. A process is a running instance of it, with its own PID, memory and user. One program can have many processes.

</details>

2. What does `STAT` `T` mean in `ps`?

<details>
<summary>Answer</summary>

The process is stopped (paused), for example after `Ctrl + Z` or `SIGSTOP`.

</details>

3. Which two signals can a process never catch?

<details>
<summary>Answer</summary>

`SIGKILL` (9) and `SIGSTOP` (19).

</details>

4. Why try `SIGTERM` before `SIGKILL`?

<details>
<summary>Answer</summary>

`SIGTERM` lets the program clean up (finish writes, close connections). `SIGKILL` ends it at once, with no chance to clean up.

</details>

5. A command exited with status 137. What does it mean?

<details>
<summary>Answer</summary>

128 + 9: it was killed by `SIGKILL`.

</details>

6. What does `$!` hold, and what does `$?` hold?

<details>
<summary>Answer</summary>

`$!` is the PID of the last background job. `$?` is the exit status of the last command.

</details>

## Summary

- A process is a running program with a PID, a parent (PPID), a user and a state. Processes form a tree with PID 1 at the root.
- `ps`, `pgrep`, `top` and `/proc/<PID>` show what runs. `&`, `jobs`, `fg`, `bg` manage jobs of your shell.
- Signals are messages to processes. `kill` sends them (`SIGTERM` by default). `SIGKILL` and `SIGSTOP` cannot be caught.
- Be polite: `SIGTERM` first, `SIGKILL` last. `pkill` matches by name; check with `pgrep -a` first.
- The exit status is 0 for success; `128 + N` means "ended by signal N" (130, 137, 143); 127 means "command not found".
- A Rust service catches `SIGTERM` (for example with `signal-hook`), finishes its work and returns from `main` with status 0.
- Keep the decision (`action_for`) separate from the I/O so you can test it.

## Sources and further reading

- [signal(7) — overview of signals](https://man7.org/linux/man-pages/man7/signal.7.html) — the list of all signals and their default actions.
- [ps(1) manual](https://man7.org/linux/man-pages/man1/ps.1.html) — every column and option of `ps`.
- [GNU Bash Reference Manual — Job Control](https://www.gnu.org/software/bash/manual/html_node/Job-Control.html) — `jobs`, `fg`, `bg`, `wait`.
- [signal-hook on docs.rs](https://docs.rs/signal-hook/0.4.6/signal_hook/) — the crate used in Section 5.
- [Rust standard library — std::process](https://doc.rust-lang.org/std/process/index.html) — `exit`, `Command` and exit statuses in Rust's standard library.
- [rustup — installation](https://rustup.rs/) — the official Rust installer used inside the VM.

---

[🏠 Start](../START-HERE.md) · [← 11.4 Users, groups and permissions](04-users-groups-and-permissions.md) · [📋 Phase 11](README.md) · [11.6 Services with systemd →](06-services-with-systemd.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
