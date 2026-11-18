<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.3 Test doubles: fakes, stubs and mocks](03-test-doubles.md) · [📋 Phase 09](README.md) · [09.5 Snapshot testing →](05-snapshot-testing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/04-property-based-testing.md) <!-- nav -->

# Lesson 09.4 — Property-based testing

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 4 of 14 · ⏱ about 135 minutes

## What you will learn

- Why tests with hand-picked examples miss bugs, and what a *property* is.
- How to write property tests with the `proptest` crate: strategies, `proptest!`, `prop_assert!`.
- How `proptest` finds a failing input and *shrinks* it to the smallest one that still fails.
- Five patterns for finding good properties, and how to avoid properties that are too weak.

## Before you start

- You finished [Lesson 09.3 — Test doubles: fakes, stubs and mocks](03-test-doubles.md).
- You know unit tests and edge cases ([Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)). There, Section 7 only mentioned `proptest` by name. Now you use it.
- You met the bill splitter in [Lesson 09.1](01-the-test-pyramid.md). Here its `split_bill` function comes back, because a function with a clear rule ("the shares add up to the total") is perfect for properties.

## New words

| Word | Meaning |
|---|---|
| **property** | A statement that must be true for *every* valid input, for example "the shares always add up to the total". |
| **property-based test** | A test that checks a property on many automatically generated inputs, instead of a few chosen by hand. |
| **example-based test** | An ordinary test: one hand-chosen input and one expected output. |
| **`proptest`** | A crate for property-based testing in Rust. |
| **strategy** | A recipe that tells `proptest` how to generate values of some type (for example "any `u64`" or "a text of 0 to 60 letters a or b"). |
| **case** | One generated input set, and one run of the test body with it. By default `proptest` runs 256 cases per test. |
| **shrinking** | After a failure, trying smaller and simpler inputs, to find the *minimal failing input* that is easy to read. |
| **minimal failing input** | The simplest input found that still makes the test fail. |
| **round trip** | Doing an operation and its reverse (encode, then decode) and expecting the original back. |
| **oracle** | A trusted, simpler (or slower) way to get the right answer, to compare against, for example the standard library's `sort`. |
| **invariant** | A fact that stays true however the input changes, for example "the output has the same length as the input". |
| **idempotent** | An operation that gives the same result if you apply it twice as if you applied it once (sorting a sorted list changes nothing). |
| **regression file** | A file in which `proptest` saves a failing input, so that it is tried again first on the next run. |

## 1. The limit of hand-picked examples

An example-based test looks like this:

```rust
assert_eq!(split_bill(1000, 3), vec![334, 333, 333]);
```

It is clear and fast, but it checks **one** input out of billions. You choose the inputs, so you choose the ones you
already thought about, and bugs love the ones you did not think about: a huge number, a single person, a text with ten
equal letters.

A **property-based test** turns the question around. Instead of "for 1000 and 3, the answer is
`[334, 333, 333]`", you state a rule that holds for all inputs: "for **any** total and **any** number of people, the
shares add up to the total". A library then generates hundreds of inputs, including nasty ones, and checks the rule on each.
If it finds a counter-example, it simplifies it and shows you the smallest one.

The two kinds complement each other. Examples document the behaviour ("this is what 10.00 split three ways looks
like") and are easy to read. Properties search for the surprises.

## 2. Setup and the first properties

```bash
cd ~/projects/rust
git switch main
git switch -c lesson/09-04-property-tests
cargo new --lib code/p09-proptest
cd code/p09-proptest
cargo add --dev proptest@1
```

Make `src/lib.rs` contain the module list. We will write the modules in this and the next sections:

```rust
pub mod rle;
pub mod sorting;
pub mod split;
```

Create `src/split.rs`:

```rust
/// Splits `total_cents` between `people` (at least 1). The shares add up to the total
/// exactly; the first `total_cents % people` people pay one cent more than the others.
pub fn split_bill(total_cents: u64, people: u64) -> Vec<u64> {
    let base = total_cents / people;
    let extra = total_cents % people;
    (0..people)
        .map(|i| if i < extra { base + 1 } else { base })
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    #[test]
    fn example_ten_dollars_between_three() {
        assert_eq!(split_bill(1000, 3), vec![334, 333, 333]);
    }

    proptest! {
        #[test]
        fn shares_add_up_to_the_total(total in any::<u64>(), people in 1u64..=1000) {
            let shares = split_bill(total, people);
            prop_assert_eq!(shares.iter().sum::<u64>(), total);
        }

        #[test]
        fn there_is_one_share_per_person(total in any::<u64>(), people in 1u64..=1000) {
            prop_assert_eq!(split_bill(total, people).len() as u64, people);
        }

        #[test]
        fn nobody_pays_more_than_one_cent_above_anybody_else(
            total in any::<u64>(),
            people in 1u64..=1000,
        ) {
            let shares = split_bill(total, people);
            let max = shares.iter().max().unwrap();
            let min = shares.iter().min().unwrap();
            prop_assert!(max - min <= 1, "max {} and min {}", max, min);
        }
    }
}
```

To keep the other modules compiling, create empty files `src/rle.rs` and `src/sorting.rs` for now. Run:

```bash
cargo test --lib split
```

```text
running 4 tests
test split::tests::example_ten_dollars_between_three ... ok
test split::tests::there_is_one_share_per_person ... ok
test split::tests::shares_add_up_to_the_total ... ok
test split::tests::nobody_pays_more_than_one_cent_above_anybody_else ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

**What happens:**

1. `use proptest::prelude::*` imports `proptest!`, `prop_assert!` and the strategy functions.
2. The macro `proptest! { … }` wraps ordinary-looking test functions. Each parameter has the form `name in strategy`:
   `total in any::<u64>()` means "`total` is any `u64`, from 0 to 18,446,744,073,709,551,615", and `people in 1u64..=1000` means
   "a number from 1 to 1000" (a range is a strategy, too).
3. The body is a test about a property. `prop_assert_eq!` and `prop_assert!` are like `assert_eq!` and `assert!`, but they
   report a failure in a way that `proptest` can use for shrinking and for a readable message. The third property
   shows that, like `assert!`, `prop_assert!` accepts a message with format arguments.
4. For each test, `proptest` generates **256 cases** and runs the body with each. 256 × 4 tests finish in
   about 10 milliseconds, because `split_bill` is fast.
5. We wrote four tests and no fixed number except the one example. We said *what must always hold*: the sum is right,
   the count is right, and the spread is at most one cent. Together these three properties nearly define the function.

> 💡 **Tip:** Keep the example test next to the property tests. When a property fails, the example shows what the function
> was meant to do.

## 3. Finding a real bug

Suppose a colleague "optimises" the division and writes it with floating-point numbers:

```rust
    let base = (total_cents as f64 / people as f64).floor() as u64;
```

Replace the line `let base = total_cents / people;` with this one and run the tests again. The example test still passes, because
1000 / 3 works fine in floating point. A property test finds the problem:

```text
---- split::tests::shares_add_up_to_the_total stdout ----
proptest: Saving this and future failures in …/proptest-regressions/split.txt
proptest: If this test was run on a CI system, you may wish to add the following line to your copy of the file. (You may need to create it.)
cc 902a0585c76884d24a6dc54991b0019277c5eef93e5d9803f997808069671741
thread 'split::tests::shares_add_up_to_the_total' (4769191) panicked at src/split.rs:21:5:
Test failed: assertion failed: `(left == right)`
  left: `2519100732292684288`,
 right: `2519100732292684033` at src/split.rs:25.
minimal failing input: total = 2519100732292684033, people = 1
	successes: 0
	local rejects: 0
	global rejects: 0
```

**What it means:**

1. `proptest` tried random values and found a failing case immediately. The numbers on your screen will be different:
   `proptest` starts from a random seed each time.
2. It then **shrank** the failure. It tried to make `total` and `people` smaller and simpler, while the test
   still failed, and stopped at `people = 1` and a still-huge `total`. The result is the *minimal failing input*, printed
   in the line that starts with `minimal failing input`.
3. Why does it fail? A 64-bit floating-point number (`f64`) keeps only 53 bits of precision, so integers above
   about 9 × 10¹⁵ get rounded: `…033` became `…288`. The shares no longer add up. With ordinary examples (small amounts) you would
   never have seen this.
4. `proptest` saved the failing case in the file `proptest-regressions/split.txt`. On every later run it tries the
   saved cases **first**, so a fixed bug stays fixed. **Commit this folder to Git.**
5. The report is *reproducible*: even though the generation is random, the input in the line `minimal failing input` is
   exact, so you can copy it into an ordinary example test: `assert_eq!(split_bill(2519100732292684033, 1).iter().sum::<u64>(), 2519100732292684033);`.

Put the integer division back (`let base = total_cents / people;`), delete the file `proptest-regressions/split.txt`
and run again to see all tests pass.

> 🔒 **Security:** Integer overflow and rounding errors in money, sizes and lengths are a common source of security bugs.
> Properties over `any::<u64>()` are a cheap way to search for them.

> ⚡ **Performance:** The division with `u64` is also faster than converting to `f64` and back. The "clever" version was both slower and wrong.

## 4. Strategies: describing the inputs

A strategy decides which values are generated. These are the ones you will use most:

| Strategy | Generates |
|---|---|
| `any::<T>()` | any value of the type `T` (`u8`, `i32`, `bool`, `String`, `char`, …) |
| `0u32..10`, `1u64..=1000` | a number in a range (`..` excludes the end, `..=` includes it) |
| `"[a-z]{1,8}"` | a text matching a small regular expression (here: 1 to 8 lowercase letters) |
| `(s1, s2)` | a tuple, with one value from each strategy |
| `prop::collection::vec(s, 0..5)` | a `Vec` of 0 to 4 values, each from the strategy `s` |
| `s.prop_map(\|x\| x * 2)` | the values of `s` transformed by a function |
| `prop_oneof![Just("yes"), Just("no")]` | one of several strategies; `Just(x)` always gives `x` |

Add this test inside the `proptest! { … }` block of `src/split.rs` to see them work. The test checks that every strategy gives
what it promises:

```rust
        #[test]
        fn a_tour_of_strategies(
            n in 0u32..10,
            text in "[a-z]{1,8}",
            pair in (any::<bool>(), 1u8..=6),
            list in prop::collection::vec(0u8..=9, 0..5),
            even in (0u32..100).prop_map(|x| x * 2),
            word in prop_oneof![Just("yes"), Just("no")],
        ) {
            prop_assert!(n < 10);
            prop_assert!((1..=8).contains(&text.len()));
            prop_assert!((1..=6).contains(&pair.1));
            prop_assert!(list.len() < 5);
            prop_assert!(even % 2 == 0);
            prop_assert!(word == "yes" || word == "no");
        }
```

**What happens:** each parameter is generated independently. `prop_map` is how you build your own values from simple ones: here
every even number below 200.

Sometimes a case does not make sense for a property. `prop_assume!(condition)` throws the case away (it counts as a
"local reject") and `proptest` tries another. Add this one too:

```rust
        #[test]
        fn an_inexact_split_gives_two_different_amounts(total in any::<u64>(), people in 2u64..=1000) {
            prop_assume!(total % people != 0);
            let shares = split_bill(total, people);
            prop_assert_ne!(shares.first(), shares.last());
        }
```

If the total divides evenly, all shares are equal and the property would be wrong, so we skip those cases. Use
`prop_assume!` for rare exceptions only: if it rejects most cases, `proptest` gives up with an error ("too many
rejects"). In that case build the right input with a better strategy instead.

> 💡 **Tip:** The number of cases is 256 by default. For a quick run use `PROPTEST_CASES=10 cargo test`; before a release try
> `PROPTEST_CASES=10000 cargo test`. In code you can set `#![proptest_config(ProptestConfig::with_cases(1000))]` as the first
> line inside the `proptest!` block.

## 5. Five patterns for finding properties

The hard part is not the library, it is knowing what to assert. These patterns cover most cases:

| Pattern | Idea | Example |
|---|---|---|
| **Round trip** | do an operation, then its reverse, and expect the original | `decode(encode(x)) == x` |
| **Invariant** | something that stays true for any input | the output has the same length as the input |
| **Oracle** | compare with a trusted, simpler way | our sort equals the standard library's sort |
| **Idempotence** | doing it twice equals doing it once | sorting a sorted list changes nothing |
| **Bounds / no crash** | the result stays inside limits, or the code does not panic | `x.clamp(low, high)` is between `low` and `high` |

### The round trip: run-length encoding

*Run-length encoding* (RLE) shortens text with repeated letters: `"aaabcc"` becomes `"3a1b2c"` (3 times `a`, 1 time `b`, 2 times
`c`). Create `src/rle.rs`, first with a decoder that has a subtle bug:

```rust
/// Run-length encoding: `"aaabcc"` becomes `"3a1b2c"`.
pub fn encode(text: &str) -> String {
    let mut result = String::new();
    let mut chars = text.chars().peekable();
    while let Some(c) = chars.next() {
        let mut count = 1;
        while chars.peek() == Some(&c) {
            chars.next();
            count += 1;
        }
        result.push_str(&count.to_string());
        result.push(c);
    }
    result
}

/// The reverse of `encode`: `"3a1b2c"` becomes `"aaabcc"`.
pub fn decode(encoded: &str) -> String {
    let mut result = String::new();
    let mut chars = encoded.chars();
    while let (Some(count), Some(c)) = (chars.next(), chars.next()) {
        let Some(count) = count.to_digit(10) else {
            break;
        };
        for _ in 0..count {
            result.push(c);
        }
    }
    result
}

#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    #[test]
    fn encodes_runs() {
        assert_eq!(encode("aaabcc"), "3a1b2c");
        assert_eq!(encode(""), "");
    }

    #[test]
    fn decodes_runs() {
        assert_eq!(decode("3a1b2c"), "aaabcc");
    }

    proptest! {
        #[test]
        fn decoding_an_encoded_text_gives_the_original(text in "[ab]{0,60}") {
            prop_assert_eq!(decode(&encode(&text)), text);
        }
    }
}
```

**What happens:**

1. `encode` goes through the characters; for each one it counts how many equal characters follow (`peekable` and
   `peek` look at the next character without consuming it) and writes the count and the character.
2. `decode` takes the characters in pairs: a count, then the character to repeat. (`while let (Some(count), Some(c)) = …`
   stops when the input has no complete pair.)
3. The two example tests pass. The property is the round trip: for any text of the letters `a` and `b` (0 to 60 of them),
   `decode(encode(text))` must be the original. We use only two letters so that long runs of equal letters occur often.

```bash
cargo test --lib rle
```

```text
---- rle::tests::decoding_an_encoded_text_gives_the_original stdout ----
proptest: Saving this and future failures in …/proptest-regressions/rle.txt
cc 25e7af3a55e5cd185e8cfce5beb83edb9a54a99eadb85d54fb8d10d4d65050a4
thread 'rle::tests::decoding_an_encoded_text_gives_the_original' (4755197) panicked at src/rle.rs:48:5:
Test failed: assertion failed: `(left == right)`
  left: `"0"`,
 right: `"bbbbbbbbbb"` at src/rle.rs:51.
minimal failing input: text = "bbbbbbbbbb"
	successes: 56
	local rejects: 0
	global rejects: 0
```

**What it means:** the minimal failing input is ten equal letters (yours may be ten `a`). `encode` writes `"10b"` (the count has two
digits), but `decode` reads exactly one character as the count: `'1'`, then repeats the character `'0'` once, and the result is `"0"`.
The two examples used counts of 1 to 3, so they never showed it. The shrinking is useful: a failing text of 57 letters would be hard
to understand; ten equal letters point straight at "counts of 10 or more".

Fix `decode` so that it collects all the digits of a count:

```rust
/// The reverse of `encode`: `"3a1b2c"` becomes `"aaabcc"`.
pub fn decode(encoded: &str) -> String {
    let mut result = String::new();
    let mut digits = String::new();
    for c in encoded.chars() {
        if c.is_ascii_digit() {
            digits.push(c);
        } else {
            let count: usize = digits.parse().unwrap_or(0);
            result.extend(std::iter::repeat_n(c, count));
            digits.clear();
        }
    }
    result
}
```

**What happens:** digits are collected into `digits`; when a non-digit arrives, it is the character to repeat, and `digits`
holds its count (`"10"` parses as 10). `repeat_n(c, count)` produces `c` exactly `count` times. Delete `proptest-regressions/rle.txt`
(or keep it: it makes sure that "bbbbbbbbbb" is tried first from now on) and the property passes.

### Invariant, oracle, idempotence: sorting

Create `src/sorting.rs`. Insertion sort is simple, and the standard library gives us a trusted oracle:

```rust
/// Sorts numbers from the smallest to the largest with insertion sort.
pub fn insertion_sort(numbers: &[i32]) -> Vec<i32> {
    let mut result = numbers.to_vec();
    for i in 1..result.len() {
        let mut j = i;
        while j > 0 && result[j - 1] > result[j] {
            result.swap(j - 1, j);
            j -= 1;
        }
    }
    result
}

#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    fn is_sorted(numbers: &[i32]) -> bool {
        numbers.windows(2).all(|pair| pair[0] <= pair[1])
    }

    proptest! {
        #[test]
        fn the_result_is_sorted(numbers in prop::collection::vec(any::<i32>(), 0..100)) {
            prop_assert!(is_sorted(&insertion_sort(&numbers)));
        }

        #[test]
        fn the_result_has_the_same_elements(numbers in prop::collection::vec(any::<i32>(), 0..100)) {
            let mut expected = numbers.clone();
            expected.sort();
            // The oracle: the standard library's sort must agree with ours.
            prop_assert_eq!(insertion_sort(&numbers), expected);
        }

        #[test]
        fn sorting_twice_changes_nothing(numbers in prop::collection::vec(any::<i32>(), 0..100)) {
            let once = insertion_sort(&numbers);
            prop_assert_eq!(insertion_sort(&once), once);
        }
    }
}
```

**What happens:** `windows(2)` gives every pair of neighbours (Lesson 05.5), so `is_sorted` checks that no pair is out of
order. The second test uses the **oracle** pattern: it says nothing about *how* to sort, it only demands the same answer as a function we
trust. The third uses **idempotence**. Run `cargo test` to see that all tests pass. (The `vec` strategy gives 0 to 99 numbers,
and `any::<i32>()` includes `i32::MIN`, `i32::MAX` and many duplicates.)

## 6. Properties that are too weak

A property that is too weak passes for wrong code. See it: add a broken "sort" to the test module, with the property "the result is
sorted" and a stronger one:

```rust
    /// A broken "sort" that returns nothing.
    fn broken_sort(_numbers: &[i32]) -> Vec<i32> {
        Vec::new()
    }

    proptest! {
        #[test]
        fn a_weak_property_accepts_the_broken_sort(numbers in prop::collection::vec(any::<i32>(), 0..100)) {
            prop_assert!(is_sorted(&broken_sort(&numbers)));
        }

        #[test]
        fn a_stronger_property_rejects_it(numbers in prop::collection::vec(any::<i32>(), 0..100)) {
            let result = broken_sort(&numbers);
            prop_assert!(is_sorted(&result));
            prop_assert_eq!(result.len(), numbers.len());
        }
    }
```

```text
test sorting::tests::a_stronger_property_rejects_it ... FAILED
test sorting::tests::a_weak_property_accepts_the_broken_sort ... ok
…
minimal failing input: numbers = [
    0,
]
```

**What it means:** an empty list is "sorted", so the weak property is satisfied by a function that does nothing. The stronger property
also demands the same length, and it fails with the smallest possible counter-example, a list with the single number `0`.
Whenever you write a property, ask: *"What is the stupidest function that would still pass?"* If there is one, add a property that
rejects it. (Delete the two experimental tests afterwards so that your test suite is green.)

## Common mistakes

**1. Writing the code again inside the property.** `prop_assert_eq!(split_bill(t, p), my_own_copy_of_split_bill(t, p))` tests
nothing: both sides have the same bug. State a *different* fact about the result: the sum, the count, the spread.

**2. A strategy that never reaches the bug.** With `"[a-z]{0,40}"` ten equal letters in a row almost never occur, and the faulty
`decode` would pass. Choose strategies so that interesting cases are *likely*: a small alphabet, small ranges, and a few
extremes.

**3. Properties with a hidden precondition.** The round trip `decode(encode(text)) == text` is false for text with digits (`"1"` is
encoded as `"11"`, which is ambiguous). If you widen the strategy to `"[ab1]{0,30}"` the test fails with the minimal input `text = "1"`.
Decide: restrict the inputs (and document it) or change the format. The failure shows a gap in the *specification*, not only in the code.

**4. Using `assert!` instead of `prop_assert!`.** A plain `assert!` also fails the test (`proptest` catches the panic), but the
`prop_` versions are faster, because there is no panic to unwind, and they give a cleaner report. Use them inside `proptest!`.

**5. Expecting the same numbers every run.** The seed is random, so the failing input differs between runs. If you need
to reproduce a failure, use the line `minimal failing input` or the saved file in `proptest-regressions/`.

## Try it yourself

1. Create the package, `src/split.rs` and run the four tests. Then add the tour of strategies and `prop_assume!`.
2. Make the floating-point change of Section 3, read the report, and compare your numbers with the lesson. Revert the change and delete
   `proptest-regressions/split.txt`.
3. Write `src/rle.rs` with the faulty `decode`, see the failure, fix it. Run `PROPTEST_CASES=2000 cargo test`.
4. Write `src/sorting.rs`. Add the broken sort experiment, see it fail, delete it.
5. Run `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`, then commit, including the folder
   `proptest-regressions` if it exists: `git add code/p09-proptest && git commit -m "Add p09-proptest: property tests for splitting, RLE and sorting"`.

## Exercises

### Exercise 1 — One more property (easy)

Add a property to `src/split.rs`: the shares never increase from left to right (each share is greater than or equal to the next).

<details>
<summary>Hint</summary>

Use `shares.windows(2)` and check `pair[0] >= pair[1]` for all pairs, as `is_sorted` does for sorting.

</details>

<details>
<summary>Answer</summary>

```rust
        #[test]
        fn shares_never_increase(total in any::<u64>(), people in 1u64..=1000) {
            let shares = split_bill(total, people);
            prop_assert!(shares.windows(2).all(|pair| pair[0] >= pair[1]));
        }
```

Why it works: the first `total % people` shares are `base + 1` and the rest are `base`, so the list goes down at most once.
This property would catch a bug that gave the extra cents to the last people.

</details>

### Exercise 2 — Properties of `clamp` (medium)

`x.clamp(low, high)` returns `x` limited to the range from `low` to `high`. Write a property test with two facts: the result is
always between `low` and `high`, and if `x` is already in the range, the result is `x`. Generate `low` and `high` so that `low <= high`
always, without `prop_assume!`.

<details>
<summary>Hint</summary>

Generate `low` and a non-negative `width`, then use `high = low + width`. Keep the numbers small (for example
`-1000..1000`) so that the addition cannot overflow. Check with `(low..=high).contains(&x)`.

</details>

<details>
<summary>Answer</summary>

```rust
        #[test]
        fn clamp_stays_inside_the_limits(x in any::<i32>(), low in -1000i32..1000, width in 0i32..1000) {
            let high = low + width;
            let result = x.clamp(low, high);
            prop_assert!(low <= result && result <= high);
            if (low..=high).contains(&x) {
                prop_assert_eq!(result, x);
            }
        }
```

Why it works: building `high` from `low` and a width makes the invalid case (`low > high`, where `clamp` panics) impossible by
construction. This is better than rejecting cases, because no generated case is wasted.

</details>

### Exercise 3 — Digits in the text (hard)

Change the strategy of the RLE round-trip test to `"[ab1]{0,30}"` and run it. What is the minimal failing input, and why does it fail?
Then decide how to fix the problem and write the fix as a test.

<details>
<summary>Hint</summary>

Encode the text `"1"` by hand and then decode the result. Two ways out: forbid digits in the input (a documented precondition), or make
the format unambiguous (for example, encode a digit as an escape sequence).

</details>

<details>
<summary>Answer</summary>

The minimal failing input is `text = "1"`. `encode("1")` gives `"11"` (count 1, character `'1'`), and `decode("11")` reads both
digits as one count and finds no character to repeat, so it returns `""`.

The simplest fix is the documented precondition "the text must not contain digits": say so in the doc comment of `encode`, and keep
the strategy of letters only. As a test of the precondition, state that the function is meant for letters:

```rust
        #[test]
        fn round_trip_for_letters(text in "[a-c]{0,40}") {
            prop_assert_eq!(decode(&encode(&text)), text);
        }
```

A fuller fix would make `encode` return `Result<String, EncodeError>` and refuse digits, with a test that
`encode("a1")` is an `Err`. The property test did its job: it showed that the format has a gap.

</details>

## Quiz

1. What is the difference between an example-based test and a property-based test?

<details>
<summary>Answer</summary>

An example-based test checks one hand-chosen input against one expected output. A property-based test states a rule that must hold for all
valid inputs, and the library checks it on many generated inputs.

</details>

2. What is shrinking?

<details>
<summary>Answer</summary>

After finding a failing input, `proptest` tries smaller and simpler inputs that still fail, and reports the minimal failing input, which is
much easier to understand than the first random one.

</details>

3. Why should you commit the `proptest-regressions` folder?

<details>
<summary>Answer</summary>

It stores the failing inputs that were found. On every later run, by you or by a colleague or by CI (continuous integration, the server that tests every change), they are tried first, so a bug that
was fixed cannot come back unnoticed.

</details>

4. Name three patterns for finding a property.

<details>
<summary>Answer</summary>

For example: the round trip (`decode(encode(x)) == x`), the invariant (the length stays the same), the oracle (compare with the standard library),
idempotence (applying twice equals applying once), and bounds / no crash.

</details>

5. A property says "the result of `sort` is in order". Why is it too weak?

<details>
<summary>Answer</summary>

A function that returns an empty list satisfies it. A good property set also demands that the result has the same elements as the input (the same
length, or equality with an oracle).

</details>

6. What does `prop_assume!` do, and when is it a bad idea?

<details>
<summary>Answer</summary>

It discards the current case if the condition is false. It is a bad idea when it rejects most of the cases, because `proptest` gives up. Build the right
inputs with a better strategy instead.

</details>

## Summary

- A property is a rule that must hold for every valid input. A property-based test checks it on many generated inputs and finds the cases you did not think of.
- With `proptest`: `proptest! { #[test] fn name(x in strategy) { prop_assert!(…); } }`. Strategies: `any::<T>()`, ranges, regular expressions,
  tuples, `vec`, `prop_map`, `prop_oneof!`.
- When a case fails, `proptest` shrinks it to a minimal failing input and saves it in `proptest-regressions/`; commit that folder.
- The default is 256 cases; `PROPTEST_CASES` changes it.
- Patterns: round trip, invariant, oracle, idempotence, bounds / no crash. Ask "what is the stupidest function that passes?" to avoid weak properties.
- Examples document the behaviour, properties search for surprises. Use both.

## Sources and further reading

- [The Proptest Book](https://proptest-rs.github.io/proptest/intro.html) — the official guide: strategies, shrinking, configuration and the `proptest!` macro.
- [proptest on docs.rs](https://docs.rs/proptest/1/proptest/) — the API reference, including the list of strategies.
- [The Rust Standard Library — slice::windows](https://doc.rust-lang.org/std/primitive.slice.html) — the method used to compare neighbours.

---

[🏠 Start](../START-HERE.md) · [← 09.3 Test doubles: fakes, stubs and mocks](03-test-doubles.md) · [📋 Phase 09](README.md) · [09.5 Snapshot testing →](05-snapshot-testing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
