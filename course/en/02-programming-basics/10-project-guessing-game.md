<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.9 Debugging basics](09-debugging-basics.md) · [📋 Phase 02](README.md) · [02.11 Checkpoint →](11-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/10-project-guessing-game.md) <!-- nav -->

# Lesson 02.10 — Project: the guessing game

> **Phase 02 — Programming basics in Rust** · Lesson 10 of 11 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- How to plan a small program and build it step by step, in milestones.
- How to use an external crate (`rand`) with Cargo.
- How to use `Ordering` and `match` to compare numbers.
- How to keep a program testable by separating logic from input and output.
- How to work with Git like a professional: a branch for the feature, a commit for every milestone, a merge and a tag at the end.

## Before you start

- You finished [Lesson 02.9 — Debugging basics](09-debugging-basics.md) and all the lessons before it: variables, types, operators, functions, control flow, formatting, input, tests.
- Git works and you have your learning repository in `~/projects/rust` ([Lesson 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md)). You know branches and merging ([Lesson 01.6](../01-git-and-version-control/06-branches-and-merging.md)).
- You have an internet connection for the first build (Cargo downloads the `rand` crate). After that, it works offline.

## New words

| Word | Meaning |
|---|---|
| **crate** | A unit of Rust code that is compiled together: a program or a library. Other people's libraries are also called crates. |
| **dependency** | A crate that your project uses. |
| **crates.io** | The public website and registry where Rust crates are published. |
| **`Cargo.toml`** | The file that describes your project: name, version, edition and dependencies. |
| **`Cargo.lock`** | The file where Cargo records the **exact** versions of all crates used, so that every build is the same. |
| **semantic versioning** (SemVer) | The version numbers `MAJOR.MINOR.PATCH`. A change in `MAJOR` may break your code; `MINOR` and `PATCH` should not. |
| **random number** | A number that is not predictable. Computers make *pseudo-random* numbers with an algorithm. |
| **enum** (enumeration) | A type with a fixed list of possible values, called variants. `Ordering` is an enum. Phase 04, Lesson 3 teaches enums. |
| **`Ordering`** | The standard enum with three variants: `Less`, `Equal`, `Greater`. It is the result of comparing two values. |
| **environment variable** | A named value that the operating system gives to a program. You can set it in the terminal before the command. |
| **feature branch** | A Git branch made for one piece of work, merged back into `main` when it is done. |
| **tag** | A name attached to a commit in Git, usually for a release. |
| **README** | A text file that explains what a project is and how to use it. |
| **acceptance test** | A check that the finished program behaves as the contract says. Here: a command and its expected output. |
| **milestone** | A step of the plan with a clear goal and a clear way to check that it is done. |
| **contract** | The exact description of what the program must do: input, output and rules. |

## 1. What you will build

A number guessing game for the terminal. The computer thinks of a secret number from 1 to 100. The player has 7 attempts. After every guess the game says *Too small!* or *Too big!*, until the player wins, loses, or gives up.

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
50
Too big!
Attempt 2 of 7 - type your guess:
25
Too small!
...
```

You will use everything from this phase: variables and constants, `u32`, `if` and `match`, a loop, functions, `Result`, reading and parsing input, formatting, tests, and the debugging habits. You will also meet three new things: an external crate, `Ordering`, and an environment variable.

**There is no complete solution in this lesson.** You write the program yourself, with a contract, a plan, hints and checks. If you are stuck, read the hint of the milestone, then the lesson that teaches the idea.

## 2. New tools you need

### 2.1 Using a crate: `rand`

The Rust standard library has no random numbers. They come from the community crate **`rand`**. A project lists the crates it needs in `Cargo.toml`, under `[dependencies]`. The easy way to add one is a command:

```bash
cargo add rand@0.10
```

This edits `Cargo.toml` for you:

```toml
[dependencies]
rand = "0.10"
```

What happens:

1. `rand = "0.10"` means "any version compatible with 0.10". By the rules of **semantic versioning**, Cargo may use `0.10.4` or `0.10.5`, but not `0.11`. (For versions below 1.0 the middle number is treated like a major version.)
2. At the next `cargo build`, Cargo downloads `rand` and its own dependencies from **crates.io**, compiles them, and writes the exact versions into **`Cargo.lock`**. Commit `Cargo.lock` for programs: it makes your build repeatable.
3. In Rust 1.100, if you add a crate and do not use it yet, Cargo warns: `unused dependency 'rand'`. The warning disappears when you use the crate.

Cargo's own downloads are cached in `~/.cargo`, so later builds work offline. Phase 05, Lesson 8 explains versions and dependencies in depth.

To make a random number in the range 1 to 100 (inclusive):

```rust
use rand::RngExt;

fn main() {
    let n = rand::rng().random_range(1..=100);
    println!("{n}");
}
```

- `rand::rng()` gives the random number generator of the current thread.
- `.random_range(1..=100)` takes a number from the range, including both ends (Lesson 02.5).
- `use rand::RngExt;` imports the **trait** that provides `random_range`. Without it you get this error:

```text
error[E0599]: no method named `random_range` found for struct `ThreadRng` in the current scope
 --> src/main.rs:2:25
  |
2 |     let n = rand::rng().random_range(1..=100);
  |                         ^^^^^^^^^^^^
  |
  = help: items from traits can only be used if the trait is in scope
```

(The same rule as `Write` in Lesson 02.7: to call a method that belongs to a trait, import the trait.) Notice the line `items from traits can only be used if the trait is in scope`, which names the fix.

> 🔒 **Security:** `rand::rng()` is good for games and simulations, but its numbers are not meant for passwords or keys. Phase 10 explains cryptographically secure randomness.

### 2.2 Comparing with `Ordering`

You can compare two numbers with `<`, `>` and `==`. The standard library also gives you a method `cmp` that returns the result as a value of the type `Ordering`. `Ordering` is an **enum**: a type that can be only one of a fixed list of values:

```text
Ordering::Less      the first value is smaller
Ordering::Equal     they are equal
Ordering::Greater   the first value is bigger
```

```rust
use std::cmp::Ordering;

fn main() {
    let secret = 42;
    for guess in [10, 42, 99] {
        match guess.cmp(&secret) {
            Ordering::Less => println!("{guess}: Too small!"),
            Ordering::Greater => println!("{guess}: Too big!"),
            Ordering::Equal => println!("{guess}: You win!"),
        }
    }
}
```

```text
10: Too small!
42: You win!
99: Too big!
```

What happens:

1. `use std::cmp::Ordering;` imports the enum.
2. `guess.cmp(&secret)` compares `guess` with `secret`. The `&` before `secret` lends the value (you know this from `&mut input` in Lesson 02.7; Phase 03 explains it).
3. `match` has an arm for each of the three variants, written `Ordering::Less` and so on. Because `Ordering` has exactly three values, the arms cover everything and no `_` arm is needed. If you forget one, the compiler tells you which:

```text
error[E0004]: non-exhaustive patterns: `std::cmp::Ordering::Equal` not covered
 --> src/main.rs:5:11
  |
5 |     match guess.cmp(&secret) {
  |           ^^^^^^^^^^^^^^^^^^ pattern `std::cmp::Ordering::Equal` not covered
```

A function that returns an `Ordering` is easy to test: `assert_eq!(compare_guess(10, 50), Ordering::Less)` works, because `Ordering` can be compared and printed.

### 2.3 Environment variables

An **environment variable** is a named text value that is passed to a program when it starts. In the terminal you write it before the command:

```bash
GUESS_SECRET=42 cargo run -q
```

A Rust program reads it with `std::env::var("NAME")`. The result is a `Result`: `Ok(text)` if the variable exists, `Err(_)` if it does not:

```rust
fn main() {
    match std::env::var("GUESS_SECRET") {
        Ok(text) => println!("GUESS_SECRET = {text:?}"),
        Err(e) => println!("GUESS_SECRET is not set: {e}"),
    }
}
```

```text
GUESS_SECRET is not set: environment variable not found
```

and, when run as `GUESS_SECRET=42 cargo run -q`:

```text
GUESS_SECRET = "42"
```

Why do we need it? A game with a random secret cannot be checked with a fixed expected output. If `GUESS_SECRET` is set, the game uses it as the secret number, and now you can write **acceptance tests**: a command with a known input and a known output.

> 🔒 **Security:** A hook like `GUESS_SECRET` is fine for a learning game, but a real program must never let an environment variable (or any input) switch off a protection or change a secret. In Phase 10 you will learn how to treat configuration and secrets.

### 2.4 The Git workflow for this project

You will work like a team does:

1. Create a **feature branch** for the project: `git switch -c feature/guessing-game`.
2. Make a **commit at the end of every milestone**, with a [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) message (`feat(p02): …`, `test(p02): …`, `docs(p02): …`).
3. When everything is done and checked, go back to `main`, **merge** the branch with `git merge --no-ff` (it always creates a merge commit, so the history shows the feature as one unit), and add a **tag**.

At the end, `git log --oneline --graph --decorate` should look like this (your hashes and number of commits will differ):

```text
*   ddc7cc5 (HEAD -> main, tag: p02-guessing-game-v0.1) feat(p02): merge the guessing game
|\  
| * e74cdf2 (feature/guessing-game) feat(p02): add parse_guess with tests
| * 72e71a2 feat(p02): add rand dependency
| * 2104478 feat(p02): create guessing game project
|/  
* c669cf5 chore: add .gitignore
```

## 3. The project

### What you will build

A command-line program `p02-guessing-game` in `~/projects/rust/code/p02-guessing-game`:

- the computer picks a secret number from 1 to 100;
- the player has 7 attempts;
- after each valid guess the program prints whether it was too small or too big;
- the player wins by guessing the number, loses after 7 wrong valid guesses, or can type `quit`;
- wrong input (text, a number outside the range) never crashes the program and does not use up an attempt.

### Contract

**Start.** The program prints one line:

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
```

**The secret.** If the environment variable `GUESS_SECRET` exists and contains a whole number from 1 to 100, that number is the secret. Otherwise the secret is random, from 1 to 100 inclusive.

**Every round.** The program prints `Attempt N of 7 - type your guess:` where `N` is the number of the next attempt (1, 2, …), then reads one line from standard input. Then, in this order:

| The line (after trimming spaces) | The program does |
|---|---|
| `quit`, in any letters (`Quit`, `QUIT`) | prints `You gave up. The number was S.` and ends (`S` is the secret) |
| the input has ended (EOF, End Of File: there is nothing more to read) | prints `Input ended. The number was S.` and ends |
| is not a whole number from 1 to 100 | prints a message **to standard error** (see below) and starts the round again **without using up an attempt**. The text of the attempt counter stays the same. |
| a valid guess, smaller than the secret | counts one attempt; prints `Too small!` |
| a valid guess, bigger than the secret | counts one attempt; prints `Too big!` |
| a valid guess, equal to the secret | counts one attempt; prints `You win! You guessed it in N attempts.` (or `1 attempt.` if N is 1) and ends |

**Invalid input messages** go to standard error (`eprintln!`), in one of these forms:

```text
'abc' is not a whole number. Please type a whole number from 1 to 100.
0 is outside the range 1..=100. Please type a whole number from 1 to 100.
```

The first is for text that is not a number (including an empty line and negative numbers or decimals); the second is for numbers that are whole and non-negative but outside 1–100.

**Losing.** After 7 valid guesses without a win, the program prints `Game over! The number was S.` and ends.

**Exit code** is always `0`. **Output** goes to standard output, except the invalid input messages.

**Required structure.** The logic must be in functions that can be tested, with exactly these names and signatures:

```rust
fn parse_guess(text: &str) -> Result<u32, String>;          // trims, parses, checks 1..=100
fn compare_guess(guess: u32, secret: u32) -> Ordering;
fn win_message(attempts: u32) -> String;                    // "You win! ..." with singular/plural
fn pick_secret() -> u32;                                    // GUESS_SECRET or random
```

**Quality gates.** The project must have at least 6 unit tests in `src/main.rs` (`cargo test` passes), no warnings from `cargo clippy`, code formatted with `cargo fmt`, a short `README.md` in the project, and a Git history with one commit per milestone.

### Step-by-step plan

#### M1 — Project, branch and dependency

**Goal.** A new project with the `rand` crate, on a feature branch.

**Guidance.**

1. In your learning repository, create the branch `feature/guessing-game` (from `main`).
2. Run `cargo new code/p02-guessing-game` and then `cargo add rand@0.10` inside the project.
3. In `src/main.rs`, print a random number from 1 to 100 with the line `The secret number is N` (a temporary line for checking).
4. Commit.

<details>
<summary>Hint</summary>

Commands: `cd ~/projects/rust`, `git switch -c feature/guessing-game`, `cargo new code/p02-guessing-game`, `cd code/p02-guessing-game`, `cargo add rand@0.10`. Remember `use rand::RngExt;`. Check that the root `.gitignore` already has `target/` (Lesson 02.1) before you `git add`.

</details>

**Done when:**

```bash
cd ~/projects/rust/code/p02-guessing-game
cargo run -q
cargo run -q
git branch --show-current
git log --oneline -n 1
```

prints two lines of the form `The secret number is N` (usually with different numbers, each from 1 to 100), then `feature/guessing-game`, then a commit such as `feat(p02): create guessing game project`.

#### M2 — The testable functions, test first

**Goal.** The three pure functions, written test-first (TDD, Test-Driven Development, Lesson 02.8).

**Guidance.** Write the functions `parse_guess`, `compare_guess` and `win_message` from the contract. For each one, first write the tests in `#[cfg(test)] mod tests`, see them fail (use `todo!()`), then write the body. Put the limits in constants: `MIN_NUMBER`, `MAX_NUMBER`, `MAX_ATTEMPTS`.

Tests to write (at least):

- `parse_guess`: `"1"`, `"100"` and `"  42\n"` are accepted; `"0"` and `"101"` are rejected; `"abc"`, `""`, `"-5"` and `"3.5"` are rejected.
- `compare_guess`: less, greater and equal.
- `win_message`: `1` gives `You win! You guessed it in 1 attempt.`; `5` gives `You win! You guessed it in 5 attempts.`

<details>
<summary>Hint</summary>

`parse_guess`: `match text.trim().parse::<u32>()`. The type `u32` already rejects `-5` and `3.5`. For the range, use `(MIN_NUMBER..=MAX_NUMBER).contains(&n)` (the `contains` method of a range) or two comparisons. `Err` carries a `String` made with `format!`: `'abc' is not a whole number` or `0 is outside the range 1..=100` (the first part of the messages in the contract; `main` adds `. Please type a whole number from 1 to 100.` when it prints them). `compare_guess` is one line: `guess.cmp(&secret)`. `win_message`: use `if attempts == 1 { … } else { … }` and return a `String` (`String::from("…")` or `format!`).

</details>

**Done when:**

```bash
cargo test
```

prints a result like:

```text
running 5 tests
test tests::compare_guess_orders_correctly ... ok
test tests::parse_guess_accepts_numbers_in_range ... ok
test tests::parse_guess_rejects_out_of_range ... ok
test tests::parse_guess_rejects_text_and_signs ... ok
test tests::win_message_uses_singular_and_plural ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(You may have more tests; the names and the order will be your own. The compiler may warn that the functions are "never used" until M4. That is fine for now.) Commit: `feat(p02): add guess parsing and comparing with tests`.

#### M3 — Picking the secret

**Goal.** `pick_secret()` that supports the `GUESS_SECRET` hook.

**Guidance.** Write a small helper `random_secret() -> u32` and `pick_secret() -> u32`. `pick_secret` reads `GUESS_SECRET` with `std::env::var`; if it is a valid secret (use `parse_guess`!), return it; in every other case (not set, not a number, out of range) return a random number. Add a test that 1000 random secrets are always in range. In `main`, print `The secret number is N` again to check.

<details>
<summary>Hint</summary>

Two nested `match` expressions do the job: the outer one on `std::env::var("GUESS_SECRET")` with arms `Ok(text)` and `Err(_)`, the inner one on `parse_guess(&text)` with arms `Ok(n) => n` and `Err(_) => random_secret()`. The `Err(_)` of the outer match also gives `random_secret()`. Note that `pick_secret` uses the environment, so do not write a test that sets variables (tests run in parallel; Phase 09 shows safe ways).

</details>

**Done when:**

```bash
GUESS_SECRET=42 cargo run -q
GUESS_SECRET=abc cargo run -q
GUESS_SECRET=500 cargo run -q
cargo test
```

The first command prints exactly `The secret number is 42`. The second and the third print `The secret number is N` with a random `N` (they ignore the bad value). `cargo test` passes, including the new test. Commit: `feat(p02): pick the secret number`.

#### M4 — One round

**Goal.** Print the banner and play exactly **one** round.

**Guidance.** In `main`: print the start line; print `Attempt 1 of 7 - type your guess:`; read a line; parse it with `parse_guess` (for now, on an error print the message with `eprintln!` and stop); compare with the secret using `match` on the `Ordering`; print `Too small!`, `Too big!` or the `win_message`. Remove the temporary "The secret number is" line.

<details>
<summary>Hint</summary>

Use the pattern from Lesson 02.7: `let mut line = String::new(); io::stdin().read_line(&mut line).expect("failed to read a line");`. The start line has constants inside it: `println!("Guess the number between {MIN_NUMBER} and {MAX_NUMBER}! You have {MAX_ATTEMPTS} attempts. Type 'quit' to give up.");`. Use `match parse_guess(&line)` with `Ok(n) => n` and `Err(message) => { eprintln!(...); return; }`, where the `eprintln!` prints `{message}. Please type a whole number from {MIN_NUMBER} to {MAX_NUMBER}.`.

</details>

**Done when:**

```bash
echo 50 | GUESS_SECRET=42 cargo run -q
echo 42 | GUESS_SECRET=42 cargo run -q
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Too big!
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
You win! You guessed it in 1 attempt.
```

Commit: `feat(p02): play one round`.

#### M5 — The full game

**Goal.** The loop with attempts, losing, `quit`, end of input and invalid input.

**Guidance.** Turn the single round into a loop that runs while `attempts < MAX_ATTEMPTS`. Count only valid guesses. Handle, in the order of the contract: end of input (`read_line` returned `0`), `quit` (case-insensitive: `eq_ignore_ascii_case`), invalid input (`eprintln!` and `continue`), then the comparison. After the loop print `Game over! The number was S.` Remember what `return` inside `main` does: it ends the program.

<details>
<summary>Hint</summary>

Create a **new** `String` at the start of every iteration. Use `let bytes = io::stdin().read_line(&mut line)...;` and `if bytes == 0 { … return; }`. For the quit test: `if line.trim().eq_ignore_ascii_case("quit")`. The invalid-input branch must not increase `attempts`, but it must also not print `Attempt 2` yet: the counter printed is `attempts + 1`, and `attempts` did not change. On a win, print `win_message(attempts)` and `return`.

</details>

**Done when** (all commands use `GUESS_SECRET=42`):

```bash
printf '50\n25\n37\n43\n40\n42\n' | GUESS_SECRET=42 cargo run -q
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Too big!
Attempt 2 of 7 - type your guess:
Too small!
Attempt 3 of 7 - type your guess:
Too small!
Attempt 4 of 7 - type your guess:
Too big!
Attempt 5 of 7 - type your guess:
Too small!
Attempt 6 of 7 - type your guess:
You win! You guessed it in 6 attempts.
```

```bash
printf 'abc\n0\n101\n\n42\n' | GUESS_SECRET=42 cargo run -q 2>&1
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
'abc' is not a whole number. Please type a whole number from 1 to 100.
Attempt 1 of 7 - type your guess:
0 is outside the range 1..=100. Please type a whole number from 1 to 100.
Attempt 1 of 7 - type your guess:
101 is outside the range 1..=100. Please type a whole number from 1 to 100.
Attempt 1 of 7 - type your guess:
'' is not a whole number. Please type a whole number from 1 to 100.
Attempt 1 of 7 - type your guess:
You win! You guessed it in 1 attempt.
```

```bash
printf '10\nQuit\n' | GUESS_SECRET=42 cargo run -q
printf '10\n' | GUESS_SECRET=42 cargo run -q
printf '1\n2\n3\n4\n5\n6\n7\n8\n' | GUESS_SECRET=42 cargo run -q | tail -3
printf 'abc\n42\n' | GUESS_SECRET=42 cargo run -q 2>/dev/null
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Too small!
Attempt 2 of 7 - type your guess:
You gave up. The number was 42.
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Too small!
Attempt 2 of 7 - type your guess:
Input ended. The number was 42.
```

```text
Attempt 7 of 7 - type your guess:
Too small!
Game over! The number was 42.
```

```text
Guess the number between 1 and 100! You have 7 attempts. Type 'quit' to give up.
Attempt 1 of 7 - type your guess:
Attempt 1 of 7 - type your guess:
You win! You guessed it in 1 attempt.
```

The last output shows that the message about `abc` went to stderr (hidden by `2>/dev/null`) and the attempt counter stayed at 1. Commit: `feat(p02): the full game loop`.

#### M6 — Finish: quality, README, merge and tag

**Goal.** A project that you would be proud to show.

**Guidance.**

1. Run `cargo fmt`, `cargo clippy` (no warnings), `cargo test` (all pass).
2. Write `README.md` in the project folder (10–20 lines): what the game is, how to run it (`cargo run`), how to play, how to test (`cargo test`), and the `GUESS_SECRET` hook, with a sentence that it exists for testing.
3. Play the game yourself with a real random secret and see that the numbers feel right.
4. Commit (`docs(p02): add README`), then go back to `main`, merge with `--no-ff`, tag the result, and delete the feature branch.

<details>
<summary>Hint</summary>

Commands: `git switch main`, `git merge --no-ff feature/guessing-game -m "feat(p02): merge the guessing game"`, `git tag -a p02-guessing-game-v0.1 -m "Guessing game finished"`, `git branch -d feature/guessing-game`. If `cargo clippy` complains about something, read the `help:` text; most suggestions are short. If you use `if let` (which Phase 04 teaches), Clippy may suggest a "collapsed" form; the nested `match` from M3 avoids that.

</details>

**Done when:**

```bash
cargo fmt --check && echo "fmt ok"
cargo clippy -q && echo "clippy ok"
cargo test 2>&1 | grep "test result"
git log --oneline --graph --decorate | head -8
```

prints `fmt ok`, `clippy ok` (with no warnings above it), a line `test result: ok. N passed; 0 failed` with `N` at least 6, and a graph where `main` has a merge commit with your milestone commits on the side branch and the tag `p02-guessing-game-v0.1`.

### Stretch goals

1. **Same-line prompt.** Print the prompt without a new line (`print!`) and flush (Lesson 02.7), so the player types right after `Your guess: `.
2. **Hot and cold.** Add the hint `Very close!` when the guess is within 3 of the secret.
3. **Difficulty levels.** Ask at the start for `easy` (1–50, 10 attempts), `normal` (1–100, 7 attempts) or `hard` (1–1000, 10 attempts). Replace the constants with variables, and put the settings in a function that returns a tuple.
4. **Play again.** After the game ask `Play again? (y/n)` and loop.
5. **Best score.** Keep the fewest attempts in the session and print it when the player wins.
6. **A smarter test.** Add a test for a function `fn next_round_message(attempts_used: u32) -> String` that you write, which gives `Attempt N of 7 - type your guess:`. Separate everything you can from `main`.
7. **Binary search.** Show that 7 attempts are enough for 1–100: compute `log2(100)` and explain why the best strategy is to always guess the middle of the remaining range. Write a small function that plays the game automatically with this strategy and test that it never needs more than 7 guesses.

## Common mistakes

- **Using `.cmp()` without `&`:** `guess.cmp(secret)` is a type error (`expected &u32, found u32`). Write `guess.cmp(&secret)`.
- **Forgetting `use rand::RngExt;`:** the error `no method named random_range` with `items from traits can only be used if the trait is in scope`.
- **Counting invalid input as an attempt:** increase `attempts` only after the guess is valid.
- **Reusing the input `String`:** `read_line` appends. Create a new `String` in each iteration.
- **Not trimming:** `"42\n"` fails to parse. `parse_guess` must trim.
- **Forgetting `mut` on `attempts`:** `E0384 cannot assign twice to immutable variable`.
- **Committing everything at the end:** one giant commit teaches nobody. Commit at each milestone.
- **Committing `target/`:** make sure the root `.gitignore` contains `target/`.
- **An infinite loop on EOF:** if `read_line` returns `0` and you do not stop, the program spins forever. Check the byte count.

> 🧪 **Testing:** The functions in the contract are pure: they take values and return values. This is the design that makes tests easy. The input/output part (`main`) is small and is checked by the acceptance commands in the milestones. This split is a pattern you will use in every project of this course.

> 📏 **Convention:** Keep `main` short and readable. Constants for the limits, functions with one job, doc comments `///` on the functions, tests at the bottom of the file, `cargo fmt` and `cargo clippy` before every commit.

> ⚡ **Performance:** The game is limited by the human, not by the computer; nothing here needs optimizing. A useful habit anyway: do not call `rand::rng()` inside a hot loop if you can create the generator once. For this game it is irrelevant.

## Quiz

1. Why does the program use `GUESS_SECRET`?

<details>
<summary>Answer</summary>

A random secret makes the output different every time, so it cannot be checked automatically. With `GUESS_SECRET` the secret is known, and you can write acceptance tests with a fixed input and a fixed expected output.

</details>

2. What is `Ordering`, and what are its values?

<details>
<summary>Answer</summary>

An enum from `std::cmp` returned by `cmp`. Its values are `Less`, `Equal` and `Greater`.

</details>

3. What is the difference between `Cargo.toml` and `Cargo.lock`?

<details>
<summary>Answer</summary>

`Cargo.toml` is written by you and says which crates and which version ranges you want (`rand = "0.10"`). `Cargo.lock` is written by Cargo and records the exact versions that were used, so builds are repeatable.

</details>

4. Why are `parse_guess`, `compare_guess` and `win_message` separate functions and not code inside `main`?

<details>
<summary>Answer</summary>

Functions with clear inputs and outputs can be tested with `assert_eq!` without a human typing. `main`, which reads from the terminal, stays small.

</details>

5. Why should invalid input not use up an attempt?

<details>
<summary>Answer</summary>

It is the contract of the game: a typing mistake is not a guess. The loop only increases `attempts` after a valid guess.

</details>

6. What does `git merge --no-ff` do, and why is it used here?

<details>
<summary>Answer</summary>

It always creates a merge commit, even when Git could just move the branch pointer. The history then shows the whole feature (all milestone commits) as one unit that is easy to read and, if necessary, to revert.

</details>

## Summary

- A good plan splits a program into milestones, each with a goal and a check.
- Add a crate with `cargo add`; Cargo downloads it, records the exact version in `Cargo.lock`, and you import the traits you need (`use rand::RngExt;`).
- `Ordering` and `match` give a clear three-way comparison with no forgotten case.
- Environment variables (`std::env::var`) can be a hook that makes a random program testable.
- Pure functions plus a thin `main` make code testable; acceptance tests check the whole program with `printf … | cargo run`.
- Use a feature branch, one commit per milestone, `merge --no-ff` and a tag, and keep `fmt`, `clippy` and `test` green.

## Sources and further reading

- [The Rust Programming Language — 2. Programming a Guessing Game](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html) — the official tutorial that this project is based on, built in a different way.
- [The Cargo Book — Specifying Dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html) — version requirements such as `"0.10"`.
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — why both files exist and what to commit.
- [rand on docs.rs](https://docs.rs/rand/0.10.4/rand/) — the documentation of the `rand` crate, including `RngExt`.
- [Standard library — std::cmp::Ordering](https://doc.rust-lang.org/std/cmp/enum.Ordering.html) — the enum used for comparing values.
- [Standard library — std::env::var](https://doc.rust-lang.org/std/env/fn.var.html) — reading environment variables.
- [Pro Git — Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — the branch and merge workflow used in this project.
- [Semantic Versioning 2.0.0](https://semver.org/) — the rules of version numbers.

---

[🏠 Start](../START-HERE.md) · [← 02.9 Debugging basics](09-debugging-basics.md) · [📋 Phase 02](README.md) · [02.11 Checkpoint →](11-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
