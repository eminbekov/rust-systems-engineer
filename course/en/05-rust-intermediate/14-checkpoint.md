<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.13 Project: minigrep](13-project-minigrep.md) · [📋 Phase 05](README.md) · [Phase 06 — Rust advanced →](../06-rust-advanced/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/14-checkpoint.md) <!-- nav -->

# Lesson 05.14 — Checkpoint

> **Phase 05 — Rust intermediate** · Lesson 14 of 14 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

This lesson teaches nothing new. It checks that you can use what Phase 05 taught, and it shows you where to go back if something is still weak. You will:

- Answer 20 questions about generics, traits, lifetimes, closures, iterators, tests, documentation, Cargo, smart pointers, threads, the practical crates, benchmarks and Git.
- Solve 5 practical tasks, each with tests, that mix several lessons.
- Find the lessons you should repeat, with direct links.

## Before you start

- You finished [Lesson 05.13 — Project: minigrep](13-project-minigrep.md). Your `minigrep` works, its tests pass, and it is committed.
- Do the quiz **closed book**: no lessons, no compiler, no search. Write your answer on paper or in your notes first, then open the hidden answer. A wrong answer found now is cheap; the same mistake found in a job is not.
- For the practical tasks you may use the compiler and the lessons, but try first without hints.
- Your learning repository is clean (`git status` in `~/projects/rust` says `nothing to commit, working tree clean`). Work on a branch: `git switch -c lesson/05-14-checkpoint`.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A pause at the end of a phase to test yourself before you go on. |
| **self-test** | A test that you give yourself and mark yourself, to find gaps. |
| **closed book** | Answering from memory only, without looking anything up. |
| **quality gate** | The commands that must pass before you accept your own work: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`. |
| **API** (Application Programming Interface) | The public functions and types of a library that other code uses. |

## Self-test quiz

Mark one point for each answer that is **correct and complete**. Be honest: nobody checks but you.

### Generics, traits and lifetimes

1. What is monomorphization, and what does it cost?

<details>
<summary>Answer</summary>

The compiler makes a separate copy of a generic function or type for each concrete type that is used (`largest::<i32>`, `largest::<f64>`, …). Because of this the generic code is as fast as hand-written code (no run-time cost). The costs are a longer compile time and a bigger binary ([Lesson 05.1](01-generics.md)).

</details>

2. In `fn largest<T: PartialOrd + Copy>(items: &[T]) -> T`, why are both bounds needed?

<details>
<summary>Answer</summary>

`PartialOrd` is needed to compare two items with `>`. `Copy` is needed to take a value out of the slice (`let mut max = items[0]`) and to return it without moving it out of a borrowed slice. The compiler checks the body using only the abilities named in the bounds ([Lesson 05.1](01-generics.md)).

</details>

3. What does this program print? (Do not run it.)

```rust
fn largest<T: PartialOrd + Copy>(items: &[T]) -> T {
    let mut max = items[0];
    for &item in items {
        if item > max {
            max = item;
        }
    }
    max
}

fn main() {
    println!("{}", largest(&[3, 9, 4]));
    println!("{}", largest(&[1.5, 0.2]));
    println!("{}", largest(&['x', 'b']));
}
```

<details>
<summary>Answer</summary>

```text
9
1.5
x
```

The same function works for integers, floats and characters, because all of them implement `PartialOrd` and `Copy`. Characters are compared by their code (`'x'` is bigger than `'b'`).

</details>

4. What is the orphan rule, and what do you do when it stops you?

<details>
<summary>Answer</summary>

You may implement a trait for a type only if the trait or the type is defined in your own crate. So you cannot write `impl Display for Vec<i32>` (both are foreign). The workaround is a *newtype*: a tuple struct of your own that wraps the foreign type (`struct Numbers(Vec<i32>);`) and implements the trait for it ([Lesson 05.2](02-traits.md)).

</details>

5. What is the difference between a required method and a default method in a trait? What does `#[derive(Debug, Clone, PartialEq)]` do?

<details>
<summary>Answer</summary>

A required method has only a signature; every type that implements the trait must write it. A default method has a body in the trait; implementors get it for free and may override it. A `derive` attribute asks the compiler to write the standard implementation of the named traits for your struct or enum ([Lesson 05.2](02-traits.md)).

</details>

6. Does a lifetime annotation such as `'a` make a value live longer? What is its job?

<details>
<summary>Answer</summary>

No. It never changes how long anything lives. It only *describes* the relation between the lifetimes of inputs and outputs (for example "the result is valid as long as `contents` is"), so that the compiler can check that no reference outlives the value it points to ([Lesson 05.3](03-lifetimes.md)).

</details>

7. This code does not compile. What is the error, and how do you fix it?

```rust
fn longest(a: &str, b: &str) -> &str {
    if a.len() > b.len() { a } else { b }
}
```

<details>
<summary>Answer</summary>

`error[E0106]: missing lifetime specifier`. The function returns a reference, and the compiler cannot tell whether it comes from `a` or from `b`. The elision rules do not help, because there are two input references. Fix: `fn longest<'a>(a: &'a str, b: &'a str) -> &'a str` ([Lesson 05.3](03-lifetimes.md)).

</details>

### Closures and iterators

8. What is the difference between `Fn`, `FnMut` and `FnOnce`?

<details>
<summary>Answer</summary>

`Fn` only reads what it captured and can be called many times. `FnMut` changes what it captured and can be called many times (it needs `mut`). `FnOnce` consumes (moves out) what it captured and can be called only once. Every closure is at least `FnOnce`; ask for the weakest trait your function needs ([Lesson 05.4](04-closures.md)).

</details>

9. What does this program print? (Do not run it.)

```rust
fn main() {
    let words = vec!["apple", "bob", "cat", "doge"];
    let n: usize = words.iter().filter(|w| w.len() > 3).map(|w| w.len()).sum();

    let mut counter = 0;
    let mut inc = || {
        counter += 2;
        counter
    };
    inc();
    inc();

    println!("{n} {counter}");
}
```

<details>
<summary>Answer</summary>

```text
9 4
```

The words longer than 3 letters are `apple` (5) and `doge` (4), and 5 + 4 = 9. The closure `inc` borrows `counter` mutably and adds 2 twice, so `counter` is 4. The mutable borrow ends after the last use of `inc`, so `println!` may read `counter`.

</details>

10. What does this program print? (Do not run it.)

```rust
fn main() {
    let lazy = (1..4).map(|x| {
        println!("working on {x}");
        x * 10
    });
    println!("created the iterator");
    let total: i32 = lazy.sum();
    println!("total = {total}");
}
```

<details>
<summary>Answer</summary>

```text
created the iterator
working on 1
working on 2
working on 3
total = 60
```

Iterator adapters such as `map` are lazy: nothing happens when you create the pipeline. The work starts when a consumer (`sum`) asks for items ([Lesson 05.5](05-iterators.md)).

</details>

11. What are the differences between `iter()`, `iter_mut()` and `into_iter()` on a `Vec<String>`?

<details>
<summary>Answer</summary>

`iter()` gives `&String` (shared borrows; the vector is still usable). `iter_mut()` gives `&mut String` (you can change the items). `into_iter()` gives `String` and consumes the vector (it is moved) ([Lesson 05.5](05-iterators.md)).

</details>

### Tests, documentation and Cargo

12. What is the difference between a unit test, an integration test and a doc test? Where does each live? Why does an integration test need a `lib.rs`?

<details>
<summary>Answer</summary>

A unit test lives in the same file in `#[cfg(test)] mod tests` and can see private items. An integration test is a file in `tests/`, compiled as a separate crate, and sees only the public API. A doc test is a code block in a `///` comment that `cargo test` compiles and runs. An integration test can only `use` a library crate; it cannot import a binary crate, so the logic must be in `src/lib.rs` ([Lesson 05.6](06-testing-in-depth.md), [Lesson 05.7](07-documentation.md)).

</details>

13. Which versions does `serde = "1.0.200"` accept? And `itertools = "0.15"`? Should the `Cargo.lock` of a program be in Git?

<details>
<summary>Answer</summary>

`"1.0.200"` accepts `1.0.200` up to, but not including, `2.0.0`. `"0.15"` accepts `0.15.0` up to, but not including, `0.16.0` (for `0.x` crates the minor number is the "breaking" number). Yes, commit `Cargo.lock` for programs, so everyone builds with the same versions; never commit `target/` ([Lesson 05.8](08-cargo-dependencies-and-versions.md)).

</details>

14. A function from a crate's documentation is "not found", though the crate is in `Cargo.toml`. What is the most likely cause, and how do you fix it?

<details>
<summary>Answer</summary>

The function is behind a Cargo *feature* that is not switched on (`error[E0599]: no function found …`). Read the feature list on docs.rs or in `cargo info <crate>`, then add `features = ["name"]` to the dependency (`cargo add <crate> --features name`) ([Lesson 05.8](08-cargo-dependencies-and-versions.md)).

</details>

### Smart pointers and threads

15. Pick the right type: (a) a recursive `List` enum; (b) a graph node shared by several owners in one thread; (c) a counter that eight threads increase; (d) a parent link in a tree whose children are owned by the parent.

<details>
<summary>Answer</summary>

(a) `Box<List>`: a fixed-size pointer breaks the infinite size. (b) `Rc<T>`, and `Rc<RefCell<T>>` if the node must change. (c) `Arc<Mutex<u64>>` (or an `AtomicU64` for a plain counter). (d) `Weak<T>` for the link up, so there is no reference cycle ([Lesson 05.9](09-smart-pointers.md), [Lesson 05.10](10-threads-channels-and-shared-state.md)).

</details>

16. What does this program print? (Do not run it.)

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let a = Rc::new(RefCell::new(vec![1]));
    let b = Rc::clone(&a);
    b.borrow_mut().push(2);
    println!("{:?} {}", a.borrow(), Rc::strong_count(&a));
}
```

<details>
<summary>Answer</summary>

```text
[1, 2] 2
```

`a` and `b` are two owners of the same vector, so the push through `b` is visible through `a`, and the strong count is 2. `RefCell` allows the change behind a shared `Rc`; its rules are checked at run time ([Lesson 05.9](09-smart-pointers.md)).

</details>

17. Why must a closure passed to `thread::spawn` usually be `move`? Why can `Rc` not be sent to a thread? What is a deadlock, and how do you avoid it?

<details>
<summary>Answer</summary>

A spawned thread can outlive the function that started it, so it must own its data (`move`); otherwise error `E0373`. `Rc` has a non-atomic counter and is not `Send`, so the compiler refuses it (`E0277`); use `Arc`. A deadlock is two or more threads waiting for each other's locks forever. Avoid it by taking locks in the same order everywhere, holding them briefly, and not calling unknown code while holding one ([Lesson 05.10](10-threads-channels-and-shared-state.md)).

</details>

### Practical crates, benchmarks and Git

18. Which crate would you use for each job, and why: (a) turn a struct into JSON; (b) parse `--count 3`; (c) the error type of a library; (d) the error type at the top of an application? What does `#[from]` do?

<details>
<summary>Answer</summary>

(a) `serde` with `serde_json` (`#[derive(Serialize, Deserialize)]`). (b) `clap` (`#[derive(Parser)]`). (c) `thiserror`: a precise enum that callers can match on. (d) `anyhow`: one catch-all error type with `.context(...)`. `#[from]` implements `From<SourceError>` for your enum so that `?` converts automatically, and records the source error ([Lesson 05.11](11-practical-crates-serde-clap-errors.md)).

</details>

19. A benchmark reports `41 ns` for a loop of 200 million iterations. What happened, and what are four rules of honest measuring?

<details>
<summary>Answer</summary>

The compiler removed the work (the result was never used) or computed it at compile time, so only the clock reading was measured. Rules: use `--release` (or `cargo bench`); use `black_box` for inputs and results; run many times with warm-up and look at the median and the spread; change one thing at a time and compare with a saved baseline; keep a correctness test; record the environment ([Lesson 05.12](12-benchmarking-basics.md)).

</details>

20. In `minigrep` you made the output 16 times faster. Which Git steps do you take so that the change is safe and provable? What are the three exit statuses of the tool?

<details>
<summary>Answer</summary>

Work on a branch (`git switch -c perf/buffered-output`), change one thing, run the tests, measure before and after with the same command, commit with the numbers in the message (`perf(minigrep): buffer the output (734 ms -> 45 ms)`), merge, delete the branch, and tag the release (`git tag -a minigrep-v0.1.0`). Exit statuses: `0` (something was selected, no error), `1` (nothing selected), `2` (any error) ([Lesson 05.13](13-project-minigrep.md)).

</details>

**Your score:** count your correct answers. 18–20: excellent. 14–17: good, repeat the lessons of the questions you missed (the table below). Fewer than 14: repeat the weak lessons and redo their exercises before Phase 06.

## Practical tasks

Create one practice package for the tasks: `cd ~/projects/rust && git switch -c lesson/05-14-checkpoint && cargo new --lib code/p05-checkpoint`. Add the dependencies with `cargo add serde@1 --features derive`, `cargo add serde_json@1 thiserror@2 anyhow@1`. Write every task test-first: the tests, then the code. After each task, run the quality gate (`cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`) and commit with `git add . && git commit -m "feat(p05): …"`.

### Task 1 — A generic stack with Display

Write `Stack<T>` (a struct with a `Vec<T>` inside) with `new`, `push`, `pop` (returns `Option<T>`), `peek` (returns `Option<&T>`), `len` and `is_empty`. Derive `Debug` and `Default`. Implement `Display` for `Stack<T>` when `T: Display`: the items from the bottom to the top in square brackets, separated by a comma and a space (`[a, b]`; an empty stack is `[]`). Test: last in, first out; `peek`; the display of an empty and a filled stack.

<details>
<summary>Hint</summary>

`impl<T> Stack<T> { … }` for the methods, and a separate `impl<T: fmt::Display> fmt::Display for Stack<T>` for the printing. In `fmt`, use `write!(f, "[")?;`, then loop with `enumerate()` and write `", "` before every item except the first, then `write!(f, "]")`. `Vec::last()` gives `Option<&T>`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fmt;

#[derive(Debug, Default)]
pub struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Stack { items: Vec::new() }
    }

    pub fn push(&mut self, item: T) {
        self.items.push(item);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    pub fn peek(&self) -> Option<&T> {
        self.items.last()
    }

    pub fn len(&self) -> usize {
        self.items.len()
    }

    pub fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}

impl<T: fmt::Display> fmt::Display for Stack<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[")?;
        for (i, item) in self.items.iter().enumerate() {
            if i > 0 {
                write!(f, ", ")?;
            }
            write!(f, "{item}")?;
        }
        write!(f, "]")
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn stack_is_last_in_first_out() {
        let mut s = Stack::new();
        assert!(s.is_empty());
        s.push(1);
        s.push(2);
        s.push(3);
        assert_eq!(s.len(), 3);
        assert_eq!(s.peek(), Some(&3));
        assert_eq!(s.pop(), Some(3));
        assert_eq!(s.pop(), Some(2));
        assert_eq!(s.pop(), Some(1));
        assert_eq!(s.pop(), None);
    }

    #[test]
    fn stack_displays_bottom_to_top() {
        let mut s = Stack::new();
        assert_eq!(s.to_string(), "[]");
        s.push("a");
        s.push("b");
        assert_eq!(s.to_string(), "[a, b]");
        let words: Stack<String> = Stack::default();
        assert_eq!(format!("{words}"), "[]");
    }
}
```

Why it works: the methods need no bounds, because they only move values around. `Display` needs `T: Display` because it prints the items. `#[derive(Default)]` on a generic struct adds the bound `T: Default` to the generated implementation, so `Stack::<String>::default()` works because `String` implements `Default`.

</details>

### Task 2 — Lifetimes and iterators

Write two functions without allocating new strings for the results: `longest_line(text: &str) -> Option<&str>` (the longest line, or `None` for an empty text) and `parse_pairs(text: &str) -> Vec<(&str, &str)>`. The second one splits every line at the first `=`, trims both parts, and skips lines that have no `=` or an empty key. Use iterator adapters. Test with the text `"name = Ana\nbroken line\n= no key\nage=30\n"`; the expected result is `[("name", "Ana"), ("age", "30")]`.

<details>
<summary>Hint</summary>

`text.lines().max_by_key(|line| line.len())` for the first one (the elided lifetime ties the result to `text`). For the second: `text.lines().filter_map(|line| …).collect()`, and inside the closure use `line.split_once('=')?` (the `?` works on `Option` inside a closure that returns `Option`).

</details>

<details>
<summary>Answer</summary>

```rust
pub fn longest_line(text: &str) -> Option<&str> {
    text.lines().max_by_key(|line| line.len())
}

pub fn parse_pairs(text: &str) -> Vec<(&str, &str)> {
    text.lines()
        .filter_map(|line| {
            let (key, value) = line.split_once('=')?;
            let key = key.trim();
            if key.is_empty() {
                return None;
            }
            Some((key, value.trim()))
        })
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_the_longest_line() {
        assert_eq!(longest_line("ab\nabcd\nabc"), Some("abcd"));
        assert_eq!(longest_line(""), None);
    }

    #[test]
    fn parses_key_value_lines() {
        let text = "name = Ana\nbroken line\n= no key\nage=30\n";
        assert_eq!(parse_pairs(text), vec![("name", "Ana"), ("age", "30")]);
    }
}
```

Why it works: with one input reference, the elision rule gives the result the same lifetime as `text`, so no annotation is needed, and the returned `&str` values are slices of the input (zero copy). `split_once` returns `Option<(&str, &str)>`, `trim` returns a sub-slice, and `filter_map` drops the `None` lines.

</details>

### Task 3 — Threads and channels

Write `word_frequencies(texts: Vec<String>) -> BTreeMap<String, usize>`. Start one thread per text. Each thread counts its words (in lower case) in its own `HashMap` and **sends the finished map** through a channel. The main thread receives all maps and merges them into one `BTreeMap`. Test with `"The cat and the dog"` and `"a cat"`: `the` = 2, `cat` = 2, `dog` = 1, and 5 different words.

<details>
<summary>Hint</summary>

Create `let (tx, rx) = mpsc::channel::<HashMap<String, usize>>();`, give each thread `tx.clone()` in a `move` closure, then `drop(tx)` so that the channel closes, and loop `for partial in rx`. Merge with `*total.entry(word).or_insert(0) += n;`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::{BTreeMap, HashMap};
use std::sync::mpsc;
use std::thread;

pub fn word_frequencies(texts: Vec<String>) -> BTreeMap<String, usize> {
    let (tx, rx) = mpsc::channel::<HashMap<String, usize>>();

    for text in texts {
        let tx = tx.clone();
        thread::spawn(move || {
            let mut counts = HashMap::new();
            for word in text.split_whitespace() {
                *counts.entry(word.to_lowercase()).or_insert(0) += 1;
            }
            tx.send(counts).unwrap();
        });
    }
    drop(tx);

    let mut total = BTreeMap::new();
    for partial in rx {
        for (word, n) in partial {
            *total.entry(word).or_insert(0) += n;
        }
    }
    total
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_words_with_channels() {
        let texts = vec![
            String::from("The cat and the dog"),
            String::from("a cat"),
        ];
        let counts = word_frequencies(texts);
        assert_eq!(counts["the"], 2);
        assert_eq!(counts["cat"], 2);
        assert_eq!(counts["dog"], 1);
        assert_eq!(counts.len(), 5);
    }
}
```

Why it works: no data is shared while the threads work; each owns its map, and ownership moves through the channel (message passing). The original `tx` is dropped, so the receiver loop ends when the last thread has finished. The test does not depend on the arrival order, because merging is the same in any order. A `BTreeMap` also gives sorted output.

</details>

### Task 4 — Configuration with serde, thiserror and anyhow

Define `Config { name: String, port: u16, verbose: bool }` that can be read from JSON (`verbose` is optional and `false` by default). Write `parse_config(text: &str) -> Result<Config, ConfigError>` where `ConfigError` (use `thiserror`) has three variants: invalid JSON (with `#[from]`), an empty or blank name, and a port below 1024 (message `port 80 is not allowed (use 1024 to 65535)`). Then write `describe(text: &str) -> anyhow::Result<String>` that returns `"web on port 8080"` for good input and adds the context `cannot use the configuration` to errors. Test every error variant and the `{:#}` form of the error from `describe`.

<details>
<summary>Hint</summary>

`#[derive(Deserialize, PartialEq, Debug)]` and `#[serde(default)]` on `verbose`. For the test of the invalid-JSON variant use `matches!(parse_config("not json"), Err(ConfigError::Json(_)))`. `describe` is `parse_config(text).context("cannot use the configuration")?` and then `format!`.

</details>

<details>
<summary>Answer</summary>

```rust
use anyhow::Context;
use serde::Deserialize;
use thiserror::Error;

#[derive(Debug, Deserialize, PartialEq)]
pub struct Config {
    pub name: String,
    pub port: u16,
    #[serde(default)]
    pub verbose: bool,
}

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("the configuration is not valid JSON")]
    Json(#[from] serde_json::Error),
    #[error("the name must not be empty")]
    EmptyName,
    #[error("port {0} is not allowed (use 1024 to 65535)")]
    BadPort(u16),
}

pub fn parse_config(text: &str) -> Result<Config, ConfigError> {
    let config: Config = serde_json::from_str(text)?;
    if config.name.trim().is_empty() {
        return Err(ConfigError::EmptyName);
    }
    if config.port < 1024 {
        return Err(ConfigError::BadPort(config.port));
    }
    Ok(config)
}

pub fn describe(text: &str) -> anyhow::Result<String> {
    let config = parse_config(text).context("cannot use the configuration")?;
    Ok(format!("{} on port {}", config.name, config.port))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn config_is_checked() {
        let ok = parse_config(r#"{"name":"web","port":8080}"#).unwrap();
        assert_eq!(ok, Config { name: "web".into(), port: 8080, verbose: false });

        assert!(matches!(parse_config("not json"), Err(ConfigError::Json(_))));
        assert!(matches!(
            parse_config(r#"{"name":" ","port":8080}"#),
            Err(ConfigError::EmptyName)
        ));
        let err = parse_config(r#"{"name":"web","port":80}"#).unwrap_err();
        assert_eq!(err.to_string(), "port 80 is not allowed (use 1024 to 65535)");
    }

    #[test]
    fn describe_adds_context() {
        assert_eq!(
            describe(r#"{"name":"web","port":8080}"#).unwrap(),
            "web on port 8080"
        );
        let err = describe(r#"{"name":"web","port":80}"#).unwrap_err();
        assert_eq!(
            format!("{err:#}"),
            "cannot use the configuration: port 80 is not allowed (use 1024 to 65535)"
        );
    }
}
```

Why it works: `serde` checks the shape (a missing `port` or a text where a number is expected is a `Json` error), and your code checks the meaning (blank name, reserved port). `thiserror` gives the library part precise variants; `anyhow` at the top adds context, and `{:#}` prints the whole chain on one line. The `matches!` macro tests a variant without requiring `PartialEq` on the error type.

</details>

### Task 5 — Measure and release with Git (no new code)

Take your `minigrep` from Lesson 05.13 and prove that the work is complete and reproducible. First switch to the branch that holds it (`git switch project/05-minigrep`, or `main` if you already merged it), then:

1. Run the quality gate and `cargo test` in the project folder. Everything must pass without warnings.
2. Check your history: `git log --oneline` shows one commit per milestone and one `perf` commit with before/after numbers in its message.
3. Show that a release is tagged and that the tag points to the right commit.
4. Make sure the benchmark, the integration tests and the README are all in the repository (`git ls-files code/p05-minigrep`), and the `target` folder is not.
5. Write three lines in your learning notes: the slowest part you found, how you proved the improvement, and one stretch goal you want to try.

<details>
<summary>Hint</summary>

`git tag` lists tags; `git log --oneline --decorate` shows which commit has the tag. If you forgot the tag, add it: `git tag -a minigrep-v0.1.0 -m "minigrep 0.1.0"`. `git ls-files` lists the files that Git tracks; `target/` must not appear.

</details>

<details>
<summary>Answer</summary>

```bash
cd ~/projects/rust/code/p05-minigrep
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
git log --oneline --decorate -4
git tag
git ls-files .
```

A tidy history looks like this (the IDs differ on your computer):

```text
08bfc4e (HEAD -> project/05-minigrep, tag: minigrep-v0.1.0) perf(minigrep): buffer the output (734 ms -> 45 ms)
0cd2c44 feat(minigrep): add the search function
5f38302 feat(minigrep): parse the command line
```

`git tag` prints `minigrep-v0.1.0`. `git ls-files .` lists `Cargo.toml`, `Cargo.lock`, `README.md`, `benches/search.rs`, `src/lib.rs`, `src/main.rs` and `tests/cli.rs`, and no file from `target/`. An *annotated* tag (`-a`) stores who made it, when, and a message; `git show minigrep-v0.1.0` displays them.

Why it works: a reproducible project can be built by anybody from the repository alone (`Cargo.lock` is in, `target/` is out), proves its behavior (tests), and shows its history of decisions (one commit per milestone, numbers in the performance commit). That is what makes a small tool a piece of engineering work.

</details>

## If you got something wrong

| Topic | Lesson to repeat |
|---|---|
| Generic functions and types, trait bounds, monomorphization | [Lesson 05.1 — Generics](01-generics.md) |
| Traits, `derive`, `Display`, `From`, orphan rule | [Lesson 05.2 — Traits](02-traits.md) |
| Lifetime annotations, elision, structs with references | [Lesson 05.3 — Lifetimes](03-lifetimes.md) |
| Closures, `move`, `Fn` / `FnMut` / `FnOnce` | [Lesson 05.4 — Closures](04-closures.md) |
| Iterators, adapters, consumers, `collect` | [Lesson 05.5 — Iterators](05-iterators.md) |
| TDD, unit and integration tests, test options | [Lesson 05.6 — Testing in depth](06-testing-in-depth.md) |
| Doc comments, doc tests, README, changelog | [Lesson 05.7 — Documentation](07-documentation.md) |
| Version requirements, `Cargo.lock`, features, `cargo tree` | [Lesson 05.8 — Cargo: dependencies and versions](08-cargo-dependencies-and-versions.md) |
| `Box`, `Rc`, `RefCell`, `Weak`, `Drop`, `Deref` | [Lesson 05.9 — Smart pointers](09-smart-pointers.md) |
| Threads, channels, `Arc<Mutex<T>>`, `Send` / `Sync` | [Lesson 05.10 — Threads, channels and shared state](10-threads-channels-and-shared-state.md) |
| `serde`, `clap`, `thiserror`, `anyhow` | [Lesson 05.11 — Practical crates: serde, clap, thiserror and anyhow](11-practical-crates-serde-clap-errors.md) |
| Release mode, `black_box`, `criterion`, `hyperfine` | [Lesson 05.12 — Benchmarking basics](12-benchmarking-basics.md) |
| Library plus thin binary, exit status, integration tests of a CLI | [Lesson 05.13 — Project: minigrep](13-project-minigrep.md) |
| Ownership, borrowing and references (behind many errors here) | [Lesson 03.5 — Mutable references and the borrowing rules](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) |
| `Result`, `?` and custom errors | [Lesson 04.10 — Error handling](../04-structs-enums-collections-errors/10-error-handling.md) |
| Branches, merging, undoing changes | [Lesson 01.6 — Branches and merging](../01-git-and-version-control/06-branches-and-merging.md) and [Lesson 01.5 — Undoing changes safely](../01-git-and-version-control/05-undoing-changes-safely.md) |

When you repeat a lesson, do not only reread it. Redo its exercises on a branch, and write the tests first.

## What's next

You finished Phase 05. You can now write generic, tested, documented code; use the common crates; share work between threads safely; and measure speed. Before you go on:

1. Merge your checkpoint branch (`git switch main && git merge lesson/05-14-checkpoint`) and delete the finished lesson branches you no longer need (`git branch -d <name>`).
2. Add a few lines to your learning notes: what was hard in this phase, what you want to remember, and which of the stretch goals of `minigrep` you plan to do.
3. Take a break. A day off helps your memory more than another hour of reading.

In **Phase 06 — Rust advanced** you will go deeper into the parts of the language that you touched here: trait objects and dynamic dispatch (`Box<dyn Trait>`), advanced traits and patterns, interior mutability in depth, lifetimes in depth, `unsafe` code and Miri, macros, `async`/`await` and calling C code. The phase ends with a multithreaded web server and the checkpoint of Stage 1. Start with [Lesson 06.1 — Trait objects and dynamic dispatch](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md).

## Sources and further reading

- [The Rust Programming Language — Chapter 10: Generic Types, Traits, and Lifetimes](https://doc.rust-lang.org/book/ch10-00-generics.html) — the official chapter behind Lessons 05.1 to 05.3.
- [The Rust Programming Language — Chapter 13: Functional Language Features](https://doc.rust-lang.org/book/ch13-00-functional-features.html) — closures and iterators.
- [The Rust Programming Language — Chapter 15: Smart Pointers](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html) and [Chapter 16: Fearless Concurrency](https://doc.rust-lang.org/book/ch16-00-concurrency.html) — Lessons 05.9 and 05.10.
- [The Cargo Book](https://doc.rust-lang.org/cargo/) — the full reference for dependencies, features, profiles and tests.
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — short runnable examples for all topics; good for practice.
- [Pro Git](https://git-scm.com/book/en/v2) — the free official Git book; chapters 2 (basics) and 3 (branching) cover your daily commands.

---

[🏠 Start](../START-HERE.md) · [← 05.13 Project: minigrep](13-project-minigrep.md) · [📋 Phase 05](README.md) · [Phase 06 — Rust advanced →](../06-rust-advanced/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
