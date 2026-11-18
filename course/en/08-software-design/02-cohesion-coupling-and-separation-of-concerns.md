<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.1 Good design: KISS, DRY and YAGNI](01-good-design-kiss-dry-yagni.md) · [📋 Phase 08](README.md) · [08.3 The SOLID principles →](03-solid-principles.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/02-cohesion-coupling-and-separation-of-concerns.md) <!-- nav -->

# Lesson 08.2 — Cohesion, coupling and separation of concerns

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 2 of 17 · ⏱ about 135 minutes

## What you will learn

- What **cohesion** and **coupling** mean, and why we want high cohesion and low coupling.
- How to split a program that "does everything" into parts that each handle one **concern**.
- How to hide the inside of a module with private fields, so other code cannot depend on it.
- How to design functions that are easy to test because they do not print or read files.
- When splitting goes too far.

## Before you start

- You finished [Lesson 08.1 — Good design: KISS, DRY and YAGNI](01-good-design-kiss-dry-yagni.md).
- You know modules and visibility (`pub`, private by default) from [Lesson 04.6 — Modules, crates and packages](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md), and structs and methods from [Lesson 04.2](../04-structs-enums-collections-errors/02-methods-and-associated-functions.md).
- You can write tests ([Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)).

## New words

| Word | Meaning |
|---|---|
| **cohesion** | How closely the things inside one part (a function, a struct, a module) belong together. *High* cohesion: everything in it serves one purpose. |
| **coupling** | How much one part depends on the details of another part. *Low* (or *loose*) coupling: you can change one part without touching the other. |
| **concern** | One separate topic that a program must handle, for example "read the input", "apply the business rules", "show the result". |
| **separation of concerns** | Keeping each concern in its own part of the code. |
| **encapsulation** | Hiding the inside of a part (its fields and helper functions) and offering a small set of public operations instead. |
| **side effect** | Anything a function does besides returning a value: printing, reading a file, changing a global. |
| **pure function** | A function that only reads its arguments and returns a value, with no side effects. The same input always gives the same output. |
| **I/O** (Input/Output) | Everything that talks to the world outside the program: the keyboard, the screen, files, the network. |

## 1. Two measuring sticks

Think of a kitchen. In a well-organised kitchen, all knives are in one drawer and all spices on one shelf. That is **high cohesion**: the things in one place belong together. If you must open five drawers to find the pieces of one tool, cohesion is low.

Now think of two cooks sharing the kitchen. If cook A has to ask cook B before moving a pan, because B secretly depends on where it stands, they are **tightly coupled**. If each cook has their own station and they exchange finished plates through a window, coupling is **low**.

In code:

| | What it measures | Goal | Warning sign |
|---|---|---|---|
| **Cohesion** | Do the contents of this part belong together? | High | A module called `utils`; a struct whose fields are used by different, unrelated methods |
| **Coupling** | How much does this part know about the insides of other parts? | Low | A small change in one place forces edits in many others |

These two ideas are the base of most design advice. **Separation of concerns** is the practical habit that gives you both: put each concern in its own part, and let the parts talk through small, clear interfaces.

## 2. A program that does everything

A program reads lines like `ana 90` (a name and points), skips bad lines, prints each score and the average. Here it is in one block:

```rust
fn main() {
    let input = "ana 90\nbo 72\nbad line\ncy 85";
    let mut total = 0;
    let mut count = 0;
    for line in input.lines() {
        let parts: Vec<&str> = line.split_whitespace().collect();
        if parts.len() != 2 {
            println!("skipped: {line}");
            continue;
        }
        match parts[1].parse::<u32>() {
            Ok(points) => {
                total += points;
                count += 1;
                println!("{:<6}{:>4}", parts[0], points);
            }
            Err(_) => println!("skipped: {line}"),
        }
    }
    println!("average: {:.1}", total as f64 / count as f64);
}
```

**What happens:**

1. `input` is a fixed text here, so that the example is easy to run.
2. For every line, the code splits it into words (this is **parsing**) and checks that there are exactly two.
3. It parses the points, adds them to `total`, and prints the line at once (this is **reporting**).
4. At the end it computes the average and prints it (this is **calculating**).

```text
ana     90
bo      72
skipped: bad line
cy      85
average: 82.3
```

It works. But look at what happens when the requirements change, as they always do:

- "Read the lines from a file." The loop must change, and the parsing sits in the middle of it.
- "Print the report as CSV (comma-separated values)." You must edit the `println!` calls that are mixed into the parsing loop.
- "What if there are no valid lines?" `total as f64 / count as f64` divides zero by zero, and the program prints `average: NaN`. To fix it and to *test* the fix, you must run the whole program and read what it prints.

Three different topics (parsing, calculating, reporting) are tangled in one function. This is **low cohesion**: the function has three reasons to change. And you cannot test any single part alone, because every part prints.

## 3. Separate the concerns

We give every concern a home. Parsing turns text into a `Score`. Calculating works on plain numbers. Reporting turns data into text. Only `main` touches the screen:

```rust
mod parse {
    #[derive(Debug, PartialEq)]
    pub struct Score {
        pub name: String,
        pub points: u32,
    }

    pub fn parse_line(line: &str) -> Result<Score, String> {
        let parts: Vec<&str> = line.split_whitespace().collect();
        if parts.len() != 2 {
            return Err(format!("expected `name points`, got {line:?}"));
        }
        let points = parts[1]
            .parse::<u32>()
            .map_err(|_| format!("bad number in {line:?}"))?;
        Ok(Score {
            name: parts[0].to_string(),
            points,
        })
    }
}

mod stats {
    pub fn average(points: &[u32]) -> Option<f64> {
        if points.is_empty() {
            return None;
        }
        let total: u32 = points.iter().sum();
        Some(f64::from(total) / points.len() as f64)
    }
}

mod report {
    use crate::parse::Score;

    pub fn render(scores: &[Score], average: Option<f64>) -> String {
        let mut text = String::new();
        for score in scores {
            text.push_str(&format!("{:<6}{:>4}\n", score.name, score.points));
        }
        match average {
            Some(avg) => text.push_str(&format!("average: {avg:.1}\n")),
            None => text.push_str("average: n/a\n"),
        }
        text
    }
}

fn main() {
    let input = "ana 90\nbo 72\nbad line\ncy 85";
    let mut scores = Vec::new();
    for line in input.lines() {
        match parse::parse_line(line) {
            Ok(score) => scores.push(score),
            Err(message) => println!("skipped: {message}"),
        }
    }
    let points: Vec<u32> = scores.iter().map(|s| s.points).collect();
    print!("{}", report::render(&scores, stats::average(&points)));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_line_accepts_a_name_and_points() {
        let score = parse::parse_line("ana 90").unwrap();
        assert_eq!(
            score,
            parse::Score {
                name: "ana".to_string(),
                points: 90
            }
        );
    }

    #[test]
    fn parse_line_rejects_bad_input() {
        assert!(parse::parse_line("bad line").is_err());
        assert!(parse::parse_line("ana").is_err());
    }

    #[test]
    fn average_of_nothing_is_none() {
        assert_eq!(stats::average(&[]), None);
        assert_eq!(stats::average(&[90, 80]), Some(85.0));
    }

    #[test]
    fn render_prints_one_line_per_score_and_the_average() {
        let scores = vec![parse::Score {
            name: "bo".to_string(),
            points: 72,
        }];
        assert_eq!(
            report::render(&scores, Some(72.0)),
            "bo      72\naverage: 72.0\n"
        );
    }
}
```

**What happens:**

1. `mod parse` knows how a line looks. `parse_line` returns `Result<Score, String>`: either a score, or a message that says what is wrong.
2. `mod stats` knows only about numbers. `average` takes `&[u32]` (a slice) and returns `Option<f64>`: `None` for an empty list, so the divide-by-zero case cannot even happen.
3. `mod report` knows how a report looks. `render` does **not** print; it builds and returns a `String`. Changing to CSV means changing only this function.
4. `main` is the only place with side effects. It reads the input, collects the scores, and prints.

```text
skipped: bad number in "bad line"
ana     90
bo      72
cy      85
average: 82.3
```

The skipped-line message appears first because `main` prints it while it collects the scores, before the report is printed.

Now every part is a **pure function**, and testing is easy. The tests at the end of the file check each concern alone, with no printing:

```text
running 4 tests
test tests::average_of_nothing_is_none ... ok
test tests::parse_line_accepts_a_name_and_points ... ok
test tests::parse_line_rejects_bad_input ... ok
test tests::render_prints_one_line_per_score_and_the_average ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** A good rule: keep the "thinking" in pure functions, and keep the side effects (reading, printing, saving) in a thin layer at the edge, usually `main`. Pure functions are easy to test; the thin layer is so small that it has little to get wrong. Some engineers call this "functional core, imperative shell".

> 📏 **Convention:** Inside a function, a good test of cohesion is its name. If you need "and" in the name (`parse_and_print`), it probably does two things.

## 4. Reduce coupling

### Hide the representation

Look at the module `inventory` below. It stores items in a `Vec`. The field is private, and the code outside tries to reach into it:

```rust
mod inventory {
    pub struct Inventory {
        items: Vec<(String, u32)>,
    }

    impl Inventory {
        pub fn new() -> Self {
            Inventory { items: Vec::new() }
        }
        pub fn add(&mut self, name: &str, count: u32) {
            self.items.push((name.to_string(), count));
        }
    }
}

fn main() {
    let mut inv = inventory::Inventory::new();
    inv.add("bolt", 40);
    println!("{}", inv.items[0].1);
}
```

```text
error[E0616]: field `items` of struct `Inventory` is private
  --> src/main.rs:19:24
   |
19 |     println!("{}", inv.items[0].1);
   |                        ^^^^^ private field

For more information about this error, try `rustc --explain E0616`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

The compiler enforces low coupling here. The outside code is not allowed to know that the items are in a vector, so it cannot depend on it. The `Inventory` author can now change the storage, for example to a `HashMap`, and only the code inside the module must change:

```rust
mod inventory {
    use std::collections::HashMap;

    pub struct Inventory {
        items: HashMap<String, u32>, // was a Vec before; nobody outside notices
    }

    impl Inventory {
        pub fn new() -> Self {
            Inventory {
                items: HashMap::new(),
            }
        }
        pub fn add(&mut self, name: &str, count: u32) {
            *self.items.entry(name.to_string()).or_insert(0) += count;
        }
        pub fn count(&self, name: &str) -> u32 {
            self.items.get(name).copied().unwrap_or(0)
        }
    }
}

fn main() {
    let mut inv = inventory::Inventory::new();
    inv.add("bolt", 40);
    inv.add("bolt", 2);
    println!("{}", inv.count("bolt"));
    println!("{}", inv.count("nut"));
}
```

**What happens:**

1. The field `items` is private. The only public operations are `new`, `add` and `count`: that is the whole **interface** of the module.
2. `add` and `count` were rewritten for the `HashMap`, but `main` is exactly what a user of the old `Vec` version would write, and it still works.
3. The program prints the count of `"bolt"` (40 + 2) and of `"nut"` (never added).

```text
42
0
```

This is **encapsulation**. If the field were `pub`, every user could write `inv.items[0]`, and the storage could never change without breaking them all.

> 💡 **Tip:** Start with everything private. Make an item `pub` only when something outside needs it, and use `pub(crate)` for items that other modules of your crate need but the outside world should not see. It is easy to make something public later, but very hard to take it back.

### Ask for the least

The function `stats::average` takes `&[u32]`, a plain list of numbers. It could have taken `&[Score]`, but then `stats` would need to know about `Score` and the `parse` module. Because it asks only for what it really needs, it works with scores, with temperatures, with anything. The more specific the parameter, the tighter the coupling.

A related habit: do not reach through many layers, such as `order.customer().address().city().name()`. Every dot is a dependency on the inside of another type. If you need the city name, ask the order for it with one method (`order.shipping_city()`), and let the order decide how to find it. This habit has a name, the *Law of Demeter*, but the practical version is simple: talk to your neighbours, not to your neighbours' neighbours.

## 5. Raise cohesion

Cohesion applies to types too. This `Order` holds two groups of fields. One group is used by `total_cents`, the other by `greeting`:

```rust
struct Order {
    customer_name: String,
    customer_email: String,
    prices_cents: Vec<u64>,
    discount_percent: u64,
}

impl Order {
    fn total_cents(&self) -> u64 {
        let sum: u64 = self.prices_cents.iter().sum();
        sum - sum * self.discount_percent / 100
    }
    fn greeting(&self) -> String {
        format!("Dear {} <{}>", self.customer_name, self.customer_email)
    }
}

fn main() {
    let order = Order {
        customer_name: "Ana".to_string(),
        customer_email: "ana@mail.org".to_string(),
        prices_cents: vec![1000, 550],
        discount_percent: 10,
    };
    println!("{}", order.greeting());
    println!("{}", order.total_cents());
}
```

```text
Dear Ana <ana@mail.org>
1395
```

Neither method uses all the fields. This is a sign that `Order` is really two things glued together. When the way you contact customers changes (a phone number, a language), `Order` changes, even though nothing about prices changed. Split it:

```rust
struct Customer {
    name: String,
    email: String,
}

impl Customer {
    fn greeting(&self) -> String {
        format!("Dear {} <{}>", self.name, self.email)
    }
}

struct Order {
    prices_cents: Vec<u64>,
    discount_percent: u64,
}

impl Order {
    fn total_cents(&self) -> u64 {
        let sum: u64 = self.prices_cents.iter().sum();
        sum - sum * self.discount_percent / 100
    }
}

fn main() {
    let customer = Customer {
        name: "Ana".to_string(),
        email: "ana@mail.org".to_string(),
    };
    let order = Order {
        prices_cents: vec![1000, 550],
        discount_percent: 10,
    };
    println!("{}", customer.greeting());
    println!("{}", order.total_cents());
}
```

The output is the same, but now each struct has one reason to change. You can also use `Customer` for other things, such as newsletters, without dragging a price list along. (The next lessons use this "has a" building block a lot: Lesson 08.4 explains composition.)

> 📏 **Convention:** A quick check for a struct: list its fields and its methods, and mark which method uses which field. If the marks form two separate clusters, you probably have two structs.

## 6. When NOT to split

Separation has a price: more names, more files, more places to look. Be careful in these cases:

- **Too small.** A module with one function that is called once is noise. Split when a part has its own reason to change, or when you want to test it alone, not before (this is YAGNI from Lesson 08.1).
- **Pass-through layers.** A function that only calls another function with the same arguments adds a hop and no value.
- **Splitting the same concern.** Cutting one idea into three modules just to make files shorter can *raise* coupling, because the modules must now know about each other's insides.
- **Measuring lines instead of reasons.** A 200-line function with one clear purpose can be better than ten 20-line functions that are all needed together. The question is always: what changes for what reason?

## Common mistakes

**Mistake 1: the `utils` module.** A module called `utils`, `helpers` or `common` collects whatever does not fit elsewhere: string tricks, date parsing, email sending. It has no purpose, so it has the lowest cohesion possible. Name modules after what they are about (`text`, `dates`, `mail`).

**Mistake 2: everything `pub`.** Making all fields and functions public "to be safe" couples every part to every other. If something is `pub` it is a promise.

**Mistake 3: functions that print.** A function that calls `println!` in the middle of its logic cannot be reused for a file, a web page or a test. Return a `String` or a value and let the caller decide.

**Mistake 4: modules that depend on each other in both directions.** If `parse` uses `report`, and `report` uses `parse`, you cannot change either alone. Within one crate Rust accepts such cycles, but they are a design problem; draw the arrows between your modules and keep them pointing in one direction (Lesson 08.14 shows layers).

## Try it yourself

1. Create the package and start with the tangled program:

```bash
cd ~/projects/rust
cargo new code/p08-coupling
cd code/p08-coupling
```

2. Put the tangled program from Section 2 (the one where `main` does everything) in `src/main.rs` and run it with `cargo run`. You see the output from Section 2.

3. Try to write a test for "the average of an empty list". Notice that you cannot: there is no function to call.

4. Refactor step by step, running `cargo run` after each step; the output must not change (apart from the order of the skip message):
   1. Move parsing into `mod parse` with `parse_line`.
   2. Move the average into `mod stats` as a function over `&[u32]` that returns `Option<f64>`.
   3. Move formatting into `mod report` as a function that returns a `String`.
   4. Make `main` do only the glue.

5. Add the four tests from Section 3 (or your own versions), and run:

```bash
cargo test
cargo clippy
git add .
git commit -m "refactor(p08): separate parse, stats and report"
```

**What happens:** In step 3 you could not test the average because it was buried inside `main`. After the split, `stats::average(&[])` is a one-line test. Testability is a very good measure of separation.

## Exercises

### Exercise 1 — Cohesion or coupling? (easy)

For each sentence, say whether it is about cohesion or about coupling, and whether it describes something good or bad:

1. "Changing the name of a field in `Config` forced me to edit nine other files."
2. "Everything in `geometry.rs` is about shapes and their areas."
3. "The `Report` struct has a field only used by the email code and a field only used by the printing code."

<details>
<summary>Hint</summary>

Coupling is about the arrows between parts. Cohesion is about what is inside one part.

</details>

<details>
<summary>Answer</summary>

1. Coupling, bad (tight): other files depend on a detail of `Config`.
2. Cohesion, good (high): the module has one topic.
3. Cohesion, bad (low): two unrelated groups of fields live in one struct.

Why it works: coupling = arrows between parts; cohesion = unity inside a part.

</details>

### Exercise 2 — Hide the representation (medium)

The module `temp` exposes its field. Make the field private, give the type a constructor `from_celsius` and a method `fahrenheit`, and update `main`. Then **write a test** that `100.0` °C is `212.0` °F and `0.0` °C is `32.0` °F.

```rust
mod temp {
    pub struct Reading {
        pub celsius: f64,
    }
}

fn main() {
    let r = temp::Reading { celsius: 100.0 };
    println!("{}", r.celsius * 9.0 / 5.0 + 32.0);
}
```

<details>
<summary>Hint</summary>

Do not add a getter for `celsius` unless something needs it (YAGNI). A function that creates a value is usually called a constructor; in Rust it is an associated function without `self`.

</details>

<details>
<summary>Answer</summary>

```rust
mod temp {
    pub struct Reading {
        celsius: f64,
    }

    impl Reading {
        pub fn from_celsius(celsius: f64) -> Self {
            Reading { celsius }
        }
        pub fn fahrenheit(&self) -> f64 {
            self.celsius * 9.0 / 5.0 + 32.0
        }
    }
}

fn main() {
    let r = temp::Reading::from_celsius(100.0);
    println!("{}", r.fahrenheit());
}

#[cfg(test)]
mod tests {
    use super::temp::Reading;

    #[test]
    fn boiling_point_in_fahrenheit() {
        assert_eq!(Reading::from_celsius(100.0).fahrenheit(), 212.0);
        assert_eq!(Reading::from_celsius(0.0).fahrenheit(), 32.0);
    }
}
```

Why it works: outside the module nobody can see the field, so the author may later store kelvin or fahrenheit instead and only edit `temp`. `main` and the test use the public interface only.

</details>

### Exercise 3 — Sort the grab-bag (hard)

A project has a file `utils.rs` with four functions: `slugify` (turns a title into a text that is safe for a URL (Uniform Resource Locator, a web address)), `parse_date`, `send_email` and `hash_password`. Propose modules with names that tell their purpose, and explain why these functions should not stay together.

<details>
<summary>Hint</summary>

Ask for each function: what would force it to change? A new email provider changes `send_email`; a new date format changes `parse_date`; a new hashing algorithm changes `hash_password`.

</details>

<details>
<summary>Answer</summary>

```rust
#![allow(dead_code)]

mod text {
    pub fn slugify(title: &str) -> String {
        title.to_lowercase().replace(' ', "-")
    }
}

mod dates {
    pub fn parse_date(text: &str) -> Option<(u32, u32, u32)> {
        let mut parts = text.split('-').map(|p| p.parse::<u32>().ok());
        Some((parts.next()??, parts.next()??, parts.next()??))
    }
}

mod mail {
    pub fn send_email(to: &str, body: &str) {
        println!("to {to}: {body}");
    }
}

mod passwords {
    pub fn hash_password(password: &str) -> String {
        format!("hash-of-{}", password.len()) // placeholder, not real hashing
    }
}

fn main() {
    println!("{}", text::slugify("Hello World"));
    println!("{:?}", dates::parse_date("2026-11-18"));
    mail::send_email("ana@mail.org", "hi");
    println!("{}", passwords::hash_password("secret"));
}
```

Why it works: each module changes for its own reason, and a reader looking for password code looks in `passwords`. The bodies here are only small placeholders (the `hash_password` function does not really hash). In a real project, `passwords` and `mail` would be security-sensitive, so keeping them in separate modules also makes them easy to review. The functions have nothing in common except "we did not know where to put them" — a sign of low cohesion.

</details>

## Quiz

1. What do we want: high or low cohesion? High or low coupling?

<details>
<summary>Answer</summary>

High cohesion and low coupling.

</details>

2. Why does making a field private reduce coupling?

<details>
<summary>Answer</summary>

Other code cannot depend on how the data is stored, so the owner can change the storage freely.

</details>

3. Why is `render(...) -> String` easier to test than `print_report(...)`?

<details>
<summary>Answer</summary>

It has no side effects; a test can call it and compare the returned text with `assert_eq!`.

</details>

4. Which part of the program should do the input and output in a well-separated design?

<details>
<summary>Answer</summary>

A thin outer layer, usually `main`; the rest should be pure functions.

</details>

5. Name one danger of splitting code into too many tiny modules.

<details>
<summary>Answer</summary>

More names and files to navigate, pass-through functions that add nothing, and sometimes higher coupling because the pieces must know about each other.

</details>

## Summary

- **Cohesion** is how well the contents of one part belong together; **coupling** is how much parts depend on each other's insides. Aim for high cohesion and low coupling.
- **Separation of concerns**: parsing, calculating and reporting belong in different places.
- Keep side effects (printing, files, network) in a thin outer layer; build the middle from pure functions that are easy to test.
- Hide representation with private fields; the compiler (error E0616) then enforces low coupling.
- Ask only for what you need: `&[u32]` instead of `&Vec<Score>`; avoid long chains of calls through other types.
- A struct whose fields split into two clusters used by different methods is probably two structs.
- Do not split for its own sake: modules named `utils`, pass-through layers and one-function modules cost more than they give.

## Sources and further reading

- [The Rust Programming Language — 7. Managing Growing Projects with Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html) — the official chapter on modules and privacy.
- [The Rust Programming Language — 18.1 Characteristics of Object-Oriented Languages](https://doc.rust-lang.org/book/ch18-01-what-is-oo.html) — includes a section on encapsulation that hides implementation details.
- [Rust API Guidelines — Future proofing](https://rust-lang.github.io/api-guidelines/future-proofing.html) — why private fields let you change a type without breaking users.
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) — a community book of Rust idioms, patterns and anti-patterns; later lessons of this phase use it again.

---

[🏠 Start](../START-HERE.md) · [← 08.1 Good design: KISS, DRY and YAGNI](01-good-design-kiss-dry-yagni.md) · [📋 Phase 08](README.md) · [08.3 The SOLID principles →](03-solid-principles.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
