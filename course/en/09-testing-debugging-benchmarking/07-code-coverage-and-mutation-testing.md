<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.6 Test-driven development in practice](06-tdd-in-practice.md) · [📋 Phase 09](README.md) · [09.8 Fuzzing →](08-fuzzing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md) <!-- nav -->

# Lesson 09.7 — Code coverage and mutation testing

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 7 of 14 · ⏱ about 120 minutes

## What you will learn

- What code coverage is, how to measure it with `cargo llvm-cov`, and how to read the report.
- Why 100 % coverage does not mean that the tests are good.
- What mutation testing is, and how `cargo mutants` shows you tests that check nothing.
- How to turn a "missed mutant" into a better test, and how to use both tools without chasing numbers.

## Before you start

- You finished [Lesson 09.6 — Test-driven development in practice](06-tdd-in-practice.md).
- You know unit tests and `cargo test` ([Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)). Section 7 of that lesson mentioned coverage and mutation testing by name; this lesson uses them.
- The tools are installed once with `cargo install` (see Section 2). You also need the Rust component `llvm-tools-preview`; Section 2 adds it with `rustup`.

## New words

| Word | Meaning |
|---|---|
| **code coverage** | A measurement of which parts of your code were executed while the tests ran. |
| **line coverage** | The share of source lines that were executed at least once, in percent. |
| **region** | A piece of code that the compiler counts separately for coverage: for example one branch of an `if`. Region coverage is stricter than line coverage. |
| **instrumentation** | Extra counting code that the compiler adds to a program so that it records what was executed. |
| **LLVM** (the name is historical) | The compiler back end that Rust uses to produce machine code; it also provides the coverage counters. |
| **LCOV** (the name of the `lcov` coverage tool) | A plain-text file format for coverage data (usually a file `lcov.info`). It is named after the `lcov` tool, and many editors and coverage services can read it. |
| **`cargo-llvm-cov`** | A tool, used as `cargo llvm-cov`, that runs your tests with instrumentation and prints a coverage report. |
| **mutation testing** | Changing your code on purpose in small ways (a *mutation*) and checking that at least one test fails. |
| **mutant** | One changed copy of your program, for example with `+` replaced by `-`. |
| **caught / killed mutant** | A mutant that makes a test fail. Good: your tests noticed the change. |
| **missed / surviving mutant** | A mutant for which all tests still pass. Bad: your tests do not notice this change. |
| **equivalent mutant** | A mutant that changes the code but not its behaviour, so no test can catch it. |
| **`cargo-mutants`** | A tool, used as `cargo mutants`, that creates mutants and runs your tests on each. |
| **test timeout** | A time limit for a test run; a mutant that makes the program loop for ever is stopped by it. |
| **boundary value** | A value exactly on the edge between two cases, for example weight 500 when the limit is "up to 500". |

## 1. Two questions about tests

You have written tests. Two questions are still open:

1. **Which code do my tests run?** Code that no test runs is certainly untested. *Coverage* answers this.
2. **If I broke the code, would a test notice?** A test can run a line and still check nothing about it. *Mutation testing* answers this.

The first question has an easy, fast answer, but it is easy to misread. The second is slower, but it measures what you really want: the power of the tests to find bugs.

## 2. Setup

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/09-07-coverage
cargo new --lib code/p09-coverage
cd code/p09-coverage
rustup component add llvm-tools-preview
cargo install --locked cargo-llvm-cov cargo-mutants
```

`rustup component add` installs the LLVM tools that read the coverage counters (nothing happens if they are already installed). The last command
compiles the two tools; it takes a few minutes the first time. Check them:

```bash
cargo llvm-cov --version
cargo mutants --version
```

```text
cargo-llvm-cov 0.9.1
cargo-mutants 27.1.0
```

Both tools put their results in folders inside the project. Create a `.gitignore` file in the project folder with these two lines. (Inside your learning repository `cargo new` does not create one; the `/target` line repeats the `target/` rule of the root `.gitignore` from Lesson 05.1, which is harmless.)

```text
/target
/mutants.out*
```

## 3. The code under test, and a weak test

Replace `src/lib.rs`:

```rust
/// Price of sending a parcel, in cents. Express delivery costs twice as much.
pub fn shipping_cost(weight_grams: u32, express: bool) -> u32 {
    let base = if weight_grams <= 500 {
        500
    } else if weight_grams <= 2_000 {
        900
    } else {
        1_500
    };
    if express { base * 2 } else { base }
}

/// Orders of 5,000 cents or more ship for free.
pub fn shipping_for_order(order_cents: u32, weight_grams: u32, express: bool) -> u32 {
    if order_cents >= 5_000 {
        0
    } else {
        shipping_cost(weight_grams, express)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_runs() {
        shipping_cost(100, false);
        shipping_cost(1_000, true);
        shipping_cost(3_000, false);
        shipping_for_order(6_000, 100, false);
        shipping_for_order(100, 100, false);
    }
}
```

**What happens:** `shipping_cost` has three weight classes (up to 500 g costs 500 cents, up to 2,000 g costs 900, more costs 1,500) and doubles the price for express.
`shipping_for_order` makes big orders free. The test `it_runs` calls the functions with inputs for every branch, but it **never looks at a result**: there is not a single
`assert`. A tired programmer, or a program that wants to "reach 100 %", writes tests like this. Run `cargo test`: one test passes.

## 4. Measuring coverage

```bash
cargo llvm-cov
```

The command compiles your tests with instrumentation, runs them, and prints the usual test output followed by a table. The real table is wide; here are the
columns that matter (the *Branches* columns, which show `-` on stable Rust, are left out):

```text
Filename       Regions  Missed Regions  Cover     Functions  Missed Functions  Executed   Lines  Missed Lines  Cover
…/src/lib.rs        30               0  100.00%           3                 0  100.00%      20             0  100.00%
TOTAL               30               0  100.00%           3                 0  100.00%      20             0  100.00%
```

**What it means:**

- **Lines**: 20 lines can be executed and all 20 were. Line coverage is 100 %.
- **Functions**: 3 functions (`shipping_cost`, `shipping_for_order` and the test itself), all called.
- **Regions**: 30 regions (pieces such as each branch of an `if`, and each side of `&&`), none missed.

Everything is executed, and still the tests prove nothing: you could replace the body of `shipping_cost` with `0` and `cargo test` would stay green. **Coverage tells you what ran, not what was
checked.**

Other forms of the report:

| Command | What it gives you |
|---|---|
| `cargo llvm-cov --text` | the source code with an execution count in front of every line (best for finding holes) |
| `cargo llvm-cov --html` | a folder `target/llvm-cov/html/` with a browsable report; add `--open` to open it in the browser |
| `cargo llvm-cov --lcov --output-path lcov.info` | a file in the LCOV format, which many tools and services read |
| `cargo llvm-cov --summary-only` | only the table, with no source |
| `cargo llvm-cov --fail-under-lines 90` | exit with an error code if line coverage is below 90 % (for CI, continuous integration: the server that checks every change, Lesson 09.12) |
| `cargo llvm-cov --ignore-filename-regex 'tests/'` | leave out files whose path matches the pattern |

To see how the `--text` view helps, replace `it_runs` with two small real tests (we need them in the next sections anyway):

```rust
    #[test]
    fn a_light_parcel_costs_500() {
        assert_eq!(shipping_cost(100, false), 500);
    }

    #[test]
    fn a_big_order_ships_free() {
        assert_eq!(shipping_for_order(6_000, 100, false), 0);
    }
```

```bash
cargo llvm-cov --text
```

```text
    1|       |/// Price of sending a parcel, in cents. Express delivery costs twice as much.
    2|      1|pub fn shipping_cost(weight_grams: u32, express: bool) -> u32 {
    3|      1|    let base = if weight_grams <= 500 {
    4|      1|        500
    5|      0|    } else if weight_grams <= 2_000 {
    6|      0|        900
    7|       |    } else {
    8|      0|        1_500
    9|       |    };
   10|      1|    if express { base * 2 } else { base }
                               ^0
   11|      1|}
   12|       |
   13|       |/// Orders of 5,000 cents or more ship for free.
   14|      1|pub fn shipping_for_order(order_cents: u32, weight_grams: u32, express: bool) -> u32 {
   15|      1|    if order_cents >= 5_000 {
   16|      1|        0
   17|       |    } else {
   18|      0|        shipping_cost(weight_grams, express)
   19|       |    }
```

**What happens:** the number before `|` is the line number; the second number says how many times the line ran. A `0` marks code that no test executed: the medium and heavy
classes (lines 5, 6, 8) and the call in line 18. The `^0` under line 10 is a *region* that never ran: the express branch `base * 2`. The summary line now says
79.31 % of regions and 78.95 % of lines. The report shows exactly which tests are missing.

Notice what happened: after we replaced one useless test with two real tests, **coverage went down** (100 % to 79 %), while the quality of the tests went up. This is why the percentage
alone is a poor measure.

> 💡 **Tip:** Use coverage to find *holes*, not to reach a number. Open the `--html` report after a feature and look at the red lines: each one is a question, "is it OK that nothing
> tests this?". Sometimes the answer is yes (code that prints a usage message), sometimes it is a missing test.

## 5. Mutation testing

Mutation testing measures the power of the tests directly. The tool changes your program in small, mechanical ways, one change at a time, and runs the tests on every changed copy
(a **mutant**). If the tests still pass, the change went unnoticed: a **missed** mutant, which points at a missing or weak test. The typical mutations are: replace a function body with a constant
(`0`, `1`, `true`, `false`, an empty `Vec`), and swap an operator (`+` with `-` or `*`, `<` with `>`, `==` with `!=`).

Put the weak `it_runs` test back (the code from Section 3) and look at what the tool would do:

```bash
cargo mutants --list
```

```text
src/lib.rs:3:5: replace shipping_cost -> u32 with 0
src/lib.rs:3:5: replace shipping_cost -> u32 with 1
src/lib.rs:3:32: replace <= with > in shipping_cost
src/lib.rs:5:28: replace <= with > in shipping_cost
src/lib.rs:10:23: replace * with + in shipping_cost
src/lib.rs:10:23: replace * with / in shipping_cost
src/lib.rs:15:5: replace shipping_for_order -> u32 with 0
src/lib.rs:15:5: replace shipping_for_order -> u32 with 1
src/lib.rs:15:20: replace >= with < in shipping_for_order
```

Each line is one mutant: *file:line:column: what is replaced*. Now run them. `cargo mutants` copies the project to a temporary folder, applies each mutation, builds, and runs `cargo test`:

```bash
cargo mutants
```

```text
Found 9 mutants to test
ok       Unmutated baseline in 0s build + 0s test
 INFO Auto-set test timeout to 20s
MISSED   src/lib.rs:3:5: replace shipping_cost -> u32 with 0 in 0s build + 0s test
MISSED   src/lib.rs:3:5: replace shipping_cost -> u32 with 1 in 0s build + 0s test
MISSED   src/lib.rs:3:32: replace <= with > in shipping_cost in 0s build + 0s test
MISSED   src/lib.rs:5:28: replace <= with > in shipping_cost in 0s build + 0s test
MISSED   src/lib.rs:10:23: replace * with + in shipping_cost in 0s build + 0s test
MISSED   src/lib.rs:10:23: replace * with / in shipping_cost in 0s build + 0s test
MISSED   src/lib.rs:15:5: replace shipping_for_order -> u32 with 0 in 0s build + 0s test
MISSED   src/lib.rs:15:5: replace shipping_for_order -> u32 with 1 in 0s build + 0s test
MISSED   src/lib.rs:15:20: replace >= with < in shipping_for_order in 0s build + 0s test
9 mutants tested in 4s: 9 missed
```

**What it means:**

1. First the tool runs the tests on the *unmutated* code (the "baseline"). They must pass, otherwise the result would mean nothing.
2. **All 9 mutants are missed**, although line coverage is 100 %. Every mutation, even "the function always returns 0", is invisible to the tests, because the test checks nothing. This is the proof
   that coverage and test quality are different things.
3. The tool also writes the details into the folder `mutants.out/` (`missed.txt`, `caught.txt`, and a log for each mutant). The folder is in your `.gitignore`.

## 6. Killing the mutants

Replace `it_runs` by the two real tests of Section 4 and run `cargo mutants` again:

```text
Found 9 mutants to test
ok       Unmutated baseline in 0s build + 0s test
 INFO Auto-set test timeout to 20s
MISSED   src/lib.rs:5:28: replace <= with > in shipping_cost in 0s build + 0s test
MISSED   src/lib.rs:10:23: replace * with + in shipping_cost in 0s build + 0s test
MISSED   src/lib.rs:10:23: replace * with / in shipping_cost in 0s build + 0s test
MISSED   src/lib.rs:15:5: replace shipping_for_order -> u32 with 0 in 0s build + 0s test
9 mutants tested in 4s: 4 missed, 5 caught
```

Five mutants are now **caught**. The four missed ones each tell you what is missing:

| Missed mutant | Why the tests do not notice | The test that kills it |
|---|---|---|
| line 5: `<=` with `>` | no test uses a weight between 501 and 2,000 | a medium parcel |
| line 10: `*` with `+` or `/` | no test uses `express = true` | an express parcel |
| line 15: `shipping_for_order` always `0` | the only test expects `0` | an order that is *not* free |

Write those tests (and the heavy class, which coverage also flagged), and add boundary tests, which come next:

```rust
    #[test]
    fn a_medium_parcel_costs_900() {
        assert_eq!(shipping_cost(1_000, false), 900);
    }

    #[test]
    fn a_heavy_parcel_costs_1500() {
        assert_eq!(shipping_cost(3_000, false), 1_500);
    }

    #[test]
    fn express_costs_twice_as_much() {
        assert_eq!(shipping_cost(100, true), 1_000);
    }

    #[test]
    fn a_small_order_pays_for_shipping() {
        assert_eq!(shipping_for_order(100, 100, false), 500);
    }

    #[test]
    fn the_weight_limits_belong_to_the_cheaper_class() {
        assert_eq!(shipping_cost(500, false), 500);
        assert_eq!(shipping_cost(501, false), 900);
        assert_eq!(shipping_cost(2_000, false), 900);
        assert_eq!(shipping_cost(2_001, false), 1_500);
    }

    #[test]
    fn five_thousand_cents_is_enough_for_free_shipping() {
        assert_eq!(shipping_for_order(4_999, 100, false), 500);
        assert_eq!(shipping_for_order(5_000, 100, false), 0);
    }
```

```bash
cargo test && cargo llvm-cov --summary-only && cargo mutants
```

```text
test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
Filename       Regions  Missed Regions  Cover     Functions  Missed Functions  Executed   Lines  Missed Lines  Cover
…/src/lib.rs        84               0  100.00%          10                 0  100.00%      41             0  100.00%
Found 9 mutants to test
ok       Unmutated baseline in 0s build + 0s test
 INFO Auto-set test timeout to 20s
9 mutants tested in 4s: 9 caught
```

Now both numbers are 100 %, and this time the 100 % means something: every line runs **and** every mutation is noticed.

**The boundary tests.** `the_weight_limits_belong_to_the_cheaper_class` checks the values exactly on the edges: 500 and 501, 2,000 and 2,001, and the same for 4,999 and 5,000. Bugs love borders
(is it "up to 500" or "less than 500"?). Notice that `cargo mutants` did not produce the mutant "`<=` becomes `<`", so even the 9 caught mutants did not prove that the border is right. If you change
`weight_grams <= 500` to `< 500`, only the boundary test fails. The tools find holes; **thinking about edge cases is still your job**.

## 7. Using the tools wisely

**Coverage**

- It is a *negative* signal: low coverage in an important module is a warning, high coverage proves little.
- Do not set a target of 100 %. Chasing the number produces tests like `it_runs`. A team goal such as "at least 80 % lines, and no drop in a pull request" works better as a safety net in CI:
  `cargo llvm-cov --fail-under-lines 80` (Lesson 09.12 builds the CI pipeline).
- Exclude what should not count: test helpers, generated code (`--ignore-filename-regex`).

**Mutation testing**

- It is slow: every mutant needs a build and a test run. On a big project it can take hours. Run it on a **small, important part** (`cargo mutants -f src/lib.rs`), or only on what a branch changed
  (`cargo mutants --in-diff changes.diff`, where `changes.diff` is the output of `git diff`), or at night in CI.
- Not every missed mutant is a problem. An **equivalent mutant** changes the code but not the behaviour (for example, replacing `x * 1` by `x / 1`), so no test could catch it. Read each survivor
  and decide: a missing test, or irrelevant? Code that you decide to leave out can be marked with the attribute `#[mutants::skip]` from the `mutants` crate (see its documentation).
- A mutant that makes the program loop forever is stopped by the *timeout* (`cargo mutants` set it to 20 seconds automatically in our run) and is counted separately as "timeout".
- Make sure that the tests pass without mutation first (the "baseline"). A flaky test (Lesson 05.6) makes the results unreliable.

**Blind spots of both tools** (an exercise shows one): they cannot see a test that is *missing for a requirement that is not in the code*, such as an insurance minimum that is never mutated, and they
do not invent edge cases. The three habits together, hand-picked examples, properties (Lesson 09.4) and these tools, complement each other.

> 🧪 **Testing:** A good routine for an important module: write tests first (TDD, test-driven development, Lesson 09.6), look at `cargo llvm-cov --html` for untested code, run `cargo mutants` on that module, and turn every
> relevant survivor into a test.

> ⚡ **Performance:** Coverage instrumentation slows the tests down (often by a factor of 1.5 to 3) and mutation testing multiplies the test time by the number of mutants. Run both in separate CI jobs, not
> in the quick edit-and-test loop.

## Common mistakes

**1. Treating the percentage as the goal.** Tests without assertions, like `it_runs`, give 100 % and protect nothing. Measure with mutants, and read the red lines.

**2. Forgetting the component.** Without `llvm-tools-preview` the tool stops with an error that asks you to run `rustup component add llvm-tools-preview`. Run the command and try again.

**3. Running `cargo mutants` when the tests are red.** The baseline fails, and the tool stops with `ERROR cargo test failed in an unmutated tree, so no mutants were tested`, because every mutant would "fail" for the wrong reason. Make the tests green first.

**4. Committing `mutants.out` or `target/llvm-cov`.** They are build results, often large. Keep them in `.gitignore`.

**5. Ignoring every survivor.** "The tool is too strict." Sometimes it is, but each survivor is a statement "I changed your code, and nobody complained". Read it before you dismiss it.

## Try it yourself

1. Create the package, install the tools, write the code and the weak test `it_runs`. Run `cargo llvm-cov` (100 %) and `cargo mutants` (9 missed).
2. Replace `it_runs` by the two real tests. Run `cargo llvm-cov --text`, find the lines with `0`, then run `cargo mutants` (4 missed).
3. Add the six tests of Section 6. Check: 8 tests, 100 % lines, 9 of 9 mutants caught.
4. Make the border mistake: change `weight_grams <= 500` to `weight_grams < 500`. Which test fails? Change it back.
5. Run `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`, then commit: `git add code/p09-coverage && git commit -m "Add p09-coverage: coverage and mutation testing"`.

## Exercises

### Exercise 1 — Find the holes (easy)

With only the two tests `a_light_parcel_costs_500` and `a_big_order_ships_free`, run `cargo llvm-cov --summary-only --show-missing-lines`. Which lines does it list as not covered, and what behaviour do they belong to?

<details>
<summary>Hint</summary>

The list appears under `Uncovered Lines:`. Compare it with the `--text` view of Section 4.

</details>

<details>
<summary>Answer</summary>

```text
Uncovered Lines:
…/src/lib.rs: 5-6, 8, 18
```

Lines 5 and 6 are the medium class (more than 500 g, up to 2,000 g), line 8 is the heavy class, and line 18 is the call to `shipping_cost` from `shipping_for_order` for an order that is not free. Each of these
is a missing test. (The `--text` view also shows the never-run express branch of line 10 as `^0`.)

</details>

### Exercise 2 — Kill one mutant (medium)

After the two real tests, `cargo mutants` reports `MISSED src/lib.rs:5:28: replace <= with > in shipping_cost`. Explain in one sentence why the tests miss it, write the smallest test that kills it, and check this with `cargo mutants`.

<details>
<summary>Hint</summary>

Line 5 is the condition `weight_grams <= 2_000`. Which weights reach it, and which of them give a different answer when the condition is inverted?

</details>

<details>
<summary>Answer</summary>

The only weight tested is 100 g, and it is decided by the first condition (`<= 500`), so line 5 is never evaluated. A weight of 1,000 g reaches the second condition, where the original code answers 900 and the
mutant answers 1,500:

```rust
    #[test]
    fn a_medium_parcel_costs_900() {
        assert_eq!(shipping_cost(1_000, false), 900);
    }
```

With this test, `cargo mutants` reports one mutant fewer: `replace <= with > in shipping_cost` (line 5) is caught, because the mutant makes `shipping_cost(1_000, false)` return 1,500.

</details>

### Exercise 3 — A blind spot (hard)

Add this function to the library, with the test `big_orders_pay_two_percent` that asserts `insurance(10_000) == 200`:

```rust
/// Insurance costs 2% of the order (one fiftieth, rounded down), but at least 100 cents.
pub fn insurance(order_cents: u32) -> u32 {
    (order_cents / 50).max(100)
}
```

Run `cargo mutants` and `cargo llvm-cov --summary-only`. Both are satisfied. Now delete `.max(100)` from the function: do the tests notice? What does this show, and what test do you add?

<details>
<summary>Hint</summary>

`cargo mutants --list` shows which mutants exist for `insurance`. Is there one for the `max` call? Think about which inputs make `max(100)` matter.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn big_orders_pay_two_percent() {
        assert_eq!(insurance(10_000), 200);
    }
```

`cargo mutants` lists four mutants for `insurance` (the function replaced by `0` or `1`, and `/` replaced by `%` or `*`) and catches all of them; coverage is 100 %. If you delete `.max(100)`, the tests still
pass: the minimum rule is **not tested**, and `cargo mutants` has no mutant that removes a method call such as `.max(100)`. The tools cannot check requirements that your tests do not mention. Add a test
for the rule, with an input where the minimum matters:

```rust
    #[test]
    fn small_orders_pay_the_minimum_insurance() {
        assert_eq!(insurance(0), 100);
        assert_eq!(insurance(5_000), 100);
    }
```

With `.max(100)` removed, `insurance(0)` returns 0 and the test fails. Lesson: use the tools to find holes, and use the list of requirements to decide what the tests must say.

</details>

## Quiz

1. What does line coverage of 100 % tell you, and what does it not tell you?

<details>
<summary>Answer</summary>

It tells you that every line was executed at least once while the tests ran. It does not tell you that the results were checked: a test without assertions can give 100 %.

</details>

2. What is a mutant, and what does it mean when it is "missed"?

<details>
<summary>Answer</summary>

A mutant is a copy of your program with one small deliberate change (for example `+` replaced by `-`). If it is missed, all tests still pass with the change, so no test notices that behaviour.

</details>

3. Why does `cargo mutants` first run the tests on the unmodified code?

<details>
<summary>Answer</summary>

To make sure the tests pass at the start (the baseline). If they fail without a mutation, a failing run on a mutant would prove nothing.

</details>

4. Why can coverage go down while the tests get better?

<details>
<summary>Answer</summary>

A weak test can execute many lines without checking anything. Replacing it by a real test that executes fewer lines lowers the coverage, but the new test actually checks the results.

</details>

5. Name two ways to make mutation testing faster on a large project.

<details>
<summary>Answer</summary>

Run it only on a part (`-f src/file.rs` or a package), only on the changes of a branch (`--in-diff`), run mutants in parallel with `-j`, or run it in a separate nightly CI job.

</details>

6. Which tool shows you "the lines no test runs", and which shows you "tests that check nothing"?

<details>
<summary>Answer</summary>

`cargo llvm-cov` shows the lines that no test runs. `cargo mutants` shows the changes that no test notices, which means tests that check nothing about that code.

</details>

## Summary

- Coverage (`cargo llvm-cov`) tells you which code ran during the tests: use `--text` or `--html` to find holes, `--fail-under-lines N` as a safety net in CI.
- 100 % coverage does not mean good tests: a test without assertions executes everything and checks nothing.
- Mutation testing (`cargo mutants`) changes the code in small ways and checks that a test fails. A missed mutant is a missing or weak test; a caught mutant is a test that works.
- Turn each relevant survivor into a test. Some survivors are equivalent mutants and can be ignored.
- Add boundary tests yourself: the tools do not invent them. Both tools have blind spots.
- Mutation testing is slow; run it on the important or the changed code. Keep `mutants.out*` out of Git.

## Sources and further reading

- [cargo-llvm-cov on GitHub](https://github.com/taiki-e/cargo-llvm-cov) — installation, all report formats and options, and how to use the tool in CI.
- [cargo-mutants (mutants.rs)](https://mutants.rs/) — the manual of `cargo mutants`: the list of mutations, `--in-diff`, skipping code, and interpreting the results.
- [The rustc book — Instrumentation-based code coverage](https://doc.rust-lang.org/rustc/instrument-coverage.html) — how the compiler produces the coverage data that `cargo llvm-cov` reads.
- [The Rustup book — Components](https://rust-lang.github.io/rustup/concepts/components.html) — what `rustup component add llvm-tools-preview` installs.

---

[🏠 Start](../START-HERE.md) · [← 09.6 Test-driven development in practice](06-tdd-in-practice.md) · [📋 Phase 09](README.md) · [09.8 Fuzzing →](08-fuzzing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
