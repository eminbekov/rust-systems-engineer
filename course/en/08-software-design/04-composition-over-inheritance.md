<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.3 The SOLID principles](03-solid-principles.md) · [📋 Phase 08](README.md) · [08.5 Type-driven design: make invalid states impossible →](05-type-driven-design.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/04-composition-over-inheritance.md) <!-- nav -->

# Lesson 08.4 — Composition over inheritance

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 4 of 17 · ⏱ about 120 minutes

## What you will learn

- What inheritance is, and why Rust does not have it for structs.
- How to build bigger types from smaller ones (**composition**) and share behaviour with traits.
- How to extend an existing type by **wrapping** it, for example a writer that counts bytes.
- Why using `Deref` to imitate inheritance is an anti-pattern.
- How to choose between a struct field, a trait, a wrapper and an enum.

## Before you start

- You finished [Lesson 08.3 — The SOLID principles](03-solid-principles.md).
- You know structs and methods ([Lesson 04.1](../04-structs-enums-collections-errors/01-structs.md) and [Lesson 04.2](../04-structs-enums-collections-errors/02-methods-and-associated-functions.md)), traits with default methods ([Lesson 05.2](../05-rust-intermediate/02-traits.md)) and the `Deref` trait ([Lesson 05.9](../05-rust-intermediate/09-smart-pointers.md)).
- You remember the newtype pattern and supertraits from [Lesson 06.2](../06-rust-advanced/02-advanced-traits-and-types.md).

## New words

| Word | Meaning |
|---|---|
| **inheritance** | A feature of some programming languages: a new type is declared as a "child" of an existing "parent" type and automatically receives the parent's fields and methods, which it may replace. Rust has no inheritance of fields or method bodies between structs. |
| **is-a / has-a** | Two kinds of relationship. "A dog *is an* animal" is the inheritance way of thinking. "A player *has a* position" is composition. |
| **composition** | Building a bigger type by putting smaller types inside it as fields. |
| **component** | A small type that does one job and is used as a field of bigger types (`Position`, `Health`). |
| **delegation** (forwarding) | A method of the outer type that does its work by calling the same job on an inner value. |
| **default method** | A method of a trait that has a body. Implementers get it for free and may replace it. |
| **wrapper** (decorator) | A type that holds another value of some type, offers the same interface, and adds behaviour around it. |
| **anti-pattern** | A common solution that looks fine but causes more problems than it solves. Lesson 08.10 collects many. |

## 1. What inheritance is, and why design advice avoids it

In some languages you can write "type `Dog` extends type `Animal`". The `Dog` then receives all fields and methods of `Animal` by itself, and can replace some of them. This models an **is-a** relation: a dog *is an* animal.

It looks like a great way to reuse code, but experience with large programs showed some problems:

- The child depends on the *insides* of the parent. A change in the parent can break every child (very tight coupling, Lesson 08.2).
- Hierarchies are rigid. What is a robot dog? Is it an `Animal` or a `Machine`? You can only have one family tree.
- The "is-a" claim is often false in practice. A square *is a* rectangle in geometry, but code that sets the width of a rectangle independently of its height breaks when it receives a square.

A famous book on design patterns (1994) gave the advice that became a slogan: **favour composition over inheritance**. Rust follows it by design: a struct cannot inherit from another struct. Instead Rust gives you four tools: fields (composition), traits (shared behaviour), wrappers (extension) and enums (a closed set of variants). We look at each.

## 2. Composition: has-a

Composition means that a bigger type *has* smaller types inside. Each small type, a **component**, does one job. Here a game object gets its abilities from components:

```rust
struct Position {
    x: i32,
    y: i32,
}

impl Position {
    fn move_by(&mut self, dx: i32, dy: i32) {
        self.x += dx;
        self.y += dy;
    }
}

struct Health {
    points: u32,
}

impl Health {
    fn damage(&mut self, amount: u32) {
        self.points = self.points.saturating_sub(amount);
    }
    fn is_alive(&self) -> bool {
        self.points > 0
    }
}

struct Player {
    name: String,
    position: Position,
    health: Health,
}

struct Wall {
    position: Position, // a wall has a position but no health
}

fn main() {
    let mut player = Player {
        name: "Ana".to_string(),
        position: Position { x: 0, y: 0 },
        health: Health { points: 30 },
    };
    let mut wall = Wall {
        position: Position { x: 5, y: 5 },
    };

    player.position.move_by(2, 3);
    wall.position.move_by(-1, 0);
    player.health.damage(40);

    println!(
        "{} is at ({}, {})",
        player.name, player.position.x, player.position.y
    );
    println!("wall is at ({}, {})", wall.position.x, wall.position.y);
    println!("alive: {}", player.health.is_alive());
}
```

**What happens:**

1. `Position` can move itself with `move_by`. `Health` can take damage, and `saturating_sub` stops at zero instead of going below.
2. `Player` *has a* `Position` and *has a* `Health`. `Wall` has only a `Position`, because a wall has no health.
3. `player.position.move_by(2, 3)` calls the job on the inner value. The player's position is (0, 0) + (2, 3).
4. The wall starts at (5, 5) and moves by (-1, 0). The player takes 40 damage, more than the 30 health points it has, so `health.points` becomes 0.

```text
Ana is at (2, 3)
wall is at (4, 5)
alive: false
```

What did we gain?

- **Mix and match.** A `Wall` can have a position without health. With a family tree you would need a class between them that has a position but not health.
- **Small, testable parts.** `Health` and `Position` are cohesive (Lesson 08.2) and can be tested alone.
- **No hidden coupling.** `Player` uses only the public methods of `Health`. If `Health` changes how it stores the points, `Player` does not care.

> 📏 **Convention:** When you feel the urge to inherit "to reuse code", ask: *what* do I want to reuse? Data and the behaviour on that data? Make it a component and hold it in a field.

## 3. Sharing behaviour with traits

A trait does not describe what a thing *is*; it describes what it can *do*. A trait can also contain **default methods**, which give behaviour to every implementer, a bit like a parent giving methods to its children, but without sharing data:

```rust
trait Named {
    fn name(&self) -> &str;

    // A default method: shared behaviour that every implementer gets for free.
    fn greet(&self) -> String {
        format!("Hello, I am {}", self.name())
    }
}

struct Dog {
    name: String,
}

struct Robot {
    id: u32,
    label: String,
}

impl Named for Dog {
    fn name(&self) -> &str {
        &self.name
    }
}

impl Named for Robot {
    fn name(&self) -> &str {
        &self.label
    }

    // This type replaces the default.
    fn greet(&self) -> String {
        format!("BEEP. Unit {} ({})", self.id, self.label)
    }
}

fn main() {
    let dog = Dog {
        name: "Rex".to_string(),
    };
    let robot = Robot {
        id: 7,
        label: "R7".to_string(),
    };
    println!("{}", dog.greet());
    println!("{}", robot.greet());
}
```

**What happens:**

1. `Named` requires one thing: `name`. `greet` has a body that uses `name`, so it is a default method.
2. `Dog` implements only `name`, and receives `greet` for free.
3. `Robot` implements `name` and also replaces `greet` with its own text.
4. `main` calls `greet` on both. Each uses its own version.

```text
Hello, I am Rex
BEEP. Unit 7 (R7)
```

The trait carries the *behaviour* (what is common); each struct keeps its own *data*. A function that takes `&impl Named` accepts both. If a trait needs another trait's methods, use a supertrait (Lesson 06.2): `trait Pet: Named { ... }`.

## 4. Extending a type by wrapping it

Inheritance is often used to "take a class and add one thing". In Rust you wrap the value in a new struct, offer the same trait, and delegate the real work to the inner value. This is a **wrapper** (also called a decorator). Here is a writer that counts how many bytes pass through it:

```rust
use std::io::{self, Write};

struct CountingWriter<W: Write> {
    inner: W,
    bytes: usize,
}

impl<W: Write> CountingWriter<W> {
    fn new(inner: W) -> Self {
        CountingWriter { inner, bytes: 0 }
    }
}

impl<W: Write> Write for CountingWriter<W> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        let written = self.inner.write(buf)?; // delegate to the inner writer
        self.bytes += written;
        Ok(written)
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}

fn main() -> io::Result<()> {
    let mut out = CountingWriter::new(io::stdout());
    writeln!(out, "hello")?;
    writeln!(out, "composition")?;
    println!("bytes written: {}", out.bytes);
    Ok(())
}
```

**What happens:**

1. `CountingWriter<W: Write>` holds any writer `inner` (Lesson 08.3 explained `Write`) and a counter.
2. `impl Write for CountingWriter<W>` makes the wrapper a writer itself, so it can be used everywhere a writer is expected.
3. In `write`, the real writing is **delegated** to `self.inner.write(buf)?`; the wrapper adds only the counting. `flush` is delegated as it is.
4. `main` wraps the standard output. The two lines add 6 + 12 bytes (each includes the newline `\n`).

```text
hello
composition
bytes written: 18
```

The standard library is full of wrappers: `BufWriter` and `BufReader` add buffering to any writer or reader, and iterator adapters such as `map` and `filter` wrap another iterator. You can stack them: a counting writer around a buffered writer around a file. The wrapper does not know what it wraps, only the trait, which keeps the coupling low.

## 5. `Deref` is not inheritance

Beginners sometimes notice that `Deref` lets `Dog` use the methods of an inner `Animal` (Lesson 05.9), and they use it to fake inheritance. It looks like it works:

```rust
use std::ops::Deref;

struct Animal {
    name: String,
}

impl Animal {
    fn name(&self) -> &str {
        &self.name
    }
}

trait Speak {
    fn speak(&self) -> String;
}

impl Speak for Animal {
    fn speak(&self) -> String {
        format!("{} makes a sound", self.name)
    }
}

struct Dog(Animal);

impl Deref for Dog {
    type Target = Animal;
    fn deref(&self) -> &Animal {
        &self.0
    }
}

fn announce(speaker: &impl Speak) {
    println!("{}", speaker.speak());
}

fn main() {
    let dog = Dog(Animal {
        name: "Rex".to_string(),
    });
    println!("{}", dog.name()); // works through Deref
    announce(&dog); // does not work: Dog is not an Animal
}
```

```text
error[E0277]: the trait bound `Dog: Speak` is not satisfied
  --> src/main.rs:41:14
   |
41 |     announce(&dog); // does not work: Dog is not an Animal
   |     -------- ^^^^ unsatisfied trait bound
   |     |
   |     required by a bound introduced by this call
   |
help: the trait `Speak` is not implemented for `Dog`
   ...

For more information about this error, try `rustc --explain E0277`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

The call `dog.name()` compiled, because the compiler follows `Deref` to find a method. But `Dog` is not an `Animal`: the trait `Speak`, implemented for `Animal`, is not implemented for `Dog`, and generic code that requires `Speak` rejects it.

This is the **Deref polymorphism anti-pattern**. `Deref` is meant for *smart pointers* (`Box`, `Rc`, `String` to `str`), types whose whole purpose is to act like the thing inside. Using it for "is-a" gives surprising method lookups, and it hides which type really has the method. The honest fix is explicit delegation: implement the trait for `Dog` and forward, `impl Speak for Dog { fn speak(&self) -> String { self.0.speak() } }`, and write a plain method `fn name(&self) -> &str { self.0.name() }` if you need it.

## 6. Which tool? And when not to compose

| You want to... | Use | Example |
|---|---|---|
| Reuse data and operations on it | A component in a **field** | `Player { health: Health }` |
| Say "these types can do X" and share code for X | A **trait**, maybe with default methods | `Named::greet` |
| Add behaviour around an existing type | A **wrapper** that delegates | `CountingWriter<W>` |
| Model "exactly one of these cases" | An **enum** | `Shape::Circle`, `Shape::Rect` |

**When NOT to compose (or not too much):**

- Wrapping a type and forwarding 20 methods by hand is a lot of boilerplate. Consider whether a trait with default methods fits better, or whether a method on the wrapper that returns `&Inner` is enough.
- Do not split a struct into a dozen one-field "components" just in case. Make a component when it has behaviour of its own and more than one user (the rule of three again).
- Very deep nesting (`a.b.c.d.value`) is coupling in disguise (Lesson 08.2). Give the outer type a method for what callers need.

## Common mistakes

**Mistake 1: faking inheritance with `Deref`.** See Section 5. If you notice that you wrote `impl Deref for MyType` and the target is not a pointer-like thing, stop and use delegation.

**Mistake 2: a family tree in structs.** `struct Dog { animal: Animal }` inside `struct Puppy { dog: Dog }` is inheritance with extra typing. Ask what the real reuse is and give it its own name.

**Mistake 3: a trait that only wraps one struct.** If the trait has one implementer and nobody plans another, the struct alone is enough (YAGNI, "You Aren't Gonna Need It", Lesson 08.1).

## Try it yourself

1. Create the package:

```bash
cd ~/projects/rust
cargo new code/p08-composition
cd code/p08-composition
```

2. Copy the components and `main` from Section 2 into `src/main.rs` and run `cargo run`. You see the three lines from Section 2.

3. Add a new type `Enemy` with a `Position` and a `Health`, and a `Trap` that has only a `Position`. You may not copy any method: all the moving and damage must come from the components. Print where each of them is after moving it.

4. Add a trait `Describe` with two required methods, `label(&self) -> String` and `is_alive(&self) -> bool` (it can ask the `Health` component), and a default method `describe(&self) -> String` that returns the label followed by `" (alive)"` or `" (gone)"`. Implement it for `Player` and `Enemy` only (a trap has no health, so it cannot be alive). Notice that you needed no inheritance: a trait and components were enough.

5. Run the checks and save:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): components and a describe trait"
```

**What happens:** each new entity is assembled from parts that already exist and are tested. Adding `Trap` needed no change to `Player`.

## Exercises

### Exercise 1 — Field, trait, wrapper or enum? (easy)

Which tool fits best?

1. Several services need to write log messages, each with the same logger.
2. `Invoice` and `Customer` can both be printed in a report.
3. An order is always in exactly one of the states open, paid or shipped.

<details>
<summary>Hint</summary>

Is it "something I have", "something I can do", or "one of several cases"?

</details>

<details>
<summary>Answer</summary>

1. Composition: each service holds (or is given) a logger as a field.
2. A trait (for example `Report`) that both types implement.
3. An enum with three variants (Lesson 08.8 goes deeper into states).

Why it works: field = has-a, trait = can-do, enum = one-of.

</details>

### Exercise 2 — Extract a component, and test it (medium)

`Car` and `Boat` repeat the same refuel logic with a limit of 50 liters. Extract a `FuelTank` component with `refuel`, make both types hold one, and **write a test** for `refuel` (once when there is room, once when the tank is nearly full).

```rust
struct Car {
    fuel: u32,
}

struct Boat {
    fuel: u32,
}

impl Car {
    fn refuel(&mut self, liters: u32) {
        self.fuel = (self.fuel + liters).min(50);
    }
}

impl Boat {
    fn refuel(&mut self, liters: u32) {
        self.fuel = (self.fuel + liters).min(50);
    }
}

fn main() {
    let mut car = Car { fuel: 45 };
    let mut boat = Boat { fuel: 10 };
    car.refuel(20);
    boat.refuel(20);
    println!("{} {}", car.fuel, boat.fuel);
}
```

<details>
<summary>Hint</summary>

The duplicated knowledge is "a tank holds at most 50 liters". Put the number in an associated constant of `FuelTank`.

</details>

<details>
<summary>Answer</summary>

```rust
struct FuelTank {
    liters: u32,
}

impl FuelTank {
    const CAPACITY: u32 = 50;

    fn refuel(&mut self, liters: u32) {
        self.liters = (self.liters + liters).min(Self::CAPACITY);
    }
}

struct Car {
    tank: FuelTank,
}

struct Boat {
    tank: FuelTank,
}

fn main() {
    let mut car = Car {
        tank: FuelTank { liters: 45 },
    };
    let mut boat = Boat {
        tank: FuelTank { liters: 10 },
    };
    car.tank.refuel(20);
    boat.tank.refuel(20);
    println!("{} {}", car.tank.liters, boat.tank.liters);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn refuel_stops_at_capacity() {
        let mut tank = FuelTank { liters: 45 };
        tank.refuel(20);
        assert_eq!(tank.liters, 50);
    }

    #[test]
    fn refuel_adds_when_there_is_room() {
        let mut tank = FuelTank { liters: 10 };
        tank.refuel(20);
        assert_eq!(tank.liters, 30);
    }
}
```

Why it works: the limit now lives in one place (DRY, "Don't Repeat Yourself", Lesson 08.1). `Car` and `Boat` both *have a* tank, and the tests cover the shared logic once.

</details>

### Exercise 3 — Write a wrapper (hard)

Write `UppercaseWriter<W: Write>` that turns every ASCII letter (a basic English letter) to uppercase before passing the bytes to the inner writer. **Write a test** that wraps a `Vec<u8>` and checks the bytes.

<details>
<summary>Hint</summary>

Implement `write` and `flush` as in `CountingWriter`. In `write`, build an uppercased copy with `to_ascii_uppercase`, send it with `write_all`, and return `buf.len()` (the number of bytes you accepted, not the length of the copy). A `Vec<u8>` implements `Write`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io::{self, Write};

struct UppercaseWriter<W: Write> {
    inner: W,
}

impl<W: Write> Write for UppercaseWriter<W> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        let upper: Vec<u8> = buf.iter().map(|b| b.to_ascii_uppercase()).collect();
        self.inner.write_all(&upper)?;
        Ok(buf.len())
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}

fn main() -> io::Result<()> {
    let mut out = UppercaseWriter {
        inner: io::stdout(),
    };
    writeln!(out, "quiet text")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn text_is_uppercased_before_it_reaches_the_inner_writer() {
        let mut out = UppercaseWriter { inner: Vec::new() };
        write!(out, "abc 123").unwrap();
        assert_eq!(out.inner, b"ABC 123");
    }
}
```

Why it works: the wrapper changes the data and delegates the rest. The test reads the inner field `out.inner` directly, which is possible because the test module is in the same file. Returning `buf.len()` tells the caller that all of its bytes were accepted.

</details>

## Quiz

1. What does "has-a" mean in design?

<details>
<summary>Answer</summary>

One type contains another as a field and uses it: composition.

</details>

2. How does a Rust trait share code without sharing data?

<details>
<summary>Answer</summary>

By default methods: they are written once in the trait and use only the required methods of the trait.

</details>

3. Why does `announce(&dog)` fail in the `Deref` example although `dog.name()` works?

<details>
<summary>Answer</summary>

Method lookup follows `Deref`, but a trait bound needs the type itself to implement the trait. `Dog` does not implement `Speak`; only `Animal` does.

</details>

4. Name two standard-library wrappers.

<details>
<summary>Answer</summary>

`BufWriter` and `BufReader`; also iterator adapters such as `Map` and `Filter`.

</details>

## Summary

- Rust has no inheritance between structs; it gives you composition, traits, wrappers and enums instead.
- **Composition** (has-a): put small cohesive components into fields. Mix and match is easy, and each part is tested alone.
- **Traits** share behaviour: required methods plus default methods; data stays in each struct.
- A **wrapper** extends an existing type: hold it, offer the same trait, delegate, and add your behaviour.
- `Deref` is for smart pointers; using it to imitate inheritance is an anti-pattern. Delegate explicitly.
- Choose by the question: have (field), can do (trait), around (wrapper), one of (enum).
- Do not over-compose: skip one-field "components", hand-written forwarding of dozens of methods and deep field chains.

## Sources and further reading

- [The Rust Programming Language — 18.1 Characteristics of Object-Oriented Languages](https://doc.rust-lang.org/book/ch18-01-what-is-oo.html) — the official discussion of inheritance and what Rust offers instead.
- [Rust Design Patterns — Deref polymorphism](https://rust-unofficial.github.io/patterns/anti_patterns/deref.html) — why `Deref` should not be used to imitate inheritance.
- [Rust Design Patterns — Newtype](https://rust-unofficial.github.io/patterns/patterns/behavioural/newtype.html) — wrapping a type to give it new behaviour.
- [std::io::BufWriter](https://doc.rust-lang.org/std/io/struct.BufWriter.html) — a real wrapper from the standard library.

---

[🏠 Start](../START-HERE.md) · [← 08.3 The SOLID principles](03-solid-principles.md) · [📋 Phase 08](README.md) · [08.5 Type-driven design: make invalid states impossible →](05-type-driven-design.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
