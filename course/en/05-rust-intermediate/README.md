<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.12 Checkpoint](../04-structs-enums-collections-errors/12-checkpoint.md) · [05.1 Generics →](01-generics.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/README.md) <!-- nav -->

# Phase 05 — Rust intermediate

> **Stage 1 — Foundations: tools, Git and the Rust language** · ⏱ about 5 weeks at ~10 hours per week

## What this phase is about

After Phase 04 you can describe data with structs and enums, keep it in vectors and hash maps, and handle errors. This
phase teaches the tools that turn such programs into **reusable, tested and documented code**. You will write one function
that works for many types (generics), describe shared abilities (traits), and tell the compiler how long borrowed data lives
(lifetimes). You will pass small pieces of behaviour around (closures) and process sequences of data in a clear, fast style (iterators).

The second half of the phase is about the everyday work of an engineer. You will test your code properly and document it,
bring in other people's code with Cargo and crates, use smart pointers, share work between threads, and measure speed
with benchmarks. You will meet `serde`, `clap`, `thiserror` and `anyhow`, the crates that almost every real Rust program uses.

Throughout the phase you also practise the engineering habits around the code: one Git branch per lesson, small commits with
meaningful messages, `git add -p`, `git stash`, `git bisect`, a pre-commit hook that runs `cargo fmt`, `cargo clippy` and `cargo test`,
and tagged releases with a changelog. Good code that nobody can understand, test or release is not yet engineering.

The phase ends with a real command-line tool, `minigrep`, and a checkpoint. After it you are ready for the advanced parts of Rust in Phase 06.

## What you will be able to do

- [ ] Write generic functions and types with trait bounds, and explain why generic code is as fast as hand-written code.
- [ ] Define traits, implement standard traits (`Display`, `From`, `Add`, `Default`, `Iterator`) and derive the common ones.
- [ ] Read and write lifetime annotations, and return borrowed data from a function without copying it.
- [ ] Use closures and iterator pipelines (`map`, `filter`, `fold`, `collect`) and choose between a loop and a pipeline.
- [ ] Test code test-first, cover errors and edge cases, and organize unit, integration and doc tests.
- [ ] Document a public API (the functions and types that other code may use) with examples that run as tests, generate it with `cargo doc`, and keep a README and a changelog.
- [ ] Add dependencies with Cargo, use smart pointers, share state between threads, and measure speed with a benchmark.
- [ ] Use Git as an engineering tool: branches, partial staging, hooks, bisect, tags.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 05.1 | [Generics](01-generics.md) | 4 h |
| 05.2 | [Traits](02-traits.md) | 4 h |
| 05.3 | [Lifetimes](03-lifetimes.md) | 4 h |
| 05.4 | [Closures](04-closures.md) | 4 h |
| 05.5 | [Iterators](05-iterators.md) | 4 h |
| 05.6 | [Testing in depth](06-testing-in-depth.md) | 3.5 h |
| 05.7 | [Documentation](07-documentation.md) | 3 h |
| 05.8 | [Cargo: dependencies and versions](08-cargo-dependencies-and-versions.md) | 3 h |
| 05.9 | [Smart pointers](09-smart-pointers.md) | 3 h |
| 05.10 | [Threads, channels and shared state](10-threads-channels-and-shared-state.md) | 3 h |
| 05.11 | [Practical crates: serde, clap, thiserror and anyhow](11-practical-crates-serde-clap-errors.md) | 4.5 h |
| 05.12 | [Benchmarking basics](12-benchmarking-basics.md) | 3 h |
| 05.13 | [Project: minigrep](13-project-minigrep.md) | 4 h |
| 05.14 | [Checkpoint](14-checkpoint.md) | 2.5 h |

## Projects in this phase

- **Practice packages** (`code/p05-lessons`, `code/p05-textkit`): small examples from Lessons 05.1 to 05.7, each lesson on its own Git branch.
  `p05-textkit` grows from a first failing test into a documented, tested and tagged library (version 0.1.0).
- **`minigrep`** (Lesson 05.13): a command-line search tool in the spirit of `grep`, built with tests, error handling, a library plus a thin `main`,
  documentation and a Git commit for every milestone.
- **Checkpoint** (Lesson 05.14): a self-test quiz and practical tasks that show whether you are ready for Phase 06.

## How to study this phase

- **Type every example yourself** in `code/p05-lessons` and run it. Then change it on purpose and read the compiler's message.
  In this phase the error messages are the teacher: generics, traits and lifetimes errors are the ones that explain the rules best.
- **Follow the Git steps in each lesson.** One branch per lesson, a commit after every working example, and the message in the form `feat(p05): …`.
  By the end of the phase your history should read like a diary of what you learned.
- **Do not skip the order.** Lessons 05.1 to 05.3 (generics, traits, lifetimes) belong together, and closures and iterators build on them. If a
  lesson feels hard, reread the previous one and redo its exercises.
- **Write the tests before the code** whenever an exercise asks you to write both. Watch the test fail first; that is the habit that makes
  projects safe to change.
- **Plan about 10 hours a week.** Two or three lessons a week is a good pace; leave a full day for the `minigrep` project and do not rush it.

---

[🏠 Start](../START-HERE.md) · [← 04.12 Checkpoint](../04-structs-enums-collections-errors/12-checkpoint.md) · [05.1 Generics →](01-generics.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
