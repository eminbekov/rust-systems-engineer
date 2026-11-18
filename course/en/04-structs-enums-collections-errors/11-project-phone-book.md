<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.10 Error handling](10-error-handling.md) · [📋 Phase 04](README.md) · [04.12 Checkpoint →](12-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/11-project-phone-book.md) <!-- nav -->

# Lesson 04.11 — Project: phone book

> **Phase 04 — Structs, enums, collections and errors** · Lesson 11 of 12 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to combine structs, enums, `BTreeMap`, strings, `Result` and modules in one real program.
- How to design a small *command language* and turn text into an enum (a parser).
- How to separate logic that you can test (a library crate) from input/output (a binary crate).
- How to read a program's input from a terminal or from a pipe, and how to load and save a text file.
- How to work like a professional: one Git commit per milestone, a feature branch, a merge, a tag, and a clean `cargo fmt`, `cargo clippy` and `cargo test`.

## Before you start

- You finished [Lesson 04.10 — Error handling](10-error-handling.md). This project uses everything from Lessons 04.1 to 04.10, especially [modules](06-modules-crates-and-packages.md), [strings](08-strings-in-depth.md), [maps](09-hash-maps.md) and [errors](10-error-handling.md).
- You remember the Git basics from Phase 01: [commits](../01-git-and-version-control/03-your-first-repository-and-commits.md), [history](../01-git-and-version-control/04-viewing-history-and-changes.md), [branches and merging](../01-git-and-version-control/06-branches-and-merging.md) and [commit conventions](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md). In this project you use all of them.
- You know how to write and run tests ([Lesson 02.8](../02-programming-basics/08-your-first-tests.md)).
- This lesson is a **project**: it gives you the goal, the exact rules (the *contract*) and a plan with checks. It does **not** give the full solution. Small hints are hidden under "Hint"; try first without them.

## New words

| Word | Meaning |
|---|---|
| **REPL** (Read-Eval-Print Loop) | A program that repeats: read a command, run it, print the result, and read the next one. Shells and the Python prompt work this way. |
| **command language** | The small set of words a user types to control a program, for example `add alice 5550100`. |
| **parser** | Code that turns text into structured data. Here: a line of text into a `Command` enum. |
| **prompt** | A short mark (here `> `) that a program prints to show it is waiting for input. |
| **pipe** (`\|`) | A shell feature that sends the output of one command into the input of another: `printf 'a\n' \| cargo run`. |
| **terminal vs. pipe** | When a person types, standard input is a *terminal*. When another program or a pipe feeds it, standard input is *not* a terminal. A program can ask which one it is. |
| **EOF** (End Of File) | The signal that there is no more input. In a terminal you send it with `Ctrl+D`. `read_line` then returns `0` bytes. |
| **flush** | To force buffered output to appear now. `print!` without a new line may stay in a buffer until you flush. |
| **library crate** | Code in `src/lib.rs` that other code (including your own binary and tests) can use. |
| **binary crate** | Code in `src/main.rs` that becomes a runnable program. |
| **contract** | The exact, written rules of how the program must behave, including every message. |
| **milestone** | A planned step with a clear goal and a check ("done when"). Written M1, M2, ... |
| **acceptance test** | A check that shows the program does what the contract says, from the outside. |
| **linter** | A tool that reads your code and warns about suspicious or unidiomatic parts. Rust's linter is **Clippy**. |
| **`rustfmt`** | The official tool that formats Rust code in one standard style. You run it with `cargo fmt`. |
| **README** | A text file (here `README.md`) that tells a reader what the project is and how to use it. |
| **feature branch** | A Git branch made for one piece of work, merged back into `main` when it is done. |
| **tag** | A permanent, named label on one commit in Git, such as `v0.1.0`. |
| **Conventional Commits** | A naming style for commit messages: `feat: add search`, `fix: ...`, `docs: ...`, `test: ...`, `chore: ...`. |
| **CLI** (Command-Line Interface) | A way to use a program by typing commands in a terminal. |
| **I/O** (Input/Output) | Reading from and writing to the outside world: the keyboard, the screen, files. |
| **CI** (Continuous Integration) | A server that automatically builds and tests your code after every push to a remote repository. |

## The project

### What you will build

A **phone book** program named `p04-phonebook`. It reads commands from the keyboard (or from a
pipe), one per line, and answers on the screen:

```text
> add alice +998-90-111-22-33
ok: added +998-90-111-22-33 to alice
> add alice +998-91-444-55-66
ok: added +998-91-444-55-66 to alice
> get alice
alice: +998-90-111-22-33, +998-91-444-55-66
> find al
alice
> quit
```

(The `> ` marks and your typing appear only in a terminal; section "Contract" explains why.)
You can start it with a file name, and then the phone book is loaded from that file at the start
and saved to it at the end.

Your code goes in `~/projects/rust/code/p04-phonebook`.

### Contract

**Usage**

```text
p04-phonebook [FILE]
```

- Without `FILE`, the phone book lives only in memory.
- With `FILE`, the program loads the book from it at the start and saves the book to it when the
  program ends normally (`quit` or end of input). A missing file is **not** an error: it means an
  empty book, and the file is created at the end.
- The program reads standard input line by line until `quit` or end of input (EOF).
- It prints the `> ` prompt (no new line after it) **only when standard input is a terminal**. In a pipe,
  the output must contain only the replies.
- A blank line (or a line with only spaces) does nothing and prints nothing.
- Exit code `0` on normal exit. Exit code `1` if the file cannot be read, contains bad data, or cannot be
  written. These messages go to **stderr**, in the form `error: <message>`.

**Rules**

- A **name** is one word (no spaces). Names are case-insensitive: store and print them in lower case.
- A **phone number** is an optional `+`, then only digits and `-`, with **at least 7 digits**.
  Valid: `+998-90-123-45-67`, `5550100`. Invalid: `12345`, `12a45678`, `+`, `""`.
- One name can have several numbers; keep them in the order they were added.
- The same number twice for the same name is an error.
- Commands are case-insensitive too (`QUIT` works), but their arguments are used as typed (then names are lowered).

**Commands and exact output** (every reply is one `println!`; `list`, `find` and `help` print several lines)

| Input | Output |
|---|---|
| `add <name> <phone>` | `ok: added <phone> to <name>` |
| same number again | `error: <name> already has <phone>` |
| bad number | `error: invalid phone '<phone>'` |
| `get <name>` | `<name>: <phone1>, <phone2>` (in the order added) |
| `del <name>` | `ok: deleted <name>` |
| `get` or `del` for an unknown name | `error: no entry for '<name>'` |
| `list` | one `<name>: <phones>` line per name, sorted by name; or `(empty)` |
| `find <prefix>` | the names that start with the prefix (case-insensitive), sorted, one per line; or `(no matches)` |
| `count` | `<n> names, <m> numbers` |
| `help` | the list of commands (format below) |
| `quit` | ends the loop (prints nothing) |
| wrong number of arguments | `error: usage: add <name> <phone>`, `error: usage: get <name>`, `error: usage: del <name>`, `error: usage: find <prefix>`; for `list`, `count`, `help`, `quit` with extra words: `error: usage: this command takes no arguments` |
| any other first word | `error: unknown command '<word>' (type help)` |

The text of `help`:

```text
commands:
  add <name> <phone>
  get <name>
  del <name>
  list
  find <prefix>
  count
  help
  quit
```

**File format** (what `save` writes and `load` reads): one line per name, in name order. The line
is the name followed by each phone, all separated by single spaces, and it ends with a new line.
An empty book gives an empty file. Example:

```text
alice +1-555-0100 +1-555-0199
bob 5550123
```

When loading, blank lines are skipped; a phone that is invalid or repeated makes the whole load fail with
`error: bad data on line <N>: <reason>` (lines are counted from 1), for example
`error: bad data on line 2: invalid phone '12'`.

**Suggested types** (you may change the details, but keep the behaviour):

```rust
// src/phonebook.rs
pub enum PhoneBookError { InvalidPhone(String), Duplicate { name: String, phone: String }, NoEntry(String) }
pub fn is_valid_phone(phone: &str) -> bool;
pub struct PhoneBook { /* a BTreeMap<String, Vec<String>> */ }
impl PhoneBook {
    pub fn new() -> PhoneBook;
    pub fn add(&mut self, name: &str, phone: &str) -> Result<(), PhoneBookError>;
    pub fn get(&self, name: &str) -> Result<&[String], PhoneBookError>;
    pub fn delete(&mut self, name: &str) -> Result<(), PhoneBookError>;
    pub fn count_names(&self) -> usize;
    pub fn count_numbers(&self) -> usize;
    pub fn list(&self) -> Vec<(&String, &Vec<String>)>;
    pub fn find(&self, prefix: &str) -> Vec<&String>;
}

// src/command.rs
pub enum Command { Add { name: String, phone: String }, Get(String), Del(String),
                   List, Find(String), Count, Help, Quit }
pub enum ParseError { Unknown(String), Usage(&'static str) }
pub fn parse_command(line: &str) -> Result<Command, ParseError>;
pub fn execute(book: &mut PhoneBook, command: Command) -> String;   // the text to print

// src/storage.rs
pub enum StorageError { Io(std::io::Error), BadLine { line: usize, reason: String } }
pub fn to_text(book: &PhoneBook) -> String;
pub fn from_text(text: &str) -> Result<PhoneBook, StorageError>;
pub fn load(path: &str) -> Result<PhoneBook, StorageError>;
pub fn save(book: &PhoneBook, path: &str) -> Result<(), StorageError>;
```

Why this design? The three modules hold the **logic** and know nothing about the keyboard, so tests can
call them directly. Only `src/main.rs` talks to the terminal. This split (logic in a library, thin
`main`) is how real Rust programs are organized. `ParseError`, `PhoneBookError` and `StorageError` each need
`Display` messages that match the contract, so `main` can print `error: {e}`.

The project layout:

```text
code/p04-phonebook/
├── .gitignore         ← contains: /target
├── Cargo.toml
├── README.md
└── src/
    ├── lib.rs         ← declares the modules; "pub mod phonebook;" ...
    ├── main.rs        ← the loop: read, parse, execute, print
    ├── phonebook.rs   ← PhoneBook, PhoneBookError, is_valid_phone
    ├── command.rs     ← Command, ParseError, parse_command, execute
    └── storage.rs     ← StorageError, to_text, from_text, load, save
```

The library crate is named `p04_phonebook` inside Rust code (a dash in the package name becomes an
underscore, see [Lesson 04.6](06-modules-crates-and-packages.md)). In `main.rs` you write, for example,
`use p04_phonebook::{parse_command, execute, PhoneBook};` after re-exporting those names in `lib.rs` with `pub use`.

### Step-by-step plan

Do the milestones **in order**. Finish each one, run its "Done when" checks, and **make a Git commit**
before you start the next. The commit message names the milestone, in the Conventional Commits style from
[Lesson 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md).

#### M1 — Project and repository

**Goal:** an empty project that builds, with a README and a correct `.gitignore`, committed to Git.

**Guidance:**

1. Create the project: `cd ~/projects/rust && cargo new code/p04-phonebook && cd code/p04-phonebook`.
2. Check which Git repository you are in: `git rev-parse --show-toplevel`. It should print the folder of your
   learning repository from [Lesson 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md). If it prints an error ("not a git repository"), run `git init` here.
3. When `cargo new` runs inside an existing Git repository, it does **not** create a `.gitignore`. Create one
   containing the single line `/target`, so the build folder is never committed. Keep `Cargo.lock`
   in Git: this project is a program, and a program should commit its lock file (explained in [Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
4. Create `src/lib.rs` with the three module lines (`pub mod phonebook;`, `pub mod command;`, `pub mod storage;`) and
   three empty files `src/phonebook.rs`, `src/command.rs`, `src/storage.rs`.
5. Write a short `README.md` with the title and one sentence. You will extend it in M7.

<details>
<summary>Hint</summary>

The commands in order: `printf '/target\n' > .gitignore`, `touch src/phonebook.rs src/command.rs src/storage.rs`.
Then, from the repository root, `git status` should show the new files but **not** `target/`.

</details>

**Done when:**

```bash
cargo build
git status --short -uall
git add .
git commit -m "chore: start p04-phonebook"
git log --oneline -1
```

- `cargo build` ends with `Finished` and no warnings.
- Before `git add`, `git status --short -uall` lists each new file of `code/p04-phonebook` with `??`
  (for example `?? code/p04-phonebook/.gitignore` and `?? code/p04-phonebook/Cargo.toml`), and **no path that
  contains `target/`**. (The option `-uall` shows every untracked file instead of only the folder.)
- The last command shows one line: a short hash and `chore: start p04-phonebook`.

#### M2 — The `PhoneBook` type

**Goal:** a tested `PhoneBook` with `add`, `get`, `delete`, `count_names` and `count_numbers`, plus phone validation.

**Guidance:**

- In `src/phonebook.rs`, write `PhoneBookError` with `Debug`, `PartialEq`, `Display` and `Error` ([Lesson 04.10](10-error-handling.md)).
  The `Display` text must match the contract: `invalid phone '12'`, `alice already has 5550100`, `no entry for 'bob'`.
- Store contacts in a `BTreeMap<String, Vec<String>>`. A `BTreeMap` keeps the names sorted, which `list` and
  `find` need later. (A `HashMap` also works if you sort in M3.)
- `is_valid_phone`: remove an optional leading `+` (`strip_prefix`), then go through the characters: a digit
  is counted, a `-` is allowed, anything else makes the phone invalid. At the end require at least 7 digits.
- `add` must **validate first**, then lower-case the name, then use the `entry` API (API, Application Programming Interface: the functions a type offers you, here `entry` and its helpers, Lesson 04.9) to get the vector for this
  name, then reject a duplicate, then `push`. Careful: a failed `add` must not leave an empty vector behind.
  Think about when `entry(...).or_default()` creates the vector, and where you check the phone.
- Write the tests as you go. A test for each rule: valid and invalid phones; add + get with different upper/lower
  case in the name; order of numbers; duplicate; invalid phone does not create a name; delete + delete again;
  both counts.

<details>
<summary>Hint</summary>

Skeleton (the shape only):

```rust
#[derive(Debug, Default)]
pub struct PhoneBook {
    entries: BTreeMap<String, Vec<String>>,
}

pub fn add(&mut self, name: &str, phone: &str) -> Result<(), PhoneBookError> {
    if !is_valid_phone(phone) { return Err(PhoneBookError::InvalidPhone(phone.to_string())); }
    let name = name.to_lowercase();
    let phones = self.entries.entry(name.clone()).or_default();
    // if phones.contains(&phone.to_string()) { return Err(...Duplicate...) }
    // phones.push(...); Ok(())
}
```

`#[derive(Default)]` gives you `PhoneBook::default()`, so `new()` can just call it. `contains` needs a
`&String`, so pass `&phone.to_string()`.
For the duplicate check you need `name` after it was used for `entry`, which is why the skeleton clones it
(`name.clone()`). In tests, `book.get("alice").unwrap()` returns `&[String]`; compare with `["a", "b"]`.

</details>

**Done when:**

```bash
cargo test
```

prints a list of your tests, all `ok`, and `test result: ok.` with at least **6 passed** and `0 failed`. Then:

```bash
cargo clippy --all-targets -- -D warnings
cargo fmt --check
git add . && git commit -m "feat: add PhoneBook with add, get, delete and counts"
```

`clippy` and `fmt --check` print nothing (or `Finished`) and exit with code 0.

#### M3 — Sorted `list` and `find`

**Goal:** `PhoneBook::list` and `PhoneBook::find`, sorted, case-insensitive for the prefix.

**Guidance:**

- `list` returns all contacts sorted by name. With a `BTreeMap` this is just the iteration order.
- `find(prefix)`: lower-case the prefix, go through the keys, keep those that `starts_with` it.
- Tests: add `carol`, `alice`, `alan` in this order; `list` gives `alan`, `alice`, `carol`; `find("AL")` gives
  `alan`, `alice`; `find("z")` is empty; `list` of an empty book is empty.

<details>
<summary>Hint</summary>

`self.entries.iter().collect()` gives a `Vec<(&String, &Vec<String>)>` already in key order. For `find`, make a
`Vec<&String>`, loop over `self.entries.keys()` and `push` the matching ones.

</details>

**Done when:** `cargo test` passes with the new tests (at least 2 more), and

```bash
git add . && git commit -m "feat: add sorted list and find"
```

#### M4 — Parsing commands

**Goal:** `Command`, `ParseError` and `parse_command` in `src/command.rs`, fully tested, with no input/output code.

**Guidance:**

- `parse_command(line)` receives a line that is **not blank**. Split it into words with `split_whitespace`
  and collect them into a `Vec<&str>` ([Lesson 04.8](08-strings-in-depth.md)).
- Lower-case the first word and `match` on it together with the number of words. A `match` on a tuple such as
  `(word.as_str(), words.len())` lets you write arms like `("add", 3)` for the correct shape and
  `("add", _)` for the wrong shape (a usage error).
- `ParseError::Usage` carries the usage text; `ParseError::Unknown` carries the word as typed. Implement `Display`
  to match the contract: `usage: add <name> <phone>` and `unknown command 'frobnicate' (type help)`.
- Tests: one test that parses every command; one with extra spaces around words; one for each kind of wrong
  argument count; one for an unknown command; one for upper-case `QUIT`.

<details>
<summary>Hint</summary>

The core of the function:

```rust
let words: Vec<&str> = line.split_whitespace().collect();
let Some(first) = words.first() else { /* the line was blank */ };
match (first.to_lowercase().as_str(), words.len()) {
    ("add", 3) => Ok(Command::Add { name: words[1].to_string(), phone: words[2].to_string() }),
    ("add", _) => Err(ParseError::Usage("add <name> <phone>")),
    // ... get, del, find, list, count, help, quit ...
    _ => Err(ParseError::Unknown(first.to_string())),
}
```

An or-pattern such as `("list" | "count" | "help" | "quit", _)` lets one arm cover several words. Put the
exact-shape arms (`("list", 1)`) before it.

</details>

**Done when:** `cargo test` passes with at least **4 new** tests for `parse_command`. Then
`cargo clippy --all-targets -- -D warnings` and `cargo fmt --check` are silent, and:

```bash
git add . && git commit -m "feat: parse commands into the Command enum"
```

#### M5 — Running commands and the loop

**Goal:** `execute` and `main`: the program works from a pipe and from a terminal. (No file yet.)

**Guidance:**

- `execute(&mut book, command) -> String` returns the text to print for a command, exactly as in the
  contract. For `list` and `find`, join the lines with `"\n"`. Test it **without** `main`: parse a line, execute
  it, compare the string.
- `src/lib.rs` re-exports the names `main` needs: `pub use command::{execute, parse_command, Command};` and
  `pub use phonebook::PhoneBook;`.
- `main`: create a `PhoneBook`; make a `String` for the line; `loop`: if input is a terminal, print the prompt
  and flush; clear the line buffer and call `stdin.read_line(&mut line)`; a result of `Ok(0)` means EOF, so `break`; a blank
  line means `continue`; otherwise `parse_command`; `Quit` ends the loop; a command is executed and its text
  printed with `println!`; a `ParseError` is printed as `error: {e}`.
- Terminal check: `use std::io::IsTerminal;` and then `std::io::stdin().is_terminal()`.
- Flush: `use std::io::Write;` and `std::io::stdout().flush()` after `print!("> ")`. Without the flush, the
  prompt may appear late, because `print!` without a new line is buffered.
- If reading the input fails, print `error: cannot read input: {e}` to stderr and `exit(1)`.

<details>
<summary>Hint</summary>

The loop in outline:

```rust
let stdin = std::io::stdin();
let interactive = stdin.is_terminal();
let mut line = String::new();
loop {
    if interactive { print!("> "); std::io::stdout().flush().expect("cannot flush stdout"); }
    line.clear();                              // read_line appends, so clear first
    let bytes_read = match stdin.read_line(&mut line) { Ok(n) => n, Err(e) => { /* print, exit(1) */ } };
    if bytes_read == 0 { break; }
    if line.trim().is_empty() { continue; }
    match parse_command(&line) {
        Ok(Command::Quit) => break,
        Ok(command) => println!("{}", execute(&mut book, command)),
        Err(e) => println!("error: {e}"),
    }
}
```

Note that parse errors are printed to **stdout** here (they are replies in the language of the program, so
the contract requires them in the normal output), while file errors in M6 go to stderr.

</details>

**Done when:** first the main acceptance test, a piped session:

```bash
printf 'add Alice +1-555-0100\nadd alice +1-555-0199\nget ALICE\ncount\ndel bob\nfrobnicate\nquit\n' | cargo run -q
```

must print exactly:

```text
ok: added +1-555-0100 to alice
ok: added +1-555-0199 to alice
alice: +1-555-0100, +1-555-0199
1 names, 2 numbers
error: no entry for 'bob'
error: unknown command 'frobnicate' (type help)
```

Then a bigger session that shows the other commands:

```bash
printf 'add alice +1-555-0100\nadd alice +1-555-0100\nadd bob 12\nadd bob\n\n   \nadd Carol 5550123\nadd alan +998-90-111-22-33\nlist\nfind AL\nfind z\ndel carol\nlist\ncount\nhelp\nquit\nadd never 1234567\n' | cargo run -q
```

must print exactly:

```text
ok: added +1-555-0100 to alice
error: alice already has +1-555-0100
error: invalid phone '12'
error: usage: add <name> <phone>
ok: added 5550123 to carol
ok: added +998-90-111-22-33 to alan
alan: +998-90-111-22-33
alice: +1-555-0100
carol: 5550123
alan
alice
(no matches)
ok: deleted carol
alan: +998-90-111-22-33
alice: +1-555-0100
2 names, 2 numbers
commands:
  add <name> <phone>
  get <name>
  del <name>
  list
  find <prefix>
  count
  help
  quit
```

(The last line, `add never ...`, comes after `quit`, so it must not run.) Also check the edge cases:

```bash
printf 'list\nfind a\ncount\n' | cargo run -q
printf 'add x 1234567' | cargo run -q
```

```text
(empty)
(no matches)
0 names, 0 numbers
ok: added 1234567 to x
```

The second command has no final new line and no `quit`: the end of input must end the program cleanly.
Run `cargo run` without a pipe and check that you see the `> ` prompt. Type `help`, then `quit`.
Finally `cargo test`, `cargo clippy --all-targets -- -D warnings`, `cargo fmt --check` and:

```bash
git add . && git commit -m "feat: add the command loop"
```

#### M6 — Load and save (on a feature branch)

**Goal:** `cargo run -- contacts.txt` remembers the contacts between runs. You develop this on a **feature
branch** and merge it, as a team would.

**Guidance:**

1. Create the branch: `git switch -c feature/save-load`.
2. In `src/storage.rs`, write `StorageError` (`Io` and `BadLine`) with `Display`, `Error` and `From<std::io::Error>`
   ([Lesson 04.10](10-error-handling.md)). Messages: `file error: <io error>` and `bad data on line <N>: <reason>`.
3. `to_text(&PhoneBook) -> String` follows the file format in the contract. `from_text(&str) -> Result<PhoneBook,
   StorageError>` reads it: for each non-blank line, the first word is the name and every other word is a phone to
   `add`; if `add` fails, return `BadLine` with the line number (counting from 1) and the error's text (`e.to_string()`).
4. `load(path)` uses `std::fs::read_to_string`. If the error's `kind()` is `std::io::ErrorKind::NotFound`, return an
   empty book; any other error becomes `StorageError::Io`. `save(&book, path)` uses `std::fs::write(path, to_text(book))`.
5. Tests for the pure functions (no files needed): a round trip `to_text` then `from_text` then `to_text` gives the same text;
   a bad phone on line 2 is reported as line 2; `load` of a file that does not exist gives an empty book.
6. In `main`: read the file name with `std::env::args().nth(1)`. This gives an `Option<String>`: `args()` yields the
   program name first, then each argument, and `nth(1)` takes the second item or `None` if there is none. If a path is
   given, `load` it at the start (on error print `error: {e}` to stderr and `exit(1)`). When the loop ends, `save` it
   (the same error handling).
7. Merge: `git switch main`, then `git merge --no-ff feature/save-load`, then delete the branch.

<details>
<summary>Hint</summary>

A `match` with a guard is a clear way to treat "not found" as success:

```rust
match std::fs::read_to_string(path) {
    Ok(text) => from_text(&text),
    Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(PhoneBook::new()),
    Err(e) => Err(StorageError::Io(e)),
}
```

In `main`, keep the path as `Option<String>` and use `match &path { Some(p) => ..., None => ... }`
to choose between loading a file and starting with `PhoneBook::new()`. A small helper function that
saves and exits on error keeps `main` readable.

</details>

**Done when:**

```bash
rm -f contacts.txt
printf 'add alice +1-555-0100\nadd Bob 5550123\nadd alice +1-555-0199\nquit\n' | cargo run -q -- contacts.txt
cat contacts.txt
```

prints

```text
ok: added +1-555-0100 to alice
ok: added 5550123 to bob
ok: added +1-555-0199 to alice
alice +1-555-0100 +1-555-0199
bob 5550123
```

(the last two lines come from `cat`). A second run must remember the data and save the changes:

```bash
printf 'list\ndel bob\nadd zed 7777777\nquit\n' | cargo run -q -- contacts.txt
cat contacts.txt
```

```text
alice: +1-555-0100, +1-555-0199
bob: 5550123
ok: deleted bob
ok: added 7777777 to zed
alice +1-555-0100 +1-555-0199
zed 7777777
```

Then check the error path and the exit code:

```bash
printf 'alice 1111111\nbob 12\n' > bad.txt
cargo run -q -- bad.txt < /dev/null
echo $?
rm bad.txt contacts.txt
```

```text
error: bad data on line 2: invalid phone '12'
1
```

Do not commit `contacts.txt` (you removed it). Now merge:

```bash
cargo test && cargo clippy --all-targets -- -D warnings && cargo fmt --check
git add . && git commit -m "feat: load and save the phone book"
git switch main
git merge --no-ff feature/save-load
git branch -d feature/save-load
git log --oneline --graph
```

`git log --oneline --graph` shows the branch and the merge commit, like this (your hashes differ):

```text
*   890f511 Merge branch 'feature/save-load'
|\
| * f7a35b3 feat: load and save the phone book
|/
* 40e2830 feat: add the command loop
* 64b0477 feat: parse commands into the Command enum
...
```

(If Git opens an editor for the merge message, save and close it.)

#### M7 — Quality, README and release tag

**Goal:** the project is clean, documented and tagged `v0.1.0`.

**Guidance:**

- Run `cargo fmt` (it rewrites files in the standard style), then `cargo clippy --all-targets -- -D warnings`. The
  option `-D warnings` turns every warning into an error. Fix everything Clippy reports; read each message,
  because Clippy often explains the idiomatic way.
- Search your code for `unwrap()` and `expect(` outside the tests. Each one must be a case you can prove
  cannot fail (or be replaced with proper error handling). Explain any remaining `expect` with a good message.
- Extend `README.md`: what it does, how to run it (`cargo run -- contacts.txt`), the command table, the file
  format, and how to run the tests.
- Tag the release.

<details>
<summary>Hint</summary>

A good README has these headings: *What it is*, *Run it*, *Commands*, *File format*, *Tests*.
Create an annotated tag with `git tag -a v0.1.0 -m "p04-phonebook 0.1.0"` and look at it with `git tag`
and `git show v0.1.0 --stat`.

</details>

**Done when:**

```bash
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
git status --short
git add . && git commit -m "docs: write the README and tidy the code"
git tag -a v0.1.0 -m "p04-phonebook 0.1.0"
git tag
```

- the first line prints no warnings and ends with `test result: ok.`;
- `git status --short` before the commit lists only your README and code changes (no `target/`, no `contacts.txt`);
- `git tag` prints `v0.1.0`;
- `git log --oneline` shows one commit per milestone (M1 to M7), with Conventional Commits messages.

If you use a remote (GitHub or GitLab, [Lesson 01.8](../01-git-and-version-control/08-remotes-github-and-gitlab.md)), push the branch and the tag: `git push` and `git push --tags`.

### Stretch goals

1. **Names with spaces:** accept `add "Ali Valiyev" +998-90-123-45-67` (quotes around the name). Think about how the
   file format and `parse_command` must change.
2. **`rename <old> <new>`:** move all numbers to a new name. What if the new name exists?
3. **`undo`:** undo the last change. Keep a `Vec` of reverse operations (an enum!) and pop from it.
4. **An integration test:** run the real program from a test in `tests/cli.rs` and compare its output with the contract.
5. **Exit code for `del` of unknown names** in a non-interactive run (return 3 at the end if any command failed). Is it a good idea? Discuss with yourself, then decide.
6. **Better errors:** replace the manual `Display` and `Error` code with the `thiserror` crate after [Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md).

<details>
<summary>Hint for stretch goal 4</summary>

Cargo gives integration tests the path of the compiled program in `env!("CARGO_BIN_EXE_p04-phonebook")`. Start it with
`std::process::Command`, set `.stdin(Stdio::piped())` and `.stdout(Stdio::piped())`, write the input with
`write_all`, call `wait_with_output()` and compare `String::from_utf8(output.stdout)` with the expected text.
Remember that the prompt is not printed because stdin is a pipe.

</details>

## Common mistakes

**1. Printing the prompt in a pipe.** The expected output of the acceptance tests contains no `> `. Print it only if `stdin.is_terminal()`.

**2. Forgetting to flush.** `print!("> ")` has no new line, so the text may stay in a buffer, and the user types without seeing the prompt. Call `io::stdout().flush()`.

**3. `read_line` appends.** If you never call `line.clear()`, the second command contains the first one too.

**4. Not trimming the line.** `read_line` keeps the new line at the end. `split_whitespace` removes it for the parser, but `line.trim().is_empty()` is the right test for a blank line.

**5. Relying on `HashMap` order.** If you used a `HashMap`, `list` and `find` must sort; otherwise the output changes from run to run and the acceptance test fails randomly.

**6. A half-done `add`.** When `entry(...).or_default()` runs before the validation, an invalid phone can leave a name with an empty list of numbers (`count` would say `1 names, 0 numbers`). Validate first.

**7. Committing `target/` or `contacts.txt`.** Check `git status` before every `git add .`. Build output and test data do not belong in Git.

**8. `unwrap()` on input from the user or the file.** A typo would crash the program. Handle every `Result` (Lesson 04.10).

## Quiz

1. Why does the project keep the logic in a library crate and only the terminal code in `main.rs`?

<details>
<summary>Answer</summary>

Logic in the library can be tested by calling functions directly, without typing into a terminal or reading
files. `main.rs` stays small (read a line, call the library, print the answer), so there is little untested code.

</details>

2. How does the program know whether to print the `> ` prompt?

<details>
<summary>Answer</summary>

It asks `std::io::stdin().is_terminal()` (the `IsTerminal` trait). A person typing gives a terminal and the
prompt is shown; a pipe or a file is not a terminal and the prompt is skipped, so scripts get clean output.

</details>

3. What does `read_line` return at the end of the input, and what do you do?

<details>
<summary>Answer</summary>

`Ok(0)`: zero bytes were read, which means EOF. The loop must `break`. (A blank line is `Ok(1)` for the new line
character, not `Ok(0)`.)

</details>

4. Why is a missing data file not treated as an error in `load`?

<details>
<summary>Answer</summary>

The first time the program runs there is no file yet, and that is a normal situation. The contract says a
missing file means an empty book, and the file is created when the program saves. Other I/O errors (no permission,
a directory instead of a file) are real errors.

</details>

5. Why did we use a feature branch and `git merge --no-ff` for M6?

<details>
<summary>Answer</summary>

A branch keeps unfinished work away from `main` until it passes all checks. `--no-ff` always creates a merge
commit, so the history shows that a group of commits was one feature. It is practice for team work.

</details>

6. What does `cargo clippy --all-targets -- -D warnings` do?

<details>
<summary>Answer</summary>

It runs the Clippy linter on all code, including tests, and `-D warnings` makes every warning an error, so the
command fails (exit code not 0) until all warnings are fixed. This is how teams keep a codebase clean in CI.

</details>

## Summary

- A real program is built in small, checked steps: the contract first, then milestones, with a commit for each.
- Keep logic (`lib.rs` and modules) separate from input/output (`main.rs`), so the logic is easy to test.
- Turn text into an enum with a parser (`parse_command`), then run the enum with one `match`.
- Use a `BTreeMap` when you need sorted output, validate before you change data, and keep errors as enums with `Display`.
- Terminal and pipe differ: print the prompt only for a terminal, flush after `print!`, and treat `Ok(0)` from `read_line` as end of input.
- A missing file can be a normal situation; other I/O errors are real errors, reported on stderr with exit code 1.
- Work like a professional: `.gitignore` for `target/`, Conventional Commits, a feature branch with a merge, a tag, and clean `cargo fmt`, `cargo clippy` and `cargo test`.

## Sources and further reading

- [The Rust Programming Language — 12. An I/O Project: Building a Command Line Program](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) — a similar project in the official book (you will build `minigrep` in Phase 05).
- [std::io::Stdin](https://doc.rust-lang.org/std/io/struct.Stdin.html) — `read_line`, `lines` and other ways to read input.
- [std::io::IsTerminal](https://doc.rust-lang.org/std/io/trait.IsTerminal.html) — how to tell a terminal from a pipe.
- [std::fs](https://doc.rust-lang.org/std/fs/index.html) — `read_to_string`, `write` and other file functions.
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — why a program commits its `Cargo.lock`.
- [Pro Git — 3.2 Git Branching: Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — branches and merges in depth.
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) — the specification of the commit message style.
- [Clippy documentation](https://doc.rust-lang.org/clippy/) — what Clippy checks and how to configure it.

---

[🏠 Start](../START-HERE.md) · [← 04.10 Error handling](10-error-handling.md) · [📋 Phase 04](README.md) · [04.12 Checkpoint →](12-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
