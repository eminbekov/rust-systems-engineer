<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.10 Anti-patterns and code smells](10-anti-patterns-and-code-smells.md) · [📋 Phase 08](README.md) · [08.12 Project layout and workspaces →](12-project-layout-and-workspaces.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/11-conventions-naming-clippy-and-api-guidelines.md) <!-- nav -->

# Lesson 08.11 — Conventions: naming, Clippy and the Rust API Guidelines

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 11 of 17 · ⏱ about 135 minutes

## What you will learn

- The naming conventions of Rust: casing, and the prefixes `as_`, `to_`, `into_`, `is_`, `iter`.
- How `rustfmt` fixes formatting for the whole team, and how to configure it.
- How to use Clippy's lint groups, lint levels (`allow`, `warn`, `deny`, `expect`) and the `[lints]` table in `Cargo.toml`.
- What the Rust API Guidelines ask of a public API (Application Programming Interface: the items your crate offers to other code), shown on one small library.

## Before you start

- You finished [Lesson 08.10 — Anti-patterns and code smells](10-anti-patterns-and-code-smells.md).
- You know the quality gate from [Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md) and doc comments from [Lesson 05.7](../05-rust-intermediate/07-documentation.md).
- Create two practice packages: `cargo new code/p08-conventions --lib` (for Section 5) and `cargo new code/p08-gate` (for "Try it yourself"). Other examples can go into any scratch package.

## New words

| Word | Meaning |
|---|---|
| **convention** | An agreed way of doing something that the language does not enforce, but that everybody follows so that code looks familiar. |
| **`rustfmt`** | The official tool that rewrites Rust code into the standard layout. You run it with `cargo fmt`. |
| **lint level** | What happens when a lint fires: `allow` (nothing), `warn` (a warning), `deny` (an error), `forbid` (an error that cannot be switched off), `expect` (like `allow`, but a warning if the lint did *not* fire). |
| **lint group** | A named set of lints that you switch on or off together, like `clippy::pedantic`. |
| **TOML** (Tom's Obvious Minimal Language) | The text format of `Cargo.toml`, `rustfmt.toml` and `clippy.toml`. |
| **RFC** (Request for Comments) | A written proposal for a change to Rust. Rust's naming rules come from a numbered RFC (number 430). |
| **Rust API Guidelines** | A checklist, written by the Rust library team, for designing public APIs (API: Application Programming Interface, the functions and types a crate offers to other code) that feel "Rusty". |
| **`#[non_exhaustive]`** | An attribute that tells other crates: "I may add variants or fields later", so they must handle the unknown. |
| **`#[must_use]`** | An attribute that makes the compiler warn when a caller ignores the result. |
| **CI** (Continuous Integration) | A server that runs your checks (format, lints, tests) automatically on every change. Phase 09 builds CI pipelines. |

## 1. Why conventions

A convention costs nothing and pays back every day. If every Rust project formats code the same way, names
things the same way and uses the same lint settings, you can read a stranger's code as fast as your own, tools
work without configuration, and code reviews (Lesson 08.13) discuss design instead of spaces and names. Most
conventions in this lesson are checked by a tool:

| Convention | Checked by | Command |
|---|---|---|
| Naming | the compiler (casing) and Clippy | `cargo build`, `cargo clippy` |
| Layout of the code | `rustfmt` | `cargo fmt --check` |
| Idioms, common mistakes | Clippy | `cargo clippy -- -D warnings` |
| Documentation | the lint `missing_docs`, doc tests | `cargo build`, `cargo test --doc` |
| Public API design | people, with the API Guidelines | code review |

## 2. Naming

### 2.1 Casing

The compiler warns when a name has the wrong style. In a scratch package, `src/main.rs`:

```rust
struct shopping_cart {
    ItemCount: u32,
}

const maxItems: u32 = 10;

fn AddItem(cart: &mut shopping_cart) {
    cart.ItemCount += 1;
}

fn main() {
    let mut cart = shopping_cart { ItemCount: 0 };
    AddItem(&mut cart);
    println!("{} of {}", cart.ItemCount, maxItems);
}
```

`cargo build` compiles it and prints four warnings (the first two shown):

```text
warning: type `shopping_cart` should have an upper camel case name
 --> src/main.rs:1:8
  |
1 | struct shopping_cart {
  |        ^^^^^^^^^^^^^ help: convert the identifier to upper camel case: `ShoppingCart`

warning: structure field `ItemCount` should have a snake case name
 --> src/main.rs:2:5
  |
2 |     ItemCount: u32,
  |     ^^^^^^^^^ help: convert the identifier to snake case: `item_count`
```

The rules (from the API Guidelines, which follow RFC 430):

| Item | Style | Example |
|---|---|---|
| Types: `struct`, `enum`, `trait`, type aliases; enum variants | `UpperCamelCase` | `ShoppingCart`, `Zone::Foreign` |
| Functions, methods, modules, variables, fields | `snake_case` | `add_item`, `item_count` |
| Constants and statics | `SCREAMING_SNAKE_CASE` | `MAX_ITEMS` |
| Generic type parameters | one capital letter, or short `UpperCamelCase` | `T`, `Item` |
| Lifetimes | short lowercase | `'a` |

In `UpperCamelCase`, an acronym counts as one word: `Uuid`, `HttpServer`, not `UUID` or `HTTPServer`. A package name
usually has dashes (`p08-conventions`); the crate name in code has underscores (`p08_conventions`).

### 2.2 Words with a meaning

Some prefixes mean the same thing in the whole standard library, so a name tells you what a method does and what it costs:

| Name | Means | Cost | Standard-library example |
|---|---|---|---|
| `as_x` | a borrowed view of the data, as another borrowed type | free | `String::as_str`, `Vec::as_slice` |
| `to_x` | a new value made from a borrowed one (by value for `Copy` types) | may be expensive | `str::to_uppercase` |
| `into_x` | consumes `self` and returns the data in another form | usually cheap | `String::into_bytes` |
| `x()` | a getter: the name of the field, without `get_` | free | `Vec::len` |
| `x_mut()` | the same, as a mutable borrow | free | `slice::first_mut` |
| `is_x`, `has_x` | a yes-no question | cheap | `Vec::is_empty` |
| `iter`, `iter_mut`, `into_iter` | borrow items, borrow mutably, consume | free | `Vec::iter` |
| `new`, `with_x`, `from_x` | constructors | | `Vec::with_capacity` |

```rust
fn main() {
    let text = String::from("Hello");

    let view: &str = text.as_str(); // as_: a free, borrowed view
    let upper: String = view.to_uppercase(); // to_: new value, may cost work
    println!("{view} {upper}");

    let bytes: Vec<u8> = text.into_bytes(); // into_: consumes `text`
    println!("{bytes:?}");

    let mut numbers = vec![1, 2, 3];
    for n in numbers.iter_mut() {
        *n *= 10; // iter_mut: borrows items mutably
    }
    for n in numbers {
        print!("{n} "); // into_iter: consumes the vector
    }
    println!();
}
```

**What happens:**

1. `as_str` only looks at `text`; `to_uppercase` allocates a new `String`. The prefix tells you which is cheap.
2. `into_bytes` takes `self` by value. After it, `text` is moved; using it again would be a compile error.
3. The last `for` takes `numbers` by value, which calls `into_iter`.

```text
Hello HELLO
[72, 101, 108, 108, 111]
10 20 30 
```

> 📏 **Convention:** Do not name a getter `get_name`. Name it `name`. The word `get` is kept for a "get by key" such as
> `HashMap::get`. Rust code reads `user.name()`, not `user.get_name()`.

## 3. Formatting with `rustfmt`

Arguments about formatting waste time, and a big diff full of whitespace hides the real change. Rust ends the
discussion with one tool, `rustfmt`, which comes with your toolchain and runs through Cargo. Take this untidy but valid code:

```rust
fn   add(a:i32,b:i32)->i32{
a+b
}

fn main(){
    let total=add( 2,3 );
    println!("{total}");
        if total>3 {println!("big");}
}
```

`cargo fmt --check` only *checks*: it prints what it would change and exits with status 1, which is what a CI job uses.

```text
Diff in .../src/main.rs:1:
-fn   add(a:i32,b:i32)->i32{
-a+b
+fn add(a: i32, b: i32) -> i32 {
+    a + b
 }
 
-fn main(){
-    let total=add( 2,3 );
+fn main() {
+    let total = add(2, 3);
     println!("{total}");
-        if total>3 {println!("big");}
+    if total > 3 {
+        println!("big");
+    }
 }
```

`cargo fmt` (without `--check`) rewrites the files. RustRover can do the same with *Reformat Code*, and you can
let it run `rustfmt` when you save.

**Configuration.** Almost nobody changes the defaults, and that is the point: the default style is what other
projects use. If you must, create `rustfmt.toml` in the project root. For example, `max_width = 60` (the default is 100) breaks
longer lines:

```rust
fn main() {
    let message = format!(
        "{} items cost {} in total, including {} tax",
        3, 100, 20
    );
    println!("{message}");
}
```

Many options exist only in the nightly compiler. On stable, `rustfmt` ignores them and says so:
``Warning: can't set `imports_granularity = Crate`, unstable features are only available in nightly channel.``
When hand-made layout carries meaning, such as a table of numbers, put `#[rustfmt::skip]` before the item and `rustfmt` leaves it alone.

> 📏 **Convention:** Run `cargo fmt` before every commit, and `cargo fmt --check` in CI. Never mix a formatting-only
> change with a real change in one commit; it makes review impossible.

## 4. Clippy in depth

In Lesson 08.10 you ran `cargo clippy`. Now you learn to control it.

### 4.1 Lint groups

Clippy has hundreds of lints, sorted into groups:

| Group | Default level | What it contains |
|---|---|---|
| `clippy::correctness` | deny (error) | Code that is almost certainly wrong. |
| `clippy::suspicious` | warn | Code that is probably wrong. |
| `clippy::style` | warn | Code that works but is not idiomatic. |
| `clippy::complexity` | warn | Code that does something simple in a complicated way. |
| `clippy::perf` | warn | Code that can easily be made faster. |
| `clippy::pedantic` | allow (off) | Strict, opinionated lints. Some give false alarms. |
| `clippy::nursery` | allow (off) | New lints that are still being tested. |
| `clippy::restriction` | allow (off) | Lints that forbid a feature (`unwrap_used`, `indexing_slicing`). Never switch on the whole group; choose single lints. |
| `clippy::cargo` | allow (off) | Checks of `Cargo.toml` (missing description, duplicate versions). |

Switch a group on from the command line with `-W` (warn) or `-D` (deny). This code is correct and the default Clippy is silent:

```rust
fn average(values: &[f64]) -> f64 {
    values.iter().sum::<f64>() / values.len() as f64
}

fn main() {
    let numbers = vec![1.0, 2.0, 4.5];
    println!("{}", average(&numbers));
}
```

With `cargo clippy -- -W clippy::pedantic`:

```text
warning: casting `usize` to `f64` may cause a loss of precision (`usize` can be up to 64 bits wide depending on the target architecture, but `f64`'s mantissa is only 52 bits wide)
 --> src/main.rs:2:34
  |
2 |     values.iter().sum::<f64>() / values.len() as f64
  |                                  ^^^^^^^^^^^^^^^^^^^
  = note: `-W clippy::cast-precision-loss` implied by `-W clippy::pedantic`
  = help: to override `-W clippy::pedantic` add `#[allow(clippy::cast_precision_loss)]`
```

This is typical for `pedantic`: the lint is *right* in theory (no slice has 2^52 items in practice), and you decide it does not matter here.

### 4.2 Lint levels and exceptions

You choose a level in three places: in `Cargo.toml` for the whole package (Section 4.3), in the source for a crate or module
(`#![warn(missing_docs)]`), and in the source for one item (`#[allow(clippy::cast_precision_loss)]` before a function).

For one exception you have two attributes. `#[allow(...)]` is silent forever. `#[expect(...)]` (stable since Rust 1.81) does the same
**and warns when the lint stops firing**, so stale exceptions disappear. Always give a reason:

```rust
#[expect(
    clippy::cast_precision_loss,
    reason = "slice lengths are far below 2^52, so the conversion is exact"
)]
fn average(values: &[f64]) -> f64 {
    values.iter().sum::<f64>() / values.len() as f64
}

fn main() {
    let numbers = vec![1.0, 2.0, 4.5];
    println!("{}", average(&numbers));
}
```

With `pedantic` on, `cargo clippy` is silent and `cargo run` prints `2.5`. If you later change `average` so that it no longer casts
(say, it divides by `3.0`), the exception is unnecessary, and Clippy tells you:

```text
warning: this lint expectation is unfulfilled
 --> src/main.rs:2:5
  |
2 |     clippy::cast_precision_loss,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: slice lengths are far below 2^52, so the conversion is exact
  = note: `#[warn(unfulfilled_lint_expectations)]` on by default
```

> ⚠️ **Common mistake:** A crate-wide `#![allow(clippy::all)]` or `#![allow(warnings)]` to "get rid of the noise". It turns the
> tool off. Allow one lint, on one item, with a reason.

### 4.3 The `[lints]` table, `clippy.toml` and `--fix`

Since Cargo 1.74, lint levels can live in `Cargo.toml`, so every command (and RustRover) sees the same policy:

```toml
[lints.rust]
missing_docs = "warn"

[lints.clippy]
pedantic = { level = "warn", priority = -1 }
unwrap_used = "deny"
```

**What happens:**

1. `[lints.rust]` sets levels for compiler lints; `[lints.clippy]` for Clippy lints.
2. `pedantic` is a group. `priority = -1` means "apply this first", so a single lint, like `unwrap_used`, can override the group.
   (Without a priority, Clippy's lint `lint_groups_priority` reports that the group and the single lint conflict.)
3. `unwrap_used` is from the `restriction` group: we choose this one lint on purpose. With it, any `.unwrap()` on a
   `Result` is ``error: used `unwrap()` on a `Result` value`` and the check fails; a public function without a doc comment
   gives `warning: missing documentation for a function`.

Some lints have *settings*. They go into `clippy.toml` in the project root. For example, `too-many-arguments-threshold = 3`
makes Clippy report `this function has too many arguments (4/3)` for a function with four parameters.

`cargo clippy --fix` applies the suggestions that are safe. It refuses to run if you have uncommitted changes (commit
first, or pass `--allow-dirty`). A successful run prints `Fixed src/main.rs (1 fix)`. Read `git diff` afterwards: the tool is good, but you are responsible for the commit.

> 🧪 **Testing:** The quality gate of a project is three commands: `cargo fmt --check`, `cargo clippy --all-targets -- -D
> warnings` and `cargo test`. `--all-targets` makes Clippy check tests and examples too, not only the main code.

## 5. The Rust API Guidelines

The **Rust API Guidelines** are a checklist (about 50 items, each with a code like `C-GETTER`) that the Rust library team
wrote from long experience with the standard library. They answer: "What should the public side of my crate look like?"

| Group | What it asks (selected items) |
|---|---|
| Naming | Casing; `as_`/`to_`/`into_`; getters without `get_`; `iter`/`iter_mut`/`into_iter`. |
| Interoperability | Implement the common traits: `Debug` always, and `Clone`, `PartialEq`, `Default`, `Display` when they make sense; `From` and `TryFrom` for conversions; error types implement `std::error::Error`. |
| Documentation | Every public item has a doc comment; a runnable example; `# Errors`, `# Panics` and `# Safety` sections when they apply. |
| Predictability | Constructors are associated functions (`new`); no out-parameters. |
| Flexibility | Accept the most general borrowed type (`&str`, `&[T]`). |
| Type safety | Newtypes for distinct meanings; an `enum` instead of a `bool` argument; builders for many options (Lesson 08.6). |
| Dependability | Validate arguments in constructors. |
| Future proofing | Keep struct fields private; mark public enums that may grow `#[non_exhaustive]`. |

You do not memorise it; you open the checklist when a public API is nearly done and walk through it. Here is a small
library that applies several items. In `code/p08-conventions/src/lib.rs`:

```rust
//! Temperatures that cannot be physically impossible.

#![warn(missing_docs)]

use std::fmt;

const ABSOLUTE_ZERO: f64 = -273.15;

/// A temperature in degrees Celsius, never below absolute zero.
///
/// # Examples
///
/// ```
/// use p08_conventions::Celsius;
///
/// let warm = Celsius::new(21.5).unwrap();
/// assert_eq!(warm.degrees(), 21.5);
/// assert!(!warm.is_freezing());
/// assert!(Celsius::new(-300.0).is_err());
/// ```
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd, Default)]
pub struct Celsius(f64);

/// The ways in which creating a [`Celsius`] can fail.
#[derive(Debug, Clone, PartialEq)]
#[non_exhaustive]
pub enum TemperatureError {
    /// The value is below absolute zero (-273.15).
    BelowAbsoluteZero,
    /// The value is not a number (`NaN`).
    NotANumber,
}

impl Celsius {
    /// Creates a temperature.
    ///
    /// # Errors
    ///
    /// Returns [`TemperatureError::BelowAbsoluteZero`] for values under
    /// absolute zero, and [`TemperatureError::NotANumber`] for `NaN`.
    pub fn new(degrees: f64) -> Result<Celsius, TemperatureError> {
        if degrees.is_nan() {
            Err(TemperatureError::NotANumber)
        } else if degrees < ABSOLUTE_ZERO {
            Err(TemperatureError::BelowAbsoluteZero)
        } else {
            Ok(Celsius(degrees))
        }
    }

    /// The value in degrees Celsius.
    #[must_use]
    pub fn degrees(self) -> f64 {
        self.0
    }

    /// Converts to degrees Fahrenheit.
    #[must_use]
    pub fn to_fahrenheit(self) -> f64 {
        self.0 * 9.0 / 5.0 + 32.0
    }

    /// Is the temperature below the freezing point of water?
    #[must_use]
    pub fn is_freezing(self) -> bool {
        self.0 < 0.0
    }
}

impl fmt::Display for TemperatureError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            TemperatureError::BelowAbsoluteZero => write!(f, "below absolute zero"),
            TemperatureError::NotANumber => write!(f, "not a number"),
        }
    }
}

impl std::error::Error for TemperatureError {}
```

**What happens:** each feature is one guideline.

1. **Type safety, dependability:** `Celsius` is a newtype with a *private* field. The only way to build one is
   `Celsius::new`, which validates the value; a `Celsius` below absolute zero cannot exist (Lesson 08.5).
2. **Naming:** `degrees()` is a getter without `get_`; `to_fahrenheit` takes a `Copy` value by value; `is_freezing` is a question.
3. **Interoperability:** `Debug`, `Clone`, `Copy`, `PartialEq`, `PartialOrd`, `Default` are derived. The error type
   implements `Display` and `std::error::Error`, so it works with `?` and with `Box<dyn Error>`.
4. **Future proofing:** `#[non_exhaustive]` lets you add a variant later without breaking code in other crates that matches on the
   enum (they must write a `_` arm).
5. **Documentation:** `#![warn(missing_docs)]` makes the compiler complain about every public item without a doc comment;
   `# Errors` says when `new` fails; the example is a doc test (Lesson 05.7).
6. `#[must_use]` makes the compiler warn if someone writes `c.to_fahrenheit();` and throws the result away.

Run `cargo test`, `cargo clippy -- -W clippy::pedantic` and `cargo fmt --check`. Clippy and `fmt` print nothing apart from
`Checking`/`Finished`, and the test prints:

```text
   Doc-tests p08_conventions
test src/lib.rs - Celsius (line 13) ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Now delete the line `/// Converts to degrees Fahrenheit.` and build again. The compiler complains, naming the exact method:

```text
warning: missing documentation for a method
  --> src/lib.rs:58:5
   |
58 |     pub fn to_fahrenheit(self) -> f64 {
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Put the line back. (Add `#[test]` functions of your own in Exercise 3.)

> 💡 **Tip:** The guidelines are written for *public* APIs that other people use. For private helpers inside your own program,
> the ideas still help, but do not spend an hour on `#[non_exhaustive]` for an enum that only your own code sees.

## Common mistakes

1. **Arguing about style instead of running `cargo fmt`.** The tool decides; the discussion is over.
2. **A `get_` prefix on getters** (`get_name`). Use `name`. Clippy does not catch this; code review does.
3. **`allow` without a reason.** Prefer `#[expect(..., reason = "...")]`.
4. **Switching on the whole `restriction` group.** Its lints contradict each other. Choose single lints.
5. **Making a field `pub` "for now".** A public field is part of the API for good. Start private and add methods.

## Try it yourself

Run the quality gate on a sloppy program. In `code/p08-gate/src/main.rs`:

```rust
struct shopping_cart { items: Vec<u32> }

impl shopping_cart {
    fn get_total(&self)->u32 {
        let mut total=0;
        for i in 0..self.items.len() { total+=self.items[i]; }
        return total;
    }
}

fn main(){
    let cart=shopping_cart{items:vec![1,2,3]};
    println!("{}", cart.get_total());
}
```

1. Run `cargo fmt --check`. It prints a diff (the first part is `-struct shopping_cart { items: Vec<u32> }`).
2. Run `cargo clippy`. You see three warnings: the type name, `unneeded return statement` and the index loop.
3. Fix everything by hand: rename the type to `ShoppingCart`, rename `get_total` to `total`, replace the loop and the `return` with
   `self.items.iter().sum()`, and run `cargo fmt`.
4. Run `cargo fmt --check && cargo clippy -- -D warnings && cargo run`. The gate is silent (apart from `Checking`/`Finished`) and the program prints `6`.

## Exercises

### Exercise 1 — Choose the right names (easy)

Rename these methods of a `Name` type according to the conventions: `GetValue(&self) -> &str`, `fn to_str_ref(&self) -> &str` (a free view of the
inner text), `fn isEmpty(&self) -> bool`, `fn into_string_owned(self) -> String` and `fn get_len(&self) -> usize`.

<details>
<summary>Hint</summary>

Methods are `snake_case`. A free borrowed view is `as_...`. A getter drops `get_`. A method that consumes `self` is `into_...`.

</details>

<details>
<summary>Answer</summary>

`value`, `as_str`, `is_empty`, `into_string`, `len`.

Why it works: `GetValue` has the wrong casing and a `get_` flavour; the free view is `as_str` (as in `String::as_str`); the question is `is_empty`;
the consuming conversion is `into_string`; the length is `len`.

</details>

### Exercise 2 — Set a lint policy (medium)

In a library package, add a `[lints]` table to `Cargo.toml` that makes the compiler warn about missing documentation, turns on the Clippy `pedantic` group
as warnings, and makes `clippy::unwrap_used` an error. Write a public function that calls `unwrap()` and run `cargo clippy`. What do you expect to see?

<details>
<summary>Hint</summary>

Two tables, `[lints.rust]` and `[lints.clippy]`. The group needs `priority = -1`.

</details>

<details>
<summary>Answer</summary>

```toml
[lints.rust]
missing_docs = "warn"

[lints.clippy]
pedantic = { level = "warn", priority = -1 }
unwrap_used = "deny"
```

`cargo clippy` prints ``error: used `unwrap()` on a `Result` value`` (the check fails), and warnings for a missing `# Panics` section
(`missing_panics_doc`, a `pedantic` lint) and for undocumented public items.

</details>

### Exercise 3 — Document with a test (medium)

Write `pub fn is_leap_year(year: u32) -> bool` for the Gregorian calendar (divisible by 4, except years divisible by 100, unless also divisible by 400) in a
library package `code/p08-calendar`. Document it with a `# Examples` section whose doc test checks 2024, 1900 and 2000, and make it pass
`cargo clippy -- -W clippy::pedantic`.

<details>
<summary>Hint</summary>

The example is a code block inside `///` comments, and it must `use` your crate by its underscore name (`p08_calendar`). If Clippy suggests
`is_multiple_of`, take its advice: it exists on unsigned integers.

</details>

<details>
<summary>Answer</summary>

```rust
//! Calendar helpers.

/// Tells whether a year is a leap year in the Gregorian calendar.
///
/// A year is a leap year if it is divisible by 4, except years divisible by 100,
/// unless they are also divisible by 400.
///
/// # Examples
///
/// ```
/// use p08_calendar::is_leap_year;
///
/// assert!(is_leap_year(2024));
/// assert!(!is_leap_year(1900));
/// assert!(is_leap_year(2000));
/// ```
#[must_use]
pub fn is_leap_year(year: u32) -> bool {
    year.is_multiple_of(4) && (!year.is_multiple_of(100) || year.is_multiple_of(400))
}
```

`cargo test --doc` prints `test src/lib.rs - is_leap_year (line 10) ... ok`. A first version written as `year % 4 == 0 && ...` compiles, but Clippy's
`manual_is_multiple_of` lint suggests exactly the code above.

</details>

## Quiz

1. What do the prefixes `as_`, `to_` and `into_` promise?

<details>
<summary>Answer</summary>

`as_` is a free borrowed view; `to_` makes a new value from a borrowed one (can cost work); `into_` consumes `self`.

</details>

2. What is the difference between `cargo fmt` and `cargo fmt --check`?

<details>
<summary>Answer</summary>

`cargo fmt` rewrites the files. `cargo fmt --check` changes nothing: it prints the differences and exits with status 1 if there are any, so a script or CI job can fail.

</details>

3. Why prefer `#[expect(lint, reason = "...")]` over `#[allow(lint)]`?

<details>
<summary>Answer</summary>

`expect` warns when the lint no longer fires, so exceptions that are no longer needed get removed, and the `reason` tells the next reader why the exception exists.

</details>

4. Why is the field of `Celsius` private?

<details>
<summary>Answer</summary>

So that the only way to create a value is `Celsius::new`, which checks the rule. A public field would let anyone build an impossible value and would become a
permanent part of the API.

</details>

## Summary

- Conventions make code look familiar and let tools check it: the compiler (casing), `rustfmt` (layout), Clippy (idioms) and the API Guidelines (public design).
- Casing: `UpperCamelCase` types, `snake_case` functions and variables, `SCREAMING_SNAKE_CASE` constants. Prefixes carry meaning: `as_` (free view), `to_` (new value),
  `into_` (consumes), `is_`/`has_` (question); getters have no `get_`.
- `cargo fmt` formats, `cargo fmt --check` verifies; keep the default style.
- Clippy lints come in groups; `pedantic` is opinionated, `restriction` is for choosing single lints. Levels are `allow`, `warn`, `deny`, `forbid` and `expect`. Set the
  project policy in the `[lints]` table of `Cargo.toml`, and use `#[expect(..., reason = "...")]` for exceptions.
- The quality gate is `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test`.
- The Rust API Guidelines: implement the common traits, validate in constructors, keep fields private, document with runnable examples, and use `#[non_exhaustive]` and
  `#[must_use]` where they fit.

## Sources and further reading

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — the full checklist with an explanation and an example for every item.
- [Rust API Guidelines — Naming](https://rust-lang.github.io/api-guidelines/naming.html) — casing, conversions (`as_`, `to_`, `into_`), getters and iterator names.
- [The Rust Style Guide](https://doc.rust-lang.org/stable/style-guide/) — the formatting rules that `rustfmt` implements.
- [rustfmt configuration](https://rust-lang.github.io/rustfmt/) — every `rustfmt.toml` option, and which ones need nightly.
- [Clippy documentation](https://doc.rust-lang.org/stable/clippy/) — lint groups, configuration and `clippy.toml`.
- [The Cargo Book — the `[lints]` section](https://doc.rust-lang.org/cargo/reference/manifest.html) — how to set lint levels for a package.
- [The Rust Reference — Diagnostic attributes](https://doc.rust-lang.org/reference/attributes/diagnostics.html) — `allow`, `warn`, `deny`, `forbid`, `expect` and `must_use`.

---

[🏠 Start](../START-HERE.md) · [← 08.10 Anti-patterns and code smells](10-anti-patterns-and-code-smells.md) · [📋 Phase 08](README.md) · [08.12 Project layout and workspaces →](12-project-layout-and-workspaces.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
