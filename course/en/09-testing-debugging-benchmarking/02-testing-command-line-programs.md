<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.1 The test pyramid and a testing strategy](01-the-test-pyramid.md) · [📋 Phase 09](README.md) · [09.3 Test doubles: fakes, stubs and mocks →](03-test-doubles.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/02-testing-command-line-programs.md) <!-- nav -->

# Lesson 09.2 — End-to-end tests for command-line programs

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 2 of 14 · ⏱ about 120 minutes

## What you will learn

- How to test a finished command-line program (stdin, arguments, stdout, stderr, exit code) with the `assert_cmd` crate.
- How to describe the expected output with *predicates* (the `predicates` crate) instead of comparing whole strings.
- How to give each test its own temporary files and folders with the `tempfile` crate.
- What an end-to-end test (E2E for short) should check, and what it should leave to unit tests.

## Before you start

- You finished [Lesson 09.1 — The test pyramid and a testing strategy](01-the-test-pyramid.md). There you wrote an end-to-end test with `std::process::Command` and `env!("CARGO_BIN_EXE_…")`. It worked, but you had to collect the output by hand, convert bytes to text and compare everything exactly. This lesson replaces that code with a tool made for the job.
- You know how a program reads standard input and arguments ([Lesson 02.7](../02-programming-basics/07-reading-input-and-parsing.md)) and how it reports errors with an exit code ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).

## New words

| Word | Meaning |
|---|---|
| **assert_cmd** | A crate that runs your program in a test and gives short, readable checks for the exit code, stdout and stderr. |
| **predicates** | A crate of small yes/no checks on values ("does this text contain `red`?") that can be combined with `and`, `or` and `not`. |
| **predicate** | One such check: a value that answers `true` or `false` for a given input. |
| **tempfile** | A crate that creates temporary files and folders and deletes them automatically. |
| **temporary directory** | A folder created just for one test, in the system's temporary location, removed when the test ends. |
| **fixture** | The prepared data a test needs before it runs: files, folders, input text. |
| **builder** | An object configured by calling several methods in a row (`.arg(…).env(…).assert()`), each returning the object itself. |
| **stdin** (standard input) | The channel through which a program reads input that is piped to it or typed by the user. |
| **timeout** | A time limit. A program that does not finish within it is stopped and the test fails, instead of waiting for ever. |
| **dev-dependency** | A crate used only by tests and examples, listed in `[dev-dependencies]` in `Cargo.toml`; it is not part of your released program. |

## 1. Why a crate for this?

The test from Lesson 09.1 looked like this:

```rust
let output = program().args(["ten", "3"]).output().unwrap();
assert_eq!(output.status.code(), Some(2));
let stderr = String::from_utf8(output.stderr).unwrap();
assert_eq!(stderr, "error: 'ten' is not a valid amount\n");
```

It works, but it has problems that grow with every new test: you repeat the conversion from bytes to text; to feed
standard input you must write extra code with threads or pipes; a program that waits for input would make the test
hang for ever; and when `assert_eq!` fails, you see two strings but not the command that produced them.

`assert_cmd` fixes these problems. Its tests read like a description of the behaviour: *run this program with these
arguments and this input; it must succeed; stdout must be this.*

## 2. The program to test: `linecount`

Today's program counts the lines (or, with `--words`, the words) of a file, or of standard input when no file is
given. It also has failure paths: a missing file, and wrong arguments. Create the package:

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/09-02-cli-tests
cargo new code/p09-cli-tests
cd code/p09-cli-tests
cargo add --dev assert_cmd@2 predicates@3 tempfile@3
```

`cargo add --dev` writes the three crates into `[dev-dependencies]`. Open `Cargo.toml` and add a `[[bin]]` section at
the end, so that the program is called `linecount` and not `p09-cli-tests`:

```toml
[package]
name = "p09-cli-tests"
version = "0.1.0"
edition = "2024"

[dependencies]

[dev-dependencies]
assert_cmd = "2"
predicates = "3"
tempfile = "3"

[[bin]]
name = "linecount"
path = "src/main.rs"
```

Replace `src/main.rs`:

```rust
use std::io::{self, Read};
use std::process::ExitCode;

/// Reads the whole input: a file if a path is given, otherwise standard input.
fn read_input(path: Option<&str>) -> io::Result<String> {
    match path {
        Some(path) => std::fs::read_to_string(path),
        None => {
            let mut text = String::new();
            io::stdin().read_to_string(&mut text)?;
            Ok(text)
        }
    }
}

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().skip(1).collect();
    let (count_words, path) = match args.as_slice() {
        [] => (false, None),
        [flag] if flag == "--words" => (true, None),
        [path] => (false, Some(path.as_str())),
        [flag, path] if flag == "--words" => (true, Some(path.as_str())),
        _ => {
            eprintln!("usage: linecount [--words] [FILE]");
            return ExitCode::from(2);
        }
    };

    let text = match read_input(path) {
        Ok(text) => text,
        Err(error) => {
            eprintln!("error: cannot read the input: {error}");
            return ExitCode::from(1);
        }
    };

    if count_words {
        println!("{} words", text.split_whitespace().count());
    } else {
        println!("{} lines", text.lines().count());
    }
    ExitCode::SUCCESS
}
```

**What happens:**

1. `match args.as_slice()` looks at the arguments as a slice and picks the shape: none, `--words`, a path,
   or `--words` and a path. Anything else is a usage error: a message on stderr and exit code 2.
2. `read_input` reads a file when it has a path and standard input otherwise. Both can fail; the failure becomes
   exit code 1 and a message on stderr.
3. `main` returns `ExitCode`. That is the way to end a program with a chosen exit code without calling `process::exit`.
4. `text.lines().count()` counts lines; `text.split_whitespace().count()` counts words.

Try it:

```bash
printf 'roses are red\nviolets are blue\n' | cargo run -q -- --words
cargo run -q -- nothing.txt; echo "exit code: $?"
cargo run -q -- a b c; echo "exit code: $?"
```

```text
6 words
error: cannot read the input: No such file or directory (os error 2)
exit code: 1
usage: linecount [--words] [FILE]
exit code: 2
```

## 3. The first test with `assert_cmd`

Create `tests/cli.rs`:

```rust
use std::time::Duration;

use assert_cmd::Command;
use predicates::prelude::*;

fn linecount() -> Command {
    let mut cmd = Command::cargo_bin("linecount").unwrap();
    cmd.timeout(Duration::from_secs(10));
    cmd
}

#[test]
fn counts_lines_from_standard_input() {
    linecount()
        .write_stdin("one\ntwo\nthree\n")
        .assert()
        .success()
        .stdout("3 lines\n")
        .stderr("");
}
```

```bash
cargo test --test cli
```

```text
running 1 test
test counts_lines_from_standard_input ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.13s
```

**What happens:**

1. `assert_cmd::Command` is a wrapper around the standard `Command` with extra methods. `Command::cargo_bin("linecount")`
   finds the binary that Cargo built for this package (the name is the one in `[[bin]]`) and returns a `Result`,
   because the binary might not exist.
2. `cmd.timeout(…)` sets a time limit of ten seconds. If the program waits for input that never comes, the test fails
   after ten seconds instead of hanging. We put it in the helper `linecount()` so that no test forgets it.
3. `.write_stdin("…")` supplies the standard input. Without it the program would read nothing.
4. `.assert()` runs the program and returns an `Assert` object that holds everything the program did.
   Each following call is a check: `.success()` demands exit code 0; `.stdout("3 lines\n")` demands that stdout is
   *exactly* that text; `.stderr("")` demands that nothing was written to stderr.
5. The checks are a chain of method calls (a *builder*), so the test reads from top to bottom like the sentence in
   the introduction.

> 📏 **Convention:** Put the common setup (the binary name, the timeout) in one helper function such as `linecount()`.
> Every test then starts from the same, safe default.

## 4. Predicates: describing what you expect

An exact comparison is the right choice when the output is short and fixed. It is wrong when part of the output you
do not control: a path, a time, or a message written by the operating system. For these cases `.stdout(…)` and
`.stderr(…)` also accept a **predicate**.

A predicate is a tiny object with one question: "is this value acceptable?" Predicates are independent of
`assert_cmd`, so you can try them in an ordinary test (add this to `tests/cli.rs`):

```rust
#[test]
fn predicates_answer_yes_or_no() {
    let has_red = predicate::str::contains("red");
    assert!(has_red.eval("roses are red"));
    assert!(!has_red.eval("violets are blue"));

    let red_roses = predicate::str::contains("red").and(predicate::str::contains("roses"));
    assert!(red_roses.eval("roses are red"));
    assert!(!red_roses.eval("apples are red"));

    let not_empty = predicate::str::is_empty().not();
    assert!(not_empty.eval("x"));
}
```

**What happens:**

1. `use predicates::prelude::*` brings in `predicate` (a module with the constructors) and the traits that give
   predicates their methods.
2. `predicate::str::contains("red")` builds a predicate; `.eval(text)` asks the question and returns a `bool`.
3. `.and(…)`, `.or(…)` and `.not()` combine predicates into a new one.

The ones you will use most:

| Predicate | True when the text… |
|---|---|
| `predicate::str::contains("x")` | contains `x` somewhere |
| `predicate::str::starts_with("x")` / `ends_with("x")` | begins / ends with `x` |
| `predicate::str::is_empty()` | is empty |
| `predicate::eq("x")` | is exactly `x` (what `.stdout("x")` uses inside) |
| `p1.and(p2)`, `p1.or(p2)`, `p.not()` | satisfies both / at least one / does not satisfy the predicate |

Now use predicates in a failure test. Add:

```rust
#[test]
fn a_missing_file_fails_with_exit_code_1() {
    linecount()
        .arg("no-such-file.txt")
        .assert()
        .failure()
        .code(1)
        .stdout("")
        .stderr(
            predicate::str::starts_with("error: cannot read the input")
                .and(predicate::str::contains("No such file")),
        );
}

#[test]
fn too_many_arguments_print_the_usage_and_exit_with_2() {
    linecount()
        .args(["--words", "a.txt", "b.txt"])
        .assert()
        .code(2)
        .stderr("usage: linecount [--words] [FILE]\n");
}
```

**What happens:**

1. `.failure()` demands a non-zero exit code; `.code(1)` demands exactly 1. Checking the exact code is worth it: scripts
   that call your program decide by the code.
2. For the missing file, the first half of the message is ours (`error: cannot read the input`), but the second half
   comes from the operating system. We check our part strictly (`starts_with`) and the system part loosely
   (`contains("No such file")`), so the test does not depend on the exact wording of a system message.
3. `.args([...])` passes several arguments at once; `.arg(x)` passes one.
4. The usage message is ours entirely, so we compare it exactly.

> 💡 **Tip:** When you check an error message, check the part that *you* wrote and the part that tells the user *what went wrong*.
> Do not check the wording of messages produced by other libraries or the system: they change between versions
> and between operating systems.

## 5. Files and folders: `tempfile`

Many command-line programs read or write files. A test must never touch your real files, and two tests must never
use the same file, because Rust runs tests **in parallel**. The `tempfile` crate gives each test its own empty place:

```rust
#[test]
fn counts_words_in_a_file() {
    let dir = tempfile::tempdir().unwrap();
    let file = dir.path().join("poem.txt");
    std::fs::write(&file, "roses are red\nviolets are blue\n").unwrap();

    linecount()
        .arg("--words")
        .arg(&file)
        .assert()
        .success()
        .stdout("6 words\n");
}

#[test]
fn a_relative_path_is_resolved_in_the_current_directory() {
    let dir = tempfile::tempdir().unwrap();
    std::fs::write(dir.path().join("notes.txt"), "a\nb\n").unwrap();

    linecount()
        .current_dir(dir.path())
        .arg("notes.txt")
        .assert()
        .success()
        .stdout("2 lines\n");
}

#[test]
fn input_can_come_from_a_file() {
    use std::io::Write;
    let mut file = tempfile::NamedTempFile::new().unwrap();
    writeln!(file, "first").unwrap();
    writeln!(file, "second").unwrap();

    linecount()
        .pipe_stdin(file.path())
        .unwrap()
        .assert()
        .success()
        .stdout("2 lines\n");
}

#[test]
fn the_temporary_directory_is_removed_when_dropped() {
    let path;
    {
        let dir = tempfile::tempdir().unwrap();
        path = dir.path().to_path_buf();
        assert!(path.exists());
    }
    assert!(!path.exists());
}
```

**What happens:**

1. `tempfile::tempdir()` creates a new, empty folder with a unique random name in the system's temporary location and
   returns a `TempDir`. `dir.path()` is its path. The test builds its fixture inside it with `std::fs::write`.
2. The folder is **deleted automatically when the `TempDir` value is dropped** (when `dir` goes out of scope at the end of the
   test; [Lesson 03.2](../03-ownership-and-borrowing/02-ownership-rules.md) explained how Rust drops values). The last test proves it: after the inner block ends, the path no longer exists.
   You never write clean-up code, and a failing test cleans up too.
3. `.current_dir(dir.path())` starts the program inside that folder, so the relative path `notes.txt` is
   resolved there.
4. `NamedTempFile::new()` creates one temporary file. `writeln!` writes lines into it. `.pipe_stdin(path)` feeds the
   file's content to the program's standard input; it returns a `Result`, hence the `.unwrap()`.

> 🔒 **Security:** Do not build temporary file names by hand (`/tmp/test.txt`). Other tests or other users on the same
> machine can use the same name, or plant a file there beforehand. `tempfile` creates names that are unique and
> unpredictable and opens the file safely.

## 6. When a test fails

An E2E test is only useful if its failure message helps. Change the expected output to `"4 lines\n"` and run the test:

```text
---- counts_lines_from_standard_input stdout ----
thread 'counts_lines_from_standard_input' (4665549) panicked at tests/cli.rs:18:10:
Unexpected stdout, failed diff original var
├── original: 4 lines
├── diff:
│   ---	orig
│   +++	var
│   @@ -1 +1 @@
│   -4 lines
│   +3 lines
└── var as str: 3 lines
command=`".../target/debug/linecount"`
stdin=(the three input lines, one per line)
code=0
stdout="3 lines\n"
stderr=""
```

**What it means:** you get a diff between the expected and the actual output, and, below it, the **whole picture of
the run**: the command that was started, the input it received, the exit code and both output channels. That is
usually enough to understand the failure without running anything by hand. If you demand `.success()` from a
program that fails, the message is `Unexpected failure.` followed by the same block with `code=1` and the stderr text.
If a predicate fails, the message names it, for example `failed var.contains(permission denied)`.

## 7. What an E2E test should check

Every E2E test costs hundreds of milliseconds, as you measured in Lesson 09.1, so use them for what only a real
process can show:

| Check with an E2E test | Leave to unit and integration tests |
|---|---|
| exit code for success and for each kind of failure | how a number is parsed or formatted |
| the main output format (stdout) | every edge case of the logic |
| the error channel (stderr), the start of messages | the exact wording of every message |
| reading stdin, arguments, files and environment | long lists of input combinations |
| the program does not hang (timeout) | internal data structures |

Two more tools of `assert_cmd` are worth knowing. `.env("NAME", "value")` sets an environment variable for the
program (and `.env_remove("NAME")` removes one), which is how you test a program that reads configuration from
the environment. `.timeout(…)` we used already.

## Common mistakes

**1. The wrong binary name.** `Command::cargo_bin("linecounter")` panics, and the message lists the real names:

```text
thread 'wrong_name_demo' (4674179) panicked at …/assert_cmd-2.2.2/src/cargo.rs:232:5:
`CARGO_BIN_EXE_linecounter` is unset
help: available binary names are "linecount"
```

**What it means:** the name must be the name of the binary (`[[bin]] name`, or the package name when there is no
`[[bin]]`), not the name of the package or the file.

**2. Forgetting the newline.** `println!` ends the line with `\n`. `.stdout("3 lines")` fails; write `.stdout("3 lines\n")`
or use `predicate::str::contains("3 lines")`.

**3. A fixed file name.** Tests that write to `test.txt` in the current folder work when run one by one and fail at random
when run in parallel (a *flaky test*, Lesson 05.6). Use `tempdir()`.

**4. Checking the exact text of an operating-system message.** `No such file or directory (os error 2)` is different
on other systems. Check only the part you own, as in Section 4.

**5. Keeping the `TempDir` alive too short.** `let path = tempfile::tempdir().unwrap().path().to_path_buf();` drops the
`TempDir` at the end of that line, so the folder is deleted at once. Keep the `TempDir` itself in a variable (`let dir = …;`)
for as long as the test needs the folder.

## Try it yourself

1. Create the package, `Cargo.toml` and `src/main.rs` as in Section 2. Run the three commands of Section 2 by hand.
2. Create `tests/cli.rs` with all the tests of Sections 3 to 5. Run `cargo test --test cli`; you should see nine passing tests.
3. Change `.stdout("3 lines\n")` to `.stdout("4 lines\n")`, run the test and read the failure. Change it back.
4. Run the quality gate: `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`.
   (`--all-targets` makes Clippy check the tests too.)
5. Commit: `git add code/p09-cli-tests && git commit -m "Add p09-cli-tests: end-to-end tests with assert_cmd"`.

## Exercises

### Exercise 1 — Any whitespace (easy)

Write a test that checks that `linecount --words` counts `"a  b\tc\n"` (two spaces, a tab) as `3 words`.

<details>
<summary>Hint</summary>

Use `.arg("--words")` and `.write_stdin("a  b\tc\n")`. The expected stdout is `"3 words\n"`.

</details>

<details>
<summary>Answer</summary>

```rust
#[test]
fn words_are_separated_by_any_whitespace() {
    linecount()
        .arg("--words")
        .write_stdin("a  b\tc\n")
        .assert()
        .success()
        .stdout("3 words\n");
}
```

Why it works: `split_whitespace` splits on any run of whitespace characters (spaces, tabs, newlines), so two spaces
and a tab still give three words.

</details>

### Exercise 2 — A directory instead of a file (medium)

What does the program do if the path is a *directory*? Write a test that creates a temporary directory, passes it as
the argument, and checks the exit code and the start of the message.

<details>
<summary>Hint</summary>

`.arg(dir.path())` accepts a `&Path`. Reading a directory as a file is an I/O error, so the program takes its "cannot
read the input" path. Do not check the system part of the message.

</details>

<details>
<summary>Answer</summary>

```rust
#[test]
fn a_directory_is_not_a_file() {
    let dir = tempfile::tempdir().unwrap();
    linecount()
        .arg(dir.path())
        .assert()
        .failure()
        .code(1)
        .stderr(predicate::str::starts_with("error: cannot read the input"));
}
```

Why it works: `read_to_string` on a directory returns an error, so the program prints our message and returns exit
code 1. The test keeps `dir` alive until the end of the function, so the directory exists while the program runs.

</details>

### Exercise 3 — Input that is not text (hard)

The program reads text. What happens when the input is two bytes that are not valid UTF-8 (`0xff, 0xfe`)? (UTF-8, Unicode Transformation Format 8-bit, is the standard way to store text as bytes; a Rust `String` must always hold valid UTF-8.) Write a
test that sends them on standard input and checks the result. Which exit code does the program use, and who decided on it?

<details>
<summary>Hint</summary>

`write_stdin` accepts anything that can become a `Vec<u8>`, so `vec![0xff, 0xfe]` works. The standard library's error
message mentions "valid UTF-8".

</details>

<details>
<summary>Answer</summary>

```rust
#[test]
fn input_that_is_not_utf8_is_rejected() {
    linecount()
        .write_stdin(vec![0xff, 0xfe])
        .assert()
        .failure()
        .code(1)
        .stderr(predicate::str::contains("valid UTF-8"));
}
```

Why it works: `read_to_string` refuses bytes that are not valid UTF-8, so `read_input` returns an error and `main`
exits with code 1. The *program's author* decided that "cannot read the input" is code 1 and "wrong arguments" is
code 2; a test makes that decision visible and keeps it stable.

</details>

## Quiz

1. What does `Command::cargo_bin("linecount")` do?

<details>
<summary>Answer</summary>

It finds the binary that Cargo built for the target named `linecount` in the current package and returns an
`assert_cmd::Command` for it (or an error if there is no such binary).

</details>

2. What is the difference between `.stdout("x\n")` and `.stdout(predicate::str::contains("x"))`?

<details>
<summary>Answer</summary>

The first demands that stdout is exactly `x\n`. The second demands only that `x` appears somewhere. Use the exact form
for short, fixed output and the predicate when part of the output varies.

</details>

3. Why must tests use `tempfile` instead of a fixed file name?

<details>
<summary>Answer</summary>

Tests run in parallel, so two tests with the same file would interfere and fail at random. `tempfile` also removes the
files automatically and avoids predictable names.

</details>

4. When does a `TempDir` delete its folder?

<details>
<summary>Answer</summary>

When the `TempDir` value is dropped, usually at the end of the test function, also when the test fails.

</details>

5. Why does the helper `linecount()` set a timeout?

<details>
<summary>Answer</summary>

A program that waits for input would otherwise make the test run for ever. With a timeout the test fails after the limit
and the problem is visible.

</details>

## Summary

- `assert_cmd` runs your finished program in a test: `Command::cargo_bin("name")`, then `.arg`, `.args`, `.write_stdin`,
  `.env`, `.current_dir`, `.timeout`, then `.assert()` and the checks `.success()`, `.failure()`, `.code(n)`, `.stdout(…)`, `.stderr(…)`.
- `.stdout(…)` and `.stderr(…)` accept an exact string or a predicate from the `predicates` crate; predicates combine with
  `and`, `or` and `not`.
- `tempfile::tempdir()` and `NamedTempFile` give each test unique files that are deleted automatically.
- A failing `assert_cmd` test prints the command, the input, the exit code and both outputs.
- Check exit codes and your own messages strictly, and other people's messages loosely.
- Use E2E tests for what only a real process shows; leave the details of the logic to unit tests.

## Sources and further reading

- [assert_cmd on docs.rs](https://docs.rs/assert_cmd/2/assert_cmd/) — the full API of the crate with examples.
- [predicates on docs.rs](https://docs.rs/predicates/3/predicates/) — all the predicates and how to combine them.
- [tempfile on docs.rs](https://docs.rs/tempfile/3/tempfile/) — `tempdir`, `NamedTempFile` and the guarantees they give.
- [Command Line Applications in Rust — Testing](https://rust-cli.github.io/book/tutorial/testing.html) — the CLI working group's guide to testing a command-line program.
- [The Cargo Book — Specifying dependencies: development dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html) — what `[dev-dependencies]` means.

---

[🏠 Start](../START-HERE.md) · [← 09.1 The test pyramid and a testing strategy](01-the-test-pyramid.md) · [📋 Phase 09](README.md) · [09.3 Test doubles: fakes, stubs and mocks →](03-test-doubles.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
