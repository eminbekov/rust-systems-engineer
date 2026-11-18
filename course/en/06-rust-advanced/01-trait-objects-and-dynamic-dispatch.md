<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 06 — Rust advanced](README.md) · [📋 Phase 06](README.md) · [06.2 Advanced traits and types →](02-advanced-traits-and-types.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md) <!-- nav -->

# Lesson 06.1 — Trait objects and dynamic dispatch

> **Phase 06 — Rust advanced** · Lesson 1 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- Why a `Vec` of generics cannot hold different types, and how `dyn Trait` solves it.
- What a trait object is, and where it can live: `Box<dyn Trait>`, `&dyn Trait`, `&mut dyn Trait`.
- How a trait object works inside: a fat pointer and a vtable.
- The difference between static dispatch and dynamic dispatch, and how to choose.
- The rules a trait must follow to be used as `dyn Trait` (dyn compatibility).
- How to read the compiler errors you meet when you use trait objects.

## Before you start

- You finished [Lesson 05.14 — Checkpoint](../05-rust-intermediate/14-checkpoint.md).
- You know generics and trait bounds from [Lesson 05.1 — Generics](../05-rust-intermediate/01-generics.md) and traits from [Lesson 05.2 — Traits](../05-rust-intermediate/02-traits.md).
- You know `Box` from [Lesson 05.9 — Smart pointers](../05-rust-intermediate/09-smart-pointers.md) and closures from [Lesson 05.4 — Closures](../05-rust-intermediate/04-closures.md).
- Create a practice project inside your learning repository: `cargo new code/p06-trait-objects`. If `git status` later shows a `target/` folder, add the line `target/` to your `.gitignore` file (see [Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)).

## New words

| Word | Meaning |
|---|---|
| **trait object** | A value of "some type that implements a trait", where the exact type is only known while the program runs. Written `dyn Trait`. |
| **`dyn`** | A keyword (short for *dynamic*) that marks a trait object type, for example `dyn Shape`. |
| **static dispatch** | The compiler decides at compile time which function a call runs. Used with generics. |
| **dynamic dispatch** | The program decides at run time which function a call runs. Used with trait objects. |
| **vtable** (virtual table) | A small table of function addresses for one type and one trait. Trait objects use it to find the right method. |
| **fat pointer** | A pointer that is twice as big as a normal one: it holds an address and one extra piece of information (a vtable address, or a length). |
| **dyn compatible** | A trait that is allowed to be used as `dyn Trait`. The older name is *object safe*. |
| **monomorphization** | The compiler makes a separate copy of a generic function for each concrete type you use. You met it in [Lesson 05.1](../05-rust-intermediate/01-generics.md). |
| **`Sized`** | A marker trait for types whose size is known at compile time. `dyn Trait` is *not* `Sized`. |
| **API** (Application Programming Interface) | The set of functions and types that a piece of code offers to other code. |
| **GUI** (Graphical User Interface) | The windows, buttons and menus of a program that you use with a mouse. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The standard way to store text as bytes. `String::from_utf8` turns bytes into text and fails if the bytes are not valid UTF-8. |

## 1. The problem: different types in one list

Imagine a drawing program. It has circles and rectangles. Both can tell you their area, so we give them a common trait with an `area` method.

Both shapes are tuple structs: `Circle(f64)` stores the radius, `Rect(f64, f64)` stores the width and the height. We want one list with a circle and a rectangle inside, and we want to loop over it. First try the obvious way:

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

fn main() {
    let shapes = vec![Circle(1.0), Rect(2.0, 3.0)];
    println!("{}", shapes.len());
}
```

```text
error[E0308]: mismatched types
  --> src/main.rs:21:36
   |
21 |     let shapes = vec![Circle(1.0), Rect(2.0, 3.0)];
   |                                    ^^^^^^^^^^^^^^ expected `Circle`, found `Rect`

For more information about this error, try `rustc --explain E0308`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

**What happens:**

1. `vec![...]` builds a `Vec<T>`. A `Vec` has exactly one element type `T`.
2. The first element is a `Circle`, so the compiler decides `T = Circle`.
3. The second element is a `Rect`. A `Rect` is not a `Circle`, so the compiler stops with `E0308` (mismatched types).

Generics do not help here. A generic `Vec<T>` where `T: Shape` still means *one* concrete `T` for the whole vector. We need a type that means "any type that implements `Shape`". Rust has one: `dyn Shape`.

## 2. `dyn Trait`: a trait object

`dyn Shape` is the type "some value whose type implements `Shape`; I do not know which one until the program runs". Such a type is called a **trait object**.

There is one catch. Different types have different sizes: a `Circle` here is 8 bytes, a `Rect` is 16 bytes. A `Vec` stores its elements side by side, so each element must have the same size. The fix is to store a **pointer** to the value instead of the value itself. All pointers have the same size. The simplest owning pointer is `Box` (you know it from [Lesson 05.9](../05-rust-intermediate/09-smart-pointers.md)): a `Box<dyn Shape>` owns a value on the heap, and the `Vec` stores only the boxes.

Here is the full program with two different shapes in one list:

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> String;
}

struct Circle(f64); // the radius
struct Rect(f64, f64); // width and height

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.0 * self.0
    }
    fn name(&self) -> String {
        String::from("circle")
    }
}

impl Shape for Rect {
    fn area(&self) -> f64 {
        self.0 * self.1
    }
    fn name(&self) -> String {
        String::from("rectangle")
    }
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![Box::new(Circle(1.0)), Box::new(Rect(2.0, 3.0))];

    for shape in &shapes {
        println!("{} has area {:.2}", shape.name(), shape.area());
    }

    let total: f64 = shapes.iter().map(|s| s.area()).sum();
    println!("total area: {:.2}", total);
}
```

```text
circle has area 3.14
rectangle has area 6.00
total area: 9.14
```

**What happens:**

1. `trait Shape` lists two methods. Anything that implements them can be used as a `Shape`.
2. `Circle` and `Rect` are two unrelated structs. Each has its own `impl Shape`.
3. `Vec<Box<dyn Shape>>` is "a vector of boxes, each box owns some `Shape`".
4. `Box::new(Circle { ... })` creates a `Box<Circle>`. Rust automatically converts it to `Box<dyn Shape>` because the target type says so. This conversion is called an *unsized coercion*: the compiler forgets the exact type and remembers only "it is a `Shape`".
5. `for shape in &shapes` loops over `&Box<dyn Shape>`. `shape.area()` calls the right `area` for each element: the circle formula for the circle, the product for the rectangle.
6. `.iter().map(|s| s.area()).sum()` adds the areas. Iterators work with trait objects like with any other type.

> 💡 **Tip:** You write `dyn` every time you name a trait object type: `Box<dyn Shape>`, `&dyn Shape`. Old code sometimes omits it (`Box<Shape>`); this is an error in edition 2021 and 2024. Always write `dyn`.

## 3. Where a trait object can live

A `dyn Trait` has no known size, so you cannot keep it directly in a variable or return it by value. You always put it behind a pointer. These are the usual choices:

| Type | Meaning | When to use it |
|---|---|---|
| `Box<dyn Trait>` | Owns the value on the heap. | A collection that owns its items; a function that returns "some value". |
| `&dyn Trait` | A shared borrow. | A function parameter that only reads. |
| `&mut dyn Trait` | An exclusive borrow. | A function parameter that changes the value or needs `&mut self` methods. |
| `Rc<dyn Trait>` / `Arc<dyn Trait>` | Shared ownership (single thread / many threads). | The same object is used in many places. |

A very common real example is `&mut dyn Write`. The trait `std::io::Write` is implemented by files, network connections, the terminal, and also by `Vec<u8>` (writing into memory). A function that takes `&mut dyn Write` works with all of them:

```rust
use std::io::Write;

fn write_report(out: &mut dyn Write, user: &str) -> std::io::Result<()> {
    writeln!(out, "report for {}", user)?;
    writeln!(out, "status: ok")?;
    Ok(())
}

fn main() {
    // 1. Write into memory.
    let mut buffer: Vec<u8> = Vec::new();
    write_report(&mut buffer, "ana").unwrap();
    println!("buffer holds {} bytes", buffer.len());

    // 2. Write to the terminal.
    let mut stdout = std::io::stdout();
    write_report(&mut stdout, "ana").unwrap();
}
```

```text
buffer holds 26 bytes
report for ana
status: ok
```

**What happens:**

1. `write_report` takes `out: &mut dyn Write`: "a mutable borrow of anything that can be written to".
2. `writeln!(out, ...)` is like `println!`, but writes into `out`. The `?` passes an error up if writing fails (you learned `?` in [Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)).
3. In the first call, `&mut buffer` is a `&mut Vec<u8>`, which becomes `&mut dyn Write`. Nothing is printed; the text goes into the vector. "report for ana\n" is 15 bytes and "status: ok\n" is 11 bytes, so 26 in total.
4. In the second call, `&mut stdout` writes to the terminal.
5. The same compiled function handled two different types.

> 🧪 **Testing:** This is why `dyn Write` is good for testable code. Your real program writes to the terminal, and your test passes a `Vec<u8>` and checks the bytes. You will do exactly this in Exercise 2.

## 4. How it works inside: fat pointer and vtable

When you call `shape.area()` on a `Box<dyn Shape>`, how does the program know which `area` to run? The pointer carries the answer.

A normal reference `&Circle` is one number: the address of the circle. A reference to a trait object, `&dyn Shape`, is **two numbers**: the address of the value, and the address of a **vtable**. That is why it is called a *fat pointer*.

```text
   Box<dyn Shape>  (16 bytes)
  +----------------+----------------+
  | data pointer   | vtable pointer |
  +-------+--------+--------+-------+
          |                 |
          v                 v
   +-------------+     +--------------------------+
   | Circle(1.0) |     | vtable for Circle: Shape |
   |   (heap)    |     |  drop function           |
   +-------------+     |  size  = 8               |
                       |  align = 8               |
                       |  area  -> Circle::area   |
                       |  name  -> Circle::name   |
                       +--------------------------+
```

The compiler builds one vtable for each pair (type, trait) that is turned into a trait object. A call `shape.area()` becomes: take the vtable pointer, find the slot for `area`, jump to the address stored there. We can check the sizes ourselves:

```rust
use std::fmt::Debug;
use std::mem::{size_of, size_of_val};

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

fn main() {
    println!("&Circle        = {} bytes", size_of::<&Circle>());
    println!("&dyn Shape     = {} bytes", size_of::<&dyn Shape>());
    println!("Box<Circle>    = {} bytes", size_of::<Box<Circle>>());
    println!("Box<dyn Shape> = {} bytes", size_of::<Box<dyn Shape>>());
    println!("&[u8]          = {} bytes", size_of::<&[u8]>());
    println!("&dyn Debug     = {} bytes", size_of::<&dyn Debug>());

    let shapes: Vec<Box<dyn Shape>> = vec![Box::new(Circle(1.0)), Box::new(Rect(2.0, 3.0))];
    for shape in &shapes {
        println!(
            "area {:.2}, the value behind the pointer is {} bytes",
            shape.area(),
            size_of_val(&**shape)
        );
    }
}
```

```text
&Circle        = 8 bytes
&dyn Shape     = 16 bytes
Box<Circle>    = 8 bytes
Box<dyn Shape> = 16 bytes
&[u8]          = 16 bytes
&dyn Debug     = 16 bytes
area 3.14, the value behind the pointer is 8 bytes
area 6.00, the value behind the pointer is 16 bytes
```

**What happens:**

1. `size_of::<T>()` returns the size of a type in bytes.
2. `&Circle` is 8 bytes: one address (this Mac has 64-bit addresses, so 8 bytes).
3. `&dyn Shape` is 16 bytes: the address plus the vtable address. `Box<dyn Shape>` is also 16 bytes.
4. `&[u8]` (a slice) is 16 bytes too. It is also a fat pointer, but the second number is a length instead of a vtable address.
5. `size_of_val(&**shape)` asks for the size of the value *behind* the pointer. The program learns it at run time from the vtable: 8 bytes for the circle, 16 for the rectangle.

> ⚡ **Performance:** A call through a vtable is an *indirect* call. The compiler usually cannot inline it (paste the function body into the caller) and the CPU (Central Processing Unit, the chip that runs the instructions) has to guess the jump target. Most programs never notice. In a loop that runs hundreds of millions of times it can matter. Measure first (see [Lesson 05.12 — Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md)); do not guess.

## 5. Static dispatch and dynamic dispatch

You have now seen two ways to write "a function that works with any `Shape`":

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

// Static dispatch: the compiler makes one copy of this function per type.
fn print_static<T: Shape>(shape: &T) {
    println!("static: {:.2}", shape.area());
}

// Dynamic dispatch: one copy; the method is found at run time.
fn print_dynamic(shape: &dyn Shape) {
    println!("dynamic: {:.2}", shape.area());
}

fn main() {
    let c = Circle(1.0);
    let r = Rect(2.0, 3.0);

    print_static(&c);
    print_static(&r);

    print_dynamic(&c);
    print_dynamic(&r);
}
```

```text
static: 3.14
static: 6.00
dynamic: 3.14
dynamic: 6.00
```

**What happens:**

1. `print_static<T: Shape>` is generic. For every type you call it with, the compiler writes a separate copy (monomorphization): one for `Circle` and one for `Rect`. Each call is a direct jump to a known address. This is **static dispatch**.
2. `print_dynamic` takes `&dyn Shape`. There is only one copy of the function. Inside it, `shape.area()` goes through the vtable. This is **dynamic dispatch**.
3. Both produce the same output. The difference is how the machine gets there.

Compare them:

| | Static dispatch (`T: Trait`, `impl Trait`) | Dynamic dispatch (`dyn Trait`) |
|---|---|---|
| When is the method chosen? | At compile time | At run time |
| Speed of a call | Direct, can be inlined | Indirect through the vtable |
| Machine code size | One copy per type (can grow big) | One copy for all types |
| Different types in one `Vec`? | No | Yes |
| Type must be known at compile time? | Yes | No |
| Return different types from one function? | No | Yes, via `Box<dyn Trait>` |
| Can add a new type later without changing the code that uses the trait? | Yes, but the user code is compiled again | Yes, even from another crate |

A practical rule:

- Start with generics or `impl Trait`. They are fast, and the compiler checks everything for you.
- Use `dyn Trait` when you need **different types behind one interface at run time**: a list of plugins, handlers chosen from a configuration file, GUI widgets, a "strategy" that the user selects.
- Use `dyn Trait` also when you want to reduce compile time and binary size, and the call is not performance critical.

> 📏 **Convention:** In a function **parameter**, prefer `impl Trait` or a generic `T: Trait` when you can; use `&dyn Trait` when you want one compiled function, or when the value is already a trait object. In a **struct field** that must hold many different types, use `Box<dyn Trait>`.

## 6. Dyn compatibility: which traits can be trait objects

Not every trait can be used as `dyn Trait`. The compiler must be able to build a vtable with fixed slots. A trait that follows the rules is called **dyn compatible** (the older name was "object safe"). The main rules for the methods of the trait are:

- A method must not return `Self`. The vtable does not know the concrete type, so it cannot know what is returned.
- A method must not have its own generic type parameters (`fn log<T>(...)`). A generic method has infinitely many versions, and a vtable needs one fixed slot.
- A method must have a receiver (`&self`, `&mut self`, `self: Box<Self>`, ...). A function without `self`, like `fn new() -> Self`, cannot be called on a trait object.
- The trait must not require `Self: Sized`.

Look at what the compiler says if you break the first rule:

```rust
trait Duplicate {
    fn duplicate(&self) -> Self;
}

struct Note(String);

impl Duplicate for Note {
    fn duplicate(&self) -> Self {
        Note(self.0.clone())
    }
}

fn main() {
    let items: Vec<Box<dyn Duplicate>> = vec![Box::new(Note(String::from("hi")))];
    println!("{}", items.len());
}
```

```text
error[E0038]: the trait `Duplicate` is not dyn compatible
  --> src/main.rs:14:28
   |
14 |     let items: Vec<Box<dyn Duplicate>> = vec![Box::new(Note(String::from("hi")))];
   |                            ^^^^^^^^^ `Duplicate` is not dyn compatible
   |
note: for a trait to be dyn compatible it needs to allow building a vtable
      for more information, visit <https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility>
  --> src/main.rs:2:28
   |
 1 | trait Duplicate {
   |       --------- this trait is not dyn compatible...
 2 |     fn duplicate(&self) -> Self;
   |                            ^^^^ ...because method `duplicate` references the `Self` type in its return type
   = help: consider moving `duplicate` to another trait
   = help: only type `Note` implements `Duplicate`; consider using it directly instead.
```

And the second rule:

```text
error[E0038]: the trait `Logger` is not dyn compatible
  --> src/main.rs:20:24
   |
20 |     let boxed: Box<dyn Logger> = Box::new(ConsoleLogger);
   |                        ^^^^^^ `Logger` is not dyn compatible
   |
note: for a trait to be dyn compatible it needs to allow building a vtable
      for more information, visit <https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility>
  --> src/main.rs:5:8
   |
 3 | trait Logger {
   |       ------ this trait is not dyn compatible...
 4 |     fn log(&self, message: &str);
 5 |     fn log_value<T: Debug>(&self, value: T);
   |        ^^^^^^^^^ ...because method `log_value` has generic type parameters
   = help: consider moving `log_value` to another trait
   = help: only type `ConsoleLogger` implements `Logger`; consider using it directly instead.
```

**What happens:**

1. `Box<dyn Duplicate>` needs a vtable for `Duplicate`.
2. The method `duplicate` returns `Self`. Behind a `dyn Duplicate` pointer nobody knows what `Self` is, so a slot for `duplicate` cannot be filled.
3. The compiler refuses with `E0038` and tells you exactly which method causes the problem and why.

There are two ways out. You can redesign the trait (for example, return `Box<dyn Duplicate>`). Or you can keep the method but exclude it from the vtable with `where Self: Sized`. The method then exists only for concrete types:

```rust
use std::fmt::Debug;

trait Logger {
    fn log(&self, message: &str);

    // A generic method cannot be called through a vtable,
    // so we exclude it from the trait object with `where Self: Sized`.
    fn log_value<T: Debug>(&self, value: T)
    where
        Self: Sized,
    {
        self.log(&format!("{:?}", value));
    }
}

struct ConsoleLogger;

impl Logger for ConsoleLogger {
    fn log(&self, message: &str) {
        println!("[console] {}", message);
    }
}

fn main() {
    let logger = ConsoleLogger;
    logger.log_value(42); // works: called on a concrete type

    let boxed: Box<dyn Logger> = Box::new(ConsoleLogger);
    boxed.log("hello from a trait object"); // works: log is in the vtable
}
```

```text
[console] 42
[console] hello from a trait object
```

**What happens:**

1. `log` is a normal method, so it goes into the vtable.
2. `log_value<T>` is generic, but its `where Self: Sized` line says "this method is only for types with a known size". `dyn Logger` has no known size, so the method is left out of the vtable, and the trait is dyn compatible again.
3. `logger.log_value(42)` works on the concrete `ConsoleLogger`.
4. `boxed.log(...)` works on the trait object. Calling `boxed.log_value(1)` would not compile.

## 7. Trait objects and lifetimes

A `Box<dyn Shape>` can hold a value that contains references (a struct with a `&'a str` field). You learned lifetimes in [Lesson 05.3](../05-rust-intermediate/03-lifetimes.md); here is the one new rule. In a function signature, `Box<dyn Shape>` means `Box<dyn Shape + 'static>`: "the value inside holds no short-lived borrows". If the value does borrow something, you must say how long the borrow lasts:

```rust
trait Shape {
    fn label(&self) -> String;
}

struct Tag<'a> {
    text: &'a str,
}

impl Shape for Tag<'_> {
    fn label(&self) -> String {
        self.text.to_string()
    }
}

fn make_tag(text: &str) -> Box<dyn Shape> {
    Box::new(Tag { text })
}

fn main() {
    let text = String::from("local");
    let shape = make_tag(&text);
    println!("{}", shape.label());
}
```

```text
error: lifetime may not live long enough
  --> src/main.rs:16:5
   |
15 | fn make_tag(text: &str) -> Box<dyn Shape> {
   |                   - let's call the lifetime of this reference `'1`
16 |     Box::new(Tag { text })
   |     ^^^^^^^^^^^^^^^^^^^^^^ returning this value requires that `'1` must outlive `'static`
   |
help: to declare that the trait object captures data from argument `text`, you can add an explicit `'_` lifetime bound
   |
15 | fn make_tag(text: &str) -> Box<dyn Shape + '_> {
   |                                          ++++
```

The fix is the lifetime of the borrow, written after the trait name: `Box<dyn Shape + 'a>`. In a signature with a single input reference you can use `'_` (the anonymous lifetime):

```rust
trait Shape {
    fn label(&self) -> String;
}

struct Tag<'a> {
    text: &'a str,
}

impl Shape for Tag<'_> {
    fn label(&self) -> String {
        self.text.to_string()
    }
}

fn make_tag<'a>(text: &'a str) -> Box<dyn Shape + 'a> {
    Box::new(Tag { text })
}

fn main() {
    let text = String::from("local");
    let shape = make_tag(&text);
    println!("{}", shape.label());
}
```

```text
local
```

**What happens:**

1. `Tag<'a>` holds a borrowed `&'a str`.
2. `Box<dyn Shape + 'a>` says: "a shape that may hold borrows that live at least as long as `'a`".
3. The returned box cannot outlive `text`. The compiler checks that in `main`.

You can also add other markers after `dyn`, for example `Box<dyn Shape + Send>` to say the value can be moved to another thread ([Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)). We go deeper into lifetimes in [Lesson 06.5](05-lifetimes-in-depth.md).

## 8. Closures and plugins as trait objects

Every closure has its own unique type, so two closures cannot go into one `Vec` as they are. A trait object fixes that: closures implement `Fn`, `FnMut` or `FnOnce`, and those are traits too.

```rust
fn main() {
    let steps: Vec<Box<dyn Fn(i32) -> i32>> = vec![
        Box::new(|x| x + 1),
        Box::new(|x| x * 10),
        Box::new(|x| x - 3),
    ];

    let mut value = 4;
    for step in &steps {
        value = step(value);
    }
    println!("result: {}", value);
}
```

```text
result: 47
```

**What happens:**

1. `Box<dyn Fn(i32) -> i32>` means "a box with any closure that takes an `i32` and returns an `i32`".
2. The three closures have three different types, but all are stored in one `Vec`.
3. The loop applies them in order: 4 + 1 = 5, 5 * 10 = 50, 50 - 3 = 47.

A bigger example is a plugin list. Every plugin has a name and a `run` method that can change the plugin's own state (`&mut self`). `Debug` is a *supertrait*: every `Plugin` must also implement `Debug`. That lets us print the whole list with `{:?}`.

```rust
use std::fmt::Debug;

trait Plugin: Debug {
    fn name(&self) -> &str;
    fn run(&mut self, input: &str) -> String;
}

#[derive(Debug)]
struct Upper;

#[derive(Debug)]
struct Counter {
    calls: u32,
}

impl Plugin for Upper {
    fn name(&self) -> &str {
        "upper"
    }
    fn run(&mut self, input: &str) -> String {
        input.to_uppercase()
    }
}

impl Plugin for Counter {
    fn name(&self) -> &str {
        "counter"
    }
    fn run(&mut self, input: &str) -> String {
        self.calls += 1;
        format!("{} (call #{})", input, self.calls)
    }
}

fn main() {
    let mut plugins: Vec<Box<dyn Plugin>> = vec![Box::new(Upper), Box::new(Counter { calls: 0 })];

    for round in 1..=2 {
        for plugin in plugins.iter_mut() {
            let out = plugin.run("hello");
            println!("round {} {:>8}: {}", round, plugin.name(), out);
        }
    }
    println!("{:?}", plugins);
}
```

```text
round 1    upper: HELLO
round 1  counter: hello (call #1)
round 2    upper: HELLO
round 2  counter: hello (call #2)
[Upper, Counter { calls: 2 }]
```

**What happens:**

1. `trait Plugin: Debug` means "to be a `Plugin` a type must also be `Debug`". Because of this, `Box<dyn Plugin>` is also `Debug`.
2. `plugins.iter_mut()` gives `&mut Box<dyn Plugin>`. We can call `run` because it takes `&mut self`.
3. `Counter` keeps state: `calls` grows with each call. `Upper` has no state.
4. `{:?}` on the vector prints the real struct of each plugin: `Upper` and `Counter { calls: 2 }`.

> 📏 **Convention:** Keep a trait object trait small. Two or three methods is normal. A big trait is hard to implement and hard to keep dyn compatible.

## Common mistakes

### 1. Putting different types in a `Vec` without `Box<dyn ...>`

You saw it in section 1: `E0308 mismatched types`. Declare the vector as `Vec<Box<dyn Shape>>` and wrap every item in `Box::new(...)`.

### 2. Returning `dyn Trait` by value

```text
error[E0746]: return type cannot be a trait object without pointer indirection
  --> src/main.rs:13:20
   |
13 | fn make_shape() -> dyn Shape {
   |                    ^^^^^^^^^ doesn't have a size known at compile-time
   |
help: consider returning an `impl Trait` instead of a `dyn Trait`
   |
13 - fn make_shape() -> dyn Shape {
13 + fn make_shape() -> impl Shape {
   |
help: alternatively, box the return type, and wrap all of the returned values in `Box::new`
   |
13 ~ fn make_shape() -> Box<dyn Shape> {
14 ~     Box::new(Circle(1.0))
   |
```

The compiler even suggests the two fixes. The function returns `dyn Shape`, which has no known size, and a function result must have one. Return `Box<dyn Shape>` (and write `Box::new(Circle { radius: 1.0 })`), or `impl Shape` if there is only one concrete type.

### 3. `impl Trait` is not a trait object

`impl Shape` in a return type means "one specific type that I do not want to name". It is still **one** type:

```text
error[E0308]: `if` and `else` have incompatible types
  --> src/main.rs:21:37
   |
21 |     if round { Circle(1.0) } else { Rect(2.0, 3.0) }
   |                -----------          ^^^^^^^^^^^^^^ expected `Circle`, found `Rect`
   |                |
   |                expected because of this
   |
help: you could change the return type to be a boxed trait object
   |
20 - fn make_shape(round: bool) -> impl Shape {
20 + fn make_shape(round: bool) -> Box<dyn Shape> {
   |
help: if you change the return type to expect trait objects, box the returned expressions
   |
21 |     if round { Box::new(Circle(1.0)) } else { Box::new(Rect(2.0, 3.0)) }
   |                +++++++++           +          +++++++++              +
```

If a function must return a circle in one branch and a rectangle in another, return `Box<dyn Shape>` and box each value.

### 4. A trait that is not dyn compatible

`E0038`, see section 6. Read the `...because method ... references the Self type` line: it names the method to change.

### 5. Forgetting the lifetime of a borrowed value

`lifetime may not live long enough ... must outlive 'static`, see section 7. Add `+ 'a` or `+ '_` to the trait object type.

## Try it yourself

Build a small plugin registry and commit it.

1. Open the project: `cd ~/projects/rust/code/p06-trait-objects`.
2. Put the `Plugin` example from section 8 into `src/main.rs`.
3. Run it: `cargo run`. You should see four lines like the output above, then the `Debug` list.
4. Add a third plugin, `Reverse`, that returns the input text backwards (hint: `input.chars().rev().collect()`). Add it to the vector. Run again.
5. Check your code: `cargo fmt` then `cargo clippy`. Both should print nothing wrong.
6. Save your work in Git (see [Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md) and the message style from [Lesson 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md)):

```sh
git add code/p06-trait-objects
git commit -m "feat(p06): plugin registry with trait objects"
```

> 💡 **Tip:** One small commit per finished exercise gives you a safe point to return to. If you break something later, `git diff` shows what changed (Lesson 01.4).

## Exercises

### Exercise 1 — A zoo (easy)

Write a trait `Animal` with `name(&self) -> String` and `sound(&self) -> String`. Implement it for `Dog`, `Cat` and a `Cow` that has a field `loud: bool` (a loud cow says `MOO`, a quiet one `moo`). Put one of each in a `Vec<Box<dyn Animal>>` and print `Dog says woof` and so on.

<details>
<summary>Hint</summary>

The vector type is `Vec<Box<dyn Animal>>`. Create the items with `Box::new(Dog)`. Loop with `for animal in &zoo`.

</details>

<details>
<summary>Answer</summary>

```rust
trait Animal {
    fn name(&self) -> String;
    fn sound(&self) -> String;
}

struct Dog;
struct Cat;
struct Cow {
    loud: bool,
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
        String::from("Cat")
    }
    fn sound(&self) -> String {
        String::from("meow")
    }
}

impl Animal for Cow {
    fn name(&self) -> String {
        String::from("Cow")
    }
    fn sound(&self) -> String {
        if self.loud {
            String::from("MOO")
        } else {
            String::from("moo")
        }
    }
}

fn main() {
    let zoo: Vec<Box<dyn Animal>> =
        vec![Box::new(Dog), Box::new(Cat), Box::new(Cow { loud: true })];
    for animal in &zoo {
        println!("{} says {}", animal.name(), animal.sound());
    }
}
```

```text
Dog says woof
Cat says meow
Cow says MOO
```

Why it works: each struct has its own `impl Animal`. The boxes hide the real types, and each call `animal.sound()` goes through the vtable of the real type.

</details>

### Exercise 2 — Test a report writer (medium)

Take `write_report` from section 3. Write a test that calls it with a `Vec<u8>` and checks that the text is exactly `"report for bob\nstatus: ok\n"`.

<details>
<summary>Hint</summary>

Create `let mut buffer: Vec<u8> = Vec::new();`, pass `&mut buffer`, then turn the bytes into a `String` with `String::from_utf8(buffer).unwrap()` and compare with `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io::Write;

fn write_report(out: &mut dyn Write, user: &str) -> std::io::Result<()> {
    writeln!(out, "report for {}", user)?;
    writeln!(out, "status: ok")?;
    Ok(())
}

fn main() {
    let mut stdout = std::io::stdout();
    write_report(&mut stdout, "ana").unwrap();
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn report_has_two_lines() {
        let mut buffer: Vec<u8> = Vec::new();
        write_report(&mut buffer, "bob").unwrap();
        let text = String::from_utf8(buffer).unwrap();
        assert_eq!(text, "report for bob\nstatus: ok\n");
    }
}
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `Vec<u8>` implements `Write`, so it can be passed as `&mut dyn Write`. The function does not know and does not care that the "file" is just memory.

</details>

### Exercise 3 — Make a trait dyn compatible (medium)

This trait has a constructor `unit() -> Self`, so it cannot be used as `dyn Shape`. Change **only the trait** so that `Vec<Box<dyn Shape>>` works and `Square::unit()` still works. The program must print `total: 10`.

```rust
trait Shape {
    fn area(&self) -> f64;
    fn unit() -> Self;
}
```

<details>
<summary>Hint</summary>

Add `where Self: Sized` to the method that returns `Self` and has no receiver.

</details>

<details>
<summary>Answer</summary>

```rust
trait Shape {
    fn area(&self) -> f64;

    // Not in the vtable: only for concrete types.
    fn unit() -> Self
    where
        Self: Sized;
}

struct Square(f64);

impl Shape for Square {
    fn area(&self) -> f64 {
        self.0 * self.0
    }
    fn unit() -> Self {
        Square(1.0)
    }
}

fn main() {
    let shapes: Vec<Box<dyn Shape>> = vec![Box::new(Square::unit()), Box::new(Square(3.0))];
    let total: f64 = shapes.iter().map(|s| s.area()).sum();
    println!("total: {}", total);
}
```

```text
total: 10
```

Why it works: `where Self: Sized` removes `unit` from the vtable. Calling `Square::unit()` on the concrete type is still fine. The areas are 1 * 1 + 3 * 3 = 10.

</details>

### Exercise 4 — An event bus (hard)

Write a struct `EventBus` that stores a list of handlers. A handler is any closure `Fn(&str) -> String`. It needs three functions: `new()`, `subscribe(&mut self, handler)` and `publish(&self, event: &str) -> Vec<String>`, which runs every handler in order and returns their results. Write two tests: one with two handlers (check the order of results), one with no handlers (the result is empty).

<details>
<summary>Hint</summary>

Store `Vec<Box<dyn Fn(&str) -> String>>`. Clippy says that this type is very complex; give it a name with `type Handler = Box<dyn Fn(&str) -> String>;` (type aliases are explained in [Lesson 06.2](02-advanced-traits-and-types.md)). In `publish`, use `self.handlers.iter().map(|h| h(event)).collect()`. A closure that uses a `String` from outside needs the `move` keyword.

</details>

<details>
<summary>Answer</summary>

```rust
// A name for a long type (Clippy warns about very complex types).
type Handler = Box<dyn Fn(&str) -> String>;

struct EventBus {
    handlers: Vec<Handler>,
}

impl EventBus {
    fn new() -> Self {
        EventBus {
            handlers: Vec::new(),
        }
    }

    fn subscribe(&mut self, handler: Handler) {
        self.handlers.push(handler);
    }

    fn publish(&self, event: &str) -> Vec<String> {
        self.handlers.iter().map(|h| h(event)).collect()
    }
}

fn main() {
    let mut bus = EventBus::new();
    let prefix = String::from("log");
    bus.subscribe(Box::new(move |e| format!("{}: {}", prefix, e)));
    bus.subscribe(Box::new(|e| e.to_uppercase()));

    for line in bus.publish("started") {
        println!("{}", line);
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn every_handler_runs_in_order() {
        let mut bus = EventBus::new();
        bus.subscribe(Box::new(|e| format!("a:{}", e)));
        bus.subscribe(Box::new(|e| format!("b:{}", e)));
        assert_eq!(bus.publish("x"), vec!["a:x", "b:x"]);
    }

    #[test]
    fn no_handlers_means_no_output() {
        let bus = EventBus::new();
        assert!(bus.publish("x").is_empty());
    }
}
```

```text
log: started
STARTED
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: each closure has its own type, but `Box<dyn Fn(&str) -> String>` hides it. `move` gives the first closure its own copy of `prefix`, so the closure does not depend on a local variable that may disappear.

</details>

## Quiz

1. Why can `vec![Circle {..}, Rect {..}]` not compile, but `Vec<Box<dyn Shape>>` can?

<details>
<summary>Answer</summary>

A `Vec<T>` has one element type, and `Circle` and `Rect` are different types. `Box<dyn Shape>` is one type (a fat pointer to "some `Shape`"), so boxes with different concrete types fit in the same vector.

</details>

2. How many bytes is a `&dyn Trait` on a 64-bit machine, and what is inside?

<details>
<summary>Answer</summary>

16 bytes: 8 for the address of the value and 8 for the address of the vtable.

</details>

3. What does a vtable contain?

<details>
<summary>Answer</summary>

For one type and one trait: the function that drops the value, the size, the alignment, and the addresses of the trait's methods.

</details>

4. What is the difference between static and dynamic dispatch?

<details>
<summary>Answer</summary>

With static dispatch (generics, `impl Trait`) the compiler knows the concrete type and chooses the method at compile time, making one copy of the code per type. With dynamic dispatch (`dyn Trait`) the method is looked up in the vtable at run time, and there is one copy of the code.

</details>

5. Why does `trait Duplicate { fn duplicate(&self) -> Self; }` not work as `dyn Duplicate`?

<details>
<summary>Answer</summary>

The method returns `Self`, a concrete type that a trait object has forgotten. The vtable cannot have a slot with an unknown return type, so the trait is not dyn compatible (error `E0038`).

</details>

6. A trait has a generic method `fn log_value<T: Debug>(&self, v: T)`. How do you keep the trait dyn compatible?

<details>
<summary>Answer</summary>

Add `where Self: Sized` to that method. It then exists only for concrete types and is left out of the vtable.

</details>

7. What does `Box<dyn Shape>` mean in a function signature when you do not write any lifetime?

<details>
<summary>Answer</summary>

`Box<dyn Shape + 'static>`: the value inside must not contain short-lived borrows. If it does, write `Box<dyn Shape + 'a>` (or `'_`).

</details>

## Summary

- A trait object, `dyn Trait`, is a value of "some type that implements `Trait`", with the type known only at run time.
- A trait object has no known size, so it lives behind a pointer: `Box<dyn T>`, `&dyn T`, `&mut dyn T`, `Rc<dyn T>`.
- A pointer to a trait object is a fat pointer: data address plus vtable address (16 bytes on 64-bit systems).
- Dynamic dispatch looks up the method in the vtable at run time. Static dispatch (generics) is decided at compile time and can be inlined.
- Use generics by default. Use `dyn Trait` for lists of mixed types, plugins, and choices made at run time.
- A trait must be dyn compatible: no `Self` in return types, no generic methods, methods have a receiver. `where Self: Sized` hides a method from the vtable.
- `Box<dyn T>` means `Box<dyn T + 'static>`. Add `+ 'a` when the value borrows data.

## Sources and further reading

- [The Rust Programming Language — 18.2 Using Trait Objects](https://doc.rust-lang.org/book/ch18-02-trait-objects.html) — the official chapter on trait objects and dynamic dispatch.
- [The Rust Reference — Trait objects](https://doc.rust-lang.org/reference/types/trait-object.html) — the exact rules, including default lifetime bounds.
- [The Rust Reference — Traits (dyn compatibility)](https://doc.rust-lang.org/reference/items/traits.html) — the full list of dyn compatibility rules.
- [Rust by Example — Returning Traits with dyn](https://doc.rust-lang.org/rust-by-example/trait/dyn.html) — a short example of `Box<dyn Trait>` as a return type.
- [std::mem::size_of_val](https://doc.rust-lang.org/std/mem/fn.size_of_val.html) — how to measure the size of a value behind a pointer.

---

[🏠 Start](../START-HERE.md) · [← Phase 06 — Rust advanced](README.md) · [📋 Phase 06](README.md) · [06.2 Advanced traits and types →](02-advanced-traits-and-types.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
