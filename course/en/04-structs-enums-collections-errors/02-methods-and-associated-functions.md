<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.1 Structs](01-structs.md) · [📋 Phase 04](README.md) · [04.3 Enums →](03-enums.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/02-methods-and-associated-functions.md) <!-- nav -->

# Lesson 04.2 — Methods and associated functions

> **Phase 04 — Structs, enums, collections and errors** · Lesson 2 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- How to attach functions to a type with an `impl` block.
- The difference between a **method** (has `self`) and an **associated function** (no `self`), and how to call each.
- The three forms of `self`: `&self`, `&mut self` and `self`, and how each one relates to ownership and borrowing.
- Useful patterns: `new` constructors, builder-style chaining, associated constants, `Default`, and printing with `Display`.
- Naming conventions for methods (`as_`, `to_`, `into_`, `is_`, `with_`).

## Before you start

- You finished [Lesson 04.1 — Structs](01-structs.md). You can define a struct, create instances and derive `Debug`.
- You remember the three ways a function can receive a value — by value, by `&` reference, by `&mut` reference ([Lesson 03.4 — References and borrowing](../03-ownership-and-borrowing/04-references-and-borrowing.md) and [Lesson 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)).

## New words

| Word | Meaning |
|---|---|
| **impl block** (implementation block) | A block `impl TypeName { ... }` that holds the functions belonging to a type. |
| **method** | A function inside an `impl` block whose first parameter is `self`. You call it with a dot: `value.method()`. |
| **associated function** | A function inside an `impl` block without `self`. You call it with `::`: `Type::function()`. |
| **receiver** | The value a method is called on — the thing to the left of the dot. Inside the method it is called `self`. |
| **`Self`** | A short name for "the type this `impl` block is for". Inside `impl Rectangle`, `Self` means `Rectangle`. |
| **`self`** | The receiver value inside a method. |
| **constructor** | A function that builds a new instance. In Rust it is only a convention: an associated function, usually named `new`. |
| **associated constant** | A `const` defined inside an `impl` block. It belongs to the type: `Type::NAME`. |
| **builder pattern** | A style where small methods each set one option and return the value, so calls can be chained: `Config::new().with_port(80)`. |
| **method chaining** | Calling several methods in a row, each on the result of the previous one. |
| **auto-referencing** | The compiler automatically adds `&`, `&mut` or `*` to the receiver so that `value.method()` matches the method's `self` form. |
| **Display** | The trait behind `{}`. You write it by hand to say how a value looks for users. |
| **Default** | A trait with one function, `default()`, that returns a sensible "empty" value of a type. |
| **trait** | A named set of abilities a type can have (explained in [Lesson 05.2 — Traits](../05-rust-intermediate/02-traits.md)). In this lesson we only copy small recipes for `Display` and `Default`. |
| **API** (Application Programming Interface) | The set of functions and types that a piece of code offers to others. The methods of a type are its API. |

## 1. From functions to methods

In Lesson 04.1 you wrote `fn area(rect: &Rectangle) -> u32` next to the struct. It works, but there is no link between the struct and the function except the parameter. With ten functions about rectangles, they scatter through the file.

Rust lets you put such functions *inside* an **`impl` block** of the type. Then `area` is "something a rectangle can do". The call changes from `area(&rect)` to `rect.area()`. Think of it as: the struct is a noun, the methods are its verbs.

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // associated function: no self, called with ::
    fn new(width: u32, height: u32) -> Self {
        Self { width, height }
    }

    fn square(size: u32) -> Self {
        Self::new(size, size)
    }

    // methods: first parameter is self
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}

fn main() {
    let big = Rectangle::new(30, 50);
    let small = Rectangle::square(10);

    println!("area of big = {}", big.area());
    println!("big can hold small? {}", big.can_hold(&small));
    println!("small can hold big? {}", small.can_hold(&big));
    println!("{:?}", small);
}
```

**What happens:**

1. `impl Rectangle { ... }` says: "everything in these braces belongs to the type `Rectangle`".
2. Inside the block, `Self` is another name for `Rectangle`. `-> Self` means "returns a `Rectangle`", and `Self { width, height }` builds one (with the field init shorthand from Lesson 04.1). If you rename the struct later, you change fewer lines.
3. `new` and `square` have **no `self` parameter**, so they are *associated functions*. They are called through the type name with two colons: `Rectangle::new(30, 50)`. They are the usual way to build values.
4. `square` calls another associated function with `Self::new(size, size)`.
5. `area` and `can_hold` have `&self` as the first parameter. They are *methods*. `big.area()` calls `area` with `big` as `self`.
6. `&self` is short for `self: &Self`, which means `self: &Rectangle`: the method borrows the rectangle for reading. `self.width` reads a field of it.
7. `can_hold(&self, other: &Rectangle)` receives a second rectangle. The call `big.can_hold(&small)` passes `&small` as `other`.

```text
area of big = 1500
big can hold small? true
small can hold big? false
Rectangle { width: 10, height: 10 }
```

> 📏 **Convention:** `new` is not a keyword. It is just a name that every Rust programmer expects for "the main constructor". When a type has an obvious way to build it, provide `Type::new(...)`.

## 2. The three forms of `self`

The first parameter of a method decides what the method may do with the value. It is the same three choices you know from ownership:

| Form | Meaning | The caller afterwards | Use it when |
|---|---|---|---|
| `&self` | borrows the value, read only | still owns the value, can keep using it | the method only looks (getters, `area`, `is_empty`) |
| `&mut self` | borrows the value, can change it | still owns it; value may be changed | the method modifies the value (`deposit`, `push`) |
| `self` | takes ownership; the value is moved in | can no longer use the value | the method consumes the value (`close`, `into_string`) |

A complete example with all three:

```rust
#[derive(Debug)]
struct BankAccount {
    owner: String,
    balance_cents: u64,
}

impl BankAccount {
    fn new(owner: &str) -> Self {
        Self {
            owner: String::from(owner),
            balance_cents: 0,
        }
    }

    // &self: read only
    fn balance_cents(&self) -> u64 {
        self.balance_cents
    }

    // &mut self: may change the account
    fn deposit(&mut self, cents: u64) {
        self.balance_cents += cents;
    }

    // returns false when there is not enough money
    fn withdraw(&mut self, cents: u64) -> bool {
        if cents > self.balance_cents {
            false
        } else {
            self.balance_cents -= cents;
            true
        }
    }

    // self: takes ownership; the account is consumed
    fn close(self) -> String {
        format!("{} closed the account with {} cents", self.owner, self.balance_cents)
    }
}

fn main() {
    let mut account = BankAccount::new("alice");
    account.deposit(10_000);
    println!("withdraw 3000: {}", account.withdraw(3_000));
    println!("withdraw 9000: {}", account.withdraw(9_000));
    println!("balance: {}", account.balance_cents());
    let receipt = account.close();
    println!("{receipt}");
}
```

**What happens:**

1. `BankAccount::new("alice")` builds an account with zero balance. The parameter is `&str`; `String::from(owner)` makes an owned copy for the struct field.
2. `account` is declared `let mut`, because we call methods with `&mut self`.
3. `deposit` adds `10_000` cents (the underscore is only a reading aid). `withdraw(3_000)` succeeds and returns `true`; the balance becomes 7000.
4. `withdraw(9_000)` is more than the balance, so nothing changes and the method returns `false`. (A better design would return a `Result` that explains *why* it failed. You will learn that in [Lesson 04.10 — Error handling](10-error-handling.md).)
5. `balance_cents()` has `&self` and the same name as the field. In Rust, a getter is named like the field, without a `get_` prefix.
6. `close(self)` takes ownership. It builds a text with `format!` (like `println!`, but returns a `String`) and returns it. After this call the account is gone: it was moved into `close` and dropped at the end of it.

```text
withdraw 3000: true
withdraw 9000: false
balance: 7000
alice closed the account with 7000 cents
```

### Errors you will meet

**A method that needs `&mut self` but declares `&self`:**

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn increment(&self) {
        self.count += 1;
    }
}
```

```text
error[E0594]: cannot assign to `self.count`, which is behind a `&` reference
 --> src/main.rs:7:9
  |
7 |         self.count += 1;
  |         ^^^^^^^^^^^^^^^ `self` is a `&` reference, so it cannot be written to
  |
help: consider changing this to be a mutable reference
  |
6 |     fn increment(&mut self) {
  |                   +++
```

Meaning: through a `&` reference you may only read. Fix: `&mut self`.

**Calling a `&mut self` method on a variable that is not `mut`:**

```rust
let c = Counter { count: 0 };
c.increment();
```

```text
error[E0596]: cannot borrow `c` as mutable, as it is not declared as mutable
  --> src/main.rs:13:5
   |
13 |     c.increment();
   |     ^ cannot borrow as mutable
   |
help: consider changing this to be mutable
   |
12 |     let mut c = Counter { count: 0 };
   |         +++
```

The call `c.increment()` secretly takes `&mut c`. That needs a mutable variable: `let mut c`.

**Using a value after a `self` method consumed it:**

```rust
let account = BankAccount { balance_cents: 500 };
let money = account.close();
println!("{}", account.balance_cents);
```

```text
error[E0382]: borrow of moved value: `account`
  --> src/main.rs:15:20
   |
12 |     let account = BankAccount { balance_cents: 500 };
   |         ------- move occurs because `account` has type `BankAccount`, which does not implement the `Copy` trait
13 |     let money = account.close();
   |                         ------- `account` moved due to this method call
14 |     println!("{money}");
15 |     println!("{}", account.balance_cents);
   |                    ^^^^^^^^^^^^^^^^^^^^^ value borrowed here after move
   |
note: `BankAccount::close` takes ownership of the receiver `self`, which moves `account`
```

The compiler even tells you which method moved the value. This is the ownership rule from Phase 03 applied to method calls.

> 💡 **Tip:** when you are unsure which form to use, start with `&self`. Change to `&mut self` when the compiler says you must write, and to `self` only when the method really ends the life of the value.

## 3. Associated functions: no `self`

A function in an `impl` block without `self` is an **associated function**. It does not work on an existing instance; it is "about the type". Constructors are the main use. You call it with `::`:

```text
let s = Rectangle::square(5);     // right: associated function, with ::
let s = r.square(5);              // wrong: it has no self, so no dot
```

The wrong version gives a helpful error:

```text
error[E0599]: no method named `square` found for struct `Rectangle` in the current scope
  --> src/main.rs:14:15
   |
 1 | struct Rectangle {
   | ---------------- method `square` not found for this struct
...
14 |     let s = r.square(5);
   |               ^^^^^^ this is an associated function, not a method
   |
   = note: found the following associated functions; to be used as methods, functions must have a `self` parameter
help: use associated function syntax instead
   |
14 -     let s = r.square(5);
14 +     let s = Rectangle::square(5);
   |
```

The opposite mistake — calling a method through the type without passing the receiver, `Rectangle::area()` — gives `error[E0061]: this function takes 1 argument but 0 arguments were supplied`, with the note `argument #1 of type &Rectangle is missing`. That is a clue to what a method really is: `area(&self)` is a normal function whose first argument is the rectangle.

## 4. How the dot call works

You may wonder: `area` expects `&Rectangle`, yet we wrote `r.area()` where `r` is a `Rectangle`. Who added the `&`? The compiler did. This is **auto-referencing**: when you write `value.method()`, the compiler looks at the method's `self` form and inserts `&`, `&mut` or `*` for you.

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // a getter has the same name as the field, no "get_" prefix
    fn width(&self) -> u32 {
        self.width
    }
}

fn main() {
    let r = Rectangle { width: 6, height: 7 };
    let a = r.area();
    let b = Rectangle::area(&r);
    let c = (&r).area();
    println!("{a} {b} {c}");
    println!("field: {}, method: {}", r.width, r.width());
}
```

**What happens:**

1. `r.area()` is the short form. The compiler turns it into `Rectangle::area(&r)`.
2. `Rectangle::area(&r)` is the long form: calling the method as a plain associated function and passing the receiver yourself.
3. `(&r).area()` has the reference written by hand; the compiler accepts it as well.
4. All three give 42.
5. `r.width` (no parentheses) is the field. `r.width()` (with parentheses) is the method of the same name. Rust tells them apart by the parentheses.

```text
42 42 42
field: 6, method: 6
```

Thanks to auto-referencing you never need a special operator for "call a method through a reference": the same dot works for values and for references.

## 5. More tools in `impl` blocks

### Several `impl` blocks and associated constants

A type may have many `impl` blocks. This is useful to group methods. Inside a block you can also define **associated constants**:

```rust
#[derive(Debug, Default)]
struct ServerConfig {
    host: String,
    port: u16,
    verbose: bool,
}

impl ServerConfig {
    const DEFAULT_PORT: u16 = 8080;

    fn new(host: &str) -> Self {
        Self {
            host: String::from(host),
            port: Self::DEFAULT_PORT,
            verbose: false,
        }
    }

    fn with_port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    fn with_verbose(mut self, verbose: bool) -> Self {
        self.verbose = verbose;
        self
    }
}

// a second impl block is allowed
impl ServerConfig {
    fn address(&self) -> String {
        format!("{}:{}", self.host, self.port)
    }
}

fn main() {
    let plain = ServerConfig::new("localhost");
    println!("{}", plain.address());

    let custom = ServerConfig::new("example.org").with_port(9000).with_verbose(true);
    println!("{:?}", custom);

    let blank = ServerConfig::default();
    println!("{:?}", blank);

    let partly = ServerConfig {
        host: String::from("db"),
        ..Default::default()
    };
    println!("{:?}", partly);
}
```

**What happens:**

1. `const DEFAULT_PORT: u16 = 8080;` is a constant that belongs to the type. Inside the block you reach it as `Self::DEFAULT_PORT`; outside it is `ServerConfig::DEFAULT_PORT`. Constants are written in `SCREAMING_SNAKE_CASE`. A named constant is better than repeating the "magic number" `8080` in many places.
2. `with_port(mut self, port: u16) -> Self` takes `self` **by value** and marks the parameter `mut`. That `mut` only allows changing the local copy of the value that was moved in. The method sets the field and returns `self`.
3. Because each `with_...` method consumes the value and returns it back, calls can be **chained**: `ServerConfig::new("example.org").with_port(9000).with_verbose(true)`. Each dot works on the result of the previous call. This is the *builder pattern*, and it is how Rust gets "optional arguments with names".
4. `address(&self)` lives in a second `impl` block. The compiler treats both blocks as one.
5. `#[derive(Default)]` asks the compiler to write `ServerConfig::default()`, which fills each field with its own default: an empty `String`, `0` for numbers, `false` for `bool`.
6. `..Default::default()` is the struct update syntax from Lesson 04.1, using a fresh default value as the base. We set only `host` and take the rest from the default.

```text
localhost:8080
ServerConfig { host: "example.org", port: 9000, verbose: true }
ServerConfig { host: "", port: 0, verbose: false }
ServerConfig { host: "db", port: 0, verbose: false }
```

> 📏 **Convention:** use `Default` when "all fields at their default" is a meaningful value. Do not derive it when the default would be a broken or confusing state (for example a port number of 0 is probably not useful, which is why `new` here sets 8080).

### Printing for users: `Display`

`{:?}` (`Debug`) is for developers. For users, a type implements **`Display`** and then works with `{}`. You write it by hand because only you know how the value should look. Treat the following as a recipe for now; the full explanation of traits comes in [Lesson 05.2](../05-rust-intermediate/02-traits.md).

```rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 3, y: -4 };
    println!("point is {}", p);
    let text = p.to_string();
    println!("{} has {} characters", text, text.len());
}
```

**What happens:**

1. `use std::fmt;` lets us write `fmt::Display` instead of `std::fmt::Display`.
2. `impl fmt::Display for Point` means "implement the ability `Display` for the type `Point`". The ability requires one method named `fmt`.
3. `fmt` receives `&self` (the point) and `f`, a `Formatter`, which is the place where the text is written.
4. `write!(f, "({}, {})", self.x, self.y)` is like `println!`, but writes into `f`. Its result (`fmt::Result`) is returned from the function.
5. Once `Display` exists, `{}` works, and you also get `.to_string()` for free. `text.len()` counts bytes; the text `(3, -4)` has 7.

```text
point is (3, -4)
(3, -4) has 7 characters
```

## 6. Naming conventions for methods

Rust programmers read a method name as a promise about cost and ownership. The official API Guidelines use these prefixes:

| Prefix | Promise | Takes | Example |
|---|---|---|---|
| `new` | builds a value | — | `Rectangle::new(3, 4)` |
| `with_` | builder: returns the value with one option set | `self` | `config.with_port(80)` |
| `is_` / `has_` | a yes/no question | `&self` | `name.is_empty()` |
| `as_` | a cheap view of the data, borrowed | `&self` | `name.as_str()` |
| `to_` | makes a new value, may cost time or memory; keeps the original | `&self` | `name.to_uppercase()` |
| `into_` | converts and **consumes** the original | `self` | `name.into_string()` |
| *(field name)* | getter | `&self` | `rect.width()` |

All of them in one small type:

```rust
struct Name {
    text: String,
}

impl Name {
    fn new(text: &str) -> Self {
        Self { text: text.to_string() }
    }
    // as_*: cheap, borrows
    fn as_str(&self) -> &str {
        &self.text
    }
    // to_*: makes a new value (costs something), keeps self
    fn to_uppercase(&self) -> String {
        self.text.to_uppercase()
    }
    // into_*: consumes self
    fn into_string(self) -> String {
        self.text
    }
    // is_*: yes/no question
    fn is_empty(&self) -> bool {
        self.text.is_empty()
    }
}

fn main() {
    let n = Name::new("ferris");
    println!("{} {} {}", n.as_str(), n.to_uppercase(), n.is_empty());
    let s: String = n.into_string();
    println!("{s}");
}
```

**What happens:**

1. `as_str(&self) -> &str` returns a borrowed view of the text. Nothing is copied.
2. `to_uppercase(&self) -> String` builds a new `String` — memory is allocated, so the name starts with `to_`.
3. `into_string(self) -> String` gives away the inner `String` and consumes the `Name`. Because it takes `self`, no copy is needed — the text is simply moved out.
4. After `n.into_string()` the variable `n` is moved. We only use `s` afterwards.

```text
ferris FERRIS false
ferris
```

> ⚡ **Performance:** the prefix tells you the price. `as_` is free, `to_` allocates, `into_` moves. Prefer `&self` (borrow) methods for reading, because they never copy the data.

## 7. Methods and design

> 📏 **Convention:** when to use a method and when a free function:
> - If the function's main job is to work on one value of your type, make it a method.
> - If it treats several values equally (such as `distance(a, b)` between two points), a free function or an associated function (`Point::distance(a, b)`) can read better.
> - Keep methods small and focused (the *single responsibility* idea: one method, one job). A method that is longer than a screen usually does too much.

> 🔒 **Security:** in this lesson every field is visible to all code, so anyone can write `account.balance_cents = 999_999`. Methods like `deposit` and `withdraw` are meant to guard the rules (a balance never goes below zero), but this works only when outsiders cannot touch the fields. In [Lesson 04.6 — Modules, crates and packages](06-modules-crates-and-packages.md) you will learn to make fields private, so that the methods are the only way in. This idea is called *encapsulation*, and it is the main way to keep data valid.

## 8. Testing methods

> 🧪 **Testing:** test a type through its public methods, like a user would. Each test builds a value, calls methods, and checks the visible result. Test the "bad" case too: what happens when a withdrawal is too large?

```rust
#[derive(Debug)]
struct BankAccount {
    balance_cents: u64,
}

impl BankAccount {
    fn new() -> Self {
        Self { balance_cents: 0 }
    }
    fn deposit(&mut self, cents: u64) {
        self.balance_cents += cents;
    }
    fn withdraw(&mut self, cents: u64) -> bool {
        if cents > self.balance_cents {
            false
        } else {
            self.balance_cents -= cents;
            true
        }
    }
}

fn main() {
    let mut a = BankAccount::new();
    a.deposit(100);
    println!("{}", a.withdraw(40));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn deposit_adds_money() {
        let mut a = BankAccount::new();
        a.deposit(500);
        assert_eq!(a.balance_cents, 500);
    }

    #[test]
    fn cannot_withdraw_more_than_balance() {
        let mut a = BankAccount::new();
        a.deposit(100);
        assert!(!a.withdraw(101));
        assert_eq!(a.balance_cents, 100); // unchanged
        assert!(a.withdraw(100));
        assert_eq!(a.balance_cents, 0);
    }
}
```

**What happens:**

1. `main` is only a small demo; `cargo run` prints `true`.
2. `cargo test` runs the two tests. Each creates its own account, so tests do not depend on each other.
3. In the second test the failed withdrawal must leave the balance unchanged (`100`) — this checks that the method does not half-do its work.

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Common mistakes

> ⚠️ **Common mistake:** `&self` where you need `&mut self` (`E0594`), or a non-`mut` variable when calling a `&mut self` method (`E0596`). See section 2.

> ⚠️ **Common mistake:** using a value after calling a method that takes `self` (`E0382`). Look at the method's signature: a bare `self` means "consumes". If you still need the value, make the method take `&self` or call `.clone()` first.

> ⚠️ **Common mistake:** calling an associated function with a dot (`E0599`) or a method without a receiver (`E0061`). Remember: no `self` means `Type::function()`, with `self` means `value.method()`.

> ⚠️ **Common mistake:** passing the same value twice as `&mut`:

```rust
BankAccount::transfer(&mut a, &mut a, 10);
```

```text
error[E0499]: cannot borrow `a` as mutable more than once at a time
  --> src/main.rs:14:35
   |
14 |     BankAccount::transfer(&mut a, &mut a, 10);
   |     --------------------- ------  ^^^^^^ second mutable borrow occurs here
   |     |                     |
   |     |                     first mutable borrow occurs here
   |     first borrow later used by call
```

This is the borrowing rule "one `&mut` at a time" from [Lesson 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md). It protects you here: transferring money from an account to itself would be a bug.

## Try it yourself

1. Create a project:

   ```text
   cd ~/projects/rust
   cargo new code/p04-methods
   cd code/p04-methods
   ```

2. Put this into `src/main.rs`:

   ```rust
   #[derive(Debug)]
   struct Task {
       title: String,
       done: bool,
   }

   impl Task {
       fn new(title: &str) -> Self {
           Self {
               title: String::from(title),
               done: false,
           }
       }

       fn complete(&mut self) {
           self.done = true;
       }

       fn label(&self) -> String {
           let mark = if self.done { "x" } else { " " };
           format!("[{}] {}", mark, self.title)
       }
   }

   fn main() {
       let mut task = Task::new("write lesson");
       println!("{}", task.label());
       task.complete();
       println!("{}", task.label());
   }
   ```

3. Run `cargo run`. Expected output:

   ```text
   [ ] write lesson
   [x] write lesson
   ```

4. Extend it: add a method `fn rename(&mut self, new_title: &str)`, add a `#[cfg(test)]` module with a test that `complete` sets `done`, and run `cargo test`.
5. Commit:

   ```text
   cd ~/projects/rust
   git add code/p04-methods
   git commit -m "feat(p04): task type with methods and tests"
   ```

## Exercises

### Exercise 1 — A circle (easy)

Write `struct Circle { radius: f64 }` with an associated function `new(radius: f64) -> Self` and methods `area(&self) -> f64` and `circumference(&self) -> f64`. Use `std::f64::consts::PI`. Write a test for the unit circle (radius 1.0) using a tolerance.

<details>
<summary>Hint</summary>

`use std::f64::consts::PI;` at the top. Area is `PI * r * r`; circumference is `2.0 * PI * r`. Compare floats with `(a - b).abs() < 1e-9`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::f64::consts::PI;

struct Circle {
    radius: f64,
}

impl Circle {
    fn new(radius: f64) -> Self {
        Self { radius }
    }

    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }

    fn circumference(&self) -> f64 {
        2.0 * PI * self.radius
    }
}

fn main() {
    let c = Circle::new(3.0);
    println!("area = {:.3}", c.area());
    println!("circumference = {:.3}", c.circumference());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn unit_circle() {
        let c = Circle::new(1.0);
        assert!((c.area() - PI).abs() < 1e-9);
        assert!((c.circumference() - 2.0 * PI).abs() < 1e-9);
    }
}
```

Output of `cargo run`:

```text
area = 28.274
circumference = 18.850
```

Why it works: both methods only read the radius, so `&self` is enough. `{:.3}` prints three digits after the decimal point.

</details>

### Exercise 2 — A counter (medium)

Write `struct Counter { count: u32 }` with `new()`, `increment(&mut self)`, `reset(&mut self)` and `value(&self) -> u32`. Write tests: a new counter is 0; after two increments it is 2; after `reset` it is 0 again.

<details>
<summary>Hint</summary>

Only `value` can use `&self`. The other two change the counter, so they need `&mut self`, and the variable in the test must be `let mut`.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug)]
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Self {
        Self { count: 0 }
    }

    fn increment(&mut self) {
        self.count += 1;
    }

    fn reset(&mut self) {
        self.count = 0;
    }

    fn value(&self) -> u32 {
        self.count
    }
}

fn main() {
    let mut c = Counter::new();
    c.increment();
    c.increment();
    println!("{}", c.value());
    c.reset();
    println!("{}", c.value());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn starts_at_zero() {
        assert_eq!(Counter::new().value(), 0);
    }

    #[test]
    fn increment_and_reset() {
        let mut c = Counter::new();
        c.increment();
        c.increment();
        assert_eq!(c.value(), 2);
        c.reset();
        assert_eq!(c.value(), 0);
    }
}
```

`cargo run` prints `2` and `0`; `cargo test` reports 2 passed.

Why it works: methods that change the data take `&mut self`; methods that only read take `&self`.

</details>

### Exercise 3 — Fix the compile errors (medium)

This program has three mistakes. Find them using the compiler messages and fix them. Expected output after the fix: `77 32`.

```rust
struct Temperature {
    celsius: f64,
}

impl Temperature {
    fn from_celsius(celsius: f64) -> Self {
        Self { celsius }
    }

    fn warm_up(&self, degrees: f64) {
        self.celsius += degrees;
    }

    fn fahrenheit(&self) -> f64 {
        self.celsius * 9.0 / 5.0 + 32.0
    }
}

fn main() {
    let t = Temperature::from_celsius(20.0);
    t.warm_up(5.0);
    let other = t.from_celsius(0.0);
    println!("{} {}", t.fahrenheit(), other.fahrenheit());
}
```

<details>
<summary>Hint</summary>

The first run reports two errors: `E0599` (`from_celsius` is an associated function, so it cannot be called with a dot) and `E0594` (`warm_up` writes through `&self`). After fixing those, the compiler will also ask you to make `t` mutable.

</details>

<details>
<summary>Answer</summary>

```rust
struct Temperature {
    celsius: f64,
}

impl Temperature {
    fn from_celsius(celsius: f64) -> Self {
        Self { celsius }
    }

    fn warm_up(&mut self, degrees: f64) {
        self.celsius += degrees;
    }

    fn fahrenheit(&self) -> f64 {
        self.celsius * 9.0 / 5.0 + 32.0
    }
}

fn main() {
    let mut t = Temperature::from_celsius(20.0);
    t.warm_up(5.0);
    let other = Temperature::from_celsius(0.0);
    println!("{} {}", t.fahrenheit(), other.fahrenheit());
}
```

The three fixes: (1) `warm_up` takes `&mut self`; (2) `let mut t`; (3) `Temperature::from_celsius(0.0)` instead of `t.from_celsius(0.0)`. The program prints `77 32` (25 °C is 77 °F, 0 °C is 32 °F).

Why it works: each fix matches a rule from this lesson: writing needs `&mut self`, calling a `&mut self` method needs a `mut` variable, and an associated function is called through the type.

</details>

### Exercise 4 — Transfer between accounts (hard)

Take a `BankAccount { balance_cents: u64 }` with `new`, `deposit`, and `withdraw` (returns `bool`). Add an associated function `transfer(from: &mut BankAccount, to: &mut BankAccount, cents: u64) -> bool` that moves money only if the sender has enough. Write two tests: a successful transfer, and a failed one that must leave **both** balances unchanged.

<details>
<summary>Hint</summary>

Reuse the other methods: `if from.withdraw(cents) { to.deposit(cents); true } else { false }`. In `main` you call it as `BankAccount::transfer(&mut a, &mut b, 400)` with two **different** accounts.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug)]
struct BankAccount {
    balance_cents: u64,
}

impl BankAccount {
    fn new(balance_cents: u64) -> Self {
        Self { balance_cents }
    }

    fn withdraw(&mut self, cents: u64) -> bool {
        if cents > self.balance_cents {
            false
        } else {
            self.balance_cents -= cents;
            true
        }
    }

    fn deposit(&mut self, cents: u64) {
        self.balance_cents += cents;
    }

    fn transfer(from: &mut BankAccount, to: &mut BankAccount, cents: u64) -> bool {
        if from.withdraw(cents) {
            to.deposit(cents);
            true
        } else {
            false
        }
    }
}

fn main() {
    let mut a = BankAccount::new(1_000);
    let mut b = BankAccount::new(0);
    println!("{}", BankAccount::transfer(&mut a, &mut b, 400));
    println!("{}", BankAccount::transfer(&mut a, &mut b, 700));
    println!("{:?} {:?}", a, b);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn transfer_moves_money() {
        let mut a = BankAccount::new(1_000);
        let mut b = BankAccount::new(0);
        assert!(BankAccount::transfer(&mut a, &mut b, 400));
        assert_eq!(a.balance_cents, 600);
        assert_eq!(b.balance_cents, 400);
    }

    #[test]
    fn failed_transfer_changes_nothing() {
        let mut a = BankAccount::new(100);
        let mut b = BankAccount::new(5);
        assert!(!BankAccount::transfer(&mut a, &mut b, 101));
        assert_eq!(a.balance_cents, 100);
        assert_eq!(b.balance_cents, 5);
    }
}
```

Output of `cargo run`:

```text
true
false
BankAccount { balance_cents: 600 } BankAccount { balance_cents: 400 }
```

Why it works: `transfer` has no receiver because two accounts play equal roles, so an associated function reads better than a method. The compiler guarantees `from` and `to` are different accounts: passing the same variable twice is rejected with `E0499`. Because `u64` cannot go below zero, the check in `withdraw` also protects against an arithmetic panic (subtracting more than you have).

</details>

## Quiz

1. What is the difference between a method and an associated function?

<details>
<summary>Answer</summary>

A method has `self` (in some form) as its first parameter and is called with a dot on a value: `rect.area()`. An associated function has no `self` and is called through the type with `::`: `Rectangle::new(3, 4)`.

</details>

2. What does `Self` mean inside `impl Rectangle`?

<details>
<summary>Answer</summary>

It is a short name for the type of the `impl` block, here `Rectangle`. `-> Self` returns a `Rectangle`.

</details>

3. Which form of `self` would you pick for a method that changes a field? For one that destroys the value?

<details>
<summary>Answer</summary>

`&mut self` for changing a field (the value stays with the caller). `self` (by value) for consuming it — the caller can no longer use the value afterwards.

</details>

4. Why can you write `rect.area()` when `area` expects `&Rectangle`?

<details>
<summary>Answer</summary>

Because of auto-referencing: the compiler adds the needed `&`, `&mut` or `*` to the receiver based on the method's signature. `rect.area()` means `Rectangle::area(&rect)`.

</details>

5. What do the prefixes `as_`, `to_` and `into_` promise?

<details>
<summary>Answer</summary>

`as_`: a cheap borrowed view (takes `&self`). `to_`: builds a new value, which may cost memory or time, and keeps the original (takes `&self`). `into_`: converts and consumes the original (takes `self`).

</details>

6. What does `ServerConfig::new("x").with_port(1).with_verbose(true)` rely on?

<details>
<summary>Answer</summary>

Each `with_` method takes `self` by value and returns `Self`, so the result of one call can be the receiver of the next. This is the builder pattern with method chaining.

</details>

7. Why does `Counter::new()` need no `mut`, but `c.increment()` (with `&mut self`) needs `let mut c`?

<details>
<summary>Answer</summary>

`Counter::new()` just returns a new value; nothing is borrowed mutably. `c.increment()` implicitly borrows `c` as `&mut`, which requires the variable to be declared mutable.

</details>

8. How do you make `println!("{}", my_value)` work for your own type?

<details>
<summary>Answer</summary>

Implement the `Display` trait for the type: `impl fmt::Display for MyType { fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result { write!(f, "...") } }`. `{:?}` needs `Debug` instead, which can be derived.

</details>

## Summary

- An `impl` block attaches functions to a type. Inside it, `Self` is the type's own name.
- A **method** has `self` first and is called with a dot; an **associated function** has no `self` and is called with `Type::name()`.
- `&self` reads, `&mut self` changes, `self` consumes. These are the ownership and borrowing rules applied to the receiver.
- `new` is a convention for constructors; `with_` methods taking and returning `self` give builder-style chaining; `#[derive(Default)]` and `..Default::default()` reduce boilerplate.
- Associated constants (`Self::DEFAULT_PORT`) replace magic numbers.
- `impl fmt::Display` makes `{}` and `.to_string()` work; `Debug` stays derived for developers.
- Name methods by their promise: `as_` (cheap view), `to_` (new value), `into_` (consume), `is_` (question), getters named like the field.
- Test types through their methods, including the failure cases.

## Sources and further reading

- [The Rust Programming Language — 5.3 Method Syntax](https://doc.rust-lang.org/book/ch05-03-method-syntax.html) — the official book chapter on this topic.
- [Rust by Example — Associated functions & Methods](https://doc.rust-lang.org/rust-by-example/fn/methods.html) — short runnable examples.
- [The Rust Reference — Associated Items](https://doc.rust-lang.org/reference/items/associated-items.html) — the precise rules for associated functions, methods and constants.
- [Rust API Guidelines — Naming (conversions)](https://rust-lang.github.io/api-guidelines/naming.html) — the official rules behind `as_`, `to_` and `into_`.
- [std::fmt::Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) — documentation of the trait used in section 5.
- [std::default::Default](https://doc.rust-lang.org/std/default/trait.Default.html) — documentation of the `Default` trait.

---

[🏠 Start](../START-HERE.md) · [← 04.1 Structs](01-structs.md) · [📋 Phase 04](README.md) · [04.3 Enums →](03-enums.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
