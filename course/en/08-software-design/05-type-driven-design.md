<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.4 Composition over inheritance](04-composition-over-inheritance.md) · [📋 Phase 08](README.md) · [08.6 The builder pattern →](06-builder-pattern.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/05-type-driven-design.md) <!-- nav -->

# Lesson 08.5 — Type-driven design: make invalid states impossible

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 5 of 17 · ⏱ about 135 minutes

## What you will learn

- Why plain `String` and `u32` values everywhere cause bugs that the compiler could prevent.
- **Parse, don't validate**: how to turn a check into a new type that carries the proof.
- How to build a "smart constructor" with a private field, and use `FromStr` and `TryFrom`.
- How to design enums so that contradictory states cannot be written at all.
- When this style costs more than it gives.

## Before you start

- You finished [Lesson 08.4 — Composition over inheritance](04-composition-over-inheritance.md).
- You know the newtype pattern and the orphan rule from [Lesson 06.2 — Advanced traits and types](../06-rust-advanced/02-advanced-traits-and-types.md). Here we go further: a newtype that can only be created in a valid state.
- You know `Result` and error types ([Lesson 04.10 — Error handling](../04-structs-enums-collections-errors/10-error-handling.md)), and the traits `Display` and `FromStr` ([Lesson 05.2](../05-rust-intermediate/02-traits.md)).

## New words

| Word | Meaning |
|---|---|
| **invariant** | A fact about a value that must always be true. "An email address has some text before and after its `@`" is an invariant of the type `Email`. |
| **primitive obsession** | Using plain types (`String`, `u32`, `bool`) for everything, so that the compiler cannot tell an email from a name or a price from a count. |
| **domain type** | A type named after an idea of the problem you solve (`Email`, `Percent`, `OrderId`), not after how it is stored. |
| **smart constructor** | A function (often called `parse` or `new`) that is the only way to create a value of a type, and that checks the invariant. It returns a `Result` or `Option`. |
| **validate** | To check a value and then throw the answer away (`fn is_valid(&str) -> bool`). The value keeps the same type. |
| **parse** | To check a value and, if it is good, return a *new value of a stricter type* that proves the check was done. |
| **boundary** | The place where data enters your program from outside: user input, files, the network. The best place to parse. |
| **`FromStr`** | The standard trait behind `text.parse::<T>()`. |
| **`TryFrom`** | The standard trait for a conversion that can fail: `T::try_from(value)` returns a `Result`. |
| **unrepresentable state** | A combination of values that the types make impossible to write. |

## 1. The problem: everything is a `String`

Look at this function and its caller:

```rust
fn is_valid_email(text: &str) -> bool {
    text.contains('@')
}

fn send_email(to: String, subject: String, body: String) {
    println!("To: {to}\nSubject: {subject}\n{body}");
}

fn main() {
    let to = "ana@mail.org".to_string();
    let subject = "Welcome".to_string();
    let body = "Glad to see you".to_string();
    if is_valid_email(&to) {
        send_email(subject, to, body); // arguments swapped: still compiles
    }
}
```

**What happens:**

1. `is_valid_email` checks that the text has an `@` and returns `bool`. The answer is used once and forgotten.
2. `send_email` takes three `String`s. The compiler sees three values of the same type and cannot know which is which.
3. In `main` the programmer swapped `to` and `subject` by mistake. The program compiles, passes the check on `to`, and sends a mail with the wrong header lines.

```text
To: Welcome
Subject: ana@mail.org
Glad to see you
```

There are two problems here:

- **Mix-ups** compile. `String` means "text"; it does not say *what kind* of text.
- **Validation is forgotten.** `send_email` cannot know that someone checked `to` earlier. Each function must check again (repeated work, Lesson 08.1) or trust blindly (a bug waiting to happen).

## 2. Parse, don't validate

The fix has one idea: instead of checking a value and keeping the same type, **check it once and return a new type** that exists only for valid values. The type is the proof.

```rust
use std::fmt;
use std::str::FromStr;

mod email {
    use super::*;

    #[derive(Debug, PartialEq)]
    pub struct Email(String); // the field is private

    #[derive(Debug, PartialEq)]
    pub enum EmailError {
        Empty,
        MissingAt,
        EmptyPart,
    }

    impl Email {
        pub fn parse(text: &str) -> Result<Email, EmailError> {
            let text = text.trim();
            if text.is_empty() {
                return Err(EmailError::Empty);
            }
            let (user, host) = text.split_once('@').ok_or(EmailError::MissingAt)?;
            if user.is_empty() || host.is_empty() {
                return Err(EmailError::EmptyPart);
            }
            Ok(Email(text.to_string()))
        }
    }

    impl FromStr for Email {
        type Err = EmailError;
        fn from_str(text: &str) -> Result<Email, EmailError> {
            Email::parse(text)
        }
    }

    impl fmt::Display for Email {
        fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
            write!(f, "{}", self.0)
        }
    }
}

use email::Email;

struct Subject(String);

// No validation inside: an `Email` is already proven valid.
fn send_email(to: &Email, subject: &Subject, body: &str) {
    println!("To: {to}\nSubject: {}\n{body}", subject.0);
}

fn main() {
    let subject = Subject("Welcome".to_string());
    match "  ana@mail.org ".parse::<Email>() {
        Ok(to) => send_email(&to, &subject, "Glad to see you"),
        Err(e) => println!("bad address: {e:?}"),
    }
    println!("{:?}", Email::parse("ana.mail.org"));
    println!("{:?}", Email::parse("@mail.org"));
    println!("{:?}", Email::parse("   "));
}
```

**What happens:**

1. `struct Email(String)` is a newtype (Lesson 06.2). Its field is **private**, so no code outside `mod email` can create or change an `Email` directly.
2. `Email::parse` is the only way in. It trims the text, rejects empty text, requires an `@` with something on both sides, and returns `Result<Email, EmailError>`. The error type is an enum that says *which* rule failed.
3. `impl FromStr` lets you write `"…".parse::<Email>()`, the standard way to parse. `impl Display` makes `{to}` print the address.
4. `send_email` takes `&Email` and `&Subject`. It does **not** check anything: if it has an `&Email`, the address is valid. The check moved to the boundary (`main`), and the proof travels with the value.
5. `main` parses `"  ana@mail.org "` (with spaces) and sends. The three other lines show the errors.

```text
To: ana@mail.org
Subject: Welcome
Glad to see you
Err(MissingAt)
Err(EmptyPart)
Err(Empty)
```

Now the mix-up from Section 1 is a compile error:

```rust
mod email {
    pub struct Email(String);
    impl Email {
        pub fn parse(text: &str) -> Option<Email> {
            text.contains('@').then(|| Email(text.to_string()))
        }
    }
}

use email::Email;

struct Subject(String);

fn send_email(to: &Email, subject: &Subject) {
    let _ = (to, subject);
}

fn main() {
    let to = Email::parse("ana@mail.org").unwrap();
    let subject = Subject("Welcome".to_string());
    send_email(&subject, &to);
}
```

```text
error[E0308]: arguments to this function are incorrect
  --> src/main.rs:21:5
   |
21 |     send_email(&subject, &to);
   |     ^^^^^^^^^^ --------  --- expected `&Subject`, found `&Email`
   |                |
   |                expected `&Email`, found `&Subject`
   |
note: function defined here
   ...

For more information about this error, try `rustc --explain E0308`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

The compiler even suggests the fix (`swap these arguments`). And because the field is private, nobody can skip the check:

```rust
mod email {
    pub struct Email(String);
}

fn main() {
    let sneaky = email::Email("not an address".to_string());
    let _ = sneaky;
}
```

```text
error[E0603]: tuple struct constructor `Email` is private
 --> src/main.rs:6:25
  |
2 |     pub struct Email(String);
  |                      ------ a constructor is private if any of the fields is private
...
6 |     let sneaky = email::Email("not an address".to_string());
  |                         ^^^^^ private tuple struct constructor
   ...

For more information about this error, try `rustc --explain E0603`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

> 🔒 **Security:** Parse at the boundary, once. Data from users, files and networks is untrusted (Phase 10). If everything inside your program uses `Email`, `Percent` and `OrderId` instead of raw text and numbers, a whole class of "forgot to check" bugs, some of them security holes, becomes impossible.

> ⚡ **Performance:** A newtype has the same layout in memory as the value inside. The extra type costs nothing at run time; only the one-time check costs something.

> 🧪 **Testing:** Test the smart constructor with accepted *and* rejected examples, including the border cases (empty, only spaces, `@` at the start). The rest of the program needs no such tests, because the type makes bad values impossible: the compiler is your test.

## 3. The standard traits for parsing

You already used them above. The rule of thumb:

| Trait | Use it for | Call |
|---|---|---|
| `FromStr` | Parsing from text | `"ana@mail.org".parse::<Email>()` |
| `TryFrom<T>` | A conversion that can fail, from another type | `Percent::try_from(150u8)` |
| `From<T>` | A conversion that **cannot** fail | `String::from("x")` |
| `Display` | Showing the value | `println!("{to}")` |

If you implement `TryFrom` for your type, you also get `.try_into()` for free. Exercise 2 uses `TryFrom`.

> 📏 **Convention:** Name the smart constructor `parse` when it reads text, `new` or `try_new` when it takes already-typed parts. Do not create `Email::new` that accepts anything. If a constructor cannot fail, the type probably has no invariant, and a plain struct is enough.

## 4. Make invalid states unrepresentable

Newtypes protect single values. The same idea protects whole structures: design the types so that contradictions cannot be written. Look at a task list:

```rust
struct Task {
    title: String,
    done: bool,
    completed_on_day: Option<u32>,
}

fn describe(task: &Task) -> String {
    if task.done {
        match task.completed_on_day {
            Some(day) => format!("{}: done on day {day}", task.title),
            None => format!("{}: done on ??? (invalid!)", task.title),
        }
    } else if task.completed_on_day.is_some() {
        format!("{}: open, but has a completion day (invalid!)", task.title)
    } else {
        format!("{}: open", task.title)
    }
}

fn main() {
    let good = Task {
        title: "write".to_string(),
        done: true,
        completed_on_day: Some(12),
    };
    let bad = Task {
        title: "test".to_string(),
        done: true,
        completed_on_day: None,
    };
    println!("{}", describe(&good));
    println!("{}", describe(&bad));
}
```

**What happens:**

1. `done` says whether the task is finished. `completed_on_day` says when. Two fields, four combinations, and only **two** of them make sense: (done, with a day) and (not done, no day).
2. `describe` must handle the two impossible combinations with "invalid!" branches. Every other function that touches a `Task` must do the same, or ignore the problem and hope.
3. In `main`, a careless line builds the combination `done: true, completed_on_day: None`.

```text
write: done on day 12
test: done on ??? (invalid!)
```

The root of the bug is that the type allows more states than reality has. Use an enum to describe the **real** states, and put the data where it belongs:

```rust
enum Status {
    Open,
    Done { on_day: u32 },
}

struct Task {
    title: String,
    status: Status,
}

fn describe(task: &Task) -> String {
    match task.status {
        Status::Open => format!("{}: open", task.title),
        Status::Done { on_day } => format!("{}: done on day {on_day}", task.title),
    }
}

fn main() {
    let tasks = [
        Task {
            title: "write".to_string(),
            status: Status::Done { on_day: 12 },
        },
        Task {
            title: "test".to_string(),
            status: Status::Open,
        },
    ];
    for task in &tasks {
        println!("{}", describe(task));
    }
}
```

**What happens:**

1. `Status` has exactly the two real states. The day exists **only** inside `Done`, so it cannot be forgotten and cannot appear on an open task.
2. `describe` is a `match` with two arms. There is no "invalid" arm to write, because there is no invalid value.
3. If a third state is added later (`Cancelled`), the compiler lists every `match` to update (Lesson 08.3).

```text
write: done on day 12
test: open
```

A small checklist for such designs:

- Two `bool`s that depend on each other? Probably one enum.
- An `Option` that must be `Some` exactly when something else is true? Put the data inside the enum variant that needs it.
- A `Vec` that is never empty? A type with a `first` item and a `rest` (Exercise 3).
- A string with a format? A newtype with a smart constructor.

## 5. When this style costs more than it gives

- **Too many types.** A newtype for every `String` in a 50-line script is noise. Use domain types where a mix-up or a bad value would hurt: identifiers, money, units, validated user input.
- **A `pub` field on the newtype** destroys the guarantee. `pub struct Email(pub String)` lets anybody write `Email("rubbish".into())`.
- **Conversion boilerplate.** You need `Display`, `FromStr`, sometimes `AsRef<str>` and comparisons. Derive what you can (`#[derive(Debug, PartialEq, Clone)]`) and add the rest when a caller needs it.
- **Invariants that change.** If the rules for a valid email change every month, hard-coding them in a type means many edits. Keep types for rules that are stable.
- **Data that arrives from outside in bulk.** Libraries that fill your types automatically (for example from JSON, JavaScript Object Notation, a popular text format for data) can bypass your constructor unless you tell them to use it. Always re-check where data enters (you met `serde` in Lesson 05.11).

## Common mistakes

**Mistake 1: a smart constructor that does not check.** `fn new(text: &str) -> Email { Email(text.to_string()) }` gives the *look* of safety and none of it.

**Mistake 2: `unwrap()` on every parse.** In a real program, a bad value is an expected event. Return the error up with `?` (Lesson 04.10) or show a message to the user; `unwrap` is for tests and for cases that are truly impossible.

**Mistake 3: "validate" with a `bool` and then use the raw value.** If a function returns `bool` and the caller continues with the same `String`, the knowledge is lost. Return the parsed type instead.

**Mistake 4: boolean flags instead of an enum.** `is_admin: bool, is_banned: bool, is_guest: bool` has eight combinations; probably three are real. Think about the real states first.

## Try it yourself

You will work test-first on a new domain type, `Subject` (an email subject line).

1. Create the package and copy the whole first program of Section 2 (the `email` module, `Subject`, `send_email` and `main`) into `src/main.rs`:

```bash
cd ~/projects/rust
cargo new code/p08-types
cd code/p08-types
```

2. Replace the `Subject` stub (`struct Subject(String);`) by a real type, with a `parse` function that returns `Result<Subject, SubjectError>`. The rules: the text is trimmed; empty text is rejected (`SubjectError::Empty`); more than 40 **characters** is rejected (`SubjectError::TooLong`). Derive `Debug` and `PartialEq` for `Subject` and `SubjectError`, so that `assert_eq!` can compare them. Start by writing these tests at the end of the file, and watch them fail to compile, then pass:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_a_normal_subject() {
        assert_eq!(Subject::parse(" Hi "), Ok(Subject("Hi".to_string())));
    }

    #[test]
    fn rejects_an_empty_subject() {
        assert_eq!(Subject::parse("   "), Err(SubjectError::Empty));
    }

    #[test]
    fn rejects_a_subject_longer_than_40_characters() {
        assert_eq!(Subject::parse(&"a".repeat(41)), Err(SubjectError::TooLong));
    }

    #[test]
    fn counts_characters_not_bytes() {
        // 40 letters of two bytes each: 80 bytes, but only 40 characters
        assert!(Subject::parse(&"é".repeat(40)).is_ok());
    }
}
```

3. Change `send_email` and `main` to use `Subject::parse` and handle the error. Run `cargo test`:

```text
running 4 tests
test tests::accepts_a_normal_subject ... ok
test tests::counts_characters_not_bytes ... ok
test tests::rejects_a_subject_longer_than_40_characters ... ok
test tests::rejects_an_empty_subject ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

4. Run `cargo run` and check that a bad subject prints a message instead of sending a mail. Finish with:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "feat(p08): Email and Subject domain types"
```

**What happens:** The test `counts_characters_not_bytes` guards against the `len()` mistake from Lesson 08.1: 40 letters like `é` take 80 bytes, but they are 40 characters and must be accepted.

## Exercises

### Exercise 1 — Find the invalid states (easy)

For each struct, list one combination of values that is allowed by the types but makes no sense:

1. `struct Light { on: bool, brightness: u8 }` where brightness 0 means dark.
2. `struct Door { open: bool, locked: bool }`
3. `struct Page { number: u32, total: u32 }`, a page number inside a document.

<details>
<summary>Hint</summary>

Write all combinations of the fields, then cross out the ones that cannot happen in reality.

</details>

<details>
<summary>Answer</summary>

1. `on: true, brightness: 0` (on but dark), and `on: false, brightness: 200`.
2. `open: true, locked: true`: an open door that is locked.
3. `number: 9, total: 3`: page 9 of a 3-page document.

Why it works: for (1) and (2) an enum such as `enum Door { Open, Closed, Locked }` has only real states; for (3) a smart constructor `Page::new(number, total)` can check `number <= total`.

</details>

### Exercise 2 — A `Percent` type, with tests (medium)

`discounted` accepts any `u8` as a percent, so `150` compiles. (In a debug build the second call panics with `attempt to subtract with overflow`.) Create `Percent` with a private value and `TryFrom<u8>` that accepts 0 to 100. Change `discounted` to take a `Percent`, and **write tests** for 0, 100 and 101.

```rust
fn discounted(price_cents: u64, percent: u8) -> u64 {
    price_cents - price_cents * u64::from(percent) / 100
}

fn main() {
    println!("{}", discounted(2000, 25));
    println!("{}", discounted(2000, 150)); // nonsense, but it compiles
}
```

<details>
<summary>Hint</summary>

`impl TryFrom<u8> for Percent { type Error = OutOfRange; fn try_from(value: u8) -> Result<Percent, OutOfRange> { ... } }`. In `main`, `Percent::try_from(25)` returns a `Result`; use `.map(|p| discounted(2000, p))` to continue.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq, Clone, Copy)]
struct Percent(u8);

#[derive(Debug, PartialEq)]
struct OutOfRange(u8);

impl TryFrom<u8> for Percent {
    type Error = OutOfRange;

    fn try_from(value: u8) -> Result<Percent, OutOfRange> {
        if value <= 100 {
            Ok(Percent(value))
        } else {
            Err(OutOfRange(value))
        }
    }
}

fn discounted(price_cents: u64, percent: Percent) -> u64 {
    price_cents - price_cents * u64::from(percent.0) / 100
}

fn main() {
    println!("{:?}", Percent::try_from(25).map(|p| discounted(2000, p)));
    println!("{:?}", Percent::try_from(150).map(|p| discounted(2000, p)));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_zero_to_one_hundred() {
        assert_eq!(Percent::try_from(0), Ok(Percent(0)));
        assert_eq!(Percent::try_from(100), Ok(Percent(100)));
    }

    #[test]
    fn rejects_values_above_one_hundred() {
        assert_eq!(Percent::try_from(101), Err(OutOfRange(101)));
    }

    #[test]
    fn discount_uses_the_percent() {
        assert_eq!(discounted(2000, Percent(25)), 1500);
    }
}
```

Why it works: a `Percent` can be created only through `try_from`, so `discounted` never sees a value above 100. The error carries the bad value for a good message. (In a real project the type would live in its own module so that the field is really private; here, to keep the example short, everything is in one file.)

</details>

### Exercise 3 — A list that is never empty (hard)

`Vec<T>::iter().max()` returns an `Option` because the vector can be empty. Create `NonEmptyVec<T>` that stores a `first` item and a `rest` vector, with `new(first)`, `from_vec(Vec<T>) -> Option<Self>`, `push`, and a method `max` that returns `&T` (not an `Option`). **Write tests** for `max` and for `from_vec` of an empty vector.

<details>
<summary>Hint</summary>

`max` can start from `&self.first` and fold over `self.rest`. For `from_vec`, remove the first item with `items.remove(0)` after checking the vector is not empty.

</details>

<details>
<summary>Answer</summary>

```rust
struct NonEmptyVec<T> {
    first: T,
    rest: Vec<T>,
}

impl<T: Ord> NonEmptyVec<T> {
    fn new(first: T) -> Self {
        NonEmptyVec {
            first,
            rest: Vec::new(),
        }
    }

    fn from_vec(mut items: Vec<T>) -> Option<Self> {
        if items.is_empty() {
            return None;
        }
        let first = items.remove(0);
        Some(NonEmptyVec { first, rest: items })
    }

    fn push(&mut self, item: T) {
        self.rest.push(item);
    }

    // No Option here: a non-empty list always has a largest item.
    fn max(&self) -> &T {
        self.rest
            .iter()
            .fold(&self.first, |best, x| if x > best { x } else { best })
    }
}

fn main() {
    let mut list = NonEmptyVec::new(3);
    list.push(9);
    list.push(4);
    println!("{}", list.max());
    println!("{}", NonEmptyVec::<i32>::from_vec(vec![]).is_none());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn max_of_one_item_is_that_item() {
        assert_eq!(*NonEmptyVec::new(5).max(), 5);
    }

    #[test]
    fn max_finds_the_largest() {
        let list = NonEmptyVec::from_vec(vec![2, 8, 3]).unwrap();
        assert_eq!(*list.max(), 8);
    }

    #[test]
    fn an_empty_vec_cannot_become_a_non_empty_one() {
        assert!(NonEmptyVec::<i32>::from_vec(Vec::new()).is_none());
    }
}
```

Why it works: the type itself guarantees that there is at least one item, so `max` needs no `Option`. The "empty" case is handled once, at the boundary (`from_vec`), and never again. That is "parse, don't validate" for a collection.

</details>

## Quiz

1. What is the difference between validating and parsing?

<details>
<summary>Answer</summary>

Validating returns an answer (`bool`) and keeps the same type, so the knowledge is lost. Parsing returns a new, stricter type, so the knowledge travels with the value.

</details>

2. Why must the field of a newtype with an invariant be private?

<details>
<summary>Answer</summary>

Otherwise any code can create the type directly and skip the check, so the guarantee is worthless.

</details>

3. What does it mean to make an invalid state unrepresentable?

<details>
<summary>Answer</summary>

To design the types so that a contradictory combination of values cannot be written, for example by putting data inside the enum variant that needs it.

</details>

4. Does a newtype cost run-time speed or memory?

<details>
<summary>Answer</summary>

No. It has the same layout as the inner value; only the one-time check costs something.

</details>

5. Where should untrusted data be parsed?

<details>
<summary>Answer</summary>

At the boundary, where it enters the program, once.

</details>

## Summary

- Plain `String`, `u32` and `bool` do not say what a value means, so the compiler cannot catch mix-ups or forgotten checks.
- **Parse, don't validate**: check once at the boundary and return a stricter type that carries the proof.
- A **smart constructor** with a private field is the only way to create the type; use `FromStr`, `TryFrom` and `Display` for the standard interfaces.
- Design enums so that each variant holds exactly the data valid in that state; contradictory combinations then cannot be written.
- Newtypes are free at run time, but use them where a mix-up or a bad value would hurt, not for every string.
- A `pub` field or a constructor that does not check removes the guarantee.

## Sources and further reading

- [Alexis King — Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — the essay that gave this idea its name (the examples use another language, but the idea is the same).
- [Rust API Guidelines — Type safety](https://rust-lang.github.io/api-guidelines/type-safety.html) — newtypes for static distinctions, and custom types instead of `bool` arguments.
- [Rust Design Patterns — Newtype](https://rust-unofficial.github.io/patterns/patterns/behavioural/newtype.html) — the newtype pattern with its trade-offs.
- [std::str::FromStr](https://doc.rust-lang.org/std/str/trait.FromStr.html) and [std::convert::TryFrom](https://doc.rust-lang.org/std/convert/trait.TryFrom.html) — the standard conversion traits.

---

[🏠 Start](../START-HERE.md) · [← 08.4 Composition over inheritance](04-composition-over-inheritance.md) · [📋 Phase 08](README.md) · [08.6 The builder pattern →](06-builder-pattern.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
