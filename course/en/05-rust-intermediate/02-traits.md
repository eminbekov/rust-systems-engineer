<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.1 Generics](01-generics.md) · [📋 Phase 05](README.md) · [05.3 Lifetimes →](03-lifetimes.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/02-traits.md) <!-- nav -->

# Lesson 05.2 — Traits

> **Phase 05 — Rust intermediate** · Lesson 2 of 14 · ⏱ about 4 hours (3 sittings)

## What you will learn

- What a trait is: a named set of abilities that different types can share.
- How to define a trait, implement it for your types, and give methods default bodies.
- How to accept "anything that has this ability" in a function: `impl Trait`, trait bounds and `where`.
- How to use the standard traits: `#[derive(...)]`, `Display`, `From`/`Into` and operators such as `+`.
- Blanket implementations, supertraits and the orphan rule, and how to read the common trait errors.

## Before you start

- You finished [Lesson 05.1 — Generics](01-generics.md). Trait bounds were introduced there, and
  today you learn what stands on the right side of the colon.
- You have the practice package `code/p05-lessons` from Lesson 05.1 (a folder `examples/` inside it).
- You know modules ([Lesson 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md))
  and the `Option`/`Result` types ([Lesson 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)).

## New words

| Word | Meaning |
|---|---|
| **trait** | A named list of method signatures that a type can promise to provide. Like a job description: it says *what* must be possible, not *how*. |
| **implement** | To write `impl Trait for Type { … }`, giving the type the methods the trait lists. |
| **default method** | A method that has a body inside the trait. Types get it for free and may replace it. |
| **`impl Trait`** | A way to write a parameter or return type as "some type that implements this trait" without naming it. |
| **derive** | `#[derive(Trait)]`: ask the compiler to write a standard trait implementation for you. |
| **operator overloading** | Making `+`, `-`, `*`, `==` and similar operators work for your own types by implementing a trait. |
| **blanket implementation** | An implementation for *every* type that meets a bound: `impl<T: Display> Shout for T`. |
| **supertrait** | A trait that requires another trait: `trait Greeter: Named` means every `Greeter` must also be `Named`. |
| **orphan rule** | You may implement a trait for a type only if the trait **or** the type is defined in your own crate. |
| **associated function** | A function in an `impl` or a trait that has no `self`, called as `Type::function()`. |
| **associated type** | A type name inside a trait that each implementation fills in (`type Output = …`). |
| **scope** | The set of names that are visible at a given place in the code. A trait's methods can be called only if the trait is in scope. |
| **`Debug`, `Display`** | Traits for printing: `{:?}` is for programmers, `{}` is for users. |

## 1. What a trait is

Think of a job description: "Driver: must be able to `start()`, `stop()`, `turn()`." Many people
can be drivers, each in their own way. The description does not say *how* to drive; it only lists
what you must be able to do. A **trait** is such a description for types.

Here is a trait `Shape` for things that have an area:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> String;

    fn describe(&self) -> String {
        format!("{} with area {:.2}", self.name(), self.area())
    }
}

struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }

    fn name(&self) -> String {
        String::from("circle")
    }
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn name(&self) -> String {
        String::from("rectangle")
    }

    fn describe(&self) -> String {
        format!("{}x{} rectangle", self.width, self.height)
    }
}

fn main() {
    let c = Circle { radius: 2.0 };
    let r = Rectangle { width: 3.0, height: 4.5 };
    println!("{}", c.describe());
    println!("{}", r.describe());
}
```

**What happens:**

1. `trait Shape { … }` declares the trait. `fn area(&self) -> f64;` ends with `;` and has no body:
   it is only a *signature*. Every implementing type must provide the body.
2. `describe` **has** a body. It is a **default method**: any type that implements `Shape` gets
   `describe` automatically. Notice that the default body can call the other trait methods
   (`self.name()`, `self.area()`).
3. `impl Shape for Circle { … }` says "`Circle` is a `Shape`" and provides the two required
   methods. `Circle` does not write `describe`, so it uses the default.
4. `Rectangle` also writes its own `describe`. That *overrides* the default.
5. You call trait methods with the usual dot syntax: `c.describe()`.

```text
circle with area 12.57
3x4.5 rectangle
```

> 📏 **Convention:** Trait names are `CamelCase` and usually name an ability or a role (`Shape`,
> `Display`, `Iterator`). Method names are `snake_case`. Keep traits small: two or three methods
> that belong together. A small trait is easy to implement and easy to understand.

## 2. Traits as parameters and return types

A trait is useful because a function can say "I accept any type that is a `Shape`". There are three
spellings, and all mean the same thing:

```rust
fn print_area<T: Shape>(shape: &T) {
    println!("area of the {} is {:.1}", shape.name(), shape.area());
}

fn print_description(shape: &impl Shape) {
    println!("{}", shape.describe());
}

fn print_both<A, B>(first: &A, second: &B)
where
    A: Shape,
    B: Shape,
{
    println!("{} and {}", first.name(), second.name());
}
```

**What happens:**

1. `T: Shape` — a generic function with a trait bound (Lesson 05.1). Use it when two arguments
   must be the *same* type, or when you need to name the type again.
2. `&impl Shape` — the short form. Each `impl Shape` argument may be a different type.
3. `where` — the same bound, moved below the signature. Use it when the list is long.

With the `Circle` and `Rectangle` above:

```rust
fn main() {
    let c = Circle { radius: 2.0 };
    let r = Rectangle { width: 3.0, height: 4.5 };
    print_area(&c);
    print_area(&r);
    print_description(&c);
    print_description(&r);
    print_both(&c, &r);
}
```

```text
area of the circle is 12.6
area of the rectangle is 13.5
circle with area 12.57
3x4.5 rectangle
circle and rectangle
```

A function can also *return* "some type that implements a trait":

```rust
fn unit_square() -> impl Shape {
    Rectangle { width: 1.0, height: 1.0 }
}
```

The caller only knows "it is a `Shape`" and can call `unit_square().describe()`. The restriction:
the function must return **one** concrete type. It cannot return a `Circle` in one branch and a
`Rectangle` in another. (Lesson 06.1 shows trait objects, which allow that.)

If you pass a type that does not implement the trait, the compiler tells you precisely:

```rust
fn main() {
    print_area(&5);
}
```

```text
error[E0277]: the trait bound `{integer}: Shape` is not satisfied
  --> examples/err_bound.rs:18:16
   |
18 |     print_area(&5);
   |     ---------- ^^ the trait `Shape` is not implemented for `{integer}`
   |     |
   |     required by a bound introduced by this call
   |
help: the trait `Shape` is implemented for `Circle`
```

**What it means:** `print_area` requires `T: Shape`, and the number `5` has no `impl Shape`. The
`help:` line lists the types that do.

> ⚡ **Performance:** Functions with `T: Shape` or `impl Shape` use **static dispatch**. As in
> Lesson 05.1, the compiler makes one copy per concrete type, and the call to `area` is resolved at
> compile time, with no run-time lookup. This is the fast, default way to use traits.

## 3. The standard traits and `derive`

The standard library defines dozens of traits. A few appear in almost every program:

| Trait | What it gives you | How you get it |
|---|---|---|
| `Debug` | printing with `{:?}` for programmers | `#[derive(Debug)]` |
| `Clone` | `.clone()`: an explicit deep copy | `#[derive(Clone)]` |
| `Copy` | the value is copied automatically on assignment (Lesson 03.3) | `#[derive(Clone, Copy)]` |
| `PartialEq` / `Eq` | `==` and `!=` | `#[derive(PartialEq, Eq)]` |
| `PartialOrd` / `Ord` | `<`, `>`, `.max()`, sorting | `#[derive(PartialOrd, Ord)]` |
| `Hash` | usable as a `HashMap` key (Lesson 04.9) | `#[derive(Hash)]` |
| `Default` | `Type::default()`, a sensible "empty" value | `#[derive(Default)]` |
| `Display` | printing with `{}` for users, and `.to_string()` | write it by hand |
| `From` / `Into` | conversions between types | write `From` by hand |
| `Add`, `Mul`, … | operators `+`, `*`, … | write by hand |

The attribute `#[derive(...)]` asks the compiler to write the obvious implementation. For a struct,
"obvious" means "do it field by field":

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord, Default)]
struct Version {
    major: u32,
    minor: u32,
}

fn main() {
    let a = Version { major: 1, minor: 4 };
    let b = Version { major: 1, minor: 10 };
    let c = a; // Copy: a is still usable
    println!("{:?} {:?}", a, b);
    println!("a == c: {}", a == c);
    println!("a < b: {}", a < b);
    println!("default: {:?}", Version::default());
    println!("max: {:?}", a.max(b));

    let mut notes: HashMap<Version, &str> = HashMap::new();
    notes.insert(a, "stable");
    notes.insert(b, "beta");
    println!("note for a: {:?}", notes.get(&a));
}
```

**What happens:**

1. One line of `derive` gives `Version` nine abilities.
2. `let c = a;` copies `a` (because of `Copy`), so `a` is still valid on the next line.
3. `a < b` compares field by field in the order they are written: `major` first (1 and 1: equal),
   then `minor` (4 < 10), so `a < b` is true. **The order of fields matters** for a derived `Ord`.
4. `Version::default()` makes `Version { major: 0, minor: 0 }`: the default of every field.
5. Because of `Hash` and `Eq`, `Version` can be a key of a `HashMap`.

```text
Version { major: 1, minor: 4 } Version { major: 1, minor: 10 }
a == c: true
a < b: true
default: Version { major: 0, minor: 0 }
max: Version { major: 1, minor: 10 }
note for a: Some("stable")
```

Forget a derive and the compiler asks for it:

```rust
#[derive(Debug)]
struct Money {
    cents: i64,
}

fn main() {
    let a = Money { cents: 100 };
    let b = Money { cents: 100 };
    println!("{}", a == b);
}
```

```text
error[E0369]: binary operation `==` cannot be applied to type `Money`
 --> examples/err_compare.rs:9:22
  |
9 |     println!("{}", a == b);
  |                    - ^^ - Money
  |                    |
  |                    Money
  |
note: an implementation of `PartialEq` might be missing for `Money`
help: consider annotating `Money` with `#[derive(PartialEq)]`
```

> 📏 **Convention:** Derive `Debug` on almost every type you write, so that you can print it while
> debugging. Add `Clone`, `PartialEq` and `Default` when they make sense. Derive `Copy` only for
> small types that behave like plain values.

## 4. Implementing standard traits by hand

### `Display`: printing for users

`{:?}` (`Debug`) is for you, the programmer. `{}` (`Display`) is for the user of your program, and
Rust cannot guess its format, so you write it:

```rust
use std::fmt;

struct Temperature {
    celsius: f64,
}

impl fmt::Display for Temperature {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:.1} °C", self.celsius)
    }
}

fn main() {
    let t = Temperature { celsius: 21.456 };
    println!("now: {}", t);
    let text: String = t.to_string();
    println!("as text: {text} (length {})", text.len());
}
```

**What happens:**

1. `impl fmt::Display for Temperature` implements the trait from `std::fmt`.
2. The only method is `fmt`. It receives `f`, a `Formatter`: the "output pen". `fmt::Result` is
   `Result<(), fmt::Error>`, as in Lesson 04.10.
3. `write!(f, …)` works like `println!`, but writes into `f` instead of the screen. Its result is
   the return value.
4. After that, `{}` works with `Temperature`, and you get `.to_string()` for free (Section 5
   explains why).
5. `len()` is the number of **bytes**, not characters: `°` takes 2 bytes in UTF-8 (Unicode Transformation Format, 8-bit: the standard way of storing text as bytes, Lesson 04.8),
   so the text `21.5 °C` is 8 bytes long.

```text
now: 21.5 °C
as text: 21.5 °C (length 8)
```

Without such an implementation, printing with `{}` fails. Here `p` is a value of a small struct `Point { x: i32, y: i32 }` that has no `Display` implementation:

```text
error[E0277]: `Point` doesn't implement `std::fmt::Display`
 --> examples/err_nodisplay.rs:8:20
  |
8 |     println!("{}", p);
  |               --   ^ `Point` cannot be formatted with the default formatter
  |
  = note: in format strings you may be able to use `{:?}` (or `{:#?}` for pretty-print) instead
```

### `From` and `Into`: conversions

`From<A> for B` says "a `B` can be made from an `A`". If you write it, you get `Into` for free:

```rust
struct Celsius(f64);
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

fn show(t: impl Into<Fahrenheit>) {
    let f: Fahrenheit = t.into();
    println!("{:.1} °F", f.0);
}

fn main() {
    let boiling = Fahrenheit::from(Celsius(100.0));
    println!("{:.1}", boiling.0);
    let body: Fahrenheit = Celsius(36.6).into();
    println!("{:.1}", body.0);
    show(Celsius(0.0));
    show(Fahrenheit(70.0));
}
```

**What happens:**

1. `struct Celsius(f64);` is a *tuple struct* (Lesson 04.1): one unnamed field, read as `.0`.
2. `impl From<Celsius> for Fahrenheit` defines the conversion. `Self` is `Fahrenheit`.
3. `Fahrenheit::from(Celsius(100.0))` and `Celsius(36.6).into()` do the same thing. `into()` needs
   a type annotation (`let body: Fahrenheit`) so the compiler knows the target.
4. `show` accepts `impl Into<Fahrenheit>`: a `Celsius`, or a `Fahrenheit` (every type converts into
   itself). This makes functions flexible for the caller.

```text
212.0
97.9
32.0 °F
70.0 °F
```

> 💡 **Tip:** The `?` operator (Lesson 04.10) uses `From` to convert errors. That is why you can
> write `?` on a function that returns a different error type, once `From` is implemented.

### `Add`: operators are traits

`a + b` is just a call to the method `add` of the trait `std::ops::Add`. Implement the trait and
the operator works for your type. This is called **operator overloading**:

```rust
use std::ops::{Add, Mul};

#[derive(Debug, Clone, Copy, PartialEq, Default)]
struct Vec2 {
    x: f64,
    y: f64,
}

impl Add for Vec2 {
    type Output = Vec2;

    fn add(self, other: Vec2) -> Vec2 {
        Vec2 { x: self.x + other.x, y: self.y + other.y }
    }
}

impl Mul<f64> for Vec2 {
    type Output = Vec2;

    fn mul(self, k: f64) -> Vec2 {
        Vec2 { x: self.x * k, y: self.y * k }
    }
}

fn main() {
    let a = Vec2 { x: 1.0, y: 2.0 };
    let b = Vec2 { x: 0.5, y: -1.0 };
    println!("{:?}", a + b);
    println!("{:?}", a * 3.0);
    println!("{:?}", Vec2::default() + a);
}
```

**What happens:**

1. `type Output = Vec2;` is an **associated type**: each implementation states what type the
   operator returns. (`Iterator` uses the same idea; Lesson 05.5.)
2. `impl Add for Vec2` means `Vec2 + Vec2`. `impl Mul<f64> for Vec2` means `Vec2 * f64`. The
   `<f64>` is the type on the right side of the `*`.
3. `add` takes `self` by value. That is fine here because `Vec2` is `Copy`, so `a` stays usable.
4. `Vec2::default()` comes from `derive(Default)` and is the zero vector.

```text
Vec2 { x: 1.5, y: 1.0 }
Vec2 { x: 3.0, y: 6.0 }
Vec2 { x: 1.0, y: 2.0 }
```

> 📏 **Convention:** Overload an operator only when its meaning is obvious (adding vectors,
> money, durations). A `+` that does something surprising makes code harder to read.

## 5. More trait features

### Trait bounds on `impl` blocks and traits with `Self`

You can implement a trait, or add methods, only for types that satisfy a bound:

```rust
use std::fmt::{self, Display};

struct Wrapper<T> {
    value: T,
}

impl<T: Display> Display for Wrapper<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.value)
    }
}

impl<T: PartialOrd> Wrapper<T> {
    fn bigger_than(&self, other: &T) -> bool {
        self.value > *other
    }
}

fn main() {
    let w = Wrapper { value: 7 };
    println!("{} {}", w, w.bigger_than(&5));
    let s = Wrapper { value: "text" };
    println!("{}", s);
}
```

**What happens:** `Wrapper<T>` is `Display` **only when** `T` is `Display`. The method
`bigger_than` exists only when `T` is `PartialOrd`. This is exactly how `Vec<T>` is `Debug` only
if `T` is `Debug`.

```text
[7] true
[text]
```

A trait can also contain functions without `self`, which return `Self`. Combined with a bound, you
can write generic code that creates values:

```rust
trait Zero {
    fn zero() -> Self;
}

impl Zero for i32 {
    fn zero() -> Self {
        0
    }
}

impl Zero for f64 {
    fn zero() -> Self {
        0.0
    }
}

fn sum_all<T: Zero + std::ops::Add<Output = T> + Copy>(list: &[T]) -> T {
    let mut total = T::zero();
    for &x in list {
        total = total + x;
    }
    total
}

fn main() {
    println!("{}", sum_all(&[1, 2, 3]));
    println!("{}", sum_all(&[0.5, 0.25]));
    println!("{}", sum_all::<i32>(&[]));
}
```

**What happens:** `T::zero()` calls the associated function of whatever type `T` is. We even
implemented our own trait for the built-in types `i32` and `f64` (allowed: the trait is ours, see
the orphan rule below). `Add<Output = T>` says that adding two `T` values gives a `T` again.

```text
6
0.75
0
```

### Blanket implementations

An implementation for *all* types that meet a bound is called a **blanket implementation**:

```rust
use std::fmt::Display;

trait Shout {
    fn shout(&self) -> String;
}

impl<T: Display> Shout for T {
    fn shout(&self) -> String {
        format!("{}!", self.to_string().to_uppercase())
    }
}

fn main() {
    println!("{}", "hello".shout());
    println!("{}", 42.shout());
    println!("{}", 2.5.shout());
}
```

**What happens:** every `Display` type, now and in the future, gets `shout`. The standard library
does the same: `impl<T: Display> ToString for T` is why every `Display` type has `.to_string()`.

```text
HELLO!
42!
2.5!
```

### Supertraits

A trait can require another trait. `Greeter: Named` means "to be a `Greeter`, a type must also be
`Named`", so default methods of `Greeter` may use `name()`:

```rust
trait Named {
    fn name(&self) -> String;
}

trait Greeter: Named {
    fn greet(&self) -> String {
        format!("Hello, {}!", self.name())
    }
}

struct Student {
    name: String,
}

impl Named for Student {
    fn name(&self) -> String {
        self.name.clone()
    }
}

impl Greeter for Student {}

fn main() {
    let s = Student { name: String::from("Dilnoza") };
    println!("{}", s.greet());
}
```

**What happens:** `impl Greeter for Student {}` has an empty body: everything comes from defaults,
and `Named` is implemented separately. If you forget `impl Named`, the compiler refuses.

```text
Hello, Dilnoza!
```

### The orphan rule

You may implement a trait for a type only if **the trait or the type belongs to your crate**. You
cannot, for example, implement the standard `Display` for the standard `Vec<i32>`:

```rust
use std::fmt;

impl fmt::Display for Vec<i32> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "a vector")
    }
}
```

```text
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
 --> examples/err_orphan.rs:3:1
  |
3 | impl fmt::Display for Vec<i32> {
  | ^^^^^^^^^^^^^^^^^^^^^^--------
  |                       |
  |                       `Vec` is not defined in the current crate
  |
  = note: define and implement a trait or new type instead
```

**Why the rule exists:** if two different libraries both implemented `Display` for `Vec<i32>`, the
compiler could not choose between them. **The fix** is the hint in the last line: wrap the foreign
type in your own struct (`struct Numbers(Vec<i32>);`, the "newtype" pattern, which Phase 08 covers)
and implement the trait for your wrapper.

### What traits cannot do yet

All the code above has a limit: `Vec<Shape>` is impossible, because a vector holds values of **one**
type, and `Circle` and `Rectangle` are two types. A list of "different shapes" needs *trait
objects* (`dyn Shape`), which are the topic of [Lesson 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md).
Today you use traits through generics, which is the faster option.

## Common mistakes

**1. Missing a required method** — `error[E0046]: not all trait items implemented, missing: 'name'`:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> String;
}

struct Square(f64);

impl Shape for Square {
    fn area(&self) -> f64 {
        self.0 * self.0
    }
}
```

```text
error[E0046]: not all trait items implemented, missing: `name`
 --> examples/err_missing.rs:8:1
  |
3 |     fn name(&self) -> String;
  |     ------------------------- `name` from trait
...
8 | impl Shape for Square {
  | ^^^^^^^^^^^^^^^^^^^^^ missing `name` in implementation
```

**What it means:** the trait lists `name` without a body, so every implementation must provide it.
Write the method, or give it a default body in the trait.

**2. Calling a trait method without the trait in scope.**

```rust
mod animals {
    pub trait Speak {
        fn speak(&self) -> String;
    }

    pub struct Dog;

    impl Speak for Dog {
        fn speak(&self) -> String {
            String::from("woof")
        }
    }
}

fn main() {
    let d = animals::Dog;
    println!("{}", d.speak());
}
```

```text
error[E0599]: no method named `speak` found for struct `Dog` in the current scope
  --> examples/err_scope.rs:17:22
   |
17 |     println!("{}", d.speak());
   |                      ^^^^^ method not found in `Dog`
   |
   = help: items from traits can only be used if the trait is in scope
help: trait `Speak` which provides `speak` is implemented but not in scope; perhaps you want to import it
   |
 1 + use crate::animals::Speak;
   |
```

**What it means:** the method belongs to the *trait*, not to `Dog`. To call it you must import the
trait with `use`. The compiler even gives you the exact line to add.

**3. Missing `derive`** (Section 3): `E0369` for `==`, `E0277` for `{:?}` without `Debug`. Add the
derive.

**4. Implementing a foreign trait for a foreign type** (Section 5): `E0117`. Use a newtype wrapper.

**5. Returning two different types from `-> impl Trait`.** The function must return a single
concrete type; use an enum or wait for trait objects (Lesson 06.1).

## Try it yourself

1. Go to your repository and start a branch for this lesson:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-02-traits
   ```

2. Save the `Shape` program of Section 1 as `code/p05-lessons/examples/traits_shape.rs` and run it:

   ```bash
   cd code/p05-lessons
   cargo run --example traits_shape
   ```

   Add `print_area` and `print_both` from Section 2 and run again.

3. Commit it. Commit **small and often**: after every working example, not once at the end of the
   day. Use a *type* in the message so the log can be read at a glance: `feat` (new thing), `fix`,
   `test`, `docs`, `refactor`.

   ```bash
   cd ~/projects/rust
   git add .
   git commit -m "feat(p05): Shape trait with a default method"
   ```

4. Add the `Display` example (`traits_display.rs`). Before committing, review what exactly you are
   about to save:

   ```bash
   git add .
   git diff --staged --stat
   ```

   ```text
    code/p05-lessons/examples/traits_display.rs | 18 ++++++++++++++++++
    1 file changed, 18 insertions(+)
   ```

   `--staged` shows what is in the "staging area" (Lesson 01.3), and `--stat` shows only a summary
   per file. It is a good last look before every commit: *is only what I wanted in there?*

   ```bash
   git commit -m "feat(p05): Display for Temperature"
   ```

5. Do Exercise 5 (a `Display` for a stack, with tests) and commit it as `test(p05): …`. Then look
   at the story you wrote:

   ```bash
   git log --oneline
   ```

   ```text
   56700dc test(p05): tests for Display of Stack
   475649f feat(p05): Display for Temperature
   474f069 feat(p05): Shape trait with a default method
   8f26f07 chore: start
   ```

   (Your hashes will differ, and below these lines you will also see the older commits from earlier lessons.) A reader can understand the work from the messages alone.

6. Merge the lesson into `main` (`git switch main`, `git merge lesson/05-02-traits`) and delete the
   branch as in Lesson 05.1.

## Exercises

### Exercise 1 — Animals (easy)

Define a trait `Animal` with required methods `name(&self) -> String` and `sound(&self) -> String`,
and a default method `speak(&self) -> String` that returns `"<name> says <sound>"`. Implement it
for `Dog` (a unit struct) and for `Cat` (a struct with a `name` field) where `Cat` overrides
`speak` to return `"<name> purrs"`. Write tests for both.

<details>
<summary>Hint</summary>

The default method can call `self.name()` and `self.sound()` inside `format!`. A unit struct is
declared as `struct Dog;` and used as the value `Dog`. Override by writing `fn speak` inside
`impl Animal for Cat`.

</details>

<details>
<summary>Answer</summary>

```rust
trait Animal {
    fn name(&self) -> String;
    fn sound(&self) -> String;

    fn speak(&self) -> String {
        format!("{} says {}", self.name(), self.sound())
    }
}

struct Dog;
struct Cat {
    name: String,
}

impl Animal for Dog {
    fn name(&self) -> String {
        String::from("Dog")
    }

    fn sound(&self) -> String {
        String::from("woof")
    }
}

impl Animal for Cat {
    fn name(&self) -> String {
        self.name.clone()
    }

    fn sound(&self) -> String {
        String::from("meow")
    }

    fn speak(&self) -> String {
        format!("{} purrs", self.name)
    }
}

fn main() {
    println!("{}", Dog.speak());
    println!("{}", Cat { name: String::from("Murka") }.speak());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn default_speak_uses_name_and_sound() {
        assert_eq!(Dog.speak(), "Dog says woof");
    }

    #[test]
    fn cat_overrides_speak() {
        let c = Cat { name: String::from("Tom") };
        assert_eq!(c.speak(), "Tom purrs");
    }
}
```

```text
Dog says woof
Murka purrs
```

Why it works: `Dog` uses the default `speak`; `Cat` replaces it. `cargo test --example <name>` reports
`2 passed`.

</details>

### Exercise 2 — Display with a test (easy)

Implement `Display` for `struct Temperature { celsius: f64 }` so that `21.456` prints as
`21.5 °C`. Write tests that call `.to_string()` for a positive and a negative value.

<details>
<summary>Hint</summary>

Use `write!(f, "{:.1} °C", self.celsius)`. `{:.1}` means "one digit after the decimal point".
`to_string()` is available as soon as `Display` is implemented.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fmt;

struct Temperature {
    celsius: f64,
}

impl fmt::Display for Temperature {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:.1} °C", self.celsius)
    }
}

fn main() {
    println!("{}", Temperature { celsius: -3.04 });
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn displays_one_decimal_and_unit() {
        let t = Temperature { celsius: 21.456 };
        assert_eq!(t.to_string(), "21.5 °C");
    }

    #[test]
    fn displays_negative_values() {
        assert_eq!(Temperature { celsius: -3.04 }.to_string(), "-3.0 °C");
    }
}
```

```text
-3.0 °C
```

Why it works: testing the *text* an object produces is the simplest way to test `Display`.

</details>

### Exercise 3 — Add two vectors (medium)

Implement `Add` for `Vec2 { x: f64, y: f64 }` (derive `Debug`, `Clone`, `Copy`, `PartialEq`,
`Default`). Write one test for adding two vectors and one test that adding `Vec2::default()`
changes nothing.

<details>
<summary>Hint</summary>

`impl Add for Vec2 { type Output = Vec2; fn add(self, other: Vec2) -> Vec2 { … } }` and
`use std::ops::Add;`. To compare vectors in `assert_eq!` you need `PartialEq` and `Debug`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy, PartialEq, Default)]
struct Vec2 {
    x: f64,
    y: f64,
}

impl Add for Vec2 {
    type Output = Vec2;

    fn add(self, other: Vec2) -> Vec2 {
        Vec2 { x: self.x + other.x, y: self.y + other.y }
    }
}

fn main() {
    let a = Vec2 { x: 1.0, y: 2.0 };
    println!("{:?}", a + Vec2::default());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_component_by_component() {
        let a = Vec2 { x: 1.0, y: 2.0 };
        let b = Vec2 { x: 0.5, y: -1.0 };
        assert_eq!(a + b, Vec2 { x: 1.5, y: 1.0 });
    }

    #[test]
    fn default_is_the_zero_vector() {
        let a = Vec2 { x: 4.0, y: 5.0 };
        assert_eq!(a + Vec2::default(), a);
    }
}
```

```text
Vec2 { x: 1.0, y: 2.0 }
```

Why it works: the numbers in these tests (1.0, 0.5, …) are exactly representable in binary, so
comparing floats with `==` is safe here. In general, compare floats with a small tolerance (Lesson 05.6).

</details>

### Exercise 4 — Two-way temperature conversion (medium)

Using `Celsius(f64)` and `Fahrenheit(f64)`, implement `From<Celsius> for Fahrenheit` **and**
`From<Fahrenheit> for Celsius`. Write a generic function `print_in_fahrenheit<T: Into<Fahrenheit>>(t: T)`
that prints the value with one decimal and the text ` °F`. Test that 100 °C is 212 °F and that a
round trip of 36.6 °C comes back (almost) unchanged.

<details>
<summary>Hint</summary>

°F = °C × 9 / 5 + 32, and °C = (°F − 32) × 5 / 9. For the round-trip test compare with a tolerance:
`(back.0 - 36.6).abs() < 1e-9`. Derive `Debug` and `PartialEq` on the structs so `assert_eq!` works.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq)]
struct Celsius(f64);
#[derive(Debug, PartialEq)]
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Self {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

impl From<Fahrenheit> for Celsius {
    fn from(f: Fahrenheit) -> Self {
        Celsius((f.0 - 32.0) * 5.0 / 9.0)
    }
}

fn print_in_fahrenheit<T: Into<Fahrenheit>>(t: T) {
    let f: Fahrenheit = t.into();
    println!("{:.1} °F", f.0);
}

fn main() {
    print_in_fahrenheit(Celsius(100.0));
    print_in_fahrenheit(Fahrenheit(70.0));
    let c: Celsius = Fahrenheit(212.0).into();
    println!("{:?}", c);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn boiling_point() {
        assert_eq!(Fahrenheit::from(Celsius(100.0)), Fahrenheit(212.0));
    }

    #[test]
    fn round_trip_is_close() {
        let back: Celsius = Fahrenheit::from(Celsius(36.6)).into();
        assert!((back.0 - 36.6).abs() < 1e-9);
    }
}
```

```text
212.0 °F
70.0 °F
Celsius(100.0)
```

Why it works: each `From` gives the matching `Into` automatically. The tolerance in the second test
is needed because floating-point arithmetic is rarely exact.

</details>

### Exercise 5 — Display for a generic stack (hard)

Take the `Stack<T>` from Lesson 05.1 and implement `Display` for `Stack<T>` **only when** `T: Display`.
The format is `[1, 2, 3]`, and `[]` for an empty stack. Write tests for both cases.

<details>
<summary>Hint</summary>

`impl<T: Display> Display for Stack<T>`. Use `write!(f, "[")?;` first. Loop over `&self.items` with a
variable `first` (a `bool`) so you write `", "` before every item except the first. The `?` after
each `write!` passes along an error. Finish with `write!(f, "]")`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fmt::{self, Display};

struct Stack<T> {
    items: Vec<T>,
}

impl<T: Display> Display for Stack<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[")?;
        let mut first = true;
        for item in &self.items {
            if !first {
                write!(f, ", ")?;
            }
            write!(f, "{}", item)?;
            first = false;
        }
        write!(f, "]")
    }
}

fn main() {
    let s = Stack { items: vec![1, 2, 3] };
    println!("{}", s);
    let empty: Stack<String> = Stack { items: vec![] };
    println!("{}", empty);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn shows_items_separated_by_commas() {
        let s = Stack { items: vec!["a", "b"] };
        assert_eq!(s.to_string(), "[a, b]");
    }

    #[test]
    fn empty_stack_is_two_brackets() {
        let s: Stack<i32> = Stack { items: Vec::new() };
        assert_eq!(s.to_string(), "[]");
    }
}
```

```text
[1, 2, 3]
[]
```

Why it works: the bound `T: Display` on the `impl` makes `Stack<T>` printable exactly when its items
are printable, and the same code serves numbers, `&str` and `String`.

</details>

## Quiz

1. What is the difference between a *required* method and a *default* method in a trait?

<details>
<summary>Answer</summary>

A required method has only a signature (ends with `;`); every implementing type must write its body. A default method has a body in the trait; implementing types get it for free and may override it.

</details>

2. What is the difference between `fn f<T: Shape>(x: &T)` and `fn f(x: &impl Shape)`?

<details>
<summary>Answer</summary>

They mean the same for one parameter. The `T` form lets you name the type again (for example to make two arguments the same type or to use `T` in the body); the `impl` form is shorter when you do not need that.

</details>

3. Why does `println!("{}", p)` fail for a struct, and what are two ways to fix it?

<details>
<summary>Answer</summary>

Because the struct does not implement `Display`, and Rust cannot guess a user-facing format. Either implement `std::fmt::Display` by hand, or print with `{:?}` after adding `#[derive(Debug)]`.

</details>

4. What does `impl From<A> for B` give you besides `B::from(a)`?

<details>
<summary>Answer</summary>

A free implementation of `Into<B> for A`, so you can write `a.into()` (with the target type known), and the `?` operator can use it to convert error types.

</details>

5. State the orphan rule and give one thing it forbids.

<details>
<summary>Answer</summary>

You may implement a trait for a type only if the trait or the type is defined in your crate. So you cannot implement the standard `Display` for the standard `Vec<i32>`. Wrap the type in your own struct (newtype) instead.

</details>

6. Why can't you call `d.speak()` when `Speak` is defined in a module you did not import?

<details>
<summary>Answer</summary>

Trait methods can be called only when the trait is in scope. The method belongs to the trait, so you need `use path::Speak;`. The compiler (`E0599`) tells you which `use` line to add.

</details>

## Summary

- A trait is a named set of method signatures. `impl Trait for Type` gives a type those abilities.
- Required methods have only signatures; default methods have bodies and can be overridden.
- Accept "anything with this ability" using `T: Trait`, `&impl Trait` or `where`; this is static dispatch and has no run-time cost.
- `#[derive(Debug, Clone, PartialEq, …)]` writes standard implementations for you. Field order matters for derived ordering.
- Write `Display` by hand for user-facing text, `From` for conversions (you get `Into` free), and `Add`/`Mul` for operators.
- Blanket implementations, supertraits and bounds on `impl` blocks let one trait definition cover many types.
- The orphan rule forbids foreign trait + foreign type; use a newtype wrapper.
- Git habit: small commits with a type (`feat`, `test`, `fix`), and `git diff --staged --stat` before every commit.

## Sources and further reading

- [The Rust Programming Language — 10.2 Traits](https://doc.rust-lang.org/book/ch10-02-traits.html) — the official chapter on defining and using traits.
- [Rust by Example — Traits](https://doc.rust-lang.org/rust-by-example/trait.html) — short runnable examples, including `derive` and operator overloading.
- [std::fmt::Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) — the trait and examples of implementing it.
- [std::convert::From](https://doc.rust-lang.org/std/convert/trait.From.html) — conversions and the `Into` relationship.
- [std::ops::Add](https://doc.rust-lang.org/std/ops/trait.Add.html) — operator traits.
- [std::default::Default](https://doc.rust-lang.org/std/default/trait.Default.html) — `Default` and `derive(Default)`.
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) — the message format behind `feat:`, `fix:`, `test:`.

---

[🏠 Start](../START-HERE.md) · [← 05.1 Generics](01-generics.md) · [📋 Phase 05](README.md) · [05.3 Lifetimes →](03-lifetimes.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
