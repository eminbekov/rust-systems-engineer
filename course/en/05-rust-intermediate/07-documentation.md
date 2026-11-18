<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.6 Testing in depth](06-testing-in-depth.md) · [📋 Phase 05](README.md) · [05.8 Cargo: dependencies and versions →](08-cargo-dependencies-and-versions.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/07-documentation.md) <!-- nav -->

# Lesson 05.7 — Documentation

> **Phase 05 — Rust intermediate** · Lesson 7 of 14 · ⏱ about 3 hours (2 sittings)

## What you will learn

- The three layers of documentation: comments for maintainers, doc comments for users, and project files (README, CHANGELOG).
- How to write `///` and `//!` doc comments with Markdown and the standard sections (`# Examples`, `# Errors`, `# Panics`).
- How doc tests turn your examples into tests that cannot go out of date.
- How to generate and read HTML (HyperText Markup Language, the language of web pages) documentation with `cargo doc`, and how to catch missing docs and broken links.
- How to document a project in Git: a README, a changelog, version tags, and how `git log` and `git blame` help.

## Before you start

- You finished [Lesson 05.6 — Testing in depth](06-testing-in-depth.md) and have the library package `code/p05-textkit` with `slugify` and `parse_duration`.
- You know modules and `pub` ([Lesson 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md)), `Result` ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) and Markdown ([Lesson 00.8](../00-getting-started/08-study-routine-notes-and-checkpoint.md)).
- Git: commits, branches and the remote ([Phase 01](../01-git-and-version-control/03-your-first-repository-and-commits.md)).

## New words

| Word | Meaning |
|---|---|
| **documentation** | Text that explains what code is for and how to use it. |
| **doc comment** | A comment that becomes part of the generated documentation. It starts with `///` (for the next item) or `//!` (for the enclosing item). |
| **rustdoc** | The tool, shipped with Rust, that turns doc comments into HTML pages. `cargo doc` runs it. |
| **doc test** | A Rust code block inside a doc comment. `cargo test` compiles and runs it. |
| **intra-doc link** | A link written as ``[`Name`]`` that points to another item in your code; rustdoc checks that it exists. |
| **README** | The first file people read in a project: what it is, how to use it, how to build it. |
| **changelog** | A file that lists what changed in each version, for users. |
| **SemVer** (Semantic Versioning) | The versioning rule `MAJOR.MINOR.PATCH`: bump MAJOR for breaking changes, MINOR for new features, PATCH for bug fixes. |
| **Git tag** | A permanent name for one commit, typically a release such as `v0.1.0`. |
| **public API** (Application Programming Interface) | The items marked `pub` that other code may use. It is what you must document. |
| **docs.rs** | The website that builds and hosts the documentation of every crate published on crates.io. |

## 1. Three layers of documentation

Code answers the question "how does this work?" for the computer. People need answers to other questions,
and each answer has its own place:

| Layer | Reader | Question | Where |
|---|---|---|---|
| comments `//` | the next developer who **changes** the code | *Why* is it done this way? | inside function bodies |
| doc comments `///` and `//!` | the developer who **uses** the code | What does it do, how do I call it, what can go wrong? | above `pub` items, at the top of the crate |
| project files | everyone, incl. users who never read code | What is this? How do I install it? What changed? | `README.md`, `CHANGELOG.md`, Git history |

The more of these you write, the less you depend on remembering things. In six months you will be a stranger
to your own code.

## 2. Doc comments

A line that starts with three slashes `///` is a **doc comment**. It documents the item **right below it**.
The text is Markdown (Lesson 00.8): `` `code` ``, lists, bold, links and so on.

```rust
/// Turns a title into lowercase, dash-separated text that is safe to use in a URL.
///
/// Letters and digits are kept (letters become lowercase). Every run of other
/// characters becomes a single dash, and dashes at the start or the end are removed.
///
/// # Examples
///
/// ```
/// use p05_textkit::slugify;
///
/// assert_eq!(slugify("Hello World"), "hello-world");
/// assert_eq!(slugify("  Rust -- is... fun!  "), "rust-is-fun");
/// assert_eq!(slugify("!!!"), "");
/// ```
pub fn slugify(input: &str) -> String {
    // (body as in Lesson 05.6)
```

**What happens:**

1. The **first line** is a one-sentence summary. It appears alone in lists of functions, so make it a complete
   sentence that starts with a verb ("Turns…", "Returns…", "Parses…") and ends with a period.
2. After an empty `///` line comes the longer explanation: rules, special cases, units.
3. A heading `# Examples` introduces a code block. Rust Markdown headings are written with `#`, and **the
   standard section names are a convention that every Rust reader knows** (table below).
4. The fenced block with three backticks and no language is compiled as Rust **and run**: more in Section 3.

### Standard sections

| Section | Write it when | Content |
|---|---|---|
| `# Examples` | always, for every public function | at least one short, runnable example |
| `# Errors` | the function returns `Result` | when and which errors are returned |
| `# Panics` | the function can panic | the exact conditions |
| `# Safety` | the function is `unsafe` (Lesson 06.6) | what the caller must guarantee |

A function that returns `Result` documents its errors:

```rust
/// Reads a duration such as `"1h30m15s"` and returns the number of seconds.
///
/// The text is a list of parts. Each part is a whole number followed by one of
/// the units `h` (hours), `m` (minutes) or `s` (seconds). Spaces around the
/// whole text are ignored.
///
/// # Errors
///
/// Returns a [`ParseError`] if the text is empty, if a part has no number or no
/// unit, if the unit is unknown, or if the total does not fit into a `u64`.
///
/// # Examples
///
/// ```
/// use p05_textkit::{parse_duration, ParseError};
///
/// assert_eq!(parse_duration("1h30m15s"), Ok(5415));
/// assert_eq!(parse_duration("5x"), Err(ParseError::UnknownUnit('x')));
/// ```
pub fn parse_duration(input: &str) -> Result<u64, ParseError> {
```

and a function that can panic documents it (and tests it):

```rust
/// Returns `part` as a whole-number percentage of `whole`, rounded down.
///
/// # Panics
///
/// Panics if `whole` is zero, because a percentage of nothing is undefined.
///
/// # Examples
///
/// ```
/// assert_eq!(p05_textkit::percent(1, 4), 25);
/// ```
///
/// The panic is part of the contract, so it is tested too:
///
/// ```should_panic
/// p05_textkit::percent(1, 0);
/// ```
pub fn percent(part: u32, whole: u32) -> u32 {
    assert!(whole != 0, "whole must not be zero");
    ((part as u64 * 100) / whole as u64) as u32
}
```

**What happens:** `assert!(cond, "message")` panics if `cond` is false (Lesson 05.6). The `# Panics` section makes this behaviour a
documented promise. The `u64` conversions prevent overflow when computing `part * 100` for large `part`.

Also document the types and every enum variant, because the compiler will warn you if you do not (Section 4):

```rust
/// The ways in which [`parse_duration`] can fail.
#[derive(Debug, PartialEq)]
pub enum ParseError {
    /// The text is empty or contains only spaces.
    Empty,
    /// A unit letter has no number in front of it, as in `"h"`.
    MissingNumber,
    /// The text ends with digits that have no unit, as in `"10"`.
    MissingUnit,
    /// The unit letter is not one of `h`, `m` or `s`.
    UnknownUnit(char),
    /// The result does not fit into a `u64`.
    TooLarge,
}
```

### `//!` documents the container

`//!` documents the thing that **contains** the comment, not the next item. It is used at the very top of `src/lib.rs`
to describe the whole crate (and the same way at the top of a module file):

```rust
//! # p05-textkit
//!
//! Small helpers for turning titles into URL-friendly text and for reading
//! durations such as `1h30m`.
//!
//! ```
//! use p05_textkit::{parse_duration, slugify};
//!
//! assert_eq!(slugify("Hello World"), "hello-world");
//! assert_eq!(parse_duration("1m30s"), Ok(90));
//! ```

#![warn(missing_docs)]
```

This is the front page of your documentation: what the crate is for, and a first example. `#![warn(missing_docs)]` (with the `!`) is an
attribute for the whole crate: it asks the compiler to warn about public items without documentation.

> 📏 **Convention:** Write documentation for **users**: they know Rust but not your code. Say what a function *does*, not how it is
> written inside. Describe the units (seconds? bytes?), the ranges and the special cases (empty input, zero). Avoid words like
> "simply" or "obviously"; what is obvious to you is not to them.

## 3. Doc tests: examples that cannot go stale

Documentation with code examples has an old problem: the code changes and the examples silently stop working.
Rust solves this: **every Rust code block in a doc comment is a test**. `cargo test` compiles and runs it, in the section
named `Doc-tests`:

```text
   Doc-tests p05_textkit

running 6 tests
test src/lib.rs - (line 6) ... ok
test src/lib.rs - parse_duration (line 87) ... ok
test src/lib.rs - percent (line 134) ... ok
test src/lib.rs - parse_duration (line 77) ... ok
test src/lib.rs - slugify (line 22) ... ok
test src/lib.rs - percent (line 140) - should panic ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. Each code block becomes a small program. Rustdoc wraps your lines in a `fn main() { … }`, compiles it as an
   *external* user of your crate (so you need the `use p05_textkit::…` line, as in Lesson 05.6's integration tests), and runs it.
2. An `assert_eq!` that fails, or a panic, fails the doc test. The names show where each test is: the file, the item and the line
   of the code block.
3. Run only doc tests with `cargo test --doc`.

If an example is wrong, the failure points at it:

```rust
/// Doubles a number.
///
/// ```
/// assert_eq!(d2::double(2), 5);
/// ```
pub fn double(x: i32) -> i32 {
    x * 2
}
```

```text
---- src/lib.rs - double (line 6) stdout ----
Test executable failed (exit status: 101).

stderr:

thread 'main' panicked at .../doctest_bundle_2024.rs:6:1:
assertion `left == right` failed
  left: 4
 right: 5
```

**What it means:** the documentation promised `5`, and the code gives `4`. The doc test caught the lie before a user did. (Here the
example is wrong; in real life, when behaviour changes, a failing doc test reminds you to update the docs.)

### Hiding lines and using `?`

Lines that start with `# ` (a hash and a space) are compiled but **not shown** in the rendered documentation.
They hide boilerplate so that the example stays readable. A typical use is a `main` function that returns `Result`, which allows `?`:

```rust
/// ```
/// # fn main() -> Result<(), p05_textkit::ParseError> {
/// let seconds = p05_textkit::parse_duration("2m")?;
/// assert_eq!(seconds, 120);
/// # Ok(())
/// # }
/// ```
```

The reader of the HTML page sees only the two middle lines; the compiler sees a complete program. (If the example has no `fn main`
of its own, rustdoc supplies one.)

### Code block attributes

You can put a word after the opening backticks to change what happens with the block:

| Marker | Meaning |
|---|---|
| ` ```no_run ` | compile it, but do not run it (for code that starts a server, reads the network, runs forever) |
| ` ```ignore ` | skip it completely; shown as `ignored`. Use rarely, because nothing checks it |
| ` ```should_panic ` | the example must panic (as in `percent(1, 0)` above) |
| ` ```compile_fail ` | the example must **fail to compile** (to show what the compiler rejects) |
| ` ```text ` | not Rust at all: plain text, an output, a shell command |

```text
test src/lib.rs - serve_forever (line 17) - compile ... ok          <- no_run
test src/lib.rs - serve_forever (line 21) ... ignored                <- ignore
test src/lib.rs - serve_forever (line 25) - compile fail ... ok     <- compile_fail
```

> 🧪 **Testing:** A doc test is a **test of your public API from the user's point of view**, like an integration test (Lesson
> 05.6), but it is also the example the user reads. Keep it short, show the most common use first, and do not repeat
> every edge case here: edge cases belong in normal unit tests.

## 4. `cargo doc`: generating and checking

`cargo doc` builds the HTML documentation of your crate **and of all its dependencies** into `target/doc/`. The most useful form is:

```bash
cargo doc --no-deps --open
```

- `--no-deps`: document only your crate (much faster).
- `--open`: open the result in your browser. The start page is `target/doc/p05_textkit/index.html`.

```text
 Documenting p05-textkit v0.1.0 (...)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
   Generated .../target/doc/p05_textkit/index.html
```

You will see your `//!` text on the front page, then a list of functions and types, each with its summary line; the
detail pages show the full text, the sections and the examples (with a "Run" button). The search box at the top works offline.
This is the same site that docs.rs shows for crates on crates.io. The standard library's documentation at
doc.rust-lang.org is built by the same tool, from the same kind of comments.

> 💡 **Tip:** `cargo doc --open` is a good habit before you consider a library "done": read your own
> documentation as a stranger would. In RustRover you can also hover over a name, or use *View → Quick Documentation*,
> to see its doc comment without leaving the editor.

### Intra-doc links

Link to other items by putting their name in square brackets and backticks, as in ``[`ParseError`]``. Rustdoc finds the item, turns it into a link,
and **warns if the item does not exist**. You can use paths too: ``[`crate::slugify`]``, ``[`Vec::push`]``.

### Catching missing docs and broken links

With `#![warn(missing_docs)]` the compiler warns about every public item without a doc comment, and rustdoc warns
about links that lead nowhere. Here is a small crate with both problems:

```rust
#![warn(missing_docs)]
//! A tiny crate.

/// Doubles a number. See also [`triple_it`] and [`Missing`].
pub fn double(x: i32) -> i32 {
    x * 2
}

pub fn undocumented() {}
```

```text
warning: missing documentation for a function
  --> src/lib.rs:13:1
   |
13 | pub fn undocumented() {}
   | ^^^^^^^^^^^^^^^^^^^^^
   |
note: the lint level is defined here
  --> src/lib.rs:1:9
   |
 1 | #![warn(missing_docs)]
   |         ^^^^^^^^^^^^

warning: unresolved link to `triple_it`
 --> src/lib.rs:4:34
  |
4 | /// Doubles a number. See also [`triple_it`] and [`Missing`].
  |                                  ^^^^^^^^^ no item named `triple_it` in scope
  |
  = note: `#[warn(rustdoc::broken_intra_doc_links)]` on by default
```

**What it means:** the first warning lists a public function nobody documented; the others show names that do not exist
(maybe renamed or misspelled). Fix them and run `cargo doc` until it is silent. Later, a CI job can run
`RUSTDOCFLAGS="-D warnings" cargo doc --no-deps` to make such warnings fail the build.

## 5. Comments inside the code: explain *why*

Doc comments are for users. Ordinary comments `//` are for the person who will **change** the code. The rule:
**do not repeat what the code says; explain what it cannot say.**

```rust
// Bad: says what the code obviously does.
// add one to counter
counter += 1;

// Bad: the comment has become false.
// retry three times
for _ in 0..5 { /* ... */ }

// Good: says WHY.
// The server closes idle connections after 30 s, so we retry once with a fresh one.
```

Guidelines:

- Prefer **clear names** to comments. If you need a comment to explain `let d = 86400;`, name it `let seconds_per_day = 86_400;`.
- Explain *why*: the reason for a surprising choice, the bug being worked around, the rule from a specification, the units.
- Mark unfinished work with `TODO:` or `FIXME:` plus a reason. Rust's `todo!()` macro is the compiled version of the same idea (Lesson 05.6).
- **Do not leave commented-out code.** Git remembers every old version (Phase 01): delete the code and commit.
- 🔒 **Security:** never put passwords, tokens or private addresses in comments or doc examples: documentation is published, and Git history is permanent.

## 6. Project documentation, and Git as documentation

A project is more than its code. Three files and the Git history complete the picture.

### README.md

The README is the first thing a visitor reads (GitHub and GitLab show it on the project's front page). A minimal, useful README has:

1. the name and **one sentence** about what the project is for;
2. a short usage example;
3. how to build, test and run it;
4. the licence and where to ask questions (when the project is public).

```markdown
# p05-textkit

Small helpers for turning titles into URL-friendly text and for reading
durations such as `1h30m`.

## Usage

(a short code example)

## Development

(cargo fmt --check && cargo clippy -- -D warnings && cargo test)
```

### CHANGELOG.md and SemVer

A changelog tells **users** what changed between versions, in human words (not a dump of commits). The common format
is *Keep a Changelog*: newest version first, an `Unreleased` section at the top, and groups `Added`, `Changed`, `Fixed`, `Removed`.

```markdown
# Changelog

## [Unreleased]

### Added
- `percent`: a whole-number percentage.

## [0.1.0] - 2026-11-17

### Added
- `slugify`: turn a title into URL-friendly text.
- `parse_duration`: read durations such as `1h30m15s`.
```

Version numbers follow **Semantic Versioning** `MAJOR.MINOR.PATCH`: `0.1.0` → `0.1.1` for a bug fix, `0.2.0` for new features,
`1.0.0` when you promise the API is stable. Before `1.0` anything may change; after it, a *breaking change* (removing or changing a public
function) requires a new MAJOR number. Lesson 05.8 explains how Cargo uses these numbers to choose dependency versions.

### Git: tags, `git log` and `git blame`

When you publish a version, mark the exact commit with an annotated **tag**:

```bash
git tag -a v0.1.0 -m "First release of p05-textkit"
git tag
git show v0.1.0 --stat
```

```text
v0.1.0
tag v0.1.0
Tagger: Student <s@example.com>
Date:   Tue Nov 17 16:51:03 2026 +0500

First release of p05-textkit
```

The `-a` makes an *annotated* tag (it stores who made it, when, and a message). `git tag` lists tags; `git show` shows the tag and the commit. To send tags to
the remote you need `git push --tags` (Lesson 01.8). Now Git can answer "what changed since the release?" — the raw material for the next changelog entry:

```bash
git log --oneline v0.1.0..HEAD
git log --oneline --decorate | head -4
```

```text
903adfc docs(p05): explain panics in percent
48ab9f0 feat(p05): add percent
903adfc (HEAD -> main) docs(p05): explain panics in percent
48ab9f0 feat(p05): add percent
ab9558f (tag: v0.1.0) feat(p05): slugify and parse_duration with documentation
```

`v0.1.0..HEAD` means "the commits after `v0.1.0`, up to now". Because the messages follow the Conventional Commits format (Lesson 05.2:
`feat`, `fix`, `docs`…), you can write the changelog from them: every `feat:` is an *Added*, every `fix:` is a *Fixed*.

**Commit messages are documentation too.** A good message explains *why* the change was made, in a first line of up to about 70
characters, then (after an empty line) the details. Six months from now `git log` and `git blame` are the only
explanation of why a line exists. `git blame` shows who last changed each line, and in which commit:

```bash
git blame -L 1,3 code/p05-textkit/README.md
```

```text
^ab9558f (Student 2026-11-17 16:51:03 +0500 1) # p05-textkit
^ab9558f (Student 2026-11-17 16:51:03 +0500 2) 
^ab9558f (Student 2026-11-17 16:51:03 +0500 3) Small helpers for turning titles into URL-f
```

(`-L 1,3` limits it to lines 1 to 3. The commit hash at the start of the line leads you to `git show <hash>`, which shows the full message
of the commit and its change. Despite the name, `blame` is for understanding, not for finding someone to scold.)

## Common mistakes

**1. Documenting what instead of why or how to use.** "Increments the counter" for `fn increment()` helps nobody. Say when to call it and what
it guarantees.

**2. Examples that are never run.** Using ` ```ignore ` everywhere defeats the purpose. Prefer ` ```no_run ` for code that cannot run in tests.

**3. Forgetting the `use` in a doc test.** The test is an external user of your crate, so `slugify("x")` alone fails with `cannot find function`.
Write `use p05_textkit::slugify;` (or hide it with `# use …;`).

**4. Using `?` in a doc test without a `main` that returns `Result`.** Add the hidden `# fn main() -> Result<(), …> { … # Ok(()) # }` lines.

**5. Broken intra-doc links** after renaming an item. `cargo doc` warns: read the warnings.

**6. Comments that lie.** A wrong comment is worse than none. When you change code, change its comments in the same commit.

**7. A README that only the author can follow.** Test it: ask a friend (or yourself, in a fresh folder) to follow it exactly.

## Try it yourself

On a new branch, document `code/p05-textkit` fully and release version 0.1.0.

1. Create the branch:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-07-docs
   ```

2. In `code/p05-textkit/src/lib.rs`, add the crate-level `//!` text with `#![warn(missing_docs)]`, a `///` comment with `# Examples` for
   `slugify`, `# Errors` and `# Examples` for `parse_duration`, documentation for every variant of `ParseError`, and the `percent` function with
   `# Panics`. Write each as shown in Section 2.

3. Run the checks until all are clean:

   ```bash
   cd code/p05-textkit
   cargo test --doc
   cargo doc --no-deps --open
   cargo clippy -- -D warnings
   ```

   Your browser opens the documentation. Click through: the front page, `slugify`, `ParseError` with its variants, and the "Run" button on an
   example. Then **break something on purpose**: change an expected value in an example, run `cargo test --doc`, and read the failure; add an
   undocumented `pub fn` and read the `missing_docs` warning; fix both.

4. Commit the code, then add `README.md` and `CHANGELOG.md` in `code/p05-textkit` as in Section 6 and commit them separately:

   ```bash
   cd ~/projects/rust
   git add code/p05-textkit/src
   git commit -m "docs(p05): document the public API of textkit"
   git add code/p05-textkit/README.md code/p05-textkit/CHANGELOG.md
   git commit -m "docs(p05): add README and CHANGELOG"
   ```

5. Merge into `main` and tag the release (a tag marks the commit that is now `HEAD` on `main`):

   ```bash
   git switch main
   git merge lesson/05-07-docs
   git tag -a v0.1.0 -m "First release of p05-textkit"
   git tag
   ```

6. Make one more change (for example the `percent` function), then see what has happened since the release:
   `git log --oneline v0.1.0..HEAD`. Put that change under `## [Unreleased]` in `CHANGELOG.md` and commit it.

If you use GitHub or GitLab, push the tag too: `git push --tags`. Platforms show tags as releases.

## Exercises

### Exercise 1 — Document a function (easy)

Write a full doc comment (one-line summary, a short explanation, `# Examples` with three assertions) for this function, and check it with
`cargo test --doc`. Replace `my_crate` by the name of your crate (with underscores).

```rust
pub fn clamp_percent(value: i32) -> u8 {
    value.clamp(0, 100) as u8
}
```

<details>
<summary>Hint</summary>

Start with a verb ("Limits…"). Explain what happens below 0 and above 100. In the example, `use my_crate::clamp_percent;` and assert a value inside
the range, one below and one above.

</details>

<details>
<summary>Answer</summary>

```rust
/// Limits a number to the range 0 to 100 and returns it as a percentage.
///
/// Values below 0 become 0, and values above 100 become 100.
///
/// # Examples
///
/// ```
/// use my_crate::clamp_percent;
///
/// assert_eq!(clamp_percent(42), 42);
/// assert_eq!(clamp_percent(-5), 0);
/// assert_eq!(clamp_percent(150), 100);
/// ```
pub fn clamp_percent(value: i32) -> u8 {
    value.clamp(0, 100) as u8
}
```

`cargo test --doc` reports one passing doc test.

Why it works: the summary tells what the function is for, the second line states the special cases, and the three assertions show the normal case and both borders.

</details>

### Exercise 2 — Fix a broken doc test (easy)

This doc test fails. Run it, read the failure, and decide whether the **example** or the **code** is wrong. Fix it.

```rust
/// Doubles a number.
///
/// ```
/// assert_eq!(my_crate::double(2), 5);
/// ```
pub fn double(x: i32) -> i32 {
    x * 2
}
```

<details>
<summary>Hint</summary>

The failure shows `left: 4` and `right: 5`. `left` is the real result of the code. Which is correct: the function name and body, or the number 5 in the example?

</details>

<details>
<summary>Answer</summary>

The code is right (doubling 2 gives 4); the example has a typo:

```rust
/// Doubles a number.
///
/// ```
/// assert_eq!(my_crate::double(2), 4);
/// ```
pub fn double(x: i32) -> i32 {
    x * 2
}
```

Why it works: a doc test fails whenever the example and the code disagree; you must decide which one is the truth. If you were not sure, the function's
name, its other tests and the documentation text decide.

</details>

### Exercise 3 — `# Errors` and `# Panics` (medium)

Write documentation, including the right standard section and a doc test that proves it, for both functions:

```rust
pub fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

pub fn item_at(list: &[i32], index: usize) -> i32 {
    list[index]
}
```

<details>
<summary>Hint</summary>

`divide` returns `Result`, so it needs `# Errors`, and the example can check `divide(7, 2) == Ok(3)` and `divide(1, 0).is_err()`. `item_at` panics when
`index >= list.len()`, so it needs `# Panics`, one normal example, and a second code block marked `should_panic`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Divides `a` by `b` and rounds toward zero.
///
/// # Errors
///
/// Returns an error message if `b` is zero.
///
/// # Examples
///
/// ```
/// use my_crate::divide;
///
/// assert_eq!(divide(7, 2), Ok(3));
/// assert!(divide(1, 0).is_err());
/// ```
pub fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

/// Returns the item at `index`.
///
/// # Panics
///
/// Panics if `index` is not smaller than `list.len()`.
///
/// # Examples
///
/// ```
/// assert_eq!(my_crate::item_at(&[10, 20, 30], 1), 20);
/// ```
///
/// ```should_panic
/// my_crate::item_at(&[10, 20, 30], 3);
/// ```
pub fn item_at(list: &[i32], index: usize) -> i32 {
    list[index]
}
```

In the verified project, `cargo test --doc` ran five doc tests (including the two for these functions and one that is marked `should panic`) and all passed.

Why it works: `# Errors` tells callers what to handle, `# Panics` tells them what to avoid, and each claim has a test, so it cannot become untrue unnoticed.

</details>

### Exercise 4 — Prepare a release (medium)

Starting with the branch from "Try it yourself": write a `CHANGELOG.md` with an `Unreleased` section and a `0.1.0` section, tag the release, make a new
commit (`feat: …`), and show with one command which commits came after the tag. Write down the commands.

<details>
<summary>Hint</summary>

Tag with `git tag -a v0.1.0 -m "…"`. The commits after a tag are `git log --oneline v0.1.0..HEAD`. A new feature goes under `## [Unreleased]` / `### Added`.

</details>

<details>
<summary>Answer</summary>

```markdown
# Changelog

All notable changes to this project are documented in this file.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- `percent`: a whole-number percentage.

## [0.1.0] - 2026-11-17

### Added
- `slugify`: turn a title into URL-friendly text.
- `parse_duration`: read durations such as `1h30m15s`.
```

```bash
git add code/p05-textkit/CHANGELOG.md
git commit -m "docs(p05): add CHANGELOG"
git tag -a v0.1.0 -m "First release of p05-textkit"
# ...implement and commit a new feature...
git commit -am "feat(p05): add percent"
git log --oneline v0.1.0..HEAD
```

```text
48ab9f0 feat(p05): add percent
```

(Your hash will be different.) Why it works: the tag fixes the release in history, `v0.1.0..HEAD` lists only the later commits, and the changelog keeps the
`Unreleased` section ready for the next version.

</details>

## Quiz

1. What is the difference between `//`, `///` and `//!`?

<details>
<summary>Answer</summary>

`//` is an ordinary comment for people reading the code. `///` is a doc comment for the item right below it. `//!` is a doc comment for the item that contains it (a crate or module), usually at the top of `lib.rs`.

</details>

2. Why is the first line of a doc comment special?

<details>
<summary>Answer</summary>

It is used as the summary in lists of items. It should be a one-sentence description that starts with a verb and ends with a period.

</details>

3. What happens to a ` ``` ` code block in a doc comment when you run `cargo test`?

<details>
<summary>Answer</summary>

It is compiled and run as a doc test, as a small program that uses your crate from the outside. If it fails to compile or panics, the doc test fails.

</details>

4. How do you show a short example in the docs that really needs `fn main` and `Ok(())` to compile?

<details>
<summary>Answer</summary>

Put those lines in the code block but start them with `# ` (a hash and a space). They are compiled, but hidden in the rendered documentation.

</details>

5. Which standard section do you write for a function that returns `Result`, and which for a function that can panic?

<details>
<summary>Answer</summary>

`# Errors` for the `Result` function, and `# Panics` for the function that can panic (plus `# Safety` for `unsafe` functions).

</details>

6. Why should comments in the code explain *why* and not *what*?

<details>
<summary>Answer</summary>

The code already shows what happens, and a comment that repeats it will become wrong when the code changes. The reason behind a decision (a workaround, a rule, a unit) is not visible in the code, so only a comment can keep it.

</details>

7. What does the tag `v0.1.0` give you, and what does `git log --oneline v0.1.0..HEAD` show?

<details>
<summary>Answer</summary>

The tag is a permanent name for the release commit. The command lists the commits made after that release, which is the raw material for the next changelog entry.

</details>

## Summary

- Three layers of documentation: `//` comments explain *why* to maintainers, `///` and `//!` explain *what and how* to users, and README, CHANGELOG and Git history explain the project.
- A doc comment starts with a one-sentence summary and uses Markdown plus the standard sections `# Examples`, `# Errors`, `# Panics`, `# Safety`.
- Every Rust code block in a doc comment is a doc test run by `cargo test`; hide boilerplate with `# `, and use `no_run`, `should_panic`, `compile_fail` when needed.
- `cargo doc --no-deps --open` builds and shows the HTML docs; `#![warn(missing_docs)]` and intra-doc links `` [`Name`] `` keep them complete and correct.
- Do not comment what the code says; explain why. Never leave commented-out code, and never put secrets into docs.
- Keep a README (what, how to use, how to build) and a CHANGELOG (Keep a Changelog format), and number versions with SemVer.
- `git tag -a`, `git log v0.1.0..HEAD` and `git blame` turn Git into documentation: write commit messages for the reader of the future.

## Sources and further reading

- [The rustdoc book](https://doc.rust-lang.org/rustdoc/) — the complete guide to `rustdoc`, `cargo doc` and doc comments.
- [The rustdoc book — How to write documentation](https://doc.rust-lang.org/rustdoc/how-to-write-documentation.html) — sections, Markdown, hidden lines and links.
- [The rustdoc book — Documentation tests](https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html) — all the code block attributes.
- [The rustdoc book — Lints](https://doc.rust-lang.org/rustdoc/lints.html) — `broken_intra_doc_links` and the other rustdoc warnings.
- [Rust API Guidelines — Documentation](https://rust-lang.github.io/api-guidelines/documentation.html) — what good Rust documentation contains.
- [The Cargo Book — cargo doc](https://doc.rust-lang.org/cargo/commands/cargo-doc.html) — options of `cargo doc`.
- [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/) and [Semantic Versioning 2.0.0](https://semver.org/) — the changelog format and the version numbers.
- [Git documentation — git tag](https://git-scm.com/docs/git-tag) and [git blame](https://git-scm.com/docs/git-blame).

---

[🏠 Start](../START-HERE.md) · [← 05.6 Testing in depth](06-testing-in-depth.md) · [📋 Phase 05](README.md) · [05.8 Cargo: dependencies and versions →](08-cargo-dependencies-and-versions.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
