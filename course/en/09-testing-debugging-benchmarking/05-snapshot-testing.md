<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.4 Property-based testing](04-property-based-testing.md) · [📋 Phase 09](README.md) · [09.6 Test-driven development in practice →](06-tdd-in-practice.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/05-snapshot-testing.md) <!-- nav -->

# Lesson 09.5 — Snapshot testing

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 5 of 14 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a snapshot test is, and when it is better than a long `assert_eq!`.
- How to use the `insta` crate and the `cargo insta` tool: create, review, accept and reject snapshots.
- How to snapshot text, `Debug` output and JSON, and how to hide values that change on every run (redactions).
- How snapshot tests behave in CI (Continuous Integration), and the habits that keep them trustworthy.

## Before you start

- You finished [Lesson 09.4 — Property-based testing](04-property-based-testing.md).
- You know `assert_eq!` and how to read a failing test ([Lesson 02.8](../02-programming-basics/08-your-first-tests.md), [Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)). Section 7 of Lesson 05.6 mentioned `insta` by name; now you use it.
- You know how to format text with `format!` and width options such as `{:<10}` and `{:>7}` ([Lesson 02.6](../02-programming-basics/06-printing-formatting-and-comments.md)), and what `#[derive(Debug)]` gives you.

## New words

| Word | Meaning |
|---|---|
| **snapshot** | A saved copy of the output of your code, kept in a file next to the code, that later runs are compared with. |
| **snapshot test** | A test that produces some output and compares it with the saved snapshot. A difference makes the test fail. |
| **`insta`** | A crate for snapshot testing in Rust (macros such as `assert_snapshot!`). |
| **`cargo-insta`** | A command-line tool, used as `cargo insta …`, to review and accept the snapshots that `insta` creates. |
| **review** | Looking at the difference between the old and the new snapshot and deciding whether the change is intended. |
| **accept / reject** | Accept: make the new output the saved snapshot. Reject: throw it away. |
| **inline snapshot** | A snapshot written inside the Rust source file, in the test itself, instead of a separate file. |
| **redaction** | Replacing a value that changes on every run (an id, a time) with a fixed placeholder before comparing. |
| **serialization** | Turning a value into text or bytes in a known format, for example JSON (JavaScript Object Notation, a text format for structured data). |
| **serde** | The crate that provides `#[derive(Serialize)]`, used to turn Rust values into JSON and other formats (Lesson 05.11). |
| **environment variable** | A named setting that the system gives to a program, such as `CI=true`. |

## 1. The problem: long expected values

Suppose a function draws an invoice as a table. To test it with `assert_eq!` you must write the whole expected text in
the test:

```rust
assert_eq!(
    render_invoice(&invoice),
    "Invoice #1042 for Ada Lovelace\n--------------------------------\nPens        2 x   1.50 =    3.00\n…"
);
```

This is awkward: a long line full of `\n`, you must count spaces by hand, and every small change of the format (a wider
column) forces you to retype all the expected texts in all the tests.

**Snapshot testing** solves this. The first time, you run the code and the library *saves its output in a file*. You look at the file once, and if it
looks right, you accept it. From then on the test compares the output with the saved file and fails on any difference. It is like
taking a photograph of the correct answer and checking against the photograph later.

The idea works best when the output is large, structured, easy to read for a human, and stable: rendered text, formatted reports, `Debug`
output of a data structure, JSON. It is not suited to a single number or a yes/no answer: there `assert_eq!` is shorter and
clearer.

## 2. Setup

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/09-05-snapshots
cargo new --lib code/p09-snapshots
cd code/p09-snapshots
```

Open `Cargo.toml` and replace its content:

```toml
[package]
name = "p09-snapshots"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }

[dev-dependencies]
insta = { version = "1", features = ["json", "redactions"] }
```

`serde` is needed by the code (it derives `Serialize`). `insta` is only for tests. Its `json` feature adds `assert_json_snapshot!`, and
`redactions` adds the hiding of values (Section 6).

Install the review tool (once; it is a separate program, like `cargo-nextest`):

```bash
cargo install --locked cargo-insta
cargo insta --version
```

```text
cargo-insta 1.49.0
```

If the tool is missing, `cargo insta` prints `error: no such command: `insta``.

## 3. The code under test

Replace `src/lib.rs` (the tests are added in a moment):

```rust
use serde::Serialize;

#[derive(Debug, Clone, Serialize)]
pub struct Item {
    pub name: String,
    pub quantity: u32,
    pub unit_cents: u64,
}

#[derive(Debug, Clone, Serialize)]
pub struct Invoice {
    pub number: u32,
    pub customer: String,
    pub items: Vec<Item>,
}

fn money(cents: u64) -> String {
    format!("{}.{:02}", cents / 100, cents % 100)
}

/// Draws the invoice as a text table, 32 characters wide.
pub fn render_invoice(invoice: &Invoice) -> String {
    let line = "-".repeat(32);
    let mut out = format!(
        "Invoice #{} for {}\n{line}\n",
        invoice.number, invoice.customer
    );
    let mut total = 0;
    for item in &invoice.items {
        let cost = u64::from(item.quantity) * item.unit_cents;
        total += cost;
        out.push_str(&format!(
            "{:<10} {:>2} x {:>6} = {:>7}\n",
            item.name,
            item.quantity,
            money(item.unit_cents),
            money(cost)
        ));
    }
    out.push_str(&format!("{line}\n{:<25}{:>7}\n", "Total", money(total)));
    out
}

/// A receipt for a paid invoice. The `id` is different every time it is created.
#[derive(Debug, Serialize)]
pub struct Receipt {
    pub id: u128,
    pub paid_cents: u64,
    pub invoice: Invoice,
}

pub fn make_receipt(invoice: Invoice) -> Receipt {
    let now = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .expect("the clock is before 1970")
        .as_nanos();
    let paid_cents = invoice
        .items
        .iter()
        .map(|item| u64::from(item.quantity) * item.unit_cents)
        .sum();
    Receipt {
        id: now,
        paid_cents,
        invoice,
    }
}
```

**What happens:**

1. `Invoice` and `Item` hold the data. `#[derive(Debug)]` lets us print them with `{:?}`, and `#[derive(Serialize)]` lets `serde` turn
   them into JSON.
2. `render_invoice` builds the table with width specifiers: `{:<10}` pads the name on the right to 10 characters, `{:>2}`
   pads the quantity on the left to 2, and so on. The last line pads `Total` to 25 characters and the sum to 7.
3. `make_receipt` makes a `Receipt` with an `id` that is the current time in nanoseconds. It is different every run, and this will
   matter in Section 6.

## 4. The first snapshot test

Add a test module to the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn sample_invoice() -> Invoice {
        Invoice {
            number: 1042,
            customer: "Ada Lovelace".to_string(),
            items: vec![
                Item {
                    name: "Pens".to_string(),
                    quantity: 2,
                    unit_cents: 150,
                },
                Item {
                    name: "Notebook".to_string(),
                    quantity: 1,
                    unit_cents: 420,
                },
            ],
        }
    }

    #[test]
    fn invoice_text() {
        insta::assert_snapshot!(render_invoice(&sample_invoice()));
    }
}
```

Run `cargo test`. The test **fails**, because there is no saved snapshot yet:

```text
running 1 test
stored new snapshot …/src/snapshots/p09_snapshots__tests__invoice_text.snap.new
test tests::invoice_text ... FAILED
failures:
---- tests::invoice_text stdout ----
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ Snapshot Summary ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Snapshot file: src/snapshots/p09_snapshots__tests__invoice_text.snap
Snapshot: invoice_text
Source: src/lib.rs:94
────────────────────────────────────────────────────────────────────────────────
Expression: render_invoice(&sample_invoice())
────────────────────────────────────────────────────────────────────────────────
+new results
────────────┬───────────────────────────────────────────────────────────────────
          1 │+Invoice #1042 for Ada Lovelace
          2 │+--------------------------------
          3 │+Pens        2 x   1.50 =    3.00
          4 │+Notebook    1 x   4.20 =    4.20
          5 │+--------------------------------
          6 │+Total                       7.20
────────────┴───────────────────────────────────────────────────────────────────
To update snapshots run `cargo insta review` or set `INSTA_UPDATE=always`
```

**What happens:**

1. `insta::assert_snapshot!(value)` converts the value to text and looks for the file named after the test:
   `src/snapshots/<crate>__<module>__<test name>.snap`. There is none.
2. `insta` writes the output into a new file `….snap.new` ("a candidate") and fails the test. A new snapshot must be *approved by a human*
   before it counts; otherwise a test could pass just because it had never been checked.
3. The report shows the new output with a `+` in front of every line. This is your moment to **read it**: is this the invoice you want? The
   columns line up, the total is 7.20 (2 × 1.50 + 4.20). It is right.

Accept it. The simplest way is the review tool (`cargo insta review` shows each change and asks `a`ccept, `r`eject or `s`kip; for all at
once there are `cargo insta accept` and `cargo insta reject`):

```bash
cargo insta accept
```

```text
insta review finished
accepted:
  src/lib.rs (invoice_text.snap)
```

Now look at the file that was created:

```bash
cat src/snapshots/p09_snapshots__tests__invoice_text.snap
```

```text
---
source: src/lib.rs
expression: render_invoice(&sample_invoice())
---
Invoice #1042 for Ada Lovelace
--------------------------------
Pens        2 x   1.50 =    3.00
Notebook    1 x   4.20 =    4.20
--------------------------------
Total                       7.20
```

The part between the `---` lines is metadata (which file and expression made it); below it is the snapshot itself. **This file belongs in Git.**
It is part of the specification of your program. Run `cargo test` again: the test passes.

## 5. A change in the output

A week later someone changes the code: `"Total"` becomes `"TOTAL"` in `render_invoice`. Run `cargo test`:

```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ Snapshot Summary ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Snapshot file: src/snapshots/p09_snapshots__tests__invoice_text.snap
Snapshot: invoice_text
Source: src/lib.rs:94
────────────────────────────────────────────────────────────────────────────────
Expression: render_invoice(&sample_invoice())
────────────────────────────────────────────────────────────────────────────────
-old snapshot
+new results
────────────┬───────────────────────────────────────────────────────────────────
    2     2 │ --------------------------------
    3     3 │ Pens        2 x   1.50 =    3.00
    4     4 │ Notebook    1 x   4.20 =    4.20
    5     5 │ --------------------------------
    6       │-Total                       7.20
          6 │+TOTAL                       7.20
────────────┴───────────────────────────────────────────────────────────────────
To update snapshots run `cargo insta review` or set `INSTA_UPDATE=always`
```

**What it means:** the report is a **diff**. The two columns of numbers are the line numbers in the old and in the new text; `-` marks
the line from the old snapshot and `+` the line from the new output. Only one line changed, and you see it at once, without
counting characters.

Now you decide:

- **The change is intended** (we really wanted capital letters): run `cargo insta accept`. The `.snap` file changes, and the
  change appears in your Git diff, where a reviewer can see it.
- **The change is a bug**: fix the code, and the test passes again.

This is the **workflow**; the commands are:

| Command | What it does |
|---|---|
| `cargo test` | runs the tests; new or changed snapshots are written to `.snap.new` files and the tests fail |
| `cargo insta test` | runs the tests and collects all pending snapshots; with `--review` it opens the review right after, with `--accept` it accepts everything |
| `cargo insta review` | goes through the pending snapshots one at a time: accept, reject or skip |
| `cargo insta accept` / `reject` | accepts / rejects all pending snapshots |
| `cargo insta pending-snapshots` | lists the snapshots that wait for a decision |

> ⚠️ **Common mistake:** Accepting without reading. `cargo insta accept` is fast, so it is tempting to press it every time. But a
> snapshot you did not read protects nothing: it will happily "approve" a bug. **Read every diff** the way you read a code review.
> The Git diff of the `.snap` files is the second chance to notice a mistake.

> 📏 **Convention:** Commit the `.snap` files. Do not commit the `.snap.new` files and the `.pending-snap` files: they are
> temporary. Add them to `.gitignore`:
> ```text
> *.snap.new
> .*.pending-snap
> ```

## 6. Other kinds of snapshots

### `Debug` output

`assert_debug_snapshot!` saves the `{:#?}` (pretty `Debug`) form of any value that implements `Debug`. It is ideal for a data structure when you want to see all
its fields. Add to `mod tests`:

```rust
    #[test]
    fn invoice_debug_view() {
        insta::assert_debug_snapshot!(sample_invoice());
    }
```

After `cargo insta test --accept` the file `…invoice_debug_view.snap` contains:

```text
---
source: src/lib.rs
expression: sample_invoice()
---
Invoice {
    number: 1042,
    customer: "Ada Lovelace",
    items: [
        Item {
            name: "Pens",
            quantity: 2,
            unit_cents: 150,
        },
        Item {
            name: "Notebook",
            quantity: 1,
            unit_cents: 420,
        },
    ],
}
```

### JSON and redactions

`assert_json_snapshot!` saves the value as JSON (it needs `Serialize` and the `json` feature). But our `Receipt` has an `id` that is the current time. Without
any help, the test would fail on every run:

```rust
    #[test]
    fn receipt_as_json() {
        let receipt = make_receipt(sample_invoice());
        insta::assert_json_snapshot!(receipt);
    }
```

After you accept the first run, the second run shows a diff like this:

```text
    1     1 │ {
    2       │-  "id": 1794984885511749000,
          2 │+  "id": 1794984893895514000,
    3     3 │   "paid_cents": 720,
```

The `id` differs every time. A **redaction** replaces a value at a path with a fixed text *before* the comparison. Write it as the second argument:

```rust
    #[test]
    fn receipt_as_json() {
        let receipt = make_receipt(sample_invoice());
        insta::assert_json_snapshot!(receipt, { ".id" => "[id]" });
    }
```

After accepting, the saved snapshot contains `"id": "[id]"`, and the test is stable:

```text
{
  "id": "[id]",
  "paid_cents": 720,
  "invoice": {
    "number": 1042,
    "customer": "Ada Lovelace",
    "items": [
      {
        "name": "Pens",
        "quantity": 2,
        "unit_cents": 150
      },
      {
        "name": "Notebook",
        "quantity": 1,
        "unit_cents": 420
      }
    ]
  }
}
```

**What happens:** the path `".id"` selects the field `id` of the top-level value. The selector syntax has `.field`, `[0]` for an item of a list,
`[]` for *every* item of a list (as in `".items[].unit_cents"`), and `.*` for every field. Redactions are the standard answer to ids, times,
random values and anything else that you do not control. (Better still, when you can, is to pass a fixed clock into the code, as in
Lesson 09.3; use a redaction when you cannot.)

If you forget the `json` feature, the macro does not exist:

```text
error[E0433]: cannot find `assert_json_snapshot` in `insta`
   --> src/lib.rs:105:16
    |
105 |         insta::assert_json_snapshot!(receipt, { ".id" => "[id]" });
    |                ^^^^^^^^^^^^^^^^^^^^ could not find `assert_json_snapshot` in `insta`
```

### Inline snapshots

A snapshot can live in the test itself. Write `@""` as the second argument, run `cargo insta test --accept`, and `cargo-insta` **edits your
source file** and fills the text in:

```rust
    #[test]
    fn an_empty_invoice_inline() {
        let invoice = Invoice {
            number: 1,
            customer: "Nobody".to_string(),
            items: vec![],
        };
        insta::assert_snapshot!(render_invoice(&invoice), @"
        Invoice #1 for Nobody
        --------------------------------
        --------------------------------
        Total                       0.00
        ");
    }
```

Inline snapshots are good for short outputs, because the expected value is right next to the test. Long outputs belong in files.

## 7. Snapshots in CI and in a team

On your computer, a new snapshot is written to a `.snap.new` file, so that you can review it. On a CI server nobody can press `accept`, and writing
files would only hide the problem. `insta` knows this: when the environment variable `CI` is set (GitHub Actions, GitLab CI and most others
set it), it writes **nothing** and the test simply fails. You can control the behaviour with `INSTA_UPDATE`:

| `INSTA_UPDATE` | Behaviour |
|---|---|
| `auto` (default) | `no` when `CI` is set, `new` otherwise |
| `new` | write `.snap.new` files, do not touch the accepted snapshots |
| `no` | write nothing; the test just fails |
| `always` | overwrite the saved snapshots at once (use only when you will read the Git diff afterwards) |

Try it: change `"Total"` to `"TOTAL"` again and run `CI=true cargo test`. The test fails and `src/snapshots/` contains no `.snap.new`.
Run `INSTA_UPDATE=always cargo test`, and the snapshot file is rewritten, so that you can see the change in `git diff`.

Habits that keep a snapshot suite healthy:

- **Review the `.snap` diff in every pull request**, like code. A reviewer who sees `+TOTAL` can ask "was this intended?".
- **Keep snapshots small and stable.** A 2,000-line snapshot cannot be read, so changes slip through. Snapshot the part that matters.
- **Remove noise before you save:** ids, times, random values, absolute paths (redactions, or pass fixed values in).
- **Use ordinary assertions for single facts.** `assert_eq!(total, 720)` is clearer than a snapshot of one number.
- **Name tests well.** The test name is part of the file name. Renaming a test creates a new snapshot and leaves the old file behind.

> 🔒 **Security:** A snapshot is a file in your repository, and everyone can read it. Do not snapshot responses that contain real tokens,
> passwords or personal data, or use redactions for those fields.

## Common mistakes

**1. Accepting everything.** See the warning in Section 5.

**2. Output that changes between runs.** Times, random ids, absolute paths, the order of items in a `HashMap`. The test fails randomly. Use
redactions, a fixed clock, or sort the data before the snapshot.

**3. Forgetting the `.snap` files in Git.** On your computer the test passes; in CI it fails with "no snapshot", because the file was never
committed. Run `git status` and look for `src/snapshots/`.

**4. Snapshotting the wrong thing.** Snapshotting `Debug` output of a type whose fields change often makes a noisy test that fails after every harmless
change. Snapshot the *result that matters to the user*.

**5. Missing a feature.** `assert_json_snapshot!` needs the `json` feature, and redactions need `redactions` (Section 6). The compiler
error `cannot find assert_json_snapshot in insta` means the feature is not in `Cargo.toml`.

## Try it yourself

1. Create the package, `Cargo.toml`, `src/lib.rs` and the first test. Install `cargo-insta`. Run `cargo test`, read the report, run `cargo insta accept`.
2. Change `"Total"` to `"TOTAL"`, run `cargo test` and read the diff. Run `cargo insta reject`, then put `"Total"` back.
3. Add `invoice_debug_view`, `receipt_as_json` (first without the redaction, twice, to see it fail, then with it) and `an_empty_invoice_inline`
   (with `@""`). Accept with `cargo insta test --accept`.
4. Run `CI=true cargo test` after a change, and check that nothing is written.
5. Add `.gitignore` lines for `*.snap.new` and `.*.pending-snap`. Run `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`,
   then commit, with the snapshots: `git add code/p09-snapshots && git commit -m "Add p09-snapshots: snapshot tests with insta"`.

## Exercises

### Exercise 1 — Read before you accept (easy)

Add a test `a_long_item_name` with this invoice: number 7, customer `"Grace Hopper"`, one item `"Coffee beans"` (12 letters), quantity 3, unit price
1299. Accept the snapshot only after reading it. What is wrong with the picture?

<details>
<summary>Hint</summary>

Use `insta::assert_snapshot!(render_invoice(&invoice))`. Compare the length of the item line with the dashed line.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn a_long_item_name() {
        let invoice = Invoice {
            number: 7,
            customer: "Grace Hopper".to_string(),
            items: vec![Item {
                name: "Coffee beans".to_string(),
                quantity: 3,
                unit_cents: 1299,
            }],
        };
        insta::assert_snapshot!(render_invoice(&invoice));
    }
```

The new snapshot is:

```text
Invoice #7 for Grace Hopper
--------------------------------
Coffee beans  3 x  12.99 =   38.97
--------------------------------
Total                      38.97
```

The item line is two characters longer than the line of dashes: the name is wider than the 10 characters of its column and pushes
the rest to the right. A hand-written `assert_eq!` would probably have been written from the *code's* output and hidden
this; reading a snapshot makes it visible. (Do you accept it? Not yet: the next exercise fixes the code.)

</details>

### Exercise 2 — Fix the code, review the diff (medium)

Fix `render_invoice` so that a name longer than 10 characters is cut to 10. Run the tests, read the diff of
`a_long_item_name`, and accept it only if it shows exactly what you intended.

<details>
<summary>Hint</summary>

`item.name.chars().take(10).collect::<String>()` gives the first 10 characters. Use that value in the `format!` instead of `item.name`.

</details>

<details>
<summary>Answer</summary>

In the loop of `render_invoice`:

```rust
        let name: String = item.name.chars().take(10).collect();
        out.push_str(&format!(
            "{:<10} {:>2} x {:>6} = {:>7}\n",
            name,
            item.quantity,
            money(item.unit_cents),
            money(cost)
        ));
```

The diff shows exactly one changed line, and the other tests do not change:

```text
    3       │-Coffee beans  3 x  12.99 =   38.97
          3 │+Coffee bea  3 x  12.99 =   38.97
```

Why it works: the line now has the same width as the dashes. (`chars().take(10)` counts characters and not bytes, so the cut never splits a letter
in the middle.) Accept the change with `cargo insta accept`, and the new snapshot goes into Git together with the fix.

</details>

### Exercise 3 — Hide the prices (hard)

Write a test `prices_hidden_with_a_redaction` that saves `sample_invoice()` as JSON, but with every `unit_cents` replaced by `"[price]"`.

<details>
<summary>Hint</summary>

The selector for a field inside every element of the list `items` is `".items[].unit_cents"`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn prices_hidden_with_a_redaction() {
        insta::assert_json_snapshot!(sample_invoice(), {
            ".items[].unit_cents" => "[price]",
        });
    }
```

The saved snapshot contains `"unit_cents": "[price]"` in both items:

```text
{
  "number": 1042,
  "customer": "Ada Lovelace",
  "items": [
    {
      "name": "Pens",
      "quantity": 2,
      "unit_cents": "[price]"
    },
    {
      "name": "Notebook",
      "quantity": 1,
      "unit_cents": "[price]"
    }
  ]
}
```

Why it works: `[]` in the selector means "every element of the list", so one redaction covers all items.

</details>

## Quiz

1. What happens the first time a snapshot test runs and no snapshot file exists?

<details>
<summary>Answer</summary>

`insta` writes the output to a `.snap.new` file and the test fails. A human must review and accept the snapshot before it counts.

</details>

2. What is the difference between `cargo insta accept` and `cargo insta reject`?

<details>
<summary>Answer</summary>

`accept` makes the new output the saved snapshot (`.snap` file); `reject` deletes the pending `.snap.new` files and keeps the old snapshot.

</details>

3. Why does a test with `assert_json_snapshot!(receipt)` fail on every run, and how do you fix it?

<details>
<summary>Answer</summary>

The `id` field contains the current time, so the output differs each time. Use a redaction, for example `{ ".id" => "[id]" }`, or make the code take a fixed clock.

</details>

4. What does `insta` do on a CI server, and why?

<details>
<summary>Answer</summary>

When `CI` is set, it writes no `.snap.new` files and just fails the test, because nobody can review and accept a snapshot there.

</details>

5. Which files should be committed to Git, and which not?

<details>
<summary>Answer</summary>

Commit the `.snap` files (they are part of the specification). Do not commit `.snap.new` and `.pending-snap` files: they are temporary.

</details>

6. When is `assert_eq!` better than a snapshot?

<details>
<summary>Answer</summary>

For a single, small fact (a number, a yes/no answer, a short string). A snapshot is for large, structured, stable output that is easy to read as a whole.

</details>

## Summary

- A snapshot test compares the output of your code with a saved file. A difference fails the test; a human reviews and accepts intended changes.
- With `insta`: `assert_snapshot!` (text), `assert_debug_snapshot!` (`Debug`), `assert_json_snapshot!` (JSON, feature `json`), inline snapshots with `@""`.
- The workflow: `cargo test` (or `cargo insta test`) → read the diff → `cargo insta accept` or fix the code. Always read the diff.
- Redactions (`{ ".id" => "[id]" }`) hide values that change on every run; a fixed clock is better when you can control it.
- Commit `.snap` files; ignore `*.snap.new` and `.*.pending-snap`. On CI (`CI` is set) `insta` writes nothing and fails; `INSTA_UPDATE` changes this.
- Use snapshots for large, readable, stable output; use `assert_eq!` for single facts.

## Sources and further reading

- [insta documentation (insta.rs)](https://insta.rs/docs/) — the official guide: snapshot kinds, settings and workflow.
- [insta on docs.rs](https://docs.rs/insta/1/insta/) — the API reference, including the macros and the redaction syntax.
- [insta — the `cargo insta` command](https://insta.rs/docs/cli/) — all the subcommands of the review tool.
- [serde.rs](https://serde.rs/) — the documentation of `serde`, which makes JSON snapshots possible.

---

[🏠 Start](../START-HERE.md) · [← 09.4 Property-based testing](04-property-based-testing.md) · [📋 Phase 09](README.md) · [09.6 Test-driven development in practice →](06-tdd-in-practice.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
