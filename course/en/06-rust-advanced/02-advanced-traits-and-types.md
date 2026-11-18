<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.1 Trait objects and dynamic dispatch](01-trait-objects-and-dynamic-dispatch.md) · [📋 Phase 06](README.md) · [06.3 Patterns in depth →](03-patterns-in-depth.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/02-advanced-traits-and-types.md) <!-- nav -->

# Lesson 06.2 — Advanced traits and types

> **Phase 06 — Rust advanced** · Lesson 2 of 12 · ⏱ about 4 hours (3 sittings)

## What you will learn

- Associated types, and when to choose them instead of a generic trait.
- Default type parameters and operator overloading (`+`, `-`, ...).
- Fully qualified syntax to call the exact method you mean.
- Supertraits, the newtype pattern and the orphan rule.
- Type aliases, the never type `!`, and types without a known size (`str`, `[T]`, `dyn Trait`).
- Function pointers (`fn`) compared with closures, and how to return a closure.

## Before you start

- You finished [Lesson 06.1 — Trait objects and dynamic dispatch](01-trait-objects-and-dynamic-dispatch.md).
- You know traits ([Lesson 05.2](../05-rust-intermediate/02-traits.md)), generics ([Lesson 05.1](../05-rust-intermediate/01-generics.md)), closures ([Lesson 05.4](../05-rust-intermediate/04-closures.md)) and iterators ([Lesson 05.5](../05-rust-intermediate/05-iterators.md)).
- Create a practice project: `cargo new code/p06-advanced-traits`. Every example below can go into `src/main.rs` one at a time.

## New words

| Word | Meaning |
|---|---|
| **associated type** | A type name declared inside a trait (`type Item;`). Each implementation chooses the real type. |
| **default type parameter** | A generic parameter with a default value, written `<Rhs = Self>`. |
| **operator overloading** | Making an operator such as `+` work for your own type by implementing a trait from `std::ops`. |
| **fully qualified syntax** | The long form of a call, `<Type as Trait>::function()`, that says exactly which trait's function you want. |
| **supertrait** | A trait that another trait requires: in `trait A: B`, the trait `B` is a supertrait of `A`. |
| **newtype pattern** | A tuple struct with one field that wraps another type to give it a new meaning or new trait implementations. |
| **orphan rule** | You may implement a trait for a type only if the trait or the type is defined in your own crate. |
| **type alias** | A new name for an existing type, written `type Name = ...;`. It is not a new type. |
| **never type** (`!`) | The type of an expression that never produces a value, such as `panic!()` or `continue`. |
| **DST** (Dynamically Sized Type) | A type whose size is not known at compile time: `str`, `[T]` and `dyn Trait`. |
| **function pointer** | A value of type `fn(A) -> B` that points to a function. |
| **KiB / MiB** (kibibyte / mebibyte) | 1 KiB = 1024 bytes, 1 MiB = 1024 KiB. Used in Exercise 1. |
| **JSON** (JavaScript Object Notation) | A popular text format for data, such as `{"name":"ana"}`. Used in Exercise 2. |
| **XML** (Extensible Markup Language) | A text format for data that uses tags, such as `<user>ana</user>`. Used in Exercise 2. |

## 1. Associated types

In [Lesson 05.5](../05-rust-intermediate/05-iterators.md) you used `Iterator`, which contains the line `type Item;`. That is an **associated type**: a placeholder type inside a trait. The type that implements the trait fills it in.

Here is our own trait, a container that can give you its first element:

```rust
// A trait with an associated type: each implementor picks ONE Item.
trait Container {
    type Item;
    fn first(&self) -> Option<&Self::Item>;
    fn size(&self) -> usize;
}

struct Playlist {
    songs: Vec<String>,
}

struct Temperatures(Vec<f64>);

impl Container for Playlist {
    type Item = String;
    fn first(&self) -> Option<&String> {
        self.songs.first()
    }
    fn size(&self) -> usize {
        self.songs.len()
    }
}

impl Container for Temperatures {
    type Item = f64;
    fn first(&self) -> Option<&f64> {
        self.0.first()
    }
    fn size(&self) -> usize {
        self.0.len()
    }
}

// The caller does not need to write the item type.
fn is_empty<C: Container>(c: &C) -> bool {
    c.size() == 0
}

fn main() {
    let playlist = Playlist {
        songs: vec![String::from("Intro"), String::from("Finale")],
    };
    let temps = Temperatures(vec![]);

    println!("first song: {:?}", playlist.first());
    println!("playlist empty? {}", is_empty(&playlist));
    println!("first temperature: {:?}", temps.first());
    println!("temps empty? {}", is_empty(&temps));
}
```

```text
first song: Some("Intro")
playlist empty? false
first temperature: None
temps empty? true
```

**What happens:**

1. `type Item;` inside `trait Container` says: "every implementation will name one item type".
2. `Playlist` sets `type Item = String;` and `Temperatures` sets `type Item = f64;`. Inside the trait, `Self::Item` means "the item type of this implementation".
3. `is_empty<C: Container>` is generic over the container only. The caller never writes the item type.
4. `first()` returns `Option<&Self::Item>`: for the playlist it is `Option<&String>`, for the temperatures `Option<&f64>`.

**Associated type or generic parameter?** Compare it with a generic trait:

```rust
// A generic trait: one type can implement it many times, with different T.
trait Convert<T> {
    fn convert(&self) -> T;
}

struct Celsius(f64);

impl Convert<f64> for Celsius {
    fn convert(&self) -> f64 {
        self.0 * 1.8 + 32.0
    }
}

impl Convert<String> for Celsius {
    fn convert(&self) -> String {
        format!("{:.1} C", self.0)
    }
}

fn main() {
    let t = Celsius(21.5);
    let fahrenheit: f64 = t.convert();
    let text: String = t.convert();
    println!("{} / {}", fahrenheit, text);
}
```

```text
70.7 / 21.5 C
```

**What happens:**

1. `Convert<T>` has a type parameter. `Celsius` implements it twice: once for `f64`, once for `String`.
2. The compiler cannot guess which `convert` you mean, so we write the wanted type in each `let` (`let fahrenheit: f64 = ...`).

The rule of thumb:

| Use | When |
|---|---|
| **Associated type** | For each implementing type there is exactly **one** sensible choice (the `Item` of one iterator). Callers then never write the type. |
| **Generic parameter** | One type may implement the trait **several times** with different types (`From<A>` and `From<B>` for the same target). |

## 2. Default type parameters and operator overloading

Rust lets you give a generic parameter a default value. The most famous use is the trait `Add`, which powers the `+` operator:

```rust
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

`Rhs` (right-hand side) defaults to `Self`: "by default you add a value of the same type". `Output` is an associated type that tells what `a + b` produces. Implementing the traits from `std::ops` is called **operator overloading**:

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

// `Add` is `trait Add<Rhs = Self>`: Rhs defaults to the same type.
impl Add for Point {
    type Output = Point;
    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

#[derive(Debug)]
struct Millimeters(u32);
struct Meters(u32);

// Here we choose a different Rhs: millimeters plus meters.
impl Add<Meters> for Millimeters {
    type Output = Millimeters;
    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + other.0 * 1000)
    }
}

fn main() {
    let a = Point { x: 1, y: 2 };
    let b = Point { x: 10, y: 20 };
    println!("{:?}", a + b);

    let length = Millimeters(250) + Meters(2);
    println!("{:?}", length);
}
```

```text
Point { x: 11, y: 22 }
Millimeters(2250)
```

**What happens:**

1. `impl Add for Point` uses the default `Rhs = Self`, so `a + b` adds two `Point`s. `a + b` is the same as `a.add(b)`.
2. The `Point` is `Copy`, so using `a` and `b` in the addition does not move them away.
3. `impl Add<Meters> for Millimeters` picks a different right-hand side. Now millimeters plus meters is allowed and converts the meters to millimeters.
4. 250 + 2 * 1000 = 2250.

> 📏 **Convention:** Overload an operator only when its meaning is obvious (`+` for numbers, vectors, durations). Do not use `+` for something unrelated, such as "send a message".

## 3. Fully qualified syntax

A type can have a method with the same name as a method from a trait, and two traits can have the same method name. Rust must know which one you want:

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

impl Human {
    fn fly(&self) {
        println!("*waves arms*");
    }
}

trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Dog {
    fn baby_name() -> String {
        String::from("Spot")
    }
}

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn main() {
    let person = Human;
    person.fly(); // the method of Human itself
    Pilot::fly(&person); // short form: the trait name
    <Human as Wizard>::fly(&person); // full form: <Type as Trait>

    println!("{}", Dog::baby_name());
    println!("{}", <Dog as Animal>::baby_name());
}
```

```text
*waves arms*
This is your captain speaking.
Up!
Spot
puppy
```

**What happens:**

1. `Human` has its own `fly`, plus `fly` from `Pilot` and `fly` from `Wizard`.
2. `person.fly()` picks the method written directly on the type.
3. `Pilot::fly(&person)` names the trait, and passes the value as the first argument (`self`).
4. `<Human as Wizard>::fly(&person)` is the **fully qualified syntax**: `<Type as Trait>::function(...)`. It always works.
5. `Dog::baby_name()` picks the associated function written on `Dog`. Associated functions without `self` have no value to guide the choice, so for the trait version we must use `<Dog as Animal>::baby_name()`.

What if you try the short form for the trait function?

```rust
trait Animal {
    fn baby_name() -> String;
}

struct Dog;

impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}

fn main() {
    println!("{}", Animal::baby_name());
}
```

```text
error[E0790]: cannot call associated function on trait without specifying the corresponding `impl` type
  --> src/main.rs:14:20
   |
 2 |     fn baby_name() -> String;
   |     ------------------------- `Animal::baby_name` defined here
...
14 |     println!("{}", Animal::baby_name());
   |                    ^^^^^^^^^^^^^^^^^^^ cannot call associated function of trait
   |
help: use the fully-qualified path to the only available implementation
   |
14 |     println!("{}", <Dog as Animal>::baby_name());
   |                    +++++++       +
```

The compiler suggests the fully qualified path. `Animal::baby_name()` has no `self`, so Rust cannot know which type's version you mean.

## 4. Supertraits

Sometimes a trait needs another trait to do its job. A **supertrait** says: "to implement me, a type must also implement that trait". A default method of the trait can then use the supertrait's methods:

```rust
use std::fmt::Display;

// `OutlinePrint: Display` means: every OutlinePrint type must also be Display.
trait OutlinePrint: Display {
    fn outline(&self) {
        let text = self.to_string(); // allowed because of the supertrait
        let line = "*".repeat(text.len() + 4);
        println!("{}", line);
        println!("* {} *", text);
        println!("{}", line);
    }
}

struct Point {
    x: i32,
    y: i32,
}

impl Display for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

impl OutlinePrint for Point {}

fn main() {
    Point { x: 3, y: -7 }.outline();
}
```

```text
***********
* (3, -7) *
***********
```

**What happens:**

1. `trait OutlinePrint: Display` means that every type with `OutlinePrint` must also have `Display`.
2. Because of that, the default method `outline` can call `self.to_string()` (a method that every `Display` type has).
3. `impl OutlinePrint for Point {}` is empty: it only uses the default method. It compiles only because `Point` implements `Display` above it.
4. The text `(3, -7)` has 7 characters, so the line of stars has 7 + 4 = 11 characters: the text plus `* ` on the left and ` *` on the right.

> 💡 **Tip:** You met a supertrait in [Lesson 06.1](01-trait-objects-and-dynamic-dispatch.md): `trait Plugin: Debug`. A supertrait is often used for `Debug`, `Display`, `Send` or `Clone`.

## 5. The newtype pattern and the orphan rule

You can implement a trait for a type only if **your crate defines the trait or the type**. This is the **orphan rule**. It protects you: if two crates could both implement `Display` for `Vec<String>`, the compiler could not choose, and your program would break when a dependency changed.

```rust
use std::fmt;

impl fmt::Display for Vec<String> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.join(", "))
    }
}

fn main() {
    let v = vec![String::from("a"), String::from("b")];
    println!("{}", v);
}
```

```text
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
 --> src/main.rs:3:1
  |
3 | impl fmt::Display for Vec<String> {
  | ^^^^^^^^^^^^^^^^^^^^^^-----------
  |                       |
  |                       `Vec` is not defined in the current crate
  |
  = note: impl doesn't have any local type before any uncovered type parameters
  = note: for more information see https://doc.rust-lang.org/reference/items/implementations.html#orphan-rules
  = note: define and implement a trait or new type instead

For more information about this error, try `rustc --explain E0117`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

The last help line gives the standard fix: the **newtype pattern**. You wrap the foreign type in your own tuple struct. The wrapper is your own type, so you may implement any trait for it:

```rust
use std::fmt;
use std::ops::Deref;

// A newtype: a tuple struct with exactly one field.
struct Names(Vec<String>);

impl fmt::Display for Names {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

// Deref lets us call Vec methods (len, iter, ...) directly on Names.
impl Deref for Names {
    type Target = Vec<String>;
    fn deref(&self) -> &Vec<String> {
        &self.0
    }
}

// Newtypes also stop you from mixing up values of the same inner type.
struct UserId(u64);
struct OrderId(u64);

fn load_user(id: UserId) -> String {
    format!("user #{}", id.0)
}

fn load_order(id: OrderId) -> String {
    format!("order #{}", id.0)
}

fn main() {
    let names = Names(vec![String::from("Ana"), String::from("Bo")]);
    println!("{}", names);
    println!("{} names", names.len());

    println!("{}", load_user(UserId(7)));
    println!("{}", load_order(OrderId(7)));
    // load_user(OrderId(7)); // would not compile: expected `UserId`, found `OrderId`
}
```

```text
[Ana, Bo]
2 names
user #7
order #7
```

**What happens:**

1. `struct Names(Vec<String>);` is a newtype: a struct with one field. It costs nothing at run time; the wrapper has the same layout as the inner `Vec`.
2. `impl fmt::Display for Names` is allowed, because `Names` is defined here. `self.0` is the inner vector.
3. A wrapper hides the methods of the inner type. `impl Deref for Names` brings them back: `names.len()` works because `Names` can be dereferenced to a `Vec<String>`.
4. `UserId(u64)` and `OrderId(u64)` both wrap a `u64`, but they are different types. Passing an `OrderId` to `load_user` is a compile error. Newtypes turn mix-ups into compile errors for free.

> 📏 **Convention:** Use newtypes for identifiers and units (`UserId`, `Meters`). A function `fn charge(user: u64, amount: u64)` is easy to call with the arguments swapped; `fn charge(user: UserId, amount: Cents)` is not.

## 6. Type aliases

A **type alias** gives an existing type another name. It does not create a new type, so no conversion is needed:

```rust
use std::fmt;

#[derive(Debug)]
struct AppError(String);

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "app error: {}", self.0)
    }
}

// An alias gives a new NAME to a type. It is not a new type.
type Result<T> = std::result::Result<T, AppError>;
type Callback = Box<dyn Fn(i32) -> i32 + Send + 'static>;
type Meters = u32;

fn parse_age(text: &str) -> Result<u8> {
    text.trim()
        .parse::<u8>()
        .map_err(|e| AppError(format!("bad age {:?}: {}", text, e)))
}

fn run(callbacks: &[Callback], start: i32) -> i32 {
    callbacks.iter().fold(start, |acc, cb| cb(acc))
}

fn main() {
    println!("{:?}", parse_age(" 41 "));
    match parse_age("abc") {
        Ok(age) => println!("age {}", age),
        Err(e) => println!("{}", e),
    }

    let callbacks: Vec<Callback> = vec![Box::new(|x| x + 1), Box::new(|x| x * 2)];
    println!("{}", run(&callbacks, 5));

    let distance: Meters = 100;
    let other: u32 = distance; // same type, no conversion needed
    println!("{}", distance + other);
}
```

```text
Ok(41)
app error: bad age "abc": invalid digit found in string
12
200
```

**What happens:**

1. `type Result<T> = std::result::Result<T, AppError>;` fixes the error type. All functions in the program can write `Result<u8>` instead of `Result<u8, AppError>`. The standard library does the same with `std::io::Result<T>`.
2. `type Callback = Box<dyn Fn(i32) -> i32 + Send + 'static>;` shortens a long type that is used in several places (Clippy warns about "very complex types").
3. `type Meters = u32;` is only another name. `distance` can be assigned to a `u32` variable directly. This is why an alias adds **no** safety; a newtype does.

## 7. The never type `!`

Some expressions never finish with a value: `panic!()`, `continue`, `break`, `return`, `std::process::exit()`, an endless `loop {}`. Their type is `!`, called the **never type** (also "empty type"). Because nothing can ever exist of this type, the compiler can turn `!` into any other type. That is why `continue` can sit in a `match` arm where a `u32` is expected:

```rust
fn main() {
    let inputs = ["10", "x", "30", "", "50"];
    let mut total: u32 = 0;

    for text in inputs {
        // `continue` has type `!`, so it fits where a u32 is expected.
        let number: u32 = match text.parse() {
            Ok(n) => n,
            Err(_) => continue,
        };
        total += number;
    }
    println!("total = {}", total);

    let value: u32 = match "7".parse::<u32>() {
        Ok(n) => n,
        Err(_) => panic!("not a number"), // panic! also has type `!`
    };
    println!("value = {}", value);
}
```

```text
total = 90
value = 7
```

**What happens:**

1. `text.parse()` tries to turn each text into a `u32` ([Lesson 02.7](../02-programming-basics/07-reading-input-and-parsing.md)). `"x"` and `""` fail.
2. In the `Ok(n)` arm the value is a `u32`. In the `Err(_)` arm `continue` has type `!`, which fits as a `u32` because it never produces a value: the loop jumps to the next item.
3. So only `10 + 30 + 50 = 90` is added.
4. `panic!` has type `!` too, so `Err(_) => panic!(...)` compiles. Functions such as `Option::unwrap` rely on the same idea: the failing branch panics and never returns a value.

You can write a function that never returns, with `-> !`:

```rust
fn fail(message: &str) -> ! {
    println!("fatal: {}", message);
    std::process::exit(3)
}

fn main() {
    let value = 5;
    let x: i32 = if value > 0 {
        10
    } else {
        fail("math is broken")
    };
    println!("x = {}", x);
    fail("stopping here");
}
```

```text
x = 10
fatal: stopping here
```

**What happens:**

1. `fail` prints a message and calls `std::process::exit(3)`, which ends the program with exit code 3. Its type is `!`, so `fail` can be declared `-> !`.
2. In `let x: i32 = if ... { 10 } else { fail(...) }`, the `else` branch has type `!` and fits where an `i32` is needed.
3. `value` is 5, so the condition `value > 0` is true and the first line prints `x = 10`.
4. Then `fail("stopping here")` ends the program. In the terminal ([Lesson 00.4](../00-getting-started/04-terminal-basics.md)) the command `echo $?` prints the exit code of the last program; run it after `cargo run` or after the compiled binary and you will see `3`.

## 8. Types without a known size (DST) and `?Sized`

Most types have a size known at compile time (`i32` is 4 bytes). A few do not. They are called **DST**s (Dynamically Sized Types):

- `str` (text of any length). You use it as `&str` or `Box<str>`.
- `[T]` (a slice of any length). You use it as `&[T]`.
- `dyn Trait` (any type that implements a trait, [Lesson 06.1](01-trait-objects-and-dynamic-dispatch.md)).

A DST always lives **behind a pointer**, and the pointer is a fat pointer that also carries the missing information (a length or a vtable). The compiler needs sizes to place local variables on the stack, so it refuses to create a DST by value:

```text
error[E0277]: the size for values of type `str` cannot be known at compilation time
 --> src/main.rs:2:9
  |
2 |     let text: str = *"hello";
  |         ^^^^ doesn't have a size known at compile-time
  |
  = help: the trait `Sized` is not implemented for `str`
  = note: all local variables must have a statically known size
help: consider borrowing here
  |
2 |     let text: &str = *"hello";
  |               +

For more information about this error, try `rustc --explain E0277`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

There is a marker trait, `Sized`, that means "the size is known at compile time". In every generic function, `T` has an automatic `T: Sized` bound. To accept DSTs you must opt out with `?Sized` ("maybe sized"):

```rust
use std::fmt::Display;

// `?Sized` means "T may be a type without a known size".
fn show<T: Display + ?Sized>(value: &T) {
    println!("show: {}", value);
}

fn main() {
    show(&42); // T = i32, which is Sized
    show("hello"); // T = str, which is NOT Sized: only works because of ?Sized

    let boxed: Box<str> = "boxed text".into(); // an owned str on the heap
    println!("{} ({} bytes)", boxed, boxed.len());
}
```

```text
show: 42
show: hello
boxed text (10 bytes)
```

**What happens:**

1. `show<T: Display + ?Sized>(value: &T)` accepts a reference to any `T`, sized or not. The reference itself has a known size, so this is fine.
2. `show(&42)` uses `T = i32`. `show("hello")` uses `T = str`, which is only possible because of `?Sized`.
3. `Box<str>` is an owned text on the heap without extra capacity. `"boxed text".into()` builds it from a `&str`. It is 10 bytes long.

## 9. Function pointers, closures and returning closures

In [Lesson 05.4](../05-rust-intermediate/04-closures.md) you passed closures to functions with `F: Fn(...)`. You can also pass a normal function. Rust has a type for "a pointer to a function": `fn(i32) -> i32` (with a lowercase `fn`; do not mix it up with the `Fn` trait).

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

// `fn(i32) -> i32` is a function POINTER type (lowercase fn).
fn apply_twice(f: fn(i32) -> i32, x: i32) -> i32 {
    f(f(x))
}

// `F: Fn(i32) -> i32` accepts functions AND closures that capture variables.
fn apply_generic<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n
}

fn make_op(kind: &str) -> Box<dyn Fn(i32) -> i32> {
    if kind == "double" {
        Box::new(|x| x * 2)
    } else {
        Box::new(|x| x - 1)
    }
}

fn main() {
    println!("{}", apply_twice(add_one, 5));
    println!("{}", apply_twice(|x| x * 3, 5)); // a closure without captures also works

    let offset = 100;
    println!("{}", apply_generic(|x| x + offset, 5)); // captures `offset`

    let words: Vec<String> = [1, 2, 3].iter().map(ToString::to_string).collect();
    println!("{:?}", words);

    // An enum variant with data is a function too: Some(1), Some(2).
    let options: Vec<Option<i32>> = [1, 2].into_iter().map(Some).collect();
    println!("{:?}", options);

    let add5 = make_adder(5);
    println!("{}", add5(10));
    println!("{} {}", make_op("double")(8), make_op("other")(8));
}
```

```text
7
45
105
["1", "2", "3"]
[Some(1), Some(2)]
15
16 7
```

**What happens:**

1. `apply_twice(f: fn(i32) -> i32, x)` accepts a plain function. `apply_twice(add_one, 5)` is `add_one(add_one(5))` = 7.
2. A closure that **captures nothing** (`|x| x * 3`) can be converted to a function pointer, so it is accepted too: (5 * 3) * 3 = 45.
3. `apply_generic<F: Fn(i32) -> i32>` accepts any closure, also one that captures `offset`.
4. `ToString::to_string` is a function, so it can be passed to `map` directly. Likewise `Some` is a function from `T` to `Option<T>`, so `.map(Some)` works. The same holds for tuple-struct and enum constructors.
5. `make_adder` returns `impl Fn(i32) -> i32`: one specific closure type, hidden. `move` copies `n` into the closure.
6. `make_op` can return different closures from different branches, so each closure is boxed and the type is `Box<dyn Fn(i32) -> i32>` ([Lesson 06.1](01-trait-objects-and-dynamic-dispatch.md)). `impl Fn` would not compile here.

What if the closure captures a variable and the parameter is a function pointer?

```text
error[E0308]: mismatched types
 --> src/main.rs:7:26
  |
7 |     println!("{}", apply(|x| x + offset, 5));
  |                    ----- ^^^^^^^^^^^^^^ expected fn pointer, found closure
  |                    |
  |                    arguments to this function are incorrect
  |
  = note: expected fn pointer `fn(i32) -> i32`
                found closure `{closure@src/main.rs:7:26: 7:29}`
note: closures can only be coerced to `fn` types if they do not capture any variables
 --> src/main.rs:7:34
  |
7 |     println!("{}", apply(|x| x + offset, 5));
  |                                  ^^^^^^ `offset` captured here
note: function defined here
 --> src/main.rs:1:4
  |
1 | fn apply(f: fn(i32) -> i32, x: i32) -> i32 {
  |    ^^^^^ -----------------

For more information about this error, try `rustc --explain E0308`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

| Take as a parameter | Accepts | Use when |
|---|---|---|
| `fn(i32) -> i32` | Functions and non-capturing closures | You talk to C code (Lesson 06.10) or want a simple, fixed-size value. |
| `F: Fn(i32) -> i32` | Anything callable, with or without captures | Almost always. This is the default. |
| `Box<dyn Fn(i32) -> i32>` | Any callable, stored with a hidden type | A list of mixed callables, or a field in a struct. |

## Common mistakes

### 1. Calling a trait's associated function without the type

`Animal::baby_name()` gives `E0790`. Write `<Dog as Animal>::baby_name()`.

### 2. Implementing a foreign trait for a foreign type

`impl Display for Vec<String>` gives `E0117` (orphan rule). Wrap the type in a newtype.

### 3. Expecting a type alias to protect you

`type UserId = u64; type OrderId = u64;` are the same type. Passing an `OrderId` where a `UserId` is expected compiles. Use a newtype struct instead.

### 4. Forgetting `?Sized`

`fn show<T: Display>(v: &T)` rejects `show("hello")` with "the size for values of type `str` cannot be known at compilation time". Add `+ ?Sized`.

### 5. Passing a capturing closure as `fn(...)`

`E0308 ... expected fn pointer, found closure`, see section 9. Change the parameter to `impl Fn(...)`.

## Try it yourself

1. Open `~/projects/rust/code/p06-advanced-traits`.
2. Copy the `Point` and `Add` example (section 2) into `src/main.rs` and run `cargo run`.
3. Add `impl Sub for Point` (use the `Sub` trait from `std::ops`, method `sub`). Print `b - a`. You should get `Point { x: 9, y: 18 }`.
4. Add `impl std::ops::Neg for Point` (the unary minus: `-a`). `Neg` has only the associated type `Output` and the method `neg(self)`. Print `-a`; expect `Point { x: -1, y: -2 }`.
5. Run `cargo fmt` and `cargo clippy`.
6. Commit: `git add code/p06-advanced-traits` then `git commit -m "feat(p06): operator overloading for Point"`.

## Exercises

### Exercise 1 — Bytes with units (easy)

Create `struct Bytes(u64)`. Implement `Add` and `Sub`, and `Display` so that `Bytes(1536)` prints `1.5 KiB`, `Bytes(1024 * 1024)` prints `1.0 MiB` and `Bytes(512)` prints `512 B`. Write tests for the addition and for the three outputs.

<details>
<summary>Hint</summary>

Use `if self.0 >= 1024 * 1024 { ... } else if self.0 >= 1024 { ... } else { ... }` inside `fmt`, and `{:.1}` to print one decimal place. Convert with `self.0 as f64`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fmt;
use std::ops::{Add, Sub};

#[derive(Debug, Clone, Copy, PartialEq)]
struct Bytes(u64);

impl Add for Bytes {
    type Output = Bytes;
    fn add(self, other: Bytes) -> Bytes {
        Bytes(self.0 + other.0)
    }
}

impl Sub for Bytes {
    type Output = Bytes;
    fn sub(self, other: Bytes) -> Bytes {
        Bytes(self.0 - other.0)
    }
}

impl fmt::Display for Bytes {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        if self.0 >= 1024 * 1024 {
            write!(f, "{:.1} MiB", self.0 as f64 / (1024.0 * 1024.0))
        } else if self.0 >= 1024 {
            write!(f, "{:.1} KiB", self.0 as f64 / 1024.0)
        } else {
            write!(f, "{} B", self.0)
        }
    }
}

fn main() {
    let total = Bytes(1000) + Bytes(536);
    println!("{}", total);
    println!("{}", Bytes(3 * 1024 * 1024 / 2));
    println!("{}", Bytes(900) - Bytes(100));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_sizes() {
        assert_eq!(Bytes(1) + Bytes(2), Bytes(3));
    }

    #[test]
    fn prints_human_units() {
        assert_eq!(Bytes(512).to_string(), "512 B");
        assert_eq!(Bytes(1536).to_string(), "1.5 KiB");
        assert_eq!(Bytes(1024 * 1024).to_string(), "1.0 MiB");
    }
}
```

```text
1.5 KiB
1.5 MiB
800 B
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the traits `Add` and `Sub` give meaning to `+` and `-`. `Display` gives `to_string()` for free, which the tests use.

</details>

### Exercise 2 — Two traits, one method name (medium)

Two traits `Json` and `Xml` both have `fn render(&self) -> String`. `User` implements both. Print the JSON line `{"name":"ana"}` and the XML line `<user>ana</user>` for the same user.

<details>
<summary>Hint</summary>

`user.render()` is ambiguous. Use `<User as Json>::render(&user)` or `Xml::render(&user)`.

</details>

<details>
<summary>Answer</summary>

```rust
trait Json {
    fn render(&self) -> String;
}

trait Xml {
    fn render(&self) -> String;
}

struct User {
    name: String,
}

impl Json for User {
    fn render(&self) -> String {
        format!("{{\"name\":\"{}\"}}", self.name)
    }
}

impl Xml for User {
    fn render(&self) -> String {
        format!("<user>{}</user>", self.name)
    }
}

fn main() {
    let user = User {
        name: String::from("ana"),
    };
    println!("{}", <User as Json>::render(&user));
    println!("{}", Xml::render(&user));
}
```

```text
{"name":"ana"}
<user>ana</user>
```

Why it works: the path names the trait, so the compiler knows which `render` to call. When the method has a `self` argument, `Trait::method(&value)` is enough, because the type of `value` selects the implementation.

</details>

### Exercise 3 — Temperature newtypes (medium)

Create `Celsius(f64)` and `Fahrenheit(f64)`. Implement `From<Celsius> for Fahrenheit` and `From<Fahrenheit> for Celsius`. Write a test that 100 degrees Celsius is 212 degrees Fahrenheit, and a test that converting there and back gives the original value.

<details>
<summary>Hint</summary>

F = C * 9 / 5 + 32 and C = (F - 32) * 5 / 9. Compare floating-point numbers with a small tolerance: `(a - b).abs() < 1e-9`, never with `==`.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, Clone, Copy)]
struct Celsius(f64);

#[derive(Debug, Clone, Copy)]
struct Fahrenheit(f64);

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Fahrenheit {
        Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
    }
}

impl From<Fahrenheit> for Celsius {
    fn from(f: Fahrenheit) -> Celsius {
        Celsius((f.0 - 32.0) * 5.0 / 9.0)
    }
}

fn main() {
    let boiling: Fahrenheit = Celsius(100.0).into();
    println!("{:?}", boiling);
    let freezing = Celsius::from(Fahrenheit(32.0));
    println!("{:?}", freezing);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn celsius_to_fahrenheit() {
        let f: Fahrenheit = Celsius(100.0).into();
        assert!((f.0 - 212.0).abs() < 1e-9);
    }

    #[test]
    fn round_trip() {
        let c = Celsius::from(Fahrenheit::from(Celsius(36.6)));
        assert!((c.0 - 36.6).abs() < 1e-9);
    }
}
```

```text
Fahrenheit(212.0)
Celsius(0.0)
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `From` gives you `.into()` automatically. The two newtypes cannot be mixed up by accident, although both hold an `f64`.

</details>

### Exercise 4 — A trait with an associated type (hard)

Define `trait Source { type Item; fn next_item(&mut self) -> Option<Self::Item>; }`. Implement it for `Countdown(u32)` (items `u32`, counting down to 1) and for `Letters` (items `char`). Write a generic function `collect_all<S: Source>(source: S) -> Vec<S::Item>`. Test it on `Countdown(3)` and `Countdown(0)`.

<details>
<summary>Hint</summary>

The return type uses the associated type of the generic parameter: `Vec<S::Item>`. Loop with `while let Some(item) = source.next_item()`. The parameter must be `mut source`.

</details>

<details>
<summary>Answer</summary>

```rust
trait Source {
    type Item;
    fn next_item(&mut self) -> Option<Self::Item>;
}

struct Countdown(u32);

impl Source for Countdown {
    type Item = u32;
    fn next_item(&mut self) -> Option<u32> {
        if self.0 == 0 {
            None
        } else {
            self.0 -= 1;
            Some(self.0 + 1)
        }
    }
}

struct Letters {
    rest: Vec<char>,
}

impl Source for Letters {
    type Item = char;
    fn next_item(&mut self) -> Option<char> {
        if self.rest.is_empty() {
            None
        } else {
            Some(self.rest.remove(0))
        }
    }
}

// `S::Item` is the associated type of whatever source we are given.
fn collect_all<S: Source>(mut source: S) -> Vec<S::Item> {
    let mut items = Vec::new();
    while let Some(item) = source.next_item() {
        items.push(item);
    }
    items
}

fn main() {
    println!("{:?}", collect_all(Countdown(3)));
    println!(
        "{:?}",
        collect_all(Letters {
            rest: "hey".chars().collect()
        })
    );
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn countdown_collects_in_order() {
        assert_eq!(collect_all(Countdown(3)), vec![3, 2, 1]);
    }

    #[test]
    fn empty_source_gives_empty_vec() {
        assert!(collect_all(Countdown(0)).is_empty());
    }
}
```

```text
[3, 2, 1]
['h', 'e', 'y']
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: each implementation chooses its own `Item`, and `S::Item` in the function signature follows that choice. This is the same design as `Iterator`.

</details>

## Quiz

1. When do you prefer an associated type to a generic parameter on a trait?

<details>
<summary>Answer</summary>

When each implementing type has exactly one natural choice for that type (like the `Item` of an iterator). Then callers never write the type. Use a generic parameter when the same type should implement the trait several times with different type arguments (like `From<A>` and `From<B>`).

</details>

2. What is the default for `Rhs` in `trait Add<Rhs = Self>`, and what does it mean?

<details>
<summary>Answer</summary>

`Self`: unless you say otherwise, you add a value of the same type, so `impl Add for Point` adds two points.

</details>

3. What is the fully qualified syntax, and when is it needed?

<details>
<summary>Answer</summary>

`<Type as Trait>::function(...)`. It is needed when the short form is ambiguous: two traits with the same method name, or an associated function without `self`.

</details>

4. What does the orphan rule forbid, and how do you work around it?

<details>
<summary>Answer</summary>

It forbids implementing a trait for a type when both the trait and the type come from other crates. Work around it with the newtype pattern: wrap the type in your own tuple struct and implement the trait for the wrapper.

</details>

5. Why does a type alias not protect you from mixing up `UserId` and `OrderId`?

<details>
<summary>Answer</summary>

An alias is only another name for the same type. `type UserId = u64;` and `type OrderId = u64;` are both `u64`, so the compiler treats them as equal. A newtype struct creates a new, different type.

</details>

6. What is the type of `continue`, and why can it appear in a `match` arm that must return a `u32`?

<details>
<summary>Answer</summary>

Its type is `!` (never). It never produces a value, so the compiler lets it be converted to any type, including `u32`.

</details>

7. Why can you not write `let s: str = ...`, and what do you write instead?

<details>
<summary>Answer</summary>

`str` is a dynamically sized type: its size is not known at compile time, so it cannot be a local variable. Use it behind a pointer: `&str`, `Box<str>`, or `String` for an owned, growable text.

</details>

8. What is the difference between `fn(i32) -> i32` and `F: Fn(i32) -> i32`?

<details>
<summary>Answer</summary>

`fn(i32) -> i32` is a function pointer type: it accepts plain functions and closures that capture nothing. `Fn(i32) -> i32` is a trait: it accepts any callable, including closures that capture variables. Prefer the trait for parameters.

</details>

## Summary

- An associated type (`type Item;`) lets each implementation choose one type; callers never write it. A generic trait parameter allows many implementations for one type.
- Default type parameters (`Add<Rhs = Self>`) make common cases short. Operator overloading is implementing traits from `std::ops`.
- `<Type as Trait>::function()` removes ambiguity between same-named methods and calls trait functions without `self`.
- A supertrait (`trait A: B`) requires `B` and lets `A`'s default methods use it.
- The orphan rule says you need to own the trait or the type. The newtype pattern is the workaround and also prevents mix-ups of values.
- A type alias is only a new name; a newtype is a new type.
- `!` is the type of expressions that never finish; `str`, `[T]` and `dyn Trait` are DSTs that live behind pointers; `?Sized` accepts them in generics.
- `fn(...)` is a function pointer type; `impl Fn` and `Box<dyn Fn>` return closures.

## Sources and further reading

- [The Rust Programming Language — 20.2 Advanced Traits](https://doc.rust-lang.org/book/ch20-02-advanced-traits.html) — associated types, default parameters, fully qualified syntax, supertraits and the newtype pattern.
- [The Rust Programming Language — 20.3 Advanced Types](https://doc.rust-lang.org/book/ch20-03-advanced-types.html) — newtypes, type aliases, the never type and dynamically sized types.
- [The Rust Programming Language — 20.4 Advanced Functions and Closures](https://doc.rust-lang.org/book/ch20-04-advanced-functions-and-closures.html) — function pointers and returning closures.
- [std::ops::Add](https://doc.rust-lang.org/std/ops/trait.Add.html) — the `Add` trait with examples; the module lists all other operator traits.
- [Rust by Example — New Type Idiom](https://doc.rust-lang.org/rust-by-example/generics/new_types.html) — a short newtype example.
- [The Rust Reference — Special types and traits](https://doc.rust-lang.org/reference/special-types-and-traits.html) — what `Sized` and `Copy` mean exactly.

---

[🏠 Start](../START-HERE.md) · [← 06.1 Trait objects and dynamic dispatch](01-trait-objects-and-dynamic-dispatch.md) · [📋 Phase 06](README.md) · [06.3 Patterns in depth →](03-patterns-in-depth.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
