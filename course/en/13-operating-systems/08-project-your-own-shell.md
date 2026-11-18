<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.7 Signals in your programs](07-signals-in-your-programs.md) · [📋 Phase 13](README.md) · [13.9 Threads and scheduling →](09-threads-and-scheduling.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/13-operating-systems/08-project-your-own-shell.md) <!-- nav -->

# Lesson 13.8 — Project: your own shell

> **Phase 13 — Operating systems from the inside** · Lesson 8 of 18 · ⏱ about 8 hours (4 sittings)

## What you will learn

- How to put `fork`, `exec`, `waitpid`, `pipe`, `dup2` and signal settings together into one real program.
- How to split a command-line program into a tested parser and a thin part that talks to the kernel.
- How to test a program that reads standard input, with `assert_cmd` integration tests.
- How a shell decides what runs inside itself (built-ins) and what runs in a child process.

## Before you start

- You finished Lessons 13.1 to 13.7, especially [Lesson 13.5 — fork, exec and wait](05-fork-exec-and-wait.md), [Lesson 13.6 — Pipes and redirection from the inside](06-pipes-and-redirection-inside.md) and [Lesson 13.7 — Signals in your programs](07-signals-in-your-programs.md). Keep them open: this project uses something from each.
- You can write integration tests in `tests/` ([Lesson 05.6 — Testing in depth](../05-rust-intermediate/06-testing-in-depth.md)) and you built a command-line tool with a library and a thin `main` in [Lesson 05.13 — Project: minigrep](../05-rust-intermediate/13-project-minigrep.md).
- You work on the Mac in `~/projects/rust/code/p13-rsh` and build and run in the VM `dev`, as in Lesson 13.1 (`rsync` the folder to `~/p13-rsh`).

## New words

| Word | Meaning |
|---|---|
| **REPL** (Read–Eval–Print Loop) | A program that repeatedly reads a line, runs it, shows the result and asks again. Every interactive shell is a REPL. |
| **prompt** | The short text a shell prints when it waits for your command, here `rsh> `. |
| **token** | One meaningful piece of the input line: a word such as `ls`, or an operator such as `\|` or `>`. |
| **tokenizer** (lexer) | The function that cuts a line into tokens, handling spaces and quotes. |
| **parser** | The function that turns a list of tokens into a structure the program can run: commands, arguments, redirections. |
| **built-in** | A command the shell runs inside its own process, such as `cd`. It must be a built-in because it changes the shell itself. |
| **integration test** | A test that runs your finished binary from the outside (Lesson 05.6). |
| **`assert_cmd`** | A crate for integration tests of command-line programs: run the binary, feed standard input, check output and exit code. |
| **VM**, **OS** | Known from earlier lessons: Virtual Machine (your VM `dev`); Operating System. |

## The project

### What you will build

`rsh` — a small but real Unix shell. It reads command lines, starts programs with `fork` and `exec`, connects them with
pipes, redirects input and output to files, has the built-ins `cd`, `exit` and `export`, and survives Ctrl-C while
the program it runs does not. A session in a terminal looks like this:

```text
rsh> echo hello
hello
rsh> sleep 30
^C
rsh> cat /etc/passwd | cut -d: -f1 | sort | head -3 > /tmp/users.txt
rsh> cat < /tmp/users.txt
_apt
backup
bin
rsh> exit
```

(This output and all outputs below come from a reference solution checked in a Linux container. Your user names may
differ.) You will not get a complete solution: the contract, the milestones, small hints and the acceptance tests ("Done
when") are your guide.

```text
 one line  ──>  tokenize  ──>  parse  ──>  built-in?  ──yes──>  run in the shell (cd, exit, export)
                (parse.rs)    (parse.rs)       │
                                               no
                                               v
                               for each command: pipe2, fork; in the child: reset signals,
                               dup2 pipes and files onto 0/1, execvp  (exec.rs)
                                               │
                                               v
                               parent: close pipe ends, waitpid all, remember the last status
```

### Contract

**Package and files.** A binary package `code/p13-rsh` named `rsh`:

```text
code/p13-rsh/
├── Cargo.toml
├── README.md          ← what rsh does, how to build, run and test it
├── src/
│   ├── main.rs        ← the REPL loop: prompt, read a line, call parse and exec
│   ├── parse.rs       ← tokenize() and parse(), with unit tests
│   ├── builtins.rs    ← cd, exit, export
│   └── exec.rs        ← fork, exec, pipes, redirections, waiting
└── tests/
    └── cli.rs         ← assert_cmd integration tests
```

Dependencies (crates.io versions checked on 2026-11-17):

```toml
[dependencies]
nix = { version = "0.32", features = ["fs", "process", "signal"] }

[dev-dependencies]
assert_cmd = "2.2"
predicates = "3.1"
```

**Input.** `rsh` takes no arguments. It reads lines from standard input until end of file. It prints the prompt `rsh> ` to
**standard error**, and only when standard input is a terminal: then tests that feed input through a pipe see clean
output.

**Syntax of one line.**

| Rule | Example | Meaning |
|---|---|---|
| words are separated by spaces or tabs | `ls   -l` | `["ls", "-l"]` |
| `'...'` and `"..."` keep spaces and operators inside one word; the quotes are removed | `echo 'a  b' "c\|d"` | `["echo", "a  b", "c\|d"]` |
| quotes can be part of a word | `x"y z"` | `"xy z"` |
| `\|`, `<`, `>` are operators, with or without spaces around them | `ls\|wc` | two commands |
| `< file` / `> file` | `sort < in > out` | stdin from `in`, stdout to `out` (create or truncate, mode 644) |
| a pipeline is commands joined by `\|` | `a \| b \| c` | stdout of each goes to stdin of the next |
| an empty line or a line with only spaces does nothing | | |

No variables (`$X`), no `;`, `&&`, `&`, `>>` or `2>` — those are stretch goals.

**Built-ins** (run inside the shell, only when the command is alone on its line):

| Command | Behaviour | Status |
|---|---|---|
| `cd DIR` | change the shell's current directory | 0, or 1 with `rsh: cd: DIR: No such file or directory` (the OS error text) |
| `cd` | go to `$HOME` | 0 |
| `exit [N]` | end the shell with status N (default: the status of the last command) | — |
| `export NAME=VALUE` | set an environment variable that later programs inherit | 0 |

**Exit statuses and errors.** After each line, the shell remembers a status: the exit code of the last command of the
pipeline, or 128 + the signal number if it was killed. At end of file, `rsh` exits with the last status.

| Situation | Message on standard error | Status |
|---|---|---|
| program not found | `rsh: NAME: command not found` | 127 |
| redirection file cannot be opened | `rsh: FILE: No such file or directory` (the OS error text) | 1 |
| unclosed quote | `rsh: unclosed quote` | 2 |
| `\|` at the start or end, or `>`/`<` without a file | `rsh: syntax error ...` (any text after `syntax error`) | 2 |

**Signals.** Ctrl-C while a program runs ends that program (status 130) and **not** the shell; the shell then prints a new
line and a new prompt. Ctrl-C at the prompt does not end the shell. Programs started by `rsh` get default signal settings:
`SIGINT` and `SIGPIPE` are reset to default in every child before `exec`.

**Quality.** `cargo test` passes (unit tests in `parse.rs`, integration tests in `tests/cli.rs`), `cargo clippy` gives no
warnings, `cargo fmt --check` is clean, and there is one Git commit per milestone.

### Step-by-step plan

Create the package on the Mac and copy it into the VM to build (Lesson 13.1):

```bash
cd ~/projects/rust
cargo new code/p13-rsh --name rsh
cd code/p13-rsh
cargo add nix@0.32 --features fs,process,signal
cargo add --dev assert_cmd@2.2 predicates@3.1
```

#### M1 — The loop: run one program

**Goal.** A REPL that reads lines, splits them on whitespace, runs the program with `fork` + `execvp`, waits, and prints
`command not found` for unknown programs.

**Guidance.**

1. In `main`, decide once whether to print a prompt: `std::io::stdin().is_terminal()` (the trait `std::io::IsTerminal`).
   Print the prompt with `eprint!` and flush standard error.
2. Read with `read_line`. A result of `Ok(0)` means end of file: leave the loop and `std::process::exit(last_status)`.
3. For now `line.split_whitespace()` is your parser. Skip empty lines.
4. Write `run(argv: &[String]) -> i32` in `exec.rs`: build the `CString`s **before** `fork`, `execvp` in the child,
   `waitpid` in the parent. Turn the `WaitStatus` into a number (Lesson 13.5, Exercise 2).
5. If `execvp` fails with `ENOENT`, the child prints `rsh: NAME: command not found` and exits with 127.

<details>
<summary>Hint</summary>

`rsh` has only one thread, so after `fork` the child may use `eprintln!` and `std::process::exit` (Lesson 13.5). Keep the
`// SAFETY:` comment that says so. `execvp` returns `Err(e)` only on failure; compare it with `nix::errno::Errno::ENOENT`.

</details>

Add the first integration test in `tests/cli.rs` now; it is the model for all later ones:

```rust
use assert_cmd::cargo::cargo_bin_cmd;

#[test]
fn runs_a_program() {
    cargo_bin_cmd!("rsh")
        .write_stdin("echo hello world\n")
        .assert()
        .success()
        .stdout("hello world\n");
}
```

`cargo_bin_cmd!("rsh")` builds a command for your compiled binary, `write_stdin` feeds it input through a pipe (so no
prompt is printed), and `assert()` checks the result. `.code(127)` and `.stderr("...")` check the exit code and standard
error.

**Done when:**

```bash
cargo build -q
printf 'echo hello world\nls /no-such-dir\nno-such-cmd\n' | ./target/debug/rsh; echo "status $?"
cargo test -q --test cli
```

```text
hello world
ls: cannot access '/no-such-dir': No such file or directory
rsh: no-such-cmd: command not found
status 127
```

plus `test result: ok` for your tests. Also add a test `unknown_command` that checks code 127 and the exact message.
Commit: `git commit -m "feat(p13-rsh): M1 run one program"`.

#### M2 — Tokenizer and parser, test first

**Goal.** Replace `split_whitespace` with `tokenize` and `parse` in `parse.rs`, fully covered by unit tests, before they
are connected to anything that forks.

**Guidance.** Suggested types (you may choose others):

```rust
#[derive(Debug, PartialEq)]
pub enum Token {
    Word(String),
    Pipe,
    Less,
    Greater,
}

#[derive(Debug, PartialEq, Default)]
pub struct Command {
    pub argv: Vec<String>,
    pub stdin: Option<String>,
    pub stdout: Option<String>,
}

pub fn tokenize(line: &str) -> Result<Vec<Token>, String> {
    todo!()
}

pub fn parse(line: &str) -> Result<Vec<Command>, String> {
    todo!()
}
```

`parse` returns the pipeline: one `Command` per part between `|`. Write the tests first (Lesson 05.6), one per row of the
syntax table in the contract, plus the error cases: `"echo 'oops"`, `"ls |"`, `"| ls"`, `"ls >"`. Then the empty line:
`parse("")` returns an empty `Vec`.

<details>
<summary>Hint</summary>

Walk over `line.chars()` with a `String` for the current word and a `bool` "am I inside a word?". A space ends a word; an
operator ends a word **and** pushes its own token; a quote character starts a small inner loop that copies characters until
the same quote comes back (or the line ends: error). Remember that `""` is a word too (an empty one), which is why you need the
`bool` and cannot just test `word.is_empty()`.

</details>

**Done when:** `cargo test -q parse` shows all your parser tests passing, and the parser is used by the REPL:

```bash
printf '%s\n' "echo 'a   b' \"c|d\" x\"y z\"" "echo 'oops" | ./target/debug/rsh; echo "status $?"
```

```text
a   b c|d xy z
rsh: unclosed quote
status 2
```

Commit: `feat(p13-rsh): M2 tokenizer and parser`.

#### M3 — Built-ins: `cd`, `exit`, `export`

**Goal.** Commands that must change the shell itself run inside the shell.

**Guidance.** Before forking, check if the line is a single command whose `argv[0]` is a built-in. `cd` uses
`std::env::set_current_dir`; `cd` alone uses the `HOME` variable. `export NAME=VALUE` uses `std::env::set_var`, which is
`unsafe` in edition 2024 because another thread could read the environment at the same moment; `rsh` has one thread, so
explain that in a `// SAFETY:` comment. Children inherit the environment and the current directory automatically through
`fork` and `exec`.

<details>
<summary>Hint</summary>

Why can `cd` not be a normal program? A child process changing its own directory has no effect on its parent: the child
exits, the shell stays where it was. For the error message, `io::Error`'s `Display` ends with ` (os error 2)`; cut it off, or
build the text from `e.kind()`.

</details>

**Done when:**

```bash
./target/debug/rsh <<'EOF'
cd /tmp
pwd
cd /nope
export GREETING=hi
printenv GREETING
exit 3
echo never printed
EOF
echo "status $?"
```

```text
/tmp
rsh: cd: /nope: No such file or directory
hi
status 3
```

Add an integration test for this script (check `.code(3)` and `.stdout("/tmp\nhi\n")`). Commit:
`feat(p13-rsh): M3 built-ins`.

#### M4 — Redirection: `<` and `>`

**Goal.** `cmd < in > out` works.

**Guidance.** In the child, after `fork` and before `execvp`: if `stdin` is set, `open` the file read-only and
`dup2_stdin`; if `stdout` is set, `open` with `O_WRONLY | O_CREAT | O_TRUNC` and mode `0o644`, then `dup2_stdout`
(Lesson 13.6, Section 4). If `open` fails, print `rsh: FILE: <error text>` and exit the child with 1.

<details>
<summary>Hint</summary>

Open the files **in the child**, not in the shell: then a failed `open` affects only that command, and the shell's own
descriptors never change. Open them with `O_CLOEXEC`: the copies made by `dup2` on 0 and 1 stay open, the originals close at
`exec`.

</details>

**Done when:**

```bash
./target/debug/rsh <<'EOF'
echo saved > /tmp/rsh-out.txt
cat < /tmp/rsh-out.txt
wc -c < /tmp/rsh-out.txt > /tmp/rsh-count.txt
cat /tmp/rsh-count.txt
cat < /nope
EOF
echo "status $?"
```

```text
saved
6
rsh: /nope: No such file or directory
status 1
```

Add an integration test that writes into a file in `std::env::temp_dir()` and reads it back. Commit:
`feat(p13-rsh): M4 redirection`.

#### M5 — Pipelines

**Goal.** `a | b | c` of any length; the status is the status of the last command.

**Guidance.** This is Lesson 13.6 (Section 3 and Exercise 3), for N commands. Create a pipe for every `|`, start
**all** children, and only then wait for all of them. A command can have both a pipe and a redirection (`sort < in | uniq >
out`): apply the pipe first, then the file, so the file wins, as in bash.

<details>
<summary>Hint</summary>

Keep `previous_read_end: Option<OwnedFd>`. For command `i`: if it is not the last, create `pipe2(O_CLOEXEC)`; spawn the child
with `previous_read_end` as stdin and the new write end as stdout; then, in the parent, **drop** both of these (they now live
in the child) and keep the new read end for the next command. If a pipeline hangs, some process still holds a write end:
check `ls -l /proc/<pid>/fd` of the stuck process.

</details>

**Done when:**

```bash
./target/debug/rsh <<'EOF'
cat /etc/passwd | cut -d: -f1 | sort | head -3
yes | head -2
printf 'b\na\n' | sort > /tmp/rsh-pipe.txt
cat /tmp/rsh-pipe.txt
EOF
echo "status $?"
printf 'ls |\n' | ./target/debug/rsh; echo "status $?"
```

```text
_apt
backup
bin
y
y
a
b
status 0
rsh: syntax error near `|`
status 2
```

`yes | head -2` must end at once and print **nothing** on standard error. If you see `yes: standard output: Broken pipe`, the
child inherited the ignored `SIGPIPE` from the Rust runtime (Lesson 13.7): that is M6. Add an integration test with a
three-command pipeline. Commit: `feat(p13-rsh): M5 pipelines`.

#### M6 — Ctrl-C: the child stops, the shell stays

**Goal.** The behaviour from the Signals part of the contract.

**Guidance.** At the start of `main`, set `SIGINT` to ignore with `nix::sys::signal::signal(Signal::SIGINT,
SigHandler::SigIgn)`. In every child, before `execvp`, set `SIGINT` and `SIGPIPE` back to `SigHandler::SigDfl`. All
processes stay in one process group, so the terminal sends Ctrl-C to the shell and to the running program; the shell ignores
it, the program dies with status 130. If the last status is 130 and the shell is interactive, print an empty line to standard
error before the next prompt.

<details>
<summary>Hint</summary>

Remember the table "Signal settings across `fork` and `exec`" from Lesson 13.7: an ignored signal stays ignored after
`exec`, so without the reset in the child, Ctrl-C would not stop `sleep 30` either. You cannot press Ctrl-C in an automated
test, but a program can send `SIGINT` itself: `kill -INT $PPID` from a child sends it to the shell, and `kill -INT $$` makes
a `sh` kill itself.

</details>

**Done when** (the automated part):

```bash
./target/debug/rsh <<'EOF'
sh -c 'kill -INT $PPID'
echo still here
sh -c 'kill -INT $$'
EOF
echo "status $?"
```

```text
still here
status 130
```

and by hand in the VM terminal: start `./target/debug/rsh`, run `sleep 30`, press Ctrl-C. You see `^C`, a new line and a new
prompt, and `echo alive` still works. Put the script above into an integration test (`.code(130)` and
`.stdout(predicates::str::contains("still here"))`). Commit: `feat(p13-rsh): M6 signals`.

#### M7 — Finish: tests, lint, README

**Goal.** A project you can show.

**Guidance.** Integration tests for every row of the contract's error table; `cargo clippy --all-targets` without warnings;
`cargo fmt`; a `README.md` with what `rsh` supports, what it does not, how to build, run and test it (in the VM, because it
uses Linux system calls). Check for zombies: while `rsh` runs, `ps -o pid,stat,comm --ppid <pid of rsh>` must not show `Z`
after commands finish.

<details>
<summary>Hint</summary>

Integration tests start many `rsh` processes in parallel threads; each test has its own process, so they do not disturb each
other — but use different file names in `temp_dir()` for each test that writes files.

</details>

**Done when:**

```bash
cargo fmt --check && cargo clippy -q --all-targets && cargo test -q
```

prints only test results, all `ok`, for example (the reference solution has 2 unit tests and 7 integration tests; your
numbers will differ):

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s


running 7 tests
.......
test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Commit: `feat(p13-rsh): M7 tests, lint and README`.

### Stretch goals

- `>>` (append) and `2>` / `2>&1` for standard error.
- `$NAME` and `$?` expansion outside single quotes.
- `;` and `&&` between commands.
- Real job control: put each pipeline into its own process group with `setpgid`, give it the terminal with `tcsetpgrp`,
  take the terminal back afterwards (Lesson 13.7, Section 5); then `&`, `jobs` and `fg`.
- Line editing and history with the `rustyline` crate (version 19, checked 2026-11-17).

## Common mistakes

> ⚠️ **Common mistake:** The pipeline hangs after the first command finished. A write end is still open somewhere: in the
> shell itself, or in a later child that inherited it. Create pipes with `O_CLOEXEC` and drop the parent's copies right after
> each `fork` (Lesson 13.6).

> ⚠️ **Common mistake:** Running `cd` in a child. It "works" without an error, but the shell's directory never changes.

> ⚠️ **Common mistake:** Printing the prompt to standard output. Then `printf 'echo hi\n' | rsh | wc -l` counts the prompt
> too, and every integration test has to strip it. The contract puts it on standard error and only for terminals.

> ⚠️ **Common mistake:** Forgetting to wait for every child of a pipeline. The shell then fills up with zombies, and the
> status of the line may come from the wrong command.

## Quiz

1. Why must `cd`, `exit` and `export` be built-ins?

<details>
<summary>Answer</summary>

They change the shell process itself: its current directory, its life, its environment. A child process can only change
its own copy, which disappears when it exits.

</details>

2. In which process do you call `dup2` and `open` for `cmd < in > out`, and why?

<details>
<summary>Answer</summary>

In the child, between `fork` and `exec`. Then only the new program's descriptors change, and a failing `open` affects only
that command; the shell's own standard input and output stay untouched.

</details>

3. Why does `rsh` reset `SIGINT` and `SIGPIPE` to default in the child?

<details>
<summary>Answer</summary>

Because ignored signals stay ignored across `exec`. The shell ignores `SIGINT` (to survive Ctrl-C) and the Rust runtime ignores
`SIGPIPE`; without the reset, the programs it starts would also ignore Ctrl-C and broken pipes.

</details>

4. Why does the prompt go to standard error and only when standard input is a terminal?

<details>
<summary>Answer</summary>

So that the shell's standard output contains only the output of the commands. Scripts and tests that pipe input into `rsh` get
clean output; a person at a terminal still sees the prompt.

</details>

5. Why are the tokenizer and parser in their own module with unit tests?

<details>
<summary>Answer</summary>

They are pure functions from text to data, so they can be tested quickly and thoroughly without starting any process. The part
that forks and execs stays small and is covered by a few integration tests.

</details>

## Summary

- A shell is a REPL: read a line, tokenize, parse, run built-ins in the shell and everything else in children, wait, repeat.
- Built-ins exist because only the shell's own process can change its directory, environment or life.
- Redirections and pipes are prepared in the child between `fork` and `exec` with `open`, `pipe2` and `dup2`; the parent closes its pipe ends and waits for every child.
- The shell ignores `SIGINT`; children reset `SIGINT` and `SIGPIPE` to default before `exec`.
- A pure, unit-tested parser plus `assert_cmd` integration tests make a systems program safe to change.

## Sources and further reading

- [POSIX Shell Command Language](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/V3_chap02.html) — the official description (from POSIX, the Portable Operating System Interface standard) of how a shell parses and runs commands.
- [assert_cmd — docs.rs](https://docs.rs/assert_cmd/2.2.2/assert_cmd/) — testing command-line programs.
- [nix::unistd — docs.rs](https://docs.rs/nix/0.32.0/nix/unistd/index.html) — `fork`, `execvp`, `pipe2`, `dup2`.
- [GNU Bash Reference Manual — Shell Operation](https://www.gnu.org/software/bash/manual/html_node/Shell-Operation.html) — the steps a real shell goes through for each line.
- [setpgid(2) — Linux manual page](https://man7.org/linux/man-pages/man2/setpgid.2.html) — process groups, for the job-control stretch goal.

---

[🏠 Start](../START-HERE.md) · [← 13.7 Signals in your programs](07-signals-in-your-programs.md) · [📋 Phase 13](README.md) · [13.9 Threads and scheduling →](09-threads-and-scheduling.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
