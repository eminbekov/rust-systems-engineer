<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.16 Project: refactor a messy program](16-project-refactor-a-messy-program.md) · [📋 Phase 08](README.md) · [Phase 09 — Testing, debugging and benchmarking →](../09-testing-debugging-benchmarking/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/17-checkpoint.md) <!-- nav -->

# Lesson 08.17 — Checkpoint

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 17 of 17 · ⏱ about 135 minutes

## What you will learn

- Whether you can explain the design principles of this phase (KISS, DRY, YAGNI, cohesion and coupling, SOLID, composition) in your own words.
- Whether you can choose and write the patterns: parse-don't-validate types, builders, RAII guards, state machines and typestate, strategies and dependency injection.
- Whether you can recognise smells, follow the conventions, organise a project, refactor under tests, review code, and keep dependencies pointing inwards.
- Which lessons to repeat before you start Phase 09.

## Before you start

- You finished [Lesson 08.16 — Project: refactor a messy program](16-project-refactor-a-messy-program.md), and your branches are committed.
- Do the quiz **without** looking at the lessons. Write your answer first, then open the hidden answer.
- Do the practical tasks in a new package: `cargo new --lib code/p08-checkpoint`. Put each task in its own file (`src/port.rs`, `src/transaction.rs`, `src/discount.rs`, `src/label.rs`) and declare the modules in `src/lib.rs` with `pub mod port;` and so on.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A lesson with no new material: a self-test and practical tasks that show what you remember and can do. |
| **self-test** | Answering questions from memory before you check the answer. It helps learning more than rereading. |
| **KISS** (Keep It Simple, Stupid) | Choose the simplest solution that works. |
| **DRY** (Don't Repeat Yourself) | Every piece of knowledge has one home. |
| **YAGNI** (You Aren't Gonna Need It) | Do not build for an imagined future. |
| **SOLID** | Five design principles named by their initials: Single responsibility, Open/closed, Liskov substitution, Interface segregation, Dependency inversion. |
| **RAII** (Resource Acquisition Is Initialization) | A value takes a resource when it is created and gives it back when it is dropped. |
| **API** (Application Programming Interface) | The items that a crate offers to other code. |
| **TCP** (Transmission Control Protocol) | The reliable, ordered network connection on which web traffic runs. |
| **LLDB** | The debugger of the LLVM compiler project (the name comes from LLVM, which once stood for Low Level Virtual Machine); Phase 09 uses it. |

## Self-test quiz

### Principles

1. What do KISS, DRY and YAGNI each say, and what is the "rule of three"?

<details>
<summary>Answer</summary>

KISS: choose the simplest solution that works. DRY: every piece of *knowledge* has one home (not "never repeat any text"). YAGNI: do not build for an imagined future. The rule of three: tolerate a first duplication; extract an abstraction when the same thing appears a third time, because by then you can see what really varies.

</details>

2. What are cohesion and coupling, and which direction do you want for each?

<details>
<summary>Answer</summary>

Cohesion is how well the contents of one part belong together; coupling is how much parts depend on each other's insides. Aim for high cohesion and low coupling. Private fields (the compiler enforces them with error E0616) and narrow parameter types such as `&[u32]` reduce coupling.

</details>

3. A trait has a method, and one implementation fills it with `unimplemented!()`. Which SOLID letter is broken, and what is the fix?

<details>
<summary>Answer</summary>

**I**, interface segregation. Split the trait into smaller ones, so that a type implements only what it can really do; this turns a run-time panic into a compile-time error.

</details>

4. For "types of shape that users may extend", do you use an `enum` or a trait? And for "the five states of an order"?

<details>
<summary>Answer</summary>

An open set that others extend: a trait (open/closed principle). A closed set that you control, such as order states: an `enum`, because a `match` without `_` makes the compiler list every place to update when you add a variant.

</details>

5. Why is implementing `Deref` to get "inheritance" an anti-pattern, and what do you use instead?

<details>
<summary>Answer</summary>

`Deref` only changes method lookup: the outer type does not become the inner type, trait bounds do not see it, and every public method of the inner type leaks out. Use composition (a field) and delegating methods, a trait for shared behaviour, or a wrapper.

</details>

### Types and patterns

6. What does "parse, don't validate" mean? Give an example type.

<details>
<summary>Answer</summary>

Check the input once at the boundary and return a stricter type that carries the proof, so that the rest of the program cannot receive an invalid value. Example: `Port::new(u16) -> Result<Port, PortError>` with a private field, or `Title::new(&str)`. Implement `FromStr`/`TryFrom` for the standard interfaces.

</details>

7. When is a builder better than `Default` with struct-update syntax (`..Default::default()`)?

<details>
<summary>Answer</summary>

When some fields are required, when there are rules to check (so `build()` returns a `Result`), when values are computed, or when settings are repeated. For a small struct with good defaults and no rules, `Default` is enough.

</details>

8. What is RAII, and in which cases does `Drop` *not* run?

<details>
<summary>Answer</summary>

Resource Acquisition Is Initialization: a value takes a resource when created and gives it back in `Drop`, so cleanup is written once and cannot be forgotten. `Drop` does not run on `std::process::exit`, on abort, or after `mem::forget`; so it is no place for must-happen logic.

</details>

9. When do you choose an `enum` state machine, and when typestate?

<details>
<summary>Answer</summary>

An `enum` when the state comes from run-time data (input, a file, a network message). Typestate (one type per state, transitions that take `self`) when the order of calls is written in your own code, so that wrong orders become compile errors.

</details>

10. Name four ways to express a strategy in Rust, and the main trade-off of each.

<details>
<summary>Answer</summary>

An `enum` (closed set, simple, all variants known); a generic parameter `P: Trait` (callers choose at compile time, fast, but the type parameter spreads); `Box<dyn Trait>` (chosen at run time, simple types, a small cost per call); a closure (one function, lightest). 

</details>

11. What is dependency injection, and where is the composition root?

<details>
<summary>Answer</summary>

A type receives the things it depends on (as traits) through `new` or parameters instead of creating them itself. The composition root is the one place, usually `main`, that chooses the real parts and wires them together. In tests you inject hand-written fakes.

</details>

### Quality and conventions

12. What is the difference between a code smell and an anti-pattern? Name three Rust-specific anti-patterns.

<details>
<summary>Answer</summary>

A smell is a hint of a possible design problem (name the concrete problem before you change anything). An anti-pattern is a popular recipe that usually makes things worse. Rust examples: `clone` just to silence the borrow checker, `unwrap` on outside data, `Rc<RefCell<T>>` everywhere, `Deref` as inheritance, `#![deny(warnings)]` in source.

</details>

13. What do the prefixes `as_`, `to_` and `into_` promise, and how do you name a getter?

<details>
<summary>Answer</summary>

`as_`: a free borrowed view. `to_`: a new value made from a borrowed one (can cost work). `into_`: consumes `self`. A getter has the name of the field without `get_`: `name()`.

</details>

14. What is the difference between `#[allow(lint)]` and `#[expect(lint, reason = "...")]`, and where do you set the lint policy of a project?

<details>
<summary>Answer</summary>

`expect` also warns when the lint stops firing, so stale exceptions disappear, and it carries a reason. The project policy goes in the `[lints]` table of `Cargo.toml` (or `[workspace.lints]` with `[lints] workspace = true` in each member). The quality gate is `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`.

</details>

15. What does a Cargo workspace share, and what does Cargo refuse to build?

<details>
<summary>Answer</summary>

One `Cargo.lock`, one `target` folder, and (through inheritance) versions, dependencies and lint settings. Cargo refuses to build a dependency cycle between crates, so the direction of dependencies is enforced.

</details>

### Refactoring, architecture, operations

16. What is a refactoring, and what must you have before you start? What is a characterization test?

<details>
<summary>Answer</summary>

A change to the structure of code without changing its observable behaviour, in small steps with tests green after each one. You need a safety net of tests; if there is none, you write characterization (golden) tests that record what the code does now, including odd results.

</details>

17. In what order does a reviewer look at a change, and what do the labels `blocking:`, `suggestion:` and `nit:` mean?

<details>
<summary>Answer</summary>

Design, correctness, tests, readability, and style last (the tools check formatting and lints). `blocking:` must change before the merge; `suggestion:` is a good idea that is the author's choice; `nit:` is a small matter of taste that may be ignored.

</details>

18. State the dependency rule of ports-and-adapters architecture, and the difference between a port and an adapter.

<details>
<summary>Answer</summary>

Source-code dependencies point towards the domain: the core never imports adapters or technology. A port is a trait owned by the core describing what it needs; an adapter is a type in the outer layer that implements a port (driven) or calls the core (driving). `main` wires them together.

</details>

19. Name four of the twelve factors with their Rust form.

<details>
<summary>Answer</summary>

For example: III config in the environment (read once into a typed `Config`, never print secrets); XI logs to standard output; VI stateless processes (no important data in memory); VII the program binds its own port (`TcpListener::bind`); II dependencies in `Cargo.toml` and a committed `Cargo.lock` (`--locked`); IX disposability (start fast, handle `SIGTERM`).

</details>

20. In the project, why did you keep the bug in the summary and use a `Vec` instead of a `HashMap`?

<details>
<summary>Answer</summary>

A refactoring must not change behaviour, so the bug is kept and fixed later in a separate commit that also updates the golden files. A `HashMap` iterates in an unspecified order, which would make the summary differ between runs; a `Vec` of pairs keeps the order of first appearance.

</details>

## Practical tasks

Each task has its tests in the answer. Write your own tests first (Lesson 05.6), then compare. Run `cargo test` and `cargo clippy --all-targets -- -D warnings` for every task.

### Task 1 — A type that cannot be wrong (Lessons 08.5 and 08.11)

In `src/port.rs`, write `Port`, a TCP port number from 1 to 65535. `Port::new(u16)` rejects 0; `"8080".parse::<Port>()` works (and ignores spaces around the number); text that is not a number or is larger than 65535 is `PortError::NotANumber`, and 0 is `PortError::Zero`. The field must be private, there must be a `number()` getter, and `PortError` implements `Display` and `std::error::Error`. Write tests.

<details>
<summary>Hint</summary>

Implement `std::str::FromStr` with `type Err = PortError;`. Parse as `u16` first (a number above 65535 fails to parse), then call `Port::new`. Remember: no `get_` prefix on the getter.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fmt;
use std::str::FromStr;

/// A TCP port that a service may listen on: 1 to 65535.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Port(u16);

/// Why a port could not be created.
#[derive(Debug, PartialEq, Eq)]
pub enum PortError {
    /// The text is not a number from 0 to 65535.
    NotANumber,
    /// Port 0 is not allowed here.
    Zero,
}

impl fmt::Display for PortError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            PortError::NotANumber => write!(f, "not a number from 1 to 65535"),
            PortError::Zero => write!(f, "port 0 is not allowed"),
        }
    }
}

impl std::error::Error for PortError {}

impl Port {
    /// Creates a port.
    ///
    /// # Errors
    ///
    /// Returns [`PortError::Zero`] for 0.
    pub fn new(number: u16) -> Result<Port, PortError> {
        if number == 0 {
            Err(PortError::Zero)
        } else {
            Ok(Port(number))
        }
    }

    /// The port number.
    #[must_use]
    pub fn number(self) -> u16 {
        self.0
    }
}

impl FromStr for Port {
    type Err = PortError;

    fn from_str(text: &str) -> Result<Port, PortError> {
        let number: u16 = text.trim().parse().map_err(|_| PortError::NotANumber)?;
        Port::new(number)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_ports() {
        assert_eq!("8080".parse::<Port>().unwrap().number(), 8080);
        assert_eq!(" 443 ".parse::<Port>().unwrap().number(), 443);
    }

    #[test]
    fn rejects_zero_text_and_big_numbers() {
        assert_eq!("0".parse::<Port>(), Err(PortError::Zero));
        assert_eq!("http".parse::<Port>(), Err(PortError::NotANumber));
        assert_eq!("70000".parse::<Port>(), Err(PortError::NotANumber));
        assert_eq!(PortError::Zero.to_string(), "port 0 is not allowed");
    }
}
```

Why it works: the only ways to get a `Port` are `Port::new` and `FromStr`, and both reject 0, so every `Port` in the program is valid. `u16` already rejects numbers above 65535 and negative numbers when parsing.

</details>

### Task 2 — A guard that rolls back (Lesson 08.7)

In `src/transaction.rs`, write `Transaction<'a>`, a guard that holds `&'a RefCell<Vec<String>>` (a log). `Transaction::begin(log)` records `"begin"`. `commit(self)` records `"commit"`. If the guard is dropped without `commit`, it records `"rollback"`. Write the function `transfer(log, fail: bool) -> Result<(), String>` that begins a transaction, returns `Err` early when `fail` is true, and commits otherwise. Test both paths.

<details>
<summary>Hint</summary>

Keep a `committed: bool` field. `commit` takes `self` (so it can be called only once), records the word, sets the flag and lets the guard drop. In `Drop::drop`, record `"rollback"` only if the flag is false. The early `return` runs `Drop` automatically.

</details>

<details>
<summary>Answer</summary>

```rust
use std::cell::RefCell;

/// A guard: if it is dropped without `commit`, it records a rollback.
pub struct Transaction<'a> {
    log: &'a RefCell<Vec<String>>,
    committed: bool,
}

impl<'a> Transaction<'a> {
    /// Starts a transaction and records `begin`.
    pub fn begin(log: &'a RefCell<Vec<String>>) -> Transaction<'a> {
        log.borrow_mut().push(String::from("begin"));
        Transaction {
            log,
            committed: false,
        }
    }

    /// Finishes the transaction successfully.
    pub fn commit(mut self) {
        self.log.borrow_mut().push(String::from("commit"));
        self.committed = true;
    }
}

impl Drop for Transaction<'_> {
    fn drop(&mut self) {
        if !self.committed {
            self.log.borrow_mut().push(String::from("rollback"));
        }
    }
}

/// Pretends to move money; fails when `fail` is true.
///
/// # Errors
///
/// Returns an error when `fail` is true.
pub fn transfer(log: &RefCell<Vec<String>>, fail: bool) -> Result<(), String> {
    let transaction = Transaction::begin(log);
    if fail {
        return Err(String::from("insufficient funds"));
    }
    transaction.commit();
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_successful_transfer_commits() {
        let log = RefCell::new(Vec::new());
        transfer(&log, false).unwrap();
        assert_eq!(*log.borrow(), ["begin", "commit"]);
    }

    #[test]
    fn a_failed_transfer_rolls_back() {
        let log = RefCell::new(Vec::new());
        assert!(transfer(&log, true).is_err());
        assert_eq!(*log.borrow(), ["begin", "rollback"]);
    }
}
```

Why it works: on the early `return Err(...)` the variable `transaction` goes out of scope, `Drop` runs and logs the rollback; on success `commit(self)` consumes the guard after setting the flag, so `Drop` stays silent. Cleanup is written once and cannot be forgotten.

</details>

### Task 3 — A strategy, injected, and a fake (Lessons 08.3 and 08.9)

In `src/discount.rs`, write a trait `DiscountPolicy` with `fn discount_cents(&self, total_cents: u64) -> u64`, two policies (`NoDiscount` and `PercentOff(u64)`, rounding the discount down and treating more than 100 percent as 100), and `Cart<P: DiscountPolicy>` with `new(policy)`, `add(price_cents)` and `total()`. Test the cart with each policy, and with a hand-written fake policy that always returns the same discount.

<details>
<summary>Hint</summary>

`total()` is the sum of the prices minus `self.policy.discount_cents(sum)`. The fake lives in the `tests` module only: a tuple struct `Fixed(u64)` that implements the trait.

</details>

<details>
<summary>Answer</summary>

```rust
/// A rule that decides how many cents are taken off a total.
pub trait DiscountPolicy {
    /// The discount for `total_cents`, never more than the total.
    fn discount_cents(&self, total_cents: u64) -> u64;
}

/// No discount at all.
pub struct NoDiscount;

impl DiscountPolicy for NoDiscount {
    fn discount_cents(&self, _total_cents: u64) -> u64 {
        0
    }
}

/// A percentage off, rounded down.
pub struct PercentOff(pub u64);

impl DiscountPolicy for PercentOff {
    fn discount_cents(&self, total_cents: u64) -> u64 {
        total_cents * self.0.min(100) / 100
    }
}

/// A cart of prices in cents. The policy is injected.
pub struct Cart<P: DiscountPolicy> {
    prices: Vec<u64>,
    policy: P,
}

impl<P: DiscountPolicy> Cart<P> {
    /// Creates an empty cart that uses `policy`.
    pub fn new(policy: P) -> Cart<P> {
        Cart {
            prices: Vec::new(),
            policy,
        }
    }

    /// Adds an item.
    pub fn add(&mut self, price_cents: u64) {
        self.prices.push(price_cents);
    }

    /// The total after the discount.
    pub fn total(&self) -> u64 {
        let sum: u64 = self.prices.iter().sum();
        sum - self.policy.discount_cents(sum)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    /// A fake policy for tests: always the same discount.
    struct Fixed(u64);

    impl DiscountPolicy for Fixed {
        fn discount_cents(&self, _total_cents: u64) -> u64 {
            self.0
        }
    }

    #[test]
    fn no_discount_charges_the_sum() {
        let mut cart = Cart::new(NoDiscount);
        cart.add(250);
        cart.add(400);
        assert_eq!(cart.total(), 650);
    }

    #[test]
    fn percent_off_rounds_the_discount_down() {
        let mut cart = Cart::new(PercentOff(15));
        cart.add(999);
        assert_eq!(cart.total(), 999 - 149);
    }

    #[test]
    fn a_fake_policy_makes_the_test_independent_of_the_rule() {
        let mut cart = Cart::new(Fixed(100));
        cart.add(1000);
        assert_eq!(cart.total(), 900);
    }
}
```

Why it works: `Cart` does not know which rule it uses; the caller (the composition root) injects it. The third test shows the benefit: the cart's addition and subtraction are tested without depending on the percentage rule.

</details>

### Task 4 — Refactor under test (Lessons 08.10 and 08.13)

Here is a smelly function (put it in `src/label.rs`):

```rust
pub fn label(temp: i32, unit: &str) -> String {
    let mut text = String::new();
    if unit == "f" {
        if temp <= 32 {
            text = format!("{} F: freezing", temp);
        } else if temp >= 212 {
            text = format!("{} F: boiling", temp);
        } else {
            text = format!("{} F: liquid", temp);
        }
    } else {
        if temp <= 0 {
            text = format!("{} C: freezing", temp);
        } else if temp >= 100 {
            text = format!("{} C: boiling", temp);
        } else {
            text = format!("{} C: liquid", temp);
        }
    }
    text
}
```

First write a characterization test with a table that covers both units, all three words, the borders, and an unknown unit such as `"k"` (what does the function do with it?). Then refactor: an `enum Unit`, named constants and no duplicated text, keeping the signature and the test unchanged. Check with `cargo clippy` before and after.

<details>
<summary>Hint</summary>

Borders: 0 and 1, 99 and 100 for Celsius; 32 and 33, 211 and 212 for Fahrenheit. Run the function to see what it does with `"k"`: the `else` branch means anything that is not `"f"` is Celsius, and the test must say so. Clippy reports `value assigned to 'text' is never read` on the original.

</details>

<details>
<summary>Answer</summary>

```rust
const FREEZING_C: i32 = 0;
const BOILING_C: i32 = 100;
const FREEZING_F: i32 = 32;
const BOILING_F: i32 = 212;

#[derive(Clone, Copy)]
enum Unit {
    Celsius,
    Fahrenheit,
}

impl Unit {
    fn parse(text: &str) -> Unit {
        if text == "f" {
            Unit::Fahrenheit
        } else {
            Unit::Celsius
        }
    }

    fn symbol(self) -> &'static str {
        match self {
            Unit::Celsius => "C",
            Unit::Fahrenheit => "F",
        }
    }

    fn limits(self) -> (i32, i32) {
        match self {
            Unit::Celsius => (FREEZING_C, BOILING_C),
            Unit::Fahrenheit => (FREEZING_F, BOILING_F),
        }
    }
}

/// Describes a temperature, e.g. `"-5 C: freezing"`.
#[must_use]
pub fn label(temp: i32, unit: &str) -> String {
    let unit = Unit::parse(unit);
    let (freezing, boiling) = unit.limits();
    let word = if temp <= freezing {
        "freezing"
    } else if temp >= boiling {
        "boiling"
    } else {
        "liquid"
    };
    format!("{temp} {}: {word}", unit.symbol())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn pins_down_the_old_behaviour() {
        let cases = [
            // (temp, unit, expected)
            (-5, "c", "-5 C: freezing"),
            (0, "c", "0 C: freezing"),
            (1, "c", "1 C: liquid"),
            (99, "c", "99 C: liquid"),
            (100, "c", "100 C: boiling"),
            (32, "f", "32 F: freezing"),
            (33, "f", "33 F: liquid"),
            (212, "f", "212 F: boiling"),
            (50, "k", "50 C: liquid"),
        ];
        for (temp, unit, expected) in cases {
            assert_eq!(label(temp, unit), expected);
        }
    }
}
```

Why it works: the table pins down the borders and the unknown unit (`"k"` is treated as Celsius, as before), and the same table passes on the original and on the refactored version. The words and the limits now exist once; to add Kelvin you add an `enum` variant and the compiler shows every `match` that needs a new arm.

</details>

## If you got something wrong

| Topic | Lesson to repeat |
|---|---|
| KISS, DRY, YAGNI, the rule of three | [Lesson 08.1 — Good design: KISS, DRY and YAGNI](01-good-design-kiss-dry-yagni.md) |
| Cohesion, coupling, separation of concerns | [Lesson 08.2 — Cohesion, coupling and separation of concerns](02-cohesion-coupling-and-separation-of-concerns.md) |
| The five SOLID principles | [Lesson 08.3 — The SOLID principles](03-solid-principles.md) |
| Composition, wrappers, `Deref` is not inheritance | [Lesson 08.4 — Composition over inheritance](04-composition-over-inheritance.md) |
| Parse, don't validate; newtypes; invalid states | [Lesson 08.5 — Type-driven design](05-type-driven-design.md) |
| Builders, `Default`, required fields | [Lesson 08.6 — The builder pattern](06-builder-pattern.md) |
| RAII, `Drop`, guards | [Lesson 08.7 — RAII and guards](07-raii-and-guards.md) |
| State machines, typestate | [Lesson 08.8 — State machines and the typestate pattern](08-state-machines-and-typestate.md) |
| Strategy, dependency injection, fakes | [Lesson 08.9 — Strategy and dependency injection](09-strategy-and-dependency-injection.md) |
| Smells and Rust anti-patterns | [Lesson 08.10 — Anti-patterns and code smells](10-anti-patterns-and-code-smells.md) |
| Naming, `rustfmt`, Clippy levels, API Guidelines | [Lesson 08.11 — Conventions: naming, Clippy and the Rust API Guidelines](11-conventions-naming-clippy-and-api-guidelines.md) |
| Package layout, workspaces, dependency cycles | [Lesson 08.12 — Project layout and workspaces](12-project-layout-and-workspaces.md) |
| Characterization tests, refactoring steps, code review | [Lesson 08.13 — Refactoring and code review](13-refactoring-and-code-review.md) |
| Layers, ports and adapters, the dependency rule | [Lesson 08.14 — Layers and hexagonal architecture](14-layers-and-hexagonal-architecture.md) |
| Configuration, logs, stateless processes, `SIGTERM` | [Lesson 08.15 — The twelve-factor app](15-the-twelve-factor-app.md) |
| Golden tests, bug-compatible refactoring | [Lesson 08.16 — Project: refactor a messy program](16-project-refactor-a-messy-program.md) |
| Writing tests, `Result` and errors (behind many tasks) | [Lesson 05.6 — Testing in depth](../05-rust-intermediate/06-testing-in-depth.md) and [Lesson 04.10 — Error handling](../04-structs-enums-collections-errors/10-error-handling.md) |
| Traits and trait objects | [Lesson 05.2 — Traits](../05-rust-intermediate/02-traits.md) and [Lesson 06.1 — Trait objects and dynamic dispatch](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) |

When you repeat a lesson, redo its exercises on a branch, and write the tests first.

## What's next

You finished Phase 08. You can now see why code hurts, name the principle that is broken, pick a pattern that fits, and change a program safely under tests. Before you go on:

1. Commit the checkpoint package (`git add code/p08-checkpoint && git commit -m "feat(p08): checkpoint tasks"`), check that `git status` is clean, and that the project branch of Lesson 08.16 is merged into `main`. Delete it when you no longer need it (`git branch -d project/p08-refactor`).
2. In your learning notes, write down three smells that you recognise in your own earlier code, and the lesson that fixes each.
3. Take a break. A day off helps your memory more than another hour of reading.

In **Phase 09 — Testing, debugging and benchmarking** you will build on this phase: the test pyramid, end-to-end tests of command-line programs (the golden tests of the project, done properly), test doubles, property-based tests, snapshot tests, test-driven development, coverage and mutation testing, fuzzing, debugging with LLDB and RustRover, logging and tracing, benchmarking, and continuous integration. Start with [Lesson 09.1 — The test pyramid and a testing strategy](../09-testing-debugging-benchmarking/01-the-test-pyramid.md).

## Sources and further reading

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — the checklist for public APIs (Lesson 08.11).
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) — idioms, patterns and anti-patterns in Rust, free online.
- [The Rust Programming Language — Chapter 18: Object-Oriented Programming Features](https://doc.rust-lang.org/book/ch18-00-oop.html) — trait objects and the state pattern.
- [The Cargo Book — Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) — sharing dependencies, versions and lints.
- [The Twelve-Factor App](https://12factor.net/) — the twelve rules in the original words.
- [Refactoring — Martin Fowler's catalogue](https://refactoring.com/catalog/) — names and steps of the classic refactorings.

---

[🏠 Start](../START-HERE.md) · [← 08.16 Project: refactor a messy program](16-project-refactor-a-messy-program.md) · [📋 Phase 08](README.md) · [Phase 09 — Testing, debugging and benchmarking →](../09-testing-debugging-benchmarking/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
