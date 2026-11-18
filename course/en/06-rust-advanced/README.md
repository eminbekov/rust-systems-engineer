<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.14 Checkpoint](../05-rust-intermediate/14-checkpoint.md) · [06.1 Trait objects and dynamic dispatch →](01-trait-objects-and-dynamic-dispatch.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/README.md) <!-- nav -->

# Phase 06 — Rust advanced

> **Stage 1 — Foundations: tools, Git and the Rust language** · ⏱ about 4–5 weeks at ~10 hours per week

## What this phase is about

You already write real Rust programs: you can model data with structs and enums, borrow values safely, and reuse code with generics, traits and iterators. This phase opens the parts of the language that you meet when you read the standard library, big crates and systems code: how Rust chooses between fast static code and flexible dynamic code, how patterns and lifetimes work in full detail, and how to change data through a shared reference.

Then we go to the edge of the safe language. You will learn what `unsafe` means, why it is a promise and not a switch, and how to check your promises with Miri, a tool that runs your code and reports undefined behaviour. After that you will write macros (code that writes code), get a first look at `async`/`await`, and call a C library from Rust.

Why does a systems engineer need this? Operating systems, databases, network proxies and async runtimes are all built with these tools. You do not write `unsafe` or macros every day, but you must be able to read them, judge whether they are correct, and write a small piece when the job needs it. The phase ends with a project that joins many ideas: a multithreaded web server built from scratch, followed by a checkpoint that closes Stage 1.

You will also keep the habits from earlier phases: every lesson ends with a small commit in Git, and every exercise is checked with `cargo test`, `cargo clippy` and `cargo fmt`.

## What you will be able to do

- [ ] Choose between generics (static dispatch) and trait objects (dynamic dispatch), and explain the vtable and the fat pointer.
- [ ] Use associated types, operator overloading, supertraits, the newtype pattern and fully qualified syntax.
- [ ] Read and write every kind of pattern: ranges, guards, `@` bindings, slice patterns, `let else` and let chains.
- [ ] Pick the right interior-mutability type (`Cell`, `RefCell`, `OnceLock`, `LazyLock`, `Mutex`) and avoid run-time borrow panics.
- [ ] Explain lifetime elision, `'static`, variance and `for<'a>`, and fix lifetime errors from the compiler message.
- [ ] Write small `unsafe` code with `// SAFETY:` comments and a safe wrapper, and run it under Miri.
- [ ] Write declarative and procedural macros, and use async/await with an async runtime.
- [ ] Call C functions through FFI, and build and test a multithreaded web server.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 06.1 | [Trait objects and dynamic dispatch](01-trait-objects-and-dynamic-dispatch.md) | 4 h |
| 06.2 | [Advanced traits and types](02-advanced-traits-and-types.md) | 4 h |
| 06.3 | [Patterns in depth](03-patterns-in-depth.md) | 3 h |
| 06.4 | [Interior mutability](04-interior-mutability.md) | 3 h |
| 06.5 | [Lifetimes in depth](05-lifetimes-in-depth.md) | 4 h |
| 06.6 | [Unsafe Rust and Miri](06-unsafe-rust-and-miri.md) | 4 h |
| 06.7 | [Declarative macros](07-declarative-macros.md) | 3.5 h |
| 06.8 | [Procedural macros](08-procedural-macros.md) | 3 h |
| 06.9 | [Introduction to async/await](09-async-await-introduction.md) | 4 h |
| 06.10 | [FFI: calling C code](10-ffi-calling-c.md) | 4 h |
| 06.11 | [Project: multithreaded web server](11-project-multithreaded-web-server.md) | 4 h |
| 06.12 | [Checkpoint: Stage 1 complete](12-checkpoint-stage-1-complete.md) | 3 h |

## Projects in this phase

- **Practice projects (lessons 06.1 to 06.10).** Every lesson has a small practice project, `code/p06-<topic>` (for example `code/p06-trait-objects` and `code/p06-unsafe`). Each one ends with a Git commit.
- **Project 06.11 — multithreaded web server.** A web server written from scratch with threads, a thread pool and graceful shutdown, with tests and a README.
- **Checkpoint 06.12.** A self-test and practical tasks that confirm you are ready for Stage 2.

## How to study this phase

1. **Type the code, do not paste it.** Unsafe code, macros and lifetimes only become clear when your hands write them. Read the "What happens" list after you have run the example, not before.
2. **Read compiler errors slowly.** This phase has many real error messages on purpose. For each one, find the error code (for example `E0038`) and try `rustc --explain E0038` in your terminal.
3. **Run Miri on every `unsafe` exercise.** Install the nightly toolchain once (Lesson 06.6). A passing test is not a proof; Miri catches mistakes that tests cannot see.
4. **Commit after each lesson.** Use short messages such as `feat(p06): plugin registry with trait objects`. If an experiment goes wrong, `git diff` and `git restore` bring you back.
5. **Take breaks between lessons 06.5 to 06.9.** They are the densest. If you get stuck, go back one lesson, re-run its examples, and then continue.

---

[🏠 Start](../START-HERE.md) · [← 05.14 Checkpoint](../05-rust-intermediate/14-checkpoint.md) · [06.1 Trait objects and dynamic dispatch →](01-trait-objects-and-dynamic-dispatch.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
