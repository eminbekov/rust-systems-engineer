<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.5 Fault injection and chaos testing](05-fault-injection-and-chaos-testing.md) · [📋 Phase 20](README.md) · [20.7 Supply-chain security in depth →](07-supply-chain-security-in-depth.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/06-formal-verification-with-kani.md) <!-- nav -->

# Lesson 20.6 — Formal verification basics with Kani

> **Phase 20 — Reliability and hardening** · Lesson 6 of 11 · ⏱ about 135 minutes

## What you will learn

- The difference between a test, which checks some inputs, and a proof, which checks *all* inputs.
- How the model checker Kani works, how to install it, and how to write a *proof harness* with `kani::any()`.
- How to read a counterexample and turn it into a normal unit test.
- How to state assumptions with `kani::assume`, how to verify loops with `#[kani::unwind]`, and what "bounded" means.
- Where Kani fits next to tests, property tests and fuzzing, and how to run it in CI (continuous integration).

## Before you start

- You finished [Lesson 20.5 — Fault injection and chaos testing](05-fault-injection-and-chaos-testing.md).
- You know property-based testing from [Lesson 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md) and integer overflow from [Lesson 12.2 — Integers in memory](../12-computer-systems/02-integers-in-memory.md).
- About 1 GB (gigabyte) of free disk space for Kani and its own toolchain.
- Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p20-kani
cd code/p20-kani
```

## New words

| Word | Meaning |
|---|---|
| **formal verification** | Proving with mathematical methods that a program has a property for every possible input, instead of trying examples. |
| **model checking** | A kind of formal verification: a tool explores *all* possible states of a program (or of a model of it) and looks for one that breaks a rule. |
| **Kani** | A model checker for Rust, developed by Amazon Web Services as open source. It checks for panics, overflow, invalid memory use and failed assertions. |
| **CBMC** (C Bounded Model Checker) | The model-checking engine that Kani uses under the hood. Kani translates your Rust code into a form that CBMC can check. |
| **SAT solver** (satisfiability solver) | A program that decides whether a big logical formula can be made true, and if yes, finds values that do it. CBMC uses one to search for a bad input. |
| **proof harness** | A function marked `#[kani::proof]` that sets up inputs and states what must be true. It looks like a test, but Kani checks it for all inputs. |
| **symbolic value** | A value from `kani::any()` that stands for *every* possible value of its type at once, not for one random value. |
| **assumption** | A condition given with `kani::assume(...)`: Kani only considers inputs for which it is true. |
| **counterexample** | Concrete input values for which a check fails. Kani can print them as a unit test (*concrete playback*). |
| **bounded** | Checked only up to a limit: loops up to a number of rounds, arrays up to a length. A bounded proof says nothing about bigger inputs. |
| **unwinding** | Turning a loop into a fixed number of copies of its body so that the checker can reason about it. `#[kani::unwind(n)]` sets the number. |

## 1. Tests check examples; proofs check everything

A unit test checks one input. A property test (Lesson 09.4) checks a few hundred random inputs. A fuzzer (Lessons 09.8 and 20.3) checks millions. All of them are **examples**. A function of two `u32` values has 2⁶⁴ possible inputs, about 18 billion billion. Even at a billion tests per second, trying them all would take more than 500 years.

**Formal verification** answers the question differently: it *reasons* about all inputs at once. **Kani** does this by **model checking**:

```text
your Rust code          Kani            CBMC                    SAT solver
+ a proof harness  ──►  translates ──►  builds one huge   ──►  searches for values
                        to a model      logical formula:        that make the formula
                                        "some check fails"      true
                                                                 │
                         VERIFICATION: SUCCESSFUL ◄── no values exist
                         VERIFICATION: FAILED     ◄── found them: a counterexample
```

If the solver proves that no input can make a check fail, the property holds for *every* input: that is a proof, not a test. Kani automatically adds checks for everything that could panic or be undefined: arithmetic overflow, division by zero, out-of-bounds indexing, `unwrap` on `None`, and invalid pointers in `unsafe` code. Your own `assert!` calls are checked too.

The price: it works on small pieces of code, it can be slow, and loops must be **bounded** (Section 4).

## 2. Install Kani and write a first proof

Kani is installed in two steps (from the official installation guide): a small installer from crates.io, then the real tools, which `cargo kani setup` downloads into `~/.kani` (set the environment variable `KANI_HOME` to use another folder):

```bash
cargo install --locked kani-verifier
cargo kani setup
cargo kani --version
```

```text
   Installed package `kani-verifier v0.68.0` (executables `cargo-kani`, `kani`)
[0/5] Running Kani first-time setup...
...
[2/5] Downloading Kani release bundle: kani-0.68.0-aarch64-apple-darwin.tar.gz
[3/5] Installing rust toolchain version: nightly-2026-08-21-aarch64-apple-darwin
...
[5/5] Successfully completed Kani first-time setup.
Kani Rust Verifier 0.68.0 (cargo plugin)
CBMC 6.11.0
```

(Shortened. Kani brings its own nightly toolchain with a fixed date; you do not need to change your own toolchains.)

Kani sets the condition `cfg(kani)` while it compiles, so proofs live in a module that normal builds ignore. Tell the compiler that `kani` is a known condition name, as you did for `loom` in Lesson 14.9. Add to `Cargo.toml`:

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ["cfg(kani)"] }
```

Now the code. Replace `src/lib.rs`:

```rust
//! Small functions with proofs checked by Kani.

/// The point halfway between `a` and `b`, rounded down.
pub fn midpoint(a: u32, b: u32) -> u32 {
    (a + b) / 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn midpoint_of_small_numbers() {
        assert_eq!(midpoint(2, 8), 5);
        assert_eq!(midpoint(3, 4), 3);
    }
}

#[cfg(kani)]
mod proofs {
    use super::*;

    #[kani::proof]
    fn midpoint_is_between_the_two_values() {
        let a: u32 = kani::any();
        let b: u32 = kani::any();
        let m = midpoint(a, b);
        assert!(a.min(b) <= m && m <= a.max(b));
    }
}
```

**What happens:**

1. `midpoint` is the classic way to compute the middle of two numbers, used for example in binary search (Lesson 07.10). The unit test passes.
2. `#[cfg(kani)]` compiles the `proofs` module only when Kani runs. The `kani` crate is available there automatically; you do not add it to `Cargo.toml`.
3. `#[kani::proof]` marks a **proof harness**. `kani::any()` gives a **symbolic value**: not one random `u32`, but "any `u32` at all".
4. The assertion states the property: the midpoint lies between the two values. Kani also checks every arithmetic operation for overflow.

```bash
cargo test -q
cargo kani --output-format terse
```

```text
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
Kani Rust Verifier 0.68.0 (cargo plugin)
CBMC 6.11.0
...
Checking harness proofs::midpoint_is_between_the_two_values...

VERIFICATION RESULT:
 ** 1 of 16 failed
Failed Checks: attempt to add with overflow
 File: "src/lib.rs", line 5, in midpoint

VERIFICATION:- FAILED
Verification Time: 0.029502625s

Manual Harness Summary:
Verification failed for - proofs::midpoint_is_between_the_two_values
Complete - 0 successfully verified harnesses, 1 failures, 1 total.
```

**How to read it:**

1. `--output-format terse` shows only the failed checks. Without it, Kani lists all 16 checks it made, most of them inside the standard library. (Kani also prints a `warning` about `#![feature(register_tool)]` from its own setup; you can ignore it.)
2. `attempt to add with overflow` at line 5: `a + b` can be bigger than `u32::MAX`. The unit test with 2 and 8 could never show this.
3. The whole search took 0.03 seconds for all 2⁶⁴ input pairs.

Which inputs? Ask for a **counterexample** as a ready-made test with *concrete playback* (still an unstable option, hence `-Z`):

```bash
cargo kani -Z concrete-playback --concrete-playback=print --output-format terse
```

```text
Concrete playback unit test for `proofs::midpoint_is_between_the_two_values`:
```
```rust
/// Test generated for harness `proofs::midpoint_is_between_the_two_values` 
///
/// Check for `assertion`: "attempt to add with overflow"

#[test]
fn kani_concrete_playback_midpoint_is_between_the_two_values_5132311777728133568() {
    let concrete_vals: Vec<Vec<u8>> = vec![
        // 2147483648
        vec![0, 0, 0, 128],
        // 2147483648
        vec![0, 0, 0, 128],
    ];
    kani::concrete_playback_run(concrete_vals, midpoint_is_between_the_two_values);
}
```

The comments show the values: `a = 2147483648` and `b = 2147483648` (2³¹ each, so the sum is 2³², one more than `u32::MAX`). The generated test runs only with Kani's helper library; it is easier to copy the *values* into a normal unit test (Exercise 3).

Fix `midpoint` so that it never adds two large numbers:

```rust
pub fn midpoint(a: u32, b: u32) -> u32 {
    let (low, high) = (a.min(b), a.max(b));
    low + (high - low) / 2
}
```

```bash
cargo kani --output-format terse
```

```text
Checking harness proofs::midpoint_is_between_the_two_values...

VERIFICATION RESULT:
 ** 0 of 17 failed

VERIFICATION:- SUCCESSFUL
Verification Time: 0.051254917s

Manual Harness Summary:
Complete - 1 successfully verified harnesses, 0 failures, 1 total.
```

`high - low` cannot underflow because `high >= low`, and `low + (high - low) / 2` is at most `high`. Kani confirms it for every pair of `u32` values. (The standard library has this function too: `a.midpoint(b)`.)

> 🧪 **Testing:** A proof harness is a test with a much stronger promise. Write it like a property test: "for all inputs, this property holds". Many property tests from Lesson 09.4 can be turned into Kani proofs by replacing the generated values with `kani::any()`.

## 3. Assumptions: saying what the inputs may be

Most functions have rules for their inputs. Add this to `src/lib.rs` (above the `tests` module), and a second harness to the `proofs` module:

```rust
/// What percentage of `total` is `part`, rounded down.
pub fn percent(part: u32, total: u32) -> u32 {
    part * 100 / total
}
```

```rust
    #[kani::proof]
    fn percent_is_at_most_100() {
        let part: u32 = kani::any();
        let total: u32 = kani::any();
        let p = percent(part, total);
        assert!(p <= 100);
    }
```

`--harness` runs only the harnesses whose name contains the given text:

```bash
cargo kani --output-format terse --harness percent_is_at_most_100
```

```text
Checking harness proofs::percent_is_at_most_100...

VERIFICATION RESULT:
 ** 3 of 5 failed
Failed Checks: assertion failed: p <= 100
 File: "src/lib.rs", line 53, in proofs::percent_is_at_most_100
Failed Checks: attempt to multiply with overflow
 File: "src/lib.rs", line 11, in percent
Failed Checks: attempt to divide by zero
 File: "src/lib.rs", line 11, in percent

VERIFICATION:- FAILED
```

(The line numbers depend on where you placed the code.) Three problems at once. Two of them are about *misuse*: `total = 0`, and `part` bigger than `total` (then the result is more than 100 %). Those are the caller's rules, and the function should document them. **Assumptions** tell Kani to consider only inputs that follow the rules. Add one line to the harness, before the call:

```rust
        kani::assume(total > 0 && part <= total);
```

```text
Checking harness proofs::percent_is_at_most_100...

VERIFICATION RESULT:
 ** 1 of 5 failed
Failed Checks: attempt to multiply with overflow
 File: "src/lib.rs", line 11, in percent

VERIFICATION:- FAILED
```

The overflow remains, and it is a real bug: for any `part` above 42 949 672, `part * 100` does not fit in a `u32`, even with perfectly valid inputs such as `percent(50_000_000, 100_000_000)`. Exercise 2 asks you to fix it.

> ⚠️ **Common mistake:** An assumption that is too strong proves nothing. `kani::assume(total > 0 && part <= total && part < 1000)` would make the harness pass and hide the overflow. Assume only what the function's documentation demands from its callers, and enforce those rules in code (a check, a type such as `NonZeroU32`) where you can.

## 4. Loops and unwinding

A model checker cannot run a loop "until it stops"; it has to know how many rounds to consider. Add a function with a loop, and a harness that checks it for **every slice of up to 4 bytes**:

```rust
/// How many bytes in `data` are zero.
pub fn count_zeros(data: &[u8]) -> usize {
    let mut count = 0;
    for &byte in data {
        if byte == 0 {
            count += 1;
        }
    }
    count
}
```

```rust
    #[kani::proof]
    #[kani::unwind(2)]
    fn count_zeros_is_at_most_the_length() {
        let data: [u8; 4] = kani::any();
        let len: usize = kani::any();
        kani::assume(len <= data.len());
        let slice = &data[..len];
        assert!(count_zeros(slice) <= slice.len());
    }
```

**What happens:**

1. `let data: [u8; 4] = kani::any();` makes four symbolic bytes: every possible content at once.
2. A symbolic `len` from 0 to 4 makes `slice` stand for every slice of length 0, 1, 2, 3 or 4.
3. `#[kani::unwind(2)]` says "consider at most 2 rounds of every loop". That is too few on purpose.

```bash
cargo kani --output-format terse --harness count_zeros
```

```text
Checking harness proofs::count_zeros_is_at_most_the_length...

VERIFICATION RESULT:
 ** 1 of 79 failed (78 undetermined)
Failed Checks: unwinding assertion loop 0
 File: "src/lib.rs", line 17, in count_zeros

VERIFICATION:- FAILED
[Kani] info: Verification output shows one or more unwinding failures.
[Kani] tip: Consider increasing the unwinding value or disabling `--unwinding-assertions`.
```

Kani does not pretend: it reports that the loop can run more often than 2 times, so it could not check the rest (`78 undetermined`). The bound must be one more than the largest number of rounds (one extra check that the loop really ends). Change it to `#[kani::unwind(5)]`:

```text
Checking harness proofs::count_zeros_is_at_most_the_length...

VERIFICATION RESULT:
 ** 0 of 79 failed (1 unreachable)

VERIFICATION:- SUCCESSFUL
Verification Time: 0.20878696s
```

The property is now **proven for all slices of up to 4 bytes**, nothing more. That is what **bounded** means. In practice it is still very valuable: most bugs in loops show up with tiny sizes (empty, one element, the wrap-around point), and Kani checks all of those completely. `unreachable` means one check sits on a path that can never run, which is fine here.

> ⚡ **Performance:** The time Kani needs grows quickly with the bound and with the number of symbolic bytes. Start with small bounds (2–8), and grow them only while the run stays fast. A harness that takes minutes should be split into smaller ones.

## 5. Where Kani fits

| Technique | Inputs checked | Good for | Weak at |
|---|---|---|---|
| Unit tests | the examples you write | known cases, documentation | the cases you did not think of |
| Property tests (09.4) | hundreds of random inputs | quick, broad checks on every build | rare edge values |
| Fuzzing (09.8, 20.3) | millions of inputs, guided by coverage | parsers, big code, long runs | proving absence of bugs |
| **Kani** | **all** inputs, within bounds | small critical functions: arithmetic, index math, `unsafe` blocks, parsers of fixed-size headers | big programs, deep loops, threads and async code, code that calls the operating system |

Use Kani for the small pieces where a bug is expensive and the input space is huge but the code is short: size calculations, index arithmetic in ring buffers (Exercise 4), bit manipulation, and the safety conditions of `unsafe` blocks (Lesson 10.12). Big projects use it for exactly that: the Rust standard library and Amazon's Firecracker virtual machine monitor have Kani proofs for such pieces.

Kani runs in CI like any other check. Save as `.github/workflows/kani.yml`:

```yaml
name: Kani

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  proofs:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v7
      - name: Install Kani
        run: |
          cargo install --locked kani-verifier
          cargo kani setup
      - name: Run all proof harnesses
        run: cargo kani --output-format terse
```

```bash
actionlint .github/workflows/kani.yml && echo "actionlint: no problems"
```

```text
actionlint: no problems
```

(The commands are the same as on your Mac; the workflow itself runs on GitHub's Linux machines, which this course cannot use. The Kani project also publishes a ready-made action, `model-checking/kani-github-action`.) `cargo kani` exits with an error when any harness fails, so the job fails too.

> 🔒 **Security:** Integer overflow in size and index calculations is one of the most common roots of memory bugs in `unsafe` code. A Kani harness over the arithmetic that computes a buffer size or an offset gives you a guarantee that no amount of testing can give.

## Common mistakes

**1. Forgetting `#[cfg(kani)]`.** Without it, `cargo build` tries to compile `kani::any()` and fails with ``error[E0433]: cannot find module or crate `kani` in this scope``. Keep proofs in a `#[cfg(kani)]` module.

**2. Assuming too much.** Every `kani::assume` removes inputs from the proof. Check each assumption against the function's documented contract (Section 3).

**3. Reading "SUCCESSFUL" as "correct".** Kani proves exactly the checks it made: no panics, no overflow, and *your* assertions. If the harness asserts nothing about the result, a wrong result still passes. Write assertions that describe what the function must do.

**4. Forgetting that proofs are bounded.** `#[kani::unwind(5)]` with a 4-byte array proves nothing about a 100-byte slice. Write the bound into the harness name or a comment.

## Try it yourself

1. Install Kani with the two commands of Section 2 and check `cargo kani --version`.
2. Create `code/p20-kani` with the overflowing `midpoint`, its unit test and the proof. Run `cargo test`, then `cargo kani --output-format terse`.
3. Print the counterexample with concrete playback, then fix `midpoint` and run Kani again.
4. Add `percent` and its harness. Run it without and with the assumption.
5. Add `count_zeros` with `#[kani::unwind(2)]`, read the unwinding failure, and change the bound to 5.

## Exercises

### Exercise 1 — Three failures, three inputs (easy)

For each of the three failed checks of `percent_is_at_most_100` (Section 3, before the assumption), give one concrete pair `(part, total)` that triggers it.

<details>
<summary>Hint</summary>

Division by zero needs one special value of `total`. For the multiplication, find the smallest `part` for which `part * 100 > u32::MAX`.

</details>

<details>
<summary>Answer</summary>

- **Divide by zero:** `(1, 0)`: any `part` with `total = 0`.
- **Multiply with overflow:** `(42_949_673, 100_000_000)`: `u32::MAX` is 4 294 967 295, so every `part` from 42 949 673 up overflows when multiplied by 100.
- **Assertion `p <= 100`:** `(5, 1)`: `part` bigger than `total` gives 500 %.

</details>

### Exercise 2 — Fix `percent` and prove it (medium)

Change `percent` so that it cannot overflow for any input that follows the rules `total > 0` and `part <= total`, and so that the harness with the assumption passes.

<details>
<summary>Hint</summary>

Do the multiplication in a wider type: `u64::from(part) * 100` always fits in a `u64`.

</details>

<details>
<summary>Answer</summary>

```rust
/// What percentage of `total` is `part`, rounded down.
/// The caller must make sure that `total > 0` and `part <= total`.
pub fn percent(part: u32, total: u32) -> u32 {
    let wide = u64::from(part) * 100 / u64::from(total);
    wide as u32 // at most 100, because part <= total
}
```

```text
Checking harness proofs::percent_is_at_most_100...
...
VERIFICATION:- SUCCESSFUL
```

Why it works: `u32::MAX * 100` is about 4.3 × 10¹¹, far below `u64::MAX`, so the multiplication cannot overflow. With `part <= total`, the quotient is at most 100, so `as u32` loses nothing, and Kani proves the assertion `p <= 100` for every allowed pair.

</details>

### Exercise 3 — From counterexample to regression test (medium)

Write a normal unit test (it must run with `cargo test`, without Kani) that would have caught the original `midpoint` bug, using the values from the concrete playback. Add a second assertion with `u32::MAX`.

<details>
<summary>Hint</summary>

The playback values were `2147483648` and `2147483648`. What is the midpoint of two equal numbers?

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn midpoint_of_huge_numbers_found_by_kani() {
        assert_eq!(midpoint(2_147_483_648, 2_147_483_648), 2_147_483_648);
        assert_eq!(midpoint(u32::MAX, u32::MAX - 2), u32::MAX - 1);
    }
```

```text
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: with the old `(a + b) / 2`, both lines panic with `attempt to add with overflow` in a debug build. The test keeps the finding forever, also on machines where Kani is not installed, the same way fuzzer findings become regression tests.

</details>

### Exercise 4 — Prove the ring-buffer step (hard)

Ring buffers (Lessons 07.5 and 20.3) move an index forward with `(index + 1) % capacity`. Write `pub fn next_index(index: usize, capacity: usize) -> usize` and a harness that proves, for every `capacity > 0` and every `index < capacity`: the result is smaller than `capacity`; the last index wraps to 0; every other index moves forward by exactly one.

<details>
<summary>Hint</summary>

Use two symbolic `usize` values, one `kani::assume`, and an `if index == capacity - 1` with `assert_eq!` in both branches.

</details>

<details>
<summary>Answer</summary>

```rust
/// The slot after `index` in a ring buffer with `capacity` slots.
pub fn next_index(index: usize, capacity: usize) -> usize {
    (index + 1) % capacity
}
```

```rust
    #[kani::proof]
    fn next_index_stays_inside_and_wraps() {
        let capacity: usize = kani::any();
        let index: usize = kani::any();
        kani::assume(capacity > 0 && index < capacity);
        let next = next_index(index, capacity);
        assert!(next < capacity);
        if index == capacity - 1 {
            assert_eq!(next, 0);
        } else {
            assert_eq!(next, index + 1);
        }
    }
```

```text
Checking harness proofs::next_index_stays_inside_and_wraps...

VERIFICATION RESULT:
 ** 0 of 9 failed

VERIFICATION:- SUCCESSFUL
Verification Time: 0.9576767s
```

Why it works: Kani checks all 2¹²⁸ pairs of `usize` values that satisfy the assumption, including `index = usize::MAX - 1` with `capacity = usize::MAX`, where a careless `index + 1` could overflow. Since `index < capacity <= usize::MAX`, `index + 1` never overflows, and Kani confirms it.

</details>

## Quiz

1. What is the difference between a property test and a Kani proof harness?

<details>
<summary>Answer</summary>

A property test runs the code with many random example inputs. A Kani harness uses symbolic values (`kani::any()`), and Kani checks the property for every possible input (within the bounds), so a success is a proof.

</details>

2. Which checks does Kani add automatically, without any `assert!` from you?

<details>
<summary>Answer</summary>

Checks for everything that can panic or is undefined: arithmetic overflow, division by zero, out-of-bounds indexing, `unwrap` on `None` or `Err`, and invalid pointer use in `unsafe` code.

</details>

3. What does `kani::assume(x > 0)` do, and what is its danger?

<details>
<summary>Answer</summary>

It limits the proof to inputs where `x > 0`. If the assumption is stronger than the function's real contract, it hides bugs: the excluded inputs are never checked.

</details>

4. Kani reports `unwinding assertion loop 0`. What does it mean?

<details>
<summary>Answer</summary>

A loop can run more rounds than the `unwind` bound allows, so Kani could not check everything. Increase `#[kani::unwind(n)]` (to one more than the largest number of rounds) or make the inputs smaller.

</details>

5. A harness over `[u8; 4]` with `unwind(5)` succeeds. What exactly has been proven?

<details>
<summary>Answer</summary>

That the property holds for every possible content of the inputs up to that size (here every slice of up to 4 bytes). Nothing is proven for longer inputs: the proof is bounded.

</details>

6. For which code is Kani a better choice than fuzzing?

<details>
<summary>Answer</summary>

For small, critical functions with a huge input space: arithmetic on sizes and indexes, bit manipulation, the safety conditions of `unsafe` blocks. Fuzzing is better for big code such as whole parsers, and for long runs over realistic inputs.

</details>

## Summary

- Tests, property tests and fuzzers check examples; **formal verification** checks all inputs. Kani does this by model checking: CBMC and a SAT solver search for any input that breaks a check.
- A **proof harness** is a `#[kani::proof]` function in a `#[cfg(kani)]` module; `kani::any()` gives symbolic values. Kani checks overflow, panics and memory safety automatically, plus your assertions.
- A failure comes with a **counterexample**; concrete playback prints the values, and they become a normal regression test.
- `kani::assume` states the input rules; assume only what the contract demands.
- Loops need `#[kani::unwind(n)]`, and proofs are **bounded**: they hold up to the sizes you chose.
- Use Kani for small, critical code (sizes, indexes, `unsafe`), and run `cargo kani` in CI.

## Sources and further reading

- [The Kani Rust Verifier](https://model-checking.github.io/kani/) — the official book: [installation](https://model-checking.github.io/kani/install-guide.html), [first steps](https://model-checking.github.io/kani/tutorial-first-steps.html) and [loop unwinding](https://model-checking.github.io/kani/tutorial-loop-unwinding.html).
- [Kani — attributes](https://model-checking.github.io/kani/reference/attributes.html) — `#[kani::proof]`, `#[kani::unwind]` and the other attributes.
- [Kani — concrete playback](https://model-checking.github.io/kani/reference/experimental/concrete-playback.html) — how counterexamples become unit tests.
- [Kani in GitHub CI](https://model-checking.github.io/kani/install-github-ci.html) — the official GitHub Action and its options.
- [CBMC](https://github.com/diffblue/cbmc) — the bounded model checker under the hood.

---

[🏠 Start](../START-HERE.md) · [← 20.5 Fault injection and chaos testing](05-fault-injection-and-chaos-testing.md) · [📋 Phase 20](README.md) · [20.7 Supply-chain security in depth →](07-supply-chain-security-in-depth.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
