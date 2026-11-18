<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.10 Project: your learning repository, and checkpoint](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) · [02.1 Variables and mutability →](01-variables-and-mutability.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/README.md) <!-- nav -->

# Phase 02 — Programming basics in Rust

> **Stage 1 — Foundations: tools, Git and the Rust language** · ⏱ about 3 weeks at ~10 hours per week

## What this phase is about

Every program, in every language, is built from the same few blocks: it keeps data in **variables**, gives the data **types**, calculates with **operators**, groups code into **functions**, makes decisions with **conditions**, repeats work with **loops**, and talks to the user through **input and output**. This phase teaches these blocks in Rust, one at a time, and shows you the real compiler messages that you will meet every day.

You will also learn two habits that separate a hobby coder from an engineer. The first is **testing**: writing small checks that prove your code works and that run again after every change. The second is **debugging**: finding the cause of a bug with a method instead of by guessing, using prints, `dbg!`, a backtrace and the debugger of RustRover. Git runs through the whole phase: you commit after every lesson, and in the project you work on a feature branch, make one commit per milestone, merge, and tag the result.

At the end you put everything together in a project: a number guessing game for the terminal, with an external crate (`rand`), unit tests, a README and a clean Git history. This phase is the base for everything that follows. The next phase (ownership and borrowing) explains the idea that makes Rust special, and it is much easier when these basics feel natural.

## What you will be able to do

- [ ] Create variables with `let`, `mut` and `const`, and explain shadowing and scope.
- [ ] Choose between the integer, float, `bool` and `char` types, and explain overflow and `as` conversions.
- [ ] Write functions with parameters and return values, and use tuples and arrays.
- [ ] Control the flow of a program with `if`, `match`, `loop`, `while` and `for`.
- [ ] Format output with `println!`, `format!`, `{:?}` and format specifiers, and comment code well.
- [ ] Read input from the terminal, parse it, and handle wrong input without crashing.
- [ ] Write unit tests (`#[test]`, `assert_eq!`, `should_panic`) and use test-driven development.
- [ ] Find bugs with `dbg!`, backtraces, the debugger, Clippy and `git bisect`, and build a small tested program with a Git feature branch, one commit per milestone, a merge and a tag.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 02.1 | [Variables and mutability](01-variables-and-mutability.md) | 2.5 h |
| 02.2 | [Data types](02-data-types.md) | 3 h |
| 02.3 | [Operators and expressions](03-operators-and-expressions.md) | 3 h |
| 02.4 | [Functions](04-functions.md) | 2.5 h |
| 02.5 | [Control flow: conditions and loops](05-control-flow.md) | 3 h |
| 02.6 | [Printing, formatting and comments](06-printing-formatting-and-comments.md) | 3 h |
| 02.7 | [Reading input and parsing text](07-reading-input-and-parsing.md) | 3 h |
| 02.8 | [Your first tests](08-your-first-tests.md) | 3 h |
| 02.9 | [Debugging basics](09-debugging-basics.md) | 3 h |
| 02.10 | [Project: the guessing game](10-project-guessing-game.md) | 2.5 h |
| 02.11 | [Checkpoint](11-checkpoint.md) | 120 min |

## Projects in this phase

- **`code/p02-basics`** — a practice project with one small program per lesson in `src/bin/` (Lessons 02.1–02.9).
- **`code/p02-first-tests`** — a small library with your first unit tests (Lesson 02.8).
- **`code/p02-guessing-game`** — the main project: a terminal number guessing game with the `rand` crate, tests, a README, a feature branch and a tag (Lesson 02.10).
- **Checkpoint tasks** — four short programs (a temperature table, FizzBuzz, a sum calculator, a bug hunt), each with tests and a commit (Lesson 02.11).

## How to study this phase

1. **Type the code yourself.** Do not copy and paste. Typing makes you notice every character, and the compiler errors teach you more than the working code does.
2. **Break things on purpose.** Remove a `mut`, add a semicolon, use a wrong type, and read the error. By the end of the phase the error messages should feel like friends.
3. **Run the examples, then change them.** Change a number, a type, a condition, and predict the result before you run. Prediction is what builds understanding.
4. **Commit after every lesson** with a clear message (`feat(p02): …`, `test(p02): …`). A good Git history is a diary of your progress and a safety net.
5. **Do the checkpoint without notes**, then review only the topics you missed. If a topic stays unclear, return to it after a day and after a week.

---

[🏠 Start](../START-HERE.md) · [← 01.10 Project: your learning repository, and checkpoint](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) · [02.1 Variables and mutability →](01-variables-and-mutability.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
