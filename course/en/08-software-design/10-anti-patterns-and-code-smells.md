<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.9 Strategy and dependency injection](09-strategy-and-dependency-injection.md) · [📋 Phase 08](README.md) · [08.11 Conventions: naming, Clippy and the Rust API Guidelines →](11-conventions-naming-clippy-and-api-guidelines.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/10-anti-patterns-and-code-smells.md) <!-- nav -->

# Lesson 08.10 — Anti-patterns and code smells

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 10 of 17 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a *code smell* and an *anti-pattern* are, and why a smell is a question, not a verdict.
- The most common smells in any language: magic numbers, stringly-typed data, boolean parameters, long parameter lists, duplicated code, "god" types.
- Five anti-patterns that are specific to Rust: needless `clone`, `unwrap` on outside data, `Rc<RefCell<T>>` everywhere, "inheritance" through `Deref`, and `#![deny(warnings)]`.
- How Clippy (the Rust linter) finds some smells for you, and how to decide whether to fix one.

## Before you start

- You finished [Lesson 08.9 — Strategy and dependency injection](09-strategy-and-dependency-injection.md). The principles and patterns of Lessons 08.1 to 08.9 give you the "good" side; this lesson shows the "bad" side.
- You know `enum` and `match` ([Lesson 04.3](../04-structs-enums-collections-errors/03-enums.md)), `Result` ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) and smart pointers ([Lesson 05.9](../05-rust-intermediate/09-smart-pointers.md)).
- Create a practice package once: `cargo new code/p08-smells`. In each example below, replace the content of `src/main.rs`, then run `cargo run`. (In this phase, "example" code goes into `src/main.rs` of a `code/p08-<topic>` package unless the lesson says otherwise.)

## New words

| Word | Meaning |
|---|---|
| **code smell** | A surface sign in the code that *may* point to a deeper design problem. The code works, but something feels wrong. |
| **anti-pattern** | A common "solution" that looks reasonable but usually causes more problems than it solves. |
| **magic number** | A bare number in the code whose meaning is not clear from the code around it, like `86400` or `0.8`. |
| **stringly-typed** | Using `String` or `&str` for data that has only a few valid values (a status, a country code), instead of an `enum`. |
| **primitive obsession** | Using plain `u32`, `bool` or `String` for everything, instead of small types with names and rules. |
| **boolean blindness** | A `bool` argument whose meaning you cannot see at the call site: `render(true, false)`. |
| **god type** | A struct that knows and does too much: dozens of fields and methods about unrelated topics. |
| **linter** | A tool that reads your code without running it and warns about suspicious or unidiomatic code. A single warning rule is a **lint**. |
| **Clippy** | The official Rust linter. You run it with `cargo clippy`. |

## 1. Smells and anti-patterns

A **code smell** is a hint. A long function is a smell: it breaks nothing today, but long functions are hard
to read and to test, and they hide bugs. If you smell smoke, you check; you do not call the fire brigade
at once. An **anti-pattern** is stronger: a *recipe* that people keep using because it looks like a fix, while
it creates new problems. "The borrow checker complains, so I add `.clone()` everywhere" is one: it hides the real
question (who should own this data?).

Three rules for the whole lesson:

1. A smell is a *question*. Sometimes the answer is "it is fine".
2. Name the **concrete problem** (a bug, a hard test, a hard change). If you cannot name one, leave the code
   alone (Lesson 08.1: KISS, "Keep It Simple, Stupid", and YAGNI, "You Aren't Gonna Need It").
3. Fix smells **with tests in place** ([Lesson 08.13](13-refactoring-and-code-review.md) shows how).

## 2. Smells in any code

We will use one running example: a function that prices the shipping of a parcel.

### 2.1 Magic numbers, stringly-typed data and boolean blindness

```rust
fn shipping(weight: u32, country: &str, express: bool) -> u32 {
    let mut cost = if weight < 1000 {
        500
    } else if weight < 5000 {
        900
    } else {
        1500
    };
    if country == "DE" {
        cost = cost * 80 / 100;
    } else if country != "FR" {
        cost += 700;
    }
    if express {
        cost *= 2;
    }
    cost
}

fn main() {
    println!("{}", shipping(500, "DE", false));
    println!("{}", shipping(500, "de", false));
    println!("{}", shipping(500, "DE", true));
}
```

**What happens:**

1. `weight` is a `u32`. The function does not say whether it is grams or kilograms; the numbers `1000` and
   `5000` have no names. Those are **magic numbers**.
2. `country` is a `&str`. Only the exact text `"DE"` counts as home; `"FR"` is a neighbour; every other text
   is foreign. That is **stringly-typed** data.
3. `express` is a `bool`. At the call site, `shipping(500, "DE", true)` does not tell the reader what `true`
   means. That is **boolean blindness**.
4. The second call passes `"de"` in lowercase. Nothing complains.

```text
400
1200
800
```

The second line is the real problem. A typo in the country code silently gives a wrong price (`1200`
instead of `400`): no error, no warning, and the bug may live in production for months.

Here is the same logic after small, mechanical changes: constants with names, an `enum` for the zone, an
`enum` for the speed, and a function for the price tiers.

```rust
const LIGHT_LIMIT_GRAMS: u32 = 1_000;
const MEDIUM_LIMIT_GRAMS: u32 = 5_000;
const LIGHT_PRICE: u32 = 500;
const MEDIUM_PRICE: u32 = 900;
const HEAVY_PRICE: u32 = 1_500;
const FOREIGN_SURCHARGE: u32 = 700;
const HOME_DISCOUNT_PERCENT: u32 = 20;

#[derive(Debug, Clone, Copy, PartialEq)]
enum Zone {
    Home,
    Neighbour,
    Foreign,
}

impl Zone {
    fn from_country_code(code: &str) -> Option<Zone> {
        match code {
            "DE" => Some(Zone::Home),
            "FR" => Some(Zone::Neighbour),
            "US" | "JP" => Some(Zone::Foreign),
            _ => None,
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq)]
enum Speed {
    Standard,
    Express,
}

fn base_price(weight_grams: u32) -> u32 {
    if weight_grams < LIGHT_LIMIT_GRAMS {
        LIGHT_PRICE
    } else if weight_grams < MEDIUM_LIMIT_GRAMS {
        MEDIUM_PRICE
    } else {
        HEAVY_PRICE
    }
}

fn shipping(weight_grams: u32, zone: Zone, speed: Speed) -> u32 {
    let mut cost = base_price(weight_grams);
    match zone {
        Zone::Home => cost = cost * (100 - HOME_DISCOUNT_PERCENT) / 100,
        Zone::Neighbour => {}
        Zone::Foreign => cost += FOREIGN_SURCHARGE,
    }
    if speed == Speed::Express {
        cost *= 2;
    }
    cost
}

fn main() {
    println!("{:?}", Zone::from_country_code("de"));
    let zone = Zone::from_country_code("DE").unwrap();
    println!("{}", shipping(500, zone, Speed::Standard));
    println!("{}", shipping(500, zone, Speed::Express));
}
```

**What happens:**

1. Each constant has a name that carries the unit (`_GRAMS`) or the meaning (`FOREIGN_SURCHARGE`). A reader
   no longer asks "what is 700?", and a price change happens in one place.
2. `Zone` has three variants. The only place where text becomes a `Zone` is `from_country_code`, which
   returns `Option<Zone>`: an unknown or mistyped code gives `None`, and the caller must decide what to do.
3. `match zone` has no `_` branch, so if you add a fourth zone later, the compiler points at every `match` you
   must update.
4. `Speed::Express` is readable at the call site. (Lesson 08.5 explains the idea behind this: make invalid
   states impossible.)

```text
None
400
800
```

> ⚠️ **Common mistake:** Naming a constant after its value: `const FIVE_THOUSAND: u32 = 5_000;`. The name must
> say what the number *means* (`MEDIUM_LIMIT_GRAMS`), not repeat what it *is*.

### 2.2 The catalogue

Here are the smells you meet most often, each with the concrete problem it causes and the usual direction of
the fix. Use the table as a checklist during code review (Lesson 08.13).

| Smell | How it looks | Problem it causes | Usual fix |
|---|---|---|---|
| **Long function** | One function of 80 lines with comments like `// step 2` | Hard to read, hard to test one step alone | Extract functions, one per step |
| **Magic number** | `if age > 17`, `* 86400` | Meaning hidden; a change must be found by search | A named `const` or an `enum` |
| **Stringly-typed data** | `status: String` with values `"open"`, `"closed"` | A typo is a silent bug, not a compile error | `enum` + one `parse` function |
| **Boolean blindness** | `draw(true, false, true)` | The call site is unreadable; arguments get swapped | `enum` per flag, or an options struct |
| **Long parameter list** | A function with 8 parameters | Hard to call; order mistakes | A struct for the group, or a builder (Lesson 08.6) |
| **Primitive obsession** | `user_id: u64` and `order_id: u64` | You can pass an order id where a user id is expected | A newtype per id (Lesson 08.5) |
| **Duplicated code** | The same 10 lines in three places | A bug fix must be repeated, and one copy is forgotten | Extract one function (but see the rule of three, Lesson 08.1) |
| **God type** | `struct App` with 30 fields: database, screen state, settings | Everything depends on everything; tests need all of it | Split by responsibility (Lesson 08.2) |
| **Feature envy** | A method that mostly reads fields of *another* struct | The logic sits far from the data it uses | Move the method to the type that owns the data |
| **Shotgun surgery** | One small change needs edits in ten files | Easy to miss one place | Gather the scattered knowledge into one module |

Notice that every row has a "problem it causes". That is the test from Section 1: a smell without a
concrete problem is just a style opinion.

> 📏 **Convention:** A function that fits on one screen (about 30 to 40 lines), takes at most four or five
> parameters, and does one thing is a good target. Clippy warns at **eight** parameters
> (`clippy::too_many_arguments`); that is a safety net, not a goal.

## 3. Anti-patterns that are common in Rust

### 3.1 Needless `clone` to make the compiler quiet

The borrow checker (Lesson 03.5) sometimes says "no", and the quickest way to a green build is `.clone()`.
Sometimes that is right. Often it hides a wrong signature. Here `longest_word` takes `text: String`, so it
*takes ownership*, and a caller who wants to keep `text` must copy the whole text first:

```rust
fn longest_word(text: String) -> String { /* ... */ }

let word = longest_word(text.clone());
```

But the function only *reads* the text; it never needed ownership. It works, and that is why it is an
anti-pattern: nobody notices that every call copies the text. The fix is in the signature. Take `&str` (a
borrowed view) and return a borrowed part of it:

```rust
fn longest_word(text: &str) -> &str {
    let mut best = "";
    for word in text.split_whitespace() {
        if word.len() > best.len() {
            best = word;
        }
    }
    best
}

fn main() {
    let text = String::from("the quick brown fox");
    let word = longest_word(&text);
    println!("{word} (from: {text})");
}
```

```text
quick (from: the quick brown fox)
```

There is no copy: `word` points into `text`. The rule of thumb: **when you feel the urge to write `.clone()`,
ask who should own the data**. Often the answer is "nobody needs to own it; borrow it".

> ⚡ **Performance:** A `clone` of a `String` or a `Vec` allocates new memory and copies every element. Cloning
> a small value once is fine; cloning a big value in a loop gives a slow program with no visible cause in the
> code. Measure first (Lesson 05.12), but do not add clones "just in case".

### 3.2 `unwrap` and `expect` on data that comes from outside

`unwrap()` means "if this is an error, crash the whole program". In an example or a test that is fine. In
code that handles *input* (a file, a network message, text from a user) it turns a normal event (bad
input) into a crash:

```rust
fn parse_port(text: &str) -> u16 {
    text.trim().parse().unwrap()
}

fn main() {
    println!("{}", parse_port("8080"));
    println!("{}", parse_port("80 80"));
}
```

```text
8080

thread 'main' (4616161) panicked at src/main.rs:2:25:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

(The number in brackets is an identifier of the thread; yours differs.) The message says what went wrong
technically, but not *which input* was bad or what the user should do. The fix is to return a `Result` and let
the caller decide (Lesson 04.10):

```rust
fn parse_port(text: &str) -> Result<u16, std::num::ParseIntError> {
    text.trim().parse()
}
```

> 🔒 **Security:** A program that panics on bad input can be crashed by anyone who can send it bad input.
> Treat "a crash caused by input" as a bug, and handle it with an error. Phase 10 (Lesson 10.4) is about this.

When is `unwrap` fine? When the failure would mean a bug in *your* code, not bad input: for example
`"127.0.0.1".parse::<IpAddr>().unwrap()` on a constant. Then prefer `expect("explain the reason")`, so the
panic message tells the reader which assumption broke.

### 3.3 `Rc<RefCell<T>>` everywhere

`Rc<RefCell<T>>` (Lessons 05.9 and 06.4) gives you shared, mutable data, and it moves the borrow checking
from compile time to **run time**. Beginners meet it as "the thing that makes the compiler stop complaining".
The complaint was a design signal, and now the problem arrives later, as a crash:

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let counter = Rc::new(RefCell::new(0));
    let first = counter.borrow_mut();
    let mut second = counter.borrow_mut();
    *second += 1;
    println!("{first} {second}");
}
```

```text

thread 'main' (4616312) panicked at src/main.rs:7:30:
RefCell already borrowed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Two mutable borrows at the same time is exactly what the compiler would have rejected. With `RefCell`, it
compiles and fails while the program is running, maybe only on one rare path. Reach for `Rc<RefCell<T>>` only
when you really have shared ownership that cannot be expressed with a single owner and references, such as a
graph where nodes point to each other. For most programs the better design is one owner, with `&mut`
passed to the functions that change the value.

### 3.4 "Inheritance" through `Deref`

Rust has no inheritance (Lesson 08.4). Some people imitate it: `Employee` implements `Deref` with
`Target = Person`, so that `employee.greet()` finds `Person::greet`:

```rust
use std::ops::Deref;

struct Person {
    name: String,
}

impl Person {
    fn greet(&self) -> String {
        format!("Hello, I am {}", self.name)
    }
}

struct Employee {
    person: Person,
}

impl Deref for Employee {
    type Target = Person;
    fn deref(&self) -> &Person {
        &self.person
    }
}

trait Describe {
    fn describe(&self) -> String;
}

impl Describe for Person {
    fn describe(&self) -> String {
        format!("person {}", self.name)
    }
}

fn print_description<T: Describe>(item: &T) {
    println!("{}", item.describe());
}

fn main() {
    let boss = Employee { person: Person { name: String::from("Ana") } };
    println!("{}", boss.greet());
    print_description(&boss);
}
```

```text
error[E0277]: the trait bound `Employee: Describe` is not satisfied
  --> src/main.rs:41:23
   |
41 |     print_description(&boss);
   |     ----------------- ^^^^^ unsatisfied trait bound
   |     |
   |     required by a bound introduced by this call
   |
help: the trait `Describe` is not implemented for `Employee`
...
```

**What happens:**

1. `boss.greet()` compiles: when a method is not found on `Employee`, the compiler tries `*boss`, a `Person`.
2. But `Deref` only provides *method lookup*. `Employee` is **not** a `Person` and does not get its traits,
   so `print_description(&boss)` fails. Also, every public method that `Person` gets in the future silently
   becomes a method of `Employee`.

`Deref` is meant for smart pointers (`Box`, `Rc`, `String` to `str`), where one value *points to* another. For
"has a", use **composition** (Lesson 08.4): a private field, plus the methods you choose to offer.

### 3.5 `#![deny(warnings)]` in the source code

`#![deny(warnings)]` at the top of `main.rs` turns every compiler warning into an error. It sounds strict and
good. With `let unused = 5;` in `main`, the build fails like this:

```text
error: unused variable: `unused`
 --> src/main.rs:4:9
  |
4 |     let unused = 5;
  |         ^^^^^^ help: if this is intentional, prefix it with an underscore: `_unused`
  |
note: the lint level is defined here
 --> src/main.rs:1:9
  |
1 | #![deny(warnings)]
  |         ^^^^^^^^
```

The problem is not today's code but **tomorrow's compiler**. Every Rust release can add warnings. A crate with
`deny(warnings)` that built yesterday may stop building after you, or a user of your library, update Rust,
although the code did not change. Keep warnings as warnings in the source, and make them errors only in the
quality gate: `cargo clippy -- -D warnings` (Lesson 05.6), and later in CI (Continuous Integration: a server that runs your checks automatically; Phase 09). Your own check is
strict; nobody else's build breaks.

## 4. Let Clippy find the easy ones

Many smells are detected by **Clippy**. Put this working but unidiomatic code into `src/main.rs`:

```rust
fn total(prices: &Vec<u32>) -> u32 {
    let mut sum = 0;
    for i in 0..prices.len() {
        sum += prices[i];
    }
    sum
}

fn label(a: u32, b: u32, c: u32, d: u32, e: u32, f: u32, g: u32, h: u32) -> u32 {
    a + b + c + d + e + f + g + h
}

fn main() {
    println!("{}", total(&vec![1, 2, 3]));
    println!("{}", label(1, 2, 3, 4, 5, 6, 7, 8));
}
```

Run `cargo clippy`. The real output (shortened):

```text
warning: writing `&Vec` instead of `&[_]` involves a new object where a slice will do
 --> src/main.rs:1:18
  |
1 | fn total(prices: &Vec<u32>) -> u32 {
  |                  ^^^^^^^^^
  = note: `#[warn(clippy::ptr_arg)]` on by default
help: change this to
  |
1 - fn total(prices: &Vec<u32>) -> u32 {
1 + fn total(prices: &[u32]) -> u32 {

warning: the loop variable `i` is only used to index `prices`
 --> src/main.rs:3:14
  |
3 |     for i in 0..prices.len() {
  |              ^^^^^^^^^^^^^^^
  = note: `#[warn(clippy::needless_range_loop)]` on by default

warning: this function has too many arguments (8/7)
  --> src/main.rs:10:1
   |
10 | fn label(a: u32, b: u32, c: u32, d: u32, e: u32, f: u32, g: u32, h: u32) -> u32 {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   = note: `#[warn(clippy::too_many_arguments)]` on by default
```

**What happens:**

1. `&Vec<u32>` (and `&String`) as a parameter is needlessly narrow. A `&[u32]` accepts a `Vec`, an array or part
   of a vector; a `&str` accepts a `String` or a literal. This is the same lesson as the `clone` example: take the
   most general *borrowed* type that does the job.
2. `for i in 0..prices.len()` with `prices[i]` is an index loop. An iterator (`prices.iter().sum()`) cannot go
   out of bounds and tells the reader what you want.
3. Eight parameters is the "long parameter list" smell. Clippy's default limit is seven.

Not every smell has a lint, and not every lint fits your case. How to choose lint levels, `allow` a lint
with a reason and configure Clippy for a project is in
[Lesson 08.11](11-conventions-naming-clippy-and-api-guidelines.md).

## 5. What to do when you smell something

1. **Name the problem.** "This function is long" describes; "I cannot test the discount rule without building
   a whole order" is a *problem*.
2. **Check the cost of leaving it.** Code that nobody changes can stay ugly. Fix the smells in code that changes
   every week.
3. **Put tests around the behaviour first** (characterization tests, Lesson 08.13), then fix **one smell at a
   time** in a small commit.
4. **Do not trade one smell for another.** Replacing three duplicated lines by a four-level abstraction is not
   progress (Lesson 08.1).

> 🧪 **Testing:** Smells turn into tests easily. For the shipping example,
> `assert_eq!(Zone::from_country_code("de"), None)` pins down the behaviour that was a silent bug before.

## Common mistakes

1. **Treating every smell as a bug.** A `match` with ten arms may be the clearest code possible. Name the
   problem first.
2. **"Fixing" a smell without tests.** You cannot know that the behaviour stayed the same.
3. **Silencing the tool instead of reading it.** `#[allow(clippy::too_many_arguments)]` added only to remove
   the warning hides the signal. If you allow a lint, write a comment saying why.
4. **Over-fixing.** Turning every `bool` and `String` into a type, a trait and a module makes a small program
   harder to read. Fix what hurts.

## Try it yourself

1. Put the code of Section 4 into `code/p08-smells/src/main.rs`; `cargo clippy` shows three warnings.
2. Change `total` to take `&[u32]` with the body `prices.iter().sum()`, and call it as `total(&[1, 2, 3])`.
3. Change `label` to `fn label(values: [u32; 8]) -> u32` with the body `values.iter().sum()`, and call it as
   `label([1, 2, 3, 4, 5, 6, 7, 8])`.
4. Run `cargo clippy -- -D warnings` (only `Checking` and `Finished` appear), then `cargo run`:

```text
6
36
```

## Exercises

### Exercise 1 — Name the smell (easy)

For each fragment, name the smell and say in one sentence what concrete problem it causes.

1. `connect("db", 5432, true, false, true)` for `fn connect(host: &str, port: u16, tls: bool, retry: bool, verbose: bool)`.
2. `if user.role == "admin" || user.role == "Admin"` repeated in six files.
3. A `struct Shop` with fields `customers`, `invoices`, `theme_color`, `mail_password`, `window_size`.
4. `let timeout = 30 * 60 * 1000;` in the middle of a function.

<details>
<summary>Hint</summary>

Look at the table in Section 2.2. Ask: "what happens when someone changes it, or makes a typo?"

</details>

<details>
<summary>Answer</summary>

1. *Boolean blindness* and a *long parameter list*: nobody knows which `true` is `tls`, and swapping two flags
   compiles. Fix: an options struct or an `enum` per flag.
2. *Stringly-typed data* and *duplicated code*: the two spellings show the strings already disagree. Fix:
   `enum Role` with one `is_admin` method.
3. A *god type*: business data, settings and screen state are mixed, so a test of invoices needs a
   mail password and a window size. Fix: split by responsibility.
4. A *magic number*: the reader must calculate that it means 30 minutes in milliseconds. Fix:
   `const SESSION_TIMEOUT: Duration = Duration::from_secs(30 * 60);`.

</details>

### Exercise 2 — Replace the strings with an enum, and test it (medium)

A support-ticket program stores `status: String` with the values `"open"`, `"pending"` and `"closed"`, and a
function `can_edit(status: &str) -> bool` that is `false` only for `"closed"`. Write an enum `Status` with
`Status::parse(text) -> Result<Status, UnknownStatus>` and a method `can_edit`. Write tests: known words parse,
a typo such as `"Open"` is an error, and a closed ticket cannot be edited.

<details>
<summary>Hint</summary>

Let `UnknownStatus` be a tuple struct holding the bad text. Derive `Debug` and `PartialEq` on both types so that
`assert_eq!` works.

</details>

<details>
<summary>Answer</summary>

In a library package (`cargo new --lib code/p08-tickets`), `src/lib.rs`:

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Status {
    Open,
    Pending,
    Closed,
}

#[derive(Debug, PartialEq)]
pub struct UnknownStatus(pub String);

impl Status {
    pub fn parse(text: &str) -> Result<Status, UnknownStatus> {
        match text {
            "open" => Ok(Status::Open),
            "pending" => Ok(Status::Pending),
            "closed" => Ok(Status::Closed),
            other => Err(UnknownStatus(other.to_string())),
        }
    }

    pub fn can_edit(self) -> bool {
        self != Status::Closed
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_known_words() {
        assert_eq!(Status::parse("open"), Ok(Status::Open));
        assert_eq!(Status::parse("closed"), Ok(Status::Closed));
    }

    #[test]
    fn rejects_typos() {
        assert_eq!(Status::parse("Open"), Err(UnknownStatus("Open".to_string())));
    }

    #[test]
    fn closed_cannot_be_edited() {
        assert!(Status::Open.can_edit());
        assert!(!Status::Closed.can_edit());
    }
}
```

Why it works: the only place where text becomes a status is `parse`. After that, the compiler knows a `Status`
has exactly three values. `can_edit` uses `!=`, which works because we derived `PartialEq`.

</details>

### Exercise 3 — Replace `Deref` by composition (hard)

Fix the `Employee` of Section 3.4 without `Deref`. `Employee` keeps a private `Person`, offers `greet()` and
a read-only `person()`. Write a test for the greeting.

<details>
<summary>Hint</summary>

`fn greet(&self) -> String { self.person.greet() }` is a *delegating* method. Add `Employee::new(name)`, since a
private field cannot be filled from outside.

</details>

<details>
<summary>Answer</summary>

```rust
pub struct Person {
    pub name: String,
}

impl Person {
    pub fn greet(&self) -> String {
        format!("Hello, I am {}", self.name)
    }
}

pub struct Employee {
    person: Person,
}

impl Employee {
    pub fn new(name: &str) -> Employee {
        Employee { person: Person { name: name.to_string() } }
    }
    pub fn person(&self) -> &Person {
        &self.person
    }
    pub fn greet(&self) -> String {
        self.person.greet()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn employee_greets_through_its_person() {
        let e = Employee::new("Ana");
        assert_eq!(e.greet(), "Hello, I am Ana");
        assert_eq!(e.person().name, "Ana");
    }
}
```

Why it works: `Employee` *has a* `Person` and says so. Only the methods you write are its interface; if you
want `Employee` to implement `Describe`, you write that `impl` yourself, often delegating to the inner `Person`.

</details>

## Quiz

1. What is the difference between a code smell and a bug?

<details>
<summary>Answer</summary>

A bug is wrong behaviour. A smell is a sign that the *design* may be a problem. Smelly code can be correct today;
the cost shows up when you change, test or read it.

</details>

2. When is `unwrap()` acceptable?

<details>
<summary>Answer</summary>

In tests and examples, and when a failure would mean a bug in your own code (a constant that cannot fail to
parse); then `expect("reason")` is better. Never for data from outside the program.

</details>

3. In the shipping example, why did the typo `"de"` give a wrong price without any error, and what removes the problem?

<details>
<summary>Answer</summary>

The country was a `&str`, and every text except two exact values counted as "foreign". Converting the text once, in a
function that returns `Option<Zone>` (or a `Result`), turns the typo into a visible `None` or an error.

</details>

4. Why is implementing `Deref` to get "inheritance" an anti-pattern?

<details>
<summary>Answer</summary>

`Deref` only changes method lookup. The outer type does not become the inner type, so trait bounds and generic
code do not work with it, and all public methods of the inner type leak into the outer one. Use composition.

</details>

## Summary

- A **code smell** is a hint that the design may have a problem; an **anti-pattern** is a popular recipe that
  usually makes things worse. Always name the concrete problem before changing code.
- The common smells: long functions, magic numbers, stringly-typed data, boolean parameters, long parameter
  lists, primitive obsession, duplicated code, god types, feature envy and shotgun surgery.
- Replace stringly-typed data with an `enum` that has one `parse` function; replace magic numbers with named
  constants.
- Rust-specific anti-patterns: needless `clone` (fix the signature), `unwrap` on outside data (return
  `Result`), `Rc<RefCell<T>>` everywhere (prefer one owner and `&mut`), `Deref` as inheritance (use
  composition) and `#![deny(warnings)]` in source (use `-D warnings` in the gate).
- Take `&str` and `&[T]` in parameters, not `&String` and `&Vec<T>`.
- Clippy detects many smells; fix smells in small steps, with tests in place.

## Sources and further reading

- [Rust Design Patterns — Anti-patterns](https://rust-unofficial.github.io/patterns/anti_patterns/index.html) — the community book's list of Rust anti-patterns, including clone to satisfy the borrow checker, `deny(warnings)` and `Deref` polymorphism.
- [Clippy documentation](https://doc.rust-lang.org/stable/clippy/) — how to run and configure Clippy.
- [The Clippy lint list](https://rust-lang.github.io/rust-clippy/stable/index.html) — every lint, with an explanation and an example of what it flags.
- [The Rust Programming Language — Chapter 9: Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html) — when to panic and when to return `Result` (including the section on when `unwrap` is fine).
- [The Rust Programming Language — Chapter 15: Smart Pointers](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html) — `Deref`, `Rc` and `RefCell` as they are meant to be used.

---

[🏠 Start](../START-HERE.md) · [← 08.9 Strategy and dependency injection](09-strategy-and-dependency-injection.md) · [📋 Phase 08](README.md) · [08.11 Conventions: naming, Clippy and the Rust API Guidelines →](11-conventions-naming-clippy-and-api-guidelines.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
