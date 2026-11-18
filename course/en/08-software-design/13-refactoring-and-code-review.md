<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.12 Project layout and workspaces](12-project-layout-and-workspaces.md) · [📋 Phase 08](README.md) · [08.14 Layers and hexagonal architecture →](14-layers-and-hexagonal-architecture.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/13-refactoring-and-code-review.md) <!-- nav -->

# Lesson 08.13 — Refactoring and code review

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 13 of 17 · ⏱ about 120 minutes

## What you will learn

- What refactoring is, and why it needs tests first.
- How to write *characterization tests* that pin down what messy code does today.
- A safe refactoring loop with small steps, shown on one real function, including a "harmless" change that the tests catch.
- The most useful refactorings in Rust, and how the compiler and the IDE (Integrated Development Environment) help.
- How to review someone else's code, and how to write and receive review comments.

## Before you start

- You finished [Lesson 08.12 — Project layout and workspaces](12-project-layout-and-workspaces.md).
- You know the smells of [Lesson 08.10](10-anti-patterns-and-code-smells.md), the quality gate of [Lesson 08.11](11-conventions-naming-clippy-and-api-guidelines.md) and how to write tests ([Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)).
- You know pull requests from [Lesson 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md). This lesson is about *what* to look for in a review; Lesson 01.9 is about the mechanics.
- Create a library: `cargo new code/p08-invoice --lib`.

## New words

| Word | Meaning |
|---|---|
| **refactoring** | Changing the internal structure of code to make it easier to understand and change, **without changing what it does** for its users. |
| **observable behaviour** | Everything that callers can see: return values, output, errors, side effects. Refactoring keeps it the same. |
| **characterization test** (golden test) | A test that records what the code does *now*, including odd results, so that you notice when a change alters it. |
| **safety net** | A set of fast tests that tell you within seconds if a change broke something. |
| **code review** | Reading a change by someone else before it is merged, to find bugs and design problems and to share knowledge. |
| **nit** | A very small review comment about taste or style ("nit: rename this"). The author may ignore it. |
| **IDE** (Integrated Development Environment) | A program for writing code with search, refactoring and debugging tools built in. RustRover is one. |

## 1. What refactoring is

**Refactoring** means improving the *structure* of code while keeping its *behaviour*. You do not add features, you do not fix
bugs; you make the code easier to read, test and change. Two rules define it:

1. **Behaviour stays the same.** If the output changes, it is not a refactoring; it is a change.
2. **You work in small steps**, and after each step the code compiles and the tests pass.

Think of two hats. With the *feature* hat you add behaviour and write new tests. With the *refactoring* hat you change structure and
touch no tests. Wear one hat at a time, and commit between hats. A commit that mixes both cannot be reviewed, and when a bug appears you cannot tell which change caused it.

The most important sentence of this lesson: **you can only refactor safely if something tells you quickly that the behaviour has not changed.** That something is a set of tests.

## 2. The safety net: characterization tests

What if the code has no tests, which is the usual situation? You write tests that describe what it does *today*. They are called
**characterization tests**. The rules:

- Do not decide what the code *should* do. Run it, look at the result, and write that result into the test.
- Include odd cases, even if the result looks wrong (rounding, empty input, unknown kind). Somebody may depend on it. If you believe it is a bug, record it, finish the refactoring, and fix the bug in a separate commit.
- Cover every branch. A branch without a case is a branch that a refactoring can silently break.

Here is our patient, a function that prints one line of an invoice. Put it in `code/p08-invoice/src/lib.rs`:

```rust
pub fn invoice_line(qty: u32, unit_cents: u32, kind: &str, vip: bool) -> String {
    let mut total = qty * unit_cents;
    if kind == "book" {
        if qty >= 10 {
            total = total * 85 / 100;
        } else {
            total = total * 95 / 100;
        }
    } else if kind == "food" {
        total = total + total * 7 / 100;
    } else {
        if qty >= 10 {
            total = total * 90 / 100;
        }
    }
    if vip {
        total = total - total * 5 / 100;
    }
    let dollars = total / 100;
    let cents = total % 100;
    if cents < 10 {
        format!("{} x {} = {}.0{}", qty, kind, dollars, cents)
    } else {
        format!("{} x {} = {}.{}", qty, kind, dollars, cents)
    }
}
```

**What happens:** the function multiplies quantity by price, applies rules by kind of goods (books: 5% off, 15% off from ten pieces; food: 7% tax; others: 10% off
from ten pieces), takes 5% off for VIP (Very Important Person) customers, and formats the money in two ways. All amounts are whole cents, and divisions round down. Smells: magic numbers,
a stringly-typed `kind`, nested `if`, duplicated formatting.

To write the characterization test, run the function on a handful of inputs and copy the results. Choose at least one input for each branch. Add this at the end of the file:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn pins_down_current_behaviour() {
        let cases = [
            // (qty, unit_cents, kind, vip, expected text)
            (1, 1000, "book", false, "1 x book = 9.50"),
            (10, 1000, "book", false, "10 x book = 85.00"),
            (3, 199, "food", false, "3 x food = 6.38"),
            (3, 199, "food", true, "3 x food = 6.07"),
            (10, 250, "toy", false, "10 x toy = 22.50"),
            (2, 250, "toy", true, "2 x toy = 4.75"),
            (1, 1, "pen", false, "1 x pen = 0.01"),
            (1, 1005, "pen", false, "1 x pen = 10.05"),
            (7, 333, "gift", true, "7 x gift = 22.15"),
        ];
        for (qty, unit, kind, vip, expected) in cases {
            assert_eq!(invoice_line(qty, unit, kind, vip), expected);
        }
    }
}
```

**What happens:**

1. Each tuple is one input and the output we saw when we ran the original code. For example, 3 pieces of food at 199 cents: 597, plus 7% tax (`597 * 7 / 100 = 41`) is 638, which prints as `6.38`.
2. `1 x pen = 0.01` and `10.05` cover both formatting branches (cents below 10 and not).
3. The test is a *table*; to add a case you add a line (Lesson 05.6).

`cargo test` passes on the untouched code:

```text
running 1 test
test tests::pins_down_current_behaviour ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Commit this: `git add . && git commit -m "test(p08): characterization tests for invoice_line"`. The safety net is in place.

> 🧪 **Testing:** For a program that prints text, a characterization test often compares the whole output with a stored "golden" file. That is the technique of the project in Lesson 08.16.
> Phase 09 teaches snapshot testing, which automates it.

## 3. The loop: change, test, commit

Now refactor in steps. After each step, run `cargo test`. If it is red, undo the last step (`git restore .`) instead of debugging a big change.

**Step 1: remove the duplicated formatting.** Both branches print the same, except for a zero. The format `{:02}` pads a number to two digits with zeros. Extract the money formatting into a function:

```rust
fn format_money(cents_total: u32) -> String {
    format!("{}.{:02}", cents_total / 100, cents_total % 100)
}
```

and replace the last `if cents < 10 { ... } else { ... }` of `invoice_line` with one line: `format!("{} x {} = {}", qty, kind, format_money(total))`. Green. Commit: `refactor(p08): extract format_money`.

**Step 2: a "harmless" simplification that is not.** The VIP line `total = total - total * 5 / 100;` looks like a clumsy way to write `total * 95 / 100`. Try it:

```text
---- tests::pins_down_current_behaviour stdout ----

thread 'tests::pins_down_current_behaviour' (4805567) panicked at src/lib.rs:45:13:
assertion `left == right` failed
  left: "3 x food = 6.06"
 right: "3 x food = 6.07"
```

(The line number and the thread number differ on your computer.)

The two formulas are equal in school arithmetic but not with whole numbers that round down: `638 - 638 * 5 / 100` is `638 - 31 = 607`, while `638 * 95 / 100` is `606`. A reviewer would likely
approve the "simplification". The test caught it in a second. Undo it. (If the business rule really is "95% rounded down", that is a *behaviour change*: a feature-hat commit with its own test.)

**Step 3: name the numbers.** Replace `10`, `7`, `5`, `85`, `95` and `90` by constants whose names say what they mean. Green.

**Step 4: replace the string with an enum.** The comparisons `kind == "book"` are the stringly-typed smell. Create an `enum Kind { Book, Food, Other }` with a `parse` function that maps `"book"` and `"food"`, and everything else to `Other`
(that is what the old code did with an unknown kind, and the tests pin it). The public function keeps its signature, so the tests do not change. Replace the chain of `if` by a `match` on `(kind, bulk)`. Green.

**Step 5: extract the arithmetic.** `x * p / 100` appears many times. Name it `percent_of`. Green.

Here is the code after the five steps. The tests module is unchanged and stays at the end of the file:

```rust
const BULK_QUANTITY: u32 = 10;
const FOOD_TAX_PERCENT: u32 = 7;
const VIP_DISCOUNT_PERCENT: u32 = 5;
const BOOK_BULK_PAY_PERCENT: u32 = 85;
const BOOK_PAY_PERCENT: u32 = 95;
const OTHER_BULK_PAY_PERCENT: u32 = 90;

#[derive(Clone, Copy)]
enum Kind {
    Book,
    Food,
    Other,
}

impl Kind {
    fn parse(text: &str) -> Kind {
        match text {
            "book" => Kind::Book,
            "food" => Kind::Food,
            _ => Kind::Other,
        }
    }
}

/// `percent` percent of `amount`, rounded down.
fn percent_of(amount: u32, percent: u32) -> u32 {
    amount * percent / 100
}

fn after_kind_rules(total: u32, qty: u32, kind: Kind) -> u32 {
    let bulk = qty >= BULK_QUANTITY;
    match (kind, bulk) {
        (Kind::Book, true) => percent_of(total, BOOK_BULK_PAY_PERCENT),
        (Kind::Book, false) => percent_of(total, BOOK_PAY_PERCENT),
        (Kind::Food, _) => total + percent_of(total, FOOD_TAX_PERCENT),
        (Kind::Other, true) => percent_of(total, OTHER_BULK_PAY_PERCENT),
        (Kind::Other, false) => total,
    }
}

fn format_money(cents_total: u32) -> String {
    format!("{}.{:02}", cents_total / 100, cents_total % 100)
}

pub fn invoice_line(qty: u32, unit_cents: u32, kind: &str, vip: bool) -> String {
    let mut total = after_kind_rules(qty * unit_cents, qty, Kind::parse(kind));
    if vip {
        total -= percent_of(total, VIP_DISCOUNT_PERCENT);
    }
    format!("{qty} x {kind} = {}", format_money(total))
}
```

**What happens:**

1. `invoice_line` is now four lines that read like the business rule: compute the total, apply the rules for the kind, take the VIP discount, format.
2. The rules for the kinds sit in one `match` with no `_` arm, so a new kind (`Kind::Toy`) makes the compiler point at the one place to update.
3. `(Kind::Food, _)` shows what the old code did implicitly: food has no bulk discount.
4. The rounding is still "divide last, round down", exactly as before: the tests prove it. `cargo test` and `cargo clippy -- -D warnings` are green.

The result has more lines than the original, and that is fine. Refactoring is not about fewer lines; it is about the next change being easy. A good question to ask: "What would I have to touch to add a new kind of goods, or change the VIP
discount?" Before, you read all 25 lines; now you touch one line.

## 4. A catalogue of useful refactorings

| Refactoring | Use it when | In Rust |
|---|---|---|
| **Rename** | A name hides the meaning | The IDE renames everywhere, including other files and tests. |
| **Extract function** | A block has a name in your head ("compute the VIP price") | The IDE's *Extract Function* detects the parameters and the return type. |
| **Inline function** | A function only hides a trivial expression | The opposite of extract. |
| **Introduce constant** | A magic number | `const NAME: u32 = ...;` |
| **Replace string/flag with enum** | A stringly-typed value or boolean argument | `enum` + `match` with no `_` arm. |
| **Introduce parameter object** | The same 3 arguments travel together | A small `struct`; later a builder (Lesson 08.6). |
| **Replace nested `if` with early return** | Pyramids of braces | `return`, `?` and `let ... else { return ...; }` |
| **Replace loop with iterator chain** | A loop that builds a value | `map`, `filter`, `sum`, `collect` (Lesson 05.5). Only if it reads better. |
| **Move function to the type that owns the data** | A function mostly uses another type's fields (feature envy) | Make it a method in that type's `impl`. |
| **Split module / extract crate** | A file or module does several jobs | New file with `mod`; a crate only for a real boundary (Lesson 08.12). |
| **Replace `clone` by a borrow** | Clones added to satisfy the borrow checker | Change `String` parameters to `&str` (Lesson 08.10). |

Two Rust-specific helpers deserve a special mention.

**Lean on the compiler.** Rust's type checker is a very good safety net for *structural* changes. Want to change a field from `u32` to a newtype `Cents`? Change the definition and follow the compile errors one by one; when the code compiles again, every use has been
updated. The same works for adding a variant to an `enum` that is matched without `_`: the compiler lists every `match` to update. This is why we avoid `_ =>` arms for enums we own.

**Use the tools.** RustRover's *Refactor* menu has *Rename*, *Extract Function*, *Inline*, *Change Signature*, *Move* and more, and shows a preview of the changes. The keyboard shortcut of each item is written in the menu; *Rename* is `Shift+F6`.
Tool-made refactorings are usually safe, but you still run the tests. And `cargo clippy --fix` (Lesson 08.11) applies the mechanical ones.

## 5. When not to refactor

- **There are no tests and you cannot write any.** Write characterization tests first, or leave the code alone.
- **The code never changes.** An ugly function that has been stable for years costs nothing; do not touch it for beauty.
- **A deadline is tomorrow.** Refactoring in the same change as an urgent fix makes the change risky. Fix first; refactor in the next change.
- **You want to rewrite everything.** A big rewrite has no safety net and loses forgotten knowledge hidden in the old code (those "odd" cases). Improve it step by step instead.
- **There is no smell with a problem** (Lesson 08.10). "I would write it differently" is not a reason.

## 6. Code review

A **code review** is when another person reads your change before it is merged. Lesson 01.9 showed the pull-request workflow. Here is the thinking.

### 6.1 Why review

Review finds bugs, but its other benefits are as important: the code stays understandable for at least two people, the team learns each other's techniques, and the standard of the code rises. Nobody writes perfect code alone.

### 6.2 If you are the author

- **Keep the change small**: one idea, ideally a few hundred lines at most. Nobody can really review 2,000 lines.
- **Explain in the description** what changed and *why*, and how you tested it (the What / Why / How / Testing template of Lesson 01.9).
- **Review your own diff first.** You will find leftover debug prints and forgotten files.
- **Run the quality gate** (`cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`) before you ask. A human should not tell you about a missing space.
- **Separate refactoring from behaviour change** into different commits or pull requests.

### 6.3 If you are the reviewer

Read in this order. The earlier items are the expensive ones to fix later:

1. **Design.** Does the change belong here? Does it respect the layers and boundaries (Lesson 08.14)? Is a simpler solution possible?
2. **Correctness.** Edge cases: empty input, zero, very large numbers, errors. What happens when it fails?
3. **Tests.** Is the new behaviour tested, including the failure cases? Would the test fail if the code were wrong?
4. **Readability.** Can you understand it without the author? Names, function size, comments that explain *why*.
5. **Style.** Last, and only what no tool checks. Formatting and Clippy belong to the machine.

A Rust-specific checklist to keep next to you:

| Look for | Question |
|---|---|
| `unwrap()` / `expect()` | Can this fail because of input from outside? Then it must be a `Result`. |
| `clone()` | Is it needed, or does a signature take an owned value without reason? |
| `pub` items, public fields | Is each really part of the API (Application Programming Interface)? Could it be private or `pub(crate)`? |
| New dependency in `Cargo.toml` | Is it needed, maintained, licensed correctly (Phase 10)? |
| `unsafe` | Is there a `// SAFETY:` comment, and is the reasoning correct (Phase 06, Phase 10)? |
| Error handling | Are errors typed and informative? Is an error swallowed with `let _ =`? |
| Public functions | Documented? `# Errors` and `# Panics` sections? An example? |
| Numbers | Overflow? Casts with `as`? Money in `f64`? |

### 6.4 How to write a comment

- **Comment on the code, not the person.** "This function can panic on an empty list" is better than "you forgot the empty list".
- **Say why, and offer a way.** "`&Vec<f64>` forces callers to own a `Vec`; `&[f64]` accepts more."
- **Label the weight** so that the author knows what is required. Common labels: `blocking:` (must change before merge), `suggestion:` (a good idea; your choice), `question:` (I do not understand, please explain), `nit:` (taste; ignore if you like).
- **Praise good things.** A short "nice test" is information, too.
- **Ask a question when unsure.** You may have missed the reason.

### 6.5 If you receive comments

Do not defend; understand. If you disagree, explain your reasoning with facts (a test, a measurement, a link). If a thread goes beyond two rounds, talk to the person. When the reviewer is right, change the code; when you both are right, write the reason down in a code comment so that the next reader does not ask again.

## Common mistakes

1. **Refactoring without tests.** You hope that nothing broke. Write characterization tests first.
2. **A giant step.** Ten changes in one commit, and one of them broke a test. Use steps of a few minutes each and commit often.
3. **Mixing a refactoring with a bug fix or a feature.** Separate commits.
4. **"Fixing" the odd result in a characterization test.** The test records what is; change the expected value only in a deliberate, separate behaviour change.
5. **Reviewing only style.** The machine does the style. People look at design, correctness and tests.
6. **A review comment that is only "this is bad".** Say what is wrong, why, and what you would do.

## Try it yourself

Do the five steps of Section 3 on your own copy, committing after each green run. Use a branch: `git switch -c refactor/invoice-line`.

1. Write the characterization test and commit it (Section 2).
2. Extract `format_money`. Run `cargo test`. Commit.
3. Try replacing the VIP line by `total * 95 / 100` and see the red test (the output in Section 3). Undo with `git restore .`.
4. Introduce constants, then the enum and `match`, then `percent_of`. Run `cargo test` and `cargo clippy -- -D warnings` after each step. Commit after each.
5. Add `Kind::Toy`, which gives 20% off from 5 pieces. First add a failing test for it (the feature hat!). Then make it pass. Notice how the compiler shows you the `match` to update.

## Exercises

### Exercise 1 — Write the safety net (medium)

The function `fn shipping(weight_grams: u32, express: bool) -> u32` returns 500 for weights under 1000 grams, 900 for weights under 5000, and 1500 otherwise; `express` doubles the price. You know only this
description. Write a characterization-style test with a table that covers every branch **and** the borders (999, 1000, 4999, 5000).

<details>
<summary>Hint</summary>

Borders are where `<` and `<=` mistakes live. For each border, test the number just below and the number at the border. Include `express = true` for at least two cases.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn shipping(weight_grams: u32, express: bool) -> u32 {
    let base = if weight_grams < 1000 {
        500
    } else if weight_grams < 5000 {
        900
    } else {
        1500
    };
    if express { base * 2 } else { base }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn covers_every_branch_and_border() {
        let cases = [
            // (weight, express, expected)
            (0, false, 500),
            (999, false, 500),
            (1000, false, 900),
            (4999, false, 900),
            (5000, false, 1500),
            (999, true, 1000),
            (5000, true, 3000),
        ];
        for (weight, express, expected) in cases {
            assert_eq!(
                shipping(weight, express),
                expected,
                "weight {weight}, express {express}"
            );
        }
    }
}
```

Why it works: each tier and each border is in the table, so changing `<` to `<=` or a price would turn the test red. The custom message after `expected` tells you which case failed.

</details>

### Exercise 2 — Refactor under test (medium)

Using the table of Exercise 1, refactor `shipping` into three parts: named constants for the limits and prices, a function `base_price(weight_grams: u32) -> u32`, and a function `shipping` that applies `express`. Show that the
same test still passes.

<details>
<summary>Hint</summary>

Copy the `if` chain into `base_price`, replace the numbers by constants one at a time, and run the test after each move.

</details>

<details>
<summary>Answer</summary>

```rust
const LIGHT_LIMIT_GRAMS: u32 = 1_000;
const MEDIUM_LIMIT_GRAMS: u32 = 5_000;
const LIGHT_PRICE: u32 = 500;
const MEDIUM_PRICE: u32 = 900;
const HEAVY_PRICE: u32 = 1_500;
const EXPRESS_FACTOR: u32 = 2;

fn base_price(weight_grams: u32) -> u32 {
    if weight_grams < LIGHT_LIMIT_GRAMS {
        LIGHT_PRICE
    } else if weight_grams < MEDIUM_LIMIT_GRAMS {
        MEDIUM_PRICE
    } else {
        HEAVY_PRICE
    }
}

pub fn shipping(weight_grams: u32, express: bool) -> u32 {
    let base = base_price(weight_grams);
    if express { base * EXPRESS_FACTOR } else { base }
}
```

The `tests` module of Exercise 1 is unchanged and passes. (The `express: bool` parameter is itself a smell, Lesson 08.10; changing it to an `enum` changes the signature, so you would update the test table in a separate step.)

</details>

### Exercise 3 — Review a change (hard)

A teammate opens a pull request with this code. Write at least five review comments, each with a label (`blocking:`, `suggestion:`, `question:` or `nit:`).

```rust
pub fn load_prices(path: &str) -> Vec<f64> {
    let text = std::fs::read_to_string(path).unwrap();
    let mut prices = Vec::new();
    for line in text.lines().clone() {
        let p: f64 = line.parse().unwrap();
        prices.push(p);
    }
    prices
}

pub fn average(prices: &Vec<f64>) -> f64 {
    let mut sum = 0.0;
    for i in 0..prices.len() { sum += prices[i]; }
    sum / prices.len() as f64
}
```

<details>
<summary>Hint</summary>

Use the Rust checklist of Section 6.3 line by line. Also ask what `average(&vec![])` returns, and whether money should be a float. The PR (pull request) contains no tests.

</details>

<details>
<summary>Answer</summary>

Possible comments:

1. `blocking:` `read_to_string(path).unwrap()` and `line.parse().unwrap()` crash the program on a missing file or a bad line, and the input comes from outside. Return a `Result` (for example `io::Result<Vec<_>>`) and tell the caller which line was bad.
2. `blocking:` `average` of an empty list divides by zero and returns `NaN` (checked: `average(&vec![])` prints `NaN`). Return `Option<_>` or document and test the behaviour.
3. `blocking:` there are no tests. Add tests for an empty list and for a normal list, and for a bad line.
4. `suggestion:` `&Vec<f64>` should be `&[f64]`, and `path: &str` could be `&Path`; Clippy reports the first one (`ptr_arg`) and the index loop (`needless_range_loop`). Use `prices.iter().sum::<f64>()`.
5. `suggestion:` prices are money; `f64` accumulates rounding errors. Store whole cents in an integer type.
6. `nit:` `text.lines().clone()` clones an iterator for no reason; loop over `text.lines()`.
7. `nit:` the public functions have no doc comments (`# Errors`).
8. `question:` is it intended that blank lines are an error? (`"".parse::<f64>()` fails.)

For comparison, a corrected `average` is `pub fn average(prices: &[u64]) -> Option<u64>` with `let count = u64::try_from(prices.len()).ok()?; prices.iter().sum::<u64>().checked_div(count)`; the tests
`assert_eq!(average(&[]), None)` and `assert_eq!(average(&[100, 201]), Some(150))` pass.

</details>

## Quiz

1. What defines a refactoring, and what is *not* a refactoring?

<details>
<summary>Answer</summary>

A change to the structure of code that keeps its observable behaviour. Adding a feature, fixing a bug or changing an output is not a refactoring.

</details>

2. What is a characterization test, and why do you record odd results?

<details>
<summary>Answer</summary>

A test that records what the code does now. Odd results are recorded because somebody may depend on them, and because the refactoring must not change behaviour; a suspected bug is fixed later in a separate change.

</details>

3. Why did replacing `total - total * 5 / 100` by `total * 95 / 100` fail the test?

<details>
<summary>Answer</summary>

Integer division rounds down. `638 - 31` is 607 but `638 * 95 / 100` is 606, so the two expressions are different for whole numbers. Only a test with real values shows it.

</details>

4. In what order does a reviewer look at a change?

<details>
<summary>Answer</summary>

Design first, then correctness, tests, readability, and style last (and only what the tools do not check).

</details>

5. What do the labels `blocking:` and `nit:` tell the author?

<details>
<summary>Answer</summary>

`blocking:` must be changed before the merge. `nit:` is a small matter of taste that the author may ignore.

</details>

## Summary

- **Refactoring** changes structure, not behaviour, in small steps with the tests green after each one. Wear one hat at a time: refactoring or new behaviour, in separate commits.
- No tests? Write **characterization tests** first: run the code, record what it does (including odd results), cover every branch and border.
- Even a "harmless" simplification can change integer rounding; the test catches it in seconds.
- Useful refactorings: rename, extract function, introduce constant, replace string/flag with `enum`, parameter object, early return, move to the owning type. Lean on the compiler and use the IDE.
- Do not refactor code that nobody changes, code without a safety net, or in the middle of an urgent fix.
- In a **review**, look at design, correctness and tests first; leave formatting and lints to the tools. Write comments about the code, give reasons, and label their weight (`blocking:`, `suggestion:`, `question:`, `nit:`).

## Sources and further reading

- [Refactoring — Martin Fowler's catalogue](https://refactoring.com/catalog/) — the names and steps of the classic refactorings.
- [Google Engineering Practices — How to do a code review](https://google.github.io/eng-practices/review/) — what reviewers look for, and how to write and handle review comments.
- [RustRover documentation — Refactoring source code](https://www.jetbrains.com/help/rust/refactoring-source-code.html) — the refactorings of the IDE, and how to run them.
- [The Rust Programming Language — Chapter 11: Writing Automated Tests](https://doc.rust-lang.org/book/ch11-00-testing.html) — the tools behind the safety net.
- [Clippy lint list](https://rust-lang.github.io/rust-clippy/stable/index.html) — lints that point at refactoring opportunities.

---

[🏠 Start](../START-HERE.md) · [← 08.12 Project layout and workspaces](12-project-layout-and-workspaces.md) · [📋 Phase 08](README.md) · [08.14 Layers and hexagonal architecture →](14-layers-and-hexagonal-architecture.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
