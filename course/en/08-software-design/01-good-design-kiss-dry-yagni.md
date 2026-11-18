<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 08 — Software design: principles, patterns and conventions](README.md) · [📋 Phase 08](README.md) · [08.2 Cohesion, coupling and separation of concerns →](02-cohesion-coupling-and-separation-of-concerns.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/01-good-design-kiss-dry-yagni.md) <!-- nav -->

# Lesson 08.1 — Good design: KISS, DRY and YAGNI

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 1 of 17 · ⏱ about 135 minutes

## What you will learn

- Why code is read much more often than it is written, and what "good design" means in practice.
- **KISS**: why the simplest code that works is usually the best code.
- **DRY**: how to remove duplicated *knowledge*, and why not every repeated line should be merged.
- **YAGNI**: why you should not build things "just in case".
- What to do when these three principles disagree.

## Before you start

- You finished Phase 07, the last lesson being [Lesson 07.20 — Checkpoint](../07-algorithms-and-data-structures/20-checkpoint.md).
- You can write tests with `#[test]` and `assert_eq!` ([Lesson 05.6 — Testing in depth](../05-rust-intermediate/06-testing-in-depth.md)).
- You have seen closures and `Box<dyn Fn>` ([Lesson 05.4 — Closures](../05-rust-intermediate/04-closures.md) and [Lesson 06.1 — Trait objects](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md)).

## New words

| Word | Meaning |
|---|---|
| **design** (of software) | The decisions about how code is split into parts, what the parts are called, and how they work together. Every program has a design, good or bad. |
| **design principle** | A rule of thumb that helps you choose between two ways of writing code. It is advice from experience, not a law, and it has exceptions. |
| **maintainability** | How easy it is to read, change and fix a program months later, often by someone else. |
| **KISS** (Keep It Simple, Stupid) | Choose the simplest solution that does the job. The word "stupid" is not about you: it says that a *simple* design is hard to break. |
| **DRY** (Don't Repeat Yourself) | Every piece of knowledge should have one single place in the code. |
| **YAGNI** (You Aren't Gonna Need It) | Do not add a feature, or a way to be flexible, before you have a real need for it. |
| **duplication** | The same idea written in more than one place. |
| **abstraction** | A name (a function, a type, a trait) that hides details so that you can talk about an idea without repeating its steps. |
| **refactoring** | Changing the structure of code without changing what it does. Lesson 08.13 teaches it fully. |
| **rule of three** | A guideline: copy once if you must, but extract a shared abstraction when the same code appears a third time. |

## 1. What "good design" means

Imagine you open a program you wrote three months ago. You must change one rule: "orders of 10 items or more get a discount". How long does it take to find the place? How sure are you that the change breaks nothing else?

That is what design is about. The experience of many teams is the same: programmers spend far more time **reading** code than writing it. So good design is not about clever tricks. It is about making the next change **cheap and safe**.

A **design principle** is a short rule that helps with that. In this lesson we learn three that every engineer meets in the first weeks of a job:

| Principle | One-line version | Fights against |
|---|---|---|
| KISS | Prefer the simple solution. | Cleverness, needless complexity |
| DRY | One place for each piece of knowledge. | Copy-and-paste |
| YAGNI | Build only what you need now. | Guessing the future |

Principles are not commandments. For each one we look at a typical problem (the "before"), a better version (the "after") and a "when NOT to apply it".

> 💡 **Tip:** All code of this phase lives in `code/p08-<topic>`, one Cargo package per lesson (this one: `code/p08-kiss-dry`). Examples are shown as if they were `src/main.rs`. To keep several examples in one package, save each as `src/bin/<name>.rs` and run it with `cargo run --bin <name>`; compiler messages then show that path instead of `src/main.rs`. The messages in the lessons were produced in a package called `demo`; yours show your own package name (for example `p08-kiss-dry`).

## 2. KISS: keep it simple

A small shop calculation. The price of an order is `unit price × quantity`. An order of 10 or more items gets 10 % off, and club members get another 5 % off. Money is counted in **cents** (whole numbers), so we never need floating-point numbers for it.

The first version was written by someone who loved flexibility:

```rust
type Rule = Box<dyn Fn(u64) -> u64>;

fn make_rules(quantity: u32, is_member: bool) -> Vec<Rule> {
    let mut rules: Vec<Rule> = Vec::new();
    if quantity >= 10 {
        rules.push(Box::new(|cents| cents - cents / 10));
    }
    if is_member {
        rules.push(Box::new(|cents| cents - cents / 20));
    }
    rules
}

fn total_cents(unit_cents: u64, quantity: u32, is_member: bool) -> u64 {
    make_rules(quantity, is_member)
        .iter()
        .fold(unit_cents * u64::from(quantity), |sum, rule| rule(sum))
}

fn main() {
    println!("{}", total_cents(250, 4, false));
    println!("{}", total_cents(250, 12, true));
}
```

**What happens:**

1. `type Rule = Box<dyn Fn(u64) -> u64>` names "a boxed closure that takes a price and returns a new price" (Lessons 05.4 and 06.1).
2. `make_rules` builds a `Vec` with one closure for every discount that applies. Each closure is allocated on the heap by `Box::new`.
3. `total_cents` starts from `unit_cents * quantity` and uses `fold` to pass the price through the rules, one after another.
4. `main` prints the total of an order with no discount, and of an order with both discounts.

```text
1000
2565
```

It works. So what is wrong? To answer "what does a member pay for 12 items?" you must read a type alias, a function that returns a vector of closures, and a `fold`. A new reader must understand the *mechanism* before they can see the *rules*, and the rules are all that matters here.

The simple version:

```rust
fn total_cents(unit_cents: u64, quantity: u32, is_member: bool) -> u64 {
    let mut total = unit_cents * u64::from(quantity);
    if quantity >= 10 {
        total -= total / 10; // bulk discount: 10 %
    }
    if is_member {
        total -= total / 20; // member discount: 5 %
    }
    total
}

fn main() {
    println!("{}", total_cents(250, 4, false));
    println!("{}", total_cents(250, 12, true));
}
```

It prints `1000` and `2565` again. The rules (10 % and 5 %) are visible at a glance, nothing is allocated, and every discount has its own line where you can put a `println!`.

> 📏 **Convention:** Simple does not mean "fewest lines" or "no abstractions". It means the reader must hold as few ideas in their head as possible. A plain `if` that everybody understands beats a clever `fold` that needs a minute of thought.

**When NOT to apply KISS.** Keep the flexible version if the rules must really change at run time, for example when a shop owner edits discounts in a settings file. Then the list of rules *is* the requirement, and a plain `if` cannot express it. KISS says "do not pay for flexibility you do not need", not "never be flexible". (Lesson 08.9 shows the *strategy* pattern, a tidy way to build such flexibility when it is needed.)

> ⚡ **Performance:** The simple version is also faster, because it allocates nothing. But that is a side effect. Do not choose a design for speed before you have measured (Phase 19 teaches how).

## 3. DRY: one place for each piece of knowledge

The name DRY comes from the book *The Pragmatic Programmer*. The idea is about **knowledge**: each fact or rule of your program should be written in exactly one place. The rule is *not* "never write two similar lines".

A typical problem. Two forms of a program check that a name is not empty and not longer than 20 letters. The second checker was made by copy and paste:

```rust
fn check_username(name: &str) -> Result<(), String> {
    if name.is_empty() {
        return Err("username is empty".to_string());
    }
    if name.len() > 20 {
        return Err("username is too long".to_string());
    }
    Ok(())
}

fn check_project_name(name: &str) -> Result<(), String> {
    if name.is_empty() {
        return Err("project name is empty".to_string());
    }
    if name.len() >= 20 {
        return Err("project name is too long".to_string());
    }
    Ok(())
}

fn main() {
    let name = "abcdefghijklmnopqrst"; // exactly 20 letters
    println!("{:?}", check_username(name));
    println!("{:?}", check_project_name(name));
}
```

**What happens:**

1. The two functions are copies; only the words in the messages differ.
2. `check_project_name` has `>=` where `check_username` has `>`. Somebody typed it wrong, or "fixed" one copy and forgot the other.
3. `main` checks a name with exactly 20 letters. The rule says 20 is allowed.

```text
Ok(())
Err("project name is too long")
```

The same rule gives **two different answers**. That is the real danger of duplication: when the knowledge ("a name has at most 20 characters") changes, you must remember every copy, and one day you will forget one. Both copies also share another bug: `len()` counts *bytes*, not characters (Lesson 04.8), so the name "Ünal" counts as 5.

The fix is one function that holds the knowledge, and one constant for the limit:

```rust
const MAX_NAME_CHARS: usize = 20;

fn check_name(kind: &str, name: &str) -> Result<(), String> {
    if name.is_empty() {
        return Err(format!("{kind} is empty"));
    }
    if name.chars().count() > MAX_NAME_CHARS {
        return Err(format!("{kind} is too long"));
    }
    Ok(())
}

fn main() {
    let name = "abcdefghijklmnopqrst"; // exactly 20 letters
    println!("{:?}", check_name("username", name));
    println!("{:?}", check_name("project name", name));
}
```

```text
Ok(())
Ok(())
```

**What happens:**

1. `MAX_NAME_CHARS` is the single place where the limit lives.
2. `check_name` takes the part that differs (the `kind` of name, used in the message) as a parameter.
3. `name.chars().count()` counts characters, so the byte bug is fixed once, for everybody.
4. The 20-letter name is accepted everywhere.

> 🔒 **Security:** Duplicated *checks* are a security risk. If a permission check is copied to five places and a bug is fixed in four of them, the fifth is a hole. Keep important rules (who may do what, which input is valid) in exactly one function.

### DRY is about knowledge, not about looks

Some code looks the same but is not the same knowledge. Take `format_price` (prints `1999` cents as `$19.99`) and `format_weight` (prints `2500` grams as `2.500 kg`). Both divide a number and print two parts. A keen programmer sees a repeat and merges them:

```rust
fn format_fixed(value: u64, divisor: u64, width: usize, prefix: &str, suffix: &str) -> String {
    format!(
        "{prefix}{}.{:0width$}{suffix}",
        value / divisor,
        value % divisor
    )
}

fn main() {
    println!("{}", format_fixed(1999, 100, 2, "$", ""));
    println!("{}", format_fixed(2500, 1000, 3, "", " kg"));
}
```

```text
$19.99
2.500 kg
```

It works, but read the calls: what are `100` and `2` in `format_fixed(1999, 100, 2, "$", "")`? Next month prices must show a thousands separator ("$1,999.00") and weights must round to one digit. Every change forces a new parameter or flag into `format_fixed`, and every caller is affected. Prices and weights change for **different reasons**, so they are different knowledge. The "duplication" was an accident of the first version.

This is called the **wrong abstraction**. Sandi Metz, an experienced engineer, said it in one sentence: duplication is far cheaper than the wrong abstraction.

> 📏 **Convention:** Use the **rule of three**. The first time, write the code. The second time, copy it if you must, and take note. The third time, you can see what really varies between the copies, so extract a function or a type. Before the third copy you are only guessing.

**When NOT to apply DRY:**

- the pieces look alike but change for different reasons (the example above);
- the shared function would need flag parameters like `fn run(kind: bool, extra: bool)`: that is two functions glued together;
- the copies live in different crates or services that must be able to change independently.

## 4. YAGNI: do not build for an imagined future

YAGNI comes from a way of working called Extreme Programming. The idea: a feature added because "we will surely need it later" is often wrong, or never used. Meanwhile it costs you today: more code to read, to test and to keep working.

A settings holder, built by a programmer who imagined many future uses:

```rust
use std::collections::HashMap;

trait Storage {
    fn save(&mut self, key: &str, value: &str);
    fn load(&self, key: &str) -> Option<String>;
}

impl Storage for HashMap<String, String> {
    fn save(&mut self, key: &str, value: &str) {
        self.insert(key.to_string(), value.to_string());
    }
    fn load(&self, key: &str) -> Option<String> {
        self.get(key).cloned()
    }
}

struct Settings<S: Storage> {
    storage: S,
    retry_count: u32,
    on_change: Option<Box<dyn Fn(&str)>>,
}

impl<S: Storage> Settings<S> {
    fn set(&mut self, key: &str, value: &str) {
        self.storage.save(key, value);
    }
    fn get(&self, key: &str) -> Option<String> {
        self.storage.load(key)
    }
}

fn main() {
    let mut settings = Settings {
        storage: HashMap::new(),
        retry_count: 3,
        on_change: None,
    };
    settings.set("theme", "dark");
    println!("{:?}", settings.get("theme"));
}
```

**What happens:**

1. `trait Storage` means "something that saves and loads text". Its only implementer is `HashMap<String, String>`.
2. `Settings<S: Storage>` is generic over the storage, so that one day a file or a database can be plugged in.
3. `retry_count` and `on_change` were added "for later". Nothing reads them.
4. `main` uses all this machinery to store one string.

It prints `Some("dark")`. Even the compiler is suspicious of this code. When it builds it, it says:

```text
warning: fields `retry_count` and `on_change` are never read
  --> src/main.rs:19:5
   |
17 | struct Settings<S: Storage> {
   |        -------- fields in this struct
18 |     storage: S,
19 |     retry_count: u32,
   |     ^^^^^^^^^^^
20 |     on_change: Option<Box<dyn Fn(&str)>>,
   |     ^^^^^^^^^
   |
   = note: `#[warn(dead_code)]` (part of `#[warn(unused)]`) on by default

warning: `demo` (bin "demo") generated 1 warning
```

Compare with the version that solves today's problem:

```rust
use std::collections::HashMap;

fn main() {
    let mut settings: HashMap<String, String> = HashMap::new();
    settings.insert("theme".to_string(), "dark".to_string());
    println!("{:?}", settings.get("theme"));
}
```

It prints `Some("dark")` too. It is a third of the size, and everybody understands it at first reading. If a second kind of storage appears next year, you will *then* know what it needs (may saving fail? is it slow?) and you can write a trait with the right methods. Small, clear code is easy to change. A wrong abstraction that has spread through the program is not.

> 💡 **Tip:** The Rust compiler helps you apply YAGNI. Warnings such as `dead_code` and `unused` point at fields, functions and imports that nothing uses. Treat them as "delete this" and keep your build free of warnings.

**When NOT to apply YAGNI.** Some things are very expensive to change later. Thinking about them early is not speculation; it is design:

- **Public interfaces.** Once other programs or people depend on your function names or file format, every change breaks them.
- **Stored data formats.** Data on disk outlives code. A version number in a file format costs one field today and saves a painful migration later.
- **Security and correctness.** Never postpone input checking with "we will add it later".
- **Seams for testing.** A small trait that lets a test replace the clock or the network is often justified before there is a second real implementation (Lesson 08.9).

YAGNI also does not mean "no design" or "no tests". It means: do not build features that nobody asked for.

## 5. When the principles disagree

The three principles pull in different directions, and that is normal. When you are unsure, ask these questions in this order:

1. Does the code work, and is it tested? If not, fix that first. Everything else needs tests as a safety net.
2. Is the same *knowledge* written twice? Merge it (DRY), keeping the rule of three in mind.
3. Is something here that nobody uses or needs yet? Remove it (YAGNI).
4. Can a new reader understand it quickly? If not, simplify it (KISS).

> 🧪 **Testing:** Put tests around code **before** you refactor it. A refactoring must not change behaviour, and only a test can tell you that it did not. You will practise this in the next section.

## Common mistakes

**Mistake 1: making code generic "just in case".** A function that takes "anything" often cannot do anything with it:

```rust
fn describe<T>(item: T) -> String {
    format!("item: {}", item)
}

fn main() {
    println!("{}", describe(42));
}
```

```text
error[E0277]: `T` doesn't implement `std::fmt::Display`
 --> src/main.rs:2:25
  |
2 |     format!("item: {}", item)
  |                    --   ^^^^ `T` cannot be formatted with the default formatter
  |                    |
  |                    required by this formatting parameter
  |
  = note: in format strings you may be able to use `{:?}` (or `{:#?}` for pretty-print) instead
help: consider restricting type parameter `T` with trait `Display`
  |
1 | fn describe<T: std::fmt::Display>(item: T) -> String {
  |              +++++++++++++++++++

For more information about this error, try `rustc --explain E0277`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

The compiler is right: `T` can be any type, and not every type can be printed with `{}`. To make `describe` work you must add a bound (`T: std::fmt::Display`). If the function is only ever called with numbers, `fn describe(item: i32)` is simpler. Add generics when the *second* real use appears, not before.

**Mistake 2: merging code only because it looks alike.** Ask first whether the pieces change for the same reason (Section 3).

**Mistake 3: "simple" means "short".** A 200-character iterator chain is short but not simple. If a line needs a comment to be understood, split it into named steps.

**Mistake 4: YAGNI as an excuse for sloppiness.** "We don't need tests yet" or "we don't need error handling yet" are not YAGNI. Tests and error handling are part of the feature you build today.

## Try it yourself

You will pin down the behaviour of a function with tests, then simplify the function and watch the tests stay green.

1. Create the package:

```bash
cd ~/projects/rust
cargo new code/p08-kiss-dry
cd code/p08-kiss-dry
```

2. Replace `src/main.rs` with the over-flexible first version from Section 2 (the code with `make_rules`, with its `main`), and add this test module at the end of the file:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn small_order_has_no_discount() {
        assert_eq!(total_cents(250, 4, false), 1000);
    }

    #[test]
    fn ten_items_get_the_bulk_discount() {
        assert_eq!(total_cents(100, 9, false), 900);
        assert_eq!(total_cents(100, 10, false), 900);
    }

    #[test]
    fn member_discount_is_applied_after_bulk_discount() {
        assert_eq!(total_cents(250, 12, true), 2565);
    }
}
```

3. Run the tests. They pass:

```bash
cargo test
```

```text
running 3 tests
test tests::member_discount_is_applied_after_bulk_discount ... ok
test tests::small_order_has_no_discount ... ok
test tests::ten_items_get_the_bulk_discount ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The order of the `test` lines on your computer can differ, because tests run in parallel.)

4. Now replace `make_rules` and `total_cents` with the simple `total_cents` from Section 2. Keep `main` and the `tests` module, and run `cargo test` again. The result is the same three lines, all `ok`.

5. Check the code and save your work:

```bash
cargo fmt
cargo clippy
git add .
git commit -m "refactor(p08): simplify total_cents"
```

**What happens:** The tests did not care *how* the total is calculated, only *what* it is. That is why you were free to change the inside. The more behaviour your tests describe, the braver your refactoring can be.

## Exercises

### Exercise 1 — Name the principle (easy)

For each situation, say which principle (KISS, DRY or YAGNI) is broken:

1. A program has a `trait Plugin` and a plugin registry, but only one plugin exists and nobody plans another.
2. The rule "a PIN (Personal Identification Number, a short secret code) has 4 digits" is written in eight different functions.
3. A function that adds two numbers uses a macro, three generic parameters and a builder.

<details>
<summary>Hint</summary>

Ask: is something built that nobody needs (YAGNI)? Is the same knowledge written many times (DRY)? Is the solution harder than the problem (KISS)?

</details>

<details>
<summary>Answer</summary>

1. YAGNI: a plugin system for one plugin is flexibility nobody asked for.
2. DRY: the PIN rule is knowledge that belongs in one function or constant.
3. KISS: `a + b` is the simple solution.

Why it works: principles are questions you ask of the code. A situation can break several at once (the plugin registry in 1 is also not simple); you only need to see the problem.

</details>

### Exercise 2 — Remove duplication, and test it (medium)

These two functions repeat the "trim and capitalise the first letter" logic. Extract it into a function `capitalize` and make both functions use it. **Write a test** for `capitalize` that covers a normal name, an empty string and a name that starts with a non-ASCII letter (ASCII is the basic set of English letters, digits and symbols) such as `"élan"`.

```rust
fn welcome(name: &str) -> String {
    let mut chars = name.trim().chars();
    let fixed = match chars.next() {
        Some(first) => first.to_uppercase().collect::<String>() + chars.as_str(),
        None => String::new(),
    };
    format!("Welcome, {fixed}!")
}

fn goodbye(name: &str) -> String {
    let mut chars = name.trim().chars();
    let fixed = match chars.next() {
        Some(first) => first.to_uppercase().collect::<String>() + chars.as_str(),
        None => String::new(),
    };
    format!("Goodbye, {fixed}. See you soon!")
}

fn main() {
    println!("{}", welcome("  ana "));
    println!("{}", goodbye("bo"));
}
```

<details>
<summary>Hint</summary>

The shared knowledge is only "how to turn `  ana ` into `Ana`". The messages ("Welcome" and "Goodbye") are different and stay in their own functions.

</details>

<details>
<summary>Answer</summary>

```rust
fn capitalize(text: &str) -> String {
    let mut chars = text.trim().chars();
    match chars.next() {
        Some(first) => first.to_uppercase().collect::<String>() + chars.as_str(),
        None => String::new(),
    }
}

fn welcome(name: &str) -> String {
    format!("Welcome, {}!", capitalize(name))
}

fn goodbye(name: &str) -> String {
    format!("Goodbye, {}. See you soon!", capitalize(name))
}

fn main() {
    println!("{}", welcome("  ana "));
    println!("{}", goodbye("bo"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn capitalize_handles_normal_empty_and_non_ascii_text() {
        assert_eq!(capitalize("  ana "), "Ana");
        assert_eq!(capitalize(""), "");
        assert_eq!(capitalize("élan"), "Élan");
    }
}
```

Why it works: `capitalize` holds the one piece of knowledge, and `welcome` and `goodbye` only choose their wording. The test calls `capitalize` directly, so all three cases are checked without printing anything. `cargo test` reports 1 test passed.

</details>

### Exercise 3 — Merge or not? (hard)

Two functions `valid_for_signup` and `valid_for_newsletter` do exactly the same check on an email address today (the user part is not empty and the host part contains a dot). Next quarter, signups must reject addresses ending with `@example.com`, while the newsletter keeps accepting them. Should the two functions be merged into one? Show a design that respects DRY without creating a wrong abstraction.

<details>
<summary>Hint</summary>

Separate what is *really* shared (what an email address looks like) from what belongs to each use (the signup policy).

</details>

<details>
<summary>Answer</summary>

Do not merge the two public functions. Extract only the shared knowledge, and keep one small function for each purpose:

```rust
// The one piece of knowledge that is really shared.
fn looks_like_email(text: &str) -> bool {
    match text.split_once('@') {
        Some((user, host)) => !user.is_empty() && host.contains('.'),
        None => false,
    }
}

// Each rule can now change on its own.
fn valid_for_signup(text: &str) -> bool {
    looks_like_email(text) && !text.ends_with("@example.com")
}

fn valid_for_newsletter(text: &str) -> bool {
    looks_like_email(text)
}

fn main() {
    println!("{}", valid_for_signup("ana@mail.org"));
    println!("{}", valid_for_signup("ana@example.com"));
    println!("{}", valid_for_newsletter("ana@example.com"));
}
```

Why it works: `looks_like_email` is the shared knowledge, defined once. `valid_for_signup` and `valid_for_newsletter` can change independently, and no `bool` flag is needed. A merged `fn valid(text, for_signup: bool)` would be the wrong abstraction.

</details>

## Quiz

1. What does DRY say should be unique: lines of code, or pieces of knowledge?

<details>
<summary>Answer</summary>

Pieces of knowledge. Two similar lines that change for different reasons do not violate DRY.

</details>

2. In the first DRY example (`check_username` and `check_project_name`), why did the two checks give different answers?

<details>
<summary>Answer</summary>

The rule was copied, and one copy was changed (`>=` instead of `>`). Duplicated knowledge drifts apart.

</details>

3. Name two things that are expensive to change later, so YAGNI does not apply to them.

<details>
<summary>Answer</summary>

Any two of: public interfaces, stored data formats, security checks, the seams needed for tests.

</details>

4. What does the rule of three say?

<details>
<summary>Answer</summary>

Wait until the same code appears a third time before you extract an abstraction; by then you can see what really varies.

</details>

5. A function `fn render(kind: bool, flag: bool, ...)` was made by merging two functions. Which principle does this break, and why?

<details>
<summary>Answer</summary>

DRY used wrongly. Flag parameters show that two different things were glued together: the wrong abstraction.

</details>

## Summary

- Good design makes the next change cheap and safe; code is read far more often than it is written.
- **KISS**: choose the simple solution. A plain `if` beats a clever pipeline of boxed closures, unless run-time flexibility is the real requirement.
- **DRY**: every piece of *knowledge* has one home. Duplicated rules drift apart and cause bugs.
- DRY does not mean "merge everything that looks alike". The wrong abstraction costs more than a little duplication; use the rule of three.
- **YAGNI**: do not build for an imagined future. Compiler warnings about unused code often point at it.
- Public interfaces, stored data formats and security are exceptions: think about them early.
- Principles are heuristics, not laws. Put tests around the code first, then refactor.

## Sources and further reading

- [Martin Fowler — Yagni](https://martinfowler.com/bliki/Yagni.html) — a short essay on what YAGNI means and when it does not apply.
- [Sandi Metz — The Wrong Abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction) — why duplication is cheaper than the wrong abstraction.
- [Martin Fowler — Beck Design Rules](https://martinfowler.com/bliki/BeckDesignRules.html) — four simple rules for a good design: passes the tests, reveals intention, no duplication, fewest elements.
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — the official checklist of conventions for Rust libraries (Lesson 08.11 uses it).
- [The Rust Programming Language — 3.5 Control Flow](https://doc.rust-lang.org/book/ch03-05-control-flow.html) — a reminder about `if`, which is often all you need.

---

[🏠 Start](../START-HERE.md) · [← Phase 08 — Software design: principles, patterns and conventions](README.md) · [📋 Phase 08](README.md) · [08.2 Cohesion, coupling and separation of concerns →](02-cohesion-coupling-and-separation-of-concerns.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
