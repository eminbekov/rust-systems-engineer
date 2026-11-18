<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.17 Checkpoint](../08-software-design/17-checkpoint.md) · [09.1 The test pyramid and a testing strategy →](01-the-test-pyramid.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/README.md) <!-- nav -->

# Phase 09 — Testing, debugging and benchmarking

> **Stage 2 — Engineering fundamentals** · ⏱ about 3–4 weeks at ~10 hours per week

## What this phase is about

A program that you cannot check is a program that you cannot change. Tests, a debugger, logs and measurements are the four ways in which an engineer *knows* what the code does, instead of hoping. In Phases 02 and 05 you wrote your first tests, used print debugging and timed a function. This phase turns those first steps into a professional toolbox.

You begin with the big picture: the test pyramid, which says how many tests of each kind to write and where to put them. Then you learn the techniques that go beyond hand-picked examples: end-to-end tests of command-line programs, test doubles (fakes, stubs and mocks), property-based tests that search for the input you did not think of, and snapshot tests for large outputs. You practise test-driven development on a real session, and you learn two ways to test your tests: code coverage and mutation testing.

The second half is about finding and understanding problems. You fuzz a parser to look for crashes, step through a program in the LLDB debugger inside RustRover, and add structured logging with `tracing`, so that a running program can explain itself. You learn to benchmark honestly with `criterion` and `hyperfine`, and to avoid the measurements that lie. Finally you put everything on a server: a CI (continuous integration) pipeline on GitHub Actions and GitLab CI that runs your checks on every change. The phase ends with a project: a library with unit, property, snapshot and fuzz tests, benchmarks and a CI pipeline.

Every lesson uses a small package in `code/p09-<topic>` and shows real output from real commands, including the failures. Read the failures carefully: learning to read a red test or a crash report is half of the skill.

## What you will be able to do

- [ ] Choose the level of a test (unit, integration, end-to-end) and explain why a healthy test suite has the shape of a pyramid.
- [ ] Test a command-line program from the outside with `assert_cmd`, predicates and temporary files.
- [ ] Replace slow or unpredictable dependencies with test doubles, and tell a fake, a stub, a spy and a mock apart.
- [ ] Write property-based tests with `proptest` and snapshot tests with `insta`, and review a snapshot change responsibly.
- [ ] Work test-first on a small feature and fix a bug by writing the failing test before the fix.
- [ ] Measure coverage with `cargo llvm-cov`, find weak tests with `cargo mutants`, and explain why 100 % coverage is not enough.
- [ ] Find crashes with a fuzzer, debug a program with breakpoints in LLDB and RustRover, and add structured logs with `tracing`.
- [ ] Benchmark with `criterion` and `hyperfine`, and write a CI pipeline for GitHub Actions and GitLab CI that runs the quality gate.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 09.1 | [The test pyramid and a testing strategy](01-the-test-pyramid.md) | 2.5 h |
| 09.2 | [End-to-end tests for command-line programs](02-testing-command-line-programs.md) | 120 min |
| 09.3 | [Test doubles: fakes, stubs and mocks](03-test-doubles.md) | 135 min |
| 09.4 | [Property-based testing](04-property-based-testing.md) | 135 min |
| 09.5 | [Snapshot testing](05-snapshot-testing.md) | 2.5 h |
| 09.6 | [Test-driven development in practice](06-tdd-in-practice.md) | 135 min |
| 09.7 | [Code coverage and mutation testing](07-code-coverage-and-mutation-testing.md) | 120 min |
| 09.8 | [Fuzzing](08-fuzzing.md) | 135 min |
| 09.9 | [Debugging in depth: LLDB and RustRover](09-debugging-with-lldb-and-rustrover.md) | 135 min |
| 09.10 | [Logging and tracing](10-logging-and-tracing.md) | 105 min |
| 09.11 | [Benchmarking in depth](11-benchmarking-in-depth.md) | 120 min |
| 09.12 | [Continuous integration: GitHub Actions and GitLab CI](12-continuous-integration.md) | 135 min |
| 09.13 | [Project: a well-tested library with CI](13-project-tested-library-with-ci.md) | 8 h |
| 09.14 | [Checkpoint](14-checkpoint.md) | 120 min |

## Projects in this phase

- **Small packages in the lessons** (`code/p09-pyramid`, `code/p09-cli-tests`, `code/p09-doubles`, `code/p09-proptest`, `code/p09-snapshots`, `code/p09-tdd`, `code/p09-coverage`, `code/p09-fuzzing`, `code/p09-debugging`, `code/p09-tracing`, `code/p09-bench`, `code/p09-ci`): each lesson has its own follow-along package with tests, real command output and exercises that ask you to write a test.
- **[Project 09.13 — A well-tested library with CI](13-project-tested-library-with-ci.md)**: build a small library test-first, add property, snapshot and fuzz tests, measure coverage and mutants, add a benchmark, and write the CI pipeline that checks all of it. The project lives in `code/p09-durparse`: a parser and formatter for durations such as `1h30m`.

## How to study this phase

1. **Type the code and break it on purpose.** Almost every lesson shows a bug and the test that finds it. Introduce the bug yourself and watch the test fail; a test that you have never seen fail is a test you cannot trust.
2. **Read every failure message to the end.** Test reports, shrunk inputs, snapshot diffs and mutant lists all say exactly what is wrong. Ask "is this the failure I expected?" before you change any code.
3. **Keep the gate green.** Before each commit run `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test`. It is the habit that the CI lesson at the end automates.
4. **Apply each technique to your own code.** After each lesson, open a program from Phases 04 to 08 and add one property test, one snapshot or one coverage run to it. A technique you used twice is a technique you own.
5. **Measure, do not guess.** This applies to performance, to coverage and to bugs alike. Numbers, logs and debugger sessions beat intuition, and they are the evidence you will show in a code review.

---

[🏠 Start](../START-HERE.md) · [← 08.17 Checkpoint](../08-software-design/17-checkpoint.md) · [09.1 The test pyramid and a testing strategy →](01-the-test-pyramid.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
