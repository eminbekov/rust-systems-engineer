<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.5 if let, let else and patterns](05-if-let-let-else-and-patterns.md) · [📋 Phase 04](README.md) · [04.7 Vectors →](07-vectors.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/06-modules-crates-and-packages.md) <!-- nav -->

# Lesson 04.6 — Modules, crates and packages

> **Phase 04 — Structs, enums, collections and errors** · Lesson 6 of 12 · ⏱ about 4 hours (3 sittings)

## What you will learn

- The three levels of Rust code organisation: **package**, **crate** and **module**, and how they fit together.
- How to write modules in one file and in separate files, and how paths (`crate::`, `self::`, `super::`) find items.
- Privacy: everything is private by default, `pub` opens it, and why that protects your data.
- How `use` brings names into scope, how to rename with `as` and re-export with `pub use`.
- How to build a package that has both a library and a program, and where unit tests and integration tests live.

## Before you start

- You finished [Lesson 04.5 — if let, let else and patterns](05-if-let-let-else-and-patterns.md) and the earlier lessons of this phase (structs, methods, enums, `Option`).
- You can create a project with `cargo new` and run `cargo run`, `cargo test` ([Lesson 00.6 — Your first Rust program](../00-getting-started/06-your-first-rust-program.md), [Lesson 02.8 — Your first tests](../02-programming-basics/08-your-first-tests.md)).
- Your learning repository from [Lesson 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) is ready; we commit at the end.

## New words

| Word | Meaning |
|---|---|
| **module** | A named container of code inside a crate (functions, types, other modules). It is a namespace and a privacy boundary. |
| **crate** | The unit that the compiler compiles in one go. A crate is either a **binary** (a program with `main`) or a **library** (code for others to use). |
| **package** | One folder with a `Cargo.toml`. It contains one or more crates. `cargo new` creates a package. |
| **crate root** | The source file where the compiler starts reading a crate: `src/main.rs` for a binary, `src/lib.rs` for a library. Its content forms the module called `crate`. |
| **module tree** | All modules of a crate, as a tree: the crate root at the top, child modules under it. |
| **path** | The "address" of an item in the module tree, written with `::`, for example `crate::geometry::circle::Circle`. |
| **absolute path** | A path that starts at the crate root with `crate::` (or at an external crate name). |
| **relative path** | A path that starts in the current module, or with `self::` or `super::`. |
| **`super`** | In a path: "the parent module". |
| **`pub`** | Makes an item (function, struct, module, field …) visible outside its module. Without `pub` it is private. |
| **privacy** | The rule about which code may see which item. |
| **encapsulation** | Hiding the inside of a type (private fields) and offering a small public set of functions, so that outside code cannot break the type's rules. |
| **`use`** | A declaration that brings a path into scope under a short name. |
| **re-export** | `pub use path;`: makes an item available under a new, usually shorter, public path. |
| **glob import** | `use some::path::*;` imports every public item of a module. |
| **unit test** | A test inside the source file of the code it tests, in `#[cfg(test)] mod tests`. |
| **integration test** | A test in the `tests/` folder. It is compiled as a separate crate and can use only the public API. |
| **API** (Application Programming Interface) | The public functions and types a crate offers to others. |
| **namespace** | A named area in which names cannot clash with names in other areas. |
| **TOML** (Tom's Obvious Minimal Language) | The simple file format of `Cargo.toml`. |

## 1. Why split code at all?

So far every program was one file. That works for 50 lines. A real program has thousands. If everything sits in one file and one namespace:

- you cannot find anything;
- two things want the same name (`new`, `parse`, `Error`);
- anyone can change anything, so rules like "a balance is never negative" cannot be protected;
- the compiler must rebuild everything for every change.

Rust gives you three tools: **modules** to organise and protect code inside a crate, **crates** as units of compilation and sharing, and **packages** as units of building and shipping with Cargo.

Think of a building. The **package** is the whole building with its address plan (`Cargo.toml`). A **crate** is a wing you can build and open separately: the entrance hall that visitors use (library) or the machine room that runs things (binary). **Modules** are the floors and rooms inside the wing. Doors can be locked (private) or open (`pub`).

```text
package  p04-modules                        (folder with Cargo.toml)
├── library crate  p04_modules              (root file: src/lib.rs)
│     └── module tree:   crate
│                          ├── geometry
│                          │     ├── circle
│                          │     └── rectangle
│                          └── percent
└── binary crate   p04-modules              (root file: src/main.rs)
        uses the library crate like any other crate
```

Package rules (from Cargo): a package has **at least one** crate, **at most one** library crate, and **any number** of binary crates.

## 2. Modules in a single file

First, modules written inline with `mod name { ... }` — easiest for seeing the rules.

```rust
mod restaurant {
    pub mod front {
        pub fn seat_guest() {
            println!("front: guest seated");
        }

        fn clean_tables() {
            println!("front: tables cleaned");
        }

        pub fn close() {
            clean_tables();
            super::kitchen::wash_dishes();
        }
    }

    pub mod kitchen {
        pub fn wash_dishes() {
            println!("kitchen: dishes washed");
        }

        pub fn cook(order: &str) {
            self::prepare(order);
            crate::restaurant::front::seat_guest();
        }

        fn prepare(order: &str) {
            println!("kitchen: preparing {order}");
        }
    }
}

fn main() {
    // relative path from the crate root (where main lives)
    restaurant::front::seat_guest();
    // absolute path, starting at the crate root
    crate::restaurant::kitchen::cook("soup");
    restaurant::front::close();
}
```

**What happens:**

1. `mod restaurant { ... }` defines a module. Inside it, `pub mod front` and `pub mod kitchen` are two child modules. The file's top level is the **crate root**, a module named `crate`.
2. The module tree is: `crate` → `restaurant` → `front`, `kitchen`. The `main` function lives in `crate`.
3. In `main`, `restaurant::front::seat_guest()` is a **relative path**: it starts from the current module (the crate root) and walks down by name. `crate::restaurant::kitchen::cook("soup")` is the **absolute path** for the same kind of walk: it starts with the word `crate`.
4. Inside `front::close`, `super::kitchen::wash_dishes()` goes **up one level** (`super` = parent module, `restaurant`) and then down into `kitchen`.
5. Inside `kitchen::cook`, `self::prepare(order)` means "the item `prepare` in this very module" (`self` here is the current module — not the same as the `self` of methods). `crate::restaurant::front::seat_guest()` shows an absolute path from the inside.
6. `clean_tables` and `prepare` have no `pub`: they are **private**, usable only by code in the same module (and its children). `close` and `cook` are public, so they can call them.

```text
front: guest seated
kitchen: preparing soup
front: guest seated
front: tables cleaned
kitchen: dishes washed
```

> 📏 **Convention:** use relative paths for nearby items and `crate::` paths for far ones. If you later move a module, absolute paths from `crate::` stay valid for code that is not itself moved. Choose the style your project already uses.

### The privacy rules

There are only two rules:

1. **Everything is private by default.** A private item can be used by code in the same module and in the modules nested inside it (its children and grandchildren). Parents and outsiders cannot use it.
2. **`pub` opens an item for the world outside.** To reach an item by a path, **every** step of the path must be visible to you: the modules on the way as well as the item.

Children can look at their parents' private items; parents cannot look into their children's private items. A child is "inside the house"; a parent is a neighbour.

Calling a private function from outside:

```rust
mod restaurant {
    pub mod front {
        fn clean_tables() {
            println!("front: tables cleaned");
        }
    }
}

fn main() {
    restaurant::front::clean_tables();
}
```

```text
error[E0603]: function `clean_tables` is private
  --> src/main.rs:10:24
   |
10 |     restaurant::front::clean_tables();
   |                        ^^^^^^^^^^^^ private function
   |
note: the function `clean_tables` is defined here
  --> src/main.rs:3:9
   |
 3 |         fn clean_tables() {
   |         ^^^^^^^^^^^^^^^^^
```

The module `front` is public, but the function inside is not. Fix: add `pub`, or — better — ask whether outsiders really need it.

A private **module** blocks the whole path:

```text
error[E0603]: module `front` is private
  --> src/main.rs:10:17
   |
10 |     restaurant::front::seat_guest();
   |                 ^^^^^  ---------- function `seat_guest` is not publicly re-exported
   |                 |
   |                 private module
```

(That happens when the code says `mod front` instead of `pub mod front`, even though `seat_guest` itself is `pub`.)

### Privacy of structs and enums

`pub struct` makes the *type* public, but each **field** has its own privacy. Fields are private unless marked `pub`. **Enum variants** are different: if the enum is public, all its variants are public.

```rust
mod bank {
    pub struct Account {
        pub owner: String,
        balance_cents: u64,
    }

    impl Account {
        pub fn new(owner: &str) -> Account {
            Account {
                owner: String::from(owner),
                balance_cents: 0,
            }
        }

        pub fn deposit(&mut self, cents: u64) {
            self.balance_cents += cents;
        }

        pub fn balance_cents(&self) -> u64 {
            self.balance_cents
        }
    }

    #[derive(Debug)]
    pub enum Kind {
        Checking,
        Savings,
    }
}

fn main() {
    let mut account = bank::Account::new("alice");
    account.deposit(500);
    println!("{} has {} cents", account.owner, account.balance_cents());
    println!("{:?}", bank::Kind::Savings);
}
```

**What happens:**

1. `owner` is `pub`: anyone can read it. `balance_cents` is private: only code inside `mod bank` can touch the field.
2. Outsiders get a balance through the **method** `balance_cents()` and change it only through `deposit`. This is **encapsulation**: the rules of the type live in one place.
3. `bank::Kind::Savings` works without marking each variant `pub`: variants of a public enum are public.

```text
alice has 500 cents
Savings
```

Trying to touch the private field from outside:

```text
error[E0616]: field `balance_cents` of struct `Account` is private
  --> src/main.rs:16:13
   |
16 |     account.balance_cents = 1_000_000;
   |             ^^^^^^^^^^^^^ private field
```

And trying to create an `Account` by listing the fields yourself (skipping `new`):

```text
error[E0451]: field `balance_cents` of struct `Account` is private
  --> src/main.rs:11:9
   |
 9 |     let account = bank::Account {
   |                   ------------- in this type
10 |         owner: String::from("mallory"),
11 |         balance_cents: 1_000_000,
   |         ^^^^^^^^^^^^^ private field
```

> 🔒 **Security:** this is why private fields matter. If a struct has a private field, the **only** way to create or change one is through your code, so your checks always run. A type like `Percent` (section 5) can guarantee "the value is between 0 and 100" because nobody can build a `Percent(250)` from outside. Keep the public surface of every module as small as possible: each `pub` item is a promise you must keep and a door someone can misuse.

## 3. `use`: shorter names

Writing `restaurant::front::seat_guest()` everywhere is tiring. A **`use`** declaration brings a path into scope:

```rust
mod shapes {
    pub mod round {
        pub fn describe() -> &'static str {
            "round"
        }
    }

    pub mod flat {
        pub fn describe() -> &'static str {
            "flat"
        }
    }

    pub fn secret_sauce() -> u32 {
        42
    }
}

// bring a module into scope: call functions as `round::describe()`
use shapes::round;
// bring a function into scope directly
use shapes::secret_sauce;
// rename with `as`, handy when two names collide
use shapes::flat::describe as describe_flat;
// several paths from one prefix
use std::fmt::{self, Display};
// two different `Result` types, told apart with `as`
use std::fmt::Result as FmtResult;
use std::io::Result as IoResult;

struct Tag(u32);

impl Display for Tag {
    fn fmt(&self, f: &mut fmt::Formatter) -> FmtResult {
        write!(f, "tag-{}", self.0)
    }
}

fn pretend_to_read() -> IoResult<u32> {
    Ok(7)
}

fn main() {
    println!("{}", round::describe());
    println!("{}", describe_flat());
    println!("{}", secret_sauce());
    println!("{}", Tag(9));
    println!("{:?}", pretend_to_read().ok());
}
```

**What happens:**

1. `use shapes::round;` makes the name `round` available, so we write `round::describe()`. The module name in front of the function tells the reader where it comes from.
2. `use shapes::secret_sauce;` imports the function itself, so we call `secret_sauce()` directly.
3. `use ... describe as describe_flat;` renames on import. Two things called `describe` would clash; `as` avoids that.
4. `use std::fmt::{self, Display};` is a **nested import**: `self` means `std::fmt` itself, so both `fmt` and `Display` become available. It saves repeating the prefix.
5. The standard library has more than one type called `Result` (one for formatting, one for input/output). Renaming them with `as` lets both live in the same file.
6. `use` only creates a short name. It does not copy anything and costs nothing at run time.

```text
round
flat
42
tag-9
Some(7)
```

> 📏 **Convention (idiomatic `use`):**
> - For **functions**, import the *parent module* (`use shapes::round;` then `round::describe()`), so the call shows where the function lives.
> - For **types** (structs, enums, traits), import the *type itself* (`use std::collections::HashMap;`).
> - Avoid the glob `use module::*;` in normal code: you cannot see where names come from. Two accepted uses: `use super::*;` in `mod tests`, and prelude-style modules.
> - Put `use` lines at the top of the file, ordered: standard library first, then other crates, then your own modules. `cargo fmt` sorts them.

### Re-exporting with `pub use`

A path may be long and deep because that is how you organised the files, but users of your library do not want to know that. `pub use` makes an item available under a shorter public path:

```text
pub use geometry::circle::Circle;   // in lib.rs: now `my_crate::Circle` works too
```

Section 5 uses this.

## 4. Visibility levels

Besides private and `pub`, there are narrower forms:

| Written | Visible to |
|---|---|
| *(nothing)* | the current module and the modules inside it |
| `pub(super)` | the parent module (and everything inside it) |
| `pub(crate)` | the whole crate, but not other crates |
| `pub` | everyone who can reach the path, including other crates |

`pub(crate)` is the right choice for helper functions that several modules of your crate share but that should not be part of the public API.

## 5. Modules in separate files

Real projects keep each module in its own file. The rule is simple: writing `mod name;` (with a semicolon instead of a block) tells the compiler "the module `name` lives in another file; load it". The compiler looks for it in this order:

1. `name.rs` next to the current file, or
2. `name/mod.rs` (the older style; both work).

For a module nested inside `name`, the file goes into a folder called `name/`: `name/child.rs`.

We will build a package with a library crate and a binary crate. Create it with:

```text
cd ~/projects/rust
cargo new code/p04-modules
cd code/p04-modules
mkdir tests src/geometry
```

The final layout:

```text
p04-modules/
├── Cargo.toml
├── src/
│   ├── main.rs              ← binary crate root (the program)
│   ├── lib.rs               ← library crate root
│   ├── geometry.rs          ← module `geometry`  (declares its children)
│   ├── geometry/
│   │   ├── circle.rs        ← module `geometry::circle`
│   │   └── rectangle.rs     ← module `geometry::rectangle`
│   └── percent.rs           ← module `percent`
└── tests/
    └── geometry_api.rs      ← integration tests
```

`cargo new` created `main.rs` only. The library appears as soon as you add `src/lib.rs`: Cargo sees the file and builds a library crate named after the package, with hyphens turned into underscores: `p04_modules`.

**`src/lib.rs`** — the library root:

```rust
pub mod geometry;
pub mod percent;

// re-export: callers can write `p04_modules::Circle`
pub use geometry::circle::Circle;
```

**`src/geometry.rs`** — the module `geometry`, which declares its two children:

```rust
pub mod circle;
pub mod rectangle;

// visible everywhere inside this crate, but not to other crates
pub(crate) fn round2(value: f64) -> f64 {
    (value * 100.0).round() / 100.0
}
```

**`src/geometry/circle.rs`**:

```rust
use std::f64::consts::PI;

use super::round2;

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Circle {
    radius: f64,
}

impl Circle {
    /// Returns `None` unless the radius is a positive, finite number.
    pub fn new(radius: f64) -> Option<Circle> {
        if radius > 0.0 && radius.is_finite() {
            Some(Circle { radius })
        } else {
            None
        }
    }

    pub fn radius(&self) -> f64 {
        self.radius
    }

    pub fn area(&self) -> f64 {
        round2(PI * self.radius * self.radius)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn area_of_unit_circle() {
        let c = Circle::new(1.0).unwrap();
        assert_eq!(c.area(), 3.14);
    }

    #[test]
    fn rejects_bad_radius() {
        assert_eq!(Circle::new(0.0), None);
        assert_eq!(Circle::new(-2.0), None);
        assert_eq!(Circle::new(f64::NAN), None);
    }

    #[test]
    fn unit_tests_may_touch_private_fields() {
        let c = Circle { radius: 2.0 };
        assert_eq!(c.radius, 2.0);
    }
}
```

**`src/geometry/rectangle.rs`**:

```rust
use super::round2;

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Rectangle {
    pub width: f64,
    pub height: f64,
}

impl Rectangle {
    pub fn area(&self) -> f64 {
        round2(self.width * self.height)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn area_is_rounded() {
        let r = Rectangle { width: 1.234, height: 2.0 };
        assert_eq!(r.area(), 2.47);
    }
}
```

**`src/percent.rs`**:

```rust
/// A percentage from 0 to 100. The field is private,
/// so no value outside that range can ever exist.
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Percent(u8);

impl Percent {
    pub fn new(value: u8) -> Option<Percent> {
        if value <= 100 {
            Some(Percent(value))
        } else {
            None
        }
    }

    pub fn value(&self) -> u8 {
        self.0
    }

    pub fn of(&self, amount: u32) -> u32 {
        amount * u32::from(self.0) / 100
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_0_to_100() {
        assert!(Percent::new(0).is_some());
        assert!(Percent::new(100).is_some());
    }

    #[test]
    fn rejects_101() {
        assert_eq!(Percent::new(101), None);
    }

    #[test]
    fn computes_share() {
        assert_eq!(Percent::new(25).unwrap().of(200), 50);
    }
}
```

**`src/main.rs`** — the program uses the library like any outside crate:

```rust
use p04_modules::geometry::rectangle::Rectangle;
use p04_modules::percent::Percent;
use p04_modules::Circle;

fn main() {
    let Some(circle) = Circle::new(2.0) else {
        println!("bad radius");
        return;
    };
    let rect = Rectangle {
        width: 3.5,
        height: 2.0,
    };
    println!("circle area    = {}", circle.area());
    println!("rectangle area = {}", rect.area());

    if let Some(discount) = Percent::new(15) {
        println!("15% of 200 = {}", discount.of(200));
    }
    println!("{:?}", Percent::new(150));
}
```

**`tests/geometry_api.rs`** — integration tests:

```rust
use p04_modules::geometry::rectangle::Rectangle;
use p04_modules::{Circle, percent::Percent};

#[test]
fn circle_through_the_public_api() {
    let circle = Circle::new(1.0).expect("valid radius");
    assert_eq!(circle.area(), 3.14);
    assert_eq!(circle.radius(), 1.0);
}

#[test]
fn rectangle_through_the_public_api() {
    let rect = Rectangle { width: 2.0, height: 3.0 };
    assert_eq!(rect.area(), 6.0);
}

#[test]
fn percent_cannot_exceed_100() {
    assert!(Percent::new(100).is_some());
    assert!(Percent::new(101).is_none());
}
```

**What happens:**

1. `lib.rs` is the root of the library crate. `pub mod geometry;` tells the compiler to load `src/geometry.rs` and make the module public. `pub mod percent;` loads `src/percent.rs`.
2. `geometry.rs` is itself a module with children: `pub mod circle;` loads `src/geometry/circle.rs`. The folder name `geometry/` matches the module name.
3. `round2` is `pub(crate)`: `circle` and `rectangle` call it as `super::round2` (`super` is the parent module `geometry`), but code in other crates, such as `main.rs`, cannot.
4. In `circle.rs`, the field `radius` is private, so the only way to get a `Circle` is `Circle::new`, which refuses a zero, negative, infinite or `NaN` radius and returns `None`. `NaN` ("not a number") is a special `f64` value that appears for invalid results such as `0.0 / 0.0`.
5. `Percent(u8)` is a tuple struct with a private field, an application of "make illegal states unrepresentable" ([Lesson 04.3](03-enums.md)). Its `of` method turns the value into a share of an amount.
6. The lines starting with `///` are **documentation comments**. They describe the item that follows and show up in generated documentation; [Lesson 05.7 — Documentation](../05-rust-intermediate/07-documentation.md) covers them.
7. `pub use geometry::circle::Circle;` in `lib.rs` is a re-export: outsiders can write `p04_modules::Circle` as well as the long path.
8. `main.rs` is a *different crate* that depends on the library in the same package. It refers to it by the crate name `p04_modules` (hyphen turned into underscore), not by `crate::`. It can only see `pub` items.
9. The unit tests live in `mod tests` inside each file. A child module can see its parent's private items, so the last test of `circle.rs` may build `Circle { radius: 2.0 }` directly. The tests in `tests/` are compiled as an outside crate and can use only the public API.

Run it:

```text
cargo run
```

```text
circle area    = 12.57
rectangle area = 7
15% of 200 = 30
None
```

`rectangle area = 7` and not `7.0`: `println!` prints a whole `f64` without `.0` when you use `{}`.

And the tests:

```text
cargo test
```

```text
running 7 tests
test geometry::circle::tests::unit_tests_may_touch_private_fields ... ok
test geometry::circle::tests::rejects_bad_radius ... ok
test geometry::circle::tests::area_of_unit_circle ... ok
test percent::tests::accepts_0_to_100 ... ok
test geometry::rectangle::tests::area_is_rounded ... ok
test percent::tests::computes_share ... ok
test percent::tests::rejects_101 ... ok
test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 0 tests
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

running 3 tests
test circle_through_the_public_api ... ok
test percent_cannot_exceed_100 ... ok
test rectangle_through_the_public_api ... ok
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Cargo prints a "Running ..." line before each group, and a final empty group for documentation tests; they are omitted here. The order of the tests inside a group may differ on your machine, because tests run in parallel.) The three groups are: the unit tests of the library (7), the unit tests of `main.rs` (none), and the integration tests (3). The test name shows the module path, such as `geometry::circle::tests::rejects_bad_radius` — the module tree is visible in the output.

> 📏 **Convention:** keep `main.rs` thin: it reads arguments, calls the library, prints results. Put the real logic in the library crate (`lib.rs` and modules). The library can then be tested without running the program, reused by other programs, and later split further. This is the layout of the projects in this course from now on.

> 📏 **Convention:** one module per concept, file names in `snake_case` that match the module name, and small modules: if a file grows past a few hundred lines, split it.

### More binaries and the other special folders

| Path | What it is |
|---|---|
| `src/main.rs` | the default binary crate (named after the package) |
| `src/lib.rs` | the library crate |
| `src/main.rs` | an additional binary; run it with `cargo run --bin tool` |
| `tests/*.rs` | integration tests; each file is its own crate |
| `examples/*.rs` | small example programs for users of your library, run with `cargo run --example name` |
| `benches/*.rs` | benchmarks (measuring speed, [Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)) |

## 6. Gotchas with files

### A file that is not declared is not compiled

Cargo does **not** scan your folder for `.rs` files. Only files reachable through `mod name;` declarations from the crate root are part of the crate. Create `src/extra.rs` with a test that obviously fails:

```rust
#[test]
fn this_test_fails() {
    assert_eq!(1 + 1, 3);
}
```

and run `cargo test`: all tests pass, and the new test is not even mentioned. The file is simply ignored, because nobody wrote `mod extra;`. If you add a file and "nothing happens", check that it is declared.

### The file for a declared module must exist

```text
error[E0583]: file not found for module `missing`
 --> src/lib.rs:6:1
  |
6 | pub mod missing;
  | ^^^^^^^^^^^^^^^^
  |
  = help: to create the module `missing`, create file "src/missing.rs" or "src/missing/mod.rs"
  = note: if there is a `mod missing` elsewhere in the crate already, import it with `use crate::...` instead
```

The message tells you the two places where the file may be.

### A binary crate is not the library

In `main.rs`, `crate::` means the *binary* crate, which here contains no `geometry` module:

```text
error[E0433]: cannot find `geometry` in the crate root
 --> src/main.rs:2:12
  |
2 | use crate::geometry::rectangle::Rectangle;
  |            ^^^^^^^^ could not find `geometry` in the crate root
```

Use the library's crate name: `use p04_modules::geometry::rectangle::Rectangle;`.

### Forgetting `pub` on the module

```text
error[E0603]: module `geometry` is private
 --> src/main.rs:2:18
  |
2 | use p04_modules::geometry::rectangle::Rectangle;
  |                  ^^^^^^^^             --------- struct `Rectangle` is not publicly re-exported
  |                  |
  |                  private module
  |
note: the module `geometry` is defined here
 --> src/lib.rs:1:1
  |
1 | mod geometry;
  | ^^^^^^^^^^^^
```

In `lib.rs` write `pub mod geometry;`.

### Calling a `pub(crate)` item from another crate

```text
error[E0603]: function `round2` is private
 --> src/main.rs:6:43
  |
6 |     println!("{}", p04_modules::geometry::round2(1.234));
  |                                           ^^^^^^ private function
```

`pub(crate)` stops at the crate border. The message just says "private" because, to that caller, it is.

### Typos and missing imports

```text
error[E0433]: cannot find `geometri` in `p04_modules`
 --> src/main.rs:2:18
  |
2 | use p04_modules::geometri::rectangle::Rectangle;
  |                  ^^^^^^^^ could not find `geometri` in `p04_modules`
```

```text
error[E0433]: cannot find module or crate `geometry` in this scope
 --> src/main.rs:9:16
  |
9 |     let rect = geometry::rectangle::Rectangle {
  |                ^^^^^^^^ use of unresolved module or unlinked crate `geometry`
  |
help: to make use of source file src/geometry.rs, use `mod geometry` in this file to declare the module
```

The second one is a missing `use` (the compiler's hint about `mod geometry` is wrong in this case, because the module belongs to the library crate, not to `main.rs`). Read the first lines first: *what* could not be found and *where*.

## 7. Using other people's crates, and bigger projects

Everything above works the same for crates written by others. You list them in the `[dependencies]` section of `Cargo.toml` (that file is written in **TOML**, a simple format of `key = value` lines and `[sections]`), Cargo downloads them from crates.io (the public crate registry), and you `use` them by name:

```text
[dependencies]
rand = "0.10"
```

[Lesson 05.8 — Cargo: dependencies and versions](../05-rust-intermediate/08-cargo-dependencies-and-versions.md) teaches this in full, including versions and features. When one project grows into several related packages, Cargo can build them together as a **workspace**; that also comes in Lesson 05.8.

> 🔒 **Security:** every dependency is code you run on your machine and ship to your users. Add them deliberately; [Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md) shows how to check them.

> ⚡ **Performance:** modules and files do not change how fast your program runs: the compiler sees the whole crate as one unit. What they change is **build time** and **readability**. Splitting a big program into several crates lets Cargo rebuild only the crates that changed.

## 8. Testing across modules

> 🧪 **Testing:**
> - **Unit tests** go in `#[cfg(test)] mod tests` at the bottom of the file they test. They can use private items of that file (through `use super::*;`), so you can test small helpers.
> - **Integration tests** go in `tests/`. They use the library as an outsider would (only `pub` items), so they check that your public API is complete and usable — including the re-exports.
> - A test cannot run if its file is not declared with `mod`. If a new test never appears in the output, check the `mod` line.
> - Run only part of the tests with a name filter: `cargo test percent` runs the tests whose path contains `percent`.

## Common mistakes

> ⚠️ **Common mistake:** creating `src/something.rs` and forgetting `mod something;` — the file is silently ignored (section 6).

> ⚠️ **Common mistake:** putting `pub` on everything "to make errors go away". Each `pub` widens your public API. Start private and open only what callers need.

> ⚠️ **Common mistake:** using `crate::` in `main.rs` to reach modules of `lib.rs` (`E0433`, section 6). The binary and the library are different crates.

> ⚠️ **Common mistake:** `pub struct` with private fields, then trying to build it with struct literal syntax from another module (`E0451`). Provide a `new` function.

> ⚠️ **Common mistake:** mixing the name of the package (`p04-modules`, with a hyphen) and the name of the crate in code (`p04_modules`, with an underscore).

> ⚠️ **Common mistake:** forgetting `mod tests` needs `use super::*;` to see the code above it.

> ⚠️ **Common mistake:** committing the `target/` folder to Git. Check `.gitignore` contains `target/`. In a package with a binary, **do** commit `Cargo.lock` (it fixes exact dependency versions); for a library crate that you publish, practice varies, but keeping it is harmless.

## Try it yourself

1. Build the `p04-modules` package from section 5: create the folders, type each file, then run `cargo fmt`, `cargo run`, `cargo test`, `cargo clippy`. All must pass without warnings.
2. Break it on purpose, one change at a time, read the error and undo the change:
   - remove `pub` from `pub mod geometry;` in `lib.rs`;
   - add `pub mod missing;` to `lib.rs`;
   - in `main.rs` write `Circle { radius: 2.0 }`;
   - create `src/extra.rs` with a failing test and check that `cargo test` ignores it; then add `mod extra;` to `lib.rs` and watch it fail.
3. Add a new module `temperature` in its own file with a public function `c_to_f(c: f64) -> f64` and a unit test. Declare it in `lib.rs`, re-export the function with `pub use temperature::c_to_f;`, call it from `main.rs`, and add an integration test.
4. Commit one step at a time, as in [Lesson 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md):

   ```text
   cd ~/projects/rust
   git status
   git add code/p04-modules
   git commit -m "feat(p04): geometry and percent modules with tests"
   ```

   Check with `git status` that no `target/` folder is staged. If `git status` lists `target/`, add `target/` to a `.gitignore` file first.

## Exercises

### Exercise 1 — A module with a private helper (easy)

In one file, create `mod calc` with public functions `square(x: i32)` and `cube(x: i32)` that both use a **private** helper `multiply(a, b)`. Call the public functions from `main`. Write a unit test module inside `calc`; show that a test can call the private helper.

<details>
<summary>Hint</summary>

Only `square` and `cube` get `pub`. Inside `calc`, add `#[cfg(test)] mod tests { use super::*; ... }`; because `tests` is a child of `calc`, it can see `multiply`.

</details>

<details>
<summary>Answer</summary>

```rust
mod calc {
    pub fn square(x: i32) -> i32 {
        multiply(x, x)
    }

    pub fn cube(x: i32) -> i32 {
        multiply(multiply(x, x), x)
    }

    // private helper: only code inside `calc` can call it
    fn multiply(a: i32, b: i32) -> i32 {
        a * b
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn square_and_cube() {
            assert_eq!(square(4), 16);
            assert_eq!(cube(-3), -27);
        }

        #[test]
        fn tests_can_call_private_helpers() {
            assert_eq!(multiply(6, 7), 42);
        }
    }
}

fn main() {
    println!("{}", calc::square(5));
    println!("{}", calc::cube(2));
}
```

Output of `cargo run`:

```text
25
8
```

`cargo test` reports 2 passed. Why it works: a child module (`tests`) may use the private items of its parent (`calc`), while `main` may not call `calc::multiply`.

</details>

### Exercise 2 — A type that cannot be wrong (medium)

Inside `mod weather`, define `pub struct Temperature { celsius: f64 }` (private field) with `pub fn new(celsius: f64) -> Option<Temperature>` that refuses values below absolute zero (−273.15), plus `celsius()` and `fahrenheit()` methods. Write tests for a normal value, the border value −273.15, and −300. In `main`, show that `weather::Temperature { celsius: -500.0 }` does not compile.

<details>
<summary>Hint</summary>

The field has no `pub`, so outside code must go through `new`. The check is `celsius >= -273.15`. Remember the Fahrenheit formula from Lesson 04.1: `c * 9.0 / 5.0 + 32.0`.

</details>

<details>
<summary>Answer</summary>

```rust
mod weather {
    #[derive(Debug, Clone, Copy, PartialEq)]
    pub struct Temperature {
        celsius: f64,
    }

    impl Temperature {
        pub fn new(celsius: f64) -> Option<Temperature> {
            if celsius >= -273.15 {
                Some(Temperature { celsius })
            } else {
                None
            }
        }

        pub fn celsius(&self) -> f64 {
            self.celsius
        }

        pub fn fahrenheit(&self) -> f64 {
            self.celsius * 9.0 / 5.0 + 32.0
        }
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn normal_values_are_accepted() {
            let t = Temperature::new(100.0).unwrap();
            assert_eq!(t.fahrenheit(), 212.0);
        }

        #[test]
        fn below_absolute_zero_is_rejected() {
            assert_eq!(Temperature::new(-300.0), None);
            assert!(Temperature::new(-273.15).is_some());
        }
    }
}

use weather::Temperature;

fn main() {
    match Temperature::new(36.6) {
        Some(t) => println!("{} C = {:.1} F", t.celsius(), t.fahrenheit()),
        None => println!("impossible temperature"),
    }
    println!("{:?}", Temperature::new(-500.0));
}
```

Output of `cargo run`:

```text
36.6 C = 97.9 F
None
```

The attempt `weather::Temperature { celsius: -500.0 }` in `main` fails with `error[E0451]: field `celsius` of struct `Temperature` is private`.

Why it works: because the field is private, every `Temperature` in the program was created by `new`, so every one is physically possible. The rule is enforced by the compiler, not by discipline.

</details>

### Exercise 3 — Move a module to its own file (medium)

Start with `cargo new code/p04-greeter`. In `src/main.rs`, a module `greeter` with `pub fn hello(name: &str) -> String` (returns `"Hello, NAME!"`) is written inline. Move it into `src/greeter.rs`, so that `main.rs` contains only `mod greeter;` and `main`. Keep a unit test next to the function. What does `cargo run` print for `hello("world")`?

<details>
<summary>Hint</summary>

The file `src/greeter.rs` contains the *content* of the module, without the `mod greeter { }` wrapper. In `main.rs` replace the whole block with `mod greeter;`.

</details>

<details>
<summary>Answer</summary>

`src/greeter.rs`:

```rust
pub fn hello(name: &str) -> String {
    format!("Hello, {name}!")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn greets_by_name() {
        assert_eq!(hello("Ferris"), "Hello, Ferris!");
    }
}
```

`src/main.rs`:

```rust
mod greeter;

fn main() {
    println!("{}", greeter::hello("world"));
}
```

Output of `cargo run`:

```text
Hello, world!
```

`cargo test` reports 1 passed.

Why it works: `mod greeter;` tells the compiler to load `src/greeter.rs` and treat its content as the body of the module `greeter`. The function must stay `pub` because `main` is outside the module.

</details>

### Exercise 4 — A library and a program in one package (hard)

Create the package `code/p04-text-tools` (`cargo new`). Write a library with two modules in two files: `counting` with `pub fn word_count(text: &str) -> usize` and `shouting` with `pub fn shout(text: &str) -> String` (upper-case text plus `!`). Re-export both functions at the crate root with `pub use`. The program in `main.rs` must call them through the root paths. Add unit tests in each module file and one integration test in `tests/api.rs`. Everything must pass `cargo test`, `cargo clippy` and `cargo fmt`.

(In the answer the package is named `ex4`; with your package name `p04-text-tools` the crate name in `use` lines becomes `p04_text_tools`.)

<details>
<summary>Hint</summary>

`src/lib.rs`: `pub mod counting; pub mod shouting; pub use counting::word_count; pub use shouting::shout;`. In `main.rs` and in `tests/api.rs` start with `use p04_text_tools::{shout, word_count};`. `text.split_whitespace().count()` counts words; `text.to_uppercase()` makes upper case.

</details>

<details>
<summary>Answer</summary>

`src/lib.rs`:

```rust
pub mod counting;
pub mod shouting;

pub use counting::word_count;
pub use shouting::shout;
```

`src/counting.rs`:

```rust
/// Counts the words separated by whitespace.
pub fn word_count(text: &str) -> usize {
    text.split_whitespace().count()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_words() {
        assert_eq!(word_count("one two  three"), 3);
    }

    #[test]
    fn empty_text_has_no_words() {
        assert_eq!(word_count(""), 0);
        assert_eq!(word_count("   "), 0);
    }
}
```

`src/shouting.rs`:

```rust
/// Upper-cases the text and adds an exclamation mark.
pub fn shout(text: &str) -> String {
    format!("{}!", text.to_uppercase())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn shouts() {
        assert_eq!(shout("hi there"), "HI THERE!");
    }
}
```

`src/main.rs` (for the package named `ex4`; use your own crate name):

```rust
use ex4::{shout, word_count};

fn main() {
    let text = "modules keep code tidy";
    println!("{} words", word_count(text));
    println!("{}", shout(text));
}
```

`tests/api.rs`:

```rust
use ex4::{shout, word_count};

#[test]
fn both_functions_are_reachable_from_the_crate_root() {
    assert_eq!(word_count("a b c"), 3);
    assert_eq!(shout("ok"), "OK!");
}
```

Output of `cargo run`:

```text
4 words
MODULES KEEP CODE TIDY!
```

`cargo test` reports 3 unit tests (2 + 1) and 1 integration test, all passed, and `cargo clippy` prints nothing.

Why it works: the library crate is created as soon as `src/lib.rs` exists. The two `pub mod` lines load the two files, and the two `pub use` lines let callers write `crate_name::word_count` instead of `crate_name::counting::word_count`. The test in `tests/` compiles like an outside user and proves that the re-exports really work.

</details>

## Quiz

1. What is the difference between a package, a crate and a module?

<details>
<summary>Answer</summary>

A package is a folder with `Cargo.toml` that holds one or more crates. A crate is one compilation unit, a binary or a library, starting from a root file. A module is a namespace and privacy boundary inside a crate.

</details>

2. What do `crate::`, `self::` and `super::` mean in a path?

<details>
<summary>Answer</summary>

`crate::` starts at the root of the current crate. `self::` starts at the current module. `super::` starts at the parent module.

</details>

3. Who can use a private item?

<details>
<summary>Answer</summary>

Code in the same module and in the modules nested inside it. Parent modules and outsiders cannot.

</details>

4. If `pub mod a { fn b() {} }`, can outside code call `a::b()`? Why?

<details>
<summary>Answer</summary>

No. The module `a` is public, but the function `b` is private (no `pub`), so the path is blocked at the last step (`E0603`).

</details>

5. Why does a struct with a private field need a `new` function?

<details>
<summary>Answer</summary>

Outside code cannot write the struct literal because it cannot name the private field (`E0451`). `new` is the controlled entrance where you can check the values, so no invalid instance can exist.

</details>

6. What does `mod geometry;` do, and where does the compiler look for the code?

<details>
<summary>Answer</summary>

It declares the module `geometry` whose body is in another file. The compiler loads `geometry.rs` next to the current file, or `geometry/mod.rs`. A nested module's file is in the `geometry/` folder.

</details>

7. Why can `main.rs` not write `crate::geometry::...` for a module that is declared in `lib.rs`?

<details>
<summary>Answer</summary>

`main.rs` and `lib.rs` are two separate crates of the same package. In `main.rs`, `crate` means the binary crate. The library is used by its crate name, such as `use p04_modules::geometry::...`.

</details>

8. Where do unit tests and integration tests live, and what can each see?

<details>
<summary>Answer</summary>

Unit tests live in `#[cfg(test)] mod tests` inside the source file and can use private items of that module. Integration tests live in `tests/*.rs`, are separate crates and can use only the public API.

</details>

## Summary

- A **package** (one `Cargo.toml`) holds crates: at most one library and any number of binaries. A **crate** is compiled as one unit. **Modules** organise code inside a crate into a tree.
- Paths use `::`. They start with `crate::` (absolute), with a module name or `self::` (relative), or with `super::` (parent).
- Everything is private by default; `pub` opens an item; `pub(crate)` opens it to the whole crate. Struct fields are private unless marked `pub`; variants of a public enum are public.
- Private fields plus a checking `new` give **encapsulation** — a type that cannot hold invalid data.
- `use` shortens paths; use `as` to rename, `{a, b}` to group, `pub use` to re-export a shorter public path. Idiom: import the parent module for functions, the item itself for types.
- `mod name;` loads `name.rs` (or `name/mod.rs`); nested modules live in the `name/` folder. A file that is never declared with `mod` is never compiled.
- Put the logic in `lib.rs` and modules; keep `main.rs` thin. The binary uses the library by its crate name (hyphens become underscores).
- Unit tests sit in each file; integration tests sit in `tests/` and see only the public API.

## Sources and further reading

- [The Rust Programming Language — Ch. 7 Managing Growing Projects with Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html) — the official chapter on this topic (its sub-chapters 7.1–7.5 cover each part).
- [The Rust Programming Language — 7.2 Defining Modules to Control Scope and Privacy](https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html) — modules and `pub`.
- [The Rust Programming Language — 7.5 Separating Modules into Different Files](https://doc.rust-lang.org/book/ch07-05-separating-modules-into-different-files.html) — the file rules from section 5.
- [The Cargo Book — Package Layout](https://doc.rust-lang.org/cargo/guide/project-layout.html) — the standard folders (`src/bin`, `tests`, `examples`, `benches`).
- [The Rust Reference — Visibility and Privacy](https://doc.rust-lang.org/reference/visibility-and-privacy.html) — the exact privacy rules, including `pub(crate)` and `pub(in path)`.
- [Rust by Example — Modules](https://doc.rust-lang.org/rust-by-example/mod.html) — runnable examples of modules, visibility and `use`.

---

[🏠 Start](../START-HERE.md) · [← 04.5 if let, let else and patterns](05-if-let-let-else-and-patterns.md) · [📋 Phase 04](README.md) · [04.7 Vectors →](07-vectors.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
