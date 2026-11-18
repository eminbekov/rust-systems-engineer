<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.5 Type-driven design: make invalid states impossible](05-type-driven-design.md) · [📋 Phase 08](README.md) · [08.7 RAII and guards →](07-raii-and-guards.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/06-builder-pattern.md) <!-- nav -->

# Lesson 08.6 — The builder pattern

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 6 of 17 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Why a constructor with many parameters is hard to read and to change.
- A simpler first step: `Default` and the struct update syntax `..Default::default()`.
- How to write a **builder**: setter methods that chain, and a `build` method that checks the rules.
- The two builder styles in Rust (consuming `self` and `&mut self`), and the common compiler error of the second style.
- When a builder is too much.

## Before you start

- You finished [Lesson 08.5 — Type-driven design](05-type-driven-design.md): `build()` is a smart constructor.
- You know structs and methods ([Lessons 04.1 and 04.2](../04-structs-enums-collections-errors/01-structs.md)), `Option`, `Result`, and ownership of `self` ([Lesson 03.2](../03-ownership-and-borrowing/02-ownership-rules.md)).
- You know the `Default` trait and `#[derive(...)]` ([Lesson 05.2](../05-rust-intermediate/02-traits.md)).

## New words

| Word | Meaning |
|---|---|
| **builder** | A helper type that collects settings one by one and creates the final value in a last step. |
| **method chaining** | Calling methods one after another in one expression: `a.x().y().z()`. It works when each method returns something the next method can use. |
| **struct update syntax** | `Server { port: 9000, ..other }` creates a struct from `other`, replacing the listed fields. |
| **consuming builder** | A builder whose methods take `self` by value and return it (or a new one). |
| **`&mut` builder** | A builder whose methods take `&mut self` and return `&mut Self`. |
| **optional parameter** | A setting that has a sensible default, so the caller may leave it out. Rust has no optional function arguments; builders and `Default` fill this gap. |
| **named argument** | An argument given as `name: value` at the call. Rust has none either; builder methods read like named arguments (`.port(8443)`). |

## 1. The problem: a constructor with many parameters

A server needs a host, a port, a limit of connections, and some flags. The obvious constructor:

```rust
#[allow(dead_code)] // the fields are only printed with {:?}
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
    tls: bool,
    verbose: bool,
    timeout_secs: u32,
}

impl Server {
    fn new(
        host: &str,
        port: u16,
        max_connections: u32,
        tls: bool,
        verbose: bool,
        timeout_secs: u32,
    ) -> Server {
        Server {
            host: host.to_string(),
            port,
            max_connections,
            tls,
            verbose,
            timeout_secs,
        }
    }
}

fn main() {
    // What do `true, false` and `30` mean here? Which one is `tls`?
    let server = Server::new("localhost", 8080, 100, true, false, 30);
    println!("{server:?}");
}
```

```text
Server { host: "localhost", port: 8080, max_connections: 100, tls: true, verbose: false, timeout_secs: 30 }
```

Look at the call: `Server::new("localhost", 8080, 100, true, false, 30)`. Which one is `tls` (Transport Layer Security, encrypted connections) and which is `verbose` (print extra messages)? Are `100` and `30` in the right order? The compiler cannot tell you, because `true` and `false` are both `bool`, and `100` and `30` are both numbers. And every time you add a seventh setting, all calls everywhere must change.

Rust has no named or optional arguments, so people use other tools.

## 2. First try the simplest tool: `Default` and struct update

If all fields have sensible defaults, you do not need a builder at all. Implement `Default` (Lesson 05.2) and use **struct update syntax**:

```rust
#[allow(dead_code)] // the fields are only printed with {:?}
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
    tls: bool,
}

impl Default for Server {
    fn default() -> Server {
        Server {
            host: "localhost".to_string(),
            port: 8080,
            max_connections: 100,
            tls: false,
        }
    }
}

fn main() {
    let plain = Server::default();
    let secure = Server {
        port: 8443,
        tls: true,
        ..Default::default()
    };
    println!("{plain:?}");
    println!("{secure:?}");
}
```

**What happens:**

1. `impl Default for Server` defines the usual values.
2. `Server::default()` creates a server with all defaults.
3. `Server { port: 8443, tls: true, ..Default::default() }` sets two fields by name and takes the rest from the default value. This reads like named arguments, and a new field with a default does not break existing calls.

```text
Server { host: "localhost", port: 8080, max_connections: 100, tls: false }
Server { host: "localhost", port: 8443, max_connections: 100, tls: true }
```

> 📏 **Convention:** Use `Default` plus `..Default::default()` first. It is the simplest thing (KISS, "Keep It Simple, Stupid", Lesson 08.1) and needs no extra type. Move to a builder only when this is not enough.

When is it not enough?

- Some fields are **required** and have no sensible default (a host).
- The values must be **checked** before the value exists (the port must not be zero) and you want `Result`, not a broken struct.
- Some values are **computed** from the settings, or the final type has private fields.
- The setting steps are **repeatable** or conditional (add a header, add another header, ...).

## 3. The builder

A builder is a separate type with the same settings, with one method per setting, and a final `build`. Each setter returns the builder so that calls chain:

```rust
#[allow(dead_code)] // the fields are only printed with {:?}
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    max_connections: u32,
    tls: bool,
}

#[derive(Debug, PartialEq)]
enum BuildError {
    EmptyHost,
    ZeroPort,
    NoConnections,
}

struct ServerBuilder {
    host: String,
    port: u16,
    max_connections: u32,
    tls: bool,
}

impl ServerBuilder {
    // The host is required, so it is a parameter of `new`.
    fn new(host: impl Into<String>) -> ServerBuilder {
        ServerBuilder {
            host: host.into(),
            port: 8080,
            max_connections: 100,
            tls: false,
        }
    }

    fn port(mut self, port: u16) -> ServerBuilder {
        self.port = port;
        self
    }

    fn max_connections(mut self, count: u32) -> ServerBuilder {
        self.max_connections = count;
        self
    }

    fn tls(mut self, enabled: bool) -> ServerBuilder {
        self.tls = enabled;
        self
    }

    // `build` checks the rules once and returns the finished value.
    fn build(self) -> Result<Server, BuildError> {
        if self.host.trim().is_empty() {
            return Err(BuildError::EmptyHost);
        }
        if self.port == 0 {
            return Err(BuildError::ZeroPort);
        }
        if self.max_connections == 0 {
            return Err(BuildError::NoConnections);
        }
        Ok(Server {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
            tls: self.tls,
        })
    }
}

fn main() {
    let server = ServerBuilder::new("localhost").port(8443).tls(true).build();
    println!("{server:?}");

    let broken = ServerBuilder::new("localhost").max_connections(0).build();
    println!("{broken:?}");
}
```

**What happens:**

1. `ServerBuilder::new(host)` takes only what is required. `impl Into<String>` lets the caller pass a `&str` or a `String` (the parameter is converted with `.into()`).
2. Every other setting has a default in `new`, and a setter. A setter takes `mut self` (the builder by value), changes one field and returns `self`. That is why `.port(8443).tls(true)` chains: each call consumes the old builder and gives back the same one, changed.
3. `build(self)` is the **smart constructor** of Lesson 08.5. It checks the rules and returns `Result<Server, BuildError>`. A `Server` that exists is always valid.
4. `main` builds a good server, and a bad one (zero connections), and prints both results.

```text
Ok(Server { host: "localhost", port: 8443, max_connections: 100, tls: true })
Err(NoConnections)
```

The call `ServerBuilder::new("localhost").port(8443).tls(true).build()` reads almost like English, every value has a name, and the compiler no longer lets you swap a `bool` for a `u32`.

> 📏 **Convention:** The standard names are: `Type::builder()` or `TypeBuilder::new()` to start, one method per setting named like the setting (not `set_port`), and `build()` to finish. The Rust API Guidelines (API: Application Programming Interface, what a library offers to other code; Lesson 08.11) call this the builder pattern (item C-BUILDER).

> 🧪 **Testing:** Test `build` as you test any smart constructor: one test for the defaults, one for every rule that makes it fail. You will write such tests in the exercises.

## 4. Two styles: `self` and `&mut self`

The builder in Section 3 is a **consuming** builder: every method takes `self` by value. The standard library has the other style as well: `std::process::Command` and `std::fs::OpenOptions` use `&mut self` and return `&mut Self`, while `std::thread::Builder` consumes `self`.

| | Consuming (`self -> Self`) | `&mut` (`&mut self -> &mut Self`) |
|---|---|---|
| One chain ending in `build()` | Works | Works |
| Configure in steps, with `if` or loops | Needs `builder = builder.x()` | Natural: `builder.x();` |
| Build several values from one builder | Needs `Clone` | `build(&self)` can be called many times |
| `build` can move fields out | Yes, no copies needed | No, it has to clone the data |

The `&mut` style has a well-known trap:

```rust
struct Server {
    port: u16,
}

struct ServerBuilder {
    port: u16,
}

impl ServerBuilder {
    fn new() -> ServerBuilder {
        ServerBuilder { port: 8080 }
    }
    fn port(&mut self, port: u16) -> &mut ServerBuilder {
        self.port = port;
        self
    }
    fn build(&self) -> Server {
        Server { port: self.port }
    }
}

fn main() {
    let builder = ServerBuilder::new().port(9000);
    let server = builder.build();
    println!("{}", server.port);
}
```

```text
error[E0716]: temporary value dropped while borrowed
  --> src/main.rs:23:19
   |
23 |     let builder = ServerBuilder::new().port(9000);
   |                   ^^^^^^^^^^^^^^^^^^^^           - temporary value is freed at the end of this statement
   |                   |
   |                   creates a temporary value which is freed while still in use
24 |     let server = builder.build();
   ...

For more information about this error, try `rustc --explain E0716`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

**What happens:**

1. `ServerBuilder::new()` creates a builder as a **temporary** value; it is not stored in a variable.
2. `.port(9000)` returns `&mut ServerBuilder`, a *reference to the temporary*. The reference is stored in `builder`.
3. At the end of the statement, the temporary is dropped, but `builder` still points to it. The next line would use a dangling reference, so the compiler stops it (Lesson 03.4).

The fixes: finish the chain in one expression (`...port(9000).build()`), or store the builder first in a variable (`let mut builder = ServerBuilder::new();`), as the working version does:

```rust
struct Server {
    port: u16,
}

struct ServerBuilder {
    port: u16,
}

impl ServerBuilder {
    fn new() -> ServerBuilder {
        ServerBuilder { port: 8080 }
    }
    fn port(&mut self, port: u16) -> &mut ServerBuilder {
        self.port = port;
        self
    }
    fn build(&self) -> Server {
        Server { port: self.port }
    }
}

fn main() {
    // Style 1: one chain, in a single statement.
    let first = ServerBuilder::new().port(9000).build();

    // Style 2: keep the builder in a variable, configure it step by step, reuse it.
    let mut builder = ServerBuilder::new();
    let production = true;
    if production {
        builder.port(443);
    }
    let second = builder.build();
    let third = builder.port(444).build();
    println!("{} {} {}", first.port, second.port, third.port);
}
```

```text
9000 443 444
```

`first` uses a single chain. `builder` is kept in a variable, configured with an `if`, and built twice; the second `build` reuses it with another port. Which style to choose? For a builder that `build()` turns into a value with owned data (strings, vectors), the consuming style is simpler and avoids copies. For a builder that configures an action with many optional steps (`Command`), the `&mut` style is comfortable. Both are fine; be consistent inside one crate.

## 5. Builders and the rest of the design

- **Required fields.** Put them into `new`, as `host` above. If a required field is forgotten, the code does not compile. A builder that lets you leave out a required field and complains only in `build()` moves the error from compile time to run time. Lesson 08.8 shows how the typestate pattern can even make `build()` unavailable until required fields are set.
- **Validation lives in `build`.** One place, one `Result`. Setters stay simple.
- **Repeated settings.** A setter may add to a list (`.header("a").header("b")`). Exercise 2 does this with toppings.
- **Many builders.** Writing builders by hand is repetitive. Crates generate them from a struct definition (for example `bon`, which was at version 3.10 when this course was checked). Learn to write one by hand first, so that you understand what the macro writes.

## 6. When a builder is too much

- **Two or three fields, all with defaults.** `Default` plus struct update is enough. A builder here is boilerplate (YAGNI, "You Aren't Gonna Need It", Lesson 08.1).
- **All fields required.** Then a plain constructor `new(a, b, c)` with well-typed parameters (newtypes, Lesson 08.5) is clear, and a builder adds nothing.
- **Public fields with no rules.** A struct with `pub` fields can be created with a struct literal; do not wrap it in a builder "for style".
- **A builder for each struct in the program.** Reserve builders for types that are configured in many ways, in many places: servers, requests, clients.

## Common mistakes

**Mistake 1: forgetting the `build` step.** `let server = ServerBuilder::new("h").port(80);` gives you a `ServerBuilder`, not a `Server`. The compiler reports a type mismatch only where you use it as a server.

**Mistake 2: validation in the setters.** If every setter returns `Result`, chaining becomes painful (`?` after every call). Collect first, check once in `build`.

**Mistake 3: public fields on the finished type plus a builder.** If anybody can still write `Server { port: 0, .. }`, the builder's checks can be bypassed. Make the fields private when the rules matter (Lesson 08.2).

**Mistake 4: the `&mut` builder stored from a temporary.** See Section 4.

## Try it yourself

1. Create the package and copy the first builder example of Section 3 (`ServerBuilder` with its `main`) into `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p08-builder
cd code/p08-builder
```

2. Run it with `cargo run`. You see the two lines from Section 3.

3. Add a new rule: a port below 1024 is allowed only when `tls` is on. Add a variant `BuildError::PrivilegedPortWithoutTls` and the check in `build`:

```rust
if self.port < 1024 && !self.tls {
    return Err(BuildError::PrivilegedPortWithoutTls);
}
```

4. Add the tests below and run `cargo test`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn defaults_are_valid() {
        assert!(ServerBuilder::new("localhost").build().is_ok());
    }

    #[test]
    fn empty_host_is_rejected() {
        assert_eq!(
            ServerBuilder::new("  ").build().unwrap_err(),
            BuildError::EmptyHost
        );
    }

    #[test]
    fn low_port_needs_tls() {
        let no_tls = ServerBuilder::new("h").port(80).build();
        assert_eq!(no_tls.unwrap_err(), BuildError::PrivilegedPortWithoutTls);
        assert!(ServerBuilder::new("h").port(443).tls(true).build().is_ok());
    }
}
```

```text
running 3 tests
test tests::defaults_are_valid ... ok
test tests::empty_host_is_rejected ... ok
test tests::low_port_needs_tls ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

5. Check and save:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): server builder with validation"
```

**What happens:** All the rules of a valid `Server` now live in one function, and each rule has one test. A new rule is a variant, a check and a test.

## Exercises

### Exercise 1 — Default or builder? (easy)

For each type, say whether `Default` with struct update is enough, or a builder is better:

1. `Margins { top, right, bottom, left }`, all numbers with default 0.
2. `HttpRequest` (a request of HTTP, the HyperText Transfer Protocol of the web) with a required URL (Uniform Resource Locator, a web address), an optional method, any number of headers, and the rule that GET requests (the HTTP method for reading data) have no body.
3. `Color { r, g, b }`, where all three values are always given.

<details>
<summary>Hint</summary>

Look for required fields, rules between fields, and repeated settings.

</details>

<details>
<summary>Answer</summary>

1. `Default` plus struct update: every field has a default and there are no rules.
2. A builder: the URL is required, headers repeat, and the GET/body rule needs a check in `build`.
3. Neither: a plain constructor or struct literal, because all values are required.

Why it works: the more required fields, rules and repeated settings, the more a builder pays off.

</details>

### Exercise 2 — A pizza builder with repeated settings (medium)

Write `PizzaBuilder::new(size_cm)` with a setter `topping(name)` that can be called many times, a setter `extra_cheese(bool)`, and `build()` that returns an error when there are more than 5 toppings. **Write tests** for the order of the toppings and for the limit.

<details>
<summary>Hint</summary>

The builder holds a `Vec<String>`; `topping` pushes a name and returns `self`. In the test for the limit, call `topping` in a loop with `builder = builder.topping(name)`.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq)]
struct Pizza {
    size_cm: u32,
    toppings: Vec<String>,
    extra_cheese: bool,
}

#[derive(Debug, PartialEq)]
enum PizzaError {
    TooManyToppings,
}

struct PizzaBuilder {
    size_cm: u32,
    toppings: Vec<String>,
    extra_cheese: bool,
}

impl PizzaBuilder {
    fn new(size_cm: u32) -> PizzaBuilder {
        PizzaBuilder {
            size_cm,
            toppings: Vec::new(),
            extra_cheese: false,
        }
    }

    // Can be called many times: each call adds one topping.
    fn topping(mut self, name: &str) -> PizzaBuilder {
        self.toppings.push(name.to_string());
        self
    }

    fn extra_cheese(mut self, yes: bool) -> PizzaBuilder {
        self.extra_cheese = yes;
        self
    }

    fn build(self) -> Result<Pizza, PizzaError> {
        if self.toppings.len() > 5 {
            return Err(PizzaError::TooManyToppings);
        }
        Ok(Pizza {
            size_cm: self.size_cm,
            toppings: self.toppings,
            extra_cheese: self.extra_cheese,
        })
    }
}

fn main() {
    let pizza = PizzaBuilder::new(30)
        .topping("ham")
        .topping("olives")
        .extra_cheese(true)
        .build();
    println!("{pizza:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn collects_toppings_in_order() {
        let pizza = PizzaBuilder::new(30)
            .topping("ham")
            .topping("olives")
            .build()
            .unwrap();
        assert_eq!(pizza.toppings, vec!["ham", "olives"]);
        assert!(!pizza.extra_cheese);
        assert_eq!(pizza.size_cm, 30);
    }

    #[test]
    fn six_toppings_are_too_many() {
        let mut builder = PizzaBuilder::new(30);
        for name in ["a", "b", "c", "d", "e", "f"] {
            builder = builder.topping(name);
        }
        assert_eq!(builder.build(), Err(PizzaError::TooManyToppings));
    }
}
```

Why it works: each `topping` call consumes the builder and returns it with one more item. The loop in the test shows the cost of the consuming style: you must assign the result back to `builder`.

</details>

### Exercise 3 — Reuse a configured base (hard)

A program creates a primary and a replica connection that share the host and `tls` but differ in the port. Write a consuming `ConnectionBuilder` that can be cloned, so that a shared base is configured once. **Write a test** that two builds from the same base differ only in the port.

<details>
<summary>Hint</summary>

Add `#[derive(Clone)]` to the builder and call `base.clone()` before you use a consuming setter.

</details>

<details>
<summary>Answer</summary>

```rust
#[allow(dead_code)] // the fields are only printed with {:?}
#[derive(Debug)]
struct Connection {
    host: String,
    port: u16,
    tls: bool,
}

#[derive(Clone)]
struct ConnectionBuilder {
    host: String,
    port: u16,
    tls: bool,
}

impl ConnectionBuilder {
    fn new(host: &str) -> ConnectionBuilder {
        ConnectionBuilder {
            host: host.to_string(),
            port: 80,
            tls: false,
        }
    }
    fn port(mut self, port: u16) -> ConnectionBuilder {
        self.port = port;
        self
    }
    fn tls(mut self, tls: bool) -> ConnectionBuilder {
        self.tls = tls;
        self
    }
    fn build(self) -> Connection {
        Connection {
            host: self.host,
            port: self.port,
            tls: self.tls,
        }
    }
}

fn main() {
    // A shared base, configured once.
    let base = ConnectionBuilder::new("db.local").tls(true);
    let primary = base.clone().port(5432).build();
    let replica = base.clone().port(5433).build();
    println!("{primary:?}");
    println!("{replica:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn clones_share_the_base_but_not_later_changes() {
        let base = ConnectionBuilder::new("h").tls(true);
        let a = base.clone().port(1).build();
        let b = base.clone().port(2).build();
        assert_eq!((a.port, b.port), (1, 2));
        assert!(a.tls && b.tls);
        assert_eq!(a.host, "h");
    }
}
```

Why it works: `clone` makes an independent copy of the builder, so the consuming `port` call on the copy does not touch `base`. The test checks that both connections share `host` and `tls`, and differ only in `port`.

</details>

## Quiz

1. Why is `Server::new("h", 80, 100, true, false, 30)` hard to read?

<details>
<summary>Answer</summary>

The arguments have no names and many have the same type, so the reader (and the compiler) cannot tell which is which.

</details>

2. What should you try before writing a builder?

<details>
<summary>Answer</summary>

`Default` plus struct update syntax (`..Default::default()`).

</details>

3. Where does a builder do its validation, and what does it return?

<details>
<summary>Answer</summary>

In `build`, which returns a `Result` with the finished value or a build error.

</details>

4. Why does `let b = ServerBuilder::new().port(9000);` fail for a `&mut self` builder?

<details>
<summary>Answer</summary>

The chain returns a reference to a temporary builder that is dropped at the end of the statement, so `b` would dangle (error E0716).

</details>

5. Name two cases where a builder is too much.

<details>
<summary>Answer</summary>

A struct with few fields that all have defaults (use `Default`); a struct where all fields are required (use a plain constructor).

</details>

## Summary

- A constructor with many parameters is unreadable and fragile; Rust has no named or optional arguments.
- Start with `Default` and `..Default::default()`.
- Use a **builder** when there are required fields, rules to check, computed values or repeated settings: setters chain, `build()` validates and returns `Result`.
- The consuming style (`self`) suits builders that move owned data into the result; the `&mut` style suits step-by-step configuration, but beware of temporaries (E0716).
- Put required values in `new`, rules in `build`, and keep the finished type's fields private if the rules matter.
- Do not write a builder for tiny structs or for structs with no rules.

## Sources and further reading

- [Rust API Guidelines — Builders enable construction of complex values (C-BUILDER)](https://rust-lang.github.io/api-guidelines/type-safety.html) — the official guidance on the builder, including the `self` and `&mut self` styles.
- [Rust Design Patterns — Builder](https://rust-unofficial.github.io/patterns/patterns/creational/builder.html) — the pattern with an example and its trade-offs.
- [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html) — a real `&mut self` builder in the standard library.
- [The Rust Programming Language — 5.1 Defining and Instantiating Structs](https://doc.rust-lang.org/book/ch05-01-defining-structs.html) — struct update syntax.
- [bon on docs.rs](https://docs.rs/bon/latest/bon/) — a crate that generates builders; read it after you can write one by hand.

---

[🏠 Start](../START-HERE.md) · [← 08.5 Type-driven design: make invalid states impossible](05-type-driven-design.md) · [📋 Phase 08](README.md) · [08.7 RAII and guards →](07-raii-and-guards.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
