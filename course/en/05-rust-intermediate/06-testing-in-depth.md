<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.5 Iterators](05-iterators.md) · [📋 Phase 05](README.md) · [05.7 Documentation →](07-documentation.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/06-testing-in-depth.md) <!-- nav -->

# Lesson 05.6 — Testing in depth

> **Phase 05 — Rust intermediate** · Lesson 6 of 14 · ⏱ about 3.5 hours (2 sittings)

## What you will learn

- How to work test-first (TDD: red, green, refactor) and how to read a failing test.
- The test attributes `#[should_panic]`, `#[ignore]` and tests that return `Result`, and the useful `cargo test` options.
- How to test errors, edge cases, overflow and floating-point numbers.
- The difference between unit tests, integration tests (the `tests/` folder) and doc tests, and how to structure a library so it can be tested.
- How to run the "quality gate" `cargo fmt`, `cargo clippy` and `cargo test`, run it automatically before each commit, and find a bug in history with `git bisect`.

## Before you start

- You finished [Lesson 05.5 — Iterators](05-iterators.md).
- You wrote your first tests in [Lesson 02.8 — Your first tests](../02-programming-basics/08-your-first-tests.md): `#[test]`, `assert_eq!`, `cargo test`. Here is a short reminder in Section 1.
- You know modules and crates ([Lesson 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md)) and error handling ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).
- Git: branches and commits ([Lessons 01.3 to 01.6](../01-git-and-version-control/03-your-first-repository-and-commits.md)).

## New words

| Word | Meaning |
|---|---|
| **unit test** | A small test of one function or type, written in the same file as the code, inside `#[cfg(test)] mod tests`. It can see private items. |
| **integration test** | A test in the `tests/` folder that uses your crate from the outside, like another program would. It sees only the public API (Application Programming Interface: the items that your crate offers to other code). |
| **doc test** | An example inside a documentation comment that `cargo test` compiles and runs (Lesson 05.7). |
| **TDD** (Test-Driven Development) | A way of working: write a failing test first, then the code that makes it pass, then clean up. |
| **red / green / refactor** | The three TDD steps: the test fails (red), the test passes (green), you improve the code while tests stay green. |
| **edge case** | An unusual but valid input at the border of what is allowed: empty text, zero, the maximum value. Most bugs live there. |
| **table-driven test** | One test that loops over a list of (input, expected output) cases. |
| **regression** | A bug that comes back, or something that used to work and stopped working. A *regression test* protects against it. |
| **flaky test** | A test that sometimes passes and sometimes fails without any code change. Avoid them. |
| **library crate / binary crate** | A crate built from `src/lib.rs` (code to be used by others) / from `src/main.rs` (a program you run). |
| **Git hook** | A script in `.git/hooks/` that Git runs at a certain moment, for example before a commit. |
| **`git bisect`** | A Git tool that finds the commit that introduced a bug by binary search through history. |
| **CI** (Continuous Integration) | A server that runs your checks automatically on every push. Phase 09 builds CI pipelines; the hook in this lesson is the local version. |

## 1. Reminder, and a new practice package

From Lesson 02.8 you know that a function marked `#[test]` is a test, that it passes when it does not
panic, and that `assert!`, `assert_eq!` and `assert_ne!` panic when the condition is false. All of them
accept an optional message, with the same placeholders as `println!`:

```rust
assert_eq!(parse_duration("45s"), Ok(45), "input was {:?}", "45s");
assert!(width > 2, "width must be greater than 2 but was {width}");
```

The message is printed only when the test fails, and it is the first thing you read then. A good message
says *what was being checked*.

From this lesson on we practise in a **library** package. Create it next to your other code:

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/05-06-testing
cargo new --lib code/p05-textkit
```

`--lib` creates `src/lib.rs` instead of `src/main.rs`. A library cannot be run, only tested and used by
other code, which is exactly what we want. (Edit `Cargo.toml` if you like, but the default name
`p05-textkit` is fine. In Rust code the crate is called `p05_textkit`: dashes become underscores.)

## 2. Test-driven development: red, green, refactor

TDD means writing the test **before** the code. It sounds backwards, but it gives you three things: you
think about the *interface* first, you know exactly when you are done, and every line of code exists
because a test needed it.

We will write `slugify`, a function that turns a title like `"Hello World"` into text safe for a URL
(Uniform Resource Locator, a web address): `"hello-world"`. Replace the content of `src/lib.rs`:

```rust
pub fn slugify(_input: &str) -> String {
    todo!()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn lowercases_and_joins_words_with_dashes() {
        assert_eq!(slugify("Hello World"), "hello-world");
    }
}
```

**What happens:**

1. `todo!()` is a macro that compiles but panics with "not yet implemented" if executed. It is a placeholder
   so that the code compiles and the *test* can fail on its own terms.
2. `_input` begins with an underscore, so the compiler does not warn about the unused parameter.
3. The test calls `slugify` the way a user would and states the expected result.

**Step 1, red.** Run `cargo test` (from inside `code/p05-textkit`). The test must fail:

```text
running 1 test
test tests::lowercases_and_joins_words_with_dashes ... FAILED

failures:

---- tests::lowercases_and_joins_words_with_dashes stdout ----

thread 'tests::lowercases_and_joins_words_with_dashes' panicked at src/lib.rs:2:5:
not yet implemented

failures:
    tests::lowercases_and_joins_words_with_dashes

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
```

Seeing the test fail first is important: it proves the test **can** fail. A test that has never been red may be
checking nothing.

**Step 2, green.** Write the simplest code that makes the test pass, then run again:

```rust
/// Turns a title into a lowercase, dash-separated text that is safe to use in a URL.
pub fn slugify(input: &str) -> String {
    let mut slug = String::new();
    let mut last_was_dash = true; // true at the start: no leading dash
    for ch in input.chars() {
        if ch.is_alphanumeric() {
            for lower in ch.to_lowercase() {
                slug.push(lower);
            }
            last_was_dash = false;
        } else if !last_was_dash {
            slug.push('-');
            last_was_dash = true;
        }
    }
    if slug.ends_with('-') {
        slug.pop();
    }
    slug
}
```

**What happens:**

1. We walk over the characters. A letter or digit (`is_alphanumeric`, which also works for non-English
   letters) is added in lowercase. `to_lowercase()` returns an iterator, because one character can become
   more than one in some alphabets, so we push each result.
2. Any other character (space, punctuation) becomes **one** dash, but only if the last thing we wrote
   was not already a dash. `last_was_dash` starts as `true`, so there is never a dash at the start.
3. At the end we remove a trailing dash.
4. The `///` comment above the function is documentation (Lesson 05.7).

**Step 3, refactor.** With green tests you can improve names, remove duplication or simplify, and re-run
the tests after each change. If they stay green, the behaviour is the same. Then repeat the cycle with the next
requirement: write a new failing test, make it pass.

Add more tests for the behaviour you care about. Each name is a sentence that says what the code does:

```rust
    #[test]
    fn slugify_collapses_punctuation_and_trims_dashes() {
        assert_eq!(slugify("  Rust -- is... fun!  "), "rust-is-fun");
    }

    #[test]
    fn slugify_keeps_digits_and_non_ascii_letters() {
        assert_eq!(slugify("Привет, мир 2026"), "привет-мир-2026");
    }

    #[test]
    fn slugify_of_empty_text_is_empty() {
        assert_eq!(slugify(""), "");
        assert_eq!(slugify("!!!"), "");
    }
```

(In the file, tests go inside `mod tests { … }`; rename the first one to start with `slugify_` as well for consistency.)

> 📏 **Convention:** Name a test after the behaviour (`slugify_of_empty_text_is_empty`), not after the
> function (`test_slugify_1`). When it fails, the name alone tells you what broke. Arrange the body in three parts:
> **arrange** (prepare input), **act** (call the code), **assert** (check the result). Keep it to one behaviour per test.

### Reading a failure

When an assertion fails, Rust prints both values. Suppose someone broke a test by expecting `Ok(54)`:

```text
---- tests::parse_duration_adds_all_parts stdout ----

thread 'tests::parse_duration_adds_all_parts' panicked at src/lib.rs:92:9:
assertion `left == right` failed
  left: Ok(45)
 right: Ok(54)
```

`left` is the first argument of `assert_eq!` (the actual value), `right` is the second (the expected value). The file and line
(`src/lib.rs:92:9`) point at the assertion. In RustRover you can also click the green arrow in the gutter (the margin next to the line numbers)
beside `#[test]` to run a single test, and click the red marker to jump to the failure.

## 3. Testing errors, edge cases and overflow

Good tests are mostly about what happens at the **border**. Let us add a second function that can fail, and
test every way it can fail. `parse_duration` reads text like `"1h30m15s"` and returns the number of seconds:

```rust
#[derive(Debug, PartialEq)]
pub enum ParseError {
    Empty,
    MissingNumber,
    MissingUnit,
    UnknownUnit(char),
    TooLarge,
}

/// Parses a duration such as "1h30m15s" and returns the number of seconds.
pub fn parse_duration(input: &str) -> Result<u64, ParseError> {
    let input = input.trim();
    if input.is_empty() {
        return Err(ParseError::Empty);
    }
    let mut total: u64 = 0;
    let mut digits = String::new();
    for ch in input.chars() {
        if ch.is_ascii_digit() {
            digits.push(ch);
            continue;
        }
        if digits.is_empty() {
            return Err(ParseError::MissingNumber);
        }
        let value: u64 = digits.parse().map_err(|_| ParseError::TooLarge)?;
        let seconds = match ch {
            'h' => value.checked_mul(3600),
            'm' => value.checked_mul(60),
            's' => Some(value),
            other => return Err(ParseError::UnknownUnit(other)),
        };
        let seconds = seconds.ok_or(ParseError::TooLarge)?;
        total = total.checked_add(seconds).ok_or(ParseError::TooLarge)?;
        digits.clear();
    }
    if !digits.is_empty() {
        return Err(ParseError::MissingUnit);
    }
    Ok(total)
}
```

**What happens:**

1. The error type is an enum with one variant per way to fail (Lesson 04.10). It derives `Debug` and
   `PartialEq` so that tests can compare it with `assert_eq!`.
2. The loop collects digits in a `String`. When it meets a non-digit it treats it as a unit (`h`, `m`, `s`),
   converts the collected digits to a number, multiplies, and adds to `total`.
3. `checked_mul` and `checked_add` return `None` instead of wrapping around when the result does not fit
   into a `u64`; we turn that into `ParseError::TooLarge`.

> 🔒 **Security:** Code that parses input from users or from the network must never trust the numbers. Without
> the `checked_` methods, `"9999999999999999999h"` would overflow: panic in a debug build, silently wrong answer in a release
> build. A test that feeds the **largest and the worst** inputs is how you find such bugs. Write it.

Now the tests. Each group covers one kind of case:

```rust
    #[test]
    fn parse_duration_adds_all_parts() {
        assert_eq!(parse_duration("1h30m15s"), Ok(5415));
        assert_eq!(parse_duration("45s"), Ok(45));
    }

    #[test]
    fn parse_duration_rejects_bad_input() {
        assert_eq!(parse_duration(""), Err(ParseError::Empty));
        assert_eq!(parse_duration("h"), Err(ParseError::MissingNumber));
        assert_eq!(parse_duration("10"), Err(ParseError::MissingUnit));
        assert_eq!(parse_duration("5x"), Err(ParseError::UnknownUnit('x')));
    }

    #[test]
    fn parse_duration_detects_overflow() {
        assert_eq!(
            parse_duration("99999999999999999999s"),
            Err(ParseError::TooLarge)
        );
        assert_eq!(
            parse_duration("9999999999999999999h"),
            Err(ParseError::TooLarge)
        );
    }
```

**What happens:** the first test checks the "happy path". The second has one assertion for **each** error variant.
The third feeds numbers too large for a `u64`: the first one fails in `parse()`, the second in `checked_mul`.
Comparing whole `Result` values with `assert_eq!` is the clearest way to test an error. When you only care *that* it is an
error, use `assert!(result.is_err())`, or `assert!(matches!(result, Err(ParseError::UnknownUnit(_))))`: the macro
`matches!` returns `true` when a value fits a pattern (patterns, Lesson 04.5).

### Table-driven tests

When you have many inputs for the same check, put them in a list and loop:

```rust
    #[test]
    fn table_of_valid_durations() {
        let cases = [("1m", 60), ("2h", 7200), ("1h1m1s", 3661), ("0s", 0)];
        for (text, expected) in cases {
            assert_eq!(parse_duration(text), Ok(expected), "input was {text:?}");
        }
    }
```

**What happens:** adding a case is one more tuple. The message `input was {text:?}` matters here: the loop runs inside
one test, so without it you would not know **which** case failed. (`{:?}` prints the text with quotes, which makes
spaces and empty strings visible.)

### Floating-point numbers

Floating-point numbers are rarely exact (Lesson 02.2): `0.1 + 0.2` is not `0.3`. Never compare them with
`assert_eq!`. Compare the difference with a small tolerance:

```rust
    #[test]
    fn floats_are_compared_with_a_tolerance() {
        let sum: f64 = 0.1 + 0.2;
        assert!(sum != 0.3, "0.1 + 0.2 is not exactly 0.3");
        assert!((sum - 0.3).abs() < 1e-9);
    }
```

`1e-9` means 0.000000001. `.abs()` makes the difference positive, so the check works in both directions.

## 4. More test attributes and `cargo test` options

### `#[should_panic]`

Some code is *supposed* to panic (Lesson 04.10: for bugs, not for expected errors). To test it, add `#[should_panic]`:

```rust
    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn indexing_past_the_end_panics() {
        let v: Vec<i32> = Vec::new();
        let _ = v[0];
    }
```

**What happens:** the test passes **only if** the code panics, and `expected = "…"` also requires that the panic message
**contains** that text. Always write `expected`, otherwise a panic for a completely different reason makes the test pass.
(`let _ = …` tells the compiler we deliberately ignore the value.)

### Tests that return `Result`

A test may return `Result<(), E>`. Then you can use `?` inside it instead of `unwrap`, and an `Err` fails the test:

```rust
    #[test]
    fn test_can_return_a_result() -> Result<(), ParseError> {
        let seconds = parse_duration("2m")?;
        assert_eq!(seconds, 120);
        Ok(())
    }
```

### `#[ignore]`

Slow tests (a network call, a large file) can be skipped by default:

```rust
    #[test]
    #[ignore = "slow: pretend this calls the network"]
    fn slow_test() {
        std::thread::sleep(std::time::Duration::from_secs(2));
    }
```

### Running tests: the options you need

`cargo test [FILTER] [-- FLAGS]`. Everything **before** `--` is for Cargo, everything **after** goes to the test program.

| Command | What it does |
|---|---|
| `cargo test` | run all tests (unit, integration and doc tests) |
| `cargo test slugify` | run only tests whose name **contains** `slugify` |
| `cargo test -- --ignored` | run **only** the `#[ignore]` tests |
| `cargo test -- --include-ignored` | run everything, including ignored tests |
| `cargo test -- --list` | list the tests without running them |
| `cargo test -- --nocapture` | show `println!` output of passing tests (by default it is hidden) |
| `cargo test -- --test-threads=1` | run tests one at a time, in order |
| `cargo test --lib` / `--test api` / `--doc` | run only unit tests / one integration file / doc tests |
| `cargo test --release` | build tests with optimisations (use for slow, heavy tests) |

Example of a filtered run:

```text
$ cargo test slugify
running 4 tests
test tests::slugify_collapses_punctuation_and_trims_dashes ... ok
test tests::slugify_keeps_digits_and_non_ascii_letters ... ok
test tests::slugify_of_empty_text_is_empty ... ok
test tests::slugify_lowercases_and_joins_words_with_dashes ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 7 filtered out; finished in 0.00s
```

And of the full run, where the ignored test is shown:

```text
running 11 tests
test tests::slow_test ... ignored, slow: pretend this calls the network
test tests::parse_duration_adds_all_parts ... ok
...
test result: ok. 10 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The order of lines can differ between runs, because Rust runs tests **in parallel** on several threads.)

### Tests must be independent

Because tests run in parallel and in no promised order, **a test must never depend on another test**, or on shared
state such as a global variable or the same temporary file. A test that works alone but fails together with others, or
fails once in a while, is a **flaky test**. The cure is to give each test its own data (its own temporary file name, its own
struct) and not to rely on the clock, random numbers or the network. `--test-threads=1` helps you find out whether
parallelism is the cause.

## 5. Unit tests, integration tests and doc tests

| Kind | Where | Sees | Typical use |
|---|---|---|---|
| unit test | `#[cfg(test)] mod tests` in the same file | **everything**, including private functions | one function or type |
| integration test | a file in `tests/` | **only the public API** | does the crate work as a user would use it? |
| doc test | example in a `///` comment | only the public API | examples stay correct (Lesson 05.7) |

`#[cfg(test)]` means "compile this code only when testing", so test code is not in your final program.

Integration tests live in the `tests/` folder at the package root, next to `src/`. Each file there is compiled as a
**separate crate** that imports yours like any other user:

```text
p05-textkit/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    ├── api.rs
    └── common/
        └── mod.rs      ← helper code shared by the integration tests
```

`tests/api.rs`:

```rust
mod common;

use p05_textkit::{ParseError, parse_duration, slugify};

#[test]
fn slugs_of_sample_titles() {
    let slugs: Vec<String> = common::sample_titles().into_iter().map(slugify).collect();
    assert_eq!(slugs, vec!["hello-world", "rust-friends", "spaced-out"]);
}

#[test]
fn public_api_parses_and_reports_errors() {
    assert_eq!(parse_duration("1m"), Ok(60));
    assert_eq!(parse_duration("1x"), Err(ParseError::UnknownUnit('x')));
}
```

`tests/common/mod.rs`:

```rust
pub fn sample_titles() -> Vec<&'static str> {
    vec!["Hello World", "Rust & Friends", "  spaced  out  "]
}
```

**What happens:**

1. The test file starts with `use p05_textkit::…`: it imports the library by its crate name (the package
   is called `p05-textkit`, and in code the dash becomes an underscore).
2. `mod common;` includes the helper module. The helper lives in `common/mod.rs`, **not** in `tests/common.rs`:
   Cargo treats every file directly in `tests/` as a test crate, and we do not want a test crate with zero tests called
   "common" in the output.
3. The helper has `pub` functions, because the test file is another crate in relation to it.

The output of `cargo test` now has three sections: unit tests, then each integration test file, then doc tests:

```text
     Running unittests src/lib.rs (...)
running 11 tests
...
test result: ok. 10 passed; 0 failed; 1 ignored; ...

     Running tests/api.rs (...)
running 2 tests
test public_api_parses_and_reports_errors ... ok
test slugs_of_sample_titles ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; ...

   Doc-tests p05_textkit
running 0 tests
```

### Integration tests need a library

Integration tests import your code, so the code must be a **library**. A package with only `src/main.rs` (a binary crate)
cannot be imported:

```text
error[E0432]: unresolved import `demo2`
 --> tests/it.rs:1:5
  |
1 | use demo2::double;
  |     ^^^^^ use of unresolved module or unlinked crate `demo2`
```

The professional solution, used by `ripgrep` and most CLI (Command-Line Interface) tools: put the logic in `src/lib.rs`, and keep `src/main.rs` tiny: it only
reads arguments, calls the library and prints. You will do exactly this in the minigrep project (Lesson 05.13).

### Private functions

Integration tests cannot call private functions:

```text
error[E0603]: function `secret` is private
 --> tests/it.rs:4:23
  |
4 |     assert_eq!(demo3::secret(), 42);
  |                       ^^^^^^ private function
  |
note: the function `secret` is defined here
```

Unit tests inside the same module can. Whether to test private functions at all is debated; a good rule: test the **behaviour
visible through the public API** first. Test a private helper directly when it has tricky logic of its own.

## 6. The quality gate: fmt, clippy, test

Three commands, run in this order, make a *quality gate* for any Rust project:

```bash
cargo fmt --check
cargo clippy -- -D warnings
cargo test
```

- `cargo fmt` formats the code in the standard Rust style. With `--check` it only reports differences, and exits with an
  error code if any exist. (Without `--check` it fixes the files.)
- `cargo clippy` is the official linter: it finds code that works but is unidiomatic or suspicious. `-- -D warnings`
  turns every warning into an error.
- `cargo test` runs the tests.

For example this code compiles and its test passes, but the gate rejects it:

```rust
pub fn is_blank(text: &str) -> bool {
    if text.trim().len() == 0 { return true; }
    return false;
}
```

```text
$ cargo fmt --check
Diff in src/lib.rs:1:
 pub fn is_blank(text: &str) -> bool {
-    if text.trim().len() == 0 { return true; }
+    if text.trim().len() == 0 {
+        return true;
+    }
     return false;
 }

$ cargo clippy -- -D warnings
error: unneeded `return` statement
 --> src/lib.rs:3:5
  |
3 |     return false;
  |     ^^^^^^^^^^^^
  |
help: remove `return`

error: length comparison to zero
 --> src/lib.rs:2:8
  |
2 |     if text.trim().len() == 0 { return true; }
  |        ^^^^^^^^^^^^^^^^^^^^^^ help: using `is_empty` is clearer and more explicit: `text.trim().is_empty()`
```

**What it means:** Clippy teaches idiomatic Rust: end a function with an expression, and use `is_empty()`. Treat its
messages as free code review. The fixed function is `pub fn is_blank(text: &str) -> bool { text.trim().is_empty() }`.

> 💡 **Tip:** In RustRover, Clippy can run instead of the plain build check (Settings → Languages & Frameworks → Rust →
> External Linters), so the warnings appear in the editor while you type. The first time, you do not need to set it up:
> the three commands in the terminal are enough.

## 7. Beyond the basics: what comes later

You do not need these yet, but you should know the names, because you will meet them in real projects (Phase 09 teaches
them properly):

- **Property-based testing** (`proptest`, version 1.12): instead of hand-picked examples, you state a property ("for every
  `u64` `n`, `parse_duration(&humanize(n)) == Ok(n)`") and the library tries hundreds of random inputs, then shrinks a failing
  input to the smallest one.
- **Snapshot testing** (`insta`, version 1.49): saves the output of a function in a file and shows a diff when it changes.
- **`cargo nextest`** (version 0.9): a faster test runner with clearer output, installed with `cargo install cargo-nextest`.
- **Coverage** (`cargo-llvm-cov`, version 0.9): shows which lines your tests executed. 100 % coverage does not prove the code is
  correct, but a red line tells you nothing tests it.
- **Mutation testing** (`cargo-mutants`, version 27): changes your code on purpose (turns `+` into `-`) and checks that a test
  fails. A change that nobody notices means a missing test.
- **CI** (Continuous Integration): runs the quality gate on every push, so nobody can merge broken code. Phase 09 builds it
  with GitHub Actions and GitLab CI.

## Common mistakes

**1. A test that cannot fail.** For example `assert!(true)`, or a loop over an empty list. Always watch a new test fail once
(the red step).

**2. `#[should_panic]` without `expected`.** Any panic, even from a typo, makes it pass.

**3. Comparing floats with `assert_eq!`.** Use a tolerance (Section 3).

**4. A loop in a test without a message.** When one case fails you do not know which one. Add `"input was {x:?}"`.

**5. Tests that depend on each other or on shared files.** They fail randomly. Give each test its own data.

**6. Importing a binary crate from `tests/`** (`E0432`, Section 5). Move the logic into `src/lib.rs`.

**7. Only testing the happy path.** Test empty input, one element, the maximum, and every error variant.

**8. Changing a test to make it pass.** If the code is right and the test is wrong, fix the test and understand why. If the
test is right, the code has the bug. Never silence a failing test without understanding it.

## Try it yourself

Put the lesson into practice in `code/p05-textkit`, then use two Git tools that protect the quality of your history.

1. Work test-first on the branch `lesson/05-06-testing` (created in Section 1): write the failing `slugify` test, see it
   red, make it green, and commit after each cycle:

   ```bash
   cd ~/projects/rust
   git add .
   git commit -m "test(p05): failing test for slugify"
   # ... implement ...
   git commit -am "feat(p05): implement slugify"
   ```

   Then add `parse_duration` and all the tests from this lesson, plus `tests/api.rs` and `tests/common/mod.rs`. Run the
   gate: `cargo fmt`, `cargo clippy -- -D warnings`, `cargo test`. Commit as `feat(p05): parse_duration with tests`.

2. **Install a pre-commit hook.** A *Git hook* is a script that Git runs at a certain moment. The `pre-commit` hook runs
   before each commit is created, and if it exits with an error, **the commit is cancelled**. Create the file
   `.git/hooks/pre-commit` (the `.git` folder is hidden: use the terminal):

   ```bash
   cat > .git/hooks/pre-commit <<'EOF'
   #!/bin/sh
   # Runs before every commit. A non-zero exit code cancels the commit.
   cd code/p05-textkit || exit 1
   cargo fmt --check || { echo "pre-commit: run cargo fmt first"; exit 1; }
   cargo clippy --quiet -- -D warnings || { echo "pre-commit: clippy found problems"; exit 1; }
   cargo test --quiet || { echo "pre-commit: tests failed"; exit 1; }
   EOF
   chmod +x .git/hooks/pre-commit
   ```

   `chmod +x` makes the file executable, that is, allowed to run as a program.
   Now try to commit a change that breaks `slugify` (for example, remove the lowercase conversion):

   ```bash
   git add -A
   git commit -m "refactor: simplify slugify"
   ```

   ```text
   assertion `left == right` failed
     left: "Hello-World"
    right: "hello-world"

   failures:
       tests::slugify_collapses_punctuation_and_trims_dashes
       tests::slugify_keeps_digits_and_non_ascii_letters
       tests::slugify_lowercases_and_joins_words_with_dashes

   test result: FAILED. 7 passed; 3 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s

   error: test failed, to rerun pass `--lib`
   pre-commit: tests failed
   ```

   `git log --oneline` shows no new commit: the hook protected your history. Undo the bad change (`git restore code/p05-textkit/src/lib.rs`
   or `git restore --staged` first if it was staged). Two things to know: a hook lives in `.git/` and is **not** shared through the
   repository (CI, Phase 09, is the shared version); and in an emergency `git commit --no-verify` skips it. Use that only on purpose.

3. **Find a bug with `git bisect`.** Imagine that you notice a failing test today, and the last 30 commits are a blur. `git bisect`
   finds the first bad commit by *binary search*: it checks out the commit in the middle of the range, you say "good" or "bad", and it halves the
   range again. With 30 commits you need about five steps instead of thirty. With a test it can be fully automatic:

   ```bash
   git bisect start
   git bisect bad                      # the current commit is broken
   git bisect good <hash-of-an-old-good-commit>
   git bisect run sh -c 'cd code/p05-textkit && cargo test --quiet'
   ```

   `git bisect run` runs your command at each step: exit code 0 means good, non-zero (1 to 127, except 125) means bad. At the end it
   prints the culprit:

   ```text
   Bisecting: 1 revision left to test after this (roughly 1 step)
   [0dde25671b6ff3128c8cea15b4489765f8843614] refactor(textkit): simplify slugify
   ...
   0dde25671b6ff3128c8cea15b4489765f8843614 is the first bad commit
   commit 0dde25671b6ff3128c8cea15b4489765f8843614
   Author: Student <s@example.com>
       refactor(textkit): simplify slugify
    code/p05-textkit/src/lib.rs | 4 +---
    1 file changed, 1 insertion(+), 3 deletions(-)
   bisect found first bad commit
   ```

   When you finish, `git bisect reset` returns you to the branch you were on. Bisect works well only when **commits are small and
   every commit builds**: another reason for the habits from Lessons 05.2 and 05.4. To practise, make five commits in a throw-away
   branch, break a test in the third one, and let `git bisect run` find it.

4. Merge the lesson branch into `main` as before.

## Exercises

### Exercise 1 — Write the missing tests (easy)

The function below works. Write tests for it, in a `#[cfg(test)] mod tests`: at least one normal palindrome, one ignoring case and
punctuation, two non-palindromes, and the edge cases (empty text, one letter, text with no letters).

```rust
pub fn is_palindrome(text: &str) -> bool {
    let cleaned: Vec<char> = text
        .chars()
        .filter(|c| c.is_alphanumeric())
        .flat_map(|c| c.to_lowercase())
        .collect();
    cleaned.iter().eq(cleaned.iter().rev())
}
```

<details>
<summary>Hint</summary>

Think of the border cases first: `""`, `"a"`, `"?!"` (no letters at all: the cleaned text is empty, so the answer is `true`),
`"Aa"` (different case). Use `assert!(is_palindrome(...))` and `assert!(!is_palindrome(...))`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn palindromes_ignore_case_and_punctuation() {
        assert!(is_palindrome("level"));
        assert!(is_palindrome("A man, a plan, a canal: Panama"));
        assert!(is_palindrome("Aa"));
    }

    #[test]
    fn non_palindromes() {
        assert!(!is_palindrome("hello"));
        assert!(!is_palindrome("ab"));
    }

    #[test]
    fn empty_and_single_character_texts_are_palindromes() {
        assert!(is_palindrome(""));
        assert!(is_palindrome("a"));
        assert!(is_palindrome("?!"));
    }
}
```

`cargo test` reports `3 passed`.

Why it works: the tests cover normal inputs and the borders. The decision that `"?!"` is a palindrome (an empty cleaned text) is
now documented by a test, which is useful: if someone changes the rule, the test tells them.

</details>

### Exercise 2 — Test the error paths (medium)

Write tests for this function. Cover a valid port, a port with spaces around it, the border values `0`, `65535`, `70000`, empty text,
non-numeric text and a negative number. For `70000` check that the error message mentions "too large".

```rust
pub fn parse_port(text: &str) -> Result<u16, String> {
    match text.trim().parse::<u16>() {
        Ok(0) => Err(String::from("port 0 is not allowed")),
        Ok(port) => Ok(port),
        Err(e) => Err(format!("invalid port {text:?}: {e}")),
    }
}
```

<details>
<summary>Hint</summary>

For success cases use `assert_eq!(parse_port("8080"), Ok(8080))`. For failures `assert!(parse_port("0").is_err())`.
To look inside the error use `parse_port("70000").unwrap_err()`, which returns the `String`, then `message.contains("too large")`. Add the message as the second
argument of `assert!` so a failure shows it.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_ports() {
        assert_eq!(parse_port("8080"), Ok(8080));
        assert_eq!(parse_port(" 443 "), Ok(443));
        assert_eq!(parse_port("65535"), Ok(65535));
    }

    #[test]
    fn invalid_ports_are_errors() {
        assert!(parse_port("0").is_err());
        assert!(parse_port("").is_err());
        assert!(parse_port("abc").is_err());
        assert!(parse_port("-1").is_err());
        let message = parse_port("70000").unwrap_err();
        assert!(message.contains("too large"), "message was: {message}");
    }
}
```

`cargo test` reports `2 passed`.

Why it works: `65535` is the largest valid `u16` (the border), `70000` is just outside, and `0` is the special value the function rejects on purpose.
Border values catch "off by one" mistakes.

</details>

### Exercise 3 — A table-driven test (medium)

Write one test for `slugify` that loops over at least six `(input, expected)` pairs, including `"C++ & Rust"` (→ `"c-rust"`), only punctuation
(`"---"` → `""`), a date `"2026-11-17"` (→ unchanged) and some non-English letters. The failure message must show which input failed.

<details>
<summary>Hint</summary>

`let cases = [("Hello World", "hello-world"), …];` then `for (input, expected) in cases { assert_eq!(slugify(input), expected, "slugify({input:?})"); }`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn slugify_table() {
        let cases = [
            ("Hello World", "hello-world"),
            ("  many   spaces ", "many-spaces"),
            ("C++ & Rust", "c-rust"),
            ("2026-11-17", "2026-11-17"),
            ("---", ""),
            ("ÅÄÖ", "åäö"),
        ];
        for (input, expected) in cases {
            assert_eq!(slugify(input), expected, "slugify({input:?})");
        }
    }
```

Why it works: the dash in `"2026-11-17"` is punctuation, so it is replaced by one dash, which gives the same text back. If a case fails, the message
`slugify("C++ & Rust")` plus the left/right values pinpoint it. To add a case you add one line.

</details>

### Exercise 4 — TDD: the opposite of `parse_duration` (hard)

Work test-first. Write `pub fn humanize(total_seconds: u64) -> String` that turns `5415` into `"1h30m15s"`, `0` into `"0s"`, `3600` into `"1h"`,
and `61` into `"1m1s"` (zero parts are omitted). **First** write the tests and watch them fail (use `todo!()`), then implement. Add a test that
`parse_duration(&humanize(n)) == Ok(n)` for several values of `n`. Commit after red and after green.

<details>
<summary>Hint</summary>

Hours are `total / 3600`, minutes are `(total % 3600) / 60`, seconds are `total % 60`. Build the result in a `String`: add `"{h}h"` if `h > 0`, `"{m}m"` if `m > 0`, and
`"{s}s"` if `s > 0` **or** the string is still empty (that is how `0` becomes `"0s"`).

</details>

<details>
<summary>Answer</summary>

```rust
/// Turns a number of seconds into text such as "1h30m15s". The opposite of `parse_duration`.
pub fn humanize(total_seconds: u64) -> String {
    let hours = total_seconds / 3600;
    let minutes = (total_seconds % 3600) / 60;
    let seconds = total_seconds % 60;
    let mut out = String::new();
    if hours > 0 {
        out.push_str(&format!("{hours}h"));
    }
    if minutes > 0 {
        out.push_str(&format!("{minutes}m"));
    }
    if seconds > 0 || out.is_empty() {
        out.push_str(&format!("{seconds}s"));
    }
    out
}

// add these two tests to your existing `mod tests`
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn humanize_examples() {
        assert_eq!(humanize(5415), "1h30m15s");
        assert_eq!(humanize(0), "0s");
        assert_eq!(humanize(3600), "1h");
        assert_eq!(humanize(61), "1m1s");
    }

    #[test]
    fn humanize_and_parse_duration_are_inverse() {
        for seconds in [0, 1, 59, 60, 61, 3599, 3600, 86_399, 1_000_000] {
            assert_eq!(parse_duration(&humanize(seconds)), Ok(seconds));
        }
    }
}
```

Running `cargo test` shows all tests green (in the verified project: `12 passed` in the unit tests, with the ignored slow test counted separately).

Why it works: the second test is a tiny version of a *property* ("converting there and back gives the original"), which is the idea behind
property-based testing (Section 7). The `parse_duration` function from the lesson is reused, so this test also protects it. (If you put the tests in your existing `mod tests`, remove the repeated `mod tests` and `use super::*;` lines.)

</details>

## Quiz

1. Why should you watch a new test fail before making it pass?

<details>
<summary>Answer</summary>

It proves the test can fail. A test that has never been red might check nothing (for example, it asserts something that is always true), so a green result would mean nothing.

</details>

2. What does `#[should_panic(expected = "text")]` check, and why add `expected`?

<details>
<summary>Answer</summary>

It passes only if the test panics and the panic message contains "text". Without `expected`, any panic (even from a typo or an unrelated bug) would make the test pass.

</details>

3. Why do tests need to be independent of each other?

<details>
<summary>Answer</summary>

Cargo runs tests in parallel and in no guaranteed order. Tests that share state (a global, a file) can fail randomly (flaky tests) or depend on running order.

</details>

4. What is the difference between a unit test and an integration test in Rust?

<details>
<summary>Answer</summary>

A unit test lives in the same file (`#[cfg(test)] mod tests`) and can see private items. An integration test is a file in `tests/`, compiled as a separate crate, and can use only the public API of your library.

</details>

5. Why can't you write integration tests for a package that only has `src/main.rs`?

<details>
<summary>Answer</summary>

Integration tests import your code by crate name, and a binary crate cannot be imported (`E0432`). Move the logic into `src/lib.rs` and keep `main.rs` as a thin wrapper.

</details>

6. How should you compare two `f64` values in a test, and why?

<details>
<summary>Answer</summary>

Check that the absolute difference is smaller than a small tolerance, such as `(a - b).abs() < 1e-9`. Floating-point arithmetic is usually not exact (`0.1 + 0.2 != 0.3`), so `assert_eq!` would fail on harmless rounding.

</details>

7. What does `git bisect run <command>` do?

<details>
<summary>Answer</summary>

It finds the first bad commit automatically: it repeatedly checks out a commit between a known good one and a known bad one, runs the command, treats exit code 0 as good and non-zero as bad, and halves the range until it identifies the culprit commit.

</details>

## Summary

- Work test-first when you can: red (failing test), green (simplest code), refactor (improve with tests green).
- Test the happy path, every error variant, the borders (empty, zero, maximum) and overflow; compare floats with a tolerance.
- `#[should_panic(expected = …)]`, tests returning `Result`, and `#[ignore]` handle special cases; `cargo test name`, `-- --nocapture`, `-- --list`, `-- --ignored` control runs.
- Unit tests live next to the code and see private items; integration tests live in `tests/`, need a library crate and see only the public API; doc tests come in the next lesson.
- Tests run in parallel, so they must be independent, deterministic and have good failure messages.
- The quality gate is `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`; a `pre-commit` hook can run it before every commit.
- `git bisect run cargo test` finds the commit that broke something, if commits are small and each one builds.

## Sources and further reading

- [The Rust Programming Language — 11. Writing Automated Tests](https://doc.rust-lang.org/book/ch11-00-testing.html) — the official chapter on tests, including `should_panic` and `ignore`.
- [The Rust Programming Language — 11.3 Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — unit versus integration tests, `tests/common/mod.rs`.
- [The Cargo Book — cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html) — every option of `cargo test`.
- [Clippy documentation](https://doc.rust-lang.org/stable/clippy/) — how to use and configure Clippy; the lint list is on its website.
- [std::matches! macro](https://doc.rust-lang.org/std/macro.matches.html) — checking a value against a pattern.
- [Git documentation — git bisect](https://git-scm.com/docs/git-bisect) and [Git hooks](https://git-scm.com/docs/githooks) — binary search for bugs, and the list of hooks.
- [proptest on docs.rs](https://docs.rs/proptest) — property-based testing, for later.

---

[🏠 Start](../START-HERE.md) · [← 05.5 Iterators](05-iterators.md) · [📋 Phase 05](README.md) · [05.7 Documentation →](07-documentation.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
