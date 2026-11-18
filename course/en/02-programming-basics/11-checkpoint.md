<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.10 Project: the guessing game](10-project-guessing-game.md) · [📋 Phase 02](README.md) · [Phase 03 — Ownership and borrowing →](../03-ownership-and-borrowing/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/11-checkpoint.md) <!-- nav -->

# Lesson 02.11 — Checkpoint

> **Phase 02 — Programming basics in Rust** · Lesson 11 of 11 · ⏱ about 120 minutes

## What you will learn

- Which parts of Phase 02 you already know well, and which need another look.
- How to combine variables, types, functions, loops, formatting, input, tests and Git in small programs.
- How to prepare for Phase 03, where Rust's main idea — ownership — begins.

## Before you start

- You finished [Lesson 02.10 — Project: the guessing game](10-project-guessing-game.md). The game is merged and tagged.
- Do the checkpoint **without** looking at the lessons first. Use them only after you have tried to answer. Write your answers on paper or in a file, then compare.
- Work in the practice project `~/projects/rust/code/p02-basics`; put each task in its own file in `src/bin/`.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A stop on the road where you check that you really know the material before you go on. |
| **self-test** | A test that you give to yourself, without a teacher. |
| **active recall** | Remembering an answer from your head instead of rereading it. It is one of the strongest ways to learn. |
| **spaced repetition** | Reviewing material again after growing gaps of time (a day, a week, a month) so that it stays in long-term memory. |

## Self-test quiz

Answer each question first, then open the answer. A question you cannot answer is not a failure: it tells you what to review (the table at the end links every topic to its lesson).

1. What is the difference between `let x = 5;` and `let mut x = 5;`? What error do you get if you change the first `x` later?

<details>
<summary>Answer</summary>

`let x` is immutable: the value cannot be changed after it is set. `let mut x` allows changing it. Changing an immutable variable gives `error[E0384]: cannot assign twice to immutable variable`. (Lesson 02.1)

</details>

2. What is the difference between shadowing (`let x = x + 1;`) and mutation (`x += 1;`)? Which one lets you change the type?

<details>
<summary>Answer</summary>

Shadowing creates a **new** variable with the same name; it can have a different type (`let spaces = spaces.len();`). Mutation changes the value of the **same** variable, which must be `mut`, and its type cannot change. (Lesson 02.1)

</details>

3. How is a `const` different from a `let`? Write a constant for the number of seconds in a day.

<details>
<summary>Answer</summary>

A `const` must have a written type, is computed at compile time, can be declared outside functions, can never be `mut`, and is named in `SCREAMING_SNAKE_CASE`. For example `const SECONDS_PER_DAY: u32 = 24 * 60 * 60;`. (Lesson 02.1)

</details>

4. What does this program print?

```rust
fn main() {
    let x = 5;
    let x = x * 2;
    {
        let x = x + 1;
        println!("{x}");
    }
    println!("{x}");
}
```

<details>
<summary>Answer</summary>

```text
11
10
```

The inner `x` (11) exists only inside the braces. After the block, `x` is the second variable (10). (Lesson 02.1)

</details>

5. What do `7 / 2 * 2`, `7 % 4` and `-7 / 2` print?

<details>
<summary>Answer</summary>

`6`, `3` and `-3`. Integer division cuts off the fraction: `7 / 2 = 3`, then `3 * 2 = 6`. `7 % 4` is the remainder, `3`. `-7 / 2` is truncated towards zero, `-3`. (Lesson 02.3)

</details>

6. What are the ranges of `u8` and `i8`? What does `250_u8.wrapping_add(10)` give, and what does `250_u8.checked_add(10)` give?

<details>
<summary>Answer</summary>

`u8`: 0 to 255. `i8`: −128 to 127. `wrapping_add` gives `4` (it wraps around). `checked_add` gives `None` (an overflow is reported instead of a result). In a debug build, the plain `250_u8 + 10` at run time panics with `attempt to add with overflow`; in a release build it wraps to `4`. (Lesson 02.2)

</details>

7. What is the type of `let t = (1, "a", 2.5);`, and how do you read the `2.5`? How do you get the first element of an array `arr`?

<details>
<summary>Answer</summary>

It is a tuple `(i32, &str, f64)`. `t.2` reads `2.5`. In an array, `arr[0]` is the first element. Tuples and arrays both count from 0. (Lesson 02.2)

</details>

8. What does `for i in (1..=10).step_by(3) { print!("{i} "); }` print? What changes if you write `1..10`?

<details>
<summary>Answer</summary>

`1 4 7 10 `. With `1..10` the end is excluded, and the loop would print `1 4 7 ` (10 is not reached; the last value below 10 in the sequence 1, 4, 7, 10 is 7). (Lesson 02.5)

</details>

9. What is the value of `let r = { let a = 3; a * 2; };`? What would change without the last semicolon?

<details>
<summary>Answer</summary>

`r` is `()` (the unit value), because the semicolon turns the last expression into a statement. Without the semicolon, `r` would be `6`. (Lesson 02.3)

</details>

10. What do `[{:>5}] [{:<5}] [{:^5}] [{:05}]` print for the value `42`? And `{:.2} {:+} {:#x} {:08b}` for `3.14159`, `7`, `255`, `5`?

<details>
<summary>Answer</summary>

```text
[   42] [42   ] [ 42  ] [00042]
3.14 +7 0xff 00000101
```

(Lesson 02.6)

</details>

11. What do `2 + 3 * 4 - 1` and `10 > 5 && !(3 == 4)` give? Why is `-2_i32.pow(2)` equal to `-4`?

<details>
<summary>Answer</summary>

`13` and `true`. The method call `pow` binds tighter than the unary minus, so `2.pow(2) = 4` is computed first, and then negated. Write `(-2_i32).pow(2)` to get `4`. (Lesson 02.3)

</details>

12. A function must return an `i32`, but it has a semicolon after its last expression. What is the error, and what is its fix?

<details>
<summary>Answer</summary>

`error[E0308]: mismatched types` — `expected i32, found ()`. The compiler suggests the fix: remove the semicolon so that the last expression becomes the return value. (Lessons 02.3, 02.4)

</details>

13. What do `255u8 as i8`, `3.99_f64 as u8` and `'a' as u32` give? Why can `as` be dangerous?

<details>
<summary>Answer</summary>

`-1`, `3` and `97`. `as` silently changes or loses information (wrapping, cutting off the fraction), and does not tell you. (Lesson 02.2)

</details>

14. You read a line with `read_line` and want a number. Name the three steps and the type of the value that `parse` returns. What does `read_line` return at the end of the input?

<details>
<summary>Answer</summary>

(1) Read the line into a `mut String`; (2) `trim()` the new line and spaces; (3) `parse::<T>()`, which returns a `Result<T, _>`: `Ok(value)` or `Err(error)`, handled with `match` or `expect`. At the end of the input, `read_line` returns `Ok(0)`. (Lesson 02.7)

</details>

15. In a Rust test, when does the test pass? Name two ways to check a value, and how to check that code panics.

<details>
<summary>Answer</summary>

A test passes when the function ends without a panic. Use `assert!(condition)` and `assert_eq!(left, right)` (or `assert_ne!`). For panics use `#[should_panic(expected = "text")]`. Tests live in `#[cfg(test)] mod tests` with `use super::*;`, and run with `cargo test`. (Lesson 02.8)

</details>

16. Name four tools or techniques for debugging, from the simplest to the most powerful.

<details>
<summary>Answer</summary>

`println!`/`eprintln!` with `{:?}`; `dbg!`; reading the panic message and the backtrace (`RUST_BACKTRACE=1`); a debugger (RustRover or `lldb`) with breakpoints and stepping. Also `cargo clippy`, tests, `git diff` and `git bisect`. (Lesson 02.9)

</details>

17. In Git, what is a feature branch, and why do you make one commit per milestone? What does `git merge --no-ff` do?

<details>
<summary>Answer</summary>

A feature branch is a branch for one piece of work, merged into `main` when finished. A commit per milestone gives a clear, small history that is easy to review, to `bisect` and to revert. `--no-ff` always creates a merge commit, so the history shows the feature as a unit. (Lessons 02.10 and 01.6)

</details>

18. What is the difference between `Cargo.toml` and `Cargo.lock`, and why should `target/` be in `.gitignore`?

<details>
<summary>Answer</summary>

`Cargo.toml` is your list of dependencies and version ranges; `Cargo.lock` is the exact versions that Cargo chose, for repeatable builds. `target/` holds build output, which is big, can be rebuilt, and has no place in the history. (Lessons 02.1 and 02.10)

</details>

## Practical tasks

Do the four tasks on a branch `feature/p02-checkpoint`, one file each in `src/bin/` of `p02-basics` (or a new project, if you prefer). Each task must pass `cargo fmt`, `cargo clippy` and its own tests, and end with a commit. Try the task before you open the hint.

### Task 1 — A temperature table

Write `celsius_to_fahrenheit(c: f64) -> f64` and a program that prints a table for the Celsius values −10, 0, 10, 20, 30 and 40, with both columns right-aligned and one digit after the point. Write three tests: 0 °C is 32 °F, 100 °C is 212 °F, and −40 is the same in both scales.

Expected output:

```text
 Celsius | Fahrenheit
---------+-----------
   -10.0 |       14.0
     0.0 |       32.0
    10.0 |       50.0
    20.0 |       68.0
    30.0 |       86.0
    40.0 |      104.0
```

<details>
<summary>Hint</summary>

`for c in (-10..=40).step_by(10)` gives integers; convert each with `as f64`. The headers use `{:>8}` and `{:>10}`; the values use `{:>8.1}` and `{:>10.1}`. The separator can be `{:-<8}-+-{:-<10}` applied to two empty texts.

</details>

<details>
<summary>Answer</summary>

```rust
fn celsius_to_fahrenheit(c: f64) -> f64 {
    c * 9.0 / 5.0 + 32.0
}

fn main() {
    println!("{:>8} | {:>10}", "Celsius", "Fahrenheit");
    println!("{:-<8}-+-{:-<10}", "", "");
    for c in (-10..=40).step_by(10) {
        let c = c as f64;
        println!("{:>8.1} | {:>10.1}", c, celsius_to_fahrenheit(c));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn freezing_point() {
        assert_eq!(celsius_to_fahrenheit(0.0), 32.0);
    }

    #[test]
    fn boiling_point() {
        assert_eq!(celsius_to_fahrenheit(100.0), 212.0);
    }

    #[test]
    fn minus_forty_is_the_same_in_both_scales() {
        assert_eq!(celsius_to_fahrenheit(-40.0), -40.0);
    }
}
```

Why it works: shadowing (`let c = c as f64;`) changes the type of `c` without a new name; the format specifiers make the columns line up; and the three tests use values that floats store exactly, so `assert_eq!` is safe. `cargo test` prints `test result: ok. 3 passed`.

</details>

### Task 2 — FizzBuzz as a tested function

Write `fn fizzbuzz(n: u32) -> String` that returns `"Fizz"` for multiples of 3, `"Buzz"` for multiples of 5, `"FizzBuzz"` for multiples of both and the number as text otherwise. `main` prints the results for 1 to 15, one per line. Write four tests (plain numbers, Fizz, Buzz, FizzBuzz), each with two examples.

<details>
<summary>Hint</summary>

Check the multiples of 15 first. A number becomes text with `n.to_string()` (a method that gives a `String`), or with `format!("{n}")`. Return `String::from("Fizz")` for the fixed words.

</details>

<details>
<summary>Answer</summary>

```rust
fn fizzbuzz(n: u32) -> String {
    if n % 15 == 0 {
        String::from("FizzBuzz")
    } else if n % 3 == 0 {
        String::from("Fizz")
    } else if n % 5 == 0 {
        String::from("Buzz")
    } else {
        n.to_string()
    }
}

fn main() {
    for n in 1..=15 {
        println!("{}", fizzbuzz(n));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn plain_numbers_stay_numbers() {
        assert_eq!(fizzbuzz(1), "1");
        assert_eq!(fizzbuzz(7), "7");
    }

    #[test]
    fn multiples_of_three_are_fizz() {
        assert_eq!(fizzbuzz(3), "Fizz");
        assert_eq!(fizzbuzz(9), "Fizz");
    }

    #[test]
    fn multiples_of_five_are_buzz() {
        assert_eq!(fizzbuzz(5), "Buzz");
        assert_eq!(fizzbuzz(10), "Buzz");
    }

    #[test]
    fn multiples_of_fifteen_are_fizzbuzz() {
        assert_eq!(fizzbuzz(15), "FizzBuzz");
        assert_eq!(fizzbuzz(30), "FizzBuzz");
    }
}
```

The output of `cargo run` for 1 to 15 is `1`, `2`, `Fizz`, `4`, `Buzz`, `Fizz`, `7`, `8`, `Fizz`, `Buzz`, `11`, `Fizz`, `13`, `14`, `FizzBuzz`, one per line, and `cargo test` prints `test result: ok. 4 passed`.

Why it works: because `fizzbuzz` returns the text instead of printing it, the tests can check it. The `if` chain is an expression, so its value is the function's result.

</details>

### Task 3 — A sum calculator for the terminal

Write a program that reads lines from standard input until the line `done` or the end of the input. Each valid whole number (`i64`) is added to the sum. Invalid lines are reported on **stderr** as `skipped "<text>": <reason>` and counted. At the end it prints `accepted N, skipped M, sum = S`. Put the parsing in `fn parse_number(text: &str) -> Result<i64, String>` and write two tests: one for valid inputs (`"42"`, `"  -7\n"`, `"+5"`) and one for invalid inputs (`"abc"`, `""`, `"1.5"`, `"99999999999999999999"`).

Check it with:

```bash
printf '10\nabc\n5\n\n-3\n1.5\ndone\n99\n' | cargo run -q --bin <name>
```

Expected output (the first three lines are on stderr):

```text
skipped "abc": invalid digit found in string
skipped "": cannot parse integer from empty string
skipped "1.5": invalid digit found in string
accepted 3, skipped 3, sum = 12
```

<details>
<summary>Hint</summary>

Use `loop`, a new `String` in every iteration, `bytes == 0` for the end of the input, and `line.trim() == "done"` to stop. `parse_number` can use `match text.trim().parse::<i64>()` and build the `Err` text with `format!("{:?}: {e}", text.trim())`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io;

fn parse_number(text: &str) -> Result<i64, String> {
    match text.trim().parse::<i64>() {
        Ok(n) => Ok(n),
        Err(e) => Err(format!("{:?}: {e}", text.trim())),
    }
}

fn main() {
    let mut sum = 0i64;
    let mut good = 0;
    let mut bad = 0;
    loop {
        let mut line = String::new();
        let bytes = io::stdin().read_line(&mut line).expect("failed to read a line");
        if bytes == 0 || line.trim() == "done" {
            break;
        }
        match parse_number(&line) {
            Ok(n) => {
                sum += n;
                good += 1;
            }
            Err(message) => {
                eprintln!("skipped {message}");
                bad += 1;
            }
        }
    }
    println!("accepted {good}, skipped {bad}, sum = {sum}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_numbers_with_spaces_and_signs() {
        assert_eq!(parse_number("42"), Ok(42));
        assert_eq!(parse_number("  -7\n"), Ok(-7));
        assert_eq!(parse_number("+5"), Ok(5));
    }

    #[test]
    fn rejects_bad_text() {
        assert!(parse_number("abc").is_err());
        assert!(parse_number("").is_err());
        assert!(parse_number("1.5").is_err());
        assert!(parse_number("99999999999999999999").is_err());
    }
}
```

Why it works: `10 + 5 - 3 = 12`; the line `99` after `done` is never read; the three bad lines are reported on stderr and counted; no input can make the program panic. The last test shows a number that is valid text but does not fit in an `i64`.

</details>

### Task 4 — Find the two bugs

This function should compute `n!` (`5! = 120`, `12! = 479 001 600`). Its tests fail. (a) Run the tests and read the failure. (b) Find the first bug and fix it. (c) Add a test `assert_eq!(factorial(13), 6_227_020_800)` and see what the compiler says. (d) Fix the second bug, and add a test for `20! = 2_432_902_008_176_640_000`. (e) Commit with `fix(p02): …`.

```rust
fn factorial(n: u32) -> u32 {
    let mut result = 1;
    for i in 1..n {
        result *= i;
    }
    result
}

fn main() {
    println!("5! = {}", factorial(5));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn small_factorials() {
        assert_eq!(factorial(0), 1);
        assert_eq!(factorial(1), 1);
        assert_eq!(factorial(5), 120);
    }

    #[test]
    fn twelve_factorial() {
        assert_eq!(factorial(12), 479_001_600);
    }
}
```

<details>
<summary>Hint</summary>

The first failure shows `left: 24`, `right: 120`: the last factor is missing, so the range is wrong (Lesson 02.5). After that, look at the message of the compiler for the literal `6_227_020_800` and think about the range of `u32` (Lesson 02.2).

</details>

<details>
<summary>Answer</summary>

(a) The failure:

```text
thread 'tests::small_factorials' panicked at src/main.rs:21:9:
assertion `left == right` failed
  left: 24
 right: 120

thread 'tests::twelve_factorial' panicked at src/main.rs:26:9:
assertion `left == right` failed
  left: 39916800
 right: 479001600
```

(b) Bug 1: `1..n` does not include `n` (an off-by-one error). Use `1..=n`; both tests pass.

(c) With the new test, the compiler says:

```text
error: literal out of range for `u32`
   = note: the literal `6_227_020_800` does not fit into the type `u32` whose range is `0..=4294967295`
   = note: `#[deny(overflowing_literals)]` on by default
```

(d) Bug 2: `u32` is too small for `13!`. (Even without the literal error, the multiplication would panic with `attempt to multiply with overflow` in a debug build.) Use `u64` for the parameter and the result. `20!` is the biggest factorial that fits in a `u64`.

```rust
fn factorial(n: u64) -> u64 {
    let mut result = 1;
    for i in 1..=n {
        result *= i;
    }
    result
}
```

The tests (small, `12!`, `13!` and `20!`) now pass: `test result: ok. 4 passed`.

(e) `git add code/p02-basics && git commit -m "fix(p02): include n in factorial and widen it to u64"`.

Why it works: the first bug was a wrong range, and the second one a type that is too small; the tests and the compiler found both, and the new tests keep them from returning.

</details>

### Check the Git history

When all four tasks are done, merge the branch into `main` and look at the history:

```bash
git switch main
git merge --no-ff feature/p02-checkpoint -m "feat(p02): checkpoint tasks"
git log --oneline --graph -n 10
```

You should see one commit for each task (with a message in the `type(scope): summary` form) and a merge commit on top.

## If you got something wrong

| Topic | Lesson to review |
|---|---|
| `let`, `mut`, `const`, shadowing, scope, naming | [Lesson 02.1 — Variables and mutability](01-variables-and-mutability.md) |
| Integer, float, `bool`, `char` types; overflow; tuples; arrays; `as` | [Lesson 02.2 — Data types](02-data-types.md) |
| Arithmetic, comparison, logical and bitwise operators; precedence; expressions and statements | [Lesson 02.3 — Operators and expressions](03-operators-and-expressions.md) |
| Functions, parameters, return values, recursion | [Lesson 02.4 — Functions](04-functions.md) |
| `if`, `loop`, `while`, `for`, `break`, `continue`, `match` | [Lesson 02.5 — Control flow: conditions and loops](05-control-flow.md) |
| `println!`, `format!`, `{:?}`, width and precision, comments | [Lesson 02.6 — Printing, formatting and comments](06-printing-formatting-and-comments.md) |
| `read_line`, `trim`, `parse`, `Result`, `match` on `Result` | [Lesson 02.7 — Reading input and parsing text](07-reading-input-and-parsing.md) |
| `#[test]`, `assert_eq!`, `should_panic`, TDD, `cargo test` | [Lesson 02.8 — Your first tests](08-your-first-tests.md) |
| `dbg!`, backtraces, the debugger, Clippy, `git bisect` | [Lesson 02.9 — Debugging basics](09-debugging-basics.md) |
| crates, `Cargo.toml`, `Ordering`, environment variables, feature branches | [Lesson 02.10 — Project: the guessing game](10-project-guessing-game.md) |
| `git add`, `git commit`, `.gitignore`, branches, merging | [Lesson 01.3 — Your first repository and commits](../01-git-and-version-control/03-your-first-repository-and-commits.md) and [Lesson 01.6 — Branches and merging](../01-git-and-version-control/06-branches-and-merging.md) |
| Reading compiler errors | [Lesson 00.7 — Reading errors and getting help](../00-getting-started/07-reading-errors-and-getting-help.md) |

**How to review well.** Do not reread the lesson from the top. Find the exact question you missed, open only that section, and then close the lesson and try again to write the code from memory. Repeat the review after a day and after a week (spaced repetition).

**You are ready for Phase 03 when you can:**

- [ ] Write small programs with `let`, `mut`, `const`, functions, loops and conditions without looking at the notes.
- [ ] Explain integer overflow, and choose between `i32`, `u8`, `usize` and `f64`.
- [ ] Read a compiler error, find the error code and the `help` text, and fix the mistake.
- [ ] Read a line from the user, parse it, and survive wrong input.
- [ ] Write a test before the code (TDD) for a small function, and read a failing test.
- [ ] Use `dbg!`, a backtrace and a breakpoint to find a bug.
- [ ] Work in Git with a branch, one commit per step, and a merge.

## What's next

You can now write real small programs in Rust. But there is a question that you have only touched: why did `read_line` need `&mut input`, and why do some functions take `&str` while others take `String`? The next phase answers it. It teaches **ownership and borrowing**, the idea that makes Rust different from other languages: who owns each value, how values move, and how to lend them safely. It is the most important phase of Stage 1, and it is where many learners need a little more time. Take it slowly.

- Next: [Phase 03 — Ownership and borrowing](../03-ownership-and-borrowing/01-memory-stack-and-heap.md), starting with Lesson 03.1, *Memory: the stack and the heap*.
- Optional practice before you go on (free): the exercises for the first topics of [Rustlings](https://github.com/rust-lang/rustlings) (variables, functions, `if`, primitive types), and the beginner exercises of the Rust track on [Exercism](https://exercism.org/tracks/rust). They are small, they give you automatic feedback, and they repeat what you learned in this phase.

## Summary

- Phase 02 covered variables, types, operators, functions, control flow, formatting, input, tests, debugging and a complete project.
- The self-test shows what you remember; the practical tasks show what you can do.
- Review only the topics you missed, from memory first, and repeat after a day and a week.
- Keep the habits: small functions, tests first, `fmt` and `clippy` before every commit, and small commits on a branch.
- Next comes ownership and borrowing in Phase 03.

## Sources and further reading

- [The Rust Programming Language — 3. Common Programming Concepts](https://doc.rust-lang.org/book/ch03-00-common-programming-concepts.html) — the official chapter that covers most of Phase 02.
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — short runnable examples for every topic of the phase.
- [Rustlings](https://github.com/rust-lang/rustlings) — small exercises that you solve in your editor, with automatic checks.
- [Exercism — Rust track](https://exercism.org/tracks/rust) — free exercises with optional mentoring.
- [Pro Git — Git Branching](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell) — the free book chapter on branches, for the review.

---

[🏠 Start](../START-HERE.md) · [← 02.10 Project: the guessing game](10-project-guessing-game.md) · [📋 Phase 02](README.md) · [Phase 03 — Ownership and borrowing →](../03-ownership-and-borrowing/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
