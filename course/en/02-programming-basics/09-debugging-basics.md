<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.8 Your first tests](08-your-first-tests.md) · [📋 Phase 02](README.md) · [02.10 Project: the guessing game →](10-project-guessing-game.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/09-debugging-basics.md) <!-- nav -->

# Lesson 02.9 — Debugging basics

> **Phase 02 — Programming basics in Rust** · Lesson 9 of 11 · ⏱ about 3 hours (2 sittings)

## What you will learn

- A calm, step-by-step method to find a bug, instead of guessing.
- How to read a panic message and a backtrace.
- How to look inside a running program with `println!`, `dbg!` and `debug_assert!`.
- How to use a debugger (LLDB, and the debugger in RustRover): breakpoints, stepping and watching variables.
- How Clippy and Git (`git diff`, `git bisect`) help you find bugs.

## Before you start

- You finished [Lesson 02.8 — Your first tests](08-your-first-tests.md). A failing test is the best start of a debugging session.
- You know `git log`, `git diff` and commits from [Lesson 01.4 — Viewing history and changes](../01-git-and-version-control/04-viewing-history-and-changes.md).
- You remember how to read compiler errors from [Lesson 00.7](../00-getting-started/07-reading-errors-and-getting-help.md).

## New words

| Word | Meaning |
|---|---|
| **bug** | A mistake in a program that makes it do something other than what was intended. |
| **debugging** | Finding the cause of a bug and fixing it. |
| **reproduce** | Make the bug happen again, reliably, with the same steps. |
| **backtrace** | A list of the functions that were running when a panic happened, from the newest to the oldest. |
| **debugger** | A tool that runs your program under control: you can pause it, look at variables and go step by step. |
| **LLDB** | The debugger from the LLVM project (Low Level Virtual Machine). It is used on macOS, also inside RustRover. |
| **breakpoint** | A mark on a line of code: the debugger pauses the program when it reaches that line. |
| **step over / into / out** | Debugger commands: run the current line completely / go inside the function being called / run until the current function returns. |
| **watch** | Follow the value of a variable or expression while you step through the code. |
| **debug assertion** | A check (`debug_assert!`) that runs only in debug builds and disappears in release builds. |
| **lint** | An automatic check for suspicious or unidiomatic code. Clippy is a collection of lints. |
| **bisect** | Find the first bad version by repeatedly halving the range of versions (binary search). |
| **regression test** | A test that is written to capture a bug that was fixed, so it cannot return unnoticed. |

## 1. A method, not magic

Beginners often debug by changing things at random and hoping. That wastes time and adds new bugs. Professionals follow a simple loop:

```mermaid
flowchart LR
    A[1. Reproduce] --> B[2. Isolate]
    B --> C[3. Guess the cause]
    C --> D[4. Check the guess]
    D -- wrong --> C
    D -- right --> E[5. Fix]
    E --> F[6. Add a test]
    F --> G[7. Commit]
```

1. **Reproduce.** Find the exact input and steps that make the bug appear every time. Without this you cannot know that you fixed it.
2. **Isolate.** Make the problem small. Which function is wrong? Cut away code that is not involved. A failing test for one function is the ideal result.
3. **Guess the cause.** Form one clear idea: "I think the loop stops one step too early."
4. **Check the guess.** Print or inspect the values (sections 3–5). Do not change code yet; first confirm.
5. **Fix** the cause, not the symptom.
6. **Add a test** (a **regression test**) that fails without the fix and passes with it.
7. **Commit** with a message that says what was wrong (`fix: …`).

Two helpers: explain the problem aloud to a friend or a rubber duck ("rubber duck debugging") — you often find the answer while explaining; and **take a break** — a rested mind sees the typo.

## 2. Read what Rust tells you

Rust gives a lot of information. Learn to read three things.

**The compiler error** (Lesson 00.7): the error code, the line, the `help` text, and `rustc --explain EXXXX`.

**The panic message.** For a runtime bug, the first lines tell you almost everything:

```text
thread 'main' (4149505) panicked at src/main.rs:2:5:
attempt to divide by zero
```

It contains: the thread (`main`), the **file and line** (`src/main.rs:2:5`), and the **reason** (`attempt to divide by zero`). Open that line first.

**The backtrace.** The panic line tells you *where* it failed, but the real cause is often in the code that *called* it. Set the environment variable `RUST_BACKTRACE=1` to see all the functions that were active:

```rust
fn divide(a: i32, b: i32) -> i32 {
    a / b
}

fn average(total: i32, count: i32) -> i32 {
    divide(total, count)
}

fn main() {
    println!("{}", average(10, 2));
    println!("{}", average(10, 0));
}
```

```bash
RUST_BACKTRACE=1 cargo run
```

```text
5

thread 'main' (4149505) panicked at src/main.rs:2:5:
attempt to divide by zero
stack backtrace:
   0: __rustc::rust_begin_unwind
   1: core::panicking::panic_fmt
   2: core::panicking::panic_const::panic_const_div_by_zero
   3: b_backtrace::divide
             at ./src/main.rs:2:5
   4: b_backtrace::average
             at ./src/main.rs:6:5
   5: b_backtrace::main
             at ./src/main.rs:11:20
   6: <fn() as core::ops::function::FnOnce<()>>::call_once
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

(I removed the `at /rustc/...` lines of the standard library to make it shorter.) How to read it:

1. Frames 0–2 are the machinery of the panic inside Rust. **Ignore them**; look for the first frame with the name of *your* crate: `b_backtrace::divide`.
2. Reading downwards goes from the newest call to the oldest: `divide` was called by `average` at line 6, and `average` was called by `main` at line 11, column 20.
3. So the bug is not really in `divide`: `main` asked for `average(10, 0)`, which makes the count `0`. The question becomes "who is allowed to pass zero?". This is the call-stack picture from Lesson 02.4, made visible.

## 3. Print debugging

The simplest tool is to print values at the interesting places. You know `println!("{x}")`. Three habits make it better:

- Use `{:?}` to see the exact value, including quotes and hidden characters (Lesson 02.6).
- Print to **stderr** with `eprintln!`, so debugging output does not mix with the real output.
- Label every value: `eprintln!("i = {i}, total = {total}")`, not just the numbers.

### The `dbg!` macro

`dbg!(expression)` is made for this. It prints the **file, line, the text of the expression and its value** to stderr, in Debug form, and then **returns the value**, so you can wrap it around any expression without changing the program:

```rust
fn main() {
    let a = 2;
    let b = dbg!(a * 3) + 1;
    dbg!(b);
    let pair = (a, b);
    dbg!(pair);
}
```

```text
[src/main.rs:3:13] a * 3 = 6
[src/main.rs:4:5] b = 7
[src/main.rs:6:5] pair = (
    2,
    7,
)
```

What happens: `dbg!(a * 3)` prints `a * 3 = 6` and gives `6` back, so `b` is still `7`. Pretty output (`{:#?}` style) is built in. When the bug is gone, **delete the `dbg!` lines**; `cargo clippy` can warn about leftovers (the `dbg_macro` lint is optional).

### Debug assertions

`assert!` checks stay in the final program. `debug_assert!` checks run **only in debug builds** and are removed in release builds, so you can write checks that would be too slow for the final program:

```rust
fn show_percent(p: u32) -> u32 {
    debug_assert!(p <= 100, "percent out of range: {p}");
    p
}

fn main() {
    println!("{}", show_percent(50));
    println!("{}", show_percent(150));
}
```

Debug build (`cargo run`):

```text
50

thread 'main' (4150421) panicked at src/main.rs:2:5:
percent out of range: 150
```

Release build (`cargo run --release`):

```text
50
150
```

In the release build the check vanishes, and the wrong value passes through. The debug assertion is a **tool to find bugs while you develop**, not a protection for users; for rules that must always hold, use `assert!` or return a `Result`.

> 🔒 **Security:** The difference between debug and release builds matters. Overflow checks (Lesson 02.2) and `debug_assert!` exist only in debug builds. Always run your tests in debug mode (the default for `cargo test`) to get these checks, and also think about what the release build does when the data is wrong.

## 4. A debugger

Print debugging means: stop, edit, rebuild, run, read, repeat. A **debugger** lets you pause the *running* program and look around. The debugger for Rust on macOS is **LLDB**. Cargo builds a debug version with the debug information (`Finished dev profile [unoptimized + debuginfo]`), so LLDB can show your variable names and lines.

Let us debug a function with a bug. It must return `1² + 2² + … + n²`, so for `n = 3` it must be `14`:

```rust
fn sum_of_squares(n: u32) -> u32 {
    let mut total = 0;
    for i in 1..n {
        total += i * i;
    }
    total
}

fn main() {
    println!("sum of squares up to 3 = {}", sum_of_squares(3));
    assert_eq!(sum_of_squares(3), 14);
}
```

```text
sum of squares up to 3 = 5

thread 'main' (4149754) panicked at src/main.rs:11:5:
assertion `left == right` failed
  left: 5
 right: 14
```

We have a **reproduction** (the test fails) and a smaller problem (`sum_of_squares(3)` gives 5; it should give 14). Time to check a guess.

### In RustRover (the graphical debugger)

1. **Set a breakpoint.** Click in the grey gutter to the left of the line number of `total += i * i;`. A red dot appears.
2. **Start the debugger.** Click the green *bug* icon next to the `main` function (or the Debug button in the toolbar). RustRover builds the program with debug information and runs it under LLDB.
3. The program **stops** at the breakpoint, before the line runs. The *Debug* panel shows:
   - **Frames**: the call stack (the same list as in a backtrace);
   - **Variables**: `n = 3`, `total = 0`, `i = 1`;
   - **Watches**: expressions you add yourself, for example `i * i`.
4. Move on with the stepping buttons (the names are the same in all JetBrains IDEs; the macOS shortcuts are in the *Run* menu):

| Command | What it does |
|---|---|
| **Resume** | Run until the next breakpoint or the end. |
| **Step Over** | Run the current line completely, and stop at the next line. Function calls on the line run as a whole. |
| **Step Into** | If the line calls a function, go inside it. |
| **Step Out** | Run until the current function returns to its caller. |
| **Run to Cursor** | Run until the line where the text cursor is. |
| **Evaluate Expression** | Type any expression (for example `i * i + total`) and see its value now. |

5. Press **Resume** and watch the variables: `i` becomes 2 and `total` becomes 1. Press **Resume** once more: the program does not stop again, it just finishes. The loop body ran only **twice**, but we expected three times (for 1, 2 and 3).

That is the answer to our question: the loop `1..n` does not include `n`. Our guess was right: it is the off-by-one error from Lesson 02.5.

### With LLDB in the terminal

The same session can be done in a terminal, which is useful when you work on a server or over SSH (Secure Shell, a way to log in to another computer). Build the program, then start LLDB with the program and a few commands:

```bash
cargo build --bin d_buggy
lldb target/debug/d_buggy
```

Inside LLDB (the `(lldb)` prompt) type:

```text
(lldb) breakpoint set --file d_buggy.rs --line 4
Breakpoint 1: where = d_buggy`d_buggy::sum_of_squares + 88 at d_buggy.rs:4:18, address = 0x00000001000009cc
(lldb) run
Process 79558 stopped
* thread #1, name = 'main', queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
      frame #0: 0x00000001000009cc d_buggy`d_buggy::sum_of_squares(n=3) at d_buggy.rs:4:18
   1   	fn sum_of_squares(n: u32) -> u32 {
   2   	    let mut total = 0;
   3   	    for i in 1..n {
-> 4   	        total += i * i;
    	                 ^
   5   	    }
   6   	    total
   7   	}
(lldb) frame variable
(unsigned int) n = 3
(unsigned int) total = 0
(core::ops::range::Range<unsigned int>) iter = (start = 2, end = 3)
(unsigned int) i = 1
(lldb) continue
Process 79558 stopped
...
(lldb) frame variable
(unsigned int) n = 3
(unsigned int) total = 1
(core::ops::range::Range<unsigned int>) iter = (start = 3, end = 3)
(unsigned int) i = 2
(lldb) quit
```

The important commands:

| LLDB command | Short | Meaning |
|---|---|---|
| `breakpoint set --file f.rs --line N` | `b f.rs:N` | Set a breakpoint. |
| `run` | `r` | Start the program. |
| `frame variable` | `fr v` | Show the variables of the current function. |
| `next` | `n` | Step over. |
| `step` | `s` | Step into. |
| `finish` | | Step out. |
| `continue` | `c` | Resume. |
| `bt` | | Show the backtrace. |
| `quit` | `q` | Leave LLDB. |

(`rust-lldb` is a wrapper in your Rust toolchain that adds Rust-specific pretty printers; it is used the same way. The `iter = (start = 3, end = 3)` line is the range `1..n` after the second step: the range is empty, so the loop ends. It confirms the diagnosis.)

### The fix and its test

The loop must include `n`: write `1..=n`. Before we fix it, we write a test that fails now and passes after the fix (the regression test), with edge cases `n = 1` and `n = 0`:

```rust
fn sum_of_squares(n: u32) -> u32 {
    let mut total = 0;
    for i in 1..=n {
        total += i * i;
    }
    total
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_squares_including_n() {
        assert_eq!(sum_of_squares(3), 14);
        assert_eq!(sum_of_squares(1), 1);
        assert_eq!(sum_of_squares(0), 0);
    }
}
```

> 💡 **Tip:** When should you use which tool? **`println!`/`dbg!`** for a quick look at one or two values. **A debugger** when you need to follow a long chain of events, see many variables at once, or when the bug disappears when you add prints. **A test** always, to keep the bug from returning.

## 5. Let the tools find bugs for you

Before you debug, ask the tools.

- **`cargo check`** — fast compile errors, without building the program.
- **`cargo clippy`** — hundreds of extra checks. Here is what it says about code that compiles but is not good:

```rust
fn clamp_score(x: i32) -> i32 {
    if x >= 0 && x <= 100 {
        return x;
    }
    return 100;
}

fn main() {
    let v = [1, 2, 3];
    for i in 0..v.len() {
        println!("{}", v[i]);
    }
    println!("{}", clamp_score(150));
}
```

```text
warning: unneeded `return` statement
 --> src/main.rs:5:5
  |
5 |     return 100;
  |     ^^^^^^^^^^
  |
  = note: `#[warn(clippy::needless_return)]` on by default
help: remove `return`

warning: manual `RangeInclusive::contains` implementation
 --> src/main.rs:2:8
  |
2 |     if x >= 0 && x <= 100 {
  |        ^^^^^^^^^^^^^^^^^^ help: use: `(0..=100).contains(&x)`

warning: the loop variable `i` is only used to index `v`
  --> src/main.rs:10:14
   |
10 |     for i in 0..v.len() {
   |              ^^^^^^^^^^
```

Clippy does not find the logic of your program, but it finds the shapes in which bugs love to hide (a range loop used only for indexing is where off-by-one mistakes live). **Treat Clippy warnings as part of the work**: run `cargo clippy` before every commit.

- **`cargo test`** — your tests, always.
- **RustRover** — the red and yellow marks in the editor are the compiler and Clippy working while you type. Hover over them to read the message.

## 6. Git as a debugging tool

Version control remembers every state of your code. Three commands help to find bugs:

**What did I change?** When something worked an hour ago and does not work now:

```bash
git status            # which files changed
git diff              # the exact lines changed since the last commit
git diff HEAD~3       # the changes compared with three commits ago
git log -p -n 3       # the last three commits with their changes
```

**Go back and try.** `git stash` puts your uncommitted changes aside so that you can check whether the last commit works ([Lesson 01.5](../01-git-and-version-control/05-undoing-changes-safely.md)); `git stash pop` brings them back.

**Which commit broke it? `git bisect`.** If you have many commits and know that an old one was good and the current one is bad, `git bisect` finds the first bad commit by **binary search**: it checks out the commit in the middle, you say "good" or "bad", and it halves the range again. For 1000 commits you need only about 10 checks. Even better, it can run a command for you: a command that exits with success for a good commit and with failure for a bad one — a test!

Here is a real example. A small project has 8 commits. The function `average` was correct, then one commit "simplified" it to `a + b / 2` (wrong operator precedence — Lesson 02.3), and the test `average(4, 6) == 5` started to fail:

```text
$ git log --oneline
840d9ae docs: add note 6
bea4b60 docs: add note 5
1e352c0 docs: add note 4
fa9e438 refactor: simplify average
058a626 docs: add note 3
d91fb94 docs: add note 2
b999561 docs: add note 1
50cf810 feat: add average
```

We write a tiny script `check.sh` that runs the tests and returns success or failure, then:

```bash
git bisect start
git bisect bad HEAD                 # the current version is broken
git bisect good 50cf810             # the first commit was fine
git bisect run ./check.sh           # let Git test the commits in the middle
```

```text
Bisecting: 3 revisions left to test after this (roughly 2 steps)
[058a6268025c0964fd3c294476c5cce69679d5cc] docs: add note 3
...
fa9e438f585e485a161d5e24b5f8f12150c49835 is the first bad commit
commit fa9e438f585e485a161d5e24b5f8f12150c49835
Author: Demo <demo@example.com>

    refactor: simplify average

 calc/src/lib.rs | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
bisect found first bad commit
```

```bash
git bisect reset                    # go back to where you were
git show fa9e438                    # read the change that caused the bug
```

Git tested only 3 of the 8 commits and named the culprit, together with its message and its diff. **Small commits with clear messages** (the habit from Phase 01) are what make `bisect` useful: it points to a commit of 5 lines instead of a commit of 500.

## 7. Common bug families

| Bug | Typical sign | Where to look |
|---|---|---|
| Off-by-one | The last (or first) item is missing; an index panic | Range ends `..` vs `..=`; `len()` versus `len() - 1`; `<` versus `<=` |
| Integer division | A result that is "rounded down" | `/` on integers (Lesson 02.3) |
| Overflow | A panic `attempt to add with overflow`, or a wrapped result in release | Types too small (Lesson 02.2) |
| Wrong operator precedence | A correct-looking formula gives wrong numbers | Add parentheses (Lesson 02.3) |
| Forgotten `trim()` | "invalid digit found in string" for input that looks right | Input handling (Lesson 02.7) |
| Shadowing surprise | A variable has an old value after a block | `let x` inside a block creates a new `x` (Lesson 02.1) |
| Infinite loop | The program hangs, the fan spins | The loop variable is never changed |
| Mixed units | Results are off by a factor | Floats vs integers, seconds vs milliseconds |

## Common mistakes

**1. Changing code before understanding the bug.** First reproduce and confirm the cause.

**2. Fixing the symptom.** Adding a special case so that "this input works" leaves the real bug in place.

**3. Leaving `dbg!` and debug prints in the code.** Remove them before the commit.

**4. Debugging a release build.** Debug information and variable values are missing in an optimized build; debug with `cargo build` / `cargo run` (the dev profile).

**5. Not writing the regression test.** The bug will come back.

**6. Changing many things at once.** Change one thing, run, look.

**7. Debugging when tired.** Take a break first.

> 🧪 **Testing:** A good bug report is a failing test. When someone says "your function gives a wrong answer for 3", the first thing you do is write `assert_eq!(f(3), expected)` and watch it fail. After the fix, keep the test forever.

> ⚡ **Performance:** `dbg!` and `println!` in a hot loop can slow a program by a factor of a hundred, because printing is slow (Lesson 02.6). When you debug a slow program, do not trust timings of a build with prints, and never measure speed in a debug build (Phase 05, Lesson 12).

## Try it yourself

1. In `p02-basics`, create `src/bin/09_debug.rs` with the buggy `sum_of_squares`. Run it and see the assertion fail.
2. Add `dbg!(i)` and `dbg!(total)` inside the loop; run, read the output, and note how many times the body runs.
3. Set a breakpoint in RustRover on the line `total += i * i;`. Start the debugger (the bug icon). Step with *Resume*; watch `i` and `total`. Add a watch `i * i`.
4. Do the same session with `lldb target/debug/09_debug` in the terminal (`breakpoint set`, `run`, `frame variable`, `continue`).
5. Fix the function (`1..=n`) and move your check into a `#[test]`. Run `cargo test --bin 09_debug`.
6. Run `RUST_BACKTRACE=1 cargo run --bin <name>` for the divide-by-zero program of section 2 and find the line in `main` that caused the panic.
7. Run `cargo clippy`. Commit the fix with a `fix:` message:

```bash
cd ~/projects/rust
git add code/p02-basics
git commit -m "fix(p02): include n in sum of squares"
```

## Exercises

### Exercise 1 — Find the bug with `dbg!` (easy)

This program should print `average of 4 and 6 = 5`, but it prints `7`. Use `dbg!` to look at the parts of the expression, find the problem, and fix it. The average of `1` and `2` must be `1.5`.

```rust
fn average(a: i32, b: i32) -> f64 {
    (a + b / 2) as f64
}

fn main() {
    println!("average of 4 and 6 = {}", average(4, 6));
    assert_eq!(average(4, 6), 5.0);
}
```

<details>
<summary>Hint</summary>

Write `dbg!(b / 2)` and `dbg!(a + b / 2)`. Which operator is evaluated first? (Lesson 02.3: precedence.) Also remember that integer division cuts off the fraction.

</details>

<details>
<summary>Answer</summary>

```rust
fn average(a: i32, b: i32) -> f64 {
    (a + b) as f64 / 2.0
}

fn main() {
    println!("average of 4 and 6 = {}", average(4, 6));
    assert_eq!(average(4, 6), 5.0);
    assert_eq!(average(1, 2), 1.5);
}
```

```text
average of 4 and 6 = 5
```

Why it works: `/` has higher precedence than `+`, so the old code computed `4 + (6 / 2) = 7`. The fix adds the numbers first, converts to `f64` to keep the fraction, and divides by `2.0`. The second assertion checks a case with a fraction.

</details>

### Exercise 2 — Read a backtrace (easy)

Here is part of a backtrace of a program that panicked:

```text
thread 'main' panicked at src/main.rs:2:5:
attempt to divide by zero
stack backtrace:
   2: core::panicking::panic_const::panic_const_div_by_zero
   3: b_backtrace::divide
             at ./src/main.rs:2:5
   4: b_backtrace::average
             at ./src/main.rs:6:5
   5: b_backtrace::main
             at ./src/main.rs:11:20
```

Answer: (a) In which function and on which line did the panic happen? (b) Who called that function, and from which line? (c) Which line of `main` do you have to change or check?

<details>
<summary>Hint</summary>

Ignore the frames from the standard library. Go down the list: each frame is the *caller* of the frame above it.

</details>

<details>
<summary>Answer</summary>

(a) In `divide`, line 2, column 5 (the expression `a / b`). (b) `average` called it at line 6; `average` was called by `main` at line 11, column 20. (c) Line 11 of `main`: `average(10, 0)` passes a count of zero. You should either not call it with zero or make `average` check `count == 0` and decide what to do.

Why it works: the panic line shows where the program *crashed*; the backtrace shows the *path* that led there, and the cause is often a few frames up.

</details>

### Exercise 3 — Fix with a regression test (medium)

Take the buggy `sum_of_squares` from section 4. First write a `#[test]` that fails (check `3 → 14`, `1 → 1`, `0 → 0`) and read the failure. Then fix the function. Then run the test again.

<details>
<summary>Hint</summary>

The failing run shows `left: 5`, `right: 14`: that is `1 + 4`. The value `3 * 3 = 9` is missing, so the loop does not reach `n`. Change `1..n` into `1..=n`.

</details>

<details>
<summary>Answer</summary>

```rust
fn sum_of_squares(n: u32) -> u32 {
    let mut total = 0;
    for i in 1..=n {
        total += i * i;
    }
    total
}

fn main() {
    println!("sum of squares up to 3 = {}", sum_of_squares(3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_squares_including_n() {
        assert_eq!(sum_of_squares(3), 14);
        assert_eq!(sum_of_squares(1), 1);
        assert_eq!(sum_of_squares(0), 0);
    }
}
```

Why it works: for `n = 0` the range `1..=0` is empty and the sum is `0` — a correct edge case. The test now documents all three cases and protects against the bug returning.

</details>

### Exercise 4 — Two bugs in one function (medium)

The function should count the even digits of a number (`2468 → 4`, `1357 → 0`, `4 → 1`, `1020 → 3`). The test fails. Find **both** bugs, with `dbg!` or the debugger, and fix them.

```rust
fn count_even_digits(mut n: u32) -> u32 {
    let mut count = 0;
    while n > 10 {
        if n % 10 % 2 == 0 {
            count += 1;
        }
        n = n / 100;
    }
    count
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_even_digits() {
        assert_eq!(count_even_digits(2468), 4);
        assert_eq!(count_even_digits(1357), 0);
        assert_eq!(count_even_digits(4), 1);
        assert_eq!(count_even_digits(1020), 3);
    }
}
```

<details>
<summary>Hint</summary>

The first failure shows `left: 2`, `right: 4` for `2468`: half of the digits are skipped. Print `n` at the start of each iteration. Then check the loop condition with a single-digit number such as `4`.

</details>

<details>
<summary>Answer</summary>

```rust
fn count_even_digits(mut n: u32) -> u32 {
    let mut count = 0;
    while n > 0 {
        if n % 10 % 2 == 0 {
            count += 1;
        }
        n = n / 10;
    }
    count
}
```

Why it works: bug 1 — `n / 100` removes **two** digits at a time, so every second digit is skipped (`2468` → `24` → `0`, counting only `8` and `4`); it must be `n / 10`. Bug 2 — the condition `n > 10` stops before the last digit when `n` is a single digit (`4` is not bigger than 10); it must be `n > 0`. After the fix all four assertions pass. (The number `0` has one digit but this function returns `0` for it; deciding what is right for `0` is a design question for a new test.)

</details>

### Exercise 5 — Find the bad commit with `git bisect` (hard)

Run this script in a temporary folder (not in your learning repository). It creates a repository with 8 commits; the file `value.txt` is correct while its value is below 50.

```bash
mkdir /tmp/bisect-demo && cd /tmp/bisect-demo && git init -q -b main .
for n in 10 20 30 40 50 60 70 80; do echo $n > value.txt; git add value.txt; git commit -q -m "set value to $n"; done
git log --oneline
```

Use `git bisect` with `git bisect run` to find the first commit where the value is 50 or more. The check command is `sh -c 'test "$(cat value.txt)" -lt 50'` (it succeeds when the value is below 50). Then clean up with `git bisect reset`.

<details>
<summary>Hint</summary>

Start with `git bisect start`, mark `HEAD` as bad and `HEAD~7` (the first commit) as good, then use `git bisect run` followed by the command.

</details>

<details>
<summary>Answer</summary>

```bash
git bisect start
git bisect bad HEAD
git bisect good HEAD~7
git bisect run sh -c 'test "$(cat value.txt)" -lt 50'
git bisect reset
```

The output ends with:

```text
aa6339cdc8ee7db17485442f0dc3ca6c19fb4109 is the first bad commit
commit aa6339cdc8ee7db17485442f0dc3ca6c19fb4109
Author: Demo <demo@example.com>

    set value to 50

 value.txt | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
bisect found first bad commit
```

(Your commit hash and author will differ.) Why it works: Git tested only the commits `40` and `50` — it halved the range each time — and found that the commit `set value to 50` is the first for which the command fails. `git bisect reset` returns you to `main`.

</details>

## Quiz

1. What are the first two steps of debugging?

<details>
<summary>Answer</summary>

Reproduce the bug reliably, and isolate it (make the problem as small as possible, ideally a failing test).

</details>

2. What does `dbg!(x * 2)` print, and what does it return?

<details>
<summary>Answer</summary>

It prints the file, the line, the text `x * 2` and its value to stderr, and it returns the value, so the program continues unchanged.

</details>

3. How do you get a backtrace for a panic, and how do you read it?

<details>
<summary>Answer</summary>

Run with `RUST_BACKTRACE=1`. Skip the frames of the standard library, find the first frame of your own crate, and read downwards: each frame is the caller of the frame above it.

</details>

4. What is the difference between *Step Over* and *Step Into* in a debugger?

<details>
<summary>Answer</summary>

*Step Over* runs the current line without stopping inside the functions it calls. *Step Into* enters the called function and stops at its first line.

</details>

5. Why can `debug_assert!` pass in release mode when the same condition fails in debug mode?

<details>
<summary>Answer</summary>

`debug_assert!` is removed from release builds, so the check does not run at all.

</details>

6. What does `git bisect run ./check.sh` do?

<details>
<summary>Answer</summary>

It automatically checks out commits between a good and a bad one, runs the script on each, and uses binary search to find the first commit for which the script fails.

</details>

7. What should you do after fixing a bug, besides committing?

<details>
<summary>Answer</summary>

Write (or keep) a regression test that fails without the fix and passes with it, so the bug cannot return unnoticed.

</details>

## Summary

- Debug with a method: reproduce, isolate, guess, check, fix, add a test, commit.
- Read the panic message (file, line, reason) and the backtrace (the path of calls; find your own crate's frames).
- `println!`/`eprintln!` with `{:?}` and labels, `dbg!` for quick looks, `debug_assert!` for checks in debug builds only.
- A debugger (RustRover or `lldb`) lets you pause at breakpoints, step over/into/out, and watch variables.
- `cargo check`, `cargo clippy` and `cargo test` find many bugs before you start debugging.
- Git helps: `git diff` shows what changed, `git stash` sets changes aside, `git bisect` finds the commit that introduced a bug.
- Small commits and a regression test for each bug make the next bug easier.

## Sources and further reading

- [The Rust Programming Language — 9.1 Unrecoverable Errors with panic!](https://doc.rust-lang.org/book/ch09-01-unrecoverable-errors-with-panic.html) — panics and backtraces in the official book.
- [Standard library — dbg!](https://doc.rust-lang.org/std/macro.dbg.html) — the documentation of the `dbg!` macro.
- [The LLDB Tutorial](https://lldb.llvm.org/use/tutorial.html) — the official introduction to LLDB commands.
- [RustRover documentation — Debugging](https://www.jetbrains.com/help/rust/debugging-code.html) — breakpoints, stepping and watches in the IDE.
- [Clippy documentation](https://doc.rust-lang.org/clippy/) — how to use Clippy and the list of lints.
- [Git documentation — git-bisect](https://git-scm.com/docs/git-bisect) — the full manual of `git bisect`, including `run`.
- [Pro Git — Debugging with Git](https://git-scm.com/book/en/v2/Git-Tools-Debugging-with-Git) — `git bisect` and `git blame` in the free Pro Git book.

---

[🏠 Start](../START-HERE.md) · [← 02.8 Your first tests](08-your-first-tests.md) · [📋 Phase 02](README.md) · [02.10 Project: the guessing game →](10-project-guessing-game.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
