<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 03.8 Project: word counter, and checkpoint](../03-ownership-and-borrowing/08-project-word-counter-and-checkpoint.md) · [04.1 Structs →](01-structs.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/README.md) <!-- nav -->

# Phase 04 — Structs, enums, collections and errors

> **Stage 1 — Foundations: tools, Git and the Rust language** · ⏱ about 4–5 weeks at ~10 hours per week

## What this phase is about

Until now your programs worked with single numbers, single pieces of text and short functions. Real programs describe **things**: a user, a book, a bank account, a contact in a phone book. In this phase you learn how Rust lets you describe such things exactly. A **struct** groups several named values into one type ("a contact has a name and a number"). An **enum** says that a value is one of several choices ("a command is add, remove or quit"). With `match` and `Option` the compiler makes sure that you never forget a case, and that "there is no value" is never a hidden surprise.

Next you learn to keep a growing program tidy. **Modules**, **crates** and **packages** split code into pieces with clear names and clear borders, so that every piece can hide its details and be tested alone. Then you meet the three collections you will use in almost every program: the **vector** (a growable list), the **string** in depth (what text really is inside the computer) and the **hash map** (a fast lookup table).

The last big idea is **error handling**. Files are missing, users type nonsense, networks fail. Rust does not hide this: operations that can fail return a `Result`, and you decide what to do with the failure. You will learn the `?` operator, how to define your own error type, and when a crash (`panic`) is acceptable. Everything comes together in the project of the phase, a **phone book** program with tests, clean `cargo clippy` output and one Git commit per milestone.

## What you will be able to do

- [ ] Model data with structs (named, tuple and unit-like) and give them behaviour with `impl` blocks, methods and associated functions.
- [ ] Model "one of several choices" with enums that carry data, and handle every case with `match`, `if let` and `let else`.
- [ ] Use `Option<T>` instead of "null" values and choose between `match`, `unwrap_or`, `map` and `?`-style early exits.
- [ ] Split a program into modules, files, a library crate and a binary crate, and decide what is `pub` and what stays private.
- [ ] Store and process data in `Vec`, `String` and `HashMap`, including the entry API, and explain how text is stored (UTF-8, the standard way to store text as bytes).
- [ ] Return `Result` from functions that can fail, propagate errors with `?`, and define your own error type.
- [ ] Write unit tests and integration tests for these types, and keep `cargo clippy` and `cargo fmt` clean.
- [ ] Commit your work in small steps with clear messages, and use a branch for a bigger piece of work.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 04.1 | [Structs](01-structs.md) | 4 h |
| 04.2 | [Methods and associated functions](02-methods-and-associated-functions.md) | 4 h |
| 04.3 | [Enums](03-enums.md) | 3.5 h |
| 04.4 | [Option and match](04-option-and-match.md) | 3 h |
| 04.5 | [if let, let else and patterns](05-if-let-let-else-and-patterns.md) | 3.5 h |
| 04.6 | [Modules, crates and packages](06-modules-crates-and-packages.md) | 4 h |
| 04.7 | [Vectors](07-vectors.md) | 4 h |
| 04.8 | [Strings in depth](08-strings-in-depth.md) | 4 h |
| 04.9 | [Hash maps](09-hash-maps.md) | 4 h |
| 04.10 | [Error handling](10-error-handling.md) | 5 h |
| 04.11 | [Project: phone book](11-project-phone-book.md) | 3 h |
| 04.12 | [Checkpoint](12-checkpoint.md) | 3 h |

## Projects in this phase

- **Small practice projects** (one per lesson, in `~/projects/rust/code/`): `p04-structs`, `p04-methods`, `p04-enums`, `p04-option-match`, `p04-patterns`, `p04-modules`, and more in the later lessons. Each is a few dozen lines and ends with a Git commit.
- **Project: phone book** ([Lesson 04.11](11-project-phone-book.md)): a command-line program that stores contacts, searches them, and handles bad input with proper errors. It is split into modules, has unit and integration tests, and is built milestone by milestone.
- **Checkpoint** ([Lesson 04.12](12-checkpoint.md)): a quiz and practical tasks that show whether you are ready for Phase 05.

## How to study this phase

1. **Type every example yourself** and run it. Do not paste. The compiler errors you meet while typing are half of the lesson, and the lessons show the real error messages so you can compare.
2. **Make one small commit per lesson.** Each lesson ends with a commit of its practice project. Use the message style from [Lesson 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) (`feat(p04): …`). Run `git status` before every commit and make sure no `target/` folder is included.
3. **Use a branch for the phone book.** Start it with `git switch -c feat/p04-phone-book`, commit at each milestone, and merge it back as you practised in [Lesson 01.6 — Branches and merging](../01-git-and-version-control/06-branches-and-merging.md). If you work with a remote, open a pull request to yourself and read your own diff before merging.
4. **Always run the same four commands** before a commit: `cargo fmt`, `cargo clippy`, `cargo test`, `cargo run`. When one of them complains, read the whole message, not only the first line.
5. **Break things on purpose.** Remove a `match` arm, make a field private, call `unwrap` on `None`. Seeing the error on purpose makes it easy to recognise later.
6. **Do the exercises before you open the answers.** Open the hint first, then the answer only to compare. Write the test before the solution when the exercise asks for it.

---

[🏠 Start](../START-HERE.md) · [← 03.8 Project: word counter, and checkpoint](../03-ownership-and-borrowing/08-project-word-counter-and-checkpoint.md) · [04.1 Structs →](01-structs.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
