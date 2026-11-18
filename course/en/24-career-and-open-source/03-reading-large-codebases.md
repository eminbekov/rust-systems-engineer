<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 24.2 Building your portfolio](02-building-your-portfolio.md) · [📋 Phase 24](README.md) · [24.4 Your first open-source contribution →](04-your-first-open-source-contribution.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/03-reading-large-codebases.md) <!-- nav -->

# Lesson 24.3 — Reading large codebases

> **Phase 24 — Career and open source** · Lesson 3 of 9 · ⏱ about 135 minutes

## What you will learn

- Why reading code is a core skill, and why you never read a large project from the first line to the last.
- A six-step method: a question, the documents, a map, the entry points, one traced path, and running it.
- How to practise the method on a real open-source Rust project, ripgrep, with `git grep`, `git log` and `git blame`.
- How to build a quick size map of any Rust codebase with a small program of your own, and test it.

## Before you start

- You finished [Lesson 24.2 — Building your portfolio](02-building-your-portfolio.md).
- You know Cargo workspaces ([Lesson 08.12 — Project layout and workspaces](../08-software-design/12-project-layout-and-workspaces.md))
  and Git history commands ([Lesson 01.4 — Viewing history and changes](../01-git-and-version-control/04-viewing-history-and-changes.md)).
- Your Linux VM `dev` from [Lesson 11.1](../11-linux-shell-and-containers/01-linux-and-a-virtual-machine.md) works.
  You need it only for the optional "build and test" step.

## New words

| Word | Meaning |
|---|---|
| **codebase** | All the source code of one project, together with its tests, scripts and documents. |
| **entry point** | The place where execution starts or where outside code comes in: `main` for a program, the public items of `lib.rs` for a library. |
| **tracing a path** (code reading) | Following one behaviour through the code, from where it starts to where it happens, file by file. Not the same as the logging *tracing* of Lesson 09.10. |
| **code map** | Your own short notes that list the main parts of a codebase, what each part does and where it lives. |
| **heuristic** | A rule that is right most of the time but not always, for example "a file with a zero byte is binary". |
| **NUL byte** | The byte with the value 0 (`b'\x00'`). Normal text almost never contains it; binary files often do. |
| **partial clone** | A Git clone that downloads the history (commits and folder lists) but fetches file contents only when they are needed. It is much smaller than a full clone. |
| **`git grep`** | A Git command that searches the files of a repository. It is fast and skips files that Git ignores. |
| **`git blame`** | A Git command that shows, for every line of a file, the last commit that changed it. |
| **VM** (Virtual Machine) | A whole computer simulated by software. Your Linux VM `dev` from Lesson 11.1 runs inside your Mac. |
| **API** (Application Programming Interface) | The functions and types that a library offers to other code. |
| **I/O** (Input/Output) | Reading and writing data: files, the network, the terminal. |
| **process ID** (ID = identifier) | The number that the operating system gives to each running program (Lesson 13.2). |
| **MIT licence** | A short open-source licence named after the Massachusetts Institute of Technology (Lesson 24.2). |
| **AI** (Artificial Intelligence) | Software that imitates human abilities such as writing text or code. Many projects now have rules about AI-written contributions. |

## 1. Why reading code is a skill

Engineers spend more time **reading** code than writing it. A bug fix often means reading a thousand lines
to change three. A first open-source contribution (Lesson 24.4) starts with understanding code that other
people wrote over years. A new job starts with a codebase much larger than anything from this course.

The mistake every beginner makes is to read a large project like a book: open the first file, read it top
to bottom, then the next. After an hour you remember nothing, because nothing connected the lines to a
purpose.

Experienced engineers read with a **question**. "How does this program decide that a file is binary?" is a
good question: it has an answer, and the answer lives in a small part of the code. You follow the question
through the code, and you ignore everything else. After a few questions, the whole project starts to make
sense.

## 2. The method in six steps

| Step | What you do | Tools |
|---|---|---|
| 1. Ask a question | Write one concrete question in your notes. | your journal |
| 2. Read the documents | README, guides, CONTRIBUTING, architecture notes, the CHANGELOG. | a text editor |
| 3. Draw a map | List the crates or modules, what each does, and how big they are. | `Cargo.toml`, your `loc` program |
| 4. Find the entry points | `main` for a program; the public API of each library crate. | `cargo doc`, RustRover |
| 5. Trace one path | Search for names, jump to definitions, read the tests of that part. | `git grep`, "Go to Definition" |
| 6. Run and change it | Build, run the tests, add a print or a test, watch what changes. | `cargo test`, the VM |

Two habits make the method work. **Write while you read**: every file you understand gets one line in your
code map. And **stop when the question is answered**: you will come back for the next question.

> 💡 **Tip:** In RustRover, "Go to Definition" (Cmd+B or Cmd+click) and "Find Usages" (Option+F7) do in one
> click what `git grep` does in the terminal. Use the terminal method too: it works everywhere, also on a
> server or in a code review page.

## 3. Get a real project: ripgrep

ripgrep (the command `rg`) is a search tool written in Rust. It searches folders for a text pattern and
skips files that Git ignores. It is a good first codebase: medium size, very well documented, split into
small crates, and used by many people.

Clone it next to your course folder, not inside it (a repository inside another repository confuses Git).
A **partial clone** with `--filter=blob:none` keeps the whole history but downloads file contents only when
needed, and `git switch --detach 15.2.0` moves to the release this lesson uses:

```bash
mkdir -p ~/projects/oss
cd ~/projects/oss
git clone --filter=blob:none https://github.com/BurntSushi/ripgrep.git
cd ripgrep
git switch --detach 15.2.0
git log --oneline -1
```

```text
e89fff8 15.2.0
```

A newer release may exist when you read this. Stay on `15.2.0` for the lesson, so that file names and line
numbers match; switch to `master` afterwards with `git switch master`.

> 🔒 **Security:** Building a Rust project runs code from it on your machine: `build.rs` scripts and
> procedural macros run during `cargo build`, and tests run during `cargo test`. Reading files is safe;
> **building and running** an unfamiliar project belongs in your VM `dev` or a container, never as root.

## 4. Read the documents first

Run `ls` in the top folder. Next to `Cargo.toml` and the folders `crates`, `tests`, `fuzz` and `scripts`
you see several Markdown files and three licence files.

The Markdown files are written for you. In this project, `README.md` explains what ripgrep does,
`GUIDE.md` is a user guide, `FAQ.md` answers common questions, and `CHANGELOG.md` lists the changes of every
release. `CONTRIBUTING.md` is very short: it points to `AI_POLICY.md`. That policy allows AI tools for
writing code, but contributors must understand and explain their changes **in their own words**, comments to
maintainers must be written by a human, and fully automatic AI agents may not contribute. More and more
projects have such rules; Lesson 24.4 comes back to them.

`COPYING`, `LICENSE-MIT` and `UNLICENSE` are the licences: ripgrep is offered as "Unlicense OR MIT", the
same idea as the dual licence of Lesson 24.2.

## 5. Draw a map

The top `Cargo.toml` describes the workspace. Two parts matter most. The `[[bin]]` table names the program
and its entry point:

```toml
[[bin]]
bench = false
path = "crates/core/main.rs"
name = "rg"
```

and `[workspace] members = [...]` lists nine folders under `crates/`.

So the program `rg` starts in `crates/core/main.rs`, and the work is split into nine library crates. The
`name = ...` line of each crate's own `Cargo.toml` gives its package name; together with each crate's
`README.md` you get this map:

| Folder | Package | What it does |
|---|---|---|
| `crates/core` | `ripgrep` (the `rg` program) | Command-line flags and the glue that connects the other crates. |
| `crates/ignore` | `ignore` | Walks directories in parallel and applies `.gitignore` rules. |
| `crates/globset` | `globset` | Matches file names against glob patterns such as `*.rs`. |
| `crates/searcher` | `grep-searcher` | Reads files, splits them into lines, detects binary data. |
| `crates/matcher` | `grep-matcher` | The trait that every pattern matcher implements. |
| `crates/regex` / `crates/pcre2` | `grep-regex` / `grep-pcre2` | Two implementations of that trait. |
| `crates/printer` | `grep-printer` | Formats results: normal output, a machine-readable format, summaries. |
| `crates/cli` | `grep-cli` | Helpers for terminals and command-line programs. |
| `crates/grep` | `grep` | One crate that re-exports the `grep-*` crates. |

How big are the parts? Counting lines tells you where most of the logic lives. Add a small program to your
practice package, `code/p24-career/src/bin/loc.rs`:

```rust
// Lists the biggest Rust files under a folder: a quick map of a large codebase.
use std::fs;
use std::path::{Path, PathBuf};

/// Counts lines that are not blank and are not only a comment (`//`, `///` or `//!`).
fn code_lines(text: &str) -> usize {
    text.lines()
        .map(str::trim)
        .filter(|line| !line.is_empty() && !line.starts_with("//"))
        .count()
}

/// Collects every `.rs` file under `dir`. Skips `target` and hidden folders such as `.git`.
fn rust_files(dir: &Path, out: &mut Vec<PathBuf>) -> std::io::Result<()> {
    for entry in fs::read_dir(dir)? {
        let path = entry?.path();
        let name = path.file_name().and_then(|n| n.to_str()).unwrap_or("");
        if path.is_dir() {
            if name != "target" && !name.starts_with('.') {
                rust_files(&path, out)?;
            }
        } else if name.ends_with(".rs") {
            out.push(path);
        }
    }
    Ok(())
}

fn main() {
    let root = std::env::args().nth(1).unwrap_or_else(|| ".".to_string());
    let mut files = Vec::new();
    if let Err(err) = rust_files(Path::new(&root), &mut files) {
        eprintln!("cannot read {root}: {err}");
        std::process::exit(2);
    }

    let mut sizes: Vec<(usize, PathBuf)> = Vec::new();
    for path in files {
        if let Ok(text) = fs::read_to_string(&path) {
            sizes.push((code_lines(&text), path));
        }
    }
    // Biggest first; equal sizes in path order, so the output is stable.
    sizes.sort_by(|a, b| b.0.cmp(&a.0).then(a.1.cmp(&b.1)));

    let total: usize = sizes.iter().map(|(lines, _)| lines).sum();
    println!("{} Rust files, {total} code lines", sizes.len());
    for (lines, path) in sizes.iter().take(10) {
        let shown = path.strip_prefix(&root).unwrap_or(path);
        println!("{lines:>6}  {}", shown.display());
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn blank_and_comment_lines_do_not_count() {
        let text = "//! crate docs\n\n/// docs\nfn main() {\n    // note\n    run();\n}\n";
        assert_eq!(code_lines(text), 3);
    }

    #[test]
    fn finds_rust_files_and_skips_target() {
        let dir = std::env::temp_dir().join(format!("loc-test-{}", std::process::id()));
        fs::create_dir_all(dir.join("src")).unwrap();
        fs::create_dir_all(dir.join("target")).unwrap();
        fs::write(dir.join("src/lib.rs"), "fn a() {}\n").unwrap();
        fs::write(dir.join("target/gen.rs"), "fn b() {}\n").unwrap();
        fs::write(dir.join("notes.txt"), "not code\n").unwrap();

        let mut files = Vec::new();
        rust_files(&dir, &mut files).unwrap();
        fs::remove_dir_all(&dir).unwrap();

        assert_eq!(files, vec![dir.join("src/lib.rs")]);
    }
}
```

**What happens:**

1. `code_lines` trims each line and keeps it only if it is not empty and does not start with `//`. That
   one test removes normal comments, doc comments (`///`) and crate docs (`//!`). It is a rough count, good
   enough for a map: a line with code and a comment still counts as code.
2. `rust_files` is recursive (Lesson 07.7): for a folder it calls itself, for a `.rs` file it stores the
   path. `fs::read_dir` lists one folder; `?` passes any I/O error up to the caller.
3. Folders named `target` (build output) or starting with `.` (such as `.git`) are skipped; they are not
   source code.
4. `main` takes the folder from the first argument (or `.`), counts every file, sorts the biggest first and
   prints the total and the top ten. `strip_prefix` makes the paths shorter by removing the root folder.
5. The second test builds a tiny folder tree in the system temporary folder, with the process ID in its name
   so that parallel test runs do not collide, and deletes it before it checks the result.

Run the tests, then point the program at ripgrep's crates:

```bash
cd ~/projects/rust/code/p24-career
cargo test --bin loc
cargo run -q --bin loc -- ~/projects/oss/ripgrep/crates
```

```text
running 2 tests
test tests::blank_and_comment_lines_do_not_count ... ok
test tests::finds_rust_files_and_skips_target ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
88 Rust files, 34332 code lines
  6703  core/flags/defs.rs
  3220  printer/src/standard.rs
  1817  ignore/src/walk.rs
  1340  globset/src/glob.rs
  1306  searcher/src/searcher/glue.rs
  1184  ignore/src/dir.rs
  1075  core/flags/hiargs.rs
   995  globset/src/lib.rs
   779  matcher/src/lib.rs
   764  printer/src/hyperlink/mod.rs
```

About 34,000 lines of code (plus many comment lines) in 88 files: far too much to read at once, but small
enough to map. The biggest file, `core/flags/defs.rs`, defines all the command-line flags; it is long
because there are many flags, not because it is complicated. Now you can draw the data flow:

```text
 rg (crates/core) ── parses flags ──> HiArgs (the settings for one run)
        │
        ├── ignore::WalkBuilder ──> walks folders, applies .gitignore ──> files
        │
        └── for each file: grep-searcher ──> lines ──> grep-regex (match?) ──> grep-printer ──> terminal
```

> ⚡ **Performance:** The size map also points to the hot code. In a search tool the work per byte happens in
> the searcher and the matcher; a slow flag parser would not matter, a slow line splitter would.

## 6. Trace one question: how does ripgrep detect binary files?

You may have noticed that `rg` usually does not print lines from images or compiled programs. Let's find out
how. Start where the program starts, in `crates/core`, and search for the word "binary" in type names:

```bash
cd ~/projects/oss/ripgrep
git grep -n "BinaryDetection::quit" crates/core
```

```text
crates/core/flags/hiargs.rs:1172:            grep::searcher::BinaryDetection::quit(b'\x00')
crates/core/search.rs:135:    /// `grep::searcher::BinaryDetection::quit` if we want to skip binary files
crates/core/search.rs:151:    /// `grep::searcher::BinaryDetection::quit`, since we never want to
```

Open `crates/core/flags/hiargs.rs` around line 1160. The function `BinaryDetection::from_low_args` chooses two
settings: an **explicit** one for files that you name on the command line, and an **implicit** one for files
that `rg` finds while walking folders. By default the implicit one is `quit(b'\x00')` and the explicit one is
`convert(b'\x00')`. The doc comments in `crates/core/search.rs` (lines 130–160) explain why: files found by
walking may be skipped when they look binary, but a file that you named yourself is never skipped silently.

Now go one level down, into the library that does the work:

```bash
git grep -n "pub fn quit" crates/searcher
git grep -n "BinaryDetection::Quit(byte)" crates/searcher
```

```text
crates/searcher/src/searcher/mod.rs:79:    pub fn quit(binary_byte: u8) -> BinaryDetection {
crates/searcher/src/searcher/mod.rs:102:    pub fn quit_byte(&self) -> Option<u8> {
crates/searcher/src/line_buffer.rs:437:                BinaryDetection::Quit(byte) => {
```

The long doc comment above line 79 of `searcher/mod.rs` describes the **heuristic**: data that contains the
given byte is treated as binary. In `line_buffer.rs` near line 437 you find the place where it happens: every
time the buffer is filled from the file, the new bytes are searched for that byte; if it is found, the
buffer ends just before it and the search stops as if the file had ended there.

Finally, read a unit test of this code; tests are the most precise documentation. Here is one from the same
file (around line 795):

```rust
    #[test]
    fn buffer_binary_quit1() {
        let bytes = "homer\nli\x00sa\nmaggie\n";
        let mut linebuf = LineBufferBuilder::new()
            .binary_detection(BinaryDetection::Quit(b'\x00'))
            .build();
        let mut rdr = LineBufferReader::new(bytes.as_bytes(), &mut linebuf);

        assert!(rdr.buffer().is_empty());

        assert!(rdr.fill().unwrap());
        assert_eq!(rdr.bstr(), "homer\nli");
        rdr.consume_all();

        assert!(!rdr.fill().unwrap());
        assert_eq!(rdr.absolute_byte_offset(), 8);
        assert_eq!(rdr.binary_byte_offset(), Some(8));
    }
```

The input has a NUL byte after `li`. The test shows that the reader returns only `"homer\nli"` and reports
the binary byte at offset 8. **Answer to the question:** ripgrep treats data as binary when it contains a NUL
byte. For files found while walking folders it stops reading at the first NUL; for files you name yourself it
replaces NUL bytes and keeps searching. Write this in your code map, with the four file names.

## 7. Ask the history why

Code tells you *what* happens; history tells you *why*. Because the partial clone has the whole history, you
can ask Git about any file: `git log --oneline -- crates/searcher/src/line_buffer.rs` lists the commits that
changed it, newest first.

`git blame` answers the question "which commit last changed this line?". The option `-s` hides author names
and dates, and `-L 6,6` limits the output to line 6:

```bash
git blame -s -L 6,6 crates/searcher/src/line_buffer.rs
```

```text
1c3eebef 6) pub(crate) const DEFAULT_BUFFER_CAPACITY: usize = 64 * (1 << 10); // 64 KB
```

`git show 1c3eebef` then prints the commit message and the change. Good projects write messages that
explain the reason, and many commit messages link to an issue with the full discussion.

## 8. Run it and change it (in the VM)

Reading gives you a theory; running the code checks it. In your VM `dev` (or a Linux container), clone the
same tag and run the tests of the one crate you studied. Test names are filters, so this runs only the
binary-detection tests of the line buffer:

```bash
cargo test -p grep-searcher buffer_binary_quit
```

Example output (yours will look similar; the first build downloads and compiles the dependencies):

```text
running 5 tests
test line_buffer::tests::buffer_binary_quit1 ... ok
test line_buffer::tests::buffer_binary_quit2 ... ok
test line_buffer::tests::buffer_binary_quit3 ... ok
test line_buffer::tests::buffer_binary_quit4 ... ok
test line_buffer::tests::buffer_binary_quit5 ... ok
```

Then **change something on purpose** in your local copy: replace `b'\x00'` in `buffer_binary_quit1` with
`b'\x01'`, run the test again and read how it fails. Undo the change with `git restore .` when you are done.
This is how you confirm that you really understood a piece of code: you predict what a change will do,
then you check.

> 🧪 **Testing:** A failing test that you broke on purpose teaches you two things at once: what the code
> does, and how the project's test output looks. You will need both when you send your first pull request.

## Common mistakes

**Reading without a question.** Opening `main.rs` and reading "everything" leads nowhere. Write the question
first; stop when it is answered.

**Trying to understand every line.** You only need the lines on your path. Mark the rest "later" in your
code map.

**A shallow clone when you need history.** `git clone --depth 1` downloads only the newest commit. Then
`git log` shows one commit and `git blame` blames everything on it. Use `--filter=blob:none` instead; it is
small and keeps the history.

**Building an unknown project on your Mac as your normal user without thinking.** Build scripts and tests
run real code. Use the VM for anything you have not read.

## Try it yourself

1. Create `notes/code-map-ripgrep.md` with the map table from Section 5 in your own words.
2. Add the answer to the binary question (Section 6), with the four files and line numbers.
3. Pick a new question, for example "Where does ripgrep read `.gitignore` files?", and answer it with the
   method. Hint: start with `git grep -n "gitignore" crates/ignore/src | head`.
4. In the VM, run the tests of the crate you studied and break one test on purpose, as in Section 8.

## Exercises

### Exercise 1 — Size per crate (medium)

Print, after the top files, the number of code lines per top-level folder (for `ripgrep/crates` that is per
crate), sorted by name. Write `top_folder(root: &Path, path: &Path) -> String` and a test for it first. A
file directly inside `root` belongs to the folder `"."`.

<details>
<summary>Hint</summary>

`path.strip_prefix(root)` gives the relative path; `.components()` splits it into parts. If there are at
least two parts, the first one is the folder. A `BTreeMap<String, usize>` keeps the folders sorted.

</details>

<details>
<summary>Answer</summary>

```rust
/// Returns the first folder of `path` below `root`, or "." for files directly in `root`.
fn top_folder(root: &Path, path: &Path) -> String {
    let relative = path.strip_prefix(root).unwrap_or(path);
    let mut parts = relative.components();
    match (parts.next(), parts.next()) {
        (Some(first), Some(_)) => first.as_os_str().to_string_lossy().into_owned(),
        _ => ".".to_string(),
    }
}

#[test]
fn top_folder_is_the_first_part_below_root() {
    let root = Path::new("/r");
    assert_eq!(top_folder(root, Path::new("/r/searcher/src/lib.rs")), "searcher");
    assert_eq!(top_folder(root, Path::new("/r/build.rs")), ".");
}
```

At the end of `main`:

```rust
    let mut per_folder: std::collections::BTreeMap<String, usize> = Default::default();
    for (lines, path) in &sizes {
        *per_folder
            .entry(top_folder(Path::new(&root), path))
            .or_default() += lines;
    }
    println!("per folder:");
    for (folder, lines) in per_folder {
        println!("{lines:>6}  {folder}");
    }
```

With `~/projects/oss/ripgrep/crates`, the new part of the output is:

```text
per folder:
  1121  cli
 10352  core
  2656  globset
    81  grep
  5339  ignore
  1329  matcher
   320  pcre2
  6631  printer
  1865  regex
  4638  searcher
```

Why it works: `entry(...).or_default()` inserts 0 for a new folder and returns a mutable reference to the
count. The numbers show the shape of the project: the flags and glue (`core`), the printer, the directory
walker and the searcher are the big parts; `grep` is only a facade.

</details>

### Exercise 2 — Hidden files (medium, reading)

By default `rg` does not search hidden files. On macOS and Linux, how does ripgrep decide that a file is
hidden, and where is that code? Use the method; write down the commands you ran.

<details>
<summary>Hint</summary>

The directory walker lives in `crates/ignore`. Search for a function whose name starts with `is_hidden`.

</details>

<details>
<summary>Answer</summary>

```bash
git grep -n "fn is_hidden" crates/ignore/src
```

```text
crates/ignore/src/pathutil.rs:11:pub(crate) fn is_hidden(dent: &DirEntry) -> bool {
crates/ignore/src/pathutil.rs:28:pub(crate) fn is_hidden(dent: &DirEntry) -> bool {
crates/ignore/src/pathutil.rs:51:pub(crate) fn is_hidden(dent: &DirEntry) -> bool {
```

There are three versions, chosen at compile time with `#[cfg(unix)]`, `#[cfg(windows)]` and
`#[cfg(not(any(unix, windows)))]`. The Unix version (macOS and Linux) returns true when the file name starts
with a `.`. The Windows version also checks the "hidden" file attribute of the file system.

Why it works: `#[cfg(...)]` is the same attribute as `#[cfg(test)]` from Lesson 02.8. `#[cfg(unix)]` keeps an
item only when the program is built for a Unix-like system, so each build contains exactly one of the three
functions, and the same name can have a different body per operating system.

</details>

### Exercise 3 — Map your own capstone (easy)

Run `loc` on your capstone project (or any project of the course) and write a three-line code map: the
biggest part, the part with the most important logic, and one part you would read first as a newcomer.

<details>
<summary>Hint</summary>

The biggest file is not always the most important one; look at the names and at what `main` calls.

</details>

<details>
<summary>Answer</summary>

There is no single answer. A good map names files or modules, says in a few words what each does, and gives
a reason, for example: "`src/wal.rs` is the most important: every write goes through it first."

</details>

## Quiz

1. Why do you start reading a codebase with a question?

<details>
<summary>Answer</summary>

A question gives you a path through the code and a point where you can stop. Without it you read lines that
do not connect to anything and forget them.

</details>

2. What is the difference between `git clone --depth 1` and `git clone --filter=blob:none`?

<details>
<summary>Answer</summary>

`--depth 1` downloads only the newest commit, so there is no history. `--filter=blob:none` downloads all
commits and folder lists but fetches file contents on demand, so `git log` and `git blame` work and the clone
stays small.

</details>

3. Why should you build and test an unfamiliar project in a VM or a container?

<details>
<summary>Answer</summary>

`cargo build` runs build scripts and procedural macros, and `cargo test` runs test code. All of it is code
from people you do not know, running with your user's rights.

</details>

4. Why are unit tests good documentation when you read code?

<details>
<summary>Answer</summary>

They show exact inputs and the exact expected outputs, and they are checked on every build, so they cannot
silently become wrong the way comments can.

</details>

## Summary

- Read large codebases with a question, and stop when it is answered.
- The method: question, documents, map, entry points, trace one path, run and change.
- `Cargo.toml` workspace members and a size count (`loc`) give the first map; each crate's README explains its
  part.
- `git grep` finds names, RustRover jumps to definitions, unit tests show exact behaviour.
- `git log -- <file>` and `git blame` explain why code looks the way it does; use a partial clone to keep
  history.
- Build and run unfamiliar code only in a VM or a container, and keep your experiments local.

## Sources and further reading

- [ripgrep on GitHub](https://github.com/BurntSushi/ripgrep) — the project used in this lesson, with its
  user guide, answers to common questions and crate READMEs.
- [Git — git grep](https://git-scm.com/docs/git-grep), [git blame](https://git-scm.com/docs/git-blame) and
  [git clone](https://git-scm.com/docs/git-clone) — the reference pages, including `--filter` for partial clones.
- [The Cargo Book — Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) — how workspace
  members are declared.
- [The Cargo Book — cargo doc](https://doc.rust-lang.org/cargo/commands/cargo-doc.html) — build the API
  documentation of any project locally.
- [RustRover — Source code navigation](https://www.jetbrains.com/help/rust/navigating-through-the-source-code.html) —
  "Go to Definition", "Find Usages" and other navigation tools.
- [Rust Compiler Development Guide](https://rustc-dev-guide.rust-lang.org/) — a guide to one of the largest
  Rust codebases, and a good example of how big projects document themselves.

---

[🏠 Start](../START-HERE.md) · [← 24.2 Building your portfolio](02-building-your-portfolio.md) · [📋 Phase 24](README.md) · [24.4 Your first open-source contribution →](04-your-first-open-source-contribution.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
