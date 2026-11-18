<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.2 Methods and associated functions](02-methods-and-associated-functions.md) · [📋 Phase 04](README.md) · [04.4 Option and match →](04-option-and-match.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/03-enums.md) <!-- nav -->

# Lesson 04.3 — Enums

> **Phase 04 — Structs, enums, collections and errors** · Lesson 3 of 12 · ⏱ about 3.5 hours (2 sittings)

## What you will learn

- What an **enum** is and how it differs from a struct: "one of several choices" instead of "all of these parts".
- How to define enums, create their values and add methods to them.
- How variants can carry data (tuple-like and struct-like), and why this is so powerful.
- A first look at `match`, the tool that takes an enum apart, and why the compiler forces you to handle every variant.
- How an enum is stored in memory, how to give variants numbers, and how to turn text into an enum safely.

## Before you start

- You finished [Lesson 04.1 — Structs](01-structs.md) and [Lesson 04.2 — Methods and associated functions](02-methods-and-associated-functions.md).
- You know `if`/`else` and `bool` ([Lesson 02.5 — Control flow: conditions and loops](../02-programming-basics/05-control-flow.md)). We will meet `match` here for the first time in depth; [Lesson 04.4 — Option and match](04-option-and-match.md) finishes the topic.

## New words

| Word | Meaning |
|---|---|
| **enum** (enumeration) | A type whose value is exactly **one** of a fixed list of named choices. |
| **variant** | One choice of an enum. In `enum Light { Red, Green }`, `Red` and `Green` are variants. |
| **payload** | The data a variant carries with it, for example the `f64` in `Circle(f64)`. |
| **tuple variant** | A variant whose payload has unnamed positions: `Circle(f64)`. |
| **struct variant** | A variant whose payload has named fields: `Rectangle { width: f64, height: f64 }`. |
| **unit variant** | A variant with no payload: `Quit`. |
| **match** | An expression that compares a value against a list of **patterns** and runs the code of the first pattern that fits. |
| **arm** | One line of a `match`: a pattern, `=>`, and the code to run. |
| **pattern** | A shape that a value can be compared to, such as `Direction::North` or `Circle(r)`. Full treatment in [Lesson 04.5](05-if-let-let-else-and-patterns.md). |
| **exhaustive** | A `match` is exhaustive when it covers **every** possible value. Rust requires this. |
| **wildcard** | The pattern `_`, which matches anything and ignores it. |
| **discriminant** | The hidden number the compiler uses to tell variants apart; it can be set by hand (`Low = 1`). |
| **tag** | The part of an enum value in memory that records which variant it currently is. |
| **state machine** | A model of something that is always in exactly one state out of a few (offline / connecting / online) and moves between states by rules. |
| **cast** | Converting a value to another type with `as`, such as `level as u8`. |
| **Option** | An enum from the standard library for "a value, or nothing": `Some(value)` or `None`. Lesson 04.4 explains it fully. |

## 1. A value that is one of several choices

A struct says "this **and** this **and** this". A `User` has a name **and** an email **and** a login count.

An **enum** says "this **or** this **or** this". A traffic light is red **or** green **or** yellow — never two at once, never "none of them". A compass direction is north **or** east **or** south **or** west.

Without enums you might store a direction as a number (`0` = north, `1` = east …) or as a text (`"north"`). Both can hold nonsense: `7`, or `"nrth"`. The compiler cannot help you. With an enum, **only the valid choices exist**, and typos do not compile.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Direction {
    North,
    East,
    South,
    West,
}

fn main() {
    let heading = Direction::East;
    println!("{:?}", heading);

    if heading == Direction::East {
        println!("going east");
    }

    let all = [Direction::North, Direction::East, Direction::South, Direction::West];
    for d in all {
        println!("{:?}", d);
    }
}
```

**What happens:**

1. `enum Direction { ... }` defines a new type with four **variants**. Variant names use `UpperCamelCase`, like type names.
2. You create a value by writing the enum name, two colons, and the variant: `Direction::East`. The enum name acts as a *namespace* (a prefix that keeps names apart), so another enum may also have a variant called `North`.
3. `#[derive(Debug)]` lets us print it. `PartialEq` makes `==` work, so we can compare two directions. `Clone, Copy` make it cheap to pass around: an enum without payload is just a small number (see section 6).
4. An array holds four directions; the `for` loop prints each one.

```text
East
going east
North
East
South
West
```

## 2. A first look at `match`

To *use* an enum you usually ask: "which variant is it?" and act on the answer. The tool for that is **`match`**.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Direction {
    North,
    East,
    South,
    West,
}

impl Direction {
    fn turn_right(self) -> Direction {
        match self {
            Direction::North => Direction::East,
            Direction::East => Direction::South,
            Direction::South => Direction::West,
            Direction::West => Direction::North,
        }
    }

    fn is_vertical(self) -> bool {
        match self {
            Direction::North | Direction::South => true,
            Direction::East | Direction::West => false,
        }
    }
}

fn main() {
    let mut d = Direction::North;
    for _ in 0..5 {
        println!("{:?} (vertical: {})", d, d.is_vertical());
        d = d.turn_right();
    }
}
```

**What happens:**

1. `impl Direction { ... }` gives the enum methods, exactly as for a struct ([Lesson 04.2](02-methods-and-associated-functions.md)). `self` is taken by value; that is fine because `Direction` is `Copy`.
2. `match self { ... }` looks at the value of `self`. Each **arm** has a pattern, then `=>`, then an expression. The compiler checks the arms from top to bottom and runs the first one that fits.
3. In `turn_right` every arm returns a `Direction`. The whole `match` is an expression, so its value is the function's return value (no semicolon after it).
4. `Direction::North | Direction::South` means "either of these" (`|` reads as "or"). Both give `true`.
5. In `main`, the loop runs five times and turns right each time: after `West` it comes back to `North`.

```text
North (vertical: true)
East (vertical: false)
South (vertical: true)
West (vertical: false)
North (vertical: true)
```

### The compiler checks that you covered everything

Remove one arm and see what happens:

```rust
fn name(d: Direction) -> &'static str {
    match d {
        Direction::North => "north",
        Direction::East => "east",
        Direction::South => "south",
    }
}
```

```text
error[E0004]: non-exhaustive patterns: `Direction::West` not covered
  --> src/main.rs:10:11
   |
10 |     match d {
   |           ^ pattern `Direction::West` not covered
   |
note: `Direction` defined here
  --> src/main.rs:2:6
   |
 2 | enum Direction {
   |      ^^^^^^^^^
...
 6 |     West,
   |     ---- not covered
   = note: the matched value is of type `Direction`
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
   |
13 ~         Direction::South => "south",
14 ~         Direction::West => todo!(),
   |
```

A `match` must be **exhaustive**: every possible value needs an arm. The compiler even names the missing variant. This is one of the best safety features of Rust: if you later add a fifth variant to the enum, the compiler shows you **every `match` in your program** that must be updated. In many other languages that kind of bug is found only when the program runs.

When you really want "everything else", use the wildcard `_`:

```text
match self {
    Day::Saturday | Day::Sunday => true,
    _ => false,
}
```

> 📏 **Convention:** use `_` only when "all the rest" really is one case. If you list every variant explicitly, the compiler will warn you when someone adds a new one; with `_` it will stay silent and the new variant silently takes the default branch.

## 3. Variants that carry data

So far every variant was only a name. Rust enums can do much more: **each variant can carry its own data**, with its own shape.

```rust
#[derive(Debug)]
enum Shape {
    Circle(f64),
    Square(f64),
    Rectangle { width: f64, height: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle(radius) => std::f64::consts::PI * radius * radius,
            Shape::Square(side) => side * side,
            Shape::Rectangle { width, height } => width * height,
        }
    }
}

fn main() {
    let shapes = [
        Shape::Circle(1.0),
        Shape::Square(3.0),
        Shape::Rectangle { width: 2.0, height: 4.5 },
    ];
    for shape in &shapes {
        println!("{:?} has area {:.2}", shape, shape.area());
    }
}
```

**What happens:**

1. `Circle(f64)` is a **tuple variant**: it holds one unnamed `f64`. You create it like a function call: `Shape::Circle(1.0)`.
2. `Rectangle { width: f64, height: f64 }` is a **struct variant**: it holds named fields. You create it like a struct: `Shape::Rectangle { width: 2.0, height: 4.5 }`.
3. In `area(&self)` we match on a *reference* to the shape. In the pattern `Shape::Circle(radius)` the name `radius` is a new variable that **takes the payload out** of the variant. Because we matched on `&self`, `radius` is a `&f64` — a reference into the shape; the arithmetic `PI * radius * radius` works with references to numbers.
4. In the struct-variant pattern `Shape::Rectangle { width, height }` the names `width` and `height` pick up the fields (the same field init shorthand you know from structs).
5. Because the data is inside the variant, you can only reach it by checking which variant it is. A `Square` has no `width`, so you cannot ask for one by mistake.
6. `{:.2}` prints two digits after the decimal point.

```text
Circle(1.0) has area 3.14
Square(3.0) has area 9.00
Rectangle { width: 2.0, height: 4.5 } has area 9.00
```

A variant with no data is a **unit variant** (like `Quit` below). One enum can mix all three kinds:

```rust
#[derive(Debug)]
enum Command {
    Quit,
    Move { x: i32, y: i32 },
    Say(String),
    SetColor(u8, u8, u8),
}

fn run(command: &Command) {
    match command {
        Command::Quit => println!("bye"),
        Command::Move { x, y } => println!("moving by ({x}, {y})"),
        Command::Say(text) => println!("saying: {text}"),
        Command::SetColor(r, g, b) => println!("color is now #{r:02x}{g:02x}{b:02x}"),
    }
}

fn main() {
    let script = vec![
        Command::Say(String::from("hello")),
        Command::Move { x: 3, y: -1 },
        Command::SetColor(255, 128, 0),
        Command::Quit,
    ];
    for command in &script {
        run(command);
    }
}
```

**What happens:**

1. `Command` has four variants with four different shapes: nothing, two named numbers, one `String`, and three `u8`s.
2. `run` takes `&Command`, so the caller keeps the command. Each arm extracts exactly the data its variant has.
3. `{r:02x}` formats a number as **hex** (base 16, digits `0-9a-f`), at least 2 characters wide, padded with `0`. So `255, 128, 0` becomes `ff8000`, the common way to write colors.
4. `vec![...]` creates a growable list. It is explained in [Lesson 04.7 — Vectors](07-vectors.md); here it is just a container we can loop over.

```text
saying: hello
moving by (3, -1)
color is now #ff8000
bye
```

Without enums you would need four separate structs and a way to pass "any of them" around. With an enum, one type covers all commands, and `match` ensures that `run` handles every one of them.

> 💡 **Tip:** an enum is how you model **states**. Compare two ways to describe a network connection:
>
> - A struct with flags: `connected: bool`, `attempt: u32`, `session_id: u64`. It allows nonsense such as "not connected, but has a session id".
> - An enum where each state has only the data that makes sense in it.
>
> ```rust
> #[derive(Debug)]
> enum ConnectionState {
>     Disconnected,
>     Connecting { attempt: u32 },
>     Connected { session_id: u64 },
> }
> ```
>
> Now an invalid combination cannot even be written. Rust programmers call this **making illegal states unrepresentable**, and it is one of the most useful design habits you can learn.

```rust
#[derive(Debug)]
enum ConnectionState {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session_id: u64 },
}

fn describe(state: &ConnectionState) -> String {
    match state {
        ConnectionState::Disconnected => String::from("offline"),
        ConnectionState::Connecting { attempt } => format!("trying, attempt {attempt}"),
        ConnectionState::Connected { session_id } => format!("online, session {session_id}"),
    }
}

fn main() {
    let states = [
        ConnectionState::Disconnected,
        ConnectionState::Connecting { attempt: 2 },
        ConnectionState::Connected { session_id: 90210 },
    ];
    for s in &states {
        println!("{}", describe(s));
    }
}
```

```text
offline
trying, attempt 2
online, session 90210
```

## 4. Numbers for variants

An enum with only unit variants ("C-like enum") can give each variant a number, the **discriminant**. This is useful when the number has a meaning outside your program, such as a code in a file or a network message.

```rust
#[derive(Debug, Clone, Copy)]
enum Level {
    Low = 1,
    Medium = 5,
    High = 10,
}

impl Level {
    fn from_number(n: u8) -> Option<Level> {
        match n {
            1 => Some(Level::Low),
            5 => Some(Level::Medium),
            10 => Some(Level::High),
            _ => None,
        }
    }
}

fn main() {
    let level = Level::High;
    println!("{:?} as number = {}", level, level as u8);
    println!("{:?}", Level::from_number(5));
    println!("{:?}", Level::from_number(7));
}
```

**What happens:**

1. `Low = 1` sets the number of that variant. Without `= ...`, numbering starts at 0 and counts up.
2. `level as u8` is a **cast**: it converts the enum to its number. Going from enum to number is allowed with `as`.
3. The opposite direction is **not** automatic. Not every number is a valid level (what is `7`?). So we write `from_number`, which uses `match` on the number and returns `Option<Level>`. `Option` is an enum from the standard library: `Some(value)` means "here is a value", `None` means "nothing". It is the honest way to say "this might fail"; [Lesson 04.4](04-option-and-match.md) explains it fully.
4. `_ => None` is the wildcard arm: any other number gives `None`.

```text
High as number = 10
Some(Medium)
None
```

> 🔒 **Security:** never turn an untrusted number into an enum by "forcing" it (for example with `unsafe` tricks). An out-of-range value would create an invalid enum, which is **undefined behaviour** (behaviour the language gives no guarantee about; the program may crash or misbehave unpredictably). The safe `match`-based conversion above is the correct way: unknown input becomes `None`.

## 5. Turning text into an enum

Programs read text from files, the keyboard and the network. A common task is to convert the text into an enum, once, at the border of your program. After that the rest of the code works with the safe type.

```rust
#[derive(Debug, PartialEq)]
enum Direction {
    North,
    East,
    South,
    West,
}

fn parse_direction(text: &str) -> Option<Direction> {
    match text {
        "n" | "north" => Some(Direction::North),
        "e" | "east" => Some(Direction::East),
        "s" | "south" => Some(Direction::South),
        "w" | "west" => Some(Direction::West),
        _ => None,
    }
}

fn main() {
    println!("{:?}", parse_direction("east"));
    println!("{:?}", parse_direction("up"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_known_words() {
        assert_eq!(parse_direction("north"), Some(Direction::North));
        assert_eq!(parse_direction("w"), Some(Direction::West));
    }

    #[test]
    fn rejects_unknown_words() {
        assert_eq!(parse_direction("up"), None);
        assert_eq!(parse_direction(""), None);
    }
}
```

**What happens:**

1. `match text` compares a `&str` with string literals. `"n" | "north"` accepts either spelling.
2. Known words give `Some(Direction::...)`; everything else gives `None`. The caller must handle `None`, so unknown input can never silently become a direction.
3. The tests check good words and bad words — including the empty string, a classic edge case.

```text
Some(East)
None
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🔒 **Security:** treat all input as untrusted. Parse it into a closed set of valid values (an enum) and reject the rest. Then a `match` over the enum is complete, and no "unexpected string" can reach the deeper parts of the program.

## 6. How an enum is stored

A value of an enum must be able to hold the **largest** of its variants, plus a **tag** that records which variant it currently is:

```text
enum Number { Small(u8), Big(u64) }

 Small(7):  ┌─────┬─────────┬──────────────────────┐
            │ tag │ padding │ 7 (u8), rest unused  │
            │  0  │         │                      │
            └─────┴─────────┴──────────────────────┘
 Big(1000): ┌─────┬─────────┬──────────────────────┐
            │ tag │ padding │ 1000 (u64)           │
            │  1  │         │                      │
            └─────┴─────────┴──────────────────────┘
            same size for every value: 16 bytes
```

Check it with `size_of` (from [Lesson 04.1](01-structs.md)):

```rust
use std::mem::size_of;

#[allow(dead_code)]
enum Direction {
    North,
    East,
    South,
    West,
}

#[allow(dead_code)]
enum Number {
    Small(u8),
    Big(u64),
}

#[allow(dead_code)]
enum Shape {
    Circle(f64),
    Square(f64),
    Rectangle { width: f64, height: f64 },
}

fn main() {
    println!("Direction = {} byte", size_of::<Direction>());
    println!("Number    = {} bytes", size_of::<Number>());
    println!("Shape     = {} bytes", size_of::<Shape>());
}
```

**What happens:**

1. `#[allow(dead_code)]` tells the compiler not to warn that the variants are never created in this demo.
2. `Direction` has no payload: only a tag is needed, and one byte can count to 255, so the size is 1.
3. `Number` is 16 bytes: the `u64` needs 8 bytes and must start at a multiple of 8, so the 1-byte tag plus 7 bytes of padding come first. A `Small(7)` wastes the unused space.
4. `Shape` is 24 bytes: the biggest variant, `Rectangle`, has two `f64` (16 bytes), plus the tag, padded to 8.

```text
Direction = 1 byte
Number    = 16 bytes
Shape     = 24 bytes
```

> ⚡ **Performance:** an enum is as large as its largest variant. If one rare variant carries a huge payload, every value of the enum pays for it. The usual cure is to store the big payload on the heap (with `Box`, [Lesson 05.9 — Smart pointers](../05-rust-intermediate/09-smart-pointers.md)). Measure with `size_of` before you worry.

## 7. Enums you already use, and will use

Two enums from the standard library appear in almost every Rust program:

- `Option<T>` — `Some(value)` or `None`. It is Rust's way to say "there may be no value". Next lesson.
- `Result<T, E>` — `Ok(value)` or `Err(error)`. You saw `Result` when parsing text in Phase 02; [Lesson 04.10 — Error handling](10-error-handling.md) covers it in depth.

Both are ordinary enums with payloads, defined in the standard library with exactly the tools of this lesson. Nothing magic.

## 8. Testing enums

> 🧪 **Testing:** for every method that does a `match`, write at least one test per variant — the variants are your test cases. Add `PartialEq` and `Debug` to the enum so `assert_eq!` can compare values.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum Day {
    Monday,
    Tuesday,
    Wednesday,
    Thursday,
    Friday,
    Saturday,
    Sunday,
}

impl Day {
    fn is_weekend(self) -> bool {
        match self {
            Day::Saturday | Day::Sunday => true,
            _ => false,
        }
    }

    fn next(self) -> Day {
        match self {
            Day::Monday => Day::Tuesday,
            Day::Tuesday => Day::Wednesday,
            Day::Wednesday => Day::Thursday,
            Day::Thursday => Day::Friday,
            Day::Friday => Day::Saturday,
            Day::Saturday => Day::Sunday,
            Day::Sunday => Day::Monday,
        }
    }
}

fn main() {
    let mut day = Day::Friday;
    for _ in 0..4 {
        println!("{:?}: weekend? {}", day, day.is_weekend());
        day = day.next();
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn weekend_days() {
        assert!(Day::Saturday.is_weekend());
        assert!(Day::Sunday.is_weekend());
        assert!(!Day::Monday.is_weekend());
    }

    #[test]
    fn week_wraps_around() {
        assert_eq!(Day::Sunday.next(), Day::Monday);
        assert_eq!(Day::Monday.next(), Day::Tuesday);
    }
}
```

**What happens:**

1. `is_weekend` uses the wildcard because "every other day" really is one case.
2. `next` lists all seven variants on purpose: if you add an eighth, the compiler forces you to update it.
3. The test `week_wraps_around` checks the edge: the last day leads back to the first.

```text
Friday: weekend? false
Saturday: weekend? true
Sunday: weekend? true
Monday: weekend? false
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Common mistakes

> ⚠️ **Common mistake:** a typo in a variant name.

```text
error[E0599]: no variant, associated function, or constant named `Nort` found for enum `Direction` in the current scope
 --> src/main.rs:8:24
  |
8 |     let d = Direction::Nort;
  |                        ^^^^ variant, associated function, or constant not found in `Direction`
  |
help: there is a variant with a similar name
  |
8 |     let d = Direction::North;
  |                            +
```

The compiler suggests the right spelling. This is exactly the advantage over text values.

> ⚠️ **Common mistake:** forgetting the enum name before the variant: `let d = North;`.

```text
error[E0425]: cannot find value `North` in this scope
 --> src/main.rs:8:13
  |
8 |     let d = North;
  |             ^^^^^ not found in this scope
  |
help: consider importing this unit variant
  |
1 + use crate::Direction::North;
  |
```

Write `Direction::North`. (You could import the variants with `use Direction::*;`, but this makes it hard to see where names come from; avoid it except in small `match`es.)

> ⚠️ **Common mistake:** wrong payload.

```rust
let a = Command::Say("hello");
let b = Command::Move { x: 1 };
```

```text
error[E0308]: mismatched types
 --> src/main.rs:7:26
  |
7 |     let a = Command::Say("hello");
  |             ------------ ^^^^^^^ expected `String`, found `&str`
  |             |
  |             arguments to this enum variant are incorrect
  |
help: try using a conversion method
  |
7 |     let a = Command::Say("hello".to_string());
  |                                 ++++++++++++

error[E0063]: missing field `y` in initializer of `Command`
 --> src/main.rs:8:13
  |
8 |     let b = Command::Move { x: 1 };
  |             ^^^^^^^^^^^^^ missing `y`
```

Payloads follow the same rules as functions and structs: exact types and all fields. `"hello"` is a `&str`, but the variant wants an owned `String`.

> ⚠️ **Common mistake:** comparing with `==` or printing with `{}` without the matching trait.

```text
error[E0369]: binary operation `==` cannot be applied to type `Direction`
 --> src/main.rs:8:10
  |
8 |     if d == Direction::South {
  |        - ^^ ---------------- Direction
  |        |
  |        Direction
  |
help: consider annotating `Direction` with `#[derive(PartialEq)]`
```

```text
error[E0277]: `Direction` doesn't implement `std::fmt::Display`
 --> src/main.rs:6:20
  |
6 |     println!("{}", Direction::North);
  |               --   ^^^^^^^^^^^^^^^^ `Direction` cannot be formatted with the default formatter
  |
  = note: in format strings you may be able to use `{:?}` (or `{:#?}` for pretty-print) instead
```

Derive `PartialEq` for `==`, derive `Debug` for `{:?}`, or write `Display` by hand for `{}` ([Lesson 04.2](02-methods-and-associated-functions.md)).

## Try it yourself

1. Create a project:

   ```text
   cd ~/projects/rust
   cargo new code/p04-enums
   cd code/p04-enums
   ```

2. Put this into `src/main.rs`:

   ```rust
   #[derive(Debug, Clone, Copy, PartialEq)]
   enum Light {
       Red,
       Green,
       Yellow,
   }

   impl Light {
       fn next(self) -> Light {
           match self {
               Light::Red => Light::Green,
               Light::Green => Light::Yellow,
               Light::Yellow => Light::Red,
           }
       }

       fn seconds(self) -> u32 {
           match self {
               Light::Red => 30,
               Light::Green => 25,
               Light::Yellow => 5,
           }
       }
   }

   fn main() {
       let mut light = Light::Red;
       for _ in 0..4 {
           println!("{:?} for {} seconds", light, light.seconds());
           light = light.next();
       }
   }
   ```

3. Run `cargo run`. Expected output:

   ```text
   Red for 30 seconds
   Green for 25 seconds
   Yellow for 5 seconds
   Red for 30 seconds
   ```

4. Add a fourth variant `RedYellow` (used in some countries before green). Do **not** edit the `match`es yet — run `cargo build` and read the two `E0004` errors. Then fix both.
5. Commit your work with a message in the style of Lesson 01.9:

   ```text
   cd ~/projects/rust
   git add code/p04-enums
   git commit -m "feat(p04): traffic light enum"
   ```

## Exercises

### Exercise 1 — Coins (easy)

Define `enum Coin { Penny, Nickel, Dime, Quarter }` (derive `Debug` and `PartialEq`). Write `fn cents(coin: &Coin) -> u32` that returns 1, 5, 10 and 25. In `main`, sum the value of a small array of coins. Write a test for two of the coins.

<details>
<summary>Hint</summary>

`match coin { Coin::Penny => 1, ... }` has one arm per variant. Because `coin` is a reference, the patterns still look the same: `Coin::Penny`. Loop with `for coin in &purse { total += cents(coin); }`.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq)]
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn cents(coin: &Coin) -> u32 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}

fn main() {
    let purse = [Coin::Quarter, Coin::Dime, Coin::Dime, Coin::Penny];
    let mut total = 0;
    for coin in &purse {
        total += cents(coin);
    }
    println!("{total} cents");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn values() {
        assert_eq!(cents(&Coin::Penny), 1);
        assert_eq!(cents(&Coin::Quarter), 25);
    }
}
```

Output: `46 cents`.

Why it works: `match` is an expression; each arm gives a `u32`, and the exhaustiveness check guarantees every coin has a value. (The compiler may warn that `Nickel` is never created in `main`; the warning is harmless here.)

</details>

### Exercise 2 — Shapes with three kinds of data (medium)

Define `enum Shape` with the struct variants `Circle { radius }`, `Rectangle { width, height }` and `Triangle { base, height }` (all `f64`). Add methods `area(&self) -> f64` and `name(&self) -> &'static str`. Test the rectangle area (12.0 for 3 × 4), the triangle area (7.5 for base 6, height 2.5) and a name.

<details>
<summary>Hint</summary>

The triangle area is `0.5 * base * height`. In `name`, you do not need the payload, so write `Shape::Circle { .. }` — the `..` means "I do not care about the fields". Compare floats with a tolerance.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, Clone, PartialEq)]
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
            Shape::Rectangle { width, height } => width * height,
            Shape::Triangle { base, height } => 0.5 * base * height,
        }
    }

    fn name(&self) -> &'static str {
        match self {
            Shape::Circle { .. } => "circle",
            Shape::Rectangle { .. } => "rectangle",
            Shape::Triangle { .. } => "triangle",
        }
    }
}

fn main() {
    let shapes = [
        Shape::Circle { radius: 1.0 },
        Shape::Rectangle { width: 3.0, height: 4.0 },
        Shape::Triangle { base: 6.0, height: 2.5 },
    ];
    for s in &shapes {
        println!("{}: {:.2}", s.name(), s.area());
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rectangle_area() {
        let r = Shape::Rectangle { width: 3.0, height: 4.0 };
        assert!((r.area() - 12.0).abs() < 1e-9);
    }

    #[test]
    fn triangle_area() {
        let t = Shape::Triangle { base: 6.0, height: 2.5 };
        assert!((t.area() - 7.5).abs() < 1e-9);
    }

    #[test]
    fn names() {
        assert_eq!(Shape::Circle { radius: 1.0 }.name(), "circle");
    }
}
```

Output of `cargo run`:

```text
circle: 3.14
rectangle: 12.00
triangle: 7.50
```

Why it works: each variant carries exactly the numbers that its formula needs. `'static` in the return type means the text lives for the whole program (string literals do); it is explained in [Lesson 05.3](../05-rust-intermediate/03-lifetimes.md).

</details>

### Exercise 3 — A tiny tokenizer (hard)

Define `enum Token { Number(i64), Plus, Minus, Star }`. Write `fn tokenize_word(word: &str) -> Option<Token>`: `"+"`, `"-"`, `"*"` give the operators; a word that parses as `i64` gives `Number`; anything else gives `None`. Test `"12"`, `"-7"`, `"+"`, `"x"` and `""`.

<details>
<summary>Hint</summary>

Use `match word` with literal arms for the three operators, then a last arm that binds the word to a name (`other => ...`). Inside it use `match other.parse::<i64>() { Ok(n) => ..., Err(_) => ... }`. `"-7"` is a number, not the minus operator, because the whole word is `-7`, not `-`.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq)]
enum Token {
    Number(i64),
    Plus,
    Minus,
    Star,
}

fn tokenize_word(word: &str) -> Option<Token> {
    match word {
        "+" => Some(Token::Plus),
        "-" => Some(Token::Minus),
        "*" => Some(Token::Star),
        other => match other.parse::<i64>() {
            Ok(n) => Some(Token::Number(n)),
            Err(_) => None,
        },
    }
}

fn main() {
    for word in ["12", "+", "x", "-7", "*"] {
        println!("{:>3} -> {:?}", word, tokenize_word(word));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn numbers_and_operators() {
        assert_eq!(tokenize_word("12"), Some(Token::Number(12)));
        assert_eq!(tokenize_word("-7"), Some(Token::Number(-7)));
        assert_eq!(tokenize_word("+"), Some(Token::Plus));
    }

    #[test]
    fn garbage_is_rejected() {
        assert_eq!(tokenize_word("x"), None);
        assert_eq!(tokenize_word(""), None);
    }
}
```

Output of `cargo run`:

```text
 12 -> Some(Number(12))
  + -> Some(Plus)
  x -> None
 -7 -> Some(Number(-7))
  * -> Some(Star)
```

Why it works: the arms are tried in order, so the exact words `+`, `-`, `*` are caught first; the last arm `other` matches any word and binds it to a variable. `parse::<i64>()` returns a `Result`, and its `Err` case becomes `None`. This is the first step of every calculator or interpreter: turn text into a list of well-typed tokens.

</details>

## Quiz

1. Struct or enum: a user with a name and an age; the state of a door (open, closed, locked)?

<details>
<summary>Answer</summary>

The user is a struct (name **and** age). The door state is an enum (open **or** closed **or** locked).

</details>

2. What is a variant's payload?

<details>
<summary>Answer</summary>

The data that variant carries, for example the `f64` in `Circle(f64)`. Every variant may have a different payload, or none.

</details>

3. What does "a `match` must be exhaustive" mean, and what does the compiler do if it is not?

<details>
<summary>Answer</summary>

Every possible value must be covered by some arm. If an arm is missing, compilation fails with `E0004`, naming the uncovered case.

</details>

4. Why can adding a new variant to an enum be a *good* thing for the compiler to report?

<details>
<summary>Answer</summary>

Every `match` without a wildcard arm stops compiling, so the compiler shows all the places where the new case must be handled. Nothing is forgotten silently.

</details>

5. How big is an enum value in memory?

<details>
<summary>Answer</summary>

As big as its largest variant plus a tag that records the current variant (rounded up for alignment). All values of the same enum have the same size.

</details>

6. How do you convert an enum with discriminants to a number, and the number back to the enum?

<details>
<summary>Answer</summary>

Enum to number: `value as u8`. Number to enum: there is no automatic way, because not every number is valid. Write a function with `match` that returns `Option<Enum>` (`None` for unknown numbers).

</details>

7. Why parse text into an enum at the border of the program instead of passing strings around?

<details>
<summary>Answer</summary>

After parsing, only valid values exist inside the program, and every `match` over the enum is complete. Unknown or malicious input is rejected once, at the border, as `None` (or later an error).

</details>

8. When is `_` a good arm in a `match` over an enum, and when is it a bad idea?

<details>
<summary>Answer</summary>

Good when "everything else" really is one case (like weekend days). Bad when you should treat each variant deliberately, because a new variant would silently fall into `_` and the compiler would not warn you.

</details>

## Summary

- A **struct** is "this and this"; an **enum** is "this or this". An enum value is always exactly one variant.
- Create values with `Enum::Variant`; variants may be unit, tuple (`Circle(f64)`) or struct-like (`Rectangle { width, height }`) and each can carry different data.
- `match` takes an enum apart, binds the payload to names, and must be **exhaustive** — the compiler finds every place that needs updating when you add a variant.
- Use enums to model states and to make illegal states unrepresentable.
- Enums can have methods in `impl` blocks, and derive `Debug`, `Clone`, `Copy` and `PartialEq` like structs.
- A C-like enum can have numbers (`Low = 1`); `as` converts enum to number, but number to enum must be a checked `match`.
- In memory an enum is a tag plus room for its largest variant.
- Parse untrusted text into an enum once and reject everything else.

## Sources and further reading

- [The Rust Programming Language — 6.1 Defining an Enum](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html) — the official book chapter on this topic.
- [The Rust Programming Language — 6.2 The match Control Flow Construct](https://doc.rust-lang.org/book/ch06-02-match.html) — more on `match`, which Lesson 04.4 continues.
- [Rust by Example — Enums](https://doc.rust-lang.org/rust-by-example/custom_types/enum.html) — short runnable examples, including C-like enums.
- [The Rust Reference — Enumerations](https://doc.rust-lang.org/reference/items/enumerations.html) — the exact rules, including discriminants and casts.
- [std::mem::size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) — documentation of the function used in section 6.

---

[🏠 Start](../START-HERE.md) · [← 04.2 Methods and associated functions](02-methods-and-associated-functions.md) · [📋 Phase 04](README.md) · [04.4 Option and match →](04-option-and-match.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
