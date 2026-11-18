<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.8 State machines and the typestate pattern](08-state-machines-and-typestate.md) · [📋 Phase 08](README.md) · [08.10 Anti-patterns and code smells →](10-anti-patterns-and-code-smells.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/09-strategy-and-dependency-injection.md) <!-- nav -->

# Lesson 08.9 — Strategy and dependency injection

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 9 of 17 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The **strategy** pattern: swapping one algorithm for another behind a common interface.
- Four ways to write it in Rust (enum, generic trait, trait object, closure) and how to choose.
- **Dependency injection**: giving a type the things it depends on, instead of letting it create them.
- How injection makes code testable with fakes, with no mocking framework.
- The costs of both ideas, and when to skip them.

## Before you start

- You finished [Lesson 08.8 — State machines and the typestate pattern](08-state-machines-and-typestate.md).
- You know traits and generics ([Lessons 05.1 and 05.2](../05-rust-intermediate/01-generics.md)), closures with `Fn` ([Lesson 05.4](../05-rust-intermediate/04-closures.md)), and `dyn Trait` with static and dynamic dispatch ([Lesson 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)).
- You know the dependency inversion principle from [Lesson 08.3](03-solid-principles.md). This lesson is its practical form.

## New words

| Word | Meaning |
|---|---|
| **strategy** | One of several interchangeable ways to do the same job (for example, ways to compute a delay). The *strategy pattern* lets the code choose or swap the way without changing the code that uses it. |
| **backoff** | A waiting time between retries of a failed operation. The wait usually grows with every attempt, so a struggling server is not flooded with requests. |
| **dependency** | Something a piece of code needs in order to do its work: another type, a clock, a mail sender, a database. |
| **dependency injection** (DI) | Giving a type its dependencies from the outside (through `new` or a parameter), instead of letting it create them itself. |
| **composition root** | The one place in a program (usually `main`) that decides which real implementations are used and connects them. |
| **test double** | A stand-in for a real dependency, used in a test. |
| **fake** | A test double with a simple but working implementation, for example an in-memory store instead of a database. |
| **stub** | A test double that returns fixed answers (or does nothing). |
| **mock** | A test double that also checks how it was called (which methods, with which arguments, how many times). |

## 1. Strategy: swap the algorithm

A program retries a failed network call. Before retry number 0, 1, 2, ... it waits some **milliseconds**. How long? There are several reasonable rules:

- **Constant**: always 200 ms.
- **Exponential**: 100 ms, then 200, 400, 800... (doubling).

Each rule is a **strategy**: a way to do the same job, `delay for attempt n`. The code that retries should not care which rule is in use. Here is the simplest Rust version, with an enum:

```rust
enum Backoff {
    Constant(u64),
    Exponential { first: u64, factor: u64 },
}

impl Backoff {
    // How long to wait before retry number `attempt` (0, 1, 2, ...), in milliseconds.
    fn delay_ms(&self, attempt: u32) -> u64 {
        match self {
            Backoff::Constant(ms) => *ms,
            Backoff::Exponential { first, factor } => first * factor.pow(attempt),
        }
    }
}

fn schedule(strategy: &Backoff, attempts: u32) -> Vec<u64> {
    (0..attempts).map(|n| strategy.delay_ms(n)).collect()
}

fn main() {
    println!("{:?}", schedule(&Backoff::Constant(200), 4));
    println!(
        "{:?}",
        schedule(
            &Backoff::Exponential {
                first: 100,
                factor: 2
            },
            4
        )
    );
}
```

**What happens:**

1. `Backoff` has one variant per strategy. The data of each strategy (the constant, or the first delay and the factor) lives in its variant.
2. `delay_ms` is a `match`. Each arm is the formula of one strategy.
3. `schedule` is the "user" of the strategy. It asks for the delay of attempts 0, 1, 2 and 3 and does not know the formulas.
4. `main` prints the schedule for two strategies.

```text
[200, 200, 200, 200]
[100, 200, 400, 800]
```

For a closed set of strategies that you write yourself, this is the best choice: simple, checked by the compiler (a new variant forces every `match` to be updated, Lesson 08.3), and fast.

## 2. Strategy with a trait, or with a closure

When other code (another crate, a plugin, a test) must add strategies, or when the choice happens at run time, use a trait. The same example, with three ways to use it:

```rust
trait Backoff {
    fn delay_ms(&self, attempt: u32) -> u64;
}

struct Constant(u64);
struct Exponential {
    first: u64,
    factor: u64,
}

impl Backoff for Constant {
    fn delay_ms(&self, _attempt: u32) -> u64 {
        self.0
    }
}

impl Backoff for Exponential {
    fn delay_ms(&self, attempt: u32) -> u64 {
        self.first * self.factor.pow(attempt)
    }
}

// Generic: the compiler makes one copy of this function for every strategy type.
fn schedule(strategy: &impl Backoff, attempts: u32) -> Vec<u64> {
    (0..attempts).map(|n| strategy.delay_ms(n)).collect()
}

// Dynamic: one copy; the method is found at run time through a pointer.
fn schedule_dyn(strategy: &dyn Backoff, attempts: u32) -> Vec<u64> {
    (0..attempts).map(|n| strategy.delay_ms(n)).collect()
}

// Trait object: the strategy is chosen while the program runs.
fn from_name(name: &str) -> Option<Box<dyn Backoff>> {
    match name {
        "constant" => Some(Box::new(Constant(200))),
        "exponential" => Some(Box::new(Exponential {
            first: 100,
            factor: 2,
        })),
        _ => None,
    }
}

// Closure: the strategy is just a function.
fn schedule_with(strategy: impl Fn(u32) -> u64, attempts: u32) -> Vec<u64> {
    (0..attempts).map(strategy).collect()
}

fn main() {
    println!(
        "{:?}",
        schedule(
            &Exponential {
                first: 100,
                factor: 2
            },
            4
        )
    );

    for name in ["constant", "exponential", "unknown"] {
        match from_name(name) {
            Some(strategy) => println!("{name}: {:?}", schedule_dyn(strategy.as_ref(), 3)),
            None => println!("{name}: no such strategy"),
        }
    }

    println!(
        "{:?}",
        schedule_with(|attempt| 50 * u64::from(attempt + 1), 4)
    );
}
```

**What happens:**

1. `trait Backoff` is the shared interface. `Constant` and `Exponential` are two strategies; each is a small struct with its own data.
2. `schedule(&impl Backoff, ..)` is **generic**: the compiler makes a separate, fast copy of the function for each type you call it with (static dispatch, Lesson 06.1).
3. `schedule_dyn(&dyn Backoff, ..)` takes a **trait object**: one copy of the function, and the method is found at run time through a pointer (dynamic dispatch).
4. `from_name` turns a text (maybe from a settings file) into a `Box<dyn Backoff>`. This is where a trait object is needed: the type is not known until the program runs.
5. `schedule_with(impl Fn(u32) -> u64, ..)` takes a **closure**: the strategy is just a function, with no new type at all.
6. `main` shows all three.

```text
[100, 200, 400, 800]
constant: [200, 200, 200]
exponential: [100, 200, 400]
unknown: no such strategy
[50, 100, 150, 200]
```

How to choose:

| Form | Choose it when | Cost |
|---|---|---|
| **Enum** | You know all strategies; they are yours | Adding a strategy edits the enum |
| **Generic** (`impl Trait`) | Callers choose the strategy in their code, and speed matters | A copy of the function per type; type parameters spread to the callers |
| **Trait object** (`dyn Trait`) | The strategy is chosen at run time (configuration, user input), or you need a list of different strategies | A pointer call, and usually a `Box` |
| **Closure** (`impl Fn`) | The strategy is one small function | No named type; not ideal if the strategy has several methods or needs a name in errors |

> 📏 **Convention:** Start with the enum. Move to a trait when outsiders must add strategies. Use a closure when the "interface" is a single function. This follows the open/closed discussion of Lesson 08.3.

## 3. Dependency injection

A **dependency** is anything your code needs from outside itself. Here is a signup function. It creates its own user list and sends its own mail:

```rust
use std::collections::HashMap;

fn register(email: &str) -> Result<(), String> {
    // The function creates everything it needs, itself.
    let mut users: HashMap<String, bool> = HashMap::new();
    if users.contains_key(email) {
        return Err(format!("{email} is already registered"));
    }
    users.insert(email.to_string(), true);
    println!("SENDING MAIL to {email}: welcome!");
    Ok(())
}

fn main() {
    println!("{:?}", register("ana@mail.org"));
}
```

```text
SENDING MAIL to ana@mail.org: welcome!
Ok(())
```

It works, but notice what is wrong:

- The function **creates** its list of users, so a second call starts with an empty list again (the "already registered" branch can never run), and the real database could not be plugged in without editing the function.
- The mail is sent with `println!` inside the function. A test cannot ask "was a welcome mail sent to this address?" It could only read the screen.
- Every change to mail sending (SMTP, the Simple Mail Transfer Protocol for sending email; a queue; a different text) edits the signup logic. Two reasons to change in one function (Lesson 08.3).

**Dependency injection** fixes this with one rule: **a type receives its dependencies; it does not create them.** In Rust that means: describe what you need with a trait, and take it as a parameter of `new`. No framework is needed.

```rust
use std::collections::HashSet;

trait Mailer {
    fn send(&self, to: &str, text: &str);
}

trait UserStore {
    fn contains(&self, email: &str) -> bool;
    fn add(&mut self, email: &str);
}

// The real parts (the "production" implementations).
struct ConsoleMailer;

impl Mailer for ConsoleMailer {
    fn send(&self, to: &str, text: &str) {
        println!("MAIL to {to}: {text}");
    }
}

#[derive(Default)]
struct MemoryStore {
    emails: HashSet<String>,
}

impl UserStore for MemoryStore {
    fn contains(&self, email: &str) -> bool {
        self.emails.contains(email)
    }
    fn add(&mut self, email: &str) {
        self.emails.insert(email.to_string());
    }
}

// The service receives what it needs through `new`.
struct SignupService<S: UserStore, M: Mailer> {
    store: S,
    mailer: M,
}

impl<S: UserStore, M: Mailer> SignupService<S, M> {
    fn new(store: S, mailer: M) -> Self {
        SignupService { store, mailer }
    }

    fn register(&mut self, email: &str) -> Result<(), String> {
        if self.store.contains(email) {
            return Err(format!("{email} is already registered"));
        }
        self.store.add(email);
        self.mailer.send(email, "welcome!");
        Ok(())
    }
}

// The composition root: the one place that picks the real parts.
fn main() {
    let mut service = SignupService::new(MemoryStore::default(), ConsoleMailer);
    println!("{:?}", service.register("ana@mail.org"));
    println!("{:?}", service.register("ana@mail.org"));
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;
    use std::rc::Rc;

    // A fake that remembers what it was asked to send. Clones share the same list.
    #[derive(Clone, Default)]
    struct RecordingMailer {
        sent: Rc<RefCell<Vec<(String, String)>>>,
    }

    impl Mailer for RecordingMailer {
        fn send(&self, to: &str, text: &str) {
            self.sent
                .borrow_mut()
                .push((to.to_string(), text.to_string()));
        }
    }

    #[test]
    fn registering_sends_one_welcome_mail() {
        let mailer = RecordingMailer::default();
        let mut service = SignupService::new(MemoryStore::default(), mailer.clone());
        assert_eq!(service.register("bo@mail.org"), Ok(()));
        assert_eq!(
            *mailer.sent.borrow(),
            vec![("bo@mail.org".to_string(), "welcome!".to_string())]
        );
    }

    #[test]
    fn a_duplicate_is_rejected_and_sends_nothing() {
        let mailer = RecordingMailer::default();
        let mut service = SignupService::new(MemoryStore::default(), mailer.clone());
        service.register("bo@mail.org").unwrap();
        assert!(service.register("bo@mail.org").is_err());
        assert_eq!(mailer.sent.borrow().len(), 1);
    }
}
```

**What happens:**

1. `Mailer` and `UserStore` are the traits the service needs. They describe *what* the service needs, not *how* it is done (Lesson 08.3, dependency inversion).
2. `ConsoleMailer` and `MemoryStore` are simple real implementations. In a real application, a `MemoryStore` would be replaced by a database store; the service would not change.
3. `SignupService<S, M>` stores a store and a mailer, which are **injected** through `new`. It never creates one.
4. `register` contains only the signup rules: reject duplicates, add the user, send the welcome mail.
5. `main` is the **composition root**: the one place that chooses `MemoryStore` and `ConsoleMailer` and connects them to the service.
6. The tests build the same service with a `RecordingMailer`, a fake that stores every call in a list shared through `Rc<RefCell<..>>`. The test then reads the list.

```text
MAIL to ana@mail.org: welcome!
Ok(())
Err("ana@mail.org is already registered")
```

```text
running 2 tests
test tests::a_duplicate_is_rejected_and_sends_nothing ... ok
test tests::registering_sends_one_welcome_mail ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The first test proves the welcome mail was sent exactly once to the right address. The second proves that a duplicate sends nothing. Neither test touched a screen or a network.

> 🧪 **Testing:** A fake mailer is enough for most tests. Vocabulary: a **fake** has a simple working implementation (the in-memory store, the recording mailer); a **stub** returns fixed values or does nothing; a **mock** also verifies the calls. Mocking crates such as `mockall` (version 0.15 when this course was checked) generate mocks for you, but start with hand-written fakes: they are shorter than you think, and they test behaviour instead of the exact order of calls, which makes tests less fragile.

## 4. Generics or trait objects for the injected parts?

`SignupService<S, M>` is generic: the compiler knows the exact types, and calls are direct. The price is that the type parameters appear in the type's name. Any function that takes a `SignupService` must write `SignupService<S, M>` too (or be generic), and the parameters spread through the program.

The alternative is trait objects as fields: `store: Box<dyn UserStore>, mailer: Box<dyn Mailer>`. Then the type has no parameters, the real parts can be chosen at run time, and the price is a pointer call per method (usually unimportant next to a network call). Exercise 3 converts the service.

| | Generic parameters | `Box<dyn Trait>` fields |
|---|---|---|
| Type of the service | `SignupService<S, M>` | `SignupService` |
| Choosing the parts at run time | Hard | Easy |
| Call speed | Direct, can be inlined | Through a pointer |
| Good for | Libraries, performance-sensitive code | Applications, plugins, configuration-driven choices |

> ⚡ **Performance:** The difference between static and dynamic dispatch is a few nanoseconds per call. When the dependency does I/O (Input/Output, talking to the outside world: a database, the network), it is lost in the noise. Pick the form that keeps the code simple, and measure before you change it (Phase 19).

## 5. When NOT to use these patterns

- **Only one implementation, and it is not I/O.** A pure function such as `total_cents` does not need a `Calculator` trait. Inject things that touch the outside world (time, files, network, randomness) or that really vary.
- **A trait for every struct.** If the trait will never have a second implementation, not even a fake, delete it (YAGNI, "You Aren't Gonna Need It", Lesson 08.1).
- **Generics everywhere.** Eight type parameters in a struct is a sign to use `dyn` at that boundary, or to group related dependencies into one trait.
- **A "DI container" or framework.** Large libraries for this exist, but in Rust, plain constructor parameters and one composition root are almost always enough.
- **Hidden globals.** A `static` variable or a function that secretly reads the environment is a hidden dependency. Pass the value in (or the trait that supplies it).
- **Mocks that copy the implementation.** A test that checks "method A was called, then method B, then A again" breaks at every refactoring. Test the result that matters.

## Common mistakes

**Mistake 1: creating the dependency inside.** `fn register(&self) { let mailer = SmtpMailer::new(); ... }`. The service cannot be tested without sending real mail. Inject it.

**Mistake 2: passing the whole world.** A service that takes a giant `Context` with everything is easy to call and impossible to understand. Take only the traits you need (Lesson 08.2, ask for the least).

**Mistake 3: injecting too early.** Do not add the trait on the first day if there is a single caller and no I/O; add it when the first test or the second implementation needs it. It is a small refactoring then (Lesson 08.13).

**Mistake 4: a fake that is wrong.** A fake store that behaves differently from the real one (for example, it allows duplicates) makes tests pass and production fail. Keep fakes simple and test their rules too.

## Try it yourself

1. Create the package and copy the dependency-injection example of Section 3 (the `SignupService` with its `tests` module) into `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p08-strategy
cd code/p08-strategy
```

2. Run `cargo run`, then `cargo test`. The two tests pass.

3. In the `tests` module, add a stub that does nothing and a third test that uses it:

```rust
struct SilentMailer;

impl Mailer for SilentMailer {
    fn send(&self, _to: &str, _text: &str) {}
}

#[test]
fn a_registered_email_is_stored() {
    let mut service = SignupService::new(MemoryStore::default(), SilentMailer);
    service.register("bo@mail.org").unwrap();
    assert!(service.store.contains("bo@mail.org"));
}
```

4. Run `cargo test`. All three tests pass:

```text
running 3 tests
test tests::a_duplicate_is_rejected_and_sends_nothing ... ok
test tests::a_registered_email_is_stored ... ok
test tests::registering_sends_one_welcome_mail ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

5. Save:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): signup service with injected parts"
```

**What happens:** The test can read the private field `service.store` because the `tests` module is a child of the module where the type is defined. Notice that you built the new test without touching the service: injection made the test cheap.

## Exercises

### Exercise 1 — Which form? (easy)

Choose between enum, generic trait, trait object and closure:

1. A retry function needs a way to decide if an error is worth retrying. The rule is one small function.
2. An application loads a "storage backend" name from a settings file: `"memory"` or `"disk"`.
3. Three discount rules (none, student, senior) are all written by you, and nobody else will add one.

<details>
<summary>Hint</summary>

Who adds the strategies? When is the choice made? How many methods does the interface have?

</details>

<details>
<summary>Answer</summary>

1. A closure (`impl Fn(&Error) -> bool`): the interface is one function.
2. A trait object (`Box<dyn Storage>`): the choice is made at run time from text.
3. An enum: a closed set that you own, checked by the compiler.

Why it works: the questions "who adds strategies" and "when is the choice made" decide the form.

</details>

### Exercise 2 — Inject the clock (medium)

A greeting function must say `"Good morning"` before noon and `"Good day"` from noon. If it calls the system clock itself, it cannot be tested. Write a trait `Clock { fn hour(&self) -> u32 }`, a real `SystemClock`, the function `greeting(&impl Clock)`, and **tests** with a `FixedClock`.

<details>
<summary>Hint</summary>

The hour in UTC (Coordinated Universal Time, the world reference time) is `(seconds_since_epoch / 3600) % 24`. Get the seconds with `SystemTime::now().duration_since(UNIX_EPOCH)`. The tests need the border hours 0, 11, 12 and 23.

</details>

<details>
<summary>Answer</summary>

```rust
use std::time::{SystemTime, UNIX_EPOCH};

trait Clock {
    fn hour(&self) -> u32; // 0 to 23, in UTC
}

struct SystemClock;

impl Clock for SystemClock {
    fn hour(&self) -> u32 {
        let secs = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();
        ((secs / 3600) % 24) as u32
    }
}

fn greeting(clock: &impl Clock) -> String {
    if clock.hour() < 12 {
        "Good morning".to_string()
    } else {
        "Good day".to_string()
    }
}

fn main() {
    // The result depends on the time you run it.
    println!("{}", greeting(&SystemClock));
}

#[cfg(test)]
mod tests {
    use super::*;

    struct FixedClock(u32);

    impl Clock for FixedClock {
        fn hour(&self) -> u32 {
            self.0
        }
    }

    #[test]
    fn before_noon_it_is_morning() {
        assert_eq!(greeting(&FixedClock(0)), "Good morning");
        assert_eq!(greeting(&FixedClock(11)), "Good morning");
    }

    #[test]
    fn from_noon_it_is_day() {
        assert_eq!(greeting(&FixedClock(12)), "Good day");
        assert_eq!(greeting(&FixedClock(23)), "Good day");
    }
}
```

Why it works: `greeting` receives the clock, so the test supplies a `FixedClock` and the result no longer depends on the time of day. Time, randomness and the file system are the most common things to inject.

</details>

### Exercise 3 — Choose the mailer at run time (hard)

Change the service so that its mailer is a `Box<dyn Mailer>`, and write `mailer_from_config(name) -> Option<Box<dyn Mailer>>` that understands `"console"` and `"silent"`. **Write tests** for the known and the unknown names, and one test that the service works with the silent mailer.

<details>
<summary>Hint</summary>

The service field becomes `mailer: Box<dyn Mailer>`, and `new` takes a `Box<dyn Mailer>`. `main` calls `mailer_from_config("console").expect("unknown mailer")`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashSet;

trait Mailer {
    fn send(&self, to: &str, text: &str);
}

struct ConsoleMailer;
struct SilentMailer;

impl Mailer for ConsoleMailer {
    fn send(&self, to: &str, text: &str) {
        println!("MAIL to {to}: {text}");
    }
}

impl Mailer for SilentMailer {
    fn send(&self, _to: &str, _text: &str) {}
}

// The name usually comes from a settings file or an environment variable.
fn mailer_from_config(name: &str) -> Option<Box<dyn Mailer>> {
    match name {
        "console" => Some(Box::new(ConsoleMailer)),
        "silent" => Some(Box::new(SilentMailer)),
        _ => None,
    }
}

struct SignupService {
    known: HashSet<String>,
    mailer: Box<dyn Mailer>,
}

impl SignupService {
    fn new(mailer: Box<dyn Mailer>) -> SignupService {
        SignupService {
            known: HashSet::new(),
            mailer,
        }
    }

    fn register(&mut self, email: &str) -> bool {
        let is_new = self.known.insert(email.to_string());
        if is_new {
            self.mailer.send(email, "welcome!");
        }
        is_new
    }
}

fn main() {
    let mailer = mailer_from_config("console").expect("unknown mailer");
    let mut service = SignupService::new(mailer);
    println!("{}", service.register("ana@mail.org"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_names_are_found_and_unknown_are_not() {
        assert!(mailer_from_config("console").is_some());
        assert!(mailer_from_config("silent").is_some());
        assert!(mailer_from_config("pigeon").is_none());
    }

    #[test]
    fn registering_twice_is_rejected_with_any_mailer() {
        let mut service = SignupService::new(mailer_from_config("silent").unwrap());
        assert!(service.register("a@b.c"));
        assert!(!service.register("a@b.c"));
    }
}
```

Why it works: the service no longer has type parameters, and the choice of the mailer happens in one place (`mailer_from_config`), from data. This is the form to use when settings decide the parts.

</details>

## Quiz

1. What is a strategy?

<details>
<summary>Answer</summary>

One of several interchangeable ways to do the same job, used through a common interface.

</details>

2. Name the four forms of strategy in Rust.

<details>
<summary>Answer</summary>

An enum, a generic parameter with a trait bound, a trait object (`dyn Trait`), and a closure.

</details>

3. What is dependency injection, in one sentence?

<details>
<summary>Answer</summary>

A type receives the things it depends on (usually as traits, through `new`), instead of creating them itself.

</details>

4. What is the composition root?

<details>
<summary>Answer</summary>

The one place, usually `main`, that chooses the real implementations and connects them.

</details>

5. Why is it bad for a test to verify the exact order of calls on a mock?

<details>
<summary>Answer</summary>

It ties the test to the implementation, so a harmless refactoring breaks the test. Check the result that matters.

</details>

## Summary

- A **strategy** is one of several interchangeable ways to do a job. In Rust: enum (closed set), generic trait (callers choose, fast), `dyn Trait` (chosen at run time), closure (one function).
- **Dependency injection**: a type receives its dependencies (as traits) through `new` or parameters; `main` is the composition root that wires the real parts.
- Injection makes tests easy: use hand-written **fakes** and **stubs**; you do not need a framework.
- Generic parameters are fast but spread through the types; `Box<dyn Trait>` fields keep types simple and allow choices at run time.
- Inject what touches the outside world (time, files, network, randomness) or really varies. Do not add a trait for pure code or for a single implementation with no test need.
- Keep tests about results, not about the exact sequence of calls.

## Sources and further reading

- [Rust Design Patterns — Strategy](https://rust-unofficial.github.io/patterns/patterns/behavioural/strategy.html) — the strategy pattern in Rust, with traits and with closures.
- [The Rust Programming Language — 18.2 Using Trait Objects](https://doc.rust-lang.org/book/ch18-02-trait-objects.html) — trait objects and dynamic dispatch.
- [The Rust Programming Language — 13.1 Closures](https://doc.rust-lang.org/book/ch13-01-closures.html) — closures and the `Fn` traits.
- [Rust API Guidelines — Flexibility](https://rust-lang.github.io/api-guidelines/flexibility.html) — taking generic parameters so that callers can plug in what they have.
- [mockall on docs.rs](https://docs.rs/mockall/latest/mockall/) — a crate that generates mocks, for when hand-written fakes are not enough.

---

[🏠 Start](../START-HERE.md) · [← 08.8 State machines and the typestate pattern](08-state-machines-and-typestate.md) · [📋 Phase 08](README.md) · [08.10 Anti-patterns and code smells →](10-anti-patterns-and-code-smells.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
