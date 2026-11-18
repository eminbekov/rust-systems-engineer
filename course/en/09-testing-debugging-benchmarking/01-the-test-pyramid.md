<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 09 — Testing, debugging and benchmarking](README.md) · [📋 Phase 09](README.md) · [09.2 End-to-end tests for command-line programs →](02-testing-command-line-programs.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/01-the-test-pyramid.md) <!-- nav -->

# Lesson 09.1 — The test pyramid and a testing strategy

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 1 of 14 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The three levels of tests (unit, integration, end-to-end), what each one costs and what each one proves.
- Why a healthy test suite has the shape of a pyramid, and what goes wrong when the shape is upside down.
- How one single bug looks at each level (you will see it with real output).
- A short set of rules for deciding *what to test where*, and for keeping tests reliable.

## Before you start

- You finished [Lesson 08.17 — Checkpoint](../08-software-design/17-checkpoint.md) (Phase 08).
- You know how to write a test and run `cargo test` ([Lesson 02.8](../02-programming-basics/08-your-first-tests.md)), and you know unit tests, integration tests in `tests/`, doc tests and the `fmt` + `clippy` + `test` quality gate ([Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)). This lesson does not repeat them. It asks a new question: *with all these kinds of tests available, which ones should you write, and how many?*
- **How the code of this phase is organised.** Every lesson of Phase 09 has its own package in `code/p09-<topic>`, for example `code/p09-pyramid` in this lesson. Library code is in `src/lib.rs`, a program's entry point in `src/main.rs`, and extra tests in the `tests/` folder. When a lesson shows an output, it is the output of the file named in the text.

## New words

| Word | Meaning |
|---|---|
| **test level** | The scope of a test: how much of the program it runs at once. We use three levels: unit, integration, end-to-end. |
| **unit test** | A test of one small piece (one function or type), usually in the same file, inside `#[cfg(test)] mod tests`. |
| **integration test** | A test in the `tests/` folder that uses your crate from the outside, through its public API (Application Programming Interface: the items your crate offers to other code), without starting a program. |
| **end-to-end test** (E2E) | A test that starts the *real finished program*, gives it input the way a user would, and checks what comes out. "End to end" means: from the user's input to the program's output. |
| **test pyramid** | A picture and a rule of thumb: many fast unit tests at the bottom, fewer integration tests in the middle, a few E2E tests at the top. |
| **confidence** | How sure a passing test makes you that the program really works for a user. |
| **feedback time** | How long you wait to learn that something broke. Short feedback time makes you fast. |
| **deterministic** | Always gives the same result for the same input. A test must be deterministic. |
| **flaky test** | A test that sometimes passes and sometimes fails without any code change (Lesson 05.6). |
| **stdout / stderr** (standard output / standard error) | The two text channels of a program: normal results go to stdout, error messages go to stderr. |
| **exit code** | A number a program returns to the system when it ends. `0` means success; anything else means failure. |
| **process** | A running program. Starting a process is much slower than calling a function. |
| **Arrange–Act–Assert** (AAA) | A way to structure a test: prepare the data (arrange), call the code (act), check the result (assert). |

## 1. Why more than one kind of test?

Imagine you build a car. You check each bolt (is it tight?), you check the engine on a test bench, and at the end
you take the whole car for a test drive. Each check has a different price:

- Checking one bolt is very cheap and tells you exactly which bolt is loose, but it says nothing about whether the car drives.
- The test drive is the only check that tells you the *whole car* works, but it takes an hour, and if the car
  does not start you still have to find out why.

Tests are the same. Every test makes three trade-offs:

1. **Speed.** How long does it run? (This is your feedback time.)
2. **Precision.** When it fails, does the message point at the broken line, or only say "something is wrong"?
3. **Confidence.** How much of the real program does it exercise? A test that runs more real code proves more.

No test is best at all three. That is why we use three levels:

| | Unit | Integration | End-to-end |
|---|---|---|---|
| **Runs** | one function or type | the public API of the crate | the real, built program in its own process |
| **Speed** | microseconds | microseconds to milliseconds | hundreds of milliseconds |
| **Precision** | points at one function | points at one feature | says only "the output is wrong" |
| **Confidence** | low (one piece) | medium (pieces work together) | high (what the user gets) |
| **Breaks when** | the function changes | the public API changes | anything changes, even a message text |

The rest of this lesson shows each level on one small program.

## 2. The example program: a bill splitter

Three friends eat dinner; the bill is 10.00. Divided by three, each owes 3.3333… Money cannot be split into
fractions of a cent, so someone pays one cent more. Our program does this fairly: the shares always add up to the
exact total, and the first people pay the extra cents. The program is run like this:

```text
p09-pyramid 10.00 3
```

Create the package (a library plus a program in one package; Lesson 04.6 explains `src/lib.rs` and `src/main.rs`):

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/09-01-test-pyramid
cargo new --lib code/p09-pyramid
```

Replace `src/lib.rs` with the code below (the tests come in the next sections):

```rust
//! Bill splitting: share a total amount of money fairly between people.

use std::fmt;

/// Everything that can go wrong when a user runs the program.
#[derive(Debug, PartialEq, Eq)]
pub enum CliError {
    /// The wrong number of arguments.
    Usage,
    /// The amount of money is not a valid number like `12.50`.
    BadAmount(String),
    /// The number of people is not a whole number of at least 1.
    BadPeople(String),
}

impl fmt::Display for CliError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            CliError::Usage => write!(f, "usage: p09-pyramid <total> <people>"),
            CliError::BadAmount(text) => write!(f, "'{text}' is not a valid amount"),
            CliError::BadPeople(text) => write!(f, "'{text}' is not a valid number of people"),
        }
    }
}

impl std::error::Error for CliError {}

/// Turns text like `12.5` or `7` into cents (`1250` or `700`).
pub fn parse_money(text: &str) -> Option<u64> {
    let (whole, fraction) = match text.split_once('.') {
        Some((whole, fraction)) => (whole, fraction),
        None => (text, ""),
    };
    if whole.is_empty() || fraction.len() > 2 {
        return None;
    }
    let all_digits = |part: &str| part.chars().all(|c| c.is_ascii_digit());
    if !all_digits(whole) || !all_digits(fraction) {
        return None;
    }
    let cents_from_fraction = match fraction.len() {
        0 => 0,
        1 => fraction.parse::<u64>().ok()? * 10,
        _ => fraction.parse::<u64>().ok()?,
    };
    whole
        .parse::<u64>()
        .ok()?
        .checked_mul(100)?
        .checked_add(cents_from_fraction)
}

/// Turns cents back into text: `1250` becomes `12.50`.
pub fn format_money(cents: u64) -> String {
    format!("{}.{:02}", cents / 100, cents % 100)
}

/// Splits `total_cents` between `people`. The shares add up to the total exactly;
/// the first `total_cents % people` people pay one cent more than the others.
pub fn split_bill(total_cents: u64, people: u64) -> Vec<u64> {
    let base = total_cents / people;
    let extra = total_cents % people;
    (0..people)
        .map(|i| if i < extra { base + 1 } else { base })
        .collect()
}

/// Runs the whole program for the given arguments (without the program name)
/// and returns the text to print.
pub fn run(args: &[String]) -> Result<String, CliError> {
    let [total, people] = args else {
        return Err(CliError::Usage);
    };
    let total_cents = parse_money(total).ok_or_else(|| CliError::BadAmount(total.clone()))?;
    let people = match people.parse::<u64>() {
        Ok(n) if n >= 1 => n,
        _ => return Err(CliError::BadPeople(people.clone())),
    };
    let lines: Vec<String> = split_bill(total_cents, people)
        .iter()
        .enumerate()
        .map(|(i, cents)| format!("person {}: {}", i + 1, format_money(*cents)))
        .collect();
    Ok(lines.join("\n"))
}
```

Create `src/main.rs`:

```rust
fn main() {
    let args: Vec<String> = std::env::args().skip(1).collect();
    match p09_pyramid::run(&args) {
        Ok(text) => println!("{text}"),
        Err(error) => {
            eprintln!("error: {error}");
            std::process::exit(2);
        }
    }
}
```

**What happens:**

1. `parse_money` splits the text at the dot. It rejects an empty whole part, more than two decimals and any
   character that is not a digit, so `"-3"`, `"1.234"` and `"abc"` all give `None`. It uses `checked_mul` and
   `checked_add`, which return `None` instead of overflowing (Lesson 05.6).
2. `split_bill` gives everybody `total / people` cents (integer division drops the remainder), and the first
   `total % people` people get one more cent. The remainder is always smaller than `people`, so each
   person gets at most one extra cent.
3. `run` is the whole program as a *function*: arguments in, text or error out. The line
   `let [total, people] = args else { … }` matches a slice of exactly two items; any other length goes to the
   `else` branch and becomes a usage error.
4. `main` is only a thin shell. It collects the arguments, calls `run`, prints the text on stdout, or prints the
   error on stderr and ends with exit code 2. Because `main` has almost no logic, there is almost nothing in it that can break.

> 📏 **Convention:** Keep `main` tiny and put the logic in a library function such as `run`. A function that takes
> values and returns values is easy to test; `main`, which touches the real terminal, is not. This idea is
> called "a thin shell around a testable core".

Try it by hand:

```bash
cd code/p09-pyramid
cargo run -q -- 10.00 3
cargo run -q -- ten 3
echo "exit code: $?"
```

```text
person 1: 3.34
person 2: 3.33
person 3: 3.33
error: 'ten' is not a valid amount
exit code: 2
```

## 3. Level 1: unit tests

A unit test checks one small piece, in isolation. Add this to the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_whole_and_fractional_amounts() {
        assert_eq!(parse_money("12.50"), Some(1250));
        assert_eq!(parse_money("12.5"), Some(1250));
        assert_eq!(parse_money("7"), Some(700));
    }

    #[test]
    fn rejects_text_that_is_not_money() {
        assert_eq!(parse_money(""), None);
        assert_eq!(parse_money("-3"), None);
        assert_eq!(parse_money("1.234"), None);
        assert_eq!(parse_money("abc"), None);
    }

    #[test]
    fn formats_cents_with_two_digits() {
        assert_eq!(format_money(1250), "12.50");
        assert_eq!(format_money(5), "0.05");
    }

    #[test]
    fn leftover_cents_go_to_the_first_people() {
        assert_eq!(split_bill(1000, 3), vec![334, 333, 333]);
    }

    #[test]
    fn even_split_has_no_leftovers() {
        assert_eq!(split_bill(900, 3), vec![300, 300, 300]);
    }
}
```

```bash
cargo test --lib
```

```text
running 5 tests
test tests::even_split_has_no_leftovers ... ok
test tests::formats_cents_with_two_digits ... ok
test tests::leftover_cents_go_to_the_first_people ... ok
test tests::rejects_text_that_is_not_money ... ok
test tests::parses_whole_and_fractional_amounts ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:** every test calls one function with small, hand-picked inputs and compares the result with the
expected value. The five tests together take well under a millisecond. You can write hundreds of tests like this
and still wait no time at all. When one fails, its name says which function is wrong (Lesson 05.6 explained naming
tests as sentences).

> 🧪 **Testing:** Unit tests are the right place for *edge cases*: empty text, zero, the biggest number, a negative sign.
> They are cheap, so cover every unusual input here, not at higher levels.

## 4. Level 2: integration tests

An integration test checks that the pieces work *together*, through the public API only. Create `tests/api.rs`:

```rust
use p09_pyramid::{CliError, run};

fn args(items: &[&str]) -> Vec<String> {
    items.iter().map(|s| s.to_string()).collect()
}

#[test]
fn splits_ten_dollars_between_three_people() {
    let output = run(&args(&["10.00", "3"])).unwrap();
    assert_eq!(output, "person 1: 3.34\nperson 2: 3.33\nperson 3: 3.33");
}

#[test]
fn zero_people_is_an_error() {
    assert_eq!(
        run(&args(&["10.00", "0"])),
        Err(CliError::BadPeople("0".to_string()))
    );
}

#[test]
fn wrong_number_of_arguments_is_a_usage_error() {
    assert_eq!(run(&args(&["10.00"])), Err(CliError::Usage));
}
```

```bash
cargo test --test api
```

```text
running 3 tests
test wrong_number_of_arguments_is_a_usage_error ... ok
test zero_people_is_an_error ... ok
test splits_ten_dollars_between_three_people ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. The file in `tests/` is compiled as a separate crate and imports `p09_pyramid` like any user would. It cannot
   call private items.
2. `args` is a small helper that turns `&str` values into the `Vec<String>` that `run` wants.
3. The tests call `run`, the same function `main` calls, so the parsing, the splitting and the formatting are
   checked *together*. They check the error cases too (`BadPeople`, `Usage`), because those are the paths where
   the pieces must agree.

These tests still do not start a program, so they are about as fast as unit tests. What they cannot check is `main`
itself: whether the text really reaches stdout, and whether the exit code is right.

## 5. Level 3: end-to-end tests

An E2E test runs the real program. Cargo builds the binary before it runs the tests in `tests/` and tells the test
where the binary is, through the environment variable `CARGO_BIN_EXE_<name>` (here `<name>` is the program name
`p09-pyramid`). Create `tests/e2e.rs`:

```rust
use std::process::Command;

fn program() -> Command {
    Command::new(env!("CARGO_BIN_EXE_p09-pyramid"))
}

#[test]
fn prints_the_shares_and_exits_with_zero() {
    let output = program().args(["10.00", "3"]).output().unwrap();
    assert!(output.status.success());
    let stdout = String::from_utf8(output.stdout).unwrap();
    assert_eq!(stdout, "person 1: 3.34\nperson 2: 3.33\nperson 3: 3.33\n");
}

#[test]
fn bad_input_prints_to_stderr_and_exits_with_two() {
    let output = program().args(["ten", "3"]).output().unwrap();
    assert_eq!(output.status.code(), Some(2));
    let stderr = String::from_utf8(output.stderr).unwrap();
    assert_eq!(stderr, "error: 'ten' is not a valid amount\n");
}
```

```bash
cargo test --test e2e
```

```text
running 2 tests
test prints_the_shares_and_exits_with_zero ... ok
test bad_input_prints_to_stderr_and_exits_with_two ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.14s
```

**What happens:**

1. `env!("CARGO_BIN_EXE_p09-pyramid")` is read **when the test is compiled** and becomes the path of the built
   program, something like `…/target/debug/p09-pyramid`.
2. `Command::new(path).args([...]).output()` (from the standard library) starts the program as a new process, waits
   for it to end, and collects its stdout, stderr and exit status.
3. The first test checks the success path: exit status zero and the exact text on stdout (note the final `\n`
   that `println!` adds). The second checks the failure path: exit code 2 and the exact message on stderr.
4. Only an E2E test can prove those two facts. If someone changed `eprintln!` to `println!` in `main`, no unit test
   and no integration test would notice.

The two tests need about 0.14 seconds, while the five unit tests and three integration tests together need less than
0.01 seconds. The exact numbers depend on the computer, but the ratio is always large: starting a process is
slow compared with calling a function.

> ⚡ **Performance:** A test suite with 1,000 E2E tests of 70 ms each takes more than a minute. The same checks as unit tests
> take under a second. Slow tests get run less often, and tests that are not run protect nothing. Move every
> check you can down to a lower level.

Phase 09 teaches a better tool for E2E tests of command-line programs in [Lesson 09.2](02-testing-command-line-programs.md);
here the standard library is enough to see the idea.

## 6. One bug, three levels

Let us break the code on purpose and see which tests notice. In `split_bill` change `i < extra` to `i <= extra`
(a classic off-by-one mistake: one person too many gets the extra cent):

```rust
        .map(|i| if i <= extra { base + 1 } else { base })
```

Run all test files, even if some fail (`--no-fail-fast`, explained in "Common mistakes"):

```bash
cargo test --no-fail-fast
```

The important lines of the real output:

```text
     Running unittests src/lib.rs (…)
test tests::leftover_cents_go_to_the_first_people ... FAILED
test tests::even_split_has_no_leftovers ... FAILED
---- tests::leftover_cents_go_to_the_first_people stdout ----
assertion `left == right` failed
  left: [334, 334, 333]
 right: [334, 333, 333]
---- tests::even_split_has_no_leftovers stdout ----
assertion `left == right` failed
  left: [301, 300, 300]
 right: [300, 300, 300]
test result: FAILED. 3 passed; 2 failed; …

     Running tests/api.rs (…)
test splits_ten_dollars_between_three_people ... FAILED
assertion `left == right` failed
  left: "person 1: 3.34\nperson 2: 3.34\nperson 3: 3.33"
 right: "person 1: 3.34\nperson 2: 3.33\nperson 3: 3.33"
test result: FAILED. 2 passed; 1 failed; …

     Running tests/e2e.rs (…)
test prints_the_shares_and_exits_with_zero ... FAILED
assertion `left == right` failed
  left: "person 1: 3.34\nperson 2: 3.34\nperson 3: 3.33\n"
 right: "person 1: 3.34\nperson 2: 3.33\nperson 3: 3.33\n"
test result: FAILED. 1 passed; 1 failed; …
```

**What happens:** the same bug fails tests at all three levels, but the messages are different.

- The **unit tests** fail with the names of the two cases and small arrays. Looking at `[301, 300, 300]` where
  `[300, 300, 300]` was expected, you see at once: "even a perfect split gave someone an extra cent", so the problem
  is in `split_bill`, in the choice of who gets the extra cent. The failure is *precise*.
- The **integration test** and the **E2E test** fail with long strings. They tell you *that* the result is wrong, but
  you must read the text and think to find *where*.
- Three failures for one bug is normal and not a problem. The skill is to read the failures from the bottom of the
  pyramid up: fix the unit test failures first, and the higher ones usually turn green by themselves.

Change `<=` back to `<` before you go on.

## 7. The pyramid, and the ice-cream cone

Put the three levels one above another, the cheap one at the bottom:

```text
          /\
         /E2E\          few: the happy path and one or two failures
        /------\
       /integra-\       some: features working together
      /  tion    \
     /------------\
    /  unit tests  \    many: every function, every edge case
   /________________\
```

The wider the layer, the more tests it has. The logic behind the shape:

- The bottom is fast and precise, so you can afford a lot of it. This is where you check the *many* combinations of
  inputs.
- The top is slow and fragile (it breaks when anything changes), so you write only as many E2E tests as you need to
  prove that the pieces are really wired together: usually the main success path and one error path.

The opposite shape is called the **ice-cream cone**: many E2E tests (usually written by hand-clicking or by long
scripts), few unit tests. The team's suite is slow, fails for hard-to-find reasons, and people stop trusting it.
If you see a project where `cargo test` takes ten minutes, look for the cone.

The pyramid is a guideline, not a law. If your program is a thin wrapper over a database, integration tests may
matter more than unit tests. What matters is the reasoning: **run each check at the lowest level that can still
catch the problem.**

## 8. A testing strategy: what to test where

Use this table when you do not know where to put a test.

| You want to check… | Put the test at… | Example here |
|---|---|---|
| a calculation, a parser, an edge case | unit | `parse_money("1.234")` gives `None` |
| that two or more pieces work together | integration | `run` with `["10.00", "3"]` |
| an error that crosses several pieces | integration | `run` with `["10.00", "0"]` gives `BadPeople` |
| what the user sees: stdout, stderr, exit code | E2E | `ten 3` exits with code 2 |
| that `main` is wired to the library | E2E | one success test |
| a bug someone reported | the **lowest** level that reproduces it | usually a unit test |

And these six rules make the tests useful over time:

1. **Test behaviour, not implementation.** A test should say *what* the code promises ("the shares add up to the
   total"), not *how* it works inside ("it calls `checked_add` twice"). Then you can refactor freely (rewrite the
   inside without changing behaviour) and the tests stay green.
2. **Arrange, Act, Assert.** Write each test in three visible parts: prepare data, call the code, check the result.
   In our tests the "arrange" is often just the arguments, but when it grows, separate the parts with a blank
   line.
3. **One reason to fail.** A test that checks one behaviour tells you what is broken. A test that checks ten things
   tells you only that one of ten is broken. (Several `assert_eq!` calls on *the same* behaviour are fine.)
4. **Be deterministic.** A test must give the same answer every run. The usual enemies are the real clock, random
   numbers, the network, the order of items in a `HashMap`, and tests that share a file. Remove them from the code under
   test by passing them in as parameters (you will do this with test doubles in
   [Lesson 09.3](03-test-doubles.md)).
5. **Every bug gets a test first.** When you find a bug, first write a test that fails because of it
   (red), then fix the code (green). The bug can then never come back unnoticed (a *regression test*, Lesson 05.6).
6. **Keep the gate.** Before every commit run the quality gate from Lesson 05.6:
   `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`.

> 🔒 **Security:** Tests are code in your repository. Never put real passwords, tokens or customer data into test
> files or test output. Use obviously fake values such as `"test-password"`.

## 9. Running one level at a time

You do not always want to run everything. `cargo test --lib` runs only the unit tests of the library (fastest; use it
while coding). `cargo test --test api` and `cargo test --test e2e` run one file of `tests/`. `cargo test --doc` runs
the documentation examples. `cargo test leftover` runs the tests whose name contains `leftover`. `cargo test --no-fail-fast`
runs every test file even after a failure. A good rhythm: `cargo test --lib` while you code, the whole gate before a commit.

## Common mistakes

**1. Expecting `cargo test` to show all failures.** When a test file fails, Cargo stops before it starts the next
file. If the unit tests fail you will not see the integration and E2E results at all:

```text
error: test failed, to rerun pass `--lib`
```

Notice that the text does not mention `tests/api.rs` or `tests/e2e.rs`: they were never started. Use
`cargo test --no-fail-fast` when you want the full picture, as in Section 6.

**2. Using `CARGO_BIN_EXE_…` in a unit test.** The variable exists only for the files in `tests/`. If you write
`env!("CARGO_BIN_EXE_p09-pyramid")` inside `src/lib.rs`, the compiler says:

```text
error: environment variable `CARGO_BIN_EXE_p09-pyramid` not defined at compile time
   --> src/lib.rs:127:20
    |
127 |         let path = env!("CARGO_BIN_EXE_p09-pyramid");
    |                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |
    = help: `CARGO_BIN_EXE_p09-pyramid` may not be available for the current Cargo target
```

**What it means:** unit tests are built for the library alone, without the binary. An E2E test belongs in `tests/`.

**3. Testing everything end to end.** "It is safer if the real program runs." It is not: the suite becomes slow and
every failure is vague (Section 7, the ice-cream cone). Check each rule once at the lowest level, and keep the E2E
tests for the wiring.

## Try it yourself

1. Create the package and the three files as shown. Run `cargo test` and check that you see five unit tests, three
   integration tests and two E2E tests, all green.
2. Run `cargo test --lib`, `cargo test --test api` and `cargo test --test e2e` separately. Look at the
   `finished in` times and compare them.
3. Make the bug of Section 6 (`<=`), run `cargo test --no-fail-fast`, find the three failures, then revert it.
4. Run the quality gate: `cargo fmt --check && cargo clippy -- -D warnings && cargo test`.
5. Commit the work (one small Git step):

```bash
git add code/p09-pyramid
git commit -m "Add p09-pyramid: bill splitter with unit, integration and E2E tests"
```

## Exercises

### Exercise 1 — A new edge case (easy)

Add a unit test `parses_zero_and_small_amounts` which checks that `parse_money("0")` is `Some(0)`, `parse_money("0.05")`
is `Some(5)` and `parse_money("0.")` is `Some(0)`. Run `cargo test --lib`. Does the last one pass? If it
surprises you, decide whether the program is right or the test is.

<details>
<summary>Hint</summary>

Put the test inside `mod tests` in `src/lib.rs`. Work out `"0."` by hand: the text is split at the dot into
`"0"` and `""`. Is an empty fraction part allowed by the code?

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn parses_zero_and_small_amounts() {
        assert_eq!(parse_money("0"), Some(0));
        assert_eq!(parse_money("0.05"), Some(5));
        assert_eq!(parse_money("0."), Some(0));
    }
```

All three pass. The text `"0."` splits into `"0"` and `""`; the empty fraction has length 0, so the code treats it
as "no cents". This is a design decision: some programs would reject `"0."`. If you decide it is wrong, the test is
your specification: change the test to `None`, watch it fail (red), and then change `parse_money`
(green). Writing the test made the hidden decision visible, which is one of the best things tests do.

</details>

### Exercise 2 — Sort the tests into levels (medium)

For each check, say whether it belongs at the unit, integration or E2E level, and why:

1. `format_money(0)` must be `"0.00"`.
2. The program prints the usage message when run without arguments.
3. `run(["5", "2"])` gives `person 1: 2.50` and `person 2: 2.50`.
4. The program exits with code 2 when the amount is bad.
5. For `split_bill(1, 5)` the shares are `[1, 0, 0, 0, 0]`.

<details>
<summary>Hint</summary>

Ask what the cheapest test is that could still catch the problem. Exit codes and stderr exist only in a real process.

</details>

<details>
<summary>Answer</summary>

1. Unit: one function, a tiny edge case.
2. Integration: `run(&[])` gives `Err(CliError::Usage)`. The E2E test for a bad amount already proves that stderr is wired.
3. Integration: parsing, splitting and formatting work together through `run`.
4. E2E: the exit code belongs to the process.
5. Unit: one function, an edge case with more people than cents.

The rule: choose the lowest level that can still fail for the problem you are worried about.

</details>

### Exercise 3 — Write a test first (hard)

New requirement: the program must reject more than 1,000 people, with a new error `CliError::TooManyPeople(String)`
and the message `'<text>' is too many people (the maximum is 1000)`. Work test-first. (a) Write an integration test
in `tests/api.rs` for `["10.00", "1001"]` and see it fail (it does not compile at first: that is a red state
too). (b) Make it compile and pass. (c) Does an E2E test for this feature make sense?

<details>
<summary>Hint</summary>

Add the enum variant and the `Display` arm, then check the number in `run` after parsing it. Remember that `1000`
itself must still be allowed: write that as a second test.

</details>

<details>
<summary>Answer</summary>

Test (in `tests/api.rs`):

```rust
#[test]
fn more_than_a_thousand_people_is_an_error() {
    assert_eq!(
        run(&args(&["10.00", "1001"])),
        Err(CliError::TooManyPeople("1001".to_string()))
    );
}

#[test]
fn exactly_a_thousand_people_is_allowed() {
    assert!(run(&args(&["10.00", "1000"])).is_ok());
}
```

Code in `src/lib.rs`: add `TooManyPeople(String)` to the enum, the arm
`CliError::TooManyPeople(text) => write!(f, "'{text}' is too many people (the maximum is 1000)")`, and in
`run`, after the `match`, `if people > 1000 { return Err(CliError::TooManyPeople(people_text)); }` (keep the original
text in a variable before you shadow `people`).

(c) No. The E2E test for `ten 3` already proves that errors reach stderr with exit code 2. The new rule is logic, so
the integration test is enough.

</details>

## Quiz

1. Name the three test levels and say which is the fastest.

<details>
<summary>Answer</summary>

Unit, integration, end-to-end (E2E). Unit tests are the fastest, because they call one function in the same
process.

</details>

2. Why does the pyramid have *few* E2E tests?

<details>
<summary>Answer</summary>

They are slow (they start a process), they break for many reasons, and their failures do not point at the broken
line. A few are enough to prove that the real program is wired together; the many combinations of inputs are
cheaper to check below.

</details>

3. What can an E2E test check that an integration test of `run` cannot?

<details>
<summary>Answer</summary>

Things that belong to the real process and to `main`: the text really reaches stdout or stderr, and the exit code is right.

</details>

4. What is the "ice-cream cone", and why is it bad?

<details>
<summary>Answer</summary>

The upside-down pyramid: many E2E tests and few unit tests. The suite is slow, flaky and hard to debug, so people run
it less and trust it less.

</details>

5. A user reports a bug. What do you do first?

<details>
<summary>Answer</summary>

Write a test that reproduces the bug, at the lowest level where it can be reproduced, and see it fail. Then fix the
code and see it pass. The test stays as a regression test.

</details>

## Summary

- Every test trades speed, precision and confidence. Unit, integration and E2E tests make different trades.
- Unit tests: one piece, microseconds, precise. Integration tests: the public API, pieces together. E2E tests: the
  real program in its own process, the only way to check stdout, stderr and exit codes.
- A healthy suite is a pyramid: many unit tests, some integration tests, few E2E tests. The upside-down
  "ice-cream cone" is slow and fragile.
- Run each check at the lowest level that can still catch the problem.
- Keep `main` thin and the logic in a testable library function.
- Test behaviour not implementation; keep tests deterministic; write a test first for every bug.
- `cargo test --lib`, `--test <file>`, `--doc` and `--no-fail-fast` let you run one level or all of them.

## Sources and further reading

- [The Rust Programming Language — 11.3 Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — the official explanation of unit tests, integration tests and the `tests/` folder.
- [The Cargo Book — cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html) — every option of `cargo test`, including `--lib`, `--test` and `--no-fail-fast`.
- [The Cargo Book — Environment variables Cargo sets for crates](https://doc.rust-lang.org/cargo/reference/environment-variables.html) — where `CARGO_BIN_EXE_<name>` is described.
- [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html) — the standard-library type used to start a program in the E2E tests.
- [Martin Fowler — TestPyramid](https://martinfowler.com/bliki/TestPyramid.html) — a short article on the origin and meaning of the test pyramid.
- [Martin Fowler — The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) — a longer guide to choosing the level of each test.

---

[🏠 Start](../START-HERE.md) · [← Phase 09 — Testing, debugging and benchmarking](README.md) · [📋 Phase 09](README.md) · [09.2 End-to-end tests for command-line programs →](02-testing-command-line-programs.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
