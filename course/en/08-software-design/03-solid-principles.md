<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.2 Cohesion, coupling and separation of concerns](02-cohesion-coupling-and-separation-of-concerns.md) · [📋 Phase 08](README.md) · [08.4 Composition over inheritance →](04-composition-over-inheritance.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/03-solid-principles.md) <!-- nav -->

# Lesson 08.3 — The SOLID principles

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 3 of 17 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What the five SOLID principles say, in plain words.
- How each principle looks in Rust: structs, enums, traits and generics.
- Why Rust's `enum` and `match` sometimes beat the "open for extension" idea, and when a trait is better.
- How to protect the contract of a trait with a test.
- When SOLID turns into over-engineering.

## Before you start

- You finished [Lesson 08.2 — Cohesion, coupling and separation of concerns](02-cohesion-coupling-and-separation-of-concerns.md).
- You know traits ([Lesson 05.2](../05-rust-intermediate/02-traits.md)) and trait objects with `dyn` ([Lesson 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)).
- You know enums and `match` ([Lesson 04.3](../04-structs-enums-collections-errors/03-enums.md)).

## New words

| Word | Meaning |
|---|---|
| **SOLID** | A memorable name for five design principles: **S**ingle responsibility, **O**pen/closed, **L**iskov substitution, **I**nterface segregation, **D**ependency inversion. |
| **responsibility** | A reason for a piece of code to change. "Calculates the total" and "formats HTML" are two responsibilities. |
| **extension** | Adding new behaviour to a program, for example a new kind of shape. |
| **contract** | What a trait promises to its users: what the methods return, when they may fail, what they never do. The compiler checks only the types; the rest is written in the documentation and kept by the programmer. |
| **substitution** | Replacing one implementation of a trait by another, without the code that uses the trait noticing. |
| **abstraction** (again) | Here: a trait that the high-level code uses instead of a concrete type. |
| **high-level / low-level code** | High-level code decides *what* the program does (the business rules). Low-level code does the technical details (files, screen, network). |

## 1. What SOLID is, and what it is not

SOLID is a set of five guidelines collected by the engineer Robert C. Martin in the early 2000s. They were first written for languages built around classes and inheritance, so some words sound foreign in Rust. But the *ideas* are old and useful: each describes a way in which a program becomes painful to change. In this lesson we translate every idea into Rust tools.

| Letter | Principle | Short version | Main Rust tool |
|---|---|---|---|
| **S** | Single responsibility | A part has one reason to change. | structs, modules |
| **O** | Open/closed | Add behaviour without editing old code. | traits (or enums) |
| **L** | Liskov substitution | Every implementation honours the contract. | trait contracts, tests |
| **I** | Interface segregation | Small traits; do not force unused methods. | several small traits |
| **D** | Dependency inversion | Depend on a trait, not on a concrete detail. | generics, `impl Trait`, `dyn Trait` |

Remember Lesson 08.1: these are rules of thumb, not laws. Each section ends with "when NOT".

## 2. S — Single responsibility

*A part of the program should have one reason to change.* You met this in Lesson 08.2 as cohesion. Here is an `Invoice` that knows its lines, adds them up and also writes itself as HTML (HyperText Markup Language, the language of web pages):

```rust
struct Invoice {
    lines: Vec<(String, u64)>,
}

impl Invoice {
    fn total_cents(&self) -> u64 {
        self.lines.iter().map(|(_, cents)| cents).sum()
    }

    fn to_html(&self) -> String {
        format!("<p>Total: {} cents</p>", self.total_cents())
    }
}

fn main() {
    let invoice = Invoice {
        lines: vec![("tea".to_string(), 250), ("cake".to_string(), 400)],
    };
    println!("{}", invoice.to_html());
}
```

It prints `<p>Total: 650 cents</p>`. `Invoice` has two reasons to change: the rules of money (taxes, rounding) and the look of the page (a designer wants a table instead of a list). A change of the page can break the calculation by accident, and the designer needs to read finance code.

Move the page to its own module:

```rust
struct Invoice {
    lines: Vec<(String, u64)>,
}

impl Invoice {
    fn total_cents(&self) -> u64 {
        self.lines.iter().map(|(_, cents)| cents).sum()
    }
}

mod render {
    use super::Invoice;

    pub fn html(invoice: &Invoice) -> String {
        format!("<p>Total: {} cents</p>", invoice.total_cents())
    }
}

fn main() {
    let invoice = Invoice {
        lines: vec![("tea".to_string(), 250), ("cake".to_string(), 400)],
    };
    println!("{}", render::html(&invoice));
}
```

It prints the same line. Now `Invoice` handles the money, and `render` handles the page. A `render::csv` function can be added next to `html` without touching `Invoice`.

**When NOT:** Do not make a struct per function. "One reason to change" is about *reasons*, not about the number of methods. A `Rectangle` with `area`, `perimeter` and `scale` has one reason to change (geometry) and is fine.

## 3. O — Open/closed

*Software should be open for extension, but closed for modification:* you can add new behaviour without editing code that already works. In Rust there are two ways, and the choice matters.

**Way 1: an enum.** All shapes are variants of one type, and `area` is a `match`. Suppose a new variant `Triangle` is added and somebody forgets to update `area`:

```rust
enum Shape {
    Circle(f64),
    Rect(f64, f64),
    Triangle(f64, f64), // the new variant
}

fn area(shape: &Shape) -> f64 {
    match shape {
        Shape::Circle(r) => std::f64::consts::PI * r * r,
        Shape::Rect(w, h) => w * h,
    }
}

fn main() {
    println!("{}", area(&Shape::Rect(2.0, 3.0)));
}
```

```text
error[E0004]: non-exhaustive patterns: `&Shape::Triangle(_, _)` not covered
  --> src/main.rs:8:11
   |
 8 |     match shape {
   |           ^^^^^ pattern `&Shape::Triangle(_, _)` not covered
   |
   ...

For more information about this error, try `rustc --explain E0004`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

At first sight this looks like a violation of open/closed: to add a shape, you edit `area` and every other `match`. But look at what the compiler did: it found **every place** you forgot (error E0004, *non-exhaustive patterns*). With an enum, "forgot to handle the new case" is impossible. That is a safety net, and it is exactly right when the set of cases is **closed**: you know all of them (days of the week, the states of an order, the commands of your own program).

**Way 2: a trait.** When other people must be able to add shapes, make `Shape` a trait. New shapes are new types, and no old code is edited:

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle(f64);
struct Rect(f64, f64);

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.0 * self.0
    }
}

impl Shape for Rect {
    fn area(&self) -> f64 {
        self.0 * self.1
    }
}

// Added later, maybe by somebody else: nothing above is edited.
struct Triangle(f64, f64);

impl Shape for Triangle {
    fn area(&self) -> f64 {
        self.0 * self.1 / 2.0
    }
}

fn total_area(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle(1.0)),
        Box::new(Rect(2.0, 3.0)),
        Box::new(Triangle(4.0, 5.0)),
    ];
    println!("{}", total_area(&shapes));
}
```

**What happens:**

1. `trait Shape` is the extension point: anything with an `area` is a shape.
2. `Circle` and `Rect` implement it. `total_area` works with any `Box<dyn Shape>` (Lesson 06.1).
3. `Triangle` is added below, even in another file or crate. Neither `Circle`, `Rect` nor `total_area` changed.
4. `main` adds the areas (about 3.14, 6 and 10) and prints the sum.

```text
19.141592653589793
```

How to choose:

| | Enum | Trait |
|---|---|---|
| Adding a new **type** (variant) | Edit the enum and every `match` (the compiler lists them) | Write a new type, no old edits |
| Adding a new **operation** (a new function) | Write a new function with a `match`, no old edits | Add a method to the trait and to every implementer |
| Who can add types? | Only you | Anyone, even other crates |

> 📏 **Convention:** If the cases are known and fixed, use an `enum`. If outsiders must add cases, use a trait. When in doubt, start with the enum: it is simpler (KISS, "Keep It Simple, Stupid", Lesson 08.1), and the compiler guards it.

**When NOT:** Do not create a trait "so that it is open" when nobody will ever add a case. That is flexibility built for an imagined future, which YAGNI ("You Aren't Gonna Need It") warned about in Lesson 08.1.

## 4. L — Liskov substitution

*Any implementation of a trait must be usable wherever the trait is expected, without surprises.* The principle is named after the computer scientist Barbara Liskov. The compiler checks that your method has the right *types*, but a trait has also a **contract** that types cannot express: "never panics", "result is not larger than the input", "keeps the order".

Here is a discount trait with a contract written in its documentation, and a coupon that breaks it for cheap orders:

```rust
/// Contract: the result is never higher than `cents` and the method never panics.
trait Discount {
    fn apply(&self, cents: u64) -> u64;
}

struct Percent(u64);
struct Coupon(u64);

impl Discount for Percent {
    fn apply(&self, cents: u64) -> u64 {
        cents - cents * self.0 / 100
    }
}

impl Discount for Coupon {
    fn apply(&self, cents: u64) -> u64 {
        cents - self.0
    }
}

fn checkout(cents: u64, discounts: &[Box<dyn Discount>]) -> u64 {
    discounts.iter().fold(cents, |price, d| d.apply(price))
}

fn main() {
    let discounts: Vec<Box<dyn Discount>> = vec![Box::new(Percent(10)), Box::new(Coupon(500))];
    println!("{}", checkout(2000, &discounts));
    println!("{}", checkout(300, &discounts));
}
```

**What happens:**

1. The contract says: the result is not higher than `cents`, and `apply` never panics.
2. `Percent` honours it. `Coupon` subtracts a fixed amount, but if the price is lower than the coupon, `cents - self.0` underflows.
3. `checkout` knows only the trait. For a 2000-cent order it works. For a 300-cent order, the coupon panics.

```text
1300

thread 'main' (5591460) panicked at src/main.rs:17:9:
attempt to subtract with overflow
```

(The number after `thread 'main'` and the exact text can differ on your computer. In a `--release` build the subtraction would silently wrap around to a huge number, which is even worse.) The type signature was fine; the *behaviour* was not. `checkout` cannot protect itself against every implementation, so each implementation must keep the promise.

The fix is small, and a test can protect it for every implementation. A test that runs one "contract check" against all implementers is a good habit:

```rust
/// Contract: the result is never higher than `cents` and the method never panics.
trait Discount {
    fn apply(&self, cents: u64) -> u64;
}

struct Percent(u64);
struct Coupon(u64);

impl Discount for Percent {
    fn apply(&self, cents: u64) -> u64 {
        cents - cents * self.0 / 100
    }
}

impl Discount for Coupon {
    fn apply(&self, cents: u64) -> u64 {
        cents.saturating_sub(self.0) // never below zero
    }
}

fn main() {
    println!("{}", Percent(10).apply(2000));
    println!("{}", Coupon(500).apply(300));
}

#[cfg(test)]
mod tests {
    use super::*;

    // One test of the contract, run against every implementation.
    fn check_contract(discount: &dyn Discount) {
        for cents in [0, 1, 99, 300, 10_000] {
            assert!(
                discount.apply(cents) <= cents,
                "discount raised the price of {cents}"
            );
        }
    }

    #[test]
    fn all_discounts_keep_the_contract() {
        check_contract(&Percent(10));
        check_contract(&Percent(100));
        check_contract(&Coupon(500));
    }
}
```

```text
running 1 test
test tests::all_discounts_keep_the_contract ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** Write the contract as a function that takes `&dyn Trait` and run it on each implementation. When a new implementer is added, add one line to the test. The standard library has contracts too, for example: if `a == b` then `hash(a) == hash(b)`. A type that breaks that rule makes a `HashMap` lose its keys.

**When NOT:** Liskov is not a reason to forbid any difference between implementers. Different implementations may be faster, slower or more careful. They must only keep the promises that the trait made.

## 5. I — Interface segregation

*Do not force users of a trait to depend on methods they do not use.* In Rust this has a concrete symptom: a method that an implementer cannot support and fills with `unimplemented!()`.

```rust
trait Store {
    fn get(&self, key: &str) -> Option<String>;
    fn put(&mut self, key: &str, value: &str);
}

struct Snapshot(Vec<(String, String)>); // read-only by nature

impl Store for Snapshot {
    fn get(&self, key: &str) -> Option<String> {
        self.0
            .iter()
            .find(|(k, _)| k == key)
            .map(|(_, v)| v.clone())
    }
    fn put(&mut self, _key: &str, _value: &str) {
        unimplemented!("snapshots are read-only")
    }
}

fn save_default_language(store: &mut impl Store) {
    store.put("lang", "en");
}

fn main() {
    let mut snapshot = Snapshot(vec![("lang".to_string(), "uz".to_string())]);
    println!("{:?}", snapshot.get("lang"));
    save_default_language(&mut snapshot);
}
```

**What happens:**

1. `Store` has `get` and `put`. A `Snapshot` is a frozen copy of data, so it cannot write.
2. Its `put` calls `unimplemented!`, a macro that panics with "not implemented". The code compiles.
3. `save_default_language` accepts any `Store`, including the snapshot. The mistake is found only at run time:

```text
Some("uz")

thread 'main' (5591981) panicked at src/main.rs:16:9:
not implemented: snapshots are read-only
```

The fix is to split the trait into small pieces, so that a function asks only for what it really needs:

```rust
trait Lookup {
    fn get(&self, key: &str) -> Option<String>;
}

trait Update {
    fn put(&mut self, key: &str, value: &str);
}

struct Snapshot(Vec<(String, String)>);

impl Lookup for Snapshot {
    fn get(&self, key: &str) -> Option<String> {
        self.0
            .iter()
            .find(|(k, _)| k == key)
            .map(|(_, v)| v.clone())
    }
}

fn save_default_language(store: &mut impl Update) {
    store.put("lang", "en");
}

fn main() {
    let mut snapshot = Snapshot(vec![("lang".to_string(), "uz".to_string())]);
    println!("{:?}", snapshot.get("lang"));
    save_default_language(&mut snapshot);
}
```

Now the mistake cannot be compiled:

```text
error[E0277]: the trait bound `Snapshot: Update` is not satisfied
  --> src/main.rs:27:27
   |
27 |     save_default_language(&mut snapshot);
   |     --------------------- ^^^^^^^^^^^^^ unsatisfied trait bound
   |     |
   |     required by a bound introduced by this call
   |
help: the trait `Update` is not implemented for `Snapshot`
   ...

For more information about this error, try `rustc --explain E0277`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

A run-time panic became a compile-time error that points at the exact call. Writes need `Update`, reads need `Lookup`, and a type implements only what it can do. The standard library does the same: `Read`, `Write` and `BufRead` are separate traits, and `Iterator` and `DoubleEndedIterator` are separate too.

**When NOT:** Do not make one trait per method. Methods that always go together (`open`, `read`, `close` of one resource) belong in one trait. Split when some implementers cannot, or some users do not need, a part.

## 6. D — Dependency inversion

*High-level code should depend on an abstraction, not on a low-level detail.* The business rule "write a report" should not know whether the report goes to the screen, a file or a network connection.

The tempting version calls `println!` inside the report function. Then the function can never be tested without capturing the screen, and it cannot be used for a file. The inverted version asks for *something that can be written to*, which is the standard trait `std::io::Write`:

```rust
use std::io::{self, Write};

fn write_report(out: &mut impl Write, items: &[&str]) -> io::Result<()> {
    writeln!(out, "{} items", items.len())?;
    for item in items {
        writeln!(out, "- {item}")?;
    }
    Ok(())
}

fn main() -> io::Result<()> {
    write_report(&mut io::stdout(), &["tea", "cake"])
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn report_lists_the_items() {
        let mut buffer: Vec<u8> = Vec::new();
        write_report(&mut buffer, &["tea"]).unwrap();
        assert_eq!(String::from_utf8(buffer).unwrap(), "1 items\n- tea\n");
    }
}
```

**What happens:**

1. `write_report` takes `out: &mut impl Write`: any type that implements `Write` (a terminal, a file, a network socket, a plain `Vec<u8>`).
2. `writeln!` is like `println!` but writes to `out`. The `?` passes an error up, because writing can fail.
3. `main` plugs in the real screen: `io::stdout()`.
4. The test plugs in a `Vec<u8>`, a growable list of bytes, and then checks the text. No terminal, no file, no mess.

```text
2 items
- tea
- cake
```

The test passes too (`cargo test` shows `report_lists_the_items ... ok`).

The word "inversion" means that the arrow of dependency is turned around: instead of the report depending on the screen, both depend on the small `Write` trait. The `main` function, which knows about the real world, connects the parts. Lesson 08.9 builds on this idea (dependency injection).

**When NOT:** A small program that prints one line does not need `Write`. Add the abstraction at the places where things really differ: the real world versus tests, or file versus network.

## Common mistakes

**Mistake 1: a trait for every struct.** `trait UserRepository` with exactly one implementation, written "to follow SOLID", is just indirection (YAGNI, Lesson 08.1). A trait is justified by a second implementation or by a real need such as testing.

**Mistake 2: ignoring the contract.** Implementing a trait just to make the types match, while the behaviour surprises its users, breaks Liskov (Section 4).

**Mistake 3: counting principles instead of reading code.** SOLID is a set of questions to ask when a change feels painful. If the code is simple and changes are easy, the principles are already satisfied.

## Try it yourself

You will make the "open" version of the shapes, extend it without editing old code, and protect it with tests.

1. Create the package:

```bash
cd ~/projects/rust
cargo new code/p08-solid
cd code/p08-solid
```

2. Copy the `Shape` trait, `Circle`, `Rect`, `Triangle`, `total_area` and `main` from Section 3 into `src/main.rs`.

3. **Without changing** any of the existing items, add a new shape `Square(f64)` (area is side times side) with its own `impl Shape`. Add it to the vector in `main`. Run it with `cargo run`; the total must become about `28.14`.

4. Add a `#[cfg(test)]` module with two tests: `no_shape_has_a_negative_area` (loop over all four shapes and `assert!(shape.area() >= 0.0)`) and `total_area_adds_every_shape` (compare the total with `std::f64::consts::PI + 25.0` using `(total - expected).abs() < 1e-9`, because floating-point numbers should not be compared with `==`).

5. Run `cargo test`:

```bash
cargo test
```

```text
running 2 tests
test tests::no_shape_has_a_negative_area ... ok
test tests::total_area_adds_every_shape ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

6. Check and save:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): shapes behind a trait"
```

**What happens:** `no_shape_has_a_negative_area` is a contract check (Liskov): it runs on every shape, and a new shape that breaks the promise is caught at once. The addition of `Square` touched no old code (open/closed).

## Exercises

### Exercise 1 — Which letter? (easy)

Match each problem to a SOLID letter:

1. A struct both parses configuration and sends network requests.
2. A function `fn save(x: &mut FileStore)` cannot be tested without a real file.
3. Every time a new payment method is added, six `match` expressions on strings must be edited, and some are forgotten.

<details>
<summary>Hint</summary>

Look for the key words: "both... and" (reasons to change), "cannot be tested" (depends on a detail), "forgotten" (extension).

</details>

<details>
<summary>Answer</summary>

1. S, single responsibility.
2. D, dependency inversion: depend on a trait such as `Write` or `Read`, not on `FileStore`.
3. O, open/closed. A good fix is an `enum PaymentMethod` (the compiler finds forgotten matches), or a trait if outsiders add methods. Matching on strings is the worst of both.

Why it works: each principle names what hurts, and points at a cure.

</details>

### Exercise 2 — Test the contract (medium)

The type `Capped` below should return the price, but at most `max` cents. It breaks the contract of `Discount` (the result must never be higher than the input). **Write a test** that finds the bug, then fix the code.

```rust
/// Contract: the result is never higher than `cents`.
trait Discount {
    fn apply(&self, cents: u64) -> u64;
}

/// Meant to be "the price, but at most `max` cents".
struct Capped(u64);

impl Discount for Capped {
    fn apply(&self, _cents: u64) -> u64 {
        self.0
    }
}

fn main() {
    println!("{}", Capped(100).apply(250));
    println!("{}", Capped(100).apply(50));
}
```

<details>
<summary>Hint</summary>

Loop over several prices (including one below the cap, such as 50) and assert `apply(cents) <= cents`. The fix uses `min`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Contract: the result is never higher than `cents`.
trait Discount {
    fn apply(&self, cents: u64) -> u64;
}

struct Capped(u64);

impl Discount for Capped {
    fn apply(&self, cents: u64) -> u64 {
        cents.min(self.0)
    }
}

fn main() {
    println!("{}", Capped(100).apply(250));
    println!("{}", Capped(100).apply(50));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn capped_never_raises_the_price() {
        let discount = Capped(100);
        for cents in [0, 50, 100, 250] {
            assert!(discount.apply(cents) <= cents, "price {cents} was raised");
        }
    }
}
```

Why it works: with the buggy code, the test fails on `cents = 0` (the result 100 is higher than 0). After the fix `cents.min(self.0)` is never higher than `cents`.

</details>

### Exercise 3 — Enum or trait? (hard)

A command-line tool has the commands `start` and `stop`, today handled in a `match` on an enum. Next year other teams must be able to add their own commands from their own crates. How do you redesign it, and what do you lose?

<details>
<summary>Hint</summary>

If outsiders add cases, the set of cases is open. Think of a trait with a `run` method and a map from command names to trait objects.

</details>

<details>
<summary>Answer</summary>

Define `trait Command { fn run(&self) -> String; }`, implement it for `Start` and `Stop`, and keep a registry `HashMap<String, Box<dyn Command>>`. A plugin crate implements the trait and inserts its command into the registry; nothing else is edited.

Why it works: `Command` is the extension point. What you lose: the compiler no longer lists "forgotten" commands for you, a missing command is a run-time lookup failure (`HashMap` returns `None`), and each call goes through a `dyn` pointer (Lesson 06.1). That is a fair price only if outsiders really add commands.

</details>

## Quiz

1. What does the "S" in SOLID say, in one sentence?

<details>
<summary>Answer</summary>

A part of the program should have one reason to change.

</details>

2. Why is a non-exhaustive `match` error a feature when you use an enum?

<details>
<summary>Answer</summary>

When you add a variant, the compiler lists every `match` that you forgot to update, so nothing is missed.

</details>

3. What can a trait promise that the compiler cannot check?

<details>
<summary>Answer</summary>

Its contract: behaviour such as "never panics", "keeps the order", "result is not larger than the input".

</details>

4. How does splitting a trait help the compiler find mistakes?

<details>
<summary>Answer</summary>

A function asks only for the small trait it needs, so passing a type that lacks it is a compile error instead of a run-time `unimplemented!` panic.

</details>

## Summary

- **S**: one reason to change per part; split by reason, not by method count.
- **O**: add behaviour without editing old code. Use an `enum` for a closed set (the compiler finds forgotten cases) and a trait for an open set.
- **L**: every implementation must keep the trait's contract; protect it with one contract test run against all implementers.
- **I**: small traits; a method filled with `unimplemented!()` is the symptom. Splitting turns run-time panics into compile-time errors.
- **D**: depend on a trait (`impl Write`, `impl BufRead`), not on the screen, a file or the network; the real world is plugged in by `main`.
- SOLID is a set of questions to ask when change hurts, not a recipe for adding traits everywhere.

## Sources and further reading

- [The Rust Programming Language — 18.2 Using Trait Objects That Allow for Values of Different Types](https://doc.rust-lang.org/book/ch18-02-trait-objects.html) — the official chapter on trait objects and extension.
- [Rust API Guidelines — Flexibility](https://rust-lang.github.io/api-guidelines/flexibility.html) — advice on taking generic parameters (`impl Read`, `impl Write`) so that callers can plug in what they have.
- [std::io::Write](https://doc.rust-lang.org/std/io/trait.Write.html) and [std::io::BufRead](https://doc.rust-lang.org/std/io/trait.BufRead.html) — the standard traits used in this lesson.
- [Robert C. Martin — Principles of OOD](http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod) — the original article that collected the principles (OOD: Object-Oriented Design, design built around classes and objects).

---

[🏠 Start](../START-HERE.md) · [← 08.2 Cohesion, coupling and separation of concerns](02-cohesion-coupling-and-separation-of-concerns.md) · [📋 Phase 08](README.md) · [08.4 Composition over inheritance →](04-composition-over-inheritance.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
