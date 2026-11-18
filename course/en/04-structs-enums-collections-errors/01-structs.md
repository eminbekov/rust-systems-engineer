<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 04 — Structs, enums, collections and errors](README.md) · [📋 Phase 04](README.md) · [04.2 Methods and associated functions →](02-methods-and-associated-functions.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/01-structs.md) <!-- nav -->

# Lesson 04.1 — Structs

> **Phase 04 — Structs, enums, collections and errors** · Lesson 1 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- What a **struct** is and why programs need to group related values under one name.
- How to define a struct, create a value of it, read and change its fields.
- Three shortcuts: field init shorthand, struct update syntax, and the two "light" kinds of struct (tuple structs and unit-like structs).
- How to print a struct for debugging with `#[derive(Debug)]`, `{:?}`, `{:#?}` and `dbg!`.
- How structs interact with ownership, borrowing and `Copy`, and how a struct looks in memory.

## Before you start

- You finished [Lesson 03.5 — Mutable references and the borrowing rules](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) and [Lesson 03.3 — Move, copy and clone](../03-ownership-and-borrowing/03-move-copy-and-clone.md). Structs follow the same ownership rules as every other value.
- You know how to write a test ([Lesson 02.8 — Your first tests](../02-programming-basics/08-your-first-tests.md)) and how to print with `println!` ([Lesson 02.6 — Printing, formatting and comments](../02-programming-basics/06-printing-formatting-and-comments.md)).
- You have your learning repository from [Lesson 01.10 — Project: your learning repository, and checkpoint](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md). We will make a small commit at the end of this lesson.

## New words

| Word | Meaning |
|---|---|
| **struct** (structure) | A type you design yourself. It groups several values, each with its own name and type, into one value. |
| **field** | One named part of a struct. A `User` struct may have the fields `username` and `email`. |
| **instance** | One concrete value of a struct type. `User` is the type; one particular user is an instance. |
| **field init shorthand** | Writing `username` instead of `username: username` when a variable has the same name as the field. |
| **struct update syntax** | Writing `..other` at the end of a struct value to take the remaining fields from another instance. |
| **tuple struct** | A struct whose fields have no names, only positions: `Color(255, 0, 0)`. |
| **unit-like struct** | A struct with no fields at all: `struct Marker;`. It takes zero bytes. |
| **newtype** | A tuple struct with one field, used to give an existing type a new meaning: `struct Meters(f64);`. |
| **attribute** | A note in the form `#[...]` placed before an item. It tells the compiler something about the item. |
| **derive** | An attribute (`#[derive(Debug)]`) that asks the compiler to write standard code for your type automatically. |
| **trait** | A named set of abilities a type can have, such as "can be printed with `{:?}`". You will learn to define traits in [Lesson 05.2 — Traits](../05-rust-intermediate/02-traits.md). Here we only ask the compiler to give our types some. |
| **Debug** | The trait behind `{:?}`. It prints a value in a developer-friendly form. |
| **PartialEq** | The trait that makes `==` work for your type. |
| **dbg!** | A macro (a special kind of call, written with `!`) that prints a value together with the file and line, then gives the value back. |
| **padding** | Unused bytes the compiler adds inside a struct so that every field sits at an address it likes. |
| **FFI** (Foreign Function Interface) | A way for Rust code to call code written in another language, such as C. See [Lesson 06.10 — FFI: calling C code](../06-rust-advanced/10-ffi-calling-c.md). |
| **UpperCamelCase** | A naming style where every word starts with a capital letter and there are no underscores: `UserAccount`. Rust uses it for type names. |
| **std** (standard library) | The library that ships with Rust. `std::mem::size_of` means "the function `size_of` in the module `mem` of the standard library". |
| **stderr** (standard error) | The output channel that programs use for diagnostics and error messages, separate from the normal output. |

## 1. Why structs exist

Imagine you write a program that handles users. Each user has a name, an email, a flag that says whether the account is active, and a number of logins. Without structs you would carry four separate variables around:

```rust
let username = String::from("alice");
let email = String::from("alice@example.com");
let active = true;
let login_count = 1;
```

Now you want to pass "the user" to a function. You need four parameters. A second user means four more variables. It is easy to mix them up: `alice`'s name with `bob`'s email.

A **struct** fixes this. It is like a paper form with labelled boxes: one form, many named boxes. You design the form once (the *type*), then fill in as many copies as you need (the *instances*).

## 2. Defining and creating a struct

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    login_count: u32,
}

fn main() {
    let user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        active: true,
        login_count: 1,
    };

    println!("{} <{}>", user.username, user.email);
    println!("active: {}, logins: {}", user.active, user.login_count);
}
```

**What happens:**

1. `struct User { ... }` defines a new type called `User`. Inside the braces you list the fields: a name, a colon, a type, and a comma after each.
2. In `main` we create an instance. We write the type name and then, in braces, a value for **every** field as `name: value`. The order does not matter, but no field may be missing.
3. `user.username` reads the field `username` of the instance `user`. The dot is how you reach inside a struct.
4. The last two lines print four different fields of one value.

```text
alice <alice@example.com>
active: true, logins: 1
```

> 📏 **Convention:** type names (`User`) use `UpperCamelCase`; field names (`login_count`) use `snake_case`. `cargo clippy` and the compiler warn you if you break this rule.

### Changing fields

A struct instance is either fully mutable or fully immutable. You cannot mark only one field `mut`. The whole binding decides:

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    login_count: u32,
}

fn main() {
    let mut user = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        active: true,
        login_count: 1,
    };

    user.login_count += 1;
    user.email = String::from("alice@new-mail.example");

    println!("{} <{}>", user.username, user.email);
    println!("active: {}, logins: {}", user.active, user.login_count);
}
```

**What happens:**

1. `let mut user` makes the whole instance changeable.
2. `user.login_count += 1;` adds one to a single field.
3. `user.email = ...` replaces a `String` field. The old `String` is dropped (its memory is freed) at that moment — you learned this in [Lesson 03.2 — The ownership rules](../03-ownership-and-borrowing/02-ownership-rules.md).

```text
alice <alice@new-mail.example>
active: true, logins: 2
```

If you forget `mut`, the compiler stops you:

```text
error[E0594]: cannot assign to `user.login_count`, as `user` is not declared as mutable
  --> src/main.rs:11:5
   |
11 |     user.login_count += 1;
   |     ^^^^^^^^^^^^^^^^^^^^^ cannot assign
   |
help: consider changing this to be mutable
   |
 7 |     let mut user = User {
   |         +++
```

It reads: you assign to a field, but the variable `user` is not mutable. The fix is the one the compiler shows: `let mut user`.

## 3. Field init shorthand

Functions that build a struct often have parameters named exactly like the fields. Repeating the name is noisy:

```rust
User { username: username, email: email, active: true, login_count: 0 }
```

When the variable and the field have the same name, write the name once:

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    login_count: u32,
}

fn build_user(username: String, email: String) -> User {
    User {
        username,
        email,
        active: true,
        login_count: 0,
    }
}

fn main() {
    let user = build_user(String::from("bob"), String::from("bob@example.com"));
    println!("{} <{}> active={} logins={}", user.username, user.email, user.active, user.login_count);
}
```

**What happens:**

1. `build_user` takes two `String`s. Ownership of both moves into the function.
2. `User { username, email, ... }` is the shorthand for `username: username, email: email`. The two `String`s move again, now into the new struct.
3. The last expression of the function (no semicolon) is the returned `User`.

```text
bob <bob@example.com> active=true logins=0
```

> 💡 **Tip:** a function like `build_user` is called a *constructor*. In [Lesson 04.2 — Methods and associated functions](02-methods-and-associated-functions.md) you will learn the standard Rust way to write constructors (`User::new(...)`).

## 4. Struct update syntax

Often you want a new instance that is almost the same as an old one. Write the fields that differ, then `..old` to copy the rest:

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    login_count: u32,
}

fn main() {
    let user1 = User {
        username: String::from("alice"),
        email: String::from("alice@example.com"),
        active: true,
        login_count: 5,
    };

    let user2 = User {
        email: String::from("alice2@example.com"),
        ..user1
    };

    println!("{} <{}> logins={}", user2.username, user2.email, user2.login_count);
    // user1.active is a bool (Copy), so it is still usable:
    println!("user1 still active? {}", user1.active);
}
```

**What happens:**

1. `user2` gets its own `email`. Every other field (`username`, `active`, `login_count`) is taken from `user1`. `..user1` must be the last item and has no comma after it.
2. Taking a field is a **move or a copy**, exactly like `let a = b;`. `username` is a `String`, so it is *moved* out of `user1`. `active` and `login_count` are simple numbers/booleans that implement `Copy`, so they are copied.
3. Because `user1.username` was moved, `user1` as a whole can no longer be used. But the fields that were only copied are still fine, so `user1.active` works.

```text
alice <alice2@example.com> logins=5
user1 still active? true
```

If you try to read the moved field, you get the familiar ownership error:

```text
error[E0382]: borrow of moved value: `user1.username`
  --> src/main.rs:17:20
   |
13 |       let user2 = User {
   |  _________________-
14 | |         email: String::from("other@example.com"),
15 | |         ..user1
16 | |     };
   | |_____- value moved here
17 |       println!("{}", user1.username);
   |                      ^^^^^^^^^^^^^^ value borrowed here after move
   |
   = note: move occurs because `user1.username` has type `String`, which does not implement the `Copy` trait
```

The note at the bottom tells the whole story: `String` is not `Copy`, so `..user1` moved it. To keep `user1.username`, give `user2` its own `username` too, or use `.clone()` (see [Lesson 03.3](../03-ownership-and-borrowing/03-move-copy-and-clone.md)).

> ⚡ **Performance:** a *move* of a `String` copies only its small header (pointer, length, capacity) — not the text on the heap. That is why struct update syntax is cheap. `.clone()` copies the heap text and costs more.

## 5. Tuple structs, newtypes and unit-like structs

Sometimes field names add nothing. Rust has two lighter kinds of struct.

A **tuple struct** has fields identified by position:

```rust
struct Color(u8, u8, u8);
struct Meters(f64);
struct Marker;

fn main() {
    let red = Color(255, 0, 0);
    let length = Meters(12.5);
    let _marker = Marker;

    println!("red = ({}, {}, {})", red.0, red.1, red.2);
    println!("length = {} m", length.0);

    let Color(r, g, b) = red;
    println!("r={r} g={g} b={b}");
}
```

**What happens:**

1. `Color(u8, u8, u8)` defines a type with three unnamed fields. You create it like a function call: `Color(255, 0, 0)`.
2. You read a field with `.0`, `.1`, `.2` (position, counted from zero).
3. `Meters(f64)` has one field. This pattern is called a **newtype**. `Meters(12.5)` and a plain `f64` are different types, so you cannot add metres to seconds by accident.
4. `struct Marker;` has no fields. It is a **unit-like struct**: it carries no data, but it is still a distinct type. It is useful later, when you want a type only to attach behaviour to.
5. `let Color(r, g, b) = red;` takes the tuple struct apart into three variables. This is *destructuring*; section 9 shows more.

```text
red = (255, 0, 0)
length = 12.5 m
r=255 g=0 b=0
```

> 💡 **Tip:** use a newtype when two values have the same raw type but different meaning: `UserId(u64)` and `OrderId(u64)`. The compiler will refuse to pass an order id where a user id is expected. This kind of mistake is common and hard to find without help.

## 6. Printing a struct: `Debug`

Try to print a struct you just defined:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);
}
```

```text
error[E0277]: `Point` doesn't implement `Debug`
 --> src/main.rs:8:22
  |
8 |     println!("{:?}", p);
  |               ----   ^ `Point` cannot be formatted using `{:?}` because it doesn't implement `Debug`
  |               |
  |               required by this formatting parameter
  |
  = help: the trait `Debug` is not implemented for `Point`
help: consider annotating `Point` with `#[derive(Debug)]`
  |
1 + #[derive(Debug)]
2 | struct Point {
  |
```

The compiler does not know how you want a `Point` to look. `{:?}` needs the **`Debug`** ability (a *trait*). You can ask the compiler to write it for you with an **attribute** called `derive`:

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(Debug)]
struct Player {
    name: String,
    position: Point,
    health: u8,
}

fn main() {
    let p = Player {
        name: String::from("Zed"),
        position: Point { x: 3, y: -4 },
        health: 100,
    };
    println!("{:?}", p);
    println!("{:#?}", p);
}
```

**What happens:**

1. `#[derive(Debug)]` is placed right above the struct. It tells the compiler: "generate the code that prints this type in debug form". Every field's type must also support `Debug`, which is why `Point` has it too.
2. A `Player` contains a `Point`. Structs can hold other structs; this is how you build bigger models from small pieces.
3. `{:?}` prints on one line.
4. `{:#?}` ("pretty debug") prints on many lines with indentation. It is easier to read for big values.

```text
Player { name: "Zed", position: Point { x: 3, y: -4 }, health: 100 }
Player {
    name: "Zed",
    position: Point {
        x: 3,
        y: -4,
    },
    health: 100,
}
```

> 💡 **Tip:** while you build a program, derive `Debug` on almost every struct. It costs nothing at run time and saves a lot of time. If cargo prints a `warning: fields ... are never read` for a demo program, it only means the demo does not use the fields yet; the warnings are left out of the outputs in this lesson.

`{:?}` is for developers. For text that users see, you write your own formatting; you will do that in [Lesson 04.2](02-methods-and-associated-functions.md).

### `dbg!`

`dbg!(expression)` prints the expression, its value, and the file and line to the **error stream** (`stderr`, the channel programs use for diagnostics, separate from normal output). Then it returns the value, so you can wrap it around any expression:

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let scale = 3;
    let rect = Rectangle {
        width: dbg!(10 * scale),
        height: 20,
    };
    dbg!(&rect);
}
```

**What happens:**

1. `dbg!(10 * scale)` computes `30`, prints a line to `stderr`, and returns `30`, so `width` becomes `30`.
2. `dbg!(&rect)` receives a *reference*. If we wrote `dbg!(rect)`, the macro would take ownership and `rect` would be moved into it.

```text
[src/main.rs:10:16] 10 * scale = 30
[src/main.rs:13:5] &rect = Rectangle {
    width: 30,
    height: 20,
}
```

The numbers after the file name are the line and the column.

> 📏 **Convention:** `dbg!` is a temporary tool. Remove it before you commit. `println!` is for output that belongs to the program.

## 7. Structs, ownership and borrowing

A struct **owns** its fields. When a struct goes out of scope, all its fields are dropped. That is why we used `String` and not `&str` for text fields: the struct itself keeps the text alive.

What if you try a borrowed field?

```rust
struct Article {
    title: &str,
    words: u32,
}
```

```text
error[E0106]: missing lifetime specifier
 --> src/main.rs:2:12
  |
2 |     title: &str,
  |            ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 ~ struct Article<'a> {
2 ~     title: &'a str,
  |
```

A reference inside a struct must be proven to stay valid as long as the struct lives. The compiler needs a *lifetime* annotation (`'a`) to check that. Lifetimes are a full topic in [Lesson 05.3 — Lifetimes](../05-rust-intermediate/03-lifetimes.md). Until then, follow one simple rule:

> 📏 **Convention:** in a struct, use **owned** types (`String`, `Vec<T>`, numbers). Use `&str` fields only after you have learned lifetimes.

### Passing structs to functions

Passing a struct by value *moves* it, like any other value:

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn show(p: Point) {
    println!("({}, {})", p.x, p.y);
}

fn main() {
    let a = Point { x: 1, y: 2 };
    show(a);
    println!("{:?}", a);
}
```

```text
error[E0382]: borrow of moved value: `a`
  --> src/main.rs:14:22
   |
12 |     let a = Point { x: 1, y: 2 };
   |         - move occurs because `a` has type `Point`, which does not implement the `Copy` trait
13 |     show(a);
   |          - value moved here
14 |     println!("{:?}", a);
   |                      ^ value borrowed here after move
   |
note: consider changing this parameter type in function `show` to borrow instead if owning the value isn't necessary
```

There are two fixes. The usual one is to **borrow**: take `&Point` in the function and call `show(&a)`. The reference lets the function look without taking ownership. If the function must change the struct, take `&mut Point`:

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

fn area(rect: &Rectangle) -> u32 {
    rect.width * rect.height
}

fn grow(rect: &mut Rectangle, by: u32) {
    rect.width += by;
    rect.height += by;
}

fn main() {
    let mut rect = Rectangle { width: 30, height: 50 };
    println!("area = {}", area(&rect));
    grow(&mut rect, 10);
    println!("area after grow = {}", area(&rect));
}
```

**What happens:**

1. `area` borrows the rectangle read-only (`&Rectangle`). `main` still owns `rect` afterwards.
2. `grow` borrows it mutably (`&mut Rectangle`) and changes both fields through the reference. This follows the borrowing rules from [Lesson 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md): while the `&mut` exists, nobody else may use `rect`.
3. `main` has to say `&mut rect` at the call, so the change is visible at the call site.

```text
area = 1500
area after grow = 2400
```

The second fix is to make the struct **`Copy`**. A small struct made only of `Copy` fields can opt in:

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

fn show(p: Point) {
    println!("({}, {})", p.x, p.y);
}

fn main() {
    let a = Point { x: 1, y: 2 };
    let b = a; // copied, not moved
    show(a);
    show(b);
    println!("{:?}", a);
}
```

**What happens:**

1. `#[derive(Clone, Copy)]` asks the compiler to make the type copyable. This is only allowed when every field is `Copy` (here two `i32`s). A struct with a `String` can never be `Copy`.
2. `let b = a;` and `show(a)` now copy the 8 bytes instead of moving. `a` stays valid.
3. `Copy` requires `Clone`, so you always derive both together.

```text
(1, 2)
(1, 2)
Point { x: 1, y: 2 }
```

> 📏 **Convention:** derive `Copy` only for small "plain data" types such as points, colors or ids. For anything that owns heap data, keep the move semantics.

## 8. How a struct looks in memory

A struct is just its fields laid out one after another in a single block of memory. No hidden header, no pointer to the fields:

```text
Rectangle { width: 30, height: 50 }      (on the stack)

 address:   +0        +4
          ┌────────┬────────┐
          │   30   │   50   │
          └────────┴────────┘
           width     height      total: 8 bytes (two u32 of 4 bytes)
```

A struct with a `String` has the `String` header inline, and the text on the heap ([Lesson 03.1 — Memory: the stack and the heap](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)):

```text
User { username: String, login_count: u32 }

 stack                                         heap
┌─────────────────────────────┐
│ username.ptr  ──────────────┼──────────►  a l i c e
│ username.len  = 5           │
│ username.cap  = 5           │
│ login_count   = 1           │
└─────────────────────────────┘
```

The compiler may add **padding** and may reorder fields to keep each one aligned. You can measure the size with `std::mem::size_of`:

```rust
use std::mem::size_of;

struct Mixed {
    a: u8,
    b: u32,
    c: u8,
}

#[repr(C)]
struct MixedC {
    a: u8,
    b: u32,
    c: u8,
}

struct Pair(u32, u32);
struct Marker;

fn main() {
    println!("Pair   = {} bytes", size_of::<Pair>());
    println!("Marker = {} bytes", size_of::<Marker>());
    println!("Mixed  = {} bytes", size_of::<Mixed>());
    println!("MixedC = {} bytes", size_of::<MixedC>());
}
```

**What happens:**

1. `size_of::<T>()` returns how many bytes a value of type `T` occupies. The part in `::<...>` names the type.
2. `Pair` is two 4-byte numbers: 8 bytes. `Marker` has no fields: 0 bytes.
3. `Mixed` has 1 + 4 + 1 = 6 bytes of data. By default, Rust is free to reorder fields. It puts the `u32` first and the two `u8`s after it, so the total is 8 (6 bytes of data and 2 of padding).
4. `#[repr(C)]` forbids reordering and uses the C language layout, in the order written. The `u32` must start at a multiple of 4, so after `a` there are 3 padding bytes, and after `c` 3 more: 12 bytes in total. You need `repr(C)` when you share a struct with C code (FFI) — see [Lesson 06.10](../06-rust-advanced/10-ffi-calling-c.md).

```text
Pair   = 8 bytes
Marker = 0 bytes
Mixed  = 8 bytes
MixedC = 12 bytes
```

> ⚡ **Performance:** you rarely need to think about layout. Rust already reorders fields to waste less space. When a struct is created millions of times (for example in a large list), its size matters, and `size_of` tells you the truth.

## 9. Destructuring a struct

You can pull a struct apart into variables with the same shape you used to build it:

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 7, y: 9 };
    let Point { x, y } = p;
    println!("x={x}, y={y}");

    let Point { x: horizontal, .. } = p;
    println!("horizontal={horizontal}");
}
```

**What happens:**

1. `let Point { x, y } = p;` creates variables `x` and `y` with the values of the fields of the same name.
2. `x: horizontal` means "take field `x` and call the variable `horizontal`". `..` says "ignore the other fields".
3. `p` is still usable at the end, because both fields are `Copy` numbers.

```text
x=7, y=9
horizontal=7
```

This is a small part of a bigger tool called *patterns*. You will study them in [Lesson 04.5 — if let, let else and patterns](05-if-let-let-else-and-patterns.md).

## 10. Testing structs

> 🧪 **Testing:** to compare two structs in `assert_eq!`, the type must implement two traits: `PartialEq` (so `==` works) and `Debug` (so the test can show both values when they differ). Derive both.

```rust
#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn origin() -> Point {
    Point { x: 0, y: 0 }
}

fn main() {
    println!("{:?}", origin());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn origin_is_zero_zero() {
        assert_eq!(origin(), Point { x: 0, y: 0 });
    }
}
```

**What happens:**

1. `#[derive(Debug, PartialEq)]` gives `Point` both abilities. `PartialEq` compares field by field.
2. `#[cfg(test)] mod tests` is a module that exists only when you run `cargo test` ([Lesson 02.8](../02-programming-basics/08-your-first-tests.md)). `use super::*;` brings the parent's items into it.
3. `assert_eq!` calls `==` on two `Point`s and fails the test if they differ.

```text
running 1 test
test tests::origin_is_zero_zero ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Without `PartialEq` the test does not even compile:

```text
error[E0369]: binary operation `==` cannot be applied to type `Point`
  --> src/main.rs:21:9
   |
21 |         assert_eq!(origin(), Point { x: 0, y: 0 });
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |         |
   |         Point
   |         Point
   |
note: an implementation of `PartialEq` might be missing for `Point`
help: consider annotating `Point` with `#[derive(PartialEq)]`
```

## Common mistakes

> ⚠️ **Common mistake:** forgetting a field when creating a struct.

```rust
let p = Point { x: 7 };
```

```text
error[E0063]: missing field `y` in initializer of `Point`
 --> src/main.rs:7:13
  |
7 |     let p = Point { x: 7 };
  |             ^^^^^ missing `y`
```

Every field needs a value. Fix: `Point { x: 7, y: 0 }`. (Or use struct update syntax with a base value.)

> ⚠️ **Common mistake:** trying to make one field mutable with `let user = ...; user.field = ...;`. The error is `E0594` (see section 2). Make the whole variable `mut`.

> ⚠️ **Common mistake:** using a struct after passing it by value to a function (`E0382`, section 7). Borrow with `&` instead, or derive `Copy` for tiny plain-data types.

> ⚠️ **Common mistake:** `&str` in a field (`E0106`). Use `String` until you learn lifetimes.

> ⚠️ **Common mistake:** printing with `{}` or `{:?}` without implementing the trait. `{}` needs `Display`, which you must write by hand ([Lesson 04.2](02-methods-and-associated-functions.md)); `{:?}` needs `Debug`, which you can derive.

## Try it yourself

1. Create a project in your course folder:

   ```text
   cd ~/projects/rust
   cargo new code/p04-structs
   cd code/p04-structs
   ```

2. Replace `src/main.rs` with:

   ```rust
   #[derive(Debug)]
   struct Item {
       name: String,
       quantity: u32,
       price_cents: u32,
   }

   fn total_cents(item: &Item) -> u32 {
       item.quantity * item.price_cents
   }

   fn main() {
       let pen = Item {
           name: String::from("pen"),
           quantity: 12,
           price_cents: 150,
       };
       let notebook = Item {
           name: String::from("notebook"),
           quantity: 3,
           price_cents: 899,
       };

       for item in [&pen, &notebook] {
           println!(
               "{}: {} x {} = {} cents",
               item.name,
               item.quantity,
               item.price_cents,
               total_cents(item)
           );
       }
       println!("{:#?}", pen);
   }
   ```

3. Run it with `cargo run`. Expected output:

   ```text
   pen: 12 x 150 = 1800 cents
   notebook: 3 x 899 = 2697 cents
   Item {
       name: "pen",
       quantity: 12,
       price_cents: 150,
   }
   ```

4. Change the price of the pen and run again. Add a third `Item` made with struct update syntax (`..pen`).

> 📏 **Convention:** money is stored as whole cents in an integer (`price_cents: u32`), not as a decimal number such as `f64`. Floating-point numbers cannot represent values like 0.10 exactly, and the small errors add up.

5. Save your work in Git ([Lessons 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md) and [01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md)):

   ```text
   cd ~/projects/rust
   git status
   git add code/p04-structs
   git commit -m "feat(p04): add Item struct demo"
   ```

   `git status` should list only `code/p04-structs`. If you see a `target/` folder listed, add the line `target/` to a `.gitignore` file first: build output never belongs in Git. The commit message follows the convention from Lesson 01.9 (type, scope, short description).

## Exercises

### Exercise 1 — A book (easy)

Define a struct `Book` with the fields `title: String`, `author: String`, `pages: u32` and `available: bool`. Derive `Debug`. In `main`, create one book and print it with `{:#?}`.

<details>
<summary>Hint</summary>

Put `#[derive(Debug)]` directly above `struct Book`. Use `String::from("...")` for text fields.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug)]
struct Book {
    title: String,
    author: String,
    pages: u32,
    available: bool,
}

fn main() {
    let book = Book {
        title: String::from("The Rust Programming Language"),
        author: String::from("Steve Klabnik and Carol Nichols"),
        pages: 560,
        available: true,
    };
    println!("{:#?}", book);
}
```

Output:

```text
Book {
    title: "The Rust Programming Language",
    author: "Steve Klabnik and Carol Nichols",
    pages: 560,
    available: true,
}
```

Why it works: the derive generates the printing code, and `{:#?}` uses it in multi-line form.

</details>

### Exercise 2 — Rectangle helpers with tests (medium)

Define `Rectangle { width: u32, height: u32 }` (derive `Debug` and `PartialEq`). Write `fn is_square(rect: &Rectangle) -> bool` and `fn scale(rect: &Rectangle, factor: u32) -> Rectangle`. Write two tests: one for `is_square` (true and false cases) and one for `scale`.

<details>
<summary>Hint</summary>

Both functions take a reference, so the caller keeps the rectangle. `scale` builds and returns a **new** `Rectangle`. In the test, compare whole structs with `assert_eq!` (that is why `PartialEq` is needed).

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn is_square(rect: &Rectangle) -> bool {
    rect.width == rect.height
}

fn scale(rect: &Rectangle, factor: u32) -> Rectangle {
    Rectangle {
        width: rect.width * factor,
        height: rect.height * factor,
    }
}

fn main() {
    let r = Rectangle { width: 4, height: 4 };
    println!("{} {:?}", is_square(&r), scale(&r, 3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn square_is_detected() {
        assert!(is_square(&Rectangle { width: 5, height: 5 }));
        assert!(!is_square(&Rectangle { width: 5, height: 6 }));
    }

    #[test]
    fn scale_multiplies_both_sides() {
        let r = Rectangle { width: 2, height: 3 };
        assert_eq!(scale(&r, 4), Rectangle { width: 8, height: 12 });
    }
}
```

`cargo run` prints `true Rectangle { width: 12, height: 12 }`, and `cargo test` reports `2 passed`.

Why it works: the functions only read the rectangle, so `&Rectangle` is enough; the test compares the returned struct with an expected one field by field.

</details>

### Exercise 3 — Two newtypes (medium)

Create the newtypes `Celsius(f64)` and `Fahrenheit(f64)` (derive `Debug` and `PartialEq`). Write `fn to_fahrenheit(c: Celsius) -> Fahrenheit` using `F = C × 9 / 5 + 32`. Test the values 0 → 32, 100 → 212 and −40 → −40.

<details>
<summary>Hint</summary>

Read the inner number with `c.0`. Build the result with `Fahrenheit( ... )`. Use `9.0` and `5.0` (decimal literals), because `c.0` is an `f64`.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq)]
struct Celsius(f64);

#[derive(Debug, PartialEq)]
struct Fahrenheit(f64);

fn to_fahrenheit(c: Celsius) -> Fahrenheit {
    Fahrenheit(c.0 * 9.0 / 5.0 + 32.0)
}

fn main() {
    println!("{:?}", to_fahrenheit(Celsius(100.0)));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_points() {
        assert_eq!(to_fahrenheit(Celsius(0.0)), Fahrenheit(32.0));
        assert_eq!(to_fahrenheit(Celsius(100.0)), Fahrenheit(212.0));
        assert_eq!(to_fahrenheit(Celsius(-40.0)), Fahrenheit(-40.0));
    }
}
```

The program prints `Fahrenheit(212.0)`. Why it works: the two types hold the same raw `f64`, but the compiler keeps them apart, so you cannot pass a `Fahrenheit` where a `Celsius` is expected. (Comparing `f64` with `==` is safe here only because these particular results are exact; see the next exercise for the general approach.)

</details>

### Exercise 4 — Distance between points (hard)

Define `Point { x: f64, y: f64 }` (derive `Debug`, `Clone`, `Copy`). Write `fn distance(a: Point, b: Point) -> f64` using the Pythagorean formula: the square root of `dx² + dy²`. Write a test for the 3-4-5 triangle and one for the distance from a point to itself. Compare floating-point results with a small tolerance, not with `==`.

<details>
<summary>Hint</summary>

`f64` has a method `.sqrt()`. For the comparison write `(d - 5.0).abs() < 1e-9`, where `1e-9` means 0.000000001. Because `Point` is `Copy`, `distance(p, p)` compiles even though `p` is used twice.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}

fn distance(a: Point, b: Point) -> f64 {
    let dx = a.x - b.x;
    let dy = a.y - b.y;
    (dx * dx + dy * dy).sqrt()
}

fn main() {
    let a = Point { x: 0.0, y: 0.0 };
    let b = Point { x: 3.0, y: 4.0 };
    println!("{}", distance(a, b));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn three_four_five() {
        let d = distance(Point { x: 0.0, y: 0.0 }, Point { x: 3.0, y: 4.0 });
        assert!((d - 5.0).abs() < 1e-9);
    }

    #[test]
    fn distance_to_itself_is_zero() {
        let p = Point { x: 1.5, y: -2.0 };
        assert!(distance(p, p).abs() < 1e-9);
    }
}
```

The program prints `5`, and both tests pass.

Why it works: computers store `f64` with tiny rounding errors, so two results that are "equal" in mathematics may differ in the last bits. Testing that the difference is smaller than a tolerance is the standard fix.

</details>

## Quiz

1. What is the difference between a struct type and an instance of it?

<details>
<summary>Answer</summary>

The type (`struct User { ... }`) is the design: it lists the fields and their types. An instance is one concrete value made from that design, for example one particular user with real data.

</details>

2. Can you make only one field of a struct instance mutable?

<details>
<summary>Answer</summary>

No. Mutability belongs to the binding: `let mut user` makes all fields changeable, `let user` makes none changeable.

</details>

3. What does `..user1` do in `User { email: new_email, ..user1 }`, and what can go wrong afterwards?

<details>
<summary>Answer</summary>

It fills all fields not written explicitly with the values from `user1`. Each field is moved or copied. Fields of a non-`Copy` type (such as `String`) are moved, so `user1` can no longer be used as a whole, and you cannot read the moved fields.

</details>

4. In the exercise code, `let custom = Config { name: String::from("custom"), ..base };` compiles and `base` can still be printed, if `base` has the fields `name: String`, `verbose: bool`, `retries: u32`. Why?

<details>
<summary>Answer</summary>

`name` is given explicitly, so it is not taken from `base`. The remaining fields (`verbose` and `retries`) are `Copy`, so they are copied, not moved. Nothing was moved out of `base`, and it stays fully usable. (If a `String` field were taken from `base`, the move would make `base` unusable.)

</details>

5. Why do we use `String` instead of `&str` in struct fields for now?

<details>
<summary>Answer</summary>

A struct must own what it stores, or else the compiler must be told how long the borrowed data lives (a lifetime, `E0106`). `String` is owned, so no lifetime is needed. Lifetimes come in Lesson 05.3.

</details>

6. What is a newtype and why is it useful?

<details>
<summary>Answer</summary>

A newtype is a tuple struct with one field, such as `struct UserId(u64);`. It makes a new type from an existing one, so the compiler can tell values with different meanings apart (a `UserId` is not an `OrderId`), at no run-time cost.

</details>

7. Which two derives do you need to compare structs with `assert_eq!`?

<details>
<summary>Answer</summary>

`PartialEq` (so `==` works) and `Debug` (so the failure message can show both values).

</details>

8. How big is `struct Marker;` and why does such a type exist?

<details>
<summary>Answer</summary>

0 bytes. It carries no data but is still a distinct type, which is useful when you only need a type to attach behaviour or meaning to.

</details>

## Summary

- A **struct** groups named fields of different types into one type. Create instances with `Name { field: value, ... }` and read fields with a dot.
- Mutability belongs to the whole instance: `let mut` or nothing.
- **Field init shorthand** (`User { username, email }`) and **struct update syntax** (`..other`) reduce repetition; the update syntax moves non-`Copy` fields.
- **Tuple structs**, **newtypes** and **unit-like structs** are lightweight forms; newtypes give different meanings to the same raw type.
- `#[derive(Debug)]` with `{:?}`, `{:#?}` and `dbg!` is your first debugging tool; `#[derive(PartialEq)]` makes `==` and `assert_eq!` work.
- A struct owns its fields; use owned types in fields for now, pass `&Struct` / `&mut Struct` to functions, and derive `Clone, Copy` only for small plain-data types.
- In memory a struct is its fields one after another; Rust may reorder them and add padding, and `#[repr(C)]` fixes the C order.

## Sources and further reading

- [The Rust Programming Language — 5.1 Defining and Instantiating Structs](https://doc.rust-lang.org/book/ch05-01-defining-structs.html) — the official book chapter on this topic.
- [The Rust Programming Language — 5.2 An Example Program Using Structs](https://doc.rust-lang.org/book/ch05-02-example-structs.html) — a step-by-step rectangle example that adds `Debug` and `dbg!`.
- [Rust by Example — Structures](https://doc.rust-lang.org/rust-by-example/custom_types/structs.html) — short runnable examples of all three kinds of struct.
- [The Rust Reference — Struct types](https://doc.rust-lang.org/reference/types/struct.html) — the exact rules, including layout and `repr`.
- [Rust API Guidelines — Naming](https://rust-lang.github.io/api-guidelines/naming.html) — the official naming conventions used in this course.
- [std::mem::size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) — documentation of the function used in section 8.

---

[🏠 Start](../START-HERE.md) · [← Phase 04 — Structs, enums, collections and errors](README.md) · [📋 Phase 04](README.md) · [04.2 Methods and associated functions →](02-methods-and-associated-functions.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
