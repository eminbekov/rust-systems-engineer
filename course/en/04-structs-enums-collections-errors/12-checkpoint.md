<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.11 Project: phone book](11-project-phone-book.md) · [📋 Phase 04](README.md) · [Phase 05 — Rust intermediate →](../05-rust-intermediate/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/12-checkpoint.md) <!-- nav -->

# Lesson 04.12 — Checkpoint

> **Phase 04 — Structs, enums, collections and errors** · Lesson 12 of 12 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Which parts of Phase 04 you already know well and which ones you must review.
- How to check yourself honestly: first write an answer, then open the hidden one.
- How to combine structs, enums, modules, collections and errors in three short programs, written from a specification and tested.
- How to do a complete Git workflow with a branch, a merge conflict and a tag, without help.

## Before you start

- You finished [Lesson 04.11 — Project: phone book](11-project-phone-book.md) and all lessons before it in this phase.
- Do the quiz **from memory**: no notes, no compiler. Write each answer on paper or in a text file first, and only then open the hidden answer. A checkpoint only helps if you try to remember before looking. (Researchers call this *retrieval practice*; it works much better than reading again.)
- For the practical tasks you may use the compiler and the previous lessons, but not the answers.
- Create a project for the tasks: `cd ~/projects/rust && cargo new code/p04-checkpoint && cd code/p04-checkpoint`. Use `src/bin/` for several programs: for example put task 1 in `src/bin/shapes.rs` and run it with `cargo run --bin shapes`; test it with `cargo test --bin shapes`.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A pause at the end of a phase to test what you learned before you go on. |
| **retrieval practice** | Learning by trying to recall an answer from memory instead of rereading. |
| **regression** | A bug where something that worked before stops working after a change. Tests exist to catch regressions. |
| **merge conflict** | A situation where Git cannot join two changes automatically because both edited the same lines. |
| **conflict markers** | The lines `<<<<<<<`, `=======` and `>>>>>>>` that Git writes into a file to show the two competing versions. |

## Self-test quiz

Twenty questions. A good result is 16 or more correct. For every wrong answer, the table in
"If you got something wrong" tells you which lesson to read again.

### Structs and methods

1. What is the difference between a normal struct and a tuple struct? Give one example of each.

<details>
<summary>Answer</summary>

A normal struct has named fields: `struct User { name: String, age: u32 }`. A tuple struct has fields without names,
only positions: `struct Meters(f64);` and you read the field with `.0`. A tuple struct is useful for a *newtype*, a new
type around an existing one.

</details>

2. What does this program do? Which line does not compile and why?

```rust
struct User {
    username: String,
    email: String,
    logins: u32,
}

fn main() {
    let a = User {
        username: String::from("ali"),
        email: String::from("ali@example.com"),
        logins: 3,
    };
    let b = User { email: String::from("new@example.com"), ..a };
    println!("{} {} {}", b.username, b.email, b.logins);
    println!("{}", a.email);
    println!("{}", a.logins);
    println!("{}", a.username);
}
```

<details>
<summary>Answer</summary>

The struct update syntax `..a` copies the fields that were not listed. `username` is a `String`, so it is **moved** from `a` to `b`;
`email` was given explicitly, so `a.email` is untouched; `logins` is a `u32` and is copied. The first three `println!` lines are fine.
The last one fails:

```text
error[E0382]: borrow of moved value: `a.username`
```

You can still use `a.email` and `a.logins`, but not `a.username` and not `a` as a whole.

</details>

3. What is the difference between `&self`, `&mut self` and `self` in a method? How do you call an *associated function* such as `new`?

<details>
<summary>Answer</summary>

`&self` borrows the value to read it. `&mut self` borrows it to change it. `self` takes ownership: the value is consumed (moved into the
method) and cannot be used afterwards by the caller. An associated function has no `self` parameter; you call it on the type with `::`,
for example `Counter::new()`, while a method is called on a value with a dot: `counter.add(5)`.

</details>

4. What does this program print?

```rust
struct Counter {
    value: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { value: 0 }
    }

    fn add(&mut self, n: u32) {
        self.value += n;
    }

    fn doubled(&self) -> u32 {
        self.value * 2
    }

    fn finish(self) -> u32 {
        self.value + 100
    }
}

fn main() {
    let mut c = Counter::new();
    c.add(5);
    c.add(2);
    println!("{}", c.doubled());
    println!("{}", c.finish());
}
```

<details>
<summary>Answer</summary>

```text
14
107
```

`add` is called twice, so `value` is 7. `doubled` reads it: 14. `finish` consumes the counter: 7 + 100 = 107. After `c.finish()` the variable `c` can no longer be used.

</details>

### Enums, `Option` and `match`

5. Why is an enum with data in its variants often better than a struct with several `Option` fields and flags?

<details>
<summary>Answer</summary>

An enum makes invalid combinations impossible: a value is exactly one variant, and each variant carries exactly the data it needs
(`Shape::Circle { radius }`, `Shape::Square(side)`). With a struct of optional fields you can build contradictory states (no radius and no side,
or both) and must check them by hand.

</details>

6. What error does this code give, and what does it protect you from?

```rust
enum Light {
    Red,
    Yellow,
    Green,
}

fn main() {
    let l = Light::Red;
    let text = match l {
        Light::Red => "stop",
        Light::Green => "go",
    };
    println!("{text}");
    let _ = Light::Yellow;
}
```

<details>
<summary>Answer</summary>

```text
error[E0004]: non-exhaustive patterns: `Light::Yellow` not covered
```

A `match` must be *exhaustive*: it must handle every variant. This protects you from forgetting a case, and when you add a new variant later the compiler shows you every `match` that needs updating.

</details>

7. Rust has no `null`. What do you use for "maybe there is a value", and what are three ways to get the value out safely?

<details>
<summary>Answer</summary>

`Option<T>`: `Some(value)` or `None`. Safe ways: `match` (handle both cases), `if let Some(x) = opt { ... }` (act only when present), and
`unwrap_or(default)` / `unwrap_or_default()` (fall back to a default). Also possible: `let Some(x) = opt else { return ...; };`, and `?` in a function that returns `Option`.
`unwrap()` and `expect()` panic on `None`, so use them only when `None` is impossible.

</details>

8. What does this program print?

```rust
fn describe(n: i32) -> String {
    match n {
        i32::MIN..=-1 => String::from("negative"),
        0 => String::from("zero"),
        x if x % 2 == 0 => format!("even {x}"),
        1..=9 => String::from("small odd"),
        other => format!("big odd {other}"),
    }
}

fn main() {
    for n in [-5, 0, 4, 7, 21] {
        println!("{n}: {}", describe(n));
    }
}
```

<details>
<summary>Answer</summary>

```text
-5: negative
0: zero
4: even 4
7: small odd
21: big odd 21
```

Arms are tried from top to bottom and the first one that fits wins. `4` is not negative or zero, and the guard `x % 2 == 0` is true, so it is "even 4". `7` fails the guard and fits `1..=9`. `21` falls to the last arm, which binds the value to `other`.

</details>

### `if let`, `let else` and patterns

9. What does `let Some(c) = text.chars().next() else { return None; };` do? When is it better than `match`?

<details>
<summary>Answer</summary>

If the right side is `Some(c)`, the variable `c` is created and the code continues. Otherwise the `else` block runs, and it must leave the current path (`return`, `break`, `continue` or `panic!`).
It is better than `match` when you want the "happy path" to stay unindented and the failure case is a quick exit (an *early return*).

</details>

10. What does this program print?

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let data = ((1, 2), Point { x: 10, y: 20 });
    let ((a, _), Point { x, .. }) = data;
    println!("{a} {x}");

    let pair = (3, -3);
    match pair {
        (0, y) => println!("on the y axis at {y}"),
        (x, 0) => println!("on the x axis at {x}"),
        (x, y) if x + y == 0 => println!("opposites"),
        _ => println!("something else"),
    }
}
```

<details>
<summary>Answer</summary>

```text
1 10
opposites
```

The pattern takes the first number of the inner tuple (`_` ignores the second), and `x` of the `Point` (`..` ignores the rest of the fields). The `match` tries the arms in order: `(3, -3)` does not fit `(0, y)` or `(x, 0)`, and the guard of the third arm is true.

</details>

### Modules, crates and packages

11. What is the difference between a package, a crate and a module?

<details>
<summary>Answer</summary>

A *module* is a namespace inside a crate (`mod something`) that groups code and controls visibility. A *crate* is one compilation unit: a binary (has `main`) or a library (`lib.rs`). A *package* is what Cargo manages: one `Cargo.toml` with one or more crates (at most one library crate and any number of binaries).

</details>

12. What is the compiler's message for this code, and how do you fix it?

```rust
mod bank {
    pub fn deposit() -> u32 {
        secret() + 1
    }

    fn secret() -> u32 {
        41
    }
}

fn main() {
    println!("{}", bank::deposit());
    println!("{}", bank::secret());
}
```

<details>
<summary>Answer</summary>

```text
error[E0603]: function `secret` is private
```

Items in a module are private by default: only code inside the module (and its children) can use them. `bank::deposit` is `pub`, so it works, and it may call `secret` because it is inside `bank`. To call `secret` from `main` you would have to write `pub fn secret()`, but the better fix is usually to leave it private and use the public function.

</details>

### Vectors, strings and hash maps

13. What is the difference between `v[10]` and `v.get(10)` on a vector with 3 items?

<details>
<summary>Answer</summary>

`v[10]` panics with "index out of bounds". `v.get(10)` returns `None`, so you can handle the missing item. Use `get` for indexes that come from outside the program.

</details>

14. Why can you not call `v.push(4)` while you hold a reference `let first = &v[0];` that you use afterwards?

<details>
<summary>Answer</summary>

`push` may reallocate the vector (move its items to a bigger area of the heap), which would leave `first` pointing to freed memory. The borrowing rules forbid a mutable borrow (`push`) while a shared borrow (`first`) is alive: error E0502.

</details>

15. What does this program print, and which line panics?

```rust
fn main() {
    let s = "héllo";
    println!("{} {}", s.len(), s.chars().count());
    println!("{:?}", s.get(0..2));
    println!("{}", &s[0..3]);
    println!("{}", &s[0..2]);
}
```

<details>
<summary>Answer</summary>

```text
6 5
None
hé
```

and then a panic on the last line: `byte index 2 is not a char boundary; it is inside 'é' (bytes 1..3 of string)`. `h` takes 1 byte and `é` takes 2, so the length is 6 bytes for 5 characters. Bytes 0..2 cut `é` in half: `get` answers `None` safely, while `&s[0..2]` panics. Bytes 0..3 are exactly `hé`.

</details>

16. Write one line that adds 1 to the count of `word` in a `HashMap<String, u32>` called `counts`, creating the entry if needed.

<details>
<summary>Answer</summary>

```rust
*counts.entry(word.to_string()).or_insert(0) += 1;
```

`entry` finds the place for the key, `or_insert(0)` stores 0 if it is new and returns `&mut u32`, and `*` ... `+= 1` changes the number behind that reference. One lookup does it all.

</details>

17. Why can a `HashMap` print its entries in a different order on every run? What do you use when you need sorted output?

<details>
<summary>Answer</summary>

The order depends on hash values, and Rust seeds the hashing with a random secret on purpose (to resist hash-flooding attacks), so the order is not defined. For sorted output use a `BTreeMap`, or collect the keys into a `Vec` and `sort()` it.

</details>

### Errors

18. When do you use `panic!` and when do you return a `Result`?

<details>
<summary>Answer</summary>

`panic!` is for bugs and broken promises (a situation that should never happen in correct code). `Result` is for failures that can happen in a correct program (bad user input, a missing file, a network error), so the caller can decide how to react. Never `unwrap()` data that comes from outside the program.

</details>

19. What does the `?` operator do, where can it be used, and what does it need so that it can convert one error type to another?

<details>
<summary>Answer</summary>

On `Ok(v)` it evaluates to `v`; on `Err(e)` it returns `Err(e)` from the current function at once. It can be used only in a function whose return type is `Result` (or `Option`), for example `fn main() -> Result<(), Box<dyn Error>>`. If the error types differ, the error type of the function must implement `From<TheOtherError>` (or be `Box<dyn Error>`).

</details>

### Git and version control

20. In a Rust project, which folder must never be committed and how do you make sure of that? Which commands create a branch, commit on it, return to `main` and merge the branch with a merge commit?

<details>
<summary>Answer</summary>

Never commit `target/` (build output). Put the line `/target` in `.gitignore`. (Note: `cargo new` inside an existing repository does not create the file for you.) The commands:

```bash
git switch -c feature/name       # create and switch to the branch
git add . && git commit -m "feat: ..."
git switch main
git merge --no-ff feature/name   # always makes a merge commit
git branch -d feature/name       # delete the merged branch
```

</details>

## Practical tasks

Write each program yourself first. For each task, the **hint** gives the plan; the **answer** is one correct
solution. Every task must pass `cargo test`, `cargo clippy --all-targets -- -D warnings` and `cargo fmt --check`.
Commit after each task (one commit per task, Conventional Commits style).

### Task 1 — Shapes (structs, enums, methods, vectors)

Create `src/bin/shapes.rs`. Define an enum `Shape` with three variants: `Circle { radius: f64 }`, `Rectangle { width: f64, height: f64 }` and `Square(f64)`.
Add methods `area(&self) -> f64` and `name(&self) -> &'static str`. Write `fn largest(shapes: &[Shape]) -> Option<&Shape>`.
In `main`, build a vector with a circle of radius 1.5, a 3 × 4 rectangle and a square with side 2.5. Print one line per shape, the total area, and the
name of the largest shape. Write tests: rectangle area, square area and name, circle area close to π (compare with a tolerance),
largest of several, largest of none. Expected output:

```text
circle    area   7.07
rectangle area  12.00
square    area   6.25
total area 25.32
largest: rectangle
```

<details>
<summary>Hint</summary>

Use `match self` with one arm per variant; `Shape::Circle { radius }` binds the field, and `Shape::Circle { .. }` ignores all fields. The area of a circle is `PI * radius * radius`
with `use std::f64::consts::PI;`. For `largest`, keep a `best: Option<&Shape>` and update it in a loop. For the output use `{:<9}` (pad to 9, left) and `{:>6.2}` (width 6, two decimals, right).
Floats should not be compared with `==` after inexact calculations; use `(a - b).abs() < 0.001`. (`&'static str` is the type of a string literal, which lives as long as the program; lifetimes get their own lesson, [Lesson 05.3](../05-rust-intermediate/03-lifetimes.md).)

</details>

<details>
<summary>Answer</summary>

```rust
use std::f64::consts::PI;

#[derive(Debug)]
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Square(f64),
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => PI * radius * radius,
            Shape::Rectangle { width, height } => width * height,
            Shape::Square(side) => side * side,
        }
    }

    fn name(&self) -> &'static str {
        match self {
            Shape::Circle { .. } => "circle",
            Shape::Rectangle { .. } => "rectangle",
            Shape::Square(_) => "square",
        }
    }
}

/// The shape with the biggest area, or None for an empty slice.
fn largest(shapes: &[Shape]) -> Option<&Shape> {
    let mut best: Option<&Shape> = None;
    for shape in shapes {
        match best {
            Some(current) if current.area() >= shape.area() => {}
            _ => best = Some(shape),
        }
    }
    best
}

fn main() {
    let shapes = vec![
        Shape::Circle { radius: 1.5 },
        Shape::Rectangle { width: 3.0, height: 4.0 },
        Shape::Square(2.5),
    ];

    let mut total = 0.0;
    for shape in &shapes {
        println!("{:<9} area {:>6.2}", shape.name(), shape.area());
        total += shape.area();
    }
    println!("total area {:.2}", total);

    match largest(&shapes) {
        Some(shape) => println!("largest: {}", shape.name()),
        None => println!("no shapes"),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rectangle_area() {
        let r = Shape::Rectangle { width: 3.0, height: 4.0 };
        assert_eq!(r.area(), 12.0);
    }

    #[test]
    fn square_area_and_name() {
        let s = Shape::Square(2.5);
        assert_eq!(s.area(), 6.25);
        assert_eq!(s.name(), "square");
    }

    #[test]
    fn circle_area_is_close_to_pi() {
        let c = Shape::Circle { radius: 1.0 };
        assert!((c.area() - 3.14159).abs() < 0.001);
    }

    #[test]
    fn largest_of_several() {
        let shapes = vec![Shape::Square(1.0), Shape::Square(3.0), Shape::Square(2.0)];
        assert_eq!(largest(&shapes).unwrap().area(), 9.0);
    }

    #[test]
    fn largest_of_none() {
        assert!(largest(&[]).is_none());
    }
}
```

Why it works: each variant carries only the data it needs, and every `match` covers all three variants, so adding a fourth shape later makes the compiler point at every place to update.
The guard `Some(current) if current.area() >= shape.area() => {}` means "keep the current best"; every other case (no best yet, or a bigger shape) replaces it. (The square areas 6.25 and 9.0 are exact in floating point, so `assert_eq!` is safe there.)

</details>

### Task 2 — Inventory (modules, hash maps, errors)

Create `src/bin/inventory.rs`. Inside it write an inline module `mod inventory { ... }` with a public struct `Inventory` (it keeps a `HashMap<String, u32>`; the field is private), a public enum
`InventoryError { UnknownItem(String), NotEnough { item: String, have: u32, wanted: u32 } }` and the public methods `new()`, `add(&mut self, item: &str, quantity: u32)`,
`remove(&mut self, item: &str, quantity: u32) -> Result<(), InventoryError>` and `quantity(&self, item: &str) -> u32`. Item names are case-insensitive (store them in lower case).
A failed `remove` must leave the stock unchanged. Outside the module, implement `Display` for `InventoryError` and run this scenario in `main`: add 10 Apples, 5 apples and 2 Pears, then print
the apples quantity (case-insensitive lookup), then try to remove 4 apples, 3 pears and 1 plum, then print the apples left. Write four tests (merging case variants, remove reduces stock, too many keeps the stock, unknown item). Expected output:

```text
apples: 15
sold 4 apples
error: only 2 of 'pears' left, but 3 wanted
error: we do not sell 'plums'
apples left: 11
```

<details>
<summary>Hint</summary>

In `add`, use `*self.stock.entry(item.to_lowercase()).or_insert(0) += quantity;`. In `remove`, `match self.stock.get_mut(&item)`: `None` gives `UnknownItem`; `Some(have)` where `*have < quantity` (a match guard) gives `NotEnough`; the last arm subtracts. For the types of the module to be visible in `main` and in the tests, add `use inventory::{Inventory, InventoryError};` after the module. An `impl fmt::Display for InventoryError` is allowed outside the module (it is in the same crate), but the enum must be `pub`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fmt;

mod inventory {
    use std::collections::HashMap;

    #[derive(Debug, PartialEq)]
    pub enum InventoryError {
        UnknownItem(String),
        NotEnough { item: String, have: u32, wanted: u32 },
    }

    #[derive(Debug, Default)]
    pub struct Inventory {
        stock: HashMap<String, u32>,
    }

    impl Inventory {
        pub fn new() -> Inventory {
            Inventory::default()
        }

        pub fn add(&mut self, item: &str, quantity: u32) {
            *self.stock.entry(item.to_lowercase()).or_insert(0) += quantity;
        }

        pub fn remove(&mut self, item: &str, quantity: u32) -> Result<(), InventoryError> {
            let item = item.to_lowercase();
            match self.stock.get_mut(&item) {
                None => Err(InventoryError::UnknownItem(item)),
                Some(have) if *have < quantity => Err(InventoryError::NotEnough {
                    item,
                    have: *have,
                    wanted: quantity,
                }),
                Some(have) => {
                    *have -= quantity;
                    Ok(())
                }
            }
        }

        pub fn quantity(&self, item: &str) -> u32 {
            self.stock.get(&item.to_lowercase()).copied().unwrap_or(0)
        }
    }
}

use inventory::{Inventory, InventoryError};

impl fmt::Display for InventoryError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            InventoryError::UnknownItem(item) => write!(f, "we do not sell '{item}'"),
            InventoryError::NotEnough { item, have, wanted } => {
                write!(f, "only {have} of '{item}' left, but {wanted} wanted")
            }
        }
    }
}

fn main() {
    let mut shop = Inventory::new();
    shop.add("Apples", 10);
    shop.add("apples", 5);
    shop.add("Pears", 2);

    println!("apples: {}", shop.quantity("APPLES"));
    for (item, qty) in [("apples", 4), ("pears", 3), ("plums", 1)] {
        match shop.remove(item, qty) {
            Ok(()) => println!("sold {qty} {item}"),
            Err(e) => println!("error: {e}"),
        }
    }
    println!("apples left: {}", shop.quantity("apples"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_merges_case_variants() {
        let mut shop = Inventory::new();
        shop.add("Apples", 10);
        shop.add("apples", 5);
        assert_eq!(shop.quantity("APPLES"), 15);
    }

    #[test]
    fn remove_reduces_stock() {
        let mut shop = Inventory::new();
        shop.add("tea", 5);
        assert_eq!(shop.remove("tea", 2), Ok(()));
        assert_eq!(shop.quantity("tea"), 3);
    }

    #[test]
    fn remove_too_many_fails_and_keeps_stock() {
        let mut shop = Inventory::new();
        shop.add("tea", 1);
        let err = shop.remove("tea", 3).unwrap_err();
        assert_eq!(err, InventoryError::NotEnough { item: "tea".into(), have: 1, wanted: 3 });
        assert_eq!(shop.quantity("tea"), 1);
    }

    #[test]
    fn unknown_item() {
        let mut shop = Inventory::new();
        assert_eq!(shop.remove("milk", 1), Err(InventoryError::UnknownItem("milk".into())));
    }
}
```

Why it works: the field `stock` is private, so the only way to change the stock is through `add` and `remove`, which keep the rules (lower-case names, never negative stock). The guard arm is placed before the general `Some(have)` arm, so a too-large `quantity` is rejected before anything is subtracted. `"tea".into()` converts a `&str` to a `String` (in tests, where a short form is welcome).

</details>

### Task 3 — Config parser (strings, maps, errors with line numbers)

Create `src/bin/config.rs`. Write `fn parse_config(text: &str) -> Result<BTreeMap<String, String>, ConfigError>` for text like

```text
# server settings
port = 8080

name = demo
```

Rules: blank lines and lines that start with `#` are skipped; every other line must be `key = value` (split at the **first** `=`); spaces around key and value are removed; the value may be empty or contain `=`.
Errors (all carry the line number, counting from 1): `MissingEquals { line }`, `EmptyKey { line }`, `DuplicateKey { line, key }`. Write six tests: pairs and comments, empty value with `=` inside, and one test for each error and for empty text. In `main`, print the results of
`parse_config` for the four texts shown in the expected output:

```text
Ok({"name": "demo", "port": "8080"})
Err(MissingEquals { line: 2 })
Err(EmptyKey { line: 2 })
Err(DuplicateKey { line: 3, key: "a" })
```

(The texts are: the sample above with its comment line; `"a=1\nbroken line\n"`; `"a=1\n = 2\n"`; `"a=1\nb=2\na=3\n"`.)

<details>
<summary>Hint</summary>

Use `for (index, raw) in text.lines().enumerate()` and `let line_number = index + 1;`. After `trim()`, skip when `line.is_empty() || line.starts_with('#')`.
`let Some((key, value)) = line.split_once('=') else { return Err(...) };` handles the missing `=`. A `BTreeMap` keeps the keys sorted, which makes the debug output and the tests predictable.
Check duplicates with `map.contains_key(key)` before `insert`. Derive `Debug` and `PartialEq` for the error so you can use `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::BTreeMap;

#[derive(Debug, PartialEq)]
enum ConfigError {
    MissingEquals { line: usize },
    EmptyKey { line: usize },
    DuplicateKey { line: usize, key: String },
}

/// Parses `key = value` lines. Blank lines and lines starting with '#' are skipped.
fn parse_config(text: &str) -> Result<BTreeMap<String, String>, ConfigError> {
    let mut map = BTreeMap::new();
    for (index, raw) in text.lines().enumerate() {
        let line_number = index + 1;
        let line = raw.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        let Some((key, value)) = line.split_once('=') else {
            return Err(ConfigError::MissingEquals { line: line_number });
        };
        let key = key.trim();
        if key.is_empty() {
            return Err(ConfigError::EmptyKey { line: line_number });
        }
        if map.contains_key(key) {
            return Err(ConfigError::DuplicateKey { line: line_number, key: key.to_string() });
        }
        map.insert(key.to_string(), value.trim().to_string());
    }
    Ok(map)
}

fn main() {
    let good = "# server settings\nport = 8080\n\nname = demo\n";
    println!("{:?}", parse_config(good));
    println!("{:?}", parse_config("a=1\nbroken line\n"));
    println!("{:?}", parse_config("a=1\n = 2\n"));
    println!("{:?}", parse_config("a=1\nb=2\na=3\n"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_pairs_and_skips_comments() {
        let map = parse_config("# note\nport = 8080\n\nname=demo\n").unwrap();
        assert_eq!(map.len(), 2);
        assert_eq!(map["port"], "8080");
        assert_eq!(map["name"], "demo");
    }

    #[test]
    fn value_may_be_empty_and_contain_equals() {
        let map = parse_config("a =\nb = x=y\n").unwrap();
        assert_eq!(map["a"], "");
        assert_eq!(map["b"], "x=y");
    }

    #[test]
    fn missing_equals_reports_line() {
        assert_eq!(parse_config("a=1\nbroken\n"), Err(ConfigError::MissingEquals { line: 2 }));
    }

    #[test]
    fn empty_key_reports_line() {
        assert_eq!(parse_config("=1\n"), Err(ConfigError::EmptyKey { line: 1 }));
    }

    #[test]
    fn duplicate_key() {
        assert_eq!(
            parse_config("a=1\nb=2\na=3\n"),
            Err(ConfigError::DuplicateKey { line: 3, key: "a".into() })
        );
    }

    #[test]
    fn empty_text_is_an_empty_map() {
        assert!(parse_config("").unwrap().is_empty());
    }
}
```

Why it works: `split_once` cuts only at the first `=`, so `x=y` stays in the value. The function stops at the first problem and reports where it is. Because the function returns the parsed data and errors as values, it can be tested without files or a terminal.

</details>

### Task 4 — A complete Git workflow, with a conflict (version control)

In a scratch folder (not in your learning repository) do the following, and write down what you see after every step:

1. `mkdir ~/git-practice && cd ~/git-practice && git init -b main`, then `cargo new demo` and `cd demo && cargo build && cd ..`.
2. Run `git status --short`, then `git status --short -uall | wc -l`. What is the problem? Fix it by creating `demo/.gitignore` with `/target`, and check that `git status --short -uall` now lists only 4 files.
3. Commit everything with the message `chore: create the demo project`.
4. Create the branch `fix/greeting` and switch to it. In `demo/src/main.rs` change `Hello, world!` to `Hello, systems!`. Commit it as `fix: change the greeting to systems`.
5. Switch back to `main`. On `main`, change the same line to `Hello, Rust!` and commit it as `feat: greet Rust`.
6. Merge `fix/greeting` into `main`. Git reports a conflict. Open the file, remove the markers and make the line `Hello, Rust systems!`. Finish the merge.
7. Delete the merged branch, tag the result `v0.1.0`, and show the history as a graph. Run the program.

Expected key output (hashes will be different):

```text
?? demo/
...
Auto-merging demo/src/main.rs
CONFLICT (content): Merge conflict in demo/src/main.rs
Automatic merge failed; fix conflicts and then commit the result.
...
*   c4e4821 Merge branch 'fix/greeting'
|\
| * 676b350 fix: change the greeting to systems
* | f829c3a feat: greet Rust
|/
* 704cb66 chore: create the demo project
```

and the program prints `Hello, Rust systems!`.

<details>
<summary>Hint</summary>

Without `.gitignore`, the `-uall` listing shows every file under `demo/target/` (dozens of lines; we counted 33). Branch: `git switch -c fix/greeting`; edit; `git commit -am "..."` (the option `-a` stages changes of files that Git already knows).
After the conflict, `git status` shows `both modified: demo/src/main.rs`. In the file, the part between `<<<<<<< HEAD` and `=======` is your branch (`main`), and the part between `=======` and `>>>>>>> fix/greeting` comes from the other branch.
Keep the text you want, delete the three marker lines, then `git add demo/src/main.rs` and `git commit --no-edit`. Remember [Lesson 01.7 — Merge conflicts](../01-git-and-version-control/07-merge-conflicts.md).

</details>

<details>
<summary>Answer</summary>

```bash
mkdir ~/git-practice && cd ~/git-practice
git init -b main
cargo new demo
(cd demo && cargo build)
git status --short                    # ?? demo/
git status --short -uall | wc -l      # dozens of lines: target/ is not ignored!
printf '/target\n' > demo/.gitignore
git status --short -uall              # only .gitignore, Cargo.lock, Cargo.toml, src/main.rs
git add . && git commit -m "chore: create the demo project"

git switch -c fix/greeting
sed -i '' 's/Hello, world!/Hello, systems!/' demo/src/main.rs     # on Linux: sed -i without ''
git commit -am "fix: change the greeting to systems"

git switch main
sed -i '' 's/Hello, world!/Hello, Rust!/' demo/src/main.rs
git commit -am "feat: greet Rust"

git merge fix/greeting                # CONFLICT (content): Merge conflict in demo/src/main.rs
```

Edit `demo/src/main.rs` so that it contains no markers:

```rust
fn main() {
    println!("Hello, Rust systems!");
}
```

```bash
git add demo/src/main.rs
git commit --no-edit
git branch -d fix/greeting
git tag -a v0.1.0 -m "first release"
git log --oneline --graph
(cd demo && cargo run -q)
```

You can also open the file in RustRover and use its merge tool. Why it works: Git could not choose between two edits of the same line, so it asked you. After you decide and `git add` the file, the merge commit joins the two histories; the graph shows both branches. The `.gitignore` step matters because `cargo new` does not create one when the folder is already inside a Git repository.

</details>

## If you got something wrong

| Topic | Questions / tasks | Review |
|---|---|---|
| Structs, tuple structs, update syntax, ownership of fields | 1, 2 | [Lesson 04.1 — Structs](01-structs.md) |
| Methods, `self` kinds, associated functions | 3, 4 | [Lesson 04.2 — Methods and associated functions](02-methods-and-associated-functions.md) |
| Enums and variants with data | 5, Task 1 | [Lesson 04.3 — Enums](03-enums.md) |
| `Option`, `match`, exhaustiveness, guards | 6, 7, 8, Task 1 | [Lesson 04.4 — Option and match](04-option-and-match.md) |
| `if let`, `let else`, destructuring patterns | 9, 10 | [Lesson 04.5 — if let, let else and patterns](05-if-let-let-else-and-patterns.md) |
| Modules, `pub`, crates and packages | 11, 12, Task 2 | [Lesson 04.6 — Modules, crates and packages](06-modules-crates-and-packages.md) |
| Vectors, indexing, borrowing and growth | 13, 14 | [Lesson 04.7 — Vectors](07-vectors.md) |
| Strings, UTF-8, slicing text, splitting | 15, Task 3 | [Lesson 04.8 — Strings in depth](08-strings-in-depth.md) |
| Hash maps, the `entry` API, ordering | 16, 17, Task 2 | [Lesson 04.9 — Hash maps](09-hash-maps.md) |
| `Result`, `?`, custom errors, `panic!` | 18, 19, Tasks 2 and 3 | [Lesson 04.10 — Error handling](10-error-handling.md) |
| Project workflow: milestones, tests, clippy, fmt | all tasks | [Lesson 04.11 — Project: phone book](11-project-phone-book.md) |
| Git: `.gitignore`, branches, merging, conflicts, tags | 20, Task 4 | [Lesson 01.6 — Branches and merging](../01-git-and-version-control/06-branches-and-merging.md), [Lesson 01.7 — Merge conflicts](../01-git-and-version-control/07-merge-conflicts.md), [Lesson 01.9 — Teamwork, pull requests and conventions](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) |
| Writing tests | all tasks | [Lesson 02.8 — Your first tests](../02-programming-basics/08-your-first-tests.md) |

If you got **16 or more** quiz answers right and finished all four tasks, you are ready for the next phase. If you got **fewer than 12** right, go back
to the lessons in the table before you continue; Phase 05 builds directly on these ideas. Between 12 and 15: review only the rows of the questions you missed, then redo those questions after a day.

## What's next

Phase 05 — *Rust intermediate* — turns the code you wrote here into *reusable* code:

- [Lesson 05.1 — Generics](../05-rust-intermediate/01-generics.md): the `<T>` you have seen in `Vec<T>` and `Result<T, E>`, and how to write your own.
- [Lesson 05.2 — Traits](../05-rust-intermediate/02-traits.md): what `Display`, `Error`, `From` and `Hash` really are, and how to define your own.
- Later: lifetimes, closures, iterators, deeper testing, crates (`serde`, `clap`, `thiserror`, `anyhow`), threads, and the `minigrep` project.

Before you go:

1. Make sure your phone book project and the checkpoint tasks are committed and tagged. If you use GitHub or GitLab, push them.
2. Keep the tests running: from now on, run `cargo test`, `cargo fmt` and `cargo clippy` before every commit.
3. Take a break. Phase 05 is easier when you start fresh.

## Summary

- Phase 04 gave you the tools to model data: structs for "this and that", enums for "this or that", and methods to attach behaviour.
- `Option` and `Result` make absence and failure part of the type; `match`, `if let`, `let else` and `?` are how you deal with them.
- Modules, `pub` and crates organize code and decide what is visible.
- `Vec`, `String`, `HashMap` and `BTreeMap` hold data of any size; ownership and borrowing rules apply to them just as to every other value.
- Good programs have a clear contract, small tested functions, errors as values, clean `fmt` and `clippy`, and one Git commit for each finished step.

## Sources and further reading

- [The Rust Programming Language — 5. Using Structs to Structure Related Data](https://doc.rust-lang.org/book/ch05-00-structs.html) — structs and methods, for review.
- [The Rust Programming Language — 6. Enums and Pattern Matching](https://doc.rust-lang.org/book/ch06-00-enums.html) — enums, `Option`, `match`, `if let`, `let else`.
- [The Rust Programming Language — 7. Managing Growing Projects with Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html) — packages, crates and modules.
- [The Rust Programming Language — 8. Common Collections](https://doc.rust-lang.org/book/ch08-00-common-collections.html) — vectors, strings and hash maps.
- [The Rust Programming Language — 9. Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html) — `panic!`, `Result` and `?`.
- [Rustlings](https://github.com/rust-lang/rustlings) — small exercises that you run locally; the sections on structs, enums, modules, vectors, strings, hash maps and error handling match this phase.
- [Pro Git — Git Branching: Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — branches, merges and conflicts in depth.

---

[🏠 Start](../START-HERE.md) · [← 04.11 Project: phone book](11-project-phone-book.md) · [📋 Phase 04](README.md) · [Phase 05 — Rust intermediate →](../05-rust-intermediate/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
