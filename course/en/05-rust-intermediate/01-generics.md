<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 05 — Rust intermediate](README.md) · [📋 Phase 05](README.md) · [05.2 Traits →](02-traits.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/01-generics.md) <!-- nav -->

# Lesson 05.1 — Generics

> **Phase 05 — Rust intermediate** · Lesson 1 of 14 · ⏱ about 4 hours (2 sittings)

## What you will learn

- Why copy-pasting the same function for every type is a problem, and how generics solve it.
- How to write generic functions, structs, enums and methods with type parameters such as `T`.
- What a *trait bound* is, and why `T: PartialOrd` is needed to compare two values of type `T`.
- Why generic code is not slower than hand-written code (monomorphization), and what it costs instead.
- How to write the *turbofish* `::<>` and how to read the most common generic compiler errors.
- How to start the practice project of this phase and save each lesson in Git on its own branch.

## Before you start

- You finished [Lesson 04.12 — Checkpoint](../04-structs-enums-collections-errors/12-checkpoint.md).
- You know structs ([Lesson 04.1](../04-structs-enums-collections-errors/01-structs.md)), methods ([Lesson 04.2](../04-structs-enums-collections-errors/02-methods-and-associated-functions.md)), `Option` ([Lesson 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)) and vectors ([Lesson 04.7](../04-structs-enums-collections-errors/07-vectors.md)).
- You have a Git repository for your learning work (from [Lesson 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md)) and your course folder is `~/projects/rust`.

## New words

| Word | Meaning |
|---|---|
| **generic** | Code written once that works for many different types. The type is a blank that the compiler fills in later. |
| **type parameter** | The name of such a blank, written in angle brackets: the `T` in `fn largest<T>(…)`. |
| **trait** | A list of abilities (methods) that a type can have, such as "can be compared" or "can be printed". Lesson 05.2 teaches traits in full; today you only need the idea. |
| **trait bound** | A rule that says which abilities a type parameter must have: `T: PartialOrd` means "`T` can be compared with `<` and `>`". |
| **`PartialOrd`** | The standard trait for types that can be ordered with `<`, `>`, `<=` and `>=`. |
| **monomorphization** | The step where the compiler makes a separate, specialised copy of generic code for every concrete type that is used. |
| **concrete type** | A real type such as `i32` or `String`, as opposed to a blank like `T`. |
| **turbofish** | The syntax `::<Type>` that tells the compiler which type to use for a type parameter: `"5".parse::<i32>()`. |
| **const generic** | A generic parameter that is a *value* (a number) instead of a type, for example the length `N` of an array `[i32; N]`. |
| **Cargo example** | A small extra program in the `examples/` folder of a package. You run it with `cargo run --example name`. |
| **branch** | A separate line of work in Git (Lesson 01.6). |

## 1. The problem: the same code for every type

Suppose you want a function that returns the largest number in a list. Here it is for `i32`
numbers, and again for `f64` numbers:

```rust
fn largest_i32(list: &[i32]) -> &i32 {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn largest_f64(list: &[f64]) -> &f64 {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let numbers = [34, 50, 25, 100, 65];
    let temperatures = [21.5, 19.0, 23.8];
    println!("largest number: {}", largest_i32(&numbers));
    println!("largest temperature: {}", largest_f64(&temperatures));
}
```

**What happens:**

1. `&[i32]` is a *slice* (Lesson 03.6): a borrowed view of a list of `i32` values.
2. `largest` starts as a reference to the first item. The `for` loop compares every item with it
   and moves the reference when it finds a bigger one.
3. `largest_f64` is the **same code**. Only the type changed.

```text
largest number: 100
largest temperature: 23.8
```

Two functions with identical bodies are a problem. If you find a bug, you must fix it twice. If
you also need `char`, `u8`, `String`… you copy again. The engineering principle **DRY** (Don't
Repeat Yourself) says: one idea should live in one place. Generics are how Rust follows DRY for
types.

## 2. Generic functions

A **generic** function has a *type parameter*, a name for "some type, to be decided later". You
write it in angle brackets after the function name:

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let numbers = [34, 50, 25, 100, 65];
    let temperatures = [21.5, 19.0, 23.8];
    let letters = ['q', 'z', 'a'];
    let words = ["pear", "apple", "zebra"];
    println!("{}", largest(&numbers));
    println!("{}", largest(&temperatures));
    println!("{}", largest(&letters));
    println!("{}", largest(&words));
}
```

**What happens:**

1. `<T: PartialOrd>` declares a type parameter named `T`. `T` is just a name; `T` for "type" is the
   usual choice.
2. `: PartialOrd` is a **trait bound**. It says: "`T` may be any type, *as long as* values of that
   type can be compared with `<` and `>`". Integers, floats, `char` and `&str` all can.
3. `list: &[T]` is a slice of `T` values and `-> &T` returns a reference to one of them.
4. The body is the same code as before. Inside it, the compiler knows only what the bound allows,
   which is exactly why the `>` comparison is accepted.
5. In `main` we call `largest` with four different types. Rust works out `T` by itself from the
   argument (this is called *type inference*).

```text
100
23.8
z
zebra
```

> 💡 **Tip:** Read `fn largest<T: PartialOrd>(list: &[T]) -> &T` as a sentence: "for any type `T`
> that can be ordered, take a slice of `T` and give back a reference to a `T`".

### Why is the bound needed?

Try to remove `: PartialOrd`:

```rust
fn largest<T>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

```text
error[E0369]: binary operation `>` cannot be applied to type `&T`
 --> examples/nobound.rs:4:17
  |
4 |         if item > largest {
  |            ---- ^ ------- &T
  |            |
  |            &T
  |
help: consider restricting type parameter `T` with trait `PartialOrd`
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {
  |             ++++++++++++++++++++++
```

**What it means:** `T` could be *any* type, including a type that cannot be compared (for example
a struct with no ordering). The compiler refuses to guess. It checks the generic function once, for
all possible types, so it only allows what the bounds promise. The `help:` line tells you the fix:
add the bound.

This is an important property of Rust: it checks a generic function when you
**write** it, not when you call it. If it compiles, it is correct for every type that satisfies
the bounds.

## 3. Generic structs

A struct can have type parameters too. A `Point` that works for integers *and* floats:

```rust
#[derive(Debug)]
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x * self.x + self.y * self.y).sqrt()
    }
}

fn main() {
    let a = Point { x: 3, y: 4 };
    let b = Point { x: 3.0, y: 4.0 };
    let c = Point { x: "left", y: "right" };
    println!("{:?} {:?} {:?}", a, b, c);
    println!("a.x = {}", a.x());
    println!("distance of b = {}", b.distance_from_origin());
}
```

**What happens:**

1. `#[derive(Debug)]` (Lesson 04.1) lets us print the struct with `{:?}`.
2. `struct Point<T>` declares one type parameter. Both fields use it, so `x` and `y` must have the
   **same** type.
3. `impl<T> Point<T>` means: "for every type `T`, here are the methods of `Point<T>`". The `<T>`
   after `impl` is required. It *declares* `T`; the `Point<T>` after it *uses* it.
4. `impl Point<f64>` has no `<T>`: these methods exist **only** for `Point<f64>`. A `Point<i32>`
   has no `distance_from_origin` method.
5. `a` is a `Point<i32>`, `b` is a `Point<f64>`, `c` is a `Point<&str>`. The compiler fills in `T`
   for each one.

```text
Point { x: 3, y: 4 } Point { x: 3.0, y: 4.0 } Point { x: "left", y: "right" }
a.x = 3
distance of b = 5
```

### More than one type parameter

If `x` and `y` may have different types, use two parameters:

```rust
#[derive(Debug)]
struct Point<X, Y> {
    x: X,
    y: Y,
}

impl<X, Y> Point<X, Y> {
    fn swap(self) -> Point<Y, X> {
        Point { x: self.y, y: self.x }
    }
}

fn main() {
    let p = Point { x: 5, y: "five" };
    println!("{:?}", p);
    let q = p.swap();
    println!("{:?}", q);
}
```

**What happens:** `p` is a `Point<i32, &str>`. `swap` consumes `p` (`self`, not `&self`, see
Lesson 04.2) and builds a `Point<&str, i32>`: the types swap places together with the values.

```text
Point { x: 5, y: "five" }
Point { x: "five", y: 5 }
```

## 4. Generic enums and the standard library

You have used generic types since Phase 04 without the name for them:

| Type | Definition (simplified) | Meaning |
|---|---|---|
| `Option<T>` | `enum Option<T> { Some(T), None }` | maybe a `T`, maybe nothing |
| `Result<T, E>` | `enum Result<T, E> { Ok(T), Err(E) }` | a `T` on success, an `E` on failure |
| `Vec<T>` | a growable list | a list of `T` |

`Vec<i32>` and `Vec<String>` are two different types made from the same generic definition.

Now we build our own generic container: a **stack**. A stack is a pile where you add (`push`) and
remove (`pop`) only at the top, like plates. It is built on a `Vec<T>`:

```rust
#[derive(Debug)]
struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { items: Vec::new() }
    }

    fn push(&mut self, item: T) {
        self.items.push(item);
    }

    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    fn peek(&self) -> Option<&T> {
        self.items.last()
    }

    fn len(&self) -> usize {
        self.items.len()
    }

    fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}

fn main() {
    let mut numbers: Stack<i32> = Stack::new();
    numbers.push(10);
    numbers.push(20);
    println!("peek = {:?}", numbers.peek());
    println!("pop = {:?}", numbers.pop());
    println!("len = {}", numbers.len());

    let mut words = Stack::new();
    words.push(String::from("hello"));
    println!("{:?}", words);
    println!("empty? {}", words.is_empty());
    words.pop();
    println!("pop on empty = {:?}", words.pop());
}
```

**What happens:**

1. `Stack<T>` holds a `Vec<T>`. `Self` inside `impl<T> Stack<T>` means `Stack<T>`.
2. `pop` returns `Option<T>`: `None` when the stack is empty. No crash, no special value. This is
   the safe style from Lesson 04.4.
3. `peek` returns `Option<&T>`, a *reference* to the top item, so we can look without removing.
4. `let mut numbers: Stack<i32> = Stack::new();` tells the compiler `T = i32`. For `words` we
   wrote nothing; Rust sees `push(String::from(…))` and infers `T = String`.
5. The last line pops from an empty stack and gets `None`.

```text
peek = Some(20)
pop = Some(20)
len = 1
Stack { items: ["hello"] }
empty? false
pop on empty = None
```

> 📏 **Convention:** A function named `new` that takes no `self` and returns `Self` is the standard
> constructor in Rust. Clippy (a tool that
> reads your code and suggests improvements; you will use it from Lesson 05.6 on) recommends an
> `is_empty` method next to `len` in public types, as the standard library does. That is why we
> wrote both.

## 5. More about trait bounds

You can require several abilities with `+`:

```rust
use std::fmt::Display;

fn show_largest<T: PartialOrd + Display>(a: T, b: T) {
    if a > b {
        println!("the larger value is {a}");
    } else {
        println!("the larger value is {b}");
    }
}

fn show_pair<T, U>(first: T, second: U)
where
    T: Display,
    U: Display + Clone,
{
    println!("({first}, {second})");
}

fn describe(item: impl Display) {
    println!("item: {item}");
}

fn main() {
    show_largest(3, 9);
    show_largest("apple", "banana");
    show_pair(1, "one");
    describe(2.5);
    describe('x');
}
```

**What happens:**

1. `Display` is the standard trait for "can be printed with `{}`". `T: PartialOrd + Display` means
   the type must be orderable **and** printable.
2. When bounds get long, move them into a **`where` clause** under the signature. It means exactly
   the same, but the function name and arguments stay easy to read.
3. `item: impl Display` is a short form (called *`impl Trait` in argument position*) of
   `fn describe<T: Display>(item: T)`. Use it when you do not need to name the type.
4. `show_largest(3, 9)` and `show_largest("apple", "banana")` use two different types. Both
   arguments must be the **same** `T`: `show_largest(3, "a")` would not compile.

```text
the larger value is 9
the larger value is banana
(1, one)
item: 2.5
item: x
```

### The `Copy` bound

What if `largest` should return the value itself, not a reference? The first idea fails:

```rust
fn largest<T: PartialOrd>(list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

```text
error[E0508]: cannot move out of type `[T]`, a non-copy slice
 --> examples/nocopy.rs:2:23
  |
2 |     let mut largest = list[0];
  |                       ^^^^^^^
  |                       |
  |                       cannot move out of here
  |                       move occurs because `list[_]` has type `T`, which does not implement the `Copy` trait
```

**What it means:** this is the ownership rule from Lesson 03.3. `list[0]` would *move* the value
out of the borrowed slice, which is not allowed for types that do not implement `Copy`. The fix is
the bound `Copy`: only types that are copied cheaply (numbers, `char`, …) are accepted.

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    println!("{}", largest(&[1, 5, 2]));
    println!("{}", largest(&[0.5, -1.0]));
}
```

```text
5
0.5
```

The price: `largest` no longer works for `String`, because `String` is not `Copy`. The reference version
(`-> &T`) is more general. Choosing between them is a normal design decision: *what do I demand
from the caller, and what do I give back?*

### What if the list is empty?

`&list[0]` panics when the slice is empty. A more careful version returns `Option`:

```rust
fn largest<T: PartialOrd>(list: &[T]) -> Option<&T> {
    let mut iter = list.iter();
    let mut largest = iter.next()?;
    for item in iter {
        if item > largest {
            largest = item;
        }
    }
    Some(largest)
}

fn main() {
    println!("{:?}", largest(&[3, 8, 1]));
    let empty: [i32; 0] = [];
    println!("{:?}", largest(&empty));
}
```

**What happens:** `list.iter()` gives an *iterator* (Lesson 05.5 explains them fully; here it is a
"walker" over the list). `iter.next()` returns the first item as `Option<&T>`, and `?` returns
`None` from our function if the list was empty (the `?` operator from Lesson 04.10 works on
`Option` too). The `for` loop then walks the remaining items.

```text
Some(8)
None
```

> 🧪 **Testing:** A generic function must be tested with **more than one type**, and with edge
> cases: an empty list, one item, equal items. Exercise 4 below does exactly this.

## 6. Monomorphization: generics cost nothing at run time

The computer's CPU (Central Processing Unit, the chip that runs instructions) has no idea what
"a type parameter" is. So how does `largest<T>` run? The compiler uses
**monomorphization** (from Greek: "making single-shaped"). For every concrete type you use, it
writes a separate copy of the function with `T` replaced:

```text
 your code                         what the compiler generates
 ─────────                         ───────────────────────────
 fn largest<T>(...)  ──┬── largest_for_i32(&[i32]) -> &i32
                       ├── largest_for_f64(&[f64]) -> &f64
                       └── largest_for_char(&[char]) -> &char
 (one definition)                  (one copy for each type you call it with)
```

(The real internal names look different; this is just the idea.)

The consequences:

- ⚡ **Performance:** the generated copies are exactly as fast as the hand-written
  `largest_i32` and `largest_f64` from the beginning of this lesson. There is no type check, no
  lookup, no indirection at run time. Rust people call this a **zero-cost abstraction**: you pay
  nothing at run time for the convenience.
- The cost moves to **compile time and file size**: many types mean many copies, so compiling takes
  longer and the program file can grow. For most programs this is not a problem. Lesson 06.1 shows
  an alternative (trait objects) that trades some speed for a smaller program.

The size of a generic type also depends on its type parameter. You can measure it with
`std::mem::size_of::<T>()`, which returns the number of bytes a value of the type occupies:

```rust
use std::mem::size_of;

#[allow(dead_code)]
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    println!("Point<u8>  = {} bytes", size_of::<Point<u8>>());
    println!("Point<i32> = {} bytes", size_of::<Point<i32>>());
    println!("Point<f64> = {} bytes", size_of::<Point<f64>>());
    println!("Option<u8>  = {} bytes", size_of::<Option<u8>>());
    println!("Option<i32> = {} bytes", size_of::<Option<i32>>());
}
```

**What happens:** `Point<u8>` has two 1-byte fields (2 bytes); `Point<i32>` two 4-byte fields (8);
`Point<f64>` two 8-byte fields (16). They are really three different types. `Option<u8>` needs
one extra byte to remember "Some or None" (2 bytes), and `Option<i32>` needs padding to keep the
number aligned (8 bytes). `#[allow(dead_code)]` silences the "field never read" warning, because
this program only measures sizes.

```text
Point<u8>  = 2 bytes
Point<i32> = 8 bytes
Point<f64> = 16 bytes
Option<u8>  = 2 bytes
Option<i32> = 8 bytes
```

(The exact sizes of types like `Option<i32>` are chosen by the compiler; they are correct on your
Mac. Phase 12 explains memory layout in depth.)

## 7. The turbofish and type inference

Most of the time Rust infers the type parameter. Sometimes it cannot, and you must say it. There
are two ways: annotate the variable, or use the **turbofish** `::<>` (it looks like a fish):

```rust
fn main() {
    let a = "42".parse::<i32>().unwrap();
    let b: f64 = "2.5".parse().unwrap();
    let v = Vec::<u8>::new();
    let c = std::mem::size_of::<u64>();
    println!("{a} {b} {} {c}", v.len());
}
```

**What happens:**

1. `"42".parse::<i32>()` — `parse` is generic over the type it produces. The turbofish says
   `i32`.
2. `let b: f64 = …parse()` — the annotation on `b` lets the compiler infer the type.
3. `Vec::<u8>::new()` — an empty vector of bytes.
4. `size_of::<u64>()` — here there is no argument to infer from, so the turbofish is the only way.

```text
42 2.5 0 8
```

Without any hint the compiler stops:

```rust
fn main() {
    let n = "42".parse().unwrap();
    println!("{}", n);
}
```

```text
error[E0284]: type annotations needed
 --> examples/turbofish_err.rs:2:9
  |
2 |     let n = "42".parse().unwrap();
  |         ^        ----- type must be known at this point
  |
help: consider giving `n` an explicit type
  |
2 |     let n: /* Type */ = "42".parse().unwrap();
  |          ++++++++++++
```

**Fix:** write `let n: i32 = …` or `parse::<i32>()`.

## 8. Const generics: numbers as parameters

A type parameter stands for a type. A **const generic** stands for a *value*, usually the length
of an array. It is declared with `const N: usize`:

```rust
fn sum_array<const N: usize>(values: [i32; N]) -> i32 {
    let mut total = 0;
    for v in values {
        total += v;
    }
    total
}

fn first_and_len<T: Copy, const N: usize>(values: [T; N]) -> (T, usize) {
    (values[0], N)
}

fn main() {
    println!("{}", sum_array([1, 2, 3]));
    println!("{}", sum_array([10, 20, 30, 40, 50]));
    println!("{:?}", first_and_len(['a', 'b']));
}
```

**What happens:** `[i32; N]` is an array of exactly `N` numbers. The compiler learns `N` from the
argument (3, then 5), and `N` can be used as an ordinary `usize` value inside the function.
Arrays of different lengths are different types, and const generics let one function serve all of
them. You will meet this mostly in library code; remember that it exists.

```text
6
150
('a', 2)
```

> 📏 **Convention:** One-letter names are normal for short generic code: `T` for "type", `E` for
> "error", `K` and `V` for "key" and "value", `N` for a number. When a function has several
> parameters with different roles, use descriptive names (`Item`, `Key`, `Value`) so the signature
> explains itself.

## Common mistakes

**1. Forgetting the bound** (Section 2): `error[E0369]: binary operation '>' cannot be applied to type '&T'`.
Add the trait the body needs: `T: PartialOrd`.

**2. Two different types where one `T` is expected.**

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let p = Point { x: 5, y: 4.0 };
    println!("{}", p.x);
}
```

```text
error[E0308]: mismatched types
 --> examples/point_mixed.rs:7:30
  |
7 |     let p = Point { x: 5, y: 4.0 };
  |                              ^^^ expected integer, found floating-point number
```

**What it means:** `T` was fixed to an integer by `x: 5`, so `y` must be an integer as well. If you
want mixed types, use two parameters, `Point<X, Y>`.

**3. Forgetting `<T>` after `impl`.**

```rust
struct Wrapper<T> {
    value: T,
}

impl Wrapper<T> {
    fn get(&self) -> &T {
        &self.value
    }
}
```

```text
error[E0425]: cannot find type `T` in this scope
 --> examples/noimpl.rs:5:14
  |
5 | impl Wrapper<T> {
  |              ^ not found in this scope
  |
help: you might be missing a type parameter
  |
5 | impl<T> Wrapper<T> {
  |     +++
```

**What it means:** `T` must be declared before it can be used. `impl<T>` declares it; `Wrapper<T>`
uses it. The `help:` shows the fix.

**4. Moving a non-`Copy` value out of a slice** (Section 5): `E0508`. Return a reference, add a
`Copy` bound, or `clone()` (with a `Clone` bound).

**5. The compiler cannot infer the type** (Section 7): `E0284: type annotations needed`. Add a type
annotation or a turbofish.

## Try it yourself

In this phase you keep all small experiments in one practice package, `code/p05-lessons`, and save
each lesson on its own Git branch. If the package does not exist yet, create it.

1. Open a terminal and go to your learning repository:

   ```bash
   cd ~/projects/rust
   git status
   ```

   `git status` should say `nothing to commit, working tree clean`. If it does not, commit or
   undo your old changes first (Lessons 01.3 and 01.5).

2. Create the package and a folder for examples:

   ```bash
   cargo new code/p05-lessons
   mkdir code/p05-lessons/examples
   ```

   Every file in `examples/` is a separate small program. You run one with
   `cargo run --example <file name without .rs>`. This way a whole phase of experiments lives in one
   package, and each experiment has its own `main`.

3. Make sure build output is never committed. The `target/` folder contains compiled files and is
   huge. Look at what Git sees:

   ```bash
   cd code/p05-lessons
   cargo build
   git status --short -uall | head -5
   ```

   If you see lines like `?? code/p05-lessons/target/...`, your repository has no `.gitignore`
   for it. From the repository root create or extend the file:

   ```bash
   cd ~/projects/rust
   echo "target/" >> .gitignore
   git status --short
   ```

   Now the `target/` files must be gone from the list. Only your source files and `.gitignore`
   remain.

4. Start a **branch for this lesson**. One branch per lesson is a good habit: your `main` branch
   always contains finished work, and a failed experiment can be thrown away.

   ```bash
   git switch -c lesson/05-01-generics
   ```

   ```text
   Switched to a new branch 'lesson/05-01-generics'
   ```

5. Create `code/p05-lessons/examples/generics_largest.rs` with the generic `largest` program from
   Section 2 and run it:

   ```bash
   cd code/p05-lessons
   cargo run --example generics_largest
   ```

   You should see `100`, `23.8`, `z`, `zebra`.

6. Save your work with a commit. The message format is from Lesson 01.9 (a short type, a colon, a
   summary):

   ```bash
   cd ~/projects/rust
   git add .
   git commit -m "feat(p05): generic largest function"
   ```

7. Repeat steps 5–6 for the `Stack<T>` program (`examples/generics_stack.rs`), then bring the
   finished lesson back to `main`:

   ```bash
   git switch main
   git merge lesson/05-01-generics
   git branch -d lesson/05-01-generics
   git log --oneline --graph
   ```

   Because `main` did not change meanwhile, Git does a *fast-forward* (it just moves `main`
   forward) and no merge commit appears. Your history is a straight line of small commits.

> 🔒 **Security:** `.gitignore` is also where secrets are kept out of history. Later in the course
> (Phase 10) you will add files such as `.env` to it. A secret that was committed once is in the
> history forever, even if you delete the file in the next commit.

## Exercises

### Exercise 1 — Swap a pair (easy)

Write a generic function `swap_pair<A, B>(pair: (A, B)) -> (B, A)`. For `swap_pair((1, "one"))`
the result must be `("one", 1)`.

<details>
<summary>Hint</summary>

A tuple's fields are `pair.0` and `pair.1`. Return a new tuple with them in the other order. You
need two type parameters because the two parts may have different types.

</details>

<details>
<summary>Answer</summary>

```rust
fn swap_pair<A, B>(pair: (A, B)) -> (B, A) {
    (pair.1, pair.0)
}

fn main() {
    println!("{:?}", swap_pair((1, "one")));
}
```

```text
("one", 1)
```

Why it works: the function takes ownership of the tuple and moves its two parts into a new tuple.
No bound is needed, because we never compare, print or copy the values; we only move them.

</details>

### Exercise 2 — Smallest value and a generic pair (medium)

1. Write `fn smallest<T: PartialOrd>(list: &[T]) -> &T`.
2. Write a struct `Pair<T>` with fields `first` and `second` and a method `larger(&self) -> &T`
   that returns a reference to the bigger of the two. Think about which bound the `impl` block needs.

<details>
<summary>Hint</summary>

For `smallest`, copy `largest` and flip the comparison. For `Pair`, the comparison `>=` between two
`T` values needs `T: PartialOrd`. Put the bound on the `impl` block: `impl<T: PartialOrd> Pair<T>`.

</details>

<details>
<summary>Answer</summary>

```rust
fn smallest<T: PartialOrd>(list: &[T]) -> &T {
    let mut smallest = &list[0];
    for item in list {
        if item < smallest {
            smallest = item;
        }
    }
    smallest
}

#[derive(Debug)]
struct Pair<T> {
    first: T,
    second: T,
}

impl<T: PartialOrd> Pair<T> {
    fn larger(&self) -> &T {
        if self.first >= self.second {
            &self.first
        } else {
            &self.second
        }
    }
}

fn main() {
    println!("{}", smallest(&[7, 3, 9]));
    println!("{}", smallest(&["pear", "apple"]));
    let p = Pair { first: 2.5, second: 1.5 };
    println!("{}", p.larger());
}
```

```text
3
apple
2.5
```

Why it works: the bound on the `impl` block means `larger` exists only for `Pair<T>` where `T`
can be compared. A `Pair` of some uncomparable type can still be created, but it simply has no
`larger` method.

</details>

### Exercise 3 — A generic queue (medium)

Write a generic struct `Queue<T>` (first in, first out) built on `Vec<T>`, with `new`, `enqueue(item)`,
`dequeue() -> Option<T>` and `len`. Then **write tests**: one that checks the order of items and
one that uses `Queue<String>`.

<details>
<summary>Hint</summary>

`enqueue` is `push` on the vector. `dequeue` must take the *first* item: `self.items.remove(0)`
removes and returns the item at index 0. Check `is_empty()` first, because `remove(0)` panics on
an empty vector. Put the tests in a `#[cfg(test)] mod tests` block and run them with
`cargo test --example <name>`.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug)]
struct Queue<T> {
    items: Vec<T>,
}

impl<T> Queue<T> {
    fn new() -> Self {
        Queue { items: Vec::new() }
    }

    fn enqueue(&mut self, item: T) {
        self.items.push(item);
    }

    fn dequeue(&mut self) -> Option<T> {
        if self.items.is_empty() {
            None
        } else {
            Some(self.items.remove(0))
        }
    }

    fn len(&self) -> usize {
        self.items.len()
    }
}

fn main() {
    let mut q = Queue::new();
    q.enqueue("a");
    q.enqueue("b");
    println!("{:?} {:?} {:?} len={}", q.dequeue(), q.dequeue(), q.dequeue(), q.len());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn queue_is_first_in_first_out() {
        let mut q = Queue::new();
        q.enqueue(1);
        q.enqueue(2);
        assert_eq!(q.dequeue(), Some(1));
        assert_eq!(q.dequeue(), Some(2));
        assert_eq!(q.dequeue(), None);
    }

    #[test]
    fn queue_works_with_strings() {
        let mut q: Queue<String> = Queue::new();
        q.enqueue(String::from("x"));
        assert_eq!(q.len(), 1);
    }
}
```

```text
Some("a") Some("b") None len=0
```

Running `cargo test --example answers` (if the file is `examples/answers.rs`) prints
`test result: ok. 2 passed` for the two tests.

Why it works: the same `Queue` code serves `&str`, `i32` and `String`. `remove(0)` shifts all other
items one place to the left, so it is slow for long queues (it takes time proportional to the length).
A real queue uses `VecDeque`, which you can look up in the standard library docs; for learning,
the simple version is fine.

</details>

### Exercise 4 — Minimum and maximum at once (hard)

Write `fn min_max<T: PartialOrd + Copy>(list: &[T]) -> Option<(T, T)>` that returns the smallest and
largest value, or `None` for an empty slice. Write tests for: several numbers, one `f64`, and the
empty slice.

<details>
<summary>Hint</summary>

Return `None` early if `list.is_empty()`. Otherwise start both `min` and `max` from `list[0]` and
update them in one loop. For the empty-slice test the compiler needs to know `T`; write
`min_max::<u8>(&[])` with the turbofish.

</details>

<details>
<summary>Answer</summary>

```rust
fn min_max<T: PartialOrd + Copy>(list: &[T]) -> Option<(T, T)> {
    if list.is_empty() {
        return None;
    }
    let mut min = list[0];
    let mut max = list[0];
    for &item in list {
        if item < min {
            min = item;
        }
        if item > max {
            max = item;
        }
    }
    Some((min, max))
}

fn main() {
    println!("{:?}", min_max(&[4, -2, 9]));
    println!("{:?}", min_max::<i32>(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn min_max_finds_both_ends() {
        assert_eq!(min_max(&[4, -2, 9]), Some((-2, 9)));
        assert_eq!(min_max(&[1.5]), Some((1.5, 1.5)));
    }

    #[test]
    fn min_max_of_empty_is_none() {
        assert_eq!(min_max::<u8>(&[]), None);
    }
}
```

```text
Some((-2, 9))
None
```

Why it works: `Copy` lets us copy values out of the slice into `min` and `max`. `assert_eq!` needs
the compared values to implement `Debug` and `PartialEq`; `Option<(T, T)>` has both whenever `T` does,
and the integer and float types used here do.

</details>

## Quiz

1. What does `fn first<T>(list: &[T]) -> &T` mean in plain words?

<details>
<summary>Answer</summary>

"For any type `T`, take a borrowed slice of `T` values and return a borrowed reference to one `T`." `T` is a type parameter: a blank the compiler fills in at each call.

</details>

2. Why does `fn largest<T>(list: &[T]) -> &T` fail to compile when the body uses `>`?

<details>
<summary>Answer</summary>

Because `T` can be any type, including types that cannot be compared. The body may only use abilities promised by the bounds. Adding `T: PartialOrd` makes the comparison legal.

</details>

3. What is the difference between `impl<T> Point<T>` and `impl Point<f64>`?

<details>
<summary>Answer</summary>

`impl<T> Point<T>` defines methods for every `Point`, whatever `T` is. `impl Point<f64>` defines methods only for `Point<f64>`. A `Point<i32>` does not have them.

</details>

4. Does using generics make a program slower at run time? What do you pay instead?

<details>
<summary>Answer</summary>

No. Monomorphization creates a specialised copy of the generic code for each concrete type, so the run-time speed equals hand-written code. You pay at compile time (more code to compile) and possibly in program size (more copies).

</details>

5. When do you need the turbofish `::<>`?

<details>
<summary>Answer</summary>

When the compiler cannot infer a type parameter from the arguments or from a type annotation, for example `"42".parse::<i32>()` or `size_of::<u64>()`.

</details>

6. What is `N` in `fn f<const N: usize>(a: [i32; N])`?

<details>
<summary>Answer</summary>

A const generic parameter: a number (a value, not a type) that is part of the array's type. The compiler takes it from the length of the array you pass.

</details>

## Summary

- A generic function or type works for many types; you write it once (DRY) and the compiler fills in the type.
- A type parameter (`T`) is declared in `<>` after the name, and after `impl` when you write methods.
- The compiler checks generic code once, using only the abilities listed in the trait bounds (`T: PartialOrd + Copy`, or a `where` clause).
- `Option<T>`, `Result<T, E>` and `Vec<T>` are generic types from the standard library.
- Monomorphization makes one copy of the code per concrete type: no run-time cost, but longer compile time and bigger files.
- Use the turbofish `::<Type>` or a type annotation when inference is not enough.
- Const generics (`const N: usize`) make numbers, such as array lengths, a parameter.
- Git habit: one branch per lesson, small commits, merge to `main` when the lesson is done.

## Sources and further reading

- [The Rust Programming Language — 10.1 Generic Data Types](https://doc.rust-lang.org/book/ch10-01-syntax.html) — the official chapter on generics.
- [Rust by Example — Generics](https://doc.rust-lang.org/rust-by-example/generics.html) — many short runnable examples.
- [The Rust Reference — Generic parameters](https://doc.rust-lang.org/reference/items/generics.html) — the exact rules, including const generics.
- [std::mem::size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) — the function used in Section 6.
- [Git documentation — git switch](https://git-scm.com/docs/git-switch) — branches with `git switch -c`.

---

[🏠 Start](../START-HERE.md) · [← Phase 05 — Rust intermediate](README.md) · [📋 Phase 05](README.md) · [05.2 Traits →](02-traits.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
