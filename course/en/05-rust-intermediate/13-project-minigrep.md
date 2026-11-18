<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.12 Benchmarking basics](12-benchmarking-basics.md) · [📋 Phase 05](README.md) · [05.14 Checkpoint →](14-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/13-project-minigrep.md) <!-- nav -->

# Lesson 05.13 — Project: minigrep

> **Phase 05 — Rust intermediate** · Lesson 13 of 14 · ⏱ about 4 hours (2 sittings)

## What you will learn

- How to plan and build a complete command-line tool in small steps, with a Git commit after each step.
- How to split a program into a library (the logic) and a thin binary (the terminal), and test both.
- How to use lifetimes, iterators, `clap`, `thiserror`, files, standard input and exit codes in one real program.
- How to write unit tests, integration tests and doc tests for a tool.
- How to measure a program with `criterion` and `hyperfine`, find a real slowness and fix it.

This is a **project lesson**. It does not give you the finished program. It gives you the target (the contract), a plan with small milestones, hints, and commands that prove each milestone is done. The code is yours to write.

## Before you start

- You finished [Lesson 05.12 — Benchmarking basics](12-benchmarking-basics.md) and, before it, every lesson of this phase. You will use generics and lifetimes ([05.1](01-generics.md), [05.3](03-lifetimes.md)), closures and iterators ([05.4](04-closures.md), [05.5](05-iterators.md)), tests and docs ([05.6](06-testing-in-depth.md), [05.7](07-documentation.md)), dependencies ([05.8](08-cargo-dependencies-and-versions.md)), and the crates `clap` and `thiserror` ([05.11](11-practical-crates-serde-clap-errors.md)).
- Git works on your computer ([Lesson 01.2](../01-git-and-version-control/02-install-and-configure-git.md)).
- Your project goes to `~/projects/rust/code/p05-minigrep`, inside your learning repository. The file `.gitignore` at the root of the repository must contain a rule for `target/` (you set it up in [Lesson 05.1](01-generics.md)). Run `git status` in `~/projects/rust` first: it should say `nothing to commit, working tree clean`.
- You are online: Cargo must download crates.

## New words

| Word | Meaning |
|---|---|
| **grep** (Global Regular Expression Print) | A classic Unix program that prints the lines of a file that match a pattern. `minigrep` is a small version of it. |
| **pattern** | The text you search for. In `minigrep` it is plain text, not a regular expression. |
| **standard input** (stdin) | The stream of data that a program reads by default: the keyboard, or the output of another program. |
| **standard output** (stdout) | The stream where a program writes its normal results. |
| **standard error** (stderr) | A second output stream for error messages, kept apart from the results. |
| **pipe** (`\|`) | A shell feature that connects the stdout of one program to the stdin of the next: `cat file \| minigrep word`. |
| **redirect** (`>`) | A shell feature that sends stdout to a file: `minigrep word file > result.txt`. |
| **exit status** | A number returned when a program ends; `0` means success. Also called exit code. |
| **environment variable** | A named setting that the shell gives to a program, such as `HOME` or `MINIGREP_IGNORE_CASE`. |
| **buffer** | A piece of memory that collects data before it is sent on, so that fewer slow operations are needed. |
| **system call** (syscall) | A request from a program to the operating system, for example "write these bytes". Each one has a cost. |
| **acceptance test** | A test that checks that the program does what the contract promises, seen from the outside. Here, the "Done when" commands. |
| **library crate / binary crate** | A crate with `src/lib.rs` (code for others to use) / a crate with `src/main.rs` (a program you run). A package can have both. |
| **TDD** (Test-Driven Development) | A way of working: write a failing test first, then the code that makes it pass, then clean up. |
| **README** | A text file at the top of a project that tells people what it is and how to use it. |
| **tag** (Git) | A fixed, named label on one commit, often used for a release such as `v0.1.0`. |
| **I/O** (Input/Output) | Reading from and writing to the outside world: files, keyboard, screen, network. |
| **CLI** (Command-Line Interface) | A program that you use by typing a command in a terminal. |

## Warm-up: meet the real grep

Before you build a tool, use the real one. Create a folder with two small files (use your own words; the examples in this lesson use exactly these texts, so that your output matches):

```bash
mkdir -p ~/projects/rust/code/p05-minigrep-data && cd ~/projects/rust/code/p05-minigrep-data
cat > poem.txt <<'EOF'
Rust is a language
that cares about memory.
Every value has an owner,
and the owner decides its fate.
RUST is loved by many,
but the compiler is strict.
EOF
cat > notes.txt <<'EOF'
todo: read the chapter about rust
todo: write the tests
done: install the toolchain
EOF
```

Now try the system's `grep` (on your Mac, `grep` may be a modern replacement, but the options below work the same):

```bash
grep Rust poem.txt        # lines that contain "Rust"
grep -i rust poem.txt     # ignore upper/lower case
grep -n owner poem.txt    # show line numbers
grep -c the poem.txt      # only count the lines
grep -v the poem.txt      # lines that do NOT contain "the"
echo $?                   # the exit status of the last command
```

Notice two things: when `grep` finds nothing, it prints nothing but its exit status is `1`; and `grep` reads standard input if you give no file: `cat poem.txt | grep -n memory`. Your tool will copy these behaviors.

## The project

### What you will build

`minigrep` is a command-line tool that prints the lines of files (or of standard input) that contain a given text. It supports four options (`-i`, `-v`, `-n`, `-c`), several files, an environment variable, and proper exit codes. It is built as a **library** (`src/lib.rs`) with all the logic and a **thin binary** (`src/main.rs`) that talks to the terminal.

You will work in this order: set up and parse arguments, write the search function with tests first, add file reading and output, add options, test the whole program from the outside, write documentation, then measure and speed it up.

```text
p05-minigrep/
├── Cargo.toml
├── README.md
├── benches/
│   └── search.rs        ← criterion benchmark (milestone M7)
├── src/
│   ├── lib.rs           ← the logic: search, run, errors
│   └── main.rs          ← the terminal: clap, exit codes
└── tests/
    └── cli.rs           ← integration tests (milestone M5)
```

### Contract

This is the exact behavior your program must have. All "Done when" checks come from it.

**Usage**

```text
minigrep [OPTIONS] <PATTERN> [FILES]...
```

| Part | Meaning |
|---|---|
| `<PATTERN>` | Required. The text to look for. A plain text, not a regular expression. A line is *selected* when it contains the pattern. An empty pattern selects every line. |
| `[FILES]...` | Zero or more files. With no file, read standard input. |
| `-i`, `--ignore-case` | Upper and lower case letters are equal. |
| `-v`, `--invert-match` | Select the lines that do **not** contain the pattern. |
| `-n`, `--line-number` | Print the line number (counting from 1) and a colon before each line. |
| `-c`, `--count` | Do not print lines; print only how many lines were selected, for each input. |
| `-h`, `--help`, `-V`, `--version` | Help and version (made by `clap`). |

**Environment.** If the variable `MINIGREP_IGNORE_CASE` is set (to any value), it works like `-i`.

**Output (stdout)**

| Situation | Format of a selected line |
|---|---|
| one input | `text` |
| one input, `-n` | `12:text` |
| more than one file | `file:text` |
| more than one file, `-n` | `file:12:text` |
| `-c`, one input | `3` |
| `-c`, more than one file | `file:3` (one line per file) |

Lines are printed in the order they appear, file after file, in the order given on the command line. The line break is not part of the text of a line.

**Errors (stderr)**

- A file that cannot be read gives one line on stderr: `minigrep: <file>: <reason>` (for example `minigrep: missing.txt: No such file or directory (os error 2)`). The other files are still searched.
- A wrong command line (no pattern, an unknown option) gives the usual `clap` message on stderr.
- Nothing but results goes to stdout.

**Exit status**

| Status | When |
|---|---|
| `0` | At least one line was selected, and every input could be read. |
| `1` | No line was selected, and every input could be read. |
| `2` | Any error: a file could not be read, or the command line was wrong. |

**Code rules**

- The logic lives in `src/lib.rs`. `src/main.rs` only parses arguments, calls the library and turns the result into an exit status.
- The function that does the matching returns **borrowed** lines (`&str` from the input text), not new `String`s. It needs a lifetime annotation: `fn search<'a>(pattern: &str, contents: &'a str, ...) -> Vec<...<'a>>`.
- Errors are a `thiserror` enum. No `unwrap()` or `expect()` outside tests.
- Dependencies: `clap` and `thiserror`; dev-dependencies `tempfile` and `criterion`.
- `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test` must pass with no warnings.

### Step-by-step plan

Work through the milestones in order. After each milestone, run its "Done when" commands, then make a Git commit. Do not look at the hints before you tried.

> 📏 **Convention:** Commit messages follow the style of this phase: a type, the scope in brackets, a colon and a short sentence in the imperative mood: `feat(minigrep): add the search function`. Types you need: `feat`, `test`, `docs`, `perf`, `chore`. One commit per logical step. Use `git add .` inside the project folder (it stages only that folder) and look at `git status --short` before you commit. Never commit the `target` folder.

---

#### M1 — Set up the project and parse the command line

**Goal:** a project that builds, is under Git, and understands the command line of the contract.

**Guidance:**

1. Make a branch for the project and create the package: `cd ~/projects/rust && git switch -c project/05-minigrep && cargo new code/p05-minigrep`. Because you are inside your learning repository, Cargo does not create a new Git repository. In `Cargo.toml`, change the package `name` to `minigrep` (the binary is then called `minigrep`).
2. Add the dependencies: `clap` with the `derive` feature, `thiserror`, and (as dev-dependencies, with `--dev`) `tempfile` and `criterion`. See [Lesson 05.8](08-cargo-dependencies-and-versions.md) and [Lesson 05.11](11-practical-crates-serde-clap-errors.md).
3. Create `src/lib.rs` (it can be empty or contain only a comment for now).
4. In `src/main.rs`, define a struct for the command line with `#[derive(Parser)]` and the arguments of the contract. Give every argument a doc comment. For this milestone, `main` only parses the arguments and prints them with `{:?}` (add `Debug` to the derive).
5. Commit (see the commit message below).

<details>
<summary>Hint 1 — the fields</summary>

`pattern: String`, `files: Vec<PathBuf>`, and four `bool` fields marked `#[arg(short, long)]`. A `Vec<PathBuf>` field is "zero or more" arguments. The flag `-v` for invert needs `#[arg(short = 'v', long)]` if the field is called `invert_match`, because `short` alone takes the first letter of the field name (`i`, which is already taken by `ignore_case`). Look at the short options that you want, and name the fields so they do not clash.

</details>

<details>
<summary>Hint 2 — the long names</summary>

`long` turns the field name into the option with dashes: `ignore_case` becomes `--ignore-case`, `line_number` becomes `--line-number`, `invert_match` becomes `--invert-match`. The contract lists them.

</details>

**Done when:**

```bash
cargo run -q -- --help
```

prints a usage line that contains `minigrep [OPTIONS] <PATTERN> [FILES]...` and lists the four options and `-h`, `-V`. (The description texts are yours.) For example:

```text
Print the lines of FILEs (or standard input) that contain PATTERN

Usage: minigrep [OPTIONS] <PATTERN> [FILES]...

Arguments:
  <PATTERN>   The text to look for
  [FILES]...  Files to search; with no file, read standard input

Options:
  -i, --ignore-case   Ignore the difference between upper and lower case
  -v, --invert-match  Select the lines that do not contain the pattern
  -n, --line-number   Print the line number before each line
  -c, --count         Print only the number of selected lines for each input
  -h, --help          Print help
  -V, --version       Print version
```

and:

```bash
cargo run -q -- ; echo "exit: $?"
```

```text
error: the following required arguments were not provided:
  <PATTERN>

Usage: minigrep <PATTERN> [FILES]...

For more information, try '--help'.
exit: 2
```

Commit: `git add . && git commit -m "feat(minigrep): parse the command line with clap"`.

---

#### M2 — The search function, test first

**Goal:** a tested function in `src/lib.rs` that selects lines. No files, no terminal yet.

**Guidance:** This is TDD ([Lesson 05.6](06-testing-in-depth.md)): red, green, refactor.

1. In `lib.rs`, define `pub struct Options { pub ignore_case: bool, pub invert: bool }` (derive `Debug`, `Default`, `Clone`, `Copy`, `PartialEq`, `Eq`) and `pub struct Match<'a> { pub line_number: usize, pub text: &'a str }` (derive `Debug`, `PartialEq`, `Eq`). The lifetime `'a` says: a `Match` cannot outlive the text it points into.
2. Write the **tests first** (they will not even compile yet). Use a shared constant:

```rust
const TEXT: &str = "Rust is fast\nrust is safe\nPython is nice";
```

Test these behaviors, one test each:
   - By default the search is case sensitive: `"Rust"` selects only line 1.
   - With `ignore_case`, `"RUST"` selects lines 1 and 2.
   - With `invert`, the pattern `"fast"` selects two lines, and `"is"` selects none.
   - An empty pattern selects every line.
   - An empty text selects nothing.
3. Write `pub fn search<'a>(pattern: &str, contents: &'a str, options: Options) -> Vec<Match<'a>>` to make the tests pass. Use iterator adapters (`lines`, `enumerate`, `filter`, `map`, `collect`) instead of a `for` loop with a mutable vector ([Lesson 05.5](05-iterators.md)).
4. Run `cargo clippy --all-targets -- -D warnings` and `cargo fmt`. Commit twice if you like: once for the tests, once for the code.

<details>
<summary>Hint 1 — case-insensitive matching</summary>

Convert both the pattern and the line with `to_lowercase()` and compare with `contains`. Convert the pattern once, outside the loop, not for every line. (Do this simple version first. Section M7 asks you to measure it.)

</details>

<details>
<summary>Hint 2 — combining invert</summary>

Compute `found` (does the line contain the pattern?) and keep the line when `found != options.invert`. That one comparison handles both modes.

</details>

<details>
<summary>Hint 3 — line numbers</summary>

`lines().enumerate()` counts from 0. Use `index + 1` for the `line_number`. Remember that you must filter first and map after, or keep the index in the pair, so that the numbers stay correct for the skipped lines.

</details>

<details>
<summary>Hint 4 — why `<'a>`?</summary>

The function has two reference parameters (`pattern` and `contents`) and returns references. The compiler cannot know which input the result borrows from, so you must say: the result lives as long as `contents` (`'a`), and `pattern` has no relation to it. If you write the wrong lifetime, the error message points at it ([Lesson 05.3](03-lifetimes.md)).

</details>

**Done when:** `cargo test` shows your tests passing (at least five for `search`), for example:

```text
running 5 tests
test tests::case_sensitive_by_default ... ok
test tests::empty_pattern_selects_every_line ... ok
test tests::empty_text_has_no_lines ... ok
test tests::ignore_case_finds_both ... ok
test tests::invert_selects_the_other_lines ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The names are yours. The order may differ.) Commit: `git add . && git commit -m "feat(minigrep): add the search function with tests"`.

---

#### M3 — Read files and print the results

**Goal:** `minigrep Rust poem.txt` works for one or more files, and read errors are handled.

**Guidance:**

1. Define `pub struct Config` in `lib.rs`: the pattern, the list of files, the `Options`, and the output switches (`line_numbers`, `count`). `main` builds it from the `clap` struct.
2. Define `pub enum GrepError` with `thiserror`. It needs at least a variant for "cannot read this file" that holds the file name and the original `io::Error` (use `#[source]`), and a variant for an error while writing the output.
3. Write `pub fn run(...)` in `lib.rs`. To make it testable, it should not use `println!` directly. Give it a **writer** for results and one for errors (generic parameters `W: Write`, `E: Write`), plus a reader for standard input (`&mut dyn Read`). In the tests you pass a `Vec<u8>` as the writer; in `main` you pass `stdout` and `stderr`.
4. For every file: read it with `std::fs::read_to_string`, call `search`, and write the selected lines. When there is more than one file, put `file:` before each line. If reading fails, write `minigrep: <error>` to the error writer, remember that something failed, and **go on with the next file**.
5. `run` returns the overall result (`Matched`, `NoMatch`, `Failed`). `main` turns it into `ExitCode::SUCCESS`, `ExitCode::from(1)` or `ExitCode::from(2)` ([Lesson 05.11](11-practical-crates-serde-clap-errors.md) showed `main` returning a status).
6. Do not support options yet except the pattern and files (you may already pass the parsed fields through). Add tests for `run` that use a writer in memory.

<details>
<summary>Hint 1 — a generic writer</summary>

`pub fn run<W: Write, E: Write>(config: &Config, stdin: &mut dyn Read, out: &mut W, err: &mut E) -> Result<Outcome, GrepError>`. Use `writeln!(out, "{}", line)?;` instead of `println!`. In a test: `let mut out = Vec::new();` and later `String::from_utf8(out).unwrap()`. A `&[u8]` (for example `"text".as_bytes()`) implements `Read`, so it can play the role of stdin.

</details>

<details>
<summary>Hint 2 — keep going after an error</summary>

Use a `match` on `fs::read_to_string(path)`. In the `Err` arm write the message and set a `failed = true` variable, and do not use `?` there (`?` would stop the whole function). At the end, `if failed { Outcome::Failed } else if selected > 0 { Outcome::Matched } else { Outcome::NoMatch }`.

</details>

<details>
<summary>Hint 3 — the label</summary>

Compute `let label_files = config.files.len() > 1;` once before the loop. A file name can be turned into text with `path.display()`.

</details>

<details>
<summary>Hint 4 — the main function</summary>

`fn main() -> ExitCode`. Lock stdout once (`let stdout = io::stdout(); let mut out = stdout.lock();`). If `run` returns an `Err`, print `minigrep: {error}` to stderr and return `ExitCode::from(2)`.

</details>

**Done when:** (run in the folder `~/projects/rust/code/p05-minigrep-data`; replace the path of the binary as needed, here it is `$B`: `B=~/projects/rust/code/p05-minigrep/target/debug/minigrep` after `cargo build`)

```bash
$B Rust poem.txt; echo "exit: $?"
$B rust poem.txt; echo "exit: $?"
$B Rust poem.txt notes.txt
$B rust poem.txt missing.txt; echo "exit: $?"
$B rust .; echo "exit: $?"
```

```text
Rust is a language
exit: 0
exit: 1
poem.txt:Rust is a language
minigrep: missing.txt: No such file or directory (os error 2)
exit: 2
minigrep: .: Is a directory (os error 21)
exit: 2
```

The second command finds nothing (the file has `Rust` and `RUST`, but no lowercase `rust`), so it prints nothing and the status is 1. Also check that `cargo test` passes. Commit: `git add . && git commit -m "feat(minigrep): read files and print the selected lines"`.

---

#### M4 — Options, standard input and the environment variable

**Goal:** all options of the contract work, and the tool reads standard input.

**Guidance:**

1. Connect `-i` and `-v` to `Options`, `-n` to `line_numbers`, `-c` to `count`.
2. If the `files` list is empty, read all of standard input into a `String` (`read_to_string`) and search it. Do not put a `file:` label in front of the lines in this case.
3. `-c` prints one number per input (with the `file:` label if there is more than one file).
4. Environment variable: `std::env::var_os("MINIGREP_IGNORE_CASE")` returns `Some(...)` if it is set. If it is set, `ignore_case` is true. Do it in `main`, not in the library (the library should not read the environment: it would make it harder to test).
5. Add unit tests for `-n`, `-c`, stdin and the label (use an in-memory writer).

<details>
<summary>Hint 1 — printing one selected line</summary>

Print in this order: the file label (if any) with a colon, the line number (if asked) with a colon, then the text. Use `write!` for the parts and `writeln!` for the last part, so you do not need to build a temporary string.

</details>

<details>
<summary>Hint 2 — a helper for one input</summary>

Make a private helper function that takes the label (`Option<&str>`), the text and the writer, and returns how many lines were selected. Then the loop over files and the stdin case both call the same helper.

</details>

<details>
<summary>Hint 3 — the order of precedence</summary>

`let ignore_case = cli.ignore_case || std::env::var_os("MINIGREP_IGNORE_CASE").is_some();`

</details>

**Done when:**

```bash
$B -i rust poem.txt
$B -n owner poem.txt
$B -c -i rust poem.txt notes.txt
$B -nv the poem.txt
cat poem.txt | $B -n memory
MINIGREP_IGNORE_CASE=1 $B rust poem.txt
$B -n -i is poem.txt notes.txt
```

```text
Rust is a language
RUST is loved by many,
3:Every value has an owner,
4:and the owner decides its fate.
poem.txt:2
notes.txt:1
1:Rust is a language
2:that cares about memory.
3:Every value has an owner,
5:RUST is loved by many,
2:that cares about memory.
Rust is a language
RUST is loved by many,
poem.txt:1:Rust is a language
poem.txt:5:RUST is loved by many,
poem.txt:6:but the compiler is strict.
```

(`-nv the` selects the lines without `the`, with their original numbers: line 4 and line 6 contain `the`.) Commit: `git add . && git commit -m "feat(minigrep): add options, standard input and MINIGREP_IGNORE_CASE"`.

---

#### M5 — Test the whole program from the outside

**Goal:** integration tests that run the real binary, so that a change that breaks the command line is caught by `cargo test`.

**Guidance:**

1. Create `tests/cli.rs`. Cargo compiles each file in `tests/` as a separate crate and builds the binary first. The path of the binary is available in the compile-time variable `env!("CARGO_BIN_EXE_minigrep")` (the name after the last underscore is your binary name).
2. Run it with `std::process::Command`. Use `tempfile::tempdir()` to make a temporary folder that is deleted when the value is dropped, so that your tests do not leave files around.
3. Here is one test to start from (type it, then write the others yourself):

```rust
use std::fs;
use std::process::Command;

use tempfile::tempdir;

fn minigrep() -> Command {
    Command::new(env!("CARGO_BIN_EXE_minigrep"))
}

#[test]
fn finds_lines_in_a_file() {
    let dir = tempdir().unwrap();
    let file = dir.path().join("poem.txt");
    fs::write(&file, "Rust is fun\nsecond line\nRust again\n").unwrap();

    let output = minigrep().arg("Rust").arg(&file).output().unwrap();

    assert!(output.status.success());
    assert_eq!(
        String::from_utf8_lossy(&output.stdout),
        "Rust is fun\nRust again\n"
    );
}
```

4. Write at least four more tests: exit status 1 when nothing matches; exit status 2 and the file name on stderr for a missing file; reading standard input; the environment variable (`Command::env`).

<details>
<summary>Hint 1 — checking the exit status</summary>

`output.status.code()` returns `Option<i32>`. Compare with `Some(1)`.

</details>

<details>
<summary>Hint 2 — feeding standard input</summary>

Build the command with `.stdin(Stdio::piped()).stdout(Stdio::piped()).spawn()`, take the pipe with `child.stdin.take().unwrap()`, write the bytes with `write_all` (`use std::io::Write`), drop the pipe (it is dropped at the end of the statement when you do not keep it), and read the result with `child.wait_with_output()`.

</details>

<details>
<summary>Hint 3 — an unexpected failure about the environment</summary>

If your shell has `MINIGREP_IGNORE_CASE` set, the other tests may behave differently. In each test you can remove it with `.env_remove("MINIGREP_IGNORE_CASE")`.

</details>

**Done when:** `cargo test` runs your unit tests and the integration tests and reports everything as passed:

```text
     Running tests/cli.rs (...)
test reads_standard_input ... ok
test exit_code_is_2_and_message_on_stderr_for_a_missing_file ... ok
test finds_lines_in_a_file ... ok
test environment_variable_turns_on_ignore_case ... ok
test exit_code_is_1_when_nothing_matches ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.28s
```

(Your names and times differ.) Commit: `git add . && git commit -m "test(minigrep): add integration tests for the command line"`.

---

#### M6 — Documentation and the quality gate

**Goal:** a project that others can understand and trust.

**Guidance:**

1. Add `///` doc comments to every public item of `lib.rs` ([Lesson 05.7](07-documentation.md)), and a `//!` comment at the top of the file with one example that runs as a **doc test**.
2. Write `README.md`: what the tool is, how to build it, the usage table (copy from the contract), two or three examples with the real output, how to run the tests, and the exit status table.
3. Run the quality gate and fix everything it reports:

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
cargo doc --no-deps
```

<details>
<summary>Hint 1 — a doc test example</summary>

At the top of `lib.rs`:

````rust
//! A small `grep`: print the lines that contain a pattern.
//!
//! ```
//! use minigrep::{search, Options};
//!
//! let found = search("t", "one\ntwo\nthree", Options::default());
//! assert_eq!(found.len(), 2);
//! ```
````

</details>

<details>
<summary>Hint 2 — a Clippy warning you may see</summary>

Clippy may suggest using a simpler form of a loop or a `match`. Read the message: each warning says what to change. If you do not agree, ask yourself why before you add `#[allow(...)]`; at this stage you almost never need it.

</details>

**Done when:** the four commands above finish without any warning or error, and `cargo test` includes a line like `Doc-tests minigrep` with `test result: ok. 1 passed`. Open `target/doc/minigrep/index.html` and check that every public item has a description. Commit: `git add . && git commit -m "docs(minigrep): document the library and write the README"`.

---

#### M7 — Measure, find the slow part, fix it

**Goal:** apply [Lesson 05.12](12-benchmarking-basics.md) to your own tool. You will find a real slowness with `hyperfine`, fix it, and prove it with numbers.

**Guidance:**

1. **Micro-benchmark `search`.** Add `[[bench]] name = "search"` with `harness = false` to `Cargo.toml` and create `benches/search.rs` with `criterion`. Build a text of 100,000 lines (for example, a loop that repeats the line `"line {i}: the quick brown fox jumps over the lazy dog, again and again"`) outside the timed code, and benchmark two cases: case sensitive and ignore-case. Use `black_box`. Run `cargo bench`.
2. **Measure the whole program.** Build with `cargo build --release`. Make a big test file where every line matches (about 74 MB):

```bash
yes 'the quick brown fox jumps over the lazy dog, again and again' | head -n 1000000 > /tmp/big.txt
```

   Install `hyperfine` (`cargo install hyperfine`) and measure: `hyperfine --warmup 2 --runs 10 "./target/release/minigrep 'lazy dog' /tmp/big.txt"`. Keep the result.
3. **Look at the numbers.** The program has to print a million lines. Think: how many times does it ask the operating system to write? (Hint below.)
4. **Make the change on a branch.** `git switch -c perf/buffered-output` (from your project branch). Change **one** thing. Run the tests. Measure again with the same `hyperfine` command. Write the before and after numbers in the commit message, for example `perf(minigrep): buffer the output (734 ms -> 45 ms)`.
5. If the numbers improved clearly, merge into the project branch: `git switch project/05-minigrep && git merge perf/buffered-output`, then delete the experiment branch with `git branch -d perf/buffered-output`. If the numbers did not improve, delete the branch with `git branch -D perf/buffered-output` and write down what you learned.
6. Add a section "Performance" to the README with the two numbers and your computer.
7. Tag the version: `git tag minigrep-v0.1.0` (the prefix tells your learning repository's tags apart). When you are happy, merge the project branch into `main` (`git switch main && git merge project/05-minigrep`); the main branch may be called `master` on your computer, check with `git branch`.

<details>
<summary>Hint 1 — where is the time?</summary>

The standard output of Rust is **line buffered**: it sends the data to the operating system at the end of every line. A million lines mean a million system calls. A system call is cheap, but a million of them are not.

</details>

<details>
<summary>Hint 2 — the fix</summary>

Wrap the output in `std::io::BufWriter::new(stdout.lock())` in `main`. A `BufWriter` collects bytes in a buffer (8 KiB by default; KiB means kibibyte: 1,024 bytes) and writes them in big pieces. Two things to remember: call `flush()` at the end and report an error if it fails (the buffer's drop also tries to flush but it cannot report errors), and note that the error messages on stderr may now appear *before* the results on stdout, because stdout waits in the buffer. That is normal for tools like `grep` when their output is not a terminal.

</details>

<details>
<summary>Hint 3 — what to expect</summary>

On the author's computer (Apple M4 Pro), 1,000,000 matching lines: before about 734 ms, after about 45 ms, which is more than 16 times faster. Your numbers will be different, but the improvement should be several times. The memory use (about 97 MiB, where MiB means mebibyte: 1,048,576 bytes) did not change, because the program still reads the whole file into memory: that is one of the stretch goals.

</details>

**Done when:**

- `cargo bench` prints times for both benchmarks (on the author's computer: about 1.67 ms case sensitive and 4.0 ms ignore-case for 100,000 lines. Ignore-case is slower, because every line is converted to lower case into a new `String`).
- `hyperfine` shows a clear difference between the version before and the version after your change.
- `git log --oneline` shows your milestone commits, and `git tag` shows `minigrep-v0.1.0`.

---

### Stretch goals

Do them on branches (`git switch -c <name>`), one at a time, with tests and a measurement when speed is the goal. Merge only what you are proud of.

1. **Read line by line.** Use `BufReader` and `lines()` to stream the file, so that a 10 GB file does not need 10 GB of memory. Compare the memory column of `hyperfine` before and after. (Careful: `search` now works on one line at a time, or on a reader.)
2. **Recursive search.** Add `-r`/`--recursive`: when a path is a directory, search all files inside it (use `std::fs::read_dir`). Skip files that are not valid UTF-8 (the standard way to store text as bytes; Rust's `str` and `String` are always UTF-8) with a clear message.
3. **Search in parallel.** Use `std::thread::scope` ([Lesson 05.10](10-threads-channels-and-shared-state.md)) to read and search several files at the same time, but print the results in the order given on the command line. Measure with many files: is it really faster? Threads have a cost.
4. **Faster ignore-case.** Avoid allocating a new `String` per line: compare without creating a lowercase copy of the line (for ASCII text, that is plain English letters from the American Standard Code for Information Interchange, `eq_ignore_ascii_case` on windows of the bytes; for full Unicode, think about the cost and the correctness). Use the criterion baseline of M7 to prove the gain.
5. **Context lines.** Add `-C N` to print `N` lines before and after each selected line, with `--` between groups, like `grep`.
6. **Colors.** Highlight the matched text in red, only when stdout is a terminal (`std::io::IsTerminal`).
7. **Regular expressions.** Add the `regex` crate ([Lesson 05.8](08-cargo-dependencies-and-versions.md) shows how to read its documentation and features) and a `-E` option. Keep the plain-text search as the default.
8. **A release.** Run `cargo build --release`, copy the binary to a folder in your `PATH`, and use your tool daily. Write down in your learning notes what you would change next.

## Common mistakes

1. **Putting the logic into `main.rs`.** Then you cannot unit-test it. Keep `main.rs` thin.
2. **Using `println!` in the library.** The function is then hard to test and cannot write to other places. Take a `Write` parameter.
3. **Using `?` in the loop over files for read errors.** The first missing file stops the whole run, but the contract says to go on. Handle the error with `match`.
4. **A lifetime error in `search`.** For example:

```text
error[E0106]: missing lifetime specifier
```

   The compiler does not know whether the returned references come from `pattern` or from `contents`. Add `'a` to `contents` and to the result ([Lesson 05.3](03-lifetimes.md)).
5. **Converting the pattern to lower case inside the loop.** It does the same work again for every line. Do it once.
6. **Forgetting the exit status.** A tool that always returns 0 cannot be used in scripts (`if minigrep -q ...`). Test it with `echo $?` and with integration tests.
7. **Forgetting to flush a `BufWriter`.** The last lines of output can be lost. Flush at the end and check the error.
8. **Committing `target/`, `big.txt`, or the benchmark output.** Check `git status` before every commit. `/target` is already in the `.gitignore`.
9. **Believing a benchmark in debug mode.** Always use `--release` or `cargo bench`.
10. **Changing many things before measuring.** Change one thing per commit.

## Quiz

1. Why is the matching function declared as `fn search<'a>(pattern: &str, contents: &'a str, ...) -> Vec<Match<'a>>`?

<details>
<summary>Answer</summary>

It returns references into `contents`, not into `pattern`. The lifetime `'a` says that the result is valid as long as `contents` is. Without it the compiler cannot know which argument the result borrows from.

</details>

2. Why does `run` take a writer parameter instead of calling `println!`?

<details>
<summary>Answer</summary>

So that tests can pass an in-memory buffer (`Vec<u8>`) and check the output, and so that the same function can write to stdout, to a file, or anywhere else.

</details>

3. What are the three exit statuses of `minigrep`, and why do they matter?

<details>
<summary>Answer</summary>

`0` (something was selected and nothing failed), `1` (nothing selected), `2` (an error). Scripts and other programs use the status to decide what to do next, for example `if minigrep ERROR log.txt; then ...`.

</details>

4. Why does the integration test use `env!("CARGO_BIN_EXE_minigrep")`?

<details>
<summary>Answer</summary>

Cargo builds the binary before it runs the integration tests and puts its path in this compile-time variable, so the test runs the real program, exactly like a user would.

</details>

5. Why was printing a million lines slow, and how was it fixed?

<details>
<summary>Answer</summary>

Standard output is line buffered, so each line caused a separate system call. Wrapping it in a `BufWriter` collects the output in a buffer and writes it in large pieces, which made the program more than 16 times faster in the author's measurement.

</details>

6. Why is the environment variable read in `main` and not in the library?

<details>
<summary>Answer</summary>

Reading the environment is a side effect that depends on the outside world. Keeping it in `main` keeps the library a plain function of its inputs, which is easy and reliable to test.

</details>

7. Why should a speed change be a separate commit with the numbers in the message?

<details>
<summary>Answer</summary>

It documents the evidence and the reason, and makes it easy to find or revert the change later ([Lesson 05.12](12-benchmarking-basics.md)).

</details>

8. A colleague says "I added threads and it must be faster". What do you ask?

<details>
<summary>Answer</summary>

"What did you measure?" Threads cost time to start and coordinate. Measure with `hyperfine` or `criterion` on realistic input, with and without threads, in release mode, and compare.

</details>

## Summary

- A good tool has a thin binary and a library with the logic, so that the logic can be tested without a terminal.
- A written contract (usage, output formats, errors, exit statuses) turns "it works" into checks you can run.
- TDD, unit tests, integration tests with the real binary, and doc tests protect different kinds of mistakes.
- Lifetimes let `search` return borrowed lines without copying.
- Errors go to stderr, results go to stdout, and the exit status tells scripts what happened.
- Measure first: `criterion` for functions, `hyperfine` for the whole program. The real slowness here was not the search but the output, and a `BufWriter` fixed it.
- One commit per milestone, one commit per optimization with the numbers, a branch for each experiment, a tag for each release.

## Sources and further reading

- [The Rust Programming Language — Chapter 12: An I/O Project: Building a Command Line Program](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) — the official `minigrep`; compare its approach with yours after you finish.
- [Command Line Applications in Rust](https://rust-cli.github.io/book/) — output, exit codes, testing CLIs, packaging.
- [std::io::BufWriter](https://doc.rust-lang.org/std/io/struct.BufWriter.html) — how the buffer works and why you must flush.
- [std::process::ExitCode](https://doc.rust-lang.org/std/process/struct.ExitCode.html) — returning an exit status from `main`.
- [The Cargo Book — Integration tests and CARGO_BIN_EXE](https://doc.rust-lang.org/cargo/reference/environment-variables.html) — the environment variables that Cargo sets, including `CARGO_BIN_EXE_<name>`.
- [Pro Git — Git Basics: Tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging) — tags and releases.

---

[🏠 Start](../START-HERE.md) · [← 05.12 Benchmarking basics](12-benchmarking-basics.md) · [📋 Phase 05](README.md) · [05.14 Checkpoint →](14-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
