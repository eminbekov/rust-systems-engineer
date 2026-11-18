<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.12 Continuous integration: GitHub Actions and GitLab CI](12-continuous-integration.md) · [📋 Phase 09](README.md) · [09.14 Checkpoint →](14-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/13-project-tested-library-with-ci.md) <!-- nav -->

# Lesson 09.13 — Project: a well-tested library with CI

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 13 of 14 · ⏱ about 8 hours (4 sittings)

## What you will learn

- How to build one small library from an empty folder to a finished, defended state, using every technique of this phase in the right order.
- How to work test-first with unit, integration and doc tests, then add a property test, a snapshot test and a fuzz target.
- How to set a coverage goal and a mutation-testing goal, and how to use them to find missing tests.
- How to add one honest benchmark and a CI workflow that guards everything.

This is a project lesson: you write the code. There is no full solution here. Each milestone has a goal, guidance, a hidden hint, and "Done when" checks that you run yourself.

## Before you start

- You finished [Lesson 09.12 — Continuous integration](12-continuous-integration.md) and the earlier lessons of this phase: [09.1](01-the-test-pyramid.md) (the pyramid), [09.4](04-property-based-testing.md) (`proptest`), [09.5](05-snapshot-testing.md) (`insta`), [09.6](06-tdd-in-practice.md) (TDD), [09.7](07-code-coverage-and-mutation-testing.md) (coverage and mutants), [09.8](08-fuzzing.md) (fuzzing), [09.11](11-benchmarking-in-depth.md) (`criterion`).
- You can parse text with `&str` and `char`/bytes ([Lesson 04.8](../04-structs-enums-collections-errors/08-strings-in-depth.md)), return errors with `Result` ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) and implement `Display` ([Lesson 05.2](../05-rust-intermediate/02-traits.md)).
- The tools are installed: `cargo-insta`, `cargo-llvm-cov` (with `rustup component add llvm-tools-preview`), `cargo-mutants`, `cargo-fuzz` (with the nightly toolchain), `actionlint`.
- Docker Desktop is running, for the last milestone.
- Your code goes in `~/projects/rust/code/p09-durparse`. Work on a Git branch and make **one commit per milestone**.

## New words

| Word | Meaning |
|---|---|
| **safety net** | All the automatic checks around a piece of code (tests, properties, snapshots, fuzzing, CI) that catch a mistake when somebody changes it. |
| **milestone** | One step of a project with its own goal and its own "Done when" check. |
| **canonical form** | The one standard way to write a value, for example `1h30m` and never `90m` for the same duration. |
| **public API** (Application Programming Interface) | The functions and types of a crate that other code may use (`pub`). Integration tests and doc tests use only these. |
| **regression test** | A test added after a bug, with the input that caused it, so the bug cannot come back unnoticed. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The way Rust stores text as bytes. A `&str` is always valid UTF-8. |
| **README** | The first file a reader opens in a project: what it is and how to run it. |

## The project

### What you will build

A small library crate `p09-durparse` that reads and writes **durations** in a compact text form, the form used in configuration files and command-line options:

```text
1h30m      →  one hour and thirty minutes
2d         →  two days
1s500ms    →  one and a half seconds
```

The functionality is small on purpose. The point of the project is not the parser; it is the **safety net** around it. When you finish, a stranger could change the code and the net would tell them whether they broke something.

The finished project has:

| Piece | Tool | Lesson |
|---|---|---|
| unit tests, integration tests, doc tests | `cargo test` | 09.1, 09.6 |
| three property tests | `proptest` | 09.4 |
| one snapshot test of error messages | `insta` | 09.5 |
| one fuzz target | `cargo-fuzz` | 09.8 |
| line coverage of at least 95% and no missed mutants | `cargo-llvm-cov`, `cargo-mutants` | 09.7 |
| one benchmark | `criterion` | 09.11 |
| a CI workflow | GitHub Actions, `actionlint` | 09.12 |
| a README | — | 05.7 |

### Contract

The library exposes exactly this public API (the names matter, because the tests in this lesson use them):

```rust
use std::time::Duration;

pub fn parse_duration(input: &str) -> Result<Duration, ParseError>;
pub fn format_duration(duration: Duration) -> String;
pub fn render_error(input: &str, error: &ParseError) -> String;

pub struct ParseError {
    pub kind: ErrorKind,
    pub position: usize, // byte index in the input
}

pub enum ErrorKind {
    Empty,
    UnexpectedChar(char),
    MissingNumber,
    MissingUnit,
    UnknownUnit(String),
    UnitOrder,
    Overflow,
}
```

`ParseError` and `ErrorKind` derive `Debug`, `Clone`, `PartialEq` and `Eq`. `ParseError` implements `std::fmt::Display` and `std::error::Error`.

**The format.** A duration is one or more *parts* written next to each other, with no spaces: a *number* (one or more decimal digits, no sign, no decimal point) followed by a *unit*. The units are lowercase: `d` (day, 24 hours), `h` (hour), `m` (minute), `s` (second) and `ms` (millisecond). Units must appear **from large to small, each at most once**. A number may have leading zeros (`007s` is 7 seconds), and numbers above 59 are allowed (`90m` is 90 minutes). The value is the sum of the parts.

**`parse_duration`** returns the sum as a `Duration` with millisecond precision, or the first error found, scanning from left to right. The error `kind` and `position` follow these rules:

| Kind | When | `position` |
|---|---|---|
| `Empty` | the input is `""` | `0` |
| `MissingNumber` | a letter where a number must start | the index of that letter |
| `UnexpectedChar(c)` | any other character where a number must start (a space, `-`, a non-ASCII letter: ASCII, the American Standard Code for Information Interchange, is the basic set of 128 characters with English letters and digits…) | the index of `c` |
| `MissingUnit` | digits not followed by an ASCII letter (end of text, or another character) | the index just after the digits |
| `UnknownUnit(text)` | the letters after the digits are not one of the five units; `text` is all of those letters | the index of the first letter |
| `UnitOrder` | the unit is not smaller than the previous one (this includes a repeated unit) | the index of the first letter of the unit |
| `Overflow` | a number does not fit in a `u64`, or the total in milliseconds does not fit in a `u64` | the index of the first digit of the part |

**Examples (your tests must contain at least these):**

| Input | Result of `parse_duration` | Then `format_duration` gives |
|---|---|---|
| `"90s"` | `Ok(90 s)` | `"1m30s"` |
| `"5m"` | `Ok(5 min)` | `"5m"` |
| `"250ms"` | `Ok(250 ms)` | `"250ms"` |
| `"0s"` | `Ok(0)` | `"0s"` |
| `"1h30m"` | `Ok(5400 s)` | `"1h30m"` |
| `"1d2h3m4s5ms"` | `Ok(93 784 005 ms)` | `"1d2h3m4s5ms"` |
| `"90m"` | `Ok(5400 s)` | `"1h30m"` |
| `"007s"` | `Ok(7 s)` | `"7s"` |
| `"18446744073709551615ms"` | `Ok(u64::MAX ms)` | `"213503982334d14h25m51s615ms"` |
| `""` | `Err(Empty, 0)` | |
| `"90"` | `Err(MissingUnit, 2)` | |
| `"h"` | `Err(MissingNumber, 0)` | |
| `"5x"` | `Err(UnknownUnit("x"), 1)` | |
| `"1H"` | `Err(UnknownUnit("H"), 1)` | |
| `"1h-5m"` | `Err(UnexpectedChar('-'), 2)` | |
| `" 1h"` | `Err(UnexpectedChar(' '), 0)` | |
| `"1.5h"` | `Err(MissingUnit, 1)` | |
| `"30m1h"` | `Err(UnitOrder, 4)` | |
| `"1h1h"` | `Err(UnitOrder, 3)` | |
| `"99999999999999999999s"` | `Err(Overflow, 0)` | |
| `"18446744073709551615d"` | `Err(Overflow, 0)` | |

**`format_duration`** writes the *canonical* text: the parts with a non-zero count, from large to small, with each unit holding the remainder (so `90s` becomes `1m30s`, never `90s`). A zero duration is `"0s"`. Anything smaller than a millisecond is dropped (`1500 µs` is `"1ms"`). For a `Duration` of whole milliseconds, `parse_duration(&format_duration(d))` must give `d` back.

**`render_error`** returns a message for people, in this shape (the wording of the first line is yours; the caret line is not):

```text
error: unknown unit `x` at position 1
  5x
   ^
```

The second line is the input indented by two spaces; the third line has two spaces plus `position` more spaces, then a `^`.

**Quality rules for the whole project:** no `unwrap()` or `expect()` on user input in `src/` (the code must never panic, whatever the input), `cargo fmt --check` and `cargo clippy --all-targets -- -D warnings` clean, every public function has a doc comment, and a short `README.md`.

### Step-by-step plan

#### M0 — Setup

**Goal:** an empty library with the dependencies and a first commit.

**Guidance:**

```bash
cd ~/projects/rust
git switch main
git switch -c project/09-13-durparse
cargo new --lib code/p09-durparse
cd code/p09-durparse
cargo add --dev proptest@1 insta@1 criterion@0.8
```

Add `/mutants.out*/` to a `.gitignore` file in the project folder (`cargo-mutants` writes its reports there). Write the `README.md` skeleton: a title, one sentence about the format, and the heading "Development" (you will fill it in at M8).

**Done when:**

```bash
cargo test
```

```text
running 0 tests
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

and `git log --oneline` shows your first commit, `project: set up p09-durparse`.

#### M1 — The first tests, red then green

**Goal:** `parse_duration` handles **one part** (`90s`, `5m`, `250ms`, `0s`).

**Guidance:** Work like Lesson 09.6. Write the *list* of tests first in a comment. Then write one test, see it fail for the right reason (`todo!()` first, then a wrong answer), make it pass with the simplest code, and repeat. Define `ParseError` and `ErrorKind` now (the contract lists them) but return only `Ok` cases at this stage. Use `Duration::from_millis` and work in milliseconds as a `u64`.

<details>
<summary>Hint</summary>

Do not write a clever loop yet. The first passing version may handle only the digits followed by the unit `s`. The next test (`5m`) forces a `match` on the unit. The test for `250ms` forces you to read two letters. Each test should force one new line of code.

</details>

**Done when:** you saw each new test fail before you made it pass, and:

```bash
cargo test
```

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(at least 4 tests, one for each of the four first rows of the example table). Commit: `parse a single part`.

#### M2 — The full parser, errors, three kinds of tests

**Goal:** `parse_duration` follows the whole contract.

**Guidance:**

1. Add the rest of the parser: several parts, the unit order rule, all the error kinds and checked arithmetic. Use `u64::checked_mul` and `checked_add`, never plain `*` and `+` on numbers that come from the input (Lesson 02.2: a debug build panics on overflow, a release build wraps around silently, and both are bugs here).
2. Put the examples of the table into a **table-driven unit test** in `src/lib.rs` (a loop over `(input, expected)` pairs with a helpful assertion message that includes the input).
3. Add an **integration test** in `tests/public_api.rs` that uses only the public API: parse a value, print the error with `to_string()`, compare `error.kind`.
4. Add a **doc test**: a `//!` comment at the top of `src/lib.rs` with an example of `parse_duration("1h30m")`.
5. Write `impl Display for ParseError` and `impl std::error::Error for ParseError`. The message must contain the position.

<details>
<summary>Hint</summary>

A scanner with an index is the simplest structure: `pos` moves over the bytes (`input.as_bytes()`); first `while` over ASCII digits, then `while` over ASCII letters. Work with the byte index because the contract speaks about byte positions. Remember to call `parse::<u64>()` on the digits and to map its error to `Overflow` (a digit string can only fail by being too large, because you checked that all characters are digits). Keep `last_unit: Option<Unit>` and compare the units with a derived `PartialOrd` (largest unit first in the `enum`).

</details>

**Done when:**

```bash
cargo test
cargo clippy --all-targets -- -D warnings
cargo fmt --check
```

```text
     Running unittests src/lib.rs (...)
test result: ok. N passed; 0 failed; ...
     Running tests/public_api.rs (...)
test result: ok. 2 passed; 0 failed; ...
   Doc-tests p09_durparse
test result: ok. 1 passed; 0 failed; ...
```

You see three result blocks (unit, integration, doc), clippy and fmt print nothing, and every row of the example table is in a test. Commit: `full parser with errors`.

#### M3 — Formatting and property tests

**Goal:** `format_duration` and three properties.

**Guidance:** Implement `format_duration` (TDD again: write the four canonical examples as tests first). Then create `tests/properties.rs` with three `proptest` properties:

1. **Round trip:** for every `u64` number of milliseconds `ms`, `parse_duration(&format_duration(Duration::from_millis(ms)))` is `Ok(Duration::from_millis(ms))`.
2. **No panic:** for any text from the strategy `"\\PC*"` (any printable characters), `parse_duration` returns without panicking (the result may be an error).
3. **Alphabet:** `format_duration(Duration::from_millis(ms))` contains only ASCII digits and the letters `d`, `h`, `m`, `s`.

<details>
<summary>Hint</summary>

The shape of a property test:

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn name_of_the_property(ms in any::<u64>()) {
        // arrange, then prop_assert_eq!(left, right);
    }
}
```

`format_duration` receives a `Duration`, whose `as_millis()` returns a `u128`: the number of days can be larger than a `u64` (see the last row of the table), so do the arithmetic in `u128` there. To see shrinking, break the function on purpose (for example drop the `ms` part) and read the minimal failing example that `proptest` prints, then fix it.

</details>

**Done when:**

```bash
cargo test --test properties
```

```text
test format_then_parse_round_trips ... ok
test parse_never_panics ... ok
test formatted_text_has_only_digits_and_units ... ok
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

(names may differ). Also: you broke the formatter on purpose once, saw a shrunk failing input such as `ms = 1`, and restored it. Commit: `format_duration and property tests`.

#### M4 — A snapshot of the error messages

**Goal:** `render_error` and one snapshot test.

**Guidance:** Write `render_error` as described in the contract. Then create `tests/snapshots.rs`: for a list of bad inputs (`""`, `"90"`, `"h"`, `"5x"`, `"1h-5m"`, `"30m1h"`, `"1h1h"`, `"99999999999999999999s"`), collect the rendered messages in one `String` (blank line between them) and compare it with `insta::assert_snapshot!`. The first run fails and writes a `.snap.new` file: read it, make sure the messages are what you want, then accept it.

<details>
<summary>Hint</summary>

```bash
cargo test --test snapshots     # fails: a new snapshot is pending
cargo insta review              # or: cargo insta accept
cargo test --test snapshots     # passes
```

Open `tests/snapshots/snapshots__*.snap` and read it as a person: is the caret under the right character in every case? Commit the `.snap` file. Then change one message (for example "unknown unit" to "bad unit"), watch the test fail with a diff, and revert.

</details>

**Done when:**

```bash
cargo test --test snapshots
git status --short
```

```text
test error_messages_with_carets ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

and no file ending in `.snap.new` exists (`git status --short` shows the `.snap` file as new, once). Commit: `snapshot test of error messages`.

#### M5 — Fuzzing

**Goal:** a fuzz target that runs for at least 60 seconds without finding a crash.

**Guidance:** Follow Lesson 09.8. Run `cargo fuzz init` in the project folder. The target converts the bytes to text (if they are not valid UTF-8, return early), calls `parse_duration`, and, **if it succeeds**, checks the property "whatever we accept survives a write and a read": `parse_duration(&format_duration(d)) == Ok(d)`. Run it with `cargo +nightly fuzz run fuzz_target_1 -- -max_total_time=60`.

If the fuzzer finds a crash, that is a *win*: reproduce it, minimize it (`cargo +nightly fuzz tmin`), write a regression unit test with the failing input, fix the library, and run again.

<details>
<summary>Hint</summary>

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    let Ok(text) = std::str::from_utf8(data) else {
        return;
    };
    // call parse_duration(text); when it is Ok(duration), assert the round trip
});
```

If your parser used `unwrap()`, plain `+` or `*`, or slicing like `&input[a..b]` on a position that is not a character boundary, this is where it shows. Your regression tests go in `src/lib.rs` or `tests/`, not in the ignored `fuzz/artifacts` folder.

</details>

**Done when:**

```bash
cargo +nightly fuzz run fuzz_target_1 -- -max_total_time=60
```

```text
...
#32935622	DONE   cov: ...
Done 32935622 runs in 61 second(s)
```

There must be no `Failing input` in the output (the numbers will differ). Every crash you found earlier has a regression test. Commit: `fuzz target` (the `fuzz/` folder, but not its `corpus`, `artifacts` or `target`).

#### M6 — Coverage and mutation goals

**Goal:** at least **95% line coverage** of `src/lib.rs`, and **no missed mutants**.

**Guidance:** Measure with `cargo llvm-cov --summary-only`. Look at the uncovered lines with `cargo llvm-cov --text` and decide: is the line reachable? If yes, write the test that reaches it. Then run `cargo mutants`. For every `MISSED` mutant read the changed line, ask "which test should have failed?", and add that test. A timeout or an "unviable" mutant is acceptable (see Lesson 09.7). Coverage is a map of what was *run*; the mutants tell you what is actually *checked*.

<details>
<summary>Hint</summary>

Typical survivors in a parser like this: a `>=` that could be a `>` (the unit-order check: do you have a test with a **repeated** unit, like `1h1h`?); `+` replaced by `-` or `*` in the total (do you test **several** parts and a sum that is not a product?); a position that is off by one (do your tests check `position`, not only `kind`?); the formatter's handling of `0` in the middle (`1h5s`: is there a `0m` that must be skipped?); `Display` messages (they are only checked by the snapshot, so keep it).

</details>

**Done when:**

```bash
cargo llvm-cov --summary-only
cargo mutants
```

```text
TOTAL     ...     98.60%     ...        (the Lines "Cover" column must be 95% or more)
```

```text
Found 48 mutants to test
ok       Unmutated baseline in 8s build + 1s test
48 mutants tested in 88s: 44 caught, 2 unviable, 2 timeouts
```

(Example: yours will have a different number of mutants, but it must not contain `missed`.) Commit: `tests for surviving mutants`.

#### M7 — One honest benchmark

**Goal:** a `criterion` benchmark of `parse_duration` and `format_duration` on the same input, with a saved baseline.

**Guidance:** Create `benches/parse.rs` and the `[[bench]]` entry with `harness = false`. Use `black_box` on the input. Use the longest realistic input (`"1d2h3m4s5ms"`), and build the `Duration` for the formatting benchmark outside the loop. Follow the checklist of Lesson 09.11: release build (`cargo bench` does it), a quiet machine, a correctness test next to it (you have it), and a wide interval means "repeat". Save the baseline: `cargo bench --bench parse -- --save-baseline main`.

<details>
<summary>Hint</summary>

```rust
c.bench_function("parse 1d2h3m4s5ms", |b| {
    b.iter(|| parse_duration(black_box("1d2h3m4s5ms")))
});
```

On an Apple M4 laptop a parse takes about 19 nanoseconds and a format about 200 nanoseconds (the formatter allocates strings). If your result is a thousand times different, check that you did not benchmark a debug build or something that the compiler removed.

</details>

**Done when:**

```bash
cargo bench --bench parse -- --warm-up-time 1 --measurement-time 2
```

```text
parse 1d2h3m4s5ms       time:   [18.564 ns 19.009 ns 19.488 ns]
format 1d2h3m4s5ms      time:   [201.97 ns 204.27 ns 207.25 ns]
```

(Your numbers will differ.) The benchmark has a baseline saved, and `cargo bench --no-run` builds without errors. Commit: `benchmark`.

#### M8 — CI and the README

**Goal:** a workflow that runs the whole net on every push, and a README that says how to run it.

**Guidance:** Create `.github/workflows/ci.yml` in the repository (see Lesson 09.12): checkout, toolchain with `rustfmt` and `clippy`, `Swatinem/rust-cache@v2`, then `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all-targets` and `cargo test --doc`. Add a job that builds the benchmarks (`cargo bench --no-run`). Add a **second workflow** `fuzz.yml` that runs the fuzz target for five minutes every week. Check both with `actionlint`. Then run the four gate commands in the clean container from Lesson 09.12 (`rust:1.100`), to prove that the project does not depend on your Mac. Finish the "Development" section of the `README.md` with the commands for each tool: tests, properties, snapshots (`cargo insta review`), fuzzing, coverage, mutants and benchmarks.

If the repository is not on GitHub yet, the workflow cannot run; do not worry. The container run is your proof, and the file is checked by `actionlint`.

<details>
<summary>Hint</summary>

When the project lives in a sub-folder of your learning repository (`code/p09-durparse`), GitHub looks for workflows only in the **root** `.github/workflows/` of the repository, and the commands must run in the project folder. Either set `defaults: run: working-directory: code/p09-durparse` at the job level, or add `working-directory:` to each `run` step. The `Swatinem/rust-cache` action also needs `workspaces: code/p09-durparse` in its `with:` block. Run `actionlint` from the repository root.

</details>

**Done when:**

```bash
actionlint
docker run --rm --name course-ci-durparse \
  -v ~/projects/rust/code/p09-durparse:/work:ro \
  -v ~/runci.sh:/runci.sh:ro \
  rust:1.100 /runci.sh
```

```text
=== Run cargo fmt --check
=== Run cargo clippy --all-targets -- -D warnings
...
=== Run cargo test --doc
...
=== all steps passed
```

`actionlint` prints nothing. (For the container run, the project needs a `Cargo.lock`; run `cargo generate-lockfile` first. The script is the one from Lesson 09.12. Snapshot tests need `tests/snapshots/*.snap` in Git, and they are included in the mounted folder.) Commit: `CI workflows and README`, and make a Git tag: `git tag p09-durparse-v0.1.0`.

### Stretch goals

1. **A command-line program.** Add `src/bin/dur.rs` that reads one argument (`dur 1h30m` prints `5400` seconds, `dur 5x` prints the rendered error to standard error and exits with code 2) and test it end to end with `assert_cmd` as in Lesson 09.2.
2. **Faster formatting.** `format_duration` allocates a `String` for every part with `format!`. Write a version that appends directly to one `String` (`write!` with `std::fmt::Write`, or `push_str` and a number-to-text helper). Keep the old benchmark, compare with `--baseline main`, and report the number in your commit message. Do the property tests still pass?
3. **Spaces and capitals.** Add a second function `parse_duration_lenient` that accepts spaces between the parts and capital letters (`"1H 30M"`). Do not change the strict one. Add its tests, property ("lenient parse of the strict text gives the same answer") and fuzz target.
4. **Fractions.** Support `1.5h`. This changes the contract (`"1.5h"` is now valid, and so the example row changes): update every test, the snapshot and the properties. Notice which tests tell you what to change. That is the safety net doing its job.
5. **A logging trace.** Add `tracing` events at the `DEBUG` level for each part parsed (Lesson 09.10), and a test that checks that the library produces no output by default.
6. **A mutation-testing job.** Add a weekly CI job that runs `cargo mutants` and uploads `mutants.out` as an artifact if there are missed mutants.

## Common mistakes

**1. Writing the code first and "adding tests" after.** The tests then describe the code that exists, bugs included. Do M1 and M2 test-first.

**2. A property test that repeats the code.** If the property computes the answer with the same algorithm as the function, it cannot disagree with it. Use a *different* idea: round trip, invariants, a simple model.

**3. Snapshots accepted without reading.** `cargo insta accept` makes any output the truth. Read the `.snap` file as a user would.

**4. Chasing 100% coverage.** The goal is 95% and the mutants. A test with no assertions raises coverage and checks nothing; the mutants expose it.

**5. `unwrap()` in the library "because the input was checked above".** The fuzzer will find the case you did not think of. Use `get`, `?` and `ok_or`.

**6. Workflows in the wrong folder.** GitHub reads only `.github/workflows/` at the **root** of the repository. A workflow inside `code/p09-durparse/.github/` never runs.

## Quiz

1. In which order should you do the milestones if you must stop after M4? Which parts of the net would you have?

<details>
<summary>Answer</summary>

The order given. After M4 you have unit, integration and doc tests, properties and a snapshot. You would not have fuzzing, the coverage and mutation check, the benchmark or CI. The cheapest-and-most-valuable first rule: tests before tools that judge tests.

</details>

2. Coverage is 100% but one mutant is missed. What does that tell you?

<details>
<summary>Answer</summary>

Every line runs during the tests, but the tests do not check the result of that line strongly enough: some assertion is missing or too weak. Coverage measures what ran, mutants measure what is verified.

</details>

3. Why does the fuzz target return early when the bytes are not valid UTF-8?

<details>
<summary>Answer</summary>

`parse_duration` takes `&str`, which is always valid UTF-8. The invalid bytes are outside the contract of the function, so there is nothing to check.

</details>

4. Why are benchmarks built, but not run, in CI?

<details>
<summary>Answer</summary>

The numbers on a shared runner are noise. Building them in CI keeps the code from rotting; judging the speed is done on a quiet computer with a baseline.

</details>

## Summary

- A safety net is built in layers: example tests, properties, snapshots, fuzzing, then tools that judge the tests (coverage and mutants), a benchmark and CI.
- Work test-first for the core, add the property test with a round trip and "never panics", snapshot human-readable output, and turn every fuzzer finding into a regression test.
- Goals make the work finite: 95% coverage and no missed mutants; "done" means the commands print the expected result.
- Prove that the project does not depend on your machine: run the gate in a clean container and validate the workflow with `actionlint`.

## Sources and further reading

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — what a good public API of a library looks like: naming, documentation, errors.
- [The Rust Programming Language — 11. Writing Automated Tests](https://doc.rust-lang.org/book/ch11-00-testing.html) — unit, integration and doc tests.
- [cargo-llvm-cov](https://github.com/taiki-e/cargo-llvm-cov) and [cargo-mutants](https://mutants.rs/) — the manuals of the coverage and mutation tools.
- [The Rust Fuzz Book](https://rust-fuzz.github.io/book/) — fuzz targets, corpora and structured input.
- [proptest](https://docs.rs/proptest/latest/proptest/) and [insta](https://insta.rs/docs/) — the property-testing and snapshot-testing libraries.

---

[🏠 Start](../START-HERE.md) · [← 09.12 Continuous integration: GitHub Actions and GitLab CI](12-continuous-integration.md) · [📋 Phase 09](README.md) · [09.14 Checkpoint →](14-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
