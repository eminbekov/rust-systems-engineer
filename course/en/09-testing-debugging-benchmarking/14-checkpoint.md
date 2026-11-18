<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.13 Project: a well-tested library with CI](13-project-tested-library-with-ci.md) · [📋 Phase 09](README.md) · [Phase 10 — Security fundamentals →](../10-security-fundamentals/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/14-checkpoint.md) <!-- nav -->

# Lesson 09.14 — Checkpoint

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 14 of 14 · ⏱ about 120 minutes

## What you will learn

- Whether you can choose the right kind of test for a problem and explain why.
- Whether you can find a bug with a property test, a fuzzer, a backtrace and a debugger.
- Whether you can measure two solutions honestly and write a CI (continuous integration) workflow that guards a project.
- Which lesson of this phase to read again for each gap.

## Before you start

- You finished [Lesson 09.13 — Project: a well-tested library with CI](13-project-tested-library-with-ci.md). If the project is not finished, finish it first: it is the best preparation.
- Do the self-test **without looking** at the lessons. Write your answers on paper or in a file, then open the hidden answers. The practical tasks may be done with the lessons open.
- Tools: `cargo-fuzz` with the nightly toolchain, `rust-lldb`, `actionlint`, and Docker (optional). Your code goes in `~/projects/rust/code/p09-checkpoint`.

## New words

| Word | Meaning |
|---|---|
| **crossover point** | The input size at which one solution stops being faster than another. A benchmark over several sizes finds it. |
| **allowed failure** | A CI job whose red result does not turn the whole pipeline red (`continue-on-error`), for example a test run on the beta compiler. |
| **oracle** | A trusted way to compute the right answer, such as a slow and simple version of a function, used by a test to judge the fast version. |

## Self-test quiz

Answer each question in your own words first.

1. A command-line program must exit with code 2 and print a message to the error output when its argument is wrong. Which level of the test pyramid can check this, and why not a unit test?

<details>
<summary>Answer</summary>

An end-to-end test: it starts the real program as a separate process (for example with `assert_cmd`) and sees the exit code and the standard error output. A unit test calls a function inside the process, so it cannot see the process exit code or the output streams. (Lessons 09.1 and 09.2.)

</details>

2. Why does a healthy suite have many unit tests, fewer integration tests and few end-to-end tests?

<details>
<summary>Answer</summary>

Lower levels are faster, more precise (a failure points to a small piece of code) and less fragile. Higher levels give more confidence that the parts work together, but they are slower and fail for more reasons. Each check should run at the lowest level that can still catch the problem. (Lesson 09.1.)

</details>

3. What is the difference between a stub, a fake and a mock?

<details>
<summary>Answer</summary>

A stub returns fixed, prepared answers. A fake has a real but simplified implementation (an in-memory `HashMap` instead of a database). A mock is told in advance which calls to expect and makes the test fail if the calls differ. Prefer real objects, then stubs and fakes; use mocks only when the call itself is the result you must check. (Lesson 09.3.)

</details>

4. Name three patterns for finding a property to test, and say what "shrinking" is.

<details>
<summary>Answer</summary>

Patterns: round trip (decode after encode gives the original), invariant (the result is always sorted, the length is unchanged), oracle (compare with a slow and simple version), idempotence (doing it twice equals doing it once), bounds / no panic. Shrinking: when a generated input fails, `proptest` tries smaller and simpler inputs that still fail and reports a minimal one. (Lesson 09.4.)

</details>

5. A `proptest` run fails and creates a `proptest-regressions` folder. What do you do with it?

<details>
<summary>Answer</summary>

Commit it. It stores the failing seed, so the same case is tried first on every later run (including in CI) and the bug cannot return unnoticed. You also fix the code, and may write the shrunk input as a normal example test. (Lesson 09.4.)

</details>

6. A snapshot test fails with a diff. What are the two correct reactions, and which file types do you commit?

<details>
<summary>Answer</summary>

Read the diff. If the change is intended, accept it (`cargo insta accept` or `cargo insta review`); if it is not, fix the code. Never accept without reading. Commit the `.snap` files; do not commit `.snap.new` files or pending snapshots. On CI, `insta` does not write files and the test fails. (Lesson 09.5.)

</details>

7. Why must you watch a new test fail ("red") before making it pass?

<details>
<summary>Answer</summary>

To prove that the test can fail, and that it fails for the right reason. A test that has never been red may check nothing. (Lessons 02.8, 05.6 and 09.6.)

</details>

8. Your coverage is 100%. Does that mean the tests are good?

<details>
<summary>Answer</summary>

No. Coverage says which lines ran, not whether anything was checked: a test without assertions runs everything and verifies nothing. Mutation testing finds such weak tests. (Lesson 09.7.)

</details>

9. What is a *missed* mutant? Is every missed mutant a missing test?

<details>
<summary>Answer</summary>

`cargo mutants` changes the code in a small way (for example `>` to `>=`) and runs the tests; a missed mutant is a change that no test noticed, which usually means a missing or weak test (often a boundary). Some are *equivalent* mutants: the changed code behaves the same, so no test can notice; those can be ignored. (Lesson 09.7.)

</details>

10. Why does `cargo fuzz` need the nightly compiler, and what do you do with a crash it finds?

<details>
<summary>Answer</summary>

It uses compiler options (sanitizer and coverage instrumentation, `-Z` flags) that exist only in nightly: `cargo +nightly fuzz run <target>`. For a crash: reproduce it with the printed command, minimize it (`cargo fuzz tmin`), fix the code and add a regular regression test, because the `artifacts` and `corpus` folders are not committed. (Lesson 09.8.)

</details>

11. In LLDB (the debugger of the LLVM project), what do `next`, `step` and `finish` do, and what is a watchpoint for?

<details>
<summary>Answer</summary>

`next` runs the current line completely and stops at the next line; `step` enters the function called on the line; `finish` runs until the current function returns. A watchpoint stops the program when a memory location is written, so it answers "who changes this value?". (Lesson 09.9.)

</details>

12. How do you stop a program exactly where it panics, and why does a breakpoint on a line often stay "pending" in a release build?

<details>
<summary>Answer</summary>

Set a breakpoint on the function `rust_panic` (`breakpoint set --name rust_panic`), run, and select your own frame with `frame select`. In a release build the optimizer can inline or remove code, so some lines have no address in the machine code; debug with the debug build. (Lesson 09.9.)

</details>

13. What does `RUST_LOG=warn,my_crate=debug` do, and why should you write `#[instrument(skip_all, ...)]` for a function that takes a password?

<details>
<summary>Answer</summary>

It shows `WARN` and above for all crates, but `DEBUG` and above for `my_crate`. `#[instrument]` without `skip` records every argument with `Debug`, so the password would be written to the log; `skip_all` stops that and you list the safe fields yourself. Logs must never contain secrets. (Lesson 09.10.)

</details>

14. In a `criterion` run you see `Change within noise threshold` together with `p = 0.00 < 0.05`. What do you conclude? And why use `iter_batched`?

<details>
<summary>Answer</summary>

The difference is statistically real but smaller than the noise threshold (2%), so do not report it as an improvement or a regression. `iter_batched` prepares a fresh input for each call and does not time the preparation, which is needed when the function changes its input (like sorting). (Lesson 09.11.)

</details>

15. Which tool answers each question: (a) which of two programs is faster as a whole, (b) did my commit slow down this function, (c) which line is slow?

<details>
<summary>Answer</summary>

(a) `hyperfine`. (b) `criterion` with a saved baseline. (c) A profiler (for example `cargo flamegraph` or `samply`), which samples where the time goes. (Lesson 09.11.)

</details>

16. A test passes on your Mac and fails in CI. Give three possible causes and the first step to find out.

<details>
<summary>Answer</summary>

Different operating system (paths, line endings), a missing or different environment variable, a hidden dependence on your files or on the order or time of tests, a different compiler version, or a flaky test. First step: run the same command in a clean Linux container (`docker run … rust:1.100`) to reproduce the failure on your computer. (Lessons 09.12 and 05.6.)

</details>

17. Why does a CI workflow set `permissions: contents: read`, and why must the pipeline work with an empty cache?

<details>
<summary>Answer</summary>

The permissions line gives the job's token the smallest rights, so a compromised step cannot change the repository. The cache is only a speed-up; it can be missing or evicted, so a pipeline that depends on it is unreliable. (Lesson 09.12.)

</details>

18. In GitLab CI, which keyword plays the role of `runs-on:` and which one is the counterpart of `actions/cache`?

<details>
<summary>Answer</summary>

`image:` (a Docker image; or runner `tags:`) and `cache:`. (Lesson 09.12.)

</details>

## Practical tasks

Create the package once: `cargo new --lib code/p09-checkpoint`, then `cd code/p09-checkpoint`.

### Task 1 — Find a bug with a property test

Add `proptest` (`cargo add --dev proptest@1`). Put this function in `src/lib.rs`. It must merge two **sorted** slices into one sorted vector, and it has a bug:

```rust
/// Merges two sorted slices into one sorted vector.
pub fn merge_sorted(a: &[i32], b: &[i32]) -> Vec<i32> {
    let mut result = Vec::with_capacity(a.len() + b.len());
    let (mut i, mut j) = (0, 0);
    while i < a.len() && j < b.len() {
        if a[i] < b[j] {
            result.push(a[i]);
            i += 1;
        } else if a[i] > b[j] {
            result.push(b[j]);
            j += 1;
        } else {
            result.push(a[i]);
            i += 1;
            j += 1;
        }
    }
    result.extend_from_slice(&a[i..]);
    result.extend_from_slice(&b[j..]);
    result
}
```

Write two properties in `tests/merge_props.rs`: the result is sorted, and nothing is lost or invented (the result equals the sorted concatenation of the inputs). Let `proptest` find the bug, read the minimal failing input, and fix the function.

<details>
<summary>Hint</summary>

Generate each input as a sorted vector: `prop::collection::vec(strategy, 0..20)` followed by `.prop_map(|mut v| { v.sort(); v })`. If the properties all pass, your numbers are too widely spread: bugs about *equal* elements need equal elements, so generate small numbers such as `-5i32..5`. For the second property build the expected vector with `[a.clone(), b.clone()].concat()` and `sort()`.

</details>

<details>
<summary>Answer</summary>

`tests/merge_props.rs`:

```rust
use p09_checkpoint::merge_sorted;
use proptest::prelude::*;

fn sorted_vec() -> impl Strategy<Value = Vec<i32>> {
    prop::collection::vec(-5i32..5, 0..20).prop_map(|mut v| {
        v.sort();
        v
    })
}

proptest! {
    #[test]
    fn result_is_sorted(a in sorted_vec(), b in sorted_vec()) {
        let merged = merge_sorted(&a, &b);
        prop_assert!(merged.windows(2).all(|pair| pair[0] <= pair[1]));
    }

    #[test]
    fn nothing_is_lost_or_invented(a in sorted_vec(), b in sorted_vec()) {
        let merged = merge_sorted(&a, &b);
        let mut expected = [a.clone(), b.clone()].concat();
        expected.sort();
        prop_assert_eq!(merged, expected);
    }
}
```

```bash
cargo test --test merge_props
```

```text
test nothing_is_lost_or_invented ... FAILED
test result_is_sorted ... ok
Test failed: assertion failed: `(left == right)` 
  left: `[0]`,
 right: `[0, 0]` at tests/merge_props.rs:23.
minimal failing input: a = [
    0,
], b = [
    0,
]
```

The shrunk input is `a = [0]`, `b = [0]`: when two elements are **equal**, the `else` branch pushes only one of them and moves both indexes, so one element disappears. The fix is to treat equality like "smaller":

```rust
        if a[i] <= b[j] {
            result.push(a[i]);
            i += 1;
        } else {
            result.push(b[j]);
            j += 1;
        }
```

Now `cargo test --test merge_props` shows `2 passed`. Notice two things. The first property (`result_is_sorted`) alone would **not** have found the bug, because the buggy result is still sorted: a property that is too weak. And with `any::<i32>()` the strategy rarely produces equal numbers, so the bug can hide: the *strategy* must be able to reach the case.

</details>

### Task 2 — Read a panic and see the value

Add this program as `src/bin/report.rs`, and this function to `src/lib.rs`:

```rust
/// The average of the values, rounded down.
pub fn average(values: &[u32]) -> u32 {
    let sum: u32 = values.iter().sum();
    sum / values.len() as u32
}
```

```rust
use p09_checkpoint::average;

fn main() {
    let morning: [u32; 3] = [4, 8, 6];
    let evening: [u32; 0] = [];
    println!("morning average: {}", average(&morning));
    println!("evening average: {}", average(&evening));
}
```

Run it. (1) Which line of **your** code is the first frame of the backtrace (use `RUST_BACKTRACE=1`)? (2) With `rust-lldb`, stop at the panic and show the argument `values`. (3) Fix `average` so that it cannot panic, and add a test.

<details>
<summary>Hint</summary>

For (2) set a breakpoint on `rust_panic`, run, use `bt 8` to find the frame of `average`, and `frame select N`, then `frame variable`. For (3): what should the average of nothing be? Make the function return an `Option<u32>`; `checked_div` returns `None` for a zero divisor.

</details>

<details>
<summary>Answer</summary>

```bash
RUST_BACKTRACE=1 cargo run --bin report
```

```text
morning average: 6

thread 'main' (5604221) panicked at src/lib.rs:26:5:
attempt to divide by zero
stack backtrace:
   0: __rustc::rust_begin_unwind
   1: core::panicking::panic_fmt
   2: core::panicking::panic_const::panic_const_div_by_zero
   3: p09_checkpoint::average
             at ./src/lib.rs:26:5
   4: report::main
             at ./src/bin/report.rs:7:37
```

(1) Frame 3: `p09_checkpoint::average` at `src/lib.rs:26:5` (your line numbers may differ), called from `report::main` line 7. (2) In `rust-lldb`:

```text
(lldb) breakpoint set --name rust_panic
(lldb) run
(lldb) bt 8
...
    frame #7: ... report`p09_checkpoint::average(values=size=0) at lib.rs:26:5
(lldb) frame select 7
(lldb) frame variable
```

`values=size=0`: the slice is empty, and `0` is the divisor. (3) The fix and the test:

```rust
/// The average of the values, rounded down; `None` for no values.
pub fn average(values: &[u32]) -> Option<u32> {
    let sum: u32 = values.iter().sum();
    sum.checked_div(values.len() as u32)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn average_of_nothing_is_none() {
        assert_eq!(average(&[]), None);
        assert_eq!(average(&[4, 8, 6]), Some(6));
    }
}
```

Update `report.rs` to print `average(&morning).unwrap()` and `average(&evening).unwrap_or(0)`: the program prints `morning average: 6` and `evening average: 0`. Why it works: the error is no longer possible, and the type `Option` forces every caller to decide what "no values" means. (The sum itself could overflow for huge inputs; a robust version would use `u64` or `checked_add`.)

</details>

### Task 3 — Let a fuzzer break a function

Add this function to `src/lib.rs`:

```rust
/// Splits `key=value` into its two parts.
pub fn split_key_value(line: &str) -> (&str, &str) {
    let at = line.find('=').unwrap();
    (&line[..at], &line[at + 1..])
}
```

Create a fuzz target for it (`cargo fuzz init`), run it for 20 seconds, report the crashing input, fix the function and add a regression test.

<details>
<summary>Hint</summary>

The target body: convert the bytes to `&str` with `std::str::from_utf8(data)` (return early on `Err`), then call the function. Run with `cargo +nightly fuzz run fuzz_target_1 -- -max_total_time=20`. For the fix, look at `str::split_once`.

</details>

<details>
<summary>Answer</summary>

`fuzz/fuzz_targets/fuzz_target_1.rs`:

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(line) = std::str::from_utf8(data) {
        let _ = p09_checkpoint::split_key_value(line);
    }
});
```

```text
thread '<unnamed>' panicked at src/lib.rs:31:29:
called `Option::unwrap()` on a `None` value
...
Failing input:

	fuzz/artifacts/fuzz_target_1/crash-da39a3ee5e6b4b0d3255bfef95601890afd80709

Output of `std::fmt::Debug`:

	[]
```

The crashing input is the **empty** input: a line without `=` makes `find` return `None`, and `unwrap` panics. Every text without `=` crashes it. The fix returns an `Option`, as `split_once` does:

```rust
/// Splits `key=value` into its two parts; `None` when there is no `=`.
pub fn split_key_value(line: &str) -> Option<(&str, &str)> {
    line.split_once('=')
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn key_value_without_equals_sign() {
        assert_eq!(split_key_value(""), None);
        assert_eq!(split_key_value("a=b=c"), Some(("a", "b=c")));
    }
}
```

After the fix, a 10-second run (`-max_total_time=10`) ends with `Done 16473967 runs in 11 second(s)` and no `Failing input`. The regression test keeps the case forever. Why it works: `split_once` splits at the first `=` and returns `None` when there is none, so the failure is a value in the type and not a panic.

</details>

### Task 4 — Measure two solutions honestly

Is `Vec::contains` or `HashSet::contains` faster for looking up a number that is **not** in the collection? Write a `criterion` benchmark (`cargo add --dev criterion@0.8`, `[[bench]]` with `harness = false`) for sizes 10, 1,000 and 100,000, and answer: is there a crossover point?

<details>
<summary>Hint</summary>

Use a `benchmark_group`, a loop over the sizes, `bench_with_input(BenchmarkId::new(name, size), ...)`, and `black_box` on the number you search. Build the vector and the set **outside** the measured closure. Search for `size + 1`, which is never in the collection. Run with `cargo bench --bench lookup -- --warm-up-time 1 --measurement-time 2`.

</details>

<details>
<summary>Answer</summary>

`benches/lookup.rs`:

```rust
use criterion::{BenchmarkId, Criterion, criterion_group, criterion_main};
use std::collections::HashSet;
use std::hint::black_box;

fn bench_lookup(c: &mut Criterion) {
    let mut group = c.benchmark_group("lookup of a missing number");
    for size in [10u32, 1_000, 100_000] {
        let vec: Vec<u32> = (0..size).collect();
        let set: HashSet<u32> = vec.iter().copied().collect();
        let missing = size + 1;
        group.bench_with_input(BenchmarkId::new("Vec::contains", size), &vec, |b, vec| {
            b.iter(|| vec.contains(black_box(&missing)))
        });
        group.bench_with_input(BenchmarkId::new("HashSet::contains", size), &set, |b, set| {
            b.iter(|| set.contains(black_box(&missing)))
        });
    }
    group.finish();
}

criterion_group!(benches, bench_lookup);
criterion_main!(benches);
```

```text
lookup of a missing number/Vec::contains/10
                        time:   [1.8436 ns 1.8905 ns 1.9437 ns]
lookup of a missing number/HashSet::contains/10
                        time:   [4.1029 ns 4.1467 ns 4.1946 ns]
lookup of a missing number/Vec::contains/1000
                        time:   [44.127 ns 51.155 ns 59.305 ns]
lookup of a missing number/HashSet::contains/1000
                        time:   [4.6973 ns 4.8279 ns 4.9711 ns]
lookup of a missing number/Vec::contains/100000
                        time:   [3.7518 µs 3.7938 µs 3.8507 µs]
lookup of a missing number/HashSet::contains/100000
                        time:   [4.7922 ns 4.8823 ns 4.9744 ns]
```

(Your numbers will differ, and the 1,000 case of `Vec` shows a wide interval: repeat it on a quiet computer.) The picture: the `HashSet` takes about 4 to 5 ns **whatever the size** (it computes a hash and looks in one place). The `Vec` has to walk through all elements, so its time grows with the size: 1.9 ns for 10 numbers, about 50 ns for 1,000 and 3.8 µs for 100,000. The **crossover point** is somewhere between 10 and 1,000 elements: for about 10 items the `Vec` is faster (no hashing), and at 1,000 items the `HashSet` already wins by a factor of ten. Where exactly the lines cross, only more sizes (for example 20, 50, 100, 200) can tell. The honest conclusion has a condition: "for lookups of missing values in collections of 1,000 items or more, use a `HashSet`; for about 10 items the `Vec` is at least as good". A lookup of a value that is found at the start of the `Vec` would give another picture, so measure the case your program really has.

</details>

### Task 5 — Write a CI workflow for a specification

Write `.github/workflows/ci.yml` for a Rust repository that meets all of these requirements: it runs on pushes to `main` and on pull requests; an older run of the same branch is cancelled when a new commit arrives; the token can only read the repository; it runs `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all-targets` and `cargo test --doc`; it uses a cache; and it runs on **both** the `stable` and the `beta` compiler, but a failure on `beta` must not turn the pipeline red. Check it with `actionlint`.

<details>
<summary>Hint</summary>

Use `strategy: matrix: toolchain: [stable, beta]` and `${{ matrix.toolchain }}`. The setting that allows a job to fail is `continue-on-error` at the job level, with an expression: `${{ matrix.toolchain == 'beta' }}`. Install with `rustup toolchain install … --profile minimal --component rustfmt,clippy` and select it with `rustup default …`.

</details>

<details>
<summary>Answer</summary>

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  test:
    name: test (${{ matrix.toolchain }})
    runs-on: ubuntu-latest
    continue-on-error: ${{ matrix.toolchain == 'beta' }}
    strategy:
      fail-fast: false
      matrix:
        toolchain: [stable, beta]
    steps:
      - uses: actions/checkout@v7
      - name: Install Rust ${{ matrix.toolchain }}
        run: |
          rustup toolchain install ${{ matrix.toolchain }} --profile minimal --component rustfmt,clippy
          rustup default ${{ matrix.toolchain }}
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test --all-targets
      - run: cargo test --doc
```

`actionlint` prints nothing (checked, from the root of a Git repository). Why it works: `concurrency` with `cancel-in-progress` cancels older runs; `permissions: contents: read` is the smallest token; the matrix makes two jobs; `continue-on-error` is true only for `beta`, so a red `beta` job is shown but does not fail the run; `fail-fast: false` lets both jobs finish. The `beta` job warns you early about a problem with a coming compiler version, without blocking your work.

</details>

## If you got something wrong

| Topic | Lesson to read again |
|---|---|
| Test pyramid, what to test where | [09.1 The test pyramid and a testing strategy](01-the-test-pyramid.md) |
| End-to-end tests of programs | [09.2 End-to-end tests for command-line programs](02-testing-command-line-programs.md) |
| Stubs, fakes, mocks | [09.3 Test doubles](03-test-doubles.md) |
| Properties, shrinking, strategies | [09.4 Property-based testing](04-property-based-testing.md) |
| Snapshots and `cargo insta` | [09.5 Snapshot testing](05-snapshot-testing.md) |
| Red, green, refactor | [09.6 Test-driven development in practice](06-tdd-in-practice.md) and [05.6 Testing in depth](../05-rust-intermediate/06-testing-in-depth.md) |
| Coverage and mutants | [09.7 Code coverage and mutation testing](07-code-coverage-and-mutation-testing.md) |
| Fuzzing | [09.8 Fuzzing](08-fuzzing.md) |
| Debugger, backtrace, watchpoints | [09.9 Debugging in depth](09-debugging-with-lldb-and-rustrover.md) and [02.9 Debugging basics](../02-programming-basics/09-debugging-basics.md) |
| Logs, spans, `RUST_LOG` | [09.10 Logging and tracing](10-logging-and-tracing.md) |
| Benchmarks, baselines, noise | [09.11 Benchmarking in depth](11-benchmarking-in-depth.md) and [05.12 Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md) |
| CI, `actionlint`, GitLab CI | [09.12 Continuous integration](12-continuous-integration.md) and [01.9 Teamwork and pull requests](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| Putting it all together | [09.13 Project: a well-tested library with CI](13-project-tested-library-with-ci.md) |

## What's next

You can now build a safety net around code, find a bug with the right tool, measure honestly and let CI guard your work. **Phase 10** looks at software from the attacker's side: [Phase 10 — Security fundamentals](../10-security-fundamentals/README.md) covers threat modeling, input validation and injection, authentication, secrets, cryptography basics, and supply-chain checks with `cargo audit` and `cargo deny`. The pipeline that you built in this phase is the natural place to run those checks, and a fuzzer and a good test suite are security tools too.

## Sources and further reading

- [The Rust Programming Language — 11. Writing Automated Tests](https://doc.rust-lang.org/book/ch11-00-testing.html) — unit, integration and doc tests, the base of every task above.
- [The Proptest Book](https://proptest-rs.github.io/proptest/intro.html) — strategies and shrinking, for Task 1.
- [The Rust Fuzz Book](https://rust-fuzz.github.io/book/) — `cargo fuzz` targets, artifacts and `tmin`, for Task 3.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — benchmark groups, `BenchmarkId` and baselines, for Task 4.
- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions) — `strategy.matrix`, `continue-on-error` and `concurrency`, for Task 5.
- [LLDB tutorial](https://lldb.llvm.org/use/tutorial.html) — breakpoints, `bt`, `frame select` and `frame variable`, for Task 2.

---

[🏠 Start](../START-HERE.md) · [← 09.13 Project: a well-tested library with CI](13-project-tested-library-with-ci.md) · [📋 Phase 09](README.md) · [Phase 10 — Security fundamentals →](../10-security-fundamentals/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
