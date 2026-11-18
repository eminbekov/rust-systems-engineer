<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 00.6 Your first Rust program](06-your-first-rust-program.md) · [📋 Phase 00](README.md) · [00.8 Study routine, notes in Markdown and checkpoint →](08-study-routine-notes-and-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/00-getting-started/07-reading-errors-and-getting-help.md) <!-- nav -->

# Lesson 00.7 — Reading errors and getting help

> **Phase 00 — Getting started** · Lesson 7 of 8 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The four kinds of trouble you meet: compile errors, warnings, panics and logic bugs.
- How to read every part of a Rust compiler error message.
- A calm routine for fixing many errors: from the top, one at a time.
- How to use `rustc --explain`, the offline documentation and the web.
- How to ask a good question to people or to an AI assistant.

## Before you start

- You finished [Lesson 00.6 — Your first Rust program](06-your-first-rust-program.md). You can create a project with `cargo new` and run it with `cargo run`.
- Create a scratch project for experiments: `cd ~/projects/rust && cargo new code/p00-errors`.

## New words

| Word | Meaning |
|---|---|
| **compile error** | A mistake that the compiler finds before the program runs. No executable is produced. |
| **warning** | A message about code that compiles but looks suspicious. The program is still built. |
| **panic** | A run-time failure: the program starts, then stops itself with an error message because something impossible happened. |
| **logic bug** | The program runs and does not complain, but gives a wrong result. |
| **error code** | A name such as `E0425` that identifies the kind of compile error. Every code has a written explanation. |
| **span** | The part of the code that a message points to, shown with `^^^^`. |
| **exit code** | A number that every program returns when it ends. `0` means success; anything else means failure. The shell shows it in `$?`. |
| **backtrace** | A list of the function calls that led to a panic, from the failure back to the start. |
| **minimal reproducible example** | The smallest complete program that still shows your problem, so others can run it and see it. |
| **documentation** (docs) | The official description of a language, a library or a tool. |
| **API** (Application Programming Interface) | The set of functions and types that a library offers to the programs that use it. |
| **AI** (Artificial Intelligence) assistant | A program such as Claude that answers questions in natural language. |

## 1. Four kinds of trouble

| Kind | When | What you see | Example |
|---|---|---|---|
| **Compile error** | `cargo build` / `cargo run` | `error[E....]` and no program | a misspelled name |
| **Warning** | the same time | `warning: ...`; the program still runs | an unused variable |
| **Panic** | while the program runs | `thread 'main' panicked at ...`, exit code 101 | the code calls `panic!` |
| **Logic bug** | while the program runs | nothing: the result is just wrong | `+` instead of `*` |

Each needs a different approach. This lesson teaches all four.

> 💡 **Tip:** Compile errors may feel annoying, but they are good news: the compiler found the
> mistake *for* you, before a user did, and told you where it is. In Rust, many bugs that other
> languages find only when the program runs are compile errors.

## 2. The anatomy of a compiler error

Take this tiny program (in `src/main.rs` of the scratch project). The variable is `city`, but
the print line says `citi`:

```rust
fn main() {
    let city = "Tashkent";
    println!("Hello from {}!", citi);
}
```

`let city = "Tashkent";` creates a variable `city` (Lesson 02.1 teaches it). Now `cargo run`:

```text
   Compiling p00-errors v0.1.0 (/Users/student/projects/rust/code/p00-errors)
error[E0425]: cannot find value `citi` in this scope
 --> src/main.rs:3:32
  |
3 |     println!("Hello from {}!", citi);
  |                                ^^^^ not found in this scope
  |
help: a local variable with a similar name exists
  |
3 -     println!("Hello from {}!", citi);
3 +     println!("Hello from {}!", city);
  |

For more information about this error, try `rustc --explain E0425`.
error: could not compile `p00-errors` (bin "p00-errors") due to 1 previous error
```

Every part has a meaning:

| Part | Text | Meaning |
|---|---|---|
| Severity | `error` | It is an error: no program will be built. (`warning` would be less serious.) |
| Code | `[E0425]` | The kind of error. You can ask for its explanation. |
| Message | `cannot find value `citi` in this scope` | One sentence that says what is wrong: the name `citi` is not known here. |
| Location | `--> src/main.rs:3:32` | `file : line : column`. Line 3, 32nd character. RustRover makes this a clickable link. |
| Snippet | `3 \|     println!(...)` | The line of code, with its number. |
| Marker | `^^^^ not found in this scope` | The exact part (the *span*) and a short label. |
| Help | `help: a local variable with a similar name exists` | A suggestion. In the little diff below it, a line with `-` is what you have now, `+` is the suggested new text. |
| Footer | `For more information ... rustc --explain E0425` | How to read more about this error. |
| Summary | `error: could not compile ... due to 1 previous error` | The count of errors. |

> 📏 **Convention:** Write the location as `file:line:column` when you talk about an error. Everyone
> understands this notation, and most editors can jump to it.

A compile error that says `warning` or `error` at the start is a *message from the compiler*: it is
not a judgement of you. The compiler is the most patient teacher you will have. Read it.

## 3. Many errors? Start at the top

Sometimes you get a screenful of errors. Do not panic. Here is a program with several mistakes:

```rust
fn main() {
    let city = "Tashkent"
    println("Hello from {}!", citi);
    let age: i32 = "twenty";
}
```

Running it shows **four** errors (shortened here; the real output is longer because of the help blocks):

```text
error: expected `;`, found `println`
 --> src/main.rs:2:26
  |
2 |     let city = "Tashkent"
  |                          ^ help: add `;` here
...
error[E0425]: cannot find value `citi` in this scope
 --> src/main.rs:3:31
...
error[E0423]: cannot find function `println` in this scope
 --> src/main.rs:3:5
...
error[E0308]: mismatched types
 --> src/main.rs:4:20
  |
4 |     let age: i32 = "twenty";
  |              ---   ^^^^^^^^ expected `i32`, found `&str`
  |              |
  |              expected due to this
...
error: could not compile `p00-errors` (bin "p00-errors") due to 4 previous errors
```

The routine:

1. **Read the first error.** Not the last. Later errors can be *caused* by earlier ones.
2. **Fix only that one** (or the obvious group on one line).
3. **Run again.** The list changes.
4. Repeat until there are no errors.

Let us follow it:

| Round | Errors | What we fix |
|---|---|---|
| 1 | 4 | line 2: add the missing `;` after `"Tashkent"` |
| 2 | 3 | line 3: `println` → `println!` |
| 3 | 2 | line 3: `citi` → `city` |
| 4 | 1 | line 4: `"twenty"` is text, but `age` was declared as a number (`i32`) → write `20` |
| 5 | 0 | the program runs |

(You may fix two errors in one round; the table shows one change per round to keep the idea clear. I ran
every round to get the real counts: 4, 3, 2, 1 and 0.) The final program prints:

```text
Hello from Tashkent!
Age: 20
```

(after we also added `println!("Age: {}", age);` to use the variable.)

### The compiler's suggestion is a guess

Look at this one:

```rust
fn main() {
    let nmae = "Ada";
    println!("Hello, {}!", name);
    println!("Bye, {}!", name);
}
```

```text
error[E0425]: cannot find value `name` in this scope
 --> src/main.rs:3:28
...
help: a local variable with a similar name exists
3 -     println!("Hello, {}!", name);
3 +     println!("Hello, {}!", nmae);
...
error[E0425]: cannot find value `name` in this scope
 --> src/main.rs:4:26
...
error: could not compile `p00-errors` (bin "p00-errors") due to 2 previous errors
```

The compiler suggests changing the *uses* to `nmae`, because that is the similar name it found. But the real
mistake is a typo in the **declaration** (`nmae` should be `name`). One fix there removes both errors.
**The compiler knows the syntax, but you know the intention.** Always ask: "which fix is what I meant?"

## 4. `rustc --explain`: the manual of every error

Every error with a code has a long explanation, with examples. It comes with the toolchain, so it also works offline:

```text
% rustc --explain E0425
An unresolved name was used.

Erroneous code examples:

...

Please verify that the name wasn't misspelled and ensure that the
identifier being referred to is valid for the given situation.
```

(Shortened.) The first line says in plain words what the code means. Then come examples of wrong code and
of the fix. The same texts are on the web at
[doc.rust-lang.org/error_codes/E0425.html](https://doc.rust-lang.org/error_codes/E0425.html), and offline
via `rustup doc --error-codes`.

Another very common code is `E0308`, "mismatched types": something of one type was found where another type was needed:

```text
error[E0308]: mismatched types
 --> src/main.rs:2:20
  |
2 |     let age: i32 = "twenty";
  |              ---   ^^^^^^^^ expected `i32`, found `&str`
  |              |
  |              expected due to this
```

How to read it: `---` marks *why* the compiler expected a number (`i32` was written next to `age`), `^^^^` marks what it found
instead (`&str` is the type of text in quotes). The notation `expected X, found Y` is the same in all such errors. Types are the topic of Lesson 02.2.

## 5. Warnings: do not ignore them

A warning means: "this compiles, but it looks wrong".

```rust
fn main() {
    let name = "Ada";
    println!("Hello!");
}
```

```text
warning: unused variable: `name`
 --> src/main.rs:2:9
  |
2 |     let name = "Ada";
  |         ^^^^ help: if this is intentional, prefix it with an underscore: `_name`
  |
  = note: `#[warn(unused_variables)]` (part of `#[warn(unused)]`) on by default

warning: `p00-errors` (bin "p00-errors") generated 1 warning
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
     Running `target/debug/p00-errors`
Hello!
```

What happens:

1. The program was built and it ran (see `Finished` and `Hello!`). A warning does not stop it.
2. The compiler noticed that `name` is created but never used. Maybe you forgot to print it.
3. There are two honest fixes: use the variable (`println!("Hello, {}!", name);`), or, if you really do not need it, rename it `_name`. A leading underscore means "I know it is unused".

> 📏 **Convention:** A professional project has **zero warnings**. One ignored warning hides the next one that
> matters. Treat `cargo clippy` messages the same way.

## 6. Panics: errors that happen while the program runs

```rust
fn main() {
    println!("Starting.");
    panic!("something went wrong");
}
```

`panic!` is a macro that stops the program on purpose with a message. Real programs panic when something
impossible happens (you will meet the real causes in Phases 02–04). It compiles without problems and fails at run time:

```text
Starting.

thread 'main' (3893372) panicked at src/main.rs:3:5:
something went wrong
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

What happens:

1. `Starting.` — the program ran normally until the panic.
2. `thread 'main' ... panicked at src/main.rs:3:5:` — which part of the program ("thread", Lesson 05.10; `main` is the first one), where (`file:line:column`), and what: the message on the next line. The number in the brackets is an internal id and is different every run.
3. `note: run with RUST_BACKTRACE=1 ...` — an offer to show more details.
4. The program ends with **exit code 101**. The shell remembers the exit code of the last command in `$?`:

```text
% ./target/debug/p00-errors
Starting.

thread 'main' (3893401) panicked at src/main.rs:3:5:
something went wrong
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
% echo $?
101
```

(Success is `0`. `cargo run` also exits with `101` when the *build* fails.)

Set the environment variable `RUST_BACKTRACE=1` for one run to see the **backtrace**:

```text
% RUST_BACKTRACE=1 cargo run
Starting.

thread 'main' (3893401) panicked at src/main.rs:3:5:
something went wrong
stack backtrace:
   0: __rustc::rust_begin_unwind
   1: core::panicking::panic_fmt
   2: p00_errors::main
             at ./src/main.rs:3:5
   3: <fn() as core::ops::function::FnOnce<()>>::call_once
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

How to read it: the list shows who called whom. Most lines belong to Rust's own library. **Look for the lines with
your project name** (`p00_errors::main` and `at ./src/main.rs:3:5`): that is the place in *your* code. Here
it is the same line that the first message gave. In bigger programs the backtrace shows the *path* that
led to the failure.

## 7. Logic bugs: no message at all

This program converts Celsius to Fahrenheit. The formula is `F = C × 9 / 5 + 32`, and 100 °C must be 212 °F:

```rust
fn main() {
    let celsius = 100;
    let fahrenheit = celsius * 9 / 5 - 32;
    println!("{} C is {} F", celsius, fahrenheit);
}
```

```text
100 C is 148 F
```

It compiles and runs, but 148 is wrong. Nothing tells you where the problem is. Your tools:

1. **Know the right answer.** You know that 100 °C = 212 °F. Without an expected result you cannot see a bug.
2. **Look at the steps.** Print the middle values:

```rust
fn main() {
    let celsius = 100;
    let scaled = celsius * 9 / 5;
    println!("scaled = {}", scaled);
    let fahrenheit = scaled - 32;
    println!("{} C is {} F", celsius, fahrenheit);
}
```

```text
scaled = 180
100 C is 148 F
```

3. `scaled` is right (100 × 9 / 5 = 180). The next line is wrong: `scaled - 32`. The formula needs `+ 32`. After the fix the program prints `100 C is 212 F`.
4. **Explain your code to a "rubber duck"** (a toy, a friend, or a note to yourself). Saying each line aloud often shows the
   mistake.
5. **Make the problem smaller.** Delete everything that is not needed to reproduce the bug.

Printing the middle values (a technique called "print debugging") is the simplest tool. Phase 02
(Lesson 02.9) teaches the next one, a real debugger, and Phase 09 teaches debugging in depth.

> 🧪 **Testing:** The best protection against logic bugs is a **test**: a small program that checks
> that 100 °C gives 212 °F. You will write your first tests in Lesson 02.8.

## 8. Finding answers: where to look, in order

When you are stuck, go from the closest and fastest source to the farthest:

1. **The error message itself.** Read it twice, slowly. Look at the `help:` and `note:` lines.
2. **`rustc --explain E....`** for errors with a code.
3. **The offline documentation:** `rustup doc --std` (the standard library), `rustup doc --book`. In the web version of the standard-library docs press `S` to focus the search box and type a name, for example `println`.
4. **Online documentation:** [docs.rs](https://docs.rs/) has the documentation of every library (crate) on crates.io; [Rust by Example](https://doc.rust-lang.org/rust-by-example/) has short examples to run.
5. **Search the web.** Copy the message text *without* your own names and paths and put it in quotes, for example `"cannot find value" "in this scope"`. Prefer results from `doc.rust-lang.org`, `users.rust-lang.org` and Stack Overflow, and check how old the answer is (Rust changes; an answer from 2016 may be out of date).
6. **Ask people.** The official forum is [users.rust-lang.org](https://users.rust-lang.org/); the page [rust-lang.org/community](https://www.rust-lang.org/community) lists chat rooms and other places.
7. **Ask an AI assistant**, with the rules from Lesson 00.1: try yourself first, ask for an explanation or a hint, and check the answer by running the code.

> 💡 **Tip — reading English documentation:** You do not need to understand every word. Read the
> *signature* (the code at the top that shows the name and types), the first sentence of the description and the
> *Examples* section. Keep a small list of words that you meet often (for example *expected*,
> *found*, *borrowed*, *moved*), with a translation, in your notes. After a month you will not need it.

## 9. How to ask a good question

A good question gets a fast answer. A bad question ("it doesn't work, help") gets nothing. Include:

1. **Goal:** what you are trying to do.
2. **Code:** the **minimal reproducible example**: the smallest program that still shows the problem. As text, not a photo or a screenshot.
3. **The exact error message**, copied as text.
4. **What you expected and what happened.**
5. **What you already tried.**
6. **Versions and system:** the output of `rustc --version`, and "macOS on an Apple M4".

An example of a good question:

```text
Title: "cannot find value in this scope" although the variable exists

Goal: print a greeting that uses the variable `city`.

Code:
    fn main() {
        let city = "Tashkent";
        println!("Hello from {}!", citi);
    }

Error:
    error[E0425]: cannot find value `citi` in this scope
     --> src/main.rs:3:32

I expected to see "Hello from Tashkent!".
I checked that the variable `city` is declared on line 2.
rustc 1.100.0 (0ef1bc386 2026-11-09), macOS on Apple M4.
```

While you write such a question you often notice the typo yourself. This is called the "rubber duck" effect, and it
is one of the main reasons to write the question carefully.

> 🔒 **Security:** Before you paste code or logs anywhere (a forum, a chat, an AI assistant), remove
> **passwords, API keys, tokens, private keys and private data**. Also be careful with long paths: they contain your
> user name. Once published on the internet, a secret must be considered stolen.

## Common mistakes

1. **Reading only the last line.** `error: could not compile ... due to 4 previous errors` says nothing useful. The information is *above* it. *Fix:* start at the first `error`.
2. **Fixing the last error first.** Often it disappears when the first one is fixed. *Fix:* from the top.
3. **Changing many things at once.** You no longer know which change helped. *Fix:* one change, one run.
4. **Applying every suggestion blindly.** `help:` is a guess (see `nmae`). *Fix:* ask what you meant.
5. **Ignoring warnings.** They pile up and hide real problems. *Fix:* zero warnings.
6. **Random changes** ("maybe if I add `&` here..."). *Fix:* read the message, form a hypothesis, test it.
7. **"It doesn't work" questions** without code, error and versions. *Fix:* use the template in section 9.
8. **Sending screenshots of text.** Nobody can copy from a picture or search it. *Fix:* paste text in a code block.

## Try it yourself

1. In your scratch project `~/projects/rust/code/p00-errors`, put into `src/main.rs`:

```rust
fn main() {
    let city = "Tashkent"
    println("Hello from {}!", citi);
    let age: i32 = "twenty";
}
```

2. Run `cargo run`. Count the errors. Write the number down: it should be 4.
3. Fix **only the first** error. Run again. Does the count change?
4. Continue, one fix per run, until the program runs. Add `println!("Age: {}", age);` at the end so that `age` is used.
5. Run `rustc --explain E0308` and read the first lines. Write one sentence in your own words.
6. Add `panic!("test");` at the end of `main`, run it, and find the line and column in the output. Then run `echo $?`.
7. Remove the `panic!`.

## Exercises

### Exercise 1 — Read the message (easy)

Look at this error and answer: (a) the error code, (b) the file, line and column, (c) what is wrong in one
sentence, (d) the compiler's suggested fix.

```text
error[E0425]: cannot find value `citi` in this scope
 --> src/main.rs:3:32
  |
3 |     println!("Hello from {}!", citi);
  |                                ^^^^ not found in this scope
  |
help: a local variable with a similar name exists
  |
3 -     println!("Hello from {}!", citi);
3 +     println!("Hello from {}!", city);
  |
```

<details>
<summary>Hint</summary>

The code is in square brackets after `error`. The location comes after `-->`. The suggestion is in the `help:` block.

</details>

<details>
<summary>Answer</summary>

(a) `E0425`. (b) `src/main.rs`, line 3, column 32. (c) The name `citi` does not exist in this place; probably a typo. (d) Replace `citi` by `city`.

Why it works: all the information is in the standard places of the message; you need only know where to look.

</details>

### Exercise 2 — Fix the program step by step (medium)

Fix this program, one error at a time. Write down the number of errors in each round and the fix you made.

```rust
fn main() {
    let word = "Rust"
    println("I like {}", word);
}
```

<details>
<summary>Hint</summary>

The first message is about a missing `;`. After it, look at the name `println`.

</details>

<details>
<summary>Answer</summary>

Round 1 (2 errors): add `;` after `"Rust"`. Round 2 (1 error): `println` → `println!`. Round 3: no errors.

```rust
fn main() {
    let word = "Rust";
    println!("I like {}", word);
}
```

Output:

```text
I like Rust
```

Why it works: each error is independent and has a clear message. Because you run after every fix, you always know which change did what.

</details>

### Exercise 3 — Find the logic bug (medium)

This program should print the area of a rectangle 6 wide and 4 high (24). It prints 10. Find the bug without a
compiler, and then fix it.

```rust
fn main() {
    let width = 6;
    let height = 4;
    println!("Area: {}", width + height);
}
```

<details>
<summary>Hint</summary>

The program compiles, so the problem is in the logic. Check the formula for an area (Lesson 00.2 has this exact program).

</details>

<details>
<summary>Answer</summary>

`width + height` is 10: the sum, not the area. The area is a product:

```rust
fn main() {
    let width = 6;
    let height = 4;
    println!("Area: {}", width * height);
}
```

Output:

```text
Area: 24
```

Why it works: the computer did what the code said (add), not what was meant (multiply). Knowing the expected result (24) is what made the bug visible.

</details>

### Exercise 4 — Predict the exit code (medium)

```rust
fn main() {
    println!("one");
    panic!("stop");
    println!("two");
}
```

What does the program print? Does it print "two"? What is the exit code? (The compiler gives a warning
about this program; what do you think it says?)

<details>
<summary>Hint</summary>

A panic ends the program right away. Think about the line after it.

</details>

<details>
<summary>Answer</summary>

It prints `one`, then the panic message (`thread 'main' ... panicked at src/main.rs:3:5:` and `stop`). It never prints `two`. The exit code is `101`. The compiler warns `unreachable statement`: code after `panic!` can never run.

Why it works: `panic!` stops the program; nothing after it executes. The shell stores the exit code in `$?`, and Rust uses 101 for a panic.

</details>

### Exercise 5 — Write the question (hard)

You ran a program with a missing semicolon:

```rust
fn main() {
    println!("first")
    println!("second");
}
```

and it printed `error: expected `;`, found `println``. Write a good forum question about it.
(Include everything from section 9.)

<details>
<summary>Hint</summary>

Goal, code, exact error, expectation, what you tried, version.

</details>

<details>
<summary>Answer</summary>

```text
Title: "expected `;`, found `println`" in a two-line program

Goal: print two lines, "first" and "second".

Code:
    fn main() {
        println!("first")
        println!("second");
    }

Error:
    error: expected `;`, found `println`
     --> src/main.rs:2:22

Expected: two lines of output.
I tried: reading the error; I think line 2 needs a `;`, but I do not understand
why the error mentions `println` (which is on line 3).
rustc 1.100.0, macOS on Apple M4.
```

Why it works: someone can copy the code, reproduce the problem within a minute, and see exactly what you are asking. (The answer to the question: the compiler reads a statement until it finds `;`; it reaches `println` on line 3 before finding it, so it points to line 2's end.)

</details>

## Quiz

1. Which kind of problem is "the program runs and prints a wrong number"?

<details>
<summary>Answer</summary>

A logic bug. There is no message; you need to know the expected result and look at the steps.

</details>

2. In `--> src/main.rs:3:32`, what do the numbers mean?

<details>
<summary>Answer</summary>

Line 3, column (character) 32 of the file `src/main.rs`.

</details>

3. Why do you fix the first error first?

<details>
<summary>Answer</summary>

Later errors are often caused by earlier ones, and they disappear when the first is fixed.

</details>

4. How do you read the long explanation of the error `E0308` offline?

<details>
<summary>Answer</summary>

`rustc --explain E0308` in the terminal, or `rustup doc --error-codes` in the browser.

</details>

5. What exit code does a panic give, and how do you see it?

<details>
<summary>Answer</summary>

101. Run `echo $?` right after the program.

</details>

6. Name four things a good question to a forum or an AI assistant contains.

<details>
<summary>Answer</summary>

Any four of: the goal, a minimal reproducible example, the exact error text, what you expected, what you tried, `rustc --version` and the system.

</details>

7. What must you never paste into a public question?

<details>
<summary>Answer</summary>

Passwords, API keys, tokens, private keys and other private data (also check long paths that contain your user name).

</details>

## Summary

- Four kinds of trouble: compile errors, warnings, panics (run time) and logic bugs (no message).
- A compiler error has a severity, a code, a message, a location `file:line:column`, a marked snippet and often a `help:`.
- With many errors: read the first one, fix one thing, run again.
- The compiler's `help:` is a good guess, not an order. You know what you meant.
- `rustc --explain E....` and `rustup doc` work offline.
- Warnings are for fixing: keep a project at zero warnings.
- A panic ends the program with exit code 101; the backtrace (`RUST_BACKTRACE=1`) shows the path to it; look for the lines with your project name.
- For logic bugs: know the expected result, print the middle values, make the problem smaller.
- Ask well: goal, minimal code, exact error text, expectation, attempts, versions. Never paste secrets.

## Sources and further reading

- [Rust Compiler Error Index](https://doc.rust-lang.org/error_codes/index.html) — the explanation of every error code, with examples.
- [The Rust Programming Language — To `panic!` or Not to `panic!`](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html) — the official view on panics; you will use it in Phase 04.
- [Standard library — `panic!`](https://doc.rust-lang.org/std/macro.panic.html) — the reference of the macro and of how panics work.
- [Rust Users Forum](https://users.rust-lang.org/) — the official place for questions from learners.
- [Rust community](https://www.rust-lang.org/community) — the list of official chats, forums and events.
- [How to create a Minimal Reproducible Example (Stack Overflow)](https://stackoverflow.com/help/minimal-reproducible-example) — a short guide on making the smallest example that shows your problem.
- [docs.rs](https://docs.rs/) — documentation of every library published on crates.io.

---

[🏠 Start](../START-HERE.md) · [← 00.6 Your first Rust program](06-your-first-rust-program.md) · [📋 Phase 00](README.md) · [00.8 Study routine, notes in Markdown and checkpoint →](08-study-routine-notes-and-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
