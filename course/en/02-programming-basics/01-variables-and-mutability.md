<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 02 — Programming basics in Rust](README.md) · [📋 Phase 02](README.md) · [02.2 Data types →](02-data-types.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/01-variables-and-mutability.md) <!-- nav -->

# Lesson 02.1 — Variables and mutability

> **Phase 02 — Programming basics in Rust** · Lesson 1 of 11 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a variable is and how to create one with `let`.
- Why Rust variables cannot be changed by default, and how `mut` changes that.
- How constants (`const`) differ from variables.
- What shadowing is, and when to use it instead of `mut`.
- How to read your first compiler errors about variables.

## Before you start

- You finished [Lesson 00.6 — Your first Rust program](../00-getting-started/06-your-first-rust-program.md) and can run `cargo new` and `cargo run`.
- You finished [Lesson 00.7 — Reading errors and getting help](../00-getting-started/07-reading-errors-and-getting-help.md), so a compiler message does not scare you.
- You know the basic Git commands from [Lesson 01.3 — Your first repository and commits](../01-git-and-version-control/03-your-first-repository-and-commits.md): `git status`, `git add`, `git commit`.
- You have your learning repository (the `~/projects/rust` folder) from [Lesson 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md).

## New words

| Word | Meaning |
|---|---|
| **variable** | A name that points to a value stored in the computer's memory. |
| **value** | A piece of data, for example the number `10` or the text `"hello"`. |
| **binding** | The link between a name and a value. In Rust, `let` creates a binding. Most people simply say "variable". |
| **immutable** | Cannot be changed after it is set. Rust variables are immutable by default. |
| **mutable** | Can be changed. You ask for this with the keyword `mut`. |
| **keyword** | A word that has a special meaning in the language (`let`, `fn`, `mut`, `const`). You cannot use it as a name. |
| **constant** | A value that never changes and is known when the program is compiled. Created with `const`. |
| **shadowing** | Creating a new variable with the same name as an old one. The new one hides ("shadows") the old one. |
| **scope** | The part of the code where a name can be used. |
| **block** | Code between `{` and `}`. A block creates a new scope. |
| **type** | The kind of a value (a whole number, a decimal number, text…). Lesson 02.2 explains types in detail. |
| **compiler** | The program (`rustc`) that translates your Rust code to a program the computer can run, and checks it for mistakes. |
| **warning** | A message from the compiler about code that is legal but probably a mistake. The program still builds. |
| **panic** | A controlled crash: the program stops and prints a message. |
| **macro** | Something that looks like a function but ends with `!`, for example `println!` or `assert_eq!`. Macros are explained in Phase 06. |
| **assertion** | A check in the code that says "this must be true". If it is false, the program panics. |
| **RAM** (Random Access Memory) | The fast, temporary memory where a running program keeps its data. |
| **snake_case** | A naming style: all letters lowercase, words separated by `_` (`total_price`). |
| **`.gitignore`** | A file that tells Git which files to never track (for example build output). |

## 1. Your practice project

For the whole of Phase 02 you will try many small programs. Instead of creating a new project for each one, you will create **one practice project** and keep each experiment in its own file.

First, make sure Git does not track the build folder. When you run `cargo build`, Cargo creates a folder named `target/` with compiled files. It is large, it can be rebuilt at any time, and it must never go into Git. Cargo normally writes a `.gitignore` for you, but **only when it creates a new Git repository**. Your `~/projects/rust` folder is already a repository, so Cargo skips that step. You add the rule yourself.

```bash
cd ~/projects/rust
echo "target/" >> .gitignore
cargo new code/p02-basics
cd code/p02-basics
mkdir src/bin
```

What happens:

1. `cd ~/projects/rust` goes to the root of your learning repository.
2. `echo "target/" >> .gitignore` appends the line `target/` to the file `.gitignore` (and creates the file if it does not exist). The pattern `target/` matches a folder called `target` at any depth, so it covers every future project.
3. `cargo new code/p02-basics` creates a new Rust project. The name `p02-basics` means "phase 02, basics". The convention from the course: `code/pNN-<name>`.
4. `mkdir src/bin` creates a folder. **Every file `.rs` placed in `src/bin/` becomes a separate small program.**

Now create the file `src/bin/01_variables.rs` and type this:

```rust
fn main() {
    let mut score = 10;
    println!("score = {score}");
    score = 11;
    println!("score = {score}");
    score += 5;
    println!("score = {score}");
}
```

Run it. Because there are several programs, you must say which one. The name is the file name without `.rs`:

```bash
cargo run --bin 01_variables
```

```text
   Compiling p02-basics v0.1.0 (/Users/you/projects/rust/code/p02-basics)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.30s
     Running `target/debug/01_variables`
score = 10
score = 11
score = 16
```

(Your path in the first line will be different. The next lines of this lesson show only the program output, without the `Compiling` lines.)

> 💡 **Tip:** If you type just `cargo run` in a project with several programs, Cargo answers `could not determine which binary to run` and lists the names it knows. Add `--bin <name>`.

Finally, check what Git sees:

```bash
cd ~/projects/rust
git status --short
```

```text
?? .gitignore
?? code/
```

Only `.gitignore` and your new `code/` folder appear. `target/` is hidden, as it should be. You will commit at the end of the lesson.

## 2. What a variable is

Imagine a shelf with labelled boxes. Each box holds one thing, and the label tells you which box is which. A **variable** works the same way: the computer's memory (RAM, Random Access Memory) is the shelf, the value is the thing in the box, and the name is the label.

```text
   name        value in memory
 +-------+    +------+
 | score | -> |  10  |
 +-------+    +------+
```

You create a variable with the keyword `let`:

```rust
let score = 10;
```

Read it as: "let the name `score` mean the value `10`". The parts are:

```text
let score = 10;
│   │       │ └─ the end of the statement
│   │       └─── the value
│   └─────────── the name you choose
└─────────────── the keyword that creates a variable
```

To use the variable, write its name. Inside `println!` you can put the name in curly braces: `println!("score = {score}")` prints `score = 10`. (The `!` shows that `println!` is a macro. Lesson 02.6 explains printing fully.)

## 3. Variables do not change by default

In many languages a variable can be changed at any time. In Rust it cannot. This is deliberate. Try it:

```rust
fn main() {
    let score = 10;
    println!("score = {score}");
    score = 11;
    println!("score = {score}");
}
```

The compiler refuses to build the program:

```text
error[E0384]: cannot assign twice to immutable variable `score`
 --> src/main.rs:4:5
  |
2 |     let score = 10;
  |         ----- first assignment to `score`
3 |     println!("score = {score}");
4 |     score = 11;
  |     ^^^^^^^^^^ cannot assign twice to immutable variable
  |
help: consider making this binding mutable
  |
2 |     let mut score = 10;
  |         +++

For more information about this error, try `rustc --explain E0384`.
```

How to read this message:

1. `error[E0384]` — the error has the code `E0384`. Every Rust error has such a code, and you can search it.
2. `cannot assign twice to immutable variable` — the problem in plain words. "Assign" means "put a value into a variable". "Immutable" means "cannot be changed".
3. The arrows `-->` give the file, the line and the column: line 4, column 5. The file name is the file that you compiled. The outputs in this course show `src/main.rs`; if you put the example into a file in `src/bin/`, you will see that file's name instead, and everything else stays the same.
4. The marks `-----` and `^^^^` point to the two places involved: the first assignment, and the second one that is not allowed.
5. `help:` — the compiler suggests the fix: write `let mut score`.

The last line tells you a useful trick. Run `rustc --explain E0384` in the terminal to read a longer explanation of any error code, offline, with examples.

**Why does Rust do this?** When a value never changes, you can read the code and trust it: the number you see at the top is the number at the bottom. Many bugs come from "something changed this variable and I did not notice". Later, when several threads run at the same time, immutable values are safe to share. Rust makes you say clearly when a change is intended.

> 📏 **Convention:** Start with plain `let`. Add `mut` only when you really need to change the value.

## 4. `mut` — variables that can change

Put the keyword `mut` (short for *mutable*) after `let`:

```rust
fn main() {
    let mut score = 10;
    println!("score = {score}");
    score = 11;
    println!("score = {score}");
    score += 5;
    println!("score = {score}");
}
```

What happens:

1. `let mut score = 10;` creates a variable that is allowed to change. It starts as `10`.
2. The first `println!` prints `score = 10`.
3. `score = 11;` puts a new value in the same box. Note that there is no `let` this time: we are not creating a variable, only assigning to the existing one.
4. `score += 5;` means "add 5 to `score`" — it is a short form of `score = score + 5;`. Lesson 02.3 shows all such operators.
5. The last `println!` prints the new value.

```text
score = 10
score = 11
score = 16
```

`mut` lets you change the **value**. It does not let you change the **type** of the variable. A variable that holds text cannot suddenly hold a number:

```rust
fn main() {
    let mut spaces = "   ";
    spaces = spaces.len();
    println!("There are {spaces} spaces");
}
```

(`spaces.len()` gives the length of the text: the number of bytes. `len` is a *method* — a function attached to a value with a dot. Methods are taught in Phase 04.)

```text
error[E0308]: mismatched types
 --> src/main.rs:3:14
  |
2 |     let mut spaces = "   ";
  |                      ----- expected due to this value
3 |     spaces = spaces.len();
  |              ^^^^^^^^^^^^ expected `&str`, found `usize`
```

`E0308 mismatched types` is one of the most common errors you will see. It says: "I expected a text (`&str`), but you gave me a number (`usize`)". The next lesson explains these type names. The next section shows how to solve this exact problem.

## 5. Constants

A **constant** is a value that is fixed for the whole program. You create it with `const`:

```rust
const SECONDS_PER_HOUR: u32 = 60 * 60;
const MAX_PLAYERS: u32 = 4;

fn main() {
    let hours = 3;
    let seconds = hours * SECONDS_PER_HOUR;
    println!("{hours} hours = {seconds} seconds");
    println!("Max players: {MAX_PLAYERS}");
}
```

What happens:

1. `const SECONDS_PER_HOUR: u32 = 60 * 60;` defines a constant. After the name comes `: u32` — the **type** (a 32-bit whole number without a sign; see Lesson 02.2). For constants, you must always write the type.
2. The value `60 * 60` is calculated by the compiler **while compiling**, not when the program runs.
3. The constants are written outside `main`, at the top of the file. Constants can live anywhere, even outside every function, and every function can use them.
4. `hours * SECONDS_PER_HOUR` multiplies `3` by `3600`.

```text
3 hours = 10800 seconds
Max players: 4
```

Differences between `let` and `const`:

| | `let` | `const` |
|---|---|---|
| Can be changed? | Only with `mut` | Never (`const mut` is an error) |
| Type written by you? | Usually not needed | Always required |
| Value known when | The program runs | The program is compiled |
| Where it can be used | Only inside functions | Anywhere, including outside functions |
| Naming | `snake_case` | `SCREAMING_SNAKE_CASE` (all capitals) |

If you try to give a constant a value that is only known at run time, the compiler stops you:

```rust
fn main() {
    let base = 3;
    const DOUBLE: u32 = base * 2;
    println!("{DOUBLE}");
}
```

```text
error[E0435]: attempt to use a non-constant value in a constant
 --> src/main.rs:3:25
  |
3 |     const DOUBLE: u32 = base * 2;
  |                         ^^^^ non-constant value
  |
help: consider using `let` instead of `const`
```

> 📏 **Convention:** Do not put "magic numbers" in your code. If you write `3600` in five places, nobody knows what it means. A named constant `SECONDS_PER_HOUR` explains itself and can be changed in one place. This is part of the **DRY** principle (Don't Repeat Yourself).

## 6. Shadowing

Rust lets you declare a new variable with the **same name** as an old one. The new one *shadows* the old one: from that line on, the name means the new variable.

```rust
fn main() {
    let x = 5;
    let x = x + 1;
    {
        let x = x * 2;
        println!("inner x = {x}");
    }
    println!("outer x = {x}");
}
```

What happens:

1. `let x = 5;` creates `x` with the value `5`.
2. `let x = x + 1;` creates a **new** variable also named `x`. Its value is calculated from the old `x` (`5 + 1`), so it is `6`. The old `x` is now hidden.
3. The curly braces `{ ... }` start a **block**. A block is a new scope.
4. Inside the block, `let x = x * 2;` creates a third `x` with value `12`. It exists only inside the block.
5. The first `println!` is inside the block and prints `12`.
6. At the closing `}`, the inner `x` disappears. The name `x` again means the one with the value `6`.
7. The second `println!` prints `6`.

```text
inner x = 12
outer x = 6
```

```text
 before the block      inside the block        after the block
 +---+                 +---+                   +---+
 | 6 |  x              |12 |  x (inner)        | 6 |  x
 +---+                 +---+  [6 is hidden]    +---+
```

**Shadowing is not the same as `mut`.** Two differences:

- With shadowing you write `let` again. It creates a new variable, so the compiler still protects you from accidental changes later: after your last `let x`, the variable is immutable again.
- A new variable may have a **different type**. This solves the problem from section 4:

```rust
fn main() {
    let spaces = "   ";
    let spaces = spaces.len();
    println!("There are {spaces} spaces");
}
```

```text
There are 3 spaces
```

The first `spaces` is text. The second `spaces` is a number computed from the text. Both use the same good name, and we do not need `spaces_text` and `spaces_count`.

When to choose which:

- Use **`mut`** when the same thing changes many times, for example a counter in a loop.
- Use **shadowing** to *transform* a value step by step (text → number) while keeping the name.

## 7. Scope: where a name lives

A name exists from the line where you create it until the end of the block `{ ... }` that contains it. Outside the block, the name does not exist.

```rust
fn main() {
    {
        let inner = 7;
        println!("inside: {inner}");
    }
    println!("outside: {inner}");
}
```

```text
error[E0425]: cannot find value `inner` in this scope
 --> src/main.rs:6:25
  |
6 |     println!("outside: {inner}");
  |                         ^^^^^
  |
help: the binding `inner` is available in a different scope in the same function
 --> src/main.rs:3:13
  |
3 |         let inner = 7;
  |             ^^^^^
```

`E0425 cannot find value` says that the name is not known here. The `help` part is clever: it found a variable with this name in another scope. Whenever you see this error, check whether you declared the variable inside a block that already ended.

Scope has a bigger meaning in Rust: when a block ends, the values created inside it are cleaned up. Phase 03 builds on this idea.

## 8. Declare now, assign later

You may create a variable without a value, if you give the type, and assign it later. It can be assigned once even if it is not `mut`:

```rust
fn main() {
    let total: i32;
    total = 30;
    println!("total = {total}");
}
```

```text
total = 30
```

The compiler checks that the variable always has a value before you read it. If you forget, you get an error:

```rust
fn main() {
    let total: i32;
    println!("total = {total}");
}
```

```text
error[E0381]: used binding `total` isn't initialized
 --> src/main.rs:3:24
  |
2 |     let total: i32;
  |         ----- binding declared here but left uninitialized
3 |     println!("total = {total}");
  |                        ^^^^^ `total` used here but it isn't initialized
```

Rust has no "empty" or random-garbage variables. This is one more mistake that cannot reach your users. In practice, you will mostly write `let name = value;` right away.

## 9. Good names

Names are the main way to make code readable. The rules of the compiler:

- A name contains letters, digits and `_`, and does not start with a digit.
- A name cannot be a keyword. `let fn = 5;` fails with `expected identifier, found keyword fn`.

The conventions of the Rust community (not forced by the compiler, but everybody follows them):

- Variables and functions: `snake_case`: `total_price`, `user_count`.
- Constants: `SCREAMING_SNAKE_CASE`: `MAX_PLAYERS`.
- Choose names that say *what the value means*: `seconds_left` is better than `s` or `data2`.
- A name that starts with `_` (for example `_ignored`) tells the compiler "I know I am not using this".

The compiler warns you about variables you never use:

```rust
fn main() {
    let count = 3;
    let _ignored = 4;
    println!("hello");
}
```

```text
warning: unused variable: `count`
 --> src/main.rs:2:9
  |
2 |     let count = 3;
  |         ^^^^^ help: if this is intentional, prefix it with an underscore: `_count`
  |
  = note: `#[warn(unused_variables)]` (part of `#[warn(unused)]`) on by default

hello
```

The program runs (it is only a warning), but `count` is probably a mistake. There is no warning for `_ignored`. **Treat warnings as work to do**: a project with no warnings is easier to trust.

> 💡 **Tip:** Three commands that keep your code healthy. Use them often, and always before a commit:
>
> - `cargo check` — checks that the code compiles, without making the program. It is faster than `cargo build`.
> - `cargo fmt` — formats your code in the standard style (the tool is called *rustfmt*).
> - `cargo clippy` — gives extra advice about style and likely mistakes (the tool is called *Clippy*).
>
> RustRover also shows the inferred type of a variable next to it, in grey ("inlay hints"). That is a great way to learn types.

## Common mistakes

**1. Forgetting `mut`.** You see `E0384 cannot assign twice to immutable variable`. Fix: write `let mut`. But first ask: do I really need to change the value? Sometimes a new `let` (shadowing) is cleaner.

**2. Using `let` when you meant to assign.** Writing `let score = 11;` inside a block creates a *new* variable that hides the old one, and the old one is unchanged after the block. If you want to change the old variable, write `score = 11;` without `let`.

**3. Trying to change the type with `mut`.** You see `E0308 mismatched types`. Use shadowing: `let spaces = spaces.len();`.

**4. A `const` without a type, or with a runtime value.** `const MAX = 5;` is an error (the type is required). `const X: u32 = base * 2;` where `base` is a `let` variable gives `E0435`.

**5. Using a variable outside its block.** You see `E0425 cannot find value`. Move the `let` out of the block, or move the code that uses it into the block.

> 🧪 **Testing:** You can check your own code with an **assertion**. `assert_eq!(a, b)` compares two values. If they are equal, nothing happens. If they differ, the program panics (stops) and shows both values:
>
> ```rust
> fn main() {
>     let a = 2;
>     let b = 3;
>     let sum = a + b;
>     assert_eq!(sum, 5);
>     println!("sum is correct: {sum}");
>     assert_eq!(sum, 6, "the sum of {a} and {b} should be 6");
>     println!("this line is never reached");
> }
> ```
>
> ```text
> sum is correct: 5
>
> thread 'main' (3860046) panicked at src/main.rs:7:5:
> assertion `left == right` failed: the sum of 2 and 3 should be 6
>   left: 5
>  right: 6
> note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
> ```
>
> The first assertion passes silently. The second fails: the program stops at line 7, shows the message you wrote, then `left` and `right`. (The number in parentheses after `thread 'main'` is different on every run.) Lesson 02.8 teaches the real Rust test system (`cargo test`). Until then, `assert_eq!` is your small safety net.

> 🔒 **Security:** Immutability helps security too. A value that cannot be changed cannot be changed by mistake, or by a bug in another part of the program. When you design data, make things read-only unless they must change.

## Try it yourself

Work in `~/projects/rust/code/p02-basics`.

1. Create `src/bin/01_variables.rs` from section 1 and run it with `cargo run --bin 01_variables`.
2. Remove the word `mut` from line 1 of `main`. Run again. Find in the error message: the error code, the line number, and the `help` text. Run `rustc --explain E0384` and read the first paragraph. Put `mut` back.
3. Add a constant `const START_SCORE: i32 = 10;` above `main`, and use it: `let mut score = START_SCORE;`. Run again.
4. Add a line `let score = score * 2;` after the last `println!`, and print `score` once more. What does it print? What would happen if you had written `let` in front of the earlier `score = 11;`?
5. Run `cargo fmt`, `cargo clippy` and `cargo check`. All three should finish without errors and warnings.
6. Save your work with Git:

```bash
cd ~/projects/rust
git add .gitignore code/p02-basics
git commit -m "feat(p02): add practice project and variables example"
git log --oneline -n 3
```

What happens: `git add` lists the paths to save (we name them, so nothing else gets in by accident). `git commit -m` makes a snapshot with a message. `git log --oneline -n 3` shows the last three snapshots. If you do not remember how, review [Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md).

> 📏 **Convention:** The message `feat(p02): …` follows the *Conventional Commits* style that [Lesson 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) described: a type (`feat`, `fix`, `docs`, `test`), a scope in parentheses, a colon, then a short summary. Use it for every commit in this phase.

## Exercises

### Exercise 1 — Fix the program (easy)

This program does not compile. Fix it with the smallest change. It must print `lives: 2`.

```rust
fn main() {
    let lives = 3;
    lives = lives - 1;
    println!("lives: {lives}");
}
```

<details>
<summary>Hint</summary>

The variable is changed after it is created. Which keyword allows that?

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let mut lives = 3;
    lives = lives - 1;
    println!("lives: {lives}");
}
```

Why it works: `mut` allows the second assignment. Output: `lives: 2`. (You could also write `lives -= 1;`.)

</details>

### Exercise 2 — Predict, then run (easy)

What does this program print? Write down your answer first, then run it.

```rust
fn main() {
    let a = 1;
    let a = a + 1;
    {
        let a = a * 10;
        println!("{a}");
    }
    println!("{a}");
}
```

<details>
<summary>Hint</summary>

Each `let a` makes a new variable. The one inside the braces lives only until the `}`.

</details>

<details>
<summary>Answer</summary>

```text
20
2
```

Why it works: the first `a` is `1`; the second is `1 + 1 = 2`. Inside the block the third `a` is `2 * 10 = 20` and is printed. After the block, the name `a` means the second variable (`2`).

</details>

### Exercise 3 — A named constant (medium)

Write a program that calculates how many minutes there are in 7 days. Use a constant `MINUTES_PER_DAY` (60 minutes × 24 hours) and a variable `days`. Print `7 days = 10080 minutes`.

<details>
<summary>Hint</summary>

A constant needs a type: `const MINUTES_PER_DAY: u32 = 60 * 24;`. Use the same type for `days` by letting the compiler infer it from the multiplication.

</details>

<details>
<summary>Answer</summary>

```rust
const MINUTES_PER_DAY: u32 = 60 * 24;

fn main() {
    let days = 7;
    let minutes = days * MINUTES_PER_DAY;
    println!("{days} days = {minutes} minutes");
}
```

Why it works: the compiler sees `days * MINUTES_PER_DAY` and decides that `days` is also a `u32`. The output is `7 days = 10080 minutes`.

</details>

### Exercise 4 — Change the type with shadowing (medium)

Start with `let input = "42";` (text). Using shadowing, make a second `input` that is the **length** of the text (`input.len()`), then a third `input` that is the second one multiplied by 10. Print the final value. It should be `20`.

<details>
<summary>Hint</summary>

`"42".len()` is `2`, because the text has two characters. Write three `let input = …;` lines in a row.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let input = "42";
    let input = input.len();
    let input = input * 10;
    println!("{input}");
}
```

Why it works: each `let` creates a new variable, so the type is allowed to change from text to number. Output: `20`.

</details>

### Exercise 5 — Write a check (medium)

Write a program with variables `width = 6` and `height = 4`. Compute `area`. Use `assert_eq!` to check that `area` is `24` and that a `perimeter` (the sum of all four sides) is `20`. If both checks pass, print `all checks passed`.

<details>
<summary>Hint</summary>

`area = width * height`, `perimeter = 2 * (width + height)`. Put the `assert_eq!` lines before the final `println!`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let width = 6;
    let height = 4;
    let area = width * height;
    let perimeter = 2 * (width + height);
    assert_eq!(area, 24);
    assert_eq!(perimeter, 20);
    println!("all checks passed");
}
```

Why it works: both assertions are true, so the program continues to the `println!`. Change a number and run again to see the panic message. Output: `all checks passed`.

</details>

## Quiz

1. What does `let x = 5; x = 6;` do, and why?

<details>
<summary>Answer</summary>

It does not compile: error `E0384 cannot assign twice to immutable variable`. Variables are immutable by default. You need `let mut x = 5;`.

</details>

2. What is the difference between `let mut x = 1; x = 2;` and `let x = 1; let x = 2;`?

<details>
<summary>Answer</summary>

The first changes the value in one variable. The second creates two different variables; the second shadows (hides) the first. With shadowing, the new variable can even have another type, and it is again immutable.

</details>

3. Name three differences between `const` and `let`.

<details>
<summary>Answer</summary>

`const` can never be `mut`; it always needs a type written by you; its value must be known at compile time; it can be declared outside functions; its name is written in `SCREAMING_SNAKE_CASE`. (Any three.)

</details>

4. Where does a variable stop existing?

<details>
<summary>Answer</summary>

At the closing `}` of the block in which it was declared. After that, using the name gives `E0425 cannot find value`.

</details>

5. Which command shows a long explanation of the error code `E0384` without internet?

<details>
<summary>Answer</summary>

`rustc --explain E0384`.

</details>

6. Why did the compiler warn about `let count = 3;` but not about `let _ignored = 4;`?

<details>
<summary>Answer</summary>

A variable that is never used is probably a mistake, so the compiler warns. A name starting with `_` tells the compiler that you know it is unused on purpose.

</details>

7. Why do we add `target/` to `.gitignore`?

<details>
<summary>Answer</summary>

`target/` holds build output. It is big, it can be recreated with `cargo build`, and it changes on every build, so it would only add noise and size to the Git history.

</details>

## Summary

- A variable is a name for a value in memory. You create it with `let name = value;`.
- Variables are immutable by default. Use `let mut` only when the value must change.
- `mut` changes the value but never the type.
- `const` defines a fixed value with a required type; the name is `SCREAMING_SNAKE_CASE`.
- Shadowing (`let x = …` again) makes a new variable, even of another type.
- A name lives until the end of its block `{ }`.
- Use `snake_case`, give names meaning, and treat warnings seriously.
- `cargo check`, `cargo fmt` and `cargo clippy` are your daily helpers. Commit with Git often.

## Sources and further reading

- [The Rust Programming Language — 3.1 Variables and Mutability](https://doc.rust-lang.org/book/ch03-01-variables-and-mutability.html) — the official book chapter on this topic.
- [Rust by Example — Variable bindings](https://doc.rust-lang.org/rust-by-example/variable_bindings.html) — short examples you can run and edit.
- [The Rust Reference — Constant items](https://doc.rust-lang.org/reference/items/constant-items.html) — the exact rules for `const`.
- [Rust API Guidelines — Naming](https://rust-lang.github.io/api-guidelines/naming.html) — the official naming conventions.
- [Error index — E0384](https://doc.rust-lang.org/error_codes/E0384.html) — the long explanation of the error from section 3.
- [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) — the commit message style used in this course.

---

[🏠 Start](../START-HERE.md) · [← Phase 02 — Programming basics in Rust](README.md) · [📋 Phase 02](README.md) · [02.2 Data types →](02-data-types.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
