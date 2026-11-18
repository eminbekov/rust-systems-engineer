<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.5 Snapshot testing](05-snapshot-testing.md) · [📋 Phase 09](README.md) · [09.7 Code coverage and mutation testing →](07-code-coverage-and-mutation-testing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/06-tdd-in-practice.md) <!-- nav -->

# Lesson 09.6 — Test-driven development in practice

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 6 of 14 · ⏱ about 135 minutes

## What you will learn

- How a real TDD (test-driven development: write a failing test first, then the code) session flows: a test list, one failing test at a time, baby steps, refactoring on green.
- Three ways to get from red to green: *fake it*, *triangulate* and *obvious implementation*.
- How to write table-driven tests with the `rstest` crate.
- How to fix a bug with TDD, and when TDD is *not* the right tool.

## Before you start

- You finished [Lesson 09.5 — Snapshot testing](05-snapshot-testing.md).
- You know the cycle red, green, refactor from [Lesson 02.8](../02-programming-basics/08-your-first-tests.md) (Section 9) and [Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md) (Section 2): write a failing test, make it pass, clean up. This lesson does not teach the cycle again. It shows how to *work* with it for an hour: what to write first, how big a step to take, and what to do when the cycle surprises you.
- You know `Result`, `enum` errors and `match` ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) and the `Vec` methods `push` and `pop` ([Lesson 04.7](../04-structs-enums-collections-errors/07-vectors.md)).

## New words

| Word | Meaning |
|---|---|
| **test list** | A short list, written before the first test, of the behaviours you want. You pick one item at a time. |
| **baby step** | A very small change: one test, then the least code that makes it pass. |
| **fake it** | Make a test pass in the simplest, even silly, way (for example, return a constant). The next test forces the real code. |
| **triangulation** | Adding a second test with different data, so that "return a constant" no longer works and the general code must appear. |
| **obvious implementation** | When you are sure of the real code, write it at once instead of faking it. |
| **table-driven test** | One test function run with many rows of (input, expected result). |
| **`rstest`** | A crate that turns one test function into many with `#[case]` attributes. |
| **RPN** (Reverse Polish Notation) | A way of writing calculations where the operator comes *after* the numbers: `3 4 +` means 3 + 4. |
| **stack** | A pile where you add and remove only at the top (`push` adds, `pop` removes the last one added). |
| **regression test** | A test written for a bug, so the bug cannot return unnoticed (Lesson 05.6). |
| **spike** | A quick experiment, written to learn how something works, and then thrown away. |

## 1. What we build: an RPN calculator

In RPN you write `3 4 +` instead of `3 + 4`, and `5 1 2 + 4 * + 3 -` instead of `5 + (1 + 2) * 4 - 3`. There are no brackets and no rules about which operator goes first,
because a **stack** does the work: read the tokens (the words) from left to right; a number is pushed on the stack; an operator pops two numbers,
calculates, and pushes the result. At the end the stack must hold exactly one number: the answer.

```text
tokens:   3     4     +
stack:   [3]  [3,4]  [7]      <- "+" popped 4 and 3 and pushed 7
```

It is a good size for practice: small, with clear rules, and with enough errors to think about. Create the package:

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/09-06-tdd
cargo new --lib code/p09-tdd
cd code/p09-tdd
cargo add --dev rstest@0.28
```

## 2. Start with a test list

Before you write any code, write down what the program must do, as short sentences. It costs two minutes and it frees your head, because you do not
have to remember everything. Our list:

```text
- an empty input is an error
- a single number gives that number
- 3 4 +  gives 7
- the other operators: - * /
- longer expressions use the stack: 2 3 4 * +  gives 14
- errors: too few operands, too many operands, a word that is not a number, division by zero
```

You will cross items out and add new ones as you learn. The rule of the session: **take one item, write one failing test for it, and make it pass. Never have
two failing tests.**

## 3. Cycle 1: the first test, and "fake it"

Write the test first. Put in `src/lib.rs` *only* this:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn empty_input_is_an_error() {
        assert_eq!(evaluate(""), Err(RpnError::Empty));
    }
}
```

`cargo test` does not even compile, and that is the first red:

```text
error[E0425]: cannot find function `evaluate` in this scope
 --> src/lib.rs:7:20
  |
7 |         assert_eq!(evaluate(""), Err(RpnError::Empty));
  |                    ^^^^^^^^ not found in this scope
```

(There is a second error about `RpnError`.) A compile error is a legitimate red. It tells you what to create. Write the smallest skeleton, with `todo!()` in the body,
so that the test compiles and fails *for the right reason*:

```rust
#[derive(Debug, PartialEq, Eq)]
pub enum RpnError {
    Empty,
}

pub fn evaluate(_input: &str) -> Result<i64, RpnError> {
    todo!()
}
```

```text
thread 'tests::empty_input_is_an_error' panicked at src/lib.rs:7:5:
not yet implemented
```

Now **fake it**: the simplest code that passes. Replace `todo!()` with `Err(RpnError::Empty)`. The test passes. This looks silly: a calculator that
always says "empty"! But it is a deliberate baby step: the test passes, the structure compiles, and the next tests will force the real
behaviour. Do not write more than the test demands.

> 💡 **Tip:** "Red for the right reason" matters. If a test fails with a typo or a wrong import, it proves nothing about the code. Always read the
> failure message and ask: *is this the failure I expected?*

## 4. Cycle 2: the next test forces the real code

Add the second item of the list:

```rust
    #[test]
    fn a_single_number_is_its_own_result() {
        assert_eq!(evaluate("5"), Ok(5));
    }
```

```text
thread 'tests::a_single_number_is_its_own_result' panicked at src/lib.rs:21:9:
assertion `left == right` failed
  left: Err(Empty)
 right: Ok(5)
```

This is the red we wanted: the faked constant no longer works. The two tests together *triangulate*: "always `Err(Empty)`" cannot satisfy both, so the code must look at
its input. Write the least code:

```rust
pub fn evaluate(input: &str) -> Result<i64, RpnError> {
    if input.trim().is_empty() {
        return Err(RpnError::Empty);
    }
    Ok(input.trim().parse().unwrap())
}
```

Both tests pass. Note the `unwrap()`: if the text is not a number, the program panics. That is wrong for a finished program, but it is
**fine for now**: no test asks for the right behaviour yet. It is on our list ("a word that is not a number"), and when we write that test, the test will force us to
replace the `unwrap`. TDD means writing the code that the tests demand, and no more; a missing test is a missing requirement.

## 5. Cycle 3: addition, and the first real algorithm

```rust
    #[test]
    fn adds_two_numbers() {
        assert_eq!(evaluate("3 4 +"), Ok(7));
    }
```

```text
thread 'tests::adds_two_numbers' panicked at src/lib.rs:10:29:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
```

Red: `"3 4 +"` is not a number. Now we are sure about the algorithm (the stack, from Section 1), so we use the **obvious implementation** and skip faking:

```rust
pub fn evaluate(input: &str) -> Result<i64, RpnError> {
    if input.trim().is_empty() {
        return Err(RpnError::Empty);
    }
    let mut stack: Vec<i64> = Vec::new();
    for token in input.split_whitespace() {
        if token == "+" {
            let b = stack.pop().unwrap();
            let a = stack.pop().unwrap();
            stack.push(a + b);
        } else {
            stack.push(token.parse().unwrap());
        }
    }
    Ok(stack.pop().unwrap())
}
```

**What happens:**

1. `input.split_whitespace()` splits the text at any spaces, tabs or newlines.
2. A `+` pops two numbers from the stack. The second one popped (`b`) is the right operand, the first (`a`) is the left: it matters for `-` and `/`.
3. Any other token is parsed as a number and pushed. At the end the last number on the stack is the answer.

All three tests pass. **Which size of step is right?** If you are sure and the code is short, take a bigger step (the obvious implementation). If the last step surprised
you, take a smaller one (fake it). The size changes during a session; that is normal.

## 6. Cycle 4: a table of cases with `rstest`

Next on the list are `-`, `*` and `/`. Three more nearly identical tests would be a lot of copying. The crate `rstest` lets one test function
run with many rows. Replace `adds_two_numbers` by a table (and add `use rstest::rstest;` in `mod tests`):

```rust
    #[rstest]
    #[case("3 4 +", 7)]
    #[case("10 4 -", 6)]
    #[case("3 4 *", 12)]
    #[case("20 4 /", 5)]
    fn applies_an_operator(#[case] input: &str, #[case] expected: i64) {
        assert_eq!(evaluate(input), Ok(expected));
    }
```

**What happens:** `#[rstest]` replaces `#[test]`. Every `#[case(...)]` line is one row: its values are given to the parameters marked `#[case]`, in order.
`rstest` generates one test per row, named `applies_an_operator::case_1`, `case_2`, and so on, so the output tells you exactly which row failed:

```text
test tests::applies_an_operator::case_1 ... ok
test tests::applies_an_operator::case_3 ... FAILED
test tests::applies_an_operator::case_2 ... FAILED
test tests::applies_an_operator::case_4 ... FAILED
---- tests::applies_an_operator::case_3 stdout ----
thread 'tests::applies_an_operator::case_3' panicked at src/lib.rs:17:38:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
```

(We made three new rows at once. Our rule says one failing test at a time; three rows of the *same* table that fail for the *same* reason count as one step.) The
quickest change that passes is a `match` with one branch per operator:

```rust
        match token {
            "+" => {
                let b = stack.pop().unwrap();
                let a = stack.pop().unwrap();
                stack.push(a + b);
            }
            "-" => {
                let b = stack.pop().unwrap();
                let a = stack.pop().unwrap();
                stack.push(a - b);
            }
            // ...and the same for "*" and "/"
            number => stack.push(number.parse().unwrap()),
        }
```

All six tests are green. Now look at that code: the same four lines are copied four times. **This is the refactor step**, and the tests make it safe. Extract the
repeated part into a function:

```rust
pub fn evaluate(input: &str) -> Result<i64, RpnError> {
    if input.trim().is_empty() {
        return Err(RpnError::Empty);
    }
    let mut stack: Vec<i64> = Vec::new();
    for token in input.split_whitespace() {
        match token {
            "+" | "-" | "*" | "/" => {
                let b = stack.pop().unwrap();
                let a = stack.pop().unwrap();
                stack.push(apply(token, a, b));
            }
            number => stack.push(number.parse().unwrap()),
        }
    }
    Ok(stack.pop().unwrap())
}

fn apply(operator: &str, a: i64, b: i64) -> i64 {
    match operator {
        "+" => a + b,
        "-" => a - b,
        "*" => a * b,
        "/" => a / b,
        _ => unreachable!(),
    }
}
```

Run the tests: still green, so the behaviour did not change. (`unreachable!()` panics if it is ever reached; here the outer `match` guarantees that it is not.)
**Refactor only on green, and run the tests after every small change.** If a test turns red, undo the last change, and you know exactly what broke it.

Now add two rows for longer expressions:

```rust
    #[case("2 3 4 * +", 14)]
    #[case("5 1 2 + 4 * + 3 -", 14)]
```

They pass **at once**. That is allowed, but be careful: a test that has never been red might test nothing. Here it is fine, because these rows check
something real (the stack works for deeper expressions), and the first four rows already proved that the table can fail. When in doubt, break the code on
purpose (for example change `a - b` to `b - a`) and see the test turn red.

## 7. Cycle 5: errors, and what the tests discovered

The last items of the list are the errors. Write all the rows at once, as a second table:

```rust
    #[rstest]
    #[case("+", RpnError::NotEnoughOperands)]
    #[case("1 +", RpnError::NotEnoughOperands)]
    #[case("1 2", RpnError::TooManyOperands)]
    #[case("1 x +", RpnError::BadToken("x".to_string()))]
    #[case("1 0 /", RpnError::DivisionByZero)]
    fn reports_errors(#[case] input: &str, #[case] expected: RpnError) {
        assert_eq!(evaluate(input), Err(expected));
    }
```

First red: a compile error, because the error variants do not exist yet:

```text
error[E0599]: no variant, associated function, or constant named `NotEnoughOperands` found for enum `RpnError` in the current scope
  --> src/lib.rs:61:27
```

Add the variants `NotEnoughOperands`, `TooManyOperands`, `BadToken(String)` and `DivisionByZero` to the enum, and you get the real red:

```text
---- tests::reports_errors::case_3 stdout ----
assertion `left == right` failed
  left: Ok(2)
 right: Err(TooManyOperands)
---- tests::reports_errors::case_2 stdout ----
called `Option::unwrap()` on a `None` value
---- tests::reports_errors::case_5 stdout ----
attempt to divide by zero
---- tests::reports_errors::case_4 stdout ----
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
```

Read these carefully, because they say something important. Case 3 shows that `"1 2"` **returned `Ok(2)`**, a wrong answer without any
warning: our calculator silently threw the `1` away. Without the test we would not have noticed. The other cases are panics from the `unwrap`s that we left on purpose. Now
replace each `unwrap` with a proper error:

```rust
pub fn evaluate(input: &str) -> Result<i64, RpnError> {
    if input.trim().is_empty() {
        return Err(RpnError::Empty);
    }
    let mut stack: Vec<i64> = Vec::new();
    for token in input.split_whitespace() {
        match token {
            "+" | "-" | "*" | "/" => {
                let b = stack.pop().ok_or(RpnError::NotEnoughOperands)?;
                let a = stack.pop().ok_or(RpnError::NotEnoughOperands)?;
                stack.push(apply(token, a, b)?);
            }
            number => {
                let value = number
                    .parse()
                    .map_err(|_| RpnError::BadToken(number.to_string()))?;
                stack.push(value);
            }
        }
    }
    match stack.as_slice() {
        [result] => Ok(*result),
        _ => Err(RpnError::TooManyOperands),
    }
}

fn apply(operator: &str, a: i64, b: i64) -> Result<i64, RpnError> {
    match operator {
        "+" => Ok(a + b),
        "-" => Ok(a - b),
        "*" => Ok(a * b),
        "/" if b == 0 => Err(RpnError::DivisionByZero),
        "/" => Ok(a / b),
        _ => unreachable!(),
    }
}
```

**What happens:** `ok_or(error)` turns an `Option` into a `Result`, so a missing operand becomes `NotEnoughOperands`, and `?` returns early with it.
`map_err` converts the parsing error into our `BadToken` with the offending word. `stack.as_slice()` matched against `[result]` means "exactly one element"; every other
length (including an empty stack) is `TooManyOperands`. (The match arm `"/" if b == 0` has a *guard*: it applies only when the condition is true.) All 13 tests pass.

## 8. Cycle 6: a bug report, with TDD

A user writes: *"`9223372036854775807 1 +` crashes the program."* (9223372036854775807 is the largest `i64`.) The TDD answer to a bug report is always the same: **first a
test that reproduces it.**

```rust
    #[test]
    fn adding_past_the_largest_number_is_an_error() {
        assert_eq!(evaluate("9223372036854775807 1 +"), Err(RpnError::Overflow));
    }
```

Add the variant `Overflow` to the enum (to make it compile) and run:

```text
---- tests::adding_past_the_largest_number_is_an_error stdout ----
thread 'tests::adding_past_the_largest_number_is_an_error' panicked at src/lib.rs:39:19:
attempt to add with overflow
```

The bug is reproduced: in a debug build, `a + b` panics when the result does not fit. Now the fix: the `checked_…` methods return `None` instead of overflowing
(Lesson 05.6):

```rust
fn apply(operator: &str, a: i64, b: i64) -> Result<i64, RpnError> {
    let result = match operator {
        "+" => a.checked_add(b),
        "-" => a.checked_sub(b),
        "*" => a.checked_mul(b),
        "/" if b == 0 => return Err(RpnError::DivisionByZero),
        "/" => a.checked_div(b),
        _ => unreachable!(),
    };
    result.ok_or(RpnError::Overflow)
}
```

All 14 tests pass, and the bug now has a permanent guard.

> 🔒 **Security:** In a *release* build (`cargo build --release`), integer overflow does not panic by default: the number silently wraps around. A calculator
> that answers `-9223372036854775808` for `9223372036854775807 1 +` is wrong without any sign of it. Code that does arithmetic on input from outside should use `checked_…`
> methods, and the test above makes sure it does.

Commit the finished work; it is a good moment, because every test is green:

```bash
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
git add code/p09-tdd
git commit -m "Add p09-tdd: an RPN calculator built test-first"
```

## 9. Habits, and when TDD is not the answer

What made the session work:

- **One failing test at a time.** When two are red, you do not know which change fixed which.
- **Read every red.** It must fail for the reason you expect. The red of Cycle 5 even revealed a wrong answer.
- **The smallest code that passes**, then clean up on green. Tests ask for behaviour, refactoring improves structure; do not mix the two in one step.
- **Commit on green.** Every green is a safe point to return to. In a real project you would commit after most cycles.
- **Keep the test list alive.** Cross out, add, reorder. A new idea during a cycle goes on the list, not into the code.
- **Listen to the tests.** A test that is hard to write is a message: the code has too many dependencies or does too much (Lesson 09.3).

TDD is a tool, not a religion. It is a poor fit when:

- **You do not yet know what to build.** Explore with a *spike*: write rough code to learn, then delete it, and write the real thing test-first with what you learned.
- **The output is mostly visual or exploratory:** a layout, a chart. Use manual checks and snapshots (Lesson 09.5) after the fact.
- **The code already exists without tests** (legacy code). First pin down what it does with *characterization tests* (snapshots are perfect), then change it.
- **The cost of a test is bigger than its value:** a one-line wrapper around a library call.

> 📏 **Convention:** Name tests after behaviour (`adding_past_the_largest_number_is_an_error`), and the table-driven test after the group of behaviour
> (`reports_errors`). The report of a failing row `reports_errors::case_4` plus the table in the source tells you which input broke.

## Common mistakes

**1. Writing the code first and the test afterwards.** The test then follows the code instead of the requirement, and you have never seen it fail. Even for a
tiny function, see the red.

**2. Too big steps.** Writing five tests and the whole algorithm, then debugging for an hour. If you are stuck, delete the last change and take a smaller step.

**3. Testing the implementation.** A test that checks the contents of the private `stack` would break when you change the data structure, although the calculator still works.
Test only what `evaluate` returns.

**4. Skipping the refactor step.** Green is not done. The code in Section 6 had four copies of the same lines; without the refactor, each new operator would add another copy.

**5. A table that hides too much.** `rstest` with 40 rows of cryptic numbers is hard to read. Group the rows (one table for results, one for errors) and choose rows that each
teach something.

## Try it yourself

1. Create the package. Follow Sections 3 to 8 in order. After each cycle run `cargo test` and make sure you see the red *before* you write the code.
2. After Cycle 4, break the code on purpose (`a - b` into `b - a`), look at the failure of `applies_an_operator::case_2`, and undo it.
3. Run `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`. Commit as shown in Section 8.

## Exercises

### Exercise 1 — Predict the colour (easy)

For each new test, predict whether it is red or green with the finished calculator, and then run it: (a) `evaluate("-3 4 +")` is `Ok(1)`; (b) `evaluate("  3   4\t+\n")` is `Ok(7)`.

<details>
<summary>Hint</summary>

For (a), is `"-3"` the same token as the operator `"-"`? For (b), what does `split_whitespace` do with several spaces?

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn negative_numbers_are_numbers() {
        assert_eq!(evaluate("-3 4 +"), Ok(1));
    }

    #[test]
    fn extra_whitespace_is_ignored() {
        assert_eq!(evaluate("  3   4\t+\n"), Ok(7));
    }
```

Both are green at once: `"-3"` is a different token from `"-"`, so it goes to the number branch and parses as -3; and `split_whitespace` ignores any amount of whitespace.
Because they were never red, check that the second test *can* fail: replace `split_whitespace()` with `split(" ")` and run it. It fails with `Err(BadToken(""))`, so the
test is real. Then undo the change.

</details>

### Exercise 2 — A new operator, test first (medium)

Add the remainder operator `%` (`7 3 %` is `1`; `7 0 %` is `Err(DivisionByZero)`). Work in the cycle: write the table first, see the red, then write the code.

<details>
<summary>Hint</summary>

Without code, `%` is parsed as a number and you get `BadToken("%")`. The `i64` method `checked_rem` returns `None` on overflow (and when the divisor is 0), so division by zero needs its own branch before it.

</details>

<details>
<summary>Answer</summary>

Test:

```rust
    #[rstest]
    #[case("7 3 %", Ok(1))]
    #[case("7 0 %", Err(RpnError::DivisionByZero))]
    fn remainder(#[case] input: &str, #[case] expected: Result<i64, RpnError>) {
        assert_eq!(evaluate(input), expected);
    }
```

Both cases are red (`BadToken("%")`). Code: add `"%"` to the operator arm in `evaluate` (`"+" | "-" | "*" | "/" | "%" =>`) and change `apply`:

```rust
        "/" | "%" if b == 0 => return Err(RpnError::DivisionByZero),
        "/" => a.checked_div(b),
        "%" => a.checked_rem(b),
```

Why it works: the guard `if b == 0` applies to both alternatives of the pattern `"/" | "%"`, so one line covers both zero divisions. All 18 tests are green.

</details>

### Exercise 3 — A bug from the field (hard)

A user reports: *"`1 2 + 3` returns an error, but I expected a list of two results, `3 3`."* Is this a bug? Decide, and write the test that
documents your decision. Then explain what a test list entry should say when a requirement is unclear.

<details>
<summary>Hint</summary>

Check the requirement in Section 1: at the end the stack must hold exactly one number. The user wants something the calculator was never meant to do.

</details>

<details>
<summary>Answer</summary>

It is not a bug: the specification says that an expression leaves exactly one number, and the calculator reports `TooManyOperands`. The report is a *feature request* (or a
misunderstanding). A test documents the decision:

```rust
    #[test]
    fn several_results_are_not_allowed() {
        assert_eq!(evaluate("1 2 + 3"), Err(RpnError::TooManyOperands));
    }
```

It is green at once, and it protects the rule. When a requirement is unclear, put a *question* on the test list ("should `1 2 3` be a list?"), ask the person who decides, and write a test only
after the answer. A test is an executable decision; do not guess.

</details>

## Quiz

1. What is the first step of a TDD cycle, and what must you check about it?

<details>
<summary>Answer</summary>

Write a failing test (red). Check that it fails for the reason you expect, by reading the message, because a test that fails for the wrong reason proves nothing.

</details>

2. What is "fake it till you make it", and why is it useful?

<details>
<summary>Answer</summary>

Making a test pass with the simplest code, even a constant. It gets you to green quickly, and the next test, with different data (triangulation), forces the real
code.

</details>

3. When do you refactor in TDD?

<details>
<summary>Answer</summary>

Only when all tests are green, in small steps, running the tests after each step. If a test turns red, undo the last change.

</details>

4. What is the first thing you do after you receive a bug report?

<details>
<summary>Answer</summary>

Write a test that reproduces the bug, at the lowest level possible, and see it fail. Then fix the code. The test remains as a regression test.

</details>

5. What does `#[case(...)]` in `rstest` do?

<details>
<summary>Answer</summary>

Each `#[case]` line is one row of a table: `rstest` generates a separate test (named `case_1`, `case_2`, …) that runs the function with the values of that row.

</details>

6. Name two situations where TDD is not the best tool.

<details>
<summary>Answer</summary>

For example: you do not yet know what to build (use a spike); the output is mostly visual; the code already exists without tests (use characterization tests first); the test costs more than it is
worth.

</details>

## Summary

- Start with a **test list**. Take one item, write one failing test, make it pass, clean up. Only one red at a time.
- **Read every red:** it must fail for the right reason. Compile errors are legitimate reds.
- Three ways to green: *fake it* (a constant), *triangulate* (a second test with other data), *obvious implementation* (when you are sure). Take big steps when sure, small steps when surprised.
- **Refactor on green only**, with the tests as the safety net. Extract duplication, then run the tests again.
- `rstest` makes table-driven tests: `#[rstest]` and one `#[case(...)]` per row; each row is its own test.
- A bug report starts with a reproducing test. Use `checked_…` arithmetic for input from outside, because release builds wrap silently.
- TDD is a tool: use a spike when exploring, and characterization tests for legacy code.

## Sources and further reading

- [rstest on docs.rs](https://docs.rs/rstest/0.28.0/rstest/) — `#[case]`, fixtures, `#[values]` and the other features of the crate.
- [The Rust Programming Language — 11.1 How to Write Tests](https://doc.rust-lang.org/book/ch11-01-writing-tests.html) — the official description of the test attributes and macros.
- [Martin Fowler — TestDrivenDevelopment](https://martinfowler.com/bliki/TestDrivenDevelopment.html) — a short description of the red-green-refactor cycle by one of the people who spread it.
- [std::primitive::i64 — checked_add](https://doc.rust-lang.org/std/primitive.i64.html) — the `checked_…` methods used to avoid overflow.

---

[🏠 Start](../START-HERE.md) · [← 09.5 Snapshot testing](05-snapshot-testing.md) · [📋 Phase 09](README.md) · [09.7 Code coverage and mutation testing →](07-code-coverage-and-mutation-testing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
