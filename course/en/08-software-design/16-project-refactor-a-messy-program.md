<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 08.15 The twelve-factor app](15-the-twelve-factor-app.md) · [📋 Phase 08](README.md) · [08.17 Checkpoint →](17-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/08-software-design/16-project-refactor-a-messy-program.md) <!-- nav -->

# Lesson 08.16 — Project: refactor a messy program

> **Phase 08 — Software design: principles, patterns and conventions** · Lesson 16 of 17 · ⏱ about 6 hours (3 sittings)

## What you will learn

- How to protect a program that has no tests with *golden tests* that run the real binary.
- How to change the structure of a 180-line program in small steps while its output stays byte-for-byte identical.
- How to apply the lessons of this phase together: separation of concerns, layers, types instead of strings, I/O behind traits, small functions and unit tests.
- How to find, record and consciously keep (for now) bugs that you discover during a refactoring.

## Before you start

- You finished [Lesson 08.15 — The twelve-factor app](15-the-twelve-factor-app.md) and, in particular, [Lesson 08.13 — Refactoring and code review](13-refactoring-and-code-review.md): you will use its loop (change, test, commit) all the time.
- You know how to test a command-line program from outside with `std::process::Command` and `env!("CARGO_BIN_EXE_<name>")` ([Lesson 05.13](../05-rust-intermediate/13-project-minigrep.md), milestone M5). Phase 09 explains this in depth.
- You know modules ([Lesson 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md)), newtypes and enums as types ([Lesson 08.5](05-type-driven-design.md)), the traits `Read`/`Write` ([Lesson 08.3](03-solid-principles.md), dependency inversion) and layers ([Lesson 08.14](14-layers-and-hexagonal-architecture.md)).
- Work on a branch: `git switch -c project/p08-refactor`.

## New words

| Word | Meaning |
|---|---|
| **golden test** | A test that runs the program on a fixed input and compares the whole output with a stored file (the "golden" file). Another name for a characterization test of a program that prints text. |
| **golden file** | The stored expected output. |
| **bug-compatible** | Keeping a known bug exactly as it is during a refactoring, so that the change of structure and the fix of the bug are two separate, reviewable steps. |
| **quirk** | A surprising but existing behaviour, such as a rounding rule or a formatting oddity. |
| **stdin / stdout / stderr** | Standard input, standard output and standard error: the three streams every program has. Input comes from stdin; normal output goes to stdout; error messages go to stderr. |
| **exit status** | The number a program returns to the shell when it ends: `0` means success. |
| **CSV** (Comma-Separated Values) | A text format with one record per line and fields separated by commas. |
| **I/O** (Input/Output) | Everything where a program talks to the outside world: reading files, the keyboard, printing. |

## Meet the messy program

A small shop has a program, `till`, that reads orders from standard input and prints a report. It works. Nobody dares to touch it. Your job is to make it clean **without changing what it prints**.

Create the package and add the second part of the `Cargo.toml` (the binary is called `till`):

```bash
cd ~/projects/rust
cargo new code/p08-refactor
cd code/p08-refactor
```

Append to `Cargo.toml`:

```toml
[[bin]]
name = "till"
path = "src/main.rs"
```

Put this code into `src/main.rs`. It is on purpose a bad program. Read it once, and list its problems before you read on.

```rust
use std::io::{self, BufRead};
use std::process;

fn main() {
    let args: Vec<String> = std::env::args().collect();
    let mode = if args.len() > 1 {
        args[1].clone()
    } else {
        String::from("receipt")
    };
    if mode != "receipt" && mode != "summary" && mode != "csv" {
        eprintln!("usage: till [receipt|summary|csv]");
        process::exit(2);
    }

    let stdin = io::stdin();
    let mut errors = 0;
    let mut orders = 0;
    let mut grand_total: u64 = 0;
    let mut cat_names: Vec<String> = Vec::new();
    let mut cat_totals: Vec<u64> = Vec::new();
    let mut cust_names: Vec<String> = Vec::new();
    let mut cust_totals: Vec<u64> = Vec::new();
    let mut line_no = 0;

    if mode == "csv" {
        println!("customer,item,qty,total_cents");
    }

    for line in stdin.lock().lines() {
        let line = line.unwrap();
        line_no += 1;
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        let parts: Vec<&str> = line.split(',').collect();
        if parts.len() != 5 {
            println!(
                "error: line {}: expected 5 fields, got {}",
                line_no,
                parts.len()
            );
            errors += 1;
            continue;
        }
        let name = parts[0].trim();
        let item = parts[1].trim();
        let qty: u64 = match parts[2].trim().parse() {
            Ok(q) => q,
            Err(_) => {
                println!(
                    "error: line {}: bad quantity '{}'",
                    line_no,
                    parts[2].trim()
                );
                errors += 1;
                continue;
            }
        };
        let price: u64 = match parts[3].trim().parse() {
            Ok(p) => p,
            Err(_) => {
                println!("error: line {}: bad price '{}'", line_no, parts[3].trim());
                errors += 1;
                continue;
            }
        };
        let category = parts[4].trim();

        let mut total = qty * price;
        if qty >= 10 {
            total = total - total * 10 / 100;
        }
        let tax_percent;
        if category == "food" {
            tax_percent = 7;
        } else if category == "drink" {
            tax_percent = 20;
        } else if category == "book" {
            tax_percent = 0;
        } else {
            println!(
                "warning: line {}: unknown category '{}', using 15%",
                line_no, category
            );
            tax_percent = 15;
        }
        total = total + total * tax_percent / 100;
        if name.starts_with("vip_") {
            total = total - total * 5 / 100;
        }

        orders += 1;
        grand_total += total;
        let mut found = false;
        for i in 0..cat_names.len() {
            if cat_names[i] == category {
                cat_totals[i] += total;
                found = true;
            }
        }
        if !found {
            cat_names.push(category.to_string());
            cat_totals.push(total);
        }
        let mut found = false;
        for i in 0..cust_names.len() {
            if cust_names[i] == name {
                cust_totals[i] += total;
                found = true;
            }
        }
        if !found {
            cust_names.push(name.to_string());
            cust_totals.push(total);
        }

        if mode == "receipt" {
            let dollars = total / 100;
            let cents = total % 100;
            if cents < 10 {
                println!(
                    "{:<8} {:<10} x{:<3} = {}.0{}",
                    name, item, qty, dollars, cents
                );
            } else {
                println!(
                    "{:<8} {:<10} x{:<3} = {}.{}",
                    name, item, qty, dollars, cents
                );
            }
        } else if mode == "csv" {
            println!("{},{},{},{}", name, item, qty, total);
        }
    }

    if mode == "receipt" {
        let dollars = grand_total / 100;
        let cents = grand_total % 100;
        if cents < 10 {
            println!("TOTAL: {}.0{} ({} orders)", dollars, cents, orders);
        } else {
            println!("TOTAL: {}.{} ({} orders)", dollars, cents, orders);
        }
    } else if mode == "summary" {
        println!("== summary ==");
        for i in 0..cat_names.len() {
            let dollars = cat_totals[i] / 100;
            let cents = cat_totals[i] % 100;
            if cents < 10 {
                println!("{:<8}: {}.0{}", cat_names[i], dollars, cents);
            } else {
                println!("{:<8}: {}.{}", cat_names[i], dollars, cents);
            }
        }
        if !cust_names.is_empty() {
            let mut best = 0;
            for i in 0..cust_names.len() {
                if cust_totals[i] > cust_totals[best] {
                    best = i;
                }
            }
            println!(
                "top customer: {} ({}.{})",
                cust_names[best],
                cust_totals[best] / 100,
                cust_totals[best] % 100
            );
        }
        println!("grand total: {}.{}", grand_total / 100, grand_total % 100);
        println!("orders: {}", orders);
    }

    if errors > 0 {
        eprintln!("{} error(s)", errors);
        process::exit(1);
    }
}
```

Compile it with `cargo build`. Rustc compiles it with no warnings; `cargo clippy` reports one (`unneeded late initialization`). It is the kind of code that works but cannot be tested or changed safely: one 170-line `main`, repeated formatting code, magic numbers (`10`, `7`, `20`, `15`, `5`), strings
for categories and modes, parallel `Vec`s that must be kept in step, and printing mixed into the calculations.

### What it does

The input has one order per line: `customer,item,quantity,unit_price_in_cents,category`. Blank lines and lines that start with `#` are ignored. Create two input files and an empty one:

```bash
mkdir -p tests/golden
cat > tests/golden/orders1.in <<'DATA'
# customer,item,qty,unit_price_cents,category
ana,coffee,2,350,drink
bo,sandwich,1,890,food
vip_cy,novel,3,1299,book
ana,water,12,100,drink
dan,gadget,1,2500,toys
DATA
cat > tests/golden/orders2.in <<'DATA'
ana,coffee,two,350,drink
bo,sandwich,1,890

cy,tea,1,250,drink
# a comment
cy,cake,1,300,food
dan,pen,1,-5,office
DATA
: > tests/golden/empty.in
```

Run the program in its three modes (`receipt` is the default):

```bash
cargo run -q -- receipt < tests/golden/orders1.in
cargo run -q -- summary < tests/golden/orders1.in
cargo run -q -- csv < tests/golden/orders1.in
```

```text
ana      coffee     x2   = 8.40
bo       sandwich   x1   = 9.52
vip_cy   novel      x3   = 37.03
ana      water      x12  = 12.96
warning: line 6: unknown category 'toys', using 15%
dan      gadget     x1   = 28.75
TOTAL: 96.66 (5 orders)
```

```text
warning: line 6: unknown category 'toys', using 15%
== summary ==
drink   : 21.36
food    : 9.52
book    : 37.03
toys    : 28.75
top customer: vip_cy (37.3)
grand total: 96.66
orders: 5
```

```text
customer,item,qty,total_cents
ana,coffee,2,840
bo,sandwich,1,952
vip_cy,novel,3,3703
ana,water,12,1296
warning: line 6: unknown category 'toys', using 15%
dan,gadget,1,2875
```

**What happens:**

1. Each order is priced: quantity times unit price; 10% off from ten pieces; tax by category (food 7%, drink 20%, book 0%, anything else 15% with a warning); and 5% off for customers whose name starts with `vip_`. All in whole cents, every division rounds down.
2. `receipt` prints a line per order and a total. `summary` prints totals per category in order of first appearance, the customer with the highest total, the grand total and the number of orders. `csv` prints raw cents.
3. Look closely at `top customer: vip_cy (37.3)`. The correct amount is `37.03`: the summary formats cents without a leading zero. That is a **bug**, and `grand total: 0.0` for an empty input has the same cause. In this project you will **not** fix it: first the structure changes, with identical output; fixing a bug is a separate step (Stretch goal 1).

Now the failure paths:

```bash
cargo run -q -- receipt < tests/golden/orders2.in; echo "exit status: $?"
cargo run -q -- bogus < tests/golden/orders1.in; echo "exit status: $?"
```

```text
error: line 1: bad quantity 'two'
error: line 2: expected 5 fields, got 4
cy       tea        x1   = 3.00
cy       cake       x1   = 3.21
error: line 7: bad price '-5'
TOTAL: 6.21 (2 orders)
3 error(s)
exit status: 1
usage: till [receipt|summary|csv]
exit status: 2
```

Notice where things go: the `error:` and `warning:` lines are printed on standard output, mixed with the report; only `3 error(s)` and the usage message go to standard error. The exit status is `1` when any line was bad and `2` for a wrong mode. All of this is *behaviour*, even if it is odd, and a refactoring must keep it.

## The project

### What you will build

You will not write new features. You will take the program above and, in five milestones, turn it into a small, layered, tested program, with a **golden test suite** that proves after every step that the output did not change by even one byte.

### Contract

**Behaviour (must stay identical, including the quirks):**

- The command line, input format, the three modes, the pricing rules, the report formats, the warnings and errors, what goes to standard output and standard error, and the exit statuses `0`, `1` and `2`, exactly as in "What it does".
- Until Stretch goal 1, the unpadded cents in `top customer` and `grand total` lines of the summary stay.

**Structure (what you build):**

- A library (`src/lib.rs`) with modules, and a thin `src/main.rs`.
- The pricing rules are in a module that does no printing and no input.
- Categories and modes are `enum`s; money is a type; there are no magic numbers left in the rules.
- The program logic is a function that takes the input and the two output streams as parameters (traits `BufRead` and `Write`), so it can be tested without starting a process.
- Unit tests for every module, plus the golden tests.
- `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test` all pass, and a short `README.md` describes the program.

### Step-by-step plan

Work in order. After each milestone: run its "Done when" commands, then commit. Never move on with a red test. If a step goes wrong, `git restore .` and try a smaller step.

---

#### M1 — Golden tests: pin down the behaviour

**Goal:** a test suite that runs the real `till` binary on fixed inputs and compares stdout, stderr and the exit status with stored files. Everything after this milestone relies on it.

**Guidance:**

1. Create `tests/golden.rs`. Cargo builds the binary first and gives the test its path in `env!("CARGO_BIN_EXE_till")` (Lesson 05.13). This helper runs the program on an input file and returns what it printed:

```rust
use std::fs;
use std::io::Write;
use std::path::PathBuf;
use std::process::{Command, Stdio};

fn golden_dir() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .join("tests")
        .join("golden")
}

/// Runs `till` with `args`, feeds it the file `input`, and returns (stdout, stderr, exit code).
fn run_till(args: &[&str], input: &str) -> (String, String, i32) {
    let text = fs::read(golden_dir().join(input)).unwrap();
    let mut child = Command::new(env!("CARGO_BIN_EXE_till"))
        .args(args)
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()
        .unwrap();
    // The program may exit before it reads its input (wrong mode), so ignore a broken pipe.
    let _ = child.stdin.take().unwrap().write_all(&text);
    let output = child.wait_with_output().unwrap();
    (
        String::from_utf8_lossy(&output.stdout).into_owned(),
        String::from_utf8_lossy(&output.stderr).into_owned(),
        output.status.code().unwrap(),
    )
}
```

2. Write the function `check(case, args, input)`. It calls `run_till`, then compares the three results with the files `tests/golden/<case>.stdout`, `.stderr` and `.code` (the exit status as text). It has a second mode: when the environment variable `UPDATE_GOLDEN` is set, it **writes** those files instead of comparing. That is how you create the golden files the first time. Use `std::env::var("UPDATE_GOLDEN").is_ok()`.
3. Write at least these nine tests, each a call of `check`:

| Test name | Arguments | Input file |
|---|---|---|
| `receipt_basic` | `receipt` | `orders1.in` |
| `default_mode_is_receipt` | *(none)* | `orders1.in` |
| `summary_basic` | `summary` | `orders1.in` |
| `csv_basic` | `csv` | `orders1.in` |
| `receipt_with_errors` | `receipt` | `orders2.in` |
| `summary_with_errors` | `summary` | `orders2.in` |
| `empty_receipt` | `receipt` | `empty.in` |
| `empty_summary` | `summary` | `empty.in` |
| `unknown_mode` | `bogus` | `orders1.in` |

4. Run `UPDATE_GOLDEN=1 cargo test` once to create the golden files, and **read them**. Compare them with the output printed above. A golden file you have never looked at proves nothing.
5. Add at least three cases of your own that cover rules the files above do not: the border between nine and ten pieces, two customers with the same total (who is the "top customer"?), a `vip_` customer with a bulk order.

<details>
<summary>Hint 1 — the helper `check`</summary>

Compute `base = golden_dir().join(case)`. Then `base.with_extension("stdout")` is the path of `<case>.stdout`. In the update mode use `fs::write`; in the compare mode use `fs::read_to_string` and `assert_eq!(actual, expected, "stdout of {case}")`. For the exit status, compare `code.to_string()` with the content of the `.code` file.

</details>

<details>
<summary>Hint 2 — a test that crashes with "Broken pipe"</summary>

For the wrong mode the program exits before it reads its input; writing to its closed input then fails. In `run_till` ignore the error of `write_all` (`let _ = ...`), as the helper above does.

</details>

**Done when:**

```bash
cargo test
```

shows all your golden tests passing (the names and the order of lines on your screen differ):

```text
test default_mode_is_receipt ... ok
test unknown_mode ... ok
test summary_with_errors ... ok
test summary_basic ... ok
test receipt_basic ... ok
test empty_receipt ... ok
test empty_summary ... ok
test csv_basic ... ok
test receipt_with_errors ... ok
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.25s
```

**And** the safety net really catches changes. In `src/main.rs` change `tax_percent = 20;` to `tax_percent = 21;` and run `cargo test`: several tests must fail (in the reference run, 6 of 9):

```text
test receipt_basic ... FAILED
test summary_basic ... FAILED
test csv_basic ... FAILED
...
test result: FAILED. 3 passed; 6 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.23s
```

Change it back, and `cargo test` is green again. A test that cannot fail is worthless; this "mutation check" proves yours can. Commit: `git add . && git commit -m "test(p08): golden tests for till"`.

---

#### M2 — Modules and layers

**Goal:** split the long `main` into a library with modules, so that each module has one job and the rules do not print.

**Guidance:**

1. Create `src/lib.rs` and move the code out of `main`. Suggested modules (Lessons 08.2 and 08.14):
   - `parse`: turns a line of text into an order, or an error;
   - `pricing`: given an order, computes the price (and the tax), with **no printing and no input**;
   - `report`: formats the receipt, the summary and the CSV text;
   - `app`: the loop that reads lines, calls the other modules and decides the exit status.
2. Move **one piece at a time** and run `cargo test` after each move. A good order: formatting of money (the repeated `if cents < 10` code) into one function; then `parse`; then `pricing`; then `report`; last the loop into `app`.
3. At this stage the types may still be simple (`String`, `u64`). The goal of M2 is *where code lives*, not yet *what types it uses*.
4. `src/main.rs` shrinks to: read the arguments, call the library, exit with the returned status.
5. The dependency rule of Lesson 08.14: `pricing` and `parse` must not import `std::io`.

<details>
<summary>Hint 1 — keep the messages</summary>

Messages such as `error: line 2: expected 5 fields, got 4` must stay character for character. Move the `format!` text unchanged, then improve it later.

</details>

<details>
<summary>Hint 2 — the order of the output lines</summary>

Warnings and errors are printed while the loop runs, mixed with the receipt lines. If your new design collects results first and prints later, the golden tests will show you that the order changed. Print in the same order.

</details>

**Done when:**

```bash
cargo test
wc -l src/main.rs
grep -n "std::io" src/pricing.rs src/parse.rs
```

`cargo test` still reports all golden tests passing (`9 passed`, or more if you added cases); `wc -l` prints a number below 30; and the `grep` prints nothing (no match). Commit: `git add . && git commit -m "refactor(p08): split till into modules"`.

---

#### M3 — Types instead of strings and numbers

**Goal:** make wrong states hard to write (Lesson 08.5) and remove magic numbers (Lesson 08.10).

**Guidance:**

1. Replace the mode string by an `enum Mode { Receipt, Summary, Csv }` with one function that converts the command-line word; unknown words give `None`, and `main` prints the usage message.
2. Replace the category string by an `enum Category { Food, Drink, Book, Other(String) }`. `Other` remembers the original word, because the warning and the summary print it.
3. Introduce a `struct Cents(u64)` for money, with `Display` that prints `dollars.cents` with two digits for the cents. **One** place formats money. The repeated `if cents < 10` code must disappear.
4. The summary's quirk (cents without a leading zero) is *not* the same format. Keep it, but make it visible: a separate, clearly named method with a comment that says it preserves the old behaviour until it is fixed on purpose.
5. Replace the parse error `println!`s by an `enum ParseError` that implements `Display` with exactly the old messages.
6. Give every number in the rules a name: bulk threshold, bulk discount, VIP (Very Important Person) prefix, VIP discount and one tax percent per category. A `match` on the category with no `_` arm says which tax applies.
7. Replace the parallel `Vec`s (`cat_names` and `cat_totals`, `cust_names` and `cust_totals`) by one `Vec<(String, Cents)>` per list or a small struct. Keep the *order of first appearance*, because the golden tests depend on it (a `HashMap` would print in a random order).

<details>
<summary>Hint 1 — where the words live</summary>

The words `"food"`, `"drink"`, `"book"` and `"receipt"`, `"summary"`, `"csv"` should each appear in **one** function that converts text to a type (and, for category, one that converts back). Everywhere else you use the `enum`.

</details>

<details>
<summary>Hint 2 — the rounding</summary>

A helper `fn percent(self, p: u64) -> Cents` that computes `self * p / 100` keeps the rounding rule in one place. Do not "simplify" `total - total * 5 / 100` to `total * 95 / 100`: the golden tests will catch it, as in Lesson 08.13.

</details>

**Done when:**

```bash
cargo test
cargo clippy --all-targets -- -D warnings
grep -rn '{}.0{}' src
grep -n '"food"\|"drink"\|"book"' src/*.rs
```

The tests pass; Clippy is silent (`Checking` and `Finished` only); the third command prints nothing; and the fourth shows matches in **one** source file only (the one that converts between text and `Category`). Commit: `git add . && git commit -m "refactor(p08): types for mode, category and money"`.

---

#### M4 — I/O behind traits

**Goal:** the program logic must not know about the real keyboard and screen, so that it can be tested with strings in memory (Lessons 08.3 and 08.14).

**Guidance:**

1. Turn the loop in `app` into a function with this shape (the names are yours):

```rust
pub fn run(
    mode: Mode,
    input: impl BufRead,
    out: &mut impl Write,
    err: &mut impl Write,
) -> io::Result<i32>
```

   It reads lines from `input`, writes the report and the warnings to `out`, writes `N error(s)` to `err` and returns the exit status. `BufRead` and `Write` are standard-library traits: a `&[u8]` is a `BufRead`; a `Vec<u8>` is a `Write` (Lesson 08.3).
2. The report functions take `out: &mut impl Write` and use `writeln!` instead of `println!`. `writeln!` returns an `io::Result`, so use `?`.
3. Only `main.rs` talks to the real terminal: it passes `stdin.lock()`, `io::stdout().lock()` and `io::stderr().lock()` to `run`, then calls `std::process::exit` with the returned status.
4. Add at least three tests that call `run` with a string as input and `Vec<u8>` as outputs, without starting a process: a receipt for one order, an input with a bad line (check the output, the error stream and the exit status), and the summary.

<details>
<summary>Hint 1 — a string as input</summary>

`"ana,coffee,2,350,drink\n".as_bytes()` is a `&[u8]`, which implements `BufRead`. After `run`, convert the buffers with `String::from_utf8(out).unwrap()`.

</details>

<details>
<summary>Hint 2 — `lines()` returns results</summary>

`input.lines()` yields `io::Result<String>`. Use `?` on the line, so that a read error becomes the error of `run`. In `main`, decide what to print when `run` itself fails.

</details>

**Done when:**

```bash
cargo test
grep -n "stdin\|stdout\|stderr\|println" src/*.rs
```

The tests pass, including your in-memory tests of `run`, and every match of the `grep` is in `src/main.rs`. Commit: `git add . && git commit -m "refactor(p08): run() takes BufRead and Write"`.

---

#### M5 — Unit tests and the quality gate

**Goal:** fast tests for every part, a clean gate, documentation.

**Guidance:**

1. Unit tests (`#[cfg(test)] mod tests` in each file): `Cents` formatting (`5` gives `0.05`, `3703` gives `37.03`, `0` gives `0.00`) and the quirk method (`3703` gives `37.3`); parsing (a valid line, a blank line, a comment, and one test for each error message); pricing (each tax rate, the border at ten pieces, the VIP discount and its rounding, the unknown category).
2. Each test checks one rule and has a name that states it (Lesson 05.6). Tests of the quirk are named so that the next reader knows it is on purpose.
3. Run the gate: `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test`.
4. Write `README.md` (Lesson 05.7): what `till` does, the input format, the modes, the exit statuses and how to run the tests. List "known quirks" and mention the unpadded cents.
5. Review your own diff as a reviewer (Lesson 08.13): any `pub` that could be private? A function longer than 30 lines? An `unwrap()` on input?

<details>
<summary>Hint 1 — how many tests?</summary>

A reasonable result is roughly ten to fifteen small unit tests next to the nine or more golden tests. The golden tests protect the whole program; the unit tests make failures easy to locate.

</details>

**Done when:**

```bash
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
```

prints no formatting diff, no Clippy warning, and two sets of passing tests: your unit tests (`test result: ok. 13 passed` in the reference run) and the golden tests (`ok. 9 passed`):

```text
test result: ok. 13 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.25s
```

And `git diff main -- tests/golden` shows **no changes to the golden files** you created in M1 (only added cases, if any). Commit: `git add . && git commit -m "test(p08): unit tests, README and clean quality gate"`. Finally bring the project into your main branch: `git switch main && git merge project/p08-refactor`.

---

### Stretch goals

1. **Fix the bug, on purpose.** Make the summary print `37.03` and `0.00`. This is a *behaviour* change: change the code, watch the golden tests `summary_basic`, `summary_with_errors` and `empty_summary` fail, check that the new text is what you want, regenerate those files with `UPDATE_GOLDEN=1`, and commit code and golden files together with the message `fix(p08): pad cents in the summary`. Delete the quirk method.
2. **Strategy.** Replace the `match mode` in `run` by a trait `Renderer` with three implementations (`ReceiptRenderer`, `SummaryRenderer`, `CsvRenderer`), as in Lesson 08.9. Is the code simpler, or just longer? Write down your answer.
3. **A workspace.** Move the library into the crate `till-core` and the thin program into `till-cli` (Lesson 08.12), with `[workspace.lints]`. The compiler now enforces that the core does not depend on the program.
4. **Property test.** With the `proptest` crate (Phase 09), check that the total of the receipt always equals the sum of the order totals.
5. **Arguments with clap.** Replace the manual argument handling with `clap` (Lesson 05.11), keeping the usage message and exit status `2` identical, or change them on purpose, with golden files updated in a separate commit.

## Common mistakes

1. **Refactoring before the golden tests exist.** You have nothing to tell you that the output is unchanged.
2. **Generating golden files and never reading them.** If the program had a bug in a case you did not look at, you have pinned the bug without knowing it.
3. **"Fixing" the cents formatting while moving it.** That changes the output. Keep it, test it, and fix it in its own commit (Stretch goal 1).
4. **Changing the order of output.** Warnings and errors are printed between the receipt lines. A design that collects first and prints later changes that.
5. **Using a `HashMap` for the totals.** Its iteration order is random, so the summary would differ from run to run. Keep the order of first appearance.
6. **Big steps.** One commit for "everything". When a golden test fails you then have no idea which change did it.
7. **Over-engineering.** Five traits and three crates for 180 lines is the opposite of the lesson. Each abstraction must have a reason you can say in one sentence.

## Quiz

1. Why do you write the golden tests *before* touching the code?

<details>
<summary>Answer</summary>

They are the safety net: they record the current behaviour, so that after every small step you know within seconds whether the output changed.

</details>

2. The summary prints `37.3` instead of `37.03`. Why do you keep it during the refactoring?

<details>
<summary>Answer</summary>

Because a refactoring must not change behaviour. If the structure change and the bug fix are mixed, a golden test failure could mean either, and the review is harder. Fix it afterwards in a separate commit that also updates the golden files.

</details>

3. Why does `run` take `impl BufRead` and `&mut impl Write` instead of using `println!` and the real standard input?

<details>
<summary>Answer</summary>

So that the logic can be tested with in-memory strings and buffers, without starting a process, and so that only `main` knows the real terminal. This is dependency inversion (Lesson 08.3) and a port of the core (Lesson 08.14).

</details>

4. Why does the pricing module not print anything?

<details>
<summary>Answer</summary>

A module with one job (calculating) is easy to test with plain values and easy to reuse; printing is a different concern (separation of concerns, Lesson 08.2).

</details>

5. Why is a `HashMap` a risky choice for the category totals here?

<details>
<summary>Answer</summary>

Its iteration order is not specified and can differ between runs, so the summary lines would appear in a different order and the golden tests would fail unpredictably. A `Vec` of pairs keeps the order of first appearance.

</details>

## Summary

- Before refactoring a program without tests, record its behaviour with **golden tests** that run the real binary and compare stdout, stderr and the exit status; check that they can fail.
- Refactor in small steps (modules, then types, then I/O traits, then unit tests) and keep every golden test green after every step; commit after each.
- Bugs and quirks that you find are **preserved first** and fixed later in a separate commit, with the golden files updated on purpose.
- The pricing rules do not print; the logic takes `BufRead` and `Write`; only `main` touches the real terminal. These are separation of concerns and dependency inversion in a few dozen lines.
- Replace strings and bare numbers with `enum`s, a money type and named constants; use a `match` without `_` arms so that the compiler guards you.
- Finish with the quality gate (`fmt`, `clippy -D warnings`, `test`), a README, and a review of your own diff.

## Sources and further reading

- [The Rust Programming Language — Chapter 11.3: Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — unit tests, integration tests and the `tests` folder.
- [Refactoring — Martin Fowler's catalogue](https://refactoring.com/catalog/) — the names of the moves you used.
- [The Rust Programming Language — Chapter 12: An I/O Project](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) — a library plus thin `main`, a similar structure.
- [Rust standard library — `BufRead`](https://doc.rust-lang.org/std/io/trait.BufRead.html) and [`Write`](https://doc.rust-lang.org/std/io/trait.Write.html) — the two traits behind M4.
- [The Cargo Book — Environment variables](https://doc.rust-lang.org/cargo/reference/environment-variables.html) — the environment variables Cargo sets for tests, including `CARGO_BIN_EXE_<name>`.

---

[🏠 Start](../START-HERE.md) · [← 08.15 The twelve-factor app](15-the-twelve-factor-app.md) · [📋 Phase 08](README.md) · [08.17 Checkpoint →](17-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
