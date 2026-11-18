<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.7 Reading input and parsing text](07-reading-input-and-parsing.md) · [📋 Phase 02](README.md) · [02.9 Debugging basics →](09-debugging-basics.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/08-your-first-tests.md) <!-- nav -->

# Lesson 02.8 — Your first tests

> **Phase 02 — Programming basics in Rust** · Lesson 8 of 11 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Why we write tests, and what a good test looks like.
- How to write a test with `#[test]` and run it with `cargo test`.
- The assertion macros: `assert!`, `assert_eq!`, `assert_ne!`.
- How to read a failing test, and how to test that a function panics.
- How to run only some tests, and how test-driven development (TDD) works.

## Before you start

- You finished [Lesson 02.7 — Reading input and parsing text](07-reading-input-and-parsing.md).
- You have used `assert_eq!` inside `main` in Lessons 02.1–02.7. In this lesson you move those checks into real tests.
- You know how to make a Git commit ([Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)).

## New words

| Word | Meaning |
|---|---|
| **test** | A small piece of code that runs your code and checks that the result is what you expect. |
| **unit test** | A test of one small part (one function). The tests in this lesson are unit tests. |
| **test runner** | The program that finds and runs all tests and reports the result. `cargo test` starts Rust's runner. |
| **assertion** | A check inside a test: "this must be true". |
| **attribute** | A label in the form `#[name]` that gives the compiler extra information about the next item. |
| **`#[test]`** | The attribute that marks a function as a test. |
| **`#[cfg(test)]`** | An attribute that says "compile the next item only when running tests". `cfg` is short for "configuration". |
| **module** | A named box of code, written `mod name { … }`. Phase 04, Lesson 6 teaches modules. |
| **library crate** | A package of code that other code can use, without a `main` function. Its root file is `src/lib.rs`. |
| **regression** | A bug that comes back after it was fixed, or something that worked before and is now broken. |
| **edge case** | An unusual input at the border of what is allowed: zero, empty text, the biggest value. |
| **TDD** (Test-Driven Development) | A way of working: write a failing test first, then the code that makes it pass, then clean up. |
| **AAA** (Arrange, Act, Assert) | The shape of a good test: prepare the data, run the code, check the result. |
| **CI** (Continuous Integration) | A service that runs your tests automatically on every change. Phase 09 teaches it. |

## 1. Why write tests?

Until now, you checked your programs by running them and looking at the output. That works for a small program, but it has three problems:

1. It is **slow**. You repeat the same steps after every change.
2. It is **forgetful**. After a week, you will not remember all the cases you checked.
3. It does not **protect** you. You change function A, and function B (which you did not look at) breaks silently. This is a **regression**.

A **test** is code that checks code. You write the check once, and the computer repeats it in a fraction of a second, every time, forever. Tests give you:

- **Confidence** to change code: if the tests pass, you probably broke nothing.
- **Documentation**: a test shows how a function is meant to be used.
- **Design feedback**: code that is hard to test is usually badly designed (this is why we separated reading from parsing in Lesson 02.7).

Professional teams run all tests automatically on every change (**CI**, Continuous Integration — Phase 09). Your first step is to learn how a test looks in Rust.

## 2. The test project

Rust has testing built into the language and into Cargo. Create a new **library** project for this lesson:

```bash
cd ~/projects/rust
cargo new code/p02-first-tests --lib
cd code/p02-first-tests
cat src/lib.rs
```

`--lib` makes a **library crate**: a package of functions with no `main`, whose root file is `src/lib.rs`. Cargo has written an example for you:

```rust
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

Run the test with:

```bash
cargo test
```

```text
   Compiling p02-first-tests v0.1.0 (/Users/you/projects/rust/code/p02-first-tests)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.19s
     Running unittests src/lib.rs (target/debug/deps/p02_first_tests-3c41ed7b09edacbb)

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests p02_first_tests

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The hash after `p02_first_tests-` is different on your computer.) How to read it:

1. `Running unittests src/lib.rs` — Cargo built a special **test version** of your code and runs the unit tests found in that file.
2. `running 1 test` and `test tests::it_works ... ok` — one test, named `tests::it_works` (module `tests`, function `it_works`). It passed.
3. `test result: ok. 1 passed; 0 failed; 0 ignored; …` — the summary. If anything fails, the line starts with `FAILED`, and `cargo test` exits with an error (non-zero exit code), which is how CI knows.
4. `Doc-tests` — Cargo also runs the code examples written in `///` documentation comments (Lesson 02.6). We have none, so `running 0 tests`.

## 3. The anatomy of a test

Let us look at the template, line by line:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

1. `mod tests { … }` creates a **module** — a named box — called `tests`. The tests live in a box of their own, next to the code they test, in the same file.
2. `#[cfg(test)]` is an **attribute**. It says "compile this module only for `cargo test`". Your final program does not contain the tests, so they cost nothing at run time.
3. `use super::*;` brings everything from the parent module (`super` = "the box around me") into the test box. Without it, the test cannot see `add`. (Remember: names exist only in their scope, Lesson 02.1.)
4. `#[test]` marks the next function as a test. Only functions with this attribute are run. A test takes no parameters and returns nothing.
5. The body does the three steps of **AAA**: *Arrange* (prepare input), *Act* (call the code), *Assert* (check the result).
6. A test **passes** if the function finishes. It **fails** if the function panics. An assertion that is false panics. That is the whole mechanism.

Forgetting `use super::*;` gives an error that now looks familiar. Here the file has a function `double` (it doubles a number) and a test that calls `double(4)`, but the test module has no `use super::*;`:

```text
error[E0425]: cannot find function `double` in this scope
  --> src/lib.rs:10:20
   |
10 |         assert_eq!(double(4), 8);
   |                    ^^^^^^ not found in this scope
   |
help: consider importing this function
   |
 7 +     use crate::double;
   |
```

## 4. Writing real tests

Replace `src/lib.rs` with these functions and tests:

```rust
use std::num::ParseIntError;

/// Returns true when `n` is even.
pub fn is_even(n: i32) -> bool {
    n % 2 == 0
}

/// Divides `a` by `b`. Panics when `b` is zero.
pub fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("cannot divide by zero");
    }
    a / b
}

/// Converts degrees Celsius to degrees Fahrenheit.
pub fn celsius_to_fahrenheit(c: f64) -> f64 {
    c * 9.0 / 5.0 + 32.0
}

/// Parses a whole number from text, ignoring spaces around it.
pub fn parse_number(text: &str) -> Result<i32, ParseIntError> {
    text.trim().parse()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn even_numbers_are_even() {
        assert!(is_even(4));
        assert!(is_even(0));
        assert!(is_even(-2));
    }

    #[test]
    fn odd_numbers_are_not_even() {
        assert!(!is_even(7));
        assert!(!is_even(-3), "-3 should not be even");
    }

    #[test]
    fn divide_works() {
        assert_eq!(divide(10, 2), 5);
        assert_eq!(divide(7, 2), 3);
        assert_ne!(divide(7, 2), 4);
    }

    #[test]
    #[should_panic(expected = "cannot divide by zero")]
    fn divide_by_zero_panics() {
        divide(1, 0);
    }

    #[test]
    fn boiling_point() {
        assert_eq!(celsius_to_fahrenheit(100.0), 212.0);
    }

    #[test]
    fn body_temperature_with_tolerance() {
        let f = celsius_to_fahrenheit(36.6);
        assert!((f - 97.88).abs() < 1e-9, "got {f}");
    }

    #[test]
    fn parse_number_accepts_spaces() {
        assert_eq!(parse_number("  42\n"), Ok(42));
    }

    #[test]
    fn parse_number_rejects_text() {
        assert!(parse_number("abc").is_err());
        assert!(parse_number("").is_err());
    }

    #[test]
    #[ignore = "slow example, run with --ignored"]
    fn a_slow_test() {
        let mut sum: u64 = 0;
        for i in 0..1_000_000u64 {
            sum += i;
        }
        assert_eq!(sum, 499_999_500_000);
    }
}
```

```text
running 9 tests
test tests::a_slow_test ... ignored, slow example, run with --ignored
test tests::body_temperature_with_tolerance ... ok
test tests::boiling_point ... ok
test tests::parse_number_accepts_spaces ... ok
test tests::even_numbers_are_even ... ok
test tests::divide_works ... ok
test tests::parse_number_rejects_text ... ok
test tests::odd_numbers_are_not_even ... ok
test tests::divide_by_zero_panics - should panic ... ok

test result: ok. 8 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The order of the lines can differ on every run: tests run in parallel, on several threads.)

What happens:

1. `assert!(condition)` passes if the condition is `true`. You can add a message that is shown on failure, with the same format rules as `println!`: `assert!(!is_even(-3), "-3 should not be even")`.
2. `assert_eq!(left, right)` passes if the values are equal; `assert_ne!(left, right)` passes if they are different. On a failure they print both values, which is why `assert_eq!` is better than `assert!(a == b)`.
3. The two values must have a type that can be compared (`PartialEq`) and shown in `{:?}` (`Debug`). All the types in this phase have both. (Phase 05 explains these traits.)
4. `#[should_panic(expected = "…")]` marks a test that **passes only if the code panics**, and (with `expected`) only if the panic message contains this text. This tests the "error" paths.
5. `#[ignore]` skips a test by default (useful for slow tests); you run ignored tests with `cargo test -- --ignored`.
6. The names of the tests are sentences that say **what should be true**: `parse_number_rejects_text`. When a test fails, the name already tells you what broke.
7. Floats are compared with a tolerance (Lesson 02.2). The message `"got {f}"` appears only if the assertion fails.

## 5. Reading a failing test

A test that never fails tells you nothing. Let us break the code on purpose: change `is_even` to `n % 2 == 1`.

```text
running 9 tests
test tests::a_slow_test ... ignored, slow example, run with --ignored
test tests::body_temperature_with_tolerance ... ok
test tests::parse_number_rejects_text ... ok
test tests::parse_number_accepts_spaces ... ok
test tests::divide_works ... ok
test tests::boiling_point ... ok
test tests::odd_numbers_are_not_even ... FAILED
test tests::divide_by_zero_panics - should panic ... ok
test tests::even_numbers_are_even ... FAILED

failures:

---- tests::odd_numbers_are_not_even stdout ----

thread 'tests::odd_numbers_are_not_even' (4113760) panicked at src/lib.rs:39:9:
assertion failed: !is_even(7)

---- tests::even_numbers_are_even stdout ----

thread 'tests::even_numbers_are_even' (4113759) panicked at src/lib.rs:32:9:
assertion failed: is_even(4)


failures:
    tests::even_numbers_are_even
    tests::odd_numbers_are_not_even

test result: FAILED. 6 passed; 2 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
```

How to read it:

1. The list shows `FAILED` for two tests. The others are still `ok`, so the bug is limited to `is_even`.
2. For each failure you see the file, the line and the column (`src/lib.rs:32:9`) and the assertion that was false: `assertion failed: is_even(4)`.
3. The summary says `FAILED. 6 passed; 2 failed`.
4. The last line, `error: test failed`, shows that the command failed (exit code not zero).

An `assert_eq!` failure also prints both sides. To see it, add this small example to `src/lib.rs`: a function `double` with a bug on purpose (it returns `n * 2 + 1`) and a test for it. Section 6 uses the same test.

```rust
/// Doubles a number. (This version has a bug on purpose.)
pub fn double(n: i32) -> i32 {
    n * 2 + 1
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn doubles() {
        println!("testing double(4)");
        assert_eq!(double(4), 8, "double(4) should be 8");
    }
}
```

```text
assertion `left == right` failed: double(4) should be 8
  left: 9
 right: 8
```

`left` is the first value given to `assert_eq!` (what the code returned, `9`), and `right` is the second one (what you expected, `8`). After you have seen the failure, fix `double` to return `n * 2`.

And the two ways in which a `should_panic` test fails. They come from two small tests that are not shown here: `no_panic_happens` expects a panic that never happens, and `wrong_message` expects a panic message that contains `negative`, but the real message is different:

```text
---- tests::no_panic_happens stdout ----
note: test did not panic as expected at src/lib.rs:30:8
---- tests::wrong_message stdout ----

thread 'tests::wrong_message' (4115500) panicked at src/lib.rs:7:9:
cannot divide by zero
note: panic did not contain expected string
      panic message: "cannot divide by zero"
 expected substring: "negative"
```

Fix the code, run `cargo test` again, and watch the red turn green.

## 6. Running some of the tests, and seeing the output

```bash
cargo test divide                 # only tests whose name contains "divide"
cargo test -- --ignored           # only the ignored tests
cargo test -- --list              # list the tests without running them
cargo test doubles -- --nocapture # show println! output of passing tests
```

```text
running 2 tests
test tests::divide_works ... ok
test tests::divide_by_zero_panics - should panic ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 7 filtered out; finished in 0.00s
```

That is the result of `cargo test divide`: the text after `cargo test` is a **filter** on the test names; `7 filtered out` are the tests that did not match (you will see `8` if you added the `doubles` example from section 5).

By default, the runner **hides** what passing tests print with `println!` (to keep the report short) and shows it only for failing tests. The flag `-- --nocapture` (after a double dash that separates Cargo's flags from the runner's flags) shows everything:

```text
testing double(4)
test tests::doubles ... ok
```

Because tests run **in parallel**, they must be **independent**: no test may rely on another test's result or on the order. A test that works only after another one is a bug waiting to happen. (`cargo test -- --test-threads=1` runs one at a time, for debugging.)

## 7. Tests in a program with `main`, and elsewhere

A binary crate (a project with `src/main.rs`, like the guessing game) can have unit tests in the same way:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    println!("2 + 3 = {}", add(2, 3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_works() {
        assert_eq!(add(2, 3), 5);
    }
}
```

```text
     Running unittests src/main.rs (target/debug/deps/bincrate-5894bf05e1c343f8)

running 1 test
test tests::add_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

In your practice project `p02-basics`, where each file in `src/bin/` is a program, you can test one file with `cargo test --bin 04_functions`.

Rust knows three kinds of tests (Phase 05, Lesson 6 covers them in depth):

| Kind | Where | What it tests |
|---|---|---|
| Unit tests | `#[cfg(test)] mod tests` in the same file | one function or module, including private items |
| Integration tests | files in the `tests/` folder | the public parts of your library, as a user sees them |
| Doc tests | code examples in `///` comments | that the documentation is correct |

## 8. What makes a good test

- **One behaviour per test**, with a name that says what should be true: `parse_number_rejects_text`, not `test1`.
- **Arrange, Act, Assert**: prepare, call, check. Keep each test short.
- **Test the edges**: zero, one, the maximum, empty text, a negative number, the biggest value plus one. Most bugs live there. The list for a number function: `0`, `1`, `-1`, `MAX`, `MIN`.
- **Test the failures**: wrong input must produce the error you designed (`is_err()`, `#[should_panic]`).
- **Be deterministic**: a test must give the same result every time. No randomness, no current time, no network (Phase 09 shows how to handle those).
- **Test behaviour, not details**: check *what* the function returns, not *how* it works inside; otherwise every refactoring breaks your tests.
- **Never delete or ignore a failing test just to get a green result.** The test is telling you something.

> 📏 **Convention:** Put unit tests at the bottom of the file in `#[cfg(test)] mod tests`. Name test functions as sentences in `snake_case`.

> 🔒 **Security:** Tests are a security tool. Write a test for every security rule ("an age above 150 is rejected", "an empty name is rejected", "a number that is too large does not overflow"). If someone breaks the rule later, a test fails before the code reaches users.

## 9. Test-driven development (TDD)

In **TDD** you write the test **before** the code. The cycle has three steps:

1. **Red** — write a test for a function that does not work yet. Run it and watch it fail.
2. **Green** — write the simplest code that makes the test pass.
3. **Refactor** — clean up the code, running the tests to be sure that nothing broke.

Let us build `grade(score: u32) -> char`: 90 and up is `'A'`, 80 and up is `'B'`, 70 and up is `'C'`, anything else is `'F'`. First the tests, and a function body that says "not written yet" — `todo!()` is a macro that panics with `not yet implemented`:

```rust
/// Turns a score from 0 to 100 into a letter grade.
pub fn grade(score: u32) -> char {
    todo!()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ninety_and_above_is_a() {
        assert_eq!(grade(100), 'A');
        assert_eq!(grade(90), 'A');
    }

    #[test]
    fn eighty_to_eighty_nine_is_b() {
        assert_eq!(grade(89), 'B');
        assert_eq!(grade(80), 'B');
    }

    #[test]
    fn seventy_to_seventy_nine_is_c() {
        assert_eq!(grade(79), 'C');
        assert_eq!(grade(70), 'C');
    }

    #[test]
    fn below_seventy_is_f() {
        assert_eq!(grade(69), 'F');
        assert_eq!(grade(0), 'F');
    }
}
```

**Red:** `cargo test` shows

```text
running 4 tests
test tests::seventy_to_seventy_nine_is_c ... FAILED
test tests::below_seventy_is_f ... FAILED
test tests::eighty_to_eighty_nine_is_b ... FAILED
test tests::ninety_and_above_is_a ... FAILED
...
thread 'tests::below_seventy_is_f' (4114830) panicked at src/lib.rs:3:5:
not yet implemented
...
test result: FAILED. 0 passed; 4 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The compiler also warns about the unused `score`.) **Green:** write the code:

```rust
pub fn grade(score: u32) -> char {
    if score >= 90 {
        'A'
    } else if score >= 80 {
        'B'
    } else if score >= 70 {
        'C'
    } else {
        'F'
    }
}
```

```text
running 4 tests
test tests::below_seventy_is_f ... ok
test tests::eighty_to_eighty_nine_is_b ... ok
test tests::ninety_and_above_is_a ... ok
test tests::seventy_to_seventy_nine_is_c ... ok
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Refactor:** the code is already short; we leave it. Notice that the tests check the *boundaries* (89 and 90, 79 and 80), where a wrong `>` instead of `>=` would be found immediately.

## Common mistakes

**1. Forgetting `#[test]`.** The function is compiled but never run, and `cargo test` says `running 0 tests`. Always check that the number of tests is what you expect.

**2. Forgetting `use super::*;`.** `E0425 cannot find function`.

**3. Thinking that a test can see only `pub` items.** A unit test in the same file can call private functions too. (Integration tests, in the `tests/` folder, see only `pub` items.)

**4. Tests that depend on each other or on the order.** They run in parallel, in any order.

**5. Comparing floats with `assert_eq!`.** Use a tolerance, unless the values are exactly representable.

**6. A test that cannot fail.** For example `assert!(true)`, or checking a value you just computed with the same code. Always ask: "what bug would this test catch?"

**7. Testing only the happy path.** Add empty, zero, huge and wrong inputs.

> ⚡ **Performance:** Tests are compiled without optimizations (`unoptimized + debuginfo`), so heavy calculations in tests are slow; `cargo test --release` builds with optimizations. Keep unit tests fast: the whole suite should run in seconds, so you run it all the time.

## Try it yourself

1. Create `code/p02-first-tests` as shown in section 2 and run `cargo test`. Read every line of the report.
2. Replace `src/lib.rs` with the code from section 4. Run `cargo test`, then `cargo test divide`, then `cargo test -- --ignored`, then `cargo test -- --list`.
3. Break `is_even` on purpose (`== 1`), read the failure, then fix it.
4. Add a new function `fn is_positive(n: i32) -> bool` with three tests (positive, zero, negative), and check that zero is *not* positive.
5. Go back to your `p02-basics` project: move the `assert_eq!` checks of one function from Lesson 02.4 (for example `is_even`) into a `#[cfg(test)] mod tests` in the same `src/bin/` file, and run `cargo test --bin <name>`.
6. Run `cargo fmt` and `cargo clippy`. Commit:

```bash
cd ~/projects/rust
git add code/p02-first-tests code/p02-basics
git commit -m "test(p02): add first unit tests"
```

The commit type `test` is a standard part of Conventional Commits ([Lesson 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md)).

## Exercises

### Exercise 1 — Tests for leap years (easy)

The function `is_leap_year` is given. **Write the tests** (not the function): 2024 and 2000 are leap years; 2023 and 1900 are not. Use `assert!` and `assert!(!…)`.

```rust
pub fn is_leap_year(year: u32) -> bool {
    (year % 4 == 0 && year % 100 != 0) || year % 400 == 0
}
```

<details>
<summary>Hint</summary>

Create `#[cfg(test)] mod tests { use super::*; … }` and one `#[test] fn` with four assertions, or four small tests with sentence names.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn is_leap_year(year: u32) -> bool {
    (year % 4 == 0 && year % 100 != 0) || year % 400 == 0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn leap_years() {
        assert!(is_leap_year(2024));
        assert!(is_leap_year(2000));
        assert!(!is_leap_year(2023));
        assert!(!is_leap_year(1900));
    }
}
```

Why it works: 1900 and 2000 are the interesting cases of the rule "divisible by 100 is not a leap year, unless divisible by 400". A good test includes them.

</details>

### Exercise 2 — Find the bug with the tests (easy)

This function has a bug, and the tests find it. Run the tests, read the failure, and fix the function (not the tests).

```rust
pub fn fahrenheit_to_celsius(f: f64) -> f64 {
    (f - 32.0) * 9.0 / 5.0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fahrenheit_conversion() {
        assert_eq!(fahrenheit_to_celsius(32.0), 0.0);
        assert_eq!(fahrenheit_to_celsius(212.0), 100.0);
        assert!((fahrenheit_to_celsius(98.6) - 37.0).abs() < 1e-9);
    }
}
```

<details>
<summary>Hint</summary>

The failure shows `left: 324.0`, `right: 100.0`. The factor is upside down: Celsius degrees are smaller than Fahrenheit degrees, so the factor must be less than 1.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn fahrenheit_to_celsius(f: f64) -> f64 {
    (f - 32.0) * 5.0 / 9.0
}
```

The failing output before the fix was:

```text
assertion `left == right` failed
  left: 324.0
 right: 100.0
```

Why it works: `(212 - 32) * 9 / 5 = 324` with the wrong factor; with `5 / 9` it is `180 * 5 / 9 = 100`. The first assertion (`32 → 0`) passed even with the bug, because `0` times any factor is `0` — a reminder to test more than one value.

</details>

### Exercise 3 — TDD: palindromic numbers (medium)

Use TDD to write `pub fn is_palindrome_number(n: u32) -> bool` (a number that reads the same from both sides: `121`, `12321`; but not `10`). First write the tests for `0`, `7`, `121`, `12321` (true) and `10`, `123` (false), and see them fail (use `todo!()` for the body). Then implement it with a `while` loop that reverses the number.

<details>
<summary>Hint</summary>

To reverse: keep `reversed = 0` and `rest = n`. While `rest > 0`: `reversed = reversed * 10 + rest % 10;` and `rest /= 10;`. At the end compare `reversed == n`.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn is_palindrome_number(n: u32) -> bool {
    let mut rest = n;
    let mut reversed = 0;
    while rest > 0 {
        reversed = reversed * 10 + rest % 10;
        rest /= 10;
    }
    reversed == n
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn palindromes() {
        assert!(is_palindrome_number(0));
        assert!(is_palindrome_number(7));
        assert!(is_palindrome_number(121));
        assert!(is_palindrome_number(12321));
        assert!(!is_palindrome_number(10));
        assert!(!is_palindrome_number(123));
    }
}
```

Why it works: for `121` the loop builds `1`, then `12`, then `121`. For `10` it builds `0`, then `1`, and `1 != 10`. The case `0` works because the loop does not run and `reversed == 0 == n`.

One more edge case to think about: for a very large `n` such as `4_000_000_009`, the reversed number `9_000_000_004` does not fit in a `u32`, and the program panics with `attempt to multiply with overflow`. Making `reversed` a `u64` removes that problem: write `let mut reversed: u64 = 0;`, add each digit as `(rest % 10) as u64`, and compare with `n as u64` (`as` converts a number to another type, Lesson 02.2). Edge cases like this are exactly what tests are for.

</details>

### Exercise 4 — Testing a panic (medium)

Write `pub fn factorial(n: u64) -> u64` that multiplies `1 * 2 * … * n`, and **panics with a message containing `too large`** when `n > 20` (because `21!` does not fit in a `u64`). Write tests for `0`, `1`, `5`, `20` (`20! = 2_432_902_008_176_640_000`), and a `#[should_panic(expected = "too large")]` test for `21`.

<details>
<summary>Hint</summary>

Use `panic!("{n}! is too large for u64");` at the start. The loop `for i in 2..=n` multiplies a `result` that starts at `1`; for `n = 0` or `1` the range `2..=n` is empty.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn factorial(n: u64) -> u64 {
    if n > 20 {
        panic!("{n}! is too large for u64");
    }
    let mut result = 1;
    for i in 2..=n {
        result *= i;
    }
    result
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn factorials() {
        assert_eq!(factorial(0), 1);
        assert_eq!(factorial(1), 1);
        assert_eq!(factorial(5), 120);
        assert_eq!(factorial(20), 2_432_902_008_176_640_000);
    }

    #[test]
    #[should_panic(expected = "too large")]
    fn factorial_of_21_panics() {
        factorial(21);
    }
}
```

Why it works: the guard makes the failure explicit and early, with a clear message, instead of an obscure `attempt to multiply with overflow` in the middle of the loop. The `should_panic` test records this behaviour.

</details>

### Exercise 5 — Tests first for a percentage parser (hard)

Specify `pub fn parse_percent(text: &str) -> Result<u8, String>` **by its tests**, then implement it. It accepts a whole number from 0 to 100, with an optional `%` at the end and optional spaces. Tests that must pass: `"50%"` → `Ok(50)`, `"50"` → `Ok(50)`, `" 7 % "` → `Ok(7)`, `"0"` → `Ok(0)`, `"100%"` → `Ok(100)`. Tests for errors: `"101%"`, `"-1"`, `"%"`, `""`, `"abc"`, `"5 0"`.

<details>
<summary>Hint</summary>

Clean the text first: `text.trim().trim_end_matches('%').trim()`. (`trim_end_matches('%')` removes the `%` characters at the end.) Then `parse::<u8>()` and check `n <= 100`. The type `u8` already rejects `-1`.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn parse_percent(text: &str) -> Result<u8, String> {
    let cleaned = text.trim().trim_end_matches('%').trim();
    match cleaned.parse::<u8>() {
        Ok(n) => {
            if n <= 100 {
                Ok(n)
            } else {
                Err(format!("{n} is more than 100"))
            }
        }
        Err(_) => Err(format!("'{}' is not a percentage", text.trim())),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn percent_accepts_valid_text() {
        assert_eq!(parse_percent("50%"), Ok(50));
        assert_eq!(parse_percent("50"), Ok(50));
        assert_eq!(parse_percent(" 7 % "), Ok(7));
        assert_eq!(parse_percent("0"), Ok(0));
        assert_eq!(parse_percent("100%"), Ok(100));
    }

    #[test]
    fn percent_rejects_bad_text() {
        assert!(parse_percent("101%").is_err());
        assert!(parse_percent("-1").is_err());
        assert!(parse_percent("%").is_err());
        assert!(parse_percent("").is_err());
        assert!(parse_percent("abc").is_err());
        assert!(parse_percent("5 0").is_err());
    }
}
```

Why it works: `"%"` becomes the empty text, which fails to parse; `"5 0"` keeps the inner space, so it is not a number. The tests describe all the rules, so you can later change the implementation without fear.

</details>

## Quiz

1. What does `#[test]` do?

<details>
<summary>Answer</summary>

It marks a function as a test. `cargo test` finds and runs only functions with this attribute.

</details>

2. When does a test pass, and when does it fail?

<details>
<summary>Answer</summary>

It passes if the function finishes without panicking. It fails if it panics (for example, because an assertion is false).

</details>

3. What is the difference between `assert!` and `assert_eq!`?

<details>
<summary>Answer</summary>

`assert!(cond)` checks a `bool`. `assert_eq!(a, b)` checks that two values are equal and, on failure, prints both of them, which makes the bug easier to find.

</details>

4. What do `#[cfg(test)]` and `use super::*;` do?

<details>
<summary>Answer</summary>

`#[cfg(test)]` compiles the module only when testing. `use super::*;` imports all the names of the parent module (the code under test) into the test module.

</details>

5. How do you test that a function panics for bad input?

<details>
<summary>Answer</summary>

Add `#[should_panic]` to the test, and better `#[should_panic(expected = "text of the message")]`. The test passes only if the code panics.

</details>

6. How do you run only the tests with `divide` in their name?

<details>
<summary>Answer</summary>

`cargo test divide`.

</details>

7. What are the three steps of TDD?

<details>
<summary>Answer</summary>

Red (write a failing test), green (write the simplest code that passes), refactor (clean up while the tests stay green).

</details>

8. Why must tests not depend on each other?

<details>
<summary>Answer</summary>

The runner runs tests in parallel and in any order, so the result of one test cannot be relied on in another.

</details>

## Summary

- A test is a function marked `#[test]`, kept in `#[cfg(test)] mod tests` with `use super::*;`. Run all tests with `cargo test`.
- A test passes if it does not panic. Use `assert!`, `assert_eq!`, `assert_ne!` (with an optional message), and `#[should_panic(expected = "…")]` for panics.
- Read failures from the top: the test name, the line, then `left` and `right`.
- Filter with `cargo test name`; `-- --nocapture` shows output; `#[ignore]` and `-- --ignored` handle slow tests.
- Good tests: one behaviour each, edge cases, bad inputs, deterministic, independent.
- TDD: red, green, refactor.
- Unit tests live next to the code; integration tests in `tests/`; doc tests in `///` comments (Phase 05).

## Sources and further reading

- [The Rust Programming Language — 11.1 How to Write Tests](https://doc.rust-lang.org/book/ch11-01-writing-tests.html) — the official book chapter: `#[test]`, assertions and `should_panic`.
- [The Rust Programming Language — 11.2 Controlling How Tests Are Run](https://doc.rust-lang.org/book/ch11-02-running-tests.html) — filters, `--nocapture`, `--ignored` and threads.
- [The Rust Programming Language — 11.3 Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — unit tests versus integration tests.
- [Rust by Example — Unit testing](https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html) — short runnable examples.
- [The Cargo Book — cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html) — all options of the command.
- [Standard library — assert_eq!](https://doc.rust-lang.org/std/macro.assert_eq.html) — the documentation of the assertion macro.

---

[🏠 Start](../START-HERE.md) · [← 02.7 Reading input and parsing text](07-reading-input-and-parsing.md) · [📋 Phase 02](README.md) · [02.9 Debugging basics →](09-debugging-basics.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
